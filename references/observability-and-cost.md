# Observability & Cost: dashboard, cost, finance, budget, feedback

Load this file when you need to: check whole-company health from the terminal (`dashboard`),
inspect or attribute spend (`cost`), record/read finance events (`finance`), set or fix spend
guard rails (`budget`), or read/export thumbs-up/down feedback on agent work (`feedback`).
Auth, profiles, and API-base resolution are covered in `auth-and-context.md` — not repeated here.

All commands accept the standard client flags: `-C/--company-id <id>` (where noted),
`--api-base <url>`, `--api-key <token>`, `--context <path>`, `--profile <name>`,
`-c/--config <path>`, `-d/--data-dir <path>`, `--json`. Always pass `--json` when scripting.
Money values are in cents.

---

## dashboard

One subcommand. Read-only; safe to poll in a loop.

### `paperclipai dashboard get -C <company-id>`

Fetches `GET /api/companies/<id>/dashboard` — the same headline numbers as the UI dashboard.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id <id>` | string | **yes** | — | Company to summarize. Defined with `requiredOption` in source: pass it on the command line. |
| `--json` | bool | no | off | Emit the raw summary object. |

Examples:

```sh
paperclipai dashboard get -C cmp_123
paperclipai dashboard get -C cmp_123 --json | jq '.costs.monthUtilizationPercent'
```

`--json` fields:

- `companyId`
- `agents` — counts: `active`, `running`, `paused`, `error`
- `tasks` — issue counts: `open`, `inProgress`, `blocked`, `done`
- `costs` — `monthSpendCents`, `monthBudgetCents`, `monthUtilizationPercent`
- `pendingApprovals` — approvals waiting on a human
- `budgets` — `activeIncidents`, `pendingApprovals`, `pausedAgents`, `pausedProjects`
- `runActivity` — per-day array: `date`, `succeeded`, `failed`, `other`, `total`

Triage pattern: nonzero `pendingApprovals` → `approval list`; nonzero `agents.error` or
`budgets.pausedAgents` → `agent` / `activity list`; `costs.monthUtilizationPercent` climbing →
`cost` breakdowns below. `budgets.pausedAgents`/`pausedProjects` are the fastest signal that a
budget tripped and stopped automation.

### Gotchas (dashboard)

- Source registers `-C` as a `requiredOption`, so commander rejects the bare command before any
  env/profile fallback can run — always pass `-C` explicitly, even if your profile has a default
  company. (The docs mention `PAPERCLIP_COMPANY_ID`/profile fallback, but the flag is still
  required by the command definition.)
- Point-in-time read, not a stream. Use `run` / `activity` commands to follow live execution.

---

## cost

Reads of company spend rollups plus one write (`event:create`). All company-scoped commands
resolve the company from `-C`, then `PAPERCLIP_COMPANY_ID`, then the context profile; they fail
with a clear error if none is set. For the read commands, the only flag beyond the standard
client flags is `-C` (`event:create` adds `--payload-json`).

### Read commands

```sh
paperclipai cost summary        -C <company-id>   # headline total — start here
paperclipai cost by-agent       -C <company-id>   # spend per agent (spot a runaway agent)
paperclipai cost by-agent-model -C <company-id>   # per agent, per model — finest view
paperclipai cost by-provider    -C <company-id>   # by upstream provider
paperclipai cost by-biller      -C <company-id>   # by who is billed
paperclipai cost by-project     -C <company-id>   # attribute spend to projects
paperclipai cost window-spend   -C <company-id>   # spend inside current rolling budget windows
paperclipai cost quota-windows  -C <company-id>   # the configured windows/limits themselves
```

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id <id>` | string | yes* | env/profile | Target company. *Required unless resolved from `PAPERCLIP_COMPANY_ID` or profile. |
| `--json` | bool | no | off | Raw JSON for scripting. |

Budget-alert triage: `cost window-spend` (how far over?) → `cost by-agent` (who did it?).
`window-spend` is the number budget policies actually check against.

### `paperclipai cost issue <issue-id>`

End-to-end cost of one issue (`GET /api/issues/<id>/cost-summary`). Accepts the UUID or the
human identifier (e.g. `PAP-39`). **Takes no `-C`** — the issue resolves its company server-side.

```sh
paperclipai cost issue PAP-39 --json
```

### `paperclipai cost event:create -C <company-id> --payload-json '{...}'`

Records a manual cost event (POST to the company's `cost-events` endpoint) for spend the runtime
did not capture, e.g. an external one-off charge.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id <id>` | string | yes* | env/profile | Target company. |
| `--payload-json <json>` | string | **yes** | — | Cost-event object, POSTed verbatim. Parsed with `JSON.parse`; malformed JSON fails before any request. |

```sh
paperclipai cost event:create -C cmp_123 \
  --payload-json '{"kind":"external","amountCents":1250,"description":"Stock photo license"}'
```

### Gotchas (cost)

- Wrap `--payload-json` in single quotes so embedded double quotes survive the shell.
- `cost issue` is the one `cost` read without `-C`; passing `--company-id` to it is an error
  (unknown option).
- The payload shape for `event:create` is whatever the server's cost-event schema expects — the
  CLI does not validate fields, only JSON syntax.

---

## finance

Broader money facts (revenue, expenses) as opposed to model/runtime spend. All company-scoped;
same `-C` resolution as `cost`.

```sh
paperclipai finance event:create -C <company-id> --payload-json '{...}'  # write (POST finance-events)
paperclipai finance events       -C <company-id>   # list recorded finance events
paperclipai finance summary      -C <company-id>   # aggregate finance picture
paperclipai finance by-biller    -C <company-id>   # totals grouped by biller
paperclipai finance by-kind      -C <company-id>   # totals grouped by event kind
```

`finance event:create` flags are identical to `cost event:create` (`--payload-json` required).
Use it when an operator closes a deal or books an expense; use the reads to reconcile against
the `cost` side.

```sh
paperclipai finance event:create -C cmp_123 \
  --payload-json '{"kind":"revenue","amountCents":250000,"description":"Acme retainer June"}'
paperclipai finance summary -C cmp_123 --json
```

---

## budget

Guard rails that the runtime enforces while agents spend. Set budgets **before** turning agents
loose. Writes: `policy:upsert`, `company:update`, `agent:update`, `incident:resolve`. Read:
`overview`.

### `paperclipai budget overview -C <company-id>`

Configured budgets plus where current spend sits against them. Read it before and after any
budget change. Standard `-C` resolution (flag → env → profile).

### `paperclipai budget policy:upsert -C <company-id> --payload-json '{...}'`

Creates or updates the budget policy (POST to `budgets/policies`). The policy decides whether a
heartbeat may spend. `--payload-json` required. Verify afterwards with `cost quota-windows` and
`cost window-spend`.

### `paperclipai budget company:update -C <company-id> --payload-json '{...}'`

PATCHes the company-wide budget with an `UpdateBudget` payload — the ceiling regardless of which
agent spends. `--payload-json` required; company required before the command runs.

### `paperclipai budget agent:update <agent-id> --payload-json '{...}'`

PATCHes one agent's budget (`PATCH /api/agents/<id>/budgets`) with an `UpdateBudget` payload.
**Positional agent id, no `-C` flag.** `--payload-json` required. Use to give a trusted agent
more headroom or throttle an overspender.

### `paperclipai budget incident:resolve <incident-id> -C <company-id> [--payload-json '{...}']`

POSTs to the company's `budget-incidents/<id>/resolve` endpoint
(`/api/companies/<company-id>/budget-incidents/<id>/resolve`), clearing the gate a tripped
budget raised.

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `<incident-id>` | positional | **yes** | — | The budget incident to resolve. |
| `-C, --company-id <id>` | string | yes* | env/profile | Required before the command runs. |
| `--payload-json <json>` | string | no | `{}` | Optional `ResolveBudgetIncident` payload (note/disposition). |

Examples:

```sh
paperclipai budget agent:update agt_456 --payload-json '{"monthlyLimitCents":50000}'
paperclipai budget incident:resolve inc_789 -C cmp_123 \
  --payload-json '{"note":"Raised agent limit; expected spike from data backfill"}'
```

Recommended order when standing up a company: `policy:upsert` → `company:update` →
`agent:update` per agent → confirm with `budget overview` + `cost window-spend` → monitor with
`cost summary` / `by-agent` / `window-spend` → on incident: triage with `cost by-agent`, adjust
the budget, then `incident:resolve`.

### Gotchas (budget)

- **Resolving an incident clears its gate.** If the underlying overspend is still in progress,
  adjust the policy/budget first or the incident will recur on the next run.
- Company and per-agent budgets compose: tightening one agent does not raise the company
  ceiling, and the ceiling still bounds the sum of all agents.
- `budget agent:update` takes a positional agent id and **rejects `-C`**; `company:update` and
  `incident:resolve` require a resolvable company.
- `UpdateBudget` / `ResolveBudgetIncident` payload shapes are server-defined; the CLI only
  checks JSON syntax.

---

## feedback

Read-only views over **feedback traces** — one vote (`up`/`down`) against one target, with a
`payloadSnapshot` and a downloadable bundle of underlying files. The CLI never computes votes;
it observes what the server stored.

| Command | Does |
|---|---|
| `feedback report` | Terminal report: summary + per-trace details for a company. |
| `feedback export` | Write votes + full trace bundles to a folder and a sibling `.zip`. |
| `feedback trace <traceId>` | Fetch one trace (`GET /api/feedback-traces/<id>`). Not company-scoped. |
| `feedback bundle <traceId>` | Fetch one trace's bundle (trace + attached files). Not company-scoped. |

### Shared filter flags (`report` and `export` only)

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `-C, --company-id <id>` | string | no | env/profile/first company | Company whose traces to read. |
| `--target-type <type>` | string | no | all | Only traces against this target type. |
| `--vote <vote>` | string | no | all | `up` or `down`. |
| `--status <status>` | string | no | all | `pending`, `sent`, `local_only`, or `failed`. |
| `--project-id <id>` | string | no | all | Only traces tied to this project. |
| `--issue-id <id>` | string | no | all | Only traces tied to this issue. |
| `--from <iso8601>` | string | no | — | Created at or after this timestamp. |
| `--to <iso8601>` | string | no | — | Created at or before this timestamp. |
| `--shared-only` | bool | no | off | Only traces eligible for sharing/export. |

Status meanings: `pending` = awaiting export/share, `sent` = already shared off-instance,
`local_only` = not eligible to leave the instance, `failed` = an export attempt failed.

### `paperclipai feedback report -C <company-id> [filters] [--payloads]`

Prints server URL + company header, a summary (thumbs up/down, downvotes with a written reason,
total, counts per status), then per-trace detail (issue ref/title, short trace id, status, date,
target label, excerpt, reason). Extra flag: `--payloads` (bool, default off) dumps the full
`payloadSnapshot` JSON per trace — verbose, debugging only.

```sh
paperclipai feedback report -C cmp_123 --vote down --project-id prj_9 --from 2026-05-25T00:00:00Z
paperclipai feedback report -C cmp_123 --json | jq '.summary'
```

`--json` fields: `apiBase`, `companyId`, `summary` (`total`, `thumbsUp`, `thumbsDown`,
`withReason`, `statuses`), `traces` (full array).

### `paperclipai feedback export -C <company-id> [filters] [--out <path>]`

Fetches matching traces plus each trace's bundle, writes a browsable folder and a stored
(uncompressed) `.zip` next to it. Extra flag: `--out <path>` (default
`./feedback-export-<timestamp>`).

Folder layout: `index.json` (manifest: `exportedAt`, `serverUrl`, `companyId`, summary with
`uniqueIssues` + sorted `issues`, file lists), `votes/` (flattened vote record per trace:
vote, target, status, consentVersion, timestamps, reason), `traces/` (full trace objects),
`full-traces/<issue>-<trace8>/` (`bundle.json` + every bundle file at its original relative
path), `<output-dir>.zip`. File names embed the issue identifier (e.g. `PAP-39-1a2b3c4d.json`).

```sh
paperclipai feedback export -C cmp_123 --issue-id iss_42 --shared-only --out ./feedback/iss_42
paperclipai feedback export -C cmp_123 --json
```

`--json` fields: `companyId`, `outputDir` (absolute), `zipPath`, `summary` (report summary plus
`uniqueIssues`, `issues`).

### `paperclipai feedback trace <trace-id>` / `paperclipai feedback bundle <trace-id>`

Single-record fetches by id; no `-C`, no filter flags. `bundle` returns what `export` writes
into one `full-traces/` directory, without a whole-company export.

```sh
paperclipai feedback trace ft_abc123 --json
paperclipai feedback bundle ft_abc123 --json | jq '.files[].path'
```

### Gotchas (feedback)

- **Silent first-company fallback:** if no `-C`, no `PAPERCLIP_COMPANY_ID`, and no profile
  default, `report` and `export` use the **first** company returned by `/api/companies`. On a
  multi-company instance, always pass `-C` or you may report/export the wrong company.
- `export` refuses a non-empty `--out` directory (and a non-directory path) — it never clobbers
  a previous export. Scripts should generate a fresh path per run or rely on the timestamped
  default.
- The `.zip` is stored, not compressed; expect it to be roughly the folder size.
- `report --json` and `export --json` share the same base `summary` shape (export adds
  `uniqueIssues`/`issues`), so dry-run a filter with `report` before committing it to disk
  with `export`.
- Use `--shared-only` plus `--from`/`--to` when preparing feedback to leave the instance; it
  restricts output to traces the server marked shareable.
