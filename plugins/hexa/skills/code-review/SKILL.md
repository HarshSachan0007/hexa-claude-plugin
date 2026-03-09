---
name: code-review
description: HEXA Code Review - Review local uncommitted changes using git diff and historical comments
allowed-tools: Bash, Read, AskUserQuestion, Edit
argument-hint: "[--fix]"
---

# HEXA Code Review (Local)

> **Context Isolation:** Ignore prior conversation. Follow only these steps.

Review uncommitted local changes (staged + unstaged + untracked) before committing. Optionally fix issues interactively.

## Usage

`/hexa:code-review` - Review only
`/hexa:code-review --fix` - Review and offer to fix issues

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

### 3. Get HEXA Token

```bash
az account get-access-token --resource "api://9a6fc9ff-370c-4fee-868e-7d6b09034369" --query accessToken -o tsv
```

Store as HEXA_TOKEN. Reuse for all HEXA API calls.

> **If token fails**: Run `az login --scope api://9a6fc9ff-370c-4fee-868e-7d6b09034369/.default`

### 4. Get Historical Comments

```bash
curl -s -H "Authorization: Bearer {HEXA_TOKEN}" \
  "https://engsys-skynet-preprod.azurewebsites.net/api/ado/pullrequest/files/comments?filePathsString={FILES}&repositoryName={REPO}&currentPullRequestId=0&organization=domoreexp&lookBackDays=60"
```

### 5. Review & Categorize

**ONLY TWO CATEGORIES - Keep it simple:**

1. **Critical** - Must fix before committing:
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
- Be selective - only report significant issues
- Cross-reference with historical comments for patterns
- If no issues found: Output "No issues found."

### 6. Output Format

```
## HEXA Code Review: {BRANCH}
**Local Changes**: {COUNT} files

### Critical Issues
#### [{ID}] {FILE}:{LINE}
**{Category}**: {Issue}
**Fix**: {Specific fix description}
*Context: {Why this matters / historical pattern}*

### Good to Have
#### [{ID}] {FILE}:{LINE}
**{Category}**: {Issue}
**Fix**: {Specific fix description}
*Context: {Why this matters / historical pattern}*

### Summary
Critical: X | Good to Have: X
```

**IMPORTANT:** Assign each issue a unique ID like `[C1]`, `[C2]` for Critical and `[G1]`, `[G2]` for Good to Have. These IDs are used in the fix selection UI.

---

## Step 7: Interactive Fix Selection (TUI)

> **MANDATORY** - Always offer to fix issues after review, even if just to confirm "no fixes needed".

After displaying the review, present a multi-select interface for the user to choose which issues to fix locally.

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

3. **Final confirmation** - Show all selected issues before applying fixes

### Example Questions

**If 2 Critical + 4 Good to Have issues exist:**

**Batch 1 - Critical:**
```
question: "Select Critical issues to fix"
header: "Critical"
multiSelect: true
options:
  - label: "Skip Critical"
    description: "Don't fix any Critical issues"
  - label: "[C1] File.cs:42 - SQL injection"
    description: "Use parameterized query"
  - label: "[C2] Auth.cs:15 - Missing null check"
    description: "Add null validation"
```

**Batch 2 - Good to Have (1/2):**
```
question: "Select Good to Have issues to fix (1/2)"
header: "GoodToHave"
multiSelect: true
options:
  - label: "Skip Good to Have"
    description: "Don't fix any Good to Have issues"
  - label: "[G1] Service.cs:100 - Use TryGetValue"
    description: "Avoid double dictionary lookup"
  - label: "[G2] Dao.cs:55 - Structured logging"
    description: "Use log parameters instead of interpolation"
  - label: "[G3] Helper.cs:200 - Remove unused column"
    description: "Remove LeftFileLineNumber from projection"
```

**Batch 3 - Good to Have (2/2):**
```
question: "Select Good to Have issues to fix (2/2)"
header: "GoodToHave"
multiSelect: true
options:
  - label: "Skip remaining"
    description: "Don't fix remaining issues"
  - label: "[G4] Query.cs:80 - Add LIMIT clause"
    description: "Prevent unbounded result sets"
```

**Final Confirmation:**
```
question: "Apply {N} selected fixes?"
header: "Confirm"
multiSelect: false
options:
  - label: "Yes, apply fixes"
    description: "Apply all selected fixes to local files"
  - label: "No, cancel"
    description: "Don't apply any fixes"
  - label: "Re-select"
    description: "Go back and change selection"
```

### Behavior Rules

1. **Skip empty categories** - Don't show batch if no issues in that category
2. **Track selections across batches** - Accumulate selected issues
3. **"Skip {Category}"** only skips that batch - continue to next
4. **Show confirmation before applying** - List all selected fixes
5. **If "Re-select"** - Restart from Batch 1
6. **If "No, cancel"** - End without applying fixes

---

## Step 8: Apply Selected Fixes

For each selected issue, use the `Edit` tool to apply the fix.

### Fix Application Process

1. **Read the file** - Use `Read` tool to get current content
2. **Identify the exact code** - Find the `old_string` to replace
3. **Apply the fix** - Use `Edit` tool with precise `old_string` and `new_string`
4. **Verify** - Confirm the edit was applied

### Edit Tool Usage

```
Edit(
  file_path: "/absolute/path/to/file.cs",
  old_string: "exact code to replace",
  new_string: "fixed code"
)
```

**CRITICAL Rules:**
- `old_string` must be **unique** in the file - include enough context
- Preserve exact indentation (tabs/spaces)
- Don't add features beyond the fix
- Keep fixes minimal and focused

---

## Step 9: Fix Summary

After applying fixes, show a summary:

**If all succeeded:**
```
Applied {N}/{N} fixes:
  [C1] File.cs:42 - SQL injection -> Fixed
  [G1] Service.cs:100 - Use TryGetValue -> Fixed

Run `git diff` to review changes before committing.
```

**If some failed:**
```
Applied {X}/{N} fixes:

Succeeded:
  [C1] File.cs:42 - SQL injection

Failed:
  [G1] Service.cs:100 - Use TryGetValue
    Error: old_string not found (code may have changed)

Troubleshooting:
- Re-run review to get updated line numbers
- Apply failed fixes manually
```
