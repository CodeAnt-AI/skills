---
name: codeant-resolve-quality-gates
description: Find the pull request for the current branch, locate the CodeAnt CI/CD Quality Gate comment, parse the failures (SAST, Secrets, Duplicate Code, SCA, IAC), and apply safe, minimal fixes for each one
---

Find the pull request for the current branch, locate the CodeAnt CI/CD Quality Gate comment, parse the listed failures, validate each finding against the current code, and apply safe, minimal fixes that do not break existing logic.

## Instructions

### Step 0 — Ensure codeant-cli is Up to Date

Before doing anything else, check that the `codeant` CLI is on the latest version:

```bash
npm view codeant-cli version
```

Compare with the installed version:

```bash
codeant --version
```

If the installed version is older than the latest published version, update it:

```bash
npm install -g codeant-cli@latest
```

If the update fails (e.g., permission error), warn the user and continue.

### Step 1 — Find the Pull Request

The goal is to identify the correct PR. Use the following logic in order:

**If the user provides a PR number** (e.g., `/codeant-resolve-quality-gates 42`):

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

### Step 1b — Track Skill Invocation

Report that this skill was invoked:

```bash
codeant track --event "skill_invoked" --props '{"skill_name": "codeant-resolve-quality-gates", "source": "cursor", "pr_number": <N>, "pr_url": "<PR_URL>"}'
```

Where `<PR_URL>` is the `url` field from the PR object found in Step 1.

### Step 2 — Locate the Quality Gate Comment

Fetch every CodeAnt comment on the PR (including general/issue comments — the quality gate result is posted as a general PR comment, not an inline review comment):

```bash
codeant pr comments --pr-number <N> --codeant-generated true
```

From the returned array, find the **most recent** comment whose `body` contains the marker string `CodeAnt Quality Gate Results`. This is the quality gate summary comment posted by CodeAnt CI/CD.

- If **no** such comment exists, tell the user: "No CodeAnt Quality Gate comment found on PR #N. Either CI hasn't run yet or quality gates are disabled." and stop.
- If the comment's overall status line shows `:white_check_mark: Overall Status: PASSED`, tell the user "Quality gates passed on PR #N — nothing to fix." and stop.

Remember the comment's `id` field — you will need it in Step 8 to mark the conversation resolved (if your provider supports it).

### Step 3 — Parse the Comment Body

The comment is markdown. Parse it into structured failures.

The body has these sections, in order:

1. **Header:** `# :checkered_flag: CodeAnt Quality Gate Results`
2. **Commit / Scan time:** `**Commit:** <hash>` and `**Scan Time:** ...`
3. **Overall status:** one of `:white_check_mark: Overall Status: PASSED`, `:x: Overall Status: FAILED`, `:warning: Overall Status: ERROR`
4. **Quality Gate Details** table — one row per enabled gate. Columns: `| Quality Gate | Status | Details |`. The gates that may appear are `**Secrets**`, `**Duplicate Code**`, `**SAST**`, `**IAC**`, `**SCA (Dependencies)**`. Each row's status is `:white_check_mark: PASSED`, `:x: FAILED`, or `:warning: ERROR`.
5. `[View Full Results](<dashboard-url>)` link.
6. **Fix-in-IDE buttons** (only when there are failed gates): a single line `[Fix in Cursor](<url>) | [Fix in VSCode Claude](<url>)` followed by a hint. Ignore these links — you are running the fix directly.
7. **Failure detail block** (only on non-bitbucket platforms, and only when there are failed gates): a top-level `<details><summary>View Failure Result</summary>` that contains nested `<details>` blocks for each failed gate.

From the **Quality Gate Details** table, collect the list of failed gates (rows whose Status is `:x: FAILED`).

From the **Failure detail block**, extract per-gate findings:

- **Secrets** — table `| File | Line | Type | Confidence |`. Each row is one secret finding.
- **Duplicate Code** — list items of the form `` - `<filename>` <window> ↔ `<filename>` <window> ... (<N> copies)``. Each list item is one duplicate-group finding.
- **SAST** — table `| Severity | File | Line | Rule | Message |`. Each row is one security finding.

**Important:** The visible failure detail block only contains rows for **Secrets**, **Duplicate Code**, and **SAST**. The comment intentionally omits detailed rows for **SCA** and **IAC** — the table only shows aggregated counts (e.g., `Rating B: 3 vulnerabilities (1 high, 2 medium)`).

For **SCA / IAC** failures, do NOT try to fabricate findings. Instead, run the equivalent local scan to get the actionable rows:

- **SCA failed:** `codeant security-analysis --uncommitted` (or rely on dependency manifest changes). Report the vulnerabilities and recommend version bumps in package files.
- **IAC failed:** point the user at the View Full Results link in the comment and offer to open the relevant IAC files based on the listed misconfiguration types.

If a local rescan is needed, ask the user first — don't run it silently.

### Step 4 — Analyze Each Finding and Assign a Verdict

For each parsed finding, do the following:

#### 4a. Read and Understand the Context

1. Open the file at the finding's path and line number. Read with **30 lines above and 30 lines below** for full context.
2. Identify what the finding describes:
   - **SAST:** the `Rule` (e.g., `python.django.security.injection.sql.sql-injection-using-format-string`) plus `Message` describes the vulnerability and intended fix.
   - **Secrets:** the `Type` (e.g., `Generic API Key`, `AWS Access Key`) tells you what was leaked. The fix is to remove the literal value and replace with an environment variable / secrets-manager lookup. **Do NOT commit the secret to git history** — if it's already in a previous commit, remind the user it must also be rotated.
   - **Duplicate Code:** each finding lists two or more `` `file` line-range `` blocks that share the same logic. The fix is usually to extract the duplicated logic into a shared helper.

#### 4b. Validate the Fix

For each finding, run through these checks:

1. **Check that the code still exists at the referenced line.** The file may have changed since the CI scan that produced this comment. If the code no longer matches what the finding describes, mark as `STALE`.

2. **Draft a minimal fix.** No suggestion is embedded in the quality gate comment — you must design the fix yourself based on the rule/type/window plus the surrounding code. Change only what is necessary to address the finding.

3. **Validate the drafted fix:**
   - Does it reference variables / functions / imports that exist in scope (add the import if missing)?
   - Does it preserve the function's signature, return type, and public API?
   - Does it preserve existing error handling and null checks?
   - Does it preserve behavior for inputs that were previously handled correctly?
   - Is the change localized to the lines relevant to the finding?

#### 4c. Assign a Verdict to Each Finding

Based on the validation, assign one of these verdicts:

**ACCEPT — Safe to apply, you should accept this.**
Assign this when ALL of these are true:
- The fix addresses a genuine security / correctness / duplication issue
- The drafted code is syntactically valid and all variables / imports are in scope
- The change does NOT alter the function's return type, signature, or public API
- The change does NOT remove or weaken existing error handling
- The change does NOT affect behavior for inputs that were previously handled correctly
- The fix is localized to the lines relevant to the issue

**LIKELY ACCEPT — Looks correct, but verify the callers.**
Assign this when:
- The fix is logically sound and addresses a real issue
- BUT it changes behavior in a way that callers or tests might depend on (e.g., a previously permissive validation now rejects some inputs; a previously string-concatenated SQL is now parameterized which may break test fixtures that exercise the raw query path)
- The fix itself is correct, but you cannot guarantee no downstream breakage without checking callers

**DO NOT ACCEPT — This could break things.**
Assign this when ANY of these are true:
- The fix would change a function's return type or public interface
- The fix would remove existing error handling or fallback logic
- The fix would restructure control flow beyond what the finding asks for
- The fix would introduce a dependency or import that doesn't exist in the project
- The fix looks like a refactor disguised as a security fix — it changes more than necessary
- For Duplicate Code findings: the only safe fix requires a broad, multi-file refactor (extract a shared module, change call sites) that cannot be done as a minimal patch
- For Secrets: the secret literal is structurally part of test data or a known-fake fixture, not a real leak

**STALE — Code has changed since the scan.**
Assign this when:
- The code at the referenced line no longer matches what the finding describes
- The file has been renamed or deleted

### Step 5 — Present the Summary with Verdicts

Before making any changes, present a clear summary to the user:

- **PR**: #N — "title" (link to PR)
- **Quality Gate status**: FAILED
- **Failed gates**: comma-separated list from the table (e.g., `SAST, Secrets, SCA`)
- **Total actionable findings parsed**: X

Then list every finding grouped by verdict. Within each verdict group, sub-group by gate type (SAST first, then Secrets, then Duplicate Code, then any SCA/IAC findings you collected via local rescan).

**ACCEPT — Safe to apply (N):**
For each, show:
- Gate (SAST / Secrets / Duplicate / SCA / IAC)
- File path and line number
- One-line summary of the issue (rule id + message for SAST, type for Secrets, etc.)
- One-line explanation of why this is safe
- The actual code change (before → after)

**LIKELY ACCEPT — Verify callers (N):**
For each, show:
- Gate
- File path and line number
- One-line summary
- What the fix changes and why it's probably correct
- What could break: specifically which callers, tests, or behaviors to check
- The actual code change (before → after)

**DO NOT ACCEPT — Could break logic (N):**
For each, show:
- Gate
- File path and line number
- One-line summary of the finding
- Specific reason the suggestion is risky
- What the user should do instead

**STALE — Code changed since scan (N):**
For each, show:
- Gate
- File path and line number
- What the comment expected vs. what's actually there now

**SCA / IAC failures (if any gate failed but findings could not be parsed from the comment):**
List the gates and tell the user: "The quality gate comment summarizes these but doesn't include row-level details. Want me to run `codeant security-analysis --uncommitted` (or open the relevant IAC files) to enumerate them?"

Then ask: "I will apply the N ACCEPT fixes now. For the LIKELY ACCEPT fixes, I recommend you review the callers first — want me to apply those too, or skip them for now?"

### Step 6 — Apply the Fixes

After the user confirms:

- Apply all **ACCEPT** fixes.
- Apply **LIKELY ACCEPT** fixes only if the user said yes.
- Do **NOT** apply DO NOT ACCEPT or STALE fixes.
- Make the smallest possible change that addresses each finding.
- If the fix requires adding an import, add it.
- If multiple findings refer to the same file, apply all fixes to that file before moving to the next file, being careful that fixes don't conflict with each other.

For **Secrets** specifically: replace the literal with an environment variable lookup (e.g., `os.getenv("FOO_API_KEY")`), and remind the user the leaked value must be rotated regardless — removing it from the working tree does not undo prior commits.

### Step 6b — Track Results

After applying fixes, report the outcome:

```bash
codeant track --event "suggestions_applied" --props '{"skill_name": "codeant-resolve-quality-gates", "source": "cursor", "pr_number": <N>, "pr_url": "<PR_URL>", "accept_count": <N>, "likely_accept_count": <N>, "do_not_accept_count": <N>, "stale_count": <N>, "total_findings": <N>, "failed_gates": "<comma-separated list>"}'
```

Use the actual counts from the verdicts assigned in Step 4. For `likely_accept_count`, only count ones the user chose to apply.

### Step 7 — Report Results

Present a final report:

**Applied (N findings):**
- For each: gate, file, line, one-line summary of what was changed, and the verdict (ACCEPT or LIKELY ACCEPT).

**Not applied — DO NOT ACCEPT (N findings):**
- For each: gate, file, line, specific reason.

**Not applied — STALE (N findings):**
- For each: gate, file, line, what changed since the scan.

**Remaining gates without row-level details (if any):**
- List SCA / IAC gates that failed but were not enumerated. Recommend the next step (local rescan, dashboard link, manifest update).

### Step 8 — Offer to Commit and Push

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

- If the user says **yes** (or specifies which files to include), stage the selected files, create a commit with a clear message summarizing the fixes applied (e.g., "Fix CodeAnt quality gate findings for PR #N"), and push to the current branch.
- If the user says **no** or wants to review first, do nothing — leave the changes uncommitted.
- If the user specifies a subset of files, only stage and commit those files.

Pushing the fix commit will trigger CI again, which will re-run the quality gates and update the comment on the PR. The user does not need to manually resolve anything on the PR — passing gates flip the overall status to PASSED on the next run.

### Important Rules

- Do **NOT** modify files that are not referenced in the parsed findings.
- Do **NOT** apply a fix if you cannot verify it is safe. It is always better to skip and explain than to break the code.
- Do **NOT** batch-apply fixes blindly. Validate each one individually.
- Keep fixes **minimal**. A SAST fix for a SQL injection should parameterize the query — not rewrite the function.
- For **Secrets**, never leave the literal in the working tree even commented out. Remind the user to rotate any leaked credential.
- For **Duplicate Code**, prefer DO NOT ACCEPT unless the duplication is tiny and the extraction is obvious — extracting a helper changes call sites and is rarely a "minimal" patch.
- Do **NOT** click or follow the Fix-in-IDE links in the comment — you are already operating inside the IDE; following the link would relaunch the IDE with a different prompt.
