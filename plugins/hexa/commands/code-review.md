---
name: hexa-code-review
description: HEXA Code Review - Review local uncommitted changes using git diff and historical comments
---

# HEXA Code Review (Local)

> **Context Isolation:** Ignore prior conversation. Follow only these steps.

Review uncommitted local changes (staged + unstaged + untracked) before committing.

## Steps

### 1. Get Repo Info

```bash
git rev-parse --abbrev-ref HEAD
git remote get-url origin | sed -n 's/.*\/_git\///p'
```

### 2. Get Local Changes

```bash
git status --short                    # list modified/staged/untracked
git diff                              # unstaged changes
git diff --cached                     # staged changes
```

For untracked files (??), read file content directly since they have no diff.

### 3. Get Skynet Token

```bash
az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv
```

Store as SKYNET_TOKEN. Reuse for all HEXA API calls.

> **If token fails**: Run `az login --scope api://9a6fc9ff-370c-4fee-868e-7d6b09034369/.default`

### 4. Get Historical Comments

```bash
curl -s -H "Authorization: Bearer {SKYNET_TOKEN}" \
  "https://engsys-skynet-preprod.azurewebsites.net/api/ado/pullrequest/files/comments?filePathsString={FILES}&repositoryName={REPO}&currentPullRequestId=0&organization=domoreexp&lookBackDays=60"
```

### 5. Review & Output

**Guidelines:**
- **Categories**: Critical, Security, Performance, CodeQuality, BestPractices
- **Severity**: Critical, Medium, Low (prioritize Critical)
- **Be selective**: Report all Critical. Limit Medium/Low to significant ones only.
- **Historical context**: Reference patterns from historical comments.
- **If no issues found**: Output "No critical issues found."

```
## HEXA Code Review: {BRANCH}
**Local Changes**: {COUNT} files

### Suggestions
#### {FILE}:{LINE}
**[{Severity}]** [{Category}] {Issue} -> {Fix}
*Context: {Why this matters}*

### Summary
Critical: X | Medium: X | Low: X
```
