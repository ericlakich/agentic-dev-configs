# Cursor

User-level configuration for Cursor, the IDE agent and the Cursor CLI (`agent`). The global instructions go into Cursor's [User Rules](https://cursor.com/docs/context/rules), and the agents install into `~/.cursor/agents/` ([subagents docs](https://cursor.com/docs/context/subagents)).

Ported from `../Claude/`.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `user-rules.md` | Cursor Settings → Customize → Rules (User Rules) | Global instructions, applied across all projects; Part C is the orchestrator procedure the main session follows |
| `agents/explorer.md` | `~/.cursor/agents/explorer.md` | Read-only codebase research |
| `agents/implementer.md` | `~/.cursor/agents/implementer.md` | Builds one scoped unit of work |
| `agents/reviewer.md` | `~/.cursor/agents/reviewer.md` | Fresh-context adversarial diff review |
| `agents/verifier.md` | `~/.cursor/agents/verifier.md` | Runs the repo's real gates, reports pass/fail |
| `agents/side-fix.md` | `~/.cursor/agents/side-fix.md` | Takes an unrelated issue to its own PR in an isolated copy, in the background |
| `skills/pr-review/SKILL.md` | `~/.cursor/skills/pr-review/` | Reviews a GitHub PR with the `reviewer` subagent and reports findings |

## Install

**Rules.** Open Cursor Settings → Customize → Rules and paste the body of `user-rules.md` into User Rules. User Rules are stored on your Cursor account and sync, so a `git pull` does not update them. Paste again after the file changes.

There is no orchestrator agent to install. The orchestrator is Part C of `user-rules.md` (see [Run the orchestrator](#run-the-orchestrator)).

**Agents.** A symlink keeps `~/.cursor/agents/` in sync with this repo. Run from the repo root:

```bash
mkdir -p ~/.cursor
p=~/.cursor/agents
if [ -e "$p" ] && [ ! -L "$p" ]; then mv -n "$p" "$p.bak"; fi
ln -sn "$PWD/Cursor/agents" ~/.cursor/agents
```

The `if` moves an existing real directory (even an empty one) to `.bak`. It leaves an existing symlink in place, so `ln -sn` then refuses with "File exists" instead of doing something silent. Plain `ln -s` is risky here: if the target is a directory, or a symlink to one, it exits 0 and creates the link *inside* it. On a re-run that means a self-referencing link inside this repo.

Linking the directory makes this repo the only source of `~/.cursor/agents/`. Add personal agents here, or copy the files (`cp`) instead of linking.

If `../Claude/` is also installed (`~/.claude/agents`), Cursor sees both sets. On a name clash the `~/.cursor/agents/` file wins. Claude's set includes `orchestrator.md`, which has no Cursor counterpart; B9 tells the main session never to spawn it.

To check an agent loaded, invoke it by name, for example `/explorer`.

**Skills.** Link the skill folder, not the whole skills directory, which may hold other skills ([skills docs](https://cursor.com/docs/context/skills)):

```bash
mkdir -p ~/.cursor/skills
ln -sn "$PWD/Cursor/skills/pr-review" ~/.cursor/skills/pr-review
```

If `~/.cursor/skills/pr-review` already exists as a real folder, move it out of `~/.cursor/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it.

Cursor also reads skills from `~/.claude/skills/`, `~/.codex/skills/`, and `~/.agents/skills/`. If other tools' `pr-review` variants are installed there, Cursor sees several, and which one wins is undocumented.

Unverified: whether a symlinked skill folder loads. If the skill does not show up, replace the link with a copy (`cp -R`).

## Run the orchestrator

Cursor has no main-thread agent: there is no `--agent` flag, and the CLI's `--mode` takes only plan or ask ([CLI parameters](https://cursor.com/docs/cli/reference/parameters)). So the orchestrator is not an agent file. Its procedure is Part C of `user-rules.md`, and B0 tells the main session to follow Part C for any code change. Start a development session as usual; there is nothing to launch.

Part C applies only to the main session. Workers ignore it (B0).

This port does not ship an orchestrator subagent. A subagent launched by another subagent cannot launch more, so under an orchestrator subagent, `side-fix` could not spawn its `reviewer` and `verifier` ([subagents](https://cursor.com/docs/subagents)).

Not shipped: a skill used as a Custom Mode. Cursor can use a skill as a Custom Mode, but only in the Agents Window and the CLI, chosen per chat, with no documented default ([skills](https://cursor.com/docs/skills#using-a-skill-as-a-custom-mode)). Part C lives in User Rules instead: they are always on, so no session depends on picking a mode, and the procedure keeps one source.

## How it works

`user-rules.md` has three parts:

- **Part A (everywhere):** no guessing (ask in one early batch), concise replies, and a precedence order: your instruction in the session, then a project workflow skill, then these User Rules.
- **Part B (development):** every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The work runs as agentic loops (B4): gather context, act, check against ground truth, decide. A loop repeats until its exit test passes, its budget runs out, progress stalls, or it needs your input. Ground truth is outside the model: command output, test results, file content, or a verdict from an agent that did not write the code.
- **Part C (orchestrator):** each outer-loop phase's exit test, decision rules, and back-edges, plus the unit brief template, ledger, escalation, and side loops. Only the main session follows it.

The orchestrator (the main session, following Part C) runs the outer loop:

```
intake -> orient -> plan -> build -> verify -> ship -> report
           ^  |      ^ ^     | ^      |  |
           +--+      | +-----+ +------+  |
                     +-------------------+

orient -> orient   unknowns get one re-brief, then go into the question batch
build  -> plan     a blocked unit whose fix fits the plan: re-plan
verify -> build    in-scope findings become fix briefs; at most 3 fix rounds
verify -> plan     a design-level finding: re-plan
any    -> you      one escalation (B4): needs-input, exhausted budget, stall, scope or contract change
```

Each phase hands bounded units to workers, and each worker runs its own inner loop:

```
intake      orchestrator      not code: answer; PR review: pr-review skill; code change: orient
orient      explorer x N      search, read, check; at most 3 widening rounds per question
plan        orchestrator      units with disjoint file scopes, exit tests, gates; one batch of questions
build       implementer x N   red, green, check; at most 3 attempts per failing check
            orchestrator      read each unit's diff and commit it
verify      reviewer          fresh every round; findings R1, R2, ... marked closed, open, or regressed
            verifier          one pass per call; a failing gate re-runs once, a changed result is FLAKY
ship        orchestrator      push, PR, move the ticket, link both ways, stop
report      orchestrator      fixed B7 format

side issue  side-fix          own ticket, branch, isolated copy, and PR; scaled-down loop, 3 verify rounds, background
```

Every worker report starts with `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`. Workers never ask you anything; they return `needs-input`, and the orchestrator batches the questions. The orchestrator keeps a ledger (B8) of units, verify rounds, open findings, and side tickets. It re-posts the ledger at every phase change and rebuilds from it after a context reset.

Hard rails (B6) apply to every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 stops and asks.

## Differences from Claude

| Feature | Claude | Cursor |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md`, symlinked from this repo | User Rules, pasted into Settings; stored on the Cursor account; not applied to Inline Edit (Cmd/Ctrl+K) |
| Main-thread orchestrator | `agents/orchestrator.md`, run with `claude --agent orchestrator` or the `agent` setting | None. Part C of User Rules; the main session follows it (B0) |
| Agent location | `~/.claude/agents/` | `~/.cursor/agents/`; also loads `~/.claude/agents/` and `~/.codex/agents/` (`.cursor/` wins on a name clash) |
| Agent format | Markdown, YAML frontmatter | Markdown, YAML frontmatter: `name`, `description`, `model`, `readonly`, `is_background` |
| Tool restriction | `tools` and `disallowedTools` per agent | No tool fields. `implementer` not spawning agents is enforced by its prompt only |
| Spawn allowlist | Optional `tools: Agent(...)` on the orchestrator (main thread only) | None; instructions only (Part C, B9) |
| Read-only enforcement | Tool list without edit tools; Bash limited by prompt | `readonly: true` on `explorer` and `reviewer` (no edits, no state-changing shell commands). `verifier` has no `readonly` because it must run builds and tests; its no-edit rule is enforced by its prompt only |
| Background | `background: true` | `is_background: true`; output is written to `~/.cursor/subagents/` |
| Worktree isolation | `isolation: worktree` in frontmatter | Requested in the spawn request (B3); the subagent runs in its own copy (a git worktree or a cloud environment) |
| Nested subagents | `side-fix` spawns `reviewer` and `verifier` | Same, but only when the main agent launched `side-fix` directly |
| Resume a worker | `SendMessage` to the agent's ID or name | Pass the agent ID the subagent run returned; it keeps its full context, also after a background run completes ([subagents](https://cursor.com/docs/subagents#resuming-subagents)) |
| Model selection | `model: inherit` | `model: inherit` (the default) or a model ID |

Hooks are not part of this port. Cursor reads hooks from `~/.cursor/hooks.json`.

## Known gaps

- User Rules do not apply to Inline Edit (Cmd/Ctrl+K). ([rules](https://cursor.com/docs/context/rules))
- User Rules live on the Cursor account, not in a file, so they can drift from `user-rules.md` until you paste again. ([help](https://cursor.com/help/customization/rules))
- There is no main-agent mechanism: no `--agent` flag, and the CLI's `--mode` takes only plan or ask. The orchestrator runs from Part C of User Rules. ([CLI parameters](https://cursor.com/docs/cli/reference/parameters))
- The spawn limit (the orchestrator spawns only the five workers) and the loop budgets (B4) are enforced by instructions only. A `subagentStart` hook could deny other launches, but hooks are not part of this port. ([hooks](https://cursor.com/docs/hooks))
- There is no per-agent tool list. Agents can only be limited with `readonly`; everything else is prompt-enforced. ([subagents](https://cursor.com/docs/context/subagents))
- Subagents share the parent checkout unless the request asks for isolation. `side-fix` is only isolated if the orchestrator asks (B3). ([subagents](https://cursor.com/docs/context/subagents))
- An isolated subagent's changes stay on its branch until the parent agent merges them. Do not merge `side-fix`'s branch into the main task; it ships as its own PR (B3). ([subagents](https://cursor.com/docs/context/subagents))
- Cloud subagents use the team's MCP servers, not your local ones, so a `side-fix` running in a cloud environment may lose tracker access. ([subagents](https://cursor.com/docs/context/subagents))
- A subagent launched by another subagent cannot launch more. If `side-fix` is not launched by the main agent, it falls back to running the gates itself and a non-independent review. ([subagents](https://cursor.com/docs/context/subagents))
- With the Claude port installed, Cursor also loads Claude's `orchestrator.md` from `~/.claude/agents/`, inferred from the agent locations above. It is written for Claude Code's main thread; B9 tells the main session never to spawn it. ([subagents](https://cursor.com/docs/context/subagents))
- Unverified: user rule files in `~/.cursor/rules`. The help center mentions them, but their format is undocumented, so none ships here. ([help](https://cursor.com/help/customization/rules))
- Unverified: whether User Rules apply in the CLI (`agent`). The CLI reads `AGENTS.md` and `CLAUDE.md` at the project root and `.cursor/rules`. If User Rules do not apply there, the CLI main agent never sees Part C. ([CLI](https://cursor.com/docs/cli/using))
- Unverified: whether Cursor reads `~/.claude/CLAUDE.md`. ([rules](https://cursor.com/docs/context/rules))
- Unverified: whether subagents receive User Rules. The agents cite rules by number (B4, B5, B6), and those references only resolve if User Rules reach the subagent. If they do, workers also see Part C; B0 tells them to ignore it. ([subagents](https://cursor.com/docs/context/subagents))
- Unverified: whether `readonly` blocks test runs. `reviewer` may run an existing test to confirm a defect; if `readonly` blocks it, the reviewer describes the test instead. ([subagents](https://cursor.com/docs/context/subagents))

## Per-repo overrides

- Repo rules add project rules on top of these User Rules: `.cursor/rules/*.mdc` (frontmatter `description`, `globs`, `alwaysApply`) and `AGENTS.md` at the root and in subdirectories; the CLI applies a root `CLAUDE.md` as a rule. Cursor ranks Team > Project > User. If a repo rule conflicts with these User Rules and the A3 order doesn't settle it, the agent stops and asks.
- A repo can add agents in `.cursor/agents/` (Cursor also reads `.claude/agents/` and `.codex/agents/`; the `.cursor/` file wins on a name clash). Project agents take precedence on a name clash, so a repo's `.cursor/agents/`, `.claude/agents/`, or `.codex/agents/` agent with the same `name` replaces the user-level one, including this port's `side-fix`. ([subagents](https://cursor.com/docs/context/subagents))
- Worktree setup commands for a repo go in `.cursor/worktrees.json`. The CLI's `-w/--worktree [name]` creates `~/.cursor/worktrees/<repo>/<name>`. ([worktrees](https://cursor.com/docs/configuration/worktrees))
