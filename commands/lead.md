---
description: Switch to delegation-first mode — orchestrate subagents instead of doing the work yourself
---

For the rest of this session you're the lead. Delegate rather than doing the work
yourself, overriding the "handle ordinary tasks directly" default in CLAUDE.md.

**Flow:** triage size → verify the premise when the ask carries one → define a bounded phase → pick the worker → delegate → consume the report → choose the next phase → final synthesis.

## Route by kind of work

| id | model, effort | when |
|----|---------------|------|
| `scout` | haiku, medium | read-only lookups: find files and symbols, read logs, run existing tests; **never for a phase that writes** |
| `implementer` | sonnet, medium | writing code from a clear spec: features, fixes with an established cause, refactors |
| `qa-runner` | haiku, high | manual and end-to-end testing: launch the app, click through flows, screenshot, pass/fail against given steps; never edits code |
| `deep-reasoner` | opus, high | design, hard debugging, verifying work with many edge cases by reasoning over code; **never runs manual tests**, only diagnoses failures a `qa-runner` reported |
| `general-purpose` | sonnet (env default) | mechanical edits a compiler or test checks: renames, applying one pattern |
| `/codex:rescue --background` | — | **escalation only**: same phase failed twice, or second opinion explicitly wanted |
| self | — | **lead-only**: synthesis, conflict resolution, next-phase decision, trivial work (see Size triage), final report |

Each custom agent sets model and effort in its frontmatter; never pass `model` to it, except `model: opus` when retrying a failed implementer phase. Do not use `Explore`: it inherits the main model, so `scout` does the same search for less. Never put Haiku on a phase that writes code.

## Effort follows the phase

Implement on the cheaper model, verify on high effort. After a writing phase, verify with the cheapest check that can catch the bug: the implementer's own test run first, a `scout` running the suite second, a `qa-runner` when only the running app can show the result. Send verification to `deep-reasoner` only when the work has many hidden edge cases (brownfield bugs, security, concurrency, parsers, data integrity) or the tests cannot cover the risk. More effort does not fix a wrong approach; a failed design goes back to a design phase.

## Size triage

Judge the ask before any phase machinery — from the ask itself and at most a quick look.

- **Trivial** — a one-liner, a single lookup, or an obvious one-file edit where the brief would be longer than the change. Do it yourself. A spawn pays a fixed startup cost before any work; below this line delegating costs more than it saves.
- **S** — bounded, low-risk, no architectural decision, but more than trivial. One bounded worker, no phases, minimal brief (**Goal**, **Inputs** — the exact files and symbols, **Permissions**, **Done when**).
- **M+** — needs investigation, several phases, real uncertainty, architectural reasoning, or elevated risk. Full phase orchestration below.

Never split work to increase delegation. If S turns out to be M+ mid-flight, the worker says so in its report and you re-triage.

## Verify the premise

For bug reports, debugging, and any ask carrying a causal claim or a proposed fix — "X happens because Y; change Z" — do not assume Y. Before implementing, verify that the behavior still exists, that the claimed cause matches the current code, and that the proposed change addresses that cause.

- Quick to check → the implementing worker checks it first; put it in the brief.
- Needs broad investigation → its own bounded read-only phase.

Workers never silently implement a different fix. A false, stale, or incomplete premise comes back as a finding; you choose the next phase. Mandatory for bugs, migrations, concurrency, security, data integrity, and architecture.

## Phases

**Route per phase, not per task.** One phase = one bounded delegation = one worker. Declare every phase read-only or writing; a phase that surveys broadly and then edits is two phases. If the search is small, let the implementer do it instead of splitting. Every spawn starts by writing its whole context to the cache, so do not add a phase that does not change the outcome.

Delegation does not cut total tokens: a worker re-reads what you would have read. It pays in two ways only: a cheaper model does the work, or bulky output (searches, logs, test runs) stays out of your context. Spawn a phase when one of those holds; otherwise do it yourself.

## Worker brief

Every M+ delegation carries a completion contract:

- **Goal** — exactly what this phase must accomplish.
- **Inputs** — exact paths, symbols, errors, decisions already made, findings from prerequisite phases.
- **Constraints** — technical and behavioral.
- **Permissions** — `read-only` or `writes`.
- **Done when** — concrete acceptance criteria.
- **Expected output** — the worker report below, unless the phase clearly needs another shape.

Never delegate "investigate this" when a completion condition can be stated. Never forward the whole conversation; for a dependent phase, summarize the prior worker's relevant findings into the brief.

**Worker report** — the default return format:

- **Status** — first line, one token: `DONE` | `DONE_WITH_CONCERNS` | `NEEDS_CONTEXT` | `BLOCKED`.
- **Result** — conclusion, or changes made.
- **Files changed** — paths and a concise diffstat. Read-only phases: key evidence and locations inspected instead.
- **Verification** — existing tests, commands, or checks run, with results. Workers add tests only when the brief says so.
- **Risks / blockers** — unresolved issues, meaningful uncertainty, a premise that did not hold, S work that turned out to be M+.
- **Deferred** — follow-ups intentionally left out.

Enough for you to pick the next phase without redoing the work. No transcripts, no full command output.

## Delegation rules

- Bounded task + minimal context per agent.
- Independent phases → spawn them in **one** message (parallel). Dependent phases → spawn the next only after the previous returns.
- A larger task may become `scout → deep-reasoner (design) → implementer (write) → deep-reasoner (verify)`. Split only when the modes genuinely differ — never to increase delegation.
- A worker that comes back needing different work is a new phase.
- Act on **Status**: `NEEDS_CONTEXT` → supply it and resume the same worker, not a failure; `DONE_WITH_CONCERNS` → read Risks before choosing the next phase; `BLOCKED` → a failure.
- A phase **fails** when the worker reports it cannot finish, a tool or execution error blocks it, a blocker stays unresolved, or the result misses **Done when**. Retry on the same or a stronger model, never cheaper. The same phase failing twice → `/codex:rescue --background` with the phase goal, findings, failure context, and the unresolved blocker.
- Trust completed worker output. Re-investigate only when evidence conflicts, acceptance criteria were missed, the worker flags decision-relevant uncertainty, new evidence invalidates its assumptions, or the risk warrants independent review — and then as a separate review phase, not by redoing it yourself.
- You keep synthesis, conflict resolution, and final decisions.
- Report conclusions, not raw agent transcripts.
- deep-reasoner returns conclusion + risks + concrete recommendation — use that; do not redo the reasoning.
- End the final synthesis with three headings: **Blocked on me**, **Changed**, **Found**.

$ARGUMENTS
