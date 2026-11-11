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
| `agents/adversarial.md` | `~/.claude/agents/adversarial.md` | Last verify pass: attacks the change for security loopholes; nothing ships as ready without its approval |
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

- **Part A (everywhere):** no guessing (gather context, then ask questions and permissions in one early batch), concise replies, and a precedence order: your instruction in the session, then a project workflow skill, then this file.
- **Part B (development):** every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The work runs as agentic loops (B4): gather context, act, check against ground truth, decide. A loop repeats until its exit test passes, its budget runs out, progress stalls, or it needs your input. Ground truth is outside the model: command output, test results, file content, or a verdict from an agent that did not write the code.

The orchestrator runs the outer loop:

```
intake -> orient -> plan -> build -> verify -> ship -> report
           ^  |      ^ ^     | ^      |  |
           +--+      | +-----+ +------+  |
                     +-------------------+

orient -> orient   unknowns get one re-brief, then go into the question batch
plan   -> you      the one question batch: questions, permissions, expected forks; then no questions until the report
build  -> plan     a blocked unit whose fix fits the plan: re-plan
verify -> build    in-scope findings and diff-caused gate failures become fix briefs; at most 3 fix rounds
verify -> build    once review passes, adversarial findings become fix briefs too; at most 3 more fix rounds;
                   the PR is ready only with the adversarial's approval, otherwise it ships as a draft
verify -> plan     a design-level finding: re-plan
any    -> park     no safe option, exhausted budget, or stall: park the affected units, finish the rest,
                   and ask in the report (B4 Autonomy); a safe, reversible gap is decided and logged
```

Each phase hands bounded units to workers, and each worker runs its own inner loop:

```
intake      orchestrator      not code: answer; PR review: pr-review skill; code change: orient
orient      orchestrator      searches the tracker for candidate tickets; loads the workflow skill
            explorer x N      code, standards, real gates; search, read, check; at most 3 widening rounds
plan        orchestrator      units with disjoint file scopes, exit tests, gates; one batch of questions and permissions;
                              then the ticket (match or new) and its branch
build       implementer x N   red, green, check; at most 3 attempts per failing check
            orchestrator      read each unit's diff and commit it
verify      reviewer          fresh every round; findings R1, R2, ... marked closed, open, regressed, or disputed
            verifier          one pass per call; a failing gate re-runs once, a changed result is FLAKY
            adversarial       last, once review passes; fresh every pass; findings S1, S2, ... with exploit paths;
                              local throwaway PoCs only; at most 3 fix rounds; the PR is ready only on its approval
ship        orchestrator      push, PR (stacked when there is more than one), move the ticket, link both ways;
                              merge bottom-up only if the batch granted it
report      orchestrator      B7: short sections, each under 250 words where possible, with a merge-order table for stacks

side issue  side-fix          own ticket, branch, worktree, and PR; scaled-down loop with its own adversarial pass,
                              3 fix rounds each, background
                              cannot spawn: stops at ready for verify; the orchestrator verifies and ships it
```

Every worker report starts with `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`; a report without it counts as `partial`. Workers never ask you anything; they return `needs-input`, and the orchestrator decides it (if a safe, reversible option exists) or parks it for the report. The orchestrator keeps a ledger (B8) of budgets, units and their worker IDs, fix rounds, findings, side tickets, PRs in stack order, your answers, the decisions it made for you, and parked questions. It re-posts the ledger at every phase change except into the report, and rebuilds from it after a context reset.

You hear from the orchestrator twice with something to answer: the question batch after Orient, and the B7 report. In between, it decides safe, reversible gaps itself and logs them, parks anything else, and finishes the rest. The only interrupt is a committed secret. When a session opens more than one PR into the same base, they form one stack (B2): side fixes at the bottom and the main PR on top, opened only after every side-fix has ended. Each PR targets the branch of the one below it, and the report's Merge order table says what to merge first.

The question batch covers workflow permissions (merge, deploys, new dependencies). Claude Code's own tool-permission prompts are separate: a run still pauses on any command or MCP tool your settings do not allow. For an unattended run, allow the commands the workflow uses (`git`, `gh`, your tracker's MCP tools) in `permissions.allow`, and keep the B6 rails.

Hard rails (B6) apply to every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 asks in the question batch.

## Agents

| Agent | Tools | Edits | Notes |
|---|---|---|---|
| `orchestrator` | All (no `tools` field) | No product code; git state, ticket and PR text | Main thread only; spawns the six workers and never edits files (both prompt-enforced) |
| `explorer` | Read, Grep, Glob, Bash, WebFetch, WebSearch | No | Returns a file map and findings tagged verified or inferred |
| `implementer` | All except Agent | In-scope files only | Tests first; leaves changes uncommitted for the orchestrator |
| `reviewer` | Read, Grep, Glob, Bash, WebFetch | No | Fresh every round; verdict plus findings with stable IDs, severity, and `path:line` |
| `verifier` | Read, Grep, Glob, Bash | No | Discovers gates from CI config; never invents a command; full or scoped runs; reports `FLAKY` |
| `adversarial` | Read, Grep, Glob, Bash, WebFetch | No | Fresh every pass; last stage of verify; verdict plus findings `S1`, `S2`, ... with exploit path and `path:line`; throwaway PoCs only in a temp dir outside the repo, against local targets only (prompt-enforced, since Bash can write) |
| `side-fix` | All | Yes | `isolation: worktree`, `background: true`; spawns only `reviewer`, `verifier`, and `adversarial` (prompt-enforced); if it cannot spawn, stops at ready for verify |

All agents use `model: inherit`, so they run on the main session's model unless the orchestrator overrides it. Per B9 (Model choice), it passes the Agent tool's `model` parameter at spawn time: a cheaper tier for bounded, checkable work such as explorer lookups and verifier runs, and the session's model for reviewers, the adversarial agent, hard implementer units, and side-fix. A model you name in the session wins.

The read-only agents have no Edit or Write tool, so the tool list blocks file edits. They keep Bash for commands like `git log` and `rg`, and only their instructions keep Bash from writing. For enforcement beyond the prompt, use a `PreToolUse` hook or Claude Code's sandbox (see the [permissions docs](https://code.claude.com/docs)). Settings-level `permissions.deny` rules apply to the whole session, not to one agent.

Subagents load `~/.claude/CLAUDE.md` by default, so every worker sees the B4 loop contract. They can spawn subagents up to a default depth of 3 (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`), so orchestrator, then side-fix, then `reviewer`, `verifier`, or `adversarial` fits.

Unverified: whether a resumed `isolation: worktree` subagent keeps its worktree, and whether a fresh one could take over an existing side-fix branch (git does not check out a branch that another worktree holds). So the orchestrator resumes side-fix and never re-spawns it fresh while its worktree exists, and side-fix confirms its worktree path on every resume.

Loop budgets (B4) are enforced by instructions only. Claude Code's `maxTurns` agent field is an optional hard budget that stops an agent after that many turns; in v2.1.246 or later its output is also marked partial. These files do not set it.

### Optional: a hard spawn allowlist

`orchestrator.md` has no `tools` field, so its spawn limits and its "no product-code edits" rule are prompt-enforced. To enforce the spawn limit, add this line to its frontmatter, with your own tracker's MCP server:

```yaml
tools: Agent(explorer, implementer, reviewer, verifier, adversarial, side-fix), Read, Grep, Glob, Bash, Skill, AskUserQuestion, SendMessage, mcp__<tracker-server>__*
```

The cost: any `tools` list drops every MCP tool it does not name (`mcp__<server>` or `mcp__<server>__*`; no `mcp__*` wildcard is documented), so the tracker and every other MCP server you rely on must be listed. Bash can still write files, so "no product-code edits" stays partly prompt-enforced. The `Agent(...)` list is honored only on a main-thread agent; inside a subagent definition it is ignored.

## Skills

| Skill | Use it for |
|---|---|
| `pr-review` | "review PR 123", or a PR URL. Takes precedence over the generic `code-review` skill for PR reviews. Fetches the PR into git's object store with hooks off and reads it there, with no checkout. Finds the ticket, reads the standards on the base branch, reads CI, spawns `reviewer`, confirms the blockers and majors, and reports in chat. Never runs the PR's code unless you ask. Posts to GitHub only when you ask, and only after you approve the exact text. |

## Per-repo overrides

- A repo's `CLAUDE.md` adds project rules on top of this one. If the two conflict and the A3 order doesn't settle it, the agent asks in the question batch, or parks the conflict after it.
- A repo's `.claude/agents/<name>.md` with the same `name` replaces the user-level agent for that repo.
- A repo override of a worker must keep the report contract: the `Status:` first line, the reviewer's finding IDs, the adversarial's verdict and `S` finding IDs, and the verifier's `Scope:` line and `FLAKY` result. Otherwise the orchestrator's exit tests cannot read its reports.
