# CodeAnt AI — Claude Code Plugin & Cursor Rules

AI-powered code review, PR comment auto-fix, secret scanning, security analysis, and static analysis — integrated into your AI coding workflow.

## Claude Code

### Install via Plugin System (Recommended)

```
/plugin marketplace add CodeAnt-AI/skills
/plugin install codeant
```

That's it. You now have access to:

| Command | Description |
|---------|-------------|
| `/codeant:review` | AI-powered code review on current changes |
| `/codeant:review-fix` | Review code and automatically fix all issues |
| `/codeant:check-pr` | Fetch unaddressed PR comments and fix them |
| `/codeant:list-comments` | List review comments on a PR |
| `/codeant:find-prs` | List and explore pull requests |

### Usage Examples

```
> /codeant:review
> /codeant:review staged files only
> /codeant:check-pr 42
> /codeant:check-pr
> /codeant:list-comments 42
> /codeant:find-prs my open PRs
> /codeant:review-fix
```

### Auto-Fix Workflow

The `/codeant:check-pr` command enables a full auto-fix loop:

1. Detects the PR for your current branch (or takes a PR number)
2. Fetches all unaddressed CodeAnt review comments
3. Applies suggested fixes where available
4. Implements fixes for issues without suggestions
5. Runs a verification review
6. Reports what was fixed and what remains

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
List open PRs in this repo
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
