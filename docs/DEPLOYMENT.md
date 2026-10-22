# Deployment

> **Releases:** each release publishes a container image
> (`ghcr.io/itworksonmymachine/form-engine:<version>`) and its Liquibase
> changelog (`form-engine-liquibase-<version>.tar.gz` on the release). To
> run a release you need only those; building from source is covered under
> "Running locally" and "Production build".

---

## Environment variables

Copy `.env.example` to `.env.local` and fill in values before running locally.
In production, set these as environment variables in your hosting platform.

### Required

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection string. Alternatively set the standard `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD` (and `PGSSLMODE`), which is how ECS injects credentials; see `deploy/aws-ecs/README.md` |
| `AUTH_SECRET` | Auth.js session encryption secret (generate with `npx auth secret`) |
| `AUTH_OIDC_ISSUER` | OIDC provider issuer URL (auth-engine, Entra, Okta, Auth0) |
| `AUTH_OIDC_CLIENT_ID` | OIDC client ID for this app |
| `AUTH_OIDC_CLIENT_SECRET` | OIDC client secret |
| `AUTH_URL` | Public base URL, e.g. `https://forms.example.com`. Required behind a load balancer: the standalone server otherwise reports `http://0.0.0.0:3000` as its origin and sign-in redirects break. Also the default base for invitation links. |

### Sign-in and tenancy

| Variable | Description |
|----------|-------------|
| `AUTH_JIT_TENANT_SLUG` | Add every signed-in user to this tenant, creating it on first sign-in (single-tenant deployments). Lowercase letters, digits, hyphens. |
| `AUTH_JIT_TENANT_NAME` | Display name used when the JIT tenant is created (default: the slug) |
| `AUTH_SWITCH_ACCOUNT_PROMPT` | OIDC `prompt` sent by "Not you?" (default `select_account`; use `login` for IdPs without an account picker) |

### Email sending

| Variable | Description |
|----------|-------------|
| `EMAIL_PROVIDER` | `smtp`, `ses`, `sendgrid`, or `console` (default: `console`, which only logs: nothing is delivered). An unrecognized value also falls back to `console` and is flagged on `/admin/email`. |
| `EMAIL_FROM_ADDRESS` | Sender email address (must be verified with the provider) |
| `EMAIL_FROM_NAME` | Sender display name |
| `EMAIL_REPLY_TO` | Optional reply-to address (default: replies go to the sender) |
| `EMAIL_SMTP_HOST` | SMTP host (when provider = `smtp`) |
| `EMAIL_SMTP_PORT` | SMTP port (default: 587) |
| `EMAIL_SMTP_USER` | SMTP username |
| `EMAIL_SMTP_PASS` | SMTP password |
| `EMAIL_SMTP_TLS` | Require TLS (default: `true`). Port 465 uses implicit TLS; other ports (587, 25) upgrade with STARTTLS, which `true` makes mandatory and `false` opportunistic. |
| `EMAIL_SES_REGION` | AWS region (when provider = `ses`, default: `us-east-1`) |
| `EMAIL_SES_ACCESS_KEY` | AWS access key. Leave unset on AWS: the task/instance role is used, and needs `ses:SendEmail` on the sender's SES identity. |
| `EMAIL_SES_SECRET_KEY` | AWS secret key |
| `EMAIL_SENDGRID_API_KEY` | SendGrid API key (when provider = `sendgrid`) |

Email is deployment configuration, not an admin setting: credentials stay
in the environment (or the platform's secret store), never in the
database. Admins can see the active provider, sender and any detected
problems at `/admin/email`, and send themselves a test email there. While
the provider is `console`, the Send dialog warns that nothing is
delivered.

For test environments, a mail catcher such as Mailpit works with the
`smtp` provider: `EMAIL_SMTP_HOST=<mailpit host>`, `EMAIL_SMTP_PORT=1025`,
`EMAIL_SMTP_TLS=false` (it has no certificate), plus `EMAIL_SMTP_USER` /
`EMAIL_SMTP_PASS` if it requires authentication. Messages are then
readable in its web UI instead of being delivered.

### Optional

| Variable | Description |
|----------|-------------|
| `DIRECTORY_PROVIDER` | `auth-engine` (default), `entra`, `okta`, or `auth0`; also selects role claims (see `AUTH_PROVIDERS.md`) |
| `ENTRA_TENANT_ID` | Required when `DIRECTORY_PROVIDER=entra` |
| `OKTA_API_TOKEN`, `OKTA_ROLE_GROUP_PREFIX` | When `DIRECTORY_PROVIDER=okta` |
| `AUTH0_MGMT_CLIENT_ID`, `AUTH0_MGMT_CLIENT_SECRET` | When `DIRECTORY_PROVIDER=auth0` |
| `NODE_TLS_REJECT_UNAUTHORIZED` | Set to `0` for local dev with self-signed certs only |

### License

| Variable | Description |
|----------|-------------|
| `FORM_ENGINE_LICENSE_KEY` | License key (secret). Without a valid key (missing, invalid, or expired) the app works normally but runs as an unlicensed copy: public and embedded forms show an "Unlicensed copy, for evaluation only" notice and `/admin` shows a banner. The banner also warns 30 days before a license expires. Keys are verified offline; nothing is sent anywhere. |

### Public URLs, framing, and CORS

| Variable | Description |
|----------|-------------|
| `APP_BASE_URL` | Public base URL (e.g. `https://forms.example.com`) used in invitation links. Recommended in production; without it the request URL is used. Client headers (Origin, X-Forwarded-Host) are never trusted. |
| `EMBED_FRAME_ANCESTORS` | Origins allowed to frame `/forms/{id}/embed`, comma-separated, or `*`. Default: same origin only. All other pages always send `X-Frame-Options: DENY`. |
| `SUBMISSIONS_CORS_ORIGINS` | Origins allowed to call `/api/submissions` from browser JS, comma-separated, or `*`. Default: none. The embed iframe posts same-origin and does not need this. |

### Bot protection

| Variable | Description |
|----------|-------------|
| `CAPTCHA_HMAC_SECRET` | Secret that signs the ALTCHA proof-of-work challenges of forms with `captcha: true` (generate with `openssl rand -hex 32`; keep it in the secret store). Without it those forms refuse submissions (503) and log an error; forms without `captcha` are unaffected. Rotating it invalidates challenges already issued (at most an hour old). |
| `RATE_LIMIT_CAPTCHA_CHALLENGES` | `/api/captcha/challenge` calls per IP per 10 minutes (default `300`) |

The widget needs a secure context (HTTPS, or `localhost` in development):
browsers only expose Web Crypto there, and over plain HTTP the widget
stays in an error state, so forms with `captcha` can't be submitted.

Self-hosted: no third-party service, account, or cookies. The widget solves
in Web Workers created from the app's own bundle (`blob:` workers); keep
that in mind if you add a `worker-src` CSP. Solved challenges are recorded
in the `captcha_spent` table (one use each).

### Rate limiting and proxies

| Variable | Description |
|----------|-------------|
| `TRUSTED_PROXY_HOPS` | Proxies in front of the app (default `1`, e.g. one ALB). The client IP is read that many entries from the right of `X-Forwarded-For`; leftmost entries are client-controlled. |
| `RATE_LIMIT_SUBMISSIONS_TOTAL` | `/api/submissions` calls per IP per 10 minutes (default `3000`; auto-save runs every few seconds per open form, and offices share NAT addresses) |
| `RATE_LIMIT_SUBMISSIONS_CREATE` | New drafts per IP per 10 minutes (default `100`) |
| `RATE_LIMIT_DISABLED` | `true` disables the app-level limiter (local testing only) |

Counters live in the unlogged `rate_limits` table (shared across tasks).
Pair them with a WAF rate-based rule at the edge.

### Webhooks

| Variable | Description |
|----------|-------------|
| `WEBHOOK_ALLOWED_HOSTS` | Exact hostnames allowed even though they resolve to private addresses (e.g. an internal n8n), comma-separated. All other webhook hosts must resolve to public addresses. |

### Integration API (M2M)

The integration API (`/api/integration/*`) is **disabled until
`M2M_AUDIENCE` is set**.

| Variable | Description |
|----------|-------------|
| `M2M_AUDIENCE` | Required. Expected `aud` of client-credentials tokens (e.g. the Entra app ID URI) |
| `M2M_ISSUER` | Token issuer if different from `AUTH_OIDC_ISSUER` (e.g. Entra v1 tokens: `https://sts.windows.net/<tid>/`) |
| `M2M_REQUIRED_SCOPE` | Scope or app role required in `scope`/`scp`/`roles` (default `forms.integration`) |
| `M2M_CLIENT_TENANTS` | JSON map of client id to allowed tenant ids, or `"*"`: `{"n8n-client":["<tenant-uuid>"]}`. Unlisted clients are refused. |

Signing keys are discovered from `<issuer>/.well-known/openid-configuration`.

See `.env.example` for the full list with comments.

---

## Prerequisites

- **Node.js 24 LTS** (or 22.22.2+; `jsdom`, used for server-side HTML sanitizing, requires it). The Docker image and CI use Node 24.
- **PostgreSQL 15+** — all form data, submissions, contacts, campaigns
  are stored in PostgreSQL
- **Liquibase** — database migrations (run before first start and on upgrades)
- **An OIDC identity provider**: auth-engine (OpenIddict), Entra ID, Okta, or Auth0

---

## Database setup

Run Liquibase migrations to create/update the schema, before the first
start and before starting each new version. From a release, unpack
`form-engine-liquibase-<version>.tar.gz` (it contains `liquibase/`) and run:

```bash
docker run --rm -v "$PWD/liquibase:/liquibase/changelog:ro" liquibase/liquibase:4.31.1 \
  --search-path=/liquibase/changelog --changelog-file=master-change-log.xml \
  --url=jdbc:postgresql://<host>:5432/<database> --username=<user> --password=<password> \
  --contexts=production update
```

From a source checkout, `cd liquibase && liquibase update` uses
`liquibase/liquibase.properties` (local development).

**Contexts:** deployments must run Liquibase with a context, e.g.
`liquibase update --contexts=production`. Changesets without a context
always run; the `dev-seed` changeset (a local developer tenant) runs only
when no context is given, as in local development. Without `--contexts`,
a deployment would get that developer tenant.

**Adopter changesets:** the master changelog optionally includes
`liquibase/changelog/adopter/change-log.xml` (`errorIfMissing="false"`). A
deploying organization adds it (e.g. to create the application's login
role in its own pipeline's syntax) next to the release's changelog, without
editing the released files.

**First tenant:** a new deployment has no tenants. Either set
`AUTH_JIT_TENANT_SLUG` (the tenant is created at the first sign-in) or
insert one: `INSERT INTO tenants (slug, name) VALUES ('<slug>', '<name>');`

---

## Running locally

```bash
npm install
cp .env.example .env.local
# fill in .env.local
npm run dev
```

The app runs at `http://localhost:3000`.

- Admin UI: `http://localhost:3000/admin` (requires OIDC login)
- Dashboard: `http://localhost:3000/dashboard`
- Public forms: `http://localhost:3000/forms/{formId}`

---

## Production build

```bash
npm run build:sdk   # builds public/form-engine-sdk.js (embed SDK)
npm run build       # output: 'standalone' in next.config.js
```

With standalone output, run the server with `node .next/standalone/server.js`
(after copying `.next/static` and `public/` next to it, as the Dockerfile
does) rather than `npm start`. The Docker image is the supported way to run
a production build.

---

## Deployment options

### Docker

Run a release:

```bash
docker run -p 3000:3000 --env-file .env.production ghcr.io/itworksonmymachine/form-engine:<version>
```

Or build from source: `docker build -t form-engine .`. The `Dockerfile` is multi-stage (deps, build, runtime), builds the embed
SDK and the standalone Next.js server, and runs as a non-root user
(`nextjs`, uid 1001) on port 3000. Environment variables are read at
runtime; none are baked into the image.

Migrations are not run by the app container. Apply the Liquibase changelog
(`liquibase/master-change-log.xml`) as a separate step before starting a
new version, with `--contexts=production` (see Database setup for a
working `liquibase/liquibase` invocation).

### Quality records and the audit log

- **Submitted records are immutable in the database.** A trigger
  (`protect_submitted_records`) rejects any change to a submitted
  submission's answers, identity, status, or timestamps, and rejects
  deleting it. This holds for every client, including direct SQL.
- **Forms with submitted records can't be deleted** (the API returns 409);
  unpublish them instead. Deleting a form removes only its drafts.
- **Audit log** (`audit_log`, append-only by trigger): submission views,
  exports (with their filters), and form deletions, visible at
  `/admin/audit` to `admin` and `security_admin`.
- **Deliberate deletion** (e.g. a records-retention procedure) must opt in
  inside its transaction and should be recorded by whoever runs it:

  ```sql
  BEGIN;
  SET LOCAL form_engine.allow_record_delete = 'on';
  DELETE FROM submissions WHERE tenant_id = '<tenant>' AND submitted_at < '<cutoff>';
  COMMIT;
  ```

  The same setting allows deleting audit entries; use it only under your
  records-retention policy.

### Health checks

| Endpoint | Use | Behavior |
|----------|-----|----------|
| `GET /api/health` | Load balancer / container liveness | 200 `{"status":"ok"}`; never touches the database, so a DB blip doesn't cycle healthy tasks |
| `GET /api/health?db=1` | Readiness / diagnostics | Also runs `SELECT 1`; 503 `{"status":"error","db":"unavailable"}` on failure (details are logged, not returned) |

The image also declares a Docker `HEALTHCHECK` against `/api/health`.

### AWS (ECS Fargate)

Workflow templates and the container contract (variables, secrets, health
check, database role) are in `deploy/aws-ecs/README.md`.

The infrastructure (ECS service, ALB rule, WAF, secrets, database
creation, DNS) lives in the deploying organization's own infrastructure
repo. Deploy workflows run from a small deployment repository of that
organization and install a published release: they copy the image into
its ECR and apply the release's changelog inside its VPC.

Requirements:
- PostgreSQL database reachable from the tasks
- OIDC provider reachable for sign-in
- Outbound network access for email sending (SMTP/SES/SendGrid) and webhooks

Other Next.js hosts (including Vercel) should work with the same
environment variables but are not tested.

---

## n8n integration

### Form submission webhooks

Forms can fire webhooks on submission to n8n for post-processing:

1. Add a **Webhook** trigger node in your n8n workflow
2. Set method to `POST`
3. Configure the webhook URL as a `WebhookTarget` in the form schema's
   `submission.targets` array
4. The form data arrives as the request body

### Campaign orchestration (coming soon)

n8n will drive campaign workflows via integration APIs under
`/api/integration/`. These endpoints use M2M bearer tokens (OAuth2 client credentials) from
the OIDC provider (see Integration API above). The campaign endpoints are not built yet.

---

## Security checklist before going to production

- [ ] Set `APP_BASE_URL`, and `TRUSTED_PROXY_HOPS` to match the proxy chain
- [ ] Add a WAF rate-based rule in front of the app (the app-level limiter is the second layer)
- [ ] Set `EMBED_FRAME_ANCESTORS` / `SUBMISSIONS_CORS_ORIGINS` only if embedding on other sites
- [ ] List internal webhook hosts (e.g. n8n) in `WEBHOOK_ALLOWED_HOSTS`
- [ ] If using the integration API: set the `M2M_*` variables
- [ ] CAPTCHA or honeypot if any high-traffic public (anonymous) forms are published
- [ ] Set `EMAIL_PROVIDER` to a real provider (not `console`)
- [ ] Ensure `AUTH_SECRET` is a strong random value
- [ ] Ensure database is not publicly accessible
- [ ] Review HIPAA implications if any form fields collect PHI
