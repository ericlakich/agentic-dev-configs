# OpenCode

User-level configuration for [OpenCode](https://opencode.ai/docs/) 1.x. Everything here installs into `~/.config/opencode/`.

Ported from `../Claude/`. Same workflow, same rules, the orchestrator and its six workers, and the `pr-review` skill.

## Contents

| File | Installs to | Purpose |
|---|---|---|
| `AGENTS.md` | `~/.config/opencode/AGENTS.md` | Global instructions loaded into every session |
| `agents/orchestrator.md` | `~/.config/opencode/agents/orchestrator.md` | Primary agent: runs the outer loop and dispatches the workers |
| `agents/explorer.md` | `~/.config/opencode/agents/explorer.md` | Read-only codebase research |
| `agents/implementer.md` | `~/.config/opencode/agents/implementer.md` | Builds one scoped unit of work |
| `agents/reviewer.md` | `~/.config/opencode/agents/reviewer.md` | Fresh-context adversarial diff review |
| `agents/verifier.md` | `~/.config/opencode/agents/verifier.md` | Runs the repo's real gates, reports pass/fail |
| `agents/adversarial.md` | `~/.config/opencode/agents/adversarial.md` | Last verify pass: attacks the change for security loopholes; nothing ships as ready without its approval |
| `agents/side-fix.md` | `~/.config/opencode/agents/side-fix.md` | Takes an unrelated issue to ready for verify in a worktree the orchestrator creates; the orchestrator verifies and ships it |
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

The `agents` link installs `orchestrator.md` together with the six workers. To start it, see [Run the orchestrator](#run-the-orchestrator).

The skill is linked one folder at a time, because `~/.config/opencode/skills/` may hold other skills. If `~/.config/opencode/skills/pr-review` already exists as a real folder, move it out of `~/.config/opencode/skills/` first (a `.bak` copy left beside it would load as a second skill). Otherwise `ln -sn` creates the link inside it. OpenCode also reads `~/.claude/skills/` and `~/.agents/skills/`. If the Claude variant of `pr-review` is installed there, OpenCode sees two `pr-review` skills. Unverified: which one wins.

Unverified: whether OpenCode loads a symlinked skill folder. If `pr-review` does not show up, remove the link and copy the folder instead (`cp -R`).

Once `~/.config/opencode/AGENTS.md` exists, OpenCode stops falling back to `~/.claude/CLAUDE.md`. It never reads `~/.claude/agents`.

Optional: to let `side-fix` run in the background, set this in your shell profile ([CLI](https://opencode.ai/docs/cli/)). Without it, `side-fix` runs in the foreground and the main task waits for it.

```bash
export OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true
```

## Run the orchestrator

The orchestrator is a primary agent (`mode: primary`): it becomes the main session, not a subagent you spawn. Start each development session with:

```bash
opencode --agent orchestrator
```

Or switch to it inside a session with Tab, the default `agent_cycle` keybind. ([source][src]: `packages/tui/src/config/keybind.ts:130`)

To make it the default instead, put `default_agent` in a personal config file and point `OPENCODE_CONFIG` at that file only in the shells or directories where every session is development work. OpenCode loads it after your global config. ([source][src]: `config/config.ts:415-416`)

```json
{ "default_agent": "orchestrator" }
```

Do not put `default_agent` in a repo's `opencode.json`: that file is usually committed and shared, and a teammate without the orchestrator agent gets `default agent "orchestrator" not found`. ([source][src]: `agent/agent.ts:330-332`)

Do not set it in your user-level `opencode.json` either. The agent's body replaces the model provider's base prompt, so a user-level default would replace it in every session, including non-development work. Environment info, `AGENTS.md`, MCP instructions, and skills are still appended. That is why `orchestrator.md` carries a short "Operating rules" section. ([source][src]: `session/llm/request.ts:60`, `session/prompt.ts:1257-1269`)

`default_agent` refuses a hidden agent, and `hidden: true` would also drop the orchestrator from Tab, so `orchestrator.md` does not set it. ([source][src]: `agent/agent.ts:334`, `packages/tui/src/context/local.tsx:78`)

A plain session follows the same loop: AGENTS.md B0 tells a main session not running as the orchestrator to read `~/.config/opencode/agents/orchestrator.md` and follow its body for any code change. That read is outside the project, so it asks for `external_directory` permission (see Known gaps).

## How it works

`AGENTS.md` has two parts:

- **Part A (everywhere):** no guessing (gather context, then ask questions and permissions in one early batch), concise replies, and a precedence order: your instruction in the session, then a project workflow skill, then this file.
- **Part B (development):** every change gets a ticket, a `<TICKET-ID>/<slug>` branch, and a PR. The work runs as agentic loops (B4): gather context, act, check against ground truth, decide. A loop repeats until its exit test passes, its budget runs out, progress stalls, or it needs your input. Ground truth is outside the model: command output, test results, file content, or a verdict from an agent that did not write the code.

The orchestrator runs the outer loop:

```
intake -> orient -> plan -> build -> verify -> ship -> report
           ^  |      ^ ^     | ^      |  |
           +--+      | +-----+ +------+  |
                     +-------------------+

orient -> orient   unknowns get one re-brief, then go into the question batch
plan   -> you      the one question batch: questions, permissions, expected forks; then no questions until the report
build  -> plan     a blocked unit whose fix fits the plan: re-plan
verify -> build    in-scope findings and diff-caused gate failures become fix briefs; at most 3 fix rounds
verify -> build    once review passes, adversarial findings become fix briefs too; from its first pass, every round
                   counts against its budget of 3; the PR is ready only with its approval, otherwise it ships as a draft
verify -> plan     a design-level finding: re-plan
any    -> park     no safe option, exhausted budget, or stall: park the affected units, finish the rest,
                   and ask in the report (B4 Autonomy); a safe, reversible gap is decided and logged
```

Each phase hands bounded units to workers, and each worker runs its own inner loop:

```
intake      orchestrator      not code: answer; PR review: pr-review skill; code change: orient
orient      explorer x N      search, read, check; at most 3 widening rounds per question
            orchestrator      tracker and candidate tickets, workflow skill, standards, real gates
plan        orchestrator      units with disjoint file scopes, exit tests, gates; one batch of questions and permissions;
                              then the ticket (match or new) and its branch
build       implementer x N   red, green, check; at most 3 attempts per failing check
            orchestrator      read each unit's diff and commit it
verify      reviewer          fresh every round; findings R1, R2, ... marked closed, open, regressed, or disputed
            verifier          one pass per call; a failing gate re-runs once, a changed result is FLAKY
            adversarial       last, once review passes; fresh every pass; findings S1, S2, ... with exploit paths;
                              local throwaway PoCs only (every shell command prompts); at most 3 rounds; the PR is ready only on its approval
ship        orchestrator      push, PR (stacked when there is more than one; every stack merge gets a verifier and a fresh
                              adversarial before it ships), move the ticket, link both ways;
                              merge bottom-up only if the batch granted it
report      orchestrator      B7: short sections, each under 250 words where possible, with a merge-order table for stacks

side issue  side-fix          own ticket, branch, and worktree; scaled-down loop to ready for verify; background when enabled
            orchestrator      side-fix's verify loop (fresh reviewer every round, 3 fix rounds, then its own adversarial pass, 3 rounds), then its PR
```

Every worker report starts with `Status: <done|partial|blocked|needs-input> after <n> passes: <reason>`; a report without it counts as `partial`. Workers never ask you anything; they return `needs-input`, and the orchestrator decides it (if a safe, reversible option exists) or parks it for the report. The orchestrator keeps a ledger (B8) of budgets, units and their worker IDs (`task_id`), fix rounds, findings, side tickets, PRs in stack order, your answers, the decisions it made for you, and parked questions. It re-posts the ledger at every phase change except into the report, and rebuilds from it after a context reset.

You hear from the orchestrator twice with something to answer: the question batch after Orient, and the B7 report. In between, it decides safe, reversible gaps itself and logs them, parks anything else, and finishes the rest. The only interrupt is a committed secret. When a session opens more than one PR into the same base, they form one stack (B2): side fixes at the bottom and the main PR on top, opened only after every side-fix has ended. Each PR targets the branch of the one below it, and the report's Merge order table says what to merge first.

The question batch covers workflow permissions (merge, deploys, new dependencies). OpenCode's own permission prompts are separate: a run still pauses on every `ask` rule, such as the `"*": ask` fallback in the explorer, reviewer, and adversarial bash maps, `"git -C *": ask` in side-fix's (answer it with "Allow once", see Known gaps), and anything your global `permission` config sets to `ask`. For an unattended run, allow the commands and tools the workflow uses (`git`, `gh`, your tracker's tools) under `permission` in your global `opencode.json`, and keep the B6 rails. An agent's own map is applied after your global rules, so its `ask` entries still prompt (see Known gaps); a repo override of that agent can change them (Per-repo overrides).

Two gates protect the workflow itself. First, workflow-control paths (each AI tool's folder, such as `.opencode/`, `.agents/` and `.claude/`, `opencode.json`/`opencode.jsonc`, and instruction files such as `AGENTS.md` and `CLAUDE.md`) and git's own config and hooks are checked, best effort and after the fact (not a sandbox), each time an implementer returns and before every review. The question batch lists the ones the plan will touch, so you can approve them up front. An unapproved change that is incidental is removed by a fix brief: a file the session created is deleted and a tracked file that was clean before the session is restored, but a file that was already untracked, ignored, or changed before the session is never deleted, overwritten, or restored (a change to it is parked). An unapproved change the ticket needs is parked, and the main PR ships as a draft. An approved change to hooks, settings, MCP config, plugins, skills, commands, or the definition of an agent the loop spawns also makes it a draft; only approved plain instruction-file changes proceed. A change to your user-level config files, wherever they live, is parked; so is any change to the main checkout when the work runs in a separate worktree because that config lives in the repo. Second, a pre-existing security issue on a public repo is parked for the report with a private-advisory recommendation, never filed publicly.

Hard rails (B6) apply to every agent: no secrets, no `--no-verify`, no skipped tests, no force-push or history rewrites, no production changes without an explicit go-ahead.

The workflow assumes the agent can reach the repo's ticket tracker (through an MCP server or CLI) and the git host's CLI for PRs. Without tracker access, B2 asks in the question batch.

Model choice (B9) falls back to configured models here. The `task` tool takes no model argument: a subagent runs on the `model` in its agent file (`provider/model-id`) or, when that is omitted, on the calling agent's model, and a resume by `task_id` recomputes the model the same way. These files omit `model`, so every worker runs on the orchestrator's model. To run a worker on a cheaper or stronger model, set `model` in a personal or repo override of that agent. ([source][src]: `tool/task.ts:43-62`, `tool/task.ts:136`, `tool/task.ts:181-209`; [agents](https://opencode.ai/docs/agents/))

Loop budgets (B4) are enforced by instructions only. OpenCode's `steps` agent field is an optional hard cap on an agent's agentic iterations, after which it must answer in text; these files do not set it. ([source][src]: `packages/core/src/v1/config/agent.ts:34-37`)

## Differences from Claude

| Feature | Claude | OpenCode |
|---|---|---|
| Global file | `~/.claude/CLAUDE.md` | `~/.config/opencode/AGENTS.md`; falls back to `~/.claude/CLAUDE.md` only when it is absent |
| Imports | `@file` in `CLAUDE.md` | No `@file` import; list files under `instructions` in `opencode.json` |
| Agent location | `~/.claude/agents/`; repo `.claude/agents/` | `~/.config/opencode/agents/`; repo `.opencode/agents/` |
| Agent format | Markdown, frontmatter `name`, `description`, `tools`, `model` | Markdown, frontmatter `description`, `mode`, `permission`; the filename is the agent name. Workers set `mode: subagent`, the orchestrator `mode: primary` |
| Main-thread orchestrator | `orchestrator.md`, run with `claude --agent orchestrator` or the `agent` setting; its body replaces Claude Code's default system prompt | `orchestrator.md` with `mode: primary`, run with Tab, `opencode --agent orchestrator`, or `default_agent` in a personal config loaded through `OPENCODE_CONFIG`; its body replaces the provider's base prompt |
| Spawn allowlist | Prompt-enforced; optional `tools: Agent(...)` list on the main-thread agent | Hard: the orchestrator's `permission.task` map (`"*": deny` first, then the six workers allowed; the last match wins). Every worker has `task: deny` |
| Orchestrator file edits | Prompt-enforced | `edit: deny` blocks the edit, write, and apply_patch tools ([source][src]: `permission/index.ts:204-213`). Bash stays at the default so it can run `git` and `gh`, and Bash can still write files, so the rule stays partly prompt-enforced. The deny does not reach the workers it spawns (Known gaps) |
| Questions to the user | `AskUserQuestion` tool | `question` permission. Custom agents inherit `question: deny`, so the orchestrator sets `question: allow`, as the built-in build and plan agents do ([source][src]: `agent/agent.ts:126`, `agent/agent.ts:148`, `agent/agent.ts:163`) |
| Resume a worker | `SendMessage` to its ID or name | `task` with the `task_id` from the earlier result (`<task id="ses_..." ...>`); it continues the same child session |
| Tool restriction | `tools` and `disallowedTools` lists | `permission` map: `allow`, `ask`, or `deny` per tool; pattern maps (used here for `bash` and `task`) |
| Read-only enforcement | Instructions only; Bash kept | `edit: deny` plus a bash map (`"*": ask`, read-only commands allowed, best effort; see Known gaps) on explorer and reviewer; the adversarial has `"*": ask` only, with no allows (see Known gaps). Verifier's bash map only denies commit, push, checkout, switch, stash, reset, rebase, merge, restore, and clean (also with git global options before the subcommand, such as `git -C <dir> push`), so it can still run builds and tests; its prompt is the main guard |
| Background | `background: true` in frontmatter | Per call, `task` with `background: true`; needs `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` |
| Worktree isolation | `isolation: worktree` | Not documented; the orchestrator creates `.worktrees/<TICKET-ID>` before spawning `side-fix` (B3) |
| Nested subagents | side-fix spawns `reviewer`, `verifier`, and `adversarial` | None: `subagent_depth` defaults to 1, and every worker has `task: deny`. side-fix stops at ready for verify; the orchestrator runs its verify loop, stacks its branch, and ships its PR (B3) |
| Model selection | `model: inherit`; the orchestrator picks per spawn with the Agent tool's `model` parameter (B9) | `model` omitted, so subagents use the caller's model; no per-spawn choice (B9) |

## Known gaps

Paths cited after a "source" link in this README are relative to `packages/opencode/src/` unless they start with `packages/`. They point at anomalyco/opencode commit `3884062`, the same at v1.18.35; "V2 source" points at commit `6608799`.

- This port targets OpenCode 1.x. OpenCode V2 recognizes `AGENTS.md` only, with no `CLAUDE.md` fallback, and renames the permission keys `bash` to `shell` and `task` to `subagent`. Its migration guide says the V1 forms are still accepted. ([V2 docs](https://opencode.ai/v2/docs))
- V2 also renames the agent prompt key `prompt` to `system`, moves the nesting limit to `experimental.subagent_depth`, and rejects a primary agent requested as a subagent. New V2 agents default to `mode: primary`, so every worker file here sets `mode: subagent` explicitly. ([V2 source][src-v2]: `packages/core/src/tool/plugin/subagent.ts:129-137` for the depth setting and the primary-agent check)
- Nesting is off by default. `subagent_depth` defaults to 1, so a subagent cannot spawn another. `"subagent_depth": 2` in `opencode.json` would allow it, but this port does not rely on it: side-fix has `task: deny`, stops at ready for verify, and the orchestrator runs its verify loop and ships its PR. ([source][src]: `tool/task.ts:111-117`)
- V1 leaves primary agents out of the `task` tool's agent list but does not reject one requested by name, so every worker keeps `task: deny`, and the orchestrator's description says never to spawn it. ([source][src]: `tool/registry.ts:265-269`, `tool/task.ts:119-138`)
- Resume needs the exact `task_id`. An unknown ID does not fail; it silently starts a fresh session with no history. ([source][src]: `tool/task.ts:47-50`, `tool/task.ts:72`, `tool/task.ts:119-138`)
- The orchestrator's `edit: deny` does not reach the subagents it spawns: a child session gets only the parent session's deny rules, not the parent agent's, so implementers keep `edit`. ([source][src]: `agent/subagent-permissions.ts:8-10`, `agent/subagent-permissions.ts:21-23`)
- The plain-session fallback (AGENTS.md B0) reads `~/.config/opencode/agents/orchestrator.md`, outside the project, so OpenCode asks for `external_directory` permission. Allowing `~/.config/opencode/agents/*` under `permission.external_directory` avoids the prompt; a leading `~/` in a permission pattern expands to your home directory. ([source][src]: `agent/agent.ts:122-125`, `tool/external-directory.ts:24-36`, `permission/index.ts:178-184`)
- No worktree isolation. OpenCode does not document per-agent worktrees, so the orchestrator creates one inside the repo and `side-fix` checks it is in that path before any change. ([agents](https://opencode.ai/docs/agents/))
- Gates that don't read git excludes (some lint and test globs) can pick up files under `.worktrees/` while a side-fix worktree exists.
- Background subagents are experimental. They need `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` and are a per-call choice with no frontmatter field. ([CLI](https://opencode.ai/docs/cli/); source: `task.ts`) A background worker notifies the caller when it completes, so the orchestrator never polls; calling `task` with the `task_id` of one still running queues a new prompt into it, so the orchestrator never does that to check status. ([source][src]: `tool/task.ts:58-60`, `tool/task.ts:267-282`)
- Unknown frontmatter keys are passed to the model provider as request options, so Claude's `disallowedTools`, `isolation`, and `background` stay out of these files. `tools` is a recognized but deprecated field; these files use `permission` instead. `model` is omitted rather than set to Claude's `inherit`. ([agents](https://opencode.ai/docs/agents/))
- Bash patterns are a backstop, not a sandbox. OpenCode splits a compound command into simple commands and checks each one against the patterns, with `*` matching anything and a trailing ` *` also matching the bare command. So `cat a > b` still matches `"cat *"`, and `"git merge*"` would also match `git merge-base`. That is why the maps deny `"git merge"` and `"git merge *"` instead. The last matching rule wins, which is why `"*": ask` comes first in the explorer, reviewer, and adversarial maps. ([agents](https://opencode.ai/docs/agents/); source: `util/wildcard.ts`)
- The explorer and reviewer bash maps are best effort, not an exec barrier; the prompt rules remain the primary limit. They are `"*": ask` first, then read-only allows, so anything off the list prompts; the last matching rule wins. Bare `git diff` and `git diff <args>` are allowed but `git difftool` is not. `git branch -a` is allowed only exactly (the explorer also allows bare `git branch` and `git branch --list`, with no pattern or other flag). `git grep` is allowed, but `-O`, `--open-files-in-pager`, and its abbreviations prompt (the pattern `*-*O*` prompts on any uppercase `O` after any dash, so ordinary searches such as `git grep -n TODO <sha>`, `git grep -n "JSON" <sha>`, and `git grep -n foo <sha> -- OpenCode/` prompt too, as does `*--o*` on forms like `--only-matching`; answer those with "Allow once"). The `gh` patterns `*-w*` and `*-?w*` similarly prompt on any `-w`-like text, such as a URL containing `my-website`, and `go list*flags*` prompts on a package path containing `flags`. These flags prompt, because they run a program, open a browser, or write a file: `git ... --out*` (covers `--output`), `rg --pre`, `rg --hostname-bin`, `gh ... --web` and `-w`, and `find` with `-exec`, `-execdir`, `-ok`, `-okdir`, `-fprint`, or `-fls`; the explorer also prompts on `go list` with `-toolexec`, `-mod` (which also covers `-modfile`), any `*flags` option (`-gcflags`, `-asmflags`, `-gccgoflags`, `-ldflags`), `-export`, or `-compiled` (which runs cgo and the C compiler). Any `-delete` text in a `find` command prompts, and the deny covers only a space-separated `-delete` flag (a tab-separated one prompts instead). Answering a prompt with "Allow always" approves the command prefix (for example `find *`, `git grep *`, `gh pr view *`) for every agent in the instance and outranks these asks and the `-delete` deny, so answer these prompts with "Allow once". Patterns match the literal command text, so quoting, escaping, tabs, or line continuations inside a flag can evade them. The `find`, `rg`, and `gh` patterns have no path token, so pathless forms such as `find -exec ...` are covered. OpenCode checks only command nodes, so `export VAR=...; <allowed command>` can steer an allowed command (`GIT_EXTERNAL_DIFF`, `GIT_CONFIG_PARAMETERS`, `GH_BROWSER`) without a prompt, and shell redirection (`cat a > b`, above) can still write. Also not chased: behavior that user-level or repo config defines (textconv helpers run on plain `git diff`, `git log -p`, and `git show`; `diff.external` and diff-driver commands run on plain `git diff`, and on `log` and `show` only with `--ext-diff`; `git cat-file --textconv` and `--filters` do the same; `git status` fsmonitor), `rg -z` decompression helpers, and bundled short flags other than `gh -?w`.
- The adversarial agent has no bash allows at all: `bash: "*": ask`, `webfetch: ask`, `edit: deny`, `task: deny`, so every shell command, git included, prompts. It reads files through OpenCode's read, grep, and glob tools. Allowed read commands were dropped because they are not safe to auto-allow: OpenCode checks only command nodes by their raw text, so a redirect on a `{ }` or `( )` group or on a skipped command like `cd` can write a file without matching a redirect rule, and a config-defined helper (`git ls-files`, `git diff`, `git blame`, `git show`, `git status -v`) can run code on an allowed read. For its cleanup check it should run `git -c core.fsmonitor=false -c core.hooksPath=/dev/null status --porcelain`. The prompts are a human check only when you read them: a prompt answered with "Allow always" widens the allowlist for the session, so answer "Allow once". `edit: deny` blocks the edit tools, so it writes PoC scripts through bash, and "temp dir outside the repo, local targets only, scrubbed environment" is prompt-enforced. Even prompted, this is not a sandbox. The reviewer's and explorer's maps keep their own caveats (see the explorer and reviewer bullet above).
- An agent's bash map is applied after your global `permission` rules, so an agent-level `"*": allow` would re-allow commands you deny globally. The implementer, verifier, and side-fix maps hold only deny and ask entries. Bash is allowed by default, so they behave the same when you have no global rules.
- The implementer, verifier, and side-fix git denies each have a mirrored `"git -* <subcommand> *"` deny (side-fix's flag-specific denies mirror with the same flag), so forms with global options before the subcommand are denied too (`git -c k=v push`, `git -C <dir> push`, `git --git-dir=<dir> push`, `git --no-pager push`). The mirror matches the subcommand word as a whole word anywhere after a leading-dash option, so it also denies some harmless commands, such as `git -C <dir> log --grep push` or, in side-fix, `git -C <dir> commit -m "fix push retry"`. Run commits without global options. For a read, quote the argument (`--grep 'push'`) or write a path as `./<path>`; both keep any global options you need, such as guard flags.
- The mirrors still do not catch environment prefixes and wrappers (`GIT_DIR=<d> git push`, `env git push`, `command git push`), an absolute path (`/usr/bin/git push`), quoting or escaping (`"git" push`), tabs, repeated spaces (`git  push`), line continuations (`git -C <dir> \` then a newline, then `push`), `sh -c "..."`, scripts or interpreters that run git, or aliases (`git -c alias.p=push p`, or an alias already in your git config). A redirect written right against the subcommand or placed right after `git` also passes: `git -c k=v push>/dev/null` passes the mirrors (plain `git push>/dev/null` is still denied), and `git >/dev/null push` passes every rule. Side-fix's flag-specific denies catch the flag only right after the subcommand, so `git reset -q --hard`, `git add -- .`, and `git commit -m x -a` pass. No map denies other subcommands with the same effect, such as `git update-ref` and `git branch -f` (move refs), `git revert`, `git cherry-pick`, `git am`, and `git pull` (create commits, merge, or rebase), and `git send-pack` (pushes). These denies are a backstop, not a sandbox; the agent prompts remain the main guard.
- "Allow always" is instance-wide. OpenCode stores the approved prefix (the command's leading words by its arity table plus ` *`, such as `git push *`, `git -C *`, or `git -c *`) in one list for the running instance, and checks that list after the agent's own permission rules for every agent, so an approval outranks any agent's `ask` or `deny` for a matching command, not only the agent that asked. Examples: approving a `git push` prompt from any agent (a global `"git push*": ask` rule prompts in the orchestrator, because it has no bash map) approves `git push *` and lifts every `git push ...` deny in side-fix, implementer, and verifier, force-push and branch-deletion forms included (the `git -* push *` mirror still denies `git -C <dir> push`). Approving any agent's `git -C <dir> ...` prompt (side-fix's `git -C *` ask, or the explorer and reviewer `"*": ask` fallback) approves `git -C *` and lifts every deny for `git -C <dir> ...` forms, including `git -C <dir> push`. Approving a `git -c core.fsmonitor=false ... status` prompt approves `git -c *` and lifts every `git -c ...` deny, `git -c k=v push --force` included. When a redirect comes right after `git` (`git 2>/dev/null log`), the parser reads the following words as the redirect's target, so the stored prefix is `git *`, which lifts every git deny in every agent. Answer with "Allow once" unless you mean to allow that prefix for every agent. ([source][src]: `permission/index.ts:28-38`, `permission/index.ts:46-52`, `permission/index.ts:73`, `permission/index.ts:145-151`, `session/tools.ts:87`, `tool/shell.ts:408-409`, `permission/arity.ts:83`)
- A repo agent with the same name as a user-level one overrides it field by field instead of replacing it. OpenCode loads the global config dir first, then `.opencode` dirs, and merges agents with `mergeDeep`, so keys the repo file omits keep their user-level values and `permission` maps merge. (source: `config/paths.ts`, `config.ts`)

[src]: https://github.com/anomalyco/opencode/blob/388406238bd5ca15564a762840a2362c3a45bd9c
[src-v2]: https://github.com/anomalyco/opencode/blob/6608799d35d96c2821a48ac1d4b26a3b84b4e433

## Per-repo overrides

- A repo's `AGENTS.md` (or `CLAUDE.md` if there is no `AGENTS.md`) adds project rules on top of this one. OpenCode loads the global file, then the project file(s), then each `instructions` entry in `opencode.json`, and combines them. If they conflict and the A3 order doesn't settle it, the agent asks in the question batch, or parks the conflict after it. ([rules](https://opencode.ai/docs/rules/))
- A repo's `.opencode/agents/<name>.md` adds agents for that repo. One with the same name as a user-level agent overrides it for that repo, field by field, with `permission` maps merged (see Known gaps). A change to an override of an agent the loop spawns ships as a draft for your review, even when you approved it (orchestrator Phase 4).
- A repo override of a worker must keep the report contract: the `Status:` first line, the reviewer's finding IDs, the adversarial's verdict and `S` finding IDs, and the verifier's `Scope:` line and `FLAKY` result. Otherwise the orchestrator's exit tests cannot read its reports.
