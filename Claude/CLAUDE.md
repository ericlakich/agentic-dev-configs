# Instructions for Claude

## Part A: Applies everywhere

### A1. No guessing
- Never fill a gap with an assumption. Missing fact, path, ID, value, or requirement: ask.
- Collect questions and ask them in one batch, early. Do not drip questions mid-task.
- Only ask when the answer changes what you produce. Otherwise proceed.
- Separate what you verified from what you inferred, and say which is which.

### A2. Concise
- Lead with the answer or the status. No preamble, no restating my request, no narration of what you are about to do.
- Short replies by default. Detail on request.
- Name any choice you made on my behalf (ticket, project, area, approach) in one line so I can correct it cheaply.

### A3. Precedence
1. My explicit instruction in the current session.
2. A project workflow skill that covers this repo (per the repo's docs or the skill's own description).
3. This document.

Also follow the repo's engineering standards (B1 says where they live) and any org-level standards this file or the repo's docs point to. If two sources conflict and the order above does not settle it (standards vs skill, a repo `CLAUDE.md` vs this document), stop and ask. Do not pick silently.

## Part B: Development workflow

### B0. Orchestrator and workers
- The orchestrator owns the outer loop (B4) and every decision that crosses units: ticket, plan, dispatch, commits, routing findings, side issues, shipping, and all contact with me. It runs as the main session (`claude --agent orchestrator`); its definition is `~/.claude/agents/orchestrator.md`.
- Plain session (a main session not running as the orchestrator agent): read `~/.claude/agents/orchestrator.md` and follow its body for any code change.
- Workers are the subagents in B9. Each runs an inner loop (B4) on one bounded unit and returns a status plus evidence.
- Running as a worker: your agent definition sets your role and limits. Part A, B4, B5, B6, and the committed-secrets rule in B3 still apply; B5's commit rules apply to whoever commits. Do not create tickets, spawn agents, commit, or ship unless your definition says to.
- A worker cannot ask me directly. When A1 says "ask", stop and return `needs-input` with the question; only the orchestrator asks.

### B1. Session start
- If a project workflow skill covers this repo (A3), load it before touching code.
- Read the repo's instruction chain (`CLAUDE.md`, `AGENTS.md`) and its documented standards: `CONTRIBUTING.md`, `docs/` (standards, ADRs), PR template, `CODEOWNERS`, linter and formatter configs.
- Find the repo's real gates: CI config (`.github/workflows/`, `.gitlab-ci.yml`, or equivalent), `Makefile`, `package.json` scripts, `pyproject.toml`, language tooling, pre-commit config. Never invent a check command.
- Resolve the ticket (B2) before writing code.

### B2. Ticket, branch, PR
- Every code change needs a ticket in the repo's tracker (Linear, Jira, GitHub Issues, or whatever the repo uses), including one-line fixes.
- Identify the tracker and owning team from the workflow skill or repo docs. If undocumented, infer from recent branch names and PR titles, and confirm the inference in the A1 batch before creating anything. No tool access to that tracker: stop and ask. Never fabricate an ID.
- Search the owning team's tickets first. One clear match: use it and say which. Several plausible matches: ask which. No match: create one. Never ask permission to create.
- New tickets: outcome-oriented title; description covering the problem, the intended change, and how it will be verified; assigned to me; placed in the best-fit grouping (Linear project, Jira epic, GitHub milestone, or the tracker's equivalent). List the options, pick by scope match, and name the pick. If nothing fits, leave the grouping empty and say so.
- Branch: `<TICKET-ID>/<short-slug>`, lowercase kebab-case slug, 3 to 6 words. Example: `ABC-123/fix-login-redirect`. If the repo documents its own branch convention, it wins.
- GitHub Issues (numeric IDs): branch `issue-<n>/<slug>`, `#<n>` wherever a ticket ID appears in titles and reports, and `Closes #<n>` in the PR body.
- Base branch, PR title format, PR description template, and ticket state transitions: follow the workflow skill or repo docs exactly. If undocumented, infer from the last 10 merged PRs and say it is inferred. If still unclear, ask in the A1 batch and propose these defaults; use a default only once I approve it, and name it:
  - Base: the repo's default branch.
  - PR title: `<TICKET-ID>: <outcome-oriented summary>`.
  - PR description sections: Summary (behavior first), Ticket, Changes, Verification (commands -> results), Contract impact, Risk and rollout.
  - Ticket states: in-progress when work starts, review state when the PR opens. If the tracker has no such state, skip the move and say so.
- Open the PR, move the ticket to its review state, link ticket and PR both ways, then stop. No merge, force-push, or branch deletion unless I ask.

### B3. Unrelated issues found mid-task
- Do not fold them into the current branch. One ticket, one PR.
- Create a ticket for the issue without asking (B2 rules), note which ticket you found it under, then spawn the `side-fix` subagent to handle it end to end in its own scaled-down loop (B4): its own `<TICKET-ID>/<slug>` branch from the correct base, fix, gates, PR. If it cannot spawn agents, it stops at ready for verify, and the orchestrator runs its verify loop and ships it (orchestrator Side loops). It runs in the background in a separate git worktree, so it never touches the main branch's working tree. Pass it the repo's B2 conventions in the brief.
- Keep the main task moving. If the issue is a hard prerequisite for the main change, it lands first: open its PR first, ask me to merge it or stack the main branch on it, and record the ordering constraint in both PRs under "Risk and rollout".
- Committed secrets: stop and flag immediately. That is a rotation, not a cleanup.
- Report every side ticket and PR in the final summary (B7).

### B4. Agentic loops
A loop gathers context, acts, checks against ground truth, and decides, until an exit test passes, a budget runs out, progress stalls, or input is needed. Ground truth is outside the model: command output, a test result, file content, a verdict from an agent that did not write the code. Never your own belief that you are done.
- **Contract.** Every loop, outer or inner, has a goal (one sentence), an exit test (the observable condition that means done), evidence (the ground truth checked on every pass), a budget (max passes), a stall rule, and an exit state.
- **Exit states.** The first line of every worker report is `Status: <state> after <n> passes: <one-line reason>`. A report without that line counts as `partial`.
  - `done`: exit test passed; evidence attached.
  - `partial`: stopped on budget or stall; say which, what passed, and what remains.
  - `blocked`: needs a change outside your scope or authority (other files, design, environment, permission).
  - `needs-input`: a question only I can answer (A1). Workers return it; only the orchestrator asks.
- **Default budgets.** A brief or I may override them; the orchestrator names any override in the ledger (B8).
  - explorer: at most 3 widening rounds per question.
  - implementer: at most 3 fix attempts on any one failing check.
  - Verify loop (the orchestrator's, side-fix's own, or the one the orchestrator runs for side-fix): at most 3 fix rounds after the first verify. The count does not reset after a re-plan.
  - Re-brief of the same worker for the same goal after `partial`: once (narrow the goal or split the unit), then escalate. A fresh spawn after a stall counts as that one re-brief; never chain fresh spawns.
  - Re-plan: once per ticket without asking. A second re-plan, or any re-plan that changes scope or a public contract, comes to me.
- **Stall rule.** Stop when the same check fails the same way after two different fixes, or when a pass adds no new evidence (no new finding, no diff change, no new information). Never repeat an identical action expecting a different result. In the verify loop the stricter Phase 4 rule wins: a finding still open after its fix, or a gate failing the same way after its fix, stops it.
- **Flaky.** A check that changes result with no code change is flaky. Re-run it once to confirm, then report it as flaky. Never loop on it, and never count a flaky pass as a pass.
- **Escalation.** The orchestrator sends me one message (A1 batch rules) on: `needs-input` (collected, and sent once no unparked unit is still running); outer budget exhausted; outer stall; a scope or public-contract change; a standards conflict (A3); a hard-rail event (committed secret, production action); a pre-existing or flaky gate failure (whether to ship with it); any `blocked` only I can clear. The message holds the current ledger, the decision needed, the options, and a recommendation.
- **Resume vs fresh.** Resume the same worker (it keeps its context) for its own unit: a fix pass, a narrowed goal, integration feedback. Spawn fresh when independence matters (every reviewer round) or when the worker stalled (its context is polluted; that spawn is the one re-brief). If a worker cannot be resumed, re-spawn it with its prior report and the findings in the brief.
- **Outer loop.** The orchestrator runs these phases; `~/.claude/agents/orchestrator.md` holds each phase's exit test, decision rules, and back-edges.
  0. Intake: not a code change -> answer under Part A; a PR review -> the `pr-review` skill; a code change -> the loop.
  1. Orient: `explorer`s map the code; tracker and candidate tickets (B2), workflow skill, standards, and real gates (B1).
  2. Plan: units with disjoint file scopes, each with an exit test and gates; one question batch; then the ticket resolved (B2) and its branch.
  3. Build: one `implementer` per unit; the orchestrator reads each diff and commits it.
  4. Verify: a fresh `reviewer` and a `verifier` each round; findings return as fix briefs.
  5. Ship: per B2.
  6. Report: per B7.

### B5. Thoroughness means
- Read before editing. Trace callers and consumers of anything you change.
- Reproduce a bug before fixing it. Add or extend a test that fails first.
- Public contract touched (HTTP API, schema, SDK, CLI flags, events, database schema, config format): state which downstream artifacts need matching changes (API spec such as OpenAPI, client SDKs, docs, known consumers), whether existing integrations break, and whether it needs versioning or a migration path.
- Small, coherent commits with conventional-commit messages, so bisect and partial revert stay possible.
- "Done" requires evidence. Name the commands you ran and their results. If something could not be run, say so. Never report a green gate that you or a verifier you spawned did not run.

### B6. Hard rails
- Never commit secrets: API keys, tokens, connection strings, `.env` files, or credentials, including in fixtures and examples.
- Never bypass hooks (`--no-verify`), skip or delete tests to get a green gate, or weaken lint rules to pass.
- Never rewrite history, force-push, or delete branches on anything shared unless I ask in the session.
- No production deploys, production data changes, or irreversible migrations without an explicit go-ahead in the session.
- Never fabricate ticket IDs, file paths, API shapes, env values, or command output. Read it or ask.

### B7. Final report format
```
Ticket: <TICKET-ID> (Project: ...)
Branch: <TICKET-ID>/slug | PR: <link> | Base: <branch>
What changed: 2 to 4 lines, behavior first
Verified: <commands run> -> <results>; anything not run and why
Loops: fix rounds <n>/<budget>; escalations: <list or none>
Side tickets: <TICKET-ID> (PR link, or in progress with its worktree path and branch), or none
Open questions / follow-ups: ..., or none
```

### B8. Context and session hygiene
- One ticket per session. Do not carry a session across unrelated tickets.
- Keep `CLAUDE.md` files short. Long references live in their own files and get `@imported`.
- Prefer `explorer` subagents over loading large parts of the codebase into the main context.
- The orchestrator keeps a ledger and re-posts it at every phase change, after every verify round, before any long step, and in every escalation. It is the source of truth after a context reset: rebuild state from the latest ledger plus `git log`, `git status`, and the tracker, never from memory. `id` is the worker ID the tool returned, to resume it by (omit it where the tool cannot resume); a side ticket shows `fix round` only while the orchestrator runs its verify loop.

```
Ledger: <TICKET-ID> | branch <name> | base <branch> | phase <orient|plan|build|verify|ship|report>
Budgets: default | <overrides>
Units: U1 <agent> <status> <paths> id <worker-id> | U2 ...
Verify: fix round <n>/<budget> | reviewer <verdict> | gates <PASS|FAIL|INCOMPLETE> @ <short sha>
Findings open: R3 [major] path:line, R5 [minor] ... | closed: R1, R2 | disputed: R4 <reason>
Side tickets: <ID> <status> id <worker-id> [fix round <n>/<budget>], or none
Waiting on: <user|agent|nothing>: <what>
Next: <one action>
```

### B9. Standing agents
Defined in `~/.claude/agents/`. A repo's `.claude/agents/` file with the same `name` overrides the user-level one for that repo. `orchestrator` runs as the main thread (B0); the other five are its workers.

| Agent | Use for | Can edit | Spawns agents |
|---|---|---|---|
| `orchestrator` | Main thread only: the outer loop (B4), ticket, plan, dispatch, commits, finding routing, ship, report | No product code; git state, ticket and PR text only | All five workers |
| `explorer` | Read-only research: file map, snippets, findings, gates, conventions | No | No |
| `implementer` | One unit of work, in-scope files only; diff summary plus out-of-scope notes | Yes, in scope | No |
| `reviewer` | Fresh-context adversarial review of a diff vs ticket and standards; severity and file:line | No | No |
| `verifier` | Runs the repo's real gates; raw pass/fail with failure excerpts | No | No |
| `side-fix` | A B3 issue end to end on its own ticket, branch, worktree, and PR | Yes | `reviewer`, `verifier` only |
