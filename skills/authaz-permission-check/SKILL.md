---
name: authaz-permission-check
description: Use when a route or operation needs an authorization check — not just "is the user signed in" but "is this user allowed to do X". Calls the SDK's authz check method (`authaz.authz.check` in JS, `authaz.Authorization.Permissions.CheckAsync` in .NET). Always passes `tenantId` in multi-tenant. Triggers on "check permission", "is allowed", "authz check", "RBAC".
---

# Check a permission

- Authaz splits **authentication** (JWT, signed by Authaz, verified by your app) from **authorization** (a runtime call). Role assignments change without re-issuing tokens, so authorization must be a live call, not a JWT read.
- **Use the SDK, never raw HTTP.** JS and .NET SDKs hit different paths (`/api/v1/authorization/check` vs `/api/v1/authorization/explain`) with different request shapes.

## 1. Pick the permission name

- Model = `resource` + `action`. Examples: `resource:"users", action:"read"` · `resource:"invoices", action:"create"` · `resource:"settings", action:"update"`.
- JS SDK: separate `resource`/`action` fields. .NET SDK: colon-joined `permission` string (e.g. `"invoices:approve"`) — same operation, different shape.
- Resource singular vs plural (`invoice:approve` vs `invoices:approve`) — either works, just stay consistent.

## 2. Set up the SDK

Skip if `Authaz.Sdk` / `@authaz/sdk` already wired via `authaz-management-api`.

**.NET:**
```csharp
builder.Services.AddAuthazSdk(opts =>
{
    opts.BaseAddress = new Uri("https://api.authaz.io");
    opts.ApiKey      = builder.Configuration["Authaz:ApiKey"];
});
```

**JavaScript:**
```ts
import { createAuthazClient } from "@authaz/sdk";

const authaz = createAuthazClient({
  clientId: process.env.AUTHAZ_CLIENT_ID!,
  clientSecret: process.env.AUTHAZ_CLIENT_SECRET!,
  apiKey: process.env.AUTHAZ_API_KEY, // optional; falls back to clientSecret
});
```

## 3. Call check server-side

**Never call from the browser** — the API key would leak.

**JS — single check:**
```ts
const result = await authaz.authz.check({
  userId: "user_01abc...",       // or `token: accessToken`
  resource: "invoices",
  action: "approve",
  tenantId: "ten_01xyz...",      // required for multi-tenant
});

// result is Result<CheckResult>
if (result.ok && result.data.allowed) {
  // permitted
}
```

`CheckResult` shape:
```ts
type CheckResult = {
  allowed: boolean;
  viaDirectGrant?: boolean;
  viaRole?: boolean;
};
```

**JS — bulk check** (preferred for multiple checks at once — one round-trip instead of several):
```ts
const result = await authaz.authz.checkBulk({
  userId: "user_01abc...",
  tenantId: "ten_01xyz...",
  checks: [
    { permission: "invoices:read", action: "read" },
    { permission: "invoices:approve", action: "approve" },
    { permission: "invoices:delete", action: "delete" },
  ],
});

// result.data.results: { permission: string; allowed: boolean }[]
```

**JS — `can()` convenience helper** (token-mode): wraps `check`, returns boolean directly, errors collapse to `false`. Use in hot paths where the trace isn't needed.
```ts
const allowed: boolean = await authaz.authz.can(
  accessToken,
  "invoices",
  "approve",
  tenantId
);
```

**.NET — single check:**
```csharp
var check = await authaz.Authorization.Permissions.CheckAsync(
    new CheckPermissionRequest(
        UserId:     User.FindFirst("sub")!.Value,
        Permission: "invoices:approve",
        TenantId:   User.FindFirst("tenant_id")?.Value));

if (!check.IsSuccess || !check.Value!.Allowed)
    return Results.Forbid();
```
.NET response includes a full audit trace (`RoleTrace`, `PolicyTrace`) — useful for debugging "why isn't this allowed".

## 4. Always pull subject + tenant from the token (never from request input)

```ts
const userId  = decodedToken.sub;
const tenant  = decodedToken.tenant_id;
const allowed = await authaz.authz.can(accessToken, "invoices", "approve", tenant);
```
```csharp
var userId   = User.FindFirst("sub")!.Value;
var tenantId = User.FindFirst("tenant_id")?.Value;
```
Token is the sole source of truth for both.

## 5. Caching

Check is fast but not free. For hot paths, cache **request-scoped only** (lifetime of the request) — never longer. Cross-request caching delays role-revocation propagation, which is usually a security bug.

## 6. Verify

1. Assign the user the role with the permission → check → `allowed: true`.
2. Revoke the role (Dashboard → User → Remove role) → check → `allowed: false` **within seconds**, not next-token.
3. Multi-tenant: assign in tenant A, check in tenant B → should be `false`. Proves `tenantId` is passed correctly.

## Anti-patterns

| Don't | Why |
|---|---|
| Read `roles` from the JWT and skip the check | JWT roles are a login-time snapshot; the SDK call is authoritative |
| Scatter permission names as magic strings in code | Centralize so renames are a single edit |
| Omit `tenantId` in multi-tenant | Every. Time. |
| Catch SDK errors and default to `allowed: true` | Default to `false` — fail closed |
| Call the authz endpoint with raw HTTP | JS/.NET SDKs disagree on path; use SDK methods |

## Source of truth

- JS SDK authz: `authaz-sdk-js/packages/core/src/authz/permissions.ts`
- JS authz types: `authaz-sdk-js/packages/core/src/authz/types.ts`
- .NET SDK authz: `authaz-sdk-dotnet/Authaz.Sdk/src/Resources/Authorization/Permissions/`

## References

- `authaz-multi-tenant` — tenant resolution and isolation
- `authaz-management-api` — wider SDK surface
