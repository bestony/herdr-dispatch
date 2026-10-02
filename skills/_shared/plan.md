# Dispatch — §2 to §5b (agent-independent)

Shared by every `dispatch-*` skill in this plugin. Nothing here depends on which coding agent runs
the lanes: it covers the run's bookkeeping (§2), the lane plan the user confirms (§3), the herdr
workspaces (§4), and the brief written into each lane's checkout (§5b).

What *is* agent-specific — pre-flighting the pane (§5a) and launching/priming the agent (§5c) —
lives in the calling skill's `references/driver.md`. §0 (invariants) and §1 (gate and parse) live in
its `SKILL.md`; §6–§8 live in `references/supervise.md`. Section numbers are continuous across all
four files, so a cross-reference means the same thing wherever you are.

Throughout, **the agent** means whichever CLI this skill dispatches (codex, grok, opencode…),
and **the driver** means that skill's `references/driver.md`.

---

## §2 Repo, run id, state file

Resolve `git rev-parse --show-toplevel`. Empty means "not a repo" — there is nothing to branch a
worktree from, so stop and tell the user in Chinese that dispatch needs a git repo. Do not fall back
to sharing the cwd. If `git rev-parse --git-dir` and `--git-common-dir` differ you are in a linked
worktree; stop and ask the user to re-run from the main checkout.

Per `~/CLAUDE.md`, run `git -C <repo> fetch origin` first and prefer `origin/<branch>` as the base, so
lanes fan out from fresh upstream rather than a stale local branch. With no remote, use the local
branch and say so in the plan.

Run id: six lowercase alphanumerics derived from `date +%s`. State dir
`~/.claude/<skill-name>/<run-id>/` — `<skill-name>` is this skill's own name, so two dispatchers
running different agents over the same repo never share a state tree. `mkdir -p` it. If another run
dir for this repo still holds non-terminal lanes, do not silently start a second one — ask the user
whether to resume, archive, or abort.

Record in the state file, before anything is created: `skill` (this skill's name), `agent_kind` (the
herdr `--kind` the driver launches), `flags`, `repo`, `base`, and your own coordinates —
`orchestrator_pane` from `$HERDR_PANE_ID` (always set when the §1 gate passed). Lanes notify
completion by prompting this pane (§5b), and each brief needs the id verbatim — never leave a lane
to find its orchestrator with `herdr agent list`. The id stays valid until the pane closes or is
moved to another workspace (a move re-qualifies it), and it never proves *who* is inside: if this
session ends, the §7 timer dies with it, later rings error out — or land as unsolicited text in
whatever session occupies the pane by then — and the run comes back only when a human re-runs this
skill with `--resume`; that resumed sweep re-arms the loop and re-records `orchestrator_pane` (§7),
restoring supervision, though briefs already written keep ringing the old pane — the timer covers
what those rings miss. Within a live session, the §7 timer is the backstop for every ring that
misses.

**Publishing preflight.** A lane finishes with *you* pushing its branch and opening its PR (§6f),
so settle *now* — before a single workspace exists — whether that will be possible, and record
each answer in the state file:

- does `origin` exist (`git -C <repo> remote get-url origin`);
- does `gh auth status` succeed, and does `gh repo view --json nameWithOwner -q .nameWithOwner`
  resolve the target repo;
- is the **PR base** — the `--base` ref with any `origin/` prefix stripped — an actual branch on
  origin (`git -C <repo> ls-remote --exit-code --heads origin <pr-base>`).

Do not install `gh`, do not authenticate it, and do not create the missing branch. None of these
failing is a reason to abort: they only narrow what §6f can do, and §3 must tell the user up front
which half they will be finishing by hand.

**Agent preflight.** Ask the driver's §5a what it needs verified once per run rather than per lane —
typically that the agent's executable resolves and which version it is. Record the version string:
every "(verified)" claim in a driver was verified against a specific version, and a mismatch is
worth one report line in §8, not an abort.

---

## §3 Plan the lanes

Split the task text on numbered items, newlines, or `;` — whichever the user actually used.

Then **group** them. This is the most consequential judgement in the skill:

- Same lane when tasks touch the same files or module, when one depends on another's output, or when
  a human would review them together. Inside a lane the tasks are an ordered checklist.
- Different lanes only when they can run concurrently without editing the same files. Merge cost is
  decided here, not at merge time.
- Never exceed `--lanes N` (default and hard max 16). If grouping yields more, merge the most related
  ones and say which and why. Do not pad the other way: three independent tasks make three lanes.
- When lanes > 4, warn the user that all lanes share **one account for this agent**, so its rate
  limits are shared and can throttle every lane at once. The driver names what that limit looks like
  when it is hit.

Per lane derive:

- `slug` — a 2–3 word kebab-case summary of the lane's work (e.g. `add-owner-filter`,
  `fix-ws-timeout`): lowercase letters, digits and `-` only, starting with a letter (reword a
  digit-leading summary). Describe the work, not the mechanics — the slug is the readable part of
  everything the user sees: lane name, workspace label, worktree, branch. Slugs are unique within
  the run: two similar tasks get distinguishing words (`fix-login-web`, `fix-login-mobile`), never
  the same slug twice — the branch below drops `<n>`, so two lanes sharing a type and issue would
  produce the same branch name.
- `name` — `<slug>-<n>` with `<n>` the lane number, e.g. `add-owner-filter-1`. herdr requires
  `[a-z][a-z0-9_-]{0,31}`; shorten an overlong slug by dropping whole words from the end — still
  letter-leading, no trailing hyphen — until the name fits 32 chars. Names carry no run id and are
  released only when an agent exits, so check `herdr agent list` and `herdr workspace list` before
  dispatch; on a collision, shorten further and append the run id's *last* three characters
  (`<slug>-<n>-<xyz>`, still within 32 — the tail of a timestamp-derived id is its fast-moving end,
  the head barely changes between same-day runs), and re-check until the name is actually free.
- `type` — the branch-type prefix that best fits the lane's overall work, with the usual
  Conventional-Commits meanings: `feature` (or `feat` — interchangeable in the branch, but in
  commit messages and PR titles the Conventional-Commits token is always `feat`) for a user-facing
  feature, `fix` for a user-facing bug fix, `docs`, `style`, `refactor`, `test`, or `chore`. One
  type per lane; a mixed lane takes the type of its primary objective.
- `issue` — the GitHub issue number the lane addresses, read from the task text (`#123`, a full
  issue URL, or wording like "issue 123"). Never invent one: a lane whose tasks name no issue has
  none, and the plan table below is where the user can supply it before anything is created.
- `branch` — `<type>/<issue>-<slug>`, the slug serving as the alias (the issue's keyword) — e.g.
  `feature/1-init`, `fix/42-ws-timeout`. When the lane has no issue number, drop that segment:
  `<type>/<slug>`.
  Nothing in this name embeds the run id, so it can collide with a branch that already exists —
  locally, on origin, or checked out in another worktree. Check
  `git -C <repo> branch --list <branch>`, `git -C <repo> ls-remote --heads origin <branch>` and
  `git -C <repo> worktree list` before dispatch; on a collision, append the run id's last three
  characters (`<type>/<issue>-<slug>-<xyz>`).
- `tasks` — the ordered checklist.
- `plan` — the lane's implementation plan, authored by **you**. The lane's agent does not plan for
  itself in this skill — it executes the plan you hand it — so this must be executable as written:
  ordered steps naming the files or modules each one touches, key decisions with a one-line
  rationale each, and where a route is genuinely open, the fork named with the default you chose, so
  the user is confirming a direction, not discovering one later. Ground every step in the repo — a
  quick Glob/Grep/Read of the code it names, never guesswork. Steps, not prose: the lane executes
  this top to bottom and records any forced deviation in `.dispatch/progress.md` (§5b).
- `acceptance` — the lane's acceptance criteria (presented to the user as 验收标准), authored by
  **you**: the explicit, checkable criteria that define done. Build them from the repo's real check
  commands — found in `package.json` / `Makefile` / `justfile` / CI config, never invented — each
  with its expected outcome, plus behavioural criteria stated concretely enough to verify from the
  diff or a command. A repo with no automated checks gets criteria over tree state and diff content
  instead, plus an explicit "no automated checks in this repo". These criteria are what the lane
  works toward, what gates its DONE (§5b), and what §6e later re-runs with its own hands — write
  nothing here you cannot check yourself.

Present the plan in Chinese: the lane table (lane / 分支 / 包含的任务), and under each lane its
实施计划 (the ordered steps) and 验收标准 bullets. Below that, state in **three** lines, so the user
sees all of them before anything is created:

- **which agent and which approval posture** the lanes will run in — the driver's §5c supplies the
  exact wording, including what `--no-yolo` does and does not change for this agent. Never describe
  a safety boundary this agent does not actually have.
- **how each lane is driven** — the driver's §5c supplies this too: an autonomous objective the
  agent continues on its own, or a single prompt that the supervision sweep re-nudges (§6c
  `idle_incomplete`). Say which, because it changes how long a silent lane may legitimately stay
  silent.
- **what happens when a lane finishes** — by default
  `验证通过后自动 push 到 origin，并对 <pr-base> 开 PR（ready for review，会触发 CI 和 reviewer 通知）`,
  or the degraded form the §2 preflight actually found (`--draft` 开草稿 PR / `--no-pr` 只 push，PR
  命令会打印出来 / `gh 未登录，只 push，PR 命令会打印出来` / `PR base 不在 origin 上，只 push，PR 命令
  留待你补 base` / `无 origin，只能本地提交`). Never promise a PR the preflight says you cannot open.

Then ask once with `AskUserQuestion`: 按此派发 / 计划或验收标准要改（在补充里说明改哪里） / 合并成
更少的 lane / 我来调整. Create nothing before that answer. The confirmation covers the lane split,
the plan **and** the acceptance criteria together — the confirmed `plan` and `acceptance` go
verbatim into each lane's brief (§5b), so dispatching without this answer would dispatch a direction
nobody agreed to. If the user asks for changes, rework the plan and ask again; do not start a
partial dispatch of the lanes they did not question.

---

## §4 One workspace per lane

**`herdr worktree create --cwd <repo>` creates TWO workspaces**, not one: the linked worktree *and* a
workspace for the base repo itself. So snapshot ids before and after and record every new one, or
cleanup will leak:

    herdr workspace list | jq -r '.result.workspaces[].workspace_id'   # before
    herdr worktree create --cwd <repo> --branch <branch> --base <base> --label <lane> --no-focus
    herdr workspace list | jq -r '.result.workspaces[].workspace_id'   # after — diff for new ids

`herdr workspace create --cwd <cwd>` is **not** an alternative here — it would point a lane at an
existing checkout, which §1 forbids. `worktree create` is the only way a lane is born. Nor is the
agent's *own* worktree feature (some of these CLIs have one): a lane must live in the worktree herdr
created and recorded, because that path is what §6e verifies and §6f pushes. The driver says so
explicitly where its agent offers one.

Read ids from the JSON, never guess — workspace ids are opaque handles like `w4B`, not `w1`:
`.result.workspace.workspace_id`, `.result.root_pane.pane_id`, `.result.worktree.path` (the lane's
checkout).

If a create fails because the name, label or worktree path is already taken — a §3 collision
surfacing at claim time, or a stale checkout directory neither list could show — rename per §3's
collision rule and retry once. On any other failure, record the lane as `failed` with the error,
continue with the rest, and report it at the end — one broken lane must not abort the dispatch.
Append each lane to the state file as it is created so a crash mid-fan-out is recoverable.

---

## §5b Write the brief into the lane's own checkout

(§5a pre-flights the pane and §5c launches the agent — both in the driver. Write the brief between
them: the file must exist before the agent is primed.)

`<checkout>/.dispatch/TASK.md`, kept out of git:

    mkdir -p <checkout>/.dispatch/help
    printf '.dispatch/\n' >> <checkout>/.git/info/exclude

Every lane has its own checkout, so `.dispatch/` never collides between lanes and needs no per-lane
subdirectory.

The brief contains, in English: **objective**; the **plan** — the §3 plan exactly as the user
confirmed it, with a standing instruction: execute it in order, and when reality contradicts a
step, record the deviation and its reason prominently in `.dispatch/progress.md` instead of
silently re-designing — and when the contradiction leaves no clear way forward, ask through the
help-request protocol below; the **checklist** as `- [ ]` items; the **acceptance criteria** — the §3
criteria exactly as confirmed, with the framing that they, not the checklist ticks, define done:
DONE is written only when every criterion verifiably holds; **boundaries** (work only in this
checkout, never `cd` to the main checkout, never touch another lane's files, never push, never
merge, never open a pull request — committing is where your job ends, and the orchestrator publishes
the branch once it has verified it); and the **commit policy** below, quoted into the brief in full.

> **Commit policy.** Commit continuously as you work, never as one lump at the end. Each commit is one
> coherent unit — a module, a file, a self-contained behaviour change — and each message follows
> Conventional Commits: `<type>(<scope>): <description>`.
>
> Pick the granularity by asking what a reviewer would want to read on its own and what a future
> `git revert` would need to take out in one piece. Concretely: a checklist item is at least one
> commit, and a large one is several; a refactor and the behaviour change that follows it are separate
> commits; a new module lands separately from the code that starts calling it; unrelated fixes you
> notice along the way never ride along inside another commit. Stage only the paths belonging to the
> unit you are committing — `git add <paths>`, never `git add -A`. Do not amend or rebase a commit you
> already made; correct it with a follow-up commit instead, because the orchestrator may already have
> read the history.
>
> **`.gitlock` protocol** (from `~/CLAUDE.md`), pinned to the **main checkout's absolute path**
> `<repo>` — each worktree has its own root, so a per-worktree lock would serialize nothing: before
> each commit, create `<repo>/.gitlock`; commit; delete it. If it already exists, another lane is
> committing — wait 30 s, re-check, and only commit once it is gone. Never delete a `.gitlock` you did
> not create.

Then the load-bearing part:

> **Progress protocol.** Keep `.dispatch/progress.md` current: after each checklist item, rewrite it
> with the checklist and its tick state, what you just did, what you are about to do, and any decision
> a fresh reader would need. Assume your context may be compacted at any moment and this file is all
> you keep. When every checklist item is done, every acceptance criterion holds, no help request is
> still open, and `git status` is clean, write `.dispatch/DONE` with a one-line summary.

`.dispatch/progress.md` carries more weight the less autonomous the agent is: for a lane driven by
re-nudges (§6c `idle_incomplete`) it is the only thing that tells the next nudge where to resume, so
the driver may strengthen this paragraph — never weaken it.

Then the notify-back, so a finished lane rings the orchestrator instead of sitting undiscovered
until the next §7 tick. Write it into the brief with `<orch-pane>` (the §2 `orchestrator_pane`),
`<run-id>`, `<lane>` and `<skill>` already substituted — the lane runs it verbatim:

> **Completion notify-back.** Immediately after writing `.dispatch/DONE`, run this exact command
> once:
>
>     herdr agent prompt <orch-pane> "[<skill> <run-id>] lane <lane> wrote DONE — run one <skill> --resume sweep now."
>
> If it errors or is rejected (e.g. `agent_blocked`), do not retry and do not investigate — the
> orchestrator also polls on a timer and will find `.dispatch/DONE` regardless. This command and
> the help ring below are the only herdr commands in your job: both target only the pane named
> here, and you never read, prompt, or send keys to any other pane or agent.

Then the help-request protocol, so a lane that is stuck asks the one participant who wrote its plan
instead of guessing, silently re-designing, or spinning until a sweep notices. Substitute the same
four values; the lane follows it verbatim:

> **Help-request protocol.** When you are stuck — a step of the plan contradicts the code, two
> readings of the task are both plausible, an acceptance criterion cannot be met as written, or the
> same failure survived two honest fix attempts — ask the orchestrator instead of guessing. Do not
> ask about what you can find out yourself by reading the repo, the brief, or the tool's own output.
>
> 1. Pick the next id: `H-<n>`, where `<n>` is one more than the highest number already in
>    `.dispatch/help/` (start at `H-1`). Write `.dispatch/help/H-<n>.request.md` with exactly these
>    sections:
>
>        # H-<n>
>        mode: continuing | waiting
>        checklist_item: <the item you were on, quoted>
>        ## Problem
>        <what is wrong: the exact error text, the file and line, the step of the plan involved>
>        ## Expected solution
>        <what you think the right fix or decision is, and why; say so when you have no candidate>
>        ## Tried
>        <each approach you tried and what happened, one bullet each, with the command you ran>
>        ## Meanwhile
>        <for `continuing`: which independent checklist items you will do while you wait;
>        for `waiting`: why nothing else can move until this is answered>
>
>    Write the file to `H-<n>.request.md.tmp` first, then rename it, so the orchestrator never reads
>    a half-written request.
> 2. Record the open request in `.dispatch/progress.md` (`waiting on H-<n>` or
>    `H-<n> open, continuing with …`), so it survives compaction.
> 3. Ring the orchestrator once, with the same no-retry rule as the completion notify-back:
>
>        herdr agent prompt <orch-pane> "[<skill> <run-id>] lane <lane> asked for help H-<n> — run one <skill> --resume sweep now."
>
> 4. **Prefer `continuing` — avoid blocking.** If any checklist item does not depend on the answer,
>    choose `continuing` and do those items now, committing as usual. Never do work that the answer
>    could invalidate, and never mark the blocked item done. Choose `waiting` only when every
>    remaining item depends on the answer: then end your turn and stop. The answer arrives as a
>    prompt from the orchestrator; do not poll, sleep, or loop on the file.
> 5. **Check for answers.** After each checklist item, and at the start of every turn, look for
>    `.dispatch/help/H-<n>.answer.md` for each request you still have open. When it exists, read it,
>    apply it, then append `applied H-<n>: <one line>` to `.dispatch/progress.md`. The answer is the
>    orchestrator's decision on the plan: follow it even where it changes a step, and record that
>    change in `progress.md` as a deviation authorised by H-<n>. If the answer says `escalated to
>    the user`, keep the same mode and wait for the final answer that follows.
> 6. At most one `waiting` request at a time. Do not re-ask the same question under a new id; if an
>    answer did not work, open a new request whose `Tried` section starts with that answer and what
>    it produced.

(The herdr skill warns against asking for file output in an initial prompt. That guidance is about
*retrieving long answers*; here the files are a state protocol that must survive compaction.)

---

Now launch the lanes: the driver's §5c. Start every lane before supervising any of them, then read
`references/supervise.md` and continue at §6.
