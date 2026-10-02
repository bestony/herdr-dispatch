# Antigravity driver — §5a, §5c, §6a, §6c, §6d, §6g, §6h

Everything in `dispatch-antigravity` that depends on the Antigravity CLI (`agy`) itself. The
agent-independent halves are `references/plan.md` (§2–§4, §5b) and `references/supervise.md` (§6b,
§6e, §6f, §6i, §7, §8); §0 and §1 are in `../SKILL.md`.

Sources: `agy --help`, the release notes agy prints with `agy changelog`, and one live lane run
under herdr. Claims marked **(verified)** were checked against agy **1.2.14** under herdr **0.9.3**
on this machine. Claims marked **(observed)** were read off real on-disk records rather than
documented; treat them as reliable but version-bound. Anything marked **(unverified)** has not been
exercised end-to-end — handle its failure path, never assume it works. §2 records the version it
actually finds; a mismatch is a report line, not an abort.

---

## §5a Pre-flight the pane

A fresh worktree is not a working environment: no `node_modules/`, no `.env`, no `.venv`, because
those are untracked or ignored.

    herdr pane send-text <pane> 'agy --version'
    herdr pane send-keys <pane> enter

Run the version, not `command -v`: a shim or a stale PATH entry can resolve while the real binary
does not, and then `agent start` just times out. `agy --version` prints the bare version
(`1.2.14`, verified) and exercises the real resolution path.

There is no `herdr pane run`: `pane send-text` only types the text, and `pane send-keys <pane> enter`
submits it. Then read the result with `herdr pane read <pane> --source visible`. On herdr 0.9.3,
`herdr pane wait-output` searches the output already on screen before it polls, so it also finds a
command that already finished (verified). Do not wait on the tool name alone: the echoed command
line contains it too.

- `agy` not found, or the version refuses to resolve → stop that lane and report it.
  `agent start --kind agy` resolves the executable from the pane's own login shell and args cannot
  redirect it.
- Dependencies missing → run the repo's install command in the pane and let it finish **before**
  starting the agent; `agent start` needs a pane at an idle interactive prompt.
- `.env` or other secrets missing → **ask the user** whether to symlink them. Never copy secrets.

**Once per run (§2's agent preflight), read agy's posture from disk** — read-only, never edited
(§0.6):

- `~/.gemini/antigravity-cli/settings.json` → `toolPermission` (e.g. `always-proceed`),
  `allowNonWorkspaceAccess`, and `trustedWorkspaces`.
- the active project: its id is in `~/.gemini/antigravity-cli/cache/default_project_id.txt`
  (observed: `default-cli-project`), its file is `~/.gemini/config/projects/<id>.json`. Record
  `settings.permissionPreset`, `settings.autoExecutionPolicy`, `settings.sandboxMode`, and every
  `projectResources.resources[].gitFolder.folderUri`. Project settings take precedence over the
  global settings file (agy release notes).

**The project-folder trap (verified).** agy adds the active project's resource folders to every
session's workspace list **ahead of** the pane's cwd. In the live test the project held
`file:///Users/<user>/Developer`; the lane's conversation recorded that folder as its first
workspace, its first `pwd` printed that folder rather than the pane's cwd, and it wrote a file
there as well as in its own checkout. So a recorded folder outside the lane checkout is a real
risk: name it in §3's posture line, and rely on the absolute-path rule that §5c adds to both the
brief and the goal objective. Do not pass `--new-project` to avoid it: that creates a persistent
project per lane under `~/.gemini/config/projects/`, which outlives the run.

Then write the brief (§5b in `references/plan.md`) before launching. **This driver strengthens the
brief's boundaries** with one more line, substituted with the checkout's absolute path:

> **Working directory.** Your checkout is `<checkout>` (absolute path). Run every shell command with
> `<checkout>` as its working directory — `cd <checkout>` first, or pass it as the command's cwd —
> and write files only under it. Other folders may appear in your workspace list; they are not
> yours. Never read or prompt another herdr pane, and never read files outside `<checkout>` except
> the ones this brief names.

---

## §5c Launch agy and set the goal

Launch with **no positional prompt and no `-i`**. The lane is driven by an agy **goal**, which is
set by slash command, and priming through `herdr agent prompt` is what lets you verify the composer
is actually accepting input first — the trust dialog below eats whatever reaches it.

    herdr agent start <lane> --kind agy --pane <pane> --timeout 120000 -- \
      --dangerously-skip-permissions

Why each part:

- `--dangerously-skip-permissions` is **on by default**: lanes run unattended and an approval
  prompt stalls a lane until the next sweep notices it. Drop it only under `--no-yolo`, and read
  §1's note first: agy has no flag that turns prompting back *on*, so if the user's settings or
  project already auto-approve, `--no-yolo` changes nothing — §3 must say so.
- **Permission bypass is not a sandbox.** agy has a separate `--sandbox` flag (terminal
  restrictions) and a project-level `sandboxMode`. This skill does not touch either; whatever the
  user configured stays in force. Report `sandboxMode` and `allowNonWorkspaceAccess` in §3 rather
  than implying an isolation that is not there — with the sandbox off and non-workspace access
  allowed, the worktree and the brief are the only boundaries.
- `--timeout 120000` because a cold start plus MCP and plugin boot can take far longer than herdr's
  30 s default.
- Never pass `-p` / `--print`, `--output-format` or `--input-format`: those are headless modes, and
  this skill needs a live TUI that herdr can read and prompt.
- Never pass `--conversation` / `-c` (`--continue`): a lane starts a fresh conversation in its own
  checkout; continuing "the most recent conversation" could attach it to a different one.

**The three lines §3 owes the user, in this driver's words:**

- approval posture — `yolo（--dangerously-skip-permissions，工具调用不再询问）` by default, or
  `--no-yolo（不传绕过参数，按你自己的 agy 权限配置执行）`; when the config auto-approves anyway, add
  `注意：你的 agy 配置（toolPermission=<v> / permissionPreset=<v>）本身就会自动批准，--no-yolo 实际不生效`;
  plus `sandboxMode`, `allowNonWorkspaceAccess`, and any project folder outside the checkout
  (`agy 当前 project 还挂着 <folder>，会出现在每个 lane 的 workspace 里，已在 brief 和 goal 里强制绝对路径`);
- how the lane is driven — `以 agy goal 模式运行（/goal 长任务）：按上面的实施计划执行、以验收标准为
  完成判据，在一次长运行里持续推进，直到 agy 自己判定目标完成；中途被中断或限流时由监督循环重新挂上 goal`;
- rate limits — all lanes share one Google account, so its model quota is shared and can throttle
  every lane at once; agy stops at once on a daily or billing cap instead of retrying (release
  notes), and a throttled lane shows up as §6c's `rate_limited` row. `/usage` in the pane shows the
  numbers.

**Then verify readiness yourself — `agent start` returning `agent_started` with
`agent_status: idle` and `interactive_ready: true` does NOT mean agy is ready (verified).** On any
directory not in `trustedWorkspaces` — i.e. every fresh worktree — agy sits on a modal:

    Do you trust the contents of this project?
    > Yes, I trust this folder
      No, exit

Read the pane:

    herdr agent read <lane> --source visible

If it shows `Do you trust the contents of this project`, confirm the highlighted
`Yes, I trust this folder` with `herdr agent send-keys <lane> enter` once (verified), then re-read
until the version banner and the empty `>` composer are visible. Only then is the lane ready for
input. Trusting appends the checkout path to `trustedWorkspaces` in the user's settings file
(verified) — a side effect §8 reports, never one you undo by editing that file. On
`agent_not_ready` the name still resolves — read, resolve the dialog, continue.

**Then set the goal** — one call, and only after the composer is visible:

    herdr agent prompt <lane> "/goal Work through .dispatch/TASK.md in <checkout>: run every command with <checkout> as the working directory and write files only under it; follow the plan in order, satisfy every acceptance criterion, keep .dispatch/progress.md updated after every checklist item, and finish by writing .dispatch/DONE and running the notify-back command TASK.md gives you."

Substitute `<checkout>` with the absolute path (§5a's project-folder trap). The slash command
executes — it does not land as literal text — and the objective rides in the same call (verified):
the pane shows the objective being restated and pursuit starts, the conversation's run status turns
`CASCADE_RUN_STATUS_RUNNING`, and the transcript records the `/goal …` text as the first
`USER_INPUT` step.

What an agy goal is, and what it changes for supervision:

- `/goal` is described by agy as "Run until the specified goal is completely finished", and its
  release notes removed any cap on how long a goal may run. Unlike codex and grok, the pursuit is
  **one long run**: the run stays `RUNNING` across many model steps and ends only when agy judges
  the goal met, is interrupted, or hits an error. A goal it judges met ends with a
  `<!-- GOAL_COMPLETE -->` marker in the final `PLANNER_RESPONSE` step (verified).
- So a lane whose run is `IDLE` **without** that marker after its last `/goal` stopped short — an
  interrupt, a quota error, or a model that gave up — and §6d re-engages it. A lane whose run is
  `IDLE` **with** the marker but no `.dispatch/DONE` met the goal by agy's reckoning while the
  completion protocol did not finish — the safe case for a continuation prompt.
- There is no `/goal status`, `pause`, `resume` or `clear` (§0.5). `esc` interrupts a running goal
  (verified: `Interrupted · What should Antigravity CLI do instead?`); that is the only pause.

Record the lane as phase `implementing` with `goal: set` and its conversation id (§6a) in the state
file. If the pane shows an unknown command, or the `/goal …` text landed in the composer as literal
input, record `goal_skipped` with the reason, surface it in §8, and prime with the one-shot prompt
instead — the sweep's nudges are then the engine:

    herdr agent prompt <lane> "Read .dispatch/TASK.md in <checkout> and work through its checklist, running every command with <checkout> as the working directory. Keep .dispatch/progress.md updated after every item. Write .dispatch/DONE when everything is finished and verified, then run the notify-back command TASK.md gives you."

Start every lane before supervising any of them.

---

## §6a Probe one lane

Write `~/.claude/dispatch-antigravity/bin/lane_state.py` once if it is absent, then call
`python3 ~/.claude/dispatch-antigravity/bin/lane_state.py <checkout> <conversation-id>`. One python3
call instead of a shell pipeline, because Bash permission rules match per shell-operator segment.

**Finding the lane's conversation.** Three sources, all verified to agree:

- `herdr agent get <lane> | jq -r '.result.agent.agent_session.value'` — herdr reports agy's
  conversation id (`source: herdr:antigravity_cli`). This is the identity check §6b runs.
- `~/.gemini/antigravity-cli/cache/last_conversations.json` — a map from absolute cwd to the newest
  conversation id started there. Lane checkouts are unique, so this is an O(1) lookup that needs no
  scan.
- `conversation_summaries.db` → the row whose `conversation_id` matches; its `workspace_uris` (a
  JSON list of `file://` URIs) must contain the checkout. It may also list the project folders from
  §5a — that is expected, not a mismatch.

A new conversation (first launch, or §6h's `/clear`) shows up in `last_conversations.json` only
after its **first turn** (verified). Before that, `probe` is `unavailable` — normal for a lane that
was just primed, an anomaly for an old one.

The helper opens `~/.gemini/antigravity-cli/conversation_summaries.db` as `file:<path>?mode=ro`,
falling back to `immutable=1` only under §0.3's no-`-wal` condition (verified), selects the one row by `conversation_id`, then reads only the **tail** (~2 MB) of
`~/.gemini/antigravity-cli/brain/<conversation-id>/.system_generated/logs/transcript.jsonl` (§0.8).
Each transcript line is one step: `step_index`, `source` (`USER_EXPLICIT` / `MODEL` / `SYSTEM`),
`type`, `status`, `created_at`, and `content` / `error` / `thinking` (observed). It reports the
probe contract's fields:

- `session` — the conversation id.
- `turn_state` — from the summary row: `status == CASCADE_RUN_STATUS_RUNNING` or
  `not_fully_idle == 1` → `working`; `CASCADE_RUN_STATUS_IDLE` → `complete`; anything else
  (including an empty status) → `unknown`. `killed == 1` is reported as an anomaly (observed field;
  semantics unverified).
- `used_pct` — **always `null`**. agy does not write context usage to disk; it is shown only in the
  `/context` panel (verified), which is a pane interaction, not a probe. agy compacts on its own
  (below), so the sweep does not need the number to decide anything.
- `compactions` — count of `SYSTEM` / `CHECKPOINT` steps in the transcript; each one's content opens
  with `# Resuming from a compaction` (observed). The transcript keeps earlier checkpoints after a
  later compaction (observed: two in one file). Count the whole tail; if the tail window ever cuts
  off the oldest one, the number only undercounts, which §6c tolerates.
- `mtime` — newest mtime among the transcript and the summary row's `last_modified_time`.
- `probe` — `ok`, or `unavailable` with a reason (no row, no transcript yet, conversation id not
  in `last_conversations.json`).
- `goal` (agy-only) — `set` when the newest `USER_INPUT` step whose content's `<USER_REQUEST>`
  block starts with `/goal` exists, else `none`; `objective` is the text after `/goal` (empty for
  a bare re-engage).
- `goal_complete` (agy-only) — `true` when a `PLANNER_RESPONSE` step **after** that newest `/goal`
  input contains `<!-- GOAL_COMPLETE -->` (verified).
- `last_error` (agy-only) — the newest `ERROR_MESSAGE` step after the newest `USER_INPUT`, with its
  `error` text and `created_at`. A quota hit reads
  `API error (attempt N): RESOURCE_EXHAUSTED (code 429): … Resets in 4h42m25s.` (observed); parse
  `Resets in` into an absolute `retry_after` from `created_at`.
- report-only extras from the summary row: `title`, `step_count`, `last_user_input_time`,
  `project_id`, `agent_name`.

---

## §6c Classify each lane, in this order

| Class | Test | Action |
| --- | --- | --- |
| `terminal` | phase is `published`, `failed`, user-paused, or `verified` with a recorded §6f degrade reason | Skip — report only; never re-verify, re-publish, or prompt a closed lane |
| `unpublished` | phase is `verified`, `publish_attempts` < 3, no recorded degrade reason, and `pushed_sha` is missing or behind the branch HEAD, or `pr_url` is missing with PRs enabled | Retry publish (§6f) |
| `done` | `.dispatch/DONE` exists **and** `turn_state == complete` | Verify (§6e), publish (§6f), mark terminal |
| `blocked` | `agent_status == blocked` | Read `--source visible`, handle (§6g) |
| `identity_mismatch` | herdr's `agent_session.value` ≠ the recorded conversation id with no §6h restart in flight, or the conversation's `workspace_uris` lacks the checkout | Surface it; steer nothing (§6b) |
| `rate_limited` | `turn_state == complete`, no `DONE`, and `last_error` is a `RESOURCE_EXHAUSTED` / 429 | Before `retry_after`: leave it, report the reset time. After it: re-engage (§6d) once per sweep |
| `help_pending` | a help request (§6j) is open, escalated, or answered but not yet delivered, **and** `turn_state == complete` | Answered and undelivered → deliver (§6d); otherwise leave it — the lane is waiting on you or the user, so never nudge it or escalate it as stuck |
| `stalled` | `state_change_seq` **and** `mtime` both unchanged ≥ 15 min | Read `--source visible` once: a parked selection list or dialog → §6d/§6g; an idle composer over unfinished work → the `idle_incomplete` action; a run still spinning on one step → escalate; never score as finished |
| `idle_incomplete` | `turn_state == complete` and no `DONE` file | `goal_complete` → continuation prompt; otherwise re-engage the goal (§6d). Record the nudge; a re-nudge with no progress since the last one escalates instead |
| `working` | otherwise | Leave it alone |

An `unknown` `agent_status` or `turn_state` is an anomaly to surface, never a completion — do not
let it fall through to `working`'s leave-it-alone.

**There is no `hot` row.** agy compacts by itself — the `CHECKPOINT` steps in §6a — and 1.2.14 has
no `/compact` command (verified: absent from the slash-command list). `compactions` is a report
line; a lane with **4 or more** compactions that then trips `idle_incomplete`'s anti-loop rule is
handed to §6h's restart recipe instead of being escalated, because a fresh conversation primed from
`.dispatch/` is the only context reset available.

**On a goal lane, `turn_state == working` really is working.** agy pursues a goal inside one run, so
unlike the codex and grok drivers there is no "between rounds" gap that looks idle: an `IDLE` run
means the pursuit ended (§5c). That is why `idle_incomplete` needs no goal-status question first,
and why a frozen `mtime` under a `RUNNING` status is what proves a lane is wedged.

One known race, by design: a notify-back can arrive before the ringing lane's run closes (the brief
fires it right after DONE is written, mid-run), so that lane may still read `working` with DONE
present — re-check it once at the end of the sweep, or leave it to the next tick; both are fine.

---

## §6d Steer a lane

All input goes through `herdr agent prompt <lane> "…"`, and every call is subject to §6b's prompt
guard: read the pane first, and if a selection list or dialog is parked there, resolve it instead
of typing.

**Re-engaging a goal** — the repair path for a pursuit that stopped short (an interrupt, a quota
error after its reset time, a model that gave up). Read `.dispatch/progress.md` first, then send
§5c's `/goal` call again, with one sentence prepended to the objective naming where to resume:

    herdr agent prompt <lane> "/goal Resume at <next unchecked checklist item> (see .dispatch/progress.md). Work through .dispatch/TASK.md in <checkout>: …same objective as §5c…"

Only re-engage a lane whose run is `IDLE` — never send `/goal` into a `RUNNING` pursuit. A second
`/goal` over a finished run started pursuit with no dialog in the live test (verified for a bare
`/goal`; a full objective over an earlier one is **unverified** for any replace prompt), so the
prompt guard's pane read after sending is mandatory: if a dialog appears, answer it with
`send-keys` by what it displays, choosing to keep working toward the TASK.md objective.

Record `nudges`, the branch HEAD sha and a hash of `progress.md` with each re-engage: one that
produces neither a new commit nor a changed `progress.md` by the next sweep means the lane is not
responding, and the second such nudge escalates instead of firing a third (or goes to §6h's restart
when §6c's compaction rule applies).

**Goal complete, but no `.dispatch/DONE`** — agy's pursuit ended with `<!-- GOAL_COMPLETE -->`
while the completion protocol did not finish. With no run in flight there is nothing to collide
with; send a plain continuation prompt naming exactly what is missing (the unticked items, the
failing acceptance criterion, the missing DONE or notify-back), not another `/goal`.

**Nudging a `goal_skipped` lane** — the engine for the one-shot regime. Read `.dispatch/progress.md`
first and write a prompt that names the next unchecked checklist item and the file it touches; a
bare "continue" wastes a turn. Same anti-loop record as above.

**Pausing.** agy has no `/goal pause`. **You** pause a lane only when the user asks: send
`herdr agent send-keys <lane> escape` (verified to interrupt a running goal), then record
`pause: {origin: user, at: <sweep time>}` in the same sweep — the lane is user-paused, terminal for
§7, and never re-engaged by a sweep until the user says so. A run that stopped short with no such
record is treated as a stop, not a pause, because agy leaves no trace that tells a keyboard `esc`
apart from a model that ended its run; if the user says they interrupted it themselves, record the
pause then.

**Delivering a help answer (§6j).** The answer file is already on disk; this decides whether the
lane also needs a prompt.

- `turn_state == working` → file only. A goal pursuit is one long run, and the lane checks
  `.dispatch/help/` after every checklist item (§5b) — never prompt into a `RUNNING` run.
- turn complete, lane has a goal → re-engage the goal as above, with the resume sentence replaced
  by `First read .dispatch/help/H-<n>.answer.md, apply it, and append 'applied H-<n>' to
  .dispatch/progress.md.` One call both delivers the answer and restarts pursuit.
- turn complete, `goal_skipped` → the answer prompt:

      herdr agent prompt <lane> "Help answer for H-<n> is in .dispatch/help/H-<n>.answer.md. Read it, apply it, append 'applied H-<n>' to .dispatch/progress.md, then continue the plan."

---

## §6g Handle a blocked lane

Under the default `--dangerously-skip-permissions` agy raises no approval prompts, so `blocked`
there is an anomaly — a trust dialog (§5c), a sign-in or MCP authentication screen, a settings
panel, or a herdr misclassification. Read the pane before assuming which. The rules below apply in
full under `--no-yolo` with a config that actually prompts.

Read with `--source visible`. agy's approval prompts state what they ask — `Run this command?`,
`Allow access to this URL?`, `Allow calling this tool?` — and may carry a `Reason:` line (release
notes); their option lists and hotkeys are **(unverified)** here, so pick by what is displayed and
answer with `send-keys`, never with a remembered key or a prompt (a prompt submitted at a parked
list presses its highlighted default).

- Benign and inside the lane's own checkout (edit its files, run its tests, read files) → approve the
  narrowest option offered. Prefer a once-only approval over an allow-always one: allow-always
  writes a permanent rule into the user's agy permissions and outlives this run.
- The one sanctioned exception below: under `--no-yolo` a finishing lane's notify-back (§5b) surfaces
  here as an approval request. If the quoted command is **exactly** the notify-back —
  `herdr agent prompt` aimed at the recorded `orchestrator_pane`, carrying this run's id and the
  lane's own name, with nothing chained after it — approve it: you briefed it, and the sweep
  reading this prompt is already the sweep it was trying to summon. Any variation — another pane,
  another run id, an extra `;`/`&&` command — is not the notify-back and falls through to the rule
  below. Under `--no-yolo` the doorbell rings late by design.
- Anything leaving the lane's blast radius — `git push`, `gh pr create`, force operations, `sudo`,
  deleting outside the checkout, reading or writing outside the checkout (including the project
  folders from §5a), reading credentials, reading another herdr pane, writing to a network target →
  **do not answer**. Pause the lane (§6d), record it, surface it to the user. Publishing being a
  normal part of this run (§6f) does not make it approvable *here*: §6f runs after verification, on
  your side of the fence. A lane asking to push is a lane that misread its brief.

`herdr agent prompt` returns `agent_blocked` while a dialog is up, so clear it with `send-keys` first.

---

## §6h Restart a lane

agy has no manual compaction (§6c), so this section is only the restart recipe. It runs for a lane
whose own agent is alive but whose conversation is no longer useful: §6c's 4-compaction anti-loop
handoff, or a conversation that keeps failing on the same internal error. A §6b id mismatch is
*not* a reason — §6b routes each of its cases itself.

1. Record `restarting: true` on the lane and rewrite the state file, so the next sweep does not
   fire the same row again off the old conversation while the restart is mid-flight.
2. `herdr agent prompt <lane> "/clear"` (alias `/new`, "Clear conversation and start a new one",
   verified) — the composer returns empty on a fresh conversation. Only when the run is `IDLE`.
3. Re-prime exactly as §5c does, in the same regime the lane was in: the `/goal` call for a goal
   lane, or the one-shot prompt otherwise. `.dispatch/progress.md` carries the lane's state across
   the reset.
4. Re-identify the conversation: the new id appears in `last_conversations.json` (and herdr's
   `agent_session.value`) only after the fresh conversation's **first turn** (verified). Poll until
   the id for the checkout **differs** from the recorded one, then record the new id and clear
   `restarting`. Never write back an id you have not seen change — that leaves the sweep reading a
   dead conversation and restarting forever.

If the pane does not respond to `/clear`, the agy process itself is gone or wedged: escalate to the
user; never `worktree remove` (§0.7), and relaunching via `agent start` needs the pane back at a
shell prompt first (§5a's checks apply again). `/quit` exits agy cleanly when the user asks for a
lane to be stopped.
