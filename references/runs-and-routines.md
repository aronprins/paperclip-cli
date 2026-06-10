# Runs, Routines, Approvals, Activity

Load this file when you need to: observe or cancel server-side heartbeat runs (`run <sub>`, `heartbeat run`),
manage recurring/event-driven automation (`routine`, `routines disable-all`), triage or raise board approvals
(`approval`), or read/write the company audit trail (`activity`). Auth, context, profiles, and the shared client
flags (`--api-base`, `--api-key`, `--context`, `--profile`, `--data-dir`, `--json`) are covered in `auth-and-context.md`.

## Critical naming overloads (read first)

| Spelling | What it is |
|---|---|
| `paperclipai run` (bare, no subcommand) | **Local setup**: bootstraps (onboard + doctor) and starts a local Paperclip instance. NOT a control-plane command. |
| `paperclipai run <sub>` (`list`, `live`, `get`, ...) | **Control plane**: API client for observing/controlling heartbeat runs on whatever instance your context points at. |
| `paperclipai heartbeat run -a <agent-id>` | **Invoke + stream**: wakes one agent now, then polls and streams its events/logs to your terminal until terminal status. |
| `paperclipai routine ...` (singular) | Control-plane REST API for routines. Day-to-day use. |
| `paperclipai routines disable-all` (plural) | Local DB maintenance escape hatch — writes directly to the local database, bypasses the API. |
| `paperclipai routine run <id>` vs `routine runs <id>` | `run` fires the routine once now; `runs` lists past firings. One letter apart. |

Agents never execute inside the CLI. Apart from bare `run` (local setup) and `routines disable-all` (direct local DB write), every command here is a read or a request to the server.

---

## Heartbeat run commands (`run <sub>`)

Run records expose: `id`, `status`, `agentId`, `invocationSource`, `triggerDetail`, `startedAt`, `finishedAt`, `logBytes`.
Terminal statuses: `succeeded`, `failed`, `cancelled`, `timed_out`.

### run list
```
paperclipai run list -C <company-id> [--agent-id <id>] [--limit <n>]
```
Lists heartbeat runs for a company, newest activity first.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id` | id | yes* | context company | Company to list runs for. *Falls back to your context's company. |
| `--agent-id` | id | no | — | Filter to one agent. |
| `--limit` | int | no | server default | Cap result count. |

```sh
paperclipai run list -C cmp_123 --agent-id agt_456 --limit 50 --json
```
`--json` returns the array of full run objects (fields above plus error/result fields).

### run live
```
paperclipai run live -C <company-id> [--limit <n>] [--min-count <n>]
```
Lists only queued and running runs. `--min-count <n>` pads with recent *completed* runs up to that count so a quiet company isn't an empty list.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id` | id | yes* | context company | Company scope. |
| `--limit` | int | no | server default | Max runs. |
| `--min-count` | int | no | — | Pad with recent completed runs. |

```sh
paperclipai run live -C cmp_123 --min-count 5
paperclipai run live -C cmp_123 --json | jq '.[].id'   # fastest way to grab an in-flight run id
```

### run get
```
paperclipai run get <run-id>
```
Returns the full run record — the single source of truth for status, timing, trigger, and `logBytes`. No company flag needed (run id is globally addressable; same for all id-addressed subcommands below).

### run events
```
paperclipai run events <run-id> [--after-seq <n>] [--limit <n>]
```
Lists the structured event stream for a run (ordered steps the runtime took).

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `--after-seq` | int | no | `0` | Return only events after this sequence number. |
| `--limit` | int | no | `200` | Max events. |

Text mode prints `seq`, `eventType`, `stream`, `level`, `message` per event; `--json` adds the full `payload`.
To follow a live run: poll on a short interval, feeding the highest `seq` you've seen back into `--after-seq`.

```sh
paperclipai run events run_789 --after-seq 140 --limit 100
```

### run log
```
paperclipai run log <run-id> [--offset <bytes>] [--limit-bytes <bytes>] [--text]
```
Reads raw log bytes for a run. Logs are byte-addressed (offsets, not line numbers).

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `--offset` | int | no | `0` | Start byte. |
| `--limit-bytes` | int | no | server decides | Max bytes to read. |
| `--text` | bool | no | off | Print only the API's `text` field to stdout, no envelope. Best for humans/pipes. |

Tail pattern: use `logBytes` from `run get` as a cursor — read a chunk, advance `--offset` by bytes consumed, repeat.
```sh
paperclipai run log run_789 --text
paperclipai run log run_789 --offset 16384 --limit-bytes 16384
```

### run issues
```
paperclipai run issues <run-id>
```
Lists the issues this run created/touched. Rows: `identifier`, `id`, `status`, `priority`, `title`, `runStatus`. Bridge to the `issue` commands.

### run cancel
```
paperclipai run cancel <run-id>
```
Requests server-side cancellation of a queued/running run. Returns the updated run record, or `null` if there was nothing to cancel. **Does not roll back** anything the agent already did (issues moved, comments posted, workspace ops). Check `run issues <run-id>` afterward.

### run workspace-operations / run workspace-log
```
paperclipai run workspace-operations <run-id>
paperclipai run workspace-log <operation-id> [--offset <bytes>] [--limit-bytes <bytes>] [--text]
```
`workspace-operations` lists commands the run executed in its workspace — rows: `id`, `status`, `phase`, `command`, `cwd`, `logBytes`. Feed an operation `id` to `workspace-log` (same flags/defaults as `run log`).
`run log` = the run's overall log; `run workspace-log` = one command's output. When debugging a failed build/test, the real error is usually in the workspace log.

```sh
paperclipai run workspace-operations run_789
paperclipai run workspace-log wop_321 --text
```

### run watchdog-decision
```
paperclipai run watchdog-decision <run-id> --decision <d> [--reason <text>] [--snoozed-until <iso8601>] [--evaluation-issue-id <id>]
```
Records your verdict on a run the watchdog flagged as stuck/runaway.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `--decision` | enum | **yes** | — | `continue` (healthy, proceed), `snooze` (suppress until a time), `dismissed_false_positive`. |
| `--reason` | text | no | — | Free-text justification. Record it. |
| `--snoozed-until` | ISO 8601 | for `snooze` | — | When the snooze expires. |
| `--evaluation-issue-id` | id | no | — | Related watchdog evaluation issue. |

```sh
paperclipai run watchdog-decision run_789 --decision continue --reason "Long compile is expected here"
paperclipai run watchdog-decision run_789 --decision snooze --snoozed-until 2026-06-11T18:00:00Z
```
Read `run events` + `run workspace-log` before deciding `continue` — a legitimate long test run looks identical to a hang until you read the log.

### heartbeat run (invoke + stream)
```
paperclipai heartbeat run -a <agent-id> [--source <s>] [--trigger <t>] [--timeout-ms <ms>] [--debug] [--json]
```
Wakes one agent immediately (POST wakeup), then polls events + log and streams them live until the run reaches a terminal status.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-a, --agent-id` | id | **yes** | — | Agent to invoke. |
| `--source` | enum | no | `on_demand` | `timer`, `assignment`, `on_demand`, `automation`. Unrecognized values silently coerce to `on_demand`. |
| `--trigger` | enum | no | `manual` | `manual`, `ping`, `callback`, `system`. Unrecognized values silently coerce to `manual`. |
| `--timeout-ms` | int | no | `0` | Max wait; `0` = wait indefinitely. On timeout the CLI reports `timed_out` and exits 1 (the server run keeps going). |
| `--debug` | bool | no | off | Raw adapter stdout/stderr JSON chunks instead of formatted output. |
| `-c, --config`, `-d, --data-dir`, `--context`, `--profile`, `--api-base`, `--api-key`, `--json` | | no | — | Standard client/config flags. |

Exit code 1 on any non-`succeeded` terminal status (or stream ending without one). May print "Heartbeat invocation was skipped" if the server declines to wake (gating).
```sh
paperclipai heartbeat run -a agt_456 --timeout-ms 600000
```

### Run gotchas
- `-C/--company-id` is only needed by `list` and `live`; everything else addresses a run/operation id directly.
- Cancellation is a server request, not a local kill — there is no local process.
- A CLI-side `heartbeat run` timeout does not cancel the server run; use `run cancel` for that.
- `run events` and `run log` are reads; nothing here "attaches" to a process.

---

## Routine commands

A routine = payload (work to spawn) + status + revision history + triggers (`schedule`, `webhook`, `api`). Statuses include `active`, `paused`, `archived`.

| Command | Syntax | Does |
|---|---|---|
| `routine list` | `paperclipai routine list -C <company-id> [--project-id <id>]` | List a company's routines. `-C` falls back to context company. |
| `routine get` | `paperclipai routine get <routine-id>` | One routine with payload, status, triggers. |
| `routine create` | `paperclipai routine create -C <company-id> --payload-json '<json>'` | Create. `--payload-json` **required**; passed straight to the API schema. |
| `routine update` | `paperclipai routine update <routine-id> --payload-json '<json>'` | Partial patch. `--payload-json` **required**. This is how you pause: `'{"status":"paused"}'`. Every update creates a revision. |
| `routine revisions` | `paperclipai routine revisions <routine-id>` | Revision history, newest first. |
| `routine revision:restore` | `paperclipai routine revision:restore <routine-id> <revision-id>` | Restore a prior revision by creating a new latest revision from it (auditable, not a history rewind). Both args positional, routine id first. |
| `routine runs` | `paperclipai routine runs <routine-id> [--limit <n>]` | List recent firings. Use to confirm a schedule actually fires. |
| `routine run` | `paperclipai routine run <routine-id> [--payload-json '<json>']` | Fire now, regardless of schedule. Payload default `{}`, merged into run input. Execution still subject to concurrency/catch-up and heartbeat gating. |
| `routine trigger:create` | `paperclipai routine trigger:create <routine-id> --payload-json '<json>'` | Attach a trigger. Payload default `{}`. |
| `routine trigger:update` | `paperclipai routine trigger:update <trigger-id> --payload-json '<json>'` | Patch a trigger. Arg is the **trigger** id, not the routine id. Payload **required**. |
| `routine trigger:delete` | `paperclipai routine trigger:delete <trigger-id>` | Remove a trigger. |
| `routine trigger:rotate-secret` | `paperclipai routine trigger:rotate-secret <trigger-id>` | Rotate a webhook trigger's secret. Old secret stops working immediately — update the external caller in the same change. |
| `routine trigger:fire` | `paperclipai routine trigger:fire <public-id> [--payload-json '<json>']` | Fire a public (`api`) trigger by its `publicId` (shareable; doesn't expose the routine id). Payload default `{}`. |

```sh
paperclipai routine create -C cmp_123 --payload-json '{"name":"Nightly status digest","description":"Summarize closed issues","status":"active"}'
paperclipai routine trigger:create rtn_42 --payload-json '{"kind":"schedule","schedule":"0 7 * * *"}'
paperclipai routine update rtn_42 --payload-json '{"status":"paused"}'
paperclipai routine trigger:fire pub_abc --payload-json '{"event":"deploy.finished"}'
```

Trigger payload settings that matter most for schedules:
- **Concurrency**: overlap vs serialize when the next firing comes due while a run is still going.
- **Catch-up**: fire missed windows after downtime, or skip to the next due window.
A "since last run" digest wants serialized + no catch-up; an idempotent fetch can tolerate catch-up. Verify with `routine runs <id>` after deploys/outages. The trigger JSON schema (API) is the source of truth for available fields — the CLI passes payloads through verbatim.

### routines disable-all (local emergency stop)
```
paperclipai routines disable-all -C <company-id> [-c <config>] [-d <data-dir>] [--json]
```
Pauses every non-`paused`, non-`archived` routine in one company by writing **directly to the local database** (starts the embedded PostgreSQL cluster if configured). Does not go through the API.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id` | id | yes* | `PAPERCLIP_COMPANY_ID` env | Company whose routines to pause. |
| `-c, --config` | path | no | default config | Paperclip config file. |
| `-d, --data-dir` | path | no | `~/.paperclip` | Data directory root. |
| `--json` | bool | no | off | Raw result object. |

`--json` output: `{companyId, totalRoutines, pausedCount, alreadyPausedCount, archivedCount}`. Text form:
`Paused 3 routine(s) for company <id> (1 already paused, 0 archived).` Errors exit with code 1.

### Routine gotchas
- `disable-all` only affects the **local** instance. To pause routines on a remote control plane, `routine update <id> --payload-json '{"status":"paused"}'` each one.
- **No bulk re-enable exists.** After `disable-all`, turn routines back on one by one with `routine update`.
- `trigger:update`/`trigger:delete`/`trigger:rotate-secret` take a *trigger* id; `trigger:create` takes the *routine* id; `trigger:fire` takes the *public* id. Three different id spaces.
- Build JSON payloads in a file/variable, not inline by hand; invalid JSON fails at `JSON.parse` before any request is made.
- Routine-spawned heartbeat runs are inspected with the `run` commands above; routine-created issues can be found via `issue` filters (`originKind=routine_execution`).

---

## Approval commands

Governance gate for board-controlled actions. Types: `hire_agent`, `approve_ceo_strategy`.
**Persona matters**: an *agent* persona requests approvals; deciding (`approve`/`reject`/`request-revision`) is **board** persona work. See `auth-and-context.md` for switching profiles.

| Command | Syntax | Does |
|---|---|---|
| `approval list` | `paperclipai approval list -C <company-id> [--status <status>]` | List the queue. `-C` is **required** (hard CLI requirement, not context fallback). Filter e.g. `--status pending`. |
| `approval get` | `paperclipai approval get <approval-id>` | Full single approval. No company flag (id is globally addressable). Use `--json` to see the full payload (hire config, strategy text, linked issues). |
| `approval create` | `paperclipai approval create -C <id> --type <t> --payload '<json>' [--requested-by-agent-id <id>] [--issue-ids <csv>]` | Create a request for a governed action. |
| `approval approve` | `paperclipai approval approve <approval-id> [--decision-note <text>] [--decided-by-user-id <id>]` | Authorize the action. |
| `approval reject` | `paperclipai approval reject <approval-id> [--decision-note <text>] [--decided-by-user-id <id>]` | Kill the request. |
| `approval request-revision` | `paperclipai approval request-revision <approval-id> [--decision-note <text>] [--decided-by-user-id <id>]` | Send it back for changes; keeps the request alive. |
| `approval resubmit` | `paperclipai approval resubmit <approval-id> [--payload '<json>']` | Resubmit after revision, optionally replacing the payload. |
| `approval comment` | `paperclipai approval comment <approval-id> --body <text>` | Comment without deciding. `--body` required. |

`approval create` flags:

| Flag | Type | Required | Effect |
|---|---|---|---|
| `-C, --company-id` | id | **yes** | Owning company. |
| `--type` | enum | **yes** | `hire_agent` or `approve_ceo_strategy`. |
| `--payload` | JSON object | **yes** | Must parse to a JSON **object** — arrays/scalars are rejected client-side with `Invalid payload JSON`. |
| `--requested-by-agent-id` | id | no | Requesting agent. |
| `--issue-ids` | csv | no | Comma-split, trimmed, blanks dropped; links issues to the approval. |

```sh
paperclipai approval list -C cmp_123 --status pending --json
paperclipai approval approve apr_55 --decision-note "Cleared budget; proceed with the hire."
paperclipai approval request-revision apr_55 --decision-note "Tighten the role to backend only, then resubmit."
paperclipai approval resubmit apr_55 --payload '{"name":"CTO","role":"backend_lead"}'
```

Text-mode `list` rows show `id`, `type`, `status`, `requestedByAgentId`, `requestedByUserId`; `--json` returns the raw array.

### Approval gotchas
- Note the flag-name inconsistency: approvals use `--payload`; routines and activity use `--payload-json`.
- Quote `--payload` as a single shell argument (single quotes around the JSON).
- `request-revision` → fix → `resubmit` is the loop; don't re-`create` from scratch. Omitting `--payload` on resubmit reuses the existing payload.
- Use `comment` to negotiate details; reserve `request-revision` for when you actually want the proposal changed.

---

## Activity commands

The company audit trail. `list` and `issue` are read-only; `create` mutates and is rarely needed (the server writes entries automatically as work happens).

| Command | Syntax | Does |
|---|---|---|
| `activity list` | `paperclipai activity list -C <company-id> [--agent-id <id>] [--entity-type <t>] [--entity-id <id>]` | List a company's activity entries. `-C` is **required** (hard requirement). Filters combine (AND). |
| `activity create` | `paperclipai activity create -C <company-id> --payload-json '<json>'` | Record an external event (deploy, incident, milestone) into the trail. Both flags **required**; server validates as a `CreateActivity` payload. |
| `activity issue` | `paperclipai activity issue <issue-id-or-identifier>` | Activity for one issue. Accepts the human identifier (e.g. `PAP-39`). No `--company-id` — company resolves from the issue. |

Entry fields (text mode and JSON): `id`, `action` (e.g. `issue.updated`, `issue.document_locked`), `actorType` (agent/user/system), `actorId`, `entityType`, `entityId`, `createdAt`. `--json` additionally includes any stored payload. Empty result prints an empty result, not an error.

```sh
paperclipai activity list -C cmp_123 --entity-type issue --entity-id iss_77 --agent-id agt_456
paperclipai activity issue PAP-39 --json
paperclipai activity create -C cmp_123 --payload-json '{"action":"deploy.completed","entityType":"project","entityId":"prj_9"}'
```

### Activity gotchas
- Prefer `activity issue <identifier>` when you already have an issue in hand — shorter than the `--entity-type issue --entity-id` filter combo and takes `PAP-39` directly.
- Don't use `activity create` for normal operations; only for stitching genuinely external events into the trail. Bad payload shapes come back as server request errors.
