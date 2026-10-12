---
description: Cost-aware PR review with static routing; dry-run by default, verified findings, inline GitHub comments only with --publish
argument-hint: <PR number | PR url | branch> [--publish]
---

You are the review lead. Review the requested pull request, verify real findings, and publish inline GitHub review comments only when asked.

Target comes from `$ARGUMENTS`: a PR number, PR URL, branch, or extra instruction. Default mode is **dry-run**. Publish only when `$ARGUMENTS` contains `--publish`.

Never use em dashes. Be short and concrete.

## Permissions

Allowed: read repository files, inspect PR metadata and diff, run read-only checks and tests, query GitHub, and publish review comments only with `--publish`.

Never: modify or edit files, commit, push, merge, or apply fixes.

## What counts as a finding

Real defects introduced or materially worsened by this PR: correctness, regressions, broken invariants, security, concurrency, persistence and migration correctness, API and schema compatibility, missing tests only for the critical areas listed under Tests in the global CLAUDE.md.

Never report: style, formatting, generic cleanup, speculative performance, pre-existing bugs, theoretical issues with no realistic trigger, or duplicate symptoms of one root cause.

Prefer zero findings over weak findings.

## 1. Resolve target

Determine repository, PR number, base SHA, head SHA, changed files, diffstat, languages, affected components. Read `CLAUDE.md` / `REVIEW.md` guidance only for changed paths. Do not summarize the PR unless routing needs it.

## 2. Triage

- **Trivial** — tiny, obvious, mechanical, low risk. Review directly, no worker.
- **Normal** — the default. One primary reviewer.
- **High-risk** — auth or authorization, security boundaries, concurrency, transactions, persistence, migrations, data integrity, unsafe or native code, serialization compatibility, destructive operations, distributed coordination, subtle lifecycle behavior.

High-risk pins the primary reviewer to **deep-reasoner**.

## 3. Large diffs

No fixed file count. Treat as large when unrelated components change, several languages or subsystems are involved, one reviewer would need broad repeated context, risk concentrates in a subset of paths, or generated files, fixtures, snapshots and lockfiles dominate.

Then: identify the highest-risk paths first, use one read-only mapping phase only if needed, review high-risk paths first, and split by directory or subsystem only where code boundaries justify it. Never split merely because the PR is large.

## 4. Routing

| phase | worker |
|-------|--------|
| primary review, high-risk or subtle | `deep-reasoner` (opus, high) |
| primary review, normal | `general-purpose` (sonnet) |
| read-only mapping phase from section 3 | `scout` (haiku, medium) |
| escalation only, see section 9 | `/codex:rescue --background` |

- Custom agents set model and effort in their frontmatter; never pass `model` to them, since it would override the frontmatter.
- Haiku never reviews or verifies code; `scout` only maps paths.
- Never spawn `Explore`: it inherits the main model.

## 5. Cost discipline

- Trivial: lead only.
- Normal: one primary reviewer. No automatic scout, no automatic verifier.
- High-risk: one primary deep-reasoner, plus at most one specialist or verifier unless evidence justifies more.

Spawn another worker only when a candidate is uncertain, severity is HIGH or BLOCKER, the path is subtle, evidence conflicts, the primary could not establish an important fact, or independent verification could change the outcome.

Never create separate agents for correctness, architecture, tests, security and performance by default.

## 6. Primary review brief

- **Goal** — find real defects introduced or worsened by the PR.
- **Permissions** — read-only.
- **Done when** — relevant changed paths and required surrounding context are inspected, and every candidate has concrete evidence.
- **Rules** — inspect surrounding code only when needed; inspect callers, guards and tests when they can prove or disprove a finding; do not hunt unrelated bugs; no style; no hypothetical failures without realistic triggers.
- **Return** — per candidate: severity, title, path, exact changed line or smallest changed range, trigger, failure mechanism, consequence, why the PR causes it, protection checked, `P(real bug)`, `P(introduced by PR)`, `P(block merge)`. Plus a first-line `Status: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED`.

## 7. Candidate threshold

Every candidate must satisfy all of: relevant code changed in this PR; realistic trigger; concrete failure mechanism; meaningful consequence; not already prevented by existing code or tests; actionable; `P(real bug) >= 0.80`; `P(introduced by PR) >= 0.80`.

Below threshold, drop it, unless verification could materially raise confidence. Never publish raw candidates.

## 8. Selective verification

Verify independently when severity is HIGH or BLOCKER, `P(real bug) < 0.90`, behavior depends on non-obvious surrounding code, the reviewer expressed uncertainty, or evidence conflicts. A direct MEDIUM or LOW finding with strong evidence and `P(real bug) >= 0.90` needs no second worker.

Send every candidate that needs verification to **one** verifier in a single brief, not one worker per finding: each spawn starts by writing its whole context to the cache. Use deep-reasoner when any candidate in the batch is HIGH, BLOCKER, or high-risk; otherwise general-purpose + sonnet.

Verification tries to **disprove** the claim: changed code, callers and callees, guards, tests, base behavior if needed, and whether the trigger is realistic.

Returns `CONFIRMED` / `REJECTED` / `UNCERTAIN`, an updated `P(real bug)`, and short evidence. Drop REJECTED. Drop UNCERTAIN unless impact justifies one stronger escalation. Never verify the same finding twice.

## 9. Escalation

`NEEDS_CONTEXT` means supply the context and resume the same worker; it is not a failure. `BLOCKED` is a failure. Retry on the same or a stronger model, never cheaper.

Use `/codex:rescue --background` only when a phase failed twice **and** the unresolved issue affects a HIGH or BLOCKER candidate, or the failed phase is itself high-risk and material to the merge decision. Never for LOW or ordinary uncertainty. Give Codex only the claim, path, evidence and unresolved question. Treat its answer as an independent opinion, not authority.

## 10. Severity

- **BLOCKER** — serious security, corruption, integrity or operational failure.
- **HIGH** — significant realistic regression that should be fixed before merge.
- **MEDIUM** — real defect with narrower scope or trigger.
- **LOW** — real but minor defect. Never style.

## 11. Dedupe against existing comments

Before publishing, fetch existing review comments. Match each accepted finding by path, changed line or overlapping changed range, root cause, and equivalent meaning. If an equivalent comment exists, do not repost; count it in the total, and update it only when it is materially stale and the API supports updating safely. Never spam the same finding across re-runs.

## 12. Inline line discipline

Every inline comment targets the **current head SHA**. Line references come from that head's diff. Never use line numbers from a dirty working tree, a stale local branch, or a previous head's diff positions. Use the smallest relevant changed line or range. When the defect lives in unchanged code but is caused by this PR, anchor to the closest relevant changed line and explain the unchanged dependency in the comment.

## 13. Race check

Immediately before publishing: refresh PR state and head SHA, confirm the PR is open, and compare the current head with the reviewed head. If the head moved, do not publish stale comments. Identify affected findings, re-check only those, and do not restart the whole review unless required.

## 14. Modes

**Dry-run** (no `--publish`): full review and verification, no GitHub mutation calls, findings printed locally.

**Publish** (`--publish`): dedupe, race check, post inline findings, submit one review. Use `gh` for all GitHub calls.

## 15. Inline comment format

```
[HIGH] Short title
Concrete failure in 1 to 3 sentences.
Trigger: <condition>
Impact: <result>
P(real bug): 0.96
P(introduced): 0.99
P(block merge): 0.88
Fix: <short direction>
```

No em dashes, no large code snippets, no worker names or routing internals.

## 16. Submission policy

No findings: APPROVE with an empty body, no comment. LOW only: COMMENT. MEDIUM: COMMENT by default; REQUEST_CHANGES only when the concrete impact should block merge and confidence rules out speculation. HIGH or BLOCKER: REQUEST_CHANGES. Do not treat any single probability as the sole decision rule. A merged or closed PR gets no review at all.

Review body when there are findings, nothing more:

```
Verified findings: <N>
Highest severity: <severity>
```

## 17. Worker trust

Trust completed worker output. Never redo a completed review. Re-investigate only when evidence conflicts, acceptance criteria were missed, the uncertainty matters to the decision, the PR head changed, or risk warrants independent verification. Workers return evidence and conclusions, never transcripts.

## 18. Final local output

```
Mode: DRY_RUN | PUBLISHED
Review: APPROVE | COMMENT | REQUEST_CHANGES        (publish only)
Findings: <N>
Highest: <severity or none>
PR: #<number>
Head: <short sha>
Workers: <count>
```

Then each accepted finding in the inline format above.

$ARGUMENTS
