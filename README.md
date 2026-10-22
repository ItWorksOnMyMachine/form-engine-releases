# Form Engine

A white-label, multi-tenant form engine. Forms are JSON schemas stored in
PostgreSQL and built in a browser-based form builder; no form needs custom
code. Respondents fill them out on a hosted page or in an embed on any
website; submissions are stored as immutable records and can be
forwarded to webhooks (e.g. n8n).

This repository holds the **releases and documentation**. The application
ships as a container image; there is no source code here.

## Features

- **Form builder:** pages, sections, conditional logic (AND/OR), repeatable
  groups, checklists (pass/fail criteria with comments), and the usual field
  types (text, email, phone, number, date, select, checkbox groups, radio,
  rating, sentiment)
- **Who can respond:** anyone with the link, invited people (single-use
  links), or signed-in members of the tenant
- **Embedding:** JS SDK for inline, dialog, and popup embeds, with prefill,
  tracking-value forwarding, and success redirects on the host page
- **Bot protection:** self-hosted proof-of-work CAPTCHA (ALTCHA, no
  third-party service), honeypot, and rate limiting
- **Records:** submitted records are immutable; views, exports, and admin
  changes go to an append-only audit log; dashboard with filters and CSV export
- **Sign-in:** any OIDC provider (Microsoft Entra ID, Okta, Auth0, OpenIddict),
  roles from the identity provider, per-form access grants
- **Accessible:** built and tested against WCAG 2.1 AA
- **White-label:** per-tenant branding; no vendor names in the product

## Releases

Each [release](../../releases) provides:

| Artifact | Where |
|----------|-------|
| Container image | `ghcr.io/itworksonmymachine/form-engine:<version>` (also `:latest`), linux/amd64 |
| Database migrations | `form-engine-liquibase-<version>.tar.gz` on the release, with `SHA256SUMS` |
| Open-source notices | `THIRD-PARTY-NOTICES.txt` on the release; also served by the app at `/third-party-notices.txt` |
| Release notes | New settings and migrations to apply for that version |

Pin a version (or the digest in the release notes) in production; `latest`
moves with every release.

## Running it

You need PostgreSQL 15+, an OIDC identity provider, and HTTPS in front of
the app (sign-in cookies and the CAPTCHA require a secure context).

1. **Create the schema** with the release's migrations (Liquibase), before
   the first start and before every upgrade:

   ```bash
   VERSION=1.0.0.0
   curl -fsSLO https://github.com/ItWorksOnMyMachine/form-engine-releases/releases/download/v$VERSION/form-engine-liquibase-$VERSION.tar.gz
   tar -xzf form-engine-liquibase-$VERSION.tar.gz   # -> ./liquibase
   docker run --rm -v "$PWD/liquibase:/liquibase/changelog:ro" liquibase/liquibase:4.31.1 \
     --search-path=/liquibase/changelog --changelog-file=master-change-log.xml \
     --url=jdbc:postgresql://<host>:5432/<database> --username=<user> --password=<password> \
     --contexts=production update
   ```

2. **Run the app** with your configuration (see `.env.example` and
   [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)):

   ```bash
   docker run -p 3000:3000 --env-file form-engine.env ghcr.io/itworksonmymachine/form-engine:$VERSION
   ```

   The container listens on port 3000 as a non-root user; `GET /api/health`
   is the liveness check.

3. **Sign in** at `/admin` with a user that has the `admin` role in your
   identity provider ([docs/AUTH_PROVIDERS.md](docs/AUTH_PROVIDERS.md)),
   then create a form ([docs/ADDING_A_FORM.md](docs/ADDING_A_FORM.md)).

Deploying on AWS ECS Fargate: GitHub Actions templates that copy a release
into your own registry and roll out the service, and that apply the
migrations inside your VPC, are in [deploy/aws-ecs](deploy/aws-ecs/README.md).

## Documentation

| Document | Contents |
|----------|----------|
| [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) | Configuration reference, database, health checks, security checklist |
| [docs/AUTH_PROVIDERS.md](docs/AUTH_PROVIDERS.md) | Setting up Entra ID, Okta, Auth0, or OpenIddict |
| [docs/ADDING_A_FORM.md](docs/ADDING_A_FORM.md) | Building, publishing, sharing, and embedding forms |
| [docs/SCHEMA_REFERENCE.md](docs/SCHEMA_REFERENCE.md) | The form JSON schema (import/export, conditions, payload shape) |
| [sdk/README.md](sdk/README.md) | Embedding forms in another website |
| [deploy/aws-ecs](deploy/aws-ecs/README.md) | ECS Fargate deployment templates |

## Licensing

The image is free to evaluate: without a license key everything works, but
public and embedded forms show an "Unlicensed copy, for evaluation only"
notice and the admin area shows a banner. A license key removes them; set
it as `FORM_ENGINE_LICENSE_KEY`. Keys are verified offline (nothing is sent
anywhere) and may carry an expiry date, after which the notice returns.
Forms and records keep working whether or not a key is present.

To license the form engine, contact [sales@clakholdings.com](mailto:sales@clakholdings.com). Terms: see
[LICENSE](LICENSE).
