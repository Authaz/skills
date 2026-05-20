---
name: authaz-management-api
description: Use when calling Authaz's Management API — creating users, assigning roles, listing tenants, sending invitations, querying audit logs, managing API keys, or anything else that's not the user-facing OAuth flow. Prefers `Authaz.Sdk` (.NET) and `@authaz/sdk` (JS) over raw HTTP. Triggers on "management API", "Authaz.Sdk", "create user via API", "assign role", "invite user", "audit log".
---

# Use the Authaz Management API

The Management API is at `https://api.rorix.io` (or the customer's API host). It is **not** the same surface as the OAuth flow at `auth.rorix.io`. The Management API is what your backend hits to administer your Authaz tenant — create users, manage roles, audit, branding.

Always authenticate with `X-API-Key: sk_live_…`. Some endpoints accept a Bearer access token instead, but API keys are the supported path for backend integrations.

## Step 1 — Issue an API key

Dashboard → API Keys → Create API Key → grant the scopes you actually need. The full key is shown **once** — copy it into your secret manager immediately.

Or:

```http
POST https://api.rorix.io/api/v1/api-keys
X-API-Key: sk_live_…   (must already have api_keys:write)

{ "name": "ingest-worker", "permissions": ["users:read", "users:write"] }
```

Scope the key to the minimum permissions. Issue separate keys per service so rotations are independent.

## Step 2 — Pick the SDK or raw HTTP

### .NET — `Authaz.Sdk`

```bash
dotnet add package Authaz.Sdk
```

```csharp
builder.Services.AddAuthazSdk();
```

`appsettings.json`:

```json
{ "Authaz": { "ApiBaseUrl": "https://api.rorix.io", "ApiKey": "sk_live_…" } }
```

```csharp
public class UsersService(IAuthazClient authaz)
{
    public async Task<string?> GetEmailAsync(string userId)
    {
        var r = await authaz.Users.GetAsync(userId);
        if (r.IsError) return null;
        return r.Value.Email;
    }
}
```

`IAuthazClient` exposes typed sub-clients: `Users`, `Roles`, `Tenants`, `Applications`, `Organizations`, `Invitations`, `Policies`, `Config`, `Branding`, `CustomDomains`, `ApiKeys`, `AuthFlow`, `Analytics`, `Audit`, `PermissionCheck`, `M2MCredentials`. Every method returns `AuthazResult<T>` — check `IsError` first; **never** wrap in `try/catch` for control flow.

### JavaScript — `@authaz/sdk`

```bash
pnpm add @authaz/sdk
```

```ts
import { createAuthazSdk } from "@authaz/sdk";

const authaz = createAuthazSdk({
  apiBaseUrl: "https://api.rorix.io",
  apiKey: process.env.AUTHAZ_API_KEY!,
});

const user = await authaz.users.get(userId);
```

The JS SDK throws on error (unlike `.NET`'s result type). Wrap calls in `try/catch` only around the boundary of the operation, not around every line.

### Raw HTTP

If you don't want a dependency, the API is plain JSON:

```bash
curl -X POST https://api.rorix.io/api/v1/users \
  -H "X-API-Key: $AUTHAZ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","name":"Alice"}'
```

But — verify response shapes against the actual API before assuming. Opaque tokens (session IDs, codes) are usually strings even when they look like GUIDs. Don't `Guid.Parse` them.

## Step 3 — Common operations

### Create a user

```http
POST /api/v1/users
{ "email": "alice@example.com", "name": "Alice" }
```

Email goes through the verification flow. To create a user *without* sending email (e.g., bulk import), pass `email_verified: true` (requires the `users:write_unverified` scope).

### Invite a user into a tenant

```http
POST /api/v1/tenants/{tenantId}/invitations
{ "email": "alice@example.com", "roles": ["editor"] }
```

The invitee receives an email; clicking lands them in Authaz Sign-In already scoped to that tenant.

### Assign / unassign a role

```http
POST /api/v1/users/{userId}/roles
{ "role_id": "role_01abc…", "tenant_id": "tenant_01xyz…" }

DELETE /api/v1/users/{userId}/roles/{roleId}?tenant_id=tenant_01xyz…
```

`tenant_id` is required for multi-tenant role assignments. Omitting it grants the role *globally* in the application — almost always wrong in B2B SaaS.

### List with pagination

```http
GET /api/v1/users?limit=50&cursor=eyJpZCI6...
```

Response includes `next_cursor`. Don't try to compute offsets; cursors are opaque. Loop until `next_cursor` is null.

### Read audit log

```http
GET /api/v1/audit?actor_id={userId}&start=2026-04-01T00:00:00Z&limit=200
```

Filter by `actor_id`, `resource_type`, `action`, time range. Audit data is append-only and survives soft-deletes.

## Step 4 — Verify

For every operation you wire up:

1. **Happy path** — call it; observe the response shape; assert exactly the fields your code reads.
2. **Permissions** — try the same call with a key that lacks the scope; expect 403 `insufficient_scope` with a useful `error_description`.
3. **Cross-tenant** — in multi-tenant, try calling a tenant-scoped endpoint with a key scoped to a different tenant; expect 403 `cross_tenant_access`.

## Anti-patterns

- **Don't store the API key in the frontend.** It's a server-only credential.
- **Don't `Guid.Parse` IDs from API responses without checking the docs.** Many IDs are string-typed.
- **Don't poll `/api/v1/users` to detect new users.** Hit the audit log filter or use webhooks (when available).
- **Don't ignore `AuthazResult.IsError`** in .NET. The result type exists *because* exceptions hide intent.
- **Don't share one API key across services.** Per-service keys keep rotation isolated.

## References

- Management API overview: <https://authaz.io/docs/management-api/overview>
- Full reference: <https://authaz.io/docs/management-api/reference>
- SDK overview: <https://authaz.io/docs/sdk/overview>
- `references/endpoints.md`, `references/error-codes.md`
