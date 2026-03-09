---
description: Review branch changes efficiently using diffs (context-optimized)
---

Efficiently rebuild context for the current branch using **diffs instead of full file reads**:

## Phase 1: Overview (Always Do)

1. Detect the parent branch dynamically:
   - First try: `git rev-parse --abbrev-ref @{upstream}` (tracking branch)
   - Fallback 1: Check if `origin/main` exists
   - Fallback 2: Check if `origin/master` exists
   - Fallback 3: Use `HEAD~1` if on detached HEAD

2. Get change statistics:
   ```bash
   git diff --stat <parent>...HEAD
   ```

3. Get recent commits on this branch:
   ```bash
   git log --oneline <parent>..HEAD
   ```

## Phase 2: Diff Review (Primary Method)

4. Show the actual changes (NOT full files):
   ```bash
   git diff <parent>...HEAD
   ```

   **For large diffs (>500 lines)**: Use `git diff --stat` to identify key files, then selectively diff important ones:
   ```bash
   git diff <parent>...HEAD -- path/to/important/file.php
   ```

## Phase 3: Summary

5. Provide a concise summary:
   - Branch name and parent branch used
   - Number of commits and files changed
   - **Key changes** (what was added/modified/removed)
   - Current state of work (complete, in-progress, blocked)
   - Any TODO comments or incomplete work visible in diffs

6. Ask the user what they want to work on next

## When to Read Full Files

Only read full files when:
- The diff shows a **new file** that needs full context
- You need to understand surrounding code not visible in diff
- The user specifically requests it

**Default behavior**: Show diffs, not full files.

---

**When to use this command:**
- After `/clear` to rebuild working context efficiently
- When starting a new session on an existing branch
- To get oriented on what has changed in the current branch

**Why diffs are better:**
- Diffs show ~10-50x less content than full files
- Focus on what actually changed, not unchanged boilerplate
- Preserves context window for actual work

**Note:** This command dynamically detects the branch parent, so it works with any base branch.
