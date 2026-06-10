# Output, Scripting, and Autonomous Operation

Load this file when you are wiring `paperclipai` into a script, CI step, jq pipeline, or an
autonomous AI operator loop: it covers the `--json` output contract, exit codes and error shapes,
non-interactive invocation, polling heartbeat runs, and the `agent local-cli` playbook
(env vars `PAPERCLIP_API_URL` / `PAPERCLIP_COMPANY_ID` / `PAPERCLIP_AGENT_ID` / `PAPERCLIP_API_KEY`).
Credential creation and context profiles are covered in `auth-and-context.md`.

---

## The `--json` output contract

Every control-plane command routes through one shared renderer with exactly two modes:

| Mode | Flag | What you get |
|---|---|---|
| Human | (default) | Compact line-oriented rendering; NOT a stable contract |
| Machine | `--json` | `JSON.stringify(apiResult, null, 2)` on stdout — nothing else |

With `--json` there are no banners, labels, or color codes on stdout. The JSON is the raw API
payload. Always pass `--json` when scripting; never parse human mode.

Human-mode behavior (so you can recognize it, not parse it):
- **Arrays**: one `key=value` line per item, leading keys in order `identifier`, `id`, `name`,
  `status`, `priority`, `title`, `action`, then remaining scalars. Nested objects are dropped
  (`[object]`), strings truncated at ~90 chars, `null` renders as `-`. Empty array prints `(empty)`.
- **Single objects**: pretty JSON (nearly identical to `--json`).
- **Scalars**: printed as-is; `null`/`undefined` prints `(null)`.

```sh
paperclipai issue get PAP-39 --json
# => { "id": "244c0c2c-...", "identifier": "PAP-39", "title": "...", "status": "in_progress", "priority": "high" }
```

Note: a `204 No Content` or empty response body renders as `null`. Some endpoints that fail to
return JSON yield the raw text string instead — guard `jq` pipelines accordingly.

## Exit codes and error shapes

Standard shell convention, uniform across all client commands (single error handler):

| Outcome | Exit code | Stream | Shape |
|---|---|---|---|
| Success | `0` | stdout | rendered result (JSON with `--json`) |
| API error (HTTP error from server) | `1` | stderr | `API error <status>: <message>[ details=<json>]` |
| Any other error (bad flag, missing company ID, connection refused) | `1` | stderr | plain message text |

- The `details=<json>` suffix appears only when the server returned a structured `details` payload.
- The API error message is taken from the response body's `error` field, else `message`, else
  `Request failed with status <status>`.
- Connection failures print a multi-line message including the exact URL tried, the cause, and a
  hint to `curl <base>/api/health`. Fix is almost always API-base resolution.
- Non-zero exit always means the command did not complete. In `paperclipai ... --json | jq ...`
  the pipeline exit status is jq's — use `set -o pipefail`.

```sh
result=$(paperclipai issue get PAP-39 --json) || exit 1          # capture stdout, fail fast
paperclipai issue list -C "$CID" --json 2>/tmp/pc-errors.log      # split streams
```

## Running non-interactively

Two things can block on a prompt; both are avoidable:

1. **Interactive board re-auth.** On a `401`, or a `403` mentioning "Board access required" /
   "Instance admin required", the CLI may launch an interactive board login — but ONLY when both
   stdin and stdout are TTYs AND no explicit API key was supplied (`--api-key` or
   `PAPERCLIP_API_KEY`; the profile-named env var also counts). Passing a key disables the
   recovery path entirely, so credential problems fail fast instead of hanging.
2. **Confirmation prompts.** Destructive commands prompt in a TTY. Pass `--yes` where supported
   (e.g. `skills remove`, `skills reset`). `company delete` requires BOTH `--yes` and
   `--confirm <id>` and is additionally gated server-side.

```sh
paperclipai issue list \
  --company-id <company-id> \
  --api-base https://paperclip.example.com \
  --api-key "$PAPERCLIP_API_KEY" \
  --json
```

## Environment variables and resolution order

| Variable | Used for | Resolution order (first wins) |
|---|---|---|
| `PAPERCLIP_API_URL` | API base | `--api-base` → `PAPERCLIP_API_URL` → profile `apiBase` → local config server port → `http://localhost:3100` |
| `PAPERCLIP_API_KEY` | bearer token | `--api-key` → `PAPERCLIP_API_KEY` → env var named by profile `apiKeyEnvVarName` → stored board credential for that base |
| `PAPERCLIP_COMPANY_ID` | company scope | `--company-id` (`-C` alias on company-scoped commands) → `PAPERCLIP_COMPANY_ID` → profile `companyId` |
| `PAPERCLIP_AGENT_ID` | which agent you are (read by skills/scripts, exported by `agent local-cli` / `connect`) | env only |
| `PAPERCLIP_RUN_ID` | run id header for agent-authenticated mutations | `--run-id` → `PAPERCLIP_RUN_ID` |

If a company-scoped command resolves no company ID, it fails with:
`Company ID is required. Pass --company-id, set PAPERCLIP_COMPANY_ID, or set context profile companyId via 'paperclipai context set'.`

The run id is sent as the `X-Paperclip-Run-Id` header. Agent-authenticated mutations
(issue checkout, release, interactions, PATCH of an in-progress issue) require it — without it
the server returns `401 Agent run id required`. Inside an adapter/embodiment context
`PAPERCLIP_RUN_ID` is already exported; in your own scripts pass `--run-id`.

Profile-independent CI setup:

```sh
export PAPERCLIP_API_URL="https://paperclip.example.com"
export PAPERCLIP_API_KEY="<token>"
export PAPERCLIP_COMPANY_ID="<company-id>"
paperclipai issue list --status todo --json
```

Use `--data-dir <path>` to isolate ALL local state (config, context, db, logs, storage, secrets)
away from `~/.paperclip` — essential in CI and on shared hosts so credentials don't leak between jobs.

## jq scripting patterns

```sh
# Pull one field (always -r for raw strings)
issue_id=$(paperclipai issue create -C "$CID" \
  --title "Ship the export pipeline" --status todo --priority high \
  --json | jq -r '.id')
paperclipai issue comment "$issue_id" --body "Kicking this off." --json

# IDs of every todo issue, one per line
paperclipai issue list -C "$CID" --status todo --json | jq -r '.[].id'

# identifier + title as TSV
paperclipai issue list -C "$CID" --json | jq -r '.[] | [.identifier, .title] | @tsv'

# Count by status
paperclipai issue list -C "$CID" --json \
  | jq 'group_by(.status) | map({status: .[0].status, count: length})'

# Loop over results
paperclipai issue list -C "$CID" --assignee-agent-id "$AID" --status in_progress --json \
  | jq -r '.[].id' | while read -r id; do paperclipai issue release "$id" --json; done

# Send JSON in: --payload-json for schema-shaped commands; stdin for skill bodies
paperclipai agent create -C "$CID" \
  --payload-json "$(jq -nc '{name:"Builder", adapterType:"codex_local"}')"
cat house-style.md | paperclipai skills create --name "House Style" --slug house-style --body-file -
```

Quote interpolated payloads. An unquoted `--payload-json` with spaces or metacharacters is split
by the shell before the CLI sees it — build with `jq -nc` and double-quote the substitution.

### Robust script skeleton

```sh
#!/usr/bin/env bash
set -euo pipefail
export PAPERCLIP_API_URL="https://paperclip.example.com"
export PAPERCLIP_API_KEY="<token>"
export PAPERCLIP_COMPANY_ID="<company-id>"

issue_id=$(paperclipai issue list --status todo --json \
  | jq -r 'sort_by(.priority) | reverse | .[0].id // empty')
[ -z "$issue_id" ] && { echo "No open work." >&2; exit 0; }
paperclipai issue comment "$issue_id" --body "Picked up by the nightly script." --json >/dev/null
```

The discipline: explicit credentials, `--json` everywhere, `set -euo pipefail`, exit codes checked.

## Triggering and polling runs

`run` is overloaded: `paperclipai run` with NO subcommand bootstraps/starts a local instance
(setup action); `paperclipai run <sub>` inspects heartbeat runs via the API (control plane).

### Trigger work

```text
paperclipai agent wake <agentRef> [-C <id>] [--source <s>] [--trigger <t>] [--reason <text>]
                       [--payload <json>] [--idempotency-key <key>] [--force-fresh-session]
```
POSTs `/api/agents/:id/wakeup` and returns immediately; the work runs server-side.
`<agentRef>` is an agent ID or shortname/url-key (`-C` needed for shortname lookup).
`--source`: `timer|assignment|on_demand|automation` (default `on_demand`).
`--trigger`: `manual|ping|callback|system` (default `manual`). `--payload` must be a JSON object.

```text
paperclipai heartbeat run -a <agentId> [--source <s>] [--trigger <t>] [--timeout-ms <ms>] [--json] [--debug]
```
Runs ONE heartbeat and streams live logs to your terminal until it finishes (still executes
server-side; the CLI streams events back). `--timeout-ms` default `0` (no limit); `--debug` shows
raw adapter stdout/stderr chunks. `-a/--agent-id` is required.

### Inspect and poll

| Command | Does | Key flags |
|---|---|---|
| `run list -C <id>` | list heartbeat runs for a company | `--agent-id <id>`, `--limit <n>` |
| `run live -C <id>` | list queued + running runs | `--limit <n>`, `--min-count <n>` (pad with recent completed) |
| `run get <runId>` | one run record | — |
| `run cancel <runId>` | cancel a queued/running run | — |
| `run events <runId>` | run event stream | `--after-seq <n>` (default 0), `--limit <n>` (default 200) |
| `run log <runId>` | raw log bytes | `--offset <bytes>` (default 0), `--limit-bytes <n>`, `--text` (print only log text) |
| `run issues <runId>` | issues associated with a run | — |
| `run workspace-operations <runId>` | workspace ops for a run | — |
| `run workspace-log <operationId>` | a workspace operation log | `--offset`, `--limit-bytes`, `--text` |
| `run watchdog-decision <runId>` | record a watchdog decision | `--decision <snooze\|continue\|dismissed_false_positive>` (required), `--reason`, `--snoozed-until <iso8601>` (required for snooze), `--evaluation-issue-id` |

Run `status` values: `queued`, `scheduled_retry`, `running`, `succeeded`, `failed`, `cancelled`, `timed_out`.

```sh
# Wake an agent, then poll until the run reaches a terminal status
paperclipai agent wake "$AID" -C "$CID" --reason "nightly sweep" --json
run_id=$(paperclipai run list -C "$CID" --agent-id "$AID" --limit 1 --json | jq -r '.[0].id')
while :; do
  status=$(paperclipai run get "$run_id" --json | jq -r '.status')
  case "$status" in succeeded|failed|cancelled|timed_out) break;; esac
  sleep 10
done
echo "run $run_id finished: $status"

# Tail events incrementally
seq=0
events=$(paperclipai run events "$run_id" --after-seq "$seq" --json)
```

## Autonomous AI operator playbook

**What runs where:** the CLI does NOT run the model. Agent work (LLM calls, coding, research)
executes server-side inside the Paperclip runtime and its adapters; the CLI triggers
(`agent wake`, `heartbeat run`) and observes (`run *`). The one exception is `agent local-cli`,
which sets up YOUR machine so a local AI (Claude, Codex) runs *as* that agent and reports back
through the API.

```text
paperclipai agent local-cli <agentRef> -C <company-id> [--key-name <name>] [--no-install-skills]
```

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `<agentRef>` | arg | yes | — | agent ID or shortname/url-key |
| `-C, --company-id <id>` | string | yes | — | company scope (required even if env/profile set; it is a `requiredOption` on this command) |
| `--key-name <name>` | string | no | `local-cli` | label for the created API key |
| `--no-install-skills` | bool | no | installs | skip installing Paperclip skills into `~/.codex/skills` and `~/.claude/skills` |

In one shot it: (1) creates a long-lived agent API key, (2) symlinks the Paperclip skills for
Codex and Claude, (3) prints the four shell exports to run before launching the AI:

```sh
export PAPERCLIP_API_URL='...'
export PAPERCLIP_COMPANY_ID='...'
export PAPERCLIP_AGENT_ID='...'
export PAPERCLIP_API_KEY='...'
```

With `--json` it emits `{ agent: {id,name,urlKey,companyId}, key: {id,name,createdAt,token},
skills: [...], exports: "<the export lines>" }` — script-friendly:

```sh
eval "$(paperclipai agent local-cli "$AID" -C "$CID" --json | jq -r '.exports')"
```

**Operator loop pattern** (the AI, holding those four env vars, typically): list its assigned
issues (`issue list --assignee-agent-id "$PAPERCLIP_AGENT_ID" --json`), check out / work / comment /
release issues, post results, and poll approvals — all with `--json`, all exit-code-checked.
Agent-authenticated mutations need `PAPERCLIP_RUN_ID` / `--run-id` (see env table above).

## Gotchas

- **`run` naming overload**: bare `paperclipai run` starts a local server; `run <sub>` is the
  heartbeat-runs API client. Never bare-`run` in an operator script expecting run inspection.
- **One-time secret**: the agent API key token (from `token agent create` and `agent local-cli`)
  is shown in plaintext exactly once at creation. Capture it from the `--json` output immediately;
  it cannot be retrieved later.
- **Persona scope**: an agent key is scoped to one company + one agent. Org-building commands
  (company/agent creation, token minting) need a board credential. A `403 Board access required`
  in a script means you are running with an agent key where a board token is needed.
- **Required scoping**: `run list`, `run live`, `issue list`, etc. hard-require a company ID
  (flag, env, or profile). `agent local-cli` requires `-C` explicitly regardless.
- **`401 Agent run id required`**: agent-persona mutations (checkout/release/interactions/
  in-progress PATCH) without `--run-id`/`PAPERCLIP_RUN_ID`.
- **Destructive/irreversible**: `company delete` needs `--yes` AND `--confirm <id>` (plus a
  server-side gate); `skills remove`/`skills reset` need `--yes` non-interactively. `run cancel`
  kills in-flight work.
- **Interactive auth leak**: a command run in a TTY without an explicit key can pop a browser
  board-login on 401/403. Always set `PAPERCLIP_API_KEY` (or `--api-key`) in automation.
- **Human output is not parseable**: nested fields dropped, 90-char truncation, ordering can
  change between releases. `--json` only.
- **Attachment content types**: file uploads infer MIME from extension and the server matches the
  allowlist by EXACT string — unknown extensions or charset-suffixed types fail with
  `422 Unsupported attachment content type`.
- **Cost**: every wake/heartbeat spends real provider money server-side. Set per-agent and
  company budgets before scripting wake loops; Paperclip auto-pauses agents at 100% of budget.

See also: `auth-and-context.md` (credentials, profiles, `connect`, `token`, API-base details).
