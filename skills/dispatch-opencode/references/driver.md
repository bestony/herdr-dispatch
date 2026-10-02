# Opencode driver — §5a, §5c, §6a, §6c, §6d, §6g, §6h

Everything in `dispatch-opencode` that depends on opencode itself. The agent-independent halves are
`references/plan.md` (§2–§4, §5b) and `references/supervise.md` (§6b, §6e, §6f, §6i, §7, §8);
§0 and §1 are in `../SKILL.md`.

Sources for the claims below: `opencode --help` and its subcommand help, and the schema and contents
of `~/.local/share/opencode/opencode.db` on this machine (opencode 1.18.29). Claims marked
**(observed)** were read off that database; **(unverified)** means the path has not been exercised
end-to-end — handle its failure, never assume it works. Opencode moves fast and stores more of its
behaviour in the binary than in documentation, so this driver is deliberately written to degrade to
pane reads and git state whenever a probe does not return what it expects.

---

## §5a Pre-flight the pane

A fresh worktree is not a working environment: no `node_modules/`, no `.env`, no `.venv`, because
those are untracked or ignored.

    herdr pane send-text <pane> 'opencode --version'
    herdr pane send-keys <pane> enter

Run the version, not `command -v`: a shim can resolve while the real binary does not, and then
`agent start` just times out. Record the version — this driver's "(observed)" claims are bound to it.

There is no `herdr pane run`: `pane send-text` only types the text, and `pane send-keys <pane> enter`
submits it. Then read the result with `herdr pane read <pane> --source visible`. On herdr 0.9.3,
`herdr pane wait-output` searches the output already on screen before it polls, so it also finds a
command that already finished (verified). Do not wait on the tool name alone: the echoed command
line contains it too.

- `opencode` not found → stop that lane and report it. `agent start --kind opencode` resolves the
  executable from the pane's own login shell and args cannot redirect it.
- Dependencies missing → run the repo's install command in the pane and let it finish **before**
  starting the agent; `agent start` needs a pane at an idle interactive prompt.
- `.env` or other secrets missing → **ask the user** whether to symlink them. Never copy secrets.

Once per run, not per lane, resolve two things and record them in the state file:

    opencode debug paths     # where the database actually lives — never assume the default
    opencode models          # that a provider is configured and which model is default

A lane whose first prompt dies on a missing provider looks identical to a lane that ignored its
brief. If `opencode debug paths` does not name a database, §6a has no probe: record
`probe: unavailable` for the whole run up front and tell the user in §3 that supervision will run on
pane reads and git state alone, which is materially weaker.

Then write the brief (§5b in `references/plan.md`) before launching.

---

## §5c Launch opencode and prime the lane

    herdr agent start <lane> --kind opencode --pane <pane> --timeout 120000 -- --auto

Why each part:

- `--auto` auto-approves permissions that are not explicitly denied — opencode's own help calls it
  dangerous, and it is the yolo analogue here: lanes run unattended, and a permission prompt stalls a
  lane until the next sweep notices it. Under `--no-yolo`, omit it and handle every prompt in §6g.
  `--auto` is **not** a sandbox: it widens what runs without asking, and the worktree plus the
  brief's boundaries (§5b) remain the only containment.
- No positional `[project]` argument. Opencode starts in the pane's own cwd, which herdr already put
  in the lane's checkout; passing a path is a chance to silently start the lane in the wrong repo.
- No `--prompt`. The lane is primed through `herdr agent prompt` below, after the composer has been
  seen, so a startup dialog cannot eat the objective.
- No `--session` / `--continue`: each lane is a new session, and §6a identifies it by directory.
- Do not pass `--mini` by default. It is opencode's minimal interface and **(unverified)** under
  herdr's detection; if `herdr agent read` on this lane returns nothing usable — a full-screen TUI
  that scrollback reads cannot reach — restart the lane with `--mini` added, and record that you did,
  because it changes what every later pane read looks like.
- Never `opencode run`, `serve`, `web` or `acp`: those are non-interactive or headless modes, and
  this skill needs a live TUI that herdr can read and prompt.

**The three lines §3 owes the user, in this driver's words:**

- approval posture — `--auto（自动批准未被显式拒绝的工具调用，opencode 官方标注为 dangerous）` by
  default, or `--no-yolo（工具调用会在 pane 里弹权限提示，由监督循环处理，lane 会因此停顿到下一轮
  sweep）`; either way, no sandbox — the worktree is the boundary;
- how the lane is driven — `以单次 prompt 启动，之后由监督循环（默认每 5 分钟）读 progress.md 续跑：
  opencode 每个 turn 结束就停，没有 goal 模式，所以推进速度取决于 sweep 频率`;
- context and rate limits — all lanes share one provider account, so its limits are shared and can
  throttle every lane at once; and state whether `--compact-at` is set (§1), because with it unset
  this skill never compacts a lane on its own.

**Then verify readiness yourself.** Read the pane:

    herdr agent read <lane> --source visible

Wait until the composer is visible and idle before sending anything. Opencode's first-run behaviour
in an unseen directory is **(unverified)** here — a project-init or trust prompt is possible — so if
any dialog is up, resolve it with `send-keys` matching the option displayed, never a remembered key.
On `agent_not_ready` the name still resolves — read, resolve, continue.

**Then prime the lane** — one call, only after the composer is visible:

    herdr agent prompt <lane> "Read .dispatch/TASK.md in this directory and work through its checklist in order. Keep .dispatch/progress.md updated after every item — assume your context may be compacted at any time and that file is all you keep. Write .dispatch/DONE only when every checklist item is done, every acceptance criterion in TASK.md verifiably holds and git status is clean, then run the notify-back command TASK.md gives you."

Record the lane as phase `implementing`. This is the **nudge-driven** regime that
`references/supervise.md` §7 refers to: the lane will stop at the end of this turn, and every
subsequent step comes from §6d's continuation prompts. Start every lane before supervising any of
them.

---

## §6a Probe one lane

Write `~/.claude/dispatch-opencode/bin/lane_state.py` once if it is absent, then call
`python3 ~/.claude/dispatch-opencode/bin/lane_state.py <checkout>`. Python's stdlib `sqlite3` is
enough; one python3 call instead of a shell pipeline, because Bash permission rules match per
shell-operator segment.

Open the database from §5a's `opencode debug paths` as `file:<path>?mode=ro` — read-only, **not**
`immutable` (§0.3).

**Finding the lane's session** (observed schema):

```sql
SELECT id, title, time_updated
  FROM session
 WHERE directory = ?          -- the lane's checkout, absolute
   AND parent_id IS NULL      -- exclude subagent sessions
 ORDER BY time_updated DESC
 LIMIT 1;
```

`parent_id` matters: opencode's subagents get their own session rows, and steering one of those
instead of the lane's own session would be invisible from the outside. Record the session id.

**Reading the last assistant turn.** The schema has moved: this version defines both a legacy
`message(id, session_id, time_created, data)` table and a newer
`session_message(id, session_id, type, seq, data)`. On this machine only the legacy table holds rows,
so the **legacy shape is observed and the new one is unverified**. Query `session_message` first
(`ORDER BY seq DESC`), fall back to `message` (`ORDER BY time_created DESC`), and in both cases parse
the `data` column as JSON looking for an entry with `role == "assistant"` carrying:

- `tokens` — `{input, output, reasoning, cache: {read, write}}` plus `total` (observed: `total`
  equals `input + output + reasoning + cache.read`, i.e. the size of that call's context);
- `time` — `{created, completed}` in epoch milliseconds;
- `finish` — observed values `"stop"` (the turn ended) and `"tool-calls"` (more work follows);
- `modelID`, `providerID`, `cost` for the report.

If neither table yields an entry with those keys, **do not guess**: report `probe: unavailable` with
which table was tried, and let §6c fall back to `agent_status`, the session row's `time_updated`, and
git state. A probe that silently returns wrong numbers is worse than one that admits it is blind.

The helper reports the probe contract's fields:

- `turn_state` — `complete` when the newest assistant entry has `finish == "stop"` **and**
  `time.completed` is present; `working` when `finish` is `"tool-calls"` or `time.completed` is
  missing; `unknown` otherwise (surfaced, never guessed).
- `used_pct` — **always `null`**. Opencode publishes no context window on disk, so there is no
  denominator (§1). Report `context_tokens` = the entry's `tokens.total` instead, and let
  `--compact-at` decide anything that depends on it.
- `compactions` — **`null`**. There is no verified compaction marker in this schema; the
  `session_context_epoch` table looks related but is **(unverified)** as a counter. Say `null` rather
  than reporting a number you cannot defend, and count the `/compact` commands *you* sent in the
  state file instead — that is a fact you own.
- `mtime` — the session row's `time_updated` (epoch ms), for stall detection.
- `probe` — `ok` / `unavailable` with a reason.
- `todo` (opencode-only, observed) — `SELECT content, status FROM todo WHERE session_id = ? ORDER BY
  position`. Opencode maintains its own task list per session; it is an independent read on what the
  lane thinks it is doing, and §6e uses it to cross-check `progress.md`. A `todo` list with unfinished
  rows while `.dispatch/DONE` exists is a contradiction worth blocking on.
- report-only extras: `session.title` (opencode auto-titles sessions — a title that has nothing to do
  with the lane's objective is an early sign the lane misread its brief), and `cost`.

`opencode stats --project ''` reports token and cost statistics for the current project and is worth
one call at report time, not every sweep.

---

## §6c Classify each lane, in this order

| Class | Test | Action |
| --- | --- | --- |
| `terminal` | phase is `published`, `failed`, user-paused, or `verified` with a recorded §6f degrade reason | Skip — report only; never re-verify, re-publish, or prompt a closed lane |
| `unpublished` | phase is `verified`, `publish_attempts` < 3, no recorded degrade reason, and `pushed_sha` is missing or behind the branch HEAD, or `pr_url` is missing with PRs enabled | Retry publish (§6f) |
| `done` | `.dispatch/DONE` exists **and** `turn_state == complete` | Verify (§6e), publish (§6f), mark terminal |
| `blocked` | `agent_status == blocked` | Read `--source visible`, handle (§6g) |
| `blind` | `probe == unavailable` for two consecutive sweeps | The database is not answering for this lane. Read `--source visible` once, judge from git state, and escalate rather than steering a lane you cannot see |
| `help_pending` | a help request (§6j) is open, escalated, or answered but not yet delivered, **and** `turn_state == complete` | Answered and undelivered → deliver (§6d); otherwise leave it — the lane is waiting on you or the user, so do not send a continuation prompt and do not count it toward `no_progress` |
| `stalled` | `state_change_seq` **and** `mtime` both unchanged ≥ 15 min **and** `turn_state == working` | A turn that started and then froze — usually a provider stall, a permission prompt under `--no-yolo`, or a tool waiting on input. Read `--source visible` once: a dialog → §6g; a wedged tool → escalate; never score as finished |
| `hot` | `--compact-at N` is set, `context_tokens ≥ N`, **and** `turn_state == complete` | Compact (§6h) |
| `idle_incomplete` | `turn_state == complete` **and** no `DONE` file | **The engine.** Read `progress.md` and the `todo` rows, send a specific continuation prompt (§6d), record the nudge |
| `working` | otherwise | Leave it alone |

An `unknown` `agent_status` is an anomaly to surface, never a completion — do not let it fall
through to `working`'s leave-it-alone.

**`idle_incomplete` is the common case, not the exception.** In the goal-mode dispatchers this row is
a repair path; here it is how work happens. Expect most lanes to match it on most sweeps, and expect
the run's pace to be roughly one turn per sweep interval — that is what §3's second line promised the
user, and it is why §7's timer must stay armed.

One known race, by design: a notify-back can arrive before the ringing lane's final turn closes (the
brief fires it right after DONE is written, mid-turn), so that lane may still read `working` with
DONE present — re-check it once at the end of the sweep, or leave it to the next tick; both are fine.

---

## §6d Steer a lane — writing the continuation prompt

This is the most consequential thing the sweep does in this skill. Every prompt below is subject to
§6b's prompt guard: read the pane first, and if a dialog or selection list is parked there, resolve
it instead of typing.

**Read `.dispatch/progress.md` and the session's `todo` rows first.** A bare "continue" burns a turn
re-deriving state the lane already wrote down, and the two sources together catch the case where they
disagree. A good continuation prompt names, in one or two sentences:

- the next unchecked checklist item, quoted from `progress.md`;
- the file or module the §3-confirmed plan says that item touches;
- the acceptance criterion it has to satisfy;
- and the reminder to update `progress.md` and, when everything holds, to write `.dispatch/DONE` and
  run the notify-back.

When `progress.md` is missing or stale (its checklist state contradicts `git log`, or the `todo` rows
tell a different story), say so in the prompt and ask the lane to rewrite it from the actual tree
before continuing — an out-of-date progress file misleads every later sweep, including the one that
judges §6e.

**Anti-loop accounting is mandatory.** With prompts as the engine, a lane that has quietly stopped
responding looks exactly like a lane that is working. So with every nudge record, in the lane's state
entry: `nudges` (a count), the branch HEAD sha, and a hash of `progress.md`. On the next sweep:

- new commit or changed `progress.md` → the nudge worked; reset the no-progress counter.
- neither changed → increment `no_progress`. At `no_progress == 1`, nudge once more, differently:
  quote the exact blocker if the pane shows one, or narrow the ask to a single file. At
  `no_progress == 2`, **stop nudging**: record `escalated` with the last two prompts and what
  `progress.md` last said, and report the lane as awaiting-user. Never fire a third identical nudge.

**Delivering a help answer (§6j).** A lane that is mid-turn (`turn_state == working`) gets the file
only: it checks `.dispatch/help/` after every checklist item and at the start of every turn (§5b). A
lane at rest gets the answer prompt, which **replaces** this sweep's continuation prompt — one
prompt per lane per sweep — and counts as a nudge for the anti-loop accounting above:

    herdr agent prompt <lane> "Help answer for H-<n> is in .dispatch/help/H-<n>.answer.md. Read it, apply it, append 'applied H-<n>' to .dispatch/progress.md, then continue with <next unchecked item> in <file>."

Fill the tail from `progress.md` exactly as for any continuation prompt. While a lane's only open
request is unanswered or escalated, `help_pending` keeps it out of `idle_incomplete`: nudging a lane
that is correctly waiting would only make it guess.

**When the lane reports a real blocker** outside the help-request protocol — in `progress.md` or on
screen, a missing secret, a broken upstream, a contradiction in the task — do not improvise scope. If the answer is inside the §3-confirmed plan (a decision the
brief already made, a misread step), send the correction as a normal prompt. Otherwise escalate with
the blocker quoted (record `escalated`).

**Pausing a lane** is a user decision, not a sweep's: record
`pause: {origin: user, at: <sweep time>}` in the state file and stop nudging it. Such a lane is
terminal for §7 until the user says otherwise.

---

## §6g Handle a blocked lane

Under `--auto` most tool calls never prompt, so `blocked` there is usually a dialog opencode raises
for something `--auto` does not cover — a permission the config explicitly denies, or a first-run
prompt — or a herdr misclassification. Under `--no-yolo`, prompts are the normal case and this
section is the hot path. Read the pane before assuming which:

    herdr agent read <lane> --source visible

Opencode's permission dialogs and their hotkeys are **(unverified)** in this driver, so answer only
with `send-keys` matching the option actually displayed. Never type a remembered key, and never send a
prompt at a parked list — it presses the highlighted default.

- Benign and inside the lane's own checkout (edit its files, run its tests, read files) → approve the
  narrowest option offered. Prefer a once-only approval over any "always/remember" option: the latter
  is written into opencode's `permission` table and outlives this run.
- The one sanctioned exception below: under `--no-yolo` a finishing lane's notify-back (§5b) surfaces
  here as a permission request for a `herdr agent prompt` command. If the quoted command is
  **exactly** the notify-back — aimed at the recorded `orchestrator_pane`, carrying this run's id and
  the lane's own name, with nothing chained after it — approve it: you briefed it, and the sweep
  reading this dialog is already the sweep it was trying to summon. Any variation — another pane,
  another run id, an extra `;`/`&&` command — is not the notify-back and falls through to the rule
  below. This also means that under `--no-yolo` the doorbell rings late by design.
- Anything leaving the lane's blast radius — `git push`, `gh pr create`, force operations, `sudo`,
  deleting outside the checkout, reading credentials, writing to a network target → **do not answer**.
  Pause the lane, record it, surface it to the user. Publishing being a normal part of this run (§6f)
  does not make it approvable *here*: §6f runs after verification, on your side of the fence. A lane
  asking to push is a lane that misread its brief.

`herdr agent prompt` returns `agent_blocked` while a dialog is up, so clear it with `send-keys` first.

---

## §6h Compact a lane

    herdr agent prompt <lane> "/compact" --wait --until idle --timeout 120000

`/compact` exists in this opencode build but its runtime behaviour here is **(unverified)** — the
command name was read out of the binary, not exercised. So treat the first use in a run as a probe:

- **Only between turns.** Gate on `turn_state == complete` from §6a. A slash command sent mid-turn is
  refused; take the refusal, clear the composer, retry next sweep.
- **Judge it by effect.** §6a cannot count compactions (`compactions: null`), so the evidence is
  indirect: the *next* assistant turn's `tokens.total` should drop sharply. Record the before and
  after in the state file and report both in §8.
- **If it does not work** — herdr reports the prompt stalled, the pane shows an unknown command, or
  the next turn's tokens do not drop — record `compact_unavailable` once, stop trying it for the rest
  of the run, and use the restart recipe instead when a lane gets too large. Do not retry a command
  this build does not have, sweep after sweep.
- **After compacting, re-prime by prompt.** Unlike a goal-mode agent, opencode has nothing to resume
  on its own: a compacted lane is idle, so send §6d's continuation prompt in the same sweep, built
  from `progress.md` — which is exactly why §5b insists that file stays current.
- **Stop after 3 compactions**, and hand off to the restart recipe.

**The restart recipe — a fresh session on the same pane, for a lane whose own agent is alive:**

1. Record `restarting: true` on the lane and rewrite the state file, so the next sweep does not fire
   the same row again off the old session while the restart is mid-flight.
2. `herdr agent prompt <lane> "/new"` — the composer returns to an empty new session.
3. Re-prime with §5c's exact priming prompt. The fresh session has no history, so `.dispatch/TASK.md`
   and `.dispatch/progress.md` are the entire handover — check `progress.md` is current *before*
   sending, and if it is not, that is an escalation, not a restart.
4. Re-identify the session: §6a's query returns a **new** session id for the same directory. Poll it
   until the id **differs** from the recorded one, then record the new id and clear `restarting`.
   Never write back an id you have not seen change — that leaves the sweep reading a dead session and
   restarting forever.

If the pane does not respond to `/new`, the opencode process itself is gone or wedged: escalate to
the user; never `worktree remove` (§0.7), and relaunching via `agent start` needs the pane back at a
shell prompt first (§5a's checks apply again).
