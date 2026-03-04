---
name: plan-to-beads
description: This skill should be used when converting a plan document into tracked Beads issues. It carefully reviews a plan file, identifies all discrete work items, creates a Beads epic with child issues, sets up dependency graphs, and assigns priorities and labels. Triggers on "create issues from plan", "convert plan to beads", "break down plan into issues", or after completing a /workflows:plan.
---

# Plan to Beads

Convert a plan document into a fully tracked set of Beads issues with an epic, child tasks, dependency graph, and proper prioritization.

## When to Use This Skill

This skill is valuable when:
- A plan has been created via `/workflows:plan` or `/deepen-plan` and is ready for execution
- Breaking down a large plan into trackable, assignable work items
- Setting up a dependency-aware execution order for a multi-phase project
- Converting acceptance criteria into verifiable task items

This skill can be skipped when:
- The plan is trivial (single task, no phases)
- Work will be done immediately in one session without tracking
- Issues already exist for this plan

**After conversion, beads is the single source of truth.** The plan markdown becomes a read-only reference document for design context, rationale, and code examples. All work tracking (status, progress, assignments, dependencies) lives in beads. Do NOT track progress by checking off plan markdown checkboxes — use `bd update`, `bd close`, and `bd epic status` instead.

## Prerequisites

Beads must be initialized in the project:

```bash
bd status    # Check if initialized
bd init      # Initialize if needed
```

## Core Process

### Phase 1: Locate and Parse the Plan

**Step 1: Find the plan file**

```bash
# Check for recent plans
ls -la docs/plans/*.md 2>/dev/null | tail -10
```

If no plan path is provided, ask: "Which plan should I convert to Beads issues? Provide the path (e.g., `docs/plans/2026-01-15-feat-my-feature-plan.md`)."

Do not proceed without a valid plan file.

**Step 2: Read and analyze the plan**

Read the plan file completely. Extract and catalog:

| Element | What to Look For |
|---------|-----------------|
| **Title** | From YAML frontmatter `title:` or first `# heading` |
| **Type** | From frontmatter `type:` (feat, fix, refactor) |
| **Phases** | `#### Phase N:` sections with tasks and deliverables |
| **Acceptance Criteria** | `- [ ]` checkbox items under Acceptance Criteria |
| **Red Tests** | Items under "Red Tests (Write First)" — these become beads issues for writing failing tests |
| **Review Checkpoints** | Items under "Review Checkpoints" — these become beads `chore` issues that gate phase transitions |
| **Dependencies** | Explicit mentions of ordering ("after X", "requires Y", "blocked by Z") |
| **Technical Risks** | Items in Risk Analysis or Dependencies sections |
| **Non-functional Requirements** | Performance targets, security requirements, accessibility |

**Step 3: Build a work item manifest**

Create a structured manifest of all identified work items:

```
Epic: [Plan Title]
  Phase 1: [Phase Name]
    Task 1.1: [Description] — type: [task|feature|chore] priority: [0-4]
    Task 1.2: [Description] — type: [task|feature|chore] priority: [0-4]
  Phase 2: [Phase Name]
    Task 2.1: [Description] — depends on: [1.1, 1.2]
    ...
  Standalone:
    Task S1: [Non-phase item from acceptance criteria]
```

### Phase 2: Review and Validate

Before creating issues, review the manifest for quality.

**Validation checks:**

- [ ] Every task is discrete and actionable (not vague like "make it work")
- [ ] No duplicate or overlapping tasks
- [ ] Dependencies form a valid DAG (no cycles)
- [ ] Priority assignments reflect plan intent (critical-path items get higher priority)
- [ ] Labels are consistent and useful for filtering
- [ ] Task granularity is appropriate (not too coarse, not too fine)

**Granularity guidelines:**

| Too Coarse | Right Size | Too Fine |
|-----------|------------|----------|
| "Build the backend" | "Create user registration endpoint with validation" | "Add email field to user model" |
| "Add tests" | "Write integration tests for payment flow" | "Test that amount > 0" |
| "Set up infrastructure" | "Configure CI pipeline for automated testing" | "Add pytest to requirements.txt" |

A task should represent a coherent unit of work that can be completed, reviewed, and verified independently. Aim for tasks that are independently deliverable.

**Present the manifest to the user for approval:**

Use AskUserQuestion to present the manifest summary and ask:
"I've identified [N] work items from the plan. Here's the breakdown: [manifest summary]. Should I proceed with creating these as Beads issues?"

Options:
1. **Create all issues** — Proceed with the full manifest
2. **Adjust granularity** — Make tasks more or less granular
3. **Remove items** — Specify tasks to skip
4. **Add items** — Include additional work items not in the plan

### Phase 3: Write and Execute a Batch Creation Script

**CRITICAL: Performance requirement.** `bd` auto-commits to Dolt after every write by default, which is slow when creating many issues. Creating issues and deps individually is unacceptably slow for plans with 10+ items. ALWAYS write a single bash script that creates everything with `--dolt-auto-commit batch`, then commits once at the end.

**NEVER** run `bd create` or `bd dep add` as individual Bash tool calls in a loop. **NEVER** run multiple `bd` write commands in parallel — this corrupts the SQLite WAL.

Write a bash script to `/tmp/plan_to_beads_$$.sh` that does ALL of the following in one execution:

```bash
#!/bin/bash
set -e

# ============================================================
# Phase 3a: Create the Epic
# ============================================================
EPIC_ID=$(bd create "[Plan Title]" \
  -t epic -p 1 \
  -d "Plan: [plan file path]. [1-2 sentence summary]" \
  -l plan,[type-label] \
  --dolt-auto-commit batch --silent)
echo "Epic: $EPIC_ID"

# ============================================================
# Phase 3b: Create All Child Issues
# ============================================================
# Capture each ID for dependency setup later
ID_1=$(bd create "[Task 1 description]" \
  -t [type] -p [0-4] \
  -d "[Detailed description]" \
  -l [labels] \
  --parent "$EPIC_ID" \
  --dolt-auto-commit batch --silent)
echo "Task 1: $ID_1"

ID_2=$(bd create "[Task 2 description]" \
  -t [type] -p [0-4] \
  -d "[Detailed description]" \
  -l [labels] \
  --parent "$EPIC_ID" \
  --dolt-auto-commit batch --silent)
echo "Task 2: $ID_2"

# ... repeat for all tasks ...

# ============================================================
# Phase 3c: Set Up All Dependencies
# ============================================================
# Task 2 depends on Task 1
bd dep add "$ID_2" "$ID_1" --dolt-auto-commit batch --quiet
# ... repeat for all dependencies ...

# ============================================================
# Phase 3d: Single commit + verify
# ============================================================
echo "Committing..."
bd dolt commit -m "Batch create issues from plan"
echo "Checking for cycles..."
bd dep cycles --quiet
echo "Done. All issues and deps created."
```

Then execute it in ONE Bash tool call:

```bash
chmod +x /tmp/plan_to_beads_$$.sh && bash /tmp/plan_to_beads_$$.sh
```

**This approach creates 30+ issues with full dependency graph in under 60 seconds** instead of 20+ minutes with individual calls.

**Type label mapping:**
- `feat` plan → label `feature`
- `fix` plan → label `bugfix`
- `refactor` plan → label `refactor`

**Priority assignment rules:**

| Priority | When to Assign |
|----------|---------------|
| 0 (critical) | Blocks everything else, foundational setup |
| 1 (high) | Core functionality, on the critical path |
| 2 (normal) | Standard implementation tasks |
| 3 (low) | Polish, optimization, nice-to-have |
| 4 (backlog) | Future considerations, stretch goals |

**Label conventions:**

- Phase labels: `phase-1`, `phase-2`, `phase-3`
- Domain labels: `backend`, `frontend`, `database`, `api`, `testing`, `docs`, `infra`
- Concern labels: `security`, `performance`, `accessibility`

**Issue type mapping:**

| Plan Element | Beads Type |
|-------------|-----------|
| New endpoint/component/feature | `feature` |
| Database migration/schema change | `task` |
| Red test writing (from "Red Tests" section) | `chore` |
| Test writing | `chore` |
| Configuration/setup | `chore` |
| Bug fix identified in plan | `bug` |
| Documentation | `chore` |
| Refactoring | `task` |
| Review checkpoint (from "Review Checkpoints" section) | `chore` |

**Dependency sources:**

1. **Explicit plan ordering** — Phase 1 tasks block Phase 2 tasks
2. **Implicit data dependencies** — "Create model" blocks "Create endpoint using model"
3. **Technical prerequisites** — "Set up database" blocks "Write migrations"
4. **Testing dependencies** — Implementation tasks block their test tasks

**Dependency rules:**
- Only add dependencies where there is a genuine ordering constraint
- Do not create dependencies between independent tasks in the same phase
- Cross-phase dependencies should flow forward (Phase 1 → Phase 2 → Phase 3)
- Within a phase, minimize dependencies to allow parallel work

### Phase 4: Verify and Report

After all issues and dependencies are created, verify the result.

```bash
# Show full epic with children
bd children [epic-id]

# Show dependency tree
bd dep tree [epic-id]

# Show what's ready to start
bd ready

# Check for cycles
bd dep cycles

# Overall status
bd status
```

**Present a summary report:**

```
## Beads Issues Created

**Epic:** [epic-id] — [Plan Title]
**Total Issues:** [count]
**Ready to Start:** [count] (no blockers)

### By Phase
- Phase 1: [count] issues ([list ids])
- Phase 2: [count] issues ([list ids]) — blocked by Phase 1
- Phase 3: [count] issues ([list ids]) — blocked by Phase 2

### By Priority
- Critical (P0): [count]
- High (P1): [count]
- Normal (P2): [count]
- Low (P3): [count]

### Dependency Graph
[Output from bd dep tree]

### Next Steps
Run `bd ready --sort priority` to see unblocked work items.
```

## Post-Creation Options

After creating all issues, announce: "Beads is now the source of truth for this epic. The plan at `[path]` is a reference document for design context — all progress tracking happens in beads."

Present options using AskUserQuestion:

**Question:** "Created [N] Beads issues from plan. What would you like to do next?"

**Options:**
1. **Start working** — Run `bd ready --sort priority` and begin the first task
2. **View dependency tree** — Show the full dependency graph
3. **Adjust priorities** — Modify priority assignments
4. **Start `/workflows:work`** — Begin implementing using the work workflow with the plan

## Edge Cases

**Plan with no phases:**
- Create the epic and flat list of tasks (no phase labels or cross-phase dependencies)
- Derive ordering from the sequence of acceptance criteria

**Plan with "A LOT" detail level:**
- Implementation phases map directly to Beads phases
- Quality gates become separate chore issues
- Non-functional requirements become labeled tasks

**Plan with "MINIMAL" detail level:**
- Each acceptance criterion becomes one task
- Infer dependencies from context
- Ask user to clarify ordering if ambiguous

**Existing issues for this plan:**
- Search for existing issues: `bd search "[plan title]"`
- If found, ask: "Found existing issues matching this plan. Should I skip duplicates, update existing, or create new ones?"

## Integration with Other Workflows

| Previous Step | This Skill | Next Step |
|--------------|-----------|-----------|
| `/workflows:plan` | Convert plan → Beads issues | `/workflows:work` |
| `/deepen-plan` | Convert enhanced plan → Beads issues | `bd ready` |
| `/plan_review` | Address feedback, then convert → Beads issues | `/workflows:work` |
