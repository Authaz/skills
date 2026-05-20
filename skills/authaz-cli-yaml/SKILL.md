---
name: authaz-cli-yaml
description: Use when configuring Authaz declaratively via the `authaz` CLI — apply a YAML file, validate before push, export current state. Best for repeatable environments (staging vs prod), CI pipelines, or "infra as code" workflows. Triggers on "authaz cli", "authaz apply", "yaml config authaz", "declarative authaz".
---

# Configure Authaz with the CLI

The `authaz` CLI lets you describe an application's configuration as YAML and apply it like Terraform or `kubectl`. Use this when you have multiple environments, want changes reviewable, or want to recreate an app from scratch.

## Step 1 — Install and authenticate

The CLI ships in the Authaz repo as `Authaz.Cli/`. Install per the project's instructions, then:

```bash
authaz login            # device-code flow against your identity domain
authaz whoami           # confirm you're authenticated
```

For CI, use a long-lived API key instead of device-code:

```bash
export AUTHAZ_API_KEY=sk_live_…
authaz whoami           # picks up the env var
```

## Step 2 — Author the YAML

Start by exporting an existing application:

```bash
authaz export --application-id app_01abc… > my-app.yaml
```

Or hand-author. Minimal shape (single-tenant, password + Google):

```yaml
application:
  name: my-app
  tenancy_type: single_tenant
auth_providers:
  password:
    enabled: true
    min_length: 12
    require_mfa: false
  google:
    enabled: true
    client_id: ${env:GOOGLE_CLIENT_ID}
    client_secret: ${env:GOOGLE_CLIENT_SECRET}
auth_flow:
  allowed_callback_urls:
    - https://my-app.example.com/auth/callback
    - http://localhost:3000/auth/callback
  allowed_logout_redirect_urls:
    - https://my-app.example.com
branding:
  primary_color: "#1F2937"
password_policy:
  min_length: 12
  require_uppercase: true
  require_number: true
mfa:
  required: false
  trusted_device_days: 30
signup:
  enabled: true
  require_email_verification: true
```

`${env:VARNAME}` interpolates from the environment at apply time — never inline secrets in checked-in YAML.

## Step 3 — Validate before applying

```bash
authaz validate my-app.yaml
```

Validation runs locally — it catches schema errors and references to providers/permissions that don't exist. Run this in CI as a required check before merge.

## Step 4 — Apply

```bash
authaz apply my-app.yaml --application-id app_01abc…
```

The CLI shows a diff (current → target) and asks for confirmation. In CI, pass `--yes` to skip the prompt.

For a fresh environment with no existing app:

```bash
authaz apply my-app.yaml --create
```

This calls `POST /api/v1/applications` first, then applies the rest.

## Step 5 — Imperative shortcuts

For one-off tweaks without authoring YAML:

```bash
authaz oauth add --provider google --client-id … --client-secret …
authaz oauth enable --provider google
authaz oauth list

authaz mfa require        # password provider
authaz mfa optional
authaz mfa set-trusted-device-days 30

authaz password-policy set --min-length 14 --require-uppercase

authaz branding set --primary-color "#1F2937"
authaz branding apply-preset minimal

authaz signup enable
authaz signup set --require-email-verification
```

These mutate the live config the same way `apply` does — useful for fast iteration, less reproducible than YAML.

## Step 6 — Verify

After every apply:

1. `authaz export --application-id app_01abc…` — should match the YAML you applied.
2. Open Authaz Sign-In for the app; confirm the providers and branding match.
3. Run a smoke test: hit `/universal/auth/providers` and confirm the enabled list.

## Anti-patterns

- **Don't commit `client_secret` values into YAML.** Use `${env:…}` interpolation and hold the secrets in CI variables / a secrets manager.
- **Don't apply without `validate` first.** A typo in a provider name silently disables that provider.
- **Don't mix CLI imperative commands with YAML apply on the same env in the same hour.** They both work, but interleaving creates "what's the source of truth" confusion. Pick one as primary.
- **Don't try to use `authaz apply` to migrate users or create roles.** It's app-config only — users and role assignments live in the Management API. See `authaz-management-api`.
- **Don't run `authaz apply` from a developer laptop on prod.** Run it from CI with audited credentials.

## References

- Management API overview: <https://authaz.io/docs/management-api/overview>
- `authaz-add-provider` for imperative provider setup
- `authaz-management-api` for user/role/tenant operations
- `references/endpoints.md`
