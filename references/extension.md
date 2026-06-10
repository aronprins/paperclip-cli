# Secrets, Cloud Sync, Plugins

Load this file when managing credentials (`paperclipai secrets …`), pushing a local company to a Paperclip Cloud stack (`paperclipai cloud …`), or installing/developing/operating plugins (`paperclipai plugin …`).
All commands accept the common client flags (`--data-dir`, `--api-base`, `--api-key`, `--context`, `--profile`, `--json`) — see auth-and-context.md. Only the extra flags are listed below.

---

## Secrets (`paperclipai secrets <sub>`)

Hard guarantee: the CLI never prints secret plaintext. `list`, `usage`, `access-events`, and even `create`/`rotate` output return metadata only.

### List / inspect

```
paperclipai secrets list -C <company-id>
```
Lists secret metadata for a company: `id`, `name`, `key`, `provider`, `status`, `managedMode`, `latestVersion`, externalRef presence. `--json` returns the raw `CompanySecret[]` array.

```
paperclipai secrets declarations -C <company-id> [--include <set>] [--kind <kind>]
```
Shows the portable env declarations a company export would require on a target instance (run before handing someone an export). Rows: `key`, `scope` (`company` | `project:<slug>` | `agent:<slug>`), `kind` (`secret`|`plain`), `requirement`, `portability`, `hasDefault`, `description`.

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id` | id | yes | — | Company scope |
| `--include` | csv | no | `company,agents,projects` | Export preview include set: `company,agents,projects,issues,tasks,skills` (`tasks` aliases `issues`) |
| `--kind` | enum | no | `all` | Filter: `all` \| `secret` \| `plain` |

```sh
paperclipai secrets declarations -C cmp_123 --include agents,projects --kind secret --json
```

### Create (Paperclip-managed value)

```
paperclipai secrets create -C <company-id> --name <name> (--value <v> | --value-env <ENV>) [--key <key>] [--provider <id>] [--description <text>]
```

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id` | id | yes | — | Company scope |
| `--name` | string | yes | — | Display name |
| `--value` | string | one of | — | Plaintext value (lands in shell history — avoid) |
| `--value-env` | env name | one of | — | Read value from named env var; errors if empty/unset |
| `--key` | string | no | — | Portable secret key other resources reference |
| `--provider` | id | no | company default (managed local encryption) | Secret provider id |
| `--description` | string | no | — | Free text |

Exactly one of `--value`/`--value-env`; both or neither errors. Prefer `--value-env`:

```sh
export ANTHROPIC_API_KEY=sk-ant-...
paperclipai secrets create -C cmp_123 --name anthropic-api-key --value-env ANTHROPIC_API_KEY
```

`--json` returns the created `CompanySecret` record (metadata, no value).

### Link (external vault, value never copied)

```
paperclipai secrets link -C <company-id> --name <name> --provider <id> --external-ref <ref> [--key <key>] [--provider-version-ref <ref>] [--description <text>]
```
Registers a pointer to a provider-owned secret (stored with `managedMode: external_reference`; `secrets list` shows `externalRef=yes`).

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id` | id | yes | — | Company scope |
| `--name` | string | yes | — | Display name |
| `--provider` | id | yes | — | e.g. `aws_secrets_manager` |
| `--external-ref` | string | yes | — | ARN / name / path in the provider vault |
| `--key` | string | no | — | Portable secret key |
| `--provider-version-ref` | string | no | — | Pin a provider version id/label |
| `--description` | string | no | — | Free text |

```sh
paperclipai secrets link -C cmp_123 --name prod-stripe-key --provider aws_secrets_manager \
  --external-ref arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/stripe-AbCdEf
```

### Update / rotate / audit / delete (operate on a secret id, no `-C`)

```
paperclipai secrets update <secret-id> --payload-json '<json>'      # metadata only (PATCH; server UpdateSecret shape)
paperclipai secrets rotate <secret-id> (--value <v> | --value-env <ENV>)   # replaces value, bumps version
paperclipai secrets usage <secret-id>                               # agents/projects whose env references it
paperclipai secrets access-events <secret-id>                       # read-audit trail
paperclipai secrets delete <secret-id> --yes --confirm <secret-id>  # both flags required
```

```sh
export NEW_KEY=sk-ant-new...
paperclipai secrets rotate sec_abc --value-env NEW_KEY
paperclipai secrets usage sec_abc --json   # run BEFORE delete/rotate
paperclipai secrets delete sec_abc --yes --confirm sec_abc
```

`delete` refuses unless `--yes` is present AND `--confirm` exactly matches the positional secret id.

### Provider health and descriptors

```
paperclipai secrets doctor -C <company-id>      # per-provider status: ok|warn|error, warnings, missing config, credential source, backup guidance
paperclipai secrets providers -C <company-id>   # provider descriptors usable with create/link/provider-configs
```

For AWS: do NOT store AWS bootstrap credentials inside Paperclip secrets; the runtime resolves the standard AWS credential chain — `doctor` confirms it is wired.

### Provider vault configs

A company can hold multiple vault configs (one default). Mutating subcommands take raw `--payload-json` matching the server contract.

| Subcommand | Args | Required flags | Does |
|---|---|---|---|
| `provider-configs` | — | `-C` | List configs |
| `provider-config:create` | — | `-C`, `--payload-json` | Create config |
| `provider-config:discovery-preview` | — | `-C`, `--payload-json` | Preview which secrets a vault would discover (no import) |
| `provider-config:get` | `<configId>` | — | Show one |
| `provider-config:update` | `<configId>` | `--payload-json` | PATCH one |
| `provider-config:default` | `<configId>` | — | Make it the company default vault |
| `provider-config:health` | `<configId>` | — | Health-check one |
| `provider-config:delete` | `<configId>` | — | Delete one |

```sh
paperclipai secrets provider-configs -C cmp_123 --json
paperclipai secrets provider-config:default spc_456
```

### Remote import

```
paperclipai secrets remote-import:preview -C <company-id> --payload-json '<json>'
paperclipai secrets remote-import -C <company-id> --payload-json '<json>'
```
Pull selected secrets from a configured vault into Paperclip's registry. Always run `:preview` first; both require `-C` and `--payload-json` (source + selection).

### Migrate inline env

```
paperclipai secrets migrate-inline-env -C <company-id> [--apply]
```
Finds sensitive inline agent env values, creates/rotates managed secrets for them, and rewrites agent env to `secret_ref` (`version: latest`). Dry run by default; `--apply` commits.

Dry-run output: `agentsToUpdate`, `secretsToCreate`, `secretsToRotate`, plus candidates. "Sensitive" is detected by env-key name regex (`token`/`*token` suffix, `api_key`, `access_token`, `auth*`, `password`/`passwd`, `secret`, `credential`, `jwt`, `private_key`, `bearer`, `cookie`, `connectionstring`, …). Generated secret names follow `agent_<first8OfAgentId>_<envkey>` with the env key lowercased; an existing secret with that name is rotated rather than duplicated.

### Secrets gotchas

- Company-scoped subcommands (`list`, `declarations`, `create`, `link`, `doctor`, `providers`, `provider-configs`, `provider-config:create`, `provider-config:discovery-preview`, `remote-import*`, `migrate-inline-env`) require `-C/--company-id`. Id-scoped ones (`update`, `rotate`, `usage`, `access-events`, `delete`, `provider-config:get/update/default/health/delete`) do not.
- `update` ≠ `rotate`: `update` patches metadata only; `rotate` changes the value.
- Plaintext is never retrievable via the CLI — keep your own copy at creation time if you need it elsewhere.
- Deleting a secret that agent env still references breaks those agents; check `usage` first.
- `--value` vs `--value-env` is exactly-one-of; `--value-env` fails on empty/unset vars.

---

## Cloud sync (`paperclipai cloud <sub>`)

Experimental, one-directional: local instance → cloud stack. There is no `cloud pull`. Only two commands exist: `connect` and `push`. (`cloud-store.ts` / `cloud-transfer.ts` are internal helpers — connection storage and bundle building — not commands.)

### `cloud connect`

```
paperclipai cloud connect <remote-url> [--no-browser]
```
Authorizes this local instance against a cloud stack: discovery via `/.well-known/paperclip-upstream` (must advertise `paperclip-upstream-discovery-v1`, a matching transfer-schema major, and the `cloud_sync` feature flag), then PKCE browser auth with loopback callback, falling back to device-code flow (verification URL + one-time user code) if no browser or `--no-browser`. Mints a per-instance Ed25519 key pair; token scopes: `upstream_import:preview|read|write`.

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `--no-browser` | bool | no | off | Force device-code flow |

```sh
paperclipai cloud connect https://acme.paperclip.cloud --no-browser
```

Connection saved locally under the instance root at `secrets/cloud-upstream-connections.json` (mode 0600), keyed by stack origin; the most recent connect becomes current. `--json` prints a redacted record — id, remoteUrl, targetOrigin, stackId, targetCompanyId, scopes, token expiry — never the access token or private key.

### `cloud push`

```
paperclipai cloud push --company <local-company-id> [--remote-url <url>] [--dry-run] [--max-entities-per-chunk <n>]
```
Exports the local company (manifest, agents, projects, issues, skills, markdown-backed files), builds a signed upstream transfer bundle, and previews (`--dry-run`) or applies it against the stored connection. Refuses unless the local instance experimental setting `enableCloudSync` is `true`.

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `--company` | id | yes | — | Local company to export and push |
| `--remote-url` | url | no | current/only connection | Select among multiple stored connections |
| `--dry-run` | bool | no | off | Preview only, nothing applied upstream |
| `--max-entities-per-chunk` | int | no | 100 | Upload chunk size |

```sh
paperclipai cloud push --company cmp_123 --dry-run
paperclipai cloud push --company cmp_123 --remote-url https://acme.paperclip.cloud
```

Human output: `run=<run-id>`, `manifest=<hash>`, and counts `create/update/adopt/skip/conflict/staleMapping`, then warnings (yellow), first 10 conflicts (red, then truncation count), and on apply the last run events. `--json` emits `{ result, events }` with the full coordinator result and run events.

Outcome meanings: `create` new upstream entity; `update` rewritten to match local; `adopt` already-matching entity linked without rewrite; `skip` unchanged; `conflict` unreconcilable; `staleMapping` recorded source→target mapping no longer valid. Content-hash idempotency means re-pushing an unchanged company adopts/skips, never duplicates.

Exit codes:

| Code | Meaning |
|---|---|
| 0 | Completed, no conflicts |
| 2 | Push ran but `conflict + staleMapping > 0` — fix and re-push (not a hard failure) |
| 3 | Transfer schema major mismatch — upgrade CLI or stack first |

### Cloud gotchas

- `cloud connect` works without `enableCloudSync`; the first `push` aborts with "Cloud sync is disabled…" until the instance experimental setting is on. The flag lives on the local instance, not the CLI.
- Common client flags here point at your LOCAL instance (export source); the cloud stack auth comes from the stored connection + signing key.
- Always `--dry-run` before applying; exit code 2 still means the apply happened for non-conflicting entities.
- Multiple connected stacks: omit `--remote-url` and you get the most recently connected stack — pass it explicitly in scripts.

---

## Plugins (`paperclipai plugin <sub>`)

Plugins run server-side in the Paperclip runtime; the CLI registers, toggles, and proxies to them. Exception: a local-path install runs trusted local code from your disk. `plugin init` is the only command that never touches the API.

Statuses: `ready` (running), `installed` (registered, not fully started), `upgrade_pending`, `disabled`, `error` (check `lastError` / logs).

### Author and install

```
paperclipai plugin init <packageName> [--output <dir>] [--template <t>] [--category <c>] [--display-name <n>] [--description <d>] [--author <a>] [--sdk-path <path>]
```
Scaffolds a local plugin project; folder name is the package with scope stripped (`@acme/plugin-linear` → `plugin-linear/`).

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `--output` | dir | no | cwd | Parent directory |
| `--template` | enum | no | `default` | `default` \| `connector` \| `workspace` \| `environment` |
| `--category` | enum | no | — | `connector` \| `workspace` \| `automation` \| `ui` \| `environment` |
| `--display-name` / `--description` / `--author` | string | no | — | Manifest fields |
| `--sdk-path` | path | no | — | Use a local unpublished `@paperclipai/plugin-sdk` |

After init: `cd <folder> && pnpm install && pnpm dev`, then `paperclipai plugin install <folder>`. Keep `pnpm dev` running — Paperclip watches `dist` and reloads the worker.

```
paperclipai plugin install <package> [-l|--local] [--version <v>]
```
Installs from a local path (auto-detected: absolute, `./`, `../`, `~`, or existing relative dir; resolved to absolute) or npm.

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `-l, --local` | bool | no | off | Force local-path treatment |
| `--version` | semver | no | latest | npm only; rejected with local paths |

```sh
paperclipai plugin install ./my-plugin
paperclipai plugin install @acme/plugin-linear --version 1.2.0
```

Success prints key, version, status. A `lastError` recorded during load is surfaced as a warning even on a successful install call — confirm status is `ready` before relying on it.

### Operate

```
paperclipai plugin list [--status <s>]            # --status: ready|error|disabled|installed|upgrade_pending
paperclipai plugin inspect <pluginKeyOrId>        # full record + complete lastError; exits non-zero if not found (usable as presence check)
paperclipai plugin enable <pluginKeyOrId>         # bring disabled/error plugin online
paperclipai plugin disable <pluginKeyOrId>        # stop without losing config/state
paperclipai plugin uninstall <pluginKeyOrId> [--force]   # --force = hard-purge all state/config
paperclipai plugin examples                       # bundled example plugins with ready-to-run install commands
```

### Runtime surfaces

POST-style commands take `--payload-json <json>` (default `{}`). `<pluginId>` accepts plugin key or db id.

```
paperclipai plugin ui-contributions                          # instance-wide read
paperclipai plugin tools                                     # instance-wide read
paperclipai plugin tool:execute --payload-json '{"tool":"...","input":{}}'
paperclipai plugin health <pluginId>
paperclipai plugin logs <pluginId>
paperclipai plugin upgrade <pluginId> [--payload-json '{}']  # advance an upgrade_pending plugin
paperclipai plugin config <pluginId>
paperclipai plugin config:set <pluginId> --payload-json '{"configJson":{"apiKey":"..."}}'
paperclipai plugin config:test <pluginId> --payload-json '{"configJson":{"apiKey":"..."}}'  # validate before committing
paperclipai plugin jobs <pluginId>
paperclipai plugin job:runs <pluginId> <jobId>
paperclipai plugin job:trigger <pluginId> <jobId> [--payload-json '{}']
paperclipai plugin webhook <pluginId> <endpointKey> [--payload-json '{}']
paperclipai plugin dashboard <pluginId>
paperclipai plugin data <pluginId> <key> [--payload-json '{}']     # URL-keyed read endpoint
paperclipai plugin action <pluginId> <key> [--payload-json '{}']   # URL-keyed mutation endpoint
```

### Bridge channels

```
paperclipai plugin bridge:data <pluginId> [--payload-json '{}']
paperclipai plugin bridge:action <pluginId> [--payload-json '{}']
paperclipai plugin bridge:stream <pluginId> <channel> [--duration-ms <ms>]
```
`bridge:stream` writes the raw stream to stdout until interrupted or `--duration-ms` elapses — always set a duration in scripts.

### Local folder bindings (company-scoped: require `-C`)

```
paperclipai plugin local-folders <pluginId> -C <company-id>
paperclipai plugin local-folder:status <pluginId> <folderKey> -C <company-id>
paperclipai plugin local-folder:validate <pluginId> <folderKey> -C <company-id> [--payload-json '{}']
paperclipai plugin local-folder:set <pluginId> <folderKey> -C <company-id> --payload-json '{"path":"/abs/path"}'
```
`local-folder:set` is the only one where `--payload-json` is required (it is a PUT, not POST). Validate before set.

### Plugin gotchas

- Local-path installs execute code from your disk inside the runtime — only install paths you control.
- `--version` with a local path is rejected; it is npm-only.
- `uninstall` without `--force` keeps config so a reinstall restores it; `--force` is the irreversible clean slate.
- Naming overload: `config` (read) vs `config:set`/`config:test`; `bridge:data`/`bridge:action` (bridge POSTs) vs `data`/`action` (URL-keyed endpoints taking an extra `<key>` arg); `jobs` (list) vs `job:runs`/`job:trigger` (need both `<pluginId> <jobId>`).
- Debug order for `error` status: `plugin health` → `plugin logs` → `plugin inspect` (full `lastError`).
- All plugin commands except `init` need a reachable API; `init` works offline.

---

## Cross-family notes

- None of these commands document a board-vs-agent persona restriction; authentication/persona resolution is covered in auth-and-context.md.
- Destructive/irreversible: `secrets delete` (double-confirm), `secrets provider-config:delete`, `plugin uninstall --force`, `secrets migrate-inline-env --apply`, `cloud push` without `--dry-run`.
- `--payload-json` commands pass raw JSON to the server contract — invalid JSON fails at parse time, invalid shapes fail server-side.
