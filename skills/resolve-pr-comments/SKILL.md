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

### Step 4 — Analyze Each Comment and Assign a Verdict

For each inline comment (grouped by file to minimize re-reading), do the following:

#### 4a. Read and Understand the Context

1. Read the file at the comment's `line` number, with **30 lines above and 30 lines below** for full context.
2. Read the comment `body` carefully. Identify:
   - **What is the problem?** — What the reviewer says is wrong.
   - **Is there a code suggestion?** — Look for fenced code blocks (` ```suggestion `, ` ```python `, ` ```js `, etc.) or inline code that represents a replacement.
   - **What is the intent?** — What behavior should the code have after the fix.

#### 4b. Validate the Suggestion

For each comment, run through these checks:

1. **Check that the code the comment references still exists.** The file may have changed since the review. If the code at the referenced line no longer matches what the comment describes, mark as `STALE`.

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

3. **If no code suggestion is present:**
   - Analyze the comment to understand the requested change.
   - Draft a **minimal fix** — change only what is necessary to address the concern.
   - Do NOT refactor surrounding code, rename variables, or "improve" things beyond the scope of the comment.
   - Run the same validation checks as above on your drafted fix.

#### 4c. Assign a Verdict to Each Comment

Based on the validation, assign one of these verdicts to every comment:

**ACCEPT — Safe to apply, you should accept this.**
Assign this when ALL of these are true:
- The suggestion fixes a genuine bug, security issue, or correctness problem
- The suggested code is syntactically valid and all variables/imports are in scope
- The change does NOT alter the function's return type, signature, or public API
- The change does NOT remove or weaken existing error handling
- The change does NOT affect behavior for inputs that were previously handled correctly
- The fix is localized — it only touches the lines relevant to the issue

**LIKELY ACCEPT — Looks correct, but verify the callers.**
Assign this when:
- The suggestion is logically sound and fixes a real issue
- BUT it changes behavior in a way that callers or tests might depend on (e.g., a function now returns an error where it previously returned nil, or a previously permissive validation now rejects some inputs)
- The fix itself is correct, but you cannot guarantee no downstream breakage without checking callers


**DO NOT ACCEPT — This could break things.**
Assign this when ANY of these are true:
- The suggestion changes a function's return type or public interface
- The suggestion removes existing error handling or fallback logic
- The suggestion restructures control flow (reordering if/else, changing loop logic) beyond what the comment asks for
- The suggestion introduces a dependency or import that doesn't exist in the project
- The suggestion looks like a refactor disguised as a fix — it changes more than necessary
- You cannot understand what the suggestion does or why it's better

**STALE — Code has changed since the review.**
Assign this when:
- The code at the referenced line no longer matches what the comment describes
- The file has been renamed or deleted

### Step 5 — Present the Summary with Verdicts

Before making any changes, present a clear summary to the user:

- **PR**: #N — "title" (link to PR)
- **Total unresolved CodeAnt comments**: X

Then list every comment grouped by verdict:

**ACCEPT — Safe to apply (N):**
For each, show:
- File path and line number
- One-line summary of the issue
- One-line explanation of why this is safe: what exactly the fix does and why it cannot break anything
- The actual code change (before → after) so the user can see it

**LIKELY ACCEPT — Verify callers (N):**
For each, show:
- File path and line number
- One-line summary of the issue
- What the fix changes and why it's probably correct
- What could break: specifically which callers, tests, or behaviors to check
- The actual code change (before → after)

**DO NOT ACCEPT — Could break logic (N):**
For each, show:
- File path and line number
- One-line summary of what the comment asks for
- Specific reason why the suggestion is risky — what exactly could break
- What the user should do instead (e.g., "review manually", "check with the team", "test this path first")

**STALE — Code changed since review (N):**
For each, show:
- File path and line number
- What the comment expected to find vs. what's actually there now

**Requires manual review (N):**
- PR-level comments that don't point to specific code — summarize each one.

Then ask the user: "I will apply the N ACCEPT fixes now. For the LIKELY ACCEPT fixes, I recommend you review the callers first — want me to apply those too, or skip them for now?"

### Step 6 — Apply the Fixes

After the user confirms:

- Apply all **ACCEPT** fixes.
- Apply **LIKELY ACCEPT** fixes only if the user said yes.
- Do **NOT** apply DO NOT ACCEPT or STALE fixes.
- Make the smallest possible change that addresses each comment.
- If the fix requires adding an import, add it.
- If multiple comments refer to the same file, apply all fixes to that file before moving to the next file, being careful that fixes don't conflict with each other.

### Step 7 — Report Results

Present a final report:

**Applied (N comments):**
- For each: file, line, one-line summary of what was changed, and the verdict (ACCEPT or LIKELY ACCEPT).

**Not applied — DO NOT ACCEPT (N comments):**
- For each: file, line, specific reason the suggestion is risky.

**Not applied — STALE (N comments):**
- For each: file, line, what changed since the review.

**Requires manual review (N comments):**
- For each: the PR-level comment body summarized.

### Important Rules

- Do **NOT** commit or push changes. Let the user review diffs and decide.
- Do **NOT** modify files that are not referenced in the comments.
- Do **NOT** apply a suggestion if you cannot verify it is safe. It is always better to skip and explain than to break the code.
- Do **NOT** batch-apply suggestions blindly. Validate each one individually.
- If two comments conflict (e.g., one says add a check, another says remove it), flag both and ask the user.
- Keep fixes **minimal**. A fix for a missing null check should add the null check — not restructure the function.
