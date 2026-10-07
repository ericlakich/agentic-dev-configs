# Cursor

User-level configuration for Cursor, the IDE agent and the Cursor CLI (`agent`). The global instructions go into Cursor's [User Rules](https://cursor.com/docs/context/rules), and the agents install into `~/.cursor/agents/` ([subagents docs](https://cursor.com/docs/context/subagents)).

Ported from `../Claude/`.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `user-rules.md` | Cursor Settings → Customize → Rules (User Rules) | Global instructions, applied across all projects |
| `agents/explorer.md` | `~/.cursor/agents/explorer.md` | Read-only codebase research |
| `agents/implementer.md` | `~/.cursor/agents/implementer.md` | Builds one scoped unit of work |
| `agents/reviewer.md` | `~/.cursor/agents/reviewer.md` | Fresh-context adversarial diff review |
| `agents/verifier.md` | `~/.cursor/agents/verifier.md` | Runs the repo's real gates, reports pass/fail |
| `agents/side-fix.md` | `~/.cursor/agents/side-fix.md` | Takes an unrelated issue to its own PR in an isolated copy, in the background |
| `skills/pr-review/SKILL.md` | `~/.cursor/skills/pr-review/` | Reviews a GitHub PR with the `reviewer` subagent and reports findings |

## Install

**Rules.** Open Cursor Settings → Customize → Rules and paste the body of `user-rules.md` into User Rules. User Rules are stored on your Cursor account and sync, so a `git pull` does not update them. Paste again after the file changes.

**Agents.** A symlink keeps `~/.cursor/agents/` in sync with this repo. Run from the repo root:

```bash
mkdir -p ~/.cursor
p=~/.cursor/agents
if [ -e "$p" ] && [ ! -L "$p" ]; then mv -n "$p" "$p.bak"; fi
ln -sn "$PWD/Cursor/agents" ~/.cursor/agents
```

The `if` moves an existing real directory (even an empty one) to `.bak`. It leaves an existing symlink in place, so `ln -sn` then refuses with "File exists" instead of doing something silent. Plain `ln -s` is risky here: if the target is a directory, or a symlink to one, it exits 0 and creates the link *inside* it. On a re-run that means a self-referencing link inside this repo.

Linking the directory makes this repo the only source of `~/.cursor/agents/`. Add personal agents here, or copy the files (`cp`) instead of linking.

If `../Claude/` is also installed (`~/.claude/agents`), Cursor sees both sets. On a name clash the `~/.cursor/agents/` file wins.

To check an agent loaded, invoke it by name, for example `/explorer`.

**Skills.** Link the skill folder, not the whole skills directory, which may hold other skills ([skills docs](https://cursor.com/docs/context/skills)):

```bash
mkdir -p ~/.cursor/skills
ln -sn "$PWD/Cursor/skills/pr-review" ~/.cursor/skills/pr-review
```

If `~/.cursor/skills/pr-review` already exists as a real folder, move it out of `~/.cursor/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it.

Cursor also reads skills from `~/.claude/skills/`, `~/.codex/skills/`, and `~/.agents/skills/`. If other tools' `pr-review` variants are installed there, Cursor sees several, and which one wins is undocumented.

Unverified: whether a symlinked skill folder loads. If the skill does not show up, replace the link with a copy (`cp -R`).

## Differences from Claude

| Feature | Claude | Cursor |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md`, symlinked from this repo | User Rules, pasted into Settings; stored on the Cursor account; not applied to Inline Edit (Cmd/Ctrl+K) |
| Agent location | `~/.claude/agents/` | `~/.cursor/agents/`; also loads `~/.claude/agents/` and `~/.codex/agents/` (`.cursor/` wins on a name clash) |
| Agent format | Markdown, YAML frontmatter | Markdown, YAML frontmatter: `name`, `description`, `model`, `readonly`, `is_background` |
| Tool restriction | `tools` and `disallowedTools` per agent | No tool fields. `implementer` not spawning agents is enforced by its prompt only |
| Read-only enforcement | Tool list without edit tools; Bash limited by prompt | `readonly: true` on `explorer` and `reviewer` (no edits, no state-changing shell commands). `verifier` has no `readonly` because it must run builds and tests; its no-edit rule is enforced by its prompt only |
| Background | `background: true` | `is_background: true`; output is written to `~/.cursor/subagents/` |
| Worktree isolation | `isolation: worktree` in frontmatter | Requested in the spawn request (B3); the subagent runs in its own copy (a git worktree or a cloud environment) |
| Nested subagents | `side-fix` spawns `reviewer` and `verifier` | Same, but only when the main agent launched `side-fix` directly |
| Model selection | `model: inherit` | `model: inherit` (the default) or a model ID |

Hooks are not part of this port. Cursor reads hooks from `~/.cursor/hooks.json`.

## Known gaps

- User Rules do not apply to Inline Edit (Cmd/Ctrl+K). ([rules](https://cursor.com/docs/context/rules))
- User Rules live on the Cursor account, not in a file, so they can drift from `user-rules.md` until you paste again. ([help](https://cursor.com/help/customization/rules))
- There is no per-agent tool list. Agents can only be limited with `readonly`; everything else is prompt-enforced. ([subagents](https://cursor.com/docs/context/subagents))
- Subagents share the parent checkout unless the request asks for isolation. `side-fix` is only isolated if the orchestrator asks (B3). ([subagents](https://cursor.com/docs/context/subagents))
- An isolated subagent's changes stay on its branch until the parent agent merges them. Do not merge `side-fix`'s branch into the main task; it ships as its own PR (B3). ([subagents](https://cursor.com/docs/context/subagents))
- Cloud subagents use the team's MCP servers, not your local ones, so a `side-fix` running in a cloud environment may lose tracker access. ([subagents](https://cursor.com/docs/context/subagents))
- A subagent launched by another subagent cannot launch more. If `side-fix` is not launched by the main agent, it falls back to running the gates itself and a non-independent review. ([subagents](https://cursor.com/docs/context/subagents))
- Unverified: user rule files in `~/.cursor/rules`. The help center mentions them, but their format is undocumented, so none ships here. ([help](https://cursor.com/help/customization/rules))
- Unverified: whether User Rules apply in the CLI (`agent`). The CLI reads `AGENTS.md` and `CLAUDE.md` at the project root and `.cursor/rules`. ([CLI](https://cursor.com/docs/cli/using))
- Unverified: whether Cursor reads `~/.claude/CLAUDE.md`. ([rules](https://cursor.com/docs/context/rules))
- Unverified: whether subagents receive User Rules. The agents cite rules by number (B5, B6), and those references only resolve if User Rules reach the subagent. ([subagents](https://cursor.com/docs/context/subagents))
- Unverified: whether `readonly` blocks test runs. `reviewer` may run an existing test to confirm a defect; if `readonly` blocks it, the reviewer describes the test instead. ([subagents](https://cursor.com/docs/context/subagents))

## Per-repo overrides

- Repo rules add project rules on top of these User Rules: `.cursor/rules/*.mdc` (frontmatter `description`, `globs`, `alwaysApply`) and `AGENTS.md` at the root and in subdirectories; the CLI applies a root `CLAUDE.md` as a rule. Cursor ranks Team > Project > User. If a repo rule conflicts with these User Rules and the A3 order doesn't settle it, the agent stops and asks.
- A repo can add agents in `.cursor/agents/` (Cursor also reads `.claude/agents/` and `.codex/agents/`; the `.cursor/` file wins on a name clash). Project agents take precedence on a name clash, so a repo's `.cursor/agents/`, `.claude/agents/`, or `.codex/agents/` agent with the same `name` replaces the user-level one, including this port's `side-fix`. ([subagents](https://cursor.com/docs/context/subagents))
- Worktree setup commands for a repo go in `.cursor/worktrees.json`. The CLI's `-w/--worktree [name]` creates `~/.cursor/worktrees/<repo>/<name>`. ([worktrees](https://cursor.com/docs/configuration/worktrees))
