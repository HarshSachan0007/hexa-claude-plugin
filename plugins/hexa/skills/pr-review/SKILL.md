---
name: pr-review
description: HEXA PR Review - Review a remote ADO Pull Request using historical comments and patterns
allowed-tools: Bash, Read, AskUserQuestion
argument-hint: "<PR_ID> [ORG]"
---

# HEXA PR Review (Historical Expertise Agent)

> **Context Isolation:** Ignore prior conversation. Follow only these steps.

## Goal
Use historical PR comments to draw patterns and insights for reviewing code. Cross-reference current changes with past feedback to identify recurring issues, bad patterns, or regressions.

## Usage
`/hexa-pr-review <PR_ID>` or `/hexa-pr-review <PR_ID> <ORG>`

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

**IMPORTANT:** Always use `-NoProfile` flag to avoid PowerShell profile interference issues.

```bash
powershell.exe -NoProfile -Command 'az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv'
```

Store as HEXA_TOKEN. Reuse for all HEXA API calls.

> **If token fails or returns empty:**
> 1. Run `powershell.exe -NoProfile -Command 'az login --scope api://9a6fc9ff-370c-4fee-868e-7d6b09034369/.default'`
> 2. After login completes, retry the token fetch
> 3. Continue with the review - do not stop

### 3. Get File Diffs

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/pullrequest/diffs?pullRequestId={PR_ID}&organization={ORG}" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

### 4. Get Historical Comments - Progressive Lookback

Get comments from PRs that modified the same files.

**File Path Formatting (CRITICAL):**
- **ALWAYS use commas (`,`) to separate file paths** - Example: `/src/File1.cs,/src/File2.cs`
- **NEVER use pipes (`|`)**
- URL-encode the comma-separated list

**Progressive Lookback (60 -> 90 -> 180 -> 365 days):**

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; $files = "{URL_ENCODED_FILES}"; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/pullrequest/files/comments?filePathsString=$files&repositoryName={REPO}&currentPullRequestId={PR_ID}&organization={ORG}&lookBackDays=60" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

**Rules:**
- Start with 60 days
- If empty response, extend: 60 -> 90 -> 180 -> 365
- Stop when comments are found
- Note in output which lookback period was used

### 5. Review Guidelines

**Categories:** Critical, Security, Performance, CodeQuality, BestPractices

**Severity Categorization:**
1. **Critical Fixes** - Must be fixed before merging (blocking)
2. **Good to Have Fixes**:
   - **High** - Important for code quality/reliability
   - **Medium** - Maintainability and best practices
   - **Low** - Minor improvements

**Guidelines:**
- Cross-reference code changes with historical comments
- Focus on impactful, non-obvious issues
- Skip if the comment is already made in the current PR

**Historical Reasoning (REQUIRED for each issue):**
- If pattern was flagged before: explicitly state it with reference
- If no direct precedent: state "No direct historical precedent found, but this follows best practices for [reason]"

### 6. Output Format

```
## PR Review: {TITLE}
**PR #{ID}** | {AUTHOR} | {SOURCE} -> {TARGET}

### Critical Fixes (Blocking)
#### {FILE}:{LINE}
**{Category}**: {Issue description and suggested fix}
*Historical Context: {Required - pattern from history OR best practice rationale}*

### Good to Have
#### [{High|Medium|Low}] {FILE}:{LINE}
**{Category}**: {Issue description and suggested fix}
*Historical Context: {Required - pattern from history OR best practice rationale}*

### Summary
- **Files Changed:** X
- **Historical Lookback:** X days
- **Blocking Issues:** X
- **High Priority Suggestions:** X
- **Overall Risk:** Low | Medium | High
- **Recommendation:** Approve | Request Changes | Needs Discussion
```

---

## Step 7: Push Comments to PR (Batched Multi-Select)

> **MANDATORY - NEVER SKIP THIS STEP!** Always present the push options to the user.

After showing the review, present multi-select options for pushing comments by severity.

### Step 8: Get ADO Token (if pushing)

```bash
powershell.exe -NoProfile -Command 'az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv'
```

### Step 9: Push Selected Comments

**API Endpoint:**
```
POST https://dev.azure.com/{ORG}/{PROJECT}/_apis/git/repositories/{REPO_ID}/pullRequests/{PR_ID}/threads?api-version=7.1
```

**Thread Status:**
- `status = 1` = Active (for Critical/High)
- `status = 4` = Closed/Resolved (for Medium/Low)

### Comment Template:

```markdown
## {SEVERITY} | {CATEGORY}

### Issue
In `{MethodOrClassName}`: {CLEAR_DESCRIPTION}

### Suggestion
{ACTIONABLE_RECOMMENDATION}

### Historical Context
{REQUIRED: Why this matters}

---
_Automated review by HEXA_
```

### Step 10: Confirmation

```
Posted {N}/{N} comments to PR #{PR_ID}:
  - {FILE}:{LINE} - {Issue title} (Thread {ID})

View PR: https://dev.azure.com/{ORG}/{PROJECT}/_git/{REPO}/pullrequest/{PR_ID}
```
