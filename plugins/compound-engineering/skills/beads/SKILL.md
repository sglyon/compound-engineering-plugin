---
name: beads
description: This skill should be used when managing issue tracking, task management, and work item coordination using the Beads (`bd`) CLI. It provides workflows for creating issues, managing dependencies, triaging findings, and tracking work across agent sessions.
---

# Beads Issue Tracking Skill

## Overview

Beads (`bd`) is a distributed, git-backed graph issue tracker designed for AI coding agents. Issues are stored in `.beads/issues.jsonl` alongside code, versioned and merged like code with zero-conflict hash-based IDs.

This skill should be used when:
- Creating new issues from findings or feedback
- Managing issue lifecycle (open -> in_progress -> closed)
- Triaging pending items for approval
- Checking or managing dependencies between issues
- Converting PR comments or code findings into tracked work
- Finding unblocked work items ready to start
- Tracking progress with comments during execution

## Prerequisites

Beads must be initialized in the project:

```bash
# Check if already initialized
bd status

# If not, initialize
bd init
```

## AI Agent Quickstart

Before starting work in a beads-enabled project, run:

```bash
# Get AI-optimized workflow context (~80 lines of guidance)
bd prime

# Get the AGENTS.md snippet to add to a project
bd onboard
```

`bd prime` outputs dynamic workflow context. Add `bd hooks install` to auto-inject it at session start.

## Issue Model

Each bead (issue) has these key fields:

| Field | Values | Description |
|-------|--------|-------------|
| `status` | `open`, `in_progress`, `blocked`, `deferred`, `closed` | Workflow state |
| `priority` | 0 (critical), 1 (high), 2 (normal), 3 (low), 4 (backlog) | Urgency level |
| `issue_type` | `bug`, `feature`, `task`, `epic`, `chore` | Category |
| `labels` | arbitrary strings | Tags for filtering |
| `assignee` | string | Who is working on it |
| `dependencies` | blocks, parent-child, related | Relationships to other issues |

**Priority mapping from P1/P2/P3:**
- P1 (critical) -> priority 0
- P2 (important) -> priority 1
- P3 (nice-to-have) -> priority 2

## Common Workflows

### Creating a New Issue

```bash
# Basic creation
bd create "Fix N+1 query in users controller" -t bug -p 1

# With full details
bd create "Add OAuth support" \
  -t feature \
  -p 1 \
  -d "Users need to authenticate via Google OAuth" \
  -l auth,backend \
  -a "agent-name"

# Quick capture (returns only the ID for scripting)
bd q "Investigate slow page load on /dashboard"
```

**When to create an issue:**
- Requires more than 15-20 minutes of work
- Needs research, planning, or multiple approaches considered
- Has dependencies on other work
- Requires approval or prioritization
- Part of a larger feature or refactor
- Technical debt needing documentation

**When to act immediately instead:**
- Issue is trivial (< 15 minutes)
- Complete context available now
- No planning needed
- User explicitly requests immediate action
- Simple bug fix with obvious solution

### Finding Work

```bash
# Show unblocked issues ready to start (no open blockers)
bd ready

# Sort by priority (critical first)
bd ready --sort priority

# Show only unassigned work
bd ready --unassigned

# Show blocked issues and what blocks them
bd blocked

# List all open issues
bd list --status open

# List by priority
bd list --priority 0          # critical only
bd list --priority-max 1      # critical and high (0 and 1)

# List by label
bd list --label security
bd list --label code-review

# Search across all fields
bd search "payment"
```

### Updating Issues

```bash
# Claim and start work (--claim atomically sets assignee + status=in_progress)
bd update <id> --claim

# Change priority
bd update <id> --priority 0

# Add labels
bd label add <id> security
bd label add <id> code-review

# Add a comment (work log entry)
bd comments add <id> "Investigated root cause - N+1 in users#index, loading 50 queries per page load"

# Update description
bd update <id> --description "Updated description with new findings"

# Append notes
bd update <id> --append-notes "Found similar issue in posts controller"
```

### Managing Dependencies

```bash
# Issue A blocks issue B (B cannot start until A is closed)
bd dep add <B> <A>

# Shorthand: A blocks B
bd dep <A> --blocks <B>

# Create a related link (non-blocking)
bd dep relate <A> <B>

# Show dependency tree
bd dep tree <id>

# List what blocks/depends on an issue
bd dep list <id>

# Check for dependency cycles
bd dep cycles
```

### Closing Issues

```bash
# Close with reason
bd close <id> --reason "Fixed in commit abc123"

# Close and get suggestion for next work
bd close <id> --suggest-next

# Reopen if needed
bd reopen <id>
```

### Deferring Issues

```bash
# Defer (hidden from bd ready until specified time)
bd defer <id> --until tomorrow
bd defer <id> --until "+6h"

# Bring back a deferred issue
bd undefer <id>
```

### Triaging Pending Items

```bash
# List all open issues for triage
bd list --status open

# Show overview stats
bd status

# Mark as duplicate
bd duplicate <id> --of <canonical-id>

# Bulk operations with labels
bd label add <id> approved
bd label add <id> wont-fix
```

### Working with Epics

```bash
# Create an epic
bd create "User Authentication System" -t epic -p 1

# Add child issues to epic
bd create "Implement login form" -t task --parent <epic-id>
bd create "Add password reset" -t task --parent <epic-id>

# Check epic progress
bd epic status

# List children of an epic
bd children <epic-id>

# Visualize dependency graph
bd graph <epic-id>
```

## Integration with Development Workflows

| Trigger | Flow | Tool |
|---------|------|------|
| Code review | `/workflows:review` -> Findings -> `/triage` -> Beads issues | Review agents + beads skill |
| PR comments | `/resolve_pr_parallel` -> Individual fixes -> Beads issues | gh CLI + beads skill |
| Code TODOs | `/resolve_todo_parallel` -> Fixes + Complex issues | Agent + beads skill |
| Planning | Brainstorm -> Create issues -> Work -> Close | beads skill |
| Feedback | Discussion -> Create issue -> Triage -> Work | beads skill + slash commands |

## Quick Reference

**Creating:**
```bash
bd create "Title" -t task -p 1 -l label1,label2    # Create issue
bd q "Quick note"                                    # Quick capture
```

**Finding work:**
```bash
bd ready                    # Unblocked work
bd ready --sort priority    # By priority
bd blocked                  # What's stuck
bd list --status open       # All open
bd search "keyword"         # Full-text search
```

**Updating:**
```bash
bd update <id> --claim                         # Claim + set in_progress atomically
bd comments add <id> "Progress note"           # Add comment
bd close <id> --reason "Done"                  # Complete
```

**Dependencies:**
```bash
bd dep add <blocked> <blocker>    # A blocks B
bd dep tree <id>                  # Visualize
bd dep cycles                     # Check cycles
```

**Labels:**
```bash
bd label add <id> tag-name        # Add label
bd label remove <id> tag-name     # Remove label
bd label list-all                 # All labels in use
```

**Batch operations (CRITICAL for performance):**

`bd` auto-commits to Dolt after every write by default, which can be slow for bulk operations. For bulk operations, use `--dolt-auto-commit batch` to defer commits, then run `bd dolt commit` once at the end. Never run concurrent `bd` writes — they corrupt the SQLite WAL.

```bash
# WRONG — each command auto-commits individually
bd create "Issue 1" -t task -p 1
bd create "Issue 2" -t task -p 1
bd dep add issue-2 issue-1
# Total: slow

# RIGHT — write a bash script with --dolt-auto-commit batch
bd create "Issue 1" -t task -p 1 --dolt-auto-commit batch --silent
bd create "Issue 2" -t task -p 1 --dolt-auto-commit batch --silent
bd dep add issue-2 issue-1 --dolt-auto-commit batch
bd dolt commit -m "Batch create issues"   # One commit at the end
# Total: much faster
```

For creating many issues (10+), write a single bash script that does all creates, deps, and label operations with `--dolt-auto-commit batch`, then one `bd dolt commit` at the end. Execute the entire script in one Bash tool call.

**Sync with git:**
```bash
bd sync              # Export DB to JSONL (git-tracked)
```

Two sync topologies:
- **Server mode** (client → remote Dolt server): All reads/writes go directly to the shared server. No push/pull needed — data is already shared.
- **Embedded + remote mode**: Use `bd dolt push` / `bd dolt pull` to sync a local embedded Dolt store with a remote (DoltHub, etc.).

## Key Distinctions

**Beads system (this skill):**
- Git-backed issue tracker using `.beads/issues.jsonl`
- Hash-based IDs (e.g., `bd-a1b2`) prevent merge conflicts
- Dependency-aware graph with blocking relationships
- Persistent across sessions and branches
- `bd ready` finds unblocked work automatically

**Built-in TodoWrite/TaskCreate tools:**
- In-memory task tracking during agent sessions
- Temporary tracking for single conversation
- Not persisted to disk across sessions
- Use for simple within-session checklists only

## JSON Output

All `bd` commands support `--json` for machine-readable output:

```bash
bd ready --json          # JSON array of ready issues
bd show <id> --json      # Full issue details as JSON
bd list --json           # All matching issues as JSON
```
