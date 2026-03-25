---
name: codeant-implement-repo-learnings
description: Learn your team's review patterns from PR history, project guidelines (.cursorrules, CLAUDE.md), and coding conventions to generate custom CodeAnt review rules in .codeant/review.json
---

Analyze the last 100 pull requests to extract recurring human review feedback patterns, combine them with project guidelines from committed config files (.cursorrules, CLAUDE.md, .guidelines, etc.), and generate a `.codeant/review.json` with custom review rules tailored to the team's standards. Each candidate rule is presented for interactive approval before being included.

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

### Step 0b — Verify Prerequisites

Verify the repo has a remote and detect the SCM platform:

```bash
git remote -v
```

If no remote is found, tell the user: "This repo has no remote. Please add one or run this from a repo with PR history." and stop.

The `codeant` CLI auto-detects the repository name, remote platform (GitHub/GitLab/Bitbucket/Azure), and default branch from the git remote URL. You do not need to manually detect these.

Verify the user has a token set for the detected platform:

```bash
codeant pr list --state closed --limit 1
```

If this fails with an authentication error, tell the user: "Please set your SCM token first: `codeant set-token <github|gitlab|bitbucket|azure> <your-token>`" and stop.

### Step 0c — Check for Existing review.json

Before starting, check if a `.codeant/review.json` already exists:

```bash
cat .codeant/review.json 2>/dev/null
```

If it exists:
1. Parse the existing rules and store them.
2. Tell the user: "Found an existing `.codeant/review.json` with N rules. I'll merge new rules with the existing ones — existing rules will be preserved, and I'll flag any duplicates or conflicts."
3. During rule generation (Step 9), compare each candidate rule against existing rules:
   - If a candidate rule is **semantically identical** to an existing rule (same intent, same file patterns) → **SKIP** and note it as "already covered."
   - If a candidate rule **overlaps** with an existing rule but adds specificity or covers a different angle → present both to the user and ask which to keep, or whether to merge them.
   - If a candidate rule is **entirely new** → present for approval as normal.

If it does not exist, continue normally.

### Step 0d — Track Skill Invocation

```bash
codeant track --event "skill_invoked" --props '{"skill_name": "codeant-implement-repo-learnings", "source": "claude-code"}'
```

### Step 1 — Fetch the Last 100 Merged Pull Requests

Fetch the most recent 100 closed/merged PRs to ensure a rich feedback history. Use pagination with offset:

```bash
codeant pr list --state closed --limit 100
```

The output is a JSON array of PR objects with fields: `number`, `title`, `state`, `merged`, `author`, `sourceBranch`, `targetBranch`, `createdAt`, `updatedAt`, `url`.

Filter to keep only PRs where `merged` is `true` (closed but not merged PRs are not useful).

If fewer than 10 merged PRs are returned, warn the user: "Only N merged PRs found. The generated rules may be less accurate with limited history. Continuing anyway."

If zero merged PRs are returned, stop and tell the user: "No merged PRs found in this repo. Custom rules require PR history to learn from."

Tell the user: "Found N merged PRs. Fetching review comments..."

### Step 2 — Fetch Human Review Comments from Each PR

For each merged PR, fetch **only non-CodeAnt comments** using the `--codeant-generated false` flag. This is critical — without this flag, the CLI returns all comments including CodeAnt bot reviews, quality gate reports, and coverage reports which would overwhelm the results.

```bash
codeant pr comments --pr-number <N> --codeant-generated false
```

The output is a JSON array of comment objects. The fields vary slightly by platform (GitHub, GitLab, Bitbucket, Azure DevOps) — the CLI auto-detects the platform from the git remote.

**Common fields (all platforms):**

| Field              | Description                                                        |
|--------------------|--------------------------------------------------------------------|
| `id`               | Unique comment identifier (integer)                                |
| `type`             | `"review"` (inline on code) or `"issue"` (general PR comment)     |
| `author`           | Comment author login string (e.g., `"Chhinna"`, `"Sagar-CodeAnt"`)|
| `body`             | The full comment text in markdown                                  |
| `path`             | File path the comment refers to (null for general comments)        |
| `line`             | Line number the comment refers to (null for general comments)      |
| `createdAt`        | ISO 8601 timestamp when the comment was posted                     |
| `updatedAt`        | ISO 8601 timestamp when the comment was last edited                |
| `isCodeantComment` | Boolean — always `false` when using `--codeant-generated false`    |

**Platform-specific fields:**

| Field              | GitHub | GitLab | Bitbucket | Azure DevOps |
|--------------------|--------|--------|-----------|--------------|
| `inReplyToId`      | Yes    | No     | No        | No           |
| `resolved`         | No     | Yes    | Yes       | Yes          |
| `discussionId`     | No     | Yes    | No        | No           |
| `threadId`         | No     | No     | No        | Yes          |

**Important**: Process PRs in batches of 10–15 to be efficient. After each batch, tell the user the progress: "Processed X/Y PRs, collected Z comments so far..."

### Step 3 — Filter: Keep Only Genuine Human Peer-Review Comments

Even with `--codeant-generated false`, additional filtering is needed. The filtering strategy must account for platform differences.

#### 3a. Remove remaining bot comments

Filter out comments where `author` matches bot patterns (case-insensitive):
- Contains `[bot]` suffix (e.g., `dependabot[bot]`, `renovate[bot]`)
- Contains `-bot` suffix (e.g., `snyk-bot`)
- Exact matches: `dependabot`, `renovate`, `codecov`, `sonarcloud`, `github-actions`, `copilot`, `mergify`, `stale`, `allcontributors`, `semantic-release-bot`, `greenkeeper`

#### 3b. Remove PR author self-replies to bot reviews

PR authors often reply to CodeAnt bot suggestions with acknowledgements like "it is correct", "updated", "this is okay. no change needed here", "frontend will handle this". These are NOT peer review feedback and must be filtered out.

**The detection strategy differs by platform:**

**GitHub** — Use `inReplyToId`:
1. Comments where `inReplyToId` is not null AND the comment author is the **same person as the PR author** → **DISCARD**. These are the PR author responding to bot reviews.
2. Comments where `inReplyToId` is not null AND the comment author is **different from the PR author** → **KEEP**. These may be peer reviewers adding context to a thread.

**GitLab / Bitbucket / Azure DevOps** — No `inReplyToId` field exists. Use content-based heuristics instead:
1. If the comment author is the **same person as the PR author**, AND the comment body matches a bot-reply pattern → **DISCARD**. Bot-reply patterns include:
   - Short acknowledgements: "it is correct", "this is correct", "updated", "fixed", "done", "agreed", "will do", "noted"
   - Dismissals: "this is okay", "no change needed", "current behaviour is expected", "not applicable", "this is fine"
   - Deferrals: "frontend will handle this", "will refactor later", "out of scope", "this is a product decision"
   - Bot invocations: starts with `@codeant` or `@codeant-ai`
2. If the comment author is **different from the PR author** → **KEEP** (peer reviewer).
3. If the comment author is the PR author but the body contains a substantive code suggestion or convention reference → **KEEP** (author self-reviewing or adding context).

#### 3c. Remove general PR comments (type: "issue")

Comments with `type: "issue"` and `path: null` are general PR conversation comments — not inline code review feedback. These include:
- Bot trigger commands (e.g., `@codeant-ai : review`)
- Status updates
- Coverage/quality gate reports pasted by humans

**Discard** all comments where `type` is `"issue"`.

#### 3d. Remove very short non-actionable comments

Filter out comments where `body` length is < 15 characters — these are typically reactions like "LGTM", "+1", "nice", "thanks", "updated", "fixed".

#### 3e. On GitLab/Bitbucket/Azure — Skip already-resolved threads

On platforms that provide the `resolved` field (GitLab, Bitbucket, Azure DevOps), also consider: resolved comments are less likely to contain patterns worth codifying if the fix was trivial. However, **do NOT discard resolved comments** — they often contain the best feedback (reviewer pointed out an issue → author fixed it → thread resolved). Only use the `resolved` field as a signal during Step 4 classification: resolved comments with substantive feedback are high-confidence patterns.

After all filtering, tell the user: "Collected N human review comments from M pull requests (platform: <detected-platform>)."

### Step 4 — Classify Comments: Extract Actionable Feedback

From the filtered human comments, classify each into one of two categories:

**FEEDBACK (keep)** — Actionable review suggestions. These suggest a concrete change or enforce a convention:
- Suggestions to change implementation ("can you change the route to X", "use X instead of Y", "consider doing X")
- Code organization feedback ("can you move these to X", "this belongs in the service layer")
- Resource/import concerns ("do we really need a new logger? can we import from X", "can we use it here as well from X")
- File structure feedback ("we can move this file to a folder inside X")
- Naming/convention feedback ("we use camelCase for X", "prefix private methods with _")
- Security concerns ("don't expose X", "sanitize this input")
- Performance suggestions ("this is O(n^2), use a map", "avoid calling X in a loop")
- Testing feedback ("add a test for edge case X")
- Pattern enforcement ("we always do X when Y", "follow the pattern in Z")
- Error handling ("wrap this in try/except", "handle the null case")

**NOT FEEDBACK (discard)** — Non-actionable comments:
- Acknowledgements of bot suggestions: "it is correct", "this is correct", "current behaviour is expected", "this is okay. no change needed here", "this is good, will refactor it accordingly"
- Descriptive/explanatory comments ("this function does X", "I see what's happening here")
- Questions without suggestions ("why is this needed?", "what does this do?")
- Approvals ("LGTM", "looks good", "nice", "+1", "approved")
- Merge/process comments ("can you rebase?", "ready to merge")
- Status updates ("fixed in latest commit", "addressed your comment", "updated")
- Product decisions without code implications ("this is a product based decision", "frontend will handle this")
- Emoji-only reactions
- Bot invocation commands ("@codeant-ai : review", "@codeant: hey...")
- Very short comments (< 15 characters) that are likely reactions

Use the following heuristic to classify:
1. Does the comment suggest a **concrete change** to the code? → FEEDBACK
2. Does it reference a **pattern, convention, or standard** the team follows? → FEEDBACK
3. Does it point out a **potential bug, security issue, or performance problem**? → FEEDBACK
4. Is it purely descriptive, procedural, or conversational? → NOT FEEDBACK

After classification, tell the user: "Found N actionable feedback comments out of M total human comments."

If fewer than 5 feedback comments are found, warn: "Very few actionable feedback comments found. The generated rules may be generic. Consider running this again after more PRs have been reviewed."

### Step 5 — Analyze Bug-Fix and Revert Commits

Mine the git history for commits that fix bugs or revert changes — these reveal what patterns *cause* production issues and are high-confidence rule candidates.

#### 5a. Find bug-fix commits

```bash
git log --oneline --all -500 --grep="fix" --grep="bug" --grep="hotfix" --grep="patch" --grep="resolve" --or
```

Also search for revert commits:

```bash
git log --oneline --all -500 --grep="revert" --grep="Revert"
```

#### 5b. Analyze the fixes

For each bug-fix or revert commit (up to 50 most recent), examine what was changed:

```bash
git show --stat <commit-hash>
git show --format="" <commit-hash>
```

Look for patterns in what keeps getting fixed:

1. **Repeated fix locations**: If the same file or directory gets bug fixes 3+ times, it's a hotspot that needs stricter review rules.
2. **Repeated fix types**: Group fixes by category:
   - **Null/undefined checks**: Missing null guards, optional chaining, default values
   - **Error handling**: Missing try/catch, uncaught promise rejections, swallowed errors
   - **Type mismatches**: Wrong types passed, incorrect return types, type coercion bugs
   - **Async issues**: Missing await, blocking calls in async context (e.g., `time.sleep` in async handlers), race conditions
   - **Off-by-one / boundary errors**: Array index issues, pagination bugs, range errors
   - **Missing validation**: Unvalidated user input, missing field checks on request bodies
   - **Resource leaks**: Unclosed connections, file handles, missing cleanup
   - **API contract violations**: Wrong status codes, missing response fields, inconsistent error formats
3. **Revert reasons**: If the commit message or PR body explains why something was reverted, extract the root cause. Common patterns: "broke X", "caused Y in production", "regression in Z".

#### 5c. Cross-reference with PR feedback

If a bug-fix pattern aligns with feedback found in Step 4 (e.g., reviewers keep saying "add error handling" AND there are multiple commits fixing missing error handling), that's a **very high confidence** rule. Mark these as "reinforced by bug history" during rule generation.

Tell the user: "Analyzed N bug-fix commits and M reverts. Found P recurring fix patterns."

If zero bug-fix commits are found, skip this step silently and continue.

### Step 6 — Read Project Guidelines and Config Files

Search the repository root for any committed guidelines or configuration files that encode team standards:

```bash
ls -la .cursorrules .cursor/rules/*.mdc .cursor/rules/*.md CLAUDE.md .claude/CLAUDE.md .github/CONTRIBUTING.md CONTRIBUTING.md .guidelines .github/CODEOWNERS 2>/dev/null
```

Read any files that exist. Focus on extracting **code logic, architecture, and convention rules** — not styling/formatting:

- **`.cursorrules` / `.cursor/rules/*`** — AI coding assistant rules (often contain team conventions)
- **`CLAUDE.md` / `.claude/CLAUDE.md`** — Claude Code project instructions
- **`CONTRIBUTING.md`** — Contribution guidelines (focus on code conventions, not formatting)
- **`.guidelines`** — Explicit team guidelines

**Ignore styling/formatting files** — do NOT read or extract rules from:
- `.editorconfig`, `.prettierrc*`, `.eslintrc*` (formatting rules only)
- `CODE_STYLE.md`, `STYLE_GUIDE.md` (styling/formatting guides)
- `.clang-format`, `.clang-tidy`
- Linter configs that are purely about whitespace, indentation, or code style

The goal is to capture **logic, architecture, security, and convention patterns** — not formatting preferences that are already handled by formatters and linters.

For each file found, extract any rules that are:
1. Not already enforced by a linter or formatter
2. About code logic, architecture, security, patterns, or team conventions (NOT formatting/styling)
3. Specific enough to be actionable in a review

Tell the user: "Found K guideline/config files with relevant conventions."

### Step 7 — Detect the Primary Languages and Project Structure

Detect the primary languages in the repo:

```bash
git ls-files | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -20
```

Also detect the directory structure for precise file globs:

```bash
git ls-files | head -200
```

Map extensions to glob patterns:
- `.ts` / `.tsx` → `**/*.ts`, `**/*.tsx`
- `.js` / `.jsx` → `**/*.js`, `**/*.jsx`
- `.py` → `**/*.py`
- `.go` → `**/*.go`
- `.rs` → `**/*.rs`
- `.java` → `**/*.java`
- `.rb` → `**/*.rb`
- etc.

Use actual directory paths from the project when a rule only applies to certain areas (e.g., `src/api/**/*.ts` for API-specific rules).

### Step 8 — Analyze and Cluster Feedback into Rule Categories

Analyze all collected feedback comments to identify **recurring patterns**. Group similar feedback into clusters:

1. **Frequency analysis**: Which types of feedback appear 2+ times across different PRs? These are candidates for rules. 3+ times = strong candidates.
2. **Theme extraction**: Group comments by theme:
   - Error handling patterns
   - Naming conventions
   - Security practices
   - Performance patterns
   - Testing requirements
   - Code organization / architecture
   - API design
   - Logging / observability
   - Input validation
   - Resource management (connections, files, memory)
   - Concurrency / async patterns
   - Documentation requirements
   - Type safety
   - Import organization
3. **Cross-reference with guidelines**: If a guideline file mentions a convention AND reviewers frequently enforce it in PRs, it's a high-confidence rule.
4. **Cross-reference with bug-fix history** (from Step 5): If a pattern appears in both PR feedback AND bug-fix commits, it's a **very high confidence** rule. Mark these with a confidence indicator.
5. **Bug-fix-only patterns**: Some patterns may only appear in the bug-fix history (no reviewer ever mentioned it, but it keeps getting fixed). These are still valid rule candidates — present them with the bug-fix evidence.
6. **Language/framework context**: Tailor rules to the detected languages and frameworks.

**Confidence levels for each candidate rule:**
- **HIGH** — Pattern found in 3+ PR comments, OR found in both PR comments AND bug-fix history, OR found in both PR comments AND guideline files
- **MEDIUM** — Pattern found in 2 PR comments, OR found in bug-fix history only (2+ fix commits), OR found in guideline files only with strong specificity
- **LOW** — Pattern found in 1 PR comment + 1 guideline reference, or 1 bug-fix commit + 1 PR comment

Only generate rules with MEDIUM or HIGH confidence. LOW confidence patterns should be mentioned to the user but not proposed as rules.

For each identified cluster, draft a candidate rule with:
- A clear, specific description of what to enforce
- The file patterns it applies to
- Whether it's IDE-only, PR-only, or both
- The evidence: which PR comments, bug-fix commits, and/or guideline files support this rule
- The confidence level (HIGH/MEDIUM)

#### 8b. Check against existing rules

If an existing `.codeant/review.json` was found in Step 0c, compare each candidate rule against the existing rules:
- **Already covered** — candidate is semantically identical to an existing rule → skip and note it
- **Overlapping** — candidate overlaps with an existing rule but adds specificity → present both and ask the user whether to keep the existing rule, replace it with the new one, or keep both
- **New** — candidate covers something not in the existing rules → present for approval

### Step 9 — Interactive Rule-by-Rule Confirmation

**This step is critical.** Do NOT skip it. Present each candidate rule to the user one at a time and get their approval before including it.

For each candidate rule, present it like this:

---

**Rule N of M**: `rule-id-here` — Confidence: **HIGH**

**Description**: "Full description of the rule"

**Applies to**: `["glob-pattern-1", "glob-pattern-2"]`

**Scope**: `["ide", "pr"]`

**Evidence**:
- PR #X: "reviewer comment that supports this rule..."
- PR #Y: "another reviewer comment..."
- Bug-fix commit `abc123`: "fix: missing null check in payment handler" (fixes same pattern)
- Guideline file: `.cursorrules` mentions "..."

**Action**: Should I include this rule? You can:
- **Yes** — Include as-is
- **Modify** — Tell me what to change (description, files, scope)
- **Skip** — Don't include this rule

---

Wait for the user's response before moving to the next rule. If the user says "modify", apply their changes and confirm the updated version before proceeding.

If there are many rules (>15), after presenting the first 5, ask the user: "There are M more rules to review. Would you like to continue one-by-one, or should I show the rest in a batch and you can tell me which to remove?"

### Step 10 — Generate the Final review.json

After all rules have been confirmed, convert the approved rules into the CodeAnt `review.json` format:

```json
{
    "id": "unique-kebab-case-id",
    "description": "Clear, specific description of what to enforce and why",
    "files": ["glob-pattern-1", "glob-pattern-2"],
    "scope": ["ide", "pr"]
}
```

**Rule quality guidelines:**

- **Be specific, not vague.** Bad: "Write clean code." Good: "All public API endpoints must validate request body fields before processing — return 400 with a descriptive message for missing or malformed fields."
- **Include the 'why' when it's not obvious.** Bad: "Don't use any." Good: "Use TypeScript's unknown instead of any for external API response types — any disables type checking and has caused production type errors."
- **Reference the team's actual patterns.** If the feedback says "use our ErrorResponse type", the rule should mention that specific type.
- **Use precise file globs.** Match the actual project structure.
- **Set scope appropriately:**
  - `["ide"]` — for style/convention rules developers should see while coding
  - `["pr"]` — for rules that need full PR context (e.g., "every PR touching the API must update the OpenAPI spec")
  - `["ide", "pr"]` — for critical rules that should be enforced everywhere (security, error handling)

### Step 11 — Write the review.json

Create the `.codeant` directory and write the `review.json`:

```bash
mkdir -p .codeant
```

**If no existing `review.json` was found** — write the file with the approved rules:

```json
{
    "rules": [
        {
            "id": "rule-id-1",
            "description": "...",
            "files": ["..."],
            "scope": ["ide", "pr"]
        }
    ]
}
```

**If an existing `review.json` was found** — merge the new approved rules into the existing file:
1. Keep all existing rules that were not marked as "replaced" during Step 9.
2. Append the newly approved rules.
3. Remove any existing rules the user explicitly chose to replace.
4. Ensure no duplicate `id` values — if a new rule has the same `id` as an existing one, suffix it with `-v2` or ask the user to rename.
5. Write the merged result.

After writing, show the user a diff summary: "Added N new rules, kept M existing rules, replaced P rules."

### Step 12 — Present the Final Summary

Show the user a summary:

**Custom Rules Generated:**
- **Source data**: N human feedback comments from M pull requests + P bug-fix/revert commits + K guideline files
- **Rules approved**: X out of Y candidates
- **Rules skipped**: Z
- **Existing rules preserved**: E (if applicable)

Then list the final rules grouped by theme:

For each rule show:
- Rule ID
- Description (full)
- File patterns
- Scope

### Step 13 — Offer Next Steps

Tell the user:

"Custom rules written to `.codeant/review.json`. These rules will be applied on top of CodeAnt's default bug and security detection during IDE and PR reviews.

Next steps:
1. **Commit** the `.codeant/review.json` file so your team benefits from these rules
2. **Test** by running `/codeant-review` on your current changes to see the rules in action
3. **Iterate** — you can re-run `/codeant-implement-repo-learnings` anytime to update rules as your team's practices evolve
4. **Edit directly** — modify `.codeant/review.json` anytime to add, remove, or tweak rules

Would you like me to commit the file now?"

### Step 13b — Track Results

```bash
codeant track --event "custom_rules_generated" --props '{"skill_name": "codeant-implement-repo-learnings", "source": "claude-code", "rules_count": <N>, "prs_analyzed": <M>, "feedback_comments": <K>, "bugfix_commits": <P>, "guideline_files": <G>, "rules_skipped": <Z>, "existing_rules_preserved": <E>}'
```

### Important Rules

- Do **NOT** generate rules that duplicate what linters already enforce (eslint, pylint, etc.). CodeAnt rules should capture team-specific conventions that tools don't catch.
- Do **NOT** generate vague or generic rules like "write clean code" or "follow best practices." Every rule must be specific and actionable.
- Do **NOT** include rules based on a single comment unless it's also supported by a guideline file or bug-fix history. A pattern from PR feedback alone must appear in at least 2 different PRs to be worth codifying.
- Do **NOT** make up rules that aren't supported by the PR feedback, bug-fix history, or guideline files. Every rule must trace back to actual evidence.
- Do **NOT** overwrite existing `.codeant/review.json` rules without user confirmation. Always merge, never replace.
- Do **NOT** include PII, secrets, or internal URLs in rule descriptions.
- Do **NOT** skip the interactive confirmation step. Every rule must be approved by the user before being written.
- **Prefer quality over quantity.** 15 excellent rules are better than 40 mediocre ones.
- If a rule is language-specific, ensure the file glob only matches that language.
- If the repo is a monorepo, try to scope rules to the appropriate subdirectory rather than using `**/*` everywhere.
- Always show the evidence (PR comments / guideline references) for each rule so the user can judge its validity.
