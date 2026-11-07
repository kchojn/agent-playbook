# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.5.0] - 2026-11-07

### Added

- Plugin manifests, so the repo installs in one command and updates with `/plugin update`.
- An MIT license, a contributing guide, this changelog, and a `.gitignore`.
- A six-part brief and a return format for the `/tradecraft:pr-review` verifier.
- README sections for install, prerequisites, state locations, and headless use.
- A social preview card for the GitHub repository page, in the same palette as the flowcharts.

### Changed

- The repository is renamed from agent-playbook to claude-code-tradecraft.
- The repo ships as the `tradecraft` plugin. Commands are now `/tradecraft:lead`, `/tradecraft:pr-review`, and `/tradecraft:pr-review-watch`, and agents are `tradecraft:<name>`. This is a breaking change: the bare names no longer exist.
- Slash commands moved from `commands/<name>.md` to `skills/<name>/SKILL.md`.
- `deep-reasoner` can no longer edit files.
- `reviewer` may copy tests into a throwaway worktree that the brief names.
- The `oracle` description now opens with "Use when".
- The retry rule names the stronger worker.
- The review event for a MEDIUM finding follows the submission policy, so a MEDIUM whose impact blocks merge can request changes.
- From the third review round, a round whose only confirmed findings are LOW posts a comment review with their count instead of approving.
- The review event is defined once, in the submission policy section.
- The finder brief uses the shared brief vocabulary and the bare status token.
- The research doc marks superseded gates and records limits that were set without a measurement.
- The flowcharts are redrawn in a new palette, saved as the `tradecraft` diagram-design profile, with the accent on one step each and the full command names in the watcher flow.
- Flowcharts and docs are ASCII only, and the review flow label shows both PLAUSIBLE thresholds.

### Fixed

- The watcher lock failed on a fresh machine because its parent directory did not exist.
- Two PRs with the same number in different repositories shared one worktree path.
- A dry-run exception in the eligibility check could never apply and is removed.
- Three cross-references in the review skill pointed at the wrong section.
- The trivial review tier had nobody to issue a verdict.
- The research doc's line limits were off by one against the skill.
- Two non-ASCII multiplication signs are replaced.

## [0.4.0] - 2026-11-05

### Added

- `/pr-review-watch`: one pass finds open PRs that requested your review and runs `/pr-review --publish` on each in its own headless `claude -p` session, three at a time.
- Clone roots come from the arguments or from `~/.config/pr-review-watch/roots`.
- Flowcharts for `/pr-review` and `/pr-review-watch`, and a `CLAUDE.md` for contributors.
- `/pr-review` posts comments in a fixed shape (Trigger, Effect, Checked), reads from a detached worktree at the head commit, and fetches `pull/<n>/head` so fork PRs work.

### Changed

- The watcher reviews only requests made in the last 24 hours, checks live PR state because the search index lags on merges, and makes one review attempt per PR head.
- `/pr-review` derives the review event from the findings at publish time.
- A re-review covers only the commits the author added since the last reviewed head. A rebase or a merge of the base no longer reopens reviewed code. From the third review only MEDIUM and above post.
- `/lead` allows ten workers in parallel.
- On a disputed verdict, `/lead` and `/pr-review` ask Codex through the plugin's companion script instead of `/codex:rescue`.
- A run with an unfinished phase publishes nothing, and one request creates the review and its comments together.

### Fixed

- A bare watcher run skipped every PR as "no local clone" and then reported that no requests were found.
- A watcher pass no longer picks up an old review request on a PR that got a comment today.
- `/pr-review` retries a fetch once when another review of the same clone holds a ref lock.

## [0.3.0] - 2026-10-26

### Added

- `/pr-review` reads `REVIEW.md` for ignored paths, priorities, and nitpick limits.
- `/pr-review` narrows later review rounds to CONFIRMED findings, asks the author to re-evaluate the approach, and treats threads the author resolved as final. With `--publish` it resolves its own threads.
- Triage of CI workflows, release, deploy, and auto-update code as high-risk.

### Fixed

- Posted comments rendered as plain text. They now keep the blank lines GitHub needs around `<details>`, bold the severity title, and quote evidence in a language-tagged fence.

## [0.2.0] - 2026-10-21

### Added

- The `oracle` agent: a one-shot, read-only escalation on Fable at high effort.
- The `reviewer` agent: read-only, for the find and verify phases.
- The `/lead` flow diagram in the README.
- Token-test findings on delegation, briefs, and clearing history in the docs.
- `/pr-review` checks fix PRs at the base commit and reports sibling defects.
- Incremental re-reviews and a cached brief prefix shared by parallel agents.

### Changed

- `/lead`, the agents, and `/pr-review` are rewritten and tightened against the docs.
- `/pr-review` drops private rule files when it reviews someone else's PR.
- `/pr-review` posts PLAUSIBLE items inline as `[Optional]` and uses an empty body for an approval.

### Removed

- An environment-specific `gh` note.

## [0.1.0] - 2026-10-15

### Added

- `/lead`, a delegation-first command, with the agent roster and the orchestration best practices doc.
- `/pr-review`, a review that splits recall-biased finders from a precision-biased verifier and widens the scope of what it examines.

### Changed

- `/pr-review` scores confidence on whether the defect exists, and posts CONFIRMED findings at every severity.
- High-confidence PLAUSIBLE items appear in the review body, and a gap between what is shown and what is executed rates MEDIUM.

[Unreleased]: https://github.com/kchojn/claude-code-tradecraft/compare/v0.5.0...HEAD
[0.5.0]: https://github.com/kchojn/claude-code-tradecraft/compare/v0.4.0...v0.5.0
[0.4.0]: https://github.com/kchojn/claude-code-tradecraft/compare/v0.3.0...v0.4.0
[0.3.0]: https://github.com/kchojn/claude-code-tradecraft/compare/v0.2.0...v0.3.0
[0.2.0]: https://github.com/kchojn/claude-code-tradecraft/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/kchojn/claude-code-tradecraft/releases/tag/v0.1.0
