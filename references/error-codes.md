# Authaz error codes

Common errors a developer will see while integrating Authaz, with the actionable fix. If you see one of these, lead with the fix — don't add error handling for "what if the server is down" type cases that won't help the user.

## OAuth 2.1 errors (returned to the redirect URI as `?error=`)

| `error` | What it means | Fix |
|---|---|---|
| `invalid_request` | The authorize request is malformed | Check `client_id`, `redirect_uri`, `scope`, `state`, `code_challenge` are all present and URL-encoded |
| `unauthorized_client` | The client is not allowed to use this grant type | Check the application's allowed grant types in the Dashboard |
| `access_denied` | The end user denied consent or the auth flow failed | User-facing — render a message and offer "Try again" |
| `invalid_redirect_uri` | The `redirect_uri` is not on the allow list | Add the exact URL (including port and trailing slash) to **Auth Flow Configuration → Allowed callback URLs** |
| `unsupported_response_type` | A response type other than `code` was requested | Use `response_type=code` only |
| `invalid_scope` | A scope was requested that the app doesn't have | Remove the bad scope or grant it on the app |
| `server_error` | Authaz failed | Retry with backoff; if persistent, check status page |
| `temporarily_unavailable` | Authaz is overloaded | Retry with backoff |

## Token endpoint errors (returned as JSON 400)

| `error` | Fix |
|---|---|
| `invalid_grant` | The auth code expired (60s TTL), was already used, or `code_verifier` doesn't match the original `code_challenge` |
| `invalid_client` | `client_id` / `client_secret` mismatch or the client doesn't exist |
| `invalid_request` | Missing `grant_type`, `code`, `redirect_uri`, or `code_verifier` |
| `unsupported_grant_type` | Authaz only accepts `authorization_code`, `refresh_token`, `client_credentials` |

## JWT validation errors (when verifying tokens server-side)

| Error | Fix |
|---|---|
| `kid not found in JWKS` | The token was signed with a key that's no longer published — the user's session is from before a key rotation. Force re-login. |
| `invalid signature` | Token was tampered with, or you're verifying against the wrong app's JWKS. Confirm the JWKS URL uses the correct `client_id`. |
| `token expired` | Refresh the access token using the refresh token, or force a new login. |
| `aud mismatch` | The token was issued for a different application. |
| `iss mismatch` | The token's issuer doesn't match `AUTHAZ_IDENTITY_DOMAIN`. Check for a custom-domain mismatch. |
| `clock skew too large` | The server's clock is more than 60s off real time. Fix NTP. |

## Management API errors (returned as JSON with `error` and `error_description`)

| HTTP | `error` | Fix |
|---|---|---|
| 401 | `unauthenticated` | `X-API-Key` is missing, malformed, or revoked |
| 403 | `insufficient_scope` | The API key lacks the permission needed; grant it in the Dashboard or via `POST /api/v1/api-keys/{id}/permissions` |
| 403 | `cross_tenant_access` | You passed a `tenant_id` the API key isn't scoped to |
| 404 | `not_found` | Resource doesn't exist or the API key can't see it (tenant scope) |
| 409 | `conflict` | Duplicate (e.g., user with that email already exists) |
| 422 | `validation_failed` | Body shape doesn't match the schema; the response includes a `fields` array |

## SDK-specific errors

### `Authaz.Sdk` (.NET)

The SDK returns `AuthazResult<T>`, **not exceptions**. Always pattern-match:

```csharp
var result = await authaz.Users.GetAsync(userId);
if (!result.IsSuccess)
{
    // result.Error is a typed AuthazError (NotFound, Unauthorized, Forbidden,
    // Validation, RateLimited, …). Pattern-match on the subtype.
}
```

### `@authaz/next`, `@authaz/hono`, `@authaz/react`

Errors propagate as JSON `{ error, error_description }` from the auth handler routes. Wrap calls in `try/catch` and surface the `error` field to the user; never leak `error_description` (it can include internal hints).

## When to retry

Retry only on transient errors: `temporarily_unavailable`, HTTP 502/503/504, network timeouts. Use exponential backoff with jitter, max 3 attempts. Never retry `invalid_grant`, `invalid_client`, `validation_failed` — those are programming bugs.
