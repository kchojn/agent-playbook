# agent-playbook

Personal collection of Claude Code commands, subagent definitions, and notes on how to orchestrate agents well.

## How `/lead` runs

![The /lead orchestration flow](docs/lead-flow.svg)

Trivial work stays with the lead. Everything else runs as bounded phases: define the phase, delegate it to one worker, check the result against "done when", retry on a stronger model (a second failure goes to `oracle`), and synthesize when no phase remains. Source: `docs/lead-flow.html`.

## How `/pr-review` runs

![The /pr-review flow](docs/pr-review-flow.svg)

One recall-biased finder lists every candidate defect with a failure scenario; one precision-biased verifier tries to disprove each and must cite `file:line`. Only CONFIRMED findings post as findings and set the review event; PLAUSIBLE ones with confidence 70 or more post as `[Optional]`. Dry-run by default; `--publish` posts one GitHub review with the comments inside it. Sources for every rule: `docs/pr-review-research.md`. Source: `docs/pr-review-flow.html`.

## How `/pr-review-watch` runs

![The /pr-review-watch flow](docs/pr-review-watch-flow.svg)

`/loop 15m /pr-review-watch` runs one pass every 15 minutes. A pass finds open PRs that asked for your review in the last 24 hours, drops the ones already reviewed at their head commit, and runs `/pr-review --publish` on the rest, each in its own `claude -p` process, three at a time, so the watching session holds one line per PR and nothing lands in `/resume`. Clone roots come from the arguments or `~/.config/pr-review-watch/roots`. Source: `docs/pr-review-watch-flow.html`.

## Layout

| Path | Contents |
|------|----------|
| `commands/` | Slash commands (`lead`, `pr-review`, `pr-review-watch`). Copy to `~/.claude/commands/`. |
| `agents/` | Subagent definitions (`scout`, `qa-runner`, `implementer`, `reviewer`, `deep-reasoner`, `oracle`). Copy to `~/.claude/agents/`. |
| `docs/orchestration-best-practices.md` | Sourced rules for delegation, model routing, briefs, and evals. |
| `docs/lead-review.md` | `/lead` checked against those rules, with gaps. |
| `docs/pr-review-research.md` | What published review systems do to catch real bugs without noise, and which of it `/pr-review` adopts. |
| `docs/*-flow.html`, `docs/*-flow.svg` | The three flowcharts above and their sources. |

## Use

```sh
cp commands/*.md ~/.claude/commands/
cp agents/*.md ~/.claude/agents/
```

Then run `/lead <task>`, `/pr-review <pr>`, or `/loop 15m /pr-review-watch` in Claude Code.

## Add to this repo

Put new commands, agents, and findings here. Cite the source for every rule in `docs/`.
