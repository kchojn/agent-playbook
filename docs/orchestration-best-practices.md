# Orchestration best practices

Rules for a lead agent that delegates to subagents in Claude Code. Each rule cites its source. Where a secondary source conflicts with the official docs, the docs win.

## 1. Decide whether to delegate

- Start with the simplest design. Add agents only when they measurably improve the result. [Building effective agents]
- Stay in the main thread for quick targeted edits, iterative work, shared multi-phase context, and latency-sensitive steps. A non-fork subagent starts cold. [Claude Code docs]
- Delegate work shaped like "find out X and report": test runs, log reading, doc fetches, wide searches. The verbose output stays in the subagent's context. [Claude Code docs]
- Multi-agent pays off on breadth-first work that splits into independent threads. Token use explained about 80% of the quality gain in Anthropic's research system, at roughly 15x the tokens of a chat. Spend that only where the task is worth it. [Multi-agent research system]

## 2. Scale effort to the task

Anthropic's starting heuristics:

| Task | Workers | Tool calls each |
|------|---------|-----------------|
| Simple fact-finding | 1 | 3-10 |
| Direct comparison | 2-4 | 10-15 |
| Complex research | 10+ with divided scope | as needed |

Put such rules in the lead's prompt. Without them, agents over-spawn on simple queries. Add explicit guardrails against runaway spawning and endless searching. [Multi-agent research system]

## 3. Route models by phase

- Cheap model for work whose result you can check cheaply: file discovery, log scanning, running existing tests.
- Mid model for implementation from a clear spec.
- Strongest model for design, hard debugging, and review of work with many edge cases.
- Never put the cheapest model on a phase that writes or reviews code. Inconsistent results are often a routing error, not a design error.
- Model resolution order: per-call `model` parameter, frontmatter `model`, `CLAUDE_CODE_SUBAGENT_MODEL`, the main conversation's model. `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` overrides every definition. [Claude Code docs]
- Verify with the cheapest check that can catch the bug. A test run costs one turn; more effort costs thinking on every turn.

## 4. Write the brief as a contract

Every delegation states:

1. **Goal**: what the phase must accomplish.
2. **Inputs**: exact paths, symbols, errors, prior decisions.
3. **Constraints**: technical and behavioral limits.
4. **Permissions**: read-only or writes.
5. **Done when**: concrete acceptance criteria.
6. **Expected output**: the report format.

Vague briefs make workers duplicate each other or leave gaps. Anthropic's example: "research the semiconductor shortage" sent several workers on the same search. Give an objective, an output format, tool and source guidance, and clear task boundaries. [Multi-agent research system]

Subagents do not see the conversation. Pass the findings they need and nothing more. A subagent with `omitClaudeMd: true`, and the built-in Explore and Plan agents, skip CLAUDE.md, so restate any rule they must follow. [Claude Code docs]

## 5. Standardize the report

Return a fixed shape so the lead can choose the next phase without redoing the work:

- Status token: `DONE`, `DONE_WITH_CONCERNS`, `NEEDS_CONTEXT`, `BLOCKED`.
- Result, files changed, verification run, risks, deferred items.
- No transcripts and no full command output. For large artifacts, write to a file and return the path. [Orchestrating sub-agents; Konishi guide]

## 6. Parallelize by dependency

- Independent phases go in one message. Dependent phases wait for the earlier result. [Claude Code docs]
- Concurrent writers get disjoint file sets. If files overlap, use `isolation: worktree`. [Konishi guide]
- Group work by shared subsystem, not by task count. Never split work only to delegate more.
- Defaults: 3 nesting layers, 20 concurrent subagents. Tune with `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` and `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`. [Claude Code docs]

## 7. Keep judgment and risk in the lead

- Synthesis, conflict resolution, and the next-phase decision stay with the lead.
- Destructive or outward-facing actions (push, delete, force operations, publishing) stay in the main thread behind confirmation. Never delegate them.
- A phase fails when the worker is blocked, errors, or misses "Done when". Retry on the same or a stronger model, never a cheaper one. After two failures, escalate to an independent second opinion.

## 8. Verify the premise

For bugs and any ask of the form "X happens because Y, change Z", check that the behavior still exists and that Y matches the current code before implementing. Workers report a false premise as a finding. They do not build a different fix.

## 9. Define agents tightly

- `description` is a routing rule. Open with "Use for..." or "Use when...", name the trigger, keep it short. Combined descriptions over 15,000 tokens trigger a startup warning. Put detail in the body. [Claude Code docs]
- Restrict tools to the job. A read-only scout gets `tools: Read, Grep, Glob, Bash`. A tester gets `disallowedTools: Edit, Write, NotebookEdit`.
- Set `maxTurns` as a runaway guard where a bounded task has a known size.
- Fewer, sharper agents beat a large roster. One author built 100 subagents and kept about 12. [100 subagents]
- Unknown frontmatter fields are ignored silently. Check names against the docs: `disallowedTools`, `omitClaudeMd`, `maxTurns`, `isolation`.

## 10. Evaluate with a small set

- Test the lead with about 20 realistic tasks. Large effects show in small samples. [Multi-agent research system]
- Judge the outcome, not the path. Valid paths vary.
- Use an LLM judge with a rubric (accuracy, completeness, tool efficiency) and a single 0.0-1.0 score plus pass/fail. Keep human review for edge cases.
- Run the exact prompts and tools yourself first to see failure modes.

## Sources

- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [How we built our multi-agent research system - Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Building effective agents - Anthropic](https://www.anthropic.com/engineering/building-effective-agents)
- [Claude Code subagents and multi-agent orchestration guide - Konishi](https://hidekazu-konishi.com/entry/claude_code_subagents_and_orchestration_guide.html)
- [CLAUDE.md rules for subagent delegation - gist](https://gist.github.com/techygarg/f8f98a2f026538fad4a69b593a964d95)
- [Orchestrating sub-agents for cost-efficient engineering - DEV](https://dev.to/kasundsilva/orchestrating-sub-agents-for-cost-efficient-engineering-1pl3)
- [I built 100 Claude Code subagents - DEV](https://dev.to/suraj_khaitan_f893c243958/i-built-100-claude-code-subagents-these-are-the-12-that-actually-earn-their-context-10nn)
