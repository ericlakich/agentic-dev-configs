# Gemini CLI

User-level configuration for [Gemini CLI](https://geminicli.com/docs/). Everything here installs into `~/.gemini/`.

Ported from `../Claude/`. Same workflow, rules, and five agents, in Gemini CLI's native format.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `GEMINI.md` | `~/.gemini/GEMINI.md` | Global instructions loaded into every session |
| `agents/explorer.md` | `~/.gemini/agents/explorer.md` | Read-only codebase research |
| `agents/implementer.md` | `~/.gemini/agents/implementer.md` | Builds one scoped unit of work |
| `agents/reviewer.md` | `~/.gemini/agents/reviewer.md` | Fresh-context adversarial diff review |
| `agents/verifier.md` | `~/.gemini/agents/verifier.md` | Runs the repo's real gates, reports pass/fail |
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
| Nested subagents | side-fix spawns `reviewer` and `verifier` | Not possible; side-fix stops at `ready for verify`, the orchestrator verifies and ships |
| Model selection | `model: inherit` | `model` omitted, which defaults to `inherit` |

Per-agent frontmatter. "Read set" is `read_file`, `read_many_files`, `grep_search`, `glob`, `list_directory`.

| Agent | `tools` | `max_turns` | `timeout_mins` |
|---|---|---|---|
| `explorer` | read set, `run_shell_command`, `web_fetch`, `google_web_search` | default (30) | default (10) |
| `implementer` | read set, `run_shell_command`, `write_file`, `replace`, `web_fetch`, `google_web_search`, `write_todos`, `activate_skill`, `mcp_*` | 60 | 30 |
| `reviewer` | read set, `run_shell_command`, `web_fetch` | default (30) | default (10) |
| `verifier` | read set, `run_shell_command` | default (30) | 30 |
| `side-fix` | same as `implementer` | 100 | 60 |

The defaults are 30 turns and 10 minutes. Full test suites and whole tickets outrun them, so implementer, verifier, and side-fix get more. The implementer and side-fix lists are explicit so they exclude `ask_user`: subagents return questions to the orchestrator (B0). Both keep `activate_skill` and `mcp_*`, so they can load a workflow skill and reach the tracker and git host, as in Claude.

Back-to-back subagent calls in one turn run in parallel, so B4's parallel build units work as in Claude. [scheduler.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/scheduler/scheduler.ts)

## Known gaps

- Strict frontmatter schema. Any unknown key makes an agent fail to load, so Claude agent files (`disallowedTools`, `isolation`, `background`) cannot be copied as-is. [Subagents](https://geminicli.com/docs/core/subagents/)
- No nesting. Subagents cannot call other subagents, even with `*`. The orchestrator runs `reviewer` and `verifier` on side-fix's worktree, then ships. [Subagents](https://geminicli.com/docs/core/subagents/)
- No background mode. side-fix pauses the main task until it returns. [Subagents](https://geminicli.com/docs/core/subagents/)
- No per-agent worktree. B3 has the orchestrator create one under `.worktrees/`. Session-level worktrees exist (`gemini --worktree <name>` with `experimental.worktrees: true`, default `false`, created under `.gemini/worktrees/`) but cover a whole session, not one agent. [Subagents](https://geminicli.com/docs/core/subagents/)
- File tools skip git-ignored paths (`context.fileFiltering.respectGitIgnore`, default `true`, which also honors `.git/info/exclude`). `read_file` rejects them; `glob`, `read_many_files`, and `list_directory` omit them. `.worktrees/` is excluded, so B3 and side-fix read there with `run_shell_command`. (`pr-review` never checks out the PR, so it reads everything through git.) `write_file` and `replace` only check workspace access, so edits work. Alternative: set `"context": {"fileFiltering": {"respectGitIgnore": false}}` in `~/.gemini/settings.json`; file tools then also see other ignored files, such as `node_modules`. [read-file.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/tools/read-file.ts), [fileDiscoveryService.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/services/fileDiscoveryService.ts)
- Gates that don't read git excludes (some lint and test globs) can pick up files under `.worktrees/` while a side-fix worktree exists.
- Read-only is prompt-enforced. explorer, reviewer, and verifier have no edit tools but keep `run_shell_command`, which can write; verifier needs it for builds and tests. For a hard limit, add policy TOML in `~/.gemini/policies/` with `subagent = "<name>"`. None ships here because priority semantics are unverified. [Policy engine](https://github.com/google-gemini/gemini-cli/blob/main/docs/reference/policy-engine.md)

## Per-repo overrides

- A repo's `GEMINI.md` files (workspace dirs and their parents up to `.git`, plus files loaded just in time when a tool touches a directory) are concatenated with this one. If the two conflict and the A3 order doesn't settle it, the agent stops and asks. [GEMINI.md](https://geminicli.com/docs/cli/gemini-md/)
- `CLAUDE.md` is not read by default; B1 tells the agent to read `AGENTS.md` and `CLAUDE.md` itself. To load other names automatically, set `context.fileName` in `~/.gemini/settings.json`, for example `["AGENTS.md","GEMINI.md"]`. Each name is also looked up in `~/.gemini/`.
- A repo's `.gemini/agents/<name>.md` with the same `name` takes precedence over the user-level agent (trusted folders only, after you acknowledge the agent). [registry.ts](https://github.com/google-gemini/gemini-cli/blob/249d51c/packages/core/src/agents/registry.ts)
- `agents.overrides.<name>` in settings can disable or tune one agent.
