# gsdu — GSD Autopilot Plugin

A Claude Code plugin that auto-pilots [GSD (Get Shit Done)](https://github.com/gsd-build/get-shit-done) milestone workflows.

After your milestone roadmap is created, run a single command to automatically plan, execute, verify, and audit all remaining phases.

## What it does

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

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed
- [GSD](https://github.com/gsd-build/get-shit-done) installed (`~/.claude/get-shit-done/`)
- An active milestone with a roadmap (`.planning/ROADMAP.md`)

## Installation

### Option 1: Plugin (recommended)

In Claude Code:

```bash
# Add marketplace and install
/plugin marketplace add minhoyoo-iotrust/gsdu
/plugin install gsdu
```

For local development:

```bash
claude --plugin-dir /path/to/gsdu
```

### Option 2: Local skill (per project)

Copy `skills/autopilot/` to your project's `.claude/skills/gsd-autopilot/`:

```bash
cp -r skills/autopilot/ /your/project/.claude/skills/gsd-autopilot/
```

Invoked as `/gsd-autopilot` instead of `/gsdu:autopilot`.

### Option 3: Global skill

Copy to `~/.claude/skills/gsd-autopilot/` for all projects:

```bash
cp -r skills/autopilot/ ~/.claude/skills/gsd-autopilot/
```

## Usage

```bash
# Run from first incomplete phase
/gsdu:autopilot

# Start from a specific phase
/gsdu:autopilot 270

# Preview execution plan without running
/gsdu:autopilot --dry-run
```

## Safety rails

- Phase gap closure loops capped at 2 iterations
- Milestone gap closure loops capped at 2 iterations
- `checkpoint:human-action` always pauses (never auto-approved)
- Errors stop execution with a progress report
- Resumable: re-run after interruption and it picks up where it left off

## How it works

The plugin orchestrates existing GSD workflows without modifying them:

1. **gsd-tools CLI** — state queries and mutations
2. **GSD workflow files** — passed to subagents as execution context
3. **GSD subagent types** — `gsd-planner`, `gsd-executor`, `gsd-verifier`, `gsd-integration-checker`

The orchestrator uses ~15% of context. Each phase plan/execute gets a fresh 200k context via Task subagents.

## License

MIT
