---
name: review-local
description: Run a CodeAnt AI code review on local changes (uncommitted, staged, last commit, or against a branch), present findings, and apply safe minimal fixes for all issues
---

Run a CodeAnt AI code review on local changes, present all findings clearly, and apply safe minimal fixes for every issue found.

## Instructions

### Step 1 — Determine the Review Scope

Choose the scope based on what the user says:

| User says                                    | Flag to use                |
|----------------------------------------------|----------------------------|
| "staged" or "staged files"                   | `--staged`                 |
| "uncommitted" or "my changes" or "wip"       | `--uncommitted`            |
| "last commit"                                | `--last-commit`            |
| "last N commits" (where N ≤ 5)               | `--last-n-commits <N>`     |
| "committed" or "unpushed"                    | `--committed`              |
| a specific branch name (e.g., "against main")| `--base <branch>`          |
| a specific commit hash                       | `--base-commit <hash>`     |
| "everything" or "all changes"                | `--all`                    |
| nothing specific                             | `--uncommitted` (default)  |

### Step 2 — Run the CodeAnt Review

```bash
codeant review <scope-flag>
```

Let the user know the review is running — it may take 30–90 seconds depending on the size of the diff.

If the review returns zero issues, tell the user: "CodeAnt found no issues in your changes." and stop. Do not invent problems.

### Step 3 — Present the Findings

The review output contains code suggestions. Each suggestion has:

| Field            | Description                                    |
|------------------|------------------------------------------------|
| `issue_content`  | Description of what's wrong                    |
| `relevant_file`  | File path where the issue is                   |
| `start_line`     | Line number where the issue starts             |
| `label`          | Category: Code Quality, Security, Performance, Maintainability, etc. |

Present the results in a structured format:

**Overview:**
- Total issues found
- Breakdown by label/category
- Number of files affected

**Issues listed by file:**

For each issue show:
- **File** and **line number**
- **Category** label
- **Description** — what is wrong and why it matters

Highlight **Security** issues at the top — they deserve immediate attention regardless of category.

### Step 4 — Fix Each Issue

Work through issues file-by-file (to minimize context switches). For each issue:

#### 4a. Read the Context

Read the file at the reported `start_line`, with **25 lines above and 25 lines below** for full context.

#### 4b. Understand the Issue

Read the `issue_content` carefully. Understand:
- What exactly is wrong with the current code.
- What the correct behavior should be.
- What the minimal change is to address it.

#### 4c. Validate Before Fixing

Before writing any change:

1. **Verify the code at the reported line matches the issue description.** If the code has already been changed (e.g., by a previous fix in this session), re-read the file and adjust.

2. **Assess the impact of the fix:**
   - Does this change alter the function's return type or public API?
   - Does it change behavior for inputs that were previously handled correctly?
   - Does it remove or weaken existing error handling?
   - Could it introduce a regression in callers of this code?

3. **If the fix is safe** (addresses only the reported issue, does not change behavior for other code paths), apply it.

4. **If the fix could break something** (changes a public interface, alters control flow for non-error paths, affects multiple callers), flag it to the user: "This fix may affect [specific thing]. Skipping — please review manually." Do NOT apply it.

#### 4d. Apply Minimal Fixes

- Change **only** what is necessary to resolve the issue.
- Do not refactor, rename, restructure, or "improve" surrounding code.
- If the fix requires a new import, add it.
- If multiple issues are in the same file, apply all safe fixes to that file before moving on, checking that fixes don't conflict.

### Step 5 — Run Verification Review

After all fixes are applied, run the review again with the same scope:

```bash
codeant review <same-scope-flag>
```

This confirms:
- The original issues are resolved.
- The fixes did not introduce new problems.

### Step 6 — Report Results

**Initial review:**
- Total issues found, breakdown by category.

**Fixes applied (N):**
- For each: file, line, one-line description of what was changed.

**Skipped — may break logic (N):**
- For each: file, line, specific reason it was not safe to auto-fix.

**Verification:**
- Results of the second review pass.
- If clean: "Verification passed — no remaining issues."
- If new issues found: list them and explain.

### Important Rules

- Do **NOT** commit or push changes. Let the user review diffs first.
- Do **NOT** modify files outside the review scope.
- Do **NOT** apply a fix if you cannot verify it is safe. Skip and explain.
- Keep fixes **minimal**. Do not over-engineer or refactor.
- If unsure about a fix, ask the user rather than guessing.
