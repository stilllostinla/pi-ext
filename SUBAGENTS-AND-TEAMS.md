# Pi Subagents + Agent Teams — Setup & Usage Report

Date: 2026-10-14
Host: Termux (`/storage/emulated/0/Documents/git`)
Pi version: 1.1.0 (`@earendil-works/pi-coding-agent`)
Session model: `meta/muse-spark-1.3-contributor` (`PI_PROVIDER=openrouter`, `PI_MODEL=meta/muse-spark-1.3-contributor`)
Git protocol: `https` via `gh auth git-credential` as `stilllostinla`

## TL;DR

- **Subagents extension = installed.** Source: `examples/extensions/subagent/` from pi 1.1.0.
- **Agent Teams = no dedicated extension in pi 1.1.0.** The `subagent` tool's **Parallel mode** *is* the teams equivalent (up to 8 tasks, 4 concurrent), **Chain mode** is the pipeline equivalent. `handoff.ts` and `plan-mode/` are related but not teams.
- All 4 agents pinned to **your current model**: `meta/muse-spark-1.3-contributor` (upstream examples use `claude-haiku-4-5` / `claude-sonnet-4-5`).
- Run `/reload` in pi TUI after install, then use `/implement`, `/scout-and-plan`, `/implement-and-review` or call the `subagent` tool directly.

## 1. What I installed

Copied (not symlinked, so upgrades don't break you) from:
```
/data/data/com.termux/files/home/.pi/agent/install/releases/1.1.0/node_modules/@earendil-works/pi-coding-agent/examples/extensions/subagent/
```

To:
```
~/.pi/agent/extensions/subagent/index.ts   # tool registration, spawning, UI
~/.pi/agent/extensions/subagent/agents.ts  # discovery (user vs project scope)
~/.pi/agent/agents/scout.md
~/.pi/agent/agents/planner.md
~/.pi/agent/agents/reviewer.md
~/.pi/agent/agents/worker.md
~/.pi/agent/prompts/implement.md           # -> /implement
~/.pi/agent/prompts/scout-and-plan.md      # -> /scout-and-plan
~/.pi/agent/prompts/implement-and-review.md # -> /implement-and-review
```

Agent dir resolves per `docs/configuration.md`: `<agent-dir>` defaults to `~/.pi/agent` (`PI_CODING_AGENT_DIR` overrides). Project-level would be `.pi/agents/`, `.pi/extensions/`, `.pi/prompts/` (requires project trust).

Verify after `/reload`:
```bash
ls ~/.pi/agent/extensions/subagent/
ls ~/.pi/agent/agents/
ls ~/.pi/agent/prompts/
grep -H "^model:" ~/.pi/agent/agents/*.md
# all should show: model: meta/muse-spark-1.3-contributor
```

## 2. Subagents — how it works

Each invocation spawns an isolated `pi` subprocess:
```
pi --model <agent.model or inherit> --system <agent.systemPrompt> "<task>"
```
- Isolated context window — doesn't pollute main thread.
- Streams tool calls live (`$ command`, `read path:1-10`, `grep /pat/ in path`, etc.).
- Collapsed view: last ~10 items + `3 turns ↑input ↓output $cost ctx:tokens model`. `Ctrl+O` expands to full markdown.
- `Ctrl+C` kills child.
- Parallel output capped at 50KB per task for parent model (full in tool details).

### Agents installed

| Agent | File | Tools | Model | Use for |
|-------|------|-------|-------|---------|
| `scout` | `agents/scout.md` | `read, grep, find, ls, bash` | `meta/muse-spark-1.3-contributor` | Fast recon, returns compressed context |
| `planner` | `agents/planner.md` | `read, grep, find, ls` | same | Turns context+reqs into numbered plan, no edits |
| `reviewer` | `agents/reviewer.md` | `read, grep, find, ls, bash` | same | Read-only review (`git diff/log/show` only) |
| `worker` | `agents/worker.md` | all (no `tools:` limit) | same | General implementation |

Upstream defaults were Haiku for scout, Sonnet for others. I overwrote all to your session model per your request. If you want to follow upstream cost/latency split later, set e.g. `model: meta/muse-spark-1.1` for scout and keep `1.3-contributor` for planner/reviewer/worker. Omit `model:` entirely to always inherit whatever `/model` is active.

Agent frontmatter format:
```md
---
name: scout
description: Fast codebase recon...
tools: read, grep, find, ls, bash
model: meta/muse-spark-1.3-contributor
---
System prompt...
```

## 3. Agent Teams — there is no `teams` extension

Searched in 1.1.0:
```
examples/extensions/ = subagent/, handoff.ts, plan-mode/, ... (no team*)
docs/ grep "team" = only handoff.ts comment "now implement this for teams as well"
```

So in pi vocabulary:

- **Team = Parallel mode.** One prompt fans out to N agents concurrently.
```json
{
  "tasks": [
    {"agent": "scout", "task": "find auth code"},
    {"agent": "scout", "task": "find payment code"}
  ]
}
```
Limits: max 8 tasks, 4 concurrent. UI shows `2/3 done, 1 running` live.

- **Pipeline = Chain mode.** Sequential with `{previous}` placeholder.
```json
{
  "chain": [
    {"agent": "scout", "task": "find code relevant to: Redis caching"},
    {"agent": "planner", "task": "plan Redis caching using: {previous}"},
    {"agent": "worker", "task": "implement: {previous}"}
  ]
}
```
Stops at first failure, reports which step failed.

- **Single = 1 agent, 1 task.**
```json
{"agent": "scout", "task": "find all auth code"}
```

Natural language also works — the model maps it to the tool:
```
Use scout to find all authentication code
Run 2 scouts in parallel: one to find models, one to find providers
Use a chain: first have scout find the read tool, then have planner suggest improvements
```

## 4. Slash-workflows (prompt templates)

| Command | File | Flow |
|---------|------|------|
| `/implement <query>` | `prompts/implement.md` | scout → planner → worker |
| `/scout-and-plan <query>` | `prompts/scout-and-plan.md` | scout → planner (no code) |
| `/implement-and-review <query>` | `prompts/implement-and-review.md` | worker → reviewer → worker |

Examples:
```
/scout-and-plan refactor auth to support OAuth
/implement add Redis caching to the session store
/implement-and-review add input validation to API endpoints
```

## 5. Security model

From `subagent/README.md`:

- Default loads **only** `~/.pi/agent/agents/*.md` (user-level, you control).
- `.pi/agents/*.md` (project-local, repo-controlled) loads only with `agentScope: "project"` or `"both"`. Project agent same-name overrides user agent.
- Interactive run prompts to confirm project agents in untrusted projects. `confirmProjectAgents: false` disables.
- Reviewer/worker prompts say bash is read-only, but "tool permissions are not perfectly enforceable" — treat agents as semi-trusted code. Only enable project agents for repos you trust.

## 6. Files to edit to customize

- Add new agent: create `~/.pi/agent/agents/my-agent.md` with frontmatter above, `/reload`, then `Use my-agent to ...`.
- Change model per agent: edit `model:` line, `/reload`.
- Change tools per agent: edit `tools:` line (comma or `[a, b]` YAML).
- Add team workflow: create `~/.pi/agent/prompts/my-flow.md` with `description:` frontmatter + instructions to use `subagent` tool with `tasks:` or `chain:`.

## 7. Troubleshooting

- Tool not showing: run `/reload` in TUI. Extensions load from `~/.pi/agent/extensions/*/index.ts` — verify `index.ts` + `agents.ts` are siblings in `subagent/`.
- Agents not found: check `~/.pi/agent/agents/*.md` have `name:` + `description:` frontmatter, `.md` extension.
- Model error: `pi --model meta/muse-spark-1.3-contributor --help` must work. If OpenRouter key missing, set `OPENROUTER_API_KEY` before launching pi.
- Dubious ownership on Termux shared storage: use `git -c safe.directory='*' ...` or `git config --global --add safe.directory /storage/emulated/0/Documents/git/pi-ext`.
- Output truncated: expand with `Ctrl+O`, full per-task output is in tool details (50KB cap is only what parent model sees).

---
Installed by Muse Spark (`meta/muse-spark-1.3-contributor`) for `stilllostinla`.
