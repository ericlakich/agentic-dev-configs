# agentic-dev-configs

User-level configuration for AI coding agents, one directory per tool. Each directory carries the same development workflow, an orchestrator, the same five worker subagents, and the same skills, written in that tool's native format. Where a tool lacks a capability the workflow relies on, the port adapts the mechanics and keeps the rules.

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

**Part A** applies to every task: never guess (ask in one early batch), keep replies short, separate verified from inferred, and follow a fixed precedence (your instruction in the session, then a project workflow skill, then the global file).

**Part B** is the development workflow. Every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The work runs as agentic loops: gather context, act, check against ground truth (command output, test results, or a verdict from an agent that did not write the code), and decide. A loop repeats until its exit test passes, its budget runs out, progress stalls, or it needs your input.

| Agent | Role | Edits |
|---|---|---|
| `orchestrator` | Runs as the main session and owns the outer loop: ticket, plan, dispatch, commits, finding routing, ship, report | No product code; git state, ticket and PR text |
| `explorer` | Read-only research: file map, call chains, the repo's real gates and conventions | No |
| `implementer` | Builds one scoped unit, tests first; leaves the diff for the orchestrator to commit | In-scope files only |
| `reviewer` | Fresh-context adversarial review of a diff against the ticket and standards | No |
| `verifier` | Runs build, tests, lint, and type check exactly as CI does; reports raw results | No |
| `side-fix` | Takes an unrelated issue found mid-task to its own ticket, branch, and PR without touching the main working tree | Yes |

- **Outer loop.** The orchestrator runs intake, orient, plan, build, verify, ship, and report, and handles all contact with you.
- **Inner loops.** Each worker runs its own loop on one bounded unit. Its report starts `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`. Workers never ask you anything; they return `needs-input`, and the orchestrator batches the questions.
- **Verify.** An evaluator loop: each round, a fresh `reviewer` and a `verifier` check the full diff, and findings go back to an implementer as fix briefs. At most 3 fix rounds.
- **Limits.** Every loop has a budget and a stall rule: stop when the same check fails the same way after two different fixes, or when a pass adds no new evidence. When a budget runs out, progress stalls, or only you can decide, the orchestrator escalates in one message: the ledger, the decision needed, the options, and a recommendation.
- **Ledger.** The orchestrator tracks units, verify rounds, open findings, and side tickets in a ledger. It re-posts the ledger at every phase change and rebuilds from it after a context reset.

`Claude/README.md` has the phase diagram and each worker's loop; B4 of the global file has the full contract.

Hard rails hold for every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, and no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 stops and asks.

## Skills

| Skill | What it does | Tier | Owner | Review by |
|---|---|---|---|---|
| `pr-review` | Reviews a GitHub PR by number or URL: fetches it with git hooks off and reads it from the object store (no checkout), finds its ticket, reads CI, runs the `reviewer` subagent, confirms the serious findings, and reports in chat. Never runs the PR's code unless asked. Posts to GitHub only when asked, after you approve the exact text. | Personal | Eric Lakich | 2027-04-07 |

Every tool gets its own copy in `<Tool>/skills/pr-review/`. The steps, rules, and report format are identical. Only two spots differ: the line naming the tool's global instructions file, and the "Tool notes" section at the end.

Install each copy into its own tool's skills directory (see the tool's README), one skill folder at a time. Two things to know:

- Don't put any variant in `~/.agents/skills/`. Codex, Cursor, Gemini, Copilot, and OpenCode all read it, and Gemini prefers it over its own directory, so one tool's variant would load in the others.
- Cursor also reads `~/.claude/skills/` and `~/.codex/skills/`, and OpenCode also reads `~/.claude/skills/`. With several tools installed, those two can see more than one `pr-review`. The copies differ only in those two spots.

## How the tools differ

Section numbers, rules, agent roles, loop contracts, and report formats match across directories. Where a tool lacks a capability, the orchestrator's form, B0, B3, B4's resume line, B9, and side-fix's hand-off change:

| Capability | Claude | Codex | Cursor | Gemini | Copilot | OpenCode |
|---|---|---|---|---|---|---|
| Edit tools removed from read-only agents | Yes | No (prompt only) | Partly (`readonly` on explorer and reviewer) | Yes | Yes | Yes (`permission`) |
| side-fix runs in the background | Yes | Yes (spawns never block) | Yes | No | Yes, per spawn | Opt-in, experimental |
| Per-agent worktree | Yes | No | Yes, requested per spawn (worktree or cloud copy) | No | No | No |
| A subagent can spawn subagents | Yes | V1 no, V2 yes (the port keeps spawning to the orchestrator) | One level below main | No | Yes | No by default (`subagent_depth` defaults to 1); the port does not rely on nesting |
| Orchestrator form | Main-thread agent (`claude --agent orchestrator`) | Part C of the global file | Part C of the global file | Part C of the global file | Agent file (`copilot --agent orchestrator`) | `mode: primary` agent (`opencode --agent orchestrator`) |
| Hard spawn allowlist on the orchestrator | Opt-in `tools` line; drops unlisted MCP tools | No (instructions only) | No (instructions only) | No (instructions only) | No (instructions only) | Yes (`permission.task`) |
| Resume a worker with its context | Yes (`SendMessage`) | Yes (V1 `send_input`, V2 `followup_task`) | Yes (agent ID) | No (re-spawn) | Running agents only (`write_agent`); otherwise re-spawn | Yes (`task_id`) |

Five fallbacks cover the gaps:

- **No per-agent worktree (Codex, Gemini, Copilot, OpenCode):** the orchestrator creates `.worktrees/<TICKET-ID>` inside the repo, excluded through `.git/info/exclude` and never `.gitignore`, and passes side-fix the path and base SHA. side-fix confirms it is in that worktree before changing anything. Gates that don't read git excludes, such as some lint and test globs, can pick up those files while the worktree exists.
- **side-fix can't spawn (Codex, Gemini, OpenCode):** side-fix stops at "ready for verify". The orchestrator runs side-fix's verify loop against the worktree (a fresh `reviewer` and a `verifier` each round, at most 3 fix rounds), sends findings back to side-fix, and ships the side PR itself.
- **Spawn blocked at runtime (any tool):** side-fix runs the gates itself and says in its report that the review was not independent.
- **No main-thread agent (Codex, Cursor, Gemini):** the orchestrator procedure is Part C of the global file; it applies only to the main session, and workers ignore it.
- **No resume (Gemini; Copilot after a worker finishes):** re-spawn with the prior report and findings.

## Keeping the ports in sync

`Claude/` is the canonical version. Change it first, then carry the same change into each port.

The ports keep Claude's section numbers (A1 to A3, B0 to B9) and wording, so a diff shows only the tool-specific adaptations. From the repo root, compare Part A and Part B of each global file:

```bash
for f in Codex/AGENTS.md Cursor/user-rules.md Gemini/GEMINI.md Copilot/copilot-instructions.md OpenCode/AGENTS.md; do
  echo "== $f"; diff <(sed -n '/^## Part A/,/^## Part C/p' Claude/CLAUDE.md) <(sed -n '/^## Part A/,/^## Part C/p' "$f")
done
```

Expect hunks in A3, B0, B1, B3, B4, B8, and B9. B0's launch line and B4's "Resume vs fresh" line differ per tool. In Part C ports, the last hunk is the `## Part C` heading.

Then compare the orchestrator. Part C ports hold it at the end of the global file; Copilot and OpenCode have an agent file:

```bash
for f in Codex/AGENTS.md Cursor/user-rules.md Gemini/GEMINI.md; do
  echo "== $f"; diff <(sed '1,/^---$/d' Claude/agents/orchestrator.md) <(sed -n '/^## Part C/,$p' "$f")
done
for f in Copilot/agents/orchestrator.agent.md OpenCode/agents/orchestrator.md; do
  echo "== $f"; diff Claude/agents/orchestrator.md "$f"
done
```

Every hunk should be a tool mechanic: frontmatter, Part C's heading and heading levels, Operating rules, spawn and resume lines, or side-fix's verify loop. Diff the workers and skills file by file the same way, for example `diff Claude/skills/pr-review/SKILL.md Gemini/skills/pr-review/SKILL.md`.

## Adding a tool

1. From the tool's official docs, find: the global instructions path, the subagent and skills directories and file formats, the supported frontmatter fields, and whether it supports tool restriction, background runs, per-agent worktrees, and nested subagents. Also find whether the main session can run as a custom agent, whether a subagent can be resumed with its context, and whether spawns can be restricted per agent.
2. Create `<Tool>/` with the instructions file, the agent definitions, the skills, and a README with the same sections as the existing ports (Contents, Install, Differences from Claude, Known gaps, Per-repo overrides). Ship six agents (the orchestrator and the five workers) if the main session can run as a custom agent; otherwise ship the five workers and put the orchestrator in Part C of the global file.
3. Change only what the tool's mechanics force. Use the fallbacks above where the tool lacks isolation, nesting, a main-thread agent, or resume.
4. Add the tool to the Layout and capability tables and to both sync commands here.

## Notes

- This repo is public. Never commit auth files, API keys, or a copy of a tool's home directory. With symlinked installs, anything a tool writes into a linked directory lands in this working tree, so check `git status` before committing.
- The tool facts behind each port were checked against official docs and source in October 2026. These tools change quickly. If an agent or skill stops loading, check the port's Known gaps and the tool's current docs first.
