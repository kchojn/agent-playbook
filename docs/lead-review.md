# Review of `/lead` against the best practices

Checked against [orchestration-best-practices.md](orchestration-best-practices.md).

## Already covered

| Practice | Where in `commands/lead.md` |
|----------|-----------------------------|
| Skip delegation for trivial work | Size triage: Trivial / S / M+ |
| Route model by phase | Route table; "Effort follows the phase" |
| No cheap model on code | "Never put Haiku on a phase that writes code" |
| Brief as contract | Worker brief: Goal, Inputs, Constraints, Permissions, Done when, Expected output |
| Fixed report shape | Worker report with Status token |
| Parallel only when independent | Delegation rules |
| Retry never cheaper; escalate after two failures | Delegation rules |
| Verify the premise | "Verify the premise" section |
| Lead keeps synthesis | Delegation rules |

## Gaps

1. **No worker-count rule.** Add Anthropic's scaling table (1 worker for a lookup, 2-4 for a comparison, more only when scope divides cleanly) and a cap on spawns per task.
2. **No write-conflict rule.** Parallel writing phases need disjoint files, or `isolation: worktree`.
3. **No large-output rule.** Tell workers to write big results to a file in the scratchpad and return the path.
4. **No CLAUDE.md propagation rule.** `scout` and `qa-runner` set `omitClaudeMd: true`, so the brief must restate any rule they need.
5. **No runaway guard.** Add `maxTurns` to each agent definition.
6. **No destructive-action rule.** State that push, delete, force operations, and publishing stay in the lead's thread.
7. **No eval loop.** Keep about 20 real tasks and score the lead's routing on outcome, cost, and turns.

## Agent definitions

- `scout`: tool allowlist and `omitClaudeMd` match the practice. Add `maxTurns`.
- `qa-runner`: `disallowedTools` blocks edits. Add `maxTurns`.
- `implementer`, `deep-reasoner`: no tool limits and CLAUDE.md loaded. Correct for writers and reasoners. Add `maxTurns` to `implementer`.
