---
name: codeant-scans-dead-code
description: Fetch the top 10 dead code scan findings on the current branch, validate each against the source, and apply safe minimal fixes
---

Fetch the top 10 Dead Code findings from the latest scan on the current branch, validate each against the current source code, and apply only safe, minimal fixes.

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
codeant track --event "skill_invoked" --props '{"skill_name": "codeant-scans-dead-code", "source": "cursor", "repo": "<REPO>", "branch": "<BRANCH>", "severity": "<SEVERITY>"}'
```

### Step 2 — Fetch scan findings

Run the following command (omit `--severity` when the user chose "all"):

```bash
codeant scans results \
  --repo <owner/repo> \
  --branch <current-branch> \
  --types dead_code \
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

If `findings` is an empty array after a successful call, tell the user: "No Dead Code findings matching your filters on branch `<branch>`." and stop.

### Step 3 — Categorize findings

Each finding object contains: `severity`, `category`, `file_path`, `line_number`, `check_id`, `check_name`, `message`.

Group findings by `file_path` to minimize re-reading. For each finding:

1. Read `file_path` with **30 lines above and 30 lines below** `line_number` for full context.
2. Read `message` and `check_name` carefully to understand:
   - **What is the problem?** — What code is flagged as unreachable, unused, or dead.
   - **What is the fix?** — Typically: remove the dead code entirely.

### Step 4 — Analyze each finding and assign a verdict

For each finding, determine whether it can be safely fixed. Assign one of four verdicts:

**ACCEPT — Safe to apply.**
Assign this when ALL of these are true:
- The dead code described in `message` is still present at `line_number`
- The code is definitively unreachable or unused (not just conditionally unused)
- Removing it does NOT alter any exported/public interface
- Removing it does NOT affect any test fixtures, mocks, or debug utilities
- The fix is localized — it removes only the flagged lines

**LIKELY ACCEPT — Looks correct, but verify usage.**
Assign this when:
- The code appears unused within this file but may be called dynamically (e.g., via reflection, `eval`, dynamic import) or exported and used by external consumers not visible in this repo

**DO NOT ACCEPT — Could break things.**
Assign this when ANY of these are true:
- The flagged code is part of a public API, exported symbol, or interface
- The code might be used by consumers outside this repository
- Removing it would alter the module's exported surface
- The code appears in a test helper, stub, or fixture used indirectly

**STALE — Code has changed since the scan.**
Assign this when:
- The code at `line_number` no longer matches what the finding describes
- The file has been renamed or deleted

For each ACCEPT or LIKELY ACCEPT finding, draft the minimal fix: remove only the flagged dead code.

### Step 5 — Present the summary with verdicts

Before making any changes, present a clear summary:

- **Repo**: `owner/repo`
- **Branch**: `<branch>`
- **Scan type**: Dead Code
- **Total findings fetched**: X (up to 10)

Then list every finding grouped by verdict:

**ACCEPT — Safe to apply (N):**
For each, show:
- File path and line number
- Severity and check name
- One-line summary of what dead code was found
- One-line explanation of why removal is safe
- The actual code change (the lines to be removed)

**LIKELY ACCEPT — Verify usage (N):**
For each, show:
- File path and line number
- Severity and check name
- What would be removed and why it's probably safe
- What to check: dynamic usage, external consumers
- The actual code change

**DO NOT ACCEPT — Could break things (N):**
For each, show:
- File path and line number
- Severity and check name
- Specific reason why removal is risky
- What the user should do instead

**STALE — Code changed since scan (N):**
For each, show:
- File path and line number
- What the finding expected to find vs. what's actually there now

Then ask the user: "I will apply the N ACCEPT fixes now. For the LIKELY ACCEPT fixes, I recommend you verify external usage first — want me to apply those too, or skip them for now?"

### Step 6 — Apply the fixes

After the user confirms:

- Apply all **ACCEPT** fixes.
- Apply **LIKELY ACCEPT** fixes only if the user said yes.
- Do **NOT** apply DO NOT ACCEPT or STALE fixes.
- Remove only the flagged dead code — do not clean up surrounding code beyond what is needed.
- If multiple findings refer to the same file, apply all fixes to that file before moving to the next.

### Step 6b — Track results

After applying fixes, report the outcome:

```bash
codeant track --event "suggestions_applied" --props '{"skill_name": "codeant-scans-dead-code", "source": "cursor", "repo": "<REPO>", "branch": "<BRANCH>", "accept_count": <N>, "likely_accept_count": <N>, "do_not_accept_count": <N>, "stale_count": <N>, "total_findings": <N>}'
```

Use the actual counts from the verdicts assigned in Step 4. For `likely_accept_count`, only count ones the user chose to apply.

### Step 7 — Report results

Present a final report:

**Applied (N findings):**
- For each: file, line, one-line summary of what was removed, and the verdict (ACCEPT or LIKELY ACCEPT).

**Not applied — DO NOT ACCEPT (N findings):**
- For each: file, line, specific reason removal is risky.

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

- If the user says **yes** (or specifies which files to include), stage the selected files, create a commit with a clear message (e.g., "Remove dead code via CodeAnt scan on branch <branch>"), and push to the current branch.
- If the user says **no** or wants to review first, do nothing — leave the changes uncommitted.
- If the user specifies a subset of files, only stage and commit those files.

### Important Rules
- Do **NOT** modify files not referenced in the findings.
- Do **NOT** remove code you are not certain is dead. It is always better to skip and explain than to delete live code.
- Do **NOT** batch-apply removals blindly. Validate each one individually.
- Keep fixes **minimal**. Remove only the dead code — do not reorganize, rename, or refactor anything else.
