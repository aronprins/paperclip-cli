# Paperclip CLI Skill

**A skill that teaches an AI agent to operate the Paperclip CLI (`paperclipai`) end to end — running an instance and an entire AI-agent company from the terminal.**

Hand this skill to an AI agent and it can stand up a Paperclip instance, build the org, hire and wake agents, watch runs live, set budgets, manage routines, and script the whole control plane — without a browser, and without guessing at flags.

---

## What It Does

The Paperclip CLI is huge: two command layers, two personas, ~280 subcommands across 24 command families. An AI operator that hasn't internalized the conventions burns tokens rediscovering them on every task — or worse, gets them silently wrong: the wrong payload flag, a missing `--company-id`, a secret it can never read back, a `wake` loop with no budget that quietly spends real money.

This skill front-loads all of it so the agent operates the CLI correctly on the first try:

| Piece | Purpose |
|---|---|
| `SKILL.md` | The core mental model, nine golden rules, environment checks, five worked playbooks, and a routing table to the references. Loaded first, kept small. |
| `references/` | Nine dense reference files — one per command-family group — with exact syntax, flag tables, `--json` output shapes, copy-pasteable examples, and a gotchas section per family. Loaded on demand. |

---

## How It Works

### 1. Mental model first

Before any command runs, the skill establishes what actually matters:

- **Two layers.** Setup commands (`onboard`, `doctor`, bare `run`, `db:backup`, worktrees) work on local config and need no credential. Control-plane commands are HTTP clients for the server API and need an API base plus a credential.
- **The CLI never runs the model.** Agent work executes server-side; the CLI conducts and observes. The one exception — `agent local-cli` — is exactly how you hand the CLI to an AI operator in the first place.
- **Two personas.** Board operator (instance-wide authority) and agent (scoped to one company + one agent), each with its own credential type.
- **Resolution orders.** How the API base, credential, and company scope are actually resolved — so misconfiguration is *diagnosed* instead of blindly retried.

### 2. Golden rules

Nine rules that prevent the most common failure classes: always `--json` when parsing, the bare-`run` vs `run <sub>` overload, secrets shown exactly once, `--payload` vs `--payload-json` spelling, never letting a command prompt in automation, persona errors as diagnostics, and the fact that every wake spends real provider tokens.

### 3. Worked playbooks

Five end-to-end sequences in exact commands:

- Bootstrap a headless instance and first company (board)
- Operate as an agent: pick up → work → update → comment
- Wake an agent and observe the run live
- Set up a recurring routine with a schedule trigger
- Budget guardrails and incident recovery

### 4. Progressive disclosure

A routing table maps any task to the one reference file worth reading — so the agent loads exactly what it needs and nothing else:

| Task | Reference |
|---|---|
| Install, `onboard`, bare `run`, `doctor`, worktrees | `references/setup-and-instance.md` |
| Personas, `connect`, `auth`, tokens, context profiles | `references/auth-and-context.md` |
| Companies, goals, projects, issues | `references/org.md` |
| Agent lifecycle, wake, `agent local-cli`, prompts, teams | `references/agents-and-prompts.md` |
| Heartbeat runs, routines, approvals, activity | `references/runs-and-routines.md` |
| Dashboard, cost, budgets, feedback | `references/observability-and-cost.md` |
| Workspaces, adapters, assets, company skills | `references/platform.md` |
| Secrets, cloud sync, plugins | `references/extension.md` |
| `--json` contract, exit codes, autonomous operator loop | `references/scripting-and-automation.md` |

---

## What It Covers

The references span the full CLI surface — not a curated subset:

| Area | What the agent can do |
|---|---|
| **Instance & setup** | `onboard`, bootstrap and start a local server, `doctor`, `configure`, DB backups, allowed hostnames, worktrees, env-lab |
| **Auth & context** | `connect` wizard, board/agent personas, device-code login, board & agent tokens, invites, context profiles, `whoami` |
| **Org** | Companies, goals, projects, issues — CRUD, checkout/release, comments, documents, work products, labels, attachments, holds |
| **Agents & prompts** | Hire/create/pause/terminate, `wake`, permissions, config revisions, agent skills & instructions, `agent local-cli`, prompt handoff, team catalog |
| **Runs & routines** | Heartbeat runs (`run <sub>`), live streaming, routines + triggers, approvals, activity feeds |
| **Observability & cost** | Dashboard, cost attribution, finance events, budgets, incident recovery, feedback export/trace |
| **Platform** | Execution/project workspaces, environments + leases, org chart, adapters, assets, company skills library |
| **Extension** | Managed & vault-linked secrets, cloud sync/push, plugins (install, config, jobs, bridges) |
| **Scripting** | The `--json` contract, exit codes, jq patterns, non-interactive rules, poll loops, the autonomous operator loop |

Each reference carries exact syntax, flag tables, and `--json` output shapes — so the agent operates the real command surface, not an approximation of it.

---

## Installation

This skill is a set of plain markdown files any AI agent can load. For [Claude Code](https://claude.ai/code), clone it into your skills folder:

```bash
git clone https://github.com/aronprins/paperclip-cli ~/.claude/skills/paperclip-cli
```

For other agents, point them at `SKILL.md` (it routes to the `references/` files on demand) however your harness loads skills or context.

The agent loads it whenever a task involves the Paperclip CLI — bootstrapping or operating an instance, building and driving a company from the terminal, scripting against the control plane, or running as an agent via `agent local-cli`.

---

## Requirements

- An AI agent that can load markdown skill files (e.g. [Claude Code](https://claude.ai/code))
- The Paperclip CLI (`paperclipai`) on your `PATH`
- A Paperclip instance to operate against (local or remote)

---

## Part of the Paperclip Ecosystem

Paperclip CLI Skill is one skill in the broader **Paperclip** framework for building AI-run companies. Other skills in the ecosystem handle founder interviews and company constitutions, agent hiring, task coordination, and plugin creation.

> Paperclip is a framework for companies that run on AI agents — not companies that use AI as a tool.

---

## Author

Built by [Aron Prins](https://github.com/aronprins).

If you're building an AI-run company on Paperclip and need help with setup, architecture, or hands-on consulting — reach out on X:

**[@aronprins](https://x.com/aronprins)**

Whether you're stuck or just want someone to point you in the right direction, DM me — happy to help :)

Follow for updates on Paperclip, new skills, and AI company building.

---

## License

MIT
