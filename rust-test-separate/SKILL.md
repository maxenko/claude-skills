---
name: rust-test-separate
description: "Separates inline #[cfg(test)] modules from Rust implementation files into dedicated test files. Use when user says 'separate tests', 'extract tests', 'move tests to separate files', 'split tests from implementation', 'reorganize rust tests', or asks about test file organization in Rust. Do NOT use for writing new tests, running tests, or non-Rust codebases."
allowed-tools: "Read Write Edit Glob Grep Bash"
argument-hint: "[path-to-rust-crate-or-file]"
---

# Rust Test Separator

You separate inline `#[cfg(test)] mod tests { ... }` blocks from Rust implementation files into dedicated test files, following the conventions used by tokio, bevy, and axum.

## Critical Principles

**1. Preserve the module hierarchy.** The extracted test module must remain a child of the implementation module so `use super::*` continues to work and private function access is retained. This means using the `foo.rs` + `foo/tests.rs` pattern (or `#[cfg(test)] mod tests;` declaration), NOT moving tests to `tests/` at the top level.

**2. Never move unit tests to `tests/` (integration test directory).** The `tests/` directory creates separate crates that cannot access private or `pub(crate)` items. Only integration tests belong there. This skill handles unit test extraction only.

**3. Always keep `#[cfg(test)]` on the module declaration.** After extraction, the implementation file must have `#[cfg(test)] mod tests;` — without `#[cfg(test)]`, the test file compiles into release builds.

**4. Don't separate small test modules.** If the `#[cfg(test)] mod tests` block is under ~150 lines, leave it inline. Extraction adds indirection without meaningful benefit. The user can override this with explicit instructions.

## Process

### Phase 1: Scan, Plan & Confirm

Determine the target scope from `$ARGUMENTS`:

- **Single file**: If `$ARGUMENTS` is a `.rs` file path, read that file directly. Skip glob/grep.
- **Directory or omitted**: Glob `src/**/*.rs` (relative to the argument path or cwd), then grep for `#[cfg(test)]` as a coarse filter.

**Filtering candidates.** The grep is a pre-filter. When reading each file, skip it if:
- It contains `#[cfg(test)] mod tests;` (semicolon, no braces) — already separated.
- The `#[cfg(test)]` is only on individual functions or `use` statements, not a `mod` block.
- The test module has a `#[path = "..."]` attribute — warn the user and skip (non-standard layout that extraction could break).

For each valid candidate with an inline `#[cfg(test)] mod ... { ... }` block, record:
- File path
- Estimated line count of the test module
- Whether the file already uses directory module form (`mod.rs` pattern or `foo.rs` + `foo/` directory)

Note: files in `src/bin/` are valid candidates — the same extraction mechanics apply.

**Crate root files (`lib.rs`, `main.rs`)**: Extracting from these creates `src/lib/tests.rs` or `src/main/tests.rs`, which is technically valid but unconventional. Flag these to the user with a note that most projects keep crate root tests inline, and only extract if the user explicitly confirms.

Present a table with the planned action:

```
| File | Test Lines | Case | Action |
|------|-----------|------|--------|
| src/parser.rs | 340 | A | Extract |
| src/lib.rs | 45 | - | Skip (small, crate root) |
| src/utils/mod.rs | 220 | B | Extract |
```

**Case definitions:**
- **Case A — File needs directory restructuring.** The file is a standalone `foo.rs` with no existing `foo/` directory. Extraction requires creating `foo/` and either converting `foo.rs` → `foo/mod.rs` (mod.rs convention) or keeping `foo.rs` and adding `foo/tests.rs` alongside it (Rust 2018+ convention).
- **Case B — Directory already exists.** The file is already `foo/mod.rs` or has an existing `foo/` directory. Just add `tests.rs` inside the existing directory.

**Action criteria:**
- **Extract**: Test block >= 150 lines (or user explicitly requested this file)
- **Skip**: Test block < 150 lines, user didn't specifically request it, or crate root file without explicit confirmation
- User can override any action before proceeding

Wait for user confirmation before executing.

### Phase 2: Execute

For each planned extraction, execute in this order:

**Step 1: Create the directory (if Case A)**
```bash
mkdir -p src/foo
```

**Step 2: Write the new test file**

Create `src/foo/tests.rs` (or appropriate path). Before creating, verify `tests.rs` doesn't already exist at that path. If it does:
- If it already contains the separated tests (idempotent re-run), skip this file.
- If it exists with different content, report to the user and stop — don't overwrite.

Write the file with the contents of the `mod tests` block. If the original block already contained `use super::*;`, preserve it as-is. If it did not, add `use super::*;` at the top.

Preserve:
- All `use` statements from within the `mod tests` block
- All `#[test]` functions exactly as they were
- All helper functions, fixtures, and constants from within the test module
- All inner attributes (e.g., `#![allow(unused)]`) from the test module
- All sub-modules within the test module
- The original ordering of items

Do NOT add:
- `#[cfg(test)]` at the top of the file (the `mod tests;` declaration in the parent handles this)
- Any module-level documentation that wasn't there before

**Step 3: Modify the implementation file**

Replace the entire `#[cfg(test)] mod tests { ... }` block with:
```rust
#[cfg(test)]
mod tests;
```

**Preserve outer attributes.** If the original `mod tests` had outer attributes (e.g., `#[allow(clippy::...)]`) above `#[cfg(test)]`, transfer them to the new declaration:
```rust
#[allow(clippy::needless_pass_by_value)]
#[cfg(test)]
mod tests;
```

**File restructuring (Case A only):**
- **Rust 2018+ convention** (default): Keep `foo.rs` in place, create `foo/tests.rs` alongside it. No file moves needed.
- **`mod.rs` convention**: Create `foo/` directory, write the edited implementation content to `foo/mod.rs`, then delete the original `foo.rs` only after verifying the new file exists. Update the parent module's `mod foo;` declaration if necessary.

**Step 4: Move `#[cfg(test)]` imports**

If the implementation file has `#[cfg(test)] use ...` statements *outside* the test module, move ALL of them into `tests.rs`. They are test-only by definition.

Path adjustments for these moved imports (NOT for imports that were already inside `mod tests`):
- `use crate::...` paths need no change (absolute)
- `use super::...` paths need one extra `super::` prefix (becoming `use super::super::...`) because from inside `tests.rs`, `super` resolves to the implementation module, not its parent

Imports that were already inside the `mod tests { }` block need zero path changes — they already resolved relative to the test module's scope.

**Step 5: Check `Cargo.toml`**

If Case A restructured a file (e.g., `src/foo.rs` → `src/foo/mod.rs`), check `Cargo.toml` for explicit `path` entries (in `[lib]`, `[[bin]]`, `[[example]]`, etc.) pointing to the old file path. Update any stale paths.

**Step 6: Verify**

After all extractions, run:
```bash
cargo check --tests 2>&1
```

If there are errors:
- Read the error output carefully
- Common fixes: adjust `use` paths, add `pub(crate)` to items that tests need across module boundaries, fix `super::` paths
- Apply fixes and re-check
- If a fix requires making a private item `pub(crate)`, flag this to the user — it may indicate the test should actually remain inline

Report the final result: which files were extracted, any visibility changes required, and the `cargo check` outcome. Optionally suggest running `cargo test` to confirm tests still pass at runtime.

## Module Convention Detection

Before any file moves, determine which module convention the project uses:

**`mod.rs` convention (traditional)**:
```
src/parser/mod.rs      <- module root
src/parser/lexer.rs    <- sub-module
```

**Non-`mod.rs` convention (Rust 2018+)**:
```
src/parser.rs          <- module root
src/parser/lexer.rs    <- sub-module
```

Check existing directory modules in the project. Use whichever convention is already in use. If the project has no directory modules yet, prefer the Rust 2018+ convention (keeping `foo.rs` alongside `foo/` directory) unless the user specifies otherwise.

## Edge Cases

- **Multiple `#[cfg(test)]` blocks in one file**: Merge into a single `tests.rs`, preserving all contents. If merged blocks contain duplicate item names, report the conflict to the user rather than silently breaking compilation.
- **Nested `#[cfg(test)]` modules** (e.g., `mod tests` containing `mod integration`): Preserve the nesting structure in `tests.rs`.
- **Test modules with names other than `tests`**: Preserve the original name. Use `#[cfg(test)] mod original_name;` in the declaration.
- **`#[cfg(test)]` on individual functions** (not in a module): Leave these alone — they're a different pattern.
- **`#[cfg(test)]` on `impl` blocks**: Leave these in the implementation file. These are test helper methods on the struct itself (e.g., builder methods for test fixtures) and should stay near the type definition.
- **Test modules using `use crate::...` instead of `use super::*`**: Fully qualified `use crate::...` paths need no adjustment after extraction. Only `use super::...` paths are affected by the module depth change, and only for imports moved from *outside* the test module (see Step 4).
- **Files that are already `tests.rs`**: Skip. Don't re-extract.
- **Workspace crates**: Process one crate at a time. If `$ARGUMENTS` points to a workspace root, ask which crate(s) to process.
- **`build.rs`**: Excluded — it is a separate build script compilation unit, not a library/binary module.

Consult `references/rust-test-module-reference.md` for visibility rules, empirical project data, and rare edge case patterns.
