# GitHub Copilot CLI

User-level configuration for [GitHub Copilot CLI](https://docs.github.com/copilot). Everything here installs into `~/.copilot/`. Ported from `../Claude/`.

This port targets Copilot CLI only. VS Code Copilot Chat reads repo files through its own settings (for example `chat.useClaudeMdFile` for `CLAUDE.md`), and its user-level paths were not checked.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `copilot-instructions.md` | `~/.copilot/copilot-instructions.md` | Global instructions loaded into every session |
| `agents/orchestrator.agent.md` | `~/.copilot/agents/orchestrator.agent.md` | Session agent: runs the outer loop and dispatches the workers |
| `agents/explorer.agent.md` | `~/.copilot/agents/explorer.agent.md` | Read-only codebase research |
| `agents/implementer.agent.md` | `~/.copilot/agents/implementer.agent.md` | Builds one scoped unit of work |
| `agents/reviewer.agent.md` | `~/.copilot/agents/reviewer.agent.md` | Fresh-context adversarial diff review |
| `agents/verifier.agent.md` | `~/.copilot/agents/verifier.agent.md` | Runs the repo's real gates, reports pass/fail |
| `agents/side-fix.agent.md` | `~/.copilot/agents/side-fix.agent.md` | Takes an unrelated issue to its own PR, in the background, in a worktree the orchestrator creates |
| `skills/pr-review/SKILL.md` | `~/.copilot/skills/pr-review/` | Reviews a GitHub PR with the `reviewer` subagent and reports findings |

## Install

Symlinks keep `~/.copilot/` in sync with this repo, so a `git pull` updates your setup. Run from the repo root:

```bash
mkdir -p ~/.copilot/skills
for p in ~/.copilot/copilot-instructions.md ~/.copilot/agents; do
  if [ -e "$p" ] && [ ! -L "$p" ]; then mv -n "$p" "$p.bak"; fi
done
ln -sn "$PWD/Copilot/copilot-instructions.md" ~/.copilot/copilot-instructions.md
ln -sn "$PWD/Copilot/agents" ~/.copilot/agents
ln -sn "$PWD/Copilot/skills/pr-review" ~/.copilot/skills/pr-review
```

The loop moves an existing real file or directory to `.bak` (`-n` never overwrites an earlier `.bak`; if one exists, the move is skipped and `ln -sn` refuses). It leaves an existing symlink in place, so `ln -sn` then refuses with "File exists" instead of doing something silent. Plain `ln -s` is risky here: if the target is a directory, or a symlink to one, it exits 0 and creates the link *inside* it.

The agents directory holds `orchestrator.agent.md` too, so the link (or the copy below) installs it with the five workers. Linking the agents directory makes this repo the only source of your user-level agents. Add personal agents here, or copy the files (`cp`) instead of linking. Skills are linked one folder at a time, because `~/.copilot/skills/` may hold other skills too. If `~/.copilot/skills/pr-review` already exists as a real folder, move it out of `~/.copilot/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it.

Copilot also reads personal skills from `~/.agents/skills/`, so keep other tools' variants of `pr-review` out of it. ([add skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills))

Confirm it loaded: `/instructions` lists the loaded instruction files, and the `/agent` picker lists the agents, including `orchestrator`. To start development sessions with it, see [Run the orchestrator](#run-the-orchestrator).

Unverified: whether Copilot CLI follows a symlinked agents directory or a symlinked skill folder. If an agent or the skill does not show up, copy instead:

```bash
rm ~/.copilot/agents ~/.copilot/skills/pr-review   # removes the symlinks only
cp -R "$PWD/Copilot/agents" ~/.copilot/
cp -R "$PWD/Copilot/skills/pr-review" ~/.copilot/skills/
```

No settings change is needed. User settings live in `~/.copilot/settings.json`; `config.json` is auto-managed.

## Run the orchestrator

The orchestrator is a session agent: it becomes the main session, not a subagent you spawn. Start each development session with:

```bash
copilot --agent orchestrator
```

Or pick `orchestrator` in the `/agent` picker of a running session. No settings key makes it the default agent, so for a shortcut add a shell alias to `~/.zshrc` or `~/.bashrc`, for example:

```bash
alias copilot-dev='copilot --agent orchestrator'
```

Keep plain `copilot` for non-development work. Whether the agent's body replaces or adds to Copilot CLI's built-in prompt is not documented, so `orchestrator.agent.md` carries a short "Operating rules" section in case it replaces it. Repo instructions still load for a `--agent` session.

The frontmatter sets `infer: false`, so the main agent does not auto-delegate to the orchestrator (CLI reference: "Allow auto-delegation by the main agent", default `true`). It also sets `include-custom-instructions: true` for consistency with the other agents. That key has no effect here: it applies when an agent runs as a subagent, and a session agent gets repo instructions anyway. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))

A plain session follows the same loop: `copilot-instructions.md` B0 tells a main session not running as the orchestrator to read `~/.copilot/agents/orchestrator.agent.md` and follow its body for any code change.

Unverified: whether the personal `~/.copilot/copilot-instructions.md` loads for a `--agent` session. It is inferred to load; the orchestrator's Operating rules tell it to read the file if Part B is missing. See [Known gaps](#known-gaps).

## How it works

`copilot-instructions.md` has two parts:

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
ship        orchestrator      push, PR (stacked when there is more than one), move the ticket, link both ways;
                              merge bottom-up only if the batch granted it
report      orchestrator      B7: short sections, each under 250 words where possible, with a merge-order table

side issue  side-fix          own ticket, branch, and PR in a worktree the orchestrator creates; scaled-down loop, 3 fix rounds, spawned in the background
                              cannot spawn: stops at ready for verify; the orchestrator verifies and ships it
```

Every worker report starts with `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`; a report without it counts as `partial`. Workers never ask you anything; they return `needs-input`, and the orchestrator decides it (if a safe, reversible option exists) or parks it for the report. The orchestrator keeps a ledger (B8) of budgets, units and their worker IDs, fix rounds, findings, side tickets, PRs in stack order, your answers, the decisions it made for you, and parked questions. It re-posts the ledger at every phase change and rebuilds from it after a context reset.

You hear from the orchestrator twice with something to answer: the question batch after Orient, and the B7 report. In between, it decides safe, reversible gaps itself and logs them, parks anything else, and finishes the rest. The only interrupt is a committed secret. When a session opens more than one PR into the same base, they form one stack (B2): side fixes at the bottom, the main PR above them, and a draft always on top. Each PR targets the branch of the one below it, and the report's Merge order table says what to merge first.

The question batch covers workflow permissions (merge, deploys, new dependencies). Copilot CLI's own tool approvals are separate: a run still pauses on any shell command, file write, or MCP tool you have not allowed. For an unattended run, start the session with `--allow-tool` rules for what the workflow uses, for example `copilot --agent orchestrator --allow-tool='shell(git:*),shell(gh:*),write,<tracker-mcp-server>'`, and keep the B6 rails. `--deny-tool` rules win over allow rules. Unverified: whether these rules also cover the subagents the orchestrator spawns. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))

Hard rails (B6) apply to every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 asks in the question batch.

Loop budgets (B4) are enforced by instructions only.

## Differences from Claude

| Feature | Claude | Copilot CLI |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md` | `~/.copilot/copilot-instructions.md`, plain Markdown, no frontmatter |
| Agent location | `~/.claude/agents/`; repo `.claude/agents/` wins on a name clash | `~/.copilot/agents/`; on an ID clash user-level wins, then repo `.github/agents/`, then `.claude/agents/` |
| Agent format | `<name>.md`, YAML frontmatter | `<name>.agent.md`, YAML frontmatter; ID is the filename; `include-custom-instructions: true` loads repo instructions into a subagent |
| Main-thread orchestrator | `claude --agent orchestrator` or the `agent` setting; the body replaces the default system prompt | `copilot --agent orchestrator` or `/agent`; no settings key (use a shell alias); `infer: false` stops auto-delegation; whether the body replaces the built-in prompt is not documented |
| Spawn allowlist | Prompt-enforced; opt-in `tools: Agent(...)` allowlist on the main-thread agent | Instructions only: no per-agent allowlist; the `agent` alias is all or nothing; `subagents.disabledSubagents` is session-wide |
| Resume a worker | `SendMessage` to its ID or name; it keeps its history | `write_agent` messages a running agent; resuming a finished one is not documented, so re-spawn it with its prior report and the findings |
| Tool restriction | `tools` allowlist, `disallowedTools` | `tools` alias list (`read`, `edit`, `search`, `execute`, `web`, `todo`, `agent`); no deny list; implementer's list leaves out MCP tools, which Claude's implementer has; side-fix and orchestrator omit it to keep all tools |
| Read-only enforcement | Prompt; Bash kept | Prompt; no `edit` alias, but `execute` kept |
| Background | `background: true` on side-fix | Chosen per spawn, not in the agent file; the orchestrator spawns side-fix in the background (B3, B9) |
| Worktree isolation | `isolation: worktree` on side-fix | Per session only (`/new worktree`, `--worktree`); the orchestrator creates `.worktrees/<TICKET-ID>` for side-fix (B3) |
| Nested subagents | side-fix spawns `reviewer` and `verifier`; if it cannot spawn, it stops at ready for verify | Same, with the same fallback; tool filters carry into nested subagents |
| Model selection | `model: inherit` | `model` unset, so it inherits |

## Known gaps

- Read-only is prompt-enforced. explorer, reviewer, and verifier have no `edit` alias, but `execute` can still write files, and verifier needs it to run builds and tests. ([custom agents configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration))
- User agents override repo agents. A repo's `.github/agents/reviewer.agent.md` or `.claude/agents/` reviewer does not replace yours; yours wins. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
- Background is per spawn. No agent-file key makes side-fix run in the background, so it does so only when the orchestrator spawns it that way. `read_agent` checks a background agent's status, and `/tasks` lists running tasks. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
- Custom agents spawned as subagents get only the repo's instruction files (`copilot-instructions.md`, `AGENTS.md`, `CLAUDE.md`), and only through `include-custom-instructions: true`, which every agent here sets. Unverified: whether the personal `~/.copilot/copilot-instructions.md` reaches them. If it does not, subagents never see Part A, B0, or the B4 loop contract, and the workers' references to B4 and side-fix's references to B2, B5, and B6 of `copilot-instructions.md` point at nothing. Each worker's Loop section and report format still carry its own budget, stall rule, and `Status:` line. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference), [create custom agents](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli))
- Not documented: whether the orchestrator's body replaces or adds to Copilot CLI's built-in prompt when it runs as the session agent. `orchestrator.agent.md` keeps an "Operating rules" section in case it replaces it. ([custom agents configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration), [create custom agents](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli))
- Inferred, not verified: the personal `~/.copilot/copilot-instructions.md` loads for a `--agent` session. Repo instructions do: the CLI reference lists the session agent, including one selected with `--agent`, as receiving them. If the personal file does not load, the orchestrator never sees Part A or B, so its Operating rules tell it to read the file when Part B is missing. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
- `infer` is unsettled. The CLI reference lists `infer` ("Allow auto-delegation by the main agent", default `true`), and `orchestrator.agent.md` sets `infer: false`. The cross-product custom agents configuration page calls `infer` retired in favor of `disable-model-invocation`, which the CLI reference does not list. Not documented: whether `infer: false` also blocks an explicit `task` call that names the agent. B0, B9, and the agent's description say never to spawn it. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference), [custom agents configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration))
- Spawn limits are instructions only. There is no per-agent spawn allowlist: the `agent` tool alias is all or nothing, and `subagents.disabledSubagents` turns subagents off for the whole session. The orchestrator's "five workers only" and side-fix's "`reviewer` and `verifier` only" rest on the prompt. ([custom agents configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration), [CLI config dir reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference))
- Resume is limited. `write_agent` sends a message to a running agent; resuming a finished one is not documented. B4 re-spawns a finished worker with its prior report and the findings in the brief, so the new instance starts without the old one's context. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
- The default subagent depth is 6 in the CLI reference's limits table and 4 for `COPILOT_SUBAGENT_MAX_DEPTH` on the same page. Either way orchestrator, then side-fix, then `reviewer` or `verifier` fits. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
- Gates that don't read git excludes (some lint and test globs) can pick up files under `.worktrees/` while a side-fix worktree exists. ([gitignore](https://git-scm.com/docs/gitignore))
- Unverified: `@relative/path` imports in the personal instructions file. They work in repo `.github/copilot-instructions.md`, `AGENTS.md`, and `CLAUDE.md`. ([custom instructions](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions))
- Not part of this port: hooks (`~/.copilot/hooks/*.json`). ([use hooks](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/use-hooks))

## Per-repo overrides

- Copilot CLI reads a repo's `.github/copilot-instructions.md`, `.github/instructions/**`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, and `.claude/rules`, at the root, the cwd, the directories in between, and nested paths. They are combined with this file and de-duplicated, with no general precedence order. If they conflict and the A3 order doesn't settle it, the agent asks in the question batch, or parks the conflict after it. ([custom instructions](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions))
- `/instructions` toggles individual files; `--no-custom-instructions` skips them all.
- A repo's `.github/agents/<name>.agent.md` or `.claude/agents/` agent does not replace the user-level agent with the same ID. The user-level agent wins, then `.github/agents/`, then `.claude/agents/`. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
- A repo agent that stands in for a worker (it loads only where you have no user-level agent with that ID) must keep the report contract: the `Status:` first line, the reviewer's finding IDs, and the verifier's `Scope:` line and `FLAKY` result. Otherwise the orchestrator's exit tests cannot read its reports.
