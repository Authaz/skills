# Use the Authaz Management API

Management API: `https://api.authaz.io` (hosted product) — **not** the OAuth flow at `https://auth.authaz.io`, and **not** the host the `authaz` CLI talks to. Different hosts, auth (`X-API-Key` vs cookies/JWTs), audiences.

## Which host each client uses — don't copy a URL between them

| Client | Host | Who sets it |
|---|---|---|
| `authaz` CLI | `https://dashboard.authaz.io` + `--identity-prefix /api` (the Dashboard BFF, paths under `cli/…`) | CLI default; `--dashboard-url` overrides, self-hosted only |
| `Authaz.Sdk` (.NET) | `https://api.authaz.io` | SDK default — leave it alone |
| `@authaz/sdk` (JS) | `https://api.authaz.io` | **you must pass it** — the built-in default is a dead host, see below |

The CLI never calls `api.authaz.io`, and the SDKs never call `dashboard.authaz.io`. A URL that worked in one is wrong in the other.

Always use the SDK — `Authaz.Sdk` (.NET) or `createAuthazClient` from `@authaz/sdk` (JS); raw HTTP requires checking SDK source.

**Both SDKs use the same `/api/v1/...` prefix.** (Earlier revisions of this skill claimed JS used `/v1/...` — wrong; verified in `authaz-sdk-js/packages/core/src/management/*.ts`, every path literal is `/api/v1/…`.) The `/api` is load-bearing: production routes `api.authaz.io` with `PathPrefix: /api` (`authaz/Authaz/appsettings.json:48`), so `https://api.authaz.io/v1/users` 404s while `/api/v1/users` 401s.

## Step 1 — Issue an API key

Dashboard → API Keys → **Create**. Grant only needed scopes; full key shown **once** — copy to secret manager immediately. Issue separate keys per service so rotations are independent.

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

Dev: use user secrets instead of committing `ApiKey`:

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

Inject `IAuthazClient`:

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

SDK does **not throw** on logical errors (404, 403, validation). Only `try/catch` transport-level exceptions (`HttpRequestException`).

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

Full surface: `authaz-sdk-dotnet/Authaz.Sdk/src/Resources/`.

### JavaScript — `@authaz/sdk`

```bash
pnpm add @authaz/sdk
```

```ts
import { createAuthazClient, isOk } from "@authaz/sdk";

const authaz = createAuthazClient({
  clientId: process.env.AUTHAZ_CLIENT_ID!,
  clientSecret: process.env.AUTHAZ_CLIENT_SECRET!,
  apiKey: process.env.AUTHAZ_API_KEY, // optional; falls back to clientSecret
  // REQUIRED on the hosted product. `authazDomain` only drives OAuth; management calls resolve
  // their base from `apiDomain` (or `baseAddress`), whose built-in default is `https://api.authaz.com`
  // — a host that doesn't serve Authaz at all (TLS handshake fails; the string appears nowhere in the
  // server repo). Omit this and every management call dies at connect time.
  apiDomain: process.env.AUTHAZ_API_DOMAIN ?? "https://api.authaz.io",
});

// Org-management endpoints (users, roles, invitations) authenticate a human
// actor in addition to the API key, so pass the signed-in user's access token.
const result = await authaz.users.list({ pageSize: 20 }, { accessToken });
if (isOk(result)) {
  console.log(result.data); // typed list response
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

**Result type**: every call returns `Result<T>`. Use `isOk(result)` / `isErr(result)`. SDK does not throw on logical errors.

## Step 3 — Common operations (use the SDK; don't paraphrase HTTP)

### Onboard / list users

JS SDK has no `users.create` — onboard with `invitations.send(...)` (below). List existing users:

```ts
// JS — second arg carries the signed-in user's access token (required)
const list = await authaz.users.list({ pageSize: 50 }, { accessToken });
```

```csharp
// .NET
var list = await authaz.Users.ListAsync(pageSize: 50);
```

### Invite into a tenant

```ts
const invite = await authaz.invitations.send(
  {
    email: "alice@example.com",
    tenantId: "ten_...",
    roleId: "role_...",
  },
  { accessToken },
);
```

```csharp
var invite = await authaz.Invitations.SendAsync(new SendInvitationRequest(/* ... */));
```

Omitting `tenantId` doesn't fail — it mints a **tenantless (app-wide) grant**. If the surface sending the invite is scoped to one tenant (e.g. an org's own "team" page), always pass that tenant's id explicitly; never let it default to null just because the form only collects an email and a role.

### `Role.isGlobal` is a catalog flag, not a grant scope

Non-obvious and easy to get backwards: `isGlobal` on a **role definition** (`OwnerTenantId == null`) means the role is *defined app-wide* and available for assignment in any tenant — built-ins like `owner`/`writer`/`reader` are all `isGlobal`. It does **not** mean "assigning this role grants app-wide access." Scope is decided at **grant/invitation time** by whether you pass a `tenantId`:

- `roles.list()` filtered to `isGlobal` → the app-wide role catalog, usable as options in any tenant's invite form.
- The role a tenant itself defines (`role.tenantId === thatTenantId`) is *also* assignable there — the assignable set for a given tenant's invite form is `isGlobal || role.tenantId === tenantId`, not just `isGlobal`.
- The actual access boundary comes from the `tenantId` on the invitation/assignment, not from `isGlobal` on the role.

Don't gate "does this role grant everything" on `role.isGlobal` — that's answering a different question (is it centrally defined) than the one you mean to ask (does the assignment carry a tenant scope).

### List members of one tenant: use `roles.getUsersWithRoles`, not `users.listRoles`

To answer "who belongs to tenant X, and with what roles", call the tenant-scoped roster endpoint and filter server-side by passing `tenantId`:

```ts
const rolesResult = await authaz.roles.getUsersWithRoles({ tenantId }, { accessToken });
// rolesResult.data: { userId, roles: { roleId, roleName }[] }[]
```

Do **not** try to reconstruct this by calling `users.listRoles(userId)` per user and filtering on `roles[].tenantId` — that field is the role's *definition* tenant (null for `isGlobal` built-ins like `owner`), not the *assignment* scope. Filtering on it silently drops every member holding a built-in role, which in practice is most members. `getUsersWithRoles({ tenantId })` filters on the assignment scope correctly; use it for both the member list and for finding which role assignments to unassign when removing a member.

### Remove a member from a tenant

There's no single "remove member" call — unassign every role they hold in that tenant:

```ts
const { data } = await authaz.roles.getUsersWithRoles({ tenantId }, { accessToken });
const theirRoles = data.find(e => e.userId === userId)?.roles ?? [];
for (const role of theirRoles) {
  await authaz.roles.unassign(role.roleId, userId, tenantId, { accessToken });
}
```

This only revokes access in `tenantId` — assignments the user holds in other tenants are untouched, so a shared-pool user removed from one tenant keeps working in the others (see `multi-tenant.md`).

### Check a permission

```ts
const check = await authaz.authz.check({
  userId: "user_...",            // or `token: "..."`
  resource: "invoices",
  action: "approve",
  tenantId: "ten_...",
});
// check.data.allowed: boolean
```

```csharp
var check = await authaz.Authorization.Permissions.CheckAsync(
    new CheckPermissionRequest(userId: "user_...", permission: "invoices:approve", tenantId: "ten_..."));
```

JS splits `resource`/`action`; .NET takes a colon-joined `permission` string. Same operation, different shapes.

### Pagination

JS list ops accept `{ pageSize, cursor }`; response shape varies per endpoint (e.g. `ListUsersResponse` = `{ data, nextCursor, pageSize }` — see `management/types.ts` for exact shapes). .NET equivalents take `pageSize`/`cursor` and return `ListResponse<T>`. Loop until `nextCursor` is null/empty.

## Step 4 — Verify

For every operation wired up:

1. **Happy path** — call it; observe response; assert fields your code reads.
2. **Permissions** — call with a key lacking the scope; expect 403 `Forbidden`.
3. **Cross-tenant** — if multi-tenant, call a tenant-scoped endpoint with a key scoped to a different tenant; expect 403.

## Anti-patterns

- **Don't store the API key in the frontend.** Server-only credential.
- **Don't use `result.IsError`** in .NET — it's `result.IsSuccess` (or `result.Error != null`).
- **Don't wrap SDK calls in `try/catch` for control flow.** Use the result type.
- **Don't `Guid.Parse` IDs from API responses without checking the docs.** Many Authaz IDs are prefixed strings (`user_01abc…`), not GUIDs.
- **Don't drop the `/api` prefix in raw `curl` calls.** Both SDKs use `/api/v1/...`; `/v1/...` 404s. Prefer SDK methods anyway.
- **Don't rely on `@authaz/sdk`'s default `apiDomain`.** It's `https://api.authaz.com`, which doesn't resolve to Authaz — always pass `apiDomain` (or `baseAddress`).
- **Don't point the SDK at `dashboard.authaz.io`** because that's what `authaz whoami` uses. That's the CLI's BFF, a different surface.
- **Don't share one API key across services.** Per-service keys keep rotation isolated.
- **Don't treat `role.isGlobal` as "grants app-wide access."** It only means the role is centrally *defined*; the actual access scope comes from the `tenantId` passed at invite/assign time.
- **Don't send an invitation without `tenantId` from a tenant-scoped surface.** A missing `tenantId` produces a tenantless (app-wide) grant, not an error.
- **Don't filter `users.listRoles()` results on `roles[].tenantId` to find a tenant's members.** That's the role's definition tenant (often null), not the assignment scope — use `roles.getUsersWithRoles({ tenantId })` instead.

## Source of truth

- .NET sample: `authaz-sdk-dotnet/Authaz.Sdk.Sample/Program.cs`
- .NET SDK surface: `authaz-sdk-dotnet/Authaz.Sdk/src/`
- JS SDK entry point: `authaz-sdk-js/packages/core/src/index.ts`
- JS SDK client: `authaz-sdk-js/packages/core/src/client.ts`

## References

- `endpoints.md` — high-level catalog of sub-clients
- `error-codes.md` — error shape and retry guidance
- `authaz-cli` skill — the CLI's own host/flags, for the contrast above
- `permission-check.md` — runtime `authz.check` patterns
- `multi-tenant.md` — tenant scoping
