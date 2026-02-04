---
name: resolve_parallel
description: Resolve all TODO comments using parallel processing
argument-hint: "[optional: specific TODO pattern or file]"
---

Resolve all TODO comments using parallel processing.

## Workflow

### 1. Analyze

Gather the things todo from above.

### 2. Plan

Create beads issues for all unresolved items grouped by type. Make sure to look at dependencies that might occur and prioritize the ones needed by others. For example, if you need to change a name, you must wait to do the others.

```bash
# Create an issue for each TODO item
bd create "Resolve TODO: <description>" -t chore -p 2 -l todo-comment

# Track dependencies between them
bd dep add <blocked-id> <blocker-id>
```

Output a mermaid flow diagram showing how we can do this. Can we do everything in parallel? Do we need to do one first that leads to others in parallel? I'll put the to-dos in the mermaid diagram flow-wise so the agent knows how to proceed in order.

### 3. Implement (PARALLEL)

Spawn a pr-comment-resolver agent for each unresolved item in parallel.

So if there are 3 comments, it will spawn 3 pr-comment-resolver agents in parallel. Like this:

1. Task pr-comment-resolver(comment1)
2. Task pr-comment-resolver(comment2)
3. Task pr-comment-resolver(comment3)

Always run all in parallel subagents/Tasks for each item.

### 4. Commit & Resolve

- Commit changes
- Close each resolved issue:
  ```bash
  bd close <id> --reason "TODO resolved in commit <sha>"
  ```
- Push to remote
