# Review of `/lead` against the best practices

Checked against [orchestration-best-practices.md](orchestration-best-practices.md) on 2026-10-21. Every rule has a home; the table says where.

| Practice | Where in `commands/lead.md` |
|----------|-----------------------------|
| Skip delegation for trivial work | Size triage: Trivial / S / M+ |
| Worker count scales with the task, with a cap | Size triage, last paragraph |
| Route model by phase | Workers table; Verification follows the phase |
| No cheap model on code | Workers: "Never Haiku on a phase that writes or reviews code" |
| Brief as contract | Worker brief |
| Restate rules for agents that skip CLAUDE.md | Worker brief: Constraints |
| Workers batch reads and return big output as a file | Worker brief: rules for every brief |
| Parallel workers share a cached prefix | Worker brief: rules for every brief |
| Fixed report shape | Worker report |
| Parallel only when independent; disjoint files or a worktree for writers | Delegation rules |
| Turn limit is not a provider failure | Delegation rules |
| Retry never cheaper; escalate after two failures | Workers; Delegation rules |
| Escalation to the most capable model, once, read-only; a different provider for disputes | Workers table: `oracle`, `/codex:rescue`; Delegation rules |
| Destructive actions stay in the lead | Delegation rules |
| Verify the premise | Verify the premise |
| Lead keeps synthesis | Workers table: "you"; Delegation rules |
| User-only command | frontmatter `disable-model-invocation: true` |

Not in the command, on purpose: the eval loop from section 10. It is a practice for maintaining the command, not a rule for running it.

## Agent definitions

| Agent | Tools | CLAUDE.md | Turn guard |
|-------|-------|-----------|------------|
| `scout` | Read, Grep, Glob, Bash | skipped | 40 |
| `reviewer` | Read, Grep, Glob, Bash | skipped | none: review length follows the diff |
| `qa-runner` | all but Edit, Write, NotebookEdit | skipped | 80 |
| `implementer` | all | loaded | none: the brief bounds the files |
| `deep-reasoner` | all | loaded | none |
| `oracle` | Read, Grep, Glob, Bash | loaded | none: one call, long turns expected |
