# CodeAnt AI — Claude Code Plugin & Cursor Skills

AI-powered code review and PR comment resolution — integrated into your AI coding workflow.

## Claude Code

### Install via Plugin System (Recommended)

```
/plugin marketplace add CodeAnt-AI/skills
/plugin install codeant
```

That's it. You now have access to:

| Command | Description |
|---------|-------------|
| `/codeant-resolve-pr-comments` | Fetch all unaddressed CodeAnt review comments on a PR and fix them |
| `/codeant-review` | Run a CodeAnt code review on local changes and fix all issues |
| `/codeant-implement-repo-learnings` | Learn team review patterns from PR history and guidelines, generate custom rules in `.codeant/review.json` |

### Usage Examples

```
> /codeant-resolve-pr-comments 42
> /codeant-resolve-pr-comments
> /codeant-review
> /codeant-review staged files only
> /codeant-review last commit
> /codeant-implement-repo-learnings
```

### Resolve PR Comments Workflow

The `/codeant-resolve-pr-comments` command enables a full auto-fix loop:

1. Detects the PR for your current branch (or takes a PR number)
2. Fetches all unaddressed CodeAnt review comments
3. Presents a summary grouped by file and severity
4. Applies suggested fixes where available
5. Implements fixes for issues without suggestions
6. Runs a verification review
7. Reports what was fixed and what remains

### Review Local Workflow

The `/codeant-review` command reviews and fixes your local changes:

1. Runs a CodeAnt AI review on your uncommitted, staged, or last commit changes
2. Presents findings grouped by severity with security issues highlighted first
3. Fixes every issue found, starting with critical severity
4. Runs a verification review to confirm all fixes are clean
5. Reports initial findings, fixes applied, and verification results

## Cursor

### Install (Cursor 2.4+ — Skills Format)

```bash
mkdir -p .cursor/skills
git clone https://github.com/CodeAnt-AI/skills.git /tmp/codeant-skills
cp -r /tmp/codeant-skills/cursor/skills/* .cursor/skills/
rm -rf /tmp/codeant-skills
```

This installs three skills:

| Slash Command | Description |
|---------------|-------------|
| `/codeant-review` | Run a CodeAnt code review on local changes and fix all issues |
| `/codeant-resolve-pr-comments` | Fetch unresolved CodeAnt review comments on a PR and fix them |
| `/codeant-implement-repo-learnings` | Learn team review patterns and generate custom rules |

Then use slash commands or ask Cursor naturally:

```
> /codeant-review
> /codeant-resolve-pr-comments 42
> Review my changes with CodeAnt
> Fix all unaddressed CodeAnt comments on my current PR
> /codeant-implement-repo-learnings
```

<details>
<summary>Legacy install (older Cursor versions using .mdc rules)</summary>

```bash
mkdir -p .cursor/rules
git clone https://github.com/CodeAnt-AI/skills.git /tmp/codeant-skills
cp /tmp/codeant-skills/cursor/codeant.mdc .cursor/rules/
rm -rf /tmp/codeant-skills
```

Then ask Cursor naturally:

```
Review my changes with CodeAnt
Fix all unaddressed CodeAnt comments on PR #42
```

Note: The legacy `.mdc` rule does not include the `codeant-implement-repo-learnings` skill or the verdict system. We recommend migrating to the Skills format.
</details>

## Prerequisites

- [CodeAnt CLI](https://docs.codeant.ai/cli/setup) installed and authenticated:

```bash
npm install -g codeant-cli
codeant login
```

- For PR features, configure your SCM token:

```bash
codeant set-token github <your-token>
```

## Documentation

- [Claude Code Integration Guide](https://docs.codeant.ai/cli/claude-code-integration)
- [Cursor Integration Guide](https://docs.codeant.ai/cli/cursor-integration)
- [CodeAnt CLI Docs](https://docs.codeant.ai/cli/setup)
