# Setup & Instance Management

Load this file when installing the Paperclip CLI, bootstrapping or repairing a local instance (`onboard`, `run`, `doctor`, `configure`, `env`, `allowed-hostname`, `db:backup`), or managing dev tooling (`worktree*`, `env-lab`). These commands operate on the local install — config files, embedded DB, secrets, ports — not the control plane. For tokens, profiles, personas, and `--api-base` resolution see auth-and-context.md. Three further setup-family commands live in sibling files: `auth bootstrap-ceo` (auth-and-context.md), `heartbeat run` and `routines disable-all` (runs-and-routines.md — the latter writes directly to the local DB and works with the server down).

## Installation

Requires Node.js >= 20 on PATH. The npm package `paperclipai` exposes one binary, `paperclipai`.

```sh
npx paperclipai onboard --yes      # zero-install quickstart (always latest)
npm install -g paperclipai         # persistent global install
```

Inside the Paperclip monorepo, substitute `pnpm paperclipai <command>` (runs from source via tsx).

Cold start to running server: `paperclipai onboard` then `paperclipai run`. Bare `run` also hands off to onboarding automatically when no config exists and the terminal is interactive.

## Local paths

Default instance state lives under `~/.paperclip/instances/<instance-id>` (default id `default`):

| Data | Path |
|---|---|
| Config | `~/.paperclip/instances/default/config.json` |
| Embedded DB | `~/.paperclip/instances/default/db` |
| Logs | `~/.paperclip/instances/default/logs` |
| Storage (local disk) | `~/.paperclip/instances/default/data/storage` |
| Secrets key | `~/.paperclip/instances/default/secrets/master.key` |

Shared flags on every setup command: `-c, --config <path>` (config file) and `-d, --data-dir <path>` (relocate all state away from `~/.paperclip`).

---

## `paperclipai onboard`

`paperclipai onboard [-y] [--run] [--bind <mode>] [-c <path>] [-d <path>]`

Interactive first-run wizard: writes `config.json`, provisions `PAPERCLIP_AGENT_JWT_SECRET` into the adjacent `.env`, and creates the local secrets master key.

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `-y, --yes` | bool | no | false | Accept Quickstart defaults non-interactively and start immediately. Without `--bind`, forces trusted-local loopback and ignores conflicting reachability env vars. |
| `--run` | bool | no | false | Start the server immediately after saving config (keeps prompts interactive). |
| `--bind <mode>` | enum | no | — (quickstart binds loopback) | Quickstart reachability preset: `loopback`, `lan`, or `tailnet`. |
| `-c, --config <path>` | string | no | instance default | Write config to a specific path. |
| `-d, --data-dir <path>` | string | no | `~/.paperclip` | Isolate all local state. |

First prompt picks a path: **Quickstart** (embedded PostgreSQL, file logging, local-disk storage, local-encrypted secrets, loopback; honors env overrides like `DATABASE_URL`, `PAPERCLIP_PUBLIC_URL`, `PAPERCLIP_DEPLOYMENT_MODE`) or **Advanced** (step-by-step prompts for database, LLM, logging, server/auth, storage, secrets; tests DB connection and validates LLM key inline).

```sh
paperclipai onboard --yes              # CI/scripts: non-interactive quickstart + boot
paperclipai onboard --bind lan         # quickstart reachable on the LAN
```

Re-running on an existing config preserves it unchanged (still tops up JWT secret / secrets key). Use `configure` to change settings, not re-onboarding.

## `paperclipai run` (local bootstrap — no subcommand)

`paperclipai run [-i <id>] [--bind <mode>] [--no-repair] [-c <path>] [-d <path>]`

Bootstraps and starts a local server: resolves instance + config, loads instance `.env`, hands off to onboarding if no config (TTY only), runs `doctor` with repair on, then starts the server.

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `-i, --instance <id>` | string | no | `default` | Local instance id; run multiple isolated instances side by side. |
| `--bind <mode>` | enum | no | — | First run only: forwarded to onboarding (`loopback`/`lan`/`tailnet`). Ignored once a config exists — change binding via `configure --section server`. |
| `--repair` / `--no-repair` | bool | no | repair on | `--no-repair` makes doctor read-only (report, change nothing). |
| `-c, --config <path>` | string | no | instance default | Specific config file. |
| `-d, --data-dir <path>` | string | no | `~/.paperclip` | Isolate local state. |

```sh
paperclipai run --instance dev               # second isolated instance
paperclipai run --data-dir ./tmp/pc-dev      # state confined to a directory
```

Exit behavior (verified in source): exits 1 if no config + non-TTY (tells you to run `paperclipai onboard` once, then retry), or if any doctor check fails ("Doctor found blocking issues. Not starting server."). When the instance is `authenticated` mode + embedded DB, it auto-generates a bootstrap CEO invite after startup (see `auth bootstrap-ceo` in auth-and-context.md).

## `paperclipai doctor`

`paperclipai doctor [--repair] [-y] [-c <path>] [-d <path>]`

Diagnostic checks with optional repair; `run` invokes it for you — call directly when you only want the report.

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `--repair` | bool | no | off | Attempt to repair fixable issues (prompts per repair unless `-y`). |
| `-y, --yes` | bool | no | false | Skip per-repair confirmation prompts. |
| `-c, --config <path>` | string | no | instance default | Config file. |
| `-d, --data-dir <path>` | string | no | `~/.paperclip` | Isolate local state. |

Checks in order (stops early only if config invalid): config validity, deployment/auth mode compatibility, agent JWT secret, secrets adapter, storage, database connectivity, LLM provider, log directory, listen port. Each line prints `✓ pass` / `! warn` / `✗ fail` with a repair hint; summary counts passed/warned/failed.

```sh
paperclipai doctor                    # report only
paperclipai doctor --repair --yes     # fix everything fixable, no prompts
```

## `paperclipai env`

`paperclipai env [-c <path>] [-d <path>]`

Prints the resolved environment variables a deployment needs — each marked `set`, `default`, or `missing` — plus a ready-to-paste `export` block. Required: `PAPERCLIP_AGENT_JWT_SECRET`, `DATABASE_URL`. Optional: `PORT`, `PAPERCLIP_PUBLIC_URL`, `BETTER_AUTH_TRUSTED_ORIGINS`, agent-JWT/heartbeat-scheduler tunables, secrets/storage providers. Missing values render as `<set-this-value>` placeholders. Warns (does not fail) on a missing/unparseable config. Only flags: `-c`, `-d`.

```sh
paperclipai env        # inspect resolved env / seed a container deployment
```

## `paperclipai configure`

`paperclipai configure [-s <section>] [-c <path>] [-d <path>]`

Updates configuration sections on an existing install. Interactive menu by default; `--section` configures one section and exits. Errors if no config exists (run `onboard` first).

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `-s, --section <s>` | enum | no | menu loop | One of `llm`, `database`, `logging`, `server`, `storage`, `secrets`. |
| `-c, --config <path>` | string | no | instance default | Config file. |
| `-d, --data-dir <path>` | string | no | `~/.paperclip` | Isolate local state. |

Sections: `server` (deployment mode, exposure, host binding, port, served UI, auth base URL) · `database` (embedded vs external Postgres, connection string, backups) · `storage` (`local_disk` or `s3`) · `secrets` (provider, strict mode, local key file) · `logging` (mode, directory) · `llm` (provider, API key).

```sh
paperclipai configure --section server     # change binding/port after onboarding
```

## `paperclipai db:backup`

`paperclipai db:backup [--dir <path>] [--retention-days <n>] [--filename-prefix <p>] [--json]`

One-off database snapshot using the current config, independent of the server's scheduled backups. Connection string resolution order: `DATABASE_URL` env → `config.database.connectionString` (postgres mode) → embedded default `postgres://paperclip:paperclip@127.0.0.1:<embeddedPort>/paperclip` (port default 54329). Works with the server down (connects to embedded port directly).

| Flag | Type | Req | Default | Effect |
|---|---|---|---|---|
| `--dir <path>` | string | no | config backup dir, else `<instance-root>/data/backups` | Output directory (instance default verified in source: `~/.paperclip/instances/<id>/data/backups`). |
| `--retention-days <n>` | int >= 1 | no | config value or 30 | Daily-pruning window (weekly=4, monthly=1 are fixed). |
| `--filename-prefix <p>` | string | no | `paperclip` | Backup filename prefix. |
| `--json` | bool | no | false | Print metadata as JSON after the human output. |
| `-c` / `-d` | string | no | — | Config / data-dir as usual. |

`--json` keys: `backupFile`, `sizeBytes`, `prunedCount`, `backupDir`, `retentionDays`, `connectionSource`.

```sh
paperclipai db:backup --dir /backups/paperclip --retention-days 14 --json
```

## `paperclipai allowed-hostname`

`paperclipai allowed-hostname <host> [-c <path>] [-d <path>]`

Adds a hostname (normalized, lowercased) to `server.allowedHostnames` in the local config so it is accepted in `authenticated` + `private` mode (e.g. a Tailscale machine name). Idempotent — reports if already present.

```sh
paperclipai allowed-hostname my-host.ts.net
```

Takes effect only after a **server restart**. Inert outside authenticated/private mode (the command notes this). Verified in source: with no config it prints an error and returns without writing (exit 0).

### Gotchas — core setup family

- `run` vs `run <sub>`: bare `run` is local bootstrap; `run list/get/events/log/cancel` are heartbeat-run inspection over the API — a different feature entirely.
- Standalone `doctor` does **not** exit non-zero on failed checks (verified: the CLI ignores the returned counts); only `run` converts doctor failures into exit 1. Don't gate scripts on `doctor`'s exit code — parse output instead.
- `doctor --repair` writes local files (JWT `.env`, secrets key, log dir). Review before using on shared instances; pair with `--yes` only when you trust the repairs.
- `onboard --yes` forcibly ignores reachability env vars unless `--bind` is given — surprising in CI where you exported `PAPERCLIP_PUBLIC_URL`.
- `run --bind` only matters on the very first run; afterwards use `configure --section server`.
- Docs list `--fix` as an alias of doctor's `--repair`; in source it is registered as a command-level alias — prefer `--repair`.
- Bringing the server up gives the CLI no authenticated identity. Connect + pick a persona next (auth-and-context.md).
- `db:backup --retention-days` rejects non-positive/non-integer values with an error.

---

## Worktree dev tooling

Isolated Paperclip instance per git worktree: own config at `<worktree>/.paperclip/config.json`, own embedded Postgres on its own ports, data under a worktree home (default `~/.paperclip-worktrees`, override `--home` or `PAPERCLIP_WORKTREES_DIR`). Instance id derives from the worktree name unless `--instance` is passed.

**Naming is mixed on purpose** — colon-namespaced top-level commands (`worktree:make`, `worktree:list`, `worktree:merge-history`, `worktree:cleanup`) vs `worktree` subcommands (`worktree init`, `worktree env`, `worktree reseed`, `worktree repair`). Type them exactly as shown.

### `paperclipai worktree:make <name>`

Creates `~/paperclip-<name>` as a new git worktree (auto-prefixes `paperclip-`), initializes an isolated instance inside it, and seeds its DB from a source instance.

| Flag | Type | Default | Effect |
|---|---|---|---|
| `--start-point <ref>` | string | env `PAPERCLIP_WORKTREE_START_POINT` | Remote ref to base the new branch on. |
| `--instance <id>` | string | derived from name | Explicit isolated instance id. |
| `--home <path>` | string | `~/.paperclip-worktrees` | Worktree-instance home root (env `PAPERCLIP_WORKTREES_DIR`). |
| `--from-config <path>` | string | — | Source `config.json` to seed from. |
| `--from-data-dir <path>` | string | — | Source `PAPERCLIP_HOME` for deriving source config. |
| `--from-instance <id>` | string | `default` | Source instance id. |
| `--server-port <port>` | int | next free | Preferred server port. |
| `--db-port <port>` | int | next free | Preferred embedded Postgres port. |
| `--seed-mode <mode>` | enum | `minimal` | `minimal` (skips heavy history tables) or `full`. |
| `--preserve-live-work` | bool | off | Don't quarantine copied timers/assigned open issues. |
| `--no-seed` | bool | off | Skip DB seeding entirely. |
| `--force` | bool | off | Replace existing repo-local config + instance data. |

```sh
paperclipai worktree:make feature-x --seed-mode minimal
```

### `paperclipai worktree init`

Same as `worktree:make` minus worktree creation — for a worktree you already created by hand. Writes `.paperclip/config.json` + `.env`, allocates free ports, seeds the DB. Flags: `--name <name>` (display name to derive instance id) plus the same `--instance`, `--home`, `--from-config`, `--from-data-dir`, `--from-instance`, `--server-port`, `--db-port`, `--seed-mode` (default `minimal`), `--preserve-live-work`, `--no-seed`, `--force` as `worktree:make`. Refuses to clobber existing config/data without `--force`.

```sh
paperclipai worktree init --name feature-x
```

### `paperclipai worktree env`

Prints shell exports pointing the current shell at the worktree-local instance. Flags: `-c, --config <path>`, `--json`.

```sh
eval "$(paperclipai worktree env)"
```

Emits `PAPERCLIP_CONFIG` (always), plus `PAPERCLIP_HOME`, `PAPERCLIP_INSTANCE_ID`, `PAPERCLIP_CONTEXT` and any other entries from the worktree `.env`. `--json` returns the same keys as a flat JSON object — use that from automation.

### `paperclipai worktree:list`

Lists git worktrees and flags which look like Paperclip worktrees. Flag: `--json` (array of objects incl. `branchLabel`, `worktree` path, `isCurrent`, `hasPaperclipConfig`).

### `paperclipai worktree:merge-history [source]`

Preview/import issue + comment history from another worktree's instance into the current one. Previews by default; writes only with `--apply`.

| Flag | Type | Default | Effect |
|---|---|---|---|
| `[source]` | positional | — | Back-compat alias for `--from`. |
| `--from <worktree>` | string | — | Source: path, dir name, branch name, or `current`. |
| `--to <worktree>` | string | `current` | Target worktree. |
| `--company <id-or-prefix>` | string | — | Shared company id or issue prefix in both instances. |
| `--scope <items>` | csv | `issues,comments` | What to import. |
| `--apply` | bool | off | Apply after previewing. |
| `--dry` | bool | off | Preview only. |
| `--yes` | bool | off | Skip the apply confirmation. |

```sh
paperclipai worktree:merge-history --from paperclip-feature-x --company <company-id> --apply --yes
```

Merges Paperclip data only — never git history.

### `paperclipai worktree reseed`

Re-seeds an existing worktree-local instance's DB from another instance/worktree. **Destructive to the target DB**; confirms first; defaults to `full` seed mode (unlike make/init/repair, which default `minimal`).

| Flag | Type | Default | Effect |
|---|---|---|---|
| `--from <worktree>` | string | — | Source selector (or `current`). Mutually exclusive with `--from-config`. |
| `--to <worktree>` | string | `current` | Target worktree. |
| `--from-config` / `--from-data-dir` / `--from-instance` | string | — | Alternative source trio. |
| `--seed-mode <mode>` | enum | `full` | `minimal` or `full`. |
| `--preserve-live-work` | bool | off | Don't quarantine copied live work. |
| `--yes` | bool | off | Skip the destructive confirmation. |
| `--allow-live-target` | bool | off | Override the stopped-target-DB guard. |

```sh
paperclipai worktree reseed --from current --to paperclip-feature-x --yes
```

### `paperclipai worktree repair`

Creates or repairs a linked worktree-local instance without touching the primary checkout — for missing/broken/never-initialized `.paperclip` config. `--branch <name>` with an unregistered branch creates a worktree under `<repo>/.paperclip/worktrees/`. Flags: `--branch <name>`, `--home <path>`, `--from-config`, `--from-data-dir`, `--from-instance` (default `default`), `--seed-mode` (default `minimal`), `--preserve-live-work`, `--no-seed` (metadata-only repair), `--allow-live-target`. Safe to run from inside the broken worktree.

### `paperclipai worktree:cleanup <name>`

Removes a worktree, its branch, and its isolated instance data. Name auto-prefixed `paperclip-`. Flags: `--instance <id>`, `--home <path>`, `--force`.

```sh
paperclipai worktree:cleanup feature-x
```

### Gotchas — worktree family

- **Quarantine on seed (default):** copied timer heartbeats are disabled, running agents reset to idle, in-progress assigned issues unassigned and moved to `blocked` (with comment), routines paused. Pass `--preserve-live-work` only if you want the clone to actually run that work — otherwise duplicated agents will execute twice.
- `reseed` overwrites the target DB. Stop the target's server first; `--allow-live-target` bypasses the guard and risks corruption.
- `reseed`: pass `--from` **or** the `--from-config`/`--from-data-dir`/`--from-instance` trio, never both.
- `cleanup` refuses on uncommitted changes or unique commits; `--force` deletes unconditionally — worktree, branch, and instance data.
- After `cd` into a worktree, run `eval "$(paperclipai worktree env)"` or commands will still target `~/.paperclip`.
- Seed-mode defaults differ: make/init/repair → `minimal`, reseed → `full`.

---

## `paperclipai env-lab`

Disposable local infrastructure fixtures for adapter/host-config development — currently a single SSH server fixture. State tracked at `<instanceRoot>/env-lab/ssh-fixture/state.json`. All subcommands take `-i, --instance <id>` (default current/default instance) and `--json`.

| Subcommand | Syntax | Does |
|---|---|---|
| `up` | `paperclipai env-lab up [-i <id>] [--json]` | Start the SSH fixture; prints host, port, user, workspace, log path. |
| `status` | `paperclipai env-lab status [-i <id>] [--json]` | Show fixture state or report nothing running. |
| `down` | `paperclipai env-lab down [-i <id>] [--json]` | Stop the fixture; reports if none was running. |
| `doctor` | `paperclipai env-lab doctor [-i <id>] [--json]` | Check prerequisites + status (incl. client private key and known-hosts paths when running). |

```sh
paperclipai env-lab doctor       # run first on a new machine
paperclipai env-lab up --json
paperclipai env-lab down
```

`--json` shapes (verified in source): `up` → `{ state, environment }`; `status` → `{ running, state, environment, statePath }`; `down` → `{ stopped, statePath }`; `doctor` → `{ statePath, ssh: { supported, reason, running, state, environment } }`. `state` includes `host`, `port`, `username`, `workspaceDir`, `sshdLogPath`, `clientPrivateKeyPath`, `knownHostsPath`.

### Gotchas — env-lab

- Run `env-lab doctor` before `up` on a new machine; it names missing prerequisites instead of letting `up` fail.
- Always `env-lab down` when done — otherwise a stray SSH server stays bound to a local port.
- `doctor` can report stale state: state file exists but the process is dead ("state exists, but the process is not running").
