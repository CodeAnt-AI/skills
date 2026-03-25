---
name: codeant-review
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

### Step 1b — Track Skill Invocation

Report that this skill was invoked:

```bash
codeant track --event "skill_invoked" --props '{"skill_name": "codeant-review", "source": "claude-code", "scope": "<scope-flag>"}'
```

### Step 2 — Run the CodeAnt Review

```bash
codeant review <scope-flag>
```

Let the user know the review is running — it may take 30–90 seconds depending on the size of the diff.

If the review returns zero issues, tell the user: "CodeAnt found no issues in your changes." and stop. Do not invent problems.

### Step 3 — Parse the Findings

The review output contains code suggestions. Each suggestion has:

| Field            | Description                                    |
|------------------|------------------------------------------------|
| `issue_content`  | Description of what's wrong                    |
| `relevant_file`  | File path where the issue is                   |
| `start_line`     | Line number where the issue starts             |
| `label`          | Category: Code Quality, Security, Performance, Maintainability, etc. |

### Step 4 — Analyze Each Issue and Assign a Verdict

For each issue (grouped by file to minimize re-reading), do the following:

#### 4a. Read and Understand the Context

1. Read the file at the reported `start_line`, with **30 lines above and 30 lines below** for full context.
2. Read the `issue_content` carefully. Identify:
   - **What is the problem?** — What is wrong with the current code.
   - **What is the fix?** — Draft a minimal change that addresses only the reported issue.
   - **What is the intent?** — What behavior should the code have after the fix.

#### 4b. Validate the Fix

For each issue, run through these checks:

1. **Check that the code at the reported line matches the issue description.** The code may have changed (e.g., by a previous fix in this session). If it no longer matches, mark as `STALE`.

2. **Verify the fix is syntactically valid in context:**
   - Does it reference variables/functions that exist in scope?
   - Does it use imports that are already present (or need to be added)?
   - Does it match the language and style of the surrounding code?

3. **Verify the fix does not break logic:**
   - Does it change the return type or signature of a function?
   - Does it alter control flow in a way that affects callers?
   - Does it remove error handling or null checks?
   - Does it change the behavior for edge cases?

#### 4c. Assign a Verdict to Each Issue

Based on the validation, assign one of these verdicts to every issue:

**ACCEPT — Safe to apply.**
Assign this when ALL of these are true:
- The fix addresses a genuine bug, security issue, or correctness problem
- The fix is syntactically valid and all variables/imports are in scope
- The change does NOT alter the function's return type, signature, or public API
- The change does NOT remove or weaken existing error handling
- The change does NOT affect behavior for inputs that were previously handled correctly
- The fix is localized — it only touches the lines relevant to the issue

**LIKELY ACCEPT — Looks correct, but verify the callers.**
Assign this when:
- The fix is logically sound and addresses a real issue
- BUT it changes behavior in a way that callers or tests might depend on (e.g., a function now returns an error where it previously returned nil, or a previously permissive validation now rejects some inputs)
- The fix itself is correct, but you cannot guarantee no downstream breakage without checking callers

**DO NOT ACCEPT — This could break things.**
Assign this when ANY of these are true:
- The fix changes a function's return type or public interface
- The fix removes existing error handling or fallback logic
- The fix restructures control flow beyond what the issue asks for
- The fix introduces a dependency or import that doesn't exist in the project
- The fix looks like a refactor disguised as a fix — it changes more than necessary
- You cannot understand what the fix does or why it's better

**STALE — Code has changed.**
Assign this when:
- The code at the reported line no longer matches what the issue describes
- The file has been renamed or deleted

### Step 5 — Present the Summary with Verdicts

Before making any changes, present a clear summary to the user:

- **Review scope**: the flag used
- **Total issues found**: N
- **Breakdown by category**: label counts

Then list every issue grouped by verdict:

**ACCEPT — Safe to apply (N):**
For each, show:
- File path and line number
- Category label
- One-line summary of the issue
- One-line explanation of why this is safe: what exactly the fix does and why it cannot break anything
- The actual code change (before → after) so the user can see it

**LIKELY ACCEPT — Verify callers (N):**
For each, show:
- File path and line number
- Category label
- One-line summary of the issue
- What the fix changes and why it's probably correct
- What could break: specifically which callers, tests, or behaviors to check
- The actual code change (before → after)

**DO NOT ACCEPT — Could break logic (N):**
For each, show:
- File path and line number
- Category label
- One-line summary of what the issue asks for
- Specific reason why the fix is risky — what exactly could break
- What the user should do instead (e.g., "review manually", "check with the team", "test this path first")

**STALE — Code changed (N):**
For each, show:
- File path and line number
- What the issue expected to find vs. what's actually there now

Highlight **Security** issues at the top of each verdict group — they deserve immediate attention.

Then ask the user: "I will apply the N ACCEPT fixes now. For the LIKELY ACCEPT fixes, I recommend you review the callers first — want me to apply those too, or skip them for now?"

### Step 6 — Apply the Fixes

After the user confirms:

- Apply all **ACCEPT** fixes.
- Apply **LIKELY ACCEPT** fixes only if the user said yes.
- Do **NOT** apply DO NOT ACCEPT or STALE fixes.
- Make the smallest possible change that addresses each issue.
- If the fix requires adding an import, add it.
- If multiple issues refer to the same file, apply all fixes to that file before moving to the next file, being careful that fixes don't conflict with each other.

### Step 6b — Track Results

After applying fixes, report the outcome:

```bash
codeant track --event "suggestions_applied" --props '{"skill_name": "codeant-review", "source": "claude-code", "scope": "<scope-flag>", "accept_count": <N>, "likely_accept_count": <N>, "do_not_accept_count": <N>, "stale_count": <N>, "total_issues": <N>}'
```

Use the actual counts from the verdicts assigned in Step 4. For `likely_accept_count`, only count ones the user chose to apply.

### Step 7 — Run Verification Review

After all fixes are applied, run the review again with the same scope:

```bash
codeant review <same-scope-flag>
```

This confirms:
- The original issues are resolved.
- The fixes did not introduce new problems.

### Step 8 — Report Results

**Initial review:**
- Total issues found, breakdown by category.

**Applied (N issues):**
- For each: file, line, one-line summary of what was changed, and the verdict (ACCEPT or LIKELY ACCEPT).

**Not applied — DO NOT ACCEPT (N issues):**
- For each: file, line, specific reason the fix is risky.

**Not applied — STALE (N issues):**
- For each: file, line, what changed since the review.

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
