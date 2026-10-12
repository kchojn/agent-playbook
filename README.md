# agent-playbook

Personal collection of Claude Code commands, subagent definitions, and notes on how to orchestrate agents well.

## Layout

| Path | Contents |
|------|----------|
| `commands/` | Slash commands (`lead`, `pr-review`). Copy to `~/.claude/commands/`. |
| `agents/` | Subagent definitions (`scout`, `qa-runner`, `implementer`, `deep-reasoner`). Copy to `~/.claude/agents/`. |
| `docs/orchestration-best-practices.md` | Sourced rules for delegation, model routing, briefs, and evals. |
| `docs/lead-review.md` | `/lead` checked against those rules, with gaps. |

## Use

```sh
cp commands/*.md ~/.claude/commands/
cp agents/*.md ~/.claude/agents/
```

Then run `/lead <task>` in Claude Code.

## Add to this repo

Put new commands, agents, and findings here. Cite the source for every rule in `docs/`.
