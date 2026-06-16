---
description: "Guidance for GitHub Copilot when working on the dotnet/winforms repository."
---

# GitHub Copilot Development Environment Instructions

This document provides specific guidance for GitHub Copilot when working on the dotnet/winforms repository. It serves as context for understanding the project structure, development workflow, and best practices.

## Repository Overview

**Windows Forms (WinForms)** is a UI framework for building Windows desktop applications with .NET. This repository contains the runtime libraries for `System.Windows.Forms`, `System.Drawing`, `System.Windows.Forms.Design`, and related assemblies.

### Key Technologies

- **.NET SDK** — Version is **ALWAYS** defined in `global.json` at the repository root.
  - **main branch**: Tracks the latest .NET preview.
  - Verify: `cat global.json | grep -A2 "dotnet"` or `Get-Content global.json`.
- **MSBuild / Arcade build system** — `.\build.cmd` wraps `eng\common\Build.ps1`.
- **Testing frameworks**:
  - **xUnit v3** with **Microsoft.Testing.Platform** — test projects build as self-contained executables.
  - `dotnet test --filter` does **not** work. Use `--filter-method` / `--filter-class` passed after `--` or invoke the test executable directly.
- **Windows-only** — the runtime and test execution require Windows. Linux supports command-line restore/build only.

## Development Environment Setup

This guidance assumes:
- Repository is cloned and the repo-local SDK is set up (`.\Restore.cmd` completed).
- Visual Studio 2022 with workloads listed in `WinForms.vsconfig` is available for IDE workflows.
- The repo-local `.dotnet` SDK folder is on `PATH` (done automatically by `.\start-vs.cmd` / `.\start-code.cmd`).

## Project Structure

### Important Directories

```
src\
  System.Windows.Forms\               ← Core WinForms runtime
  System.Drawing.Common\              ← System.Drawing GDI+ wrapper
  System.Windows.Forms.Primitives\    ← Shared primitives (DPI, theming, interop)
  System.Windows.Forms.Design\        ← Designer / DesignSurface runtime
  System.Windows.Forms.Analyzers\     ← Roslyn analyzers
  test\unit\System.Windows.Forms\     ← Main unit test project
eng\                                  ← Build engineering (Arcade)
.github\                              ← GitHub configuration, skills, agents
docs\                                 ← Developer documentation
```

### Source Organization

- Controls live under `src\System.Windows.Forms\System\Windows\Forms\Controls\`.
- Interop (P/Invoke) is under `src\System.Windows.Forms.Primitives\System\Windows\Forms\Nativemethods\` and adjacent files.
- Dark-mode and visual theming code lives alongside the controls that use it.
- Designer support for a control is in `System.Windows.Forms.Design\`, mirroring the runtime path.

## Development Workflow

### Build

```powershell
# Full restore + build (required first time)
.\build.cmd

# Build a single project (fast inner-loop, after at least one full build)
dotnet build src\System.Windows.Forms\System.Windows.Forms.csproj

# Release build
.\build.cmd -configuration Release

# Create NuGet packages
.\build.cmd -pack
```

Refer to the `building-code` skill for complete details.

### Test

```powershell
# Run all unit tests
.\build.cmd -test

# Run tests for a single project (from test project directory)
pushd src\test\unit\System.Windows.Forms
dotnet test

# Run a single test method (preferred — invoke the test executable directly)
$env:DOTNET_ROOT = "$PWD\.dotnet"
$env:PATH = "$PWD\.dotnet;$env:PATH"
& "artifacts\bin\System.Windows.Forms.Tests\Debug\net11.0-windows7.0\System.Windows.Forms.Tests.exe" `
  --filter-method "System.Windows.Forms.Tests.ButtonTests.Button_AutoSizeModeGetSet"
```

Refer to the `running-tests` skill for complete details, including filter syntax and troubleshooting.

### Key Test Project Paths

| Test suite | Project directory |
|------------|-------------------|
| System.Windows.Forms.Tests | `src\test\unit\System.Windows.Forms` |
| System.Drawing.Common.Tests | `src\System.Drawing.Common\tests` |
| System.Windows.Forms.Primitives.Tests | `src\System.Windows.Forms.Primitives\tests\UnitTests` |
| System.Windows.Forms.Design.Tests | `src\System.Windows.Forms.Design\tests\UnitTests` |
| System.Windows.Forms.Analyzers.Tests | `src\System.Windows.Forms.Analyzers\tests\UnitTests` |

## Contribution Guidelines

### Handling Existing PRs for Assigned Issues

**🚨 CRITICAL REQUIREMENT: Always develop your own solution first, then compare with existing PRs.**

1. **Develop your own solution first** — Analyze the issue independently before looking at existing PRs.
2. **Search for existing PRs** — After developing your solution, search for open PRs addressing the same issue.
3. **Compare and evaluate** — Determine which solution better addresses the issue.
4. **Document your decision** — In your PR description, compare your solution with existing PRs and explain your choice.

### Auto-Generated Files (Never Commit)

These files are auto-generated and **must NOT be committed**:
- `cgmanifest.json` — Generated during CI builds.
- `*.Designer.cs` patterns under `eng\` — Build artifacts.

**For AI agents:** Always reset changes to these files before committing.

### PublicAPI Files

When adding new public APIs:
- Add entries to `PublicAPI.Unshipped.txt` in the relevant project.
- **Never disable analyzers** to bypass `PublicAPI.Unshipped.txt` issues.
- Refer to the `new-control-api` skill for full guidance.

### Branching

- `main` — Bug fixes and .NET preview work.
- For servicing backports, use the `release/X.0` branches (managed via the backport workflow).
- **NEVER commit directly to `main`** — always create a feature branch.

### Git Workflow (Copilot CLI Rules)

**🚨 CRITICAL Git Rules:**

1. **NEVER commit directly to `main`** — always create a feature branch:
   ```powershell
   git checkout -b fix/issue-12345
   ```

2. **When amending an existing PR**, check out the PR branch directly — do NOT create a new branch off a PR branch:
   ```powershell
   gh pr checkout 12345
   git add .
   git commit -m "fix: Description of the change"
   ```

3. **Do NOT rebase, squash, or force-push** unless explicitly requested.

4. **STOP and ask the user before pushing** after committing local changes.

**Safe Git Workflow:**
```powershell
# Create a feature branch (NEVER work directly on main)
git checkout -b fix/issue-12345

# Commit changes
git add .
git commit -m "fix: Description of change

Fixes #12345

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"

# Ask user before pushing
```

### Opening PRs

All PRs should follow the template in `.github/PULL_REQUEST_TEMPLATE.md`. The description must:
- Reference the issue being fixed (e.g., `Fixes #12345`).
- Summarize what changed and why.
- List any test changes.

### Documentation

- Update XML documentation for all public API changes.
- Follow existing code documentation patterns (see `coding-standards` skill).
- Update relevant docs in `docs\` when behavior changes.

## Coding Standards

Follow the conventions in the `coding-standards` skill, including:
- **C# 14** language features on **.NET 10** target.
- Nullable reference types enabled everywhere.
- `_camelCase` for instance fields, `s_camelCase` for statics.
- Allman brace style.
- Throw-helpers (`ArgumentNullException.ThrowIfNull`, etc.) instead of manual checks.
- XML documentation on all public and internal types.

For modernizing *existing* files, use the `code-modernization` skill.

## Custom Agents and Skills

The repository includes specialized custom agents and reusable skills for specific tasks.

### Skills vs Agents

| Aspect | Skills | Agents |
|--------|--------|--------|
| **Invoke** | Load in Copilot chat | Delegate to agent |
| **Output** | Instructions, recommendations | Actions, changes applied |
| **Example** | `building-code` → how to build | `fix-issue` → applies a fix |

### Available Custom Agents

1. **fix-issue** (`.github/agents/fix-issue.agent.md`)
   - **Purpose**: End-to-end agent for investigating and fixing a GitHub issue in dotnet/winforms.
   - **Use when**: "Fix issue #XXXXX", "Investigate and patch #XXXXX", "Apply a fix for #XXXXX".
   - **Workflow**: Reads issue → analyzes codebase → invokes `try-fix` skill → verifies tests → opens PR.
   - **Do NOT use for**: Code review only, documentation updates, or triaging issues.

### Available Skills

Skills are located in `.github/skills/`. Load them in Copilot chat or have agents invoke them.

#### Build & Test Skills

1. **building-code** (`.github/skills/building-code/SKILL.md`)
   - How to restore, build, and package the WinForms repository.

2. **running-tests** (`.github/skills/running-tests/SKILL.md`)
   - How to run unit and integration tests, filter individual tests, and interpret results.

3. **download-sdk** (`.github/skills/download-sdk/SKILL.md`)
   - How to install the required .NET preview runtime when test executables fail with "framework not found".

#### Code Generation Skills

4. **coding-standards** (`.github/skills/coding-standards/SKILL.md`)
   - C# 14 / .NET 10 coding conventions for **new** source files.

5. **code-modernization** (`.github/skills/code-modernization/SKILL.md`)
   - Rules for modernizing and refactoring **existing** source files.

6. **new-control-api** (`.github/skills/new-control-api/SKILL.md`)
   - How to add new public APIs (properties, methods, events) to WinForms controls.

#### Test-Writing Skills

7. **control-api-tests** (`.github/skills/control-api-tests/SKILL.md`)
   - Writing unit tests for new public APIs on WinForms controls.

8. **gdi-rendering-tests** (`.github/skills/gdi-rendering-tests/SKILL.md`)
   - Writing bitmap-based rendering tests for `System.Drawing` / GDI+.

#### Domain Skills

9. **using-and-extending-gdi-plus** (`.github/skills/using-and-extending-gdi-plus/SKILL.md`)
   - GDI/GDI+ patterns for drawing, imaging, and font APIs.

#### Workflow Skills

10. **try-fix** (`.github/skills/try-fix/SKILL.md`)
    - **Internal skill used by the `fix-issue` agent.** Proposes ONE fix approach, builds, tests, and reports results. Can be invoked directly to try a specific fix idea.
    - Trigger: "Try to fix issue #XXXXX", "Attempt a fix for..."

11. **issue-triage** (`.github/skills/issue-triage/SKILL.md`)
    - Query and triage open GitHub issues needing milestones or labels.
    - Trigger: "Triage issues", "Find issues needing attention".

### Delegation Policy

When a user request matches an agent trigger phrase, **delegate to that agent immediately**. Examples:

- "Fix issue #12345" → invoke **fix-issue** agent
- "Triage open issues" → invoke **issue-triage** skill
- "Build the project" → load **building-code** skill
- "Write a test for..." → load **control-api-tests** skill
