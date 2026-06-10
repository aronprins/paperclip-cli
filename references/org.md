# Company, Project, Goal, Issue

Load this file when creating or managing the org hierarchy via `paperclipai`: companies (top-level container), goals (strategic objectives), projects (work groupings), and issues (the core work objects, including comments, documents, checkout, holds, attachments, labels). For credentials, context resolution, and shared flags (`--api-base`, `--api-key`, `--context`, `--profile`, `--data-dir`, `--json`), see auth-and-context.md — those flags work on every command below and are not repeated.

Hierarchy: company > goal (nestable tree) > project (links to goals via `--goal-ids`) > issue (optional `projectId`, `goalId`, `parentId`). Goal resolution for an issue: issue's own goal → project's goal → company default goal.

---

## Company

Top-level container. Board credential sees all companies; an agent credential is scoped to exactly one. `--json` returns the full server record; human output is summarized.

### Read

```sh
paperclipai company list                 # all visible companies (agent cred: only its own)
paperclipai company get <company-id>     # full record for one company
paperclipai company current              # company in scope (from -C / context / PAPERCLIP_COMPANY_ID / agent cred) — no ID needed
paperclipai company stats                # instance-level stats (GET /api/companies/stats)
```

`company list` human output: `id`, `name`, `status`, monthly budget and spend (cents), whether new agents require board approval.

### Create / update / branding — raw JSON passthrough

The CLI does NOT validate these payloads; the server applies `CreateCompany` / `UpdateCompany` / `UpdateCompanyBranding` schemas. Bad fields fail server-side — add `--json` and read the response on rejection.

| Command | Route | Flags |
|---|---|---|
| `company create` | `POST /api/companies` | `--payload-json <json>` (required) |
| `company update <companyId>` | `PATCH /api/companies/{id}` | `--payload-json <json>` (required; only fields to change) |
| `company branding:update <companyId>` | `PATCH /api/companies/{id}/branding` | `--payload-json <json>` (required; branding fields only) |

```sh
paperclipai company create --payload-json '{"name":"Acme","issuePrefix":"ACME"}'
paperclipai company update <company-id> --payload-json '{"budgetMonthlyCents":500000}'
```

### Archive (reversible) vs delete (permanent)

```sh
paperclipai company archive <company-id>          # POST /{id}/archive, no payload — reversible
paperclipai company delete PAP --yes --confirm PAP # permanent, multiple guard rails
```

`company delete <selector>` — selector is a UUID or an issue prefix (e.g. `PAP`).

| Flag | Type | Required | Default | Effect |
|---|---|---|---|---|
| `--by <mode>` | `auto\|id\|prefix` | no | `auto` | How to interpret the selector. Ambiguous `auto` matches are rejected. |
| `--yes` | bool | yes | false | Without it the command errors before any API call. |
| `--confirm <value>` | string | yes | — | Must equal the target's ID or issue prefix (prefix match case-insensitive). |

Deletion also requires the server to have `PAPERCLIP_ENABLE_COMPANY_DELETION` on. No undo. Agent credentials cannot do a board-wide lookup — they can only delete their own scoped company. Prefer `archive`.

### Export / import (portable packages)

```sh
paperclipai company export <company-id> --out ./exports/acme --include company,agents,projects,issues,skills
paperclipai company import ./exports/acme --target new --new-company-name "Acme (imported)"
paperclipai company import owner/repo --target existing -C <company-id> --collision replace --yes
```

`export` flags: `--out <path>` (required, output dir); `--include <csv>` of `company`, `agents`, `projects`, `issues` (alias `tasks`), `skills` (default `company,agents` — widen explicitly for a full clone); `--skills <csv>`, `--projects <csv>`, `--issues <csv>`, `--project-issues <csv>` to narrow; `--expand-referenced-skills` (default off) to vendor skill contents. Non-empty output dir: interactive confirm, hard refusal when non-interactive — use an empty dir headless.

`import <source>` — source: local dir, local `.zip`, GitHub URL, or `owner/repo[/path]`. Generic HTTP URLs rejected. Flags: `--include <csv>` (default: all five groups); `--target new|existing` (inferred from `-C`/context if omitted); `-C, --company-id <id>` (required for `--target existing`); `--new-company-name <name>`; `--agents <list|all>` (default `all`); `--collision rename|skip|replace` (default `rename`); `--ref <git-ref>` (GitHub only); `--paperclip-url <url>` (alias for `--api-base`); `--yes`; `--dry-run` (preview only). Always runs a server-side preview first. With `--json` or a non-interactive terminal you MUST pass `--yes` to apply. Process-adapter agents with no explicit adapter fall back to `claude-local`.

Raw API passthrough (no local file I/O, `--payload-json` required): `company export:preview <id>`, `company export:api <id>`, `company import:preview <id>`, `company import:apply <id>` → `POST /api/companies/{id}/exports[/preview]` and `/imports/{preview,apply}`. Prefer the high-level commands.

### Feedback traces (company-scoped)

Both require `-C, --company-id <id>` as a flag (not positional).

```sh
paperclipai company feedback:list -C <company-id> --status open --vote down
paperclipai company feedback:export -C <company-id> --from 2026-01-01 --format ndjson --out ./feedback.ndjson
```

Shared filters: `--target-type`, `--vote`, `--status`, `--project-id`, `--issue-id`, `--from <iso8601>`, `--to <iso8601>`, `--shared-only`, `--include-payload`. `feedback:export` adds `--out <path>` (else stdout) and `--format json|ndjson` (default `ndjson`); export includes payloads by default, list only with `--include-payload`.

### Company gotchas

- `feedback:list`, `feedback:export`, and `import` take the company as `-C/--company-id`; everywhere else it is a positional `<companyId>`.
- `create`/`update`/`branding:update` forward `--payload-json` verbatim — validation errors come from the server, not the CLI.
- `delete` needs `--yes` AND a matching `--confirm` AND server policy enabled. `archive` is the safe alternative.
- Headless import: pass `--yes` (and run `--dry-run --json` first to inspect the preview plan).

---

## Goal

Company-scoped strategic objectives. Nestable via `--parent-id`; ownable via `--owner-agent-id`. Treat as durable: update status (e.g. active → achieved/abandoned) rather than delete.

```sh
paperclipai goal list -C <company-id>            # one line per goal: id, status, title, level, parentId, ownerAgentId
paperclipai goal get <goal-id>                   # by ID, no -C needed
paperclipai goal create -C <company-id> --title "Grow revenue 30% this year"
paperclipai goal update <goal-id> --status achieved
paperclipai goal delete <goal-id> --yes
```

`goal create` flags: `-C, --company-id <id>` (required), `--title <title>` (required), `--description <text>`, `--level <level>` (e.g. company vs team — server defaults when omitted), `--status <status>` (server default when omitted), `--parent-id <id>`, `--owner-agent-id <id>`. Capture the returned `id` for `project create --goal-ids` or an issue's `--goal-id`.

`goal update <goalId>` flags (all optional, patch semantics; no `-C`): `--title`, `--description <text|null>`, `--level`, `--status`, `--parent-id <id|null>`, `--owner-agent-id <id|null>`. Literal string `null` clears the nullable fields — the only way to detach a parent or remove an owner without deleting.

```sh
# Nest a sub-goal under a parent, owned by an agent
paperclipai goal create -C <company-id> --title "Launch paid tier" --parent-id <parent-goal-id> --owner-agent-id <agent-id>
# Promote to top level
paperclipai goal update <goal-id> --parent-id null
```

Find children with `--json` + jq: `paperclipai goal list -C <id> --json | jq '[.[] | select(.parentId == "<goal-id>")]'`.

### Goal gotchas

- `delete` requires `--yes` ("Deletion requires --yes." otherwise). Deleting orphans linked projects/issues/child goals — prefer `--status achieved|abandoned`.
- Ownership (`--owner-agent-id`) is accountability, not assignment — agents pick up work through issues.
- `get`/`update`/`delete` address by goal ID directly; only `list`/`create` take `-C`.

---

## Project

Groups work inside a company; carries status, optional lead agent, goal links, and execution defaults (`env`, workspace policy) that issues inherit. `get`/`update`/`delete` accept a UUID **or shortname** — pass `-C` whenever you use a shortname so the server resolves it.

```sh
paperclipai project list -C <company-id>                       # required -C; one line per project: id, name, status, urlKey, goalIds, leadAgentId
paperclipai project get launch-site -C <company-id>            # shortname needs -C; UUID does not
paperclipai project create -C <company-id> --name "Launch Site" --goal-ids <g1>,<g2> --lead-agent-id <agent-id>
paperclipai project update <project-id> --status in_progress
paperclipai project delete launch-site -C <company-id> --yes
```

`project create` flags: `-C` (required), `--name` (required), `--description <text>`, `--status <status>` (e.g. `planned`), `--goal-ids <csv>` (preferred), `--goal-id <id>` (deprecated single-goal form), `--lead-agent-id <id>`, `--target-date <date>`, `--color <value>`, `--env-json <json>`, `--execution-workspace-policy-json <json>`. Payload is validated locally against the server's create schema before sending.

`project update <ref>` flags (patch — omit = unchanged): `-C` (for shortname lookup), `--name`, `--description <text|null>`, `--status`, `--goal-ids <csv>` (**replaces the whole set** — pass the full list, not a delta), `--goal-id <id|null>` (deprecated), `--lead-agent-id <id|null>`, `--target-date <date|null>`, `--color <value|null>`, `--env-json <json|null>`, `--execution-workspace-policy-json <json|null>`, `--archived-at <iso8601|null>` (timestamp archives; `null` unarchives).

```sh
# Archive / unarchive
paperclipai project update <project-id> --archived-at 2026-06-01T00:00:00Z
paperclipai project update <project-id> --archived-at null
# Secret-backed env binding inherited by the project's issues at execution time
paperclipai project create -C <company-id> --name "Ops" \
  --env-json '{"OPENAI_API_KEY":{"kind":"secret","secretName":"openai-api-key"}}'
# Workspace policy on, default mode shared
paperclipai project update <project-id> --execution-workspace-policy-json '{"enabled":true,"defaultMode":"shared_workspace"}'
```

### Project gotchas

- Literal `null` clears nullable fields; omitting leaves them unchanged (applies to description, goal-id, lead-agent-id, target-date, color, env-json, execution-workspace-policy-json, archived-at).
- `--env-json` / `--execution-workspace-policy-json` are `JSON.parse`d locally — malformed JSON fails fast with `Invalid JSON: ...` before any request.
- `delete` requires `--yes` and is immediate; verify the shortname with `project get` first.
- Use `--goal-ids`, not the deprecated `--goal-id`.

---

## Issue

The core work object — Paperclip's communication model is issues plus comments, not chat. Most issue-scoped commands accept a UUID or human identifier (e.g. `PC-12`). Collection/create commands need `-C`.

### Core CRUD

```sh
paperclipai issue list -C <company-id> --status todo,in_review --assignee-agent-id <agent-id> --match "billing"
paperclipai issue get PC-12
paperclipai issue create -C <company-id> --title "Fix invoice rounding" --priority high --project-id <p> --assignee-agent-id <a>
paperclipai issue update <issue-id> --status in_review --comment "Ready for review"
paperclipai issue delete <issue-id> --yes
```

| Command | Flags |
|---|---|
| `issue list` | `-C <id>` (required unless in context); `--status <csv>`; `--assignee-agent-id <id>`; `--project-id <id>`; `--match <text>` (local case-insensitive match on identifier/title/description, applied after the server responds) |
| `issue get <idOrIdentifier>` | — |
| `issue create` | `-C <id>` + `--title` required; optional `--description`, `--status`, `--priority`, `--assignee-agent-id`, `--project-id`, `--goal-id`, `--parent-id` (child of another issue), `--request-depth <n>`, `--billing-code <code>` |
| `issue update <issueId>` | any subset of the create fields, plus `--comment <text>` (comment in the same call) and `--hidden-at <iso8601\|null>` (`null` clears) |
| `issue delete <issueId>` | `--yes` required — no prompt, just refusal without it |
| `issue heartbeat-context <issueId>` | read-only: the heartbeat context the server would assemble; debug why an agent did/didn't pick up the task |

Parent/child structure is the dependency mechanism at this surface: `--parent-id` on create/update, or `issue child:create <parentIssueId> --payload-json '{"title":"Subtask"}'` (a `CreateChildIssue` payload).

### Checkout and release (single-assignee enforcement)

```sh
paperclipai issue checkout <issue-id> --agent-id <agent-id> [--expected-statuses todo,in_review]
paperclipai issue release <issue-id>          # back to todo, assignee cleared
paperclipai issue force-release <issue-id>    # admin escape hatch — overrides an active checkout
```

`checkout`: `--agent-id <id>` required; `--expected-statuses <csv>` defaults to `todo,backlog,blocked` — checkout fails unless the issue is in one of them (optimistic guard against double-claiming). Use `force-release` only when an agent is genuinely stuck.

### Comments

```sh
paperclipai issue comment <issue-id> --body "Picking this up now"
paperclipai issue comment <issue-id> --body "Fix regressed" --reopen     # reopens done/cancelled work
paperclipai issue comment <issue-id> --body "Any update?" --resume       # wakes assignee when resumable
paperclipai issue comments <issue-id> --order asc --limit 50 [--after-comment-id <id>]
paperclipai issue comment:get <issue-id> <comment-id>
paperclipai issue comment:delete <issue-id> <comment-id>
```

`comment` requires `--body <text>`; `--reopen` and `--resume` are the flags that change workflow state, not just annotate.

### Documents (keyed, revisioned markdown)

```sh
paperclipai issue documents <issue-id> [--include-system]
paperclipai issue document:get <issue-id> spec
paperclipai issue document:put <issue-id> spec --title "Spec" --body-file ./spec.md --change-summary "Initial draft"
paperclipai issue document:delete <issue-id> spec
paperclipai issue document:lock <issue-id> spec        # / document:unlock
paperclipai issue document:revisions <issue-id> spec
paperclipai issue document:restore <issue-id> spec <revision-id>
```

`document:put` flags: `--title`, `--format` (default `markdown`), `--body <markdown>` or `--body-file <path>` (prefer the file for multi-line), `--change-summary <text>`, `--base-revision-id <id>` (optimistic concurrency — write fails if the doc moved past it).

### Work products (JSON payloads)

```sh
paperclipai issue work-products <issue-id>
paperclipai issue work-product:create <issue-id> --payload-json '{"kind":"report","title":"Q2 analysis"}'
paperclipai issue work-product:update <work-product-id> --payload-json '{"title":"Q2 analysis (final)"}'
paperclipai issue work-product:delete <work-product-id>
```

Payload schemas: `CreateIssueWorkProduct` / `UpdateIssueWorkProduct`. **`update` and `delete` take the work-product ID, not the issue ID.**

### Interactions (e.g. `ask_user_questions`)

```sh
paperclipai issue interactions <issue-id>
paperclipai issue interaction:create <issue-id> --payload-json '{...}'   # CreateIssueThreadInteraction
paperclipai issue interaction:accept <issue-id> <interaction-id> [--selected-client-keys a,b] [--selected-option-ids id1,id2]
paperclipai issue interaction:reject <issue-id> <interaction-id> [--reason "Out of scope"]
paperclipai issue interaction:cancel <issue-id> <interaction-id> [--reason "No longer needed"]
paperclipai issue interaction:respond <issue-id> <interaction-id> --answers-json '[{"key":"q1","value":"yes"}]' [--summary-markdown "Confirmed"]
```

`interaction:respond` requires `--answers-json` (the answers array). `--selected-option-ids` (checkbox option IDs) exists in the CLI source though it is absent from the doc page.

### Tree state, preview, and holds (subtree pause/resume/cancel/restore)

```sh
paperclipai issue tree-state <root-issue-id>
paperclipai issue tree-preview <root-issue-id> --payload-json '{"mode":"pause"}'   # PreviewIssueTreeControl — always run before a broad pause/cancel
paperclipai issue tree-holds <root-issue-id> [--status active|released] [--mode pause|resume|cancel|restore] [--include-members]
paperclipai issue tree-hold:create <root-issue-id> --payload-json '{"mode":"pause","reason":"awaiting budget"}'  # CreateIssueTreeHold
paperclipai issue tree-hold:get <root-issue-id> <hold-id>
paperclipai issue tree-hold:release <root-issue-id> <hold-id> [--payload-json '{}']   # payload defaults to {}
```

A hold on a root fans out across the entire subtree — `tree-preview` tells you exactly which issues it will touch.

### Attachments

```sh
paperclipai issue attachments <issue-id>
paperclipai issue attachment:upload <issue-id> -C <company-id> --file ./diagram.png [--comment-id <comment-id>]
paperclipai issue attachment:download <attachment-id> --out ./diagram.png
paperclipai issue attachment:delete <attachment-id>
```

`upload` requires `-C` and `--file <path>`. `download` without `--out` streams raw bytes to stdout — redirect (`> out.bin`) for binary content. Download/delete take the **attachment** ID.

### Labels (company-scoped, shared across issues)

```sh
paperclipai issue label:list -C <company-id>
paperclipai issue label:create -C <company-id> --name "needs-review" --color "#4f46e5"
paperclipai issue label:delete <label-id>
```

`label:create` requires `--name` and `--color <hex>`.

### Approvals (link existing — creation lives in `paperclipai approval`)

```sh
paperclipai issue approvals <issue-id>
paperclipai issue approval:link <issue-id> <approval-id>
paperclipai issue approval:unlink <issue-id> <approval-id>
```

### Read / archive inbox state (per current credential)

```sh
paperclipai issue read <issue-id>      # / unread
paperclipai issue archive <issue-id>   # / unarchive
```

### Recovery

```sh
paperclipai issue recovery-actions <issue-id>
paperclipai issue recovery:resolve <issue-id> --outcome restored --source-issue-status todo --resolution-note "Re-queued"
```

`recovery:resolve` flags: `--outcome restored|false_positive|blocked|cancelled` (required); `--source-issue-status` (required: `todo`/`done`/`in_review` for `restored`; `blocked` only valid with the `blocked` outcome); `--action-id <id>` when multiple actions are active; `--resolution-note <text>`.

### Runs (read-only; server-side heartbeat runs)

```sh
paperclipai issue runs <issue-id>        # all runs
paperclipai issue live-runs <issue-id>   # queued + running only
paperclipai issue active-run <issue-id>  # the single active run, or null
```

### Feedback votes and traces

```sh
paperclipai issue feedback:votes <issue-id>
paperclipai issue feedback:vote <issue-id> --payload-json '{"targetType":"comment","targetId":"<id>","vote":"up"}'  # UpsertIssueFeedbackVote
paperclipai issue feedback:list <issue-id> --vote down --status open --include-payload
paperclipai issue feedback:export <issue-id> --from 2026-01-01 --format ndjson --out ./feedback.ndjson
```

Filters: `--target-type`, `--vote`, `--status`, `--from`, `--to`, `--shared-only`, `--include-payload` — like the company variants but **without** `--project-id`/`--issue-id` (the issue is already the scope); export adds `--out` and `--format json|ndjson` (default `ndjson`) and includes payloads by default.

### Issue gotchas

- UUID **or** identifier (`PC-12`) works on issue-scoped commands; `work-product:update/delete`, `attachment:download/delete`, `label:delete` take their own resource IDs, not issue IDs.
- `delete` and `force-release` are deliberate actions: `delete` needs `--yes` with no soft prompt; `force-release` overrides a live agent checkout.
- `checkout` default `--expected-statuses` is `todo,backlog,blocked` — pass your own CSV to claim an issue in another status.
- Status transitions are just `issue update --status <s>`, but `comment --reopen` is the way to reopen done/cancelled work, and `comment --resume` wakes the assignee.
- No dedicated "dependency" command exists: model dependencies with `--parent-id` / `child:create` (hierarchy) and `tree-hold` (blocking a subtree).
- `--match` filtering on `list` is local (client-side) — combine with server-side `--status`/`--project-id` filters to keep responses small.
- Payload-JSON commands (`work-product:*`, `interaction:create`, `tree-*`, `child:create`, `feedback:vote`) forward JSON to named server schemas; malformed JSON or wrong shapes error per schema.
