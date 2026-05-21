---
name: authaz-add-provider
description: Use when enabling or configuring an authentication provider on an Authaz application — password, Google / Microsoft / Apple / GitHub social login, magic link, or M2M client credentials. Triggers on "add Google login", "enable magic link", "add OAuth provider", "M2M client credentials".
---

# Add or configure an authentication provider

Authaz applications can have multiple providers enabled at once. The hosted Sign-In picks them up automatically — you don't need to redeploy your app. You only edit the application config.

## Step 1 — Identify the provider

| Provider | What it gives the user | Notes |
|---|---|---|
| `password` | Email + password (with optional MFA) | Enabled by default on new apps |
| `google` | "Continue with Google" | Needs a Google Cloud OAuth client ID + secret |
| `microsoft` | "Continue with Microsoft" | Same shape — Azure AD app registration |
| `apple` | "Continue with Apple" | Apple Developer Sign In with Apple — uses a private key, not a secret |
| `github` | "Continue with GitHub" | GitHub OAuth App — requires `read:user` and `user:email` |
| `magic_link` | Email-delivered one-time codes (passwordless) | No external IdP setup |
| `passkey` | WebAuthn / FIDO2 | **Coming soon** — not yet generally available |
| `saml` | Enterprise SSO via SAML 2.0 IdP | **Coming soon** — not yet generally available |
| `m2m` | OAuth 2 client credentials grant | For service-to-service, not end users |

If the user asks for `passkey` or `saml`, say it's not yet shipping and offer the closest available substitute (password+MFA for high-assurance, or pre-issued M2M credentials for service auth).

## Step 2 — Get IdP credentials (for social providers)

Each social provider has its own developer console flow. The Authaz-side callback lives at `https://auth.authaz.io/universal/oauth/callback/{provider}` (or your custom identity domain if configured).

- **Google**: GCP → APIs & Services → Credentials → "OAuth 2.0 Client ID". Authorized redirect URI: `https://auth.authaz.io/universal/oauth/callback/google`.
- **Microsoft**: Azure Portal → Microsoft Entra ID → App registrations → Authentication → add `https://auth.authaz.io/universal/oauth/callback/microsoft` as a redirect URI.
- **Apple**: Apple Developer → Certificates, Identifiers & Profiles → Service ID → enable "Sign in with Apple" → return URL `https://auth.authaz.io/universal/oauth/callback/apple`. You upload a `.p8` private key, not a secret.
- **GitHub**: GitHub → Settings → Developer settings → OAuth Apps → callback URL `https://auth.authaz.io/universal/oauth/callback/github`.

The exact callback path on the Authaz side is fixed — don't paraphrase it. Verify against `references/endpoints.md` if unsure.

## Step 3 — Enable the provider

### Option A — Dashboard (recommended for first-time setup)

Authentication → Providers → toggle the provider → paste the IdP credentials. The Sign-In screen updates immediately.

### Option B — Management API

Password (toggle the policy fields):

```http
PATCH https://api.authaz.io/api/v1/applications/{appId}/auth-providers/password
X-API-Key: sk_live_…

{ "enabled": true, "min_length": 12, "require_mfa": false }
```

Social (Google example):

```http
PUT https://api.authaz.io/api/v1/applications/{appId}/auth-providers/google
X-API-Key: sk_live_…

{ "enabled": true, "client_id": "…", "client_secret": "…" }
```

Magic link:

```http
PATCH https://api.authaz.io/api/v1/applications/{appId}/auth-providers/magic-link
X-API-Key: sk_live_…

{ "enabled": true }
```

M2M (issue a service credential):

```http
POST https://api.authaz.io/api/v1/applications/{appId}/m2m-credentials
X-API-Key: sk_live_…

{ "name": "ingest-worker", "permissions": ["events:write"] }
```

The response includes `client_id` and `client_secret` once — store them in the calling service's secret manager.

### Option C — `authaz` CLI (declarative)

```bash
authaz oauth add --provider google --client-id ... --client-secret ...
authaz oauth enable --provider google
authaz mfa require    # password provider only
```

Or apply declarative YAML — see `authaz-cli`.

## Step 4 — Verify

1. Open Authaz Sign-In for your application — the new provider's button appears.
2. Click it; complete the IdP flow; land back on your app signed in.
3. Decode the access token — `sub` is the Authaz user ID (not the upstream IdP's). Authaz reconciles social sign-ins to a single user record by verified email.

For M2M: hit the token endpoint with the issued credentials:

```bash
curl -X POST https://auth.authaz.io/universal/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=…" -d "client_secret=…" -d "scope=events:write"
```

You should get an `access_token` back.

## Anti-patterns

- **Don't enable a provider without setting up the IdP redirect URI first.** Authaz will redirect to the IdP, and the IdP will reject it. The flow looks broken when it's just unconfigured.
- **Don't ship M2M client secrets in source code.** Treat them like API keys.
- **Don't enable Apple Sign In just because it exists** — Apple charges $99/year for the developer membership it requires.
- **Don't disable password without confirming users have an alternative** — locks out anyone whose only login was email/password.
- **Don't try to enable `passkey` or `saml` yet** — they are documented but not generally available.

## References

- Authentication overview: <https://authaz.io/docs/authentication/overview>
- Per-provider docs: `https://authaz.io/docs/authentication/{password,oauth2,social-login,magic-link,m2m,api-keys}`
- `references/endpoints.md`, `references/error-codes.md`
- `authaz-cli` for declarative config and per-section commands
