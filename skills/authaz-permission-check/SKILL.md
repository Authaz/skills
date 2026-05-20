---
name: authaz-permission-check
description: Use when a route or operation needs an authorization check — not just "is the user signed in" but "is this user allowed to do X". Calls Authaz's `/api/v1/authz/check` (single) or `/api/v1/authz/check-bulk`. Always passes `tenant_id` in multi-tenant. Triggers on "check permission", "is allowed", "authz check", "RBAC", "Zeratul check".
---

# Check a permission

Authaz separates **authentication** ("is this a real user") from **authorization** ("is this user allowed to do X"). Authentication lives in the JWT — Authaz signs it, your app verifies the signature. Authorization is a runtime call to Authaz, because role assignments change without re-issuing tokens.

## Step 1 — Pick the permission name

Permissions are `resource:action`. Examples:

- `users:read`, `users:write`, `users:delete`
- `invoices:create`, `invoices:read`, `invoices:approve`
- `settings:update`

Decide the names up-front; they're cheap to add but expensive to rename. Keep the resource singular when possible (`invoice:approve`) for consistency, or plural — just pick one and stay with it.

If the permission doesn't exist yet, define it in the Dashboard (Authorization → Permissions) or via:

```http
POST https://api.rorix.io/api/v1/permissions
X-API-Key: sk_live_…

{ "name": "invoices:approve", "description": "Approve a posted invoice" }
```

Then attach it to a role:

```http
POST https://api.rorix.io/api/v1/roles/{roleId}/permissions
X-API-Key: sk_live_…

{ "permissions": ["invoices:approve"] }
```

## Step 2 — Call the check from your backend

The check is a server-side call. Don't do it from the browser — the API key would leak.

### Single check

```http
POST https://api.rorix.io/api/v1/authz/check
X-API-Key: sk_live_…   (or Authorization: Bearer <user access token>)
Content-Type: application/json

{
  "user_id": "user_01abc…",
  "permission": "invoices:approve",
  "tenant_id": "tenant_01xyz…"
}
```

Response:

```json
{ "allowed": true, "reason": "role:approver in tenant_01xyz" }
```

`tenant_id` is **required for multi-tenant apps**. Omitting it falls back to a global scope and almost always returns `false` — or worse, returns `true` for a permission the user has in another tenant.

### Bulk check (preferred when you'll check multiple in one request)

```http
POST https://api.rorix.io/api/v1/authz/check-bulk
X-API-Key: sk_live_…

{
  "user_id": "user_01abc…",
  "tenant_id": "tenant_01xyz…",
  "permissions": ["invoices:read", "invoices:approve", "invoices:delete"]
}
```

Response:

```json
{ "results": [
  { "permission": "invoices:read", "allowed": true },
  { "permission": "invoices:approve", "allowed": true },
  { "permission": "invoices:delete", "allowed": false }
] }
```

Cheaper than three single calls and the latency is sub-millisecond on the Authaz side.

## Step 3 — Wire it into your code

### Hono / Node

```ts
async function canApprove(userId: string, tenantId: string) {
  const r = await fetch("https://api.rorix.io/api/v1/authz/check", {
    method: "POST",
    headers: {
      "X-API-Key": process.env.AUTHAZ_API_KEY!,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      user_id: userId, permission: "invoices:approve", tenant_id: tenantId,
    }),
  }).then(r => r.json() as Promise<{ allowed: boolean }>);
  return r.allowed;
}
```

Always pull `userId` from `claims.sub` and `tenantId` from `claims.tenant_id` — not from request bodies, query strings, or headers.

### .NET (`Authaz.Sdk`)

```csharp
var check = await _authaz.PermissionCheck.CheckAsync(new()
{
    UserId = User.FindFirst("sub")!.Value,
    Permission = "invoices:approve",
    TenantId = User.FindFirst("tenant_id")?.Value,
});

if (check.IsError || !check.Value.Allowed)
    return ResultsExt.Forbidden(); // or Results.Forbid() in vanilla apps
```

The SDK returns `AuthazResult<T>` — check `IsError` before reading `Value`.

### Caching

The check is fast but not free. For hot paths, cache the result for the lifetime of the request (request-scoped) — never longer. Caching across requests means a role revocation takes time to propagate, which is usually a security bug.

## Step 4 — Verify

1. Assign the user the role that has the permission. Call the check → `allowed: true`.
2. Revoke the role assignment (Dashboard → User → Remove role). Call the check → `allowed: false`. **Within seconds**, not next-token.
3. In multi-tenant: assign in tenant A, check in tenant B — should be `false`. This proves you're passing `tenant_id` correctly.

## Anti-patterns

- **Don't read `roles` from the JWT and skip the check.** Roles in the JWT are a snapshot at login time. Authoritative authorization is the API call.
- **Don't put permission names in code as magic strings everywhere.** Put them in one constants module so renames are a single edit.
- **Don't omit `tenant_id` in multi-tenant.** Every. Time.
- **Don't use the user's own access token to call `/authz/check`** unless your security model has been reviewed for the implications — typically use the application's API key.
- **Don't wrap the check in `try/catch` and default to `allowed: true` on error.** Default to `false` — fail closed.

## References

- Authorization overview: <https://authaz.io/docs/authorization/overview>
- `references/endpoints.md` — `/api/v1/authz/check` and bulk variant
- `references/error-codes.md` — 403 `cross_tenant_access` etc.
- `authaz-multi-tenant` — tenant resolution and isolation
