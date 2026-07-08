---
name: authaz-permission-check
description: Use when a route or operation needs an authorization check — not just "is the user signed in" but "is this user allowed to do X". Calls the SDK's authz check method (`authaz.authz.check` in JS, `authaz.Authorization.Permissions.CheckAsync` in .NET). Always passes `tenantId` in multi-tenant. Triggers on "check permission", "is allowed", "authz check", "RBAC".
---

# Check a permission

Authaz separates **authentication** (the JWT) from **authorization** (a runtime call). Authentication lives in the JWT — Authaz signs it, your app verifies the signature. Authorization is a runtime call to Authaz because role assignments change without re-issuing tokens.

Use the SDK. **Do not call the authz endpoint with raw HTTP** — the JS SDK and the .NET SDK use different paths (`/api/v1/authorization/check` vs `/api/v1/authorization/explain`), and the request shapes differ.

## Step 1 — Pick the permission name

Authaz's authz model uses `resource` + `action`. Examples:

- `resource: "users", action: "read"`
- `resource: "invoices", action: "create"`
- `resource: "settings", action: "update"`

The JS SDK splits them into separate fields. The .NET SDK takes a colon-joined `permission` string (e.g., `"invoices:approve"`). Same operation, different surface shape.

Keep the resource singular when possible (`invoice:approve`) for consistency, or plural — just pick one and stay with it.

## Step 2 — Set up the SDK

If you already have `Authaz.Sdk` / `@authaz/sdk` wired up via `authaz-management-api`, skip this step.

### .NET

```csharp
builder.Services.AddAuthazSdk(opts =>
{
    opts.BaseAddress = new Uri("https://api.authaz.io");
    opts.ApiKey      = builder.Configuration["Authaz:ApiKey"];
});
```

### JavaScript

```ts
import { createAuthazClient } from "@authaz/sdk";

const authaz = createAuthazClient({
  clientId: process.env.AUTHAZ_CLIENT_ID!,
  clientSecret: process.env.AUTHAZ_CLIENT_SECRET!,
  apiKey: process.env.AUTHAZ_API_KEY, // optional; falls back to clientSecret
});
```

## Step 3 — Call check from the backend

The check is server-side. **Never** call it from the browser — the API key would leak.

### JS — single check

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

### JS — bulk check (preferred when checking multiple at once)

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

Cheaper than three single calls — one round-trip instead of three.

### JS — convenience helper for token-mode

If you have the user's access token (e.g., from the OIDC session), the SDK exposes a `can()` shorthand:

```ts
const allowed: boolean = await authaz.authz.can(
  accessToken,
  "invoices",
  "approve",
  tenantId
);
```

This wraps `check` and returns the boolean directly (errors collapse to `false`). Use it in hot paths where you don't need the trace.

### .NET — single check

```csharp
var check = await authaz.Authorization.Permissions.CheckAsync(
    new CheckPermissionRequest(
        UserId:     User.FindFirst("sub")!.Value,
        Permission: "invoices:approve",
        TenantId:   User.FindFirst("tenant_id")?.Value));

if (!check.IsSuccess || !check.Value!.Allowed)
    return Results.Forbid();
```

The .NET response shape includes a full audit trace (`RoleTrace`, `PolicyTrace`) — useful for debugging "why isn't this allowed".

## Step 4 — Always pull subject + tenant from the token

```ts
// Get subject/tenant from token claims, NEVER from request input
const userId  = decodedToken.sub;
const tenant  = decodedToken.tenant_id;
const allowed = await authaz.authz.can(accessToken, "invoices", "approve", tenant);
```

```csharp
var userId   = User.FindFirst("sub")!.Value;
var tenantId = User.FindFirst("tenant_id")?.Value;
```

Treat the token as the sole source of truth for both.

## Step 5 — Caching

The check is fast but not free. For hot paths, cache the result for the **lifetime of the request** (request-scoped) — never longer. Caching across requests means a role revocation takes time to propagate, which is usually a security bug.

## Step 6 — Verify

1. Assign the user the role that has the permission. Call the check → `allowed: true`.
2. Revoke the role assignment (Dashboard → User → Remove role). Call the check → `allowed: false` **within seconds**, not next-token.
3. In multi-tenant: assign in tenant A, check in tenant B — should be `false`. This proves you're passing `tenantId` correctly.

## Anti-patterns

- **Don't read `roles` from the JWT and skip the check.** Roles in the JWT are a snapshot at login time. Authoritative authorization is the SDK call.
- **Don't put permission names in code as scattered magic strings.** Centralize them so renames are a single edit.
- **Don't omit `tenantId` in multi-tenant.** Every. Time.
- **Don't catch SDK errors and default to `allowed: true`.** Default to `false` — fail closed.
- **Don't call the authz endpoint with raw HTTP.** The JS and .NET SDKs disagree on the path; use SDK methods.

## Source of truth

- JS SDK authz: `authaz-sdk-js/packages/core/src/authz/permissions.ts`
- JS authz types: `authaz-sdk-js/packages/core/src/authz/types.ts`
- .NET SDK authz: `authaz-sdk-dotnet/Authaz.Sdk/src/Resources/Authorization/Permissions/`

## References

- `authaz-multi-tenant` — tenant resolution and isolation
- `authaz-management-api` — wider SDK surface
