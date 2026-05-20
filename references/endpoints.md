# Authaz endpoint reference

Canonical hostnames and routes for an Authaz integration.

Two URLs you need from the **Authaz Dashboard** before you start:

- **Your identity domain** — where end users sign in (e.g., `https://auth.your-app.com` for a custom domain, or the default subdomain Authaz issued you).
- **The Management API base URL** — `https://api.authaz.io` for the hosted product, or your self-hosted address.

Throughout this file, `auth.example.com` stands in for your identity domain — replace it with the value from the Dashboard.

## Identity domain (Authaz Sign-In + OAuth 2.1)

Example: `https://auth.example.com` (your identity domain, from the Dashboard)

| Purpose | Method | Path |
|---|---|---|
| Authorization endpoint | GET | `/universal/oauth2/authorize` |
| Token endpoint | POST | `/universal/oauth2/token` |
| Provider discovery | GET | `/universal/auth/providers` |
| Per-application JWKS | GET | `/universal/.well-known/jwks/{client_id}.json` |
| OIDC discovery | GET | `/.well-known/openid-configuration` |
| Logout | GET | `/universal/oauth2/logout` |

PKCE (S256) is **required** on the authorization-code grant. Implicit and password grants are not supported.

## Management API

Hosted base URL: `https://api.authaz.io` (self-hosted: substitute your own host).

Authentication: `X-API-Key: sk_live_…` header. (Some endpoints accept a Bearer access token instead — see the docs page.)

| Group | Path prefix | Notes |
|---|---|---|
| Applications | `/api/v1/applications` | Create / list / update / delete |
| Tenants | `/api/v1/tenants` | Create / list; multi-tenant only |
| Tenant invitations | `/api/v1/tenants/{tenantId}/invitations` | Invite a user into a tenant |
| Users | `/api/v1/users` | Read / list / update; create via signup or invite |
| Roles | `/api/v1/roles` | Define and list roles |
| Role assignments | `/api/v1/users/{userId}/roles` | Assign / unassign |
| Permission check | `/api/v1/authz/check` | Single check; pass `tenant_id` in multi-tenant |
| Bulk permission check | `/api/v1/authz/check-bulk` | Multiple permissions in one call |
| API keys | `/api/v1/api-keys` | Issue / revoke |
| M2M credentials | `/api/v1/m2m-credentials` | Client-credentials clients |
| Audit log | `/api/v1/audit` | Read; filter by actor, resource, time |
| Branding | `/api/v1/branding` | Sign-In theme, logo, colors |
| Custom domains | `/api/v1/custom-domains` | DNS-validated identity-domain aliases |

Tenant-scoped variants live under `/api/v1/applications/{appId}/tenants/{tenantId}/{feature}` — never use `?tenantId=` query params.

Full reference: <https://authaz.io/docs/management-api/reference>

## SDKs

| Stack | Package | Notes |
|---|---|---|
| Next.js | `@authaz/next` | Mounts the auth handler at `/api/auth/*`; ships `requireUser`, `withAuth`, middleware |
| Hono | `@authaz/hono` | `createAuthazHandler`, `createAuthMiddleware`, `isAuthenticated` |
| React SPA | `@authaz/react` | `AuthazProvider`, `useUser`, `useRequireUser`, `<SignInButton>` |
| Browser/Node core | `@authaz/sdk` | Lower-level HTTP + token primitives |
| .NET | `Authaz.Sdk` (NuGet) | Typed Management API client + ASP.NET OIDC integration |

The JS SDK reads `AUTHAZ_CLIENT_ID`, `AUTHAZ_CLIENT_SECRET`, `AUTHAZ_IDENTITY_DOMAIN` from env. The .NET SDK reads `Authaz:*` from `IConfiguration`.

## Standard claims in the access token

| Claim | Type | Always present? | Notes |
|---|---|---|---|
| `sub` | string | yes | The user's Authaz user ID |
| `email` | string | yes (when known) | The user's primary verified email |
| `tenant_id` | string | multi-tenant only | The tenant the user is signed in to right now |
| `roles` | string[] | when assigned | Role names in the current tenant scope |
| `aud` | string | yes | The application's client ID |
| `iss` | string | yes | The identity domain |
| `exp` | number | yes | Unix timestamp |

`tenant_id` is bound at login and immutable for the token's lifetime. Tenant switching = new login.
