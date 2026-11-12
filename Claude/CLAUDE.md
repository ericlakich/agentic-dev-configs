# Instructions for Claude

## Part A: Applies everywhere

### A1. No guessing
- Never fill a gap with a silent assumption. Missing fact, path, ID, value, or requirement: ask, in the batch below.
- Gather context first, then ask everything in one batch, early: every question and every permission the work will need through completion. Do not drip questions mid-task.
- Only ask when the answer changes what you produce. Otherwise proceed. A question I leave unanswered in the batch takes the default you proposed, which counts as my approval; a permission I leave unanswered is not granted.
- Development work, once that batch is answered: no more questions. A gap found later is decided and logged, or parked for the final report (B4, Autonomy).
- Separate what you verified from what you inferred, and say which is which.

### A2. Concise
- Lead with the answer or the status. No preamble, no restating my request, no narration of what you are about to do.
- Short replies by default. Detail on request.
- Name any choice you made on my behalf (ticket, project, area, approach) in one line so I can correct it cheaply.

### A3. Precedence
1. My explicit instruction in the current session.
2. A project workflow skill that covers this repo (per the repo's docs or the skill's own description).
3. This document.

Also follow the repo's engineering standards (B1 says where they live) and any org-level standards this file or the repo's docs point to. If two sources conflict and the order above does not settle it (standards vs skill, a repo `CLAUDE.md` vs this document), ask: in the question batch, or, in development work after the batch, park the conflict (B4, Autonomy). Do not pick silently.

## Part B: Development workflow

### B0. Orchestrator and workers
- The orchestrator owns the outer loop (B4) and every decision that crosses units: ticket, plan, dispatch, commits, routing findings, side issues, shipping, and all contact with me. It runs as the main session (`claude --agent orchestrator`); its definition is `~/.claude/agents/orchestrator.md`.
- Plain session (a main session not running as the orchestrator agent): read `~/.claude/agents/orchestrator.md` and follow its body for any code change.
- Workers are the subagents in B9. Each runs an inner loop (B4) on one bounded unit and returns a status plus evidence.
- Running as a worker: your agent definition sets your role and limits. Part A, B4, B5, B6, and the committed-secrets rule in B3 still apply; B5's commit rules apply to whoever commits. Do not create tickets, spawn agents, commit, or ship unless your definition says to.
- A worker cannot ask me directly. When A1 says "ask", stop and return `needs-input` with the question; the orchestrator decides or parks it (B4, Autonomy).

### B1. Session start
- If a project workflow skill covers this repo (A3), load it before touching code.
- Read the repo's instruction chain (`CLAUDE.md`, `AGENTS.md`) and its documented standards: `CONTRIBUTING.md`, `docs/` (standards, ADRs), PR template, `CODEOWNERS`, linter and formatter configs.
- Find the repo's real gates: CI config (`.github/workflows/`, `.gitlab-ci.yml`, or equivalent), `Makefile`, `package.json` scripts, `pyproject.toml`, language tooling, pre-commit config. Never invent a check command.
- Resolve the ticket (B2) before writing code.

### B2. Ticket, branch, PR
- Every code change needs a ticket in the repo's tracker (Linear, Jira, GitHub Issues, or whatever the repo uses), including one-line fixes.
- Identify the tracker and owning team from the workflow skill or repo docs. If undocumented, infer from recent branch names and PR titles, and confirm the inference in the A1 batch before creating anything. No tool access to that tracker: ask in the question batch. Never fabricate an ID.
- Search the owning team's tickets first. One clear match: use it and say which. Several plausible matches: ask which. No match: create one. Never ask permission to create.
- New tickets: outcome-oriented title; description covering the problem, the intended change, and how it will be verified; assigned to me; placed in the best-fit grouping (Linear project, Jira epic, GitHub milestone, or the tracker's equivalent). List the options, pick by scope match, and name the pick. If nothing fits, leave the grouping empty and say so.
- Branch: `<TICKET-ID>/<short-slug>`, lowercase kebab-case slug, 3 to 6 words. Example: `ABC-123/fix-login-redirect`. If the repo documents its own branch convention, it wins.
- GitHub Issues (numeric IDs): branch `issue-<n>/<slug>`, `#<n>` wherever a ticket ID appears in titles and reports, and `Closes #<n>` in the PR body.
- Base branch, PR title format, PR description template, and ticket state transitions: follow the workflow skill or repo docs exactly. If undocumented, infer from the last 10 merged PRs and say it is inferred. If still unclear, ask in the A1 batch and propose these defaults; use a default only once I approve it, and name it:
  - Base: the repo's default branch.
  - PR title: `<TICKET-ID>: <outcome-oriented summary>`.
  - PR description sections: Summary (behavior first), Ticket, Changes, Verification (commands -> results), Contract impact, Risk and rollout.
  - Ticket states: in-progress when work starts, review state when the PR opens. If the tracker has no such state, skip the move and say so.
- Open the PR, move the ticket to its review state, and link ticket and PR both ways. Merge only when I granted it in the question batch (a grant covers the whole stack) or ask later, and only once the verify exit test passed; otherwise stop at the open PR. No force-push or branch deletion unless I ask.
- Stacking: when a session opens more than one PR into the same base (the main PR and side fixes), stack them in one chain. Order, bottom-up: side-fix PRs (B3) in the order they open, then the main PR, which opens only after every side-fix has ended. Side PRs never ship as drafts; only the main PR can. The lowest PR targets the base. Each PR above it starts from, or merges in, the branch of the PR below it, and targets that branch. A PR that opens out of order is moved into place: the PR that belongs above it merges in its approved SHA, re-runs its gates (a fresh reviewer first after a conflict fix) and a fresh adversarial pass, pushes normally only on its `approve`, and retargets onto it. Each PR names the PR below it under "Risk and rollout", updated whenever its target changes.
- Merging a stack: bottom-up, with merge commits pinned to the approved head (`gh pr merge <n> --merge --match-head-commit <approved sha>`, or the host's equivalent pin; never `--delete-branch` unless I ask). After each merge, retarget the next PR to the base unless the host already did. If the repo allows only squash or rebase merges (Orient checks), merge nothing and leave the stack to me. B7 lists the chain in merge order.

### B3. Unrelated issues found mid-task
- Do not fold them into the current branch. One ticket, one PR.
- Create a ticket for the issue without asking (B2 rules), note which ticket you found it under, then spawn the `side-fix` subagent to handle it end to end in its own scaled-down loop (B4): its own `<TICKET-ID>/<slug>` branch from the correct base, fix, gates, PR. If it cannot spawn agents, it stops at ready for verify, and the orchestrator runs its verify loop and ships it (orchestrator Side loops). It runs in the background in a separate git worktree, so it never touches the main branch's working tree. Pass it the repo's B2 conventions in the brief.
- Keep the main task moving. If the issue is a hard prerequisite for the main change, it lands first: like every side PR it sits below the main PR (B2), and the main branch merges in its approved SHA once its PR is open, at the first point with no unit in flight and a tree that matches the Orient snapshot (orchestrator Phase 1). Units that need it, and the next verify round, wait for that merge. Record the ordering in both PRs under "Risk and rollout".
- Committed secrets: stop and flag immediately. That is a rotation, not a cleanup.
- Pre-existing exploitable security issue, whoever reports it (reviewer, adversarial, verifier, side-fix, implementer): on a public repo (any visibility other than private, internal included), no public ticket, branch, or PR; park it for the report with a private-advisory recommendation. On a private repo, handle it as any other B3 issue, with ticket, branch, PR, and commit text naming it by ID, severity, and `path:line` only.
- Report every side ticket and PR in the final summary (B7).

### B4. Agentic loops
A loop gathers context, acts, checks against ground truth, and decides, until an exit test passes, a budget runs out, progress stalls, or input is needed. Ground truth is outside the model: command output, a test result, file content, a verdict from an agent that did not write the code. Never your own belief that you are done.
- **Contract.** Every loop, outer or inner, has a goal (one sentence), an exit test (the observable condition that means done), evidence (the ground truth checked on every pass), a budget (max passes), a stall rule, and an exit state.
- **Exit states.** The first line of every worker report is `Status: <state> after <n> passes: <one-line reason>`. A report without that line counts as `partial`.
  - `done`: exit test passed; evidence attached.
  - `partial`: stopped on budget or stall; say which, what passed, and what remains.
  - `blocked`: needs a change outside your scope or authority (other files, design, environment, permission).
  - `needs-input`: a question only I can answer (A1). Workers return it; the orchestrator handles it under Autonomy.
- **Default budgets.** A brief or I may override them; the orchestrator names any override in the ledger (B8).
  - explorer: at most 3 widening rounds per question.
  - implementer: at most 3 fix attempts on any one failing check.
  - Verify loop (the orchestrator's, side-fix's own, or the one the orchestrator runs for side-fix): at most 3 fix rounds after the first verify. The count does not reset after a re-plan.
  - Adversarial loop (the last stage of every verify loop: the orchestrator's, side-fix's own, or the one the orchestrator runs for side-fix): at most 3 rounds. From its first pass on, every round counts here, not against the verify loop: one set of committed fixes (`R` or `S`) with the checks that follow it, or one adversarial re-pass with no fixes before it. A conflict fix and the post-merge pass after it are one round; a post-merge pass on a clean merge counts only if it requests changes; the one scoped re-brief after a `partial` adversarial is not a round. The count does not reset after a re-plan. A side fix's own approve never ships it: it stops at ready to ship, and its PR opens only on the orchestrator's own adversarial `approve` at that SHA. Stall: an `S` finding still open after its fix, or the same one disputed or justified twice and rejected both times. On budget or stall, the main PR ships as a draft and is never marked ready or merged (a stack merge's conflict fix still runs, so it sits on top); a side fix is parked with no PR, or, after a restack, its PR stays at its last approved head.
  - Re-brief of the same worker for the same goal after `partial`: once (narrow the goal or split the unit), then park the unit (Autonomy). A fresh spawn after a stall counts as that one re-brief; never chain fresh spawns.
  - Re-plan: once per ticket. A second re-plan, or any re-plan that changes scope or a public contract, is parked (Autonomy).
- **Stall rule.** Stop when the same check fails the same way after two different fixes, or when a pass adds no new evidence (no new finding, no diff change, no new information). Never repeat an identical action expecting a different result. In the verify loop the stricter Phase 4 rule wins: a finding still open after its fix, or a gate failing the same way after its fix, stops it.
- **Flaky.** A check that changes result with no code change is flaky. Re-run it once to confirm, then report it as flaky. Never loop on it, and never count a flaky pass as a pass.
- **Autonomy.** The orchestrator asks me things once: the question batch in Plan, after Orient has gathered the context. The batch holds every question, every permission the work will need through completion (merge, which covers the whole stack; other B2 and B6 gated actions; new dependencies; anything outward-facing beyond opening PRs), and the forks it expects mid-build, each with the option it will take unless I say otherwise. Once I answer, it runs to the report without asking. A gap found later, including a worker's `needs-input`:
  - Safe option exists (reversible, inside the approved scope, no contract or security change): take the safest one, log it on the ledger's `Decisions` line as `self`, and continue.
  - No safe option (irreversible, outward-facing beyond what I approved, a scope or public-contract change, a security call, a standards conflict (A3), a second re-plan), or a loop that ends on budget or stall: park the affected units and anything that depends on them, finish everything else, and raise the question in the report with options and a recommendation. When parking keeps the main PR from its exit test, ship it as a draft with the parked items under "Risk and rollout".
  - A gated action (B2, B6) the batch did not grant is never taken. Park it.
  - The one interrupt: a committed secret (B3). Tell me at once, push nothing until I answer, and keep going on units that do not touch it.
- **Resume vs fresh.** Resume the same worker (it keeps its context) for its own unit: a fix pass, a narrowed goal, integration feedback. Spawn fresh when independence matters (every reviewer, verifier, or adversarial run) or when the worker stalled (its context is polluted; that spawn is the one re-brief). Never spawn a fresh side-fix while its worktree exists. If a worker cannot be resumed, re-spawn it with its prior report and the findings in the brief.
- **Outer loop.** The orchestrator runs these phases; `~/.claude/agents/orchestrator.md` holds each phase's exit test, decision rules, and back-edges.
  0. Intake: not a code change -> answer under Part A; a PR review -> the `pr-review` skill; a code change -> the loop.
  1. Orient: gather context before asking anything. The orchestrator searches the tracker (B2) and loads the workflow skill; `explorer`s map the code, standards, and real gates (B1).
  2. Plan: units with disjoint file scopes, each with an exit test and gates; the one question batch (Autonomy); then the ticket resolved (B2) and its branch. From here on, no questions until the report; a re-plan skips the batch (Autonomy).
  3. Build: one `implementer` per unit; the orchestrator reads each diff and commits it.
  4. Verify: fresh `reviewer`s and one `verifier` each round; findings return as fix briefs. Once they pass, a fresh `adversarial` attacks the change; its findings return as fix briefs too, and Verify ends only when it approves.
  5. Ship: per B2, stacked when there is more than one PR; merged only if granted.
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

### B7. Final report
The session's last message, written to be scanned. Use these sections in this order. Write bullets, one point each, the point first. Keep each section under 250 words where possible. Link to the PR, ticket, or file for detail; never paste logs, worker reports, or the ledger. Skip a section with nothing in it, except Open questions.

```
## <TICKET-ID>: <one-line outcome>
Project: ... | Branch: <branch> | PR: <link> | Base: <branch the stack merges into> | Status: ready for review, merged, or draft (why)

### What changed
- 2 to 4 bullets, behavior first

### Verification
- <command> -> <result>; anything not run and why
- Fix rounds <n>/<budget>
- Adversarial <approve|request changes|pending> @ <PR head short sha>; rounds <n>/<budget>

### Merge order (two or more PRs)
| # | PR | Branch | Targets | Depends on | Status |
|---|---|---|---|---|---|
| 1 | <link>: <title> | <branch> | <base> | none | ready, merged, or draft (why) |
| 2 | <link>: <title> | <branch> | <branch of 1> | 1 | ... |

Merge in row order with merge commits. After each merge, retarget the next PR to <base>. In a squash- or rebase-only repo, first merge the updated base into each next PR's branch.

### Findings
- Review, adversarial, and workflow-control findings fixed, justified, or disputed: one line each, with the reason
- Pre-existing problems found, with their side tickets (a pre-existing security issue on a public repo: parked, with a private-advisory recommendation, B3)

### Decisions made for you
- Each `self` decision: the gap, the option taken, why it was safe
- Permission rules added to the local settings file during the run, verbatim (orchestrator Phase 4)

### Recommendations
- What to do next or differently, most important first

### Follow-ups
- Work left: parked side tickets (worktree path and branch), parked units, cleanup such as worktrees to remove

### Open questions
- Each parked question with its options and a recommendation, or none
```

### B8. Context and session hygiene
- One ticket per session. Do not carry a session across unrelated tickets.
- Keep `CLAUDE.md` files short. Long references live in their own files and get `@imported`.
- Prefer `explorer` subagents over loading large parts of the codebase into the main context.
- The orchestrator keeps a ledger and re-posts it at every phase change except into the report, after every verify round, and before any long step. It is the source of truth after a context reset: rebuild state from the latest ledger plus `git log`, `git status`, and the tracker, never from memory. `id` is the worker ID the tool returned, to resume it by (omit it where the tool cannot resume); `Adversarial` holds the last adversarial verdict (`pending` before its first pass), its rounds, and the full SHA it attacked; `W` findings are workflow-control hits, and `Baseline` lists the files those checks compare against (orchestrator Phases 1 and 4); a side ticket shows `fix round` and `adversarial round` only while the orchestrator runs its verify loop. `PRs` lists the session's PRs bottom-up, one stack per base (B2). `Decisions` records my batch answers and permissions (`me`) and each decision made for me after the batch (`self`, with the reason). `Parked` holds the questions for the report (B4, Autonomy).

```
Ledger: <TICKET-ID> | branch <name> | base <branch> | phase <orient|plan|build|verify|ship|report>
Budgets: default | <overrides>
Units: U1 <agent> <model> <status> <paths> id <worker-id> | U2 ...
Verify: fix round <n>/<budget> | reviewer <verdict> | gates <PASS|FAIL|INCOMPLETE> @ <short sha>
Adversarial: round <n>/<budget> | verdict <approve|request changes|pending> @ <full sha>
Findings open: R3 [major] path:line, S2 [blocker] path:line, W1 path, R5 [minor] ... | closed: R1, R2, S1 | disputed: R4 <reason> | justified: R6 <reason>
Baseline: <path> <blob sha, link target, or nested HEAD>, ... | worktrees <paths> | snapshot <git status lines> | user config <path> <blob sha>, ... | git config and hooks <path> <blob sha>, ... [main checkout: ...], or none
Side tickets: <ID> <status> id <worker-id> worktree <path> [fix round <n>/<budget>, adversarial round <n>/<budget>], or none
PRs: 1 <link> <branch> -> <base> @ <approved full sha> <open|draft: why|merged> | 2 <link> <branch> -> <branch of 1> @ <approved full sha> ..., or none
Decisions: <what> -> <answer> (me | self: <reason>), or none
Parked: <unit or item>: <question> | options: <a>, <b> | rec: <a>, or none
Waiting on: <user|agent|nothing>: <what>
Next: <one action>
```

### B9. Standing agents
Defined in `~/.claude/agents/`. A repo's `.claude/agents/` file with the same `name` overrides the user-level one for that repo. `orchestrator` runs as the main thread (B0); the other six are its workers.

| Agent | Use for | Can edit | Spawns agents |
|---|---|---|---|
| `orchestrator` | Main thread only: the outer loop (B4), ticket, plan, dispatch, commits, finding routing, ship, report | No product code; git state, ticket and PR text only | All six workers |
| `explorer` | Read-only research: file map, snippets, findings, gates, conventions | No | No |
| `implementer` | One unit of work, in-scope files only; diff summary plus out-of-scope notes | Yes, in scope | No |
| `reviewer` | Fresh-context adversarial review of a diff vs ticket and standards; severity and file:line | No | No |
| `verifier` | Runs the repo's real gates; raw pass/fail with failure excerpts | No | No |
| `adversarial` | Last pass of every verify loop: attacks the change for security loopholes and exploitable issues; verdict, severity, exploit path, file:line | No (throwaway PoCs outside the repo only) | No |
| `side-fix` | A B3 issue end to end on its own ticket, branch, worktree, and PR | Yes | `reviewer`, `verifier`, `adversarial` only |

Model choice. Where the harness lets an agent choose a worker's model at spawn time (the orchestrator, and side-fix for its reviewer, verifier, and adversarial), choose, for each spawn, the most cost-efficient model that can still meet the unit's exit test. Weigh how hard the unit is against how costly a miss would be:
- A cheaper tier for bounded, checkable work: a narrow explorer lookup, a verifier run, a mechanical implementer unit with strong gates.
- The session's model, or the strongest available, where a miss is costly or hard to catch: every reviewer and adversarial run, design-heavy or cross-cutting implementer units, and side-fix.
- Cost never buys a skipped check: every result still meets its exit test and the verify loop. After a `partial` or stall that looks like a capability limit, the one re-brief may move up a tier, never down.
- A model I name in the session, or one a repo's agent override sets, wins. Where the harness cannot choose per spawn, each agent runs on its configured model.
- Record each choice on the ledger's `Units` line.
