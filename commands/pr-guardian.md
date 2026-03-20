# PR Guardian: Automated PR Completion Monitor

Long-running, language-agnostic process that monitors your PR until all checks pass and review comments are addressed.

**Usage:** `/pr-guardian [PR_NUMBER]`

## Overview

This command monitors your PR in a polling loop, handling:
- GitHub check status monitoring
- Bot review comment triage (CodeRabbit, Codex, any `[bot]` user)
- Automatic fixes with commit pushes
- Comment resolution or dismissal with reasoning
- Comprehensive audit logging

This skill is **project-agnostic** — it works with any language, framework, or repo. It only interacts with the PR via GitHub APIs and the local codebase.

## Instructions

### 1. Initialize

1. **Determine PR:**
    - If PR number provided, use it
    - Otherwise, find PR for current branch: `gh pr list --head $(git branch --show-current) --json number,url --jq '.[0].number'`
    - If no PR found, error and exit

2. **Create audit log file:**
    - Filename: `/tmp/.pr-guardian-{PR_NUMBER}.audit.log`
    - Initialize with header:
   ```
   # PR Guardian Audit Log - PR #{number}
   # Started: {timestamp}
   # Branch: {branch_name}
   # =========================================
   ```

3. **Log initialization** with PR details, branch, and starting state

### 2. Polling Loop

Run this loop until either:
- All checks pass AND no unresolved review comments remain
- User interrupts (Ctrl+C)
- 2 hours elapsed (safety timeout)

**Each iteration:**

```
[{timestamp}] === Polling Iteration {N} ===
```

#### 2a. Check GitHub Checks Status

```bash
gh pr checks {PR_NUMBER} --json name,state,conclusion --jq '.[] | "\(.name): \(.state) (\(.conclusion // "pending"))"'
```

- Log all check statuses
- If any check is FAILED:
    - Identify the failing check
    - If it's a fixable issue (linting, tests), attempt to fix
    - Log what was attempted
- If all checks PASS or PENDING, continue to comments

#### 2b. Fetch Review Comments

Fetch all review comments:
```bash
gh api repos/{owner}/{repo}/pulls/{PR_NUMBER}/comments --jq '.[] | {id, user: .user.login, body, path, line: .line, created_at, in_reply_to_id}'
```

Also fetch PR review threads:
```bash
gh api graphql -f query='
query($owner: String!, $repo: String!, $pr: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          comments(first: 10) {
            nodes {
              id
              author { login }
              body
              path
              line
            }
          }
        }
      }
    }
  }
}' -f owner="{owner}" -f repo="{repo}" -F pr={PR_NUMBER}
```

Filter for comments from:
- `coderabbitai[bot]`
- `codex` (or similar bot names)
- Any user with `[bot]` suffix

Skip already resolved threads.

#### 2c. Triage Each Unresolved Comment

For each unresolved bot comment, use this decision framework:

**Analyze the comment:**
1. Read the full comment body
2. Read the relevant code context (file + surrounding lines)
3. Classify the comment type:
    - `STYLE`: Code style, formatting, naming
    - `BUG`: Potential bug or logic error
    - `SECURITY`: Security concern
    - `PERFORMANCE`: Performance suggestion
    - `ARCHITECTURE`: Design/pattern suggestion
    - `DOCUMENTATION`: Missing or incorrect docs
    - `NITPICK`: Minor suggestion, subjective

**Decision criteria - FIX if:**
- It's a valid bug, security issue, or clear improvement
- The fix is straightforward and low-risk
- It aligns with project conventions (check project config files if available)

**Decision criteria - DISMISS if:**
- It's a false positive (code is correct)
- It contradicts project conventions
- The suggested change would break functionality
- It's purely stylistic and project doesn't enforce it
- The bot misunderstood the context

**Log the decision:**
```
[{timestamp}] COMMENT: {comment_id} by {author}
  File: {path}:{line}
  Type: {classification}
  Summary: {brief_summary}
  Decision: {FIX|DISMISS}
  Reasoning: {detailed_reasoning}
```

#### 2d. Apply Fixes

For comments marked FIX:

1. **Read current file state**
2. **Make the fix** using Edit tool
3. **Log the change:**
   ```
   [timestamp] FIX APPLIED: {comment_id}
     File: {path}
     Change: {brief description}
   ```

After all fixes for this iteration:

4. **Stage and commit:**
   ```bash
   git add -A && git commit -m "fix: address review comments

   Fixes applied for comments: {list_of_comment_ids}

   Co-Authored-By: Claude <noreply@anthropic.com>"
   ```

5. **Push:**
   ```bash
   git push
   ```

6. **Resolve the comment threads** on GitHub:
   ```bash
   gh api graphql -f query='
   mutation($threadId: ID!) {
     resolveReviewThread(input: {threadId: $threadId}) {
       thread { isResolved }
     }
   }' -f threadId="{thread_id}"
   ```

7. **Log:**
   ```
   [{timestamp}] PUSHED: {commit_sha}
   [{timestamp}] RESOLVED: {thread_ids}
   ```

#### 2e. Dismiss Comments

For comments marked DISMISS:

1. **Reply with reasoning:**
   ```bash
   gh api repos/{owner}/{repo}/pulls/{PR_NUMBER}/comments \
     --method POST \
     -f body="Thank you for the suggestion. After review, I've decided not to apply this change:

   **Reason:** {reasoning}

   If you disagree, please let me know and I'll reconsider." \
     -F in_reply_to={comment_id}
   ```

2. **Resolve the thread** (comment explains why not fixed)

3. **Log:**
   ```
   [{timestamp}] DISMISSED: {comment_id}
     Reason: {reasoning}
     Reply posted: Yes
     Thread resolved: Yes
   ```

#### 2f. Wait Before Next Poll

- If changes were made, wait 2 minutes (let CI run)
- If no changes needed, wait 1 minute
- Log: `[{timestamp}] Sleeping for {N} seconds...`

### 3. Completion

When loop ends (all checks pass, no unresolved comments):

```
[{timestamp}] === PR GUARDIAN COMPLETE ===
  Total iterations: {N}
  Fixes applied: {count}
  Comments dismissed: {count}
  Duration: {time}
  Final status: {READY_FOR_MERGE|TIMED_OUT|INTERRUPTED}
```

**Notify the user in chat:**
- Summarize what was done
- List any comments that need human attention
- Mention the audit log location in `/tmp/`

## Audit Log Format

The `/tmp/.pr-guardian-{PR_NUMBER}.audit.log` file uses this format:

```
# PR Guardian Audit Log - PR #1234
# Started: 2024-01-28T10:30:00Z
# Branch: feature/my-feature
# =========================================

[2024-01-28T10:30:05Z] INITIALIZED
  PR: #1234
  URL: https://github.com/owner/repo/pull/1234
  Branch: feature/my-feature
  Base: main

[2024-01-28T10:30:10Z] === Polling Iteration 1 ===

[2024-01-28T10:30:12Z] CHECKS STATUS
  ci/tests: success (success)
  ci/lint: in_progress (pending)
  coderabbit/review: success (success)

[2024-01-28T10:30:15Z] COMMENT: 123456 by coderabbitai[bot]
  File: src/service/payment.ts:45
  Type: BUG
  Summary: Missing null check before accessing property
  Decision: FIX
  Reasoning: Valid concern - user could be null if not authenticated

[2024-01-28T10:30:20Z] FIX APPLIED: 123456
  File: src/service/payment.ts
  Change: Added null check with early return

[2024-01-28T10:30:25Z] COMMENT: 123457 by coderabbitai[bot]
  File: src/controller/api.ts:89
  Type: STYLE
  Summary: Suggests using single quotes instead of double quotes
  Decision: DISMISS
  Reasoning: Project linter enforces double quotes

[2024-01-28T10:30:30Z] PUSHED: abc123def
[2024-01-28T10:30:32Z] RESOLVED: thread_123456
[2024-01-28T10:30:35Z] DISMISSED: 123457
  Reply posted: Yes
  Thread resolved: Yes

[2024-01-28T10:30:40Z] Sleeping for 120 seconds...

# ... more iterations ...

[2024-01-28T10:45:00Z] === PR GUARDIAN COMPLETE ===
  Total iterations: 5
  Fixes applied: 3
  Comments dismissed: 2
  Duration: 15m
  Final status: READY_FOR_MERGE
```

## Safety Considerations

1. **Each fix is a separate decision** - no batch "fix everything"
3. **Always push after fixes** - don't accumulate unpushed changes
4. **Timeout after 2 hours** - prevents infinite loops
5. **Log everything** - full transparency for review

## Tips

- Run with `--background` if you want to do other work
- Check the audit log anytime to see progress
- If interrupted, you can restart - it will pick up where it left off
- Audit logs live in `/tmp/` so they auto-clean on reboot
