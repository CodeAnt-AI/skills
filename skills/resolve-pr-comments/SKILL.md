---
name: resolve-pr-comments
description: Find the pull request for the current branch, fetch all unresolved CodeAnt AI review comments, validate each code suggestion against surrounding context, and apply only safe, minimal fixes
---

Find the pull request for the current branch, fetch all unresolved CodeAnt AI review comments, validate each code suggestion against the surrounding code context, and apply only safe, minimal fixes that do not break existing logic.

## Instructions

### Step 1 — Find the Pull Request

The goal is to identify the correct PR. Use the following logic in order:

**If the user provides a PR number** (e.g., `/codeant:resolve-pr-comments 42`):

Use it directly. Skip to Step 2.

**If no PR number is given**, detect it from the current branch:

1. Get the current branch name:
```bash
git rev-parse --abbrev-ref HEAD
```

2. If the branch is `main`, `master`, or `develop`, stop and tell the user: "You are on the default branch. Please switch to a feature branch or provide a PR number."

3. List open PRs filtered by the current branch:
```bash
codeant pr list --source-branch "<current-branch>" --state open --limit 5
```

4. The output is a JSON array of PR objects, each with these fields:
   - `number` — PR number
   - `title` — PR title
   - `state` — open/closed
   - `author` — who created it
   - `sourceBranch` — the source branch name
   - `targetBranch` — the target branch name
   - `url` — link to the PR

5. Match the correct PR:
   - If exactly **one** PR is returned whose `sourceBranch` exactly matches the current branch name, use it.
   - If **multiple** PRs match, present them to the user and ask which one to use.
   - If **zero** PRs match, tell the user: "No open PR found for branch `<branch>`. Please provide a PR number." and stop.

### Step 2 — Fetch CodeAnt Review Comments

Retrieve all review comments that CodeAnt AI posted on the PR:

```bash
codeant pr comments --pr-number <N> --codeant-generated true
```

The output is a JSON array of comment objects with these fields:

| Field              | Description                                                     |
|--------------------|-----------------------------------------------------------------|
| `id`               | Unique comment identifier                                       |
| `type`             | `"review"` (inline on code) or `"issue"` (general PR comment)  |
| `author`           | Comment author (e.g., `codeant-bot`)                            |
| `body`             | The full review comment text in markdown                        |
| `path`             | File path where the comment was left (null for general comments)|
| `line`             | Line number the comment refers to (null for general comments)   |
| `createdAt`        | When the comment was posted                                     |
| `isCodeantComment` | Boolean — true if posted by CodeAnt                             |
| `resolved`         | Boolean — true if the comment has been marked resolved          |

**Filter to unresolved comments only**: From the returned array, keep only comments where `resolved` is `false`. If all comments are resolved, tell the user "All CodeAnt comments on PR #N are already resolved." and stop.

### Step 3 — Categorize the Comments

Go through each unresolved comment and categorize it:

**Inline code comments** (`type` is `"review"` and `path` is not null):
- These point to a specific file and line — they are actionable.
- The `body` field contains the reviewer's feedback. It may include:
  - A description of the issue
  - A code suggestion embedded in a markdown fenced code block
  - Sometimes a `suggestion` block (GitHub-style suggested change)

**General PR comments** (`type` is `"issue"` or `path` is null):
- These are PR-level feedback (architecture, design, process).
- Flag these as "Requires manual review" — do NOT attempt to auto-fix.

### Step 4 — Present a Summary

Before making any changes, present a clear summary:

- **PR**: #N — "title" (link to PR)
- **Total unresolved CodeAnt comments**: X
- **Actionable** (inline with file + line): Y
- **Manual review needed** (general/PR-level): Z

For each actionable comment, show:
- File path and line number
- A one-line summary of what the comment is asking for (extracted from the `body`)
- Whether the comment includes a code suggestion

Ask the user: "Shall I proceed to fix the Y actionable comments?" Wait for confirmation before making changes.

### Step 5 — Fix Each Actionable Comment

For each inline comment (grouped by file to minimize re-reading), do the following:

#### 5a. Read and Understand the Context

1. Read the file at the comment's `line` number, with **30 lines above and 30 lines below** for full context.
2. Read the comment `body` carefully. Identify:
   - **What is the problem?** — What the reviewer says is wrong.
   - **Is there a code suggestion?** — Look for fenced code blocks (` ```suggestion `, ` ```python `, ` ```js `, etc.) or inline code that represents a replacement.
   - **What is the intent?** — What behavior should the code have after the fix.

#### 5b. Validate Before Applying

This is the most critical step. Before making ANY change:

1. **Check that the code the comment references still exists.** The file may have changed since the review. If the code at the referenced line no longer matches what the comment describes, skip this comment and flag it as "Code has changed since review — manual check needed."

2. **If a code suggestion is present in the body:**
   - Extract the suggested code from the markdown.
   - Compare it against the current code at that location.
   - Verify the suggestion is **syntactically valid** in context:
     - Does it reference variables/functions that exist in scope?
     - Does it use imports that are already present (or need to be added)?
     - Does it match the language and style of the surrounding code?
   - Verify the suggestion does **not break logic**:
     - Does it change the return type or signature of a function?
     - Does it alter control flow in a way that affects callers?
     - Does it remove error handling or null checks?
     - Does it change the behavior for edge cases?
   - If the suggestion passes all checks, apply it.
   - If it fails any check, **do NOT apply it**. Instead, flag it: "Suggestion may break existing logic — [specific reason]. Skipping."

3. **If no code suggestion is present:**
   - Analyze the comment to understand the requested change.
   - Write a **minimal fix** — change only what is necessary to address the concern.
   - Do NOT refactor surrounding code, rename variables, or "improve" things beyond the scope of the comment.
   - Apply the same validation checks as above before writing the fix.

#### 5c. Apply the Fix

- Make the smallest possible change that addresses the comment.
- If the fix requires adding an import, add it.
- If multiple comments refer to the same file, apply all fixes to that file before moving to the next file, being careful that fixes don't conflict with each other.

### Step 6 — Report Results

Present a final report:

**Fixed (N comments):**
- For each: file, line, one-line summary of what was changed.

**Skipped — code changed since review (N comments):**
- For each: file, line, reason the code no longer matches.

**Skipped — suggestion may break logic (N comments):**
- For each: file, line, specific reason the suggestion was unsafe.

**Requires manual review (N comments):**
- For each: the PR-level comment body summarized.

### Important Rules

- Do **NOT** commit or push changes. Let the user review diffs and decide.
- Do **NOT** modify files that are not referenced in the comments.
- Do **NOT** apply a suggestion if you cannot verify it is safe. It is always better to skip and explain than to break the code.
- Do **NOT** batch-apply suggestions blindly. Validate each one individually.
- If two comments conflict (e.g., one says add a check, another says remove it), flag both and ask the user.
- Keep fixes **minimal**. A fix for a missing null check should add the null check — not restructure the function.
