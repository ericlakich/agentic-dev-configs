# Codex

User-level configuration for [OpenAI Codex](https://learn.chatgpt.com/docs) ([source](https://github.com/openai/codex)). Everything here installs into `~/.codex/` (or `$CODEX_HOME` if you set it). Ported from `../Claude/`.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `AGENTS.md` | `~/.codex/AGENTS.md` | Global instructions loaded into every session; Part C is the orchestrator for the main session |
| `agents/explorer.toml` | `~/.codex/agents/explorer.toml` | Read-only codebase research (replaces Codex's built-in `explorer` role) |
| `agents/implementer.toml` | `~/.codex/agents/implementer.toml` | Builds one scoped unit of work |
| `agents/reviewer.toml` | `~/.codex/agents/reviewer.toml` | Fresh-context adversarial diff review |
| `agents/verifier.toml` | `~/.codex/agents/verifier.toml` | Runs the repo's real gates, reports pass/fail |
| `agents/side-fix.toml` | `~/.codex/agents/side-fix.toml` | Takes an unrelated issue to a ready-for-verify branch in a worktree the orchestrator creates |
| `skills/pr-review/SKILL.md` | `~/.codex/skills/pr-review/` | Reviews a GitHub PR with the `reviewer` subagent and reports findings |

## Install

Run from the repo root:

```bash
CODEX_HOME="${CODEX_HOME:-$HOME/.codex}"
mkdir -p "$CODEX_HOME/agents" "$CODEX_HOME/skills"
if [ -e "$CODEX_HOME/AGENTS.md" ] && [ ! -L "$CODEX_HOME/AGENTS.md" ]; then mv "$CODEX_HOME/AGENTS.md" "$CODEX_HOME/AGENTS.md.bak"; fi
ln -sn "$PWD/Codex/AGENTS.md" "$CODEX_HOME/AGENTS.md"
cp Codex/agents/*.toml "$CODEX_HOME/agents/"
ln -sn "$PWD/Codex/skills/pr-review" "$CODEX_HOME/skills/pr-review"
```

The `if` moves an existing real `AGENTS.md` to `AGENTS.md.bak`. It leaves an existing symlink in place, so `ln -sn` then refuses with "File exists" instead of doing something silent. Plain `ln -s` is risky here: if the target is a directory, or a symlink to one, it exits 0 and creates the link *inside* it. On a re-run that means a self-referencing link inside this repo. If `$CODEX_HOME/skills/pr-review` already exists as a real folder, move it out of `$CODEX_HOME/skills/` first, because `ln -sn` would also create the link inside it.

The role files are copied, not linked. Codex opens role files without following symlinks and rejects a linked one with "agent type is currently not available". Re-copy after each `git pull`. `cp` replaces same-named files in `$CODEX_HOME/agents/` and leaves your other roles alone. Linking the whole `agents` directory (not individual files) should work, because `O_NOFOLLOW` only checks the last path component, but it is untested in Codex.

The skill is linked one folder at a time, because the skills directory may hold other skills. It goes in `$CODEX_HOME/skills/`, not `~/.agents/skills/`. Codex's newer user path is shared with Gemini, Copilot, OpenCode, and Cursor, and Gemini prefers it over its own `~/.gemini/skills/`, so the Codex variant would load in other tools. The trade-off: `$CODEX_HOME/skills` is deprecated but still supported ([`host_roots.rs`][host-roots]). Cursor also reads `~/.codex/skills/`.

Unverified: whether Codex loads a symlinked skill folder. If `pr-review` does not show up, replace the link with a copy: `rm "$CODEX_HOME/skills/pr-review" && cp -R Codex/skills/pr-review "$CODEX_HOME/skills/"`.

If `$CODEX_HOME/AGENTS.override.md` exists and is non-empty, Codex loads it instead of `AGENTS.md`. Move it aside.

No setting is needed. Multi-agent is on by default.

## Run the orchestrator

Codex has no main-session role: a role applies only to a spawned or resumed session ([`child_config.rs`][child-config], [`role.rs`][role]). So the orchestrator is not a role file. It is Part C of `AGENTS.md`, and B0 makes every main session the orchestrator for a code change. Start a normal session:

```bash
codex
```

`~/.codex/AGENTS.md` also reaches subagents, so every worker loads Part C. B0 tells workers to ignore it, and Part C opens by saying it applies only to the main session. Both are instructions only.

Not shipped: `codex --profile orchestrator`, with a profile that sets root `developer_instructions` to the Part C text. Root `developer_instructions` adds a developer message, and children inherit it unless their role sets its own ([`child_config.rs`][child-config]). All five roles here set their own, so workers would not get it. This port keeps Part C in `AGENTS.md` instead, so every main session follows it without a flag. Untested.

## How it works

`AGENTS.md` has three parts:

- **Part A (everywhere):** no guessing (ask in one early batch), concise replies, and a precedence order: your instruction in the session, then a project workflow skill, then this file.
- **Part B (development):** every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The work runs as agentic loops (B4): gather context, act, check against ground truth, decide. A loop repeats until its exit test passes, its budget runs out, progress stalls, or it needs your input. Ground truth is outside the model: command output, test results, file content, or a verdict from an agent that did not write the code.
- **Part C (main session only):** the orchestrator's procedure: operating rules, the outer loop phase by phase, the unit brief template, the ledger, escalation, and side loops.

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

side issue  side-fix          own ticket, branch, and worktree; scaled-down loop that stops at ready for verify
            orchestrator      runs side-fix's verify loop (3 fix rounds), then pushes and opens its PR
```

Every worker report starts with `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`. Workers never ask you anything; they return `needs-input`, and the orchestrator batches the questions. The orchestrator keeps a ledger (B8) of units, verify rounds, open findings, and side tickets. It re-posts the ledger at every phase change and rebuilds from it after a context reset.

## Differences from Claude

| Feature | Claude | Codex |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md` | `~/.codex/AGENTS.md`; a non-empty `AGENTS.override.md` there wins |
| Agent location | `~/.claude/agents/`, repo `.claude/agents/` | `~/.codex/agents/**/*.toml`, repo `.codex/agents/` |
| Agent format | Markdown with YAML frontmatter | TOML: `name`, `description`, `developer_instructions` |
| Main-thread orchestrator | `agents/orchestrator.md`: `claude --agent orchestrator` or the `agent` setting | No main-session role; Part C of `AGENTS.md`, which every main session follows (B0) |
| Spawning | Agent tool | `spawn_agent` with `agent_type="<role>"` |
| Spawn allowlist | Prompt-enforced; optional `tools: Agent(...)` on the orchestrator | Instructions only. Nothing limits which role `spawn_agent` takes; the built-in `default`, `worker`, and `explorer` always exist, and an omitted `agent_type` means `default` |
| Resume a worker | `SendMessage` to its ID or name | V2: `followup_task` (new task, starts a turn) or `send_message` (queued, no turn). V1: `send_input`, and `resume_agent` for a closed agent |
| Tool restriction | `tools`, `disallowedTools` | None per role |
| Read-only enforcement | Instructions; `permissions.deny` for a hard guarantee | Instructions only; roles inherit the parent's sandbox and approval policy |
| Background | `background: true` on side-fix | Every subagent runs alongside the parent; spawning returns immediately |
| Worktree isolation | `isolation: worktree` on side-fix | None per agent; the orchestrator creates `.worktrees/<TICKET-ID>` for side-fix (B3) |
| Nested subagents | side-fix spawns `reviewer` and `verifier` | V1: off (`agents.max_depth = 1`). V2 (current models): allowed. Either way the orchestrator runs side-fix's verify loop and ships its branch |
| Model selection | `model: inherit` | No `model` set in these files; roles accept `model` and `model_reasoning_effort`, and `config.toml` has `agents.default_subagent_model` |

Not used by this port: hooks (`~/.codex/hooks.json` or `[hooks]` in `config.toml`) and custom prompts (`~/.codex/prompts`), which Codex removed.

## Known gaps

Source links point at openai/codex commit `11da6b9` (2026-10-07).

- No tool restriction per role. `explorer`, `reviewer`, and `verifier` are read-only by instruction only. Turning off `shell_tool` under `[features]` would also block `rg` and `git`, so these roles leave it on. ([source][src])
- Role files ignore `sandbox_mode`, `approval_policy`, and `mcp_servers`, even though the docs list them. Children inherit the parent's sandbox, approval policy, working directory, and permissions. These files set none of those keys. ([source][src])
- In the default workspace-write sandbox, `.git` is read-only ([`permissions.rs`][permissions]). B3's fetch, `worktree add`, and the exclude append need approval, and so does every side-fix commit. With `approval_policy = "never"`, side-fix cannot commit.
- No per-agent worktree. Worktrees exist only per session: the Codex app keeps managed worktrees under `$CODEX_HOME/worktrees`, and the CLI `--worktree` flag starts a session in a new one. B3 has the orchestrator create the side-fix worktree instead. ([docs][docs])
- Gates that don't read git excludes (some lint and test globs) can pick up files under `.worktrees/` while a side-fix worktree exists.
- Multi-agent V2 is what current models use. The bundled model catalog marks gpt-6.1-sol, gpt-6-sol, gpt-6-astra, gpt-6-luna, gpt-5.6-sol, and gpt-5.6-terra as `multi_agent_version: v2`, even when `features.multi_agent_v2` is off. Under V2, `spawn_agent` defaults `fork_turns` to `all`, so a child inherits the parent's whole history; B9 has the orchestrator pass `fork_turns: "none"`. V2 ignores `max_depth` and gives children the spawn tools, so "only the orchestrator spawns" is enforced by instructions only. ([`config/mod.rs`][config-mod], [`models.json`][models], [`spawn.rs`][spawn])
- Under V1 the default depth is 1, so side-fix cannot spawn `reviewer` or `verifier`. You can set `[agents] max_depth = 2` in `~/.codex/config.toml` to let it. This port does not rely on nesting under either version: side-fix returns ready for verify, and the orchestrator runs its verify loop, then ships. ([source][src])
- Concurrency is capped: V1 `agents.max_concurrent_threads_per_session` defaults to 6. V2 caps it at 4 threads, counting the root agent. ([source][src])
- Codex ships a Claude Code importer ([`codex-rs/external-agent-migration/`][importer]). For subagents it keeps only `name`, `description`, and the body, drops `tools`, `model`, `isolation`, and `background`, and rewrites "Claude"/"CLAUDE.md" to "Codex"/"AGENTS.md". An import of `../Claude/` would keep side-fix's own-worktree and spawning steps, which this port replaces. By the same rules it would likely bring `orchestrator` in as a spawnable role, which this port replaces with Part C (unverified). Install these files instead. Unverified: how the importer is triggered.
- No main-session role. Codex applies a role only to a spawned or resumed session, so the orchestrator is Part C of `AGENTS.md`, not a role file. ([`child_config.rs`][child-config], [`role.rs`][role])
- Workers see Part C. `~/.codex/AGENTS.md` reaches subagents, so every worker loads it; B0 and Part C's first line tell workers to ignore Part C. Instructions only.
- The spawn limit is instructions only. `spawn_agent` takes any defined role, and the built-in roles `default`, `worker`, and `explorer` always exist. A role with the same name shadows a built-in one, which is how `explorer.toml` replaces the built-in `explorer`. An omitted `agent_type` means `default`, so Part C spawns every worker by its `agent_type` and never omits it. ([`role.rs`][role], [`multi_agents_spec.rs`][ma-spec])
- Resume tools differ by version. V2 has `followup_task` (gives an agent a new task and starts a turn) and `send_message` (queues a message without a turn). V1 has `send_input` and `resume_agent`, which reopens a closed agent so it can take `send_input`. B4 and Part C resume with `followup_task` or `send_input`. ([`multi_agents_spec.rs`][ma-spec], [`spec_plan.rs`][spec-plan])
- Loop budgets (B4) and the orchestrator's "no product-code edits" rule are enforced by instructions only.
- Unverified: whether the Codex desktop app loads `~/.codex/agents/` exactly like the CLI. ([docs][docs])

[docs]: https://learn.chatgpt.com/docs
[src]: https://github.com/openai/codex
[importer]: https://github.com/openai/codex/tree/main/codex-rs/external-agent-migration
[config-mod]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/core/src/config/mod.rs
[models]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/models-manager/models.json
[spawn]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs
[permissions]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/protocol/src/permissions.rs
[host-roots]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/ext/skills/src/host_roots.rs
[layers]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/config/src/config_layer_source.rs
[loader]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/agent-roles/src/loader.rs
[child-config]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/core/src/agent/child_config.rs
[role]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/core/src/agent/role.rs
[ma-spec]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/core/src/tools/handlers/multi_agents_spec.rs
[spec-plan]: https://github.com/openai/codex/blob/11da6b9edcbbf1cffdb9b7686970b5b4d5af9c83/codex-rs/core/src/tools/spec_plan.rs

## Per-repo overrides

- A repo's `AGENTS.md` adds project rules on top of this one. Codex walks from the repo root (marked by `.git`) down to the working directory and, in each directory, takes the first of `AGENTS.override.md`, `AGENTS.md`, or a name in `project_doc_fallback_filenames` (default empty). If the repo rules and this file conflict and the A3 order doesn't settle it, the agent stops and asks.
- All repo instruction files share a 32 KiB budget (`project_doc_max_bytes`). The file that crosses it is truncated. Repo files are skipped in untrusted projects.
- A repo `CLAUDE.md` is not loaded unless it is listed in `project_doc_fallback_filenames`. B1 still tells the agent to read it as part of the repo's instruction chain.
- A role in a repo's `.codex/agents/` with the same `name` replaces the user-level one for that repo. This applies in trusted projects: project config outranks user config, and fields the repo role leaves out are merged from the user role. ([`config_layer_source.rs`][layers], [`loader.rs`][loader])
