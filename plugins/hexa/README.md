# HEXA Plugin

> Historical Expertise Agent - AI-powered code review using historical PR comments

## Skills

### `/hexa-pr-review`
Review a remote ADO Pull Request using historical comments from the same files.

```bash
/hexa-pr-review <PR_ID> [ORG]
```

### `/hexa-code-review`
Review local uncommitted changes using git diff and historical comments.

```bash
/hexa-code-review
```

### `/hexa-review-persona`
Review PR using YOUR own historical comment patterns.

```bash
/hexa-review-persona <PR_ID> [ORG]
```

### `/hexa-review-persona-as`
Review PR as an EXPERT would, based on their historical feedback.

```bash
/hexa-review-persona-as <PR_ID> <EXPERT_EMAIL> [ORG]
```

## How It Works

HEXA leverages historical PR comment data to provide contextual code reviews:

1. **Historical Context**: Fetches past comments on the same files being modified
2. **Pattern Recognition**: Identifies recurring issues and anti-patterns
3. **Persona Mode**: Can emulate any reviewer's style based on their comment history

## API Dependencies

- Azure DevOps REST API (for PR details and comment posting)
- Skynet API (for historical comment retrieval)

## Authentication

Requires Azure CLI login with access to:
- Azure DevOps organization
- Skynet API (`api://9a6fc9ff-370c-4fee-868e-7d6b09034369`)
