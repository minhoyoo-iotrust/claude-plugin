---
name: milestone-runner
description: Sequentially execute GSD milestones from objective documents. Scans for PLANNED objectives, then runs new-milestone → autopilot → complete → PR → merge for each. Use when user asks to run multiple milestones, batch execute objectives, auto-run remaining milestones, or process objective documents end-to-end. Also use when user says "milestone runner", "run milestones", "batch milestones", or "process objectives".
argument-hint: "[--dry-run] [--from <filename-prefix>] [--objectives <dir>] [--base <branch>]"
disable-model-invocation: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Agent
  - Skill
  - AskUserQuestion
---

NOTE: This file is kept for reference only. The actual entry point is
plugins/gsdu/commands/milestone-runner.md, which contains the full inlined workflow.

<objective>
Sequentially execute multiple GSD milestones from objective documents.

For each PLANNED objective: new-milestone → autopilot → complete-milestone → PR → CI → merge → pull base branch.
</objective>

<context>
**Arguments:**
- `$ARGUMENTS` — optional flags
- `--dry-run` — Show execution plan without running anything
- `--from <prefix>` — Start from objective matching this prefix (e.g., `m01-27`)
- `--objectives <dir>` — Objective documents directory (default: `objectives/`)
- `--base <branch>` — Target branch for PRs and post-merge checkout (default: repo's GitHub default branch)

**Prerequisites:**
- GSD must be installed (`~/.claude/get-shit-done/` exists)
- Objective documents with `status: PLANNED` in frontmatter
- `gh` CLI authenticated for PR/merge operations

**Invocation:**
- As plugin: `/gsdu:milestone-runner [--dry-run] [--from <prefix>] [--base <branch>]`
</context>

<success_criteria>
- [ ] All PLANNED objectives processed sequentially
- [ ] Each milestone: created, autopiloted, completed
- [ ] Each milestone: PR created, CI passed, merged to base branch
- [ ] Base branch up-to-date after each merge
- [ ] Completion report presented
</success_criteria>
