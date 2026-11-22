# AAP 2.6 on OpenShift — Migrating from GitHub Enterprise Team authentication to Keycloak (w3) authentication — v2

> **v2** updates [MIGRATION_GUIDE.md](MIGRATION_GUIDE.md) (v1) with the Phase 0–2 findings from the AAP and Keycloak environment.
>
> Steps that the findings made unnecessary have been removed. Steps that turned out to be wrong have been corrected. [Section 13](#13-what-changed-from-v1-and-why) lists every change. v1 is kept unchanged for reference.

| Item | Value (from findings) |
|---|---|
| AAP URL | `https://aap-prod-latest.prism-np.dst.ibm.com` |
| AAP CR / gateway deployment | `aap-np` / `deploy/aap-np-gateway` |
| Keycloak URL | `https://keycloak.prism-np.dst.ibm.com` (Keycloak ≥ 17, no `/auth` prefix) |
| Realm | **`prism-nonprod`** (confirmed: discovery returns HTTP 200 from the gateway pod). `aap-nonpord` from `query.md` does not exist (F9, F10) |
| Login client | `aap-np` (to be confirmed against authenticator 69, step 0.4) |
| Old authenticator | id **2** `github-enterprise-team`, slug `aap-github-github-enterprise-team`, enabled |
| New authenticator | id **69** `Keycloak SSO`, type *Keycloak*, slug `c221bc1f-b5bf-402f-8727-f938ae656956`, **already enabled**. Reused; nothing new is created |
| Other | id 1 Local (break-glass `admin`); id 35 `IBM-W3-OIDC` (direct w3 OIDC, disabled, not part of this migration) |
| Users | 253 (2 API pages) |
| Username situation | AAP usernames are **GitHub logins**. Keycloak will send the **email local part**. They don't match |

---

## Table of contents

1. [Validation of the findings](#1-validation-of-the-findings)
2. [Architecture (updated)](#2-architecture-updated)
3. [Phase 0 — Discovery and backup (remaining steps)](#3-phase-0--discovery-and-backup-remaining-steps)
4. [Phase 1 — Keycloak configuration](#4-phase-1--keycloak-configuration)
5. [Phase 2 — Network / TLS re-check](#5-phase-2--network--tls-re-check)
6. [Phase 3 — Prepare the existing "Keycloak SSO" authenticator (id 69)](#6-phase-3--prepare-the-existing-keycloak-sso-authenticator-id-69)
7. [Phase 4 — Run the tenant sync](#7-phase-4--run-the-tenant-sync)
8. [Phase 5 — Pilot, including the permission copy](#8-phase-5--pilot-including-the-permission-copy)
9. [Phase 6 — Coexistence window (1–2 weeks)](#9-phase-6--coexistence-window-12-weeks)
10. [Phase 7 — Cutover and GitHub decommission](#10-phase-7--cutover-and-github-decommission)
11. [Rollback](#11-rollback)
12. [Automation documentation (v2 changes)](#12-automation-documentation-v2-changes)
13. [What changed from v1 and why](#13-what-changed-from-v1-and-why)
14. [Troubleshooting](#14-troubleshooting)

---

## 1. Validation of the findings

| # | Finding | Verdict | Consequence |
|---|---|---|---|
| F1 | AAP is `aap-prod-latest.prism-np.dst.ibm.com`, CR `aap-np` | ✔ Accepted | All URLs and commands updated |
| F2 | Authenticator **69 "Keycloak SSO"** (type *Keycloak*) exists and is **enabled** | ⚠ **Important** | (a) Reuse it instead of creating a new Generic OIDC authenticator. Its button is already on the login page, so no new slug or callback is needed. (b) Because it is already enabled, **check now** who uses it and what maps it has (step 0.4). Without an *Allow* map, anyone who can log in to the realm can get into AAP. |
| F3 | Authenticator 35 `IBM-W3-OIDC` is disabled | ✔ | Not used. Leave it disabled. Delete it at decommission if no users are linked to it (step 10.7). |
| F4 | `count: 253`, `next` set → 2 pages | ✔ | The v1 one-page `curl` saved only 200 users. Step 0.3 replaces it with a paging export. |
| F5 | Usernames are GitHub logins, not emails → *case B* | ✔ **Conclusion correct** | Auto-migrate cannot match, so each user gets a **new** AAP account at their first Keycloak login. Permissions come back through (a) the group maps (org admin, org member, team member, from the YAML), and (b) a one-time **permission copy** for roles granted directly to users (step 8.3). **New: do NOT set "Auto migrate users to"** (see F5a). |
| F5a | (consequence of F5) | ✖ Remove v1 Phase 5 | With mismatched usernames, auto-migrate brings nothing. It is also **dangerous**: if a GitHub login happens to equal *another* person's email local part (for example GHE login `jsmith` vs `jsmith@ibm.com`), auto-migrate would give that AAP account, with its permissions, to the wrong person. Without auto-migrate, such a clash only produces a suffixed new account (`jsmith-<hash>`). That is safe, and the precheck in step 0.5 finds it. |
| F6 | IdP mapper `oidc-bluepages`: *Username Template Importer*, `${CLAIM.email \| localpart}`, target `BROKER_USERNAME`, sync mode *inherit* (IdP = *legacy*) | ✔ **Works. Keep it; don't change it** | The Keycloak username, and therefore the AAP username, will be the **local part**: `rakesh.upadhyay1`. Since auto-migrate isn't used, the v1 "lower-case full email" mapper is not needed. Changing it now would also rename nothing (sync mode legacy = import only) and would leave mixed user names. Two remaining risks, both checked by tooling: **local-part collisions** (`user1@ibm.com` vs `user1@in.ibm.com`) and a local part equal to an existing GitHub username of someone else. `BROKER_USERNAME` vs `LOCAL` makes no difference for new users: the local username is taken from the brokered username at first login. |
| F7 | IdP *Trust Email* = ON | ✔ | Keep it. The w3 email is marked verified, so the sync can rely on email to find users |
| F8 | Realm already has **PRISM first login flow**, which another application uses (earlier read as the default flow) | ✔ **Keep it unchanged** | Keycloak chooses the first login flow per identity provider (or realm-wide), not per client, so AAP can't have its own. AAP doesn't need a change: the sync no longer pre-creates users (`kc_precreate_users: false`), so AAP users go through *Create user if unique* exactly like the other application's new users. Pre-creating users with local-part usernames would be an account-takeover risk. Details and options in step 1.2. |
| F9 | Phase 2 marked "All set" | ✖ **Not all set** | **TLS and network: OK.** The gateway pod reached Keycloak and the certificate (`*.prism-np.dst.ibm.com`, issuer *IBM INTERNAL INTERMEDIATE CA*) verified against the pod's default bundle, so the v1 custom CA bundle step is removed. **But the request returned `HTTP/1.1 404`** with a 32-byte JSON body. That is exactly the size of Keycloak's `{"error":"Realm does not exist"}`. The realm **`aap-nonpord` does not exist**. See F10 for the correct name. |
| F10 | Correct realm is **`prism-nonprod`**: `GET /realms/prism-nonprod/.well-known/openid-configuration` from `aap-np-gateway` → `HTTP/1.1 200 OK`, 6879-byte JSON, TLS verified | ✔ **Resolved** | Realm used everywhere in this guide and in `automation/vars/np.yml`. **Still to do:** step 0.4 must confirm that authenticator 69's `ACCESS_TOKEN_URL` / `AUTHORIZATION_URL` point at `/realms/prism-nonprod/`. If they point at another realm, 69 is logging people in through that realm today; re-point it in step 3.1 and also check the client, groups and IdP there. All Phase 1 Keycloak settings (IdP mapper, first-broker-login flow, client `aap-np`, groups) must be checked **in `prism-nonprod`**. The findings in F6–F8 are valid only if they were read from this realm. |

**Bottom line:** the migration becomes *"new accounts plus permission copy"* instead of *"same account re-linked"*.

That is actually safer:
* The **old GitHub accounts are never modified**.
* Rollback is simply "use the GitHub button".

In return, three things must be handled:
* the permission copy (step 8.3),
* personal API tokens, which are not transferable (step 9.4),
* the clean-up of old accounts (step 10.6).

---

## 2. Architecture (updated)

### 2.1 Target state

```
                                   ┌─────────────────────────── OpenShift (aap-np) ───────────────────────────┐
 ┌────────┐  1. Sign in with       │  Platform Gateway  aap-prod-latest.prism-np.dst.ibm.com                  │
 │ User   │     "Keycloak SSO"     │  ┌──────────────────────────────────────────────────────────┐            │
 │browser │───────────────────────▶│  │ Authenticator 69 "Keycloak SSO" (type Keycloak)          │            │
 └─┬──▲───┘                        │  │  maps (kc-sync | …):                                     │            │
   │  │ 2. redirect                │  │   ALLOW      any managed group          (revoke: true)   │──▶ Controller
   │  │                            │  │   superuser  aap-np-platform-admins                      │            │
   │  │ 7. session                 │  │   org-member any group of <org>                          │            │
   │  │                            │  │   org-admin  <org>-admins-group                          │            │
   │  │                            │  │   team ×N    prism-pes-abc-…-group → …-team              │            │
   │  │                            │  └───────────────────────────────▲──────────────────────────┘            │
   │  │                            │  Authenticator 2 github-enterprise-team (until cutover, untouched)       │
   │  │                            └──────────────────────────────────┼───────────────────────────────────────┘
   │  │                                                               │ 6. code → access token (RS256, verified
   │  │                                                               │    with realm PUBLIC_KEY, aud = client)
   ▼  │                                                               │    preferred_username = email local part
 ┌────────────────────────────────────────────────────────────────────┴────┐   groups = [...]
 │ Keycloak keycloak.prism-np.dst.ibm.com  realm prism-nonprod             │
 │  client aap-np: groups mapper (access token) + audience mapper          │
 │  groups: pesabcorg-admins-group, prism-pes-abc-delivery-group, …       │
 │  IdP (OIDC, w3/bluepages): mapper oidc-bluepages ${CLAIM.email|localpart}
 │  first-login flow: "PRISM first login flow" (shared, unchanged)        │
 └───────────┬─────────────────────────────────▲──────────────────────────┘
             │ 3. broker                       │ 5. tokens
             ▼                                 │
       ┌──────────────────────┐  4. w3 login + MFA
       │ IBM w3 ID (OIDC)     │──────────────┘
       └──────────────────────┘

 Automation (same tenant YAML):
   orgs/*.yml ──▶ 02_sync_tenants.yml ──┬─▶ Keycloak Admin API: groups, members (users found by e-mail, never pre-created)
                                        └─▶ AAP Gateway API: orgs, teams, kc-sync maps on authenticator 69
   users-before.json ──▶ aap_user_migration.py (snapshot / precheck / suggest / copy) ──▶ direct role copy old→new
```

### 2.2 Identity model during coexistence

```
            GitHub button (authenticator 2)              Keycloak SSO button (authenticator 69)
                       │                                              │
                       ▼                                              ▼
       ┌───────────────────────────────┐            ┌───────────────────────────────────┐
       │ OLD account  "Rakesh-Upadhyay1"│  copy of   │ NEW account  "rakesh.upadhyay1"    │
       │  email rakesh.upadhyay1@ibm.com│──direct───▶│  email rakesh.upadhyay1@ibm.com    │
       │  direct roles + old automation │  roles     │  org/team roles from group maps    │
       │  personal API tokens           │ (script)   │  + copied direct roles             │
       │  ── never modified ──          │            │  new personal tokens if needed     │
       └───────────────────────────────┘            └───────────────────────────────────┘
                 paired by e-mail (aap_user_migration.py suggest)
```

### 2.3 Where each permission comes from on the new account

| Permission type | Source on the new account |
|---|---|
| Org Admin of `pesabcorg` | map `kc-sync \| pesabcorg \| org-admin` ← Keycloak group `pesabcorg-admins-group` ← YAML `admins` |
| Member of team `prism-pes-abc-delivery-team` | map `kc-sync \| pesabcorg \| prism-pes-abc-delivery-team \| member` ← group `prism-pes-abc-delivery-group` ← YAML |
| Org Member | map `kc-sync \| pesabcorg \| org-member` ← any group of the org |
| Anything granted **to a team** (JT execute, inventory use, …) | Inherited automatically through team membership. **No copy needed** |
| Anything granted **directly to the user** | `aap_user_migration.py copy` (step 8.3) |
| Superuser | map `kc-sync \| superuser` ← group `aap-np-platform-admins` ← `aap_superuser_members` |
| Personal API tokens | **Not transferable.** The user creates new ones |

### 2.4 Naming convention (unchanged from v1)

| YAML | AAP | Keycloak group |
|---|---|---|
| `pesabcorg:` | Organization `pesabcorg` | — |
| `pesabcorg.admins` | Org Admin | `pesabcorg-admins-group` |
| `prism-pes-abc-delivery-team` | Team `prism-pes-abc-delivery-team` | `prism-pes-abc-delivery-group` |
| `prism-pes-abc-delivery-managers` (no `-team`) | Team `prism-pes-abc-delivery-managers` | `prism-pes-abc-delivery-managers-group` (fallback `append`) |

---

## 3. Phase 0 — Discovery and backup (remaining steps)

```bash
export AAP=https://aap-prod-latest.prism-np.dst.ibm.com
export AUTH="-u admin:<password>"
oc project <aap-namespace>
```

### 0.1 Back up AAP (if not done yet)

```bash
cat <<'EOF' | oc apply -f -
apiVersion: aap.ansible.com/v1alpha1
kind: AnsibleAutomationPlatformBackup
metadata:
  name: pre-keycloak-migration
spec:
  deployment_name: aap-np
EOF
oc get ansibleautomationplatformbackup pre-keycloak-migration -w     # wait for Successful
```

### 0.2 Back up every authenticator and all maps

```bash
mkdir -p backup && cd backup
for id in 1 2 35 69; do curl -sk $AUTH "$AAP/api/gateway/v1/authenticators/$id/" > authenticator-$id.json; done
curl -sk $AUTH "$AAP/api/gateway/v1/authenticator_maps/?page_size=200" > authenticator-maps.json
jq -r '.results[] | [.id,.authenticator,.order,.map_type,.name,.revoke] | @tsv' authenticator-maps.json
```

### 0.3 Export all 253 users, with paging

The v1 command saved only page 1 (200 users). Use either of these.

**Option A, the script:** it saves JSON and is also used later by `precheck`, `suggest` and `copy`.

```bash
cd ../automation
export AAP_URL=$AAP AAP_USER=admin AAP_PASSWORD='<password>'
python3 scripts/aap_user_migration.py --insecure snapshot --out ../backup/users-before.json
# → "saved 253 users ...; N without e-mail; M superusers"
```

**Option B, `curl` + `jq`:**

```bash
url="/api/gateway/v1/users/?page_size=200"; : > users-before.tsv
while [ "$url" != "null" ]; do
  r=$(curl -sk $AUTH "$AAP$url")
  echo "$r" | jq -r '.results[] | [.id,.username,.email,.is_superuser] | @tsv' >> users-before.tsv
  url=$(echo "$r" | jq -r .next)
done
wc -l users-before.tsv          # must be 253
```

Take the snapshot **before the pilot (Phase 5)**. `suggest` uses it as a fallback to tell old accounts from new ones.

From the export, note:
* **Users without an email.** They cannot be paired automatically. Fix their email in AAP, or handle them by hand.
* **Superusers** (`is_superuser = true`, excluding `admin`). Their emails go into `aap_superuser_members` (step 4.1).

### 0.4 Inspect authenticator 69 (Keycloak SSO) and check that it uses realm `prism-nonprod`

```bash
jq '{name,enabled,create_objects,remove_users,auto_migrate_users_to,configuration}' authenticator-69.json
# quick realm check - must print prism-nonprod (twice)
jq -r '.configuration | to_entries[] | select(.value|tostring|test("/realms/")) | .value' authenticator-69.json \
  | sed -E 's#.*/realms/([^/]+)/.*#\1#'
```

You'll see fields like `ACCESS_TOKEN_URL`, `AUTHORIZATION_URL`, `KEY`, `PUBLIC_KEY`, `SECRET` (`$encrypted$`), and, depending on the version, `GROUPS_CLAIM`. Record:

| Field | What it tells you |
|---|---|
| `ACCESS_TOKEN_URL` / `AUTHORIZATION_URL` | Must contain `/realms/prism-nonprod/` (F10). A different realm means 69 logs users in through that realm today. Re-point it in step 3.1 |
| `KEY` | The client it uses. It should be `aap-np` |
| `GROUPS_CLAIM` (if present) | The token claim AAP reads groups from. Must match the Keycloak mapper (step 1.3) |
| `create_objects` / `remove_users` | Must be `true` / `false` (step 3.1) |

If `KEY` is **not** `aap-np`, or the realm isn't the one you intend, decide now:
* **Re-point 69** to `aap-np` and the intended realm. This affects whoever uses 69 today; see the next commands.
* **Or keep 69 as it is** and use its existing client. Then apply step 1.3 to that client instead.

Who uses 69 and 35 today, and what maps are on 69:

```bash
# Users per authenticator (if the user objects carry an "authenticators" list)
jq -r '.[] | select(.authenticators != null) | .authenticators[]' ../backup/users-before.json | sort | uniq -c
# Fallback endpoint:
curl -sk $AUTH "$AAP/api/gateway/v1/authenticators/69/users/?page_size=200" | jq '.count, [.results[].username]'
curl -sk $AUTH "$AAP/api/gateway/v1/authenticators/35/users/?page_size=200" | jq '.count'
# Maps on 69
curl -sk $AUTH "$AAP/api/gateway/v1/authenticator_maps/?authenticator=69&page_size=200" \
  | jq -r '.results[] | [.order,.map_type,.name,.revoke] | @tsv'
```

* **69 has users already:** those accounts are already "new" accounts. Make sure their emails are in the YAML, or in `aap_superuser_members` / `kc_extra_allowed_groups`, **before** step 7. Otherwise the Allow map (revoke: true) will lock them out.
* **69 has no Allow map:** at the moment, any user who can log in to the Keycloak realm can create an AAP account (with no rights). Step 7 closes this. If that is not acceptable even for a few days, run step 7 first.

### 0.5 Precheck the usernames that will be created

The new AAP usernames will be the email local part. Check for clashes **before** anybody logs in:

```bash
python3 scripts/aap_user_migration.py precheck --before ../backup/users-before.json --tenants 'orgs/*.yml'
```

| Output | Meaning | Fix |
|---|---|---|
| `LOCALPART-COLLISION user1: [user1@ibm.com, user1@in.ibm.com]` | Two people would get the same Keycloak/AAP username. Keycloak would refuse the second one at first login | Rare. The realm admin creates the second user by hand with a distinct username (for example `user1.in`) and links their w3 identity (*Users → user → Identity provider links*) **before** they log in |
| `USERNAME-CLASH 'jsmith' already exists in AAP as id=… email=other@…` | A **different** person already holds that username through GitHub. AAP creates `jsmith-<hash>` for the new one | Acceptable (permissions still apply). Or tell the user. Never use auto-migrate (F5a) |
| `N existing AAP users have no e-mail` | Cannot be paired by `suggest` | Add the email in AAP, or pair them by hand in `pairs.csv` |

The sync playbook also fails fast on local-part collisions inside the YAML (`kc_username_mode: localpart`).

### 0.6 Freeze window (unchanged)

No manual changes to authenticators or maps in AAP during the migration. Changes to the tenant YAML can continue.

> **Removed from Phase 0:** the v1 "decision point" table (5.3) is resolved as case B. The "plan the permission copy" step is now implemented by `scripts/aap_user_migration.py` (step 8.3).

---

## 4. Phase 1 — Keycloak configuration

All settings below are in realm **`prism-nonprod`** (F10). In the admin console, check the realm selector at the top left before changing anything.

### 1.0 Realm name: done

`kc_realm: prism-nonprod` is already set in `automation/vars/np.yml` and in `playbooks/group_vars/all/main.yml`.

> **Re-check F6–F8 in this realm.** The IdP mapper `oidc-bluepages`, *Trust Email*, and the first-broker-login flow values you reported must belong to the w3 IdP **in `prism-nonprod`**. If you read them in another realm, repeat the check here: **Identity providers → (w3/bluepages IdP) → Mappers / Settings**.

### 1.1 IdP username mapper: no change

Validated in F6. Keep `oidc-bluepages` as it is: `${CLAIM.email | localpart}`, target `BROKER_USERNAME`, sync mode *inherit* (IdP *legacy*). Also keep *Trust Email* = ON.

> Do **not** switch the IdP sync mode to *Force* during the migration. That could change existing users' attributes while the old and new accounts are being paired.

### 1.2 First login flow: keep "PRISM first login flow" unchanged; don't pre-create users (F8)

**Decision: no change to Keycloak authentication flows for this migration.** v2's earlier plan was to duplicate the default flow and add auto-link, so that pre-created users could log in silently. That plan is **withdrawn**, for the reasons below.

#### Why a separate flow for the `aap-np` client is not possible

| Question | Answer |
|---|---|
| Where is the first login flow chosen? | On the **identity provider** (*Identity providers → w3 IdP → Settings → First login flow*). It is **not** a client setting. Keycloak's documentation says it can be set per identity provider |
| So can `aap-np` have its own? | **No.** Every client in the realm that signs users in through this w3 IdP uses the same first login flow. A client's *Advanced → Authentication flow overrides* offers, to my knowledge, only **Browser flow** and **Direct grant flow**. Check the dropdowns on `aap-np` to confirm; I could not verify this in the documentation |
| What would changing the IdP's flow do? | It changes first logins for the **other application** too. Editing "PRISM first login flow" or pointing the IdP at a copy has the same effect |
| How often does the flow run? | Only when a w3 identity has **no linked Keycloak account yet**. After a user's first login the link exists and the flow never runs for that user again. Anyone who already uses the other application through w3 **never goes through this flow for AAP** |

A second IdP entry for w3 in the same realm, with its own flow, would technically work, but it needs a second client registration at w3 (different broker redirect URI) and duplicated mappers. Treat it as a last resort, and only if pre-creating users turns out to be a hard requirement.

#### What the PRISM flow does today (as reported)

```
Review Profile                                             Required
PRISM … user creation or linking                           Required
 ├─ Create User If Unique                                  Alternative   new w3 user → account created, flow ends
 └─ PRISM … Handle Existing Account                        Alternative   w3 user matches an existing account (email or username)
     ├─ Confirm Link Existing Account                      Disabled
     ├─ Account verification options (email / re-auth+OTP) Disabled
     └─ Automatically Set Existing User                    Required  ← position needs confirming, see below
PRISM … conditional organization                           Conditional   (Organization member onboard)
```

**Needs confirming in the console.** In the pasted outline, *Automatically Set Existing User* is indented like the items **inside** the disabled *Account verification options*. In Keycloak's standard flow it would be a direct child of *Handle Existing Account*. Open **Authentication → Flows → PRISM first login flow** and check which it is. The two cases behave very differently:

| Reading | Behaviour for a w3 user who matches an existing account |
|---|---|
| **A.** Direct child of *Handle Existing Account*, **Required** | The user is **silently logged in as the existing account**, with no confirmation and no verification. Keycloak's documentation says this authenticator puts the existing user into the context "without verification" and calls it dangerous wherever usernames or emails can be chosen freely |
| **B.** Inside *Account verification options* (Disabled) | The whole sub-flow is switched off, so nothing identifies the existing user. I expect the login to **fail with an error**, but did not verify that. Test it |

#### What this means for AAP

1. **Do not pre-create users in Keycloak.** Under reading A a pre-created user is linked silently (works, but see the next point); under reading B it probably fails. Since the sync can't test this safely on a shared flow, set `kc_precreate_users: false` (already set in `automation/vars/np.yml`).
2. **Who logs in how**

   | User | First AAP login |
   |---|---|
   | Already has a Keycloak account (used w3 login for the other app) | Works at once. The flow does not run. The sync has already put them in their groups |
   | **New to the realm** | Keycloak creates the account (*Create User If Unique*), then the Allow map **refuses** the AAP login, because the new account is in no group yet. After the next sync (≤ 15 min, see 9.2) they are in their groups and the next login works |

   Ask new-to-realm users to sign in once at `https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod/account` and wait for the next sync. The sync's `kc_pending_users` output lists exactly the YAML users who have no Keycloak account yet. In the pilot, use users who already have one.
3. **Username clashes now matter in Keycloak, not only in AAP.** The IdP mapper turns `jsmith@in.ibm.com` into username `jsmith`. If a **different** person already holds Keycloak user `jsmith`, then under reading A the new person's first login is silently linked to that other person's account, and AAP then gives them that person's groups. This risk exists for the other application today. For AAP the sync reports these cases in `kc_pending_users` as `… (Keycloak username X belongs to Y)`. **Resolve every such entry with the realm admin before that user logs in.**
4. If you later want pre-creation anyway, first settle reading A or B with the realm admin and test with one user, then make an explicit decision about the other application.

#### Impact on the other application

| Area | Impact |
|---|---|
| First login flow, IdP settings, mappers | **None.** No change is made |
| Tokens | None, if the `groups` and audience mappers are on **`aap-np-dedicated`** (1.3) and **not** on a realm-wide default client scope |
| *Organization member onboard* step in the flow | Also runs for AAP users' first logins. The IdP does not appear to be tied to a Keycloak Organization, so it should do nothing. Test with one new-to-realm user |
| Realm users | The sync never creates, edits or deletes users in this setup. It only reads them and changes **group membership** |
| Groups | The sync creates the groups it needs and, with `kc_prune_group_members: true`, **removes members who are not in the YAML**. **Before the first sync:** list the groups in the realm (`GET /admin/realms/prism-nonprod/groups?search=-group`). If a group such as `prism-pes-abc-delivery-group` already exists and something else maintains it (for example the other application), set `kc_prune_group_members: false` for that group or agree with the owner first |
| Service account | `aap-np-automation` holds realm-wide `manage-users`, so it can change any user's groups. Restrict who can read its secret |

Review Profile is **Required** in the flow. New-to-realm users may therefore see a profile confirmation page at their first login, as for the other application. That is expected.

### 1.3 Client used by authenticator 69 (normally `aap-np`): verify

**Clients → aap-np**

| Setting | Required value | Why |
|---|---|---|
| Client authentication | ON | AAP sends the secret |
| Standard flow | ON | Authorization-code login |
| Direct access grants / Implicit / Service accounts | OFF | Not needed |
| **Valid redirect URIs** | `https://aap-prod-latest.prism-np.dst.ibm.com/api/gateway/social/complete/c221bc1f-b5bf-402f-8727-f938ae656956/` | Callback of authenticator 69. Copy it exactly from 69's details page, trailing `/` included |
| Web origins | `https://aap-prod-latest.prism-np.dst.ibm.com` | |

**Client scopes → `aap-np-dedicated` → mappers.** The *Keycloak* authenticator type reads user data and groups from the **access token**, and validates its signature and audience. The access token must therefore contain the following.

| Mapper | Type | Settings | Check |
|---|---|---|---|
| `groups` | Group Membership | Token Claim Name = **the value of `GROUPS_CLAIM` in authenticator 69** (step 0.4). `groups` is recommended; set the AAP field to `groups` in step 3.1. **Full group path OFF.** Add to **access token ON**, ID token ON, userinfo ON | Add it if missing |
| `aap-np-audience` | Audience | Included Client Audience = `aap-np`. Add to **access token ON** | Keycloak access tokens have `aud: account` by default. AAP validates `aud` against the client ID (`KEY`). If 69 already logs people in successfully, this mapper (or an equivalent) is probably already there |

Also check that the `profile` and `email` client scopes are **Default** scopes. They provide `preferred_username` and `email`.

**Check the token:** go to **Clients → aap-np → Client scopes → Evaluate**, pick a test user who is in a test group, and look at **Generated access token**.

```json
{
  "iss": "https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod",
  "aud": ["aap-np", "..."],
  "preferred_username": "rakesh.upadhyay1",
  "email": "rakesh.upadhyay1@ibm.com",
  "groups": ["prism-pes-abc-delivery-group"]
}
```

### 1.4 Service-account client for the automation (unchanged from v1 6.6)

Create a client `aap-np-automation` with Client authentication ON and only **Service accounts roles** ON. Assign it the `realm-management` roles `manage-users`, `view-users`, `query-users` and `query-groups`. Test it:

```bash
KC=https://keycloak.prism-np.dst.ibm.com; REALM=prism-nonprod
TOKEN=$(curl -s -X POST "$KC/realms/$REALM/protocol/openid-connect/token" \
  -d grant_type=client_credentials -d client_id=aap-np-automation -d client_secret=<secret> | jq -r .access_token)
curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/$REALM/groups?max=5" | jq .
```

### 1.5 Superuser group

Create the group `aap-np-platform-admins`, or let the sync create it. Membership is managed from `aap_superuser_members` (step 4.1).

---

## 5. Phase 2 — Network / TLS re-check

**Status: ✔ done.**
* TLS and connectivity from the gateway pod are validated (F9).
* The cluster already trusts *IBM INTERNAL INTERMEDIATE CA*, so **no custom CA bundle is needed**.
* Realm `prism-nonprod` discovery returns **HTTP 200** (F10).

One more check: the **issuer** in the discovery document must exactly match the URLs in authenticator 69. The token's `iss` is compared against them. Your test used `-o /dev/null`, so the body wasn't shown. Run:

```bash
oc exec deploy/aap-np-gateway -- \
  curl -s https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod/.well-known/openid-configuration \
  | jq '{issuer, authorization_endpoint, token_endpoint}'
# expected:
#   issuer                 = https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod
#   authorization_endpoint = …/realms/prism-nonprod/protocol/openid-connect/auth   (= Keycloak Provider URL in 69)
#   token_endpoint         = …/realms/prism-nonprod/protocol/openid-connect/token  (= Keycloak Access Token URL in 69)
```

If the `issuer` shows a different hostname, for example an internal service name, set `KC_HOSTNAME` on Keycloak to `keycloak.prism-np.dst.ibm.com`.

The public key for step 3.1 can also be read from the pod:

```bash
oc exec deploy/aap-np-gateway -- curl -s https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod | jq -r .public_key
```

Also keep these from v1:
* **Clock skew.** Token validation fails if the clocks differ by more than about 60 s. Check NTP on the cluster and on Keycloak.
* **Issuer.** It must be the public hostname shown above. If Keycloak shows an internal hostname, fix `KC_HOSTNAME`.

---

## 6. Phase 3 — Prepare the existing "Keycloak SSO" authenticator (id 69)

> **Removed from v1:** creating a new Generic OIDC authenticator, registering a new callback URL, and `01_aap_keycloak_authenticator.yml`. Authenticator 69 already exists and is enabled.

### 3.1 Check and correct its settings

**Access Management → Authentication Methods → Keycloak SSO → Edit**

| Field | Value |
|---|---|
| Keycloak Access Token URL | `https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod/protocol/openid-connect/token` |
| Keycloak Provider URL | `https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod/protocol/openid-connect/auth` |
| Keycloak OIDC Key | `aap-np` |
| Keycloak OIDC Secret | secret from *Clients → aap-np → Credentials* (re-enter it only if the client changed) |
| Keycloak Public Key | the **RS256** public key from *Realm settings → Keys*, **without** the `-----BEGIN/END-----` lines. It must be the realm's *current active* RS256 key |
| Groups Claim (if the field exists) | `groups`, the same as the mapper in 1.3 |
| Enabled | On (already) |
| Create objects | **On** |
| Remove users | **Off** (otherwise every login wipes directly granted and copied roles) |
| Auto migrate users to | **empty** (F5a) |

> **Public key rotation.** This authenticator type stores the realm key itself. If Keycloak rotates or adds a new active RS256 key, Keycloak logins fail with a signature error until you paste the new key here. Add this to the Keycloak key-rotation runbook.

### 3.2 GitHub authenticator (id 2): no change

Leave it enabled. Leave *Auto migrate users to* empty. Change nothing on it until cutover.

---

## 7. Phase 4 — Run the tenant sync

### 4.1 Set the variables for this environment

`automation/vars/np.yml` (new in v2):

```yaml
aap_hostname: https://aap-prod-latest.prism-np.dst.ibm.com
kc_url: https://keycloak.prism-np.dst.ibm.com
kc_realm: prism-nonprod                 # confirmed (F10)
aap_kc_authenticator_name: Keycloak SSO # reuse id 69
kc_username_mode: localpart             # matches the oidc-bluepages mapper
aap_superuser_members:                  # emails of today's superusers (step 0.3), except 'admin'
  - someone@ibm.com
```

If 69 already has users who are **not** in any tenant YAML (step 0.4), add them to `aap_superuser_members` or to a group listed in `kc_extra_allowed_groups`. Otherwise the Allow map shuts them out.

### 4.2 Preview, then run

```bash
cd automation
ansible-playbook playbooks/02_sync_tenants.yml -e @vars/np.yml -e @vault.yml --ask-vault-pass --tags model   # preview only
ansible-playbook playbooks/02_sync_tenants.yml -e @vars/np.yml -e @vault.yml --ask-vault-pass
```

What it does, all on authenticator **69**:

**AAP:** creates the orgs and teams, if missing.

**Keycloak:**
* Creates the groups.
* Does **not** pre-create users (`kc_precreate_users: false`, see 1.2). YAML users who have no Keycloak account yet are reported as pending. They are added to their groups at the first sync after their first login. If their local part is already taken by a **different** email in the realm, that is reported too.
* Reconciles membership to match the YAML.
* Syncs `aap-np-platform-admins` from `aap_superuser_members`.

**AAP maps:**
* `kc-sync | <org> | org-member`
* `kc-sync | <org> | org-admin`
* `kc-sync | <org> | <team> | member`
* `kc-sync | superuser`
* `kc-sync | ALLOW | members of managed groups` (revoke: true)
* Deletes stale `kc-sync` maps. Maps without the prefix, such as existing hand-made maps on 69, are never touched.

The run ends with `kc_pending_users`. It must be empty, or every entry explained, before the pilot.

> **Always run the sync over all tenant files.** The Allow map is the union of every group.

---

## 8. Phase 5 — Pilot, including the permission copy

### 8.1 Choose pilot users

Pick 3–5 users:
* an org admin,
* a team member,
* a user in two tenants,
* a superuser,
* a user who has **directly granted** permissions. Find one with `role_user_assignments` or the script's dry-run.

### 8.2 Pilot users log in with "Keycloak SSO"

First, each pilot user who has never used w3 login for PRISM signs in once at `https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod/account`. Then run the sync, so their accounts are in their groups (1.2).

Ask them to use a private browser window so the GitHub session doesn't interfere. Then check:

| Check | Expected (v2) |
|---|---|
| Login succeeds | ✔ |
| Username | **new** account `rakesh.upadhyay1`, not the GitHub login. This is expected |
| Orgs / teams | as in the YAML (from the maps) |
| Team-granted permissions (JT execute, …) | present (inherited through teams) |
| Directly granted permissions | **missing until step 8.3**, then present |
| A w3 user not in any group | refused |
| Old GitHub account | unchanged, still works through the GitHub button |

### 8.3 Pair old and new accounts, then copy the direct permissions

```bash
cd automation
export AAP_URL=https://aap-prod-latest.prism-np.dst.ibm.com AAP_USER=admin AAP_PASSWORD='<password>'

# 1) pair by e-mail  (old = authenticator 2, new = authenticator 69)
python3 scripts/aap_user_migration.py --insecure suggest \
        --before ../backup/users-before.json --old-authenticator 2 --new-authenticator 69 --out pairs.csv
#    → review pairs.csv; fix AMBIGUOUS lines by hand

# 2) dry-run: shows every role that would be copied, and every role skipped because maps grant it
python3 scripts/aap_user_migration.py --insecure copy --pairs pairs.csv

# 3) apply
python3 scripts/aap_user_migration.py --insecure copy --pairs pairs.csv --apply
```

How the copy behaves:
* It copies role assignments from the gateway (`/api/gateway/v1/role_user_assignments/`) and the controller (`/api/controller/v2/role_user_assignments/`), from old user to new user. Each service's user ID is looked up by username.
* By default it **skips** *Organization Admin*, *Organization Member* and *Team Member*, because the group maps provide them.
  * The dry-run lists every skipped assignment. If an old account was manually added to a team or org that is **not** in the YAML, add it to the YAML (preferred), or re-run with fewer `--skip-role` entries.
  * Add `--skip-role "<name>"` for other map-provided role names that appear in your build's dry-run.
* It is idempotent: assignments that already exist are reported as `exists` and not created again. It can run as often as needed.

The pilot passes when the new accounts show every permission the old ones had: compare in *Users → <user> → Roles*. Then go on to Phase 6.

---

## 9. Phase 6 — Coexistence window (1–2 weeks)

### 9.1 Announcement (template)

> From <date>, please sign in to AAP with **Keycloak SSO** (w3 login). You'll get a new user name, for example `firstname.lastname`. Your organizations, teams and permissions are set up automatically. If you have never used w3 login for PRISM before, sign in once at https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod/account first, then wait 15 minutes. If your first AAP login says access denied, wait 15 minutes and try again. **Personal API tokens are not carried over:** create new ones under *User → Tokens* and update any scripts or CI that use them. The GitHub login stops working on <date + 14>.

### 9.2 Settings during the window

| Setting | Value |
|---|---|
| `aap_map_revoke` | `false` (maps only add) |
| `kc_prune_group_members` | `true` |
| Old tenant automation (direct assignment on GitHub accounts) | can keep running; it only affects old accounts |
| New sync | every **15 minutes** (a new user waits at most that long after their first login), plus on every YAML change |

### 9.3 Daily routine

Schedule it as an AAP job, or run it by hand:

```bash
python3 scripts/aap_user_migration.py --insecure suggest --before ../backup/users-before.json --out pairs.csv
python3 scripts/aap_user_migration.py --insecure copy    --pairs pairs.csv --apply
```

`suggest` also prints how many old accounts **have not yet** logged in through Keycloak. That is your migration progress. Send reminders to those users in the middle and at the end of the window.

> **Permissions granted after the snapshot.** An org admin may grant a permission to a user's *old* account during the window. The daily copy carries it over. Grants made on the *new* account are never touched. Still, ask org admins to grant to **teams**, not to individuals.

### 9.4 Things the copy can't carry over

| Item | Action |
|---|---|
| Personal API tokens (OAuth2 tokens) | The user creates new ones. Find CI or scripts that use tokens of old accounts and plan their replacement **before** cutover |
| User-level settings or preferences | The user sets them again |
| "Created by" on existing objects and jobs | Stays pointing at the old account (history only, no effect) |

---

## 10. Phase 7 — Cutover and GitHub decommission

**Day 0:**

1. Run the sync, then `suggest` and `copy --apply`, one final time. `kc_pending_users` must be empty.
2. **Disable** authenticator 2 `github-enterprise-team` (*Edit → Enabled: Off*). Don't delete it yet.
3. In `vars/np.yml`, set `aap_map_revoke: true` and re-run the sync. Removing someone from the YAML now removes their roles at their next login.
4. Retire the direct-assignment part of the old tenant automation.
5. Optional: so users skip the Keycloak account chooser, set the w3 IdP as the default. In the realm's *browser* flow, configure *Identity Provider Redirector* with the w3 IdP alias.

**Day 0 + 7 to 14:**

6. **Old GitHub accounts.** They can no longer log in, but they still **hold roles**. Delete them, using the IDs in `users-before.json` minus any you're keeping:
   * First check that every one either has a paired new account (`pairs.csv`) or belongs to a leaver.
   * Check that nothing in CI still uses their tokens.
   * Deleting a user removes their role assignments and tokens. Job history is kept.
7. Authenticator 35 `IBM-W3-OIDC`: if it has 0 linked users (step 0.4), delete it to avoid confusion.
8. Delete authenticator 2 and its maps.
9. On github.ibm.com, delete or revoke the OAuth App with client ID `ea3060887fcd501d828a`. Remove its secret from Vault and CI.
10. Update runbooks:
    * the Keycloak public-key rotation note (3.1),
    * onboarding means "add to tenant YAML", with no GitHub team 96798 any more.

---

## 11. Rollback

The old accounts are never modified, so rollback is simple.

| When | Rollback |
|---|---|
| Before or during the pilot / window | Users use the **GitHub** button. Their old accounts are intact. Optionally delete the `kc-sync \| …` maps on 69. Leftover Keycloak groups are harmless. Nothing was changed in the first login flow |
| After cutover, before 10.6 | Re-enable authenticator 2. Old accounts still have their roles |
| After 10.6 / 10.9 | Old accounts and the OAuth app are gone, so there is no rollback to GitHub. That is why those steps wait 1–2 weeks |
| Catastrophic | Restore `pre-keycloak-migration` with an `AnsibleAutomationPlatformRestore` CR (`deployment_name: aap-np`) |

Break-glass access: the local `admin` user on authenticator 1.

```bash
oc get secret aap-np-admin-password -o jsonpath='{.data.password}' | base64 -d
```

---

## 12. Automation documentation (v2 changes)

The v1 automation documentation (v1 section 15) still applies. These are the changes.

### 12.1 Files

```
automation/
├── vars/np.yml                       # NEW: values for this environment (realm, URL, authenticator, username mode)
├── scripts/aap_user_migration.py     # NEW: snapshot / precheck / suggest / copy (Python standard library only)
├── orgs/*.yml                        # tenant YAML: format UNCHANGED
└── playbooks/
    ├── 01_aap_keycloak_authenticator.yml   # NOT USED in v2 (authenticator 69 is reused)
    ├── 02_sync_tenants.yml                 # now also manages the superuser map + superuser group
    ├── filter_plugins/tenant_model.py      # + aap_localpart_collisions
    └── tasks/kc_add_member.yml             # + kc_username_mode
```

### 12.2 New and changed variables

| Variable | v2 value | Meaning |
|---|---|---|
| `kc_username_mode` | `localpart` | `email`: Keycloak username = full email (v1). `localpart`: username = part before `@`, which matches the `oidc-bluepages` mapper. In `localpart` mode users are looked up **by email only**, collisions in the YAML fail the run, and pre-creation is refused if the local part belongs to a different email |
| `aap_kc_authenticator_name` | `Keycloak SSO` | The existing authenticator 69 |
| `kc_precreate_users` | `false` | Don't pre-create Keycloak users. Keycloak creates them at first w3 login; the sync adds them to groups afterwards (1.2) |
| `aap_superuser_members` | list of emails | Members of `aap_superuser_group`. If it is empty, the group's membership isn't managed |
| `kc_realm` | `prism-nonprod` | Confirmed realm (F10). A guard still stops the sync if a `CHANGE_ME…` value is ever set |
| `aap_hostname`, `kc_url` | `prism-np` URLs | See the table at the top |

### 12.3 `scripts/aap_user_migration.py`

Environment: `AAP_URL`, `AAP_USER`, `AAP_PASSWORD`, optional `AAP_CAFILE`. Add `--insecure` to behave like `curl -k`.

| Command | When | What it does |
|---|---|---|
| `snapshot --out users-before.json` | Phase 0, before the pilot | Saves all users (all pages): id, username, email, is_superuser, authenticators |
| `precheck --before … --tenants 'orgs/*.yml'` | Phase 0 | Finds local-part collisions and usernames already taken by another person |
| `suggest --before … --old-authenticator 2 --new-authenticator 69 --out pairs.csv` | Pilot, then daily | Pairs old and new accounts by email. Reports ambiguous pairs and not-yet-migrated users |
| `copy --pairs pairs.csv [--apply] [--services gateway,controller] [--skip-role NAME]` | Pilot, then daily | Dry-run by default. Copies direct role assignments. Idempotent |

To run it in AAP during the window:
1. Create a job template that calls it through a small playbook (`ansible.builtin.command`).
2. Pass the *AAP Gateway API* credential from v1 15.7 as environment variables.
3. Keep `users-before.json` in the project repository, or as an artifact.

### 12.4 Running the sync from AAP

The same as v1 15.7. Additionally, pass `vars/np.yml` as extra vars: copy its content into the job template's *Extra variables*, or add `-e @vars/np.yml` through a survey-free wrapper playbook.

---

## 13. What changed from v1 and why

| v1 step | v2 | Reason |
|---|---|---|
| 5.2 single-page user export | **Replaced** by paging export (0.3) | 253 users > page size 200 (F4) |
| 5.3 decision table | **Removed.** Resolved as case B | F5 |
| 5.4 "plan" the permission copy | **Implemented** by `aap_user_migration.py` (8.3) | F5 |
| 6.1 change the username mapper to the lower-case full email | **Removed.** Keep `oidc-bluepages` (localpart) | Only needed for auto-migrate, which isn't used (F6) |
| 6.2 auto-link flow + pre-created users | **Removed.** Keep the shared *PRISM first login flow* unchanged; `kc_precreate_users: false` | The flow is shared with another application and can't be overridden per client; pre-creation with local-part usernames is a takeover risk (F8, 1.2) |
| 6.3/6.4 client + groups mapper | **Changed** to verify the client used by 69. Access token + **audience mapper** added | The *Keycloak* authenticator type reads and validates the access token |
| 7.2 custom CA bundle (`bundle_cacert_secret`) | **Removed** | The gateway already trusts IBM INTERNAL INTERMEDIATE CA (F9) |
| 7.1 connectivity test | **Kept** with the correct realm, and the body printed | The earlier 404 = realm does not exist (F9) |
| 8 create a new Generic OIDC authenticator (disabled), register a new callback, `01_…yml` | **Removed.** Reuse authenticator 69 | F2 |
| 10 "Auto migrate users to" | **Removed**, and explicitly **must not** be set | No benefit, risk of account takeover (F5a) |
| 11 pilot "same user ID" check | **Changed** to "new account + copy" checks | F5 |
| 12 "don't go back to the GitHub button" warning | **Removed** | Accounts are separate. Going back is harmless |
| 13 cutover | **Added** deletion of old accounts, personal-token replacement, authenticator 35 clean-up | Two accounts per person (F5), F3 |
| 14 rollback via reverse auto-migrate | **Replaced** by "use the GitHub button" | Old accounts are never modified |
| 17 assumptions about auto-migrate | **Removed** | Not applicable |

---

## 14. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `{"error":"Realm does not exist"}` / HTTP 404 on `.well-known` | Wrong realm name (for example `aap-nonpord`) | The realm is `prism-nonprod` (F10). Fix the URL, or 69's `ACCESS_TOKEN_URL` / `AUTHORIZATION_URL` |
| Keycloak: *Invalid parameter: redirect_uri* | 69's callback isn't in the client's *Valid redirect URIs* | 1.3, exact URL with slug `c221bc1f-…` and trailing `/` |
| AAP: *Signature verification failed* / *invalid token* after w3 login | Public key in 69 is outdated (key rotation) or from another realm | 3.1: paste the current RS256 key |
| AAP: *Invalid audience* | No audience mapper; `aud` doesn't contain `aap-np` | 1.3 audience mapper |
| Login works but no orgs or teams | No `groups` claim **in the access token**, claim name ≠ `GROUPS_CLAIM`, or full path ON | 1.3, then *Evaluate → access token* |
| Login refused ("not allowed") | User isn't in any managed group | Add them to the YAML, run the sync, check `kc_pending_users` |
| First AAP login refused for someone new to `prism-nonprod` | Their Keycloak account was only created at this login, so it isn't in any group yet | Expected (1.2). Wait for the next sync (≤ 15 min) and log in again |
| Keycloak shows "Account already exists" / link pages, or the user lands in someone else's account | Their local part matches an existing account in the realm (existing-account path of PRISM first login flow) | Check `kc_pending_users` for `belongs to`. Resolve the clash with the realm admin (0.5, 1.2) |
| New AAP user is called `jsmith-1a2b3c4d` | The local part equals an existing GitHub username of another person | Expected (0.5). Permissions still apply |
| Keycloak: *User exists with same username* at first login, for a second person | Local-part collision (`user1@ibm.com` / `user1@in.ibm.com`) | 0.5: the realm admin creates the second user with a distinct username and links their w3 identity |
| Direct permissions missing on the new account | Copy not run yet, or the pair is missing from `pairs.csv` | 8.3. Check the `suggest` output for AMBIGUOUS or missing emails |
| Sync `403` on `/admin/realms/...` | Service-account roles missing | 1.4 |
| Removed user keeps their role after cutover | `aap_map_revoke` still `false`, or the user hasn't logged in again | 10.3. The role is removed at the next login |

Logs:

```bash
oc logs deploy/aap-np-gateway --all-containers --since=30m | grep -iE 'keycloak|social|authenticat|map|jwt|audience'
```

On the Keycloak side: *Realm settings → Events → User events* (enable *LOGIN_ERROR* and *IDENTITY_PROVIDER_LOGIN_ERROR*).
