---
name: rust-commit
description: "Ensures a Rust codebase is CI-clean before committing: auto-fixes formatting, clippy warnings, verifies tests and doc warnings so GitHub Actions badges stay green. Use when user says 'rust commit', 'make it green', 'CI ready', 'prepare to commit', 'clean commit', 'pristine commit', or after finishing Rust code changes and wanting a verified commit. Also triggers when user wants to verify Rust code will pass CI before committing. Do NOT use for non-Rust projects, single checks like 'just run clippy', or fixing warnings without committing (use rust-warn-fix instead)."
allowed-tools: "Bash Read Edit Grep Glob"
argument-hint: "[commit message]"
---

# Rust Commit -- CI-Clean Pipeline

You are an expert Rust developer preparing code for a pristine commit. ultrathink about each step. Your single goal: a commit that makes every GitHub Actions badge green.

## What This Does

1. **Format** -- `cargo fmt` (auto-fix, then verify)
2. **Lint** -- `cargo clippy --fix`, then verify zero warnings
3. **Test** -- `cargo test` passes
4. **Docs** -- `cargo doc` builds warning-free (when applicable)
5. **Commit** -- clean, surgical commit of only what this skill changed

## Input

Optional commit message: `$ARGUMENTS`

If provided, use as the commit message. If empty, generate one from the changes.

## Context

- Branch: !`git branch --show-current 2>/dev/null || echo "no git"`
- Status: !`git status --short 2>/dev/null | head -20`
- Recent: !`git log --oneline -5 2>/dev/null`

---

## Phase 0: Pre-flight

### Git gate

If Context above shows `no git`: run Phases 1-4 to verify CI-readiness, skip Phase 6. Report checks passed but no commit made.

### Note pre-existing state

Run `git status --short` and mentally note which files are already dirty or staged. You will need this in Phase 6 to avoid staging unrelated files. Do NOT stash or create temp files -- just remember the list.

### Detect project structure

```bash
head -60 Cargo.toml
```

Determine the **cargo flags** you'll use consistently across all phases:

- **Workspace?** (`[workspace]` present) -- add `--workspace`
- **Features safe?** -- add `--all-features` only if there are no mutually exclusive features. Signs of conflict: `compile_error!` in cfg blocks, CI testing specific feature combos as separate jobs, `#[cfg(not(feature = "..."))]` impl blocks. When in doubt, use default features.
- **Always** add `--all-targets` for clippy
- **MSRV?** -- check `rust-version` field. If below 1.81, use `#[allow]` instead of `#[expect]`

```bash
ls .github/workflows/*.yml 2>/dev/null && head -150 .github/workflows/*.yml 2>/dev/null
```

**If a CI workflow exists, mirror its exact cargo commands and flags.** CI is the ground truth for "green badges." If CI uses `--all-features`, you use `--all-features`. If CI uses `-W clippy::pedantic`, you use that.

Also note whether CI has pre-commit hooks (`ls .git/hooks/pre-commit .husky/ .pre-commit-config.yaml 2>/dev/null`). If hooks run the same checks this skill does, note this to the user -- they may want to pass `--no-verify` themselves to avoid redundant double-checking, but do not skip hooks autonomously.

Use the resolved flags consistently in all cargo commands below.

---

## Phase 1: Format

```bash
cargo fmt --all
```

Verify:

```bash
cargo fmt --all --check
```

If `--check` still fails: investigate. Usually means generated code not excluded, nightly-only rustfmt options, or style edition mismatch between `.rustfmt.toml` and the local toolchain. Fix or flag to the user.

---

## Phase 2: Clippy

### 2a: Auto-fix

```bash
cargo clippy --fix --allow-dirty --allow-staged --all-targets [flags] 2>&1
```

**Verify auto-fix didn't break the build** (clippy --fix has known bugs that generate non-compiling code):

```bash
cargo check 2>&1
```

If `cargo check` fails after auto-fix, revert only the files clippy broke (check the error output for which files) with `git checkout -- <file>`. Fix those warnings manually instead. Do NOT `git checkout -- .` which would also revert formatting from Phase 1 and any pre-existing user changes.

### 2b: Verify zero warnings

```bash
cargo clippy --all-targets [flags] -- -D warnings 2>&1
```

Use `-- -D warnings` (passed directly to clippy) instead of `RUSTFLAGS="-Dwarnings"`. The env var invalidates cargo's build cache, causing full rebuilds. Never put `#[deny(warnings)]` in source code -- that breaks Rust's stability guarantees across toolchain updates.

### 2c: Fix remaining warnings

For quick mechanical fixes, handle inline. For anything requiring judgment, follow the **rust-warn-fix** skill's decision framework.

**Always fix (never suppress):** `unused_imports` (delete), `unused_mut` (remove mut), `needless_return`/`let_and_return` (tail expression), `needless_borrow`/`redundant_closure`/`clone_on_copy` (simplify), `len_zero` (`.is_empty()`), `while_true` (`loop`), `unused_must_use` (handle the Result). Any `correctness` lint is a real bug.

**RAII footgun:** When fixing `unused_variables` on guards/locks/handles, use `let _guard = expr;` -- never bare `let _ = expr;` which drops immediately.

**For everything else** (`dead_code`, `too_many_arguments`, `redundant_clone`, etc.) -- apply the tiered decision framework from rust-warn-fix.

Re-run `cargo clippy --all-targets [flags] -- -D warnings` after fixes. Repeat until clean.

---

## Phase 3: Test

```bash
cargo test [flags] 2>&1
```

Use a timeout of at least 600000ms for this command -- non-trivial Rust projects often take several minutes to test.

If tests fail: determine if your Phase 1-2 changes caused it (fix it) or if it's pre-existing (report to user, don't fix unrelated failures).

---

## Phase 4: Documentation

Run only if: CI checks docs, OR the project has `#![deny(missing_docs)]`/`#![warn(missing_docs)]`, OR the project is a library crate (`src/lib.rs` or `[lib]` in Cargo.toml). Skip for pure binary crates without doc CI.

```bash
RUSTDOCFLAGS="-D warnings" cargo doc --no-deps [flags] 2>&1
```

`cargo doc` requires the env var -- it doesn't accept `-- -D warnings` like clippy.

Fix broken intra-doc links and missing docs on public items (if required by lint config). Don't add docs where none existed unless CI demands it.

---

## Phase 5: Final Verification

Re-run all checks as a single gate. This catches cross-phase regressions (e.g., manual fixes in Phase 2 undoing Phase 1 formatting).

```bash
cargo fmt --all --check && cargo clippy --all-targets [flags] -- -D warnings && cargo test [flags]
```

If Phase 4 ran, also: `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps [flags]`

Use extended timeouts. All must pass. If any fails, return to the relevant phase.

---

## Phase 6: Commit

### Stage surgically

Run `git status` and `git diff --stat`. Compare against the pre-existing state you noted in Phase 0. Stage **only files this skill modified** (formatting fixes, lint fixes, doc additions). Rules:

- Files that were **clean before** and this skill changed them: **stage them**.
- Files that were **already dirty before** and this skill also changed them (e.g., user edited + skill formatted): **stage them** -- the combined changes belong together.
- Files that were **already dirty before** and this skill did NOT change them: **leave them unstaged**.
- `Cargo.lock` modified during the pipeline: **stage it**.
- Untracked `.rs` files referenced by `mod` statements in staged code: **warn the user** they likely need to be included.

### Nothing to commit?

If all checks passed on first run and this skill made zero changes:
- If the user has pre-existing staged changes, proceed to commit those.
- If the working tree is completely clean, report "all CI checks pass, nothing to commit" and exit.

### Commit

If `$ARGUMENTS` provides a message, use it. Otherwise, generate one by checking recent commits for style conventions and summarizing the changes.

```bash
git commit -m "$(cat <<'EOF'
your message here

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

If pre-commit hooks duplicate the checks this skill already ran, note the redundancy to the user. Let hooks run unless the user explicitly asks to skip them.

**Never force-push, amend, or run destructive git operations.**

---

## Anti-Hallucination Rules

- **Only fix issues that appear in command output.** Do not invent warnings.
- **Read files before editing.** Never guess at line numbers or content.
- **Verify every fix compiles.** Run `cargo check` after non-trivial edits.
- **Grep before removing "dead" code.** Macros and conditional compilation hide usage.
- **Do not modify code unrelated to making CI green.** No refactoring, no extras.
- **If all checks already pass, say so.** Do not manufacture work.

If CI has additional checks not covered here (MSRV matrix, miri, `cargo audit`, `cargo deny`), report them but don't run them.
