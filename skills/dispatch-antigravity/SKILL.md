---
name: dispatch-antigravity
description: Dispatch one or more tasks to Antigravity CLI (agy) agents in herdr workspaces — the orchestrator designs each lane's plan and acceptance criteria, agy pursues them in goal mode — supervise them to completion, then push each lane and open its pull request. Use when the user asks to dispatch/fan out/派发 tasks to antigravity or agy agents or herdr lanes, to run several antigravity tasks in parallel worktrees, or to resume supervision of an existing dispatch-antigravity run (`--resume`).
argument-hint: '<task-1>; <task-2>; … [--lanes N] [--base <ref>] [--no-yolo] [--yolo] [--draft] [--no-pr] [--resume] [--no-loop]'
allowed-tools: Read, Write, Edit, Glob, Grep, TodoWrite, AskUserQuestion, Skill, Bash(herdr:*), Bash(git:*), Bash(gh:*), Bash(jq:*), Bash(python3:*), Bash(agy:*), Bash(mkdir:*), Bash(ls:*), Bash(grep:*), Bash(test:*), Bash(date:*), Bash(mv:*), Bash(cat:*), Bash(printf:*)
---

# Dispatch tasks to Antigravity CLI agents running under herdr

The **raw request** is the text passed to this skill — its arguments, or, when it was invoked with
none, the task text in the user's own message.

You are an **orchestrator**. You do not implement the tasks yourself. You split the request into
lanes, give each lane its own herdr workspace and its own Antigravity CLI agent (`agy`, herdr kind
`agy`), and then keep those agents alive and moving until every lane genuinely finishes. **You do
the planning; agy does the work**: each lane gets a plan and acceptance criteria you author and the
user confirms (§3), and runs as an agy **goal** (`/goal`, §5c) — an objective agy keeps pursuing
inside one long run until it marks the goal complete itself. A lane is not finished when its agent
stops — it is finished when its work is verified, its branch is pushed, and its pull request is
open (§6f).

## How to read this skill

The procedure is split across four files, and the section numbers (§0–§8) are continuous across all
of them, so a cross-reference means the same thing wherever you are:

| Sections | File | Read it |
| --- | --- | --- |
| §0 invariants, §1 gate and parse | this file | always, and §0 again at the start of every sweep |
| §2–§4, §5b | `references/plan.md` | on a fresh dispatch, before creating anything |
| §5a, §5c, §6a, §6c, §6d, §6g, §6h | `references/driver.md` | everything agy-specific: launch, probe, classify, steer |
| §6b, §6e, §6f, §6i, §6j, §7, §8 | `references/supervise.md` | before the first supervision sweep |

`references/plan.md` and `references/supervise.md` are symlinks into the plugin's `skills/_shared/`:
the shared halves stay single-sourced while still resolving when this skill's directory is copied on
its own, which is how the [skills.sh](https://skills.sh) installer places it. Both are read by path,
not invoked.

## §0 Invariants — reread every sweep, never work from memory

1. **The state file is the only truth.** `~/.claude/dispatch-antigravity/<run-id>/state.json`. Begin
   every sweep by reading it; your conversation memory may have been compacted away. Rewrite it
   atomically (write `.tmp`, then `mv`) at the end of every sweep.
2. **Judge a lane from disk, not from the screen.** agy keeps one row per conversation in
   `~/.gemini/antigravity-cli/conversation_summaries.db` (run status, step count, last
   modification, workspaces) and one transcript per conversation under
   `~/.gemini/antigravity-cli/brain/<conversation-id>/.system_generated/logs/transcript.jsonl`
   (steps, compaction checkpoints, quota errors, the goal-complete marker). Terminal output is the
   fallback, never the primary signal.
3. **Read that database read-only, and never as `immutable`.** Open it as `file:<path>?mode=ro` — agy
   writes to it concurrently in WAL mode, so a plain read-only connection is correct, while
   `immutable=1` returns stale data. One verified exception: once every agy process has exited, agy
   checkpoints the WAL and removes `-wal` and `-shm`, and then a `mode=ro` open fails with `unable
   to open database file`. Only in that case — the open failed **and** no `-wal` file exists, so
   no writer is active — retry once with `immutable=1`. Never write to it, never `VACUUM`, never
   hold a long transaction. The database and `brain/` are shared by every agy session on this machine: a query
   that is not filtered to this lane's conversation id is a bug.
4. **`agent_status` alone never means "finished", and it can lie outright.** herdr reports agy as
   `idle` with `interactive_ready: true` while the workspace-trust dialog is still on screen
   (verified), and `done` when a run ends for any reason — goal complete, interrupt, quota error.
   Always corroborate (§6). The same goes for a lane's notify-back or help ring (§5b): it is a
   doorbell that starts a sweep sooner, never evidence that skips §6e.
5. **agy has no goal-status command.** `/goal` takes only an objective; a bare `/goal` does not
   report state, it re-engages pursuit (observed). Goal state is read from disk instead (§6a): the
   last `/goal` input and the `<!-- GOAL_COMPLETE -->` marker agy writes when it judges the goal
   met. Never send `/goal` just to look.
6. **Never touch what you did not create.** Act only on ids recorded in the state file. Never
   `herdr server stop`. Never `herdr agent focus` / `workspace focus` — it steals the human's UI focus
   and silently flips `done` to `idle`, destroying your own signal. Never edit agy's own config
   (`~/.gemini/antigravity-cli/settings.json`, `~/.gemini/config/projects/*.json`): the posture it
   sets is the user's, and §3 reports it instead of changing it.
7. **Never destroy work; publish only what you verified.** Finishing a lane means pushing *its own*
   branch and opening a PR for it (§6f) — both additive and reversible, and both yours to do, never
   the lane's. Everything else stays forbidden: never force-push (`--force`, `--force-with-lease`),
   never push the base branch or any branch absent from the state file, never merge, never
   `worktree remove`. Print those commands and let the user run them: `worktree remove` kills the
   running agy process and deletes uncommitted changes even without `--force`.
8. **A lane's transcript grows without bound.** Never parse one whole — read only its tail, and
   never open `transcript_full.jsonl`.

---

## §1 Gate and parse

Run `test "${HERDR_ENV:-}" = 1`, `test -n "${HERDR_PANE_ID:-}"` and `herdr agent list`. If
`HERDR_ENV` or `HERDR_PANE_ID` is unset or the CLI cannot reach the socket, stop and tell the user in
Chinese that this session is not inside a herdr pane, so there is nothing to dispatch into — an
exported `HERDR_ENV` alone can pass in a non-pane shell, and a run recorded without its pane id has
no working notify-back. Do not install or launch herdr, and do not run agy yourself.

Parse flags from the raw request; everything else is task text.

| Flag | Meaning | Default |
| --- | --- | --- |
| `--lanes N` | cap on concurrent lanes, 1–16 | 16 |
| `--base <ref>` | base ref for lane branches | `origin/<current>` if it exists, else current branch |
| `--no-yolo` | launch without `--dangerously-skip-permissions`, so tool calls follow the user's own agy permission config | off — **yolo is the default** |
| `--yolo` | accepted and explicit, but redundant: this is already the default | on |
| `--draft` | open pull requests as drafts instead of ready for review | off — **ready for review is the default** |
| `--no-pr` | push each verified lane but stop there; print the `gh pr create` command instead | off |
| `--resume` | skip §2–§5 (this gate and parse still run); run ONE supervision sweep over the existing state file | off |
| `--no-loop` | do not arm the recurring supervision loop after dispatch | off |

**`--no-yolo` cannot be forced from the command line.** agy has no flag that turns approval
prompting *on*: its posture comes from `toolPermission` in `~/.gemini/antigravity-cli/settings.json`
and, with higher precedence, `permissionPreset` / `autoExecutionPolicy` in the active project file
under `~/.gemini/config/projects/`. If those already auto-approve (`always-proceed`,
`AGENT_PERMISSION_PRESET_TURBO`, `CASCADE_COMMANDS_AUTO_EXECUTION_EAGER`), dropping the bypass flag
changes nothing and the lane still auto-approves. Read both files before §3 and describe what will
actually happen; under `--no-yolo` with an auto-approving config, say so plainly in §3 and let the
user decide whether to change their own config first — never edit it for them (§0.6).

**Every lane gets its own git worktree. This is not a flag and there is no opt-out.** With approvals
bypassed by default (§5c), the worktree boundary is the only thing left keeping one lane's mistakes
out of the other lanes and out of the user's own checkout — and agy's default project can point it
at a directory outside the checkout (§5a), so the brief's path discipline matters even more here.
If the request contains `--no-worktree`, **stop before creating anything**: say in Chinese that this
skill always isolates lanes in worktrees and that the flag no longer exists, and ask the user to
re-run without it.

With `--resume`, first locate the run — conversation memory may be gone (§0.1): scan
`~/.claude/dispatch-antigravity/*/state.json` for runs whose repo matches the cwd and that still hold
non-terminal lanes; one match sweeps it, several means ask the user which, none means say so and
stop. Then read `references/driver.md` and `references/supervise.md` and go to §6. With no task text
and no `--resume`, ask the user in Chinese what to dispatch, and stop.

Otherwise — a fresh dispatch — read `references/plan.md` now and continue at §2.
