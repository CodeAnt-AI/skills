# CodeAnt AI — Claude Code Plugin & Cursor Rules

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

### Usage Examples

```
> /codeant-resolve-pr-comments 42
> /codeant-resolve-pr-comments
> /codeant-review
> /codeant-review staged files only
> /codeant-review last commit
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

### Install

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
