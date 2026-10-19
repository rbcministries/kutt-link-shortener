# Deploying Kutt to Azure (go.odbm.org)

Kutt runs as a plain Node.js app on Azure App Service (no Docker), backed by
Azure Database for PostgreSQL. Short links are `https://go.odbm.org/{slug}`;
each link's destination is set when it's created (e.g. `https://odbm.org/en/{slug}`).
Admins sign in with Microsoft Entra ID.

| Component | Service | Approx. cost/month |
|---|---|---|
| App | App Service plan, Linux B1 + Web App (Node 22 LTS) | ~$13 |
| Database | PostgreSQL Flexible Server, Burstable B1ms | ~$15–20 |
| Secrets | Key Vault | ~$0–1 |
| Monitoring | Application Insights | ~$0–5 |
| HTTPS | App Service Managed Certificate | $0 |

Redis is not used: Kutt's Redis client doesn't support TLS, and Postgres handles
the expected load (up to ~1M clicks/month) on its own.

Deploys run from GitHub Actions on every push to `main`
([.github/workflows/deploy-azure.yaml](.github/workflows/deploy-azure.yaml)).

---

## 1. Decisions

- [x] Short-link domain: **go.odbm.org** (printed QR codes depend on it — never change it)
- [ ] Who adds DNS records for odbm.org
- [ ] Who creates Entra app registrations (M365 admin)
- [ ] Azure subscription and region (e.g. East US 2)
- [ ] Admin list (work email addresses)

## 2. Code and repo

- [ ] Merge the `feature/qr-download` PR into `main`
- [ ] Merge the `chore/azure-deploy` PR (this file + deploy workflow) into `main`
- [ ] In GitHub **Actions**, disable the upstream workflows that push to Docker Hub
      (`docker-build-latest`, `docker-build-development`, `docker-build-release`) —
      they would fail on every push to `main`. Disabling them in the UI avoids
      editing upstream files.
- [ ] Locally, use `npm ci` (not `npm install`) so `package-lock.json` isn't rewritten

## 3. Entra ID — admin sign-in

- [ ] App registration **"Kutt Short Links"**, single tenant
- [ ] Redirect URI (Web): `https://go.odbm.org/login/oidc`
- [ ] Client secret — record its expiry date
- [ ] Note the **Tenant ID** and **Client ID**
- [ ] Make sure the ID token carries an email: add optional claim `email`,
      or use `OIDC_EMAIL_CLAIM=preferred_username`
- [ ] Enterprise application → **Assignment required = Yes**, assign only the admins

## 4. Entra ID — GitHub deploy access

- [ ] App registration **"Kutt GitHub Deploy"**
- [ ] Federated credential: GitHub Actions, repo `rbcministries/kutt-link-shortener`,
      entity type **Branch**, branch `main`
- [ ] Role assignment: **Website Contributor** on the Web App only
- [ ] GitHub repo **secrets**: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`
- [ ] GitHub repo **variable**: `AZURE_WEBAPP_NAME` (the Web App's name)

## 5. Azure resources

- [ ] Resource group, e.g. `rg-shortlinks-prod`
- [ ] **PostgreSQL Flexible Server** — Burstable B1ms, PostgreSQL 16
  - [ ] Database `kutt`
  - [ ] Backup retention 14–35 days
  - [ ] Networking: allow Azure services; add your IP only temporarily for admin work
- [ ] **Key Vault**, with secrets:
  - [ ] `jwt-secret` — new value from `openssl rand -hex 32` (not the local one)
  - [ ] `db-password`
  - [ ] `oidc-client-secret`
- [ ] **App Service plan** — Linux, B1
- [ ] **Web App** — Publish: Code, Runtime: Node 22 LTS, OS: Linux
  - [ ] Startup command: `npm run migrate && npm start`
  - [ ] Always On: On · HTTPS Only: On · Minimum TLS: 1.2 · FTP: Disabled
  - [ ] System-assigned managed identity: On, with **Key Vault Secrets User** on the vault

## 6. Web App environment variables

Secrets use Key Vault references: `@Microsoft.KeyVault(SecretUri=https://<vault>.vault.azure.net/secrets/<name>/)`

| Setting | Value |
|---|---|
| `PORT` | `8080` |
| `NODE_ENV` | `production` |
| `TRUST_PROXY` | `true` |
| `SCM_DO_BUILD_DURING_DEPLOYMENT` | `false` (GitHub Actions installs dependencies) |
| `SITE_NAME` | e.g. `ODBM Links` |
| `DEFAULT_DOMAIN` | `go.odbm.org` |
| `DB_CLIENT` | `pg` |
| `DB_HOST` | `<server>.postgres.database.azure.com` |
| `DB_PORT` | `5432` |
| `DB_NAME` | `kutt` |
| `DB_USER` | database admin user |
| `DB_PASSWORD` | Key Vault reference → `db-password` |
| `DB_SSL` | `true` |
| `JWT_SECRET` | Key Vault reference → `jwt-secret` |
| `REDIS_ENABLED` | `false` |
| `DISALLOW_REGISTRATION` | `true` |
| `DISALLOW_ANONYMOUS_LINKS` | `true` |
| `ENABLE_RATE_LIMIT` | `true` |
| `OIDC_ENABLED` | `true` |
| `OIDC_ISSUER` | `https://login.microsoftonline.com/<tenant-id>/v2.0` |
| `OIDC_CLIENT_ID` | client ID from step 3 |
| `OIDC_CLIENT_SECRET` | Key Vault reference → `oidc-client-secret` |
| `OIDC_EMAIL_CLAIM` | `email` or `preferred_username` |
| `OIDC_BUTTON_TEXT` | `Sign in with Microsoft` |

## 7. First deploy

- [ ] Run **Deploy to Azure App Service** (Actions tab → Run workflow), or push to `main`
- [ ] Web App → **Log stream**: migrations run, then `Ready on …`
- [ ] `https://<app>.azurewebsites.net` loads the Kutt page

## 8. Domain and HTTPS

- [ ] DNS: CNAME `go` → `<app>.azurewebsites.net`
- [ ] DNS: TXT `asuid.go` → the Web App's domain verification ID
- [ ] If odbm.org DNS is on Cloudflare, set the `go` record to **DNS only** (grey cloud)
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
- [ ] Calendar reminder for the Entra client secret expiry
- [ ] Test a Postgres point-in-time restore once
- [ ] **Rollback:** re-run the deploy workflow on the last good commit
- [ ] **Monthly upstream update:** merge Kutt releases from `thedevs-network/kutt` into a
      branch, check the QR download still works locally, merge to `main` (deploys automatically)

### Runbook

- **Add an admin:** assign them to the Entra enterprise app → they sign in once →
  promote on the Admin page.
- **Remove an admin:** unassign in Entra, then ban or delete the user in Kutt.
- **Rotate the OIDC secret:** new secret in Entra → update `oidc-client-secret` in
  Key Vault → restart the Web App → delete the old secret.
- **Rotate the JWT secret:** update `jwt-secret` in Key Vault and restart (everyone is signed out).
- **Restore the database:** PostgreSQL server → Restore to a new server → point `DB_HOST` at it → restart.
