# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Prompt files for Claude Code, packaged as the `tradecraft` plugin (its own marketplace is `claude-code-tradecraft`). There is nothing to build, lint, or unit test. `skills/<name>/SKILL.md` are the slash commands, `agents/*.md` are subagent definitions, and `docs/` holds the sourced rules they are written against plus the flowcharts the README embeds.

## Working with the files

- Install: `/plugin install tradecraft --marketplace kchojn/claude-code-tradecraft`. The installed plugin is a copy, so an edit here does nothing in an installed session. `/tradecraft:pr-review-watch` runs the installed plugin's `/tradecraft:pr-review`, not the one in the working tree.
- Development loop: `claude --plugin-dir /path/to/clone` loads the working tree as the plugin.
- The only test is a real run: `/tradecraft:lead <task>`, `/tradecraft:pr-review <pr>` (dry-run by default: prints findings and dropped candidates, touches nothing on GitHub), `/tradecraft:pr-review-watch`. Headless: `claude -p "/tradecraft:pr-review <url>"`.
- Watcher state lives in `~/.cache/pr-review-watch/` (the `lock` directory and one result file per PR head); clone roots in `~/.config/pr-review-watch/roots`.
- Frontmatter is the contract. Agents use `name`, `description`, `model`, `effort`, `tools`, `disallowedTools`, `omitClaudeMd`, `maxTurns`. Skills use `description`, `argument-hint`, `disable-model-invocation`. Unknown fields are ignored silently, so a typo turns a rule off with no warning. An agent `description` is a routing rule: open with "Use for" or "Use when".
- Commit scopes are the skill names: `feat(pr-review)`, `fix(pr-review-watch)`, `feat(lead)`, `docs(readme)`.
- ASCII only, no em dashes anywhere, including inside the prompts and the flowcharts.

## How the pieces fit

Three layers, each checked against the one below it.

1. **`agents/` fixes the roster and the model routing.** `scout` (haiku) and `qa-runner` (haiku) never write or review code; `implementer` (sonnet) writes from a spec; `reviewer` (sonnet, high) finds or verifies defects read-only; `deep-reasoner` (opus) designs and debugs; `oracle` (fable) is a one-shot read-only escalation. Skills name agents and never pass `model`, except `model: opus` to retry a failed `implementer` phase; the frontmatter decides. `scout`, `reviewer`, and `qa-runner` set `omitClaudeMd: true`, so a brief must restate any rule they have to follow.
2. **`skills/` are orchestrators that share one vocabulary.** A brief is Goal, Inputs, Constraints, Permissions, Done when, Expected output. A report opens with `DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED`. A failed phase retries on the same or a stronger model, goes to `oracle` once after two failures, then to Codex, called directly through the plugin's companion script, when a verdict stays disputed. Parallel briefs to one agent type open with identical text so they share a prompt cache. `/tradecraft:pr-review` adds a finder and verifier split (recall-biased finder, precision-biased verifier that must cite `file:line`), verdicts `CONFIRMED | PLAUSIBLE | REFUTED`, and keeps confidence (does the defect exist) apart from severity (does it block). `/tradecraft:pr-review-watch` is a scheduler only. It runs one `claude -p "/tradecraft:pr-review <url> --publish"` process per PR, because a subagent cannot spawn subagents, and it runs three at a time.
3. **`docs/` holds the rules with their sources.** `orchestration-best-practices.md` governs `/tradecraft:lead` and the agents; `pr-review-research.md` governs `/tradecraft:pr-review` and `/tradecraft:pr-review-watch`; `lead-review.md` maps each practice to the line in `skills/lead/SKILL.md` that implements it.

## Changing a skill

- Every rule traces to a cited source or a recorded run, or is listed as unmeasured in `docs/pr-review-research.md` section 17. Add the rule to the matching `docs/` file with its source first, then change the command. A lesson from a real run goes in as a numbered section; `pr-review-research.md` sections 7 to 10, 15, and 16 show the shape.
- A change to `/tradecraft:lead` means updating the table in `docs/lead-review.md`.
- A change to a flow means editing `docs/<name>-flow.html` (hand-written HTML with an inline SVG) and re-exporting `docs/<name>-flow.svg`.
- The Codex escalation sentence is duplicated in the lead and pr-review skills. Change it in both.
- The README flow paragraphs restate rules from the skills. A rule change updates the README.
- Every user-visible change adds a line under `## [Unreleased]` in `CHANGELOG.md`.
- `/tradecraft:pr-review` sections refer to each other by number. Keep the numbers stable or fix every reference.
- `/tradecraft:pr-review-watch` must stay model-invocable: `/loop` fires a skill only without `disable-model-invocation`.
