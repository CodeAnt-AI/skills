---
name: codeant-scans-docstring
description: Fetch the top 10 docstring scan findings on the current branch, validate each against the source, and apply safe minimal fixes
---

Fetch the top 10 Docstring findings from the latest scan on the current branch, validate each against the current source code, and add or fix documentation where it is missing or incorrect.

## Instructions

### Step 0 — Ensure codeant-cli is up to date

Before doing anything else, check that the `codeant` CLI is on the latest version:

```bash
npm view codeant-cli version
```

Compare this with the installed version:

```bash
codeant --version
```

If the installed version is older than the latest published version, update it:

```bash
npm install -g codeant-cli@latest
```

If the update fails (e.g., permission error), warn the user and continue — a slightly outdated CLI is better than blocking the entire workflow.

### Step 1 — Gather context from git

Get the current branch and repo:

```bash
git rev-parse --abbrev-ref HEAD
git remote get-url origin
```

Parse the remote URL to `owner/repo` format:
- HTTPS: `https://github.com/owner/repo.git` → `owner/repo`
- SSH: `git@github.com:owner/repo.git` → `owner/repo`

If the repo cannot be determined from the remote, ask the user to provide it explicitly as `owner/repo`.

Use the **AskUserQuestion** tool to ask the user for the severity filter:

> "Which severity levels should I include? Options: `critical`, `high`, `medium`, `low`, `info`, `unknown`, or `all` (default). You can specify multiple comma-separated values, e.g. `critical,high`."

Wait for the user's response before continuing. Default to `all` if the user does not answer.

### Step 1b — Track skill invocation

```bash
codeant track --event "skill_invoked" --props '{"skill_name": "codeant-scans-docstring", "source": "agent-skills", "repo": "<REPO>", "branch": "<BRANCH>", "severity": "<SEVERITY>"}'
```

### Step 2 — Fetch scan findings

Run the following command (omit `--severity` when the user chose "all"):

```bash
codeant scans results \
  --repo <owner/repo> \
  --branch <current-branch> \
  --types docstring \
  --limit 10 \
  --filter-dismissed \
  --no-false-positives \
  --quiet \
  --format json
```

When the user specified a severity, add:

```bash
  --severity <user-severity>
```

Handle exit codes:
- `0` — success; parse the `findings` array from the JSON output
- `2` — no scans found for this branch; tell the user and stop
- `4` — network error; tell the user and stop
- Other non-zero — surface the error message from stderr and stop

If `findings` is an empty array after a successful call, tell the user: "No Docstring findings matching your filters on branch `<branch>`." and stop.

### Step 3 — Categorize findings

Each finding object contains: `severity`, `category`, `file_path`, `line_number`, `check_id`, `check_name`, `message`.

Group findings by `file_path` to minimize re-reading. For each finding:

1. Read `file_path` with **30 lines above and 30 lines below** `line_number` for full context.
2. Read `message` and `check_name` carefully to understand:
   - **What is the problem?** — Missing docstring, incomplete docstring, wrong format, undocumented parameters, etc.
   - **What is the fix?** — Add or correct the docstring to match the codebase's documentation style.
3. Identify the docstring convention used in the file (Google style, NumPy style, JSDoc, reStructuredText, etc.) by examining existing docstrings in the same file.

### Step 4 — Analyze each finding and assign a verdict

For each finding, determine whether it can be safely fixed. Assign one of four verdicts:

**ACCEPT — Safe to apply.**
Assign this when ALL of these are true:
- The missing or incorrect docstring described in `message` is still absent/wrong at `line_number`
- The function/class/method signature is fully readable from context
- You can write an accurate docstring that correctly describes the parameters, return values, and behavior
- Adding the docstring does NOT change any runtime behavior

**LIKELY ACCEPT — Looks correct, but verify accuracy.**
Assign this when:
- The function exists and a docstring can be drafted
- BUT the function's behavior is complex or non-obvious enough that the docstring might be inaccurate without deeper analysis

**DO NOT ACCEPT — Cannot write an accurate docstring.**
Assign this when ANY of these are true:
- The function body is too complex or opaque to summarize accurately from the 30-line context window
- The function has a large number of parameters whose types and purpose are unclear
- Writing the docstring would require understanding behavior spread across many files

**STALE — Code has changed since the scan.**
Assign this when:
- The code at `line_number` no longer matches what the finding describes (e.g., function was renamed or removed)
- The file has been renamed or deleted

For each ACCEPT or LIKELY ACCEPT finding, draft the docstring using the same style/format as other docstrings in the file.

### Step 5 — Present the summary with verdicts

Before making any changes, present a clear summary:

- **Repo**: `owner/repo`
- **Branch**: `<branch>`
- **Scan type**: Docstring
- **Total findings fetched**: X (up to 10)

Then list every finding grouped by verdict:

**ACCEPT — Safe to apply (N):**
For each, show:
- File path and line number
- Severity and check name
- One-line summary of what documentation is missing/wrong
- The drafted docstring to be added

**LIKELY ACCEPT — Verify accuracy (N):**
For each, show:
- File path and line number
- Severity and check name
- The drafted docstring
- What to verify: parameter types, return behavior, edge cases

**DO NOT ACCEPT — Cannot write accurately (N):**
For each, show:
- File path and line number
- Severity and check name
- Specific reason why an accurate docstring cannot be drafted from context

**STALE — Code changed since scan (N):**
For each, show:
- File path and line number
- What the finding expected to find vs. what's actually there now

Then ask the user: "I will add the N ACCEPT docstrings now. For the LIKELY ACCEPT ones, I recommend you review them for accuracy — want me to add those too, or skip them for now?"

### Step 6 — Apply the fixes

After the user confirms:

- Apply all **ACCEPT** fixes.
- Apply **LIKELY ACCEPT** fixes only if the user said yes.
- Do **NOT** apply DO NOT ACCEPT or STALE fixes.
- Insert the docstring immediately after the function/class/method definition line, matching the indentation and style of the file.
- If multiple findings refer to the same file, apply all fixes to that file before moving to the next.

### Step 6b — Track results

After applying fixes, report the outcome:

```bash
codeant track --event "suggestions_applied" --props '{"skill_name": "codeant-scans-docstring", "source": "agent-skills", "repo": "<REPO>", "branch": "<BRANCH>", "accept_count": <N>, "likely_accept_count": <N>, "do_not_accept_count": <N>, "stale_count": <N>, "total_findings": <N>}'
```

Use the actual counts from the verdicts assigned in Step 4. For `likely_accept_count`, only count ones the user chose to apply.

### Step 7 — Report results

Present a final report:

**Applied (N findings):**
- For each: file, line, one-line summary of what documentation was added/fixed, and the verdict (ACCEPT or LIKELY ACCEPT).

**Not applied — DO NOT ACCEPT (N findings):**
- For each: file, line, specific reason an accurate docstring could not be drafted.

**Not applied — STALE (N findings):**
- For each: file, line, what changed since the scan.

### Step 8 — Offer to commit and push

After presenting the final report, check which files were modified:

```bash
git status --short
```

List the changed files to the user and ask:

"These are the files that were changed:
- `<file1>`
- `<file2>`
- ...

Would you like me to commit and push these changes to the current branch? You can also tell me to commit only specific files."

- If the user says **yes** (or specifies which files to include), stage the selected files, create a commit with a clear message (e.g., "Add missing docstrings via CodeAnt scan on branch <branch>"), and push to the current branch.
- If the user says **no** or wants to review first, do nothing — leave the changes uncommitted.
- If the user specifies a subset of files, only stage and commit those files.

### Important Rules
- Do **NOT** modify files not referenced in the findings.
- Do **NOT** add a docstring if you cannot write it accurately. A wrong docstring is worse than no docstring.
- Do **NOT** batch-apply docstrings blindly. Validate each one individually.
- Keep fixes **minimal**. Add only the docstring — do not rename parameters, reformat the function, or change any runtime code.
- Match the docstring style already used in the file.
