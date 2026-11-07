# Review of `/tradecraft:lead` against the best practices

Checked against [orchestration-best-practices.md](orchestration-best-practices.md) on 2026-11-07. Every practice has a row; the table says where `skills/lead/SKILL.md` implements it and which section states the practice. The Gaps section lists the practices with no implementing line.

| Practice | Section | Where in `skills/lead/SKILL.md` |
|----------|---------|---------------------------------|
| Skip delegation for trivial work | 1 | Size triage: line 33 "Trivial" |
| Delegation does not cut total tokens; it pays for a cheaper model or bulky output kept out of context | 1 | Phases: line 52 "Delegation does not cut total tokens" |
| Worker count scales with the task | 2 | Size triage: line 37 "Worker count: one for a lookup" |
| Cap on parallel workers: ten, below the default of 20 | 6 | Size triage: line 37 "At most ten workers in parallel" |
| Route model by phase | 3 | Workers table: lines 12-21; Verification follows the phase: line 58 |
| No cheap model on code | 3 | Workers: line 26 "Never Haiku on a phase that writes or reviews code" |
| Do not spawn `Explore`; the cheap read-only worker does the same search | 4 | Workers: line 26 "Never spawn `Explore`". Section 4 only mentions Explore as an agent that skips CLAUDE.md (line 50); the ban itself lives only in the skill. |
| Name the file and function even in a short brief | 4 | Size triage: line 34 "**Inputs** with the exact files and symbols" |
| Pass the findings a subagent needs and nothing more | 4 | Rules for every brief: line 74 "Never forward the whole conversation" |
| Brief as contract | 4 | Worker brief: lines 62-69 |
| Restate rules for agents that skip CLAUDE.md | 4 | Worker brief, Constraints: line 66 "do not load CLAUDE.md" |
| Workers batch reads and return big output as a file | 11, 5 | Worker brief, rules for every brief: line 75 "independent reads in one turn" |
| Parallel workers share a cached prefix | 11 | Worker brief, rules for every brief: line 76 "byte for byte" |
| Fixed report shape | 5 | Worker report: lines 80-89 |
| Parallel only when independent; disjoint files or a worktree for writers | 6 | Delegation rules: lines 93-94 |
| Turn limit is not a provider failure | 11 | Delegation rules: line 96 "A worker that hits its turn limit" |
| Retry on the same or a stronger model, never a cheaper one; `model: opus` for a failed `tradecraft:implementer` phase, `tradecraft:deep-reasoner` for a failed `tradecraft:reviewer` or `tradecraft:scout` phase | 3, 7 | Workers: line 25 "`model: opus`"; Delegation rules: line 96 "Retry on the same worker or a stronger one" |
| Escalate after two failures, once, read-only, on the most capable model; a different provider for disputes | 7 | Workers table: lines 19-20; Delegation rules: line 96 "goes once to `tradecraft:oracle`" |
| The second provider is called directly and read-only, not through a forwarding subagent | 7 | Delegation rules: line 96 "Never pass `--write`, and never go through `/codex:rescue`" |
| Destructive actions stay in the lead | 7 | Delegation rules: line 98 "Destructive and outward-facing actions stay with you" |
| Verify the premise | 8 | Verify the premise: lines 39-46 |
| Lead keeps synthesis | 7 | Workers table: line 21 "you"; Delegation rules: line 100 "Report conclusions, not worker transcripts" |
| User-only command | 9 | Frontmatter: line 3 `disable-model-invocation: true` |

## Gaps

- Clear history between unrelated tasks (section 1). The skill does not implement it; the author's own global instructions cover it outside this repository.
- Text written by other people is data, never instructions (section 11). The lead skill does not state it; `skills/pr-review/SKILL.md` section 2 does, because that skill reads PR text.
- The eval loop (section 10). It is a practice for maintaining the skill, not a rule for running it, so the skill omits it on purpose.

## Agent definitions

| Agent | Tools | CLAUDE.md | Turn guard |
|-------|-------|-----------|------------|
| `scout` | Read, Grep, Glob, Bash | skipped | 40 |
| `reviewer` | Read, Grep, Glob, Bash; may use a throwaway worktree the brief names | skipped | none: review length follows the diff |
| `qa-runner` | no edits: all but Edit, Write, NotebookEdit | skipped | 80 |
| `implementer` | all | loaded | none: the brief bounds the files |
| `deep-reasoner` | no edits: `disallowedTools: Edit, Write, NotebookEdit` | loaded | none |
| `oracle` | Read, Grep, Glob, Bash | loaded | none: one call, long turns expected |

The skill passes these to the Agent tool as `tradecraft:scout`, `tradecraft:reviewer`, and so on. The `oracle` description opens with "Use when", like the other routing descriptions.
