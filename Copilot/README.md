# GitHub Copilot CLI

User-level configuration for [GitHub Copilot CLI](https://docs.github.com/copilot). Everything here installs into `~/.copilot/`. Ported from `../Claude/`.

This port targets Copilot CLI only. VS Code Copilot Chat reads repo files through its own settings (for example `chat.useClaudeMdFile` for `CLAUDE.md`), and its user-level paths were not checked.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `copilot-instructions.md` | `~/.copilot/copilot-instructions.md` | Global instructions loaded into every session |
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

Linking the agents directory makes this repo the only source of your user-level agents. Add personal agents here, or copy the files (`cp`) instead of linking. Skills are linked one folder at a time, because `~/.copilot/skills/` may hold other skills too. If `~/.copilot/skills/pr-review` already exists as a real folder, move it out of `~/.copilot/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it.

Copilot also reads personal skills from `~/.agents/skills/`, so keep other tools' variants of `pr-review` out of it. ([add skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills))

Confirm it loaded: `/instructions` lists the loaded instruction files, and the `/agent` picker lists the agents.

Unverified: whether Copilot CLI follows a symlinked agents directory or a symlinked skill folder. If an agent or the skill does not show up, copy instead:

```bash
rm ~/.copilot/agents ~/.copilot/skills/pr-review   # removes the symlinks only
cp -R "$PWD/Copilot/agents" ~/.copilot/
cp -R "$PWD/Copilot/skills/pr-review" ~/.copilot/skills/
```

No settings change is needed. User settings live in `~/.copilot/settings.json`; `config.json` is auto-managed.

## Differences from Claude

| Feature | Claude | Copilot CLI |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md` | `~/.copilot/copilot-instructions.md`, plain Markdown, no frontmatter |
| Agent location | `~/.claude/agents/`; repo `.claude/agents/` wins on a name clash | `~/.copilot/agents/`; on an ID clash user-level wins, then repo `.github/agents/`, then `.claude/agents/` |
| Agent format | `<name>.md`, YAML frontmatter | `<name>.agent.md`, YAML frontmatter; ID is the filename; `include-custom-instructions: true` loads repo instructions |
| Tool restriction | `tools` allowlist, `disallowedTools` | `tools` alias list (`read`, `edit`, `search`, `execute`, `web`, `todo`, `agent`); no deny list; implementer's list leaves out MCP tools, which Claude's implementer has; side-fix omits it to keep all tools |
| Read-only enforcement | Prompt; Bash kept | Prompt; no `edit` alias, but `execute` kept |
| Background | `background: true` on side-fix | Chosen per spawn, not in the agent file; the orchestrator spawns side-fix in the background (B3, B9) |
| Worktree isolation | `isolation: worktree` on side-fix | Per session only (`/new worktree`, `--worktree`); the orchestrator creates `.worktrees/<TICKET-ID>` for side-fix (B3) |
| Nested subagents | side-fix spawns `reviewer` and `verifier` | Same; tool filters carry into nested subagents |
| Model selection | `model: inherit` | `model` unset, so it inherits |

## Known gaps

- Read-only is prompt-enforced. explorer, reviewer, and verifier have no `edit` alias, but `execute` can still write files, and verifier needs it to run builds and tests. ([custom agents configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration))
- User agents override repo agents. A repo's `.github/agents/reviewer.agent.md` or `.claude/agents/` reviewer does not replace yours; yours wins. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
- Background is per spawn. No agent-file key makes side-fix run in the background, so it does so only when the orchestrator spawns it that way. `read_agent` checks a background agent's status, and `/tasks` lists running tasks. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
- Custom agents spawned as subagents get only the repo's instruction files (`copilot-instructions.md`, `AGENTS.md`, `CLAUDE.md`), and only through `include-custom-instructions: true`, which every agent here sets. Unverified: whether the personal `~/.copilot/copilot-instructions.md` reaches them. If it does not, subagents never see Part A or B0, and side-fix's references to B2, B5, and B6 of `copilot-instructions.md` point at nothing. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference), [create custom agents](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli))
- The default subagent depth is 6 in the CLI reference's limits table and 4 for `COPILOT_SUBAGENT_MAX_DEPTH` on the same page. Either way side-fix can spawn `reviewer` and `verifier`. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
- Gates that don't read git excludes (some lint and test globs) can pick up files under `.worktrees/` while a side-fix worktree exists. ([gitignore](https://git-scm.com/docs/gitignore))
- Unverified: `@relative/path` imports in the personal instructions file. They work in repo `.github/copilot-instructions.md`, `AGENTS.md`, and `CLAUDE.md`. ([custom instructions](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions))
- Not part of this port: hooks (`~/.copilot/hooks/*.json`). ([use hooks](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/use-hooks))

## Per-repo overrides

- Copilot CLI reads a repo's `.github/copilot-instructions.md`, `.github/instructions/**`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, and `.claude/rules`, at the root, the cwd, the directories in between, and nested paths. They are combined with this file and de-duplicated, with no general precedence order. If they conflict and the A3 order doesn't settle it, the agent stops and asks. ([custom instructions](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions))
- `/instructions` toggles individual files; `--no-custom-instructions` skips them all.
- A repo's `.github/agents/<name>.agent.md` or `.claude/agents/` agent does not replace the user-level agent with the same ID. The user-level agent wins, then `.github/agents/`, then `.claude/agents/`. ([CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference))
