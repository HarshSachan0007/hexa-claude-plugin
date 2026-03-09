---
name: review-persona-as
description: HEXA Review Persona As - Review PR as an EXPERT would, learning from their historical comment patterns
allowed-tools: Bash, Read, AskUserQuestion
argument-hint: "<PR_ID> <EXPERT_EMAIL> [ORG]"
---

# HEXA Review Persona As (Expert Mode)

> **Context Isolation:** Ignore prior conversation. Follow only these steps.

## Goal
Review a PR as a specific EXPERT would review it. This skill fetches the expert's historical PR comments and uses their patterns, priorities, and style to provide reviews as they would.

## Usage
`/hexa-review-persona-as <PR_ID> <EXPERT_EMAIL>` or `/hexa-review-persona-as <PR_ID> <EXPERT_EMAIL> <ORG>`

- `PR_ID`: The pull request ID to review
- `EXPERT_EMAIL`: The email/UPN of the expert whose review style to emulate
- `ORG`: Organization name (default: `domoreexp`)

**Example:**
```
/hexa-review-persona-as 123456 john.smith@microsoft.com
/hexa-review-persona-as 123456 jane.doe@company.com domoreexp
```

## Steps

### 0. Pre-flight Auth Check (REQUIRED)

```bash
powershell.exe -NoProfile -Command 'az account show --query "[name, user.name]" -o tsv 2>$null; if (-not $?) { exit 1 }'
```

**If auth check fails:**
1. Run: `powershell.exe -NoProfile -Command 'az login'`
2. After login succeeds, continue with Step 1

### 1. Validate Expert Email

Store the provided `EXPERT_EMAIL` as `EXPERT_UPN`.

If no email provided, use AskUserQuestion:

```
question: "Whose review style would you like to emulate?"
header: "Expert"
options:
  - label: "Enter email manually"
    description: "Provide the expert's email/UPN"
```

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

### 5. Get EXPERT'S Historical Comments

Fetch the expert's past comments on similar files.

**Progressive Lookback (60 -> 90 -> 180 -> 365 days):**

```bash
powershell.exe -NoProfile -Command '$token = az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv; $files = "{URL_ENCODED_FILES}"; $expert = "{EXPERT_UPN}"; Invoke-RestMethod -Uri "https://engsys-skynet-preprod.azurewebsites.net/api/ado/kusto/personal-historical-comments?organization={ORG}&reviewerUniqueName=$expert&filePaths=$files&lookBackDays=60" -Headers @{Authorization = "Bearer $token"} | ConvertTo-Json -Depth 20'
```

**If no historical comments found after 365 days:**
- Inform the user: "No historical comments found for {EXPERT_EMAIL} on these files."
- Ask if they want to continue with general best practices review or try another expert.

### 6. Analyze Expert's Review Patterns

From the expert's historical comments, identify:
- **Their focus areas**: What types of issues do they typically flag?
- **Their style**: Concise or detailed? Code examples?
- **Their priorities**: What do they consider critical vs nice-to-have?
- **Technical depth**: Surface-level or deep architectural concerns?

### 7. Review Using Expert's Persona

Apply the expert's historical patterns to review the current PR:

**Guidelines:**
- **Emulate their style**: Write comments as THEY would write them
- **Focus on THEIR priorities**: Emphasize issues they historically care about
- **Match their tone**: Formal, casual, direct, encouraging?

**Severity (based on expert's patterns):**
1. **Critical Fixes** - Issues the expert would block a PR for
2. **Good to Have**: High / Medium / Low

### 8. Output Format

```
## PR Review: {TITLE}
**PR #{ID}** | {AUTHOR} | {SOURCE} -> {TARGET}
**Reviewed as:** {EXPERT_NAME} ({EXPERT_EMAIL})
**Based on:** {N} historical comments over {X} days

### Expert Review Patterns Detected
- **Primary focus:** {e.g., "Security, input validation, error handling"}
- **Style:** {e.g., "Detailed with code examples, formal tone"}
- **Typical concerns:** {e.g., "Thread safety, null checks, logging"}
- **Historical comments analyzed:** {N}

### Critical Fixes (Blocking)
#### {FILE}:{LINE}
**{Category}**: {Issue description in EXPERT'S voice}
*Expert's pattern: {Reference to how they've flagged similar issues}*

### Good to Have
#### [{High|Medium|Low}] {FILE}:{LINE}
**{Category}**: {Issue description in EXPERT'S voice}
*Expert's pattern: {Reference to similar past feedback}*

### Summary
- **Files Changed:** X
- **Expert's Historical Context:** {N} comments over {X} days
- **Blocking Issues:** X
- **Overall Risk:** Low | Medium | High
- **Recommendation:** Approve | Request Changes | Needs Discussion

---
*This review emulates the review style of {EXPERT_NAME} based on their historical PR feedback.*
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

### Expert Pattern
{Reference to how the expert has flagged similar issues}

---
_Reviewed as {EXPERT_NAME} via HEXA Review Persona_
```

### Step 12: Confirmation

```
Posted {N}/{N} comments to PR #{PR_ID} (as {EXPERT_NAME}):
  - {FILE}:{LINE} - {Issue title} (Thread {ID})

View PR: https://dev.azure.com/{ORG}/{PROJECT}/_git/{REPO}/pullrequest/{PR_ID}
```
