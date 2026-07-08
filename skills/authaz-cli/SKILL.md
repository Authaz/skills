---
name: authaz-cli
description: Use when configuring an Authaz application with the `authaz` CLI — login, manage organizations / API keys / credentials, or apply declarative YAML config for auth providers / branding / MFA / tenancy / password policy. Replaces dashboard clicking for repeatable setup. Triggers on "authaz cli", "authaz login", "authaz apply", "declarative authaz".
---

# Use the `authaz` CLI

> **Last verified:** `Authaz.Cli` v0.4.2 (2026-07-02), installed from nuget.org. The CLI is pre-1.0 — commands and flags can change. If `--help` shows different commands/flags than below, trust the CLI output and report the drift.

`authaz` is a .NET global tool for configuring Authaz applications from the command line — and through declarative YAML for repeatable / version-controlled setups. Code is grounded in the standalone repo `authaz-cli/` (GitHub: `Authaz/cli`), extracted from the old `authaz` monorepo. Project lives at `authaz-cli/Authaz.Cli/`.

The CLI authenticates via `authaz login`, which opens a dashboard-driven login flow and stores credentials under a profile (default `default`). Top-level commands are: `login`, `logout`, `whoami`, `validate`, `apply`, `export`, `org`, `apikey`, `credential`.

## Step 1 — Install

The CLI ships as a .NET global tool (`PackAsTool=true`, `ToolCommandName=authaz`). **Requires the .NET 10 SDK.**

The package is published on nuget.org (current version 0.4.2). Install directly:

```bash
dotnet tool install --global Authaz.Cli
authaz --help
```

To upgrade: `dotnet tool update --global Authaz.Cli`.

## Step 2 — Authenticate

```bash
authaz login                          # opens the dashboard-driven login flow
authaz login --profile my-profile     # store credentials under a named profile
```

Options: `--dashboard-url`, `--profile`, `--identity-prefix`. Login is account-level, scoped by `--profile` — not per-application (no `--app-id`/`--app` flag).

### Check & clear

```bash
authaz whoami             # shows the current identity and organization
authaz logout             # removes the resolved profile's credentials
authaz logout --all       # removes every local profile
```

### Switch organization

```bash
authaz org list
authaz org switch <org-id>
```

If the identity belongs to more than one organization, every org-scoped command (`apply`, `credential create`, `apikey create`, etc.) silently targets whichever org is currently active — there's no confirmation prompt. Run `authaz org list` and `authaz org switch` (or check `authaz whoami`'s "Current organization") before creating or modifying anything, rather than assuming the active org is the right one.

## Step 3 — Global flags (work on every command)

| Flag | Default | Purpose |
|---|---|---|
| `--dashboard-url` | `https://dashboard.authaz.io` | Dashboard base URL — override for self-hosted |
| `--profile` | `default` | Which stored credential profile to use |
| `--identity-prefix` | `/api` | Identity service path prefix. Empty string for host-based routing |

## Step 4 — Declarative YAML (`apply` / `validate` / `export`)

This is the recommended path for production-like environments — diffable, reviewable, replayable.

### Export current configuration

```bash
authaz export                   # to stdout
authaz export --output app.yaml # to file
```

The exported YAML carries a `metadata.etag` field used by the next `apply` for optimistic concurrency.

### Application YAML reference

Field tree, derived from real `export`/`apply` round-trips (single-tenant and multi-tenant). "Required" describes the resolved schema — every application has a value for these fields. On `apply` to a **new** application, omitted fields get sane server-side defaults, so a short starter YAML (see `authaz-signup` Step 3) works fine; `spec.authentication.settings.enabled` is the one exception with no default (see the gotcha row below), so always set it explicitly.

| Field | Type | Required | Notes |
|---|---|---|---|
| `apiVersion` | string | required | `authaz/v1` |
| `kind` | string | required | `Application` |
| `metadata.id` | uuid | required for `apply` to an existing app | omit to create new |
| `metadata.name` | string | required | |
| `metadata.etag` | string | optional | optimistic-concurrency token from `export`; omit or use `--force` when applying to a different app |
| `spec.tenancy.type` | `single_tenant` \| `multi_tenant` | required | |
| `spec.tenancy.mode` | `shared` \| `isolated` | multi-tenant only | see `authaz-multi-tenant` |
| `spec.tenancy.requireTenantHint` | bool | required | |
| `spec.authentication.providers.emailPassword.enabled` | bool | required | |
| `spec.authentication.providers.emailPassword.{minLength,maxLength}` | int | required | |
| `spec.authentication.providers.emailPassword.{requireUppercase,requireLowercase,requireNumber,requireSpecial,rejectBreached}` | bool | required | |
| `spec.authentication.providers.emailPassword.historyCount` | int | required | |
| `spec.authentication.providers.emailPassword.lockout.{maxAttempts,durationMinutes}` | int | optional | omit to disable lockout |
| `spec.authentication.signup.enabled` | bool | required | |
| `spec.authentication.signup.autoCreateTenant` | bool | required | |
| `spec.authentication.signup.requireTermsAcceptance` | bool | required | |
| `spec.authentication.invitations.*` | block | **optional** | only present if the invitations feature is used — omit entirely otherwise. See `authaz-add-provider` |
| `spec.authentication.session.{timeoutMinutes,idleTimeoutMinutes,absoluteTimeoutMinutes,persistentSessionDays,maxConcurrentSessions}` | int | required | |
| `spec.authentication.session.{allowConcurrentSessions,requireReauthForSensitive}` | bool | required | |
| `spec.authentication.settings.redirectUris` | string[] | required | |
| `spec.authentication.settings.allowedScopes` | string[] | required | |
| `spec.authentication.settings.{accessTokenLifetime,refreshTokenLifetime,authorizationCodeLifetime,rememberMeTimeoutMinutes}` | int (seconds/minutes) | required | |
| `spec.authentication.settings.enabled` | bool | **required — no default-true** | **Live-verified gotcha:** omitting this field applies the app with authentication `enabled: false`. Always set it explicitly to `true` in any minimal/starter YAML. |

Minimal-but-complete, copy-pasteable (single-tenant, no invitations):

```yaml
apiVersion: authaz/v1
kind: Application
metadata:
  name: my-app
spec:
  tenancy:
    type: single_tenant
    requireTenantHint: false
  authentication:
    providers:
      emailPassword:
        enabled: true
        minLength: 8
        maxLength: 128
        requireUppercase: false
        requireLowercase: false
        requireNumber: false
        requireSpecial: false
        rejectBreached: false
        historyCount: 0
    signup:
      enabled: true
      autoCreateTenant: false
      requireTermsAcceptance: false
    session:
      timeoutMinutes: 1440
      idleTimeoutMinutes: 30
      absoluteTimeoutMinutes: 10080
      persistentSessionDays: 15
      allowConcurrentSessions: true
      maxConcurrentSessions: 5
      requireReauthForSensitive: true
    settings:
      redirectUris:
      - http://localhost:3000/auth/callback
      allowedScopes:
      - openid
      - profile
      - email
      accessTokenLifetime: 3600
      refreshTokenLifetime: 2592000
      authorizationCodeLifetime: 600
      rememberMeTimeoutMinutes: 43200
      enabled: true   # required — see gotcha above
```

For multi-tenant, add `spec.tenancy.mode` (`shared` or `isolated`) alongside `spec.tenancy.type`; for invitations, add a `spec.authentication.invitations` block (`enabled`, `expiryHours`, `maxPerUser`, `requireApproval`, `allowRoleAssignment`, `allowedRoles`, `sendEmailNotification`, `defaultRole`).

### Validate locally before applying

```bash
authaz validate --file app.yaml
```

Schema-only check; runs offline.

### Apply — with diff + confirmation

```bash
authaz apply --file app.yaml
```

The CLI prints a diff (server state → file state), asks you to confirm, then writes. ETag mismatch (someone else changed the app since you exported) fails with `conflict`.

Useful flags:

- `--application-id <ID>` — target application; if omitted, creates new or upserts by `metadata.id`
- `--yes` — skip the confirmation prompt (CI)
- `--force` — skip the ETag check (overwrite)

```bash
authaz apply --file app.yaml --application-id <id>                # apply to a specific app
authaz apply --file app.yaml --application-id <id> --yes           # CI
authaz apply --file app.yaml --application-id <id> --force --yes   # overwrite an outdated ETag
```

There is no `--dry-run` flag.

**Multi-document YAML (`---` separators) is not supported.** Split into separate files.

### Authorization YAML

If the YAML's `kind` is `Authorization` (roles, policies, permissions), `apply` validates server-side first and emits a per-change summary. Same flags apply (`--yes`, `--force`).

The concrete field shape hasn't been captured from a live example yet — `authaz export` (CLI v0.4.2, live-verified) only emits `kind: Application`, and no `kind: Authorization` schema strings exist in the installed CLI binary. If you need the real shape, check whether a newer CLI version's `authaz export` can target roles/policies directly, or use the Management API's `Authorization` resource (`.Roles`, `.Permissions`, `.Policies` — see `authaz-management-api`) instead of guessing at YAML fields.

## Step 5 — Other commands (no YAML needed)

There are no per-feature imperative subcommands (no `app`, `tenancy`, `oauth`, `magic-link`, `passkey`, `m2m`, `saml`, `mfa`, `password-policy`, `signup`, `session`, `invitation`, `branding`, or `domain` commands exist). All feature-level config (providers, branding, tenancy, MFA, password policy, etc.) goes through the YAML `export` → edit → `validate` → `apply` flow above.

The only imperative (non-YAML) commands are:

### Organization

```bash
authaz org list
authaz org switch <org-id>
```

### API keys

```bash
authaz apikey create
authaz apikey list
authaz apikey revoke <key-id>
```

### Credentials

```bash
authaz credential create
authaz credential list
authaz credential rotate <credential-id>
authaz credential revoke <credential-id>
```

## Step 6 — Common flows

### Promote a config from staging to production

```bash
authaz login --profile staging
authaz export --profile staging --output app.yaml

authaz login --profile prod
authaz apply --file app.yaml --application-id <prod-app-id> --profile prod
```

Strip `metadata.etag` from `app.yaml` before applying to a different application — the staging ETag won't match prod. Or use `--force`.

### Bulk-tweak a fleet of apps in CI

```yaml
# .github/workflows/authaz-apply.yml (sketch)
run: |
  for app in apps/*.yaml; do
    authaz apply --file "$app" --application-id "$(yq '.metadata.id' "$app")" --yes
  done
```

## Step 7 — Verify

After every change:

```bash
authaz export --application-id <id> --output current.yaml   # snapshot
diff app.yaml current.yaml                                    # confirm only what you wanted changed
```

Then open the hosted Sign-In page for the application — the providers, branding, and tenancy should match.

## Anti-patterns

- **Don't commit `metadata.etag` between environments.** It's per-app concurrency state, not config. Strip it or use `--force` when copying.
- **Don't pass `--force` blindly in CI.** It defeats the optimistic-concurrency check. Use it only when you understand the prior state.
- **Don't use `authaz` to manage users, roles, or tenants — it's app-config only.** Those operations live in the Management API (`Authaz.Sdk` or `@authaz/sdk`). See `authaz-management-api`.
- **Don't paste `client_secret` on the CLI.** OAuth secrets are set via the Dashboard; the CLI has no field for them.
- **Don't invent per-feature subcommands (`oauth`, `mfa`, `branding`, etc.).** They don't exist — everything feature-level is YAML.
- **Don't assume an env-var interpolation syntax in YAML.** The CLI does not implement `${env:VAR}` substitution — use a templating step in CI if you need secrets out-of-band.

## Source of truth

The live command surface is the authority — when in doubt, trust it over this skill:

```bash
authaz --help              # top-level commands
authaz <command> --help    # flags for a branch, e.g. `authaz apply --help`
```

If the CLI ever disagrees with this skill, the CLI wins — report the drift. The code lives in the `authaz-cli` repo (GitHub: `Authaz/cli`) if you need to dig deeper.

## References

- `authaz-add-provider` — provider-specific dashboard / IdP setup (the parts the CLI can't do, like uploading Apple's `.p8` key)
- `authaz-management-api` — for users, roles, tenants (not app config)
- `authaz-multi-tenant` — when configuring multi-tenant apps via YAML
- `references/glossary.md`
