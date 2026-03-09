---
name: hexa-review-persona
description: HEXA Review Persona - Review PR using YOUR historical comment patterns
argument-hint: "<PR_ID> [ORG]"
---

# HEXA Review Persona (Self Mode)

> **Context Isolation:** Ignore prior conversation. Follow only these steps.

## Arguments

$ARGUMENTS

**If no PR ID provided, ask:** "Please provide the PR ID you want to review. Optionally include the organization (default: domoreexp)."

## Goal
Review a PR using YOUR own historical commenting patterns. This skill fetches your past PR comments on similar files and uses them as context to provide reviews in your unique style.

Default ORG: `domoreexp`

## Steps

### 0. Pre-flight Auth Check (REQUIRED)

```bash
powershell.exe -NoProfile -Command 'az account show --query "[name, user.name]" -o tsv 2>$null; if (-not $?) { exit 1 }'
```

**If auth check fails:**
1. Run: `powershell.exe -NoProfile -Command 'az login'`
2. After login succeeds, continue with Step 1

### 1. Get Current User Identity

```bash
powershell.exe -NoProfile -Command 'az account show --query "user.name" -o tsv'
```

Store as `REVIEWER_UPN`.

### 2. Get PR Details (ADO)

```bash
powershell.exe -Command "az repos pr show --id {PR_ID} --org 'https://dev.azure.com/{ORG}' -o json"
```

### 3. Get Skynet Token

```bash
powershell.exe -NoProfile -Command 'az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv'
```

### 4. Get File Diffs

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/pullrequest/diffs?pullRequestId={PR_ID}&organization={ORG}" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

### 5. Get YOUR Historical Comments - Personal History API

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; $files = "{URL_ENCODED_FILES}"; $reviewer = "{REVIEWER_UPN}"; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/kusto/personal-historical-comments?organization={ORG}&reviewerUniqueName=$reviewer&filePaths=$files&lookBackDays=60" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

### 6. Analyze Your Review Patterns

From your historical comments, identify:
- **Your focus areas**: What types of issues do you typically flag?
- **Your style**: Are you concise or detailed?
- **Your priorities**: What do you consider critical vs nice-to-have?

### 7. Review Using Your Persona

**Guidelines:**
- **Emulate your style**: Write comments as YOU would write them
- **Focus on YOUR priorities**: Emphasize issues you historically care about
- **Be consistent**: If you've flagged similar patterns before, flag them again

### 8. Output Format

```
## PR Review: {TITLE}
**PR #{ID}** | {AUTHOR} | {SOURCE} -> {TARGET}
**Reviewed as:** {YOUR_NAME} (based on {N} historical comments)

### Your Review Patterns Detected
- Primary focus: {e.g., "Error handling, null checks, logging"}
- Historical comments analyzed: {N}

### Critical Fixes (Blocking)
#### {FILE}:{LINE}
**{Category}**: {Issue description in YOUR voice}
*Based on your history: {Reference to similar past feedback}*

### Good to Have
#### [{High|Medium|Low}] {FILE}:{LINE}
**{Category}**: {Issue description}

### Summary
- **Files Changed:** X
- **Blocking Issues:** X
- **Recommendation:** Approve | Request Changes
```

## Step 9: Push Comments to PR

After showing review, present options for pushing comments by severity.

### Get ADO Token (if pushing)

```bash
powershell.exe -NoProfile -Command 'az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv'
```

### Push Comments

```
POST https://dev.azure.com/{ORG}/{PROJECT}/_apis/git/repositories/{REPO_ID}/pullRequests/{PR_ID}/threads?api-version=7.1
```
