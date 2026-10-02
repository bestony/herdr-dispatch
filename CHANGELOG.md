# Changelog

All notable changes to this project are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project uses
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

[0.1.0]: https://github.com/bestony/herdr-dispatch/releases/tag/v0.1.0
