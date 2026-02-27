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

Launch all review agents as **background subagents** using `run_in_background: true` on the Task tool. This lets all reviewers work concurrently while the lead synthesizes findings.

Run these agents in the background at the same time:

1. Task sglyon-python-reviewer(plan content) — run_in_background: true — Review plan for Python implementation concerns
2. Task sglyon-typescript-reviewer(plan content) — run_in_background: true — Review plan for TypeScript implementation concerns
3. Task code-simplicity-reviewer(plan content) — run_in_background: true — Review plan for unnecessary complexity and over-engineering
4. Task architecture-strategist(plan content) — run_in_background: true — Review plan for architectural soundness (optional, if plan is architectural)

The lead synthesizes all reviewer feedback into a prioritized list of recommendations. Where reviewers disagree, present both perspectives.

## Output Format

Present a consolidated review with:
- **Strengths**: What the plan does well
- **Concerns**: Issues that should be addressed, prioritized by impact
- **Suggestions**: Improvements that would strengthen the plan
- **Conflicts**: Areas where reviewers disagreed (with both perspectives)
