# CodeAnt AI — Agent Skills for AI Coding Assistants

AI-powered code review and PR comment resolution — integrated into your AI coding workflow through the open Agent Skills (`SKILL.md`) format.

## Compatible AI Coding Assistants

These skills can run in AI coding assistants that support the Agent Skills standard, including:

- Claude Code
- Cursor
- GitHub Copilot in VS Code, JetBrains IDEs, the Copilot CLI, and the Copilot coding agent
- Kiro IDE, CLI, and web agent
- OpenAI Codex
- Gemini CLI and Antigravity
- Amp
- OpenCode
- Goose
- Junie
- Cline and Continue
- Other Agent Skills-compatible assistants

The repository currently provides first-party installation instructions for Claude Code and Cursor below. For another compatible assistant, copy the directories under `skills/` into that assistant's project or user skills directory. For example, use `.github/skills/` for GitHub Copilot and `.kiro/skills/` for Kiro.

Compatibility with the Agent Skills format does not automatically provide external tools or credentials. The assistant must have shell and filesystem access plus the provider authentication and CodeAnt CLI dependencies listed in [Skill Dependencies](#skill-dependencies). Individual hosts may require additional permission prompts, and not every host listed above is continuously tested by CodeAnt.

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
| `/codeant-resolve-quality-gates` | Find failed CodeAnt quality gates on a PR and fix actionable findings |
| `/codeant-review` | Run a CodeAnt code review on local changes and fix all issues |
| `/codeant-implement-repo-learnings` | Learn team review patterns from PR history and guidelines, generate custom rules in `.codeant/review.json` |
| `/codeant-scans-sast` | Fetch top 10 SAST findings on the current branch and fix them |
| `/codeant-scans-secrets` | Fetch top 10 secrets findings on the current branch and fix them |
| `/codeant-scans-dead-code` | Fetch top 10 dead code findings on the current branch and fix them |
| `/codeant-scans-anti-patterns` | Fetch top 10 anti-pattern findings on the current branch and fix them |
| `/codeant-scans-docstring` | Fetch top 10 docstring findings on the current branch and fix them |
| `/codeant-scans-complex-functions` | Fetch top 10 complex function findings on the current branch and fix them |

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

Run these commands from your project root:

```bash
mkdir -p .cursor/skills
git clone https://github.com/CodeAnt-AI/skills.git /tmp/codeant-skills
cp -r /tmp/codeant-skills/cursor/skills/* .cursor/skills/
rm -rf /tmp/codeant-skills
```

This installs these skills:

| Slash Command | Description |
|---------------|-------------|
| `/codeant-review` | Run a CodeAnt code review on local changes and fix all issues |
| `/codeant-resolve-pr-comments` | Fetch unresolved CodeAnt review comments on a PR and fix them |
| `/codeant-resolve-quality-gates` | Find failed CodeAnt quality gates on a PR and fix actionable findings |
| `/codeant-implement-repo-learnings` | Learn team review patterns and generate custom rules |
| `/codeant-scans-sast` | Fetch top 10 SAST findings on the current branch and fix them |
| `/codeant-scans-secrets` | Fetch top 10 secrets findings on the current branch and fix them |
| `/codeant-scans-dead-code` | Fetch top 10 dead code findings on the current branch and fix them |
| `/codeant-scans-anti-patterns` | Fetch top 10 anti-pattern findings on the current branch and fix them |
| `/codeant-scans-docstring` | Fetch top 10 docstring findings on the current branch and fix them |
| `/codeant-scans-complex-functions` | Fetch top 10 complex function findings on the current branch and fix them |

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

## Skill Dependencies

The skills use either the CodeAnt CLI or the source-control provider APIs directly. Runtime dependencies differ between Claude Code and Cursor:

| Skill | Claude Code | Cursor |
|-------|-------------|--------|
| `/codeant-resolve-pr-comments` | Provider APIs directly; CodeAnt CLI not used | CodeAnt CLI required |
| `/codeant-resolve-quality-gates` | CodeAnt CLI required | CodeAnt CLI required |
| `/codeant-review` | CodeAnt CLI required | CodeAnt CLI required |
| `/codeant-implement-repo-learnings` | CodeAnt CLI required | CodeAnt CLI required |
| `/codeant-scans-sast` | CodeAnt CLI required | CodeAnt CLI required |
| `/codeant-scans-secrets` | CodeAnt CLI required | CodeAnt CLI required |
| `/codeant-scans-dead-code` | CodeAnt CLI required | CodeAnt CLI required |
| `/codeant-scans-anti-patterns` | CodeAnt CLI required | CodeAnt CLI required |
| `/codeant-scans-docstring` | CodeAnt CLI required | CodeAnt CLI required |
| `/codeant-scans-complex-functions` | CodeAnt CLI required | CodeAnt CLI required |

`/codeant-resolve-quality-gates` is currently CLI-backed: it uses `codeant pr list` and `codeant pr comments`, and may use `codeant security-analysis` for detailed SCA findings.

The Claude plugin declares `codeant` as a shared plugin prerequisite because most included skills need it, so installation may still prompt for the CLI. At runtime, the Claude version of `/codeant-resolve-pr-comments` never installs, invokes, or falls back to the CodeAnt CLI.

## Prerequisites

### CLI-backed skills

For every skill marked "CodeAnt CLI required" above, install and authenticate the [CodeAnt CLI](https://docs.codeant.ai/cli/setup):

```bash
npm install -g codeant-cli
codeant login
```

For CLI-backed PR features, configure the matching source-control token:

```bash
codeant set-token <github|gitlab|bitbucket|azure> <your-token>
```

### Direct provider API skill

The Claude version of `/codeant-resolve-pr-comments` does not require CodeAnt CLI. It uses already-configured provider authentication instead:

- GitHub: authenticated `gh` CLI.
- GitLab: authenticated `glab` CLI.
- Bitbucket Cloud or Data Center: an existing OAuth, access-token, PAT, or authenticated session supported by the host.
- Azure DevOps: `AZURE_DEVOPS_PAT` when present. It may contain a plain PAT or a Base64-encoded recognized PAT/Basic credential payload; otherwise use an already-configured non-interactive OAuth or Microsoft Entra token.

## Documentation

- [Claude Code Integration Guide](https://docs.codeant.ai/cli/claude-code-integration)
- [Cursor Integration Guide](https://docs.codeant.ai/cli/cursor-integration)
- [CodeAnt CLI Docs](https://docs.codeant.ai/cli/setup)
