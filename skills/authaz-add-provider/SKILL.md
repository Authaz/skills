---
name: authaz-add-provider
description: Use when enabling or configuring an authentication provider on an Authaz application — password, Google / Microsoft / Apple / GitHub social login, magic link, passkey, SAML, or M2M client credentials. Drives the IdP-side console setup plus the Authaz-side toggle (CLI or Dashboard). Triggers on "add Google login", "enable magic link", "add OAuth provider", "M2M client credentials".
---

# Add or configure an authentication provider

Authaz applications can have multiple providers enabled at once. The hosted Sign-In picks them up automatically — you don't need to redeploy your app. You only edit the application config.

The fastest path is the CLI (`authaz oauth add`, `authaz mfa require`, etc.). Reach for the Dashboard when you need to paste a secret — **the CLI deliberately doesn't accept secrets on the command line**.

## Step 1 — Identify the provider

| Provider | What it gives the user | Notes |
|---|---|---|
| `password` | Email + password (with optional MFA) | Enabled by default on new apps |
| `google` | "Continue with Google" | Google Cloud OAuth client ID + secret |
| `microsoft` | "Continue with Microsoft" | Azure Entra ID app registration |
| `apple` | "Continue with Apple" | Apple Developer Sign In with Apple — uses a `.p8` private key, not a secret |
| `github` | "Continue with GitHub" | GitHub OAuth App with `read:user`, `user:email` |
| `magic_link` | Email-delivered one-time codes (passwordless) | No external IdP setup |
| `passkey` | WebAuthn / FIDO2 | Shipped — `authaz passkey enable` |
| `saml` | Enterprise SSO via SAML 2.0 IdP | Shipped — `authaz saml add/enable` |
| `m2m` | OAuth 2 client credentials grant | For service-to-service, not end users |

## Step 2 — IdP-side console setup (social providers only)

Every social IdP needs a redirect URI pointing back to Authaz. The path is **fixed** and verified in `authaz/Authaz.AuthServer/Auth/OAuthProvider/OAuthProviderRoutes.cs`:

```
https://auth.authaz.io/universal/auth/oauth-provider/{provider}/callback
```

(Substitute the customer's custom identity domain if they have one.)

| Provider | Where to set the redirect URI |
|---|---|
| **Google** | GCP → APIs & Services → Credentials → "OAuth 2.0 Client ID" → **Authorized redirect URIs**. Add `https://auth.authaz.io/universal/auth/oauth-provider/google/callback` |
| **Microsoft** | Azure Portal → Microsoft Entra ID → App registrations → Authentication → **Redirect URIs**. Add `https://auth.authaz.io/universal/auth/oauth-provider/microsoft/callback` |
| **Apple** | Apple Developer → Certificates, Identifiers & Profiles → Service ID → enable "Sign in with Apple" → **Return URLs**. Add `https://auth.authaz.io/universal/auth/oauth-provider/apple/callback`. Generate a `.p8` private key — upload to Authaz, **don't paste a "secret"** (Apple uses asymmetric auth) |
| **GitHub** | GitHub → Settings → Developer settings → OAuth Apps → **Authorization callback URL**. Add `https://auth.authaz.io/universal/auth/oauth-provider/github/callback` |

Capture the IdP's `client_id` and `client_secret` (or the Apple key). You'll paste these into Authaz next.

## Step 3 — Enable on Authaz

### CLI (recommended for repeatable setups)

```bash
authaz oauth add --provider google --client-id 12345.apps.googleusercontent.com --scopes "openid,email,profile"
authaz oauth enable --provider google

# Other providers:
authaz magic-link enable
authaz magic-link set --link-ttl-minutes 15

authaz passkey enable
authaz passkey set --rp-id auth.your-app.com

authaz saml add --connection-id okta --idp-metadata-url https://...
authaz saml enable --connection-id okta

authaz m2m enable
authaz m2m set --allowed-audiences "https://api.your-app.com"

authaz mfa require                              # password-provider MFA
authaz password-policy set --policy strong
```

The CLI applies the change with diff + confirmation (use `--yes` in CI, `--dry-run` to preview). See `authaz-cli` for the full reference.

**The CLI does not take `--client-secret`.** It prints: "client secret must be set separately via the dashboard." Open the Dashboard after the `oauth add` to paste the secret.

### Dashboard

Dashboard → your application → **Authentication → Providers** → toggle the provider → paste the IdP credentials (and the Apple `.p8` file if applicable). Sign-In updates immediately.

### Management API (when you need to script across many apps)

The Management API surface for auth providers lives at:

- **App-level config** (list + email/password + magic-link + passkey): `/applications/{appId}/config/{providers,email-password,magic-link/auth,passkey/auth}`
- **Per-tenant OAuth providers**: `/applications/{appId}/tenants/{tenantId}/auth-config/oauth/{providerName}`

OAuth providers are configured **per-tenant** in the API — the CLI hides this by operating on the default tenant. If you're scripting multi-tenant per-provider config, use the SDK (`authaz.tenants.*` etc.) rather than raw HTTP. The exact request shapes are not stable enough to inline here — read the SDK source (`authaz-sdk-dotnet/Authaz.Sdk/src/Resources/`) for the current contract.

## Step 4 — Verify

1. Open Authaz Sign-In for your application — the new provider's button appears.
2. Click it; complete the IdP flow; land back on your app signed in.
3. Decode the access token — `sub` is the Authaz user ID (not the upstream IdP's). Authaz reconciles social sign-ins to a single user record by verified email.

For **M2M**: hit the token endpoint with the issued credentials:

```bash
curl -X POST https://auth.authaz.io/universal/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=…" -d "client_secret=…" -d "scope=events:write"
```

You should get an `access_token` back.

## Anti-patterns

- **Don't enable a provider before setting up the IdP redirect URI.** Authaz will redirect to the IdP, and the IdP will reject it. The flow looks broken when it's just unconfigured.
- **Don't try to pass a secret on the CLI.** The CLI explicitly refuses — paste it in the Dashboard after `oauth add`.
- **Don't ship M2M client secrets in source code.** Treat them like API keys.
- **Don't enable Apple Sign In casually** — Apple requires a $99/year developer account.
- **Don't disable password without confirming users have an alternative** — locks out anyone whose only login was email/password.
- **Don't paraphrase the Authaz callback URL.** The fixed shape is `/universal/auth/oauth-provider/{provider}/callback` — verify against `authaz/Authaz.AuthServer/Auth/OAuthProvider/OAuthProviderRoutes.cs` if unsure.

## References

- Real callback path: `authaz/Authaz.AuthServer/Auth/OAuthProvider/OAuthProviderRoutes.cs`
- CLI commands: `authaz-cli` (`oauth`, `magic-link`, `passkey`, `saml`, `m2m`, `mfa`, `password-policy`)
- `references/endpoints.md`, `references/error-codes.md`
- `authaz-troubleshoot-oauth` for redirect_uri / token failures
