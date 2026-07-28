# Authaz error codes

Common errors a developer will see while integrating Authaz, with the actionable fix. If you see one of these, lead with the fix — don't add error handling for "what if the server is down" cases that won't help the user.

Verified against `authaz/Authaz.Domain/OAuth/OAuth2Error.cs`, `authaz/Authaz.Shared/Extensions/OAuth2Results.cs`, and `authaz-sdk-dotnet/Authaz.Sdk/src/Http/AuthazHttpClient.cs` on 2026-05-21.

## OAuth 2 errors (token endpoint + redirect-with-error)

These are the codes the Authaz authorization server actually emits. They appear either as `?error=...` on the redirect-back URI or in a JSON `{ error, error_description }` body from the token endpoint (RFC 6749 § 5.2).

| `error` | Fix |
|---|---|
| `invalid_request` | The request is malformed — missing `client_id` / `redirect_uri` / `scope` / `state` / `code_challenge`, or a value didn't validate. Authaz also emits this when the `redirect_uri` doesn't exactly match an allowed callback (no separate `invalid_redirect_uri` code is sent — add the exact URL to **Allowed callback URLs**) |
| `invalid_client` | `client_id` / `client_secret` mismatch, or the client doesn't exist. Returned as HTTP 401 |
| `invalid_grant` | Authorization code is expired/used, `code_verifier` doesn't match the original `code_challenge`, refresh token invalid, PKCE missing, redirect-uri mismatch on token exchange, refresh-token reuse detected |
| `unauthorized_client` | The client is not authorized to use this grant type. Returned as HTTP 401 |
| `unsupported_grant_type` | Use one of `authorization_code`, `refresh_token`, `client_credentials` |
| `invalid_scope` | A scope was requested the app doesn't allow — remove or grant it |
| `invalid_token` | Bearer token is expired, revoked, malformed, or invalid (RFC 6750 § 3.1). Returned as HTTP 401 |
| `insufficient_scope` | The bearer token lacks the scope this resource requires. Returned as HTTP 403 |
| `invalid_target` | Token-exchange `resource`/`audience` is invalid or unsupported (RFC 8693) |
| `tenant_required` | **Authaz-specific.** The flow needs a tenant hint but none was sent. Restart the authorize request with `tenant_hint=<id>` |
| `invalid_dpop_proof` | DPoP proof failed (RFC 9449). Check the `htm`/`htu`/`iat` claims and the proof binding to the access token |

## JWT validation errors (when verifying tokens server-side)

These come from your JWT library (`jose`, `Microsoft.IdentityModel.Tokens`, etc.), not from Authaz directly — but the fix is always about your verifier or Authaz's keys.

| Error | Fix |
|---|---|
| `kid not found in JWKS` | The token was signed with a key that's no longer published. Most likely the user's session predates a key rotation. Force re-login. |
| `invalid signature` | Token was tampered with, or you're verifying against the wrong JWKS. Confirm the JWKS URL is `https://auth.authaz.io/.well-known/jwks/{clientId}.json` (per-app keys, not a global JWKS) |
| `token expired` | Refresh the access token using the refresh token, or force a new login |
| `aud mismatch` | The token was issued for a different application |
| `iss mismatch` | The token's issuer doesn't match the expected identity domain. Check for a custom-domain mismatch |
| `clock skew too large` | The server's clock is more than 60s off real time. Fix NTP |

## Management API errors (RFC 7807 ProblemDetails)

The Management API returns RFC 7807 `application/problem+json` responses. Validation failures use the `ValidationProblemDetails` extension (with an `errors` map). Verified at `authaz-sdk-dotnet/Authaz.Sdk/src/Http/ProblemDetailsDto.cs`.

Shape:

```json
{ "type": "...", "title": "...", "status": 422, "detail": "...", "instance": "..." }
```

Validation shape:

```json
{ "type": "...", "title": "Validation failed", "status": 400,
  "errors": { "email": ["must be a valid email"] } }
```

| HTTP | When | Fix |
|---|---|---|
| 400 (no `errors` map) | Generic bad request | Check the body shape against the SDK's request type |
| 400 (with `errors` map) | Validation failed | Each field has at least one message — show them to the user |
| 401 | `X-API-Key` missing / malformed / revoked | Re-issue the key in the Dashboard; rotate any code paths using the old one |
| 403 | The key (or user) lacks the permission for this endpoint or tenant | Grant the permission in the Dashboard; for cross-tenant access, the key must be scoped to the tenant |
| 404 | Resource doesn't exist, or the API key can't see it (tenant scope) | Verify the resource id; verify the key's tenant scope |
| 409 | Conflict — e.g., a user with that email already exists | Look up the existing resource and merge / update instead |
| 429 | Rate limited | Honor the `Retry-After` header; back off |
| 5xx | Server-side failure | Retry on idempotent reads; surface as 502 to your caller for writes |

## SDK error mapping

### `Authaz.Sdk` (.NET)

The SDK returns `AuthazResult<T>`. **It does not throw on logical errors** (404, 403, validation). On failure, `result.Error` is one of the typed subtypes from `authaz-sdk-dotnet/Authaz.Sdk/src/Results/AuthazError.cs`:

| Subtype | Trigger |
|---|---|
| `Unauthorized` | HTTP 401 |
| `Forbidden` | HTTP 403 |
| `NotFound` | HTTP 404 |
| `Conflict` | HTTP 409 |
| `Validation` | HTTP 400 with a `ValidationProblemDetails` body. `.Fields` carries each failing field |
| `RateLimited` | HTTP 429. `.RetryAfter` is the parsed `Retry-After` header (if present) |
| `Server` | HTTP 5xx (or 400 without a validation body) |
| `Network` | `HttpRequestException`, `IOException`, or operation timeout |
| `Canceled` | The caller's `CancellationToken` was canceled |

Always pattern-match:

```csharp
var result = await authaz.Users.GetAsync(userId);
return result.Error switch
{
    null                       => Results.Ok(result.Value),
    AuthazError.NotFound       => Results.NotFound(),
    AuthazError.Unauthorized   => Results.Unauthorized(),
    AuthazError.Forbidden      => Results.Forbid(),
    AuthazError.Validation v   => Results.ValidationProblem(
                                      v.Fields.ToDictionary(f => f.Field, f => new[] { f.Message })),
    AuthazError.RateLimited r  => Results.Problem(detail: r.Message, statusCode: 429),
    _                          => Results.Problem(detail: result.Error.Message, statusCode: 500),
};
```

Every subtype carries `RequestId` (from the `X-Request-Id` response header) — log it for support tickets.

### `@authaz/sdk` (JS)

Returns `Result<T>` with `ok: true/false`. Errors are `AuthazError` (or subtypes `AuthazUnauthorizedError`, `AuthazNetworkError`, `AuthazValidationError`). Use `isOk(result)` / `isErr(result)`.

### `@authaz/next`, `@authaz/hono`, `@authaz/react`

Errors propagate as JSON `{ error, error_description }` from the auth handler routes. Wrap calls in `try/catch` and surface the `error` field to the user; never leak `error_description` (it can include internal hints).

## When to retry

Retry only on transient errors: HTTP 429 (honor `Retry-After`), 502/503/504, network timeouts. Use exponential backoff with jitter, max 3 attempts.

**Never retry**: `invalid_grant`, `invalid_client`, `unsupported_grant_type`, `Validation` (400 with errors map), `NotFound`, `Conflict`, `Forbidden`, `Unauthorized` — those are programming/permission bugs, not flakes.
