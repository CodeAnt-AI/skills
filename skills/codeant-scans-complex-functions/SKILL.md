---
name: codeant-scans-complex-functions
description: Fetch the top 10 complex functions scan findings on the current branch, validate each against the source, and apply safe minimal fixes
---

Fetch the top 10 Complex Functions findings from the latest scan on the current branch, validate each against the current source code, and apply only safe, minimal refactors to reduce complexity.

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
codeant track --event "skill_invoked" --props '{"skill_name": "codeant-scans-complex-functions", "source": "claude-code", "repo": "<REPO>", "branch": "<BRANCH>", "severity": "<SEVERITY>"}'
```

### Step 2 — Fetch scan findings

Run the following command (omit `--severity` when the user chose "all"):

```bash
codeant scans results \
  --repo <owner/repo> \
  --branch <current-branch> \
  --types complex_functions \
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

If `findings` is an empty array after a successful call, tell the user: "No Complex Functions findings matching your filters on branch `<branch>`." and stop.

### Step 3 — Categorize findings

Each finding object contains: `severity`, `category`, `file_path`, `line_number`, `check_id`, `check_name`, `message`.

Group findings by `file_path` to minimize re-reading. For each finding:

1. Read `file_path` with **30 lines above and 30 lines below** `line_number` for full context. For complex function findings you may need to read more lines to see the full function body — expand to 60 lines above and below if needed.
2. Read `message` and `check_name` carefully to understand:
   - **What metric is flagged?** — Cyclomatic complexity, cognitive complexity, nesting depth, function length, number of parameters, etc.
   - **What is the targeted complexity?** — The value that exceeds the threshold.
   - **What is a safe refactor?** — Extract a well-named helper function, simplify a conditional chain, reduce parameter count via a config object, etc.

### Step 4 — Analyze each finding and assign a verdict

For each finding, determine whether it can be safely refactored. Assign one of four verdicts:

**ACCEPT — Safe to apply.**
Assign this when ALL of these are true:
- The complexity issue described in `message` is still present at `line_number`
- A clear, minimal refactor exists (e.g., extract one inner block into a helper) that measurably reduces complexity
- The refactor does NOT change the function's public signature, return type, or observable behavior
- The extracted helper is self-contained — it does not require access to many external variables from the outer scope
- The fix is localized enough to be done safely without seeing the full call graph

**LIKELY ACCEPT — Looks correct, but verify behavior.**
Assign this when:
- A safe refactor is identifiable
- BUT the function is called in many places or has subtle side effects that make it risky to change without running tests

**DO NOT ACCEPT — Refactor is too broad or risky.**
Assign this when ANY of these are true:
- Reducing complexity requires restructuring the entire function or splitting it across multiple files
- The function's complexity stems from essential domain logic that cannot be safely extracted without deep understanding of all callers
- The function is part of a public API and extracting helpers would change observable behavior
- No safe, minimal refactor can be identified from the available context

**STALE — Code has changed since the scan.**
Assign this when:
- The code at `line_number` no longer matches what the finding describes
- The file has been renamed or deleted

For each ACCEPT or LIKELY ACCEPT finding, draft the minimal refactor. Prefer extracting a single well-named helper function over larger restructuring.

### Step 5 — Present the summary with verdicts

Before making any changes, present a clear summary:

- **Repo**: `owner/repo`
- **Branch**: `<branch>`
- **Scan type**: Complex Functions
- **Total findings fetched**: X (up to 10)

Then list every finding grouped by verdict:

**ACCEPT — Safe to apply (N):**
For each, show:
- File path and line number
- Severity and check name
- One-line summary of the complexity issue (metric + value)
- One-line explanation of the refactor approach
- The actual code change (before → after, showing the extracted helper and the simplified call site)

**LIKELY ACCEPT — Verify behavior (N):**
For each, show:
- File path and line number
- Severity and check name
- The proposed refactor
- What to verify: call sites, tests, side effects
- The actual code change (before → after)

**DO NOT ACCEPT — Refactor too broad (N):**
For each, show:
- File path and line number
- Severity and check name
- Specific reason why the refactor is risky or too large
- What the user should do instead

**STALE — Code changed since scan (N):**
For each, show:
- File path and line number
- What the finding expected to find vs. what's actually there now

Then ask the user: "I will apply the N ACCEPT refactors now. For the LIKELY ACCEPT ones, I recommend running tests after — want me to apply those too, or skip them for now?"

### Step 6 — Apply the fixes

After the user confirms:

- Apply all **ACCEPT** fixes.
- Apply **LIKELY ACCEPT** fixes only if the user said yes.
- Do **NOT** apply DO NOT ACCEPT or STALE fixes.
- Extract the minimal helper needed — do not reorganize the rest of the file.
- If multiple findings refer to the same file, apply all fixes to that file before moving to the next, ensuring extracted helpers don't conflict.

### Step 6b — Track results

After applying fixes, report the outcome:

```bash
codeant track --event "suggestions_applied" --props '{"skill_name": "codeant-scans-complex-functions", "source": "claude-code", "repo": "<REPO>", "branch": "<BRANCH>", "accept_count": <N>, "likely_accept_count": <N>, "do_not_accept_count": <N>, "stale_count": <N>, "total_findings": <N>}'
```

Use the actual counts from the verdicts assigned in Step 4. For `likely_accept_count`, only count ones the user chose to apply.

### Step 7 — Report results

Present a final report:

**Applied (N findings):**
- For each: file, line, one-line summary of what was refactored, and the verdict (ACCEPT or LIKELY ACCEPT).

**Not applied — DO NOT ACCEPT (N findings):**
- For each: file, line, specific reason the refactor is too broad or risky.

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

- If the user says **yes** (or specifies which files to include), stage the selected files, create a commit with a clear message (e.g., "Reduce function complexity via CodeAnt scan on branch <branch>"), and push to the current branch.
- If the user says **no** or wants to review first, do nothing — leave the changes uncommitted.
- If the user specifies a subset of files, only stage and commit those files.

### Important Rules
- Do **NOT** modify files not referenced in the findings.
- Do **NOT** apply a refactor you cannot verify preserves behavior. It is always better to skip and explain than to break the code.
- Do **NOT** batch-apply refactors blindly. Validate each one individually.
- Keep fixes **minimal**. Extract only what is needed to reduce the flagged metric — do not reorganize the entire function or file.
