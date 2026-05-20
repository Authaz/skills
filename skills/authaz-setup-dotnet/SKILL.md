---
name: authaz-setup-dotnet
description: Use when adding Authaz to an ASP.NET Core app. Wires the standard `Microsoft.AspNetCore.Authentication.OpenIdConnect` middleware against Authaz's OIDC, plus `Authaz.Sdk` for Management API calls. Triggers on "add Authaz to ASP.NET", "Authaz.Sdk", "OIDC Authaz", ".NET login".
---

# Set up Authaz in an ASP.NET Core app

Targets ASP.NET Core 8/9/10. Uses the **standard OIDC middleware** — Authaz is just an OIDC provider to your app — and `Authaz.Sdk` (NuGet) for Management API calls.

There is no `Authaz.AspNetCore` adapter. Don't invent one. The standard `AddOpenIdConnect` is the integration.

## Pre-flight

1. **Confirm framework.** `*.csproj` with `Microsoft.NET.Sdk.Web` and TFM `net8.0`/`net9.0`/`net10.0`.
2. **Confirm tenancy mode** (from `authaz-quickstart` if available).
3. **Confirm app + API key.** The SDK uses an API key (`sk_live_…`); the OIDC middleware uses `client_id` + `client_secret`. They are different.

## Step 1 — Application setup

Dashboard or:

```http
POST https://api.authaz.io/api/v1/applications
X-API-Key: sk_live_…

{ "name": "my-aspnet-app", "tenancy_type": "single_tenant" }
```

Allowed callback URL: `https://localhost:5001/signin-oidc` (the OIDC middleware's default — don't change unless you have a reason).

For Management API access from the app, **API Keys → Create API Key** and grant the scopes you'll actually use (e.g., `users:read`).

## Step 2 — Install

```bash
dotnet add package Authaz.Sdk
dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect
```

## Step 3 — Configure

`appsettings.json`:

```json
{
  "Authaz": {
    "Authority": "https://auth.example.com",
    "ClientId": "app_01abc…",
    "ClientSecret": "cs_live_…",
    "ApiBaseUrl": "https://api.authaz.io",
    "ApiKey": "sk_live_…"
  }
}
```

Replace `auth.example.com` with your identity domain from the Authaz Dashboard. In production, move `ClientSecret` and `ApiKey` to user secrets, env vars (`Authaz__ClientSecret`), or a secrets manager. Never commit them.

## Step 4 — Wire it up

Recipe:

- **Single-tenant**: <https://authaz.io/docs/recipes/dotnet-single-tenant>
- **Multi-tenant**: <https://authaz.io/docs/recipes/dotnet-multi-tenant>

The shape in `Program.cs`:

1. `AddAuthentication` with `CookieAuthenticationDefaults.AuthenticationScheme` as the default and `OpenIdConnectDefaults.AuthenticationScheme` as the challenge.
2. `AddCookie()` for session storage.
3. `AddOpenIdConnect(options => { Authority, ClientId, ClientSecret, ResponseType = "code", UsePkce = true, SaveTokens = true; Scope.Add("openid"); ... })`.
4. `AddAuthazSdk()` from `Authaz.Sdk` to register the typed Management API clients.
5. `[Authorize]` on protected endpoints; `app.MapGet(...).RequireAuthorization()` works on minimal APIs.
6. **For multi-tenant**, write a custom `AuthorizationHandler` that reads the `tenant_id` claim from `HttpContext.User` and scopes the policy to it. The recipe shows a `TenantPolicyRequirement` example.

Copy from the recipe — the option setters and order matter.

## Step 5 — Verify

```bash
dotnet run
```

Then:

1. Browse to `https://localhost:5001/` — public.
2. Browse to `https://localhost:5001/dashboard` — bounces to your Authaz identity domain, signs in, returns. The page shows `User.FindFirst("email")?.Value`.
3. Hit `/me/profile` — calls `IAuthazClient.Users.GetAsync(...)` and returns the JSON.
4. For multi-tenant, confirm `User.FindFirst("tenant_id")` returns a value, and the `AuthorizationHandler` enforces it.

## Anti-patterns

- **Don't manually validate the access token by hand.** `AddOpenIdConnect` does it via the published JWKS — let it.
- **Don't use `ResultExt.Forbidden()`-style helpers from the Authaz monorepo here** — that's internal Authaz code, not the SDK. Use `Results.Forbid()` (or `[Authorize(Policy=...)]` with a denied requirement).
- **Don't hard-code the authority.** It comes from config so test/staging/prod can differ.
- **Don't bind `tenant_id` to a query string** for tenant-scoped endpoints — read it from `User.Claims` only.
- **Don't disable PKCE.** Authaz requires it.
- **`Authaz.Sdk` returns `AuthazResult<T>`, not exceptions.** Check `result.IsError` — don't wrap calls in `try/catch` looking for an `AuthazException` that doesn't exist.

## References

- Recipe (single-tenant): <https://authaz.io/docs/recipes/dotnet-single-tenant>
- Recipe (multi-tenant): <https://authaz.io/docs/recipes/dotnet-multi-tenant>
- SDK reference: <https://authaz.io/docs/sdk/overview>
- `references/endpoints.md`, `references/error-codes.md`
- `authaz-management-api`, `authaz-multi-tenant`, `authaz-troubleshoot-oauth`
