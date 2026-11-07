# PR review research

What the strongest published review systems do to catch real bugs without noise, and which of it `/tradecraft:pr-review` adopts. Each rule cites its source. Vendor numbers are self-reported unless marked otherwise.

## 1. Split finding from judging

- Claude Code's built-in `/code-review` runs independent finder "angles", each returning candidates with a file, line, summary, and concrete failure scenario, then one verifier per candidate. Its prompt states the reason: "finders that silently drop half-believed candidates bypass the verify step and are the dominant cause of misses." [Claude Code binary, code-review recipe]
- Cursor's first Bugbot ran eight passes over the diff, bucketed similar bugs, took a majority vote, then ran "a validator model to catch false positives". Later, a cautious agent "was too cautious", so they moved to aggressive prompts behind the validator. Resolution rate went from 52% to over 70%. [Building Bugbot]
- Anthropic's plugin runs five Sonnet reviewers in parallel, then one scorer per issue, and posts only issues scoring at least 80 out of 100. [claude-plugins-official code-review]
- Uber's uReview stages are generate, filter, validate, dedupe. A second prompt scores confidence with thresholds per category. 75% of comments were rated useful. [uReview]
- Ellipsis attaches evidence links to each candidate and runs a hallucination check against that evidence. [How we built Ellipsis]

Adopted: section 5 (recall-biased finder, every candidate with a failure scenario passes through) and section 6 (one precision-biased verifier that must cite `file:line` evidence). High-risk PRs get a second finder that reads the files in reverse order, after Bugbot's "different ordering of the diff" trick.

## 2. Precision over recall for anything published

- OpenAI: "Precision is more important for usability than recall." A finding is worth surfacing when P(correct) x cost saved exceeds the cost of human verification plus P(incorrect) x cost of a false alarm. Stylistic comments "may even incur negative utility." [Verifying code at scale]
- Greptile audited its comments: 19% good, 2% wrong, 79% nits. Prompting against nits also cut critical comments, and an LLM judge scoring 1 to 10 was "nearly random". What worked was blocking comments similar to three or more previously downvoted ones; address rate went from 19% to over 55%. [Make LLMs shut up]
- Qodo: exclusion lists failed; a single positive instruction, "only give suggestions that address major problems and bugs", with an empty list allowed, raised acceptance 50%. [Less is more]
- Google AutoCommenter: 80% of candidate comments landed on unmodified lines; restricting to changed lines cut the share of files commented from 6% to 1.3%. Suppressing 17 non-actionable categories raised the useful ratio from 54% to 66%. [AutoCommenter, arXiv 2405.13565]
- Anthropic cloud review reports under 1% of findings marked incorrect, and tunes nits with a cap of five per review. [Claude Code review docs]

Adopted: section 1 opens with the positive bar ("problems a maintainer would fix") and allows an empty list; the never-report list follows. Section 7 caps LOW findings at five. Section 6 gates publication at CONFIRMED with confidence 80, or PLAUSIBLE at HIGH or BLOCKER with confidence 70. Section 9 dedupes against earlier comments; angle H skips complaints the author already answered as intentional.

Superseded: the gate in `skills/pr-review/SKILL.md` section 6 posts CONFIRMED at any severity, and PLAUSIBLE as `[Optional]` inline when confidence is at least 70, or at least 60 when severity is MEDIUM or higher.

## 3. The bar for a finding

The Codex review rubric lists eight tests; a finding must meaningfully affect accuracy, performance, security, or maintainability; be discrete and actionable; not demand more rigor than the codebase shows; be introduced in the commit; be something the author would fix; rest on no unstated assumption; name provably affected code rather than speculate; and not be an intentional change. "If there is no finding that a person would definitely love to see and fix, prefer outputting no findings. Do not stop at the first qualifying finding." [Codex rubric.md]

Anthropic's plugin lists false positives to skip: pre-existing issues, anything a linter, type checker, or compiler catches, pedantic nitpicks, general quality concerns unless CLAUDE.md requires them, issues silenced by an ignore comment, intentional changes, and real issues on untouched lines. [claude-plugins-official code-review]

Anthropic's security review: "Only flag issues where you're >80% confident of actual exploitability." It excludes denial of service, rate limiting, resource exhaustion, and missing validation on non-security fields unless proven. Each finding carries an exploit scenario. [claude-code-security-review]

Adopted: section 1, including the security exclusions and the pre-existing BLOCKER exception.

## 4. What to look at

- Claude Code's angles: line-by-line with the enclosing function in scope; removed-behavior auditor ("name the invariant it enforced, then search the new code for where that invariant is re-established"); cross-file tracer over callers and callees; language pitfalls; wrapper and proxy correctness; reuse, simplification, efficiency, altitude (root cause versus symptom, "special cases layered on shared infrastructure are a sign the fix isn't deep enough"), and conventions ("only flag a violation when you can quote the exact rule and the exact line"). Higher effort adds a gap sweep by "a fresh reviewer who has the verified list" focused on moved code that dropped a guard, flipped defaults, lock-scope changes, and setup/teardown asymmetry. [Claude Code binary, code-review recipe]
- Anthropic's plugin adds git blame and history, prior PR comments on the same files, and guidance in code comments as separate reviewers. [claude-plugins-official code-review]
- OpenAI: diff-only review "often misses important context about the whole codebase and the interaction of the change with its dependencies"; repo access and execution catch more and raise fewer false alarms. Extra reasoning budget "mostly improves calibration and reduces false alarms." [Verifying code at scale]
- Greptile v3 recursively follows nested calls; v2's fixed flowchart missed third-file callees. [Greptile v3]
- CodeRabbit generates grep and ast-grep checks to confirm an assumption before commenting. [CodeRabbit]
- Google's reviewer guide: design first, then "does this CL do what the developer intended", complexity, "will the tests actually fail when the code is broken", naming, comments, consistency. [Google eng-practices]
- SWE-PRBench: models "detect only 15-31% of human-flagged issues on the diff-only configuration", yet every model degraded as context grew; a 2,000-token structured diff with a summary beat a 2,500-token full-context prompt. SWR-Bench attributes 48% of false positives to missing context. The fix is targeted context, not more of it. [SWE-PRBench, arXiv 2603.26130; SWR-Bench, arXiv 2509.01494]
- Matt Pocock's skill reviews on two axes, standards and spec, so one cannot mask the other. [mattpocock-skills code-review]

Adopted: angles A to L and the gap sweep in section 5; the verifier's Grep or single test in section 6; the verifier on `deep-reasoner` (Opus, high) whenever a HIGH or BLOCKER candidate exists.

## 5. Severity, confidence, and the comment

- Codex priorities: P0 "only for universal issues that do not depend on any assumptions about the inputs", then P1 urgent, P2 normal, P3 nice to have. Comments are one paragraph, code under four lines, range of at most 10 lines, and must "clearly and explicitly communicate the scenarios, environments, or inputs that are necessary for the bug to arise". No praise. An overall verdict, "patch is correct" or "patch is incorrect", ignores nits. GitHub shows only P0 and P1. [Codex rubric.md; Codex cloud docs]
- Anthropic's plugin scoring anchors: 0 false positive or pre-existing; 25 possibly real but unverified; 50 verified but minor or rare; 75 verified and likely hit in practice; 100 certain and frequent with direct evidence. Links use the full SHA with context lines. [claude-plugins-official code-review]
- Claude Code's verifier verdicts: CONFIRMED (name inputs and wrong output, quote the line), PLAUSIBLE (mechanism real, trigger uncertain), REFUTED only when "constructible from the code". "PLAUSIBLE by default" for races, rare-path nils, boundary off-by-ones, partial failures. [Claude Code binary, code-review recipe]
- Anthropic cloud review posts a collapsed "Why this was flagged" section with the evidence. [Claude Code review docs]
- Codex prompting guide: findings first, ordered by severity with file and line; with none, "state that explicitly and mention any residual risks or testing gaps." [Codex prompting guide]

Adopted: sections 6, 7, 13, 14, and the `Residual risk` line in section 16.

## 6. Measured limits to keep in mind

- Third-party benchmarks disagree with vendor claims. PatchGuru measured Codex at 2 true positives in 23 warnings (precision 0.09); OpenCodeReview measured recall 5% and precision 28%. [arXiv 2602.05270; arXiv 2608.09290]
- Cursor ran 40 major experiments and "many changes, surprisingly, regressed our metrics"; they rely on an offline benchmark of real diffs with annotated bugs. [Building Bugbot]
- Greptile: "Both models find more bugs in code written by the other model than in code they wrote themselves." Opus volunteers 7 to 8 comments where Codex gives 1 to 2. [Greptile model inversion]

- A validator pass is not free. In a C++ defect-review study the validator cut the false-alarm rate from 94% to 75% but also cut key-bug inclusion from 37% to 20%. In security discovery, adversarial refuters killed 79% of 171 candidates, including an OpenSSL bug ten reviewers had unanimously endorsed; one empirical test settled it. [arXiv 2505.17928; Refute-or-Promote, arXiv 2604.19049]

Implication: the verifier keeps PLAUSIBLE findings at HIGH and BLOCKER rather than demanding proof, and may run one test to settle them. The dry-run `Dropped:` list in section 16 exists so the gate can be tuned on real PRs rather than trusted.

Superseded: the verifier keeps PLAUSIBLE findings by the confidence floors in `skills/pr-review/SKILL.md` section 6 (70, or 60 at MEDIUM or higher), whatever the severity.

## 7. Confidence is not severity

Lesson from the first run on a real PR: the verifier CONFIRMED a write-back bug that broke two promises in the PR's own comments, quoted the lines, then scored it 65 because the new store had no callers yet. The gate read 65 as "unsure it exists" and the run approved the PR. The cause was the rubric copied from Anthropic's plugin, whose 50 and 75 anchors mix "rare or minor" with "verified".

What the sources do:

- Codex keeps `confidence_score` (0.0 to 1.0) and `priority` (P0 to P3) as separate fields. [Codex rubric.md]
- Anthropic's security review keeps `severity` and `confidence` as distinct fields and gates only on confidence of exploitability. [claude-code-security-review]
- Anthropic's plugin folds both into one 0 to 100 score. That is the design the first version copied, and the one that failed. [claude-plugins-official code-review]
- Anthropic's cloud review posts low-severity findings inline as "Nit: a minor issue, worth fixing but not blocking", caps them at five in its example config, and "findings are tagged by severity and don't approve or block your PR." [Claude Code review docs]
- Google: "Consider labeling the severity of your comments, differentiating required changes from guidelines or suggestions", with `Nit:`, `Optional:`, and `FYI:` prefixes, and reviewers "should favor approving a CL once it is in a state where it definitely improves the overall code health." [Google eng-practices, comments and standard]
- Counterpoint: Codex on GitHub "flags only P0 and P1 issues so review comments stay focused on high-priority risks." Where P2 and P3 go is not documented. [Codex cloud docs]
- No source covers new code with no callers. The rule that unused code gets no discount is this repo's own, from the run above: the author would still fix it, which is the Codex bar.

Adopted: section 6 scores confidence on existence only; every CONFIRMED finding is posted, with severity deciding blocking versus non-blocking; PLAUSIBLE MEDIUM items go in the review body, not inline; a confirmed violation of a contract the PR itself states is at least HIGH; a run that drops a CONFIRMED finding and approves is a failed run.

Superseded: PLAUSIBLE items no longer go in the review body. Section 10 of this document records the move to inline `[Optional]` comments, and `skills/pr-review/SKILL.md` section 6 holds the current gate.

## 8. Plausible findings and the review body

Second run on a real PR: the verifier marked a key mismatch between a draft check and the act that executes it as PLAUSIBLE, LOW, confidence 70, and the gate dropped it because PLAUSIBLE LOW had no path out. The mechanism was real: the preview shows the attached file, the executed call creates a document without it. Two fixes follow.

- Any PLAUSIBLE item with confidence 70 or more now reaches the review body, capped at five lines, never inline. The body list costs the author one glance; inline plausible nits are where Greptile measured 79% of comments as unwanted. [Make LLMs shut up]
  Superseded: plausible items post inline as `[Optional]`, as section 10 of this document records; `skills/pr-review/SKILL.md` section 6 holds the current gate.
- A silent divergence between what the user is shown and what is executed is at least MEDIUM. The Codex rubric's bar, "the author of the original PR would likely fix the issue", applies regardless of how narrow the triggering input is. [Codex rubric.md]

## 9. Bug-fix PRs and private rules

Third run, on a race fix: the verifier on its own ran the PR's new test against the old code and showed it failed there, then noted two sibling sinks with the same missing check outside the PR's scope. Both checks are now required rather than left to initiative. Separately, for the second run in a row the finder built candidates on the reviewer's private `CLAUDE.local.md` and dropped them later as not binding on a teammate; private files now enter the context pack only for the reviewer's own PRs.

- "Will the tests actually fail when the code is broken?" is Google's test question; for a fix PR the cheapest answer is to run the new test at the base SHA. [Google eng-practices]
- Google's reviewer guide expects the reviewer to look at the change's effect on the rest of the system, which for a fix means the other places the same defect lives. The Codex bar, "the author of the original PR would likely fix the issue", holds for a sibling occurrence even though the rubric excludes pre-existing bugs in general; it goes in the review body, not inline. [Google eng-practices; Codex rubric.md]
- Both Codex and Claude Code read rule files from the repository tree, nearest file first; neither reads a reviewer's private files. [Codex rubric.md; Claude Code code-review recipe]

## 10. Where optional notes go

Fourth run: an approval with no confirmed findings carried a body of status lines plus two plausible notes, and it read as machine output on the PR. Plausible items that pass the gate now post inline, tagged `[Optional]`, capped at five, and never change the review event; an approval without confirmed findings has an empty body. This matches Google's advice to label minor comments at the comment itself (`Nit:`, `Optional:`) and Anthropic's cloud review, which posts Nits inline and blocks nothing. [Google eng-practices, comments; Claude Code review docs]

## 11. Cost: cache, tools, and re-reviews

Cloudflare's write-up of tuning its review agents gives measured cost changes that carry over. [Obsessive inference optimization, Cloudflare]

- Workers on the same model share a prompt cache only when their requests open with the same text. Cloudflare moved repo-specific text after a shared prefix and saw first-request cache reads rise from 14-26% to 91-99.5%. Briefs now open with the shared rules and context pack, and the worker's own part comes last.
- Tools a worker cannot use still cost tokens and calls: sub-reviewers that could see the comment-posting tool tried it in 57-76% of sessions. Finders and verifiers now run on the `reviewer` agent, or on `deep-reasoner` for the high-risk tier, instead of `general-purpose`. Both agents are read-only.
- In 64% of turns the model made a single tool call. The finder now issues its independent reads together in the first turn.
- About two-thirds of jobs were re-reviews. A re-review now covers only the commits since the reviewed head.
- On Sonnet 5.5, high effort cost 61% more per run than medium and posted 58% more critical and warning findings, so `reviewer` runs at high effort. [unverified: no run or commit records the 61% and 58% figures]
- Cloudflare strips user-controlled boundary tags from the text it reviews so a PR body cannot steer the reviewer. The context pack now marks PR text, issue text, and comments as data, and every brief that carries them says so. [AI code review at Cloudflare]

## 12. Owner guidance, convergence, and thread state

Addy Osmani's write-up of how the Claude Code team reviews its own PRs adds four rules. [How the Claude Code team does code review]

- Code owners keep a `REVIEW.md` that says what matters in their code, limits nitpicking, and lists ignored paths. The context pack now reads it next to the other rule files.
- "The review process must converge." One PR cycled more than fifty times overnight between an agent author and an agent reviewer, and about 59% of what got flagged was code present since the first pass. Re-reviews now exclude lines unchanged since the reviewed head, and from the fourth review the body asks the author to re-evaluate the approach.
- "All findings must be addressed -- fixed or the thread is resolved with a reason", and "a finding will be auto-closed if a subsequent commit addresses the problem." A thread resolved with a reason is now final, and `--publish` resolves this account's threads the verifier marks fixed.
- "Some paths always require another approval -- anything marked as sensitive by owners, and any CI workflow file." CI workflows, release, deploy, and auto-update code now triage as high-risk.

Not adopted: a vote among several agents to confirm findings (section 6's single precision-biased verifier already fills that role), and "agent approvals don't count as human approvals", which would stop this command from approving.

## 13. A comment a maintainer can scan

What the published tools put in an inline comment, and what readers complain about.

- Codex: a `[P1]`-style tag opens an imperative title of at most 80 characters; the body is one paragraph that states the scenario the bug needs; code chunks at most 3 lines; no praise; severity never overstated. A `suggestion` block only for concrete replacement code with the exact leading whitespace and no commentary inside. [Codex rubric.md; Codex review prompt]
- Anthropic's plugin: one comment per issue, a link with the full commit SHA and one line of context on each side, a committable suggestion only for small self-contained fixes and never for 6 or more lines or multi-location changes. [claude-plugins-official code-review]
- Anthropic's cloud review folds the evidence into a collapsed "Why this was flagged" section. [Claude Code review docs]
- CodeRabbit and Greptile fold their long content (committable suggestion, "prompt for AI agents"); PostHog turned the agent-prompt block off because it restated the finding. Reviews of both tools name verbosity as the main complaint. [CodeRabbit config; PostHog #105516; cubic, "How AI code review works"]
- GitHub: markdown inside `<details>` needs a blank line after `<summary>` and before `</details>`; a suggestion applies only to lines in the diff and cannot be applied on a deleted line; alerts (`> [!WARNING]`) are meant for one or two per page and their rendering in review comments is undocumented. [GitHub docs, collapsed sections; GitHub docs, commenting on a PR]
- Conventional Comments: one label, at most one decoration, never stacked. [conventionalcomments.org]

Adopted in section 13: the paragraph and fix line must stand alone; the fold is a fixed three-line list, `Trigger`, `Effect`, `Checked`, then the permalink and quoted lines, so the proof reads the same on every comment; optional `Also affected`, `Rule`, and `Spec` lines; a suggestion block of at most 5 lines and on diff lines only; no alerts, no footer.

## 14. Unattended runs

`/tradecraft:pr-review-watch` runs each review as its own `claude -p` process, from the Claude Code docs on headless mode and scheduled tasks.

- A prompt that starts with `/skill-name` expands the skill in `-p`; `--bare` would skip custom commands. [Claude Code docs, headless]
- `--permission-prompts none` denies anything that would prompt instead of hanging; `--disallowedTools` is the hard form of a "never edit" rule; `--strict-mcp-config` skips every configured MCP server; `--max-budget-usd` stops a run at an estimated spend; `--no-session-persistence` keeps the run out of `/resume`. [Claude Code docs, CLI reference and permission modes]
- Tools a worker cannot use still cost tokens and calls, which is the reason for `--strict-mcp-config` and `--disallowedTools` rather than prose. [Obsessive inference optimization, Cloudflare]
- `/loop` passes a skill as its prompt only when the skill is model-invocable; `disable-model-invocation: true` turns the fire into plain text. A loop fires only between turns and expires after seven days. [Claude Code docs, scheduled tasks]
- A subagent cannot start subagents, so a `context: fork` skill cannot run the finder and verifier; a separate process can. [Claude Code docs, subagents]
- The lock directory exists before the batch writes `lock/pid`. A second pass in that gap finds no pid, and reading that as a dead batch would remove a live lock, so a pid-less lock younger than ten minutes counts as live. [opinion: derived from steps 1 and 5 of `skills/pr-review-watch/SKILL.md`; the ten-minute bound is unmeasured]
- The Codex step for a disputed verdict follows `docs/orchestration-best-practices.md` section 7, so this document and that one together cover every rule in the command.

## 15. What a second reader found in the prompt

A Codex pass over the two command files, before any watcher run, found gaps a run would have hit. Adopted:

- Workers read the clone's working tree, not the PR head: `Read` and `Grep` on a clone checked out elsewhere review the wrong code. The lead now adds a detached worktree at the head SHA and briefs every worker with that path.
- An empty finding list from a finder that returned `BLOCKED` looked like a clean PR and could approve. A run with an unfinished phase now publishes nothing.
- A re-review counted only the findings of the new range, so a push that ignored an earlier HIGH earned an APPROVE. Earlier findings now stay CONFIRMED until the verifier marks them `FIXED`.
- Posting comments one by one could leave a half-published review that the next run treats as done. The review and its comments now go in one `POST .../pulls/{number}/reviews` call.
- Two PLAUSIBLE gates overlapped at HIGH; merged into one.
- The watcher's lock was check-then-write; `mkdir` is atomic. A review request can be withdrawn while the search index still lists it; the watcher now checks live `reviewRequests`. A `PENDING` review at the head counted as done; it no longer does. The daily window now starts yesterday UTC so a request just before midnight is not lost. One stuck review held the lock forever; `timeout` now bounds it.

Not adopted: tracking re-requests without new commits as a new review (the review at that head already exists); scaling severity down for rare triggers (section 7 argues the opposite); AGENTS.override.md discovery (Codex-specific).

## 16. Convergence on a real PR

Fifth lesson from a run: a 53-file PR got four reviews from this command in 21 hours, and the author asked for the ping-pong to stop. Matching each finding's lines to the commit that wrote them: rounds 1 to 3 flagged code present since the first commit, and only round 4 flagged the previous fix. The rule that unchanged lines are out of scope was in the command the whole time. Two clauses got around it. Round 2 ran after a force-push and round 3 after a merge of the base branch, and the text made both a full review. The finder brief also admitted "bugs in unchanged lines of a touched function", so a fix that touched a function reopened the whole function.

What the sources do:

- Anthropic's internal reviewer had the same failure: of what later rounds flagged, "about 59% of it was code that had been there since the reviewer's first pass." Fixes: "A round cap. After three rounds, the reviewer asks the author to step back"; "Diff-only re-review after the first few passes"; "No repeats of findings already posted"; a finding "clears on its own" when a later commit fixes it. PRs with five or more rounds fell from 8.9% to 1.5%, and mean reviews to pass from 2.08 to 1.32. [Solving code review at Anthropic]
- Claude Code's cloud review: a review after a push "focuses on what changed since the last review and posts fewer nits and pre-existing findings"; the body names what is "outside the scope of this re-review and not posted"; the next review "resolves its earlier thread when it judges the issue fixed"; a push with no new changes repeats nothing. [Claude Code docs, reviewing pull requests]
- Cursor Bugbot: "By default, Bugbot reviews only the changes since the previous Bugbot review" (`incremental: true`), and its pipeline ends with "Dedupe against bugs posted from previous runs." [Bugbot docs; Building Bugbot]
- CodeRabbit re-runs on each push and pauses automatic reviews after five reviewed commits. Greptile marks a comment addressed when a push touches its file and keeps unresolved findings in the summary. Copilot re-reviews only when configured to and "may repeat the same comments again". Codex posts only P0 and P1. None of them says whether it comments on unchanged lines. [CodeRabbit configuration; Greptile changelog; Copilot code review docs; Codex on GitHub]
- Google: approve "once it is in a state where it definitely improves" code health, "even if the CL isn't perfect". [Google eng-practices, standard]
- Two practitioner loops: "the reviewer reads only the commits since its last review, plus the code they touch", with a ledger of posted findings and exits on a round cap or no new blocking finding; and "the loop needs a cap, because a reviewer that is asked often enough will always find something." [How to make an agentic loop converge; Agentic code review loop]

Adopted in section 2, each step tested on the PR above:

- The new commits are `git rev-list --no-merges --right-only --cherry-pick <reviewed>...<head> ^<remote>/<base>`. `--cherry-pick` drops a commit the author only rebased; `^<base>` drops what a base merge brought in. Across the force-push it listed the 7 fix commits and not the rebased first commit; across the base merge it listed 9 author commits where the two-dot range held 55. A rebase or a base merge is no longer a full review, and an empty list ends the run.
- The finder gets the patches of those commits, not the PR diff, and reads enclosing functions to judge the new lines. Before verification, `git blame -s -L` at the head decides whether a candidate's line is new; on the PR above it put round 4's anchor on the fix commit and rounds 2 and 3's targets on original commits.
- An out-of-scope candidate is dropped unless the finder quotes the new line that changes what the old line does. A CONFIRMED BLOCKER gets one `Pre-existing` line.
- From the third review only CONFIRMED MEDIUM and above post; the fourth asks the author to step back, as Anthropic's cap does.

Not adopted: naming out-of-scope findings in the body as the cloud review does, since a count with no detail starts a thread without giving the author anything to act on; Greptile's rule that a push touching a file resolves its comment, which closes threads the fix did not address.

## 17. Limits set without a measurement

These values in the skills have no cited source and no recorded run. Dates come from `git log -S`. No commit message gives a reason for any of them.

- Confidence floor of 60 for PLAUSIBLE at MEDIUM or higher (`skills/pr-review/SKILL.md` section 6). Set 2026-10-14, reworded into the single gate 2026-11-03. No measured source; change it when a run shows a better value.
- `Up to 6 candidates per angle, up to 8 on high-risk`, and the gap sweep's `up to 8` (`skills/pr-review/SKILL.md` section 5). Set 2026-10-14. No measured source; change it when a run shows a better value.
- Watcher `-P 3` (three reviews at once). Set 2026-11-03. The skill gives a reason, enough to drain a morning's requests without hitting rate limits, but no measurement. No measured source; change it when a run shows a better value.
- Watcher `60m` timeout per review and `--max-budget-usd 10` per review (`skills/pr-review-watch/SKILL.md`). Both set 2026-11-03. No measured source; change it when a run shows a better value.

## Sources

- Claude Code built-in `/code-review` recipe, extracted from the `claude` 2.1.299 binary (strings `Angle A`, `PLAUSIBLE by default`, `Sweep for gaps`)
- [Claude Code review docs](https://code.claude.com/docs/en/code-review)
- [claude-plugins-official code-review command](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/code-review/commands/code-review.md)
- [claude-code-security-review prompts](https://github.com/anthropics/claude-code-security-review)
- [Codex review rubric](https://github.com/openai/codex/blob/main/codex-rs/prompts/templates/review/rubric.md), also embedded in the `codex` 0.162.1 binary
- [Codex cloud code review](https://developers.openai.com/codex/cloud/code-review)
- [Codex prompting guide](https://developers.openai.com/cookbook/examples/gpt-5/codex_prompting_guide)
- [A practical approach to verifying code at scale, OpenAI](https://alignment.openai.com/scaling-code-verification/)
- [Building Bugbot, Cursor](https://cursor.com/blog/building-bugbot)
- [Make LLMs shut up, Greptile](https://www.greptile.com/blog/make-llms-shut-up)
- [Greptile v3 agentic code review](https://www.greptile.com/blog/greptile-v3-agentic-code-review)
- [Greptile model inversion](https://www.greptile.com/blog/model-inversion)
- [How CodeRabbit delivers accurate reviews](https://www.coderabbit.ai/blog/how-coderabbit-delivers-accurate-ai-code-reviews-on-massive-codebases)
- [Less is more, Qodo](https://www.qodo.ai/blog/effective-code-suggestions-llms-less-is-more/)
- [How we built Ellipsis](https://www.nsbradford.com/blog/how-we-built-ellipsis)
- [uReview, Uber](https://www.uber.com/us/en/blog/ureview/)
- [AutoCommenter, Google](https://arxiv.org/abs/2405.13565)
- [What to look for in a code review, Google](https://google.github.io/eng-practices/review/reviewer/looking-for.html)
- [How to write code review comments, Google](https://google.github.io/eng-practices/review/reviewer/comments.html), [The standard of code review](https://google.github.io/eng-practices/review/reviewer/standard.html)
- [PatchGuru](https://arxiv.org/pdf/2602.05270), [OpenCodeReview](https://arxiv.org/abs/2608.09290)
- [SWE-PRBench](https://arxiv.org/abs/2603.26130), [SWR-Bench](https://arxiv.org/abs/2509.01494), [Defect-focused review](https://arxiv.org/abs/2505.17928), [Refute-or-Promote](https://arxiv.org/abs/2604.19049)
- [Obsessive inference optimization, Cloudflare](https://blog.cloudflare.com/obsessive-inference-optimization/)
- [AI code review at Cloudflare](https://blog.cloudflare.com/ai-code-review/)
- mattpocock-skills `code-review` skill, installed plugin cache
- [How the Claude Code team does code review, Addy Osmani](https://x.com/addyosmani/status/2114604912520921136)
- [Codex review prompt, public copy](https://gist.github.com/cbh123/ce4893a10ed2b87a89d9114b08118a08), [Codex on GitHub](https://learn.chatgpt.com/docs/third-party/github)
- [CodeRabbit configuration reference](https://docs.coderabbit.ai/reference/configuration), [PostHog #105516](https://github.com/PostHog/posthog/pull/105516), [How AI code review works, cubic](https://www.cubic.dev/blog/how-ai-code-review-works)
- [Greptile, your first PR review](https://www.greptile.com/docs/code-review/first-pr-review)
- [GitHub docs, collapsed sections](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/organizing-information-with-collapsed-sections), [commenting on a pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/commenting-on-a-pull-request), [basic formatting syntax, alerts](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
- [Conventional Comments](https://conventionalcomments.org)
- [Claude Code docs: headless](https://code.claude.com/docs/en/headless), [CLI reference](https://code.claude.com/docs/en/cli-reference), [permission modes](https://code.claude.com/docs/en/permission-modes), [scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks), [skills](https://code.claude.com/docs/en/skills)
- [Solving code review at Anthropic](https://claude.dev/blog/solving-code-review-at-anthropic/)
- [Claude Code docs, reviewing pull requests](https://code.claude.com/docs/en/code-review-pull-requests)
- [Bugbot docs](https://cursor.com/docs/bugbot)
- [Greptile changelog](https://www.greptile.com/changelog), [memory and learning](https://www.greptile.com/docs/how-greptile-works/memory-and-learning)
- [Copilot code review docs](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review)
- [How to make an agentic loop converge in AI code review](https://dev.to/uss/how-to-make-an-agentic-loop-converge-in-ai-code-review-4i1c), [Agentic code review loop, Robin Wieruch](https://www.robinwieruch.de/agentic-code-review-loop/)
