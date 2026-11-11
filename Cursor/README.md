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
| `agents/adversarial.md` | `~/.cursor/agents/adversarial.md` | Last verify pass: attacks the change for security loopholes; nothing ships as ready without its approval |
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

All agents use `model: inherit`, so they run on the main session's model unless the orchestrator names another when it launches them. Per B9 (Model choice), it picks a cheaper tier for bounded, checkable work such as explorer lookups and verifier runs, and the session's model for reviewers, the adversarial agent, hard implementer units, and side-fix. A model you name in the session wins. Cursor documents that the parent agent can name a model at launch, but not the parameter, whether it overrides a non-`inherit` `model` field, or which model a resumed subagent uses (unverified). On legacy request-based plans without Max Mode, subagents run on Composer regardless. ([subagents](https://cursor.com/docs/subagents))

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

This port does not ship an orchestrator subagent. A subagent launched by another subagent cannot launch more, so under an orchestrator subagent, `side-fix` could not spawn its `reviewer`, `verifier`, and `adversarial` ([subagents](https://cursor.com/docs/subagents)).

Not shipped: a skill used as a Custom Mode. Cursor can use a skill as a Custom Mode, but only in the Agents Window and the CLI, chosen per chat, with no documented default ([skills](https://cursor.com/docs/skills#using-a-skill-as-a-custom-mode)). Part C lives in User Rules instead: they are always on, so no session depends on picking a mode, and the procedure keeps one source.

## How it works

`user-rules.md` has three parts:

- **Part A (everywhere):** no guessing (gather context, then ask questions and permissions in one early batch), concise replies, and a precedence order: your instruction in the session, then a project workflow skill, then these User Rules.
- **Part B (development):** every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The work runs as agentic loops (B4): gather context, act, check against ground truth, decide. A loop repeats until its exit test passes, its budget runs out, progress stalls, or it needs your input. Ground truth is outside the model: command output, test results, file content, or a verdict from an agent that did not write the code.
- **Part C (orchestrator):** each outer-loop phase's exit test, decision rules, and back-edges, plus the unit brief template, ledger, autonomy, and side loops. Only the main session follows it.

The orchestrator (the main session, following Part C) runs the outer loop:

```
intake -> orient -> plan -> build -> verify -> ship -> report
           ^  |      ^ ^     | ^      |  |
           +--+      | +-----+ +------+  |
                     +-------------------+

orient -> orient   unknowns get one re-brief, then go into the question batch
plan   -> you      the one question batch: questions, permissions, expected forks; then no questions until the report
build  -> plan     a blocked unit whose fix fits the plan: re-plan
verify -> build    in-scope findings and diff-caused gate failures become fix briefs; at most 3 fix rounds
verify -> build    once review passes, adversarial findings become fix briefs too; from its first pass, every round
                   counts against its budget of 3; the PR is ready only with its approval, otherwise it ships as a draft
verify -> plan     a design-level finding: re-plan
any    -> park     no safe option, exhausted budget, or stall: park the affected units, finish the rest,
                   and ask in the report (B4 Autonomy); a safe, reversible gap is decided and logged
```

Each phase hands bounded units to workers, and each worker runs its own inner loop:

```
intake      orchestrator      not code: answer; PR review: pr-review skill; code change: orient
orient      explorer x N      search, read, check; at most 3 widening rounds per question
            orchestrator      tracker and candidate tickets, workflow skill, standards, real gates
plan        orchestrator      units with disjoint file scopes, exit tests, gates; one batch of questions and permissions;
                              then the ticket (match or new) and its branch
build       implementer x N   red, green, check; at most 3 attempts per failing check
            orchestrator      read each unit's diff and commit it
verify      reviewer          fresh every round; findings R1, R2, ... marked closed, open, regressed, or disputed
            verifier          one pass per call; a failing gate re-runs once, a changed result is FLAKY
            adversarial       last, once review passes; fresh every pass; findings S1, S2, ... with exploit paths;
                              local throwaway PoCs only; at most 3 rounds; the PR is ready only on its approval
ship        orchestrator      push, PR (stacked when there is more than one), move the ticket, link both ways;
                              merge bottom-up only if the batch granted it
report      orchestrator      B7: short sections, each under 250 words where possible, with a merge-order table for stacks

side issue  side-fix          own ticket, branch, isolated copy, and PR; scaled-down loop with its own adversarial pass,
                              3 rounds each, background
                              cannot spawn: stops at ready for verify; the orchestrator verifies and ships it
```

Every worker report starts with `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`; a report without it counts as `partial`. Workers never ask you anything; they return `needs-input`, and the orchestrator decides it (if a safe, reversible option exists) or parks it for the report. The orchestrator keeps a ledger (B8) of budgets, units and their agent IDs, fix rounds, findings, side tickets, PRs in stack order, your answers, the decisions it made for you, and parked questions. It re-posts the ledger at every phase change except into the report, and rebuilds from it after a context reset.

You hear from the orchestrator twice with something to answer: the question batch after Orient, and the B7 report. In between, it decides safe, reversible gaps itself and logs them, parks anything else, and finishes the rest. The only interrupt is a committed secret. When a session opens more than one PR into the same base, they form one stack (B2): side fixes at the bottom and the main PR on top, opened only after every side-fix has ended. Each PR targets the branch of the one below it, and the report's Merge order table says what to merge first.

The question batch covers workflow permissions (merge, deploys, new dependencies). Cursor's own tool approvals are separate: a run still pauses on any terminal command or MCP tool call your Run Mode does not allow. For an unattended run in the IDE, use the Allowlist or Auto-review Run Mode (Settings > Agents > Approvals & Execution) and allowlist the commands the workflow uses (`git`, `gh`, your tracker's MCP tools). In the CLI, add them to `permissions.allow` in `~/.cursor/cli-config.json`, for example `Shell(git)`, `Shell(gh)`, and `Mcp(<server>:*)`. Keep the B6 rails. ([run modes](https://cursor.com/docs/agent/security/run-modes), [CLI permissions](https://cursor.com/docs/cli/reference/permissions))

Hard rails (B6) apply to every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 asks in the question batch.

## Differences from Claude

| Feature | Claude | Cursor |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md`, symlinked from this repo | User Rules, pasted into Settings; stored on the Cursor account; not applied to Inline Edit (Cmd/Ctrl+K) |
| Main-thread orchestrator | `agents/orchestrator.md`, run with `claude --agent orchestrator` or the `agent` setting | None. Part C of User Rules; the main session follows it (B0) |
| Agent location | `~/.claude/agents/` | `~/.cursor/agents/`; also loads `~/.claude/agents/` and `~/.codex/agents/` (`.cursor/` wins on a name clash) |
| Agent format | Markdown, YAML frontmatter | Markdown, YAML frontmatter: `name`, `description`, `model`, `readonly`, `is_background` |
| Tool restriction | `tools` and `disallowedTools` per agent | No tool fields. `implementer` not spawning agents is enforced by its prompt only |
| Spawn allowlist | Optional `tools: Agent(...)` on the orchestrator (main thread only) | None; instructions only (Part C, B9) |
| Read-only enforcement | Tool list without edit tools; Bash limited by prompt | `readonly: true` on `explorer`, `reviewer`, and `adversarial` (no edits, no state-changing shell commands). `verifier` has no `readonly` because it must run builds and tests; its no-edit rule is enforced by its prompt only |
| Background | `background: true` | `is_background: true`; output is written to `~/.cursor/subagents/` |
| Worktree isolation | `isolation: worktree` in frontmatter | Requested in the spawn request (B3); the subagent runs in its own copy (a git worktree or a cloud environment) |
| Nested subagents | `side-fix` spawns `reviewer`, `verifier`, and `adversarial`; if it cannot spawn, it stops at ready for verify | Same, but only when the main agent launched `side-fix` directly; otherwise it stops at ready for verify and the orchestrator verifies and ships it |
| Resume a worker | `SendMessage` to the agent's ID or name | Pass the agent ID the subagent run returned; it keeps its full context, also after a background run completes ([subagents](https://cursor.com/docs/subagents#resuming-subagents)) |
| Model selection | `model: inherit` | `model: inherit` (the default) or a model ID |

Hooks are not part of this port. Cursor reads hooks from `~/.cursor/hooks.json`.

## Known gaps

- User Rules do not apply to Inline Edit (Cmd/Ctrl+K). ([rules](https://cursor.com/docs/context/rules))
- User Rules live on the Cursor account, not in a file, so they can drift from `user-rules.md` until you paste again. ([help](https://cursor.com/help/customization/rules))
- There is no main-agent mechanism: no `--agent` flag, and the CLI's `--mode` takes only plan or ask. The orchestrator runs from Part C of User Rules. ([CLI parameters](https://cursor.com/docs/cli/reference/parameters))
- The spawn limit (the orchestrator spawns only the six workers) and the loop budgets (B4) are enforced by instructions only. A `subagentStart` hook could deny other launches, but hooks are not part of this port. ([hooks](https://cursor.com/docs/hooks))
- There is no per-agent tool list. Agents can only be limited with `readonly`; everything else is prompt-enforced. ([subagents](https://cursor.com/docs/context/subagents))
- Subagents share the parent checkout unless the request asks for isolation. `side-fix` is only isolated if the orchestrator asks (B3). ([subagents](https://cursor.com/docs/context/subagents))
- An isolated subagent's changes stay on its branch until the parent agent merges them. Do not merge `side-fix`'s branch into the main task except as a stack merge (B2); it ships as its own PR (B3). ([subagents](https://cursor.com/docs/context/subagents))
- Cloud subagents use the team's MCP servers, not your local ones, so a `side-fix` running in a cloud environment may lose tracker access. ([subagents](https://cursor.com/docs/context/subagents))
- A subagent launched by another subagent cannot launch more. If `side-fix` is not launched by the main agent, it stops at ready for verify, and the orchestrator runs its verify loop and ships it (B3). ([subagents](https://cursor.com/docs/context/subagents))
- With the Claude port installed, Cursor also loads Claude's `orchestrator.md` from `~/.claude/agents/`, inferred from the agent locations above. It is written for Claude Code's main thread; B9 tells the main session never to spawn it. ([subagents](https://cursor.com/docs/context/subagents))
- Unverified: user rule files in `~/.cursor/rules`. The help center mentions them, but their format is undocumented, so none ships here. ([help](https://cursor.com/help/customization/rules))
- Unverified: whether User Rules apply in the CLI (`agent`). The CLI reads `AGENTS.md` and `CLAUDE.md` at the project root and `.cursor/rules`. If User Rules do not apply there, the CLI main agent never sees Part C. ([CLI](https://cursor.com/docs/cli/using))
- Unverified: whether Cursor reads `~/.claude/CLAUDE.md`. ([rules](https://cursor.com/docs/context/rules))
- Unverified: whether subagents receive User Rules. The agents cite rules by number (B4, B5, B6), and those references only resolve if User Rules reach the subagent. If they do, workers also see Part C; B0 tells them to ignore it. ([subagents](https://cursor.com/docs/context/subagents))
- Unverified: whether a CLI `permissions.allow` entry suppresses the approval prompt for a shell command or MCP tool. The docs give the format but say so only for web fetch. Test it before an unattended CLI run. ([CLI permissions](https://cursor.com/docs/cli/reference/permissions))
- `adversarial` has `readonly: true`, which blocks state-changing shell commands, so it may be unable to write or run a throwaway PoC in a temp dir, or to start a service for one. Its findings then stay `[suspected]` with the PoC described, and the orchestrator confirms them at `path:line` as it does for any `suspected` finding; `side-fix` fixes them or stops `blocked`, since it may not dispute its own blockers or majors. Its PoC rules (scrubbed environment, local backends only) apply only if `readonly` lets it run one. ([subagents](https://cursor.com/docs/context/subagents))
- Unverified: whether `readonly` blocks test runs. `reviewer` and `adversarial` may run an existing test to confirm a defect; if `readonly` blocks it, they describe the test instead. ([subagents](https://cursor.com/docs/context/subagents))

## Per-repo overrides

- Repo rules add project rules on top of these User Rules: `.cursor/rules/*.mdc` (frontmatter `description`, `globs`, `alwaysApply`) and `AGENTS.md` at the root and in subdirectories; the CLI applies a root `CLAUDE.md` as a rule. Cursor ranks Team > Project > User. If a repo rule conflicts with these User Rules and the A3 order doesn't settle it, the agent asks in the question batch, or parks the conflict after it.
- A repo can add agents in `.cursor/agents/` (Cursor also reads `.claude/agents/` and `.codex/agents/`; the `.cursor/` file wins on a name clash). Project agents take precedence on a name clash, so a repo's `.cursor/agents/`, `.claude/agents/`, or `.codex/agents/` agent with the same `name` replaces the user-level one, including this port's `side-fix`. ([subagents](https://cursor.com/docs/context/subagents))
- A repo override of a worker must keep the report contract: the `Status:` first line, the reviewer's finding IDs, the adversarial's verdict and `S` finding IDs, and the verifier's `Scope:` line and `FLAKY` result. Otherwise the orchestrator's exit tests cannot read its reports.
- Worktree setup commands for a repo go in `.cursor/worktrees.json`. The CLI's `-w/--worktree [name]` creates `~/.cursor/worktrees/<repo>/<name>`. ([worktrees](https://cursor.com/docs/configuration/worktrees))
