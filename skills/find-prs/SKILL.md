---
name: find-prs
description: List and explore pull requests in the repository
---

List and explore pull requests in the current repository.

## Instructions

1. **Determine filters from the user's request:**
   - If they mention "my PRs" or "my pull requests", try to detect the git user: `git config user.name` or `git config user.email`, then use `--author`
   - If they mention "closed" or "merged", use `--state closed`
   - If they mention a branch name, use `--source-branch <branch>`
   - Otherwise, list open PRs by default

2. **Run the appropriate command:**

```bash
codeant pr list --state <state> [--author <author>] [--source-branch <branch>] --limit 10
```

3. **Parse the JSON output and present results as a table:**
   - PR number
   - Title
   - Author
   - Source branch → Target branch
   - State

4. **If the user asks for details on a specific PR:**

```bash
codeant pr get --pr-number <N>
```

This returns additions, deletions, changed files, and review summary.

5. **Offer next steps based on context:**
   - "Want me to review the comments on this PR? Run `/codeant:list-comments <N>`"
   - "Want me to fix all unaddressed issues? Run `/codeant:check-pr <N>`"
   - "Want me to run a fresh review? Run `/codeant:review`"
