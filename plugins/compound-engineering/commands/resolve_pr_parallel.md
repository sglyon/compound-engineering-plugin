---
name: resolve_pr_parallel
description: Resolve all PR comments using parallel processing
argument-hint: "[optional: PR number or current PR]"
---

Resolve all PR comments using parallel processing.

Claude Code automatically detects and understands your git context:

- Current branch detection
- Associated PR context
- All PR comments and review threads
- Can work with any PR by specifying the PR number, or ask it.

## Workflow

### 1. Analyze

Get all unresolved comments for PR

```bash
gh pr status
bin/get-pr-comments PR_NUMBER
```

### 2. Plan

Create beads issues for all unresolved items grouped by type:

```bash
# Create an issue for each PR comment
bd create "PR comment: <summary>" -t task -p 1 -l pr-review

# Track dependencies if needed
bd dep add <blocked-id> <blocker-id>
```

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
  bd close <id> --reason "PR comment resolved"
  ```
- Run bin/resolve-pr-thread THREAD_ID_1
- Push to remote

Last, check bin/get-pr-comments PR_NUMBER again to see if all comments are resolved. They should be, if not, repeat the process from 1.
