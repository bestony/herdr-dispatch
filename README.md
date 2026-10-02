# herdr-dispatch

A Claude Code plugin that turns one Claude session into an **orchestrator** for a fleet of coding
agents.

You describe the work. Claude splits it into *lanes*, gives each lane its own git worktree and its
own agent — **codex**, **grok** or **opencode** — running inside a [herdr](https://herdr.dev)
workspace, then supervises every lane on a timer until its work is verified, its branch is pushed
and its pull request is open.

The division of labour is deliberate:

- **Claude plans.** It reads the repo, writes each lane's implementation plan and acceptance
  criteria, and asks you to confirm them before anything is created.
- **The agent executes.** Each lane gets that plan verbatim as a brief and works through it.
- **Claude publishes.** Lanes are briefed never to push, merge, or open a PR. That one operation
  which reaches a shared remote stays behind Claude's own verification of the acceptance criteria.
- **Claude answers.** A lane that is stuck asks for help instead of guessing (see
  [Asking for help](#asking-for-help)), and Claude answers from the plan it wrote.

---

## Requirements

| What | Why |
| --- | --- |
| [**herdr**](https://herdr.dev) | Creates the worktrees, workspaces and panes each lane lives in. `brew install herdr` — developed against 0.9.0 |
| **A herdr pane** | The skills refuse to run outside one — there would be nothing to dispatch into |
| **A git repository** | Every lane is a linked worktree branched off a base ref. Run from the main checkout, not a linked worktree |
| **At least one agent CLI** | `codex`, `grok`, or `opencode` — whichever dispatcher you invoke, resolvable from the pane's own login shell |
| **`python3`, `jq`** | Used to probe each lane's on-disk state |
| **`gh`, authenticated** | Optional. Without it lanes are pushed but the `gh pr create` command is printed for you to run |
| **`origin` remote** | Optional. Without it lanes stay local and are reported as such |

Nothing is installed for you. If an agent CLI or `gh` is missing, the run says so up front and
degrades — it never installs, authenticates, or creates a missing branch on your behalf.

## Installation

As a plugin — the recommended path in Claude Code:

```
/plugin marketplace add bestony/herdr-dispatch
/plugin install herdr-dispatch@herdr-dispatch
```

If the install summary says `Run /reload-plugins to activate.`, run that.

### Via skills.sh

The same three skills are also published to the [skills.sh](https://skills.sh) registry, for installs
that are not plugin-managed — or for handing them to another agent that reads `SKILL.md`:

```bash
npx skills add bestony/herdr-dispatch --list          # what is in the repo
npx skills add bestony/herdr-dispatch                 # install into ./.claude/skills/
npx skills add bestony/herdr-dispatch -g              # install into ~/.claude/skills/
npx skills add bestony/herdr-dispatch -s dispatch-codex -a claude-code -y
```

Each installed skill directory is self-contained, so `/dispatch-<agent>` works from `.claude/skills/`
with no plugin installed. Prefer the plugin when you are on Claude Code: `npx skills update` replaces
the skill directories wholesale, and only the plugin carries a version and the marketplace entry.

<details>
<summary>Local development install</summary>

```bash
git clone https://github.com/bestony/herdr-dispatch.git
claude --plugin-dir ./herdr-dispatch
```

Run `/reload-plugins` to pick up edits without restarting.

</details>

## Quick start

Start Claude Code **inside a herdr pane**, in the main checkout of the repo you want the work done
in, then:

```
/dispatch-codex Add an owner filter to the repos list; fix the websocket reconnect timeout
```

What happens next:

1. Claude fetches `origin`, picks a base ref, and checks whether it will be able to push and open PRs
   at the end — so you learn about a missing `gh` login *now*, not after ten lanes have run.
2. It groups your tasks into lanes, and for each lane writes an implementation plan grounded in the
   actual code plus acceptance criteria built from the repo's real check commands.
3. It shows you the lane table plus each lane's plan and acceptance criteria, and asks once whether
   to dispatch. **Nothing is created before you answer.**
4. On confirmation: one worktree + workspace + agent per lane, each primed with its own brief.
5. A supervision loop runs every 5 minutes. It probes each lane from disk, nudges the stopped ones,
   compacts the hot ones, resolves parked dialogs, and re-runs the acceptance criteria itself on any
   lane claiming to be done.
6. Each lane that passes gets pushed on its own branch and opened as a pull request.

The plan, the questions and the final report are in Chinese; the briefs, commits and PR bodies are
in English.

## The three dispatchers

Same procedure, same flags, different agent underneath. Pick by which CLI you have and how
autonomous you want the lanes to be.

| Skill | Agent | How a lane is driven |
| --- | --- | --- |
| `/dispatch-codex` | codex | **Goal mode** (`/goal`). Codex auto-continues toward the objective across turns; the supervision loop is a repair path |
| `/dispatch-grok` | grok (Grok Build) | **Goal mode** when `[goal] enabled = true` in `~/.grok/config.toml`, otherwise one-shot + nudges. The dispatcher reads your config and tells you which regime is in force |
| `/dispatch-opencode` | opencode | **Nudge-driven.** Opencode has no goal mode — it stops after every turn, so the loop's continuation prompt *is* the engine. Expect roughly one turn per sweep interval |

Plugin-qualified forms work too: `/herdr-dispatch:dispatch-codex`.

## Flags

```
/dispatch-<agent> <task-1>; <task-2>; … [flags]
```

Everything that is not a flag is task text. Tasks split on numbered items, newlines, or `;`.

| Flag | Meaning | Default |
| --- | --- | --- |
| `--lanes N` | Cap on concurrent lanes, 1–16 | `16` |
| `--base <ref>` | Base ref for lane branches | `origin/<current>` if it exists, else the current branch |
| `--no-yolo` | Run lanes under normal approval prompting instead of bypassing it | off — **yolo is the default** |
| `--yolo` | Accepted, but redundant — already the default | on |
| `--draft` | Open pull requests as drafts | off — **ready for review** |
| `--no-pr` | Push each verified lane, then print the `gh pr create` command instead of running it | off |
| `--resume` | Skip planning; run **one** supervision sweep over an existing run | off |
| `--no-loop` | Do not arm the recurring supervision loop after dispatch | off |
| `--compact-at N` | *(opencode only)* Send `/compact` once a lane's last turn reports ≥ N context tokens | unset — no automatic compaction |

`--compact-at` takes absolute tokens rather than a percentage because opencode publishes no context
window on disk, and the dispatcher refuses to invent a denominator. The codex and grok dispatchers
compact on a percentage they can actually read.

### Two things that are not flags

**Worktree isolation has no opt-out.** With approvals bypassed by default, the worktree boundary is
the only thing keeping one lane's mistakes out of the other lanes and out of your own checkout.
Passing `--no-worktree` stops the run before anything is created.

**Yolo is the default, and it is a real trade.** An approval overlay stalls an unattended lane until
the next sweep notices it, so lanes launch with approvals bypassed:

| Agent | Launch flag | What it gives up |
| --- | --- | --- |
| codex | `--dangerously-bypass-approvals-and-sandbox` | Approvals **and** the sandbox |
| grok | `--permission-mode bypassPermissions` | Approvals only — your `--sandbox` profile is left untouched, so if you have it set to `off`, the worktree is the only boundary |
| opencode | `--auto` | Approvals only, and opencode's own help calls it dangerous. There is no sandbox either way |

Pass `--no-yolo` to keep prompting; the loop then resolves each overlay itself, at the cost of a lane
pausing between sweeps. Under `--no-yolo` the flag is passed through *explicitly* rather than merely
omitted, because a global `yolo = true` in the agent's own config would otherwise leave the lane
auto-approving while the plan summary claimed the opposite.

## Resuming and stopping

The supervision loop lives in your Claude session. If the session ends, the timer dies with it —
lanes keep working, but nobody is watching. Bring supervision back with:

```
/dispatch-codex --resume
```

A `--resume` sweep finds the run by scanning state files for one matching your cwd, runs one full
sweep, re-arms the timer, and re-records the pane to notify. Conversation memory is never trusted:
**the state file is the only truth**, and every sweep starts by re-reading it.

The loop stops on its own once every lane is terminal — published, failed, paused by you, or
verified-but-unpublishable with a recorded reason — and tells you which lanes are waiting on what.

## Asking for help

A lane that is stuck — the plan contradicts the code, the task reads two ways, an acceptance
criterion cannot be met as written, or the same failure survived two fix attempts — does not guess.
It writes `.dispatch/help/H-<n>.request.md` with three required parts:

- **Problem** — what is wrong, with the exact error, file and plan step;
- **Expected solution** — what the lane thinks the right fix or decision is, and why;
- **Tried** — each approach it already tried, and what happened.

It then rings the orchestrator pane and picks a mode. **`continuing`** (preferred) means it moves on
to checklist items that do not depend on the answer, so the lane never sits idle. **`waiting`** means
every remaining item depends on the answer, so it ends its turn and stops.

On the next sweep Claude reads the request, investigates the repo read-only, and writes
`H-<n>.answer.md`. A lane in mid-turn picks the answer up at its next checklist item; a lane at rest
gets a prompt pointing at the file. Questions that need a change to the acceptance criteria, a wider
scope, a secret or a product decision are escalated to you instead, and the lane waits (or keeps
working on independent items) until you reply. A waiting lane is never nudged or flagged as stalled.
Answers that change a plan step are reported as deviations in the PR body and the final report.

## What a run leaves on disk

```
~/.claude/dispatch-<agent>/
├── bin/lane_state.py       # the on-disk probe helper — written once, shared by every run
└── <run-id>/
    ├── state.json          # the run's only source of truth, rewritten atomically each sweep
    └── <lane>-pr.md        # PR body, written to a file so multi-line quoting can't break

<lane-checkout>/.dispatch/    # git-excluded via .git/info/exclude
├── TASK.md             # objective, confirmed plan, checklist, acceptance criteria, boundaries
├── progress.md         # rewritten by the lane after every checklist item
├── help/               # H-<n>.request.md from the lane, H-<n>.answer.md from Claude
└── DONE                # written only when the lane believes every criterion holds
```

`.dispatch/progress.md` is the lane's memory across compaction, and it is what the next continuation
prompt is built from. `DONE` is a claim, not a verdict — Claude re-runs the acceptance criteria
itself before believing it.

## What it will not do

These are invariants, not defaults:

- **Never force-push**, never push the base branch, never push a branch absent from the state file.
- **Never merge**, never `worktree remove`, never delete a session or a worktree. Those commands are
  *printed* at the end for you to run.
- **Never touch what it did not create.** Lane names carry no run id and are reusable, so identity is
  re-confirmed from the recorded session id before any lane is steered.
- **Never approve a lane's request to push or open a PR.** A lane asking for that misread its brief;
  it gets surfaced to you, not approved.
- **Never report a half-finished lane as complete.** A lane is done when its tree is clean, its
  commits are real, and every acceptance criterion has been re-verified by the orchestrator.

## Cleanup after a run

The final report prints these — it does not run them. Order matters: `worktree remove` comes first,
because git refuses to delete a local branch still checked out in a worktree.

```bash
herdr worktree remove --workspace <ws>            # destroys the checkout AND kills its agent
gh pr merge <pr-number> --squash --delete-branch  # per lane, after you have reviewed it
git -C <repo> branch -d <branch>                  # only if the branch outlived the merge
```

`herdr worktree create` produces **two** workspaces per lane — the linked worktree and one for the
base repo — so expect two ids to clean up. And `worktree remove` discards uncommitted work in that
checkout even without `--force`.

## Repository layout

```
.claude-plugin/
├── plugin.json               # plugin manifest
└── marketplace.json          # lets this repo host itself as a marketplace
skills/
├── _shared/
│   ├── plan.md               # §2–§4, §5b — bookkeeping, lane plan, workspaces, brief
│   └── supervise.md          # §6b, §6e–§6f, §6i, §7, §8 — poll, verify, publish, loop, report
├── dispatch-codex/
│   ├── SKILL.md              # §0 invariants, §1 gate and parse
│   └── references/
│       ├── driver.md         # §5a, §5c, §6a, §6c, §6d, §6g, §6h — everything codex-specific
│       ├── plan.md           # symlink → ../../_shared/plan.md
│       └── supervise.md      # symlink → ../../_shared/supervise.md
├── dispatch-grok/            # same shape
└── dispatch-opencode/        # same shape
```

Each dispatcher is four files with **continuous section numbers §0–§8**, so a cross-reference means
the same thing wherever you are. Adding a fourth agent means writing one `SKILL.md` and one
`driver.md`, plus the two `references/` symlinks; `_shared/` itself is reused untouched.

The symlinks exist because the skills.sh installer copies a skill directory **on its own** — its
`--copy` mode and its canonical staging both dereference symlinks, so `../_shared/` would otherwise
arrive as a dangling path and every §2–§4 and §6b–§8 reference in the installed skill would point at
nothing. They keep the shared halves single-sourced in the repo while resolving after install.
`ponytail:` this depends on the installer dereferencing symlinks rather than preserving them; if a
future CLI version changes that, the fix is to copy the two files into each `references/`.

## License

MIT — see [LICENSE](LICENSE).
