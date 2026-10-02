# Codex driver — §5a, §5c, §6a, §6c, §6d, §6g, §6h

Everything in `dispatch-codex` that depends on codex itself. The agent-independent halves are
`references/plan.md` (§2–§4, §5b) and `references/supervise.md` (§6b, §6e, §6f, §6i, §7, §8);
§0 and §1 are in `../SKILL.md`.

Claims marked **(verified)** were checked against codex **0.151.0** on this machine. §2 records the
version it actually finds; a mismatch is a report line, not an abort.

---

## §5a Pre-flight the pane

A fresh worktree is not a working environment: no `node_modules/`, no `.env`, no `.venv`, because
those are untracked or ignored.

    herdr pane send-text <pane> 'codex --version'
    herdr pane send-keys <pane> enter

Run the version, not `command -v`: on an asdf machine the shim exists and resolves even when the
worktree has no `.tool-versions` entry for the plugin, and then the launch itself dies with
`No version is set for command codex` and `agent start` times out (verified). `codex --version`
exercises the real resolution path.

There is no `herdr pane run`: `pane send-text` only types the text, and `pane send-keys <pane> enter`
submits it. Then read the result with `herdr pane read <pane> --source visible`. On herdr 0.9.3,
`herdr pane wait-output` searches the output already on screen before it polls, so it also finds a
command that already finished (verified). Do not wait on the tool name alone: the echoed command
line contains it too.

- `codex` not found, or the version refuses to resolve → stop that lane and report it (for the asdf
  case, suggest the user add the tool to the repo's `.tool-versions` or set a global default).
  `agent start --kind codex` resolves the executable from the pane's own login shell and args cannot
  redirect it.
- Dependencies missing → run the repo's install command in the pane and let it finish **before**
  starting the agent; `agent start` needs a pane at an idle interactive prompt.
- `.env` or other secrets missing → **ask the user** whether to symlink them. Never copy secrets.

Then write the brief (§5b in `references/plan.md`) before launching.

---

## §5c Launch codex and set the goal

Launch codex with **no positional prompt**. The lane is driven by a codex **goal**, and a goal is
set by slash command, which cannot ride along on argv — codex itself refuses with `The session
must start before you can set a goal`. The `/goal` call below goes through `herdr agent prompt`,
and only after the pane has been verified ready — a startup modal eats whatever input reaches it,
so nothing is sent to a pane whose composer you have not seen:

    herdr agent start <lane> --kind codex --pane <pane> --timeout 120000 -- \
      --no-alt-screen -c 'tui.status_line=["context-remaining"]' \
      --dangerously-bypass-approvals-and-sandbox

Why each part:

- `--no-alt-screen` puts codex inline, so `agent read --source recent-unwrapped` reaches real
  scrollback and reads keep working while the agent is busy. Detection still recognizes codex (it
  matches on the terminal title, not the screen).
- `-c 'tui.status_line=["context-remaining"]'` forces the context indicator to a known wording. The
  CLI array replaces the config array wholesale, so the row reads `Context N% left` — percent
  **remaining**. Without it, this machine's config renders `Context N% used` and a `context left`
  grep silently finds nothing.
- `--timeout 120000` because codex cold start is 6–9 s and MCP boot can be much longer; the 30 s
  default is too tight.
- Never pass `--full-auto` — removed in codex 0.151.0, it errors at launch.
- `--dangerously-bypass-approvals-and-sandbox` is **on by default**: lanes run unattended and an
  approval overlay stalls a lane until the next sweep notices it, so the sandbox is traded away for
  throughput. Drop this flag **only** when the user passed `--no-yolo`, in which case the lane inherits
  the user's own approval config and every overlay is handled in §6g. The flag composes with goal
  mode (verified): the banner reads `permissions: YOLO mode`, the rollout's
  `thread_settings_applied` event shows `approval_policy: never` with a disabled permission
  profile, and a full goal pursuit — reads, edits, commits — runs without a single overlay.
- **Under `--no-yolo`, read `~/.codex/config.toml` first.** If it sets `approval_policy = "never"`
  or `sandbox_mode = "danger-full-access"`, merely dropping the bypass flag still launches the lane
  in YOLO mode (verified on 0.159.3: the banner still reads `permissions: YOLO mode`), and §3 would
  promise a posture the lane does not have. Then pass the safe posture explicitly:
  `-c 'approval_policy="on-request"' -c 'sandbox_mode="workspace-write"'` (verified: the banner
  loses the YOLO line). What that posture costs, all verified in one test run: every `git add` /
  `git commit` raises an approval overlay, because a linked worktree's git metadata lives in the
  main checkout's `.git`, outside the sandbox's writable root; and the §5b rings — completion and
  help — fail with `PermissionDenied`, because the sandbox blocks the herdr socket. The lane follows
  the brief's no-retry rule, so under `--no-yolo` the §7 timer is the only way the orchestrator
  learns of a help request or a DONE — say so in §3's posture line.
- The bypass is why §3 asks for confirmation, why §1's worktree rule has no opt-out, and why §5b's
  boundaries are load-bearing: the worktree plus the brief are all that keep a lane inside its own
  checkout. Never widen a lane's blast radius beyond that — no lane briefed to push, merge, or open a
  PR. Publishing is the one operation that reaches a shared remote, so it stays in the orchestrator's
  hands, behind §6e's verification (§6f).

**The three lines §3 owes the user, in this driver's words:**

- approval posture — `yolo（已绕过审批与沙箱）` by default, or `--no-yolo（沿用你自己的审批配置）`;
- how the lane is driven — `以 codex goal 模式运行（/goal 长任务）：按上面的实施计划执行、以验收标准为
  完成判据，跨 turn 自动续跑，中途暂停/受限由监督循环自动恢复`;
- rate limits — all lanes share one Codex account, so the 5-hour and weekly windows are shared and
  can throttle every lane at once (surfacing as `goal_status: usage_limited`, §6d).

**Then verify readiness yourself — `agent start` returning `agent_started` with
`agent_status: idle` and `interactive_ready: true` does NOT mean codex is ready.** On any directory
codex has not trusted before — i.e. every fresh worktree — it sits on a modal reading `Do you trust
the contents of this directory?` with `1. Yes, continue`. herdr's own `trust_directory` rule scans only
the top 20 non-empty lines and codex's ASCII banner pushes the question below that, so herdr reports
ready while the pane is actually blocked, and any prompt sent now is eaten by the modal.

    herdr agent read <lane> --source visible

If it matches `Do you trust the contents of this directory`, send `herdr agent send-keys <lane> enter`
once, then re-read until the version banner and the `› Ask Codex to do anything` composer are
visible. Only then is the lane ready for input.

On `agent_not_ready` the name still resolves — read, resolve the dialog, continue.

**Then set the goal** — one call, and only after the composer is visible:

    herdr agent prompt <lane> "/goal Work through .dispatch/TASK.md in this directory: follow its plan in order, satisfy every acceptance criterion, keep .dispatch/progress.md updated after every checklist item, and finish by writing .dispatch/DONE and running the notify-back command TASK.md gives you."

The same mechanism §6h uses for `/compact` — the slash command executes, never landing as literal
text, and the objective rides in the same call. Verified: this one prompt sets the objective **and**
starts pursuit — the status row's right edge reads `Pursuing goal (Ns)` with a running clock.

What a goal buys over a plain prompt, and what it changes for supervision:

- The objective is **thread-scoped and persistent**: codex auto-continues toward it across turns
  instead of stopping to wait for input, so an unfinished lane picks *itself* up — nudge prompts are
  a repair path (§6c `idle_incomplete`), not the engine. This is the **autonomous** regime
  `references/supervise.md` §7 refers to.
- Goal state is machine-readable. It lives in `~/.codex/goals_1.sqlite`, keyed by the session
  uuid, with status `active` / `paused` / `blocked` / `usage_limited` / `budget_limited` /
  `complete` — §6a queries it every sweep. Trap (verified): `~/.codex/sqlite/goals_1.sqlite` also
  exists and stays empty; query the root path.
- The split of authority mirrors this skill's own: the *model* may mark its goal `complete` or
  `blocked` (codex requires the same blocker to survive 3 consecutive goal turns before
  `blocked`); `paused` and the limit states are cleared only from the user side — that is the
  sweep's job (§6d).
- **Set the goal once.** A second `/goal <objective>` at a lane whose goal is `active` pops a
  `Replace goal?` selection list whose highlighted default is **Replace** (verified; assume the
  same over a `paused` goal — untested) — §6d handles that dialog if one is ever found. Only over a
  `complete` goal does a new objective start immediately with no dialog (verified); a fresh thread
  has no goal at all, which is why §6h's restart recipe can re-set one dialog-free.

Record the lane as phase `implementing` with its session uuid in the state file. If `/goal`
errors or reports unavailable, record `goal_skipped` with the reason, surface it in §8, and fall
back to the one-shot prompt — same objective, no auto-continuation, so the sweep's nudges are all
it has:

    herdr agent prompt <lane> "Read .dispatch/TASK.md in this directory and work through its checklist. Keep .dispatch/progress.md updated after every item. Write .dispatch/DONE when everything is finished and verified, then run the notify-back command TASK.md gives you."

Start every lane before supervising any of them.

---

## §6a Probe one lane — write the lane-state helper once

If `~/.claude/dispatch-codex/bin/lane_state.py` is absent, write it, then call
`python3 ~/.claude/dispatch-codex/bin/lane_state.py <session-uuid>`. One python3 call instead of a
shell pipeline, because Bash permission rules match per shell-operator segment.

Get the session uuid straight from herdr — `herdr agent get <lane> | jq -r '.result.agent.agent_session.value'`
— and the rollout is `~/.codex/sessions/YYYY/MM/DD/rollout-<ts>-<uuid>.jsonl`. That is an O(1) lookup;
do not scan every session file. Record the uuid in the state file: it also enables `codex resume <uuid>`.
A lane restarted through §6h's `/new` recipe gets a **new** uuid and rollout file — and herdr only
reports the new uuid after the fresh thread's *first turn* (verified) — the recipe re-records it.

The helper reads only the **tail** (~2 MB, §0.6) and reports the probe contract's fields:

- `used_pct` — `last_token_usage.input_tokens / model_context_window × 100` (a percentage, to
  match §6c's `≥ 70` test) from the last `token_count` event. Use `last_token_usage`, **not**
  `total_token_usage`, which is cumulative session billing and runs into the hundreds of millions.
- `turn_state` — `complete` when the last of `task_started` / `task_complete` / `turn_aborted` is
  `task_complete`, else `working`. Keep the raw `last_event` too; §6c and §6h both gate on it.
- `compactions` — count of top-level `{"type":"compacted"}` records.
- `mtime` — the rollout's modification time, for stall detection.
- `out_of_room` (codex-only) — whether the tail contains codex's `ran out of room` marker. This is
  the only place the sweep can see it: §6b forbids pane reads, so without this field §6c's
  `hard_fail` row would never fire.
- `goal_status` and `goal_tokens` (codex-only) — not from the rollout: a read-only query of
  `~/.codex/goals_1.sqlite`
  (`SELECT status, tokens_used FROM thread_goals WHERE thread_id = ?`, the session uuid). Status
  is one of `active` / `paused` / `blocked` / `usage_limited` / `budget_limited` / `complete`, or
  null when the thread has no goal; `tokens_used` is per-goal accounting (verified), reported in §8.
  The sqlite is the **authoritative** goal state: the rollout records only the goal's creation
  (`thread_goal_updated`) and the model's own `update_goal` tool calls, never a user-side pause or a
  limit hit. Remember the path trap from §5c: the root file, not `~/.codex/sqlite/goals_1.sqlite`.

---

## §6c Classify each lane, in this order

| Class | Test | Action |
| --- | --- | --- |
| `terminal` | phase is `published`, `failed`, user-paused, or `verified` with a recorded §6f degrade reason | Skip — report only; never re-verify, re-publish, or prompt a closed lane |
| `unpublished` | phase is `verified`, `publish_attempts` < 3, no recorded degrade reason, and `pushed_sha` is missing or behind the branch HEAD, or `pr_url` is missing with PRs enabled | Retry publish (§6f) — the work is done, only publishing is left |
| `done` | `.dispatch/DONE` exists **and** `turn_state == complete` | Verify (§6e), publish (§6f), mark terminal |
| `blocked` | `agent_status == blocked` | Read `--source visible`, handle (§6g) |
| `hard_fail` | `out_of_room` (§6a), and no §6h restart is recorded in flight | Compaction cannot save it — restart per §6h's recipe and re-prime from `.dispatch/` |
| `goal_parked` | `goal_status` is `paused`, `usage_limited`, or `budget_limited` | Resume or surface, per state (§6d) |
| `help_pending` | a help request (§6j) is open, escalated, or answered but not yet delivered, **and** `turn_state == complete` | Answered and undelivered → deliver (§6d); otherwise leave it — the lane is waiting on you or the user, so never nudge it or escalate it as stuck |
| `goal_blocked` | `goal_status` is `blocked` | Steer or escalate (§6d) |
| `stalled` | `state_change_seq` **and** `mtime` both unchanged ≥ 15 min, and `goal_status` is not `complete` | Read `--source visible` once: a parked selection list → §6d/§6g; an idle composer over unfinished work → the `idle_incomplete` action; otherwise escalate; never score as finished |
| `hot` | `used_pct ≥ 70` **and** `turn_state == complete` | Compact (§6h) |
| `idle_incomplete` | `done`/`idle`, no `DONE` file, **and** `goal_status` is not `active` | Read `progress.md`, send a specific continuation prompt, record the nudge; a re-nudge with no progress since the last one escalates instead |
| `working` | otherwise | Leave it alone |

An `unknown` `agent_status` is an anomaly to surface, never a completion — do not let it fall
through to `working`'s leave-it-alone.

Row-order rationale: the goal rows sit above `stalled` so a lane frozen by a pause or a rate limit
is resumed, not escalated as stuck. `idle_incomplete` excludes an `active` goal because between
auto-continued turns a goal lane briefly reads `done` with `task_complete` — that gap belongs to
codex's continuation, and what a prompt landing in it does is untested, so the sweep stays out as
a precaution; an `active` goal that is *truly* wedged surfaces through `stalled`'s frozen `mtime`.
`help_pending` sits above `goal_blocked` because a lane in `waiting` mode usually ends up there by
itself: it ends its turn, the active goal auto-continues it, and after three turns on the same
unanswered question codex lets the model mark the goal `blocked`. That is the expected shape of a
waiting goal lane, not a new blocker — the answer is the steer.

One known race, by design: a notify-back can arrive before the ringing lane's final turn closes
(the brief fires it right after DONE is written, mid-turn), so that lane may still read `working`
with DONE present — re-check it once at the end of the sweep, or leave it to the next tick; both
are fine.

---

## §6d Steer a goal lane

The goal's own state (§6a `goal_status`) decides the move. All slash commands below go through
`herdr agent prompt <lane> "…"` — the same verified mechanism as §6h's `/compact` — and every one of
them is subject to §6b's prompt guard.

**`paused` / `usage_limited` / `budget_limited` → resume or surface, per state.** The resume
command, when a state below calls for it (verified: immediate and dialog-free — codex prints
`Goal active Objective: …` and pursuit restarts):

    herdr agent prompt <lane> "/goal resume"

- `paused` — codex does not pause itself, so someone did. **You** pause a lane only when the user
  asks: send `/goal pause` (verified working, even mid-pursuit), record
  `pause: {origin: user, at: <sweep time>}` in the lane's state entry in the same sweep, and the
  lane is user-paused — terminal for §7, never resumed by a sweep. A `paused` goal with **no**
  such record was paused by a hand you cannot see — almost certainly the user's own, at the
  keyboard: do **not** resume it; escalate once (record `escalated` with the reason), report the
  lane as paused-awaiting-user, and resume only when the user says so.
- `usage_limited` — the shared Codex account hit its rate window (§3 warned about this). Resume
  once per sweep, not in a loop: if the window is still exhausted, expect the goal to just re-park
  (untested — the next sweep's `goal_status` is the answer), and that tick cadence is the retry.
- `budget_limited` — the goal carries a token budget. This skill never sets one, so a budget
  means someone configured it deliberately: surface it once (record `escalated`), and resume only
  on the user's word.

**`blocked` → steer or escalate.** The model itself marked the goal blocked, which codex only
permits after the same blocker survived 3 consecutive goal turns — so this is persistent, not a
flake. Read `.dispatch/progress.md` and the pane (`--source visible`) to see the blocker. If the
answer lies inside the §3-confirmed plan — a decision the brief already made, a misreading of a
step — send the corrective prompt with `--wait --until idle --timeout 120000` so its turn actually
ends, **then** `/goal resume`: codex refuses slash commands mid-turn, and a refused one lingers in
the composer where the next prompt fuses onto it (§6h's clear applies — `ctrl+u` then `ctrl+k`).
If the blocker is real and outside the plan's scope (a missing secret, a broken upstream, a
contradiction in the task itself), escalate to the user with the blocker quoted (record
`escalated`); do not improvise scope.

**`complete` but no `DONE`** — codex considers the goal met but the completion protocol did not
finish. The lane reaches `idle_incomplete` (a `complete` goal is neither `active` nor within
`stalled`'s test), and the continuation prompt there is safe: with no active goal there is no
auto-continuation to collide with.

**Delivering a help answer (§6j).** The answer file is already on disk; this decides whether the
lane also needs a prompt. Every prompt here is subject to §6b's prompt guard.

- `turn_state == working`, or `goal_status == active` → file only. The lane checks
  `.dispatch/help/` after every checklist item and at the start of every turn (§5b), and an active
  goal starts the next turn by itself. Do not prompt into an active goal's gap between turns — that
  collision is untested (§6c).
- `goal_status == blocked` → send the answer prompt below with `--wait --until idle --timeout
  120000`, then `/goal resume` — the same two-step as any `blocked` steer, for the same reason.
- `goal_status` is `paused`, `usage_limited` or `budget_limited` → the `goal_parked` rules above
  decide whether to resume; when they do, the resumed turn finds the file.
- no goal, or `complete` → send the answer prompt; with no pursuit running there is nothing to
  collide with.

The answer prompt:

    herdr agent prompt <lane> "Help answer for H-<n> is in .dispatch/help/H-<n>.answer.md. Read it, apply it, append 'applied H-<n>' to .dispatch/progress.md, then continue the plan."

**A `Replace goal?` selection list on screen** — found by §6b's prompt guard or `stalled`'s pane
read; do not assume herdr reports it as `blocked` (a parked codex selection list can read `done` —
verified). It means a second `/goal <objective>` reached a lane whose goal was still live. Its
highlighted default is **Replace**, and a prompt submitted at a parked list presses that default
(both verified) — so answer it only with `send-keys`:
`herdr agent send-keys <lane> 2` cancels and keeps the current goal (verified; the digit alone
selects and confirms). Then work out which sweep double-fired and fix the state file, because a
goal lane should see `/goal <objective>` exactly once in its life (§5c).

---

## §6g Handle a blocked lane

Under the default yolo mode codex raises no approval overlays, so a `blocked` lane there is an anomaly
— usually a trust modal (§5c) or a herdr misclassification, not an approval request. Read the pane
before assuming which. The rules below apply in full under `--no-yolo`.

Read with `--source visible`. Codex approval overlays are titled `Would you like to run the following
command?` / `make the following edits?` / `grant these permissions?` / `send input to the existing
terminal?`, and are selection lists with hotkeys `y` approve, `a` approve-for-session, `p`
approve-for-prefix, `d` deny, `c` cancel.

- Benign and inside the lane's own checkout (edit its files, run its tests, read files) → approve with
  `herdr agent send-keys <lane> y`.
- The one sanctioned exception to the bullet below: under `--no-yolo` a finishing lane's notify-back
  (§5b) surfaces *here*, as an approval overlay. If the quoted command is **exactly** the notify-back
  — `herdr agent prompt` aimed at the recorded `orchestrator_pane`, carrying this run's id and the
  lane's own name, with nothing chained after it — approve it with `y`: you briefed it, and the sweep
  reading this overlay is already the sweep it was trying to summon. Any variation — another pane,
  another run id, an extra `;`/`&&` command — is not the notify-back and falls through to the rule
  below. This also means that under `--no-yolo` the doorbell rings late by design: the overlay holds
  the ring until a timer sweep approves it, one more cost of `--no-yolo`, not a malfunction.
- Anything leaving the lane's blast radius — `git push`, `gh pr create`, force operations, `sudo`,
  deleting outside the checkout, reading credentials, writing to a network target → **do not answer**.
  Pause the lane, record it, surface it to the user. Publishing being a normal part of this run
  (§6f) does not make it approvable *here*: §6f runs after verification, on your side of the fence.
  A lane asking to push is a lane that misread its brief.

`herdr agent prompt` returns `agent_blocked` while a dialog is up, so clear it with `send-keys` first.

---

## §6h Compact a lane

    herdr agent prompt <lane> "/compact" --wait --until idle --timeout 120000

This is verified to execute the slash command in one call — it does not land as literal user text, and
no key choreography is needed.

- **Only between turns.** Codex refuses slash commands mid-turn, and
  `agent prompt` rejects only `blocked`, not `working`. Gate on `turn_state == complete` from
  the rollout yourself — and on a goal lane expect to lose the race sometimes: the continuation
  gap is narrow, and a `/compact` that lands after the next turn began is refused. Take the
  refusal, clear the composer, and retry next sweep; never force it.
- **Clear the composer first.** A previously rejected slash command stays in the composer with the
  cursor at position 0, so the next prompt gets *prepended* to it and fuses. `esc` alone and `ctrl+u`
  alone are both no-ops here; send `ctrl+u` then `ctrl+k` to clear.
- **Success is an effect, not a return code.** The rollout's top-level `{"type":"compacted"}` count
  must increase. On screen the marker is `Context compacted` — do **not** grep for `Long threads and
  multiple compactions`, which does not appear.
- **Do not read these as failure:** `Stream disconnected before completion`, `Falling back from
  WebSockets to HTTPS transport`, `Reconnecting…`. A compaction prints them and still succeeds. An
  empty `payload.message` is also normal — the summary is encrypted inside `replacement_history`.
- **Rejection signal:** herdr returns `agent_prompt_stalled`, and the screen shows `Unrecognized
  command`. Both are reliable; the herdr error needs no scraping.
- **After compacting a goal lane, do not re-prime by prompt.** The goal record survives — the
  sqlite row is keyed by the thread, which `/compact` does not change — and pursuit is *expected*
  to re-engage on its own, but that re-engagement is untested. So check, don't assume: on the next
  sweep, require `goal_status` still `active` **and** a moving rollout (`mtime`, a new turn); an
  `active` goal with no new turn after a compaction gets a prompt-guarded `/goal resume`, and if
  that changes nothing, escalate. Only a `goal_skipped` lane (no goal to continue it) gets the §5c
  one-shot prompt text re-sent after compaction.
- **Stop after 3 compactions.** Codex itself warns repeated compaction degrades accuracy. Instead hand
  off: restart the lane with the recipe below.

**The restart recipe — a fresh thread on the same pane, for a lane whose own agent is alive.**
The paths that end here are `hard_fail` and the 3-compaction handoff (a §6b uuid mismatch is
*not* one of them — §6b routes each of its cases itself). In order:

1. Record `restarting: true` on the lane and rewrite the state file — this is what keeps the next
   sweep's `hard_fail` row from firing again off the old rollout while the restart is mid-flight.
2. `herdr agent prompt <lane> "/new"` — the same verified slash-command mechanism as `/compact`
   and `/goal`: the pane returns to an empty composer at 100% context.
3. Re-prime: re-set the goal with §5c's exact `/goal` call (the fresh thread has no goal, so no
   `Replace goal?` dialog can appear), or the §5c one-shot for a `goal_skipped` lane.
4. One verified timing trap: the `agent_session.value` herdr reports does **not** change at
   `/new` — it changes on the new thread's *first turn*. So now poll `herdr agent get <lane>`
   until the reported uuid **differs** from the recorded one, then record the new uuid and clear
   `restarting`. Never record a value you have not seen change: writing the old uuid back leaves
   `out_of_room` pointing at the dead rollout and turns the next sweep into a restart loop.

If the pane does not respond to `/new`, the codex process itself is gone or wedged: escalate to
the user; never `worktree remove` (§0.5), and relaunching via `agent start` needs the pane back at
a shell prompt first (§5a's checks apply again).
