---
name: hexa-review-persona-as
description: HEXA Review Persona As - Review PR as an EXPERT would, learning from their historical comment patterns
argument-hint: "<PR_ID> <EXPERT_EMAIL> [ORG]"
---

# HEXA Review Persona As (Expert Mode)

> **Context Isolation:** Ignore prior conversation. Follow only these steps.

## Arguments

$ARGUMENTS

**If no PR ID or expert email provided, ask:** "Please provide the PR ID and the expert's email whose review style to emulate. Optionally include the organization (default: domoreexp)."

## Goal
Review a PR as a specific EXPERT would review it. Fetch the expert's historical PR comments and use their patterns to provide reviews as they would.

Default ORG: `domoreexp`

## Steps

### 0. Pre-flight Auth Check (REQUIRED)

```bash
powershell.exe -NoProfile -Command 'az account show --query "[name, user.name]" -o tsv 2>$null; if (-not $?) { exit 1 }'
```

**If auth check fails:**
1. Run: `powershell.exe -NoProfile -Command 'az login'`
2. After login succeeds, continue with Step 1

### 1. Store Expert Email

Store the provided `EXPERT_EMAIL` as `EXPERT_UPN`.

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

### 5. Get EXPERT'S Historical Comments

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; $files = "{URL_ENCODED_FILES}"; $expert = "{EXPERT_UPN}"; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/kusto/personal-historical-comments?organization={ORG}&reviewerUniqueName=$expert&filePaths=$files&lookBackDays=60" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

**If no comments found after 365 days:**
- Inform the user: "No historical comments found for {EXPERT_EMAIL} on these files."
- Ask if they want to continue with general best practices review.

### 6. Analyze Expert's Review Patterns

From the expert's historical comments, identify:
- **Their focus areas**: What types of issues do they typically flag?
- **Their style**: Concise or detailed? Code examples?
- **Their priorities**: What do they consider critical?

### 7. Review Using Expert's Persona

**Guidelines:**
- **Emulate their style**: Write comments as THEY would write them
- **Focus on THEIR priorities**: Emphasize issues they historically care about

### 8. Output Format

```
## PR Review: {TITLE}
**PR #{ID}** | {AUTHOR} | {SOURCE} -> {TARGET}
**Reviewed as:** {EXPERT_NAME} ({EXPERT_EMAIL})
**Based on:** {N} historical comments

### Expert Review Patterns Detected
- **Primary focus:** {e.g., "Security, input validation"}
- **Style:** {e.g., "Detailed with code examples"}
- **Historical comments analyzed:** {N}

### Critical Fixes (Blocking)
#### {FILE}:{LINE}
**{Category}**: {Issue description in EXPERT'S voice}
*Expert's pattern: {Reference to similar past feedback}*

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
