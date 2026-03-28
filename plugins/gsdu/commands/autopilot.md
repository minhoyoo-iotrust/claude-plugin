---
description: Auto-pilot remaining GSD phases (plan, execute, audit, gap closure)
argument-hint: "[start-phase] [--dry-run]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, AskUserQuestion
---

CRITICAL INSTRUCTION: Do NOT use the Skill tool to invoke gsdu:autopilot. Do NOT re-invoke this command. All workflow steps are below — execute them directly.

<objective>
Automatically execute all remaining phases in the current GSD milestone.

For each unfinished phase: plan (if needed) → execute → verify → gap closure (if needed).
After all phases complete: audit milestone → handle gaps → report.

Orchestrator stays lean (~15% context). Each phase plan/execute is delegated to subagents with fresh context.
</objective>

<arguments>
- `$ARGUMENTS` — optional start phase number and flags
- `start-phase` — Phase number to start from (default: first incomplete phase)
- `--dry-run` — Show execution plan without running anything

**Prerequisites:**
- GSD must be installed (`~/.claude/get-shit-done/` exists)
- A milestone roadmap must exist (detected via `gsd-tools.cjs init progress` — supports both standard and workstream layouts)
- At least one phase must be unfinished
</arguments>

<constraints>
- NEVER re-invoke /gsdu:autopilot or the gsdu:autopilot skill from within this workflow
- NEVER use the Skill tool to call gsdu:autopilot — this causes infinite recursion
- Use the Agent tool (not Skill tool) for all subagent delegation
- If a step fails, report the error and STOP — do not retry automatically
</constraints>

<constants>
GSD_TOOLS="node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs"
GSD_WORKFLOWS="$HOME/.claude/get-shit-done/workflows"
GSD_TEMPLATES="$HOME/.claude/get-shit-done/templates"
GSD_REFERENCES="$HOME/.claude/get-shit-done/references"
MAX_GAP_ITERATIONS=2
</constants>

<process>

<step name="preflight" priority="first">
**Verify GSD is installed and milestone is active:**

```bash
# 1. Check GSD installation
test -f "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" || echo "GSD_NOT_FOUND"
```

If `GSD_NOT_FOUND`:
```
GSD is not installed. Install it first:
  https://github.com/gsd-build/get-shit-done

Then run /gsdu:autopilot again.
```
Exit.

```bash
# 2. Load context and detect paths (supports both standard and workstream layouts)
INIT=$(node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs init progress)
```

Extract path variables from INIT JSON — these adapt automatically to workstream mode:
- `ROADMAP_PATH` ← `roadmap_path` (e.g., `.planning/ROADMAP.md` or `.planning/workstreams/{name}/ROADMAP.md`)
- `STATE_PATH` ← `state_path` (e.g., `.planning/STATE.md` or `.planning/workstreams/{name}/STATE.md`)
- `CONFIG_PATH` ← `config_path` (e.g., `.planning/config.json` or `.planning/workstreams/{name}/config.json`)
- `PLANNING_DIR` ← parent directory of `ROADMAP_PATH` (e.g., `.planning/` or `.planning/workstreams/{name}/`)
- `PHASES_DIR` ← `{PLANNING_DIR}/phases/`
- `ROADMAP_EXISTS` ← `roadmap_exists`

If `ROADMAP_EXISTS` is false:
```
No milestone roadmap found.

Start a new milestone first:
  /gsd:new-project   — for new projects
  /gsd:new-milestone  — for existing projects
```
Exit.

```bash
# 3. Analyze roadmap for phase details
ROADMAP=$(node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs roadmap analyze)
```

Extract from INIT and ROADMAP JSON:
- `phases[]` — all phases with `number`, `name`, `goal`, `disk_status`, `plan_count`, `summary_count`
- `milestone_version`
- `completed_count`, `phase_count`

**Build execution queue:**
- Filter phases where `disk_status` is NOT `complete`
- If `start-phase` argument given: skip phases before that number
- Sort by phase number ascending

If queue is empty:
```
All phases are complete! Run milestone audit:
  /gsd:audit-milestone
```
Exit.
</step>

<step name="dry_run">
**If `--dry-run` flag present, show plan and exit:**

```
## Autopilot Execution Plan

**Milestone:** v{milestone_version}
**Progress:** {completed_count}/{phase_count} phases complete

### Phases to Execute

| # | Phase | Status | Plans | Action |
|---|-------|--------|-------|--------|
| {N} | {name} | {disk_status} | {plan_count} | {plan if 0, execute if >0} |
| ... | ... | ... | ... | ... |

### After All Phases
1. Milestone audit
2. Gap closure (if needed, max {MAX_GAP_ITERATIONS} iterations)

### Safety Rails
- Phase gap closure: max {MAX_GAP_ITERATIONS} iterations per phase
- Milestone gap closure: max {MAX_GAP_ITERATIONS} iterations
- checkpoint:human-action always pauses
- Errors stop execution with progress report

Run without --dry-run to execute.
```

Exit.
</step>

<step name="phase_loop">
**Execute each phase in queue sequentially.**

For each phase `{N}` in the execution queue:

```
╔═══════════════════════════════════════════════╗
║  AUTOPILOT: Phase {N}/{phase_count}           ║
║  {phase_name}                                 ║
║  Goal: {goal}                                 ║
╚═══════════════════════════════════════════════╝
```

**2a. Check if plans exist:**

```bash
PHASE_INFO=$(node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs roadmap get-phase {N})
```

Parse `plan_count` from phase info.

**If plan_count == 0: Run plan-phase**

```
Agent(
  subagent_type="gsd-planner",
  description="Plan phase {N}",
  prompt="
    <objective>
    Create executable phase plans for Phase {N}: {phase_name}.
    Goal: {goal}
    </objective>

    <execution_context>
    @$HOME/.claude/get-shit-done/workflows/plan-phase.md
    @$HOME/.claude/get-shit-done/references/ui-brand.md
    </execution_context>

    <context>
    Phase: {N}
    Flags: --skip-research
    Auto-mode: true (skip user confirmations)
    </context>

    <files_to_read>
    - {ROADMAP_PATH}
    - {STATE_PATH}
    - .planning/REQUIREMENTS.md
    - {CONFIG_PATH} (if exists)
    - ./CLAUDE.md (if exists)
    </files_to_read>

    <process>
    Execute the plan-phase workflow end-to-end.
    Skip research (--skip-research). Skip discussion.
    Create PLAN.md files in phase directory.
    </process>
  "
)
```

After plan-phase completes, verify plans were created:
```bash
ls {PHASES_DIR}/*{N}*/*-PLAN.md 2>/dev/null | wc -l
```

If 0 plans created: report error, stop autopilot.

**If plan_count > 0: Skip planning**

```
Phase {N}: {plan_count} plan(s) already exist, skipping planning.
```

**2b. Execute phase:**

```
Agent(
  subagent_type="gsd-executor",
  description="Execute phase {N}",
  prompt="
    <objective>
    Execute all plans in Phase {N}: {phase_name}.
    Commit each task atomically. Create SUMMARY.md per plan. Run verification.
    </objective>

    <execution_context>
    @$HOME/.claude/get-shit-done/workflows/execute-phase.md
    @$HOME/.claude/get-shit-done/references/ui-brand.md
    </execution_context>

    <context>
    Phase: {N}
    Auto-mode: true
    Flags: --auto --no-transition
    </context>

    <files_to_read>
    - {STATE_PATH}
    - {ROADMAP_PATH}
    - {CONFIG_PATH} (if exists)
    - ./CLAUDE.md (if exists)
    </files_to_read>

    <process>
    Execute the execute-phase workflow end-to-end.
    Use --auto for checkpoint auto-approval (except human-action).
    Use --no-transition (autopilot handles transitions).
    Include verification step.
    </process>
  "
)
```

**2c. Check verification result:**

```bash
VERIFY_STATUS=$(grep "^status:" {PHASES_DIR}/*{N}*/*-VERIFICATION.md 2>/dev/null | tail -1 | cut -d: -f2 | tr -d ' ')
```

| Status | Action |
|--------|--------|
| `passed` | Continue to 2d |
| `human_needed` | Present items, auto-approve, continue to 2d |
| `gaps_found` | Enter gap closure loop |
| (no verification) | Warn, continue to 2d |

**Gap closure loop (max MAX_GAP_ITERATIONS):**

```
for gap_iteration in 1..MAX_GAP_ITERATIONS:

  1. Plan gap fixes:
     Agent(subagent_type="gsd-planner") with --gaps flag
     → reads VERIFICATION.md → creates gap closure PLAN.md files

  2. Execute gap fixes:
     Agent(subagent_type="gsd-executor") with --gaps-only flag

  3. Re-verify:
     Check VERIFICATION.md status again

  If passed: break loop
  If still gaps_found AND iteration == MAX_GAP_ITERATIONS:
    Log warning: "Gap closure limit reached for phase {N}. Continuing."
    break
```

**2d. Transition (mark phase complete, advance state):**

```bash
COMPLETION=$(node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs phase complete {N})
node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs(phase-{N}): complete phase execution" --files {ROADMAP_PATH} {STATE_PATH} .planning/REQUIREMENTS.md
```

```
## Phase {N}: {phase_name} — Complete

Plans: {completed}/{total} | Verification: {status}
{one-liner from SUMMARY.md}

Proceeding to Phase {N+1}...
```

</step>

<step name="milestone_audit">
**All phases complete. Run milestone audit.**

```
╔═══════════════════════════════════════════════╗
║  AUTOPILOT: Milestone Audit                   ║
║  v{milestone_version}                         ║
╚═══════════════════════════════════════════════╝
```

```
Agent(
  subagent_type="gsd-integration-checker",
  description="Audit milestone",
  prompt="
    <objective>
    Verify milestone v{milestone_version} achieved its definition of done.
    Check requirements coverage, cross-phase integration, and E2E flows.
    </objective>

    <execution_context>
    @$HOME/.claude/get-shit-done/workflows/audit-milestone.md
    </execution_context>

    <files_to_read>
    - {ROADMAP_PATH}
    - .planning/REQUIREMENTS.md
    - {STATE_PATH}
    - All SUMMARY.md and VERIFICATION.md files in {PHASES_DIR}
    </files_to_read>

    <process>
    Execute the audit-milestone workflow end-to-end.
    Create v{milestone_version}-MILESTONE-AUDIT.md.
    </process>
  "
)
```

**Check audit result:**

```bash
AUDIT_STATUS=$(grep "^status:" {PLANNING_DIR}/v*-MILESTONE-AUDIT.md 2>/dev/null | tail -1 | cut -d: -f2 | tr -d ' ')
```

| Status | Action |
|--------|--------|
| `passed` | Go to completion report |
| `gaps_found` | Enter milestone gap closure |

**Milestone gap closure (max MAX_GAP_ITERATIONS):**

```
for audit_iteration in 1..MAX_GAP_ITERATIONS:

  1. Create gap phases:
     Read MILESTONE-AUDIT.md gaps
     → Use plan-milestone-gaps workflow to create new phases in ROADMAP.md

  2. Execute gap phases:
     Re-enter phase_loop for newly created phases only

  3. Re-audit:
     Run audit-milestone again

  If passed: break
  If still gaps AND iteration == MAX_GAP_ITERATIONS:
    Log: "Milestone gap closure limit reached. Reporting as-is."
    break
```

</step>

<step name="completion_report">
**Present final results:**

```bash
PROGRESS=$(node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs progress bar --raw)
```

```
╔═══════════════════════════════════════════════╗
║  AUTOPILOT COMPLETE                           ║
╚═══════════════════════════════════════════════╝

## Milestone v{milestone_version}

{PROGRESS}

### Phases Executed
| Phase | Name | Plans | Verification | Gaps |
|-------|------|-------|--------------|------|
| {N} | {name} | {X}/{Y} | {status} | {gap_iterations or "none"} |
| ... | ... | ... | ... | ... |

### Milestone Audit
Status: {AUDIT_STATUS}
{If gaps remain: list unresolved gaps}

### Next Steps

`/gsd:complete-milestone v{milestone_version}`

<sub>/clear first for fresh context</sub>
```

</step>

</process>

<error_handling>
**On any step failure:**

1. Log which phase/step failed and the error
2. Present progress report (what completed, what remains)
3. Suggest manual recovery (display to user, do NOT auto-execute):
   - `/gsd:execute-phase {N}` — retry failed phase
   - `/gsdu:autopilot {N}` — restart autopilot from failed phase (user must run manually)
4. Do NOT attempt automatic retry of failed phases
5. Do NOT invoke /gsdu:autopilot from within this workflow — this causes infinite recursion

**classifyHandoffIfNeeded bug:**
If a subagent reports "failed" with `classifyHandoffIfNeeded is not defined`, run spot-checks:
- SUMMARY.md exists for the plan?
- Git commits present?
- No "Self-Check: FAILED" marker?

If spot-checks pass → treat as success. This is a known Claude Code runtime bug.
</error_handling>

<resumption>
Re-running `/gsdu:autopilot` after interruption:
- preflight step rebuilds the execution queue from ROADMAP.md
- Phases with all SUMMARY.md files → skipped (already complete)
- Phases with partial SUMMARY.md → execute-phase handles (skips completed plans)
- Milestone audit → re-runs if MILESTONE-AUDIT.md missing or stale

No separate resume state needed — GSD's file-based state is the resume mechanism.
</resumption>

<context_efficiency>
- Orchestrator: ~15% context (loop control, state checks, routing)
- Each plan/execute: fresh 200k via Agent subagent
- No full file contents in orchestrator — paths only, subagents read independently
- Phase results via gsd-tools CLI (JSON) and file existence checks
</context_efficiency>
