# Deploying to AWS ECS (Fargate)

Templates for running the form engine on ECS Fargate behind an ALB with
Aurora PostgreSQL, from the published releases (no source code needed).
The deploying organization keeps a small **deployment repository** with
these workflows in `.github/workflows/`, its own Liquibase changesets (if
any) in `liquibase/changelog/adopter/`, and owns its infrastructure
(Terraform or otherwise) in its own repository.

| File | Purpose |
|------|---------|
| `workflows/app-deploy.yml` | Copy a release image (`ghcr.io/itworksonmymachine/form-engine:<version>`) into ECR as the version tag + a mutable environment tag, force a new ECS deployment, tag the deployment repository |
| `workflows/liquibase-deploy.yml` | Create the database (optional db-init Lambda), download and verify the release's changelog, add the adopter changesets, upload to S3, run Liquibase as a one-off ECS task with `--contexts=production` |

Both take the release version as input (e.g. `1.0.0.0`). Run
`liquibase-deploy.yml` first, then `app-deploy.yml`, for the same version;
the release notes say when a release has migrations or new settings.

## Contract with the infrastructure

The ECS service should run the image tagged with the environment name
(e.g. `form-engine:uat`), so shipping code is `app-deploy.yml` and never a
Terraform apply.

**Container:** port `3000`; health check `GET /api/health` (liveness, no
database; use `/api/health?db=1` only for diagnostics). Runs as non-root.
`deployment_minimum_healthy_percent = 100` and a circuit breaker are
recommended for zero-downtime releases.

**Environment variables** (plain):

| Variable | Value |
|----------|-------|
| `AUTH_URL` | Public URL, e.g. `https://forms.example.com` (required) |
| `AUTH_OIDC_ISSUER` | e.g. `https://login.microsoftonline.com/<tenant-id>/v2.0` |
| `DIRECTORY_PROVIDER` | e.g. `entra` |
| `ENTRA_TENANT_ID` | when `DIRECTORY_PROVIDER=entra` |
| `AUTH_JIT_TENANT_SLUG`, `AUTH_JIT_TENANT_NAME` | single-tenant deployments |
| `PGHOST`, `PGPORT`, `PGDATABASE` | Aurora endpoint, port, database name |
| `PGSSLMODE` | `no-verify` for Aurora: TLS-encrypted, server certificate not verified. (`require` makes node-postgres verify against Node's CA list, which lacks the Amazon RDS CA, so connections fail.) |
| `TRUSTED_PROXY_HOPS` | `1` behind one ALB |
| `EMAIL_PROVIDER`, `EMAIL_FROM_ADDRESS`, `EMAIL_FROM_NAME`, `EMAIL_SES_REGION`, optional `EMAIL_REPLY_TO` | if the app sends invitations. With `ses`, leave the SES keys unset: the **task role** needs `ses:SendEmail` on the sender's SES identity (and on its configuration set, if the identity has a default one). Check with "Send test email to me" on `/admin/email`. |
| `WEBHOOK_ALLOWED_HOSTS` | internal webhook hosts (e.g. n8n), if any |

**Secrets** (injected by ECS from Secrets Manager, one key each):

| Variable | Source |
|----------|--------|
| `PGUSER`, `PGPASSWORD` | database app-user secret (`username`, `password`) |
| `AUTH_SECRET` | random, generated per environment |
| `AUTH_OIDC_CLIENT_ID`, `AUTH_OIDC_CLIENT_SECRET` | the IdP app registration |
| `FORM_ENGINE_LICENSE_KEY` | your license key (without it, public forms show an "unlicensed copy" notice) |
| `CAPTCHA_HMAC_SECRET` | random, generated per environment (required by forms with bot protection) |

Full variable reference: `docs/DEPLOYMENT.md`. The app never needs
`DATABASE_URL` in ECS; it reads the standard `PG*` variables.

**Database role:** the app needs a login role that owns (or has full DML
and DDL on) its database. If your pipeline creates that role through
Liquibase, put the changeset in `liquibase/changelog/adopter/change-log.xml`
in the deployment repository. The release's master changelog includes that
file optionally (`errorIfMissing="false"`), and `liquibase-deploy.yml` adds
it to the downloaded changelog before uploading.
`liquibase-deploy.yml` passes the app-user password to the task as
`APP_USER_PASSWORD` when `vars.DB_APP_USER_SECRET_ID` is set, **as a
SCRAM-SHA-256 verifier, not the plaintext**: Liquibase logs every statement
with its parameters substituted, and a verifier can't be used to log in.
PostgreSQL accepts it anywhere a password goes (`CREATE/ALTER ROLE ...
PASSWORD '<value>'`); don't wrap it in functions that expect plaintext.

## First deployment, per environment

1. IdP: register the application (see `docs/AUTH_PROVIDERS.md`), redirect
   URI `https://<host>/api/auth/callback/auth-engine`.
2. Infrastructure: secrets, ECR repository, ECS service, ALB rule, DNS.
3. `liquibase-deploy.yml`: creates the database and schema.
4. `app-deploy.yml`: copies the release image and rolls out the service.
5. Sign in once as an administrator (with JIT on, the tenant is created on
   first sign-in), then configure branding and import forms.
