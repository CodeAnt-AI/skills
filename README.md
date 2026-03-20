# CodeAnt AI — Claude Code Plugin & Cursor Rules

AI-powered code review, secret scanning, security analysis, and static analysis — integrated into your AI coding workflow.

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

### Usage

```
> /codeant:review
> /codeant:review-fix
> /codeant:review staged files only
> /codeant:review against the develop branch
```

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
Run a CodeAnt security scan
```

## Prerequisites

- [CodeAnt CLI](https://docs.codeant.ai/cli/setup) installed and authenticated:

```bash
npm install -g codeant-cli
codeant login
```

## Documentation

- [Claude Code Integration Guide](https://docs.codeant.ai/cli/claude-code-integration)
- [Cursor Integration Guide](https://docs.codeant.ai/cli/cursor-integration)
- [CodeAnt CLI Docs](https://docs.codeant.ai/cli/setup)
