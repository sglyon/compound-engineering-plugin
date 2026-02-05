---
name: plan_review
description: Have multiple specialized agents review a plan in parallel
argument-hint: "[plan file path or plan content]"
---

# Plan Review

Review a plan using multiple specialized agents working in parallel. Reviewers can debate and challenge each other's feedback, producing a more thorough review.

## Plan Target

<plan_target> #$ARGUMENTS </plan_target>

**If the plan target above is empty:**
1. Check for recent plans: `ls -la docs/plans/`
2. Ask the user: "Which plan would you like reviewed? Please provide the path."

## Parallelization Strategy

**Preferred: Agent Team** (requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`)

Create an agent team with 3+ reviewer teammates who discuss and debate the plan. Teammates should challenge each other's feedback and cross-reference findings.

Spawn teammates:
- **sglyon-python-reviewer**: Review plan for Python implementation concerns
- **sglyon-typescript-reviewer**: Review plan for TypeScript implementation concerns
- **code-simplicity-reviewer**: Review plan for unnecessary complexity and over-engineering
- **architecture-strategist**: Review plan for architectural soundness (optional, if plan is architectural)

The lead synthesizes all reviewer feedback into a prioritized list of recommendations. Where reviewers disagree, present both perspectives.

**Fallback: Parallel Subagents**

If agent teams are not available, use @agent mentions to review in parallel:

Have @agent-sglyon-python-reviewer @agent-sglyon-typescript-reviewer @agent-code-simplicity-reviewer review this plan in parallel.

## Output Format

Present a consolidated review with:
- **Strengths**: What the plan does well
- **Concerns**: Issues that should be addressed, prioritized by impact
- **Suggestions**: Improvements that would strengthen the plan
- **Conflicts**: Areas where reviewers disagreed (with both perspectives)
