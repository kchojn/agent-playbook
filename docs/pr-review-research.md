# PR review research

What the strongest published review systems do to catch real bugs without noise, and which of it `/pr-review` adopts. Each rule cites its source. Vendor numbers are self-reported unless marked otherwise.

## 1. Split finding from judging

- Claude Code's built-in `/code-review` runs independent finder "angles", each returning candidates with a file, line, summary, and concrete failure scenario, then one verifier per candidate. Its prompt states the reason: "finders that silently drop half-believed candidates bypass the verify step and are the dominant cause of misses." [Claude Code binary, code-review recipe]
- Cursor's first Bugbot ran eight passes over the diff, bucketed similar bugs, took a majority vote, then ran "a validator model to catch false positives". Later, a cautious agent "was too cautious", so they moved to aggressive prompts behind the validator. Resolution rate went from 52% to over 70%. [Building Bugbot]
- Anthropic's plugin runs five Sonnet reviewers in parallel, then one scorer per issue, and posts only issues scoring at least 80 out of 100. [claude-plugins-official code-review]
- Uber's uReview stages are generate, filter, validate, dedupe. A second prompt scores confidence with thresholds per category. 75% of comments were rated useful. [uReview]
- Ellipsis attaches evidence links to each candidate and runs a hallucination check against that evidence. [How we built Ellipsis]

Adopted: section 5 (recall-biased finder, every candidate with a failure scenario passes through) and section 6 (one precision-biased verifier that must cite `file:line` evidence). High-risk PRs get a second finder that reads the files in reverse order, after Bugbot's "different ordering of the diff" trick.

## 2. Precision over recall for anything published

- OpenAI: "Precision is more important for usability than recall." A finding is worth surfacing when P(correct) × cost saved exceeds the cost of human verification plus P(incorrect) × cost of a false alarm. Stylistic comments "may even incur negative utility." [Verifying code at scale]
- Greptile audited its comments: 19% good, 2% wrong, 79% nits. Prompting against nits also cut critical comments, and an LLM judge scoring 1 to 10 was "nearly random". What worked was blocking comments similar to three or more previously downvoted ones; address rate went from 19% to over 55%. [Make LLMs shut up]
- Qodo: exclusion lists failed; a single positive instruction, "only give suggestions that address major problems and bugs", with an empty list allowed, raised acceptance 50%. [Less is more]
- Google AutoCommenter: 80% of candidate comments landed on unmodified lines; restricting to changed lines cut the share of files commented from 6% to 1.3%. Suppressing 17 non-actionable categories raised the useful ratio from 54% to 66%. [AutoCommenter, arXiv 2405.13565]
- Anthropic cloud review reports under 1% of findings marked incorrect, and tunes nits with a cap of five per review. [Claude Code review docs]

Adopted: section 1 opens with the positive bar ("problems a maintainer would fix") and allows an empty list; the never-report list follows. Section 7 caps LOW findings at five. Section 6 gates publication at CONFIRMED with confidence 80, or PLAUSIBLE at HIGH or BLOCKER with confidence 70. Section 9 dedupes against earlier comments; angle H skips complaints the author already answered as intentional.

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

- Codex priorities: P0 "only for universal issues that do not depend on any assumptions about the inputs", then P1 urgent, P2 normal, P3 nice to have. Comments are one paragraph, code under four lines, range under ten lines, and must "clearly and explicitly communicate the scenarios, environments, or inputs that are necessary for the bug to arise". No praise. An overall verdict, "patch is correct" or "patch is incorrect", ignores nits. GitHub shows only P0 and P1. [Codex rubric.md; Codex cloud docs]
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
- [PatchGuru](https://arxiv.org/pdf/2602.05270), [OpenCodeReview](https://arxiv.org/abs/2608.09290)
- [SWE-PRBench](https://arxiv.org/abs/2603.26130), [SWR-Bench](https://arxiv.org/abs/2509.01494), [Defect-focused review](https://arxiv.org/abs/2505.17928), [Refute-or-Promote](https://arxiv.org/abs/2604.19049)
- mattpocock-skills `code-review` skill, installed plugin cache
