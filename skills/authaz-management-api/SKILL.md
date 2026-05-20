---
name: authaz-management-api
description: Use when calling Authaz's Management API from a backend — creating users, assigning roles, listing tenants, sending invitations, querying audit logs, managing API keys, or other admin operations. Prefers `Authaz.Sdk` (.NET) and `createAuthazClient` from `@authaz/sdk` (JS). Triggers on "management API", "Authaz.Sdk", "create user via API", "assign role", "invite user", "audit log".
---

# Use the Authaz Management API

The Management API is at `https://api.authaz.io` for the hosted product. It is **not** the same surface as the OAuth flow at `https://auth.authaz.io`. Different hosts, different auth (`X-API-Key` for management; cookies/JWTs for end-user flows), different audiences.

Always use the SDK — `Authaz.Sdk` for .NET, `createAuthazClient` from `@authaz/sdk` for JS. The endpoint paths differ between SDKs (the .NET SDK uses `/api/v1/...`, the JS SDK uses `/v1/...`), so raw HTTP requires checking the actual SDK source first. The SDKs hide this divergence.

## Step 1 — Issue an API key

Dashboard → API Keys → **Create**. Grant only the scopes you actually need. The full key is shown **once** — copy it into your secret manager immediately.

Scope keys to the minimum permissions and issue separate keys per service so rotations are independent.

## Step 2 — Install and configure

### .NET — `Authaz.Sdk`

```bash
dotnet add package Authaz.Sdk
```

`appsettings.json`:

```json
{
  "Authaz": {
    "BaseUrl": "https://api.authaz.io",
    "ApiKey": ""
  }
}
```

For development, use user secrets instead of committing `ApiKey`:

```bash
dotnet user-secrets init
dotnet user-secrets set "Authaz:ApiKey" "authaz_..."
```

`Program.cs`:

```csharp
using Authaz.Sdk;

builder.Services.AddAuthazSdk(opts =>
{
    var section = builder.Configuration.GetSection("Authaz");
    opts.BaseAddress = new Uri(section["BaseUrl"] ?? "https://api.authaz.io");
    opts.ApiKey      = section["ApiKey"];
});
```

Then inject `IAuthazClient`:

```csharp
public class UsersService(IAuthazClient authaz)
{
    public async Task<string?> GetEmailAsync(Guid userId, CancellationToken ct = default)
    {
        var result = await authaz.Users.GetAsync(userId, ct);
        if (!result.IsSuccess) return null;
        return result.Value!.Email;
    }
}
```

**Result-type contract** — every SDK call returns `AuthazResult<T>`:

- `result.IsSuccess` (bool)
- `result.Value` (T?) — populated on success
- `result.Error` (AuthazError?) — populated on failure

The SDK does **not throw** on logical errors (404, 403, validation). Wrap in `try/catch` only for transport-level exceptions (`HttpRequestException`).

Typed error subtypes:

```csharp
static IResult ToError(AuthazError error) => error switch
{
    AuthazError.NotFound e     => Results.NotFound(new { error = "not_found", message = e.Message }),
    AuthazError.Unauthorized   => Results.Unauthorized(),
    AuthazError.Forbidden      => Results.Forbid(),
    AuthazError.Validation e   => Results.ValidationProblem(
                                      e.Fields.ToDictionary(f => f.Field, f => new[] { f.Message })),
    AuthazError.RateLimited e  => Results.Problem(detail: e.Message, statusCode: 429),
    _                           => Results.Problem(detail: error.Message, statusCode: 500),
};
```

Sub-clients on `IAuthazClient`:

- `Users` — get, list, suspend, activate, sessions, role assignments
- `Applications` — list, manage applications, branding, custom domains
- `Tenants` — list, create, update; per-tenant OAuth config
- `Authorization` — `.Roles`, `.Permissions` (with `CheckAsync`), `.Policies`
- `Invitations` — send, list, pending count
- `Email` — providers, templates
- `Auth` — API keys, OAuth credentials, M2M, account recovery
- `Audit` — trail logs

See `authaz-sdk-dotnet/Authaz.Sdk/src/Resources/` for the full surface.

### JavaScript — `@authaz/sdk`

```bash
pnpm add @authaz/sdk
```

```ts
import { createAuthazClient, isOk } from "@authaz/sdk";

const authaz = createAuthazClient({
  clientId: process.env.AUTHAZ_CLIENT_ID!,
  clientSecret: process.env.AUTHAZ_CLIENT_SECRET!,
  organizationId: process.env.AUTHAZ_ORGANIZATION_ID!,
  apiKey: process.env.AUTHAZ_API_KEY, // optional; falls back to clientSecret
  // apiDomain / authazDomain default to https://api.authaz.io / https://auth.authaz.io
});

const result = await authaz.users.list({ pageSize: 20 });
if (isOk(result)) {
  console.log(result.value); // typed list response
} else {
  console.error(result.error.code, result.error.message);
}
```

Sub-clients on the JS `AuthazClient`:

- `auth` — login URL, logout URL, code exchange, token refresh
- `users`, `roles`, `tenants`, `invitations`, `policies`
- `authz` — `check`, `checkRole`, `checkBulk`, relationship grants
- `applications`, `apiKeys`, `appApiKeys`
- `trailLogs`, `m2mCertificates`

**Result type**: every call returns `Result<T>`. Use `isOk(result)` / `isErr(result)` from the SDK. The SDK does not throw on logical errors.

## Step 3 — Common operations (use the SDK; don't paraphrase HTTP)

### Create / list users

```ts
// JS
const created = await authaz.users.create({ email: "alice@example.com", name: "Alice" });
const list    = await authaz.users.list({ pageSize: 50 });
```

```csharp
// .NET
var created = await authaz.Users.CreateAsync(new CreateUserRequest("alice@example.com", "Alice"));
var list    = await authaz.Users.ListAsync(pageSize: 50);
```

### Invite into a tenant

```ts
const invite = await authaz.invitations.send({
  email: "alice@example.com",
  tenantId: "ten_...",
  roleIds: ["role_..."],
});
```

```csharp
var invite = await authaz.Invitations.SendAsync(new SendInvitationRequest(/* ... */));
```

### Check a permission

```ts
const check = await authaz.authz.check({
  userId: "user_...",            // or `token: "..."`
  resource: "invoices",
  action: "approve",
  tenantId: "ten_...",
});
// check.value.allowed: boolean
```

```csharp
var check = await authaz.Authorization.Permissions.CheckAsync(
    new CheckPermissionRequest(userId: "user_...", permission: "invoices:approve", tenantId: "ten_..."));
```

The JS SDK splits `resource` / `action`. The .NET SDK takes a colon-joined `permission` string. Same operation, different shapes.

### Pagination

JS list operations accept `{ pageSize, cursor }` and return `{ items, nextCursor }`. .NET equivalents take `pageSize` / `cursor` parameters and return `ListResponse<T>`. Loop until `nextCursor` is null/empty.

## Step 4 — Verify

For every operation you wire up:

1. **Happy path** — call it; observe the response; assert the fields your code reads.
2. **Permissions** — call with a key that lacks the scope; expect 403 `Forbidden`.
3. **Cross-tenant** — if multi-tenant, call a tenant-scoped endpoint with a key scoped to a different tenant; expect 403.

## Anti-patterns

- **Don't store the API key in the frontend.** Server-only credential.
- **Don't use `result.IsError`** in .NET — it's `result.IsSuccess` (or `result.Error != null`).
- **Don't wrap SDK calls in `try/catch` for control flow.** Use the result type.
- **Don't `Guid.Parse` IDs from API responses without checking the docs.** Many Authaz IDs are prefixed strings (`user_01abc…`), not GUIDs.
- **Don't paraphrase endpoint paths from this skill into raw `curl` calls.** The SDKs disagree on `/api/v1/...` vs `/v1/...`; use SDK methods.
- **Don't share one API key across services.** Per-service keys keep rotation isolated.

## Source of truth

- .NET sample: `authaz-sdk-dotnet/Authaz.Sdk.Sample/Program.cs`
- .NET SDK surface: `authaz-sdk-dotnet/Authaz.Sdk/src/`
- JS SDK entry point: `authaz-sdk-js/packages/core/src/index.ts`
- JS SDK client: `authaz-sdk-js/packages/core/src/client.ts`

## References

- `references/endpoints.md` — high-level catalog of sub-clients
- `references/error-codes.md` — error shape and retry guidance
- `authaz-permission-check` — runtime `authz.check` patterns
- `authaz-multi-tenant` — tenant scoping
