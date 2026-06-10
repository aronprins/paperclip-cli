# Agents, Teams, and Prompt Handoff

Load this file when you need to: manage an agent's lifecycle (hire/create, configure, wake, pause, terminate), hand the CLI to a local Claude/Codex session via `agent local-cli`, push work to an agent with the prompt commands, or browse/install catalog teams with `teams`.
Execution model: agents run **server-side**. These commands trigger and observe that work over the API. Auth, profiles, and `--api-base`/`--api-key`/`--context`/`--profile` resolution are covered in `auth-and-context.md`.

All commands accept `--json` for raw API output. Company-scoped commands take `-C, --company-id <id>` (may also resolve from profile/context/env).

---

## 1. Agent identity and inbox

These read from the perspective of the *current credential* (the agent your API key/profile is scoped to).

```sh
paperclipai agent me
paperclipai agent inbox
paperclipai agent inbox-mine --user-id <board-user-id> [--status todo,in_progress]
```

| Command | Does |
|---|---|
| `agent me` | Show current agent identity (`GET /api/agents/me`). Fastest way to confirm who a key resolves to. |
| `agent inbox` | List the current agent's assigned inbox items. |
| `agent inbox-mine` | Inbox items touched/archived by a board user. `--user-id <id>` required; `--status <csv>` filters by issue status. |

With a board credential, `agent me`/`agent inbox` describe the board's own identity — usually not what you want; use an agent context.

## 2. Listing and reading agents

```sh
paperclipai agent list -C <company-id>     # all agents: role, status, reports-to, budget vs spend (cents)
paperclipai agent get <agent-id>           # one full agent record
```

`agent list` requires `-C, --company-id`.

## 3. Creating and hiring agents

```sh
paperclipai agent create -C <company-id> --payload-json '{"name":"Builder","adapterType":"codex_local"}'
paperclipai agent hire   -C <company-id> --payload-json '{"name":"CTO","role":"cto","reason":"Lead engineering"}'
```

| Command | Does |
|---|---|
| `agent create` | Create an agent immediately from a `CreateAgent` payload (`POST /api/companies/{id}/agents`). Needs board authority. |
| `agent hire` | File an agent *hire request* (`POST /api/companies/{id}/agent-hires`) that goes through board approval — does NOT create a live agent. |

Both need company scoping and `--payload-json <json>` (required). JSON is validated locally before sending. Hire requests are resolved with the `approval` commands; a pending hired agent is activated with `agent approve <agent-id>`.

## 4. Update / delete

```sh
paperclipai agent update <agent-id> --payload-json '{"title":"Senior Builder"}'   # UpdateAgent patch
paperclipai agent delete <agent-id> --yes                                         # refuses without --yes
```

`delete` removes the record entirely; prefer `agent terminate` to retire a worker.

## 5. Lifecycle controls

Each takes one `<agentId>` and POSTs to the matching endpoint. No flags beyond common ones.

```sh
paperclipai agent pause <agent-id>       # stop being woken for new work
paperclipai agent resume <agent-id>      # resume a paused agent
paperclipai agent approve <agent-id>     # approve a pending agent (e.g. from a hire request)
paperclipai agent terminate <agent-id>   # retire the agent (keeps record; contrast with delete)
```

## 6. Waking an agent

```sh
paperclipai agent wake <agent-id-or-shortname> [-C <company-id>] \
  [--reason "Pick up the new spec"] [--payload '{"issueId":"<issue-id>"}']
```

Requests a heartbeat wakeup so the server runs the agent's adapter now (`POST /api/agents/{id}/wakeup`). Accepts an agent ID **or** shortname/url-key — pass `-C` when using a shortname.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id <id>` | string | only for shortname lookup | — | Resolve shortname within this company |
| `--source <source>` | enum | no | `on_demand` | `timer`, `assignment`, `on_demand`, `automation` |
| `--trigger <trigger>` | enum | no | `manual` | `manual`, `ping`, `callback`, `system` |
| `--reason <text>` | string | no | — | Human-readable wakeup reason |
| `--payload <json>` | JSON object | no | — | Must be a JSON **object**; anything else is rejected |
| `--idempotency-key <key>` | string | no | — | Dedupe so a re-sent wakeup does not double-fire |
| `--force-fresh-session` | boolean | no | false | Fresh adapter session instead of resuming |

Related: `agent heartbeat:invoke <agent-id>` (plain, parameter-free wakeup) and the top-level `heartbeat run -a <agent-id>` (runs one heartbeat and streams live logs) converge on the same server-side wakeup path; `agent wake` is the richest. Inspect the resulting run with the `run` commands.

`paperclipai agent claude-login <agent-id>` — triggers the server-side Claude login flow for a Claude-backed adapter when its credential needs (re)establishing.

## 7. Permissions

```sh
paperclipai agent permissions:update <agent-id> --payload-json '{"canCreateAgents":true,"canAssignTasks":true}'
```

Patches scoped permissions (`PATCH /api/agents/{id}/permissions`, `UpdateAgentPermissions` payload, required). Independent of budget and instructions.

## 8. Configuration and revisions (versioned)

```sh
paperclipai agent configuration <agent-id>                          # redacted config (secrets masked, safe to print)
paperclipai agent config-revisions <agent-id>                       # revision history
paperclipai agent config-revision:get <agent-id> <revision-id>      # one revision
paperclipai agent config-revision:rollback <agent-id> <revision-id> # roll back to a prior revision
```

## 9. Runtime state and task sessions

```sh
paperclipai agent runtime-state <agent-id>
paperclipai agent runtime-state:reset-session <agent-id> [--task-key <key>]
paperclipai agent task-sessions <agent-id>
```

`reset-session` with `--task-key` resets one task session; without it, resets the agent's session broadly. Use when an agent is wedged on stale session state.

## 10. Skills (desired-state, server-side)

```sh
paperclipai agent skills <agent-id>
paperclipai agent skills:sync <agent-id> --desired-skills paperclip,github
```

`skills:sync` **replaces** the agent's desired skill list with the comma-separated `--desired-skills` (required). Required Paperclip skills stay server-enforced regardless. These do NOT install anything locally (that's `agent local-cli`); stocking the company library is the separate `skills` command family.

## 11. Instructions (path or managed bundle)

```sh
paperclipai agent instructions-path:update <agent-id> \
  --payload-json '{"path":"/tmp/AGENTS.md","adapterConfigKey":"instructionsFilePath"}'

paperclipai agent instructions-bundle <agent-id>
paperclipai agent instructions-bundle:update <agent-id> --payload-json '{"mode":"managed"}'

paperclipai agent instructions-file:get <agent-id> --path AGENTS.md
paperclipai agent instructions-file:put <agent-id> --path AGENTS.md --content-file ./AGENTS.md
paperclipai agent instructions-file:delete <agent-id> --path AGENTS.md
```

| Command | Notes |
|---|---|
| `instructions-path:update` | Process adapters require `adapterConfigKey` in the payload; relative paths require `adapterConfig.cwd`. |
| `instructions-file:put` | Content via `--content <text>` or `--content-file <path>` (prefer the file for multi-line). Optional `--clear-legacy-prompt-template` drops an old prompt template in the same write. |
| `instructions-file:get` / `:delete` | `--path <path>` required (bundle-relative). |

## 12. `agent local-cli` — hand the CLI to a local AI

```sh
paperclipai agent local-cli <agent-id-or-shortname> -C <company-id> [--key-name <name>] [--no-install-skills]
```

In one shot: (1) mints a long-lived agent API key, (2) symlink-installs bundled Paperclip skills into `~/.codex/skills` and `~/.claude/skills` (respects `CODEX_HOME`/`CLAUDE_HOME`), (3) prints shell `export` lines for `PAPERCLIP_API_URL`, `PAPERCLIP_COMPANY_ID`, `PAPERCLIP_AGENT_ID`, `PAPERCLIP_API_KEY`.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id <id>` | string | **yes** | — | Company of the agent; also resolves shortnames |
| `--key-name <name>` | string | no | `local-cli` | Label for the minted key; empty value falls back to a timestamped name |
| `--no-install-skills` | boolean | no | installs by default | Only mint the key and print exports |

```sh
# Source exports straight into the current shell
eval "$(paperclipai agent local-cli codexcoder -C <company-id> --json | jq -r .exports)"
```

`--json` output keys: `agent` (`id`, `name`, `urlKey`, `companyId`), `key` (`id`, `name`, `createdAt`, `token`), `skills` (per-tool install summaries: `linked`/`removed`/`skipped`/`failed` arrays plus `target` dir), `exports` (the shell export block as one string).

Skill installation is idempotent (repairs broken symlinks, removes maintainer-only skills) but fails hard if the bundled `skills` directory is missing from the checkout — it will not silently install nothing.

### Agent-family gotchas

- **Secret shown once:** `local-cli` prints the API key plaintext exactly once (and inside `exports`). Treat output as a secret; revoke with `token agent revoke` when done.
- **`hire` vs `create`:** `hire` files a governed request (board must approve); `create` makes a live agent immediately. Don't `create` when governance is on.
- **`delete` vs `terminate`:** `delete --yes` is irreversible removal with no further confirmation; `terminate` retires the worker and keeps the record.
- **Shortname resolution needs `-C`:** `wake` and `local-cli` accept shortnames only with a company ID to resolve against.
- **`--payload` on `wake` must be a JSON object** — arrays/strings are rejected locally.
- **`skills:sync` replaces, not appends.** Pass the full desired list every time.
- **Identity commands follow the credential**, not a target argument: `agent me`/`inbox` under a board profile describe the board.

---

## 13. Prompt handoff — `agent-prompt`, `agent prompt`, `board prompt`

**Not chat.** A handoff creates a `todo` issue assigned to the target agent (or comments on an existing one with `--issue`), then wakes the agent (`POST /api/agents/{id}/wakeup`, `source: "on_demand"`) unless `--no-wake`. The CLI returns the created issue/comment and exits; the work runs server-side. Observe via issue/run/activity commands, not the prompt output.

Default (no `--issue`) creates an issue with: `title` = `--title` or first non-empty prompt line truncated to 100 chars; `description` = full prompt; `status` = `todo`; `priority` = `medium`; `assigneeAgentId` = target. Running it three times creates **three separate tasks** — use `--issue <id>` to continue existing work.

Shared flags on all three commands:

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `--issue <issueId>` | string (UUID or `PAP-142` style) | no | — | Post prompt as a comment on this issue instead of creating a task; `--title` is ignored |
| `--title <title>` | string | no | first prompt line | Title for the new issue |
| `--no-wake` | boolean | no | wake on | Create/comment but do not wake; nothing runs until the scheduler or an explicit `agent wake` |

### `agent-prompt` (top-level command) — explicit agent key

```sh
paperclipai agent-prompt <agent> <agent-api-key> "Draft the Q3 launch checklist and link the source issues"
```

`<agent>` may be ID, shortname (`urlKey`), or name. Key is positional. The CLI verifies the key belongs to the named agent via `/api/agents/me` — a mismatch fails with the key's real identity rather than filing against the wrong agent. Best form for scripts/configs.

### `agent prompt` — as the configured agent persona

```sh
paperclipai agent prompt "Investigate the failing nightly build and report findings"
paperclipai agent prompt --api-key-env PAPERCLIP_API_KEY "Investigate the failing nightly build"
paperclipai agent prompt --profile my-agent "Investigate the failing nightly build"
```

Extra flags: `--agent <agent>` (defaults to profile/identity agent; must match the authenticated agent) and `--api-key-env <name>` (errors if the env var is unset). An agent persona can only prompt **itself** — enforced. Requires an agent profile; a board profile is refused.

### `board prompt` — board credential targets any agent

```sh
paperclipai board prompt -C <company-id> --agent <agent-name-or-id> \
  "Take over the migration rollback and post a status update when staging is green"

# Continue an in-flight task
paperclipai board prompt -C <company-id> --agent <agent-id> --issue PAP-142 \
  "Staging DB is back up — retry the migration step and confirm row counts"
```

`--agent <agent>` is **required**; `-C, --company-id <id>` scopes the company. Actor recorded on the work is the board. Requires a board profile; an agent profile is refused.

### Result object (`--json`)

| Field | Meaning |
|---|---|
| `mode` | `issue` (new task created) or `comment` (appended to existing work) |
| `actor` | `agent` or `board` |
| `agent` | Resolved target: `id`, `name`, `urlKey` |
| `issue` / `comment` | The created object |
| `wakeup` | Wakeup response, or `null` when `--no-wake` was used |

### Prompt-family gotchas

- **Persona is enforced both ways:** `agent prompt` needs an agent profile, `board prompt` needs a board profile; wrong persona is refused with a redirect message.
- **Naming overload:** `agent-prompt` (hyphen, top-level, positional key) vs `agent prompt` (subcommand, context credential). Don't confuse them.
- **`--title` is silently ignored with `--issue`.**
- **`--no-wake` means nothing runs** from this command — `wakeup: null` + `mode: "issue"` in JSON confirms staged-only work.
- Each default invocation = a new task; batch staging should use `--no-wake` to avoid a burst of runs.

---

## 14. Team catalog — `teams`

Catalog teams are ready-made starter groups (agents + projects + tasks + required skills) shipped with the app. **Bundled** teams ship inside the app; **optional** teams are extras, some pulling skills from external sources. Browsing needs no company context; `list`, `preview`, `install` require one (`-C/--company-id` or profile).

### Browse / search / inspect (read-only, no company needed)

```sh
paperclipai teams browse [--kind bundled|optional] [--category <slug>] [--query <text>]
paperclipai teams search "engineering" [--kind bundled] [--category <slug>]
paperclipai teams inspect <catalogRef> [--file TEAM.md]
```

`<catalogRef>` = catalog team ID, key, or unique slug. `inspect` shows agents, projects, tasks, required skills, and file manifest; `--file <path>` prints one package file raw to stdout (pipeable in human mode). `search` takes the query positionally; `browse` uses `--query`.

### `teams list` — catalog with installed status (company-scoped)

```sh
paperclipai teams list -C <company-id> [--kind optional] [--category <slug>] [--query <text>]
```

Per row: installed status (not installed / installed / out of date / installed-but-missing) plus agent and project counts. "Out of date" = an installed agent's recorded origin differs from the catalog team's current content; cue to reinstall.

### `teams preview` — dry-run the import (company-scoped, writes nothing)

```sh
paperclipai teams preview core-exec-team -C <company-id>
paperclipai teams preview product-engineering --target-manager-slug cto --collision-strategy rename -C <company-id>
```

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `--target-manager-agent-id <id>` | string | no | — | Existing agent the catalog's root agents report to |
| `--target-manager-slug <slug>` | string | no | — | Same, by portable slug |
| `--agent <slug>` | repeatable | no | all | Only preview selected agent slug(s) |
| `--collision-strategy <s>` | enum | no | — | `rename`, `skip`, or `replace` for existing matches |
| `--name-override <slug=name>` | repeatable | no | — | Override an imported entity's name |
| `--selected-file <path>` | repeatable | no | — | Restrict to selected portable file(s) |
| `--allow-external-sources` | boolean | no | false | Allow GitHub/URL/skills.sh skill sources |
| `--allow-unpinned-optional-sources` | boolean | no | false | Allow optional-team external sources not pinned to a commit |
| `--allow-local-path-sources` | boolean | no | false | Dev only: allow local-path skill sources |

Reports what would be created, how required skills would be prepared, and any warnings/errors.

### `teams install` — perform the import (company-scoped)

```sh
paperclipai teams install core-exec-team -C <company-id>
paperclipai teams install product-engineering --target-manager-slug cto \
  --adapter-override senior-coder=claude_local -C <company-id>
```

Accepts **all** `preview` flags, plus:

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `--secret-value <key=value>` | repeatable | no | — | Value for a secret env input the team declares |
| `--adapter-override <slug=type>` | repeatable | no | resolved at import | Pin an imported agent to an adapter (e.g. `senior-coder=claude_local`, `codex_local`) |
| `--request-approval-on-forbidden` | boolean | no | false | On `agents:create` 403, file a board approval request instead of exiting |
| `--approval-issue-id <id>` | string | no | `PAPERCLIP_TASK_ID` env when set | Issue to link to the fallback approval request |

### Teams gotchas

- **Install creates agents** and therefore needs `agents:create` permission. Without it you get a 403 unless `--request-approval-on-forbidden` (or running inside a Paperclip task with `PAPERCLIP_TASK_ID` set) converts it into a board approval request.
- **Secrets are scrubbed:** `--secret-value` values are stripped from the stored approval request and redacted in the returned result.
- **`--collision-strategy replace` overwrites existing matches** — preview first.
- **External skill sources are blocked by default**; the three `--allow-*` flags are explicit opt-ins, and `--allow-local-path-sources` is development-only.
- Adapter is deliberately not fixed per catalog agent; use `--adapter-override` to choose runtimes at install time.

---

## See also (sibling references)

- `auth-and-context.md` — profiles, board tokens, agent keys, `--api-base`/`--context` resolution, `token agent revoke`.
- CLI docs cross-links used above: `run` (inspect wakeup runs), `issue` (the tasks handoff creates), `approval` (hire requests, forbidden-install fallback), `skills` (company skill library).
