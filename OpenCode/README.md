# OpenCode

User-level configuration for [OpenCode](https://opencode.ai/docs/) 1.x. Everything here installs into `~/.config/opencode/`.

Ported from `../Claude/`. Same workflow, same rules, the orchestrator and its five workers, and the `pr-review` skill.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `AGENTS.md` | `~/.config/opencode/AGENTS.md` | Global instructions loaded into every session |
| `agents/orchestrator.md` | `~/.config/opencode/agents/orchestrator.md` | Primary agent: runs the outer loop and dispatches the workers |
| `agents/explorer.md` | `~/.config/opencode/agents/explorer.md` | Read-only codebase research |
| `agents/implementer.md` | `~/.config/opencode/agents/implementer.md` | Builds one scoped unit of work |
| `agents/reviewer.md` | `~/.config/opencode/agents/reviewer.md` | Fresh-context adversarial diff review |
| `agents/verifier.md` | `~/.config/opencode/agents/verifier.md` | Runs the repo's real gates, reports pass/fail |
| `agents/side-fix.md` | `~/.config/opencode/agents/side-fix.md` | Takes an unrelated issue to ready for verify in a worktree the orchestrator creates; the orchestrator verifies and ships it |
| `skills/pr-review/SKILL.md` | `~/.config/opencode/skills/pr-review/` | Reviews a GitHub PR with the `reviewer` subagent and reports findings |

Not part of this port: commands (`~/.config/opencode/commands/`) and plugins (`~/.config/opencode/plugins/`).

## Install

Symlinks keep `~/.config/opencode/` in sync with this repo, so a `git pull` updates your setup. Run from the repo root:

```bash
mkdir -p ~/.config/opencode/skills
for p in ~/.config/opencode/AGENTS.md ~/.config/opencode/agents; do
  if [ -e "$p" ] && [ ! -L "$p" ]; then mv -n "$p" "$p.bak"; fi
done
ln -sn "$PWD/OpenCode/AGENTS.md" ~/.config/opencode/AGENTS.md
ln -sn "$PWD/OpenCode/agents" ~/.config/opencode/agents
ln -sn "$PWD/OpenCode/skills/pr-review" ~/.config/opencode/skills/pr-review
```

The loop moves an existing real file or directory to `.bak` (`-n` never overwrites an earlier `.bak`; if one exists, the move is skipped and `ln -sn` refuses). It leaves an existing symlink in place, so `ln -sn` then refuses with "File exists" instead of doing something silent. Plain `ln -s` is risky here: if the target is a directory, or a symlink to one, it exits 0 and creates the link *inside* it. On a re-run that means a self-referencing link inside this repo.

Linking the directory makes this repo the only source of your user-level agents in `agents/`. OpenCode also accepts the singular `~/.config/opencode/agent/` and agents defined under `"agent"` in `opencode.json`, so check those too. Add personal agents here, or copy the files (`cp`) instead of linking.

The `agents` link installs `orchestrator.md` together with the five workers. To start it, see [Run the orchestrator](#run-the-orchestrator).

The skill is linked one folder at a time, because `~/.config/opencode/skills/` may hold other skills. If `~/.config/opencode/skills/pr-review` already exists as a real folder, move it out of `~/.config/opencode/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it. OpenCode also reads `~/.claude/skills/` and `~/.agents/skills/`. If the Claude variant of `pr-review` is installed there, OpenCode sees two `pr-review` skills. Unverified: which one wins.

Unverified: whether OpenCode loads a symlinked skill folder. If `pr-review` does not show up, remove the link and copy the folder instead (`cp -R`).

Once `~/.config/opencode/AGENTS.md` exists, OpenCode stops falling back to `~/.claude/CLAUDE.md`. It never reads `~/.claude/agents`.

Optional: to let `side-fix` run in the background, set this in your shell profile ([CLI](https://opencode.ai/docs/cli/)). Without it, `side-fix` runs in the foreground and the main task waits for it.

```bash
export OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true
```

## Run the orchestrator

The orchestrator is a primary agent (`mode: primary`): it becomes the main session, not a subagent you spawn. Start each development session with:

```bash
opencode --agent orchestrator
```

Or switch to it inside a session with Tab or your `switch_agent` keybind.

For a repo where every session is development work, set it in that repo's `opencode.json` instead:

```json
{ "default_agent": "orchestrator" }
```

`default_agent` refuses a subagent or a hidden agent, so `orchestrator.md` sets `mode: primary` and does not set `hidden: true` (that would also drop it from Tab). ([source][src]: `agent/agent.ts:328-335`)

Do not set `default_agent` in your user-level `opencode.json`. The agent's body replaces the model provider's base prompt, so a user-level default would replace it in every session, including non-development work. Environment info, `AGENTS.md`, MCP instructions, and skills are still appended. That is why `orchestrator.md` carries a short "Operating rules" section. ([source][src]: `session/llm/request.ts:60`, `session/prompt.ts:1257-1269`)

A plain session follows the same loop: AGENTS.md B0 tells a main session not running as the orchestrator to read `~/.config/opencode/agents/orchestrator.md` and follow its body for any code change.

## How it works

`AGENTS.md` has two parts:

- **Part A (everywhere):** no guessing (ask in one early batch), concise replies, and a precedence order: your instruction in the session, then a project workflow skill, then this file.
- **Part B (development):** every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The work runs as agentic loops (B4): gather context, act, check against ground truth, decide. A loop repeats until its exit test passes, its budget runs out, progress stalls, or it needs your input. Ground truth is outside the model: command output, test results, file content, or a verdict from an agent that did not write the code.

The orchestrator runs the outer loop:

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

side issue  side-fix          own ticket, branch, and worktree; scaled-down loop to ready for verify; background when enabled
            orchestrator      side-fix's verify loop (fresh reviewer every round, 3 fix rounds), then its PR
```

Every worker report starts with `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`. Workers never ask you anything; they return `needs-input`, and the orchestrator batches the questions. The orchestrator keeps a ledger (B8) of units, verify rounds, open findings, and side tickets. It re-posts the ledger at every phase change and rebuilds from it after a context reset.

Hard rails (B6) apply to every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 stops and asks.

Loop budgets (B4) are enforced by instructions only. OpenCode's `steps` agent field is an optional hard cap on an agent's agentic iterations, after which it must answer in text; these files do not set it. ([source][src]: `core/src/v1/config/agent.ts:34-37`)

## Differences from Claude

| Feature | Claude | OpenCode |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md` | `~/.config/opencode/AGENTS.md`; falls back to `~/.claude/CLAUDE.md` only when it is absent |
| Imports | `@file` in `CLAUDE.md` | No `@file` import; list files under `instructions` in `opencode.json` |
| Agent location | `~/.claude/agents/`; repo `.claude/agents/` | `~/.config/opencode/agents/`; repo `.opencode/agents/` |
| Agent format | Markdown, frontmatter `name`, `description`, `tools`, `model` | Markdown, frontmatter `description`, `mode`, `permission`; the filename is the agent name. Workers set `mode: subagent`, the orchestrator `mode: primary` |
| Main-thread orchestrator | `orchestrator.md`, run with `claude --agent orchestrator` or the `agent` setting; its body replaces Claude Code's default system prompt | `orchestrator.md` with `mode: primary`, run with Tab, `opencode --agent orchestrator`, or `default_agent` in `opencode.json`; its body replaces the provider's base prompt |
| Spawn allowlist | Prompt-enforced; optional `tools: Agent(...)` list on the main-thread agent | Hard: the orchestrator's `permission.task` map (`"*": deny` first, then the five workers allowed; the last match wins). Every worker has `task: deny` |
| Orchestrator file edits | Prompt-enforced | `edit: deny` blocks the edit tools. Bash stays at the default so it can run `git` and `gh`, and Bash can still write files, so the rule stays partly prompt-enforced |
| Resume a worker | `SendMessage` to its ID or name | `task` with the `task_id` from the earlier result (`<task id="ses_...">`); it continues the same child session |
| Tool restriction | `tools` and `disallowedTools` lists | `permission` map: `allow`, `ask`, or `deny` per tool; pattern maps (used here for `bash` and `task`) |
| Read-only enforcement | Instructions only; Bash kept | `edit: deny` plus a bash map (`"*": ask`, read-only commands allowed) on explorer and reviewer. Verifier's bash map only denies commit, push, checkout, switch, stash, reset, rebase, merge, restore, and clean, so it can still run builds and tests; its prompt is the main guard |
| Background | `background: true` in frontmatter | Per call, `task` with `background: true`; needs `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` |
| Worktree isolation | `isolation: worktree` | Not documented; the orchestrator creates `.worktrees/<TICKET-ID>` before spawning `side-fix` (B3) |
| Nested subagents | side-fix spawns `reviewer` and `verifier` | None: `subagent_depth` defaults to 1, and every worker has `task: deny`. side-fix stops at ready for verify; the orchestrator runs its verify loop and ships its PR (B3) |
| Model selection | `model: inherit` | `model` omitted, so subagents use the caller's model |

## Known gaps

Paths cited after a "source" link in this README are under `packages/opencode/src/` (and `packages/core/src/` for `core/...`) at anomalyco/opencode commit `3884062`, the same at v1.18.35.

- This port targets OpenCode 1.x. OpenCode V2 recognizes `AGENTS.md` only, with no `CLAUDE.md` fallback, and renames the permission keys `bash` to `shell` and `task` to `subagent`. Its migration guide says the V1 forms are still accepted. ([V2 docs](https://opencode.ai/v2/docs))
- V2 also renames the agent prompt key `prompt` to `system`, moves the nesting limit to `experimental.subagent_depth`, and rejects a primary agent requested as a subagent. New V2 agents default to `mode: primary`, so every worker file here sets `mode: subagent` explicitly. ([V2 source][src-v2])
- Nesting is off by default. `subagent_depth` defaults to 1, so a subagent cannot spawn another. `"subagent_depth": 2` in `opencode.json` would allow it, but this port does not rely on it: side-fix has `task: deny`, stops at ready for verify, and the orchestrator runs its verify loop and ships its PR. ([source][src]: `tool/task.ts:111-117`)
- V1 leaves primary agents out of the `task` tool's agent list but does not reject one requested by name, so every worker keeps `task: deny`, and the orchestrator's description says never to spawn it. ([source][src]: `tool/registry.ts:265-269`, `tool/task.ts:119-138`)
- Resume needs the exact `task_id`. An unknown ID does not fail; it silently starts a fresh session with no history. ([source][src]: `tool/task.ts:47-50`, `tool/task.ts:72`, `tool/task.ts:119-138`)
- `orchestrator.md` must not set `hidden: true`: that drops it from Tab, and `default_agent` refuses a hidden agent. ([source][src]: `agent/agent.ts:274-287`, `agent/agent.ts:328-335`)
- No worktree isolation. OpenCode does not document per-agent worktrees, so the orchestrator creates one inside the repo and `side-fix` checks it is in that path before any change. ([agents](https://opencode.ai/docs/agents/))
- Gates that don't read git excludes (some lint and test globs) can pick up files under `.worktrees/` while a side-fix worktree exists.
- Background subagents are experimental. They need `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` and are a per-call choice with no frontmatter field. ([CLI](https://opencode.ai/docs/cli/); source: `task.ts`)
- Unknown frontmatter keys are passed to the model provider as request options, so Claude's `disallowedTools`, `isolation`, and `background` stay out of these files. `tools` is a recognized but deprecated field; these files use `permission` instead. `model` is omitted rather than set to Claude's `inherit`. ([agents](https://opencode.ai/docs/agents/))
- Bash patterns are a backstop, not a sandbox. OpenCode splits a compound command into simple commands and checks each one against the patterns, with `*` matching anything and a trailing ` *` also matching the bare command. So `cat a > b` still matches `"cat *"`, and `"git merge*"` would also match `git merge-base`. That is why the maps deny `"git merge"` and `"git merge *"` instead. The last matching rule wins, which is why `"*": ask` comes first in the explorer and reviewer maps. ([agents](https://opencode.ai/docs/agents/); source: `util/wildcard.ts`)
- An agent's bash map is applied after your global `permission` rules, so an agent-level `"*": allow` would re-allow commands you deny globally. The implementer, verifier, and side-fix maps hold only deny and ask entries. Bash is allowed by default, so they behave the same when you have no global rules.
- A repo agent with the same name as a user-level one overrides it field by field instead of replacing it. OpenCode loads the global config dir first, then `.opencode` dirs, and merges agents with `mergeDeep`, so keys the repo file omits keep their user-level values and `permission` maps merge. (source: `config/paths.ts`, `config.ts`)

[src]: https://github.com/anomalyco/opencode/blob/388406238bd5ca15564a762840a2362c3a45bd9c
[src-v2]: https://github.com/anomalyco/opencode/blob/6608799d35d96c2821a48ac1d4b26a3b84b4e433

## Per-repo overrides

- A repo's `AGENTS.md` (or `CLAUDE.md` if there is no `AGENTS.md`) adds project rules on top of this one. OpenCode loads the global file, then the project file(s), then each `instructions` entry in `opencode.json`, and combines them. If they conflict and the A3 order doesn't settle it, the agent stops and asks. ([rules](https://opencode.ai/docs/rules/))
- A repo's `.opencode/agents/<name>.md` adds agents for that repo. One with the same name as a user-level agent overrides it for that repo, field by field, with `permission` maps merged (see Known gaps).
