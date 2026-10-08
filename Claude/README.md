# Claude Code

User-level configuration for [Claude Code](https://code.claude.com/docs). Everything here installs into `~/.claude/`.

This directory is the canonical version of the workflow. The other tool directories are ports of it.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `CLAUDE.md` | `~/.claude/CLAUDE.md` | Global instructions loaded into every session |
| `agents/orchestrator.md` | `~/.claude/agents/orchestrator.md` | Main-thread agent: runs the outer loop and dispatches the workers |
| `agents/explorer.md` | `~/.claude/agents/explorer.md` | Read-only codebase research |
| `agents/implementer.md` | `~/.claude/agents/implementer.md` | Builds one scoped unit of work |
| `agents/reviewer.md` | `~/.claude/agents/reviewer.md` | Fresh-context adversarial diff review |
| `agents/verifier.md` | `~/.claude/agents/verifier.md` | Runs the repo's real gates, reports pass/fail |
| `agents/side-fix.md` | `~/.claude/agents/side-fix.md` | Takes an unrelated issue to its own PR in a background worktree |
| `skills/pr-review/SKILL.md` | `~/.claude/skills/pr-review/` | Reviews a GitHub PR with the `reviewer` subagent and reports findings |

## Install

Symlinks keep `~/.claude/` in sync with this repo, so a `git pull` updates your setup. Run from the repo root:

```bash
mkdir -p ~/.claude/skills
for p in ~/.claude/CLAUDE.md ~/.claude/agents; do
  if [ -e "$p" ] && [ ! -L "$p" ]; then mv -n "$p" "$p.bak"; fi
done
ln -sn "$PWD/Claude/CLAUDE.md" ~/.claude/CLAUDE.md
ln -sn "$PWD/Claude/agents" ~/.claude/agents
ln -sn "$PWD/Claude/skills/pr-review" ~/.claude/skills/pr-review
```

The loop moves an existing real file or directory to `.bak` (`-n` never overwrites an earlier `.bak`; if one exists, the move is skipped and `ln -sn` refuses). It leaves an existing symlink in place, so `ln -sn` then refuses with "File exists" instead of doing something silent. Plain `ln -s` is risky here: if the target is a directory, or a symlink to one, it exits 0 and creates the link *inside* it. On a re-run that means a self-referencing link inside this repo.

Skills are linked one folder at a time, because `~/.claude/skills/` usually holds other skills too. If `~/.claude/skills/pr-review` already exists as a real folder, move it out of `~/.claude/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it.

Unverified: whether Claude Code follows a symlinked `agents` directory and symlinked skill folders. If an agent or skill does not show up, replace the links with copies (`cp -R`).

This repo is public. With the directory linked, anything written to `~/.claude/agents/` lands in this working tree, so keep personal or employer-specific agents out of it (copy instead of link) and check `git status` before committing.

Start a new session to pick up changes. To confirm an agent loaded, invoke it, for example by asking for the `explorer` subagent.

## Run the orchestrator

The orchestrator is a main-thread agent: it becomes the main session, not a subagent you spawn. Start each development session with:

```bash
claude --agent orchestrator
```

For a repo where every session is development work, set it in that repo's `.claude/settings.local.json` instead:

```json
{ "agent": "orchestrator" }
```

The `--agent` flag overrides the setting. Do not set `agent` in your user-level `~/.claude/settings.json`. The agent's body replaces Claude Code's default system prompt, including its tool guidance and safety instructions, so a user-level setting would replace it in every session, including non-development work. That is why `orchestrator.md` carries a short "Operating rules" section. `CLAUDE.md` still loads.

A plain session follows the same loop: CLAUDE.md B0 tells a main session not running as the orchestrator to read `~/.claude/agents/orchestrator.md` and follow its body for any code change.

Unverified: how the desktop app and IDE extensions handle the `agent` setting. It is not documented.

## How it works

`CLAUDE.md` has two parts:

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

side issue  side-fix          own ticket, branch, worktree, and PR; scaled-down loop, 3 verify rounds, background
```

Every worker report starts with `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`. Workers never ask you anything; they return `needs-input`, and the orchestrator batches the questions. The orchestrator keeps a ledger (B8) of units, verify rounds, open findings, and side tickets. It re-posts the ledger at every phase change and rebuilds from it after a context reset.

Hard rails (B6) apply to every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 stops and asks.

## Agents

| Agent | Tools | Edits | Notes |
|---|---|---|---|
| `orchestrator` | All (no `tools` field) | No product code; git state, ticket and PR text | Main thread only; spawns the five workers and never edits files (both prompt-enforced) |
| `explorer` | Read, Grep, Glob, Bash, WebFetch, WebSearch | No | Returns a file map and findings tagged verified or inferred |
| `implementer` | All except Agent | In-scope files only | Tests first; leaves changes uncommitted for the orchestrator |
| `reviewer` | Read, Grep, Glob, Bash, WebFetch | No | Fresh every round; verdict plus findings with stable IDs, severity, and `path:line` |
| `verifier` | Read, Grep, Glob, Bash | No | Discovers gates from CI config; never invents a command; full or scoped runs; reports `FLAKY` |
| `side-fix` | All | Yes | `isolation: worktree`, `background: true`; spawns only `reviewer` and `verifier` (prompt-enforced) |

All agents use `model: inherit`, so they run on the main session's model.

The read-only agents have no Edit or Write tool, so the tool list blocks file edits. They keep Bash for commands like `git log` and `rg`, and only their instructions keep Bash from writing. For enforcement beyond the prompt, use a `PreToolUse` hook or Claude Code's sandbox (see the [permissions docs](https://code.claude.com/docs)). Settings-level `permissions.deny` rules apply to the whole session, not to one agent.

Subagents load `~/.claude/CLAUDE.md` by default, so every worker sees the B4 loop contract. They can spawn subagents up to a default depth of 3 (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`), so orchestrator, then side-fix, then `reviewer` or `verifier` fits.

Loop budgets (B4) are enforced by instructions only. Claude Code's `maxTurns` agent field (v2.1.246 and later) is an optional hard budget that stops an agent and marks its output partial; these files do not set it.

### Optional: a hard spawn allowlist

`orchestrator.md` has no `tools` field, so its spawn limits and its "no product-code edits" rule are prompt-enforced. To enforce the spawn limit, add this line to its frontmatter, with your own tracker's MCP server:

```yaml
tools: Agent(explorer, implementer, reviewer, verifier, side-fix), Read, Grep, Glob, Bash, Skill, AskUserQuestion, mcp__<tracker-server>__*
```

The cost: any `tools` list drops every MCP tool it does not name (`mcp__<server>` or `mcp__<server>__*`; no `mcp__*` wildcard is documented), so the tracker and every other MCP server you rely on must be listed. Bash can still write files, so "no product-code edits" stays partly prompt-enforced. The `Agent(...)` list is honored only on a main-thread agent; inside a subagent definition it is ignored.

## Skills

| Skill | Use it for |
|---|---|
| `pr-review` | "review PR 123", or a PR URL. Takes precedence over the generic `code-review` skill for PR reviews. Fetches the PR into git's object store with hooks off and reads it there, with no checkout. Finds the ticket, reads the standards on the base branch, reads CI, spawns `reviewer`, confirms the blockers and majors, and reports in chat. Never runs the PR's code unless you ask. Posts to GitHub only when you ask, and only after you approve the exact text. |

## Per-repo overrides

- A repo's `CLAUDE.md` adds project rules on top of this one. If the two conflict and the A3 order doesn't settle it, the agent stops and asks.
- A repo's `.claude/agents/<name>.md` with the same `name` replaces the user-level agent for that repo.
