---
name: hexa-pr-review
description: HEXA PR Review - Review a remote ADO Pull Request using historical comments and patterns
argument-hint: "<PR_ID> [ORG]"
---

# HEXA PR Review (Historical Expertise Agent)

> **Context Isolation:** Ignore prior conversation. Follow only these steps.

## Arguments

$ARGUMENTS

**If no PR ID provided, ask:** "Please provide the PR ID you want to review. Optionally include the organization (default: domoreexp)."

## Goal
Use historical PR comments to draw patterns and insights for reviewing code. Cross-reference current changes with past feedback to identify recurring issues, bad patterns, or regressions.

Default ORG: `domoreexp`

Never fall back to local git diff. If any API fails, ask user how to proceed.

## Steps

### 0. Pre-flight Auth Check (REQUIRED)

**Before doing anything else**, verify Azure CLI is authenticated:

```bash
powershell.exe -NoProfile -Command 'az account show --query "[name, user.name]" -o tsv 2>$null; if (-not $?) { exit 1 }'
```

**If auth check fails (exit code 1 or no output):**

1. Inform the user: "Azure CLI is not logged in. Starting login process..."
2. Run the interactive login:
   ```bash
   powershell.exe -NoProfile -Command 'az login'
   ```
3. Wait for login to complete (this opens a browser window)
4. After login succeeds, **continue with Step 1** - do not stop the skill

### 1. Get PR Details (ADO)
```bash
powershell.exe -Command "az repos pr show --id {PR_ID} --org 'https://dev.azure.com/{ORG}' -o json"
```
Extract: `title`, `sourceRefName`, `targetRefName`, `createdBy.displayName`, `repository.name`, `repository.id`, `repository.project.name`

### 2. Get HEXA Token

```bash
powershell.exe -NoProfile -Command 'az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv'
```

Store as HEXA_TOKEN. Reuse for all HEXA API calls.

### 3. Get File Diffs

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/pullrequest/diffs?pullRequestId={PR_ID}&organization={ORG}" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

### 4. Get Historical Comments - Progressive Lookback

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; $files = "{URL_ENCODED_FILES}"; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/pullrequest/files/comments?filePathsString=$files&repositoryName={REPO}&currentPullRequestId={PR_ID}&organization={ORG}&lookBackDays=60" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

### 5. Review Guidelines

**Categories:** Critical, Security, Performance, CodeQuality, BestPractices

**Severity:**
1. **Critical Fixes** - Must be fixed before merging
2. **Good to Have**: High / Medium / Low

### 6. Output Format

```
## PR Review: {TITLE}
**PR #{ID}** | {AUTHOR} | {SOURCE} -> {TARGET}

### Critical Fixes (Blocking)
#### {FILE}:{LINE}
**{Category}**: {Issue description}
*Historical Context: {pattern from history}*

### Good to Have
#### [{High|Medium|Low}] {FILE}:{LINE}
**{Category}**: {Issue description}

### Summary
- **Files Changed:** X
- **Blocking Issues:** X
- **Recommendation:** Approve | Request Changes
```

## Step 7: Push Comments to PR

After showing review, present options for pushing comments by severity.

### Get ADO Token (if pushing)

```bash
powershell.exe -NoProfile -Command 'az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv'
```

### Push Comments

```
POST https://dev.azure.com/{ORG}/{PROJECT}/_apis/git/repositories/{REPO_ID}/pullRequests/{PR_ID}/threads?api-version=7.1
```
