# HEXA Claude Plugin

> HEXA (Historical Expertise Agent) - AI-powered code review using historical PR comments and patterns.

A Claude Code plugin that provides intelligent PR reviews by learning from historical comments. Includes persona-based reviews that can emulate any reviewer's style.

## Features

- **PR Review**: Review remote ADO PRs using historical comments from the same files
- **Code Review**: Review local uncommitted changes before committing
- **Review Persona (Self)**: Review PRs using YOUR own historical comment patterns
- **Review Persona (Expert)**: Review PRs as any expert would, based on their historical feedback

## Installation

Add to your Claude Code settings:

```bash
claude mcp add-plugin https://github.com/HarshSachan0007/hexa-claude-plugin
```

Or add manually to `~/.claude/plugins.json`:

```json
{
  "plugins": [
    "https://github.com/HarshSachan0007/hexa-claude-plugin/plugins/hexa"
  ]
}
```

## Available Commands

| Command | Description |
|---------|-------------|
| `/hexa-pr-review <PR_ID> [ORG]` | Review a remote ADO PR using historical comments |
| `/hexa-code-review` | Review local uncommitted changes |
| `/hexa-review-persona <PR_ID> [ORG]` | Review PR using YOUR comment patterns |
| `/hexa-review-persona-as <PR_ID> <EMAIL> [ORG]` | Review PR as an expert would |

## Prerequisites

- Azure CLI installed and logged in (`az login`)
- Access to Azure DevOps organization
- Access to Skynet APIs (for historical comments)

## How It Works

### Historical Context
HEXA fetches historical PR comments from the same files being modified. This provides context about:
- Past issues flagged on these files
- Recurring patterns and anti-patterns
- Team coding standards and expectations

### Review Persona
The persona modes go further by analyzing a specific reviewer's comment history:
- **Your patterns**: What issues do you typically flag?
- **Your style**: Concise or detailed? Code examples?
- **Your priorities**: Security-focused? Performance-focused?

This allows reviews that match your (or an expert's) unique perspective.

## Example Usage

```bash
# Standard PR review with historical context
/hexa-pr-review 123456

# Review local changes before committing
/hexa-code-review

# Review as yourself
/hexa-review-persona 123456

# Review as a senior engineer would
/hexa-review-persona-as 123456 senior.dev@company.com
```

## License

MIT
