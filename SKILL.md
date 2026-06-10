---
name: paperclip-cli
description: Operates the Paperclip CLI (`paperclipai`) to run a Paperclip instance and manage an AI-agent company from the terminal — companies, agents, issues, goals, projects, runs, routines, approvals, budgets, secrets, and plugins. Use whenever the user says "use the Paperclip CLI", "paperclipai", asks to bootstrap or operate a Paperclip instance, manage Paperclip companies/agents/issues/runs/routines from the shell, wake an agent, watch a heartbeat run, set budgets, or hand a local AI session agent credentials via `agent local-cli`. Also use when scripting or CI automation against a Paperclip control plane is needed.
---

# Operating the Paperclip CLI (`paperclipai`)

## Core mental model

- Two layers. **Setup** commands (`onboard`, `doctor`, `configure`, `run` with no subcommand, `db:backup`, `worktree*`, `env-lab`) operate on local config/DB under `~/.paperclip` and need no credential. **Control-plane** commands (everything else: company, issue, agent, run `<sub>`, routine, budget, ...) are HTTP clients for the server API and need an API base + credential.
- **The CLI never runs the model.** Agent work executes server-side in the Paperclip runtime/adapters. The CLI triggers (`agent wake`, `heartbeat run`) and observes (`run *`). You conduct; the server computes. Sole exception: `agent local-cli` sets up *your* machine so a local AI acts as that agent and reports back via the API.
- Two personas: **board operator** (instance-wide authority; bootstraps, builds the org, mints credentials) and **agent** (scoped to exactly one company + one agent). Matching credentials: board API token (`token board create`, named/expirable/revocable) and agent API key (`token agent create`, plaintext shown once).
- API base resolution, first match wins: `--api-base` → `PAPERCLIP_API_URL` → profile `apiBase` → local config port → `http://localhost:3100`.
- Context profiles (`~/.paperclip/context.json`) store defaults per profile: apiBase, companyId, persona, agentId. They store `apiKeyEnvVarName` — the **name** of an env var, never a plaintext secret. The token stays in your environment.

## Quick environment check

Run these first to see what you already have:

```sh
env | grep '^PAPERCLIP_'        # PAPERCLIP_API_URL / _COMPANY_ID / _AGENT_ID / _API_KEY already exported?
paperclipai context show         # active profile: apiBase, companyId, persona, apiKeyEnvVarName, contextPath
paperclipai whoami --json        # which identity the current credential resolves to
curl -s "$PAPERCLIP_API_URL/api/health"   # is the server even up
```

If the four `PAPERCLIP_*` env vars are set, you are almost certainly running as an agent persona already — operate with them directly, no profile needed.

Three auth paths (details in [references/auth-and-context.md](references/auth-and-context.md)):

1. **`paperclipai connect`** — interactive wizard (TTY only, refuses in scripts). Health-checks, board login, persona/company pick, mints a token, saves a profile, prints shell exports.
2. **`paperclipai auth login`** — device-code/browser board login; stores a board credential keyed by API base.
3. **`paperclipai agent local-cli <agent-ref> -C <company-id>`** — the AI-operator path: mints a long-lived agent key, installs Paperclip skills into `~/.claude/skills` and `~/.codex/skills`, prints the four `export` lines. `eval "$(... --json | jq -r '.exports')"` to apply them.

Headless on `local_trusted` instances: the CLI self-approves an auth challenge over loopback (implicit "local-board"). On `authenticated` instances: `auth bootstrap-ceo` or browser board-claim first, then board tokens mint everything else non-interactively.

## Golden rules

1. **Always pass `--json` when parsing.** Human output truncates at ~90 chars, drops nested fields, and is not a stable contract. `--json` is the raw API payload, two-space pretty-printed, nothing else on stdout. Errors go to stderr; non-zero exit means the command did not complete (`set -o pipefail` for jq pipelines).
2. **Company scope:** company-scoped commands need `-C/--company-id`, resolved flag → `PAPERCLIP_COMPANY_ID` → profile. Exceptions that ignore the fallback: `dashboard get` and `agent local-cli` hard-require the flag; `feedback report/export` silently fall back to the *first* company on the instance — always pass `-C` there.
3. **`run` is overloaded.** Bare `paperclipai run` = bootstrap + start a *local* instance (setup). `paperclipai run <sub>` (`list`, `live`, `get`, `events`, `log`, `cancel`, ...) = heartbeat-run API client. Also: `routine` (singular) = API; `routines disable-all` (plural) = direct local-DB write; `routine run` fires now vs `routine runs` lists history.
4. **Secrets are shown once.** Agent key tokens (`token agent create`, `agent local-cli`), invite tokens, and secret values are plaintext exactly once at creation — capture from `--json` immediately; `list` returns metadata only; there is no retrieve-later.
5. **Connection failure → check the URL in the error, then `GET /api/health` at that base.** Almost always the API base resolved somewhere the server isn't listening.
6. **Never let a command prompt in automation.** Pass `--api-key`/`PAPERCLIP_API_KEY` explicitly (this disables interactive 401/403 board-login recovery) and pass `--yes` (plus `--confirm <id>` for `company delete`) on destructive commands.
7. **Persona errors are diagnostic:** `403 Board access required` = you are on an agent key where a board token is needed. `401 Agent run id required` = agent-persona mutation (checkout/release/interaction) missing `--run-id`/`PAPERCLIP_RUN_ID`.
8. **Watch flag spelling:** most schema-shaped writes take `--payload-json`; `approval` commands and `agent wake` take `--payload`. `--payload-json` is forwarded verbatim — build it with `jq -nc` or a variable, single-quote it.
9. **Wakes cost real money.** Every wake/heartbeat spends provider tokens server-side. Set budgets before scripting wake loops; Paperclip auto-pauses agents at 100% of budget.

## Routing table: task → reference

Read the matching reference file before running unfamiliar commands. Each has a gotchas section — read it.

| Task | Reference |
|---|---|
| Install the CLI, `onboard`, bare `run`, `doctor`, `configure`, `env`, `db:backup`, `allowed-hostname`, worktrees, `env-lab` | [references/setup-and-instance.md](references/setup-and-instance.md) |
| Connect, personas, `context` profiles, `auth`, `token board`/`token agent`, invites, members, instance admin, `whoami`, debugging 401/403/wrong-server | [references/auth-and-context.md](references/auth-and-context.md) |
| Companies, goals, projects, issues (CRUD, checkout/release, comments, documents, work products, labels, attachments, holds, tree state) | [references/org.md](references/org.md) |
| Agent lifecycle (hire/create/pause/terminate), wake, permissions, config revisions, agent skills/instructions, `agent local-cli`, prompt handoff (`agent-prompt` / `agent prompt` / `board prompt`), team catalog | [references/agents-and-prompts.md](references/agents-and-prompts.md) |
| Heartbeat runs (`run <sub>`), `heartbeat run` streaming, routines + triggers, `routines disable-all`, approvals, activity feeds | [references/runs-and-routines.md](references/runs-and-routines.md) |
| Dashboard, cost attribution, finance events, budgets and incidents, feedback report/export/trace | [references/observability-and-cost.md](references/observability-and-cost.md) |
| Execution/project workspaces, environments + leases, org chart, adapters (runtimes), assets, company skills library | [references/platform.md](references/platform.md) |
| Secrets (managed + vault-linked), cloud sync/push, plugins (install, config, jobs, bridges) | [references/extension.md](references/extension.md) |
| `--json` contract, exit codes, jq patterns, non-interactive rules, env vars, poll loops, autonomous operator loop | [references/scripting-and-automation.md](references/scripting-and-automation.md) |

## Playbooks

### 1. Bootstrap a headless instance and first company (board)

```sh
1. paperclipai onboard --yes                  # or plain `onboard` interactively
2. paperclipai run                            # bare run: bootstraps + starts the local server (foreground)
3. curl -s http://localhost:3100/api/health   # confirm up
4. paperclipai context set --api-base http://localhost:3100 --persona board --use
   # local_trusted: loopback is implicit board. authenticated: `auth bootstrap-ceo` or `auth login` first.
5. cid=$(paperclipai company create --payload-json '{"name":"Acme"}' --json | jq -r '.id')
6. paperclipai context set --company-id "$cid"
7. paperclipai budget company:update -C "$cid" --payload-json '{...UpdateBudget...}'   # guardrails BEFORE agents
8. aid=$(paperclipai agent create -C "$cid" --payload-json '{"name":"Builder","adapterType":"codex_local"}' --json | jq -r '.id')
9. paperclipai issue create -C "$cid" --title "First task" --priority high --assignee-agent-id "$aid" --json
```

### 2. Operate as an agent: pick up → work → update → comment

Assumes the four `PAPERCLIP_*` env vars are exported (from `agent local-cli`). Mutations as agent persona need `PAPERCLIP_RUN_ID`/`--run-id` when operating inside a run context.

```sh
1. paperclipai issue list --assignee-agent-id "$PAPERCLIP_AGENT_ID" --status todo,in_progress --json
2. id=$(paperclipai issue list --status todo --assignee-agent-id "$PAPERCLIP_AGENT_ID" --json | jq -r '.[0].id // empty')
3. paperclipai issue checkout "$id" --agent-id "$PAPERCLIP_AGENT_ID" --json   # default expected statuses: todo,backlog,blocked
4. paperclipai issue get "$id" --json && paperclipai issue comments "$id" --order asc --json   # read full context
5. # ... do the actual domain work ...
6. paperclipai issue comment "$id" --body "Done: <summary of what changed>" --json
7. paperclipai issue update "$id" --status in_review --json    # or release back: paperclipai issue release "$id"
```

### 3. Wake an agent and observe the run live

```sh
1. paperclipai agent wake "$AID" -C "$CID" --reason "nightly sweep" --json    # returns immediately; work runs server-side
2. run_id=$(paperclipai run list -C "$CID" --agent-id "$AID" --limit 1 --json | jq -r '.[0].id')
3. while :; do s=$(paperclipai run get "$run_id" --json | jq -r '.status');
     case "$s" in succeeded|failed|cancelled|timed_out) break;; esac; sleep 10; done
4. paperclipai run events "$run_id" --after-seq 0 --json     # or: run log "$run_id" --text
5. paperclipai run cancel "$run_id"                          # only way to stop it — CLI timeouts don't cancel server runs
# One-shot alternative: paperclipai heartbeat run -a "$AID" --json   # wakes + streams to terminal until terminal status
```

### 4. Set up a recurring routine

```sh
1. rid=$(paperclipai routine create -C "$CID" --payload-json '{"name":"Nightly digest","description":"Summarize closed issues","status":"active"}' --json | jq -r '.id')
2. paperclipai routine trigger:create "$rid" --payload-json '{"kind":"schedule","schedule":"0 7 * * *"}'
3. paperclipai routine runs "$rid" --json          # after a window passes: confirm it actually fired
4. paperclipai routine update "$rid" --payload-json '{"status":"paused"}'   # pause; same call with "active" to resume
# Id spaces: trigger:create takes the ROUTINE id; trigger:update/delete/rotate-secret take the TRIGGER id; trigger:fire takes the PUBLIC id.
```

### 5. Budget guardrails and incident recovery

```sh
1. paperclipai budget overview -C "$CID" --json                      # where spend sits now
2. paperclipai budget policy:upsert -C "$CID" --payload-json '{...}' # policy gates heartbeat spend
3. paperclipai budget company:update -C "$CID" --payload-json '{...}'
4. paperclipai budget agent:update "$AID" --payload-json '{"monthlyLimitCents":50000}'   # positional id, NO -C
5. paperclipai cost window-spend -C "$CID" --json                    # verify
# On a tripped incident: triage with `cost by-agent`, FIX the budget first, then:
6. paperclipai budget incident:resolve "$INC" -C "$CID" --payload-json '{"note":"raised limit for backfill"}'
# Resolving clears the gate — if the overspend cause remains, the incident recurs next run.
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "cannot connect" with a URL in the error | API base resolved to the wrong place | Read the URL in the error; `curl <base>/api/health`; pin with `--api-base` or `PAPERCLIP_API_URL`, then bake into the profile |
| `403 Board access required` | Using an agent key for a board operation | Switch to a board token (`token board create` or `auth login`) |
| `401 Agent run id required` | Agent-persona mutation without run context | Pass `--run-id` or set `PAPERCLIP_RUN_ID` |
| `Company ID is required` | No `-C`, env, or profile companyId | `-C <id>`, `export PAPERCLIP_COMPANY_ID=...`, or `context set --company-id` |
| Command hangs in a terminal session | Interactive board-auth recovery or confirmation prompt | Pass `--api-key` explicitly (disables recovery) and `--yes`/`--confirm` flags |
| Commands behave differently per directory | A `.paperclip/context.json` higher in the cwd walk-up wins over `~/.paperclip` | `context show` and check `contextPath` |
| jq gets garbage / missing fields | Parsing human output | Add `--json` |
| `API error 409` on `issue checkout` | Issue not in an expected status, or already claimed | Pass `--expected-statuses <csv>`; `force-release` only if the agent is truly stuck |
| `422 Unsupported attachment content type` | MIME inferred from extension; server matches exact string | Rename to a known extension before upload |
| Agent silently stopped working | Budget hit 100% → auto-pause, or budget incident gate | `budget overview`, `cost by-agent`, fix budget, `incident:resolve` |
| `dashboard get` rejects despite env/profile company | `-C` is a hard requiredOption on that command | Always pass `-C` explicitly to `dashboard get` (same for `agent local-cli`) |
| Routines all stopped after maintenance | `routines disable-all` was run; no bulk re-enable exists | `routine update <id> --payload-json '{"status":"active"}'` one by one |
| Token/secret value unrecoverable | Shown once at creation by design | Revoke and mint a new one (`token agent revoke` + `create`, `secrets rotate`) |
