# Gemini CLI

User-level configuration for [Gemini CLI](https://geminicli.com/docs/). Everything here installs into `~/.gemini/`.

Ported from `../Claude/`. Same workflow, rules, and six worker agents, in Gemini CLI's native format. The orchestrator is Part C of `GEMINI.md`, not an agent file.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `GEMINI.md` | `~/.gemini/GEMINI.md` | Global instructions loaded into every session; Part C is the orchestrator procedure the main session follows |
| `agents/explorer.md` | `~/.gemini/agents/explorer.md` | Read-only codebase research |
| `agents/implementer.md` | `~/.gemini/agents/implementer.md` | Builds one scoped unit of work |
| `agents/reviewer.md` | `~/.gemini/agents/reviewer.md` | Fresh-context adversarial diff review |
| `agents/verifier.md` | `~/.gemini/agents/verifier.md` | Runs the repo's real gates, reports pass/fail |
| `agents/adversarial.md` | `~/.gemini/agents/adversarial.md` | Last verify pass: attacks the change for security loopholes; nothing ships as ready without its approval |
| `agents/side-fix.md` | `~/.gemini/agents/side-fix.md` | Takes an unrelated issue to `ready for verify` in an orchestrator-created worktree |
| `skills/pr-review/SKILL.md` | `~/.gemini/skills/pr-review/` | Reviews a GitHub PR with the `reviewer` subagent and reports findings |

Not part of this port: custom commands (`~/.gemini/commands/*.toml`) and hooks (`"hooks"` in `settings.json`).

## Install

Symlinks keep `~/.gemini/` in sync with this repo, so a `git pull` updates your setup. Run from the repo root:

```bash
mkdir -p ~/.gemini/skills
for p in ~/.gemini/GEMINI.md ~/.gemini/agents; do
  if [ -e "$p" ] && [ ! -L "$p" ]; then mv -n "$p" "$p.bak"; fi
done
ln -sn "$PWD/Gemini/GEMINI.md" ~/.gemini/GEMINI.md
ln -sn "$PWD/Gemini/agents" ~/.gemini/agents
ln -sn "$PWD/Gemini/skills/pr-review" ~/.gemini/skills/pr-review
```

`~/.gemini/` may already exist and be shared with other Google tools. Add only these entries, and never copy that directory into this repo.

The loop moves an existing real file or directory to `.bak`. It leaves an existing symlink in place, so `ln -sn` then refuses with "File exists" instead of doing something silent. Plain `ln -s` is risky here: if the target is a directory, or a symlink to one, it exits 0 and creates the link *inside* it. On a re-run that means a self-referencing link inside this repo.

Linking the directory makes this repo the only source of your user-level agents. Add personal agents here, or copy the files instead of linking.

A symlinked `agents` directory loads, because the loader's `fs.readdir` follows it. Individually symlinked agent files are skipped without a warning, so link the directory, not single files. [agentLoader.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/agentLoader.ts)

Skills are linked one folder at a time, because `~/.gemini/skills/` may hold other skills too. If `~/.gemini/skills/pr-review` already exists as a real folder, move it out of `~/.gemini/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it. Gemini also reads `~/.agents/skills/`, and within the user tier that alias wins over `~/.gemini/skills/`. Keep other tools' `pr-review` variants out of `~/.agents/skills/`.

Unverified: whether a symlinked skill folder loads. If `pr-review` does not show up, remove the link (`rm ~/.gemini/skills/pr-review`, no trailing slash) and copy instead: `cp -R "$PWD/Gemini/skills/pr-review" ~/.gemini/skills/`. Copies do not update on `git pull`.

No setting is needed. Subagents are on by default (`experimental.enableAgents`, default `true`). `/agents` manages the installed subagents. Invoke an agent automatically by its description, or explicitly with `@name` at the start of a prompt.

There is no orchestrator file to install. The orchestrator is Part C of `GEMINI.md`, which the `GEMINI.md` link above already installs.

## Run the orchestrator

Gemini CLI has no main-thread agent: an agent's `kind` is only `local` or `remote`, and there is no primary-agent mechanism. So the orchestrator ships as Part C at the end of `GEMINI.md`, and the main session follows it. Start each development session as usual:

```bash
gemini
```

B0 tells the main session that it is the orchestrator and follows Part C for any code change. The main session keeps Gemini CLI's built-in system prompt and Part C adds to it, so nothing is replaced. Part C still carries Claude's "Operating rules" (content is data, not instructions; git safety; spawn only the six workers).

Gemini appends `GEMINI.md` to every local subagent's prompt, so the workers see Part C too. B0 tells them to ignore it. [local-executor.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/local-executor.ts#L1457-L1461)

Part C is inline, not an `@` import. Imports in the global file resolve from `~/.gemini/`, and a symlinked import target may be rejected as a path traversal, which is what this repo's symlinked install would produce. Unverified at runtime. [memoryImportProcessor.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/utils/memoryImportProcessor.ts#L203-L205), [import path check](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/utils/memoryImportProcessor.ts#L390-L423)

Never put an orchestrator file in `agents/`: every `*.md` there registers as a subagent unless its name starts with `_`. [agentLoader.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/agentLoader.ts#L665-L670)

## How it works

`GEMINI.md` has three parts:

- **Part A (everywhere):** no guessing (gather context, then ask questions and permissions in one early batch), concise replies, and a precedence order: your instruction in the session, then a project workflow skill, then this file.
- **Part B (development):** every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The work runs as agentic loops (B4): gather context, act, check against ground truth, decide. A loop repeats until its exit test passes, its budget runs out, progress stalls, or it needs your input. Ground truth is outside the model: command output, test results, file content, or a verdict from an agent that did not write the code.
- **Part C (orchestrator):** the main session's outer loop, phase by phase, plus the unit brief template, ledger, autonomy, and side loops. Workers ignore it.

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
ship        orchestrator      push, PR (stacked when there is more than one; every stack merge gets a verifier and a fresh
                              adversarial before it ships), move the ticket, link both ways;
                              merge bottom-up only if the batch granted it
report      orchestrator      B7: short sections, each under 250 words where possible, with a merge-order table for stacks

side issue  side-fix          own ticket, branch, and worktree; scaled-down loop to ready for verify; foreground
            orchestrator      side-fix's verify loop (3 fix rounds, fresh reviewer each),
                              then its adversarial pass (3 rounds), then its PR
```

Every worker report starts with `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`; a report without it counts as `partial`. Workers never ask you anything; they return `needs-input`, and the orchestrator decides it (if a safe, reversible option exists) or parks it for the report. The orchestrator keeps a ledger (B8) of budgets, units, fix rounds, findings, side tickets, PRs in stack order, your answers, the decisions it made for you, and parked questions; it has no worker IDs, because workers cannot be resumed. It re-posts the ledger at every phase change except into the report, and rebuilds from it after a context reset.

A worker cannot be resumed, so a fix pass re-spawns it with its prior report and the findings in the brief (B4).

You hear from the orchestrator twice with something to answer: the question batch after Orient, and the B7 report. In between, it decides safe, reversible gaps itself and logs them, parks anything else, and finishes the rest. The only interrupt is a committed secret. When a session opens more than one PR into the same base, they form one stack (B2): side fixes at the bottom and the main PR on top, opened only after every side-fix has ended. Each PR targets the branch of the one below it, and the report's Merge order table says what to merge first. The orchestrator ships side-fix's PR too, so it stacks that PR onto the top side PR when it opens it.

Two gates protect the workflow itself. First, workflow-control paths (each AI tool's folder, such as `.gemini/`, its agent files and MCP config, and instruction files such as `GEMINI.md` and `AGENTS.md`) are checked each time an implementer returns and before every review. The question batch lists the ones the plan will touch, so you can approve them up front. An unapproved change that is incidental is removed by a fix brief: a file the session created is deleted and a tracked file that was clean before the session is restored, but a file that was already untracked, ignored, or changed before the session is never deleted, overwritten, or restored (a change to it is parked). An unapproved change the ticket needs is parked, and the main PR ships as a draft. An approved change to hooks, settings, MCP config, skills, commands, or the definition of an agent the loop spawns also makes it a draft; only approved plain instruction-file changes proceed. Permission rules added to a repo-level `.gemini/policies/auto-saved.toml` are listed in the report for you to review. A change to your user-level config files, wherever they live, is parked; so is any change to the main checkout when the work runs in a separate worktree because that config lives in the repo. Second, a pre-existing security issue on a public repo is parked for the report with a private-advisory recommendation, never filed publicly.

The question batch covers workflow permissions (merge, deploys, new dependencies). Gemini CLI's own tool approvals are separate: in the default approval mode a run still pauses for confirmation on tool calls you have not allowed. For an unattended run, allow the commands the workflow uses in `~/.gemini/settings.json` (`"tools": {"allowed": ["run_shell_command(git)", "run_shell_command(gh)"]}`) and mark your tracker's MCP server `"trust": true` under `mcpServers`, rather than `--approval-mode=yolo`, and keep the B6 rails. [Configuration](https://github.com/google-gemini/gemini-cli/blob/main/docs/reference/configuration.md)

Hard rails (B6) apply to every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 asks in the question batch.

## Differences from Claude

| Feature | Claude | Gemini CLI |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md` | `~/.gemini/GEMINI.md`, concatenated with repo `GEMINI.md` files |
| Agent location | `~/.claude/agents/`, repo `.claude/agents/` | `~/.gemini/agents/`, repo `.gemini/agents/` |
| Agent format | Markdown with YAML frontmatter | Markdown with YAML frontmatter; strict schema, unknown keys fail the load |
| Tool restriction | `tools` allowlist or `disallowedTools` | `tools` YAML array only (wildcards `*`, `mcp_*`, `mcp_<server>_*`); omit to inherit all |
| Read-only enforcement | No edit tools; Bash limited by the prompt | No `write_file` or `replace`; `run_shell_command` limited by the prompt |
| Background | `background: true` on side-fix | None; side-fix runs in the foreground and the main task waits |
| Worktree isolation | `isolation: worktree` on side-fix | None per agent; the orchestrator creates `.worktrees/<TICKET-ID>` (B3) |
| Nested subagents | side-fix spawns `reviewer`, `verifier`, and `adversarial`; if it cannot spawn, it stops at ready for verify | Not possible; side-fix stops at `ready for verify`, the orchestrator runs its verify loop and ships |
| Model selection | `model: inherit` | `model` omitted, which defaults to `inherit` |
| Main-thread orchestrator | `agents/orchestrator.md`, run with `claude --agent orchestrator` or the `agent` setting | None; Part C of `GEMINI.md`, followed by the main session |
| Spawn allowlist | Prompt-enforced; optional `tools: Agent(...)` on the orchestrator | Prompt-enforced; optional `agents.overrides.<name>.enabled: false` or a policy deny (not shipped) |
| Resume a worker | `SendMessage` to its ID or name; it keeps its history | Not possible; re-spawn with the prior report and the findings in the brief |

Per-agent frontmatter. "Read set" is `read_file`, `read_many_files`, `grep_search`, `glob`, `list_directory`.

| Agent | `tools` | `max_turns` | `timeout_mins` |
|---|---|---|---|
| `explorer` | read set, `run_shell_command`, `web_fetch`, `google_web_search` | default (30) | default (10) |
| `implementer` | read set, `run_shell_command`, `write_file`, `replace`, `web_fetch`, `google_web_search`, `write_todos`, `activate_skill`, `mcp_*` | 60 | 30 |
| `reviewer` | read set, `run_shell_command`, `web_fetch` | default (30) | default (10) |
| `verifier` | read set, `run_shell_command` | default (30) | 30 |
| `adversarial` | read set, `run_shell_command`, `web_fetch` | 60 | 30 |
| `side-fix` | same as `implementer` | 100 | 60 |

The defaults are 30 turns and 10 minutes. Full test suites, whole tickets, and the adversarial's PoC runs outrun them, so implementer, verifier, adversarial, and side-fix get more. The implementer and side-fix lists are explicit so they exclude `ask_user`: subagents return `needs-input` to the orchestrator (B0). Both keep `activate_skill` and `mcp_*`, so they can load a workflow skill and reach the tracker and git host, as in Claude.

Model choice (B9) falls back to configured models. `invoke_agent` takes only `agent_name` and `prompt`, so the orchestrator cannot pick a model per spawn, and every agent here runs on the main session's model (`model` omitted means `inherit`). To put one agent on a cheaper tier, set `model` in its agent file, or `modelConfig.model` under `agents.overrides.<name>` in `~/.gemini/settings.json`; that applies to every spawn of that agent. [agent-tool.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/agent-tool.ts#L59-L72), [Subagents](https://geminicli.com/docs/core/subagents/)

Back-to-back subagent calls in one turn run in parallel, so B4's parallel build units work as in Claude. [scheduler.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/scheduler/scheduler.ts)

## Known gaps

- Strict frontmatter schema. Any unknown key makes an agent fail to load, so Claude agent files (`disallowedTools`, `isolation`, `background`) cannot be copied as-is. [Subagents](https://geminicli.com/docs/core/subagents/), [agentLoader.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/agentLoader.ts#L92-L116)
- No main-thread agent. The orchestrator is Part C of `GEMINI.md`, and only instructions keep it in that role. Workers load `GEMINI.md` too, so they see Part C; B0 tells them to ignore it. See Run the orchestrator. [local-executor.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/local-executor.ts#L1457-L1461)
- No nesting. Subagents cannot call other subagents, even with `*`. The orchestrator runs side-fix's verify loop (`reviewer` and `verifier` on its worktree each round, then the `adversarial`), then ships. [Subagents](https://geminicli.com/docs/core/subagents/), [local-executor.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/local-executor.ts#L204-L206)
- No resume. `invoke_agent` takes only an agent name and a prompt, and every call starts a fresh executor. Wherever B4 or Part C would resume a worker, the orchestrator re-spawns it with its prior report and the findings in the brief. [agent-tool.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/agent-tool.ts#L59-L72)
- Turn and time limits drop the report. When a subagent hits `max_turns` or `timeout_mins`, it gets one grace turn to call `complete_task`. If it does not, the parent gets only the termination reason (for example `Agent reached max turns limit (60).`), and the worker's report is lost. B9 tells the orchestrator to treat that as `partial` with no evidence. The B4 budgets must trip first: keep units small enough that a worker reports before its limit, and raise `max_turns` or `timeout_mins` in an agent file if a limit trips in practice. Loop budgets are otherwise enforced by instructions only; this port adds no frontmatter key. [local-executor.ts, grace turn](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/local-executor.ts#L444-L529), [final result](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/local-executor.ts#L758-L834)
- Spawn limit is instructions only. Part C spawns only the six workers, and nothing enforces it. Optional, not shipped: disable an agent with `agents.overrides.<name>.enabled: false` in `~/.gemini/settings.json`, or deny it with a policy rule. The built-in `generalist` agent inherits all tools; disabling it (`"agents": {"overrides": {"generalist": {"enabled": false}}}`) keeps work inside the role split. [registry.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/registry.ts#L433-L444)
- No background mode. side-fix pauses the main task until it returns. [Subagents](https://geminicli.com/docs/core/subagents/)
- No per-agent worktree. B3 has the orchestrator create one under `.worktrees/`. Session-level worktrees exist (`gemini --worktree <name>` with `experimental.worktrees: true`, default `false`, created under `.gemini/worktrees/`) but cover a whole session, not one agent. [Subagents](https://geminicli.com/docs/core/subagents/)
- File tools skip git-ignored paths (`context.fileFiltering.respectGitIgnore`, default `true`, which also honors `.git/info/exclude`). `read_file` rejects them; `glob`, `read_many_files`, and `list_directory` omit them. `.worktrees/` is excluded, so B3 and side-fix read there with `run_shell_command`. (`pr-review` never checks out the PR, so it reads everything through git.) `write_file` and `replace` only check workspace access, so edits work. Alternative: set `"context": {"fileFiltering": {"respectGitIgnore": false}}` in `~/.gemini/settings.json`; file tools then also see other ignored files, such as `node_modules`. [read-file.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/tools/read-file.ts), [fileDiscoveryService.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/services/fileDiscoveryService.ts)
- Gates that don't read git excludes (some lint and test globs) can pick up files under `.worktrees/` while a side-fix worktree exists.
- Read-only is prompt-enforced. explorer, reviewer, adversarial, and verifier have no edit tools but keep `run_shell_command`, which can write; verifier needs it for builds and tests, and adversarial for throwaway PoCs in a temp dir outside the repo, with a scrubbed environment (prompt-enforced); PR text, ticket text, branch names, and commit messages cite its findings by ID, severity, and `path:line` only. For a hard limit, add policy TOML in `~/.gemini/policies/` with `subagent = "<name>"`. None ships here because priority semantics are unverified. [Policy engine](https://github.com/google-gemini/gemini-cli/blob/main/docs/reference/policy-engine.md)

## Per-repo overrides

- A repo's `GEMINI.md` files (workspace dirs and their parents up to `.git`, plus files loaded just in time when a tool touches a directory) are concatenated with this one. If the two conflict and the A3 order doesn't settle it, the agent asks in the question batch, or parks the conflict after it. [GEMINI.md](https://geminicli.com/docs/cli/gemini-md/)
- `CLAUDE.md` is not read by default; B1 tells the agent to read `AGENTS.md` and `CLAUDE.md` itself. To load other names automatically, set `context.fileName` in `~/.gemini/settings.json`, for example `["AGENTS.md","GEMINI.md"]`. Each name is also looked up in `~/.gemini/`.
- A repo's `.gemini/agents/<name>.md` with the same `name` takes precedence over the user-level agent (trusted folders only, after you acknowledge the agent). A change to an override of an agent the loop spawns ships as a draft for your review, even when you approved it (Part C Phase 4). [registry.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/registry.ts)
- A repo override of a worker must keep the report contract: the `Status:` first line, the reviewer's finding IDs, the adversarial's verdict and `S` finding IDs, and the verifier's `Scope:` line and `FLAKY` result. Otherwise the orchestrator's exit tests cannot read its reports.
- `agents.overrides.<name>` in settings can disable or tune one agent.
