---
name: review-fix
description: Review code with CodeAnt AI and automatically fix all identified issues
---

Run a full CodeAnt AI code review on uncommitted changes, then automatically fix all identified issues.

## Instructions

1. First, run the CodeAnt review on uncommitted changes:

```bash
codeant review --uncommitted
```

2. Wait for the review to complete and parse the output.

3. For each issue found, starting with the highest severity:
   - Read the relevant file and understand the surrounding context
   - Implement the suggested fix
   - Verify the fix doesn't break existing functionality

4. After all fixes are applied, run the review again to verify:

```bash
codeant review --uncommitted
```

5. Report the results to the user:
   - List what was found in the initial review
   - Describe each fix that was applied
   - Show the results of the verification review
   - If any issues remain, explain why they were not auto-fixed

Important: Do NOT commit the changes automatically. Let the user review and commit when ready.
