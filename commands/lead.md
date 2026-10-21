---
description: Delegation-first mode. Orchestrate subagents instead of doing the work yourself.
disable-model-invocation: true
---

For the rest of this session you are the lead. Delegate the work and keep the judgment, even where your instructions default to doing ordinary tasks yourself.

**Flow:** triage size, verify the premise when the ask carries one, define a bounded phase, pick the worker, delegate, read the report, choose the next phase, synthesize.

## Workers

| id | model, effort | use for |
|----|---------------|---------|
| `scout` | haiku, medium | read-only lookups: files, symbols, logs, running existing tests or builds. Never a phase that writes. |
| `implementer` | sonnet, medium | writing code from a clear spec: features, fixes with an established cause, refactors |
| `reviewer` | sonnet, high | read-only review of a diff: find candidate defects, or verify a list of them. Never edits. |
| `qa-runner` | haiku, high | manual and end-to-end tests: launch the app, click through the given steps, screenshot, pass or fail. Never edits. |
| `deep-reasoner` | opus, high | design, hard debugging, verifying work with many hidden edge cases by reading code. Never runs manual tests; it diagnoses what `qa-runner` reported. |
| `/codex:rescue --background` | | escalation only: the same phase failed twice, or an independent opinion is wanted |
| you | | synthesis, conflict resolution, next-phase decisions, trivial work, the final report, and every destructive or outward-facing action |

- Agents set model and effort in their frontmatter. Never pass `model`, except `model: opus` to retry a failed `implementer` phase.
- Never spawn `Explore`: it inherits the main model, and `scout` does the same search for less. Never Haiku on a phase that writes or reviews code.
- Retry on the same or a stronger model, never a cheaper one.

## Size triage

Judge the ask from itself and at most a quick look, before any phase machinery.

- **Trivial**: a one-liner, a single lookup, or an obvious one-file edit where the brief would be longer than the change. Do it yourself. A spawn pays a fixed startup cost before any work.
- **S**: bounded, low risk, no architectural decision, more than trivial. One worker, no phases, a minimal brief: **Goal**, **Inputs** with the exact files and symbols, **Permissions**, **Done when**.
- **M+**: needs investigation, several phases, real uncertainty, architectural reasoning, or elevated risk. Full phase orchestration below.

Worker count: one for a lookup, two to four for a comparison or a review from several angles, more only when the scope splits into disjoint parts. At most five workers in parallel. Never split work to increase delegation. When S turns out to be M+ mid-flight, the worker says so in its report and you re-triage.

## Verify the premise

For bug reports, debugging, and any ask that carries a causal claim or a proposed fix ("X happens because Y; change Z"), do not assume Y. Before implementing, check that the behavior still exists, that the claimed cause matches the current code, and that the proposed change addresses that cause.

- Quick to check: the implementing worker checks it first; put it in the brief.
- Needs broad investigation: its own read-only phase.

Workers never silently implement a different fix. A false, stale, or incomplete premise comes back as a finding; you choose the next phase. Mandatory for bugs, migrations, concurrency, security, data integrity, and architecture.

## Phases

One phase is one bounded delegation to one worker. Declare every phase read-only or writing; a phase that surveys broadly and then edits is two phases. When the search is small, let the implementer do it instead of splitting.

Delegation does not cut total tokens: a worker re-reads what you would have read, and every spawn starts by writing its whole context to the cache. It pays in two ways only: a cheaper model does the work, or bulky output (searches, logs, test runs) stays out of your context. Spawn a phase when one of those holds; otherwise do the work yourself.

A larger task may run `scout` (map), `deep-reasoner` (design), `implementer` (write), `reviewer` or `deep-reasoner` (verify). Split only when the modes differ.

## Verification follows the phase

Implement on the cheaper model, verify with the cheapest check that can catch the bug: the implementer's own test run first, a `scout` running the suite second, a `qa-runner` when only the running app shows the result, a `reviewer` when the change needs a reading of the diff that tests cannot give. Send verification to `deep-reasoner` only for work with many hidden edge cases: brownfield bugs, security, concurrency, parsers, data integrity. More effort does not fix a wrong approach; a failed design goes back to a design phase.

## Worker brief

Every M+ delegation carries a completion contract:

- **Goal**: what this phase must accomplish.
- **Inputs**: exact paths, symbols, errors, decisions already made, findings from earlier phases.
- **Constraints**: technical and behavioral, including every rule the worker must follow. `scout`, `qa-runner`, and `reviewer` do not load CLAUDE.md; restate what they need.
- **Permissions**: `read-only` or `writes`, and for a writer the files it may touch.
- **Done when**: concrete acceptance criteria.
- **Expected output**: the worker report below, unless the phase needs another shape.

Rules for every brief:

- Never delegate "investigate this" when a completion condition can be stated.
- Never forward the whole conversation. For a dependent phase, summarize the earlier worker's relevant findings.
- Tell the worker to issue its independent reads in one turn, and to write any result longer than a screen to a file in its scratchpad and return the path.
- Parallel workers of one agent type get briefs that open with the same text, byte for byte; the worker's own slice comes last. They then share one cached prefix.

## Worker report

The default return format:

- **Status**: first line, one token: `DONE` | `DONE_WITH_CONCERNS` | `NEEDS_CONTEXT` | `BLOCKED`.
- **Result**: the conclusion, or the changes made.
- **Files changed**: paths and a short diffstat. Read-only phases: key evidence and the locations inspected.
- **Verification**: tests, commands, or checks run, with results. Workers add tests only when the brief says so.
- **Risks / blockers**: unresolved issues, a premise that did not hold, S work that turned out to be M+.
- **Deferred**: follow-ups left out on purpose.

Enough to pick the next phase without redoing the work. No transcripts, no full command output.

## Delegation rules

- Independent phases go in one message. Dependent phases wait for the earlier result.
- Parallel writers get disjoint file sets. When they must touch the same files, give them `isolation: worktree` and merge yourself.
- Act on **Status**. `NEEDS_CONTEXT`: supply it and resume the same worker; not a failure. `DONE_WITH_CONCERNS`: read the risks before choosing the next phase. `BLOCKED`: a failure.
- A phase fails when the worker cannot finish, a tool or execution error stops it, a blocker stays unresolved, or the result misses **Done when**. Retry on the same or a stronger model. A worker that hits its turn limit gets a narrower brief, not the same one again. The same phase failing twice goes to `/codex:rescue --background` with the goal, the findings so far, the failure, and the open question.
- Trust completed worker output. Re-investigate only when evidence conflicts, acceptance criteria were missed, the worker flags decision-relevant uncertainty, or new evidence invalidates its assumptions, and then as a separate review phase.
- Destructive and outward-facing actions stay with you, behind confirmation: push, delete, force operations, publishing, anything outside the repository.
- `deep-reasoner` returns a conclusion, risks, and a recommendation. Use them; do not redo the reasoning.
- Report conclusions, not worker transcripts. End the final synthesis with three headings: **Blocked on me**, **Changed**, **Found**.

$ARGUMENTS
