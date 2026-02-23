---
name: workflows:work
description: Execute work plans efficiently while maintaining quality and finishing features
argument-hint: "[plan file, specification, or todo file path]"
---

# Work Plan Execution Command

Execute a work plan efficiently while maintaining quality and finishing features.

## Introduction

This command takes a work document (plan, specification, or beads epic) and executes it systematically. The focus is on **shipping complete features** by understanding requirements quickly, following existing patterns, and maintaining quality throughout.

**Important: Beads is the single source of truth for work tracking.** Plan markdown files are reference documents that describe *what* to build and *why*. Beads issues (via `bd` CLI) track *what work is being done, by whom, and what's left*. Never use plan markdown checkboxes to track execution progress — use `bd update`, `bd close`, and `bd epic status` instead.

## Input Document

<input_document> #$ARGUMENTS </input_document>

## Execution Workflow

### Phase 1: Quick Start

1. **Read Plan and Clarify**

   - Read the work document completely
   - Review any references or links provided in the plan
   - If anything is unclear or ambiguous, ask clarifying questions now
   - Get user approval to proceed
   - **Do not skip this** - better to ask questions now than build the wrong thing

2. **Setup Environment**

   First, check the current branch:

   ```bash
   current_branch=$(git branch --show-current)
   default_branch=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')

   # Fallback if remote HEAD isn't set
   if [ -z "$default_branch" ]; then
     default_branch=$(git rev-parse --verify origin/main >/dev/null 2>&1 && echo "main" || echo "master")
   fi
   ```

   **If already on a feature branch** (not the default branch):
   - Ask: "Continue working on `[current_branch]`, or create a new branch?"
   - If continuing, proceed to step 3
   - If creating new, follow Option A or B below

   **If on the default branch**, choose how to proceed:

   **Option A: Create a new branch**
   ```bash
   git pull origin [default_branch]
   git checkout -b feature-branch-name
   ```
   Use a meaningful name based on the work (e.g., `feat/user-authentication`, `fix/email-validation`).

   **Option B: Use a worktree (recommended for parallel development)**
   ```bash
   skill: git-worktree
   # The skill will create a new branch from the default branch in an isolated worktree
   ```

   **Option C: Continue on the default branch**
   - Requires explicit user confirmation
   - Only proceed after user explicitly says "yes, commit to [default_branch]"
   - Never commit directly to the default branch without explicit permission

   **Recommendation**: Use worktree if:
   - You want to work on multiple features simultaneously
   - You want to keep the default branch clean while experimenting
   - You plan to switch between branches frequently

3. **Create or Reuse Issue Tracker**

   First, check if Beads issues already exist for this plan (e.g., from a prior `plan-to-beads` run):

   ```bash
   # Search for an existing epic matching the plan title
   bd search "<plan title>" --json
   ```

   **If matching epic and child issues exist:**
   - Announce: "Found existing Beads epic `<id>` with `<N>` child issues. Using those."
   - Run `bd ready --sort priority` to see what's ready to start
   - Skip to Phase 2

   **If no existing issues found:**
   - Invoke the `plan-to-beads` skill with the plan file path to create a full epic with child issues, dependency graph, and priority assignments
   - If `bd` is not initialized, run `bd init` first
   - After the skill completes, run `bd ready --sort priority` to confirm issues are ready

### Phase 2: Execute

1. **Write Red Tests First**

   Before implementing any task, check the plan for a "Red Tests" section. If it exists, write all failing tests for the current phase FIRST:

   ```
   # Phase start: Write ALL red tests for this phase
   - Read the plan's "Red Tests" section for the current phase
   - Write each test (unit, integration, e2e) as specified
   - Run the test suite — confirm all new tests FAIL (RED)
   - Commit red tests: git commit -m "test(scope): add failing tests for [phase/feature]"
   ```

   If the plan has no "Red Tests" section, write tests for each task as you go (test-first per task).

   **Why red tests first?** Writing tests before code ensures:
   - Requirements are unambiguous and testable
   - You know exactly when you're done (all green)
   - No untested code slips through

2. **Task Execution Loop (Red → Green)**

   For each task in priority order:

   ```
   while (bd ready --sort priority shows tasks):
     - Claim task: bd update <id> --claim
     - Read the plan for context on this task (design rationale, references, code examples)
     - Look for similar patterns in codebase
     - If no red test exists for this task, write a failing test first
     - Implement minimum code to make the test pass (GREEN)
     - Run tests — confirm the target test passes
     - Run full test suite — confirm no regressions
     - Close task: bd close <id> --reason "Completed"
     - Evaluate for incremental commit (see below)
   ```

   **IMPORTANT**: Use `bd ready --sort priority` to find the next task, NOT the plan's checkboxes. The plan is a reference document for design context — beads is the authority on what work remains. Check progress with `bd epic status`.

2. **Incremental Commits**

   After completing each task, evaluate whether to create an incremental commit:

   | Commit when... | Don't commit when... |
   |----------------|---------------------|
   | Logical unit complete (model, service, component) | Small part of a larger unit |
   | Tests pass + meaningful progress | Tests failing |
   | About to switch contexts (backend → frontend) | Purely scaffolding with no behavior |
   | About to attempt risky/uncertain changes | Would need a "WIP" commit message |

   **Heuristic:** "Can I write a commit message that describes a complete, valuable change? If yes, commit. If the message would be 'WIP' or 'partial X', wait."

   **Commit workflow:**
   ```bash
   # 1. Verify tests pass (use project's test command)
   # Examples: bun test, npm test, pytest, go test, etc.

   # 2. Stage only files related to this logical unit (not `git add .`)
   git add <files related to this logical unit>

   # 3. Commit with conventional message
   git commit -m "feat(scope): description of this unit"
   ```

   **Handling merge conflicts:** If conflicts arise during rebasing or merging, resolve them immediately. Incremental commits make conflict resolution easier since each commit is small and focused.

   **Note:** Incremental commits use clean conventional messages without attribution footers. The final Phase 4 commit/PR includes the full attribution.

3. **Review Checkpoints (After Each Phase/Chunk)**

   The plan may define explicit review checkpoints. If it does, follow them. Otherwise, trigger a review after completing each phase or logical chunk of work (e.g., 3-5 tasks).

   **When to run a checkpoint:**
   - After completing all tasks in a phase (Phase 1 → review → Phase 2)
   - After the plan's "Review Checkpoints" say to
   - Before switching from one domain to another (backend → frontend)
   - When you've accumulated significant uncommitted complexity

   **Checkpoint process:**

   ```
   # 1. Verify current phase tests are GREEN
   # Run project test suite

   # 2. Run /workflows:review on accumulated changes
   #    This creates beads issues automatically for all findings

   # 3. Check review findings
   bd list --label code-review --status open

   # 4. Address P1 (critical) findings immediately before proceeding
   bd ready --label code-review --sort priority
   # Fix P1s, close their beads issues

   # 5. Compound non-trivial fixes
   #    For each P1/P2 finding that required a non-trivial fix:
   #    Run /workflows:compound to document the problem and solution
   #    This captures the learning while context is fresh

   # 6. Log P2/P3 findings for later — they don't block the next phase
   # These beads issues persist and can be addressed in Phase 3 or before PR

   # 7. Close the review checkpoint beads issue if one exists
   #    bd close <checkpoint-id> --reason "Review complete, P1s resolved"
   ```

   **Key rule:** P1 findings from a review checkpoint BLOCK the next phase. P2/P3 findings are tracked as beads issues but don't block progress.

   **Compounding rule:** After resolving any non-trivial review finding (required research, involved a subtle bug, or taught something new), run `/workflows:compound` to document the solution in `docs/solutions/`. A fix is "non-trivial" if:
   - It took more than 15 minutes to investigate or resolve
   - The root cause was surprising or non-obvious
   - The same mistake could easily happen again
   - It involved a pattern or gotcha worth remembering

4. **Follow Existing Patterns**

   - The plan should reference similar code - read those files first
   - Match naming conventions exactly
   - Reuse existing components where possible
   - Follow project coding standards (see CLAUDE.md)
   - When in doubt, grep for similar implementations

5. **Test Continuously**

   - Run relevant tests after each significant change
   - Don't wait until the end to test
   - Fix failures immediately — a RED test that was GREEN is a regression
   - New tests should be written BEFORE the code they test (red/green cycle)

6. **Figma Design Sync** (if applicable)

   For UI work with Figma designs:

   - Implement components following design specs
   - Use figma-design-sync agent iteratively to compare
   - Fix visual differences identified
   - Repeat until implementation matches design

7. **Track Progress (Beads is the Source of Truth)**
   - **All progress tracking goes through beads.** Never rely on plan markdown checkboxes for status.
   - Update beads as you complete tasks:
     ```bash
     bd close <id> --reason "Done"          # Mark complete
     bd comments add <id> "Progress note"   # Add context
     bd create "New task" --parent <epic>    # If scope expands
     bd ready --sort priority               # What's next?
     bd epic status                         # Overall progress
     ```
   - Note any blockers or unexpected discoveries as beads comments
   - Keep user informed of major milestones (reference `bd epic status` output)

### Phase 3: Quality Check

1. **Run Core Quality Checks**

   Always run before submitting:

   ```bash
   # Run full test suite (use project's test command)
   # Examples: bun test, npm test, pytest, go test, etc.

   # Run linting (per CLAUDE.md)
   # Use project linter before pushing to origin
   ```

2. **Consider Reviewer Agents** (Optional)

   Use for complex, risky, or large changes:

   - **code-simplicity-reviewer**: Check for unnecessary complexity
   - **sglyon-python-reviewer**: Verify Python conventions (Python projects)
   - **sglyon-typescript-reviewer**: Verify TypeScript conventions (TypeScript projects)
   - **performance-oracle**: Check for performance issues
   - **security-sentinel**: Scan for security vulnerabilities

   Run reviewers in parallel with Task tool:

   ```
   Task(code-simplicity-reviewer): "Review changes for simplicity"
   Task(sglyon-typescript-reviewer): "Check TypeScript conventions"
   ```

   Present findings to user and address critical issues.

3. **Final Validation**
   - All red tests are GREEN (every test from the plan's "Red Tests" section passes)
   - All beads issues closed (`bd epic status` shows 100%)
   - All review checkpoint beads resolved (no open P1/P2 issues with `code-review` label)
   - All tests pass (full suite, not just new tests)
   - Linting passes
   - Code follows existing patterns
   - Figma designs match (if applicable)
   - No console errors or warnings

### Phase 4: Ship It

1. **Create Commit**

   ```bash
   git add .
   git status  # Review what's being committed
   git diff --staged  # Check the changes

   # Commit with conventional format
   git commit -m "$(cat <<'EOF'
   feat(scope): description of what and why

   Brief explanation if needed.

   🤖 Generated with [Claude Code](https://claude.com/claude-code)

   Co-Authored-By: Claude <noreply@anthropic.com>
   EOF
   )"
   ```

2. **Capture and Upload Screenshots for UI Changes** (REQUIRED for any UI work)

   For **any** design changes, new views, or UI modifications, you MUST capture and upload screenshots:

   **Step 1: Start dev server** (if not running)
   ```bash
   npm run dev  # or bun dev, python manage.py runserver, etc.
   ```

   **Step 2: Capture screenshots with agent-browser CLI**
   ```bash
   agent-browser open http://localhost:3000/[route]
   agent-browser snapshot -i
   agent-browser screenshot output.png
   ```
   See the `agent-browser` skill for detailed usage.

   **Step 3: Upload using imgup skill**
   ```bash
   skill: imgup
   # Then upload each screenshot:
   imgup -h pixhost screenshot.png  # pixhost works without API key
   # Alternative hosts: catbox, imagebin, beeimg
   ```

   **What to capture:**
   - **New screens**: Screenshot of the new UI
   - **Modified screens**: Before AND after screenshots
   - **Design implementation**: Screenshot showing Figma design match

   **IMPORTANT**: Always include uploaded image URLs in PR description. This provides visual context for reviewers and documents the change.

3. **Create Pull Request**

   ```bash
   git push -u origin feature-branch-name

   gh pr create --title "Feature: [Description]" --body "$(cat <<'EOF'
   ## Summary
   - What was built
   - Why it was needed
   - Key decisions made

   ## Testing
   - Tests added/modified
   - Manual testing performed

   ## Before / After Screenshots
   | Before | After |
   |--------|-------|
   | ![before](URL) | ![after](URL) |

   ## Figma Design
   [Link if applicable]

   ---

   [![Compound Engineered](https://img.shields.io/badge/Compound-Engineered-6366f1)](https://github.com/sglyon/compound-engineering-plugin) 🤖 Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   )"
   ```

4. **Notify User**
   - Summarize what was completed
   - Link to PR
   - Note any follow-up work needed
   - Suggest next steps if applicable

---

## Key Principles

### Start Fast, Execute Faster

- Get clarification once at the start, then execute
- Don't wait for perfect understanding - ask questions and move
- The goal is to **finish the feature**, not create perfect process

### The Plan is Your Reference, Beads is Your Tracker

- The plan describes *what* to build and *why* — read it for design context, code references, and rationale
- Beads tracks *what work remains* — use `bd ready` to find next tasks, `bd epic status` for progress
- Never track progress by checking off plan markdown checkboxes — that's beads' job
- Don't reinvent - match existing patterns referenced in the plan

### Red/Green TDD

- Write failing tests BEFORE writing implementation code
- Each task starts with a RED test and ends with a GREEN test
- You know you're done when all red tests are green
- If the plan has a "Red Tests" section, write them all at phase start

### Test As You Go

- Run tests after each change, not at the end
- Fix failures immediately
- Continuous testing prevents big surprises

### Review After Each Chunk, Then Compound

- Run `/workflows:review` after completing each phase or chunk of work
- Review findings become beads issues automatically
- Fix P1 findings before moving to the next phase
- P2/P3 findings are tracked but don't block progress
- Run `/workflows:compound` for non-trivial fixes — capture the learning while context is fresh

### Quality is Built In

- Follow existing patterns
- Write tests BEFORE new code (red/green)
- Run linting before pushing
- Use reviewer agents for complex/risky changes only

### Ship Complete Features

- `bd epic status` should show 100% before moving on
- Don't leave features 80% done — close every beads issue or explicitly defer it
- A finished feature that ships beats a perfect feature that doesn't

## Quality Checklist

Before creating PR, verify:

- [ ] All clarifying questions asked and answered
- [ ] All red tests are GREEN (plan's "Red Tests" section fully satisfied)
- [ ] All review checkpoint beads resolved (no open P1/P2 `code-review` issues)
- [ ] All beads issues closed (`bd epic status` shows 100%)
- [ ] Tests pass (run project's test command)
- [ ] Linting passes (use project linter)
- [ ] Code follows existing patterns
- [ ] Figma designs match implementation (if applicable)
- [ ] Before/after screenshots captured and uploaded (for UI changes)
- [ ] Commit messages follow conventional format
- [ ] PR description includes summary, testing notes, and screenshots
- [ ] PR description includes Compound Engineered badge

## When to Use Reviewer Agents

**Don't use by default.** Use reviewer agents only when:

- Large refactor affecting many files (10+)
- Security-sensitive changes (authentication, permissions, data access)
- Performance-critical code paths
- Complex algorithms or business logic
- User explicitly requests thorough review

For most features: tests + linting + following patterns is sufficient.

## Common Pitfalls to Avoid

- **Analysis paralysis** - Don't overthink, read the plan and execute
- **Skipping clarifying questions** - Ask now, not after building wrong thing
- **Ignoring plan references** - The plan has links for a reason
- **Writing tests after code** - Write RED tests first, then make them GREEN
- **Skipping review checkpoints** - Run `/workflows:review` after each phase to catch issues early
- **Ignoring review findings** - P1 beads from reviews block the next phase for a reason
- **Not compounding after fixes** - Non-trivial review fixes are learnings; run `/workflows:compound` or lose the insight
- **Testing at the end** - Test continuously or suffer later
- **Tracking progress in markdown** - Plan checkboxes are NOT work trackers; use `bd` exclusively for status
- **Forgetting Beads** - `bd ready` and `bd epic status` are the only way to know what's left
- **80% done syndrome** - Finish the feature, don't move on early
- **Over-reviewing simple changes** - Save reviewer agents for complex work
