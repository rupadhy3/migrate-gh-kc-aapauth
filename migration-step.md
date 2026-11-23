# AAP: GitHub Enterprise Team → Keycloak (w3) migration steps

Everything is set up from scratch, except the Keycloak realm **`prism-nonprod`**.

| | |
|---|---|
| AAP | `https://aap-prod-latest.prism-np.dst.ibm.com` (OpenShift, CR `aap-np`) |
| Keycloak | `https://keycloak.prism-np.dst.ibm.com`, realm `prism-nonprod` |
| Created in this guide | Keycloak: w3 identity provider, client `aap-np`, service account `aap-np-automation`, groups. AAP: authenticator `Keycloak-w3`, maps |
| Scripts | this folder, all bash; `config.env` holds every setting |

Two AAP accounts per person: GitHub usernames (e.g. `Rakesh-Upadhyay1`) won't match the Keycloak usernames (`rakesh.upadhyay1`). So each person gets a **new** account at their first Keycloak login. Group maps restore their orgs and teams, and `30_migrate_users.sh` copies the roles that were granted to them directly. Old accounts are not changed until cutover.

---

## Prerequisites

1. Workstation with `bash`, `curl`, `jq`, `oc`, and either `yq` (mikefarah v4) or `python3-pyyaml`.
2. AAP: a **local** superuser (e.g. `admin`). Its password:
   ```bash
   oc get secret aap-np-admin-password -o jsonpath='{.data.password}' | base64 -d
   ```
3. Keycloak: an admin account for realm `prism-nonprod`.
4. Copy the tenant YAML files (format unchanged) into `migration/orgs/`.
5. Create the config:
   ```bash
   cd migration
   cp config.env.example config.env && chmod 600 config.env
   vi config.env          # set AAP_PASSWORD now; the secrets are filled in at steps 4 and 6
   ```

---

## Step 1: Back up AAP

1. Operator backup:
   ```bash
   cat <<'EOF' | oc apply -f -
   apiVersion: aap.ansible.com/v1alpha1
   kind: AnsibleAutomationPlatformBackup
   metadata:
     name: pre-keycloak-migration
   spec:
     deployment_name: aap-np
   EOF
   oc get ansibleautomationplatformbackup pre-keycloak-migration -w      # wait for Successful
   ```
2. API export of authenticators, maps, users, orgs and teams:
   ```bash
   ./01_backup.sh
   ```

## Step 2: Register Keycloak at IBM w3 ID

1. Request an OIDC client from the IBM w3 ID / SSO provisioning team, with:
   - **Redirect URI:** `https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod/broker/w3/endpoint`
   - **Scopes:** `openid email profile`
2. Note the **client ID**, **client secret** and the w3 **discovery URL**. Get the URL from the w3 team, ending in `/.well-known/openid-configuration`.

> Skip this step if the realm already has a working w3 identity provider. Use its alias instead of `w3` in step 3.

## Step 3: Add the w3 identity provider in Keycloak

Keycloak admin console, realm **prism-nonprod**:

1. **Identity providers → Add provider → OpenID Connect v1.0**
   - Alias: `w3`. Display name: `IBM w3`
   - Discovery endpoint: *the w3 discovery URL*. This fills in the endpoints.
   - Client authentication: *Client secret sent as post*
   - Client ID / Client secret: *from step 2*
   - **Add**
2. Same page, **Advanced settings**:
   - Scopes: `openid email profile`
   - Trust email: **On**
   - Sync mode: **Import**
   - First login flow: keep the realm's current flow. If unset, use `first broker login`.
   - **Save**
3. **Mappers → Add mapper**
   - Name `username-localpart`, Sync mode override `Import`
   - Mapper type **Username Template Importer**
   - Template `${CLAIM.email | localpart}`, Target `LOCAL`
   - **Save**
4. Add three more mappers, type **Attribute Importer**:
   - Claim `email` → User attribute `email`
   - Claim `given_name` → `firstName`
   - Claim `family_name` → `lastName`

> **Shared realm:** if other applications already use an IdP in this realm, don't change its first login flow or mappers. The first login flow belongs to the IdP, not to a client, so changing it affects every application.

## Step 4: Create the login client `aap-np`

1. **Clients → Create client**
   - Client type `OpenID Connect`, Client ID `aap-np` → **Next**
   - Client authentication **On**. Standard flow **On**. Everything else **Off** → **Next**
   - Root URL / Home URL: `https://aap-prod-latest.prism-np.dst.ibm.com`
   - Valid redirect URIs: leave empty for now (step 7)
   - Web origins: `https://aap-prod-latest.prism-np.dst.ibm.com` → **Save**
2. **Credentials** tab: copy the **Client secret** into `config.env` as `KC_AAP_CLIENT_SECRET`.
3. **Client scopes → aap-np-dedicated → Configure a new mapper → Group Membership**
   - Name `groups`, Token claim name `groups`
   - Full group path **Off**
   - Add to ID token, access token and userinfo: **On** → **Save**
4. **aap-np-dedicated → Configure a new mapper → Audience**
   - Name `aap-np-audience`, Included client audience `aap-np`
   - Add to access token **On** → **Save**
5. **Client scopes** tab of `aap-np`: `profile` and `email` must be listed as **Default**.

## Step 5: Optional: send users straight to w3

So users don't see the Keycloak login page, go to **Clients → aap-np → Advanced → Authentication flow overrides**:

1. **Authentication → Flows → browser → ⋮ Duplicate**, name `aap-np browser`.
2. In `aap-np browser`, open the **gear** of *Identity Provider Redirector* and set **Default identity provider** to `w3`. Save.
3. Set **Browser flow** override = `aap-np browser`. Save.

This affects only `aap-np`.

## Step 6: Create the service account `aap-np-automation`

The scripts use it to manage groups.

1. **Clients → Create client**
   - Client ID `aap-np-automation` → **Next**
   - Client authentication **On**. **Service accounts roles On**. Standard flow **Off**. Direct access grants **Off** → **Save**
2. **Service accounts roles** tab → **Assign role → Filter by clients**, then assign:
   `realm-management` `manage-users`, `view-users`, `query-users`, `query-groups`
3. **Credentials** tab: copy the secret into `config.env` as `KC_AUTOMATION_CLIENT_SECRET`.
4. Check everything:
   ```bash
   ./00_preflight.sh          # must end with "Preflight passed."
   ```

## Step 7: Create the AAP authenticator (disabled)

1. Create it:
   ```bash
   ./10_authenticator.sh create
   ```
   This creates `Keycloak-w3`, type *Keycloak*, **disabled**:
   - token and authorization URLs for `prism-nonprod`
   - client `aap-np`
   - realm public key
   - groups claim `groups`
   - *Create objects* on, *Remove users* off
2. Copy the printed **Callback URL**. Paste it into **Keycloak → Clients → aap-np → Settings → Valid redirect URIs**, exactly, including the trailing `/`. Save.

<details><summary>Same thing in the AAP UI</summary>

**Access Management → Authentication Methods → Create authentication**:

| Field | Value |
|---|---|
| Name | `Keycloak-w3` |
| Type | **Keycloak** |
| Keycloak Access Token URL | `https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod/protocol/openid-connect/token` |
| Keycloak Provider URL | `https://keycloak.prism-np.dst.ibm.com/realms/prism-nonprod/protocol/openid-connect/auth` |
| Keycloak OIDC Key | `aap-np` |
| Keycloak OIDC Secret | `aap-np` client secret |
| Keycloak Public Key | *Realm settings → Keys → RS256 → Public key*, no BEGIN/END lines |
| Groups Claim | `groups` |
| Enabled | Off |
| Create objects | On |
| Remove users | Off |

</details>

## Step 8: Check usernames

```bash
./02_precheck.sh
```

Fix what it reports before step 10:

| Section | Fix |
|---|---|
| 1. Two YAML e-mails, one username | The Keycloak admin creates the second user by hand with another username, then links it: *Users → user → Identity provider links → w3* |
| 2. Username taken by another e-mail in AAP | Nothing to do. AAP gives the new user a suffixed name. Tell the user |
| 3. One e-mail on several AAP accounts | Their direct roles are all copied to the one new account. Confirm that's intended |
| 4. Accounts without e-mail | Add the e-mail in AAP (*Users → Edit*), or handle by hand |

## Step 9: Sync tenants

```bash
./20_sync.sh --dry-run     # review
./20_sync.sh
```

The sync does the following, from all tenant YAML files:

- **AAP:** creates organizations and teams.
- **Keycloak:** creates groups and sets their members:
  - `<org>-admins-group`
  - `<team minus -team>-group`
  - `aap-np-platform-admins`, from `SUPERUSERS`
- **Maps on `Keycloak-w3`:**
  - **Allow:** only members of these groups can log in
  - superuser
  - org admin
  - org member
  - team member

The sync never creates Keycloak users. People who have never logged in with w3 are listed in `state/pending.txt` and are added after their first login.

Run it **every 15 minutes**, and after each YAML change. Always over all files. Cron example:

```
*/15 * * * * cd /path/to/migration && ./20_sync.sh >> state/sync.log 2>&1
```

Any `CLASH` in `pending.txt`: the Keycloak admin resolves it before that person logs in.

## Step 10: Pilot

1. Enable the button:
   ```bash
   ./10_authenticator.sh enable
   ```
   GitHub stays enabled.
2. 3–5 pilot users (an org admin, a team member, a superuser) sign in with **Keycloak-w3** in a private window.
   - A first-time w3 user is refused once (no group yet). Run `./20_sync.sh` and they log in again.
3. Copy their direct roles:
   ```bash
   ./30_migrate_users.sh pair
   ./30_migrate_users.sh copy            # dry run: review
   ./30_migrate_users.sh copy --apply
   ```
4. Check each pilot's new account in AAP (*Users → user → Roles*): same orgs, teams and job templates as the old account.
5. Check that a w3 user who isn't in any YAML is refused.

**Go** if all of this is true.

## Step 11: Coexistence (1–2 weeks)

1. Announce:
   > Sign in to AAP with **Keycloak-w3**. You'll get a new username (`firstname.lastname`). Orgs, teams and permissions follow automatically. If your first login is refused, wait 15 minutes and try again. Re-create personal API tokens. GitHub login ends on *date*.
2. Daily:
   ```bash
   ./30_migrate_users.sh pair && ./30_migrate_users.sh copy --apply
   ./30_migrate_users.sh status          # progress
   ```
3. Find CI jobs and scripts that use API tokens of old accounts, and replace those tokens.

## Step 12: Cutover

1. Last run:
   ```bash
   ./20_sync.sh && ./30_migrate_users.sh pair && ./30_migrate_users.sh copy --apply
   ```
2. Disable GitHub login: **Access Management → Authentication Methods → github-enterprise-team → Edit → Enabled Off**. Don't delete it yet.
3. Set `REVOKE=true` in `config.env`, then run `./20_sync.sh`. People removed from the YAML now lose access at their next login.
4. Stop the old tenant automation.

## Step 13: Clean up (7–14 days after cutover)

1. Delete old accounts:
   ```bash
   ./30_migrate_users.sh delete-old             # review
   ./30_migrate_users.sh delete-old --apply
   ```
   This deletes only paired, non-local accounts.
2. Delete the `github-enterprise-team` authenticator in AAP.
3. On github.ibm.com, delete the OAuth app (client ID `ea3060887fcd501d828a`). Remove its secret from vaults and CI.

---

## Rollback

| When | Action |
|---|---|
| Before step 13 | `./10_authenticator.sh disable`, then re-enable `github-enterprise-team`. Old accounts are untouched |
| Anything broken | Restore the step 1 backup with an `AnsibleAutomationPlatformRestore` CR (`deployment_name: aap-np`) |

Local `admin` always works.

## Keycloak key rotation

After Keycloak gets a new RS256 key, run `./10_authenticator.sh refresh-key`. Until you do, Keycloak logins fail.

## Scripts

| Script | Does |
|---|---|
| `00_preflight.sh` | checks tools, AAP login, realm, service account, YAML |
| `01_backup.sh` | exports AAP objects to `state/backup-*` |
| `02_precheck.sh` | lists username and e-mail problems |
| `10_authenticator.sh` | `create` · `enable` · `disable` · `show` · `refresh-key` |
| `20_sync.sh` | YAML → AAP orgs, teams, maps + Keycloak groups (`--dry-run`) |
| `30_migrate_users.sh` | `pair` · `copy [--apply]` · `status` · `delete-old [--apply]` |
