---
name: list-comments
description: List CodeAnt review comments on a PR with filtering options
---

List CodeAnt AI review comments on a pull request, with options to filter by status.

## Instructions

1. **Identify the PR.** Use the argument provided by the user (e.g., `/codeant:list-comments 42`). If no PR number is given, detect the current branch and find its open PR:

```bash
git rev-parse --abbrev-ref HEAD
```

```bash
codeant pr list --source-branch <current-branch> --state open --limit 1
```

2. **Fetch comments based on user intent:**

   To list all CodeAnt comments:
   ```bash
   codeant pr comments --pr-number <N> --codeant-generated true
   ```

   To list only unaddressed comments:
   ```bash
   codeant pr comments --pr-number <N> --codeant-generated true --addressed false
   ```

   If the user asks about a specific topic, search instead:
   ```bash
   codeant comments search --query "<search-term>" --limit 20
   ```

3. **Present the comments in a clear format:**

   Group by file path. For each comment show:
   - Severity (CRITICAL / MAJOR / MINOR)
   - Line range (if available)
   - One-line summary of the issue
   - Whether a suggested fix is available
   - Whether it has been addressed or not

4. **Offer next steps:**
   - "Would you like me to fix all unaddressed comments? Run `/codeant:check-pr`"
   - "Would you like me to fix a specific issue?"
