---
name: authaz-setup-dotnet
description: Use when adding Authaz to an ASP.NET Core app (net8/9/10). Covers two integrations — signing users in via OpenID Connect against Authaz, and calling the Management API via `Authaz.Sdk`. Single-shot for each half. Triggers on "add Authaz to ASP.NET", "Authaz.Sdk", "OIDC Authaz", ".NET login".
---

# Set up Authaz in an ASP.NET Core app — single shot

> **Last verified:** `Authaz.Sdk` v0.1.2 (2026-05-21). The .NET SDK is pre-1.0 — APIs are unstable. If `AddAuthazSdk`, `IAuthazClient`, or `AuthazResult<T>` look different in the installed package, trust the SDK source and report the drift.

There are **two distinct integrations** for Authaz on .NET. Pick the one that matches the user's intent before writing code; they have nothing in common.

| Goal | What you wire up | Authaz packages |
|---|---|---|
| **Sign users in** (web app with a login page) | The standard `Microsoft.AspNetCore.Authentication.OpenIdConnect` middleware against Authaz's hosted OIDC | (none — pure ASP.NET) |
| **Call the Management API** (admin operations from your backend) | `Authaz.Sdk` from NuGet | `Authaz.Sdk` |

If the user wants both, do them in order: sign-in first, Management API second. Each section below is independently single-shot.

> Note: the .NET SDK ships *no* OIDC middleware. The sign-in path uses Microsoft's standard OIDC stack — Authaz is just an OAuth 2.1 / OIDC provider. Don't look for an `AddAuthazAuthentication()`; it doesn't exist.

---

## Part A — Sign users in (OpenID Connect)

### What the user must provide

Three values from the Authaz Dashboard:

| Setting | Where it comes from |
|---|---|
| `ClientId` | Dashboard → your application → Auth Flow Configuration |
| `ClientSecret` | Same place. Shown once at creation |
| `Authority` | `https://auth.authaz.io` (default). Override only for a custom identity domain |

Plus, in the Dashboard, add the callback URL to **Allowed callback URLs**. ASP.NET's OIDC handler defaults to `/signin-oidc`, so add `https://localhost:5001/signin-oidc` (or your dev port).

### Step A.1 — Install

```bash
dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect
```

### Step A.2 — Configure `appsettings.json`

```json
{
  "Authaz": {
    "Authority": "https://auth.authaz.io",
    "ClientId": "your_client_id",
    "ClientSecret": "your_client_secret"
  }
}
```

For production, move `ClientSecret` out of `appsettings.json` — use `dotnet user-secrets`, environment variables (`Authaz__ClientSecret`), or a secrets manager. Never commit it.

### Step A.3 — `Program.cs`

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

Order matters: `UseAuthentication()` before `UseAuthorization()` before the endpoints. `RequireAuthorization()` is the per-endpoint gate; on controllers use `[Authorize]`.

### Step A.4 — Run and verify

```bash
dotnet run
```

Then in the browser:

1. `https://localhost:5001/` — public.
2. `https://localhost:5001/dashboard` — redirects to `https://auth.authaz.io/...` → sign in → returns to `/dashboard` showing the user's email.
3. Inspect cookies: there's an ASP.NET auth cookie set by `AddCookie()`. Tokens (access, refresh, id) are in the auth ticket if `SaveTokens = true`.

### Failure modes for sign-in

1. **Redirect-loop or `invalid_redirect_uri`** — the allowed-callback list in the Dashboard doesn't include `/signin-oidc` on the exact host+port the dev server uses.
2. **`PKCE required`** — `UsePkce = true` is missing. Authaz mandates PKCE.
3. **`Correlation failed`** on callback — the antiforgery cookie was dropped, usually because the dev server flipped between HTTP and HTTPS mid-session. Stick to HTTPS in dev.

---

## Part B — Call the Management API from your backend

### What the user must provide

Two values from the Authaz Dashboard:

| Setting | Where it comes from |
|---|---|
| `ApiKey` | Dashboard → API Keys → Create. Shown once. Scoped to specific permissions |
| `BaseAddress` | `https://api.authaz.io` (default for the hosted product) |

Optionally `ApplicationId` if your operations need to specify an application via header.

### Step B.1 — Install

```bash
dotnet add package Authaz.Sdk
```

### Step B.2 — Configure `appsettings.json`

```json
{
  "AuthazSample": {
    "BaseUrl": "https://api.authaz.io",
    "ApiKey": "",
    "ApplicationId": ""
  }
}
```

(Rename `AuthazSample` to whatever your config section is — the SDK doesn't care.)

For development:

```bash
dotnet user-secrets init
dotnet user-secrets set "AuthazSample:ApiKey" "authaz_..."
dotnet user-secrets set "AuthazSample:ApplicationId" "00000000-..."
```

### Step B.3 — Strongly-typed options class

```csharp
// SampleOptions.cs
namespace YourApp;

public class SampleOptions
{
    public string BaseUrl { get; set; } = "https://api.authaz.io";
    public string ApiKey { get; set; } = string.Empty;
    public string ApplicationId { get; set; } = string.Empty;
}
```

### Step B.4 — `Program.cs` — register the SDK

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

`AddAuthazSdk(...)` registers `IAuthazClient` as a singleton along with the underlying `HttpClient`, retry policy, and auth handler. **Don't** also register a custom `HttpClient` named `"Authaz"` — you'll shadow the SDK's.

### Step B.5 — Use `IAuthazClient` from endpoints

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

The SDK returns `AuthazResult<T>`:

- `result.IsSuccess` — boolean
- `result.Value` — the response payload (null on failure)
- `result.Error` — typed `AuthazError` (null on success)

**Always check `IsSuccess` first.** The SDK does not throw on logical errors (404, 403, validation). Wrapping calls in `try/catch` only masks the result-type contract.

### Available sub-clients on `IAuthazClient`

- `Users` — get, list, suspend, activate, sessions, role assignments
- `Applications` — list, manage applications
- `Tenants` — list, create, update tenants
- `Authorization` — `.Roles`, `.Permissions` (including `CheckAsync`)
- `Invitations` — send, list, pending count
- `Email` — providers, templates
- `Auth` — API keys, OAuth credentials
- `Audit` — audit trail

See `authaz-sdk-dotnet/Authaz.Sdk/src/Resources/` for the full surface.

### Step B.6 — Run and verify

```bash
dotnet run
curl http://localhost:5169/users?pageSize=20
```

You should see a JSON array of users. If you get 401, the API key is missing or wrong. If you get 403, the key lacks the scope.

### Failure modes for the SDK

1. **`Unauthorized`** — `ApiKey` empty, mistyped, or revoked. Verify with `dotnet user-secrets list`.
2. **`Forbidden`** — the API key doesn't have the scope for that endpoint. Grant in the Dashboard.
3. **`Validation`** — the request body's shape doesn't match. The `AuthazError.Validation` carries a `Fields` collection with field-specific messages.
4. **`NotFound`** when the ID looks right** — the API key might be scoped to a different application/tenant; the resource exists, but is invisible to this key.

---

## Production checklist

When you move beyond `localhost`:

- [ ] **Add the prod callback URL** in the Dashboard's Allowed callback URLs: `https://your-app.com/signin-oidc` (the default OIDC callback path).
- [ ] **Move `ClientSecret` and `ApiKey` to a secrets manager** (Azure Key Vault, AWS Secrets Manager, env vars). Never commit `appsettings.json` with secrets.
- [ ] **HTTPS only.** ASP.NET's OIDC middleware sets `Secure` cookies; plain HTTP breaks them.
- [ ] **Behind a reverse proxy?** Call `app.UseForwardedHeaders(new() { ForwardedHeaders = ForwardedHeaders.XForwardedProto | ForwardedHeaders.XForwardedFor })` before `UseAuthentication()`, otherwise OIDC will build callback URLs as `http://` and the IdP will reject them.
- [ ] **Custom identity domain?** Update `Authaz:Authority` in the appropriate environment's `appsettings.{Environment}.json` and confirm the same custom domain is configured in the Dashboard.
- [ ] **Set `Cookie.SameSite` and `Cookie.SecurePolicy`** on `AddCookie` if needed for cross-site flows — defaults are `Lax` + `Always`, usually right.

## Anti-patterns

- **Don't wrap SDK calls in `try/catch` looking for an `AuthazException`.** It doesn't exist. The SDK returns `AuthazResult<T>` and never throws on logical errors.
- **Don't use `result.IsError`.** It's `result.IsSuccess` (check for true) or check `result.Error != null`.
- **Don't bypass `UsePkce = true`.** Authaz requires PKCE on the authorization-code grant.
- **Don't put `ClientSecret` or `ApiKey` in `appsettings.json` committed to source.** User secrets, env vars, or a manager.
- **Don't add `AddAuthazAuthentication()` or any Authaz-shaped OIDC helper** — they don't exist. Use Microsoft's `AddOpenIdConnect`.
- **Don't `Guid.Parse` IDs from API responses without checking the docs.** Many Authaz IDs are strings (`user_01abc…`), not GUIDs.

## Source of truth

- The SDK code above is lifted from `authaz-sdk-dotnet/Authaz.Sdk.Sample/Program.cs` and `appsettings.json`.
- The OIDC code is standard ASP.NET against the Authaz authority. There is no shipped OIDC sample in `authaz-sdk-dotnet`; if the customer hits a behavior question on the OIDC half, verify against the Microsoft OIDC handler docs.

## References

- Real SDK sample: `authaz-sdk-dotnet/Authaz.Sdk.Sample/`
- SDK source: `authaz-sdk-dotnet/Authaz.Sdk/src/`
- `authaz-management-api` — deeper coverage of SDK clients
- `authaz-troubleshoot-oauth` — OIDC failure diagnostics
- `authaz-multi-tenant`, `authaz-permission-check`
