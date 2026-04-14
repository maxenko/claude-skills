---
name: rust-ci-green
description: "Makes Rust projects CI-green by fixing all cargo fmt and clippy issues and ensuring GitHub Actions workflows produce green badges. Use when user says 'make CI green', 'green badge', 'fix CI', 'CI passing', 'fix the build', 'fmt and clippy', 'cargo fmt clippy', 'make it pass CI', 'workflow green', or when a Rust project's GitHub Actions are failing due to formatting or lint issues. Also use when setting up Rust CI from scratch. Do NOT use for test failures unrelated to fmt/clippy, deployment pipelines, or non-Rust CI. For committing after fixes, use rust-commit instead."
allowed-tools: "Bash Read Edit Write Grep Glob"
argument-hint: "[focus: fmt | clippy | workflow | all]"
---

# Rust CI Green

You are an expert Rust CI engineer. Reason carefully about each step. Your single goal: every `cargo fmt --check` and `cargo clippy` step in CI passes, producing green badges.

## What This Does

1. **Audit** -- read project config, CI workflow, and toolchain settings
2. **Format** -- auto-fix all rustfmt issues
3. **Lint** -- auto-fix then manually fix all clippy warnings
4. **Verify** -- run the exact CI pipeline locally to confirm zero issues
5. **Harden** -- ensure workflow file, toolchain, and badge are correct
6. **Report** -- summarize what changed

## Input

Optional focus: `$ARGUMENTS`

- `fmt` -- only fix formatting issues
- `clippy` -- only fix clippy warnings
- `workflow` -- only audit/fix the GitHub Actions workflow
- Empty or `all` -- full pipeline (default)

If `$ARGUMENTS` is not one of the above, treat as `all` and note the unrecognized argument to the user.

### Phase routing

| Focus     | Audit | Format | Lint | Verify | Harden | Report |
|-----------|-------|--------|------|--------|--------|--------|
| `fmt`     | Yes   | Yes    | Skip | fmt only | Skip   | Yes    |
| `clippy`  | Yes   | Skip   | Yes  | clippy only | Skip   | Yes    |
| `workflow`| Yes   | Skip   | Skip | Skip       | Yes    | Yes    |
| `all`     | Yes   | Yes    | Yes  | both       | Yes    | Yes    |

## Context

- Branch: !`git branch --show-current 2>/dev/null || echo "no git"`
- Status: !`git status --short 2>/dev/null | head -20`
- Toolchain: !`rustc --version 2>/dev/null && rustup show active-toolchain 2>/dev/null`

---

## Phase 1: Audit

Read the project's configuration to understand what CI expects.

### 1a: Project structure

```bash
cat Cargo.toml
```

Determine and record these **resolved flags** -- use them consistently in all subsequent cargo commands:

- **Workspace?** (`[workspace]` present) -- add `--workspace`
- **Edition?** (2024 has different rustfmt defaults and match ergonomics)
- **MSRV?** (`rust-version` field -- if below 1.81, use `#[allow]` not `#[expect]`)
- **Features safe?** Only use `--all-features` if no mutually exclusive features. Signs of conflict: `compile_error!` in cfg blocks, CI jobs testing specific feature combos separately, `#[cfg(not(feature = "..."))]` impl blocks. When in doubt, use default features.
- **Workspace lints?** Check for `[workspace.lints.clippy]` -- if present, respect it. Do not add per-item attributes that contradict workspace lint policy.
- **Nested workspaces?** A repo can contain multiple independent Cargo workspaces (e.g., `desktop/win/Cargo.toml` with its own `[workspace]`). The root `[workspace]` does not cover these -- they require separate `cd` + cargo commands. Scan the CI workflow (Phase 1b) for `cd` steps to find them, or search for additional `Cargo.toml` files with `[workspace]` sections.

In all subsequent cargo commands, replace `[flags]` with the resolved flags from this step (e.g., `--workspace --all-features`).

### 1b: CI workflow

```bash
ls .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null
```

If workflow files exist, read them all. **CI is the ground truth** -- extract every `run:` step related to fmt, clippy, check, test, or doc. Record the **exact commands including `cd` paths, `&&` chains, flags, and environment variables**. These become the command templates you replay in Phases 2-4.

Pay special attention to:
- **`cd` commands**: CI may `cd` into subdirectories for nested workspaces. Each `cd` + cargo chain is a separate workspace root that needs its own pass.
- **Multiple jobs doing the same check in different directories**: This means the repo has multiple independent workspaces. Build a list of `(directory, cargo_command)` pairs.
- **Flags**: If CI uses `--all-features`, you use `--all-features`. If CI uses `-W clippy::pedantic`, you use that.
- **Environment variables**: If CI sets `RUSTFLAGS: "-Dwarnings"`, note this.

Build a **workspace root list** -- one entry per independent workspace the CI checks. For single-workspace projects this is just the repo root. For multi-workspace projects this is every directory CI `cd`s into.

If no workflow exists, note this for Phase 5 and assume the repo root is the only workspace.

### 1c: Toolchain configuration

```bash
cat rust-toolchain.toml 2>/dev/null || cat rust-toolchain 2>/dev/null
```

If a pinned toolchain exists, ensure your local toolchain matches. If the CI workflow specifies a different toolchain version than what's pinned locally, flag the mismatch -- this is the #1 cause of "green locally, red in CI."

### 1d: Formatting configuration

```bash
cat rustfmt.toml 2>/dev/null || cat .rustfmt.toml 2>/dev/null
```

Check for:
- **`style_edition`**: If the project uses edition 2024 in Cargo.toml but `rustfmt.toml` doesn't set `style_edition = "2024"`, bare `rustfmt` will use 2015 style while `cargo fmt` infers 2024. This causes editor-vs-CI divergence. Flag it.
- **Nightly-only options**: If `rustfmt.toml` uses unstable options (e.g., `group_imports`, `imports_granularity`) but CI runs stable, formatting will silently differ. These options require nightly rustfmt. Flag the mismatch.

### 1e: Clippy configuration

```bash
cat clippy.toml 2>/dev/null || cat .clippy.toml 2>/dev/null
```

Check `msrv` -- if set, it takes precedence over `rust-version` in Cargo.toml. The Cargo.toml `rust-version` is only used as a fallback when `clippy.toml` does not set `msrv`. If both are set and differ, flag the mismatch.

---

## Phase 2: Fix Formatting

Skip if `$ARGUMENTS` is `clippy` or `workflow`.

For each workspace root identified in Phase 1b, run the format-and-verify cycle. Use **absolute paths** for `cd` and chain commands with `&&` in a **single Bash call** -- never split `cd` and cargo into separate Bash invocations (the working directory does not persist between calls).

Note: `cargo fmt` uses `--all` (not `--workspace`) to format all packages. Do not substitute `[flags]` into fmt commands.

### 2a: Auto-format and verify

If CI commands were extracted in Phase 1b, replay them with the fix step prepended. For example, if CI does `cd desktop/win && cargo fmt --all --check`, run:

```bash
cd /absolute/path/to/desktop/win && cargo fmt --all && cargo fmt --all --check
```

If no CI workflow was found, use the default for each workspace root:

```bash
cd /absolute/path/to/workspace && cargo fmt --all && cargo fmt --all --check
```

If `--check` still fails, investigate:
- **Generated code not excluded**: Add `#[rustfmt::skip]` or exclude in `rustfmt.toml`
- **Nightly-only rustfmt options**: If `rustfmt.toml` uses unstable options and the local toolchain is stable, `cargo fmt` silently ignores them but CI might use nightly
- **Style edition mismatch**: See 1d above
- **Macro-generated code**: `rustfmt` can't format inside macros -- expected and not a CI issue
- **Edition 2024 import sorting**: Edition 2024 changed import sorting order. `cargo fmt` follows the edition from Cargo.toml. If the project wants to opt out, set `style_edition = "2021"` in `rustfmt.toml`

---

## Phase 3: Fix Clippy

Skip if `$ARGUMENTS` is `fmt` or `workflow`.

For each workspace root identified in Phase 1b, run the clippy fix cycle. Always use **absolute paths** for `cd` and chain dependent commands with `&&` in a **single Bash call**.

### 3a: Auto-fix

```bash
cd /absolute/path/to/workspace && cargo clippy --fix --allow-dirty --allow-staged --all-targets [flags] 2>&1 && cargo check 2>&1
```

Chain `clippy --fix` and `cargo check` together -- the check immediately verifies auto-fix didn't break the build (`clippy --fix` has known bugs that generate non-compiling code).

If `cargo check` fails after auto-fix, identify which files clippy broke from the error output. For each broken file, check `git diff <file>` first -- if it contains only clippy changes, revert with `git checkout -- <file>`. If it also contains pre-existing unstaged changes, manually undo only the clippy-introduced breakage to avoid data loss. Never `git checkout -- .` -- that reverts formatting fixes from Phase 2 and all pre-existing changes. Fix those warnings manually instead.

### 3b: Check remaining warnings

Use the **exact command CI uses**, replayed with absolute paths. If you identified it in Phase 1b, mirror it. Otherwise use the standard per workspace root:

```bash
cd /absolute/path/to/workspace && cargo clippy --all-targets [flags] -- -D warnings 2>&1
```

**Why `-- -D warnings` not `RUSTFLAGS="-Dwarnings"`**: The env var changes cargo's cache key, causing full rebuilds of all dependencies. Passing `-D warnings` after `--` goes directly to clippy without affecting the cache. Never mix the two approaches in the same session -- that also invalidates the cache.

**Exception**: `cargo doc` requires the env var (`RUSTDOCFLAGS="-D warnings"`) because it doesn't support `-- -D warnings`.

### 3c: Fix remaining warnings manually

Read each file before editing. For each warning, follow the tiered decision framework below (derived from the rust-warn-fix skill). Key principles:

- **Tier 1 (always fix):** mechanical lints like `unused_imports`, `needless_return`, `len_zero`, `unused_must_use`, any `correctness` lint. These have objectively correct fixes.
- **Tier 2 (fix when safe):** context-dependent lints like `dead_code`, `redundant_clone`, `too_many_arguments`. Use judgment.
- **Tier 3 (silence with `#[expect]`):** legitimate suppressions at FFI boundaries, trait impls, serde fields, platform-specific code.
- **Tier 4 (never silence):** `unconditional_recursion`, `invalid_value`, `dangling_pointers_from_locals`, etc. These indicate bugs.

**RAII footgun** -- when fixing `unused_variables` on guards/locks/handles, use `let _guard = expr;` (lives to end of scope), never bare `let _ = expr;` (drops immediately).

**Suppression syntax**: Prefer `#[expect(lint, reason = "...")]` over `#[allow]` (stabilized Rust 1.81). If MSRV is below 1.81, use `#[allow(lint)]` with a code comment. Narrowest scope possible. Never put `#[deny(warnings)]` in source code.

### 3d: Iterate

```bash
cargo clippy --all-targets [flags] -- -D warnings 2>&1
```

Fixing one warning can reveal new ones. Repeat until clean.

### 3e: Verify formatting still passes

Manual edits might introduce formatting issues:

```bash
cargo fmt --all --check
```

If it fails, run `cargo fmt --all` and continue.

---

## Phase 4: End-to-End Verification

Skip if `$ARGUMENTS` is `workflow`.

Phases 2-3 verify individual concerns during the fix loop. This phase re-runs the relevant CI commands together to catch cross-concern regressions (e.g., clippy fixes that re-break formatting).

**Replay the exact CI commands** from Phase 1b, for each workspace root, using absolute paths and `&&` chains in single Bash calls. Run only the checks matching the current focus:
- **`fmt`**: fmt check commands only
- **`clippy`**: clippy check commands only
- **`all`**: both, e.g., `cd /absolute/path && cargo fmt --all --check && cargo clippy --all-targets [flags] -- -D warnings`

If no CI workflow was found, use the defaults above.

Use extended timeouts (600000ms) for commands involving compilation.

**All workspace roots must pass.** If any fails, return to the relevant phase.

---

## Phase 5: Harden CI

Skip if `$ARGUMENTS` is `fmt` or `clippy`.

### 5a: Create or fix the workflow

If **no workflow exists**, create `.github/workflows/ci.yml` using the templates in `references/workflow-templates.md`. Adapt to the project's flags (workspace, features, MSRV, etc.).

If a workflow exists but has issues, fix them:

| Problem | Fix |
|---------|-----|
| Uses deprecated `actions-rs/*` | Replace with `dtolnay/rust-toolchain` |
| Missing `--all-targets` on clippy | Add it -- otherwise tests/examples/benches aren't linted |
| Uses `RUSTFLAGS: "-Dwarnings"` for clippy | Switch to `-- -D warnings` to avoid cache invalidation |
| No caching | Add `Swatinem/rust-cache@v2` **after** toolchain setup |
| Cache step before toolchain step | Move cache after toolchain -- rust-cache uses rustc version as cache key |
| Hardcoded toolchain version conflicting with `rust-toolchain.toml` | Remove explicit version; let `rust-toolchain.toml` govern |
| Missing `cargo fmt --check` step | Add it as the first check -- cheapest and fastest feedback |
| Clippy and fmt in same job | Split into separate jobs for parallel execution and clearer failure signals |
| Missing `components: clippy` or `components: rustfmt` | Add them to the toolchain step |

### 5b: Validate toolchain consistency

1. **Does `rust-toolchain.toml` exist?** If not and the project is collaborative, recommend creating one:
   ```toml
   [toolchain]
   channel = "stable"
   components = ["rustfmt", "clippy"]
   ```
   Including components ensures contributors get them automatically via rustup.

2. **Does the CI workflow specify a different toolchain than `rust-toolchain.toml`?** They should agree. `dtolnay/rust-toolchain` respects the file if no explicit version is provided.

3. **Does the project have an MSRV lower than the pinned toolchain?** That's fine for development, but suggest a separate CI job that tests the MSRV if the project claims to support it.

### 5c: Validate badge

```bash
grep -E 'badge\.svg|actions/workflows' README.md 2>/dev/null
```

If no badge exists, derive the URL from the git remote and suggest adding it:
```bash
git remote get-url origin 2>/dev/null
```

Badge format:
```markdown
[![CI](https://github.com/OWNER/REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/OWNER/REPO/actions/workflows/ci.yml)
```

If a badge exists but points to a nonexistent or renamed workflow file, fix the URL.

---

## Phase 6: Report

Provide a concise summary:

- **Fmt**: X files reformatted (or "already clean")
- **Clippy**: Y warnings fixed, Z suppressed with reason (or "already clean")
- **Workflow**: created / updated / already correct
- **Badge**: added / fixed / already present
- **Toolchain**: aligned / recommended pinning / already consistent

If all checks already passed, say so and stop. Do not manufacture work.

**This skill does not commit changes.** Use `/rust-commit` after reviewing the fixes if you want to commit.

---

## Guardrails

- **Replay CI commands verbatim.** Use the exact commands, flags, `cd` paths, and `&&` chains from the CI workflow. Convert relative paths to absolute. Do not improvise your own command structure.
- **Never split `cd` and cargo into separate Bash calls.** The working directory does not persist between Bash tool invocations. Always chain with `&&` in a single call: `cd /abs/path && cargo ...`
- **Cover every workspace root.** If CI checks multiple directories, you must check all of them. Missing a workspace root means missing its failures.
- **Only fix issues that appear in command output.** Do not invent warnings.
- **Read files before editing.** Never guess at line numbers or content.
- **Verify every fix compiles.** Run `cargo check` after non-trivial edits.
- **Grep before removing "dead" code.** Macros and conditional compilation hide usage.
- **Do not modify code unrelated to making CI green.** No refactoring, no extras.
- **If all checks already pass, say so.** Do not manufacture work.
