---
name: authaz-setup-dotnet
description: Use when adding Authaz to an ASP.NET Core app (net8/9/10). Covers two integrations — signing users in via OpenID Connect against Authaz, and calling the Management API via `Authaz.Sdk`. Single-shot for each half. Triggers on "add Authaz to ASP.NET", "Authaz.Sdk", "OIDC Authaz", ".NET login".
---

# Set up Authaz in an ASP.NET Core app — single shot

> **Last verified:** `Authaz.Sdk` v0.1.2 (2026-05-21). Pre-1.0 SDK — APIs unstable. If `AddAuthazSdk`, `IAuthazClient`, or `AuthazResult<T>` differ from the installed package, trust the SDK source and report the drift.

Two distinct, unrelated integrations. Pick the one matching user intent before coding:

| Goal | What you wire up | Authaz packages |
|---|---|---|
| **Sign users in** (web app login page) | Standard `Microsoft.AspNetCore.Authentication.OpenIdConnect` middleware against Authaz's hosted OIDC | (none — pure ASP.NET) |
| **Call the Management API** (admin ops from backend) | `Authaz.Sdk` from NuGet | `Authaz.Sdk` |

If both are wanted: sign-in first, Management API second. Each section is independently single-shot.

> The sign-in path is Microsoft's standard OIDC stack (`AddOpenIdConnect`) — Authaz is just an OAuth 2.1/OIDC provider; the .NET SDK has no Authaz-specific OIDC wrapper.

---

## Part A — Sign users in (OpenID Connect)

### User must provide (from Authaz Dashboard)

| Setting | Where it comes from |
|---|---|
| `ClientId` | Dashboard → your application → Auth Flow Configuration |
| `ClientSecret` | Same place. Shown once at creation |
| `Authority` | `https://auth.authaz.io` (default). Override only for a custom identity domain |

Also add the callback URL in Dashboard → **Allowed callback URLs**. ASP.NET's OIDC handler defaults to `/signin-oidc`, so add `https://localhost:5001/signin-oidc` (or your dev port).

### A.1 — Install

```bash
dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect
```

### A.2 — `appsettings.json`

```json
{
  "Authaz": {
    "Authority": "https://auth.authaz.io",
    "ClientId": "your_client_id",
    "ClientSecret": "your_client_secret"
  }
}
```

Production: keep `ClientSecret` out of `appsettings.json` — use `dotnet user-secrets`, env vars (`Authaz__ClientSecret`), or a secrets manager. Never commit it.

### A.3 — `Program.cs`

```csharp
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(options =>
    {
        options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
    })
    .AddCookie()
    .AddOpenIdConnect(options =>
    {
        options.Authority = builder.Configuration["Authaz:Authority"];
        options.ClientId = builder.Configuration["Authaz:ClientId"];
        options.ClientSecret = builder.Configuration["Authaz:ClientSecret"];

        options.ResponseType = "code";
        options.UsePkce = true;            // required by Authaz
        options.SaveTokens = true;
        options.GetClaimsFromUserInfoEndpoint = true;

        options.Scope.Clear();
        options.Scope.Add("openid");
        options.Scope.Add("email");
        options.Scope.Add("profile");
        options.Scope.Add("offline_access");

        options.TokenValidationParameters = new TokenValidationParameters
        {
            NameClaimType = "email",
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/", () => "Public landing page");
app.MapGet("/dashboard", (HttpContext ctx) => $"Signed in as {ctx.User.Identity!.Name}")
   .RequireAuthorization();

app.Run();
```

Order matters: `UseAuthentication()` → `UseAuthorization()` → endpoints. `RequireAuthorization()` is the per-endpoint gate; on controllers use `[Authorize]`.

### A.4 — Run and verify

```bash
dotnet run
```

In the browser:

1. `https://localhost:5001/` — public.
2. `https://localhost:5001/dashboard` — redirects to `https://auth.authaz.io/...` → sign in → back to `/dashboard` showing the user's email.
3. Cookies: an ASP.NET auth cookie is set by `AddCookie()`. Tokens (access, refresh, id) are in the auth ticket if `SaveTokens = true`.

### Failure modes — sign-in

| Symptom | Cause |
|---|---|
| Redirect-loop or `invalid_redirect_uri` | Dashboard's allowed-callback list doesn't include `/signin-oidc` on the exact host+port the dev server uses |
| `PKCE required` | `UsePkce = true` missing — Authaz mandates PKCE |
| `Correlation failed` on callback | Antiforgery cookie dropped, usually from the dev server flipping HTTP/HTTPS mid-session. Stick to HTTPS in dev |

---

## Part B — Call the Management API from your backend

### User must provide (from Authaz Dashboard)

| Setting | Where it comes from |
|---|---|
| `ApiKey` | Dashboard → API Keys → Create. Shown once. Scoped to specific permissions |
| `BaseAddress` | `https://api.authaz.io` (default for hosted product) |

`AuthazClientOptions` has no `ApplicationId` — no global application-scoping option. To scope an operation to an application, pass it as an explicit parameter (e.g. `IApiKeysClient.CreateAppKeyAsync(applicationId, ...)`).

### B.1 — Install

```bash
dotnet add package Authaz.Sdk
```

### B.2 — `appsettings.json`

```json
{
  "AuthazSample": {
    "BaseUrl": "https://api.authaz.io",
    "ApiKey": ""
  }
}
```

(Rename `AuthazSample` to your config section — the SDK doesn't care.)

Development:

```bash
dotnet user-secrets init
dotnet user-secrets set "AuthazSample:ApiKey" "authaz_..."
```

### B.3 — Strongly-typed options class

```csharp
// SampleOptions.cs
namespace YourApp;

public class SampleOptions
{
    public string BaseUrl { get; set; } = "https://api.authaz.io";
    public string ApiKey { get; set; } = string.Empty;
}
```

### B.4 — `Program.cs` — register the SDK

```csharp
using Authaz.Sdk;
using YourApp;

var builder = WebApplication.CreateBuilder(args);

builder.Services.Configure<SampleOptions>(builder.Configuration.GetSection("AuthazSample"));

builder.Services.AddAuthazSdk(opts =>
{
    var sampleOpts = builder.Configuration.GetSection("AuthazSample");
    opts.BaseAddress = new Uri(sampleOpts["BaseUrl"] ?? "https://api.authaz.io");
    opts.ApiKey      = sampleOpts["ApiKey"];
});

var app = builder.Build();
```

`AddAuthazSdk(...)` registers `IAuthazClient` as a singleton plus the underlying `HttpClient`, retry policy, and auth handler. Let it own the `"Authaz"` `HttpClient` name — registering another one under that name shadows the SDK's.

### B.5 — Use `IAuthazClient` from endpoints

```csharp
// ── Users ──────────────────────────────────────────────────────────────
app.MapGet("/users", async (IAuthazClient authaz, int pageSize = 20, string? search = null) =>
{
    var result = await authaz.Users.ListAsync(pageSize: pageSize, search: search);
    return result.IsSuccess ? Results.Ok(result.Value) : ToError(result.Error!);
});

app.MapGet("/users/{id:guid}", async (Guid id, IAuthazClient authaz) =>
{
    var result = await authaz.Users.GetAsync(id);
    return result.IsSuccess ? Results.Ok(result.Value) : ToError(result.Error!);
});

app.MapPost("/users/{id:guid}/suspend", async (Guid id, IAuthazClient authaz) =>
{
    var result = await authaz.Users.SuspendAsync(id);
    return result.IsSuccess ? Results.Ok(result.Value) : ToError(result.Error!);
});

// ── Authorization check ───────────────────────────────────────────────
app.MapPost("/authorization/check", async (CheckBody body, IAuthazClient authaz) =>
{
    var result = await authaz.Authorization.Permissions.CheckAsync(
        new CheckPermissionRequest(body.UserId, body.Permission, body.TenantId?.ToString()));
    return result.IsSuccess ? Results.Ok(result.Value) : ToError(result.Error!);
});

app.Run();

// ── Error mapping ──────────────────────────────────────────────────────
static IResult ToError(AuthazError error) => error switch
{
    AuthazError.NotFound e      => Results.NotFound(new { error = "not_found", message = e.Message }),
    AuthazError.Unauthorized    => Results.Unauthorized(),
    AuthazError.Forbidden       => Results.Forbid(),
    AuthazError.Validation e    => Results.ValidationProblem(
                                       e.Fields.ToDictionary(f => f.Field, f => new[] { f.Message })),
    AuthazError.RateLimited e   => Results.Problem(detail: e.Message, statusCode: 429),
    _                            => Results.Problem(detail: error.Message, statusCode: 500),
};

record CheckBody(string UserId, string Permission, Guid? TenantId = null);
```

SDK returns `AuthazResult<T>`:

| Member | Meaning |
|---|---|
| `result.IsSuccess` | boolean |
| `result.Value` | response payload (null on failure) |
| `result.Error` | typed `AuthazError` (null on success) |

**Always check `IsSuccess` first.** The SDK doesn't throw on logical errors (404, 403, validation) — wrapping calls in `try/catch` only masks the result-type contract.

### `IAuthazClient` sub-clients

| Client | Covers |
|---|---|
| `Users` | get, list, suspend, activate, sessions, role assignments |
| `Applications` | list, manage applications |
| `Tenants` | list, create, update tenants |
| `Authorization` | `.Roles`, `.Permissions` (incl. `CheckAsync`) |
| `Invitations` | send, list, pending count |
| `Email` | providers, templates |
| `Auth` | API keys, OAuth credentials |
| `Audit` | audit trail |

Full surface: `authaz-sdk-dotnet/Authaz.Sdk/src/Resources/`.

### B.6 — Run and verify

```bash
dotnet run
curl http://localhost:5169/users?pageSize=20
```

Expect a JSON array of users. 401 → API key missing/wrong. 403 → key lacks the scope.

### Failure modes — SDK

| Symptom | Cause |
|---|---|
| `Unauthorized` | `ApiKey` empty, mistyped, or revoked. Verify with `dotnet user-secrets list` |
| `Forbidden` | API key lacks scope for that endpoint. Grant in Dashboard |
| `Validation` | Request body shape mismatch; `AuthazError.Validation` carries a `Fields` collection with field-specific messages |
| `NotFound` when the ID looks right | API key may be scoped to a different application/tenant; resource exists but is invisible to this key |

---

## Production checklist

- [ ] **Add prod callback URL** in Dashboard's Allowed callback URLs: `https://your-app.com/signin-oidc` (default OIDC callback path).
- [ ] **Move `ClientSecret` and `ApiKey` to a secrets manager** (Azure Key Vault, AWS Secrets Manager, env vars). Never commit `appsettings.json` with secrets.
- [ ] **HTTPS only** — ASP.NET's OIDC middleware sets `Secure` cookies; plain HTTP breaks them.
- [ ] **Behind a reverse proxy?** Call `app.UseForwardedHeaders(new() { ForwardedHeaders = ForwardedHeaders.XForwardedProto | ForwardedHeaders.XForwardedFor })` before `UseAuthentication()`, otherwise OIDC builds callback URLs as `http://` and the IdP rejects them.
- [ ] **Custom identity domain?** Update `Authaz:Authority` in the env's `appsettings.{Environment}.json` and confirm the same custom domain is configured in the Dashboard.
- [ ] **Set `Cookie.SameSite`/`Cookie.SecurePolicy`** on `AddCookie` if needed for cross-site flows — defaults (`Lax` + `Always`) are usually right.

## Anti-patterns

- **Don't wrap SDK calls in `try/catch` looking for `AuthazException`.** Doesn't exist. SDK returns `AuthazResult<T>`, never throws on logical errors.
- **Don't use `result.IsError`.** It's `result.IsSuccess` (check true) or `result.Error != null`.
- **Don't bypass `UsePkce = true`.** Authaz requires PKCE on the authorization-code grant.
- **Don't put `ClientSecret`/`ApiKey` in committed `appsettings.json`.** Use user secrets, env vars, or a manager.
- **Don't add `AddAuthazAuthentication()` or any Authaz-shaped OIDC helper** — doesn't exist. Use Microsoft's `AddOpenIdConnect`.
- **Don't assume every resource ID is a `Guid`.** User, application, tenant, credential, role, and invitation IDs are all `System.Guid`. Exception: `IPoliciesClient` — policy IDs are `string`.

## Source of truth

- SDK code above is lifted from `authaz-sdk-dotnet/Authaz.Sdk.Sample/Program.cs` and `appsettings.json`.
- OIDC code is standard ASP.NET against the Authaz authority — no shipped OIDC sample in `authaz-sdk-dotnet`; for OIDC behavior questions, verify against Microsoft's OIDC handler docs.

## References

- Real SDK sample: `authaz-sdk-dotnet/Authaz.Sdk.Sample/`
- SDK source: `authaz-sdk-dotnet/Authaz.Sdk/src/`
- `authaz-management-api` — deeper coverage of SDK clients
- `authaz-troubleshoot-oauth` — OIDC failure diagnostics
- `authaz-multi-tenant`, `authaz-permission-check`
