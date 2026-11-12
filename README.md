# agentic-dev-configs

User-level configuration for AI coding agents, one directory per tool. Each directory carries the same development workflow, an orchestrator, the same six worker subagents, and the same skills, written in that tool's native format. Where a tool lacks a capability the workflow relies on, the port adapts the mechanics and keeps the rules.

## Layout

| Directory | Tool | Global instructions | Subagents | Skills |
|---|---|---|---|---|
| [`Claude/`](Claude/) | Claude Code | `~/.claude/CLAUDE.md` | `~/.claude/agents/*.md` | `~/.claude/skills/` |
| [`Codex/`](Codex/) | OpenAI Codex | `~/.codex/AGENTS.md` | `~/.codex/agents/*.toml` | `~/.codex/skills/` |
| [`Cursor/`](Cursor/) | Cursor | Settings → Customize → Rules | `~/.cursor/agents/*.md` | `~/.cursor/skills/` |
| [`Gemini/`](Gemini/) | Gemini CLI | `~/.gemini/GEMINI.md` | `~/.gemini/agents/*.md` | `~/.gemini/skills/` |
| [`Copilot/`](Copilot/) | GitHub Copilot CLI | `~/.copilot/copilot-instructions.md` | `~/.copilot/agents/*.agent.md` | `~/.copilot/skills/` |
| [`OpenCode/`](OpenCode/) | OpenCode | `~/.config/opencode/AGENTS.md` | `~/.config/opencode/agents/*.md` | `~/.config/opencode/skills/` |

`Claude/README.md` explains the workflow in full. Each port's README has install steps, a table of differences from Claude, and known gaps. Install only the tools you use.

## The workflow

The global instructions file has two parts.

**Part A** applies to every task: never guess (gather context, then ask questions and permissions in one early batch), keep replies short, separate verified from inferred, and follow a fixed precedence (your instruction in the session, then a project workflow skill, then the global file).

**Part B** is the development workflow. Every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The work runs as agentic loops: gather context, act, check against ground truth (command output, test results, or a verdict from an agent that did not write the code), and decide. A loop repeats until its exit test passes, its budget runs out, progress stalls, or it needs your input.

| Agent | Role | Edits |
|---|---|---|
| `orchestrator` | Runs as the main session and owns the outer loop: ticket, plan, dispatch, commits, finding routing, ship, report | No product code; git state, ticket and PR text |
| `explorer` | Read-only research: file map, call chains, the repo's real gates and conventions | No |
| `implementer` | Builds one scoped unit, tests first; leaves the diff for the orchestrator to commit | In-scope files only |
| `reviewer` | Fresh-context adversarial review of a diff against the ticket and standards | No |
| `verifier` | Runs build, tests, lint, and type check exactly as CI does; reports raw results | No |
| `adversarial` | The last stage of every verify loop: attacks the finished change for security loopholes and exploitable issues; nothing ships as ready without its approval | No |
| `side-fix` | Takes an unrelated issue found mid-task to its own ticket, branch, and PR without touching the main working tree | Yes |

- **Outer loop.** The orchestrator runs intake, orient, plan, build, verify, ship, and report, and handles all contact with you. It gathers context first, asks one batch of questions and permission requests at plan time, then runs to the report without asking.
- **Inner loops.** Each worker runs its own loop on one bounded unit. Its report starts `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`. Workers never ask you anything; they return `needs-input`, and the orchestrator decides or parks it.
- **Verify.** An evaluator loop: each round, a fresh `reviewer` and a `verifier` check the full diff, and findings go back to an implementer as fix briefs. At most 3 fix rounds. Once they pass, a fresh `adversarial` agent attacks the change for exploitable issues, and its findings go back the same way. Nothing ships as ready without its approval: from its first pass, every round counts against its own budget of 3, and when that runs out the main PR ships as a draft. PR text, ticket text, branch names, and commit messages cite its findings by ID, severity, and `path:line` only.
- **Workflow and security gates.** Workflow-control paths (each AI tool's folder, its agent files and MCP config, and instruction files such as `CLAUDE.md` and `AGENTS.md`) are checked each time an implementer returns and before every review; the question batch lists the ones the plan will touch so you can approve them up front. Unapproved and incidental: a fix brief removes the change (a file the session created is deleted, a tracked file restored; an untracked or ignored file that existed before the session is never deleted or overwritten, and a changed one is parked). Unapproved but needed: parked, and the main PR ships as a draft. An approved change to hooks, settings, MCP config, or a spawned agent's definition also makes it a draft; approved instruction-file and skill changes proceed. A pre-existing security issue on a public repo is parked for the report with a private-advisory recommendation, never filed publicly.
- **Limits.** Every loop has a budget and a stall rule: stop when the same check fails the same way after two different fixes, or when a pass adds no new evidence. After the question batch, a gap with a safe, reversible option is decided and logged. When none exists, a budget runs out, or progress stalls, the orchestrator parks the affected work, finishes the rest, and puts the question in the final report. Only a committed secret interrupts.
- **Ledger.** The orchestrator tracks units, fix rounds, open findings, side tickets, PRs, decisions, and parked questions in a ledger. It re-posts the ledger at every phase change except into the report, and rebuilds from it after a context reset.
- **Stacked PRs.** When a session opens more than one PR into the same base, they form one stack: each PR targets the branch of the one below it. Side fixes sit below the main PR, which opens only after every side-fix has ended. Every stack merge gets a verifier and a fresh adversarial pass before the PR moves. Stacks merge bottom-up with merge commits, and only when you granted merge.
- **Model choice.** Where a tool lets the orchestrator pick a subagent's model per spawn, it picks the most cost-efficient model that can still meet the unit's exit test: a cheaper tier for bounded, checkable work, the strongest for reviewers, the adversarial agent, and hard units. Every result still passes the same checks.
- **Final report.** Short sections in a fixed order (what changed, verification, merge order, findings, decisions made for you, recommendations, follow-ups, open questions), each under 250 words where possible.

`Claude/README.md` has the phase diagram and each worker's loop; B4 of the global file has the full contract.

Hard rails hold for every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, and no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 asks in the question batch.

## Skills

| Skill | What it does | Tier | Owner | Review by |
|---|---|---|---|---|
| `pr-review` | Reviews a GitHub PR by number or URL: fetches it with git hooks off and reads it from the object store (no checkout), finds its ticket, reads CI, runs the `reviewer` subagent, confirms the serious findings, and reports in chat. Never runs the PR's code unless asked. Posts to GitHub only when asked, after you approve the exact text. | Personal | Eric Lakich | 2027-04-07 |

Every tool gets its own copy in `<Tool>/skills/pr-review/`. The steps, rules, and report format are identical. Only two spots differ: the line naming the tool's global instructions file, and the "Tool notes" section at the end.

Install each copy into its own tool's skills directory (see the tool's README), one skill folder at a time. Two things to know:

- Don't put any variant in `~/.agents/skills/`. Codex, Cursor, Gemini, Copilot, and OpenCode all read it, and Gemini prefers it over its own directory, so one tool's variant would load in the others.
- Cursor also reads `~/.claude/skills/` and `~/.codex/skills/`, and OpenCode also reads `~/.claude/skills/`. With several tools installed, those two can see more than one `pr-review`. The copies differ only in those two spots.

## How the tools differ

Section numbers, rules, agent roles, loop contracts, and report formats match across directories. Where a tool lacks a capability, the orchestrator's form, B0, B3, B4's resume, verify-budget, and adversarial-budget lines, B9, and side-fix's hand-off change:

| Capability | Claude | Codex | Cursor | Gemini | Copilot | OpenCode |
|---|---|---|---|---|---|---|
| Edit tools removed from read-only agents | Yes | No (prompt only) | Partly (`readonly` on explorer, reviewer, and adversarial) | Yes | Yes | Yes (`permission`) |
| side-fix runs in the background | Yes | Yes (spawns never block) | Yes | No | Yes, per spawn | Opt-in, experimental |
| Per-agent worktree | Yes | No | Yes, requested per spawn (worktree or cloud copy) | No | No | No |
| A subagent can spawn subagents | Yes | V1 no, V2 yes (the port keeps spawning to the orchestrator) | One level below main | No | Yes | No by default (`subagent_depth` defaults to 1); the port does not rely on nesting |
| Orchestrator form | Main-thread agent (`claude --agent orchestrator`) | Part C of the global file | Part C of the global file | Part C of the global file | Agent file (`copilot --agent orchestrator`) | `mode: primary` agent (`opencode --agent orchestrator`) |
| Hard spawn allowlist on the orchestrator | Opt-in, orchestrator only: `tools` line; drops unlisted MCP tools | No (instructions only) | No (instructions only) | Opt-in, session-wide: `agents.overrides.<name>.enabled: false` | Opt-in, session-wide: `subagents.disabledSubagents` | Yes (`permission.task`) |
| Resume a worker with its context | Yes (`SendMessage`) | Yes (V1 `send_input`, V2 `followup_task`) | Yes (agent ID) | No (re-spawn) | Finished worker: not documented, re-spawn; `write_agent` messages a running one | Yes (`task_id`) |
| Pick a subagent's model per spawn | Yes (Agent tool `model`) | Yes (`spawn_agent` `model`, an exact slug; a role file's `model` wins) | Partly: the parent can name a model at launch (parameter and precedence unverified) | No (agent file or `agents.overrides`) | No (agent file or `/subagents`) | No (agent file, else the caller's model) |

Five fallbacks cover the gaps:

- **No per-agent worktree (Codex, Gemini, Copilot, OpenCode):** the orchestrator creates `.worktrees/<TICKET-ID>` inside the repo, excluded through `.git/info/exclude` and never `.gitignore`, and passes side-fix the path and base SHA. side-fix confirms it is in that worktree before changing anything. Gates that don't read git excludes, such as some lint and test globs, can pick up those files while the worktree exists.
- **side-fix can't spawn (Codex, Gemini, OpenCode):** side-fix stops at "ready for verify". The orchestrator runs side-fix's verify loop against the worktree (a fresh `reviewer` and a `verifier` each round, at most 3 fix rounds, then the adversarial pass, at most 3 more), sends findings back to side-fix, and ships the side PR itself.
- **Spawn blocked at runtime (any tool):** side-fix stops at "ready for verify" with its worktree path, branch, and HEAD; the orchestrator runs its verify loop and ships it, as above.
- **No main-thread agent (Codex, Cursor, Gemini):** the orchestrator procedure is Part C of the global file; it applies only to the main session, and workers ignore it.
- **No documented resume (Gemini; Copilot after a worker finishes):** re-spawn with the prior report and findings.

## Keeping the ports in sync

`Claude/` is the canonical version. Change it first, then carry the same change into each port.

The ports keep Claude's section numbers (A1 to A3, B0 to B9) and wording, so a diff shows only the tool-specific adaptations. From the repo root, compare Part A and Part B of each global file:

```bash
for f in Codex/AGENTS.md Cursor/user-rules.md Gemini/GEMINI.md Copilot/copilot-instructions.md OpenCode/AGENTS.md; do
  echo "== $f"; diff <(sed -n '/^## Part A/,/^## Part C/p' Claude/CLAUDE.md) <(sed -n '/^## Part A/,/^## Part C/p' "$f")
done
```

Expect hunks only here:

- A3, B1, and B8: instruction file names and the instruction chain.
- B0: the line that says where the orchestrator runs and is defined, and the plain-session line. Codex also changes the worker line.
- B3: the side-fix bullet.
- B4: the "Resume vs fresh" line, the "Outer loop" pointer line, and, where side-fix can't spawn (Codex, Gemini, OpenCode), the verify-budget and adversarial-budget lines.
- B9: the intro line, the tool notes, in some ports the `orchestrator` and `side-fix` rows, and, where side-fix can't spawn or can't pick a model (Codex, Copilot, Gemini, OpenCode), the Model choice intro line. In Part C ports, the last hunk combines B9's changes with the `## Part C` heading.

Then compare the orchestrator. Part C ports hold it at the end of the global file; Copilot and OpenCode have an agent file:

```bash
for f in Codex/AGENTS.md Cursor/user-rules.md Gemini/GEMINI.md; do
  echo "== $f"; diff <(sed '1,/^---$/d' Claude/agents/orchestrator.md) <(sed -n '/^## Part C/,$p' "$f")
done
for f in Copilot/agents/orchestrator.agent.md OpenCode/agents/orchestrator.md; do
  echo "== $f"; diff Claude/agents/orchestrator.md "$f"
done
```

Every hunk should be a tool mechanic:

- Agent-file ports: the frontmatter.
- Part C ports: the `## Part C` heading, a line limiting Part C to the main session, and every heading one level down.
- Every port: the intro line (it names the global file), the Operating rules lines on the default prompt and on spawning and resuming, Side loops, the Orient line that resolves the user-level config path, and the Phase 4 lines that name the tool's own folder, instruction files, and local settings file. Most ports also change the background-workers line; where a worker can't be resumed, the resume lines in Phases 3 and 4 change too. Where the brief has no `Target` line (Codex, Gemini, OpenCode), the Phase 4 adversarial spawn line changes too.

Diff the workers and skills file by file the same way, for example `diff Claude/skills/pr-review/SKILL.md Gemini/skills/pr-review/SKILL.md`.

## Adding a tool

1. From the tool's official docs, find: the global instructions path, the subagent and skills directories and file formats, the supported frontmatter fields, and whether it supports tool restriction, background runs, per-agent worktrees, and nested subagents. Also find whether the main session can run as a custom agent, whether a subagent can be resumed with its context, and whether spawns can be restricted per agent.
2. Create `<Tool>/` with the instructions file, the agent definitions, the skills, and a README with the same sections as the existing ports (Contents, Install, Run the orchestrator, How it works, Differences from Claude, Known gaps, Per-repo overrides). Ship seven agents (the orchestrator and the six workers) if the main session can run as a custom agent; otherwise ship the six workers and put the orchestrator in Part C of the global file.
3. Change only what the tool's mechanics force. Use the fallbacks above where the tool lacks isolation, nesting, a main-thread agent, or resume.
4. Add the tool to the Layout and capability tables and to both sync commands here.

## Notes

- The question batch covers workflow permissions (merge, deploys, new dependencies), not each tool's own approval prompts. For an unattended run, pre-allow the commands the workflow uses (`git`, `gh`, your tracker's tools) in the tool's permission settings.
- This repo is public. Never commit auth files, API keys, or a copy of a tool's home directory. With symlinked installs, anything a tool writes into a linked directory lands in this working tree, so check `git status` before committing.
- The tool facts behind each port were checked against official docs and source in October 2026. These tools change quickly. If an agent or skill stops loading, check the port's Known gaps and the tool's current docs first.
