# Claude Code

User-level configuration for [Claude Code](https://code.claude.com/docs). Everything here installs into `~/.claude/`.

This directory is the canonical version of the workflow. The other tool directories are ports of it.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `CLAUDE.md` | `~/.claude/CLAUDE.md` | Global instructions loaded into every session |
| `agents/explorer.md` | `~/.claude/agents/explorer.md` | Read-only codebase research |
| `agents/implementer.md` | `~/.claude/agents/implementer.md` | Builds one scoped unit of work |
| `agents/reviewer.md` | `~/.claude/agents/reviewer.md` | Fresh-context adversarial diff review |
| `agents/verifier.md` | `~/.claude/agents/verifier.md` | Runs the repo's real gates, reports pass/fail |
| `agents/side-fix.md` | `~/.claude/agents/side-fix.md` | Takes an unrelated issue to its own PR in a background worktree |
| `skills/pr-review/SKILL.md` | `~/.claude/skills/pr-review/` | Reviews a GitHub PR with the `reviewer` subagent and reports findings |

## Install

Symlinks keep `~/.claude/` in sync with this repo, so a `git pull` updates your setup. Run from the repo root:

```bash
mkdir -p ~/.claude/skills
for p in ~/.claude/CLAUDE.md ~/.claude/agents; do
  if [ -e "$p" ] && [ ! -L "$p" ]; then mv -n "$p" "$p.bak"; fi
done
ln -sn "$PWD/Claude/CLAUDE.md" ~/.claude/CLAUDE.md
ln -sn "$PWD/Claude/agents" ~/.claude/agents
ln -sn "$PWD/Claude/skills/pr-review" ~/.claude/skills/pr-review
```

The loop moves an existing real file or directory to `.bak` (`-n` never overwrites an earlier `.bak`; if one exists, the move is skipped and `ln -sn` refuses). It leaves an existing symlink in place, so `ln -sn` then refuses with "File exists" instead of doing something silent. Plain `ln -s` is risky here: if the target is a directory, or a symlink to one, it exits 0 and creates the link *inside* it. On a re-run that means a self-referencing link inside this repo.

Skills are linked one folder at a time, because `~/.claude/skills/` usually holds other skills too. If `~/.claude/skills/pr-review` already exists as a real folder, move it out of `~/.claude/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it.

Unverified: whether Claude Code follows a symlinked `agents` directory and symlinked skill folders. If an agent or skill does not show up, replace the links with copies (`cp -R`).

This repo is public. With the directory linked, anything written to `~/.claude/agents/` lands in this working tree, so keep personal or employer-specific agents out of it (copy instead of link) and check `git status` before committing.

Start a new session to pick up changes. To confirm an agent loaded, invoke it, for example by asking for the `explorer` subagent.

## How it works

`CLAUDE.md` has two parts:

- **Part A (everywhere):** no guessing (ask in one early batch), concise replies, and a precedence order: your instruction in the session, then a project workflow skill, then this file.
- **Part B (development):** every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The main session orchestrates and delegates:

```
orient      explorer          map the code, find the real gates
plan        main session      scope, files, risks, one batch of questions
build       implementer x N   one per independent unit, parallel when file scopes don't overlap
            main session      review each unit's diff and commit it
verify      reviewer          fresh-context review of the diff
            verifier          run build, tests, lint, type check exactly as CI does
ship        main session      push, PR, move the ticket, link both ways, stop
report      main session      fixed B7 format

side issue  side-fix          own ticket, branch, worktree, and PR, in the background
```

Hard rails (B6) apply to every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 stops and asks.

## Agents

| Agent | Tools | Edits | Notes |
|---|---|---|---|
| `explorer` | Read, Grep, Glob, Bash, WebFetch, WebSearch | No | Returns a file map and findings tagged verified or inferred |
| `implementer` | All except Agent | In-scope files only | Tests first; leaves changes uncommitted for the orchestrator |
| `reviewer` | Read, Grep, Glob, Bash, WebFetch | No | Verdict plus findings by severity with `path:line` |
| `verifier` | Read, Grep, Glob, Bash | No | Discovers gates from CI config; never invents a command |
| `side-fix` | All | Yes | `isolation: worktree`, `background: true`; spawns only `reviewer` and `verifier` (prompt-enforced) |

All agents use `model: inherit`, so they run on the main session's model.

The read-only agents have no Edit or Write tool, so the tool list blocks file edits. They keep Bash for commands like `git log` and `rg`, and only their instructions keep Bash from writing. For enforcement beyond the prompt, use a `PreToolUse` hook or Claude Code's sandbox (see the [permissions docs](https://code.claude.com/docs)). Settings-level `permissions.deny` rules apply to the whole session, not to one agent.

## Skills

| Skill | Use it for |
|---|---|
| `pr-review` | "review PR 123", or a PR URL. Takes precedence over the generic `code-review` skill for PR reviews. Fetches the PR into git's object store with hooks off and reads it there, with no checkout. Finds the ticket, reads the standards on the base branch, reads CI, spawns `reviewer`, confirms the blockers and majors, and reports in chat. Never runs the PR's code unless you ask. Posts to GitHub only when you ask, and only after you approve the exact text. |

## Per-repo overrides

- A repo's `CLAUDE.md` adds project rules on top of this one. If the two conflict and the A3 order doesn't settle it, the agent stops and asks.
- A repo's `.claude/agents/<name>.md` with the same `name` replaces the user-level agent for that repo.
