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
`/hexa:pr-review <PR_ID>` or `/hexa:pr-review <PR_ID> <ORG>`

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

### 5. Review & Categorize

**ONLY TWO CATEGORIES - Keep it simple:**

1. **Critical** - Must be fixed before merging (blocking):
   - Security vulnerabilities
   - Breaking changes / bugs
   - Data loss risks
   - Build failures

2. **Good to Have** - Improvements worth considering:
   - Performance optimizations
   - Code quality improvements
   - Best practice violations
   - Maintainability issues

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

### Critical Issues (Blocking)
#### [{ID}] {FILE}:{LINE}
**{Category}**: {Issue}
**Fix**: {Specific fix description}
*Historical Context: {Required - pattern from history OR best practice rationale}*

### Good to Have
#### [{ID}] {FILE}:{LINE}
**{Category}**: {Issue}
**Fix**: {Specific fix description}
*Historical Context: {Required - pattern from history OR best practice rationale}*

### Summary
- **Files Changed:** X
- **Historical Lookback:** X days
- **Critical:** X | **Good to Have:** X
- **Overall Risk:** Low | Medium | High
- **Recommendation:** Approve | Request Changes | Needs Discussion
```

**IMPORTANT:** Assign each issue a unique ID like `[C1]`, `[C2]` for Critical and `[G1]`, `[G2]` for Good to Have. These IDs are used in the push selection UI.

---

## Step 7: Push Comments to PR (Batched Multi-Select)

> **MANDATORY - NEVER SKIP THIS STEP!** Always present the push options to the user, even if no issues were found.

After showing the review, present a multi-select interface for pushing comments to the PR.

### Tool Constraints
- `AskUserQuestion` supports **maximum 4 options per question**
- When more than 3 issues exist per category, use **batched questions**
- Always include a "Skip" option **at the top**

### Batching Strategy

**Batch by category**, showing one batch at a time:

1. **Critical batches first** (if any) - Show ALL critical issues across multiple batches
   - 1-3 issues: 1 batch with issues + "Skip Critical" at top
   - 4-6 issues: 2 batches
   - 7+ issues: As many batches as needed

2. **Good to Have batches** (if any) - Same batching rules
   - Split into sub-batches if more than 3 issues

3. **Final confirmation** - Show all selected issues before pushing

### Example Questions

**If 2 Critical + 4 Good to Have issues exist:**

**Batch 1 - Critical:**
```
question: "Select Critical comments to push to PR"
header: "Critical"
multiSelect: true
options:
  - label: "Skip Critical"
    description: "Don't push any Critical comments"
  - label: "[C1] File.cs:42 - SQL injection"
    description: "Use parameterized query"
  - label: "[C2] Auth.cs:15 - Missing null check"
    description: "Add null validation"
```

**Batch 2 - Good to Have (1/2):**
```
question: "Select Good to Have comments to push (1/2)"
header: "GoodToHave"
multiSelect: true
options:
  - label: "Skip Good to Have"
    description: "Don't push any Good to Have comments"
  - label: "[G1] Service.cs:100 - Use TryGetValue"
    description: "Avoid double dictionary lookup"
  - label: "[G2] Dao.cs:55 - Structured logging"
    description: "Use log parameters instead of interpolation"
  - label: "[G3] Helper.cs:200 - Remove unused column"
    description: "Remove LeftFileLineNumber from projection"
```

**Batch 3 - Good to Have (2/2):**
```
question: "Select Good to Have comments to push (2/2)"
header: "GoodToHave"
multiSelect: true
options:
  - label: "Skip remaining"
    description: "Don't push remaining comments"
  - label: "[G4] Query.cs:80 - Add LIMIT clause"
    description: "Prevent unbounded result sets"
```

**Final Confirmation:**
```
question: "Push {N} selected comments to PR #{PR_ID}?"
header: "Confirm"
multiSelect: false
options:
  - label: "Yes, push comments"
    description: "Push all selected comments to the PR"
  - label: "No, cancel"
    description: "Don't push any comments"
  - label: "Re-select"
    description: "Go back and change selection"
```

### Behavior Rules

1. **Skip empty categories** - Don't show batch if no issues in that category
2. **Track selections across batches** - Accumulate selected issues
3. **"Skip {Category}"** only skips that batch - continue to next
4. **Show confirmation before pushing** - List all selected comments
5. **If "Re-select"** - Restart from Batch 1
6. **If "No, cancel"** - End without pushing

---

## Step 8: Get ADO Token (if pushing)

```bash
powershell.exe -NoProfile -Command 'az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv'
```

Store as ADO_TOKEN. Reuse for ALL comment posts.

---

## Step 9: Push Selected Comments

**API Endpoint:**
```
POST https://dev.azure.com/{ORG}/{PROJECT}/_apis/git/repositories/{REPO_ID}/pullRequests/{PR_ID}/threads?api-version=7.1
```

**Thread Status:**
- `status = 1` = Active (for Critical - blocking issues)
- `status = 4` = Closed/Resolved (for Good to Have - non-blocking)

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

**PowerShell Escaping Rules:**
- Use single quotes for outer `-Command` argument
- NEVER use double quotes inside comment content
- Escape backticks by doubling them: ``` ``methodName`` ```

---

## Step 10: Confirmation

After pushing comments, show a summary:

**If all succeeded:**
```
Pushed {N}/{N} comments to PR #{PR_ID}:
  - [C1] File.cs:42 - SQL injection (Thread 123)
  - [G1] Service.cs:100 - Use TryGetValue (Thread 124)

View PR: https://dev.azure.com/{ORG}/{PROJECT}/_git/{REPO}/pullrequest/{PR_ID}
```

**If some failed:**
```
Pushed {X}/{N} comments to PR #{PR_ID}:

Succeeded:
  - [C1] File.cs:42 - SQL injection (Thread 123)

Failed:
  - [G1] Service.cs:100 - Use TryGetValue
    Error: {error message}

Troubleshooting:
- Check line number exists in new file version
- Verify PR access permissions
```
