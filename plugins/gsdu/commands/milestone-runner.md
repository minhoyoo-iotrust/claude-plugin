---
description: Sequentially execute GSD milestones from objective documents. Scans for PLANNED objectives, then runs new-milestone → autopilot → complete → PR → merge for each.
argument-hint: "[--dry-run] [--from <filename-prefix>] [--objectives <dir>] [--base <branch>]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, Skill, AskUserQuestion
---

CRITICAL INSTRUCTION: Do NOT use the Skill tool to invoke gsdu:milestone-runner. Do NOT re-invoke this command. All workflow steps are below — execute them directly.

<objective>
Sequentially execute multiple GSD milestones from objective documents.

For each PLANNED objective: new-milestone → autopilot → complete-milestone → PR → CI → merge → pull base branch.

Orchestrator stays lean. Each milestone phase is delegated via existing GSD skills.
</objective>

<arguments>
- `$ARGUMENTS` — optional flags
- `--dry-run` — Show execution plan without running anything
- `--from <prefix>` — Start from objective matching this prefix (e.g., `m01-27`)
- `--objectives <dir>` — Objective documents directory (default: `objectives/`)
- `--base <branch>` — Target branch for PRs and post-merge checkout (default: repo's GitHub default branch)

**Prerequisites:**
- GSD must be installed (`~/.claude/get-shit-done/` exists)
- Objective documents with `status: PLANNED` in frontmatter
- `gh` CLI authenticated for PR/merge operations
</arguments>

<constraints>
- NEVER re-invoke /gsdu:milestone-runner or the milestone-runner skill — causes infinite recursion
- NEVER use the Skill tool to call gsdu:milestone-runner
- Each GSD skill invocation (new-milestone, autopilot, complete-milestone) is called via the Skill tool
- If a step fails, report the error and STOP — do not retry automatically
- Always use `--merge` flag for PR merge (never squash)
</constraints>

<constants>
GSD_TOOLS="node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs"
DEFAULT_OBJECTIVES_DIR="objectives/"
</constants>

<step name="resolve_base_branch" priority="first">
**0. Resolve base branch:**

If `--base <branch>` argument is provided, use that value. Otherwise, auto-detect the repo's default branch:

```bash
BASE_BRANCH=$(gh repo view --json defaultBranch -q '.defaultBranch')
echo "BASE_BRANCH=${BASE_BRANCH}"
```

Store `BASE_BRANCH` for use throughout the workflow. All subsequent references to the target branch use `${BASE_BRANCH}`.
</step>

<process>

<step name="preflight" priority="first">
**1. Verify prerequisites:**

```bash
# Check GSD installation
test -f "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" || echo "GSD_NOT_FOUND"

# Check gh CLI
gh auth status 2>&1 | head -3
```

If `GSD_NOT_FOUND`:
```
GSD is not installed. Install it first:
  https://github.com/gsd-build/get-shit-done
```
Exit.

**2. Determine objectives directory:**

Use `--objectives <dir>` argument if provided, otherwise `objectives/`.

```bash
ls ${OBJECTIVES_DIR}/*.md 2>/dev/null
```

If no files found, exit with error.

**3. Scan for PLANNED objectives:**

Read each `.md` file in the objectives directory. Parse YAML frontmatter for `status: PLANNED`.

Build execution queue:
- Filter: only `status: PLANNED` documents
- Sort: by filename ascending (natural order)
- If `--from <prefix>` given: skip files before the matching prefix

For each objective, extract:
- `filename` — e.g., `m01-26-brave-search-integration.md`
- `title` — first `# ` heading in the document
- `version` — extract from frontmatter `version` field if present; otherwise derive from filename pattern:
  - Pattern `m{major}-{minor}-*` → `v{major}.{minor}` (e.g., `m01-26-*` → `v1.26`)
  - Pattern `v{version}-*` → use as-is
  - If no pattern matches: report error and ask user
- `slug` — derive from filename (strip prefix and extension, kebab-case)
- `content` — full file content (for milestone context)

If queue is empty:
```
No PLANNED objectives found in ${OBJECTIVES_DIR}/.
All objectives may already be SHIPPED or IN_PROGRESS.
```
Exit.
</step>

<step name="show_plan">
**4. Display execution plan:**

```
## Milestone Runner — Execution Plan

| # | Filename | Version | Title |
|---|----------|---------|-------|
| 1 | m01-26-brave-search-integration.md | v1.26 | 시장 데이터 파운데이션 |
| 2 | m01-27-market-analysis-engine.md | v1.27 | 시장 분석 엔진 |
| ... | ... | ... | ... |

**Steps per milestone:**
1. Write MILESTONE-CONTEXT.md from objective content
2. /gsd:new-milestone (version + context)
3. /gsdu:autopilot
4. /gsd:complete-milestone
5. Create PR → Wait CI → Merge (--merge) → Checkout ${BASE_BRANCH} → Pull
```

If `--dry-run` flag present: show plan and exit.

**Confirm with user before proceeding** (use AskUserQuestion):
```
위 {N}개 마일스톤을 순차 실행합니다. 진행할까요?
```
</step>

<step name="milestone_loop">
**5. Execute each milestone sequentially.**

For each objective in the execution queue:

```
╔══════════════════════════════════════════════════════╗
║  MILESTONE RUNNER: {index}/{total}                   ║
║  {version} — {title}                                 ║
║  Source: {filename}                                   ║
╚══════════════════════════════════════════════════════╝
```

---

**5a. Prepare milestone context:**

Write the objective document content to `.planning/MILESTONE-CONTEXT.md`:

```bash
cp ${OBJECTIVES_DIR}/{filename} .planning/MILESTONE-CONTEXT.md
```

This file is read by `/gsd:new-milestone` to understand the milestone goals.

---

**5b. Run new-milestone:**

```
Skill("gsd:new-milestone", args="{version} --reset-phase-numbers")
```

System prompt guidance for the skill execution:
- Version is `{version}` extracted from the objective
- Read MILESTONE-CONTEXT.md for goals (already written in 5a)
- Accept all defaults — select maximum scope coverage
- Do NOT ask questions — proceed autonomously with recommended options
- If research is enabled in config, run research phase

After completion, verify:
```bash
# Verify roadmap was created
test -f .planning/ROADMAP.md && echo "ROADMAP_OK" || echo "ROADMAP_MISSING"
# Verify state was updated
grep -q "milestone:" .planning/STATE.md && echo "STATE_OK" || echo "STATE_MISSING"
```

If verification fails, stop and report error.

---

**5c. Run autopilot:**

```
Skill("gsdu:autopilot")
```

This executes all phases: plan → execute → verify → gap closure → milestone audit.

After completion, verify:
```bash
# Check progress
node $HOME/.claude/get-shit-done/bin/gsd-tools.cjs progress bar --raw 2>/dev/null
```

---

**5d. Run complete-milestone:**

```
Skill("gsd:complete-milestone")
```

Guidance for the skill execution:
- Proceed with completing the milestone
- For branch handling: keep branches (PR will handle merge)
- Accept all defaults autonomously
- Create git tag

After completion, verify:
```bash
# Check state shows completed
grep -q "completed" .planning/STATE.md && echo "MILESTONE_COMPLETE" || echo "INCOMPLETE"
```

---

**5e. Create PR, wait for CI, merge:**

```bash
# Get current branch name
BRANCH=$(git branch --show-current)

# Push branch
git push -u origin ${BRANCH}
```

Create PR:
```bash
gh pr create \
  --base "${BASE_BRANCH}" \
  --title "${version}: ${title}" \
  --body "## Summary
- Milestone ${version} completed via milestone-runner
- Source objective: ${filename}

## Changes
$(git log ${BASE_BRANCH}..HEAD --oneline | head -20)

---
Generated by gsdu:milestone-runner"
```

Wait for CI:
```bash
# Get PR number
PR_NUMBER=$(gh pr view --json number -q '.number')

# Wait for CI checks to complete
gh pr checks ${PR_NUMBER} --watch --fail-fast 2>&1
```

If CI fails:
```
CI failed for ${version}. PR #${PR_NUMBER} is open on branch ${BRANCH}.
Fix the issues manually and re-run:
  /gsdu:milestone-runner --from {next_objective_prefix}
```
Stop execution.

If CI passes, merge:
```bash
# Merge with merge commit (never squash)
gh pr merge ${PR_NUMBER} --merge --delete-branch

# Switch to base branch and pull
git checkout ${BASE_BRANCH}
git pull origin ${BASE_BRANCH}
```

---

**5f. Report milestone completion:**

```
## Milestone {index}/{total} Complete

- Version: {version}
- Title: {title}
- PR: #{PR_NUMBER} (merged)
- Branch: ${BASE_BRANCH} (up to date)

{'Proceeding to next milestone...' if more remain, else 'All milestones complete!'}
```

Continue to next objective in queue.
</step>

<step name="completion_report">
**6. Final report after all milestones:**

```
╔══════════════════════════════════════════════════════╗
║  MILESTONE RUNNER — COMPLETE                         ║
╚══════════════════════════════════════════════════════╝

## Results

| # | Version | Title | PR | Status |
|---|---------|-------|----|--------|
| 1 | v1.26 | 시장 데이터 파운데이션 | #42 | Merged |
| 2 | v1.27 | 시장 분석 엔진 | #43 | Merged |
| ... | ... | ... | ... | ... |

**Total:** {N} milestones completed
**Current branch:** ${BASE_BRANCH} (up to date)
```
</step>

</process>

<error_handling>
**On any step failure:**

1. Log which milestone/step failed and the error
2. Present progress report (what completed, what remains)
3. Suggest manual recovery:
   - If failed during GSD skill: fix the issue, then `/gsdu:milestone-runner --from {failed_prefix}`
   - If failed during PR/CI: fix CI, merge manually, then `/gsdu:milestone-runner --from {next_prefix}`
4. Do NOT attempt automatic retry
5. Do NOT invoke /gsdu:milestone-runner — this causes infinite recursion

**Branch state recovery:**
If interrupted mid-milestone, the current branch may not be main. Before restarting:
```bash
git branch --show-current  # Check current branch
git stash                  # Stash any uncommitted changes if needed
```
</error_handling>

<resumption>
Re-running `/gsdu:milestone-runner` after interruption:
- Uses `--from <prefix>` to skip completed milestones
- Scans objective status: SHIPPED objectives are skipped automatically
- If a milestone is partially complete (branch exists, some phases done):
  - The GSD skills handle partial state gracefully
  - autopilot picks up from incomplete phases
  - complete-milestone checks readiness
</resumption>

<context_efficiency>
- Orchestrator: tracks only the queue and current position
- Each GSD skill invocation gets full context via Skill tool
- PR/CI/merge operations are lightweight Bash commands
- No large file contents stored in orchestrator — paths only
</context_efficiency>
