# AAP 2.6 on OpenShift — Migrating from GitHub Enterprise Team authentication to Keycloak (w3) authentication

| Item | Value |
|---|---|
| Platform | Ansible Automation Platform 2.6 (operator-based, OpenShift) |
| AAP URL | `https://aap.prism.dst.ibm.com` |
| Current authenticator | `github-enterprise-team` (type *GitHub Enterprise Team*, team ID `96798`) |
| Target IdP | Keycloak realm `aap-nonpord`, client `aap-np`, brokered to IBM w3 |
| Migration style | Both authenticators run side by side for 1–2 weeks, then GitHub is switched off |
| Source of truth for access | The **existing tenant YAML file, kept in the same format** |

> **Note on the realm name:** the realm is spelled `aap-nonpord` in the request. All examples use it as written. If the real name is `aap-nonprod`, change it in one place, `kc_realm` in the automation vars. Keycloak realm names are case- and spelling-sensitive and appear in every URL.

---

## Table of contents

1. [Summary and key design decisions](#1-summary-and-key-design-decisions)
2. [Architecture](#2-architecture)
3. [AAP 2.6 authentication concepts you need](#3-aap-26-authentication-concepts-you-need)
4. [Prerequisites and information to collect](#4-prerequisites-and-information-to-collect)
5. [Phase 0 — Discovery and backup](#5-phase-0--discovery-and-backup)
6. [Phase 1 — Keycloak configuration](#6-phase-1--keycloak-configuration)
7. [Phase 2 — OpenShift / network / TLS prerequisites](#7-phase-2--openshift--network--tls-prerequisites)
8. [Phase 3 — Create the Keycloak authenticator in AAP (disabled)](#8-phase-3--create-the-keycloak-authenticator-in-aap-disabled)
9. [Phase 4 — Run the tenant sync automation (Keycloak groups + AAP maps)](#9-phase-4--run-the-tenant-sync-automation)
10. [Phase 5 — Enable automatic user migration](#10-phase-5--enable-automatic-user-migration)
11. [Phase 6 — Pilot](#11-phase-6--pilot)
12. [Phase 7 — Enable for everyone: the coexistence window](#12-phase-7--enable-for-everyone-the-coexistence-window)
13. [Phase 8 — Cutover and GitHub decommission](#13-phase-8--cutover-and-github-decommission)
14. [Rollback plan](#14-rollback-plan)
15. [Automation documentation](#15-automation-documentation)
16. [Troubleshooting](#16-troubleshooting)
17. [Assumptions to verify in non-prod first](#17-assumptions-to-verify-in-non-prod-first)

---

## 1. Summary and key design decisions

| # | Decision | Why |
|---|---|---|
| D1 | Use AAP's **Generic OIDC** authenticator pointed at the Keycloak realm. The built-in *Keycloak* type also works (see 8.1). | Generic OIDC uses discovery (`.well-known/openid-configuration`). It picks up Keycloak signing-key rotation automatically, so you never paste a realm public key into AAP. |
| D2 | **Keep the tenant YAML exactly as it is.** The automation turns it into Keycloak groups and their members, plus AAP organizations, teams and authenticator maps. | No change for the people who maintain the YAML. One source of truth. |
| D3 | Naming convention: AAP team `prism-pes-abc-delivery-team` ⇄ Keycloak group `prism-pes-abc-delivery-group`. Org admins ⇄ Keycloak group `<org>-admins-group`, for example `pesabcorg-admins-group`. | This is the mapping you asked for. The suffixes are variables, so the convention can change without code changes. |
| D4 | AAP access is driven by **group-based Authenticator Maps**. These are evaluated every time someone logs in through Keycloak: Allow, Organization Admin, Organization Member, Team Member. | When users log in through Keycloak, AAP grants their roles from their Keycloak groups. Nobody has to add users to AAP by hand. |
| D5 | Set **"Auto migrate users to: keycloak-w3"** on the GitHub authenticator. | When a user first logs in through Keycloak, AAP moves their existing AAP account to the Keycloak authenticator. The account, and every permission an org admin gave it directly, is kept. Without this, AAP would create a second, empty account. |
| D6 | Both authenticators stay **enabled** for 1–2 weeks. Map `revoke` is **off** during coexistence and **on** after cutover. | Users are not affected, and nothing is removed until Keycloak is confirmed as the source of truth. |
| D7 | An **Allow map** only lets in users who belong to at least one managed Keycloak group. | GitHub currently lets in only team 96798. Without an Allow map, *anyone* with a w3 ID could log in to AAP (with no rights). |
| D8 | Keep a **local AAP admin** (the operator-created `admin`) as break-glass access. | Recovery access if the IdP or the maps break. |

---

## 2. Architecture

### 2.1 Current state

```
                       ┌────────────────────────────── OpenShift ──────────────────────────────┐
                       │  AAP 2.6 (AnsibleAutomationPlatform CR)                               │
  ┌────────┐  HTTPS    │  ┌──────────────────────┐   ┌───────────────┐  ┌──────────────────┐   │
  │ User   │──────────▶│  │ Platform Gateway     │──▶│ Controller    │  │ EDA / Hub (opt.) │   │
  │browser │◀──────────│  │  Authenticator:      │   └───────────────┘  └──────────────────┘   │
  └───┬────┘           │  │  github-enterprise-  │   Orgs / Teams / Roles live in the Gateway  │
      │                │  │  team (team 96798)   │   and are synced to the Controller          │
      │ OAuth2         │  └──────────┬───────────┘                                             │
      ▼                │             │ back-channel: token + /api/v3/teams/96798/memberships   │
  ┌──────────────────┐ │             ▼                                                         │
  │ github.ibm.com   │◀┼─────────────┘                                                         │
  │ (OAuth App)      │ │                                                                       │
  └──────────────────┘ └───────────────────────────────────────────────────────────────────────┘

  Tenant automation (today):
  tenant YAML ──▶ playbook ──▶ AAP: create Org, Teams, assign users directly (Org Admin / Team Member)
```

*Authentication* (who can log in) comes from GitHub team 96798.
*Authorization* (which org or team a user is in) comes from direct role assignments made by the tenant automation.

### 2.2 Target state

```
                                   ┌─────────────────────── OpenShift ────────────────────────────┐
                                   │  AAP 2.6                                                     │
 ┌────────┐  1. /login             │  ┌───────────────────────────────────────────────┐           │
 │ User   │───────────────────────▶│  │ Platform Gateway                              │           │
 │browser │                        │  │  Authenticator: keycloak-w3 (Generic OIDC)    │           │
 └─┬──▲───┘                        │  │   ├─ Allow map      (any managed group)       │           │
   │  │                            │  │   ├─ Superuser map  (aap-np-platform-admins)  │           │
   │  │ 2. redirect (auth code)    │  │   ├─ Org Admin map  (<org>-admins-group)      │──▶ Controller
   │  │                            │  │   ├─ Org Member map (any group of the org)    │           │
   │  │ 7. AAP session cookie      │  │   └─ Team map ×N    (prism-pes-abc-x-group)   │           │
   │  │                            │  └───────────────────────▲───────────────────────┘           │
   │  │                            └──────────────────────────┼───────────────────────────────────┘
   │  │                                                       │ 6. code→token exchange (back-channel)
   │  │                                                       │    ID token: preferred_username, email, groups[]
   ▼  │                                                       │
 ┌────────────────────────────────────────────────────────────┴────┐
 │ Keycloak  realm: aap-nonpord                                    │
 │  client aap-np (confidential, standard flow)                    │
 │   └─ dedicated scope mapper "groups" (Group Membership, flat)   │
 │  groups: pesabcorg-admins-group, prism-pes-abc-delivery-group…  │
 │  IdP: w3  ──(Username Template Importer: ${CLAIM.email|lowercase})
 └───────────┬─────────────────────────────────▲───────────────────┘
             │ 3. broker redirect              │ 5. assertion / tokens
             ▼                                 │
       ┌──────────────────────┐  4. w3 login + MFA
       │ IBM w3 IdP (w3id)    │──────────────┘
       └──────────────────────┘

 Tenant automation (new) — same YAML:
  tenant YAML ──▶ 02_sync_tenants.yml ──┬─▶ Keycloak Admin REST: groups + memberships (pre-creates users)
                                        └─▶ AAP Gateway API: Orgs, Teams, Authenticator Maps, Allow map
```

### 2.3 Coexistence window (1–2 weeks)

```
                         ┌─────────────── AAP login page ───────────────┐
                         │  [ Sign in with GitHub ]  [ Sign in with w3 ]│
                         └──────────┬───────────────────────┬───────────┘
                                    │                       │
                    github-enterprise-team             keycloak-w3
                    (Auto migrate users to ──────────────▶ keycloak-w3)
                                    │                       │
                                    ▼                       ▼
                          ┌──────────────────────────────────────────┐
                          │   SAME AAP user object (e.g. user1@ibm.com)│
                          │   - first w3 login re-links the account    │
                          │   - direct roles kept, map roles added     │
                          └──────────────────────────────────────────┘
```

During this window:
* Users who have not moved yet keep using GitHub with no change.
* The first w3 login re-links the existing account to Keycloak.
* The old tenant automation (direct role assignment) can keep running alongside the new Keycloak sync.

### 2.4 Login sequence (Keycloak path)

```
Browser          AAP Gateway                 Keycloak (aap-nonpord)          w3 IdP
   │ GET /           │                              │                          │
   │────────────────▶│ 302 → /realms/aap-nonpord/protocol/openid-connect/auth  │
   │◀────────────────│  ?client_id=aap-np&redirect_uri=<callback>&state=…      │
   │──────────────────────────────────────────────▶│ (kc_idp_hint or chooser) │
   │                                                │ 302 → w3 ───────────────▶│
   │◀──────────────────────────────────────────────────────────── w3 login ───│
   │──────────────────────────────────────────────▶│ broker callback          │
   │                                                │ first-broker-login flow  │
   │◀──────────────── 302 <callback>?code=… ────────│                          │
   │────────────────▶│ POST /token (code, secret) ─▶│                          │
   │                 │◀── id_token{preferred_username, email, groups[]}        │
   │                 │ auto-migrate? → evaluate maps (allow, org, team)        │
   │◀── 302 /, session cookie                       │                          │
```

### 2.5 Naming convention

| YAML element | AAP object | Keycloak group | AAP Authenticator Map(s) |
|---|---|---|---|
| top-level key `pesabcorg` | Organization `pesabcorg` | — | — |
| `pesabcorg.admins` | Organization Admin role in `pesabcorg` | `pesabcorg-admins-group` | `kc-sync \| pesabcorg \| org-admin` (Organization Admin) |
| `pesabcorg.prism-pes-abc-delivery-team` | Team `prism-pes-abc-delivery-team` in `pesabcorg` | `prism-pes-abc-delivery-group` | `kc-sync \| pesabcorg \| prism-pes-abc-delivery-team \| member` (Team Member) |
| `pesabcorg.prism-pes-abc-delivery-managers` | Team `prism-pes-abc-delivery-managers` | `prism-pes-abc-delivery-managers-group`* | `kc-sync \| pesabcorg \| prism-pes-abc-delivery-managers \| member` |
| (all groups of the org) | Organization Member role | — | `kc-sync \| pesabcorg \| org-member` |
| (all managed groups) | Login allowed | — | `kc-sync \| ALLOW \| members of managed groups` |

\* **Important:** `prism-pes-abc-delivery-managers` does **not** end in `-team`, so the `-team → -group` rule does not apply. The automation then appends `-group` (setting `kc_team_group_fallback: append`). It can also fail instead (`fail`). Pick one convention and tell tenants which one it is. Section 15.4 covers this.

---

## 3. AAP 2.6 authentication concepts you need

1. **Authenticator.** An identity source configured in the Platform Gateway: Local, LDAP, SAML, GitHub*, Generic OIDC, Keycloak, and others. Each enabled authenticator gets a button on the login page. The callback URL looks like `https://<gateway>/api/gateway/social/complete/<authenticator-slug>/`. The slug is generated when you create the authenticator. For your GitHub one it is `aap-github-github-enterprise-team`.
2. **Authenticator Map.** A rule on one authenticator that is evaluated at every login through it. It has:
   * **Map type**: `allow`, `is_superuser`, `organization`, `team`, or `role`.
   * **Trigger**: `always`, `never`, `groups` (`has_or` / `has_and` / `has_not`), or `attributes`.
   * **Role**, for example `Organization Admin`, `Organization Member`, `Team Member`, `Team Admin`.
   * **Revoke**: if the trigger is false and revoke is on, the role (or login, for an Allow map) is **removed**.
   * **Order**: maps run in ascending order, and later maps win for the same object.
3. **Create objects**: if on, maps can create the Organization or Team when it doesn't exist. Keep it on. The automation also pre-creates them.
4. **Remove users**: if on, *all* of a user's org and team memberships are wiped at each login before the maps are applied. Keep it **Off**. It would delete permissions that org admins granted directly.
5. **User identity, and why auto-migrate matters.** Every gateway user is linked to an authenticator by an *authenticator user* record (uid plus authenticator). Suppose someone logs in through a *second* authenticator with a username that already belongs to another authenticator. The gateway does **not** reuse that account. It creates a new one with a hash suffix, such as `user1@ibm.com-1a2b3c4d`, which has none of the old permissions. The **Auto migrate users to** setting on the *old* authenticator tells the gateway to move the existing account to the *new* authenticator when the usernames match.
6. **Maps only apply at login.** If you change a Keycloak group, the AAP role changes the next time that user logs in through Keycloak. API tokens that already exist keep the user's current roles until the user logs in again.

---

## 4. Prerequisites and information to collect

| # | Item | Example / where to get it | Owner |
|---|---|---|---|
| P1 | Keycloak base URL (Keycloak ≥ 17 has **no** `/auth` prefix; older versions do) | `https://keycloak.prism.dst.ibm.com` | Keycloak admin |
| P2 | Realm | `aap-nonpord` | — |
| P3 | Client ID / secret for AAP login | `aap-np` / from Credentials tab | Keycloak admin |
| P4 | w3 IdP alias in Keycloak | `w3` / `w3id` (Identity providers list) | Keycloak admin |
| P5 | Which w3 claim/attribute carries the email | OIDC: `email`; SAML: check the IdP mapper/assertion | Keycloak admin |
| P6 | Service-account client for automation | `aap-np-automation` (created in 6.6) | Keycloak admin |
| P7 | AAP admin credential for API (superuser, or a user with rights on authenticators + orgs) | local `admin` | AAP admin |
| P8 | CA chain that signs the Keycloak certificate | PEM file | Platform team |
| P9 | AAP namespace and CR name | `oc get ansibleautomationplatform -A` | OpenShift admin |
| P10 | List of platform superusers | for `aap-np-platform-admins` group | AAP admin |
| P11 | Comms channel to tenants | Slack / mail | Service owner |

Tools: `oc`, `curl`, `jq`, Ansible ≥ 2.15 with collection `ansible.platform` (from Red Hat Automation Hub). If you run the automation from AAP, you need an EE that contains `ansible.platform`.

---

## 5. Phase 0 — Discovery and backup

### 5.1 Back up AAP

```bash
oc project <aap-namespace>
cat <<'EOF' | oc apply -f -
apiVersion: aap.ansible.com/v1alpha1
kind: AnsibleAutomationPlatformBackup
metadata:
  name: pre-keycloak-migration
spec:
  deployment_name: <aap-cr-name>     # from: oc get ansibleautomationplatform
EOF
oc get ansibleautomationplatformbackup pre-keycloak-migration -w   # wait for Successful
```

### 5.2 Export the current authenticator configuration and users

```bash
export AAP=https://aap.prism.dst.ibm.com
export AUTH="-u admin:<password>"     # or: -H "Authorization: Bearer <token>"

# Authenticators (note the id and slug of github-enterprise-team)
curl -sk $AUTH "$AAP/api/gateway/v1/authenticators/" | jq '.results[] | {id,name,slug,type,enabled}'

# Existing maps on the GitHub authenticator (keep a copy)
curl -sk $AUTH "$AAP/api/gateway/v1/authenticator_maps/?page_size=200" > github-maps-backup.json

# All users with username + email — THIS IS THE MOST IMPORTANT CHECK
curl -sk $AUTH "$AAP/api/gateway/v1/users/?page_size=200" \
  | jq -r '.results[] | [.id,.username,.email,.is_superuser] | @tsv' > users-before.tsv
# Follow .next for more pages if you have > 200 users.
```

### 5.3 Decision point: do usernames match?

Open `users-before.tsv` and compare the **username** that AAP currently holds for GitHub users with what Keycloak will send as `preferred_username`. With the mapper in 6.1, that is the lower-case w3 email, for example `user1@ibm.com`.

| What you find | Consequence | Action |
|---|---|---|
| **A.** Usernames are already lower-case emails (`user1@ibm.com`), the same values your YAML uses | Auto-migrate works: same account, all permissions kept | Continue as written |
| **B.** Usernames are GitHub logins (`user1`, `First-Last`, …) that are **not** the email | Auto-migrate cannot match. Keycloak users get **new** accounts | Pick one: (1) make Keycloak emit the same value (only possible if w3 has an attribute equal to the GHE login), or (2) accept new accounts and run a one-time **permission copy** (see 5.4). Because maps restore every org and team membership, only permissions granted *directly to individual users* need copying. |
| **C.** Mixed case (`User1@IBM.com`) | Matching may be case-sensitive | Normalise. The automation and the Keycloak mapper both lower-case values |

Also check that the emails in the YAML are the **primary w3 email** exactly as w3 asserts it. IBM users can have country-specific addresses such as `user1@in.ibm.com`. If the YAML says `user1@ibm.com` but w3 asserts `user1@in.ibm.com`, the user ends up in no group.

### 5.4 Only for case B: plan the permission copy

Direct user permissions live in the role-assignment APIs:
`/api/gateway/v1/role_user_assignments/?user=<id>` and `/api/controller/v2/role_user_assignments/?user=<id>`.

Run a one-time script after each user's first Keycloak login, or in bulk at cutover. For each `(old_user, new_user)` pair matched by email, it re-creates every assignment (`role_definition`, `object_id`) for the new user. Tell org admins to **grant permissions to teams rather than individuals** from now on. Team-based permissions survive any identity change.

### 5.5 Freeze window

Agree that no one changes authenticators or maps manually in AAP during the migration. Changes to the tenant YAML can continue.

---

## 6. Phase 1 — Keycloak configuration

All of this is in the Keycloak admin console, realm **aap-nonpord**. Menu names are from Keycloak / RHBK 22–26.

### 6.1 Make the Keycloak username the lower-case w3 email

**Identity providers → `<w3 alias>` → Mappers → Add mapper**

| Field | Value |
|---|---|
| Name | `username-from-email` |
| Sync mode override | `Import` (sets the username at first login; existing users keep theirs) |
| Mapper type | `Username Template Importer` |
| Template | OIDC IdP: `${CLAIM.email \| lowercase}` · SAML IdP: `${ATTRIBUTE.<email-attribute-name> \| lowercase}` |
| Target | `LOCAL` |

Also on the IdP **Settings** tab:
* **Trust Email** = ON (w3 has already verified it).
* **Sync mode** = `Import`, or `Force` if you want the name and email refreshed at each login.

If users already exist in the realm with other usernames, check them now under **Users**. Usernames that existed before this mapper are not changed.

### 6.2 First-broker-login flow that links to pre-created users

The automation **pre-creates** Keycloak users from the YAML (`kc_precreate_users: true`). This lets a user be placed in their groups *before* their first login, so their first login already has the right access. When that user then arrives from w3, Keycloak must link them to the pre-created account silently, without asking.

**Authentication → Flows → `first broker login` → ⋮ → Duplicate** → name `first broker login - auto link`, then edit it:

| Step | Requirement |
|---|---|
| Review Profile | **Disabled** |
| *User creation or linking* (sub-flow) | Required |
| ├─ Create User If Unique | **Alternative** |
| ├─ Automatically set existing user (*Add step*) | **Alternative** |
| └─ Handle Existing Account (sub-flow) | **Disabled** |

Then **Identity providers → `<w3 alias>` → Settings → First login flow override** = `first broker login - auto link`.

> **Security note.** Auto-linking by email is safe only because w3 is a trusted corporate IdP that verifies emails, and it is the realm's only IdP. Never enable it on a realm that also has a public or social IdP.
>
> If you don't want pre-created users, set `kc_precreate_users: false`. Users who have never logged in to Keycloak are then listed as *pending* by the automation. Their first AAP login is denied by the Allow map, because they are not in any group yet. They can log in after the next sync run. Schedule the sync hourly if you choose this.

### 6.3 Configure client `aap-np`

**Clients → aap-np → Settings**

| Section | Field | Value |
|---|---|---|
| General | Client type | OpenID Connect |
| Capability config | Client authentication | **ON** (confidential) |
| | Authorization | OFF |
| | Standard flow | **ON** |
| | Direct access grants | OFF |
| | Implicit flow | OFF |
| | Service accounts roles | OFF |
| | OAuth 2.0 Device / CIBA | OFF |
| Login settings | Root URL | `https://aap.prism.dst.ibm.com` |
| | Home URL | `https://aap.prism.dst.ibm.com/` |
| | Valid redirect URIs | **(set in Phase 3, step 8.3 — exact callback URL)** |
| | Valid post logout redirect URIs | `https://aap.prism.dst.ibm.com/*` |
| | Web origins | `https://aap.prism.dst.ibm.com` |
| | Front channel logout | OFF |

**Credentials tab:** Client Authenticator = *Client Id and Secret*. Copy the **Client secret**. It goes into the AAP authenticator (`kc_aap_client_secret`).

**Advanced tab:**
* *Proof Key for Code Exchange* — leave empty.
* *Access Token Lifespan* — realm default is fine.

### 6.4 Add the `groups` claim

**Clients → aap-np → Client scopes → `aap-np-dedicated` → Add mapper → By configuration → Group Membership**

| Field | Value |
|---|---|
| Name | `groups` |
| Token Claim Name | `groups` |
| Full group path | **OFF** (claim values become `prism-pes-abc-delivery-group`, not `/prism-pes-abc-delivery-group`) |
| Add to ID token | ON |
| Add to access token | ON |
| Add to userinfo | ON |
| Add to token introspection | ON |

The dedicated scope is always included, so AAP does not need to request an extra scope.

Also confirm that **Client scopes** tab → `profile` and `email` are assigned as *Default*. They provide `preferred_username` and `email`.

### 6.5 Create the groups the automation will manage (optional)

The automation creates any group that is missing. You only need to create one group by hand:
* `aap-np-platform-admins` (optional): members become AAP **superusers**. Add your platform team.

Keep managed groups **top-level**. Don't nest them, because full path is off.

### 6.6 Service-account client for the automation

**Clients → Create client**
* Client ID: `aap-np-automation`
* Client authentication: ON
* Standard flow: OFF
* Direct access grants: OFF
* **Service accounts roles: ON**

**aap-np-automation → Service account roles → Assign role → Filter by clients → `realm-management`.** Assign:
* `manage-users` (create users, manage group membership, create groups)
* `view-users`
* `query-users`
* `query-groups`

Copy the client secret from the **Credentials** tab and store it in Vault, or in an AAP credential (section 15.7).

Quick test:

```bash
KC=https://keycloak.prism.dst.ibm.com; REALM=aap-nonpord
TOKEN=$(curl -s -X POST "$KC/realms/$REALM/protocol/openid-connect/token" \
  -d grant_type=client_credentials -d client_id=aap-np-automation -d client_secret=<secret> | jq -r .access_token)
curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/$REALM/groups?max=5" | jq .
```

### 6.7 Check what tokens will contain

1. Add a test user to a test group.
2. Go to **Clients → aap-np → Client scopes → Evaluate**.
3. Select the user.
4. Open **Generated ID token**.

You should see something like this:

```json
{
  "iss": "https://keycloak.prism.dst.ibm.com/realms/aap-nonpord",
  "aud": "aap-np",
  "preferred_username": "user1@ibm.com",
  "email": "user1@ibm.com",
  "groups": ["prism-pes-abc-delivery-group", "pesabcorg-admins-group"]
}
```

---

## 7. Phase 2 — OpenShift / network / TLS prerequisites

1. **Network reachability.** The gateway pods call Keycloak *server-to-server* to fetch the discovery document, JWKS, and tokens. Test this from inside the pod:
   ```bash
   oc -n <aap-ns> get pods | grep gateway
   oc -n <aap-ns> exec deploy/<aap-cr-name>-gateway -- \
     curl -sv https://keycloak.prism.dst.ibm.com/realms/aap-nonpord/.well-known/openid-configuration -o /dev/null
   ```
   If this fails, check `EgressNetworkPolicy` / `EgressFirewall`, any `NetworkPolicy` in the namespace, and the cluster-wide proxy (`oc get proxy cluster -o yaml`).
2. **TLS trust.** If Keycloak's certificate is signed by an internal IBM CA that is not in the gateway's trust store, add it to the AAP CR:
   ```bash
   oc -n <aap-ns> create secret generic aap-custom-ca \
     --from-file=bundle-ca.crt=./ibm-internal-ca-chain.pem
   oc -n <aap-ns> patch ansibleautomationplatform <aap-cr-name> --type merge \
     -p '{"spec":{"bundle_cacert_secret":"aap-custom-ca"}}'
   ```
   Wait for the operator to roll the pods. Avoid setting *Verify SSL = false* in production.
3. **Clock skew.** Token validation fails if cluster and Keycloak clocks drift by more than about 60 s. Check that NTP is in sync on both sides.
4. **Keycloak hostname.** The `iss` in tokens must equal the provider URL you configure in AAP. If Keycloak sits behind a proxy, make sure `KC_HOSTNAME` (or `hostname` in the Keycloak CR) is set to the public name.

---

## 8. Phase 3 — Create the Keycloak authenticator in AAP (disabled)

Create it **disabled** first, so the w3 button doesn't appear before the maps exist.

### 8.1 Using the UI

**Access Management → Authentication Methods → Create authentication**

| Field | Value |
|---|---|
| Name | `keycloak-w3` (the login button text; users see it) |
| Authentication type | **Generic OIDC** |
| OIDC Provider URL | `https://keycloak.prism.dst.ibm.com/realms/aap-nonpord` (no trailing `/.well-known…`) |
| OIDC Key | `aap-np` |
| OIDC Secret | *client secret from 6.3* |
| Verify OIDC Provider Certificate | ON |
| Username Key | `preferred_username` |
| Groups Claim | `groups` |
| Enabled | **OFF** (for now) |
| Create objects | ON |
| Remove users | **OFF** |

Leave the remaining fields at their defaults: JWT algorithms, response type, token endpoint auth method, and so on.

> **Alternative: the *Keycloak* type.** If you prefer it, you fill in:
> * *Keycloak Access Token URL* `…/realms/aap-nonpord/protocol/openid-connect/token`
> * *Keycloak Provider URL* `…/realms/aap-nonpord/protocol/openid-connect/auth`
> * *Keycloak OIDC Key* `aap-np` and the *Secret*
> * *Keycloak Public Key*: the RS256 key from *Realm settings → Keys*, without the BEGIN/END lines.
>
> The rest of this guide is the same, but you must update the public key whenever Keycloak rotates keys. That is why D1 recommends Generic OIDC.

In the wizard's **Mapping** step, add nothing. The automation creates the maps (Phase 4). Review → **Create**.

### 8.2 Or with automation

```bash
cd automation
ansible-playbook playbooks/01_aap_keycloak_authenticator.yml -e @vault.yml --ask-vault-pass
```

This creates `keycloak-w3` with `enabled: false` and, if configured, the superuser map. See section 15.

### 8.3 Register the callback URL in Keycloak

Open the new authenticator's **Details** page and copy the **Callback URL**. It has the form:

```
https://aap.prism.dst.ibm.com/api/gateway/social/complete/<slug-of-keycloak-w3>/
```

Paste it **exactly** into **Keycloak → Clients → aap-np → Settings → Valid redirect URIs**, including the trailing slash, and save. Don't use wildcards on a production client.

API alternative to read it:

```bash
curl -sk $AUTH "$AAP/api/gateway/v1/authenticators/?name=keycloak-w3" | jq -r '.results[0].configuration.CALLBACK_URL, .results[0].slug'
```

---

## 9. Phase 4 — Run the tenant sync automation

```bash
ansible-playbook playbooks/02_sync_tenants.yml -e @vault.yml --ask-vault-pass
```

For each org in every tenant YAML file, the playbook does the following.

1. **AAP:**
   * Ensures the Organization `pesabcorg` exists.
   * Ensures the Teams `prism-pes-abc-delivery-team` and `prism-pes-abc-delivery-managers` exist.
2. **Keycloak:**
   * Ensures the groups `pesabcorg-admins-group`, `prism-pes-abc-delivery-group` and `prism-pes-abc-delivery-managers-group` exist.
   * Pre-creates missing users (`username = email`).
   * Adds the users from the YAML to those groups.
   * Removes group members that are no longer in the YAML (only if `kc_prune_group_members: true`).
3. **AAP maps** on `keycloak-w3`:
   * `kc-sync | pesabcorg | org-admin`: trigger groups has_or `[pesabcorg-admins-group]` → **Organization Admin**
   * `kc-sync | pesabcorg | org-member`: trigger has_or `[all groups of the org]` → **Organization Member**
   * `kc-sync | pesabcorg | <team> | member`: trigger has_or `[<team-group>]` → **Team Member** of `<team>`
4. **Once, after all orgs:**
   * `kc-sync | ALLOW | members of managed groups`: map type *allow*, `revoke: true`, trigger has_or `[every managed group + aap-np-platform-admins]`. Anyone with a w3 ID who is not in one of these groups is refused.

> **Always run the sync over *all* tenant files.** The Allow map is rebuilt from the union of every group. Running with only one tenant's file would lock out every other tenant at their next Keycloak login.

Check in AAP: **Access Management → Authentication Methods → keycloak-w3 → Mapping**. You should see the maps.

Check in Keycloak: **Groups → prism-pes-abc-delivery-group → Members**. You should see the users.

The run also prints `kc_pending_users`, the users it could not place. Fix them before going further.

---

## 10. Phase 5 — Enable automatic user migration

**Access Management → Authentication Methods → `github-enterprise-team` → Edit**

* **Auto migrate users to** = `keycloak-w3`
* Leave everything else unchanged. In particular, *Enabled* stays **On**.
* Save.

API alternative:

```bash
KC_ID=$(curl -sk $AUTH "$AAP/api/gateway/v1/authenticators/?name=keycloak-w3" | jq '.results[0].id')
GH_ID=$(curl -sk $AUTH "$AAP/api/gateway/v1/authenticators/?name=github-enterprise-team" | jq '.results[0].id')
curl -sk $AUTH -X PATCH -H 'Content-Type: application/json' \
  "$AAP/api/gateway/v1/authenticators/$GH_ID/" -d "{\"auto_migrate_users_to\": $KC_ID}"
```

From now on, the first time a user logs in through **keycloak-w3** with a username that a GitHub-linked account already has, that account is re-linked to Keycloak. The user keeps their ID, username, direct roles, API tokens and owned credentials.

---

## 11. Phase 6 — Pilot

1. Choose 3–5 pilot users covering:
   * an org admin,
   * a team member,
   * a member of two tenants,
   * one superuser.
2. Make sure they appear in the YAML and that the sync has run.
3. **Enable** `keycloak-w3`:
   * UI: *Edit → Enabled: On*.
   * Or with automation: `aap_kc_authenticator_enabled: true` and re-run `01_…yml`.
4. Optional: limit the pilot by restricting the Allow map temporarily, for example by running the sync with `kc_extra_allowed_groups: []`. Or tell only the pilot users about the button.
5. Each pilot user logs in with **Sign in with keycloak-w3**. Check the following.

| Check | Expected |
|---|---|
| User lands in AAP without errors | ✔ |
| Username (top-right / *User details*) | **same** as before (`user1@ibm.com`), no `-<hash>` suffix |
| *User details → Authentication* / authenticator | `keycloak-w3` |
| Organizations / Teams | as defined in YAML |
| Job templates the user could run before | still runnable |
| Permissions an org admin granted directly to this user | still present |
| A non-listed w3 user tries to log in | refused ("access denied"/"not allowed") |
| Gateway log | no tracebacks (see 16) |

6. Compare before and after:
   ```bash
   curl -sk $AUTH "$AAP/api/gateway/v1/users/?username=user1@ibm.com" | jq '.results[] | {id,username,email}'
   # id must be the same as in users-before.tsv
   ```

**Go / No-go:**
* **Go** if the user IDs did not change and all the checks pass.
* **No-go** if a `-<hash>` account appeared. Go back to 5.3: the usernames don't match. Delete the duplicate user and fix the username mapping first.

---

## 12. Phase 7 — Enable for everyone: the coexistence window

**Duration:** 1–2 weeks.

1. **Announce it** (template):
   > *From <date>, please sign in to AAP with the **keycloak-w3** button (w3 login). Your existing account, jobs and permissions carry over automatically. The GitHub button will be removed on <date + 14>. Once you have used keycloak-w3, please keep using it.*
2. **Keep both automations running:**
   * The new `02_sync_tenants.yml` runs on a schedule, for example hourly, and on every YAML change.
   * The old GitHub-era tenant automation can keep making direct assignments until cutover. It works on the same user objects.
3. **Keep these settings during the window:**
   * `aap_map_revoke: false`: maps only *add* roles and never remove them, so nothing disappears unexpectedly.
   * `kc_prune_group_members: true`: Keycloak groups follow the YAML exactly.
4. **Monitor progress every day.** Count how many users are still linked to GitHub:
   ```bash
   curl -sk $AUTH "$AAP/api/gateway/v1/authenticators/$GH_ID/users/?page_size=200" | jq '.count'
   curl -sk $AUTH "$AAP/api/gateway/v1/authenticators/$KC_ID/users/?page_size=200" | jq '.count'
   ```
   If this related endpoint isn't available in your build, open each authenticator in the UI and check its **Users** tab.
5. **Look for duplicates every day:**
   ```bash
   curl -sk $AUTH "$AAP/api/gateway/v1/users/?page_size=200&username__icontains=-" | jq -r '.results[].username'
   ```
   Any `name-<8 hex>` account is a duplicate. Investigate it, then delete it.
6. **Remind** users who have not migrated in the middle and at the end of the window.

> Users who already logged in with keycloak-w3 should **not** go back to the GitHub button. Their account now belongs to keycloak-w3, so a GitHub login may create a second, empty account. Confirm this in the pilot and say it in the announcement.

---

## 13. Phase 8 — Cutover and GitHub decommission

**Day 0 (end of the window):**

1. Run the sync once more and confirm that `kc_pending_users` is empty.
2. Switch **off** the `github-enterprise-team` authenticator (*Edit → Enabled: Off*). The button disappears. **Don't delete it yet.** That keeps the accounts that haven't migrated and allows a rollback.
3. Turn revoke on: set `aap_map_revoke: true` in `group_vars` and re-run `02_sync_tenants.yml`. From now on, a user removed from the YAML loses that org or team role at their next login.
4. Retire the direct-assignment tasks of the old tenant automation. The YAML now flows only through Keycloak groups and maps.
5. Optional: so users skip the Keycloak account chooser and go straight to w3, add `kc_idp_hint=<w3 alias>` to the authorization request. In Generic OIDC this is an extra auth parameter. Or set the w3 IdP as the default in **Authentication → browser flow → Identity Provider Redirector**.

**Day 0 + 7 to 14:**

6. Handle the remaining GitHub-only users. These are users who never logged in through Keycloak in the window:
   * They can still log in through Keycloak. Because the authenticator is only disabled, auto-migrate still applies.
   * For leavers, delete their accounts after checking who owns their resources.
7. **Delete** the `github-enterprise-team` authenticator and its maps.
8. In github.ibm.com, delete or revoke the OAuth App whose client ID is `ea3060887fcd501d828a`.
9. Remove the GitHub OAuth secret from Vault and CI.
10. Update runbooks and onboarding docs, and remove the GitHub team 96798 dependency.

---

## 14. Rollback plan

| When | Rollback |
|---|---|
| Before Phase 6 (authenticator disabled) | Nothing is visible to users. Delete `keycloak-w3` and its maps if you want. |
| During pilot or coexistence | 1. Disable `keycloak-w3`. 2. On `keycloak-w3`, set **Auto migrate users to = github-enterprise-team**. A user's next GitHub login then moves the account back. 3. Remove the auto-migrate setting from the GitHub authenticator. *Test this in non-prod first.* |
| After cutover | Re-enable `github-enterprise-team` and apply the steps above. The GitHub OAuth App must still exist, which is why deletion waits 1–2 weeks. |
| Catastrophic | Restore the `pre-keycloak-migration` backup using an `AnsibleAutomationPlatformRestore` CR. You lose all changes made since the backup. |

Break-glass access: the local `admin` user, created by the operator, always works on the *Local* authenticator. Get its password with:

```bash
oc get secret <aap-cr-name>-admin-password -o jsonpath='{.data.password}' | base64 -d
```

---

## 15. Automation documentation

The code is in [`automation/`](automation/) next to this guide.

### 15.1 Layout

```
automation/
├── ansible.cfg
├── collections/requirements.yml          # ansible.platform
├── inventory/hosts.yml                   # localhost only
├── orgs/
│   └── pesabcorg.yml                     # ← tenant YAML, UNCHANGED format (one or many orgs per file)
├── vault.yml.example                     # secrets template (ansible-vault encrypt it)
└── playbooks/
    ├── group_vars/all/main.yml           # URLs, naming conventions, switches
    ├── 01_aap_keycloak_authenticator.yml # one-time: create/update keycloak-w3 + superuser map
    ├── 02_sync_tenants.yml               # recurring: YAML → Keycloak groups/members + AAP orgs/teams/maps
    ├── filter_plugins/tenant_model.py    # validates YAML, derives group names, detects collisions
    └── tasks/
        ├── process_org.yml               # per org
        ├── kc_token.yml                  # service-account token
        ├── kc_sync_group.yml             # ensure group + reconcile members
        └── kc_add_member.yml             # look up / pre-create user, add to group
```

### 15.2 Tenant YAML: unchanged

```yaml
pesabcorg:                          ## AAP Org name
  admins:                           ## AAP Org admins
    - user1@ibm.com
    - user2@ibm.com
  prism-pes-abc-delivery-team:      ## AAP org team
    - user1@ibm.com
    - user3@ibm.com
  prism-pes-abc-delivery-managers:  ## AAP org team
    - user2@ibm.com
```

Rules the automation enforces:
* Every top-level key is an AAP Organization.
* The `admins` key is reserved and means Org Admins.
* Every other key is a Team.
* Values are lists of w3 emails. They are lower-cased and de-duplicated.
* Empty lists are allowed. Writing `prism-pes-abc-x-team: []` keeps the team and empties the group.
* One file can contain several orgs, or you can use one file per tenant. All files matching `tenant_files_glob` are processed together.

### 15.3 Configuration variables (`playbooks/group_vars/all/main.yml`)

| Variable | Default | Meaning |
|---|---|---|
| `aap_hostname` | `https://aap.prism.dst.ibm.com` | Gateway URL |
| `aap_validate_certs` | `true` | TLS verification towards AAP |
| `aap_kc_authenticator_name` | `keycloak-w3` | Name of the Generic OIDC authenticator |
| `aap_kc_authenticator_enabled` | `false` | Set `true` at pilot |
| `aap_map_revoke` | `false` | `false` during coexistence, `true` after cutover |
| `aap_superuser_group` | `aap-np-platform-admins` | Keycloak group → AAP superuser (empty string disables) |
| `kc_url` | `https://keycloak.prism.dst.ibm.com` | Keycloak base URL (add `/auth` for Keycloak < 17) |
| `kc_realm` | `aap-nonpord` | Realm |
| `kc_aap_client_id` | `aap-np` | Login client |
| `kc_automation_client_id` | `aap-np-automation` | Service-account client |
| `kc_validate_certs` | `true` | TLS verification towards Keycloak |
| `aap_team_suffix` | `-team` | Suffix of AAP team names |
| `kc_team_group_suffix` | `-group` | Replacement suffix for Keycloak groups |
| `kc_team_group_fallback` | `append` | For team names not ending in `aap_team_suffix`: `append` the group suffix, or `fail` |
| `kc_org_admin_group_suffix` | `-admins-group` | Org admins group = `<org><suffix>` |
| `kc_precreate_users` | `true` | Pre-create missing users in Keycloak (needs 6.2) |
| `kc_prune_group_members` | `true` | Remove Keycloak group members that aren't in the YAML |
| `kc_extra_allowed_groups` | `[]` | Extra groups to allow in the Allow map |
| `aap_manage_allow_map` | `true` | Manage the global Allow map |
| `tenant_files_glob` | `{{ playbook_dir }}/../orgs/*.yml` | Where the tenant YAML files are |
| `aap_map_prefix` | `kc-sync` | Prefix of every map name the automation creates; maps without it are never touched |
| `aap_prune_maps` | `true` | Delete `kc-sync \| …` maps whose org/team was removed from the YAML |
| `aap_set_github_auto_migrate` | `false` | `true` lets `01_…yml` set *Auto migrate users to* on the GitHub authenticator (Phase 5) |
| `aap_github_authenticator_name` | `github-enterprise-team` | Name of the existing GitHub authenticator |

Secrets (`vault.yml`, or AAP credentials): `aap_username`, `aap_password`, `kc_aap_client_secret`, `kc_automation_client_secret`.

### 15.4 Name derivation

```
team key  prism-pes-abc-delivery-team      → group prism-pes-abc-delivery-group        (suffix replaced)
team key  prism-pes-abc-delivery-managers  → group prism-pes-abc-delivery-managers-group (fallback: append)
admins of pesabcorg                        → group pesabcorg-admins-group
```

If you would rather fail on team names that don't follow the convention, set `kc_team_group_fallback: fail`. The run then stops with a clear message.

### 15.5 Idempotency and safety

* Before anything changes, the input is validated:
  * invalid emails fail the run,
  * an org defined in two files fails the run,
  * two teams that would map to the same Keycloak group fail the run.

  `--tags model` prints the derived mapping without contacting AAP or Keycloak.
* Map names are deterministic, `kc-sync | <org> | <team> | member`, so re-runs update maps in place. With `aap_prune_maps: true`, stale `kc-sync` maps are deleted.
* AAP objects are created or updated with `state: present`. The sync never **deletes** AAP orgs or teams. Removing a team from the YAML only stops new grants: the Keycloak group is emptied, and with `revoke: true` members lose the role at their next login. Delete orphan teams manually after review.
* Keycloak groups are created if missing. Members are reconciled to match the YAML exactly when `kc_prune_group_members` is true.
* All secret-bearing tasks use `no_log: true`.
* The service-account token is refreshed for each org, because the default access-token lifetime is only 5 minutes.
* Group member listing uses `max=1000`. Raise `kc_page_max` if a group can be larger.

### 15.6 Running from the CLI

```bash
cd automation
ansible-galaxy collection install -r collections/requirements.yml   # needs Automation Hub token in ansible.cfg/env
cp vault.yml.example vault.yml && ansible-vault encrypt vault.yml
ansible-playbook playbooks/01_aap_keycloak_authenticator.yml -e @vault.yml --ask-vault-pass
ansible-playbook playbooks/02_sync_tenants.yml            -e @vault.yml --ask-vault-pass
# preview the derived model only, no changes:
ansible-playbook playbooks/02_sync_tenants.yml -e @vault.yml --ask-vault-pass --tags model
```

### 15.7 Running from AAP (recommended for the schedule)

1. **Credential types.** Go to *Automation Execution → Infrastructure → Credential Types → Create*.

   *AAP Gateway API*
   ```yaml
   # Input configuration
   fields:
     - {id: host, type: string, label: Gateway URL}
     - {id: username, type: string, label: Username}
     - {id: password, type: string, label: Password, secret: true}
   required: [host, username, password]
   # Injector configuration
   extra_vars:
     aap_hostname: '{{ host }}'
     aap_username: '{{ username }}'
     aap_password: '{{ password }}'
   ```
   *Keycloak automation client*
   ```yaml
   fields:
     - {id: client_id, type: string, label: Client ID}
     - {id: client_secret, type: string, label: Client secret, secret: true}
     - {id: login_client_secret, type: string, label: aap-np client secret, secret: true}
   required: [client_id, client_secret]
   extra_vars:
     kc_automation_client_id: '{{ client_id }}'
     kc_automation_client_secret: '{{ client_secret }}'
     kc_aap_client_secret: '{{ login_client_secret }}'
   ```
2. **Credentials.** Create one of each type. Use a dedicated local AAP service user that is a superuser (or has Platform Auditor plus admin on orgs and authenticators).
3. **Project.** Point it at the Git repository that holds `automation/` and the tenant YAML. The project needs an *Ansible Galaxy/Automation Hub* credential on its organization so that `collections/requirements.yml` can be installed. Otherwise use an EE that already contains `ansible.platform`.
4. **Job templates:**
   * `AAP | Keycloak authenticator`, using `playbooks/01_aap_keycloak_authenticator.yml`. Run it manually.
   * `AAP | Tenant sync`, using `playbooks/02_sync_tenants.yml`. Inventory: any inventory containing `localhost`. Credentials: both of the above.
5. **Triggers:**
   * A schedule, hourly or daily.
   * Optionally a webhook on the tenant-YAML repository: *Enable webhook → GitHub/GitLab*, so each merged change syncs right away.

### 15.8 Tenant onboarding: new flow

1. The tenant owner adds a YAML block or file, `pesxyzorg` with its teams, through a PR.
2. After the merge, the webhook or schedule runs `AAP | Tenant sync`. It:
   * creates the org and teams in AAP,
   * creates the groups in Keycloak and adds the users,
   * creates the maps and refreshes the Allow map.
3. The org admin grants job-template, inventory and credential permissions **to teams** in AAP, as today.
4. Users log in with keycloak-w3 and immediately get their org and teams.

---

## 16. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Keycloak shows *Invalid parameter: redirect_uri* | Callback URL in AAP ≠ *Valid redirect URIs* | Copy the exact value, including the trailing `/` (8.3) |
| AAP error after w3 login: *SSL: CERTIFICATE_VERIFY_FAILED* | Gateway doesn't trust the Keycloak CA | Use `bundle_cacert_secret` (7.2) |
| *Token is invalid / iss mismatch* | Keycloak hostname differs from the provider URL | Fix `KC_HOSTNAME`, or use the exact public URL |
| User logs in but has no orgs or teams | No `groups` claim, wrong claim name, or full path ON | 6.4 and 6.7. *Groups Claim* = `groups` in AAP |
| User refused ("not allowed") | Not in any managed group (Allow map) | Add them to the YAML and re-run the sync. Check `kc_pending_users` |
| New account `user1@ibm.com-1a2b3c4d` created | Auto-migrate not set, or usernames differ | Phase 5 and 5.3. Delete the duplicate after fixing |
| User is in a Keycloak group but the role didn't change | Maps apply only at login | Log out of AAP and log in again |
| Sync fails with 403 on `/admin/realms/...` | Service-account roles missing | 6.6 |
| Sync: *409 Conflict* creating a user | A user exists with that email but a different username | Fix that user in Keycloak. Usernames should equal the lower-case email |
| Removed user still has a role after cutover | `aap_map_revoke` still false, or they haven't logged in yet | Set `true` and re-run. The role is removed at their next login |

Logs:

```bash
oc -n <aap-ns> logs deploy/<aap-cr-name>-gateway --all-containers --since=30m | grep -iE 'authenticat|social|oidc|migrat|map'
```

On the Keycloak side, look at **Realm settings → Events → User events**. Enable the *LOGIN_ERROR* and *IDENTITY_PROVIDER_** events.

---

## 17. Assumptions to verify in non-prod first

Run the whole procedure in a non-prod AAP before production. These behaviours depend on your exact AAP 2.6 patch level and Keycloak version:

1. **Auto-migrate matching.** It matches on the username, so the Keycloak `preferred_username` must equal the existing username. Confirm in the pilot that the user **ID** stays the same (Phase 6).
2. **What happens when a migrated user goes back to the GitHub button.** It may create a duplicate account.
3. **The Allow map `revoke: true`.** Confirm that it really refuses w3 users who aren't in any managed group.
4. **`ansible.platform` module options.** The playbooks use `organization`, `team`, `authenticator` and `authenticator_map`. Run `ansible-doc ansible.platform.authenticator_map` against the collection version in your EE.
5. **The `GET /api/gateway/v1/authenticators/<id>/users/` endpoint**, used for monitoring.
6. **The Keycloak auto-link first-broker-login flow** with pre-created users: no prompt, and the account is linked.
7. **Group naming for teams that don't end in `-team`** (15.4). Agree on a convention with the tenants.
