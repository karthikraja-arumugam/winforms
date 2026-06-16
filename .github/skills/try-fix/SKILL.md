---
name: try-fix
description: >-
  Attempts ONE fix approach for a WinForms bug, builds, tests it empirically,
  and reports the result. Always explores a DIFFERENT approach from any existing
  PR fix. Invoked by the fix-issue agent (Phase 5) or directly by the user.
  Requires: git, PowerShell, .NET SDK (repo-local), Windows.
metadata:
  author: dotnet-winforms
  version: "1.0"
---

# Try-Fix Skill

Attempts **one** fix for a given problem. Receives all context upfront, implements a single approach, tests it empirically, and reports what happened.

---

## Activation Guard

🚨 **This skill is ONLY for proposing and testing code fixes.** Do NOT activate for:
- Code-review requests ("review this PR", "check code quality")
- PR summaries ("what does this PR do?")
- Test-only requests ("run tests", "check CI status")
- General architecture or API questions

If the prompt does not include a **problem to fix** and a **way to verify the fix**, do not run.

---

## Core Principles

1. **Always run once activated** — the invoker decides when; you decide what.
2. **Single-shot** — each invocation = one fix idea, tested, reported.
3. **Alternative-focused** — always propose something **different** from any existing PR fix (review PR changes first).
4. **Empirical** — actually build and run tests; do not theorize.
5. **Context-driven** — work with the provided context and git history.

**Every invocation runs all workflow steps below.**

---

## ⚠️ Sequential Execution Only

🚨 **Try-fix runs MUST be sequential — NEVER parallel.**

Each run modifies the same source files and uses the same build output. Running in parallel causes file conflicts and unreliable results.

---

## Inputs

All inputs are provided by the invoker (agent or user).

| Input | Required | Description |
|-------|----------|-------------|
| Problem | Yes | Description of the bug / issue to fix |
| Test command | Yes | Command to build and run the relevant tests (see section below) |
| Target files | Yes | Files to investigate and change |
| Hints | Optional | Suggested approaches, prior attempts, or areas to focus on |

### Standard Test Commands

| Scenario | Command |
|----------|---------|
| Run tests for a specific class | `& "artifacts\bin\System.Windows.Forms.Tests\Debug\net11.0-windows7.0\System.Windows.Forms.Tests.exe" --filter-class "System.Windows.Forms.Tests.<ClassName>"` |
| Run tests matching a pattern | `& "artifacts\bin\System.Windows.Forms.Tests\Debug\net11.0-windows7.0\System.Windows.Forms.Tests.exe" --filter-method "*<Pattern>*"` |
| Run a single project's tests | `pushd src\test\unit\System.Windows.Forms; dotnet test` |
| Run all unit tests | `.\build.cmd -test` |
| Build only | `dotnet build src\System.Windows.Forms\System.Windows.Forms.csproj` |

> **Note:** The TFM directory (`net11.0-windows7.0`) changes with each major .NET version. Check `artifacts\bin\System.Windows.Forms.Tests\Debug\` for the actual directory name after building.

Always set up the repo-local runtime before running test executables:

```powershell
$env:DOTNET_ROOT = "$PWD\.dotnet"
$env:PATH = "$PWD\.dotnet;$env:PATH"
```

---

## Outputs

| Field | Description |
|-------|-------------|
| `approach` | What fix was attempted (brief description) |
| `files_changed` | Which files were modified |
| `result` | `Pass`, `Fail`, or `Blocked` |
| `analysis` | Why it worked, or why it failed and what was learned |
| `diff` | The actual code changes (`git diff`) |

---

## Completion Criteria

The skill is complete when:
- [ ] Problem understood from provided context.
- [ ] One fix approach designed and implemented.
- [ ] Fix built and tested (iterated up to 3 times if build/test errors).
- [ ] Either: tests **PASS** ✅, or all 3 iterations exhausted and failure documented ❌.
- [ ] Analysis recorded (success explanation or failure reasoning with evidence).
- [ ] Working directory restored to a clean state.
- [ ] Results reported to the invoker.

### What Counts as "Pass" vs "Fail"

| Scenario | Result |
|----------|--------|
| Test command runs and tests pass | ✅ **Pass** |
| Test command runs and tests fail | ❌ **Fail** |
| Code does not compile | ❌ **Fail** |
| Tests cannot be run (missing runtime, etc.) | ⚠️ **Blocked** |

**NEVER claim Pass based on:**
- ❌ "Code compiles" alone
- ❌ "Logic looks correct"
- ❌ "Approach is sound"

**Pass requires:** Test command executed AND reported test success.

**Exhaustion criteria:** Stop after 3 iterations if:
1. Tests consistently fail for the same root reason.
2. Root-cause analysis reveals the approach is fundamentally flawed.
3. A completely different strategy is required.

**Never stop because of:** Compile errors (fix them), missing imports (add them).

---

## Workflow

### Step 1 — Understand the Problem and Review Existing Fixes

1. **Check if an existing PR already addresses this issue:**

```powershell
git log --oneline origin/main..HEAD
git diff origin/main HEAD --name-only
```

Read the actual code changes in any modified files. Understand the current fix approach so you can propose something **different**.

2. **Review prior attempts** if any were provided in the hints. Note what failed and why.

3. **Identify your alternative approach:**
   - If existing fix adds a guard → consider fixing the underlying cause instead.
   - If existing fix changes the call site → consider fixing the implementation instead.
   - If existing fix modifies the default value → consider validating on assignment instead.

---

### Step 2 — Design the Fix

Design **one** specific fix approach that is different from any existing attempt.

Write a brief `approach.md` in your working notes:

```markdown
## Approach: <name>

<One paragraph describing what you will change and why it fixes the bug.>

**Different from existing fix:** <Why this is meaningfully different.>
```

---

### Step 3 — Establish a Baseline (Confirm Bug Exists)

Before making changes, confirm the test **fails** on the current state of the branch to establish a broken baseline:

```powershell
# Save current state
git stash

# Run the test — it should FAIL here
<test command>

# Restore changes
git stash pop
```

If the test **passes** on the baseline (bug cannot be reproduced), stop and report this to the invoker — do not proceed with a fix.

---

### Step 4 — Implement the Fix

Apply the code change described in Step 2. Follow all conventions in the `coding-standards` skill:
- C# 14 / .NET 10 idioms.
- Nullable reference types.
- Throw-helpers (`ArgumentNullException.ThrowIfNull`, etc.).
- XML documentation on any new public/internal types.
- No trailing whitespace; CRLF line endings; UTF-8 with BOM.

**Focus on the minimum change** that fixes the bug without touching unrelated code.

---

### Step 5 — Build

```powershell
# Fast build of only the changed project
dotnet build src\System.Windows.Forms\System.Windows.Forms.csproj
```

If there are compile errors, fix them and rebuild. This counts as one of your 3 iterations only if you also run the tests.

---

### Step 6 — Run Tests

```powershell
# Set up runtime
$env:DOTNET_ROOT = "$PWD\.dotnet"
$env:PATH = "$PWD\.dotnet;$env:PATH"

# Run the test command provided by the invoker
<test command>
```

Capture the full output. Record:
- Number of tests run, passed, failed, skipped.
- Failure messages and stack traces for any failures.

**Iteration loop (up to 3 total):**
- If tests fail due to a code logic error → adjust the fix and go back to Step 4.
- If tests fail due to a different bug → investigate whether it's pre-existing.
- If the approach is fundamentally wrong → stop, record the analysis, proceed to Step 8.

---

### Step 7 — Capture the Diff

```powershell
git diff
```

Save this as the definitive record of what the fix changes.

---

### Step 8 — Restore (if approach failed)

If the fix did not achieve `Pass` after 3 iterations, **revert all changes**:

```powershell
git checkout -- .
git clean -fd
```

Confirm the working directory is clean:

```powershell
git status
```

---

### Step 9 — Report Results

Report back to the invoker with all of the following:

```markdown
## Try-Fix Result

**Approach:** <name of fix approach>

**Files Changed:**
- `path/to/file.cs` — <brief description of change>

**Result:** Pass ✅ / Fail ❌ / Blocked ⚠️

**Test Output Summary:**
- Tests run: N
- Passed: N
- Failed: N
- Relevant failure messages (if any)

**Analysis:**
<If Pass: Why the fix works. Which root cause was addressed.>
<If Fail: Why the approach didn't work. What was learned. What to try next.>

**Diff:**
```diff
<paste git diff output>
```
```

If the result is `Fail` or `Blocked`, include actionable suggestions for what to try in the next attempt.

---

## Common WinForms Fix Patterns

These patterns cover the most frequent bug categories. Use them to guide your approach design in Step 2.

### 1. Null Reference / Uninitialized State

**Symptom:** `NullReferenceException` when accessing a control property before the handle is created, or after disposal.

**Fix pattern:** Guard with `IsHandleCreated` or `IsDisposed` before accessing native resources:

```csharp
if (!IsHandleCreated || IsDisposed)
{
    return;
}
```

### 2. DPI / Scaling Issues

**Symptom:** Controls render at wrong size on high-DPI displays; hardcoded pixel values.

**Fix pattern:** Use `LogicalToDeviceUnits()` or `DpiHelper` rather than raw integers:

```csharp
int scaledPadding = LogicalToDeviceUnits(4);
```

### 3. Event Not Raised / Raised Too Many Times

**Symptom:** A `PropertyChanged` event fires when the value hasn't changed, or does not fire when it has.

**Fix pattern:** Compare old and new values before raising:

```csharp
if (_myProperty == value)
{
    return;
}
_myProperty = value;
OnMyPropertyChanged(EventArgs.Empty);
```

### 4. Designer Serialization Issues

**Symptom:** Property does not serialize to the `.Designer.cs` file, or serializes when it shouldn't.

**Fix pattern:** Check `[DefaultValue]`, `[DesignerSerializationVisibility]`, and `ShouldSerialize*()` method implementations. See `new-control-api` skill.

### 5. Interop / Handle Lifetime Bugs

**Symptom:** Crash or silent failure when calling native Win32 APIs.

**Fix pattern:** Use `SendMessage` / `PostMessage` with correct `HWND` checks; ensure the control's handle exists before P/Invoking:

```csharp
if (IsHandleCreated)
{
    PInvoke.SendMessage(this, PInvoke.WM_SETTEXT, 0, text);
}
```

### 6. Thread-Safety / Cross-Thread Access

**Symptom:** `InvalidOperationException: Cross-thread operation not valid` or race conditions.

**Fix pattern:** Use `InvokeRequired` + `BeginInvoke`/`Invoke` pattern, or switch to `Control.BeginInvoke` for fire-and-forget updates:

```csharp
if (InvokeRequired)
{
    BeginInvoke(() => UpdateUI(data));
    return;
}
UpdateUI(data);
```

---

## Anti-Patterns to Avoid

| Anti-pattern | Why | Instead |
|---|---|---|
| `catch (Exception) { }` swallowing all errors | Hides bugs | Let exceptions propagate or log them |
| `Thread.Sleep()` to fix timing issues | Masks race conditions | Fix the synchronization properly |
| `GC.Collect()` to fix resource issues | Unreliable | Use `using` / `Dispose()` correctly |
| Touching `this` after `Dispose()` | Use-after-free | Guard with `_disposed` flag |
| Hardcoded pixel values | Breaks on high-DPI | Use `LogicalToDeviceUnits()` |
| Disabling Roslyn analyzers (`#pragma warning disable`) | Hides real issues | Fix the underlying warning |
