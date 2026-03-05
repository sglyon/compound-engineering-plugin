---
name: test-browser
description: Run browser tests on pages affected by current PR or branch
argument-hint: "[PR number, branch name, or 'current' for current branch]"
---

# Browser Test Command

<command_purpose>Run end-to-end browser tests on pages affected by a PR or branch changes using rodney CLI.</command_purpose>

## CRITICAL: Use rodney CLI Only

**DO NOT use Chrome MCP tools (mcp__claude-in-chrome__*).**

This command uses the `rodney` CLI exclusively. Rodney is a persistent headless Chrome automation tool using CSS selectors. It is NOT the same as Chrome browser automation via MCP.

If you find yourself calling `mcp__claude-in-chrome__*` tools, STOP. Use `rodney` Bash commands instead.

## Introduction

<role>QA Engineer specializing in browser-based end-to-end testing</role>

This command tests affected pages in a real browser, catching issues that unit tests miss:
- JavaScript integration bugs
- CSS/layout regressions
- User workflow breakages
- Console errors

## Prerequisites

<requirements>
- Local development server running (e.g., `bun dev`, `npm run dev`, `uvicorn main:app`)
- rodney CLI installed (see Setup below)
- Git repository with changes to test
</requirements>

## Setup

**Check installation:**
```bash
command -v rodney >/dev/null 2>&1 && echo "Installed" || echo "NOT INSTALLED"
```

**Check if Chrome is already running:**
```bash
rodney status
```

See the `rodney` skill for detailed usage.

## Main Tasks

### 0. Verify rodney Installation and Start Chrome

Before starting ANY browser testing, verify rodney is installed and Chrome is running:

```bash
command -v rodney >/dev/null 2>&1 && echo "rodney installed" || echo "NOT INSTALLED - install rodney first"
rodney status || rodney start
```

If installation fails, inform the user and stop.

### 1. Ask Browser Mode

<ask_browser_mode>

Before starting tests, ask user if they want to watch the browser:

Use AskUserQuestion with:
- Question: "Do you want to watch the browser tests run?"
- Options:
  1. **Visible (watch)** - Opens visible browser window so you can see tests run (`rodney start --show`)
  2. **Headless (faster)** - Runs in background, faster but invisible (`rodney start`)

Store the choice. If Chrome isn't already running, start it with the appropriate flag.

</ask_browser_mode>

### 2. Determine Test Scope

<test_target> $ARGUMENTS </test_target>

<determine_scope>

**If PR number provided:**
```bash
gh pr view [number] --json files -q '.files[].path'
```

**If 'current' or empty:**
```bash
git diff --name-only main...HEAD
```

**If branch name provided:**
```bash
git diff --name-only main...[branch]
```

</determine_scope>

### 3. Map Files to Routes

<file_to_route_mapping>

Map changed files to testable routes:

| File Pattern | Route(s) |
|-------------|----------|
| `src/app/*` (Next.js) | Corresponding routes |
| `src/pages/*` (Next.js pages router) | Corresponding routes |
| `src/routes/*` (SvelteKit, Remix) | Corresponding routes |
| `src/components/*` | Pages using those components |
| `app/routers/*.py` (FastAPI) | Corresponding API routes |
| `app/templates/*` (Jinja) | Pages using those templates |
| `src/styles/*`, `*.css` | Visual regression on key pages |
| `internal/handlers/*.go` | Corresponding routes |

Build a list of URLs to test based on the mapping.

</file_to_route_mapping>

### 4. Verify Server is Running

<check_server>

Before testing, verify the local server is accessible:

```bash
rodney open http://localhost:3000
rodney waitload
rodney title  # Should return a page title, not an error
```

If server is not running, inform user:
```markdown
**Server not running**

Please start your development server:
- TypeScript/Next.js: `bun dev` or `npm run dev`
- Python/FastAPI: `uvicorn main:app --reload`
- Go: `go run .`

Then run `/test-browser` again.
```

</check_server>

### 5. Test Each Affected Page

<test_pages>

For each affected route, use rodney CLI commands (NOT Chrome MCP):

**Step 1: Navigate and wait for load**
```bash
rodney open "http://localhost:3000/[route]"
rodney waitstable
```

**Step 2: Verify key elements using CSS selectors**
```bash
rodney exists 'h1'              # Page heading present
rodney exists 'main'            # Primary content rendered
rodney exists '.error' && echo "Error visible!"  # Check for errors
rodney title                    # Confirm page title
```

**Step 3: Test critical interactions**
```bash
rodney click 'button.submit'    # Use CSS selector directly
rodney waitstable
rodney exists '.success-message'
```

**Step 4: Take screenshots**
```bash
rodney screenshot /tmp/page-name.png
rodney screenshot -w 375 -h 812 /tmp/page-name-mobile.png  # Mobile view
```

</test_pages>

### 6. Human Verification (When Required)

<human_verification>

Pause for human input when testing touches:

| Flow Type | What to Ask |
|-----------|-------------|
| OAuth | "Please sign in with [provider] and confirm it works" |
| Email | "Check your inbox for the test email and confirm receipt" |
| Payments | "Complete a test purchase in sandbox mode" |
| SMS | "Verify you received the SMS code" |
| External APIs | "Confirm the [service] integration is working" |

Use AskUserQuestion:
```markdown
**Human Verification Needed**

This test touches the [flow type]. Please:
1. [Action to take]
2. [What to verify]

Did it work correctly?
1. Yes - continue testing
2. No - describe the issue
```

</human_verification>

### 7. Handle Failures

<failure_handling>

When a test fails:

1. **Document the failure:**
   - Screenshot the error state: `rodney screenshot /tmp/error.png`
   - Note the exact reproduction steps

2. **Ask user how to proceed:**
   ```markdown
   **Test Failed: [route]**

   Issue: [description]
   Console errors: [if any]

   How to proceed?
   1. Fix now - I'll help debug and fix
   2. Create issue - Track with `bd create` for later
   3. Skip - Continue testing other pages
   ```

3. **If "Fix now":**
   - Investigate the issue
   - Propose a fix
   - Apply fix
   - Re-run the failing test

4. **If "Create todo":**
   - Create `{id}-pending-p1-browser-test-{description}.md`
   - Continue testing

5. **If "Skip":**
   - Log as skipped
   - Continue testing

</failure_handling>

### 8. Test Summary

<test_summary>

After all tests complete, present summary:

```markdown
## Browser Test Results

**Test Scope:** PR #[number] / [branch name]
**Server:** http://localhost:3000

### Pages Tested: [count]

| Route | Status | Notes |
|-------|--------|-------|
| `/users` | Pass | |
| `/settings` | Pass | |
| `/dashboard` | Fail | Console error: [msg] |
| `/checkout` | Skip | Requires payment credentials |

### Console Errors: [count]
- [List any errors found]

### Human Verifications: [count]
- OAuth flow: Confirmed
- Email delivery: Confirmed

### Failures: [count]
- `/dashboard` - [issue description]

### Created Todos: [count]
- `005-pending-p1-browser-test-dashboard-error.md`

### Result: [PASS / FAIL / PARTIAL]
```

</test_summary>

## Quick Usage Examples

```bash
# Test current branch changes
/test-browser

# Test specific PR
/test-browser 847

# Test specific branch
/test-browser feature/new-dashboard
```

## rodney CLI Reference

**ALWAYS use these Bash commands. NEVER use mcp__claude-in-chrome__* tools.**

```bash
# Browser lifecycle
rodney status                      # Check if Chrome is running
rodney start                       # Start headless Chrome (persists!)
rodney start --show                # Start visible Chrome (debugging)
rodney stop                        # Stop Chrome

# Navigation
rodney open <url>                  # Navigate to URL
rodney back                        # Go back
rodney reload                      # Reload page

# Reading page
rodney title                       # Page title
rodney text 'selector'             # Element text content
rodney exists 'selector'           # Exit 0 if element exists

# Interactions (CSS selectors — no discovery step needed)
rodney click 'selector'            # Click element
rodney input 'selector' "text"     # Fill input
rodney hover 'selector'            # Hover element
rodney submit 'selector'           # Submit form

# Screenshots
rodney screenshot /tmp/out.png           # Viewport screenshot
rodney screenshot -w 1280 -h 900 /tmp/out.png  # At specific viewport
rodney screenshot-el 'selector' /tmp/el.png    # Single element

# Waiting
rodney wait 'selector'             # Wait for element to appear
rodney waitload                    # Wait for page load
rodney waitstable                  # Wait for DOM to stabilize
rodney sleep 2                     # Sleep N seconds (avoid when possible)

# JavaScript (escape hatch for complex operations)
rodney js 'document.title'
rodney js '(() => { var el = document.querySelector("h1"); return el.textContent; })()'
```
