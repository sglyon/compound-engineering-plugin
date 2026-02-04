---
name: triage
description: Triage and categorize findings using Beads issue tracker
argument-hint: "[findings list or source type]"
---

- First set the /model to Haiku
- Then list all open issues: `bd list --status open`

Present all findings, decisions, or issues here one by one for triage. The goal is to go through each item and decide whether to approve it for work or reject it.

**IMPORTANT: DO NOT CODE ANYTHING DURING TRIAGE!**

This command is for:

- Triaging code review findings
- Processing security audit results
- Reviewing performance analysis
- Handling any other categorized findings that need tracking

## Workflow

### Step 1: Present Each Finding

For each finding, present in this format:

```
---
Issue #X: [Brief Title]

Severity: P1 (CRITICAL) / P2 (IMPORTANT) / P3 (NICE-TO-HAVE)

Category: [Security/Performance/Architecture/Bug/Feature/etc.]

Description:
[Detailed explanation of the issue or improvement]

Location: [file_path:line_number]

Problem Scenario:
[Step by step what's wrong or could happen]

Proposed Solution:
[How to fix it]

Estimated Effort: [Small (< 2 hours) / Medium (2-8 hours) / Large (> 8 hours)]

---
Do you want to approve this issue?
1. yes - approve for work
2. next - skip this item
3. custom - modify before approving
```

### Step 2: Handle User Decision

**When user says "yes":**

1. **If the issue already exists as a bead**, update it:

   ```bash
   bd label add <id> approved
   bd update <id> --append-notes "Approved during triage session"
   ```

2. **If creating a new issue**, create it with `bd`:

   Priority mapping:
   - P1 (CRITICAL) -> `-p 0`
   - P2 (IMPORTANT) -> `-p 1`
   - P3 (NICE-TO-HAVE) -> `-p 2`

   ```bash
   bd create "Issue Title" \
     -t task \
     -p 0 \
     -d "Problem statement and description" \
     -l code-review,security,approved
   ```

3. **Add detailed context as a comment:**

   ```bash
   bd comments add <id> "Findings: [key discoveries]. Proposed solution: [approach]. Effort: [estimate]"
   ```

4. **Confirm approval:** "Approved: `<id>` - [Title] (Priority: P1) - Ready to work on"

**When user says "next":**

- If the issue exists as a bead, close it:
  ```bash
  bd close <id> --reason "Rejected during triage - not relevant"
  ```
- Skip to the next item
- Track skipped items for summary

**When user says "custom":**

- Ask what to modify (priority, description, details)
- Update the information
- Present revised version
- Ask again: yes/next/custom

### Step 3: Continue Until All Processed

- Process all items one by one
- Don't wait for approval between items - keep moving

### Step 4: Final Summary

After all items processed:

````markdown
## Triage Complete

**Total Items:** [X] **Approved:** [Y] **Skipped:** [Z]

### Approved Issues (Ready for Work):

- `bd-a1b2` (P1) - Transaction boundary issue
- `bd-c3d4` (P2) - Cache performance improvement ...

### Skipped Items:

- Item #5: [reason] - Closed as rejected
- Item #12: [reason] - Closed as rejected

### Summary of Changes Made:

During triage, the following updates occurred:

- **Approved:** Issues labeled `approved` and ready for work
- **Rejected:** Issues closed with rejection reason

### Next Steps:

1. View approved issues ready for work:
   ```bash
   bd ready
   bd list --label approved
   ```
````

2. Start work on approved items:

   ```bash
   /resolve_todo_parallel  # Work on multiple approved items efficiently
   ```

3. Or pick individual items to work on

4. As you work, update issue status:
   ```bash
   bd update <id> --status in_progress --claim
   bd close <id> --reason "Fixed in commit abc123"
   ```

```

## Example Response Format

```

---

Issue #5: Missing Transaction Boundaries for Multi-Step Operations

Severity: P1 (CRITICAL)

Category: Data Integrity / Security

Description: The google_oauth2_connected callback in GoogleOauthCallbacks concern performs multiple database operations without transaction protection. If any step fails midway, the database is left in an inconsistent state.

Location: src/middleware/authCallbacks.ts:13-50

Problem Scenario:

1. User.update succeeds (email changed)
2. Account.save! fails (validation error)
3. Result: User has changed email but no associated Account
4. Next login attempt fails completely

Operations Without Transaction:

- User confirmation (line 13)
- Waitlist removal (line 14)
- User profile update (line 21-23)
- Account creation (line 28-37)
- Avatar attachment (line 39-45)
- Journey creation (line 47)

Proposed Solution: Wrap all operations in ApplicationRecord.transaction do ... end block

Estimated Effort: Small (30 minutes)

---

Do you want to approve this issue?

1. yes - approve for work
2. next - skip this item
3. custom - modify before approving

```

## Important Implementation Details

### Status Transitions During Triage

**When "yes" is selected:**
1. Create issue with `bd create` (or label existing with `approved`)
2. Add detailed context as comment with `bd comments add`
3. Confirm: "Approved: `<id>` - [Title] (Priority: P1)"

**When "next" is selected:**
1. Close the issue with rejection reason: `bd close <id> --reason "Rejected during triage"`
2. Skip to next item

### Progress Tracking

Every time you present an item as a header, include:
- **Progress:** X/Y completed (e.g., "3/10 completed")

Example:
```

Progress: 3/10 completed

```

### Do Not Code During Triage

- Present findings
- Make yes/next/custom decisions
- Create/update beads issues
- Do NOT implement fixes or write code
- That's for /resolve_todo_parallel phase
```

When done give these options

```markdown
What would you like to do next?

1. run /resolve_todo_parallel to resolve the issues
2. sync beads: `bd sync`
3. nothing, go chill
```
