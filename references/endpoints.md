# Authaz endpoints & SDK surface

Use the SDKs — they hide URL differences between the JS and .NET clients. This document points at *what* operations exist, not *where* each endpoint lives.

## Hosted defaults

- **Identity domain** (where end users sign in / the OAuth flow): `https://auth.authaz.io`
- **Management API** (admin from your backend): `https://api.authaz.io`

A customer can override either via custom domain. The .NET SDK's `BaseAddress` defaults to `https://api.authaz.io`, matching the hosted product. The JS SDK's baked-in `apiDomain` default is the older `https://api.authaz.com` — pass `apiDomain: "https://api.authaz.io"` explicitly rather than relying on it.

## OAuth 2.1 + OIDC flow (at the identity domain)

The OAuth path is handled for you by the framework SDKs (`@authaz/next`, `@authaz/hono`, `@authaz/react`) and by standard ASP.NET OpenID Connect middleware on .NET. You don't construct these URLs by hand. Common operations the SDK exposes through `/api/auth/*` (Next.js / Hono):

| Path (mounted by the SDK) | Purpose |
|---|---|
| `GET /api/auth/login` | Start the OAuth flow (redirects to Authaz) |
| `POST /api/auth/callback` | Receive the auth code from the browser-side callback page |
| `POST /api/auth/logout` | End the session — POST-only, same CSRF protection as the callback endpoint; a `GET` gets `405` |
| `GET /api/auth/me` | Read the current user (401 if signed out) |
| `POST /api/auth/refresh` | Refresh the access token |

PKCE (S256) is **required** on the authorization-code grant.

## Management API — sub-clients

Authaz operations are organized into typed sub-clients. The exact URL paths differ between JS and .NET SDKs; call the SDK methods instead of inventing HTTP.

### JS — `createAuthazClient` from `@authaz/sdk`

```ts
authaz.auth          // login URL, logout URL, code exchange, token refresh
authaz.users         // CRUD users
authaz.roles         // role definitions, permissions
authaz.tenants       // multi-tenancy
authaz.invitations
authaz.policies      // MFA, password, session, lockout
authaz.authz         // permission/role checks, relationship grants
authaz.applications  // CRUD applications, branding, custom domains
authaz.apiKeys
authaz.appApiKeys
authaz.trailLogs     // audit / activity log
authaz.m2mCertificates
```

### .NET — `IAuthazClient` from `Authaz.Sdk`

```csharp
authaz.Users           // CRUD, sessions, role assignments
authaz.Applications    // CRUD, branding, custom domains
authaz.Tenants
authaz.Authorization   // .Roles, .Permissions (with CheckAsync), .Policies
authaz.Invitations
authaz.Email           // providers, templates
authaz.Auth            // API keys, OAuth credentials, M2M, recovery
authaz.Audit           // trail logs
```

The lists are not 1:1 by name — they overlap conceptually but the JS SDK groups some things differently (e.g., `authaz.authz` for permission checks; .NET has `authaz.Authorization.Permissions`).

## Result types

| SDK | Type | Shape |
|---|---|---|
| JS | `Result<T>` | `{ ok: true, value: T }` or `{ ok: false, error: AuthazError }`. Use `isOk(result)` / `isErr(result)` |
| .NET | `AuthazResult<T>` | `result.IsSuccess` + `result.Value` / `result.Error`. **Not `IsError`** |

Neither SDK throws on logical errors (404, 403, validation). They throw only on transport-level exceptions (network, timeout).

## Standard claims in the access token

| Claim | Type | Always present? | Notes |
|---|---|---|---|
| `sub` | string | yes | The user's Authaz user ID |
| `email` | string | yes (when known) | The user's primary verified email |
| `tenant_id` | string | multi-tenant only | The tenant the session is bound to |
| `roles` | string[] | when assigned | Role names in the current tenant scope (snapshot — do not use for authoritative auth) |
| `aud` | string | yes | The application's client ID |
| `iss` | string | yes | The identity domain |
| `exp` | number | yes | Unix timestamp |

`tenant_id` is bound at login and immutable for the token's lifetime. Tenant switching = new login.

## SDK packages

| Stack | Package | Source of truth |
|---|---|---|
| Next.js | `@authaz/next` | `authaz-sdk-js/packages/next/src/` |
| Hono | `@authaz/hono` | `authaz-sdk-js/packages/hono/src/` |
| React SPA | `@authaz/react` | `authaz-sdk-js/packages/react/src/` |
| Browser/Node core | `@authaz/sdk` | `authaz-sdk-js/packages/core/src/` |
| .NET | `Authaz.Sdk` (NuGet) | `authaz-sdk-dotnet/Authaz.Sdk/src/` |

## Env vars the JS SDK reads (Next.js & Hono examples)

| Var | Required? |
|---|---|
| `AUTHAZ_CLIENT_ID` | yes |
| `AUTHAZ_CLIENT_SECRET` | yes |
| `AUTHAZ_TENANT_ID` | optional (`tenantId?: string` in `AuthazConfig`) — needed to scope OAuth flows; defaults to no tenant for single-tenant apps |

The .NET SDK reads from `IConfiguration` — section name is up to you, but typically `Authaz:BaseUrl` + `Authaz:ApiKey`.
