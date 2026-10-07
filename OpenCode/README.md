# OpenCode

User-level configuration for [OpenCode](https://opencode.ai/docs/) 1.x. Everything here installs into `~/.config/opencode/`.

Ported from `../Claude/`. Same workflow, same rules, same five agents, and the `pr-review` skill.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `AGENTS.md` | `~/.config/opencode/AGENTS.md` | Global instructions loaded into every session |
| `agents/explorer.md` | `~/.config/opencode/agents/explorer.md` | Read-only codebase research |
| `agents/implementer.md` | `~/.config/opencode/agents/implementer.md` | Builds one scoped unit of work |
| `agents/reviewer.md` | `~/.config/opencode/agents/reviewer.md` | Fresh-context adversarial diff review |
| `agents/verifier.md` | `~/.config/opencode/agents/verifier.md` | Runs the repo's real gates, reports pass/fail |
| `agents/side-fix.md` | `~/.config/opencode/agents/side-fix.md` | Takes an unrelated issue to its own PR in a worktree the orchestrator creates |
| `skills/pr-review/SKILL.md` | `~/.config/opencode/skills/pr-review/` | Reviews a GitHub PR with the `reviewer` subagent and reports findings |

Not part of this port: commands (`~/.config/opencode/commands/`) and plugins (`~/.config/opencode/plugins/`).

## Install

Symlinks keep `~/.config/opencode/` in sync with this repo, so a `git pull` updates your setup. Run from the repo root:

```bash
mkdir -p ~/.config/opencode/skills
for p in ~/.config/opencode/AGENTS.md ~/.config/opencode/agents; do
  if [ -e "$p" ] && [ ! -L "$p" ]; then mv -n "$p" "$p.bak"; fi
done
ln -sn "$PWD/OpenCode/AGENTS.md" ~/.config/opencode/AGENTS.md
ln -sn "$PWD/OpenCode/agents" ~/.config/opencode/agents
ln -sn "$PWD/OpenCode/skills/pr-review" ~/.config/opencode/skills/pr-review
```

The loop moves an existing real file or directory to `.bak` (`-n` never overwrites an earlier `.bak`; if one exists, the move is skipped and `ln -sn` refuses). It leaves an existing symlink in place, so `ln -sn` then refuses with "File exists" instead of doing something silent. Plain `ln -s` is risky here: if the target is a directory, or a symlink to one, it exits 0 and creates the link *inside* it. On a re-run that means a self-referencing link inside this repo.

Linking the directory makes this repo the only source of your user-level agents in `agents/`. OpenCode also accepts the singular `~/.config/opencode/agent/` and agents defined under `"agent"` in `opencode.json`, so check those too. Add personal agents here, or copy the files (`cp`) instead of linking.

The skill is linked one folder at a time, because `~/.config/opencode/skills/` may hold other skills. If `~/.config/opencode/skills/pr-review` already exists as a real folder, move it out of `~/.config/opencode/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it. OpenCode also reads `~/.claude/skills/` and `~/.agents/skills/`. If the Claude variant of `pr-review` is installed there, OpenCode sees two `pr-review` skills. Unverified: which one wins.

Unverified: whether OpenCode loads a symlinked skill folder. If `pr-review` does not show up, remove the link and copy the folder instead (`cp -R`).

Once `~/.config/opencode/AGENTS.md` exists, OpenCode stops falling back to `~/.claude/CLAUDE.md`. It never reads `~/.claude/agents`.

Optional: to let `side-fix` run in the background, set this in your shell profile ([CLI](https://opencode.ai/docs/cli/)). Without it, `side-fix` runs in the foreground and the main task waits for it.

```bash
export OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true
```

## Differences from Claude

| Feature | Claude | OpenCode |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md` | `~/.config/opencode/AGENTS.md`; falls back to `~/.claude/CLAUDE.md` only when it is absent |
| Imports | `@file` in `CLAUDE.md` | No `@file` import; list files under `instructions` in `opencode.json` |
| Agent location | `~/.claude/agents/`; repo `.claude/agents/` | `~/.config/opencode/agents/`; repo `.opencode/agents/` |
| Agent format | Markdown, frontmatter `name`, `description`, `tools`, `model` | Markdown, frontmatter `description`, `mode: subagent`, `permission`; the filename is the agent name |
| Tool restriction | `tools` and `disallowedTools` lists | `permission` map: `allow`, `ask`, or `deny` per tool; pattern maps (used here for `bash` and `task`) |
| Read-only enforcement | Instructions only; Bash kept | `edit: deny` plus a bash map (`"*": ask`, read-only commands allowed) on explorer and reviewer. Verifier's bash map only denies commit, push, checkout, switch, stash, reset, rebase, merge, restore, and clean, so it can still run builds and tests; its prompt is the main guard |
| Background | `background: true` in frontmatter | Per call, `task` with `background: true`; needs `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` |
| Worktree isolation | `isolation: worktree` | Not documented; the orchestrator creates `.worktrees/<TICKET-ID>` before spawning `side-fix` (B3) |
| Nested subagents | side-fix spawns `reviewer` and `verifier` | Same, through side-fix's `permission.task`; every other agent has `task: deny` |
| Model selection | `model: inherit` | `model` omitted, so subagents use the caller's model |

## Known gaps

- This port targets OpenCode 1.x. OpenCode V2 recognizes `AGENTS.md` only, with no `CLAUDE.md` fallback, and renames the permission keys `bash` to `shell` and `task` to `subagent`. Its migration guide says the V1 forms are still accepted. ([V2 docs](https://opencode.ai/v2/docs))
- No worktree isolation. OpenCode does not document per-agent worktrees, so the orchestrator creates one inside the repo and `side-fix` checks it is in that path before any change. ([agents](https://opencode.ai/docs/agents/))
- Gates that don't read git excludes (some lint and test globs) can pick up files under `.worktrees/` while a side-fix worktree exists.
- Background subagents are experimental. They need `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` and are a per-call choice with no frontmatter field. ([CLI](https://opencode.ai/docs/cli/); source: `task.ts`)
- Unknown frontmatter keys are passed to the model provider as request options, so Claude's `disallowedTools`, `isolation`, and `background` stay out of these files. `tools` is a recognized but deprecated field; these files use `permission` instead. `model` is omitted rather than set to Claude's `inherit`. ([agents](https://opencode.ai/docs/agents/))
- Bash patterns are a backstop, not a sandbox. OpenCode splits a compound command into simple commands and checks each one against the patterns, with `*` matching anything and a trailing ` *` also matching the bare command. So `cat a > b` still matches `"cat *"`, and `"git merge*"` would also match `git merge-base`. That is why the maps deny `"git merge"` and `"git merge *"` instead. The last matching rule wins, which is why `"*": ask` comes first in the explorer and reviewer maps. ([agents](https://opencode.ai/docs/agents/); source: `util/wildcard.ts`)
- An agent's bash map is applied after your global `permission` rules, so an agent-level `"*": allow` would re-allow commands you deny globally. The implementer, verifier, and side-fix maps hold only deny and ask entries. Bash is allowed by default, so they behave the same when you have no global rules.
- A repo agent with the same name as a user-level one overrides it field by field instead of replacing it. OpenCode loads the global config dir first, then `.opencode` dirs, and merges agents with `mergeDeep`, so keys the repo file omits keep their user-level values and `permission` maps merge. (source: `config/paths.ts`, `config.ts`)

## Per-repo overrides

- A repo's `AGENTS.md` (or `CLAUDE.md` if there is no `AGENTS.md`) adds project rules on top of this one. OpenCode loads the global file, then the project file(s), then each `instructions` entry in `opencode.json`, and combines them. If they conflict and the A3 order doesn't settle it, the agent stops and asks. ([rules](https://opencode.ai/docs/rules/))
- A repo's `.opencode/agents/<name>.md` adds agents for that repo. One with the same name as a user-level agent overrides it for that repo, field by field, with `permission` maps merged (see Known gaps).
