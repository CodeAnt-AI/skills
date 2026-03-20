---
name: check-pr
description: Fetch unaddressed CodeAnt review comments on a PR and fix them all
---

Fetch all unaddressed CodeAnt AI review comments on a pull request, then systematically fix every issue.

## Instructions

1. **Identify the PR.** Use the argument provided by the user (e.g., `/codeant:check-pr 42` or `/codeant:check-pr owner/repo#42`). If no PR number is given, detect the current branch and find its open PR:

```bash
# Get current branch name
git rev-parse --abbrev-ref HEAD
```

```bash
# List open PRs and find the one for this branch
codeant pr list --source-branch <current-branch> --state open --limit 1
```

2. **Fetch unaddressed CodeAnt comments.** Get all unresolved comments that CodeAnt generated on this PR:

```bash
codeant pr comments --pr-number <N> --codeant-generated true --addressed false
```

3. **Parse the JSON output.** Each comment includes:
   - `filePath` — the file where the issue was found
   - `lineStart` / `lineEnd` — the line range (may be null for PR-level comments)
   - `body` — the review comment describing the issue
   - `suggestedCode` — the suggested fix (if available)
   - `severity` — the issue severity

4. **Present a summary to the user.** Before fixing, show:
   - Total number of unaddressed comments
   - Grouped by file, with severity and a one-line description of each
   - Which ones have suggested fixes available

5. **Fix each issue, starting with the highest severity:**

   **If `suggestedCode` is available:**
   - Read the file at the specified line range
   - Apply the suggested code replacement directly
   - Verify the replacement makes sense in context

   **If no `suggestedCode`:**
   - Read the file and surrounding context
   - Understand the issue from the comment body
   - Implement an appropriate fix

6. **After all fixes are applied, verify:**

```bash
codeant review --uncommitted
```

7. **Report results to the user:**
   - List each comment that was fixed and how
   - List any comments that could not be auto-fixed (e.g., architectural concerns, PR-level comments) and explain why
   - Show the verification review results

Important: Do NOT commit or push changes automatically. Let the user review the diffs and decide when to commit.
