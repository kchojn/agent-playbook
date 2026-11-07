# Contributing

This repo holds prompt files for Claude Code: skills, subagent definitions, and the sourced rules they follow. There is
nothing to build, lint, or unit test. Frontmatter fields and the way the pieces fit are described in `CLAUDE.md`.

## Development loop

1. Clone the repo: `git clone https://github.com/kchojn/claude-code-tradecraft`.
2. Edit a file under `skills/`, `agents/`, or `docs/`.
3. Start Claude Code with the clone loaded as a plugin: `claude --plugin-dir /path/to/clone`.
4. Run the skill for real: `/tradecraft:lead <task>`, `/tradecraft:pr-review <pr>`, or `/tradecraft:pr-review-watch`.

The only test is a real run. `/tradecraft:pr-review <pr>` is a dry run by default: it prints findings and dropped
candidates and touches nothing on GitHub.

## Rules for a change

- Every rule in a skill traces to a cited source or a recorded run, or is listed as unmeasured in
  `docs/pr-review-research.md` section 17. Add the rule and its source to the matching file in
  `docs/` first, then change the skill.
- A change to a flow edits `docs/<name>-flow.html`, which is the source, and re-exports `docs/<name>-flow.svg` with the
  diagram-design skill's `export_svg.py` (`--system-fonts`, because GitHub renders the SVG as an image and loads no web
  fonts). The flowcharts use that skill's `tradecraft` profile; `.diagram-design` names it and the `:root` block of each
  flow file lists the tokens.
- Add one line under `## [Unreleased]` in `CHANGELOG.md` for every change a user can see.

## Commits and branches

- Commit and PR titles read `type(scope): imperative summary`. The scope is the skill name, for example
  `fix(pr-review): ...` or `feat(lead): ...`, or `readme` and `docs` for those files. Use `type: imperative summary` when
  no scope fits.
- Branches read `type/kebab-case-summary`, for example `fix/watcher-lock-path`.
