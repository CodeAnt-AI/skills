---
name: review
description: Run an AI-powered CodeAnt code review on your current changes
---

Run a CodeAnt AI code review on the current changes and report the findings.

## Instructions

1. Determine the appropriate review scope based on the user's request:
   - If they mention "staged" files, use `--staged`
   - If they mention "uncommitted" changes, use `--uncommitted`
   - If they mention "last commit", use `--last-commit`
   - If they mention a specific branch, use `--base <branch>`
   - If they mention a specific commit, use `--base-commit <commit>`
   - Otherwise, default to `--uncommitted` to review current work in progress

2. Run the CodeAnt review command:

```bash
codeant review <scope-flag>
```

3. Wait for the review to complete. It may take 30-60 seconds depending on the size of the changes.

4. Parse the output and present the findings to the user in a clear, organized format:
   - Group issues by severity (CRITICAL > MAJOR > MINOR)
   - For each issue, show the file path, line number, and description
   - Highlight any security-related findings first

5. If the user asks you to fix the issues, proceed to implement the suggested changes.

Do NOT run this command with `--all` unless the user explicitly asks to review both committed and uncommitted changes together.
