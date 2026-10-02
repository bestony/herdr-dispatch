---
name: dispatch-opencode
description: Dispatch one or more tasks to opencode agents in herdr workspaces — the orchestrator designs each lane's plan and acceptance criteria, opencode executes them turn by turn under the supervision loop — then push each lane and open its pull request. Use when the user asks to dispatch/fan out/派发 tasks to opencode agents or herdr lanes, to run several opencode tasks in parallel worktrees, or to resume supervision of an existing dispatch-opencode run (`--resume`).
argument-hint: '<task-1>; <task-2>; … [--lanes N] [--base <ref>] [--no-yolo] [--compact-at N] [--draft] [--no-pr] [--resume] [--no-loop]'
allowed-tools: Read, Write, Edit, Glob, Grep, TodoWrite, AskUserQuestion, Skill, Bash(herdr:*), Bash(git:*), Bash(gh:*), Bash(jq:*), Bash(python3:*), Bash(opencode:*), Bash(mkdir:*), Bash(ls:*), Bash(grep:*), Bash(test:*), Bash(date:*), Bash(mv:*), Bash(cat:*), Bash(printf:*)
---

# Dispatch tasks to opencode agents running under herdr

The **raw request** is the text passed to this skill — its arguments, or, when it was invoked with
none, the task text in the user's own message.

You are an **orchestrator**. You do not implement the tasks yourself. You split the request into
lanes, give each lane its own herdr workspace and its own opencode agent, and then keep those agents
alive and moving until every lane genuinely finishes. **You do the planning; opencode does the
editing** — and, unlike the goal-mode dispatchers in this plugin, **you also do the driving**: opencode has no autonomous
objective mode, so it stops at the end of every turn and the supervision sweep's continuation prompt
is what moves the lane forward (§6c `idle_incomplete`, §6d). A lane is not finished when its agent
stops — stopping is what opencode does after every turn — it is finished when its work is verified,
its branch is pushed, and its pull request is open (§6f).

## How to read this skill

The procedure is split across four files, and the section numbers (§0–§8) are continuous across all
of them, so a cross-reference means the same thing wherever you are:

| Sections | File | Read it |
| --- | --- | --- |
| §0 invariants, §1 gate and parse | this file | always, and §0 again at the start of every sweep |
| §2–§4, §5b | `references/plan.md` | on a fresh dispatch, before creating anything |
| §5a, §5c, §6a, §6c, §6d, §6g, §6h | `references/driver.md` | everything opencode-specific: launch, probe, classify, steer |
| §6b, §6e, §6f, §6i, §6j, §7, §8 | `references/supervise.md` | before the first supervision sweep |

`references/plan.md` and `references/supervise.md` are symlinks into the plugin's `skills/_shared/`:
the shared halves stay single-sourced while still resolving when this skill's directory is copied on
its own, which is how the [skills.sh](https://skills.sh) installer places it. Both are read by path,
not invoked.

## §0 Invariants — reread every sweep, never work from memory

1. **The state file is the only truth.** `~/.claude/dispatch-opencode/<run-id>/state.json`. Begin
   every sweep by reading it; your conversation memory may have been compacted away. Rewrite it
   atomically (write `.tmp`, then `mv`) at the end of every sweep.
2. **Judge a lane from disk, not from the screen.** Opencode records sessions and messages in a
   SQLite database (`~/.local/share/opencode/opencode.db` on this machine; confirm with
   `opencode debug paths`), where each assistant message carries its token counts, its completion
   timestamp and a `finish` reason. Terminal output is the fallback, never the primary signal.
3. **Read that database read-only, and never as `immutable`.** Open it as
   `file:<path>?mode=ro` — opencode is writing to it concurrently in WAL mode, so a plain read-only
   connection is correct and safe, while `immutable=1` promises sqlite the file cannot change and
   will hand you stale or torn data. Never write to it, never `VACUUM`, never hold a long
   transaction, and never `opencode session delete`.
4. **A stopped lane is the normal state, not a finished one.** Opencode has no goal mode: it answers,
   runs its tools, and stops. So `agent_status: done` / `idle` says only "the turn ended" — every
   unfinished lane in this run will look exactly like that between sweeps. Completion is
   `.dispatch/DONE` plus §6e's own verification, never the agent's status, and never a lane's
   notify-back or help ring (§5b), which is a doorbell that starts a sweep sooner and nothing more.
5. **The supervision loop is load-bearing here, not a safety net.** With no autonomous continuation,
   a lane advances only when a sweep prompts it. `--no-loop` therefore means "this run stops moving
   the moment I stop watching" — say so in Chinese when the user passes it (§7).
6. **Never touch what you did not create.** Act only on ids recorded in the state file. Never
   `herdr server stop`. Never `herdr agent focus` / `workspace focus` — it steals the human's UI focus
   and silently flips `done` to `idle`, destroying your own signal. Opencode's database is shared by
   every project on this machine: a query that is not filtered to this lane's checkout is a bug.
7. **Never destroy work; publish only what you verified.** Finishing a lane means pushing *its own*
   branch and opening a PR for it (§6f) — both additive and reversible, and both yours to do, never
   the lane's. Everything else stays forbidden: never force-push (`--force`, `--force-with-lease`),
   never push the base branch or any branch absent from the state file, never merge, never
   `worktree remove`. Print those commands and let the user run them: `worktree remove` kills the
   running opencode process and deletes uncommitted changes even without `--force`.

---

## §1 Gate and parse

Run `test "${HERDR_ENV:-}" = 1`, `test -n "${HERDR_PANE_ID:-}"` and `herdr agent list`. If
`HERDR_ENV` or `HERDR_PANE_ID` is unset or the CLI cannot reach the socket, stop and tell the user in
Chinese that this session is not inside a herdr pane, so there is nothing to dispatch into — an
exported `HERDR_ENV` alone can pass in a non-pane shell, and a run recorded without its pane id has
no working notify-back. Do not install or launch herdr, and do not run opencode yourself.

Parse flags from the raw request; everything else is task text.

| Flag | Meaning | Default |
| --- | --- | --- |
| `--lanes N` | cap on concurrent lanes, 1–16 | 16 |
| `--base <ref>` | base ref for lane branches | `origin/<current>` if it exists, else current branch |
| `--no-yolo` | launch without `--auto`, so tool calls that need permission prompt in the pane | off — **`--auto` is the default** |
| `--yolo` | accepted and explicit, but redundant: this is already the default | on |
| `--compact-at N` | send `/compact` once a lane's last turn reports ≥ N context tokens | unset — **no automatic compaction**, see below |
| `--draft` | open pull requests as drafts instead of ready for review | off — **ready for review is the default** |
| `--no-pr` | push each verified lane but stop there; print the `gh pr create` command instead | off |
| `--resume` | skip §2–§5 (this gate and parse still run); run ONE supervision sweep over the existing state file | off |
| `--no-loop` | do not arm the recurring supervision loop after dispatch | off |

**Why `--compact-at` is a flag instead of a percentage.** Every sibling dispatcher compacts at a
percentage of the context window, because its agent publishes that window on disk. Opencode's
records give exact token counts per turn but no context window to divide them by, so this skill
reports **absolute tokens** and refuses to invent a denominator. Left unset, lanes rely on
opencode's own context handling and the sweep only reports the number; set it (e.g.
`--compact-at 150000`) and §6c's `hot` row turns on. Tell the user which of the two is in force in
§3's summary.

**Every lane gets its own git worktree. This is not a flag and there is no opt-out.** With `--auto`
approving tool calls by default (§5c), the worktree boundary is the only thing left keeping one
lane's mistakes out of the other lanes and out of the user's own checkout. If the request contains
`--no-worktree`, **stop before creating anything**: say in Chinese that this skill always isolates
lanes in worktrees and that the flag no longer exists, and ask the user to re-run without it.

With `--resume`, first locate the run — conversation memory may be gone (§0.1): scan
`~/.claude/dispatch-opencode/*/state.json` for runs whose repo matches the cwd and that still hold
non-terminal lanes; one match sweeps it, several means ask the user which, none means say so and
stop. Then read `references/driver.md` and `references/supervise.md` and go to §6. With no task text
and no `--resume`, ask the user in Chinese what to dispatch, and stop.

Otherwise — a fresh dispatch — read `references/plan.md` now and continue at §2.
