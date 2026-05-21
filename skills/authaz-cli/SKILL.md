---
name: authaz-cli
description: Use when configuring an Authaz application with the `authaz` CLI — login, change auth providers / branding / MFA / tenancy / password policy, or apply declarative YAML config. Replaces dashboard clicking for repeatable setup. Triggers on "authaz cli", "authaz login", "authaz apply", "authaz oauth add", "authaz mfa", "declarative authaz".
---

# Use the `authaz` CLI

`authaz` is a .NET global tool for configuring Authaz applications from the command line — and through declarative YAML for repeatable / version-controlled setups. Code is grounded in `authaz/Authaz.Cli/`.

The CLI authenticates **per application** (device-code flow against the app's `client_id`). It doesn't use API keys for human use — but CI can shortcut with `AUTHAZ_API_TOKEN`. Credentials live encrypted at `~/.authaz/credentials.json`.

## Step 1 — Install

The CLI ships as a .NET global tool (`PackAsTool=true`, `ToolCommandName=authaz`):

```bash
dotnet tool install -g Authaz.Cli
authaz --help
```

Requires .NET 10 SDK. If installing from source while the package is pre-release:

```bash
cd authaz/Authaz.Cli
dotnet pack -c Release
dotnet tool install -g --add-source ./bin/Release Authaz.Cli
```

## Step 2 — Authenticate

### Interactive (developer laptops)

```bash
authaz login --app-id 00000000-0000-0000-0000-000000000000 --app my-app
```

`--app-id` (the application's UUID, from the Dashboard) is **required**. `--app` is a local label you choose for the credential slot — pick whatever's memorable (e.g. `my-app-staging`, `my-app-prod`).

The CLI:

1. Requests a device code from `<api-url>/universal/...`.
2. Opens your browser to the verification URL; you confirm the user code shown.
3. Polls until you approve, then stores the token under the `--app` label and marks it as the **default app**.

Subsequent commands use the default app unless you pass `--app <label>`.

### Non-interactive (CI)

Set `AUTHAZ_API_TOKEN` to a bearer token (e.g., a long-lived M2M access token):

```bash
export AUTHAZ_API_TOKEN=...
authaz apply --file app.yaml --yes
```

The CLI skips the credential store entirely when this env var is set.

### Check & clear

```bash
authaz whoami            # table of authenticated apps with expiry
authaz logout --app my-app   # remove one app
authaz logout            # remove all credentials
```

## Step 3 — Global flags (work on every command)

| Flag | Env var | Default | Purpose |
|---|---|---|---|
| `--api-url` | `AUTHAZ_API_URL` | `https://api.authaz.io` | API base URL — override for self-hosted |
| `--app` | — | the default app (set by `login`) | Which stored credential to use |
| `--identity-prefix` | `AUTHAZ_IDENTITY_PREFIX` | `/universal` | Identity service path prefix. Empty string for host-based routing |

## Step 4 — Declarative YAML (`apply` / `validate` / `export`)

This is the recommended path for production-like environments — diffable, reviewable, replayable.

### Export current configuration

```bash
authaz export                   # to stdout
authaz export --output app.yaml # to file
```

The exported YAML carries a `metadata.etag` field used by the next `apply` for optimistic concurrency.

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

- `--dry-run` — show the diff, don't apply
- `--yes` — skip the confirmation prompt (CI)
- `--force` — skip the ETag check (overwrite)

```bash
authaz apply --file app.yaml --dry-run            # preview only
authaz apply --file app.yaml --yes                # CI
authaz apply --file app.yaml --force --yes        # overwrite an outdated ETag
```

**Multi-document YAML (`---` separators) is not supported.** Split into separate files.

### Authorization YAML

If the YAML's `kind` is `Authorization` (roles, policies, permissions), `apply` validates server-side first and emits a per-change summary. Same flags apply (`--dry-run`, `--yes`).

## Step 5 — Imperative commands (no YAML needed)

Each subcommand patches a single section and runs the same diff/confirm/apply flow. All accept `--yes`, `--dry-run`, `--force`. All require an authenticated app context (via `--app` or `login`).

### Application metadata

```bash
authaz app rename --name "Acme Production"
authaz app set-description --description "Production tenant"
```

### Tenancy

```bash
authaz tenancy set --type single_tenant
authaz tenancy set --type multi_tenant --mode shared --require-tenant-hint true
```

Values:

- `--type`: `single_tenant` or `multi_tenant`
- `--mode` (multi-tenant only): `shared` or `isolated`
- `--require-tenant-hint`: `true`/`false`

### OAuth providers

```bash
authaz oauth list
authaz oauth add --provider google --client-id 12345.apps.googleusercontent.com --scopes "openid,email,profile"
authaz oauth enable --provider google
authaz oauth disable --provider google
authaz oauth remove --provider google
```

**Supported providers**: `google`, `apple`, `microsoft`, `github`. (No facebook/twitter/discord — don't make those up.)

**Client secrets are NOT taken on the command line.** The CLI prints: "client secret must be set separately via the dashboard." Add the secret in the Dashboard after `oauth add`.

### Magic link

```bash
authaz magic-link enable
authaz magic-link set --link-ttl-minutes 15 --link-length 32
authaz magic-link disable
```

### Passkey (WebAuthn)

```bash
authaz passkey enable
authaz passkey set --rp-id auth.your-app.com
authaz passkey disable
```

### M2M (machine-to-machine)

```bash
authaz m2m enable
authaz m2m set --allowed-audiences "https://api.your-app.com"
authaz m2m disable
```

### SAML

```bash
authaz saml list
authaz saml add --connection-id okta --idp-metadata-url https://...
authaz saml enable --connection-id okta
authaz saml disable --connection-id okta
authaz saml remove --connection-id okta
```

### MFA

```bash
authaz mfa require                          # enforce for all users
authaz mfa require --grace-period-days 7    # enforce after grace
authaz mfa optional
authaz mfa disable
authaz mfa set-trusted-device-days 30
```

### Password policy

```bash
authaz password-policy show
authaz password-policy set --policy strong       # preset
authaz password-policy set --policy moderate     # preset
authaz password-policy set \
  --min-length 14 \
  --require-uppercase --require-lowercase --require-number --require-special \
  --reject-breached
```

Presets: `strong`, `moderate`, `custom`. Individual flags override preset values.

### Signup

```bash
authaz signup enable
authaz signup set --allow-public-signup false --allowed-email-domains "your-app.com,acme.com"
authaz signup disable
```

### Session policy

```bash
authaz session set --idle-timeout-minutes 30 --absolute-timeout-hours 24
```

### Invitations

```bash
authaz invitation enable
authaz invitation set --ttl-days 14
authaz invitation disable
```

### Branding

```bash
authaz branding apply-preset minimal              # apply a built-in preset
authaz branding set \
  --button-color #1F2937 \
  --page-background #FFFFFF \
  --link-color #1569A8 \
  --font Inter \
  --logo https://your-app.com/logo.svg
```

Color flags must be 6-char hex (with or without `#`). At least one flag is required. **Note:** the flags are `--button-color` / `--page-background` / `--link-color`, **not** `--primary-color`.

### Custom domains

```bash
authaz domain list
authaz domain add --domain auth.your-app.com
authaz domain set-primary --domain auth.your-app.com
authaz domain remove --domain auth.your-app.com
```

## Step 6 — Common flows

### Promote a config from staging to production

```bash
authaz login --app-id <staging-uuid> --app my-app-staging
authaz export --app my-app-staging > app.yaml

authaz login --app-id <prod-uuid> --app my-app-prod
authaz apply --file app.yaml --app my-app-prod --dry-run   # review diff
authaz apply --file app.yaml --app my-app-prod --yes
```

Strip `metadata.etag` from `app.yaml` before applying to a different application — the staging ETag won't match prod. Or use `--force`.

### Bulk-tweak a fleet of apps in CI

```yaml
# .github/workflows/authaz-apply.yml (sketch)
env:
  AUTHAZ_API_TOKEN: ${{ secrets.AUTHAZ_API_TOKEN }}

run: |
  for app in apps/*.yaml; do
    authaz apply --file "$app" --yes
  done
```

## Step 7 — Verify

After every change:

```bash
authaz export --output current.yaml      # snapshot
diff app.yaml current.yaml               # confirm only what you wanted changed
```

Then open the hosted Sign-In page for the application — the providers, branding, and tenancy should match.

## Anti-patterns

- **Don't commit `metadata.etag` between environments.** It's per-app concurrency state, not config. Strip it or use `--force` when copying.
- **Don't pass `--force` blindly in CI.** It defeats the optimistic-concurrency check. Use it only when you understand the prior state.
- **Don't use `authaz` to manage users, roles, or tenants — it's app-config only.** Those operations live in the Management API (`Authaz.Sdk` or `@authaz/sdk`). See `authaz-management-api`.
- **Don't paste `client_secret` on the CLI.** OAuth secrets are set via the Dashboard; the CLI explicitly does not accept them.
- **Don't run interactively from a prod laptop without `--dry-run` first.** Apply diffs are loud for a reason — read them.
- **Don't assume an env-var interpolation syntax in YAML.** The CLI does not implement `${env:VAR}` substitution — use a templating step in CI if you need secrets out-of-band.

## Source of truth

- CLI entry: `authaz/Authaz.Cli/Program.cs` — every registered command and branch
- Auth: `authaz/Authaz.Cli/Commands/Auth/`, `authaz/Authaz.Cli/Auth/CredentialStore.cs`
- YAML apply: `authaz/Authaz.Cli/Commands/ApplyCommand.cs`
- Per-section commands: `authaz/Authaz.Cli/Commands/{OAuth,Mfa,Branding,PasswordPolicy,…}/`
- Project metadata: `authaz/Authaz.Cli/Authaz.Cli.csproj` (version, target framework, tool packaging)

## References

- `authaz-add-provider` — provider-specific dashboard / IdP setup (the parts the CLI can't do, like uploading Apple's `.p8` key)
- `authaz-management-api` — for users, roles, tenants (not app config)
- `authaz-multi-tenant` — when configuring `tenancy set --type multi_tenant`
- `references/glossary.md`
