---
name: review-persona
description: HEXA Review Persona - Review PR using YOUR historical comment patterns
allowed-tools: Bash, Read, AskUserQuestion
argument-hint: "<PR_ID> [ORG]"
---

# HEXA Review Persona (Self Mode)

> **Context Isolation:** Ignore prior conversation. Follow only these steps.

## Goal
Review a PR using YOUR own historical commenting patterns. This skill fetches your past PR comments on similar files and uses them as context to provide reviews in your unique style and focus areas.

## Usage
`/hexa-review-persona <PR_ID>` or `/hexa-review-persona <PR_ID> <ORG>`

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

Get the logged-in user's UPN (email) for fetching their historical comments:

```bash
powershell.exe -NoProfile -Command 'az account show --query "user.name" -o tsv'
```

Store as `REVIEWER_UPN`.

### 2. Get PR Details (ADO)

```bash
powershell.exe -Command "az repos pr show --id {PR_ID} --org 'https://dev.azure.com/{ORG}' -o json"
```

Extract: `title`, `sourceRefName`, `targetRefName`, `createdBy.displayName`, `repository.name`, `repository.id`, `repository.project.name`

### 3. Get HEXA Token

```bash
powershell.exe -NoProfile -Command 'az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv'
```

### 4. Get File Diffs

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/pullrequest/diffs?pullRequestId={PR_ID}&organization={ORG}" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

### 5. Get YOUR Historical Comments - Personal History API

Fetch YOUR past comments on the same files.

**Progressive Lookback (60 -> 90 -> 180 -> 365 days):**

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; $files = "{URL_ENCODED_FILES}"; $reviewer = "{REVIEWER_UPN}"; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/kusto/personal-historical-comments?organization={ORG}&reviewerUniqueName=$reviewer&filePaths=$files&lookBackDays=60" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

### 6. Analyze Your Review Patterns

From your historical comments, identify:
- **Your focus areas**: What types of issues do you typically flag?
- **Your style**: Are you concise or detailed?
- **Your priorities**: What do you consider critical vs nice-to-have?
- **Common themes**: Recurring patterns in your feedback

### 7. Review Using Your Persona

Apply your historical patterns to review the current PR:

**Guidelines:**
- **Emulate your style**: Write comments as YOU would write them
- **Focus on YOUR priorities**: Emphasize issues you historically care about
- **Be consistent**: If you've flagged similar patterns before, flag them again

**Severity (based on your patterns):**
1. **Critical Fixes** - Issues you would block a PR for
2. **Good to Have**: High / Medium / Low

### 8. Output Format

```
## PR Review: {TITLE}
**PR #{ID}** | {AUTHOR} | {SOURCE} -> {TARGET}
**Reviewed as:** {YOUR_NAME} (based on {N} historical comments over {X} days)

### Your Review Patterns Detected
- Primary focus: {e.g., "Error handling, null checks, logging"}
- Style: {e.g., "Concise with code suggestions"}
- Historical comments analyzed: {N}

### Critical Fixes (Blocking)
#### {FILE}:{LINE}
**{Category}**: {Issue description in YOUR voice}
*Based on your history: {Reference to similar past feedback}*

### Good to Have
#### [{High|Medium|Low}] {FILE}:{LINE}
**{Category}**: {Issue description in YOUR voice}
*Based on your history: {Reference to similar past feedback}*

### Summary
- **Files Changed:** X
- **Your Historical Context:** {N} comments over {X} days
- **Blocking Issues:** X
- **Overall Risk:** Low | Medium | High
- **Recommendation:** Approve | Request Changes | Needs Discussion
```

---

## Step 9: Push Comments to PR

After showing the review, present multi-select options for pushing comments by severity.

### Step 10: Get ADO Token (if pushing)

```bash
powershell.exe -NoProfile -Command 'az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv'
```

### Step 11: Push Selected Comments

**API Endpoint:**
```
POST https://dev.azure.com/{ORG}/{PROJECT}/_apis/git/repositories/{REPO_ID}/pullRequests/{PR_ID}/threads?api-version=7.1
```

### Comment Template:

```markdown
## {SEVERITY} | {CATEGORY}

### Issue
In `{MethodOrClassName}`: {CLEAR_DESCRIPTION}

### Suggestion
{ACTIONABLE_RECOMMENDATION}

### Personal Pattern
{Reference to your historical commenting pattern}

---
_Reviewed using your patterns via HEXA Review Persona_
```

### Step 12: Confirmation

```
Posted {N}/{N} comments to PR #{PR_ID} (using your review persona):
  - {FILE}:{LINE} - {Issue title} (Thread {ID})

View PR: https://dev.azure.com/{ORG}/{PROJECT}/_git/{REPO}/pullrequest/{PR_ID}
```
