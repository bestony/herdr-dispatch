# Grok driver — §5a, §5c, §6a, §6c, §6d, §6g, §6h

Everything in `dispatch-grok` that depends on grok itself. The agent-independent halves are
`references/plan.md` (§2–§4, §5b) and `references/supervise.md` (§6b, §6e, §6f, §6i, §7, §8);
§0 and §1 are in `../SKILL.md`.

Sources for the claims below: grok's own bundled user guide at `~/.grok/docs/user-guide/` (notably
`04-slash-commands.md`, `17-sessions.md`, `22-permissions-and-safety.md`, `26-config-reference.md`)
and `grok --help`. Claims marked **(observed)** were read off real session directories on this
machine rather than documented; treat them as reliable but version-bound, and record the version §2
finds. Anything marked **(unverified)** has not been exercised end-to-end — handle its failure path,
never assume it works.

---

## §5a Pre-flight the pane

A fresh worktree is not a working environment: no `node_modules/`, no `.env`, no `.venv`, because
those are untracked or ignored.

    herdr pane send-text <pane> 'grok --version'
    herdr pane send-keys <pane> enter

Run the version, not `command -v`: the executable may live outside the login shell's PATH
(`~/.grok/bin/grok` on this machine) or behind a version manager whose shim resolves while the real
binary does not, and then `agent start` just times out. `grok --version` exercises the real
resolution path. `grok doctor` additionally checks terminal, clipboard, colour and input support
without starting a session — worth one run in the *first* lane's pane when anything about the
terminal looks unusual, not in every lane.

There is no `herdr pane run`: `pane send-text` only types the text, and `pane send-keys <pane> enter`
submits it. Then read the result with `herdr pane read <pane> --source visible`. On herdr 0.9.3,
`herdr pane wait-output` searches the output already on screen before it polls, so it also finds a
command that already finished (verified). Do not wait on the tool name alone: the echoed command
line contains it too.

- `grok` not found, or the version refuses to resolve → stop that lane and report it.
  `agent start --kind grok` resolves the executable from the pane's own login shell and args cannot
  redirect it.
- Dependencies missing → run the repo's install command in the pane and let it finish **before**
  starting the agent; `agent start` needs a pane at an idle interactive prompt.
- `.env` or other secrets missing → **ask the user** whether to symlink them. Never copy secrets.
  (Note `[session] load_envrc` in `~/.grok/config.toml`: grok may load `.envrc` itself, which is not
  a reason to skip this check.)

Then write the brief (§5b in `references/plan.md`) before launching.

---

## §5c Launch grok and prime the lane

Launch with **no positional prompt**. Grok accepts one, but a goal is set by slash command, which
cannot ride along on argv, and priming through `herdr agent prompt` is what lets you verify the
composer is actually accepting input first — a startup modal eats whatever reaches it.

    herdr agent start <lane> --kind grok --pane <pane> --timeout 120000 -- \
      --no-alt-screen --permission-mode bypassPermissions

Why each part:

- `--no-alt-screen` runs grok inline instead of on the terminal's alternate screen, so
  `herdr agent read --source recent-unwrapped` reaches real scrollback and reads keep working while
  the agent is busy. Grok's docs are explicit that `--no-alt-screen` **still counts as fullscreen**
  for command availability, so nothing in the slash-command set is lost by using it.
- Do **not** pass `--minimal`. It switches to scrollback-native rendering *and* hides the
  fullscreen-only commands; `--no-alt-screen` gets the readable scrollback without that cost.
- `--permission-mode bypassPermissions` is **on by default** here: lanes run unattended and an
  approval overlay stalls a lane until the next sweep notices it. Under `--no-yolo`, pass
  `--permission-mode default` instead — explicitly, because `~/.grok/config.toml` may set
  `[ui] yolo = true` / `[ui] permission_mode = "always-approve"` globally and simply omitting the
  flag would leave the lane auto-approving (§1).
- **Permission bypass is not a sandbox.** Grok has a separate `--sandbox <profile>` (and
  `[sandbox] profile` in config, `GROK_SANDBOX` in the env). This skill does not touch it: whatever
  the user configured stays in force. If `[sandbox] profile = "off"`, say so in §3's first line
  rather than implying an isolation that is not there — the worktree is then the only boundary.
- `--timeout 120000` because a cold start plus MCP boot can take far longer than herdr's 30 s
  default.
- Never pass `-w` / `--worktree`: grok would create a *second* worktree of its own, and the lane
  would then be committing somewhere §6e never looks and §6f never pushes (§4).
- Never pass `-p` / `--single`, `--output-format` or `--max-turns`: those are headless modes, and
  this skill needs a live TUI that herdr can read and prompt.

**The three lines §3 owes the user, in this driver's words:**

- approval posture — `yolo（--permission-mode bypassPermissions，工具调用不再询问）` by default, or
  `--no-yolo（--permission-mode default，工具调用会弹审批，由监督循环处理）`; plus the sandbox profile
  actually in effect;
- how the lane is driven — with goal mode available:
  `以 grok goal 模式运行（/goal 长任务）：按上面的实施计划执行、以验收标准为完成判据，跨轮次自动推进，
  grok 会在自认完成前先做一次证据复核`; without it:
  `以单次 prompt 驱动、由监督循环按 progress.md 续跑（本机未开启 grok goal 模式）`;
- rate limits — all lanes share one Grok account, so its usage window is shared and can throttle
  every lane at once; a throttled lane shows up as a stalled turn, and `/usage` in the pane (or
  `grok usage <session-id>` from the shell) is where the numbers are.

**Then verify readiness yourself — `agent start` returning ready does NOT mean grok is ready.**
Grok keeps a trusted-folder list (`~/.grok/trusted_folders.toml`, observed), and a fresh worktree is
by definition not in it, so a first-run trust prompt is likely. Read the pane:

    herdr agent read <lane> --source visible

If anything asking about trusting this folder is on screen, resolve it with `send-keys` matching the
option actually displayed — the exact wording and hotkeys are **(unverified)** here, so read them,
never type a remembered key. Then re-read until the composer is visible and idle. Only then is the
lane ready for input. On `agent_not_ready` the name still resolves — read, resolve the dialog,
continue.

**Then prime the lane.** First decide whether goal mode is even available, from disk rather than by
trial: grok's docs state `/goal` appears only when goal mode is enabled for the session, and
`26-config-reference.md` gives the switch as `goal.enabled` (boolean, user-scoped). Read
`~/.grok/config.toml`:

- `[goal] enabled = true` → set a goal, one call, only after the composer is visible:

      herdr agent prompt <lane> "/goal Work through .dispatch/TASK.md in this directory: follow its plan in order, satisfy every acceptance criterion, keep .dispatch/progress.md updated after every checklist item, and finish by writing .dispatch/DONE and running the notify-back command TASK.md gives you."

  Then read the pane once. Grok's docs describe `/goal <objective> [--budget <tokens>]`; this skill
  never sets a budget. If the pane shows the objective accepted, record the lane as phase
  `implementing` with `goal: set`. If it shows an unknown command, or the text landed in the
  composer as literal user input, treat goal mode as unavailable and fall through to the next case.
- otherwise → record `goal_skipped` with the reason (`goal.enabled not set in ~/.grok/config.toml`),
  surface it in §8, and prime with the one-shot prompt:

      herdr agent prompt <lane> "Read .dispatch/TASK.md in this directory and work through its checklist. Keep .dispatch/progress.md updated after every item. Write .dispatch/DONE when everything is finished and verified, then run the notify-back command TASK.md gives you."

Which case applies changes the supervision regime, so record it per lane:

- **goal set** — grok works across rounds toward the objective and only marks it complete after an
  independent evidence review, so a silent lane may legitimately be mid-pursuit; nudges are a repair
  path. This is the *autonomous* regime `references/supervise.md` §7 describes.
- **goal skipped** — the lane stops at the end of each turn and the sweep's continuation prompt is
  the engine (§6c `idle_incomplete`, §6d).

**Set the goal once.** Re-sending `/goal <objective>` at a lane that already has one is how you get
a replace-or-keep dialog (**unverified** for grok, and verified to exist in the sibling codex
driver); §6d handles one if it is ever found. `/goal clear` and a fresh `/goal` are the sanctioned
way to change an objective, not a second bare `/goal`.

Start every lane before supervising any of them.

---

## §6a Probe one lane

Write `~/.claude/dispatch-grok/bin/lane_state.py` once if it is absent, then call
`python3 ~/.claude/dispatch-grok/bin/lane_state.py <checkout>`. One python3 call instead of a shell
pipeline, because Bash permission rules match per shell-operator segment.

**Finding the lane's session directory.** Grok stores sessions at
`~/.grok/sessions/<percent-encoded-cwd>/<session-uuid>/` (observed — the cwd is percent-encoded, so
`/Users/x/y` becomes `%2FUsers%2Fx%2Fy`). Do **not** reconstruct that encoding: scan
`~/.grok/sessions/*/*/summary.json` and select the entry whose `info.cwd` equals the lane's checkout
path, newest `last_active_at` first. That is one small glob over small files, it survives any change
to the encoding rule, and `summary.json` carries the identity fields you want anyway. Record the
session uuid in the state file; it is also what `grok usage <session-id>` takes.

The helper reports the probe contract's fields, plus grok's own:

- `used_pct` — `signals.json` → `contextWindowUsage`, already a percentage (observed, alongside
  `contextTokensUsed` and `contextWindowTokens` for the report). No arithmetic and no model catalog
  needed.
- `turn_state` — tail `events.jsonl` (small; one JSON object per line with `ts` and `type`): the
  last `turn_started` / `turn_ended` decides it — `turn_ended` (which carries an `outcome`) means
  `complete`, a `turn_started` with no matching `turn_ended` means `working` (observed).
- `compactions` — `signals.json` → `compactionCount` (observed).
- `mtime` — newest mtime among `events.jsonl` and `chat_history.jsonl`, for stall detection.
- `probe` — `unavailable` with a reason when no `summary.json` matches the checkout: the session may
  not have been created yet (grok writes it on first activity), which for a just-primed lane is
  normal and for an old lane is an anomaly.
- `approval_parked` (grok-only) — a `permission_requested` event in the tail with no later
  `permission_resolved` for the same tool (observed: both events carry `tool_name`, and the resolved
  one carries `decision` and `wait_ms`). This is the rare case of an approval prompt being visible
  **from disk**; §6c uses it so a `--no-yolo` run does not need pane reads to notice one.
- `goal_status` — always `unknown` from disk (§0.4). Only §6d fills it in, and only when it is
  already acting on that lane.
- report-only extras from `signals.json` (observed): `turnCount`, `errorCount`, `toolFailureCount`,
  `cancellationCount`, `gitCommitCount`, `prCreatedCount`, `inferenceIdleTimeouts`,
  `doomLoopRecoveryAttempts`, `agentLinesAdded` / `agentLinesRemoved`. `gitCommitCount` is a cheap
  cross-check against `git log` in §6e; a non-zero `prCreatedCount` is an alarm — the lane was
  briefed never to open a PR (§5b).
- report-only extras from `summary.json` (observed): `num_messages`, `current_model_id`,
  `agent_name`, `last_turn_summary`, `sandbox_profile`, `head_branch`, `git_root_dir`.
  **`head_branch` is an identity check**: it must equal the lane's branch. If it does not, the
  session you found is not this lane's — surface it and steer nothing (§6b's identity rule).

`grok usage <session-id>` prints persisted per-turn token and cost totals for the §8 report. It is a
shell subcommand, so it costs the lane nothing; run it once at report time, not every sweep.

---

## §6c Classify each lane, in this order

| Class | Test | Action |
| --- | --- | --- |
| `terminal` | phase is `published`, `failed`, user-paused, or `verified` with a recorded §6f degrade reason | Skip — report only; never re-verify, re-publish, or prompt a closed lane |
| `unpublished` | phase is `verified`, `publish_attempts` < 3, no recorded degrade reason, and `pushed_sha` is missing or behind the branch HEAD, or `pr_url` is missing with PRs enabled | Retry publish (§6f) |
| `done` | `.dispatch/DONE` exists **and** `turn_state == complete` | Verify (§6e), publish (§6f), mark terminal |
| `approval_parked` | §6a's `approval_parked`, or `agent_status == blocked` | Read `--source visible`, handle (§6g) |
| `identity_mismatch` | `summary.json`'s `head_branch` ≠ the lane's branch, or the session uuid changed with no restart in flight | Surface it; steer nothing (§6b) |
| `help_pending` | a help request (§6j) is open, escalated, or answered but not yet delivered, **and** `turn_state == complete` | Answered and undelivered → deliver (§6d); otherwise leave it — the lane is waiting on you or the user, so never nudge it, ask `/goal status`, or escalate it as stuck |
| `stalled` | `state_change_seq` **and** `mtime` both unchanged ≥ 15 min | Ask `/goal status` if the lane has a goal (§6d), else read `--source visible` once: a parked selection list → §6d/§6g; an idle composer over unfinished work → the `idle_incomplete` action; otherwise escalate; never score as finished |
| `hot` | `used_pct ≥ <auto-compact threshold> + 5`, `compactions` unchanged since the previous sweep, **and** `turn_state == complete` | Grok's own auto-compaction should have fired and did not — compact manually (§6h) |
| `idle_incomplete` | `turn_state == complete`, no `DONE` file, and either the lane has no goal, or `/goal status` (§6d) reported it not `active` | Read `progress.md`, send a specific continuation prompt, record the nudge; a re-nudge with no progress since the last one escalates instead |
| `working` | otherwise | Leave it alone |

An `unknown` `agent_status` is an anomaly to surface, never a completion — do not let it fall
through to `working`'s leave-it-alone.

**The auto-compact threshold is grok's, not yours.** Read `[session] auto_compact_threshold_percent`
from `~/.grok/config.toml` at §2 and record it; grok's documented default is 85 when the key is
absent. Grok compacts itself at that percentage, so `hot` exists only to catch the case where it
*should* have and the compaction count says it did not. Never compact a lane that is merely busy.

**On a goal lane, `turn_state == complete` is not idleness.** Grok continues across rounds, so a
lane between rounds looks exactly like a lane that stopped. That is why `idle_incomplete` requires
either "no goal" or an explicit `/goal status` answer, and why `stalled`'s frozen `mtime` — not a
finished turn — is what proves a goal lane is genuinely wedged.

One known race, by design: a notify-back can arrive before the ringing lane's final turn closes (the
brief fires it right after DONE is written, mid-turn), so that lane may still read `working` with
DONE present — re-check it once at the end of the sweep, or leave it to the next tick; both are fine.

---

## §6d Steer a lane

All slash commands go through `herdr agent prompt <lane> "…"`, and every one of them is subject to
§6b's prompt guard: read the pane first, and if a selection list is parked there, resolve it instead
of typing.

**Asking a goal lane where it stands.** `/goal status` is the only way to see goal state (§0.4), and
it costs a turn, so send it only from `stalled` or before an `idle_incomplete` nudge:

    herdr agent prompt <lane> "/goal status" --wait --until idle --timeout 120000

Then read `--source visible` and act on what it says (grok's documented verbs are `status`, `pause`,
`resume`, `clear`; the exact status wording is **unverified** here, so match loosely and escalate
rather than guess):

- **active / pursuing** — the lane is mid-pursuit. Do nothing this sweep; record that you asked, so
  a later `stalled` on the same frozen `mtime` escalates instead of asking again.
- **paused** — grok does not pause itself, so someone did. **You** pause a lane only when the user
  asks: send `/goal pause`, record `pause: {origin: user, at: <sweep time>}` in the same sweep, and
  the lane is user-paused — terminal for §7, never resumed by a sweep. A pause with no such record
  was made by a hand you cannot see, almost certainly the user's own at the keyboard: do **not**
  resume it; escalate once (record `escalated`), report it as paused-awaiting-user, and resume only
  when the user says so.
- **blocked** — read `.dispatch/progress.md` and the pane for the blocker. If the answer lies inside
  the §3-confirmed plan (a decision the brief already made, a misread step), send the corrective
  prompt with `--wait --until idle --timeout 120000` so its turn actually ends, then `/goal resume`.
  If the blocker is real and outside the plan's scope — a missing secret, a broken upstream, a
  contradiction in the task — escalate with the blocker quoted (record `escalated`); do not improvise
  scope.
- **complete, but no `.dispatch/DONE`** — the objective was met by grok's reckoning while the
  completion protocol was not finished. This is the safe case for a continuation prompt: with no
  pursuit running there is nothing to collide with.
- **no goal / unknown command** — the lane is in the one-shot regime after all. Correct the state
  file (`goal_skipped`) and treat it as `idle_incomplete` from here on.

**Resuming:** `herdr agent prompt <lane> "/goal resume"`. Resume once per sweep, never in a loop — if
the account's usage window is exhausted the lane will simply re-park, and the §7 tick cadence *is*
the retry.

**Nudging a lane with no goal** — the engine for the one-shot regime. Read `.dispatch/progress.md`
first and write a prompt that names the next unchecked checklist item and the file it touches; a
bare "continue" wastes a turn. Record `nudges`, the branch HEAD sha and a hash of `progress.md` with
each nudge: a nudge that produces neither a new commit nor a changed `progress.md` by the next sweep
means the lane is not responding to prompts, and the second such nudge escalates instead of firing a
third.

**Delivering a help answer (§6j).** The answer file is already on disk; this decides whether the
lane also needs a prompt. Every prompt here is subject to §6b's prompt guard.

- `turn_state == working` → file only. The lane checks `.dispatch/help/` after every checklist item
  and at the start of every turn (§5b).
- turn complete, lane has no goal (`goal_skipped`) → send the answer prompt below.
- turn complete, lane has a goal → ask `/goal status` first (above) — it is the one place this
  sweep may spend that turn on a `help_pending` lane. **active** → file only; the next round finds
  it. **blocked** → answer prompt with `--wait --until idle --timeout 120000`, then `/goal resume`.
  **complete** → answer prompt. **paused** → the pause rules above decide; never resume a pause you
  did not record.

The answer prompt:

    herdr agent prompt <lane> "Help answer for H-<n> is in .dispatch/help/H-<n>.answer.md. Read it, apply it, append 'applied H-<n>' to .dispatch/progress.md, then continue the plan."

**A replace-or-keep goal dialog on screen** (**unverified** for grok, but its sibling driver has it
verified for codex): it means a second `/goal <objective>` reached a lane whose goal was still live.
Answer only with `send-keys`, choosing the option that **keeps** the existing goal — read the option
list, do not type a remembered key — then work out which sweep double-fired and fix the state file.

---

## §6g Handle a blocked lane

Under the default `bypassPermissions` grok raises no approval overlays, so `blocked` there is an
anomaly — a trust modal (§5c), a model or settings dialog, or a herdr misclassification. Read the
pane before assuming which. The rules below apply in full under `--no-yolo`, and that is also when
§6a's `approval_parked` starts firing.

Read with `--source visible`. Grok's permission prompts are selection lists; their exact titles and
hotkeys are **(unverified)** here, so pick by what is displayed and answer with `send-keys`, never
with a remembered key or a prompt (a prompt submitted at a parked list presses its highlighted
default, and `~/.grok/config.toml` can set that default to
`default_selected_permission = "always_allow_all_sessions"` — the widest possible answer).

- Benign and inside the lane's own checkout (edit its files, run its tests, read files) → approve the
  narrowest option offered. Prefer a once-only approval over a remember-for-all-sessions one: the
  latter outlives this run and changes the user's machine.
- The one sanctioned exception below: under `--no-yolo` a finishing lane's notify-back (§5b) surfaces
  here as an approval request. If the quoted command is **exactly** the notify-back —
  `herdr agent prompt` aimed at the recorded `orchestrator_pane`, carrying this run's id and the
  lane's own name, with nothing chained after it — approve it: you briefed it, and the sweep reading
  this overlay is already the sweep it was trying to summon. Any variation — another pane, another
  run id, an extra `;`/`&&` command — is not the notify-back and falls through to the rule below.
  This also means that under `--no-yolo` the doorbell rings late by design.
- Anything leaving the lane's blast radius — `git push`, `gh pr create`, force operations, `sudo`,
  deleting outside the checkout, reading credentials, writing to a network target → **do not answer**.
  Pause the lane, record it, surface it to the user. Publishing being a normal part of this run (§6f)
  does not make it approvable *here*: §6f runs after verification, on your side of the fence. A lane
  asking to push is a lane that misread its brief — and §6a's `prCreatedCount` is where you would
  find out it succeeded.

`herdr agent prompt` returns `agent_blocked` while a dialog is up, so clear it with `send-keys` first.

---

## §6h Compact a lane

    herdr agent prompt <lane> "/compact" --wait --until idle --timeout 120000

Grok also accepts a focus note — `/compact keep the migration details` — which is worth using when
`progress.md` shows the lane is mid-way through one specific file.

- **Only between turns.** Gate on `turn_state == complete` from §6a yourself. A slash command that
  lands mid-turn is refused, and a refused one can linger in the composer where the next prompt
  fuses onto it; clear the composer before retrying, and take the refusal rather than forcing it.
- **Success is an effect, not a return code.** `signals.json`'s `compactionCount` must increase.
- **Expect it to be unnecessary.** Grok auto-compacts at its configured threshold (§6c), so a lane
  that reaches `hot` is already unusual; if a manual `/compact` does not raise `compactionCount`
  either, that is an escalation, not a retry loop.
- **Stop after 3 compactions.** Repeated compaction degrades accuracy. Hand off to the restart
  recipe instead.

**The restart recipe — a fresh session on the same pane, for a lane whose own agent is alive:**

1. Record `restarting: true` on the lane and rewrite the state file, so the next sweep does not fire
   the same row again off the old session directory while the restart is mid-flight.
2. `herdr agent prompt <lane> "/new"` (alias `/clear`) — the composer returns empty on a fresh
   session.
3. Re-prime exactly as §5c does, in the same regime the lane was in: a `/goal` call for a goal lane
   (the fresh session has no goal, so no replace dialog can appear), or the one-shot prompt
   otherwise.
4. Re-identify the session: a fresh session gets a **new** uuid and a **new** directory under
   `~/.grok/sessions/<encoded-cwd>/`. Poll §6a's scan until a `summary.json` appears whose `info.cwd`
   matches the checkout and whose uuid **differs** from the recorded one, then record the new uuid
   and clear `restarting`. Never write back an id you have not seen change — that leaves the sweep
   reading a dead session and restarting forever.

If the pane does not respond to `/new`, the grok process itself is gone or wedged: escalate to the
user; never `worktree remove` (§0.6), and relaunching via `agent start` needs the pane back at a
shell prompt first (§5a's checks apply again).
