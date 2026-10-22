# Auth Provider Setup

This document explains how to configure the forms engine to work with
different identity providers. The system uses a pluggable architecture —
the OIDC login flow and the directory API (user management) are both
provider-specific and can be swapped via configuration.

---

## Architecture

Authentication has two separate concerns:

1. **OIDC Login** — How users sign in. Configured via `AUTH_OIDC_*` env
   vars. Next-auth handles the protocol; you just point it at the provider.

2. **Directory API** — How the app manages users and roles (search, create,
   delete, assign roles). Each provider has its own management API. The
   forms app calls these through an abstraction layer in `src/lib/directory/`.

```
┌─────────────────────────────────────────────┐
│  src/lib/directory/                         │
│                                             │
│  types.ts         ← shared interfaces      │
│  index.ts         ← factory + re-exports   │
│  auth-engine.ts   ← OpenIddict provider    │
│  okta.ts          ← Okta                   │
│  entra.ts         ← Microsoft Entra ID     │
│  auth0.ts         ← Auth0                  │
└─────────────────────────────────────────────┘
```

The `DIRECTORY_PROVIDER` env var selects which implementation to use
(default `auth-engine`). It also selects which token claims hold roles
and which scopes are requested (`getOidcRoleConfig()` in
`src/lib/directory/index.ts`). All app code imports from
`@/lib/directory`; it never references a specific provider directly.

**Callback URL:** the next-auth provider id is `auth-engine` for every
IdP (see `src/lib/auth.ts`), so the redirect URI is always
`https://<host>/api/auth/callback/auth-engine`, even for Okta, Entra, or
Auth0.

**Status:** auth-engine is the only provider used against a live IdP so
far. Claim handling for every provider is unit-tested
(`src/lib/__tests__/auth-claims.test.ts`), and the Entra claim shape was
verified end to end against a mock OIDC server; the Okta, Entra, and Auth0
*directory* providers (user management via their APIs) remain untested.

---

## Sign-in behavior (all providers)

- **`AUTH_URL` is required in production.** The standalone server reports
  its bind address (`http://0.0.0.0:3000`) as the request origin, and load
  balancers such as AWS ALB don't send `X-Forwarded-Host`, so Auth.js
  can't infer the public URL. Set `AUTH_URL=https://<host>`.
- **Two Auth.js instances.** `src/lib/auth.config.ts` is Edge-safe and used
  by the middleware; `src/lib/auth.ts` adds the Node-only sign-in event.
  Never import database code into `auth.config.ts`.
- **JIT membership.** With `AUTH_JIT_TENANT_SLUG` set, every sign-in adds
  the user to that tenant (created on first use). Name and email on all of
  a user's memberships are refreshed at each sign-in. With JIT on, revoke
  access in the IdP: removal in `/admin/users` lasts only until the user's
  next sign-in.
- **Respondents vs. back office.** A user with no app role and no form grant
  is sent from `/admin` and `/dashboard` to `/my-forms`.
- **Switching accounts.** "Not you?" signs out locally and returns through
  the sign-in page with `prompt=select_account` (override with
  `AUTH_SWITCH_ACCOUNT_PROMPT`, e.g. `login` for IdPs without an account
  picker).

---

## Switching Providers

To switch from one provider to another:

1. Update `AUTH_OIDC_*` env vars to point at the new IdP
2. Set `DIRECTORY_PROVIDER` to the new provider name
3. Add any provider-specific env vars (see sections below)
4. Restart the app
5. All users must log out and back in (JWT tokens contain roles from
   the old IdP)

No code changes required.

---

## Provider: Auth Engine (OpenIddict)

The default provider. A self-hosted OpenIddict-based OIDC server.

### OIDC Configuration

| Variable | Value |
|----------|-------|
| `AUTH_OIDC_ISSUER` | Base URL of the auth engine (e.g., `https://login.domain.com`) |
| `AUTH_OIDC_CLIENT_ID` | `forms-engine` (registered in auth engine seed data) |
| `AUTH_OIDC_CLIENT_SECRET` | Client secret from auth engine configuration |
| `DIRECTORY_PROVIDER` | `auth-engine` |

### Role Claims

Roles arrive in the ID token as a `roles` claim (string array).
Requested via the `roles` scope.

### Directory API

Uses the auth engine's built-in directory services API:
- Authentication: OAuth2 client credentials flow (reuses `AUTH_OIDC_*` vars)
- Base URL: Same as `AUTH_OIDC_ISSUER`
- Endpoints: `/api/directory/users`, `/api/directory/apps/{appId}/roles`

### Local Development

```env
AUTH_OIDC_ISSUER=https://localhost:7091/
AUTH_OIDC_CLIENT_ID=forms-engine
AUTH_OIDC_CLIENT_SECRET=forms-engine-dev-secret
DIRECTORY_PROVIDER=auth-engine
NODE_TLS_REJECT_UNAUTHORIZED=0
```

---

## Provider: Okta

**Implementation:** `src/lib/directory/okta.ts`

### Environment Variables

```env
AUTH_OIDC_ISSUER=https://your-org.okta.com
AUTH_OIDC_CLIENT_ID=<okta-app-client-id>
AUTH_OIDC_CLIENT_SECRET=<okta-app-client-secret>
DIRECTORY_PROVIDER=okta
OKTA_API_TOKEN=<okta-admin-api-token>
OKTA_ROLE_GROUP_PREFIX=forms-engine    # optional, defaults to forms-engine
```

### Okta Setup Steps

1. **Create an OIDC App Integration**
   - Sign in to Okta Admin Console
   - Applications → Create App Integration → OIDC - Web Application
   - Sign-in redirect URI: `https://forms.domain.com/api/auth/callback/auth-engine`
   - Grant type: Authorization Code
   - Assignments: Assign the app to users/groups who need access
   - Copy the Client ID and Client Secret

2. **Enable Groups Claim**
   - In the OIDC app → Sign On → OpenID Connect ID Token
   - Groups claim type: Filter
   - Groups claim filter: `Starts with` → `forms-engine`
   - This ensures the `groups` claim in the ID token contains the
     role-mapped groups

3. **Create Role Groups**
   - Directory → Groups → Add Group for each role:
     - `forms-engine:security_admin`
     - `forms-engine:admin`
     - `forms-engine:creator`
   - Assign users to groups based on their roles

4. **Create an API Token**
   - Security → API → Tokens → Create Token
   - This token is used for the directory API calls (user management)
   - Store as `OKTA_API_TOKEN`

### How Roles Work

Okta doesn't have per-app roles. Instead, we use Groups with a naming
convention: `{prefix}:{role_name}`. The provider automatically creates
groups when assigning roles if they don't exist. The `groups` OIDC
scope and claim filter ensure these arrive in the ID token.

---

## Provider: Microsoft Entra ID

**Implementation:** `src/lib/directory/entra.ts` (directory), claim handling
in `src/lib/directory/oidc-config.ts`

### Environment Variables

```env
AUTH_URL=https://forms.example.com
AUTH_OIDC_ISSUER=https://login.microsoftonline.com/<tenant-id>/v2.0
AUTH_OIDC_CLIENT_ID=<app-registration-client-id>
AUTH_OIDC_CLIENT_SECRET=<app-registration-client-secret>
DIRECTORY_PROVIDER=entra
ENTRA_TENANT_ID=<entra-tenant-id>
# Optional, single-tenant deployments: add every signed-in user to one tenant
AUTH_JIT_TENANT_SLUG=<slug>
AUTH_JIT_TENANT_NAME=<display name>
```

### User identity: `oid`, not `sub`

With `DIRECTORY_PROVIDER=entra` the app identifies users by the `oid` claim
(the Entra object id). Entra's `sub` is a *pairwise* id, different for every
application, and Microsoft Graph never returns it; keying on it would detach
users from memberships and grants added through the directory (the
`/admin/users` page and the user picker use Graph ids). Memberships, grants,
and submitter identity therefore all store the `oid`.

Sign-in fails if a token has no `oid` claim (it is present in v2.0 ID tokens
by default).

### Entra Setup Steps

1. **Register an application**
   - Microsoft Entra ID, App registrations, New registration
   - Supported account types: single tenant
   - Redirect URI (Web): `https://<host>/api/auth/callback/auth-engine`
     (the path ends in `auth-engine` for every IdP)
   - Copy the Application (client) ID and Directory (tenant) ID

2. **Create a client secret** (Certificates & secrets). Store it as
   `AUTH_OIDC_CLIENT_SECRET`; note its expiry date.

3. **Define app roles** (App roles, Create app role; allowed member types:
   Users/Groups):

   | Display name | Value |
   |--------------|-------|
   | Security Admin | `security_admin` |
   | Admin | `admin` |
   | Creator | `creator` |
   | User | `user` |

   `user` grants nothing in the app (only the three roles above are acted
   on); it exists so the respondents group can be assigned. Entra offers
   "Default Access" only when an app defines no roles of its own.

4. **Restrict who can sign in**: Enterprise applications, the app,
   Properties, set **Assignment required** to Yes. Then under Users and
   groups, assign:
   - a users group (everyone who may fill out forms) with the User role, and
   - role groups to the matching app roles.

   Assign roles to **groups**, not the `groups` claim: tokens for users in
   many groups hit the groups-overage limit and omit that claim, while app
   roles always arrive in the `roles` claim.

   Leave **"Microsoft Entra roles can be assigned to the group"** off when
   creating these groups. It is for directory roles (Global Administrator
   etc.), not app roles; app role assignment works without it. It can't be
   changed after creation, and role-assignable groups can't be dynamic.

   **Make the users group dynamic** so nobody has to maintain it. When
   everyone in the organization may answer signed-in forms, create it as a
   security group with membership type **Dynamic User** and a rule such as:

   ```
   (user.userType -eq "Member") and (user.accountEnabled -eq true)
   ```

   `userType -eq "Member"` keeps B2B guests out (with Assignment required
   set to No instead, guests in the tenant could sign in and would be
   added to the tenant by JIT). Add a clause (e.g. on `companyName` or
   `department`) to also exclude service accounts. Use **Validate Rules**
   on a member, a guest, and a service account before saving. Dynamic
   groups need Entra ID P1, which group assignment to an app already
   requires. App assignment counts direct members only, so an existing
   "all staff" group made of nested groups won't work; a dynamic group
   avoids that.

   An existing assigned group can be switched to Dynamic User (Properties,
   Membership type) if it is cloud-only (not synced from on-premises AD),
   not role-assignable, and not mail-enabled. Its object id, and so the app
   assignment, is kept, but its current members are removed and users
   can't sign in until rule processing repopulates it. In production,
   instead assign a new dynamic group alongside the old one, wait for it to
   populate, then remove the old group's assignment.

5. **Email claim (optional)**: the app uses `email`, falling back to
   `preferred_username` (usually the UPN). It requests the `email` scope,
   but for members of your own tenant Entra only includes the claim when it
   is configured as an optional claim:
   - App registrations, the app, Token configuration, **Add optional
     claim**; token type **ID**; tick **email**; Add.
   - When prompted to "Turn on the Microsoft Graph email permission", tick
     it (the claim needs the delegated Graph `email` permission).
   - API permissions: **Grant admin consent**, so users aren't prompted.

   The claim carries the user's `mail` attribute (synced from on-premises
   AD in hybrid tenants). Users with no `mail` value still get the UPN. If
   UPNs already equal mailbox addresses, this step changes nothing visible.
   To verify, sign in and check the email shown in the sidebar.

6. **Microsoft Graph permissions**: API permissions, Add a permission,
   Microsoft Graph, **Application permissions** (not Delegated), then
   **Grant admin consent**. All three are read-only:

   | Permission | Used for |
   |------------|----------|
   | `User.Read.All` | user search (Add User, Manage Access) and reading users |
   | `Application.Read.All` | reading this app's role assignments (users page) |
   | `GroupMember.Read.All` | resolving roles assigned through groups (users page) |

   The type must be **Application**: the app calls Graph as itself
   (client credentials), and a *Delegated* `User.Read.All` is ignored for
   those calls even when it shows "Granted". That mistake shows up as
   `Entra search failed: 403 Authorization_RequestDenied` in the log and
   an error in the user picker. Enterprise applications, the app,
   **Permissions** lists what was actually consented. After granting, the
   app's cached Graph token can take up to an hour to pick up the change
   (or restart the service).

   A new registration also lists the delegated `User.Read`, and step 5
   adds the delegated `email`; both are needed for sign-in. A delegated
   `User.Read.All` is not used and can be removed.

   **Users and roles are managed in Entra, not in the app.** The users
   page shows each member's roles, whether assigned directly or through a
   group (direct group members only, as Entra does), and adds existing
   directory users to the tenant, but with this provider it offers no role
   editing and no "Create New" user. Without `Application.Read.All` and
   `GroupMember.Read.All`, roles show as "Unknown" with a warning; sign-in
   and authorization are unaffected, because roles come from the token's
   `roles` claim, which includes group assignments. A user picks up a role
   change at their next sign-in.

   **Do not grant**:
   - `User.ReadWrite.All` lets the app create, edit, and **delete any user
     in the tenant**. In hybrid/synced tenants users come from on-premises
     AD, so creating them from the app is wrong anyway (the user may fail
     to sync or be overwritten). The app never creates Entra users.
   - `AppRoleAssignment.ReadWrite.All` is not limited to this app: whoever
     holds the client secret can grant any app, including this one, further
     Graph permissions, a path to tenant-wide privilege. The app no longer
     writes role assignments, so nothing needs it.

   To take away a permission already granted: API permissions, the
   permission's **...** menu, **Revoke admin consent**, then **Remove
   permission** (removing alone can leave the consent in place).

### Integration API (M2M) with Entra

For n8n or other services calling `/api/integration/*`:

1. In this app registration, Expose an API: set the Application ID URI
   (e.g. `api://<client-id>`), and create an app role with value
   `forms.integration`, allowed member type **Applications**.
2. Register a separate app for the caller, add an API permission to this
   app's `forms.integration` role (Application permission), grant admin
   consent, and create a secret for it.
3. Configure: `M2M_AUDIENCE=api://<client-id>` and
   `M2M_CLIENT_TENANTS={"<caller-client-id>":["<tenant-uuid>"]}`.
   Entra client-credentials tokens carry the caller's id in `azp` (v2) or
   `appid` (v1) and the role in `roles`; both are handled.
4. Token version: unless the API app's manifest sets
   `accessTokenAcceptedVersion: 2`, Entra issues v1 tokens with issuer
   `https://sts.windows.net/<tenant-id>/`; set `M2M_ISSUER` to that value.

### Notes

- Sign-out is local only (the Entra session is kept for SSO). "Not you?" /
  Switch Account sends `prompt=select_account`, so Entra shows its account
  picker instead of signing the same user back in.
- User creation from the app creates a cloud-only user; with on-premises AD
  sync it may fail or be overwritten.
- Verified (2026-09-28) end to end against a mock OIDC server issuing
  Entra-shaped tokens (`oid`, `preferred_username`, `roles`, pairwise
  `sub`), including JIT membership and the respondent redirect. Not yet
  verified against a live Entra tenant; do that in the dev deployment.

---

## Provider: Auth0

**Implementation:** `src/lib/directory/auth0.ts`

### Environment Variables

```env
AUTH_OIDC_ISSUER=https://your-tenant.auth0.com
AUTH_OIDC_CLIENT_ID=<auth0-app-client-id>
AUTH_OIDC_CLIENT_SECRET=<auth0-app-client-secret>
DIRECTORY_PROVIDER=auth0
AUTH0_MGMT_CLIENT_ID=<auth0-m2m-client-id>
AUTH0_MGMT_CLIENT_SECRET=<auth0-m2m-client-secret>
```

### Auth0 Setup Steps

1. **Create a Regular Web Application (for OIDC login)**
   - Auth0 Dashboard → Applications → Create Application → Regular Web App
   - Allowed Callback URLs: `https://forms.domain.com/api/auth/callback/auth-engine`
   - Allowed Logout URLs: `https://forms.domain.com`
   - Copy the Client ID and Client Secret

2. **Create a Machine-to-Machine Application (for directory API)**
   - Applications → Create Application → Machine to Machine
   - Select the Auth0 Management API as the API
   - Authorize these scopes:
     - `read:users`, `create:users`, `update:users`, `delete:users`
     - `read:roles`, `create:role_members`, `delete:role_members`
   - Copy the Client ID and Client Secret (these are different from
     the OIDC app — store as `AUTH0_MGMT_CLIENT_ID`/`SECRET`)

3. **Create Roles**
   - User Management → Roles → Create Role for each:
     - Name: `security_admin`, Description: Can manage tenant users
     - Name: `admin`, Description: Full access to forms and submissions
     - Name: `creator`, Description: Can create new forms

4. **Create a Post-Login Action to Include Roles in Token**
   - Actions → Flows → Login → Add Action → Build Custom
   - Name: `Add Roles to Token`
   - Code:
     ```js
     exports.onExecutePostLogin = async (event, api) => {
       const roles = event.authorization?.roles ?? []
       api.idToken.setCustomClaim('roles', roles)
     }
     ```
   - Deploy and drag into the Login flow

5. **Assign Users to Roles**
   - User Management → Users → Select user → Roles → Assign Roles

### How Roles Work

Auth0 has native Roles, but doesn't include them in the ID token by
default. The post-login Action (step 4) adds a `roles` claim to the
token. The provider also checks for `https://forms.domain.com/roles`
as a fallback (a common Auth0 namespacing convention).

The directory provider uses separate M2M credentials to call the Auth0
Management API for user/role CRUD operations.

### Notes

- Auth0 has rate limits on the Management API (varies by plan). The
  forms engine caches the M2M token but makes individual API calls per
  operation. High-volume user management may hit limits.
- The `connection` parameter in user creation defaults to
  `Username-Password-Authentication`. If your Auth0 tenant uses a
  different database connection name, update `auth0.ts`.

---

## Adding a New Provider

1. **Create the implementation file**: `src/lib/directory/{name}.ts`
   - Export a const that implements `DirectoryProvider` (see `types.ts`)
   - Handle authentication to the provider's management API

2. **Register it**
   - `src/lib/directory/oidc-config.ts`: add to the `ProviderName` union and
     add a `case` in `getOidcRoleConfig()` (role claims, scopes, user-id claim)
   - `src/lib/directory/index.ts`: add a `case` in `getProvider()`

3. **Choose the user-id claim deliberately** (`userIdClaim`). It must be
   the id the provider's directory API returns, because memberships,
   grants, and submitter identity are keyed on it. Do not assume `sub`:
   - Some IdPs issue a **pairwise** `sub` (different per application), so
     it never matches the directory's user id. Entra is one; others can be
     configured that way.
   - Check what the directory API calls a user's id and which token claim
     carries the same value, and verify it with a real token.
   - Changing this later detaches every existing user from their
     memberships and grants (a data migration), so get it right before
     go-live.

4. **Document**: Add a section to this file with:
   - Required env vars
   - Role and user-id claim configuration
   - Any provider-specific setup steps

5. **Test**: add the provider's claim shape to
   `src/lib/__tests__/auth-claims.test.ts` (including a pairwise `sub`
   that differs from the directory id, if applicable). The rest of the
   suite mocks the directory module.

> **Future: configurable claims mapping.** Role and user-id claims are
> currently hard-coded per provider in `oidc-config.ts`. The intended end
> state is configuration (user-id claim, role claim, role-value mapping,
> email/name claims) so a new or unusual IdP needs no code change. This is
> planned, not built.

---

## Role Mapping Reference

All providers must map to these form engine role names:

| Role | Purpose |
|------|---------|
| `security_admin` | Manage tenant users and their roles |
| `admin` | Full access to all forms, submissions, campaigns |
| `creator` | Create new forms (becomes owner of forms they create) |

These names must match exactly. How they're stored in the IdP varies:
- **Auth Engine**: App-scoped roles in `AppRoles` table
- **Okta**: Groups named `forms-engine:security_admin`, etc.
- **Entra**: App Roles in the Azure AD app registration
- **Auth0**: Auth0 Roles assigned to users, added to token via Action
