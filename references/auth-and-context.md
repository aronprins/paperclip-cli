# Auth, Tokens, Context Profiles & Access

Load this file when you need to: connect the CLI to a Paperclip instance, figure out which identity/persona a command will run as, mint or revoke board tokens / agent keys, manage `~/.paperclip/context.json` profiles, debug "wrong server / 401 / company required" errors, or run access-control commands (invites, join requests, members, instance admin, board claim).

Binary: `paperclipai`. Verify identity fast with `paperclipai whoami` (alias: `access whoami`; `auth whoami` is a separate command that hits the same endpoint).

---

## Personas and credential types

Every profile/credential acts as one of two personas:

| Persona | Credential | Scope | Minted by |
|---|---|---|---|
| `board` | Board API token | Whole instance: every company, all admin reads/writes | `token board create`, `auth login` browser flow, `connect` |
| `agent` | Agent API key | Exactly one company + one agent | `token agent create`, `connect --persona agent`, approved join request (`join claim-key`) |

- You cannot widen an agent key into board authority — mint a board token instead.
- Both credential types print the plaintext `token` **once, at creation**. There is no "show token" command; `list` subcommands return metadata only.

## Shared client flags (accepted by every API-calling command)

| Flag | Type | Effect |
|---|---|---|
| `-d, --data-dir <path>` | path | Redirect ALL local state (config, context, db, logs, storage, secrets) away from `~/.paperclip`. Use for test instances / parallel worktrees. |
| `--api-base <url>` | url | Highest-priority API base override. |
| `--api-key <token>` | string | Highest-priority credential. Also DISABLES interactive board-auth recovery — use in scripts/CI. |
| `--run-id <id>` | string | Heartbeat run id for agent-authenticated mutations (checkout/release/interactions/in-progress update); falls back to `PAPERCLIP_RUN_ID`. |
| `--context <path>` | path | Context file path (default lookup below). |
| `--profile <name>` | string | Which context profile to use (default: context's `currentProfile`). |
| `-c, --config <path>` | path | Paperclip config file; only used to infer a local server port. |
| `--json` | bool | Raw JSON output instead of human-readable. |
| `-C, --company-id <id>` | string | Only on company-scoped commands. Overrides profile's default company. |

### Resolution orders (first match wins)

**API base** (result is normalized, trailing slashes stripped):
1. `--api-base`
2. `PAPERCLIP_API_URL` env var
3. selected profile's `apiBase`
4. local config inference: `http://<PAPERCLIP_SERVER_HOST|localhost>:<PAPERCLIP_SERVER_PORT|config server.port>`
5. `http://localhost:3100`

Connection errors name the URL tried and hint at `GET /api/health`.

**Credential** (`authSource` values in parentheses):
1. `--api-key` (`explicit`)
2. `PAPERCLIP_API_KEY` env var (`env`)
3. value of the env var *named* by the profile's `apiKeyEnvVarName`, read at call time (`profile_env`)
4. stored board credential keyed by normalized apiBase, created by `auth login`/`connect` (`stored_board`)

On 401/403 (board/instance-admin required), the CLI attempts interactive board-auth recovery — but only on a TTY and only when NO credential was resolved from sources 1–3 (`--api-key`, `PAPERCLIP_API_KEY`, or the profile's env var); i.e., only when it was running on a stored board credential or none. Headless: supply a credential up front or the command fails.

**Company scope** (company-scoped commands):
1. `-C/--company-id` → 2. `PAPERCLIP_COMPANY_ID` env var → 3. profile's `companyId`.
Missing all three errors with: ``Company ID is required. Pass --company-id, set PAPERCLIP_COMPANY_ID, or set context profile companyId via `paperclipai context set`.``

**Context file path**: `--context` → `PAPERCLIP_CONTEXT` env var → nearest `.paperclip/context.json` walking up from cwd → `~/.paperclip/context.json`. Written with `0600` perms, version-2 store: `{ version: 2, currentProfile, profiles: {<name>: {...}} }`.

---

## Context profiles (`context`)

Profile fields: `apiBase`, `companyId`, `persona` (`board`|`agent`), `agentId`, `agentName`, `apiKeyEnvVarName` — plus `tokenName`/`tokenId`/`tokenCreatedAt` metadata that `connect` records about the minted token. The profile stores the **name** of the env var holding the token — never the token itself.

### `paperclipai context show`
Show resolved current context and active profile. Flags: `--context`, `--profile`, `--data-dir`, `--json`.
```sh
paperclipai context show
paperclipai context show --profile staging --json
```
JSON keys: `contextPath`, `currentProfile`, `profileName`, `profile`, `profiles`.

### `paperclipai context list`
List all profiles. JSON: array of rows `{name, current, apiBase, companyId, persona, agentId, agentName, apiKeyEnvVarName}`.

### `paperclipai context use <profile>`
Switch the active profile. Positional profile name required.
```sh
paperclipai context use prod
```

### `paperclipai context set`
Create or update a profile field by field (creates it if missing).

| Flag | Required | Default | Effect |
|---|---|---|---|
| `--profile <name>` | no | current profile, else `default` | Profile to edit |
| `--api-base <url>` | no | — | Default API base |
| `--company-id <id>` | no | — | Default company |
| `--persona <persona>` | no | — | `board` or `agent`; anything else errors |
| `--agent-id <id>` | no | — | Agent persona: agent id |
| `--agent-name <name>` | no | — | Agent display name |
| `--api-key-env-var-name <name>` | no | — | Env var name that holds the token (recommended) |
| `--use` | no | false | Also make this profile active |

```sh
# Board profile reading its token from PAPERCLIP_API_KEY
paperclipai context set --profile prod --api-base https://paperclip.example.com \
  --persona board --api-key-env-var-name PAPERCLIP_API_KEY --use

# Agent profile pinned to a company + agent
paperclipai context set --profile acme-bot --api-base https://paperclip.example.com \
  --company-id <company-id> --persona agent --agent-id <agent-id> --agent-name "Acme Bot"
```
JSON keys: `contextPath`, `currentProfile`, `profileName`, `profile`.

**Gotchas (context):**
- Passing an empty value for a field clears it from the profile.
- `context set` is the headless replacement for `connect` — same persona-aware profile shape.
- A `.paperclip/context.json` in a parent directory silently wins over `~/.paperclip/context.json` (cwd walk-up). If commands behave differently per directory, check `context show`'s `contextPath`.

---

## Connect wizard (`connect`)

### `paperclipai connect`
Interactive one-shot setup: confirm API base (placeholder `http://localhost:3100`) → health check (`GET /api/health`) → interactive board login → pick persona → name profile → pick company (+agent for agent persona) → mint token → save profile, set it active → print shell exports.

| Flag | Required | Default | Effect |
|---|---|---|---|
| `--persona <persona>` | no | prompted | `board` or `agent`; skips the persona prompt |
| `--api-key-env-var-name <name>` | no | `PAPERCLIP_API_KEY` | Env var name recorded in the profile |
| `--token-name <name>` | no | `cli-board-<timestamp>` / `cli-agent-<timestamp>` | Label for the minted token |

```sh
paperclipai connect
paperclipai connect --persona agent --token-name laptop-cli
```

`--json` result keys: `ok`, `profile` (the profile name), `persona`, `apiBase`, `companyId` (and `agentId`/`agentName` for agent), created `key` (`id`, `name`, `createdAt`, `token`, plus `expiresAt` for board), and an `exports` block:
```sh
export PAPERCLIP_API_URL='https://paperclip.example.com'
export PAPERCLIP_COMPANY_ID='<company-id>'   # only when applicable
export PAPERCLIP_AGENT_ID='<agent-id>'       # agent persona only
export PAPERCLIP_API_KEY='<token-shown-once>'
```

**Gotchas (connect):**
- TTY-only. In CI/scripts it refuses and tells you to use `--api-base`/`--api-key` or `context set` + `token` commands.
- Passing `--api-key` does NOT make it non-interactive — it always runs the interactive board login. For scripted setup: mint a token another way, then `context set`.
- Board persona: company optional (`(none)` allowed). Agent persona: company required and must contain at least one agent.
- The plaintext token appears only in the wizard output / `exports` block. The saved profile only references the env var name.

---

## Board auth (`auth`)

Board credentials are stored locally keyed by normalized `apiBase` — one machine can hold separate credentials for local/staging/prod. Logging in as a different user for the same apiBase replaces the existing entry.

### `paperclipai auth login`
Device-code style browser approval: CLI creates a challenge, opens the approval URL, polls until approved/cancelled/expired; on approval confirms via `/api/cli-auth/me` and stores the board token for this apiBase.

| Flag | Required | Default | Effect |
|---|---|---|---|
| `--instance-admin` | no | false | Request instance-admin approval (approver must be an instance admin) |
| `-C, --company-id <id>` | no | — | Scope the requested access to one company |

```sh
paperclipai auth login
paperclipai auth login --instance-admin --api-base https://paperclip.example.com
```
JSON: `{ ok, apiBase, userId, approvalUrl }`. Cancel → exits `CLI auth challenge was cancelled.`; timeout → `CLI auth challenge expired before approval.` (just re-run).

### `paperclipai auth whoami`
Board-user identity for this apiBase (`GET /api/cli-auth/me`).
JSON: `{ user: {id, name, email} | null, userId, isInstanceAdmin, companyIds, source, keyId }`.

### `paperclipai auth logout`
Attempts server-side revoke of the stored token, then deletes the local entry. Local entry is removed even if the revoke fails. JSON: `{ ok, apiBase, revoked, removedLocalCredential }`. No stored credential → `revoked: false`, exits cleanly. Rotate = `logout` then `login`.

### `paperclipai auth revoke-current`
Server-side revoke of the board token currently in use (`POST /api/cli-auth/revoke-current`), without touching the local store.

### `paperclipai auth challenge <create|get|approve|cancel>`
Low-level challenge lifecycle for scripted approval flows. `auth login` orchestrates these normally.
```sh
paperclipai auth challenge create --payload-json '{"command":"ci-bot","requestedAccess":"board"}'
paperclipai auth challenge get <challenge-id> --token <secret>
paperclipai auth challenge approve <challenge-id> --token-env CHALLENGE_TOKEN
paperclipai auth challenge cancel <challenge-id> --token <secret>
```
- `create` requires `--payload-json` (a `CreateCliAuthChallenge` payload).
- `get`/`approve`/`cancel` take `<id>` positional and require the challenge secret via `--token <secret>` or `--token-env <name>`; missing both errors with `Challenge secret is required. Pass --token or --token-env.`

### `paperclipai auth bootstrap-ceo`
Local (database-level, not API) command: creates a one-time bootstrap invite URL for the first instance admin on a fresh `authenticated`-mode instance. Prints `Invite URL: <base>/invite/<token>` and its expiry.

| Flag | Required | Default | Effect |
|---|---|---|---|
| `-c, --config <path>` | no | resolved config | Paperclip config file |
| `-d, --data-dir <path>` | no | `~/.paperclip` | Data dir root |
| `--force` | no | false | Create a new invite even if an admin already exists |
| `--expires-hours <hours>` | no | 72 (clamped 1–720) | Invite expiry window |
| `--base-url <url>` | no | inferred from config | Public base URL used in the printed link |

```sh
paperclipai auth bootstrap-ceo
paperclipai auth bootstrap-ceo --force --expires-hours 24 --base-url https://paperclip.example.com
```

**Gotchas (auth):**
- `bootstrap-ceo` is a no-op (informational message) when deployment mode is `local_trusted`, and it needs direct DB access — start the server first when using embedded-postgres. It revokes any previous outstanding bootstrap invite. No `--json` support.
- Interactive auth recovery (mid-command re-login on 401/403) only happens on a TTY and never when `--api-key` was passed.
- `auth login` mints/stores a board credential per apiBase; there is no command to print the stored token back out.

---

## Named tokens (`token`)

### Board keys (`token board ...`) — full board authority, named, expirable, audited

`paperclipai token board create` — mint a named key for the current board user (requires an existing board credential).

| Flag | Required | Default | Effect |
|---|---|---|---|
| `-C, --company-id <id>` | no | env/profile company if set | Audit-context company only — does NOT scope the key |
| `--name <name>` | no | `cli-board` | Key label |
| `--expires-at <iso8601>` | no | — | Explicit expiry; invalid date rejected |
| `--ttl-days <days>` | no | — | Expiry N days from now; must be positive |
| `--never-expires` | no | false | Non-expiring key |

Expiry precedence: `--never-expires` > `--expires-at` > `--ttl-days` > server default.
```sh
paperclipai token board create --name ci-deploy --ttl-days 90
paperclipai token board create --name nightly --expires-at 2026-12-31T00:00:00Z
```
JSON: `{ key: { id, name, token, createdAt, lastUsedAt, revokedAt, expiresAt } }` — `token` shown once.

`paperclipai token board list` — keys for the current board user. JSON: array of `{id, name, createdAt, lastUsedAt, expiresAt, revokedAt}`. Never includes tokens.

`paperclipai token board revoke <keyId>` — revoke by key ID. JSON: `{ ok, keyId }`.

### Agent keys (`token agent ...`) — scoped to one company + one agent

All three subcommands REQUIRE `-C, --company-id <id>` and `--agent <agent>` (agent ID, shortname, or unambiguous name; resolved inside the company first — wrong company / typo fails fast with `Agent not found`).

`paperclipai token agent create`

| Flag | Required | Default | Effect |
|---|---|---|---|
| `-C, --company-id <id>` | yes | — | Company the agent belongs to |
| `--agent <agent>` | yes | — | Agent id / shortname / unambiguous name |
| `--name <name>` | no | `cli-agent` | Key label |

```sh
paperclipai token agent create --company-id <company-id> --agent <agent> --name ci-runner
```
JSON: `{ agentId, agentName, companyId, key: { id, name, token, createdAt } }` — `token` shown once.

`paperclipai token agent list --company-id <id> --agent <agent>` — JSON: `{ agentId, companyId, keys: [...] }`; key rows carry `id`, `name`, `createdAt`, `revokedAt` (and `lastUsedAt` when the server includes it). Human output shows `id`, `name`, `createdAt`, `revokedAt` only. Never tokens.

`paperclipai token agent revoke <keyId> --company-id <id> --agent <agent>` — JSON: `{ ok, agentId, companyId }` plus `keyId` when the server's delete response includes it.

**Gotchas (token):**
- The secret `token` is returned exactly once at creation; `list` returns metadata only. Capture with `--json` in scripts.
- `-C` on `token board create` is audit context only; the key still has instance-wide board authority. `-C` on `token agent *` is a hard requirement and actually scopes resolution.
- Revocation is immediate and irreversible — mint a new key rather than trying to "un-revoke".
- For a full local agent setup (key + skills + exports), prefer `agent local-cli <agent>` over hand-minting (see the agent reference).

---

## Access, identity & instance admin

Most of these are thin wrappers over API endpoints; all accept the shared client flags. Company-scoped ones take `-C/--company-id` (falls back to profile company). `--payload-json` bodies are sent verbatim.

### Health & identity
| Command | Endpoint | Notes |
|---|---|---|
| `paperclipai health` | `GET /api/health` | Reachability check |
| `paperclipai whoami` | `GET /api/cli-auth/me` | Current persona/user/scope. Alias: `access whoami` |
| `paperclipai openapi` | `GET /api/openapi.json` | Always JSON regardless of `--json` |

Run `whoami` before any state-changing command when unsure which persona/company is active.

### Profile
| Command | Endpoint | Notes |
|---|---|---|
| `profile session` | `GET /api/auth/get-session` | Raw session |
| `profile get` | `GET /api/auth/profile` | Your profile |
| `profile update` | `PATCH /api/auth/profile` | Requires `--payload-json` |
| `profile company-user <userSlug>` | `GET /api/companies/<C>/users/<slug>/profile` | Company-scoped |

### Invites
```sh
paperclipai invite create --company-id <company-id> --payload-json '{"role":"member"}'
paperclipai invite accept <token> --payload-json '{}'
```
| Command | Notes |
|---|---|
| `invite list` | Company-scoped |
| `invite create` | Company-scoped, requires `--payload-json` |
| `invite revoke <inviteId>` | Takes the **invite ID** |
| `invite show <token>` / `invite logo <token>` | Recipient view, by **token** |
| `invite onboarding <token>` / `invite onboarding:text <token>` | Structured / plain-text onboarding |
| `invite skills:index <token>` / `invite skill <token> <skillName>` | Skills offered to invitee |
| `invite accept <token>` | `--payload-json` defaults to `{}` |
| `invite test-resolution <token> --url <url>` | Diagnostic; `--url` required |

### Join requests
```sh
paperclipai join list --company-id <company-id> --status pending
paperclipai join claim-key <request-id> --claim-secret <secret>
```
| Command | Notes |
|---|---|
| `join list` | Company-scoped; filters `--status` (`pending_approval`/`approved`/`rejected`; `pending` normalized to `pending_approval`) and `--request-type` |
| `join approve <requestId>` / `join reject <requestId>` | Company-scoped |
| `join claim-key <requestId>` | Requires `--claim-secret`; returns an **agent API key** — treat output as a secret |

### Members
All company-scoped: `member list`, `member user-directory`, `member update <memberId>` (PATCH, `--payload-json` required), `member role-and-grants <memberId>`, `member permissions <memberId>` (both `--payload-json` required), `member archive <memberId>` (`--payload-json` defaults `{}`).

### Instance admin (board/instance-admin authority required, instance-wide)
| Command | Notes |
|---|---|
| `admin user list` | Optional `--query` text search |
| `admin user promote <userId>` / `admin user demote <userId>` | Grant/revoke instance-admin |
| `admin user company-access <userId>` | Read company access |
| `admin user company-access:update <userId>` | PUT, requires `--payload-json`, **replaces** access |
| `instance scheduler-heartbeats` | List scheduler heartbeats |
| `instance settings:general` / `instance settings:general:update` | Update requires `--payload-json` |
| `instance settings:experimental` / `instance settings:experimental:update` | Instance-wide flags — read first, patch one flag at a time |
| `instance database-backup` | Triggers a backup |

### Sidebar & inbox (personal preferences, not policy)
`sidebar preferences`, `sidebar preferences:update` (`--payload-json`), `sidebar project-preferences` / `:update` (company-scoped), `sidebar badges` (company-scoped), `inbox dismissals`, `inbox dismiss --payload-json '{"itemKey":"run:<run-id>"}'` (both company-scoped).

### Board claim & OpenClaw
| Command | Notes |
|---|---|
| `board-claim show <token>` | Inspect a pending claim without acting |
| `board-claim claim <token>` | `--payload-json` defaults `{}`. One-time: migrates instance ownership to your user |
| `openclaw invite-prompt` | Company-scoped, requires `--payload-json` |

Board claim only exists on a fresh `authenticated` instance whose only admin is the `local-board` placeholder; the server prints `http://<host>/board-claim/<token>?code=<code>` to its log and refreshes the 24h challenge.

### Public skills & LLM docs
`available-skill list` / `available-skill index` / `available-skill get <skillName>`; `llm agent-configuration`, `llm agent-configuration:adapter <adapterType>`, `llm agent-icons` — server-published docs for configuring agents; safe, read-only.

**Gotchas (access):**
- Naming overload: `invite revoke` takes the invite **ID**; every other invite subcommand takes the recipient **token**. Not interchangeable.
- `join claim-key` and `board-claim claim` output/perform secrets and one-shot operations: the claimed agent key is a credential, and a board claim consumes the challenge permanently.
- `admin user company-access:update` is a PUT — it replaces the user's whole access list, not a merge.
- `member archive`, `admin user promote/demote`, and `instance settings:*:update` change live access/behavior; verify identity with `whoami` first.

---

## Typical sequences

Fresh interactive machine:
```sh
paperclipai connect            # wizard: profile + token + exports
paperclipai auth whoami        # verify
paperclipai token board create --name ci --ttl-days 30
paperclipai context use prod
```

Headless (no TTY): `auth login` from another machine or `auth bootstrap-ceo` / board claim on a fresh instance, then `context set` + export the key into the env var named by `apiKeyEnvVarName`.
