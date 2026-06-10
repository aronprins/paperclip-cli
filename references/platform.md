# Platform: workspaces, environments, adapters, assets, skills

Load this file when working with execution/project workspaces, environments and leases, org charts, adapter runtimes (what executes an agent), uploading/downloading company assets, or installing/attaching company skills. Auth, contexts, profiles, API-base resolution, and the shared client flags (`--data-dir`, `--api-base`, `--api-key`, `--context`, `--profile`, `--json`) are covered in `auth-and-context.md` — every command below accepts them.

Scoping conventions used throughout: company-scoped commands take `-C, --company-id <id>` (required unless your selected profile resolves a company); ID-scoped and project-scoped commands take ids as positional arguments instead.

---

## Org chart (`org`)

```
paperclipai org get -C <company-id>
paperclipai org svg -C <company-id> [--out <path>]
paperclipai org png -C <company-id> [--out <path>]
```

- `org get` — org chart as structured JSON. Respects `--json` like any read.
- `org svg` / `org png` — binary image. With `--out <path>` writes the file and prints `{ out, bytes }`; without `--out`, raw bytes stream to stdout — always redirect (`> org.png`).

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id <id>` | string | yes* | from profile | Company to chart |
| `--out <path>` | string (svg/png only) | no | stdout | Write image to file, print `{ out, bytes }` |

```sh
paperclipai org get -C cmp_123 --json
paperclipai org png -C cmp_123 --out org.png
```

## Agent configurations (`agent-config`)

```
paperclipai agent-config list -C <company-id>
```

Lists agent configuration summaries for a company — the resolved adapter/model setup behind each agent. Read-only; use before changing environments or hiring. `--json` for diffing/scripting.

## Execution workspaces (`workspace`)

Server-side sandboxes where heartbeat runs do their work.

```
paperclipai workspace list -C <company-id>
paperclipai workspace get <workspace-id>
paperclipai workspace close-readiness <workspace-id>
paperclipai workspace operations <workspace-id>
paperclipai workspace update <workspace-id> --payload-json '{...}'
paperclipai workspace runtime-service <workspace-id> <action> [--payload-json '{...}']
paperclipai workspace runtime-command <workspace-id> <action> [--payload-json '{...}']
```

| Command | Args/flags | Effect |
|---|---|---|
| `list` | `-C <id>` required* | All execution workspaces in a company |
| `get` | `<id>` | One workspace record |
| `close-readiness` | `<id>` | Whether closing now would interrupt in-flight work |
| `operations` | `<id>` | Operation history |
| `update` | `<id>` + `--payload-json <json>` (required) | PATCH the workspace |
| `runtime-service` | `<id> <action>` + `--payload-json` (optional, default `{}`) | Control a runtime service |
| `runtime-command` | `<id> <action>` + `--payload-json` (optional, default `{}`) | Run a runtime command |

`<action>` is one of `start`, `stop`, `restart`, `run`. `--payload-json` carries the runtime target.

```sh
paperclipai workspace runtime-service ws_42 restart --payload-json '{"service":"dev-server"}'
paperclipai workspace runtime-command ws_42 run --payload-json '{"command":"npm test"}'
```

## Environments (`environment`)

Backing infrastructure (SSH hosts, containers, capabilities) that workspaces run against. A lease = a workspace's claim on an environment.

```
paperclipai environment list -C <company-id>
paperclipai environment capabilities -C <company-id>
paperclipai environment create -C <company-id> --payload-json '{...}'
paperclipai environment get <environment-id>
paperclipai environment leases <environment-id>
paperclipai environment lease <lease-id>
paperclipai environment update <environment-id> --payload-json '{...}'
paperclipai environment delete <environment-id>
paperclipai environment probe <environment-id>
paperclipai environment probe-config -C <company-id> --payload-json '{...}'
```

| Command | Args/flags | Effect |
|---|---|---|
| `list` / `capabilities` | `-C <id>` | Inventory environments / available capability set |
| `create` | `-C <id>` + `--payload-json` (required) | Create environment from JSON body |
| `get` | `<environment-id>` | One environment |
| `leases` | `<environment-id>` | All leases ON that environment |
| `lease` | `<lease-id>` | ONE lease by its own id (`/api/environment-leases/{id}`) |
| `update` | `<environment-id>` + `--payload-json` (required) | PATCH environment |
| `delete` | `<environment-id>` | Delete environment |
| `probe` | `<environment-id>` | Health/reachability check of an existing environment |
| `probe-config` | `-C <id>` + `--payload-json` (required) | Dry-run a candidate config WITHOUT persisting |

```sh
# Validate before creating
paperclipai environment probe-config -C cmp_123 --payload-json '{"kind":"ssh","host":"build-01.internal"}'
paperclipai environment create -C cmp_123 --payload-json '{"name":"build-01","kind":"ssh","host":"build-01.internal"}'
```

## Project workspaces (`project-workspace`)

Workspaces scoped to one project. Addressed by project id (+ workspace id for single-workspace ops), never `-C`.

```
paperclipai project-workspace list <project-id>
paperclipai project-workspace create <project-id> --payload-json '{...}'
paperclipai project-workspace update <project-id> <workspace-id> --payload-json '{...}'
paperclipai project-workspace delete <project-id> <workspace-id>
paperclipai project-workspace runtime-service <project-id> <workspace-id> <action> [--payload-json '{...}']
paperclipai project-workspace runtime-command <project-id> <workspace-id> <action> [--payload-json '{...}']
```

`create` takes only `<project-id>` (workspace doesn't exist yet); `update` errors if `<workspace-id>` is omitted. `--payload-json` is required on create/update, optional (default `{}`) on runtime actions. Same `<action>` set: `start|stop|restart|run`.

```sh
paperclipai project-workspace runtime-command prj_9 ws_3 run --payload-json '{"command":"pnpm build"}'
```

### Workspace/environment gotchas
- `runtime-service stop`/`restart` act on live processes a run may depend on — check `workspace get` + `workspace operations` first.
- Run `workspace close-readiness <id>` before tearing a workspace down.
- `environment delete` is destructive and has no confirmation prompt.
- `environment leases <env-id>` vs `environment lease <lease-id>`: plural lists by environment, singular resolves a lease by its own id. Don't mix the ids up.
- `org svg`/`png` and other binary outputs garble the terminal without `--out` or a redirect.

---

## Adapters (`adapter`)

Adapters are the server-side runtimes that execute an agent's work (e.g. `claude_local`, `codex_local`). An agent's `adapterType` points at one. The CLI only inspects/administers the registry — execution stays server-side.

### Inspect (instance-wide, no company scope)

```
paperclipai adapter list
paperclipai adapter get <type>
paperclipai adapter config-schema <type>
paperclipai adapter ui-parser <type>
```

- `list` — every registered adapter (built-in + external); the source of valid `<type>` strings.
- `get <type>` — full record: current settings, install source, status.
- `config-schema <type>` — JSON schema for the adapter's config. Read this BEFORE writing `--payload-json` for `adapter update`, `adapter override`, or `agent create`.
- `ui-parser <type>` — the adapter's UI parser JavaScript (how the UI renders run output).

```sh
paperclipai adapter list --json
paperclipai adapter config-schema codex_local --json
```

### Company-scoped reads (require resolvable company)

```
paperclipai adapter models <type> -C <company-id> [--refresh] [--environment-id <id>]
paperclipai adapter model-profiles <type> -C <company-id>
paperclipai adapter detect-model <type> -C <company-id>
```

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id <id>` | string | yes* | from profile | Models depend on the company's provider credentials |
| `--refresh` | boolean (`models` only) | no | false | Re-fetch provider model list instead of cache (after key rotation / new model release) |
| `--environment-id <id>` | string (`models` only) | no | — | Scope lookup for environment-aware adapters |

`model-profiles` = configured model profiles; `detect-model` = which model the adapter would actually use.

### Install / remove / control

```
paperclipai adapter install --payload-json '{"packageName":"@scope/adapter","version":"1.2.3"}'
paperclipai adapter delete <type>
paperclipai adapter update <type> --payload-json '{...}'
paperclipai adapter override <type> --payload-json '{"paused":true}'
paperclipai adapter reload <type> [--payload-json '{...}']
paperclipai adapter reinstall <type> [--payload-json '{...}']
paperclipai adapter test-environment <type> -C <company-id> [--payload-json '{...}']
```

| Command | Payload | Effect |
|---|---|---|
| `install` | `--payload-json` required | Register an external adapter package |
| `delete <type>` | — | Remove an EXTERNAL adapter registration |
| `update <type>` | `--payload-json` required (PATCH) | Change adapter settings — validate against `config-schema` first |
| `override <type>` | `--payload-json` required (PATCH) | Pause/resume a BUILT-IN adapter — how you disable a built-in without deleting it |
| `reload <type>` | optional, default `{}` | Re-read config in place |
| `reinstall <type>` | optional, default `{}` | Clean reinstall from source when reload isn't enough |
| `test-environment <type>` | `-C` required; payload optional, default `{}` | Validate an environment config for the adapter |

### Adapter gotchas
- `adapter delete` strands every agent whose `adapterType` points at it — check `agent list` per company first. Built-ins can't be deleted; use `override` to pause them.
- Adapter administration is a board-operator activity; an agent-scoped credential (bound to one company/agent) is the wrong persona for install/override/delete.
- Only `models`, `model-profiles`, `detect-model`, `test-environment` are company-scoped; everything else is instance-wide.

---

## Assets (`asset`)

Push binary/image assets into a company and pull them back by id.

```
paperclipai asset image:upload --file <path> -C <company-id> [--namespace <v>] [--alt <text>] [--title <text>]
paperclipai asset logo:upload --file <path> -C <company-id>
paperclipai asset content <asset-id> [--out <path>]
```

| Command | Flag | Type | Req | Default | Effect |
|---|---|---|---|---|---|
| `image:upload` | `--file <path>` | string | yes | — | File to upload (multipart); content type inferred from extension, original filename sent |
| | `-C, --company-id <id>` | string | yes* | from profile | Owning company |
| | `--namespace <value>` | string | no | — | Logical bucket suffix (e.g. `docs`) |
| | `--alt <text>` / `--title <text>` | string | no | — | Metadata stored with the asset |
| `logo:upload` | `--file <path>`, `-C` | as above | | | Sets the company logo (`/logo` slot); NO namespace/alt/title |
| `content` | `<asset-id>` positional | string | yes | — | Global asset id — not company-scoped |
| | `--out <path>` | string | no | stdout | Write bytes to file; prints `{ ok, out, bytes }` |

```sh
ASSET_ID=$(paperclipai asset image:upload -C cmp_123 --file ./diagram.png --json | jq -r '.id')
paperclipai asset content "$ASSET_ID" --out ./roundtrip.png
```

`--json` on uploads prints the stored asset record (includes the asset `id` to feed into `asset content`).

### Asset gotchas
- Content type is inferred from the file extension — a misnamed file is stored with the wrong type.
- `asset content` without `--out` streams raw binary to stdout; always redirect/pipe. `--json` only shapes the confirmation object in `--out` mode.
- `logo:upload` silently replaces the existing logo; there is no remove-logo subcommand.

---

## Skills

Two groups: plural `skills` (everyday use — resolves refs by `id`/`key`/`slug`, prompts, readable tables) and singular `skill` (raw payload-driven wrapper, literal ids only). Prefer `skills`. All company-scoped subcommands take `-C, --company-id <id>`.

Three distinct operations — keep them straight:
1. **Company install** (`install`, `import`, `create`, `scan-projects`) — puts a skill in the company library; attaches to NO agent.
2. **Agent attach** (`agent sync`, `agent clear`) — replaces the agent's desired non-required set (NOT additive).
3. **Adapter runtime sync** — server-side reconciliation; reported via `agent list` as an `AgentSkillSnapshot`.

### Catalog (no company context needed)

```
paperclipai skills browse [--kind bundled|optional] [--category <slug>] [--query <text>]
paperclipai skills search <query> [--kind <kind>] [--category <slug>]
paperclipai skills inspect <catalogRef>
```

`<catalogRef>` = catalog skill `id`, `key`, or unique `slug`. `inspect` shows the file manifest (path, kind, size, sha256) — read it before installing.

### Install from catalog

```
paperclipai skills install <catalogRef> -C <company-id> [--as <slug>] [--force]
```

| Flag | Default | Effect |
|---|---|---|
| `--as <slug>` | — | Override the company skill slug |
| `--force` | false | Replace a same-key catalog-managed skill if the server allows; never bypasses hard validation or hard-stop audit findings |

Catalog only. GitHub / skills.sh / local paths / URLs go through `skills import`.

### Company library

```
paperclipai skills list -C <company-id>
paperclipai skills show <skillRef> -C <company-id>
paperclipai skills file <skillRef> [--path <path>] -C <company-id>        # --path default: SKILL.md; human mode = raw content to stdout
paperclipai skills import <source> -C <company-id>                        # local path, GitHub, skills.sh, or URL
paperclipai skills create --name <name> [--slug <slug>] [--description <text>] [--body-file <path>|-] -C <company-id>
paperclipai skills scan-projects [--project-id <id>]... [--workspace-id <id>]... -C <company-id>
```

`<skillRef>` resolves by `id`, canonical `key`, or unique `slug`; ambiguous slugs are rejected. `--body-file -` reads stdin. `scan-projects` with no `--project-id`/`--workspace-id` scans all of the company's projects and workspaces; summary reports discovered/imported/updated/skipped/conflicts/warnings.

### Maintenance loop

```
paperclipai skills check [skillRef] -C <company-id>
paperclipai skills update [skillRef] | --all [--force] -C <company-id>
paperclipai skills audit [skillRef] -C <company-id>
paperclipai skills reset <skillRef> --yes [--force] -C <company-id>
paperclipai skills remove <skillRef> --yes -C <company-id>
```

| Command | Effect |
|---|---|
| `check` | Update status: `supported`, `hasUpdate`, `currentRef`/`latestRef`, `installedHash`/`originHash`, `updateHoldReason`, `auditVerdict`. Omit ref = whole library |
| `update` | Install pinned update; `--all` updates only `hasUpdate=true` skills. Ref + `--all` together is rejected |
| `audit` | Re-scan installed bytes; findings (severity, code, path, message); executes nothing |
| `reset` | Reinstall from pinned origin, DISCARDING local edits. `--yes` required non-interactively |
| `remove` | Delete from company library. Prompts in TTY; `--yes` required otherwise |

`--force` (update/reset): discards local-modification and soft-audit holds; hard-stop audit findings still block.

### Agent attach (desired skills)

```
paperclipai skills agent list <agentRef> -C <company-id>
paperclipai skills agent sync <agentRef> --skill <skillRef> [--skill <skillRef>]... -C <company-id>
paperclipai skills agent clear <agentRef> [--yes] -C <company-id>
```

`<agentRef>` = agent `id` or shortname/url-key. `sync` requires at least one `--skill` and REPLACES the full non-required desired set — list everything you want every time. `clear` sends an empty list (prompts in TTY, `--yes` otherwise). `agent list` shows the runtime snapshot: adapter type, sync `supported`, `mode`, desired count, per-skill state/origin/managed/required flags.

```sh
paperclipai skills install github-pr-workflow -C cmp_123
paperclipai skills agent sync dev-agent --skill github-pr-workflow --skill agent-browser -C cmp_123
```

### Singular `skill` group (low-level)

Literal `<skillId>` only — no key/slug resolution, no prompts. Use when scripting with known ids and raw request bodies.

```
paperclipai skill list -C <company-id>
paperclipai skill get <skill-id> -C <company-id>
paperclipai skill file <skill-id> [--path <path>] -C <company-id>           # --path default: SKILL.md
paperclipai skill file:update <skill-id> --payload-json '{"path":"SKILL.md","content":"..."}' -C <company-id>
paperclipai skill create --payload-json '{...}' -C <company-id>
paperclipai skill import --payload-json '{...}' -C <company-id>
paperclipai skill scan-projects --payload-json '{...}' -C <company-id>
paperclipai skill update-status <skill-id> -C <company-id>
paperclipai skill install-update <skill-id> -C <company-id>
paperclipai skill delete <skill-id> -C <company-id>
```

`file:update`, `create`, `import`, `scan-projects` all REQUIRE `--payload-json` (`file:update` takes a `CompanySkillFileUpdate` body).

### Skills gotchas
- `agent sync` is desired-state replacement, not additive — omitting a previously-synced skill drops it.
- Required Paperclip runtime skills (heartbeat etc.) are server-enforced and survive both `sync` and `clear`.
- Installing into the company library attaches nothing; agent attach is a separate step.
- `skills update <ref> --all` is rejected — pick one.
- `reset` discards local edits irreversibly; `remove` and `delete` are destructive. Non-interactive runs need `--yes` on `reset`, `remove`, and `skills agent clear`.
- Naming overload: plural `skills` and singular `skill` wrap the same company-skill API endpoints (the plural group additionally covers catalog and agent-sync endpoints); `skill delete` (by id, no prompt) ≈ `skills remove` (by ref, with prompt).
