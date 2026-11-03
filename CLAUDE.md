# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Prompt files for Claude Code, not a program. There is nothing to build, lint, or unit test. `commands/*.md` are slash commands, `agents/*.md` are subagent definitions, and `docs/` holds the sourced rules they are written against plus the flowcharts the README embeds.

## Working with the files

- Install: `cp commands/*.md ~/.claude/commands/ && cp agents/*.md ~/.claude/agents/`. The installed files are plain copies, not symlinks. An edit here does nothing until it is copied again, and `/pr-review-watch` runs the installed `/pr-review`, not the repo one.
- The only test is a real run: `/lead <task>`, `/pr-review <pr>` (dry-run by default: prints findings and dropped candidates, touches nothing on GitHub), `/pr-review-watch`. Headless: `claude -p "/pr-review <url>"`.
- Watcher state lives in `~/.cache/pr-review-watch/` (the `lock` directory and one result file per PR head); clone roots in `~/.config/pr-review-watch/roots`.
- Frontmatter is the contract. Agents use `description`, `model`, `effort`, `tools`, `disallowedTools`, `omitClaudeMd`, `maxTurns`. Commands use `description`, `argument-hint`, `disable-model-invocation`. Unknown fields are ignored silently, so a typo turns a rule off with no warning. A `description` is a routing rule: open with "Use for" or "Use when".
- Commit scopes are the command names: `feat(pr-review)`, `fix(pr-review-watch)`, `feat(lead)`, `docs(readme)`.
- No em dashes anywhere, including inside the prompts.

## How the pieces fit

Three layers, each checked against the one below it.

1. **`agents/` fixes the roster and the model routing.** `scout` (haiku) and `qa-runner` (haiku) never write or review code; `implementer` (sonnet) writes from a spec; `reviewer` (sonnet, high) finds or verifies defects read-only; `deep-reasoner` (opus) designs and debugs; `oracle` (fable) is a one-shot read-only escalation. Commands name agents and never pass `model`; the frontmatter decides. `scout`, `reviewer`, and `qa-runner` set `omitClaudeMd: true`, so a brief must restate any rule they have to follow.
2. **`commands/` are orchestrators that share one vocabulary.** A brief is Goal, Inputs, Constraints, Permissions, Done when, Expected output. A report opens with `DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED`. A failed phase retries on the same or a stronger model, goes to `oracle` once after two failures, then to `/codex:rescue --background` when a verdict stays disputed. Parallel briefs to one agent type open with identical text so they share a prompt cache. `/pr-review` adds a finder and verifier split (recall-biased finder, precision-biased verifier that must cite `file:line`), verdicts `CONFIRMED | PLAUSIBLE | REFUTED`, and keeps confidence (does the defect exist) apart from severity (does it block). `/pr-review-watch` is a scheduler only: one `claude -p "/pr-review <url> --publish"` process per PR, three at a time, because a subagent cannot spawn subagents.
3. **`docs/` holds the rules with their sources.** `orchestration-best-practices.md` governs `/lead` and the agents; `pr-review-research.md` governs `/pr-review` and `/pr-review-watch`; `lead-review.md` maps each practice to the line in `lead.md` that implements it.

## Changing a command

- Every rule traces to a cited source or a recorded run. Add the rule to the matching `docs/` file with its source first, then change the command. A lesson from a real run goes in as a numbered section; `pr-review-research.md` sections 7 to 10, 15, and 16 show the shape.
- A change to `/lead` means updating the table in `docs/lead-review.md`.
- A change to a flow means editing `docs/<name>-flow.html` (hand-written HTML with an inline SVG) and re-exporting `docs/<name>-flow.svg`.
- `/pr-review` sections refer to each other by number. Keep the numbers stable or fix every reference.
- `/pr-review-watch` must stay model-invocable: `/loop` fires a skill only without `disable-model-invocation`.
