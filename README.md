# gsdu — GSD Utilities Plugin

A Claude Code plugin for [GSD (Get Shit Done)](https://github.com/gsd-build/get-shit-done) workflow automation.

## Commands

### `/gsdu:autopilot`

Auto-pilots remaining GSD phases within a single milestone.

```
/gsdu:autopilot
    │
    ├─ For each unfinished phase (sequentially):
    │   ├─ Plan (if no plans exist)
    │   ├─ Execute (wave-based, with verification)
    │   └─ Gap closure (if verification finds gaps, max 2 iterations)
    │
    ├─ Milestone audit (cross-phase integration check)
    │   └─ Auto gap closure (if audit finds gaps, max 2 iterations)
    │
    └─ Completion report + /gsd:complete-milestone guidance
```

**Usage:**

```bash
/gsdu:autopilot              # Run from first incomplete phase
/gsdu:autopilot 270          # Start from a specific phase
/gsdu:autopilot --dry-run    # Preview execution plan
```

### `/gsdu:milestone-runner`

Sequentially executes multiple GSD milestones from objective documents. Automates the full cycle: new-milestone → autopilot → complete → PR → merge.

```
/gsdu:milestone-runner
    │
    ├─ Scan objectives/ for PLANNED documents
    │
    ├─ For each objective (sequentially):
    │   ├─ /gsd:new-milestone (version + context from objective)
    │   ├─ /gsdu:autopilot (plan → execute → verify all phases)
    │   ├─ /gsd:complete-milestone (archive + tag)
    │   ├─ Create PR → Wait CI → Merge (--merge)
    │   └─ Checkout main → Pull
    │
    └─ Completion report with all milestone results
```

**Usage:**

```bash
/gsdu:milestone-runner                           # Run all PLANNED objectives
/gsdu:milestone-runner --dry-run                 # Preview execution plan
/gsdu:milestone-runner --from m01-27             # Start from specific objective
/gsdu:milestone-runner --objectives docs/goals/  # Custom objectives directory
```

**Objective document format:**

```markdown
---
status: PLANNED
---

# Milestone Title

## Goal
Description of what this milestone achieves...
```

- Files must have `status: PLANNED` in YAML frontmatter
- Version is extracted from filename pattern `m{major}-{minor}-*` → `v{major}.{minor}`
- Or specify `version: v1.26` in frontmatter explicitly

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed
- [GSD](https://github.com/gsd-build/get-shit-done) installed (`~/.claude/get-shit-done/`)
- `gh` CLI authenticated (for milestone-runner PR/merge operations)
- An active milestone with a roadmap for autopilot (`.planning/ROADMAP.md`)
- Objective documents for milestone-runner (`objectives/` or custom directory)

## Installation

### Option 1: Plugin (recommended)

In Claude Code:

```
# 1. Add marketplace
/plugin marketplace add minhoyoo-iotrust/claude-plugin

# 2. Install plugin
/plugin install gsdu
```

To update the plugin:

```
/plugin update gsdu@iotrust.kr
```

To load from a local directory (for development/testing):

```bash
claude --plugin-dir /path/to/claude-plugin/plugins/gsdu
```

### Option 2: Local skill (per project)

Copy the desired skill to your project's `.claude/skills/`:

```bash
# Milestone Runner
cp -r plugins/gsdu/skills/milestone-runner/ /your/project/.claude/skills/gsd-milestone-runner/
```

### Option 3: Global skill

Copy to `~/.claude/skills/` for all projects:

```bash
cp -r plugins/gsdu/skills/milestone-runner/ ~/.claude/skills/gsd-milestone-runner/
```

## Safety rails

- Phase gap closure loops capped at 2 iterations
- Milestone gap closure loops capped at 2 iterations
- `checkpoint:human-action` always pauses (never auto-approved)
- Errors stop execution with a progress report
- Resumable: re-run with `--from` flag to pick up where it left off
- PR merge always uses `--merge` (never squash)

## How it works

Both commands orchestrate existing GSD workflows without modifying them:

1. **gsd-tools CLI** — state queries and mutations
2. **GSD workflow files** — passed to subagents as execution context
3. **GSD subagent types** — `gsd-planner`, `gsd-executor`, `gsd-verifier`, `gsd-integration-checker`
4. **GSD skills** — `/gsd:new-milestone`, `/gsd:complete-milestone` (milestone-runner only)

The orchestrators use ~15% of context. Each phase plan/execute gets a fresh 200k context via subagents.

## License

MIT
