# Instructions for Codex

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

Also follow the repo's engineering standards (B1 says where they live) and any org-level standards this file or the repo's docs point to. If two sources conflict and the order above does not settle it (standards vs skill, a repo `AGENTS.md` vs this document), stop and ask. Do not pick silently.

## Part B: Development workflow

### B0. Main session vs subagent
- The main session is the orchestrator. Ticket creation, spawning subagents, committing, and shipping (B2 to B4) are its duties.
- Running as a subagent: your agent definition sets your role and limits. Part A, B5, B6, and the committed-secrets rule in B3 still apply; B5's commit rules apply to whoever commits. Do not create tickets, spawn agents, commit, or ship unless your definition says to.
- A subagent cannot ask me directly. When A1 says "ask", stop and return the question to the orchestrator as an open question.

### B1. Session start
- If a project workflow skill covers this repo (A3), load it before touching code.
- Read the repo's instruction chain (`AGENTS.override.md`, `AGENTS.md`, `CLAUDE.md`) and its documented standards: `CONTRIBUTING.md`, `docs/` (standards, ADRs), PR template, `CODEOWNERS`, linter and formatter configs.
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
- Create a ticket for the issue without asking (B2 rules), note which ticket you found it under, then spawn the `side-fix` subagent to handle it: its own `<TICKET-ID>/<slug>` branch from the correct base, fix, gates. Codex has no per-agent worktree, so before spawning, create a linked worktree inside the repo and pass its absolute path, the remote, the base, and the base SHA in the brief:
  ```
  cd "$(git rev-parse --show-toplevel)"
  git fetch <remote> <base>
  git worktree add --no-track .worktrees/<TICKET-ID> -b <TICKET-ID>/<slug> <remote>/<base>
  git -C .worktrees/<TICKET-ID> rev-parse --show-toplevel HEAD
  ```
  `<remote>` is the remote that hosts the base (usually `origin`). The last command prints the worktree's absolute path and its HEAD, which is the base SHA, for the brief. Run the block as one shell command, because some tools start a fresh shell for each call. `--no-track` keeps the new branch from tracking the base, so a later push cannot land on the base branch. If `git check-ignore -q .worktrees/` fails, first run `echo .worktrees/ >> "$(git rev-parse --git-path info/exclude)"` (local only; works when the main checkout is itself a linked worktree; never edit the repo's `.gitignore` for this). For GitHub Issues the ID part is `issue-<n>` (branch `issue-<n>/<slug>`, worktree `.worktrees/issue-<n>`). The worktree stays until I remove it after merge (`git worktree remove .worktrees/<TICKET-ID>`). `side-fix` runs in the background in that worktree, so it never touches the main branch's working tree. Spawning does not block; collect its result when it returns. Pass it the repo's B2 conventions in the brief.
  `side-fix` cannot spawn agents. When it returns `ready for verify`, run `reviewer` and `verifier` against its worktree, then push with `git -C <worktree> push -u <remote> HEAD`, open the PR, move the ticket to its review state, and link ticket and PR both ways. If they find problems, re-spawn `side-fix` in the same worktree with the findings, then re-verify.
- Keep the main task moving. If the issue is a hard prerequisite for the main change, it lands first: open its PR first, ask me to merge it or stack the main branch on it, and record the ordering constraint in both PRs under "Risk and rollout".
- Committed secrets: stop and flag immediately. That is a rotation, not a cleanup.
- Report every side ticket and PR in the final summary (B7).

### B4. Workflow: orient, plan, build, verify, ship, report
- **Orient.** Use `explorer` subagents for codebase research. Keep the main session's context for coordination.
- **Plan.** For anything beyond a trivial fix (one file, obvious change, no contract impact), write a short plan: scope, files, approach, tests, risks, open questions. Ask the open questions in one batch. Do not start building with a material question unresolved.
- **Build.** Decompose into independent units, one `implementer` per unit, each with a precise brief: goal, files in scope, constraints, gate commands, definition of done, what to report back, and which other units are in flight. Run independent units in parallel only when their file scopes do not overlap; serialize the rest. Subagents do not expand scope; they report what they notice. The main session reviews each unit's diff and commits it.
- **Verify.** Mandatory and separate from build; starts after all build units finish. Spawn `reviewer` subagents with fresh context that did not write the code (fresh context: see B9). Give them the diff, the ticket, and the standards. They check correctness, standards conformance, test coverage, edge cases, security, and contract impact. Spawn a `verifier` to run the repo's real gates: build, tests, lint, type check. Fix findings, then re-verify.
- **Ship.** Per B2.
- **Report.** Per B7.

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
Side tickets: <TICKET-ID> (PR link), or none
Open questions / follow-ups: ..., or none
```

### B8. Context and session hygiene
- One ticket per session. Do not carry a session across unrelated tickets.
- Before any long step, write a short state block (ticket, branch, done, remaining, blockers) so the work survives a context reset.
- Keep `AGENTS.md` files short. Repo `AGENTS.md` files share a 32 KiB budget (`project_doc_max_bytes`) and anything past it is truncated. Long references live in their own files that `AGENTS.md` points to by path.
- Prefer `explorer` subagents over loading large parts of the codebase into the main context.

### B9. Standing subagents
Defined in `~/.codex/agents/*.toml`, one file per role. A repo's `.codex/agents/` role with the same `name` replaces the user-level one for that repo (trusted projects; project config outranks user config, and missing fields are merged from the user role).

- Subagents share the parent's working directory, sandbox, and approvals, and spawning does not block: the parent collects each result when it returns.
- Under multi-agent V2 (the default for current models), spawn every role with `fork_turns: "none"` so it starts with fresh context; otherwise it inherits this session's history.
- Under V1, `agents.max_depth = 1` stops subagents from spawning. V2 allows it. Either way, only the orchestrator spawns (B0).

| Agent | Use for | Can edit | Spawns agents |
|---|---|---|---|
| `explorer` | Read-only research: file map, snippets, findings, gates, conventions | No | No |
| `implementer` | One unit of work, in-scope files only; diff summary plus out-of-scope notes | Yes, in scope | No |
| `reviewer` | Fresh-context adversarial review of a diff vs ticket and standards; severity and file:line | No | No |
| `verifier` | Runs the repo's real gates; raw pass/fail with failure excerpts | No | No |
| `side-fix` | A B3 issue end to end on its own ticket, branch, worktree, and PR | Yes | No (orchestrator verifies and ships) |
