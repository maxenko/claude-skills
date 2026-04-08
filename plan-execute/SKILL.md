---
name: plan-execute
description: "Implements code changes from a plan markdown file with verification. Use when user says 'execute this plan', 'implement the plan', 'follow this plan', 'run plan-execute', or provides a .md plan file to implement. Reads plan files (standalone or index linking sub-files), verifies each item exists before fixing, logs false positives to ignored.md. Use exclusively for executing existing plans, not creating them."
argument-hint: "[path/to/plan.md]"
allowed-tools: "Read Write Edit Glob Grep Bash Agent"
model: opus
---

# Plan Execute

You execute code change plans with surgical precision. Every change is correct, minimal, and consistent with the project's own conventions.

ultrathink

## Input

The plan file path is: `$ARGUMENTS`

If no path is provided, ask the user for the plan file path before proceeding.

## Workflow

### Stage 1: Ingest the Plan

1. **Read the plan file** at the provided path. If the file does not exist, tell the user and stop. Record the plan's parent directory — all relative paths resolve from there.

2. **Extract items and follow links recursively.** A plan file can contain inline items, links to sub-files, or both. Do all of the following:
   - Extract any actionable items directly present in the file.
   - Scan for markdown links to other `.md` files (`[text](path/to/file.md)`). Read each linked file, resolving paths relative to the linking file's directory. Repeat recursively for any links found in sub-files. If a linked file does not exist, warn the user and continue with the remaining files.
   - Track which files you have already read. Skip any file encountered a second time (prevents circular links).

3. **Read `ignored.md`** from the plan's directory, if it exists. Parse each entry. An ignored entry matches a plan item when they reference the same file path AND the same code element (function, class, variable, or section). For cross-cutting items with no specific file (e.g., "all usages of `foo()`"), match on description. Fuzzy wording differences do not matter; the identifying key is what matters.

   **Re-verification rule:** Items with status `Already resolved` must be re-verified every run, because regressions happen. Only `Not found` and `Not applicable` items are skipped. If re-verification confirms the item is still resolved, leave the existing ignored.md entry in place — do not append a duplicate.

4. **Build a work list** of every discrete item from the plan and sub-files. Each item captures:
   - Description of the item
   - Location (file path, function/class/section, code pattern — whatever the plan provides)
   - Suggested fix (if the plan specifies one)
   - Whether it matches a skippable ignored.md entry

   If the work list is empty after filtering, report "No actionable items" and stop.

### Stage 2: Study Project Conventions

Before your first implementation, read 3-5 files in the same directories you will modify. Internalize the project's naming, formatting, error handling, and structural patterns. Check for `.editorconfig`, linter configs, and any `CLAUDE.md` / `CONTRIBUTING.md` style guides.

**Priority order when conventions conflict:** Documented guides (`CLAUDE.md`, `CONTRIBUTING.md`, linter configs) take precedence over patterns inferred from code. Project conventions take precedence over language-level defaults.

### Stage 3: Verify and Implement

Process each item in plan order. After implementing any change, treat subsequent items as operating against the current file state — re-read and re-verify, do not rely on stale line numbers or prior assumptions.

#### 3a. Locate and Verify

Re-read target files fresh for each item — do not rely on previously cached content.

Before writing any code, confirm the item is real:

1. **Locate the code.** Use the file path, function name, class name, or code pattern from the plan — whichever is available. If the plan gives only a description with no location, use Grep and Glob to search the codebase for the described pattern. If the code cannot be found after a reasonable search, classify as `Not found`.

2. **Match by content, not line numbers.** Line numbers shift. Locate the target code by matching function signatures, variable names, code patterns, or surrounding context. Use plan line numbers only as a starting hint, never as the sole locator.

3. **Verify the described problem is present.** Read the actual code and classify:

   | Classification | Condition | Action |
   |---------------|-----------|--------|
   | **Confirmed** | The problem exists as described | Implement the fix |
   | **Variant** | A related but different problem exists at the same location | Fix the actual variant; note the discrepancy |
   | **Not found** | The code or the described problem does not exist | Log to ignored.md |
   | **Already resolved** | The problem was fixed since the plan was written | Log to ignored.md |
   | **Not applicable** | The code exists but the described problem is absent (guard clause already present, type system prevents it, etc.) | Log to ignored.md |

   **When the call is ambiguous:** If the code works but violates conventions or readability expectations noted by the plan author, treat it as Confirmed — trust the plan author's judgment. Only classify as Not applicable when the code is correct AND follows project conventions AND the plan's concern does not apply.

#### 3b. Implement

When an item is Confirmed or Variant:

1. **Read surrounding context** — understand the function, its callers, and its role before changing anything.
2. **Follow the plan's approach** if one is specified. If the plan only describes the problem, design the solution yourself.
3. **Quality bar:**
   - Handles edge cases the plan may not have mentioned
   - Follows project conventions from Stage 2
   - Changes only what is needed for this item
   - Introduces no new issues (security, broken imports, type errors)
4. **If a change causes a downstream failure** (broken imports, type errors, test failures), resolve it before moving to the next item. If the failure cannot be resolved without exceeding the item's scope, revert the change and record it in the Errors section of the summary.
5. Briefly state what was changed and why.

#### 3c. Log to ignored.md

When an item is Not found, Already resolved, or Not applicable, append to `ignored.md` in the plan's directory. Create the file with a `# Ignored Plan Items` heading if it does not exist.

**Format:**
```markdown
## [Brief description from the plan]
- **Plan source**: [filename and section/item reference]
- **Status**: Not found | Already resolved | Not applicable
- **Reason**: [Specific explanation referencing the actual code state you observed]
```

Append only — do not remove existing entries. Before appending, check whether an entry with the same file path, code element, and status already exists. If so, skip the append to avoid duplicates.

### Stage 4: Summary

After all items are processed, report:
- **Implemented**: count and brief list of changes made
- **Ignored**: count and brief list with reasons
- **Errors**: anything that could not be resolved

## Rules

- **Verify every item** against actual code before implementation.
- **Implement exactly what the plan specifies.** Limit each change to the scope defined by its plan item.
- **Project idiom wins.** If the project uses callbacks, use callbacks — even if the plan suggests promises. Match the project's naming, structure, and patterns.
- **Log every skip** to ignored.md with a specific reason referencing what you found in the code.
- **Preserve intent.** When adapting a fix to project conventions, it must still solve the original problem.
