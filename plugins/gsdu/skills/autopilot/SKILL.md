---
name: autopilot
description: Auto-pilot remaining GSD phases — plan, execute, audit, gap closure. Use when user asks to run autopilot, auto-execute remaining phases, complete all GSD phases automatically, or run the full milestone workflow end-to-end. Also use when user says "autopilot", "auto-pilot", "run all phases", or "execute remaining phases". Requires GSD to be installed at ~/.claude/get-shit-done/.
argument-hint: "[start-phase] [--dry-run]"
disable-model-invocation: false
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Task
  - TodoWrite
  - AskUserQuestion
---

<objective>
Automatically execute all remaining phases in the current GSD milestone.

For each unfinished phase: plan (if needed) → execute → verify → gap closure (if needed).
After all phases complete: audit milestone → handle gaps → report.

Orchestrator stays lean (~15% context). Each phase plan/execute is delegated to subagents with fresh context.
</objective>

<execution_context>
@WORKFLOW.md
</execution_context>

<context>
**Arguments:**
- `$ARGUMENTS` — optional start phase number and flags
- `start-phase` — Phase number to start from (default: first incomplete phase)
- `--dry-run` — Show execution plan without running anything

**Prerequisites:**
- GSD must be installed (`~/.claude/get-shit-done/` exists)
- A milestone roadmap must exist (`.planning/ROADMAP.md`)
- At least one phase must be unfinished

**Invocation:**
- As plugin: `/gsdu:autopilot [start-phase] [--dry-run]`
- As local skill: `/gsd-autopilot [start-phase] [--dry-run]`
</context>

<process>
Execute the autopilot workflow from @WORKFLOW.md end-to-end.
Preserve all safety rails (gap closure max iterations, error handling, checkpoint passthrough).
</process>

<success_criteria>
- [ ] All remaining phases planned and executed
- [ ] Phase-level verification passed (or gaps closed)
- [ ] Milestone audit completed
- [ ] Milestone-level gaps closed (if any)
- [ ] Completion report presented with `/gsd:complete-milestone` guidance
</success_criteria>
