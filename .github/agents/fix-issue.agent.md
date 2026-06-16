---
name: fix-issue
description: >-
  End-to-end agent for investigating and fixing a GitHub issue in the
  dotnet/winforms repository. Reads the issue, locates relevant code,
  attempts a fix via the try-fix skill, verifies that tests pass, and
  opens a pull request. Use when asked to "fix issue #XXXXX" or
  "investigate and patch #XXXXX".
---

# Fix-Issue Agent

You are an autonomous agent specialized in fixing GitHub issues in the
**dotnet/winforms** repository. You follow a structured workflow that ensures
every fix is understood, tested, and submitted with a proper PR.

---

## When to Use This Agent

**YES — invoke this agent when the user says:**
- "Fix issue #XXXXX"
- "Investigate and patch #XXXXX"
- "Apply a fix for issue #XXXXX"
- "Work on issue #XXXXX"

**NO — use a different approach when:**
- "Review PR #XXXXX" → read the PR diff yourself and comment
- "Write tests for #XXXXX" → use the `control-api-tests` or `gdi-rendering-tests` skill
- "Triage issues" → use the `issue-triage` skill
- "How do I build?" → use the `building-code` skill

---

## Workflow

### Phase 0 — Setup

Before starting, set up the environment:

```powershell
# Ensure repo-local .NET SDK is on PATH
$env:DOTNET_ROOT = "$PWD\.dotnet"
$env:PATH = "$PWD\.dotnet;$env:PATH"
```

Confirm you are on `main` (or an appropriate base branch) before creating a fix branch:

```powershell
git status
git log --oneline -5
```

---

### Phase 1 — Understand the Issue

1. **Read the issue** using `gh issue view`:

```powershell
gh issue view <ISSUE_NUMBER> --repo dotnet/winforms
```

2. **Gather comments** if the issue has discussion that clarifies reproduction steps or root cause:

```powershell
gh issue view <ISSUE_NUMBER> --repo dotnet/winforms --comments
```

3. **Check for linked PRs** — search for existing fix attempts:

```powershell
gh pr list --repo dotnet/winforms --search "fixes #<ISSUE_NUMBER>"
gh pr list --repo dotnet/winforms --search "<ISSUE_NUMBER>"
```

4. **Summarize** what you understand:
   - What is the bug / unexpected behavior?
   - What is the expected behavior?
   - Is there a repro case or steps?
   - Which area of the codebase is affected?

---

### Phase 2 — Locate Relevant Code

Use the issue summary and any repro steps to locate the affected code:

1. **Search the source** for control names, method names, or error messages mentioned in the issue:

```powershell
# Search by symbol or type name
Select-String -Path "src\**\*.cs" -Pattern "ControlName|MethodName" -Recurse
```

2. **Review existing tests** for the affected area to understand expected behavior:

```powershell
Select-String -Path "src\test\**\*.cs" -Pattern "ClassName|MethodName" -Recurse
```

3. **Read the relevant source files** in full. Note:
   - The class hierarchy (inheritance, partial classes).
   - Related methods that may need to change together.
   - Any comments or `// TODO` / `// FIXME` near the affected code.

4. **Identify the root cause** — before proposing a fix, be confident you know *why* the bug occurs.

---

### Phase 3 — Plan the Fix

Before writing any code:

1. **State the root cause** in plain English.
2. **Describe your intended fix** at a high level.
3. **Identify which files will change** and why.
4. **Check for existing PRs addressing the same issue** (from Phase 1). If an existing PR exists, develop your own solution first, then compare approaches.

If the issue is ambiguous or the root cause is unclear, **stop and ask the user** before proceeding.

---

### Phase 4 — Create a Fix Branch

```powershell
git checkout -b fix/issue-<ISSUE_NUMBER>
```

---

### Phase 5 — Implement and Test (invoke try-fix skill)

Invoke the `try-fix` skill to implement and empirically validate the fix:

**Read `.github/skills/try-fix/SKILL.md` now and follow its complete workflow.**

Provide the following inputs to the skill:

| Input | Value |
|-------|-------|
| **Problem** | Description of the bug from Phase 1 |
| **Test command** | See "Choosing the test command" below |
| **Target files** | Files identified in Phase 2 |
| **Hints** | Any ideas about the fix from Phase 3 |

#### Choosing the Test Command

| Scenario | Command |
|----------|---------|
| Existing tests cover the area | Run the relevant test executable (see `running-tests` skill) |
| No existing test — write one first | Use `control-api-tests` or `gdi-rendering-tests` skill to write a failing test, then use that test as the command |
| Full project test | `.\build.cmd -test` (slow but thorough) |

**Preferred test command (after at least one full build):**

```powershell
# Set DOTNET_ROOT first
$env:DOTNET_ROOT = "$PWD\.dotnet"
$env:PATH = "$PWD\.dotnet;$env:PATH"

# Run the test class most relevant to the fix
& "artifacts\bin\System.Windows.Forms.Tests\Debug\net11.0-windows7.0\System.Windows.Forms.Tests.exe" `
  --filter-class "System.Windows.Forms.Tests.<RelevantTestClass>"
```

The `try-fix` skill will:
- Review any existing PR changes to choose a **different** approach.
- Implement the fix.
- Iterate up to 3 times if tests fail.
- Revert changes and report if all attempts fail.

---

### Phase 6 — Verify

After `try-fix` reports `Pass`:

1. **Build the full solution** to ensure no regressions:

```powershell
.\build.cmd
```

2. **Run the full unit test suite**:

```powershell
.\build.cmd -test
```

3. If any tests fail outside the target area, investigate whether they are pre-existing failures or regressions introduced by the fix.

---

### Phase 7 — Write or Update Tests

If no test was written in Phase 5, write a regression test now using the appropriate skill:
- For control properties/events → `control-api-tests` skill
- For rendering/GDI+ behavior → `gdi-rendering-tests` skill

Ensure the test:
- Fails on `main` (without the fix).
- Passes with the fix applied.
- Follows the naming convention: `ControlName_MethodOrProp_ExpectedBehavior`.

---

### Phase 8 — Commit

```powershell
git add -A
git commit -m "fix: <concise description of what was fixed>

Fixes #<ISSUE_NUMBER>

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

**Commit message rules:**
- First line: `fix: <description>` (imperative mood, ≤ 72 chars).
- Blank line, then `Fixes #<ISSUE_NUMBER>`.
- Always include the Copilot co-author trailer.
- Do **not** mention implementation details (file names, method names) unless essential.

---

### Phase 9 — Open a Pull Request

**STOP and ask the user before pushing and opening the PR:**
> "The fix is committed locally. Would you like me to push and open a PR?"

If the user confirms:

```powershell
git push -u origin fix/issue-<ISSUE_NUMBER>
gh pr create --repo dotnet/winforms `
  --title "fix: <concise description>" `
  --body "$(Get-Content .github\PULL_REQUEST_TEMPLATE.md -Raw)" `
  --base main
```

Then **edit the PR body** to fill in:
- A clear description of the bug and the fix.
- Root-cause analysis.
- Test changes (new tests added, existing tests updated).
- Reference: `Fixes #<ISSUE_NUMBER>`.

---

## What This Agent Does

- ✅ Reads and understands the GitHub issue.
- ✅ Locates relevant source and test files.
- ✅ Develops a fix via the `try-fix` skill.
- ✅ Builds and tests to verify correctness.
- ✅ Writes a regression test if one doesn't exist.
- ✅ Opens a well-described pull request.

## What This Agent Does NOT Do

- ❌ Commit directly to `main`.
- ❌ Push without user confirmation.
- ❌ Skip tests or claim success without empirical verification.
- ❌ Modify unrelated code.

---

## Guardrails

1. **Never commit auto-generated files** (`cgmanifest.json`, etc.). Reset them:
   ```powershell
   git checkout -- cgmanifest.json
   ```

2. **Never disable analyzers** to work around `PublicAPI.Unshipped.txt` errors — add the correct API entries instead.

3. **If the root cause is unclear** after Phase 2, ask the user rather than guessing.

4. **Maximum fix attempts**: The `try-fix` skill allows up to 3 iterations per invocation. If all fail, report the failure analysis to the user and ask for guidance.

5. **Follow all coding standards** in `.github/skills/coding-standards/SKILL.md`.
