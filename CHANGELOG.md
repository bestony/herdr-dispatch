# Changelog

All notable changes to this project are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project uses
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-10-02

### Added

- **`dispatch-antigravity`** — a fourth dispatcher, for the Antigravity CLI (`agy`, herdr kind
  `agy`). It uses the same shared plan and supervision halves and the same flags as the other
  dispatchers. Verified against agy 1.2.14 under herdr 0.9.3:
  - lanes run in agy goal mode (`/goal`). agy pursues the goal in one long run and writes a
    `<!-- GOAL_COMPLETE -->` marker when it judges the goal met. A run that stops without the
    marker, or stops on a quota error after its reset time, is re-engaged by the loop;
  - the probe reads `conversation_summaries.db` (run status) and the tail of each conversation's
    `transcript.jsonl` (compaction checkpoints, quota errors, the goal marker). Lane identity
    comes from herdr's `agent_session` value, which is the agy conversation id;
  - the driver accepts the workspace-trust dialog itself, because herdr reports the lane as
    `idle` and ready while the dialog is still on screen;
  - the brief and the goal both carry the checkout's absolute path. agy adds the active agy
    project's folders to the workspace list ahead of the pane's cwd, and in a test a lane ran
    commands in that folder;
  - `--no-yolo` drops `--dangerously-skip-permissions`. agy has no flag that turns prompting on,
    so the plan summary says when the user's agy config auto-approves anyway;
  - there is no `hot` row and no `--compact-at`: agy compacts by itself and has no `/compact`
    command. A lane that compacted 4 or more times and stops progressing is restarted with
    `/clear` and primed again from `.dispatch/`.
- The shared brief section now lets a driver add boundary lines for its agent.

## [0.1.0] - 2026-10-02

First tagged release.

### Added

- **Help requests from lanes to the orchestrator.** A stuck lane writes
  `.dispatch/help/H-<n>.request.md` with three required parts: the problem, the solution it
  expects, and what it already tried. Then it rings the orchestrator pane. The lane picks a mode:
  `continuing` (preferred) does the checklist items that do not depend on the answer, and
  `waiting` stops until the answer arrives. Lanes may not write `DONE` while a request is open.
- New shared supervision step §6j. Each sweep answers open requests before it classifies lanes.
  The orchestrator investigates read-only and writes `H-<n>.answer.md`. It escalates to the user
  any question that needs a change to the acceptance criteria, a wider scope, a secret, or a
  product decision. An answer that changes a plan step is reported as a deviation in the PR body
  and in the final report.
- A `help_pending` row in each driver's classification table. A lane that waits on a question is
  not nudged and is not flagged as stalled.
- Agent-specific answer delivery:
  - codex: put the answer file only while the goal is `active`; prompt and then `/goal resume`
    when the goal is `blocked`.
  - grok: check `/goal status` before delivery.
  - opencode: the answer prompt replaces the continuation prompt for that sweep.
- `CHANGELOG.md`.
- Earlier, untagged work that is part of this release:
  - the `dispatch-codex`, `dispatch-grok` and `dispatch-opencode` skills;
  - the repository as its own plugin marketplace;
  - installation through skills.sh;
  - the README and the MIT license.

### Fixed

- The brief setup wrote `.dispatch/` to `<checkout>/.git/info/exclude`. In a linked worktree,
  `.git` is a file, so this write always failed. The setup now uses
  `git rev-parse --git-path info/exclude` and does not add the entry twice.
- codex `--no-yolo`: if `~/.codex/config.toml` sets `approval_policy = "never"` or
  `danger-full-access`, the lane still ran in YOLO mode. The driver now passes the safe posture
  explicitly. It also documents that under this posture the sandbox blocks commits and the herdr
  rings, so the supervision timer is the only signal.
- Driver pre-flight used the removed `herdr pane run` command.
- PR provenance lines no longer name the model or the approval posture.

[0.2.0]: https://github.com/bestony/herdr-dispatch/releases/tag/v0.2.0
[0.1.0]: https://github.com/bestony/herdr-dispatch/releases/tag/v0.1.0
