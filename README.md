# CodeAnt AI Skills

Pre-built skills and rules for integrating [CodeAnt AI CLI](https://docs.codeant.ai/cli/setup) with AI-powered code editors.

## Available Integrations

### Claude Code

Slash commands for running CodeAnt directly inside Claude Code.

**Quick setup:**

```bash
# Clone into your project's .claude/commands directory
git clone https://github.com/CodeAnt-AI/skills.git /tmp/codeant-skills
cp /tmp/codeant-skills/claude-code/*.md .claude/commands/
```

Or install globally:

```bash
mkdir -p ~/.claude/commands
git clone https://github.com/CodeAnt-AI/skills.git /tmp/codeant-skills
cp /tmp/codeant-skills/claude-code/*.md ~/.claude/commands/
```

**Available commands:**

| Command | Description |
|---------|-------------|
| `/review` | AI-powered code review on current changes |
| `/codeant-review-fix` | Review code and automatically fix all issues |

### Cursor

Cursor rules for integrating CodeAnt into your Cursor AI workflows.

**Quick setup:**

```bash
mkdir -p .cursor/rules
git clone https://github.com/CodeAnt-AI/skills.git /tmp/codeant-skills
cp /tmp/codeant-skills/cursor/codeant.mdc .cursor/rules/
```

## Prerequisites

- [CodeAnt CLI](https://docs.codeant.ai/cli/setup) installed and authenticated
- For Claude Code: Claude Code CLI installed
- For Cursor: Cursor IDE installed

## Documentation

- [Claude Code Integration Guide](https://docs.codeant.ai/cli/claude-code-integration)
- [Cursor Integration Guide](https://docs.codeant.ai/cli/cursor-integration)
- [CodeAnt CLI Documentation](https://docs.codeant.ai/cli/setup)
