# Deploying Kutt to Azure (go.odbm.org)

Kutt runs as a plain Node.js app on Azure App Service (no Docker), backed by
Azure Database for PostgreSQL. Short links are `https://go.odbm.org/{slug}`;
each link's destination is set when it's created (e.g. `https://odbm.org/en/{slug}`).
Admins sign in with Microsoft Entra ID.

Subscription **ODB-Production** (`146df7c5-9362-4d37-b4f9-4c35de9f7f20`),
resource group **rg-shortlinks-prod**.

| Component | Resource | Region | Approx. cost/month |
|---|---|---|---|
| App | `asp-kutt-prod` (Linux B1) + `app-kutt-prod` (Node 22 LTS) | East US 2 | ~$13 |
| Database | `psql-kutt-prod` — PostgreSQL 16 Flexible Server, Burstable B1ms | **Central US** | ~$15–20 |
| Secrets | `kv-kutt-prod` | East US 2 | ~$0–1 |
| Monitoring | Application Insights | East US 2 | ~$0–5 |
| HTTPS | App Service Managed Certificate | — | $0 |

**Why the database is in Central US:** PostgreSQL Burstable (B-series) returns
`SkuNotAvailable` in East US 2 for this subscription — a Microsoft capacity/region-access
restriction, not a core quota (Microsoft support case 2610290040003364). Central US adds
roughly 25–50 ms per redirect. Geo-redundant backups are on, so backups are also kept in
East US 2 (Central US's paired region). See [Moving the database](#moving-the-database-to-east-us-2).

Redis is not used: Kutt's Redis client doesn't support TLS, and Postgres handles
the expected load (up to ~1M clicks/month) on its own.

Deploys run from GitHub Actions on every push to `main`
([.github/workflows/deploy-azure.yaml](.github/workflows/deploy-azure.yaml)).

---

## 1. Decisions

- [x] Short-link domain: **go.odbm.org** (printed QR codes depend on it — never change it)
- [x] DNS for odbm.org is on **Cloudflare**
- [x] Subscription ODB-Production; app in East US 2, database in Central US
- [ ] Admin list (work email addresses)

## 2. Code and repo

- [x] QR code PNG/SVG downloads merged
- [x] Deploy workflow and this checklist merged
- [x] Upstream `docker-build-*` workflows disabled in the Actions tab
- [x] Emails lowercased everywhere (forms and Entra sign-in), so SSO matches existing accounts
- Locally, use `npm ci` (not `npm install`) so `package-lock.json` isn't rewritten

## 3. Entra ID — admin sign-in (**ODBM Short Links**)

App ID `69978f65-94bf-4384-94ee-14d77b98cad0`, single tenant.

- [x] Redirect URI (Web): `https://go.odbm.org/login/oidc`
- [x] Client secret `kutt-prod` — **expires 19 Oct 2028**, stored in Key Vault as `oidc-client-secret`
- [x] API permissions (Microsoft Graph, delegated): `openid`, `profile`, `email`, `User.Read`
- [ ] **Grant admin consent** (needs Cloud Application Administrator or higher). Optional if
      users may consent themselves — then each admin accepts a prompt at first sign-in.
- [ ] Enterprise application → Properties → **Assignment required = Yes**
- [ ] Enterprise application → Users and groups → assign each admin

Kutt reads the email from Entra's **userinfo endpoint**, not the ID token. Optional
claims don't apply there and `preferred_username` isn't returned, so keep
`OIDC_EMAIL_CLAIM=email` and make sure each admin has an Email set on their Entra user
(anyone with an Exchange mailbox does).

## 4. Entra ID — GitHub deploy access (**ODBM Short Link GH Deploy**)

App ID `97dc4631-3b63-468a-9984-8fe53f79c2a3`, no client secret.

- [x] Federated credential `kutt-link-shortener`, subject
      `repo:rbcministries@4130161/kutt-link-shortener@1422599115:ref:refs/heads/main`
      (this repo uses GitHub's immutable, ID-based OIDC subject)
- [ ] Role assignment: **Website Contributor** on `app-kutt-prod` (needs Owner / User Access Administrator)
- [x] GitHub secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`
- [x] GitHub variable `AZURE_WEBAPP_NAME=app-kutt-prod`

## 5. Azure resources

- [x] Resource group `rg-shortlinks-prod` (East US 2)
- [x] **PostgreSQL** `psql-kutt-prod` — Central US, Burstable B1ms, PostgreSQL 16, 32 GiB with autogrow
  - [x] Database `kutt`
  - [x] Backup retention 14 days, geo-redundant backup on
  - [x] Password + Microsoft Entra authentication; Entra admin: david.flickema@odbm.org
        (Kutt uses the password login `kuttadmin`; people can use their Entra account in DB tools)
  - [x] Firewall: allow Azure services (0.0.0.0); TLS required
- [x] **Key Vault** `kv-kutt-prod` (RBAC permission model) with secrets
      `jwt-secret`, `db-password`, `oidc-client-secret`
- [x] **App Service plan** `asp-kutt-prod` — Linux, B1
- [x] **Web App** `app-kutt-prod` — Node 22 LTS, `app-kutt-prod.azurewebsites.net`
  - [x] Startup command: `npm run migrate && npm start`
  - [x] Always On · HTTPS Only · Minimum TLS 1.2 · FTP disabled · basic auth publishing off
  - [x] System-assigned managed identity on
  - [ ] **Key Vault Secrets User** on `kv-kutt-prod` for the Web App's managed identity
        (needs Owner / User Access Administrator) — until then the Key Vault references
        show `AccessToKeyVaultDenied`
- [ ] Application Insights

Creating role assignments needs **Owner**, **User Access Administrator** or
**Role Based Access Control Administrator** — Contributor can't.

## 6. Web App environment variables

All set. Secrets are Key Vault references: `@Microsoft.KeyVault(VaultName=kv-kutt-prod;SecretName=<name>)`

| Setting | Value |
|---|---|
| `PORT` | `8080` |
| `NODE_ENV` | `production` |
| `TRUST_PROXY` | `true` |
| `SCM_DO_BUILD_DURING_DEPLOYMENT` | `false` (GitHub Actions installs dependencies) |
| `SITE_NAME` | `ODBM Short Links` |
| `DEFAULT_DOMAIN` | `go.odbm.org` |
| `DB_CLIENT` | `pg` |
| `DB_HOST` | `psql-kutt-prod.postgres.database.azure.com` |
| `DB_PORT` | `5432` |
| `DB_NAME` | `kutt` |
| `DB_USER` | `kuttadmin` |
| `DB_PASSWORD` | Key Vault → `db-password` |
| `DB_SSL` | `true` |
| `JWT_SECRET` | Key Vault → `jwt-secret` |
| `REDIS_ENABLED` | `false` |
| `DISALLOW_REGISTRATION` | `true` |
| `DISALLOW_ANONYMOUS_LINKS` | `true` |
| `DISALLOW_LOGIN_FORM` | `false` (set `true` after step 9) |
| `ENABLE_RATE_LIMIT` | `true` |
| `OIDC_ENABLED` | `true` |
| `OIDC_ISSUER` | `https://login.microsoftonline.com/0e2553f0-1364-478f-8005-d9ce62146c86/v2.0` |
| `OIDC_CLIENT_ID` | `69978f65-94bf-4384-94ee-14d77b98cad0` |
| `OIDC_CLIENT_SECRET` | Key Vault → `oidc-client-secret` |
| `OIDC_EMAIL_CLAIM` | `email` |
| `OIDC_BUTTON_TEXT` | `Sign in with Microsoft` |

Check that the Key Vault references resolve: Web App → Settings → Environment variables —
each referenced setting should show a green **Key vault Reference** check.

## 7. First deploy

- [ ] Actions tab → **Deploy to Azure App Service** → Run workflow (needs both role assignments)
- [ ] Web App → **Log stream**: migrations run, then `Ready on …`
- [ ] `https://app-kutt-prod.azurewebsites.net` loads the Kutt page

## 8. Domain and HTTPS (Cloudflare)

- [ ] CNAME `go` → `app-kutt-prod.azurewebsites.net`, **Proxy status: DNS only** (grey cloud) —
      the free managed certificate can't be issued or renewed behind Cloudflare's proxy
- [ ] TXT `asuid.go` → `87F00A1806B8927384A2EC41F5EF1BC8404CB9B1465D0FEFCC8DC1E1EF560FEE`
      (the Web App's custom domain verification ID)
- [ ] Web App → Custom domains → add `go.odbm.org`
- [ ] Create the free **App Service Managed Certificate** and bind it (SNI SSL)
- [ ] `http://go.odbm.org` redirects to `https://`

## 9. First sign-in and admins

- [ ] First visit: create the admin account with your **work email**
- [ ] Sign out, **Sign in with Microsoft**, confirm it's the same account
- [ ] Each other admin signs in once with Microsoft, then promote them to admin on the
      Admin page (SSO users are created as regular users)
- [ ] Set `DISALLOW_LOGIN_FORM=true` (SSO only) and restart

## 10. Smoke test

- [ ] Create `go.odbm.org/test` → `https://odbm.org/en/` and confirm it redirects
- [ ] Download the QR code as PNG and SVG; scan both with a phone
- [ ] Change the link's destination; the same QR code follows it
- [ ] Someone not assigned in Entra can't sign in
- [ ] Slugs Kutt reserves (`login`, `admin`, `stats`, `settings`, `api`, …) are rejected

## 11. Operations

- [ ] Application Insights **availability test** on a known short link, every 5 minutes, email alert
- [ ] Calendar reminder for the Entra client secret expiry (**19 Oct 2028**)
- [ ] Node 22 LTS reaches end of support on App Service in **April 2027** — move the Web App
      runtime and the workflow's `node-version` to 24 LTS before then
- [ ] Test a Postgres point-in-time restore once
- [ ] **Rollback:** re-run the deploy workflow on the last good commit
- [ ] **Monthly upstream update:** merge Kutt releases from `thedevs-network/kutt` into a
      branch, check the QR download and Microsoft sign-in still work locally, merge to `main`
      (deploys automatically)

### Moving the database to East US 2

If Microsoft later grants Burstable access in East US 2 (support case 2610290040003364):

1. Create a new Flexible Server in East US 2 (same settings, a new name) with a `kutt` database.
2. Copy the data: `pg_dump` from `psql-kutt-prod`, `pg_restore` into the new server.
3. Set `DB_HOST` to the new server and restart the Web App.
4. Delete `psql-kutt-prod`.

Short links and QR codes are unaffected — they point at go.odbm.org, not the database.
Downtime is only the gap between the final copy and the restart.

### Runbook

- **Add an admin:** assign them to the ODBM Short Links enterprise app → they sign in once →
  promote on the Admin page.
- **Remove an admin:** unassign in Entra, then ban or delete the user in Kutt.
- **Rotate the OIDC secret:** new secret in Entra → update `oidc-client-secret` in
  Key Vault → restart the Web App → delete the old secret.
- **Rotate the JWT secret:** update `jwt-secret` in Key Vault and restart (everyone is signed out).
- **Restore the database:** PostgreSQL server → Restore to a new server → point `DB_HOST` at it → restart.
