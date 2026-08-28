---
name: codeant-scans-sast
description: Fetch the top 10 SAST scan findings on the current branch, validate each against the source, and apply safe minimal fixes
---

Fetch the top 10 SAST (Static Application Security Testing) findings from the latest scan on the current branch, validate each against the current source code, and apply only safe, minimal fixes.

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
codeant track --event "skill_invoked" --props '{"skill_name": "codeant-scans-sast", "source": "agent-skills", "repo": "<REPO>", "branch": "<BRANCH>", "severity": "<SEVERITY>"}'
```

### Step 2 — Fetch scan findings

Run the following command (omit `--severity` when the user chose "all"):

```bash
codeant scans results \
  --repo <owner/repo> \
  --branch <current-branch> \
  --types sast \
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

If `findings` is an empty array after a successful call, tell the user: "No SAST findings matching your filters on branch `<branch>`." and stop.

### Step 3 — Categorize findings

Each finding object contains: `severity`, `category`, `file_path`, `line_number`, `check_id`, `check_name`, `message`.

Group findings by `file_path` to minimize re-reading. For each finding:

1. Read `file_path` with **30 lines above and 30 lines below** `line_number` for full context.
2. Read `message` and `check_name` carefully to understand:
   - **What is the problem?** — What the check says is wrong.
   - **What is the fix?** — What the code should look like after the fix.

### Step 4 — Analyze each finding and assign a verdict

For each finding, determine whether it can be safely fixed. Assign one of four verdicts:

**ACCEPT — Safe to apply.**
Assign this when ALL of these are true:
- The issue described in `message` is still present at `line_number` in the current code
- The fix is syntactically valid and all variables/imports are in scope
- The change does NOT alter the function's return type, signature, or public API
- The change does NOT remove or weaken existing error handling
- The change does NOT affect behavior for inputs that were previously handled correctly
- The fix is localized — it only touches the lines relevant to the issue

**LIKELY ACCEPT — Looks correct, but verify callers.**
Assign this when:
- The fix is logically sound and addresses a real issue
- BUT it changes behavior in a way that callers or tests might depend on

**DO NOT ACCEPT — Could break things.**
Assign this when ANY of these are true:
- The fix changes a function's return type or public interface
- The fix removes existing error handling or fallback logic
- The fix restructures control flow beyond what the issue asks for
- The fix introduces a dependency or import that doesn't exist in the project
- The fix looks like a refactor disguised as a fix
- You cannot confidently determine what the fix should be

**STALE — Code has changed since the scan.**
Assign this when:
- The code at `line_number` no longer matches what the finding describes
- The file has been renamed or deleted

For each ACCEPT or LIKELY ACCEPT finding, draft the minimal fix that addresses the issue. Keep fixes as small as possible — add only what is needed to resolve the specific problem.

### Step 5 — Present the summary with verdicts

Before making any changes, present a clear summary:

- **Repo**: `owner/repo`
- **Branch**: `<branch>`
- **Scan type**: SAST
- **Total findings fetched**: X (up to 10)

Then list every finding grouped by verdict:

**ACCEPT — Safe to apply (N):**
For each, show:
- File path and line number
- Severity and check name
- One-line summary of the issue
- One-line explanation of why this fix is safe
- The actual code change (before → after)

**LIKELY ACCEPT — Verify callers (N):**
For each, show:
- File path and line number
- Severity and check name
- What the fix changes and why it's probably correct
- What could break: specifically which callers or behaviors to verify
- The actual code change (before → after)

**DO NOT ACCEPT — Could break logic (N):**
For each, show:
- File path and line number
- Severity and check name
- Specific reason why this fix is risky
- What the user should do instead

**STALE — Code changed since scan (N):**
For each, show:
- File path and line number
- What the finding expected to find vs. what's actually there now

Then ask the user: "I will apply the N ACCEPT fixes now. For the LIKELY ACCEPT fixes, I recommend you review the callers first — want me to apply those too, or skip them for now?"

### Step 6 — Apply the fixes

After the user confirms:

- Apply all **ACCEPT** fixes.
- Apply **LIKELY ACCEPT** fixes only if the user said yes.
- Do **NOT** apply DO NOT ACCEPT or STALE fixes.
- Make the smallest possible change that addresses each finding.
- If the fix requires adding an import, add it.
- If multiple findings refer to the same file, apply all fixes to that file before moving to the next, being careful that fixes don't conflict with each other.

### Step 6b — Track results

After applying fixes, report the outcome:

```bash
codeant track --event "suggestions_applied" --props '{"skill_name": "codeant-scans-sast", "source": "agent-skills", "repo": "<REPO>", "branch": "<BRANCH>", "accept_count": <N>, "likely_accept_count": <N>, "do_not_accept_count": <N>, "stale_count": <N>, "total_findings": <N>}'
```

Use the actual counts from the verdicts assigned in Step 4. For `likely_accept_count`, only count ones the user chose to apply.

### Step 7 — Report results

Present a final report:

**Applied (N findings):**
- For each: file, line, one-line summary of what was changed, and the verdict (ACCEPT or LIKELY ACCEPT).

**Not applied — DO NOT ACCEPT (N findings):**
- For each: file, line, specific reason the fix is risky.

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

- If the user says **yes** (or specifies which files to include), stage the selected files, create a commit with a clear message (e.g., "Fix SAST findings via CodeAnt scan on branch <branch>"), and push to the current branch.
- If the user says **no** or wants to review first, do nothing — leave the changes uncommitted.
- If the user specifies a subset of files, only stage and commit those files.

### Important Rules
- Do **NOT** modify files not referenced in the findings.
- Do **NOT** apply a fix you cannot verify is safe. It is always better to skip and explain than to break the code.
- Do **NOT** batch-apply fixes blindly. Validate each one individually.
- If two findings conflict (e.g., one says add a check, another says remove it), flag both and ask the user.
- Keep fixes **minimal**. A fix for a missing null check should add the null check — not restructure the function.
