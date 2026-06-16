---
name: issue-triage
description: >-
  Queries and triages open GitHub issues in dotnet/winforms that need
  attention — milestones, labels, or investigation. Presents issues one
  at a time and applies decisions via the GitHub CLI. Requires: gh CLI
  installed and authenticated.
metadata:
  author: dotnet-winforms
  version: "1.0"
---

# Issue Triage Skill

This skill helps triage open GitHub issues in the **dotnet/winforms** repository by:

1. Loading open issues that lack milestones or need labeling.
2. Presenting issues **one at a time** with a suggested action.
3. Applying the user's triage decision via `gh`.
4. Tracking progress through a session.

---

## Prerequisites

**GitHub CLI (`gh`) must be installed and authenticated:**

```powershell
# Install (Windows, via winget)
winget install --id GitHub.cli

# Authenticate (required before first use)
gh auth login
```

---

## When to Use

- "Find issues to triage"
- "Let's triage WinForms issues"
- "Grab me some issues to look at"
- "Triage issues without milestones"
- "Show me old issues"

---

## Triage Workflow

### Step 1 — Initialize Session

Fetch the current milestones for `dotnet/winforms` so you can suggest them accurately:

```powershell
gh api repos/dotnet/winforms/milestones --jq '.[] | {title: .title, open_issues: .open_issues}'
```

Display the results to orient the session. **Always use actual milestone names** from this output — never guess.

### Step 2 — Load Issues

Query open issues without a milestone, excluding those that need more information:

```powershell
gh issue list `
  --repo dotnet/winforms `
  --state open `
  --limit 50 `
  --json number,title,labels,milestone,createdAt,comments,url,author `
  --jq '[.[] | select(.milestone == null) | select(.labels | map(.name) | contains(["needs-info","needs-repro"]) | not)]'
```

Store the results for one-at-a-time presentation.

### Step 3 — Present ONE Issue at a Time

Present each issue in this format:

```markdown
## Issue #XXXXX

**[Title]**

🔗 [URL]

| Field | Value |
|-------|-------|
| **Author** | username |
| **Labels** | label1, label2 |
| **Area** | area-xxx (from labels) |
| **Age** | created date |
| **Comments** | count |

**My Suggestion**: `Milestone` — Reason

---

What would you like to do with this issue?
```

### Step 4 — Wait for User Decision

Wait for the user to say:
- A milestone name (e.g., "Backlog", "Future") → apply that milestone
- "yes" or "accept" → apply the suggestion
- "skip" or "next" → move on without changes
- A label name → add that label
- "close" → close the issue (ask for a reason)

### Step 5 — Apply the Decision

```powershell
# Set milestone
gh issue edit <NUMBER> --repo dotnet/winforms --milestone "Backlog"

# Add a label
gh issue edit <NUMBER> --repo dotnet/winforms --add-label "needs-info"

# Set milestone AND add label
gh issue edit <NUMBER> --repo dotnet/winforms --milestone "Future" --add-label "area-controls"

# Close with comment
gh issue close <NUMBER> --repo dotnet/winforms --comment "Closing as..."
```

### Step 6 — Continue

Automatically move to the next issue. When the batch is exhausted, reload more issues:

```powershell
gh issue list `
  --repo dotnet/winforms `
  --state open `
  --limit 50 `
  --skip <current_count> `
  --json number,title,labels,milestone,createdAt,comments,url,author `
  --jq '[.[] | select(.milestone == null) | select(.labels | map(.name) | contains(["needs-info","needs-repro"]) | not)]'
```

Do **not** stop and ask "Load more?" — just load automatically.

---

## Milestone Suggestion Logic

> **Always use actual milestone names from the `gh api` output in Step 1.**

| Condition | Suggested Milestone | Reason |
|-----------|---------------------|--------|
| Issue has a linked PR with a milestone | PR's milestone | "Linked PR already has milestone" |
| Issue labeled `regression` | Nearest servicing milestone | "Regression — needs servicing attention" |
| Issue has an open linked PR | Current servicing milestone | "Has open PR" |
| Enhancement request, no PR | Future | "Enhancement, no active work" |
| Default / unclear | Backlog | "No PR, not a regression" |

---

## WinForms Label Quick Reference

### Area Labels

| Label | Area |
|-------|------|
| `area-controls` | General WinForms controls |
| `area-data` | Data binding, DataGridView |
| `area-designer` | WinForms Designer |
| `area-drawing` | System.Drawing / GDI+ |
| `area-dialogs` | Common dialogs (FileOpen, etc.) |
| `area-accessibility` | Accessibility / MSAA / UIA |
| `area-dpi` | High-DPI and scaling |
| `area-threading` | Cross-thread / async |
| `area-interop` | P/Invoke / native interop |

### Status Labels

| Label | Meaning |
|-------|---------|
| `needs-info` | Needs more information from author |
| `needs-repro` | Cannot reproduce; repro case needed |
| `regression` | Confirmed regression from a previous version |
| `good first issue` | Suitable for new contributors |
| `help wanted` | Community help welcome |
| `duplicate` | Duplicate of another issue |

---

## Applying Common Triage Actions

```powershell
# Needs more info
gh issue edit <NUMBER> --repo dotnet/winforms --add-label "needs-info"
gh issue comment <NUMBER> --repo dotnet/winforms --body "Could you provide a minimal reproduction case?"

# Needs repro
gh issue edit <NUMBER> --repo dotnet/winforms --add-label "needs-repro"

# Mark as regression
gh issue edit <NUMBER> --repo dotnet/winforms --add-label "regression" --milestone "Current Servicing"

# Good first issue
gh issue edit <NUMBER> --repo dotnet/winforms --add-label "good first issue" --milestone "Backlog"

# Close as duplicate
gh issue close <NUMBER> --repo dotnet/winforms --comment "Closing as duplicate of #<OTHER>."
```

---

## Common Mistakes to Avoid

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| Suggesting milestone names like "SR2" without checking | Milestone may not exist | ✅ Use actual names from `gh api` output |
| Stopping when the batch is empty | More issues likely available | ✅ Reload automatically with increased offset |
| Applying labels to issues that need-info first | May mislabel | ✅ Set `needs-info` and wait for response |
| Closing issues without comment | Poor contributor experience | ✅ Always leave a closing comment |

---

## Session Tracking (Optional)

Keep a running count of triaged issues in the current session and report a summary when the user ends the session:

```
Session Summary:
  Issues triaged: N
  Milestones applied: N
  Labels applied: N
  Issues closed: N
```
