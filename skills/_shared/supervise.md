# Supervision — §6b, §6e–§6f, §6i, §7, §8 (agent-independent)

Shared by every `dispatch-*` skill in this plugin. The sweep alternates between this file and the
calling skill's `references/driver.md`:

| § | Where | What |
| --- | --- | --- |
| §6a | driver | probe one lane's state from disk |
| §6b | here | poll the fleet, prove each lane is still yours |
| §6c | driver | classify each lane (the table, and its row order) |
| §6d | driver | steer a lane that is parked, blocked, or drifting |
| §6e | here | verify a lane that claims to be finished |
| §6f | here | publish a verified lane — push, open the PR |
| §6g | driver | handle a `blocked` lane / approval overlay |
| §6h | driver | compact or restart a lane |
| §6i | here | close the sweep |
| §7, §8 | here | arm the loop, report |

Reread §0 (in `SKILL.md`) before every sweep.

## The probe contract

§6c's table and everything below are written against the fields the driver's §6a promises. A driver
must report, per lane, at least:

| Field | Meaning | Missing when |
| --- | --- | --- |
| `session` | the agent's own id for this lane's live thread, as recorded in the state file | the agent exposes none — then the driver says what it uses for identity instead |
| `turn_state` | `working` (a turn is in flight) / `complete` (the last turn ended) / `unknown` | the agent's on-disk record cannot distinguish them |
| `used_pct` | percent of the context window in use, or `null` when the agent publishes no context window | — |
| `compactions` | how many times this thread has been compacted | — |
| `mtime` | last write to the lane's own on-disk record, for stall detection | — |
| `probe` | `ok` / `unavailable` with a reason | — |

Optional fields a driver may add — `goal_status`, `out_of_room`, `commits`, `todo` — are used only
by rows its own §6c table defines. **A field a driver does not report is never assumed**: a sweep
that cannot see `used_pct` does not guess one, it reports `null` and leans on `mtime` and
`turn_state` instead.

---

## §6b Poll the whole fleet in one call

`herdr agent list` returns every lane's `agent_status` and `state_change_seq` at once. Do not read
panes during a normal sweep — it costs context and tells you less than the lane's own on-disk record
does.

Names alone do not prove identity. Lane names carry no run id and are re-usable the moment an agent
exits, so a name in your state file can now belong to a later run's agent or to one the user started
by hand. Before steering any lane by name — prompt, send-keys, compact — confirm the session id
from `herdr agent get <lane>` (or the driver's own identity check, when its agent has no herdr-visible
session) still matches the one recorded in the state file (§6a). On a mismatch, stop steering and
diagnose — the three cases look alike and only one is yours to act on:

- the name resolves to an agent in some *other* workspace → a stranger re-claimed a released name.
  Never touch it (§0.4); surface the lane.
- the name resolves in the lane's own recorded pane, under a different id → not necessarily a
  stranger: the id also changes when a fresh thread starts in the same agent process (the driver's
  §6h restart recipe — or the user's own hand). If the state file shows a restart in flight, finish
  that recipe; otherwise surface it to the user instead of guessing, because steering someone else's
  thread and abandoning your own look identical from here.
- the name resolves to nothing → the agent exited. A non-terminal lane is relaunched per the
  driver's §5c on its recorded pane once it is back at a shell (§5a's checks apply again), then
  re-primed. The §6h restart recipe does not apply — it prompts a live agent, and there is none
  left here.

**Prompt guard — the one sanctioned exception to the no-pane-reads rule.** Immediately before *any*
input this sweep sends a lane — a slash command, a steering or continuation prompt — do ONE
`herdr agent read <lane> --source visible`. If a selection list or modal is parked there, send
nothing: a prompt submitted at a parked list presses its highlighted default. Resolve it first
(§6d/§6g).

---

## §6e Verify a finished lane

`agent_status` is not evidence, and neither is the lane's own `.dispatch/DONE`:

    git -C <checkout> status --porcelain              # must be empty
    git -C <checkout> log --oneline <base>..<branch>  # must be non-empty

plus the **acceptance criteria, re-run by you**: execute each §3-confirmed criterion's command from
TASK.md in the lane's checkout and require its expected outcome; judge the non-mechanical criteria
from the diff and `.dispatch/progress.md`. The lane already claims they pass — that claim is what
you are checking, not what you are accepting. Any agent-side "objective complete" signal the driver
reports is corroboration that the agent considers itself done, never a substitute for the criteria.
Also read `progress.md` to confirm every checklist item is ticked and to see whether the lane
recorded a deviation from the §3-confirmed plan. A recorded deviation is not a failure — judge the
result on its merits — but it must surface in the PR body (§6f) and the §8 report, never silently.
If a lane claims done but fails any of these, send a corrective prompt and keep it open. Never
report a half-finished lane as complete.

**Commit granularity is a report line, not a gate.** Compare the commit count against the checklist
length; a multi-item lane that produced a single commit ignored §5b's policy. Record it and surface it
in §8, but do **not** hold the lane back and never ask the agent to rewrite history to fix it —
amending or rebasing an already-verified branch risks the work itself, which costs far more than an
ugly history. Coarse commits are a note on the PR, not a reason to redo it.

Only a lane that passes **every** one of these becomes phase `verified`, and only a `verified` lane is
eligible for §6f. Nothing unverified is ever pushed — that is the whole reason this step runs first.

---

## §6f Publish a verified lane — push the branch, open the pull request

**You publish; the lane never does.** The agents are briefed "never push, never merge, never open a
PR" (§5b) precisely so that the one operation which leaves the worktree and reaches a shared remote
stays behind §6e's verification, in the hands of the one participant that has actually checked the
work. A lane that asks to push is still an anomaly, not a shortcut — surface it per §6g, never
approve it.

Publishing is **idempotent**. It runs on every sweep until it succeeds, so record `pushed_sha`,
`pr_number`, `pr_url` and `publish_attempts` in the state file and skip whatever is already done.

**1. Preconditions — degrade with a recorded reason, never with a guess.** Re-read the §2 preflight:

- no `origin` → nothing to push to; leave the lane `verified`, report it as local-only;
- `gh` missing or unauthenticated, or `--no-pr` → do step 2, then stop and print the exact
  `gh pr create` command for the user;
- the PR base is not a branch on origin → do step 2, but do **not** open a PR against a substituted
  base. A PR aimed at a branch the lane did not fork from shows a diff that is not the lane's work.
  Report it and print the command with the base left for the user to fill in.

**2. Push exactly one branch, by explicit refspec:**

    git -C <checkout> push -u origin refs/heads/<branch>:refs/heads/<branch>

The refspec is spelled out on purpose: never `--all`, never `--force` or `--force-with-lease`, never
the base branch, never a branch that is not in the state file. A rejected non-fast-forward push means
something else moved that branch — stop, record it, surface it; force-pushing here would destroy
whatever moved it. No `--no-verify`: pre-push hooks are part of the repo's checks.

**3. Reuse an existing PR before creating one:**

    gh pr list --repo <owner/repo> --head <branch> --state all --json number,url,state

Non-empty → record it and stop; the push in step 2 already updated it. `gh pr create` errors out on a
head branch that already has a PR, and a sweep that treats that error as failure will retry forever.

**4. Write the body to a file, then create:**

    gh pr create --repo <owner/repo> --base <pr-base> --head <branch> \
      --title '<conventional-commit title>' \
      --body-file ~/.claude/<skill-name>/<run-id>/<lane>-pr.md

- **Every argument is explicit, and that is load-bearing.** `gh pr create` prompts interactively for
  anything it cannot infer — and on a fork it asks which repo to target even when it can. An
  interactive prompt inside a non-interactive Bash call hangs until the timeout, so `--repo` (from
  `gh repo view --json nameWithOwner -q .nameWithOwner`), `--base`, `--head`, `--title` and
  `--body-file` are all mandatory here. Use `--body-file`, not `--body`: a multi-line body through the
  shell is where quoting breaks.
- **Ready for review is the default**, so no `--draft` flag: §6e has already established that the
  tree is clean, the commits are real and the repo's own checks pass, and a PR nobody can review
  without first clicking a button is a PR that sits. Opening ready does page reviewers and CODEOWNERS
  and does start CI, which is the point — but it also means the provenance line in the body is the
  only thing telling them an agent wrote this, so never drop it. Pass `--draft` to land the run
  quietly instead.
- **Title:** Conventional-Commits shaped, derived from the lane's objective — the commit subject when
  the lane produced exactly one commit, otherwise `<type>(<scope>): <objective>`, with the type
  agreeing with the branch's type prefix (a `feature` branch normalizes to `feat`). No run ids, lane
  names or raw branch strings in the title; those belong in the body.
- **Body, in English:** the checklist with its final tick state; a `Closes #<issue>` line when the
  lane carries an issue number, so the merge closes it; a short summary distilled from
  `.dispatch/progress.md` (that file is git-excluded per §5b, so the reviewer cannot open it — carry
  it over, do not link to it); the acceptance criteria with each one's verified outcome (§6e);
  any recorded deviation from the §3-confirmed plan, stated as such; and a provenance line naming
  the run id and the lane, and stating that **an agent wrote the code** while the orchestrator
  authored the plan and verified the result before push. Do not name the model, its version, or the
  approval posture the lane ran under — that detail belongs in the run's own state, not in a PR the
  whole repo reads. The reviewer should know what they are reading before they start reading it.

**5. Close the lane.** Record `pr_url` and set the phase to `published`. On failure, record the
stderr and bump `publish_attempts`; after 3 failed attempts stop retrying, record that as the
lane's degrade reason, leave it `verified`, and surface it with the exact command for the user to
run by hand. The recorded reason — from here or from step 1 — is what makes a `verified` lane
terminal for §7 and keeps §6c's `unpublished` row from re-matching it forever. A publish loop that
retries forever is worse than one that hands the command back.

---

## §6i Close the sweep

Rewrite the state file atomically (write `.tmp`, then `mv`). Emit **one line per lane** — lane,
phase, the driver's own status field, `used_pct`, compactions, turn state, PR number or `—` — never
raw JSON. This sweep runs many times; verbose output is what makes a long supervision run
unaffordable.

Escalation-type outcomes are recorded in the state file the first time (`escalated`, with reason).
Later sweeps re-surface them as one report line each, never as a fresh escalation, and §7 counts
such lanes as awaiting-user.

---

## §7 Arm the recurring loop

Unless `--no-loop`, after dispatch invoke the `loop` skill with `5m` and this skill's own `--resume`
invocation (the slash form that actually resolves here — `/dispatch-<agent> --resume`, or the
plugin-qualified `/herdr-dispatch:dispatch-<agent> --resume`), so supervision continues on a timer
without holding the session.

The timer is the **fallback** for a finished lane — that announces itself through the §5b
notify-back — but it is the **only** thing that recovers everything which arrives silently: stalls,
blocked panes, hot lanes needing compaction, a lane that stopped mid-work and needs the next nudge,
a rate limit that parked the agent, and any notify-back that was rejected while this session was
blocked. That is what keeps 5 minutes load-bearing. **The less autonomous the agent, the more the
timer *is* the engine** — the driver's §6c says which regime it is in.

Tell the user in Chinese that it is armed and how to stop it. Each tick is exactly one §6 sweep; a
tick landing right after a notify-back sweep is harmless — sweeps are idempotent. A `--resume` sweep
in a session with no armed loop re-arms it whenever non-terminal lanes remain (unless the run's
recorded flags say `--no-loop`) and re-records `orchestrator_pane` as the current pane — that is how
a run whose original session died gets its supervision back (§2).

`--no-loop` disarms only the timer; the briefs still carry the notify-back, but that signal alone is
lossy — a ring landing while this session is blocked (an open `AskUserQuestion`, a permission
prompt) is rejected as `agent_blocked` and never retried (§5b) — so when the user passed
`--no-loop`, tell them in Chinese that a missed ring is recovered only by re-running this skill with
`--resume` by hand, and that a lane which stops mid-work stays stopped until such a sweep nudges it.

Stop the loop once every lane is terminal or awaiting-user, and say which lanes wait on what.
Terminal means `published`, `failed`, user-paused, or `verified` with a recorded reason why §6f
could not publish it; awaiting-user means a recorded `escalated` the user has not yet answered — a
lane whose work is done but whose branch is still unpushed is **not** terminal, and the loop is what
eventually gets it out.

Do not busy-wait inside one turn instead: a sweep is cheap, but a blocking sleep loop burns the Bash
tool's ceiling and holds the session hostage.

---

## §8 Report

Per lane: 分支, checkout 路径, 状态, 已完成/剩余清单项, commit 数（§6e 判定粒度过粗的，在这里标注一下）,
**验收标准逐条结果**（每条：通过/未通过/无法机械验证时给依据）, 该 agent 的运行情况（driver 的 §6a
报的状态字段、token/用量、有无暂停 / 限流 / 阻塞经历；驱动方式降级过的写明原因）,
是否偏离已确认的实施计划（有则一句话说明偏在哪、为什么）, compaction 次数,
**PR 链接**（未开成的写明原因，别留空）.

Name the agent and version once at the top, and say plainly which side of the line each thing is on:
you **did** push the verified branches and open their PRs (§6f); you did **not** merge anything and
did not remove any workspace. Then print — do not run — the follow-up commands:

    herdr worktree remove --workspace <ws>             # destroys the checkout AND kills its agent process
    gh pr merge <pr-number> --squash --delete-branch   # per lane, after you have reviewed it
    git -C <repo> branch -d <branch>                   # only if the branch outlived the PR merge

The block is in that order on purpose: `worktree remove` runs **before** `gh pr merge
--delete-branch`, because git refuses to delete a local branch that is still checked out in a
worktree, and the merge command will report a partial success. Warn that `worktree remove` discards
uncommitted work in that checkout, and remember each lane may have **two** workspace ids to clean up
(§4). Only when the run used `--draft`, print `gh pr ready <pr-number>` ahead of the merge command:
GitHub refuses to merge a draft outright (`Pull request is not mergeable: it is in draft state`).
