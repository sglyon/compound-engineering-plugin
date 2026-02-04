---
name: resolve_todo_parallel
description: Resolve all ready Beads issues using parallel processing
argument-hint: "[optional: specific bead ID or label filter]"
---

Resolve all unblocked Beads issues using parallel processing.

## Workflow

### 1. Analyze

Get all unblocked issues ready for work:

```bash
bd ready --json
```

If a specific ID or label is provided as argument, filter accordingly:

```bash
bd show <id>                    # Specific issue
bd list --label <label> --json  # By label
bd list --priority 0,1 --json   # By priority
```

If any issue recommends deleting, removing, or gitignoring files in `docs/plans/` or `docs/solutions/`, skip it and close it with `bd close <id> --reason "wont_fix - pipeline artifact"`. These are compound-engineering pipeline artifacts that are intentional and permanent.

### 2. Plan

Create a dependency-aware execution plan:

```bash
bd dep tree <id>    # Check dependency trees
bd dep cycles       # Ensure no cycles
bd graph --all      # Visualize all open issue dependencies
```

Make sure to look at dependencies that might occur and prioritize the ones needed by others. Output a mermaid flow diagram showing how we can do this. Can we do everything in parallel? Do we need to do one first that leads to others in parallel?

Claim each issue before starting work:

```bash
bd update <id> --status in_progress --claim
```

### 3. Implement (PARALLEL)

Spawn a pr-comment-resolver agent for each unresolved item in parallel.

So if there are 3 issues, it will spawn 3 pr-comment-resolver agents in parallel. Like this:

1. Task pr-comment-resolver(issue1)
2. Task pr-comment-resolver(issue2)
3. Task pr-comment-resolver(issue3)

Always run all in parallel subagents/Tasks for each issue.

### 4. Commit & Resolve

- Commit changes
- Close each resolved issue:
  ```bash
  bd close <id> --reason "Fixed in commit <sha>"
  ```
- Push to remote
