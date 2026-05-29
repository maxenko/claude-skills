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

### 1a: Exhaustive crate discovery

**Do not skim.** You must enumerate *every* `Cargo.toml` in the repo and classify each one. Missing a single crate means its fmt/clippy issues will ship red.

Find every manifest (prune common build/vendor dirs):

```bash
find . \( -name target -o -name node_modules -o -name vendor -o -name .git \) -prune -o -name Cargo.toml -print
```

Also run a Glob fallback in case `find` is unavailable:

```
Glob: **/Cargo.toml
```

Read **every** `Cargo.toml` found. For each, record:

- **Has `[workspace]`?** -- it is a workspace root (independent or virtual manifest).
- **Has `[package]` only?** -- it is a standalone crate OR a workspace member.
- **Workspace member?** -- matched by a `members = [...]`/`default-members` glob of some ancestor workspace root, OR `[package].workspace = "../path"` set. Use the ancestor workspace's `members` globs to decide; do not assume directory nesting alone implies membership.
- **Excluded?** -- listed in the ancestor workspace's `exclude = [...]`. Excluded crates are *not* covered by `--workspace` and must be checked separately.

Build a **check-roots list** -- the minimal set of directories that, when each is visited with the appropriate cargo invocation, covers every crate exactly once:

1. Every directory containing a `Cargo.toml` with `[workspace]` (covered via `--workspace` from that directory).
2. Every standalone `Cargo.toml` **not** claimed by any workspace root above (covered by a direct `cargo` call in its directory -- no `--workspace`).
3. Every crate listed in a workspace's `exclude` array (treat as standalone).

Print the final check-roots list back to the user before proceeding (one line per root: `path -- workspace|standalone -- flags`). This is your contract: Phases 2-4 iterate over *this exact list*.

Verification -- the union of crates reachable from the check-roots list must equal the full set of `Cargo.toml` files discovered above. If any crate is unaccounted for, stop and re-classify before proceeding.

### 1a.1: Per-root resolved flags

For each check-root, record these **resolved flags** from its `Cargo.toml` -- use them consistently in all subsequent cargo commands for that root:

- **Workspace?** (`[workspace]` present) -- add `--workspace`. Standalone crates do **not** take `--workspace`.
- **Edition?** (2024 has different rustfmt defaults and match ergonomics)
- **MSRV?** (`rust-version` field -- if below 1.81, use `#[allow]` not `#[expect]`)
- **Features safe?** Only use `--all-features` if no mutually exclusive features. Signs of conflict: `compile_error!` in cfg blocks, CI jobs testing specific feature combos separately, `#[cfg(not(feature = "..."))]` impl blocks. When in doubt, use default features.
- **Workspace lints?** Check for `[workspace.lints.clippy]` -- if present, respect it. Do not add per-item attributes that contradict workspace lint policy.

Flags are **per root**. A repo can have one workspace using `--all-features` and a standalone tool using default features -- keep them separate. In subsequent cargo commands, replace `[flags]` with the flags resolved for *that root*.

### 1b: CI workflow

```bash
ls .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null
```

If workflow files exist, read them all. **CI is the ground truth** -- extract every `run:` step related to fmt, clippy, check, test, or doc. Record the **exact commands including `cd` paths, `&&` chains, flags, and environment variables**. These become the command templates you replay in Phases 2-4.

Pay special attention to:
- **`cd` commands**: CI may `cd` into subdirectories for nested workspaces or standalone crates. Each `cd` + cargo chain corresponds to a check-root.
- **Multiple jobs doing the same check in different directories**: Cross-check these against your Phase 1a check-roots list -- they must match.
- **Flags**: If CI uses `--all-features`, you use `--all-features`. If CI uses `-W clippy::pedantic`, you use that.
- **Environment variables**: If CI sets `RUSTFLAGS: "-Dwarnings"`, note this.

Reconcile the CI-derived roots with your Phase 1a check-roots list:
- **CI covers a root you didn't find** -- add it to the list and re-verify your crate enumeration.
- **You found a crate CI does not check** -- still check it locally (this skill makes *every* crate green), and flag the gap so Phase 5 can add a CI step for it.

If no workflow exists, note this for Phase 5. Phase 1a's check-roots list is authoritative.

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

### 1f: Platform skew detection (local vs CI runner)

`cargo clippy` only lints code reachable under the **active target's `cfg`**. Code inside `#[cfg(target_os = "linux")]` blocks is invisible to clippy on Windows, and vice versa. If CI runs on a different OS than your local box and the source has any `cfg(target_*)` attributes, you can be locally green and CI-red on the same commit.

Detect it:

1. **CI runner OS(s)**: from each job's `runs-on:` line (`ubuntu-latest`, `windows-latest`, `macos-latest`, or a matrix). Map to the rustc triple — `ubuntu-latest` → `x86_64-unknown-linux-gnu`, `windows-latest` → `x86_64-pc-windows-msvc`, `macos-latest` → `aarch64-apple-darwin` (M-series; older runners may be x86_64).
2. **Local host triple**: `rustc -vV | grep host:`
3. **Platform-cfg surface area** -- grep the workspace for cfg gates that branch on platform:
   ```bash
   rg -l '\bcfg(_attr)?\s*\(\s*(target_os|target_family|target_arch|target_env|target_pointer_width|target_endian|unix|windows)\b' --type rust
   ```
   Also include shorthand `#[cfg(unix)]` / `#[cfg(windows)]` and combinations like `any(...)` / `not(...)` / `all(...)`.

If CI runner OS differs from local host **and** the grep returns any files, mark the project as **platform-cfg-skewed**. Phase 4 will run cross-target clippy for the CI triple. Otherwise, host-target clippy is sufficient.

Common skewed lints that surface only on the "other" OS:
- `unused_attributes` from `#[ignore = "..."]` stacked under `#[cfg_attr(not(target_os = "..."), ignore = "...")]` (two `#[ignore]`s on the cfg-true side).
- `dead_code` / `unused_imports` for items used only in a platform branch the local target excludes.
- `clippy::needless_borrow` / `unnecessary_cast` differences between MSVC and GNU ABI types (rare, but happens with FFI).

---

## Phase 2: Fix Formatting

Skip if `$ARGUMENTS` is `clippy` or `workflow`.

Iterate over **every check-root from Phase 1a** -- workspace roots *and* standalone crates. Do not stop at the first root, and do not stop at the first failure; collect and fix for all roots, then re-verify all. Use **absolute paths** for `cd` and chain commands with `&&` in a **single Bash call** -- never split `cd` and cargo into separate Bash invocations (the working directory does not persist between calls).

Note: `cargo fmt` uses `--all` for workspace roots. For standalone crates, plain `cargo fmt` formats just that crate; `--all` is also safe. Do not substitute `[flags]` into fmt commands.

### 2a: Auto-format and verify

If CI commands were extracted in Phase 1b, replay them with the fix step prepended. For example, if CI does `cd desktop/win && cargo fmt --all --check`, run:

```bash
cd /absolute/path/to/desktop/win && cargo fmt --all && cargo fmt --all --check
```

If no CI workflow was found (or CI doesn't cover this root), use the default for each check-root:

```bash
cd /absolute/path/to/root && cargo fmt --all && cargo fmt --all --check
```

Run this for **every entry** in the Phase 1a check-roots list. Keep a tally -- roots attempted vs. roots in the list -- and do not exit Phase 2 until those numbers match.

If `--check` still fails, investigate:
- **Generated code not excluded**: Add `#[rustfmt::skip]` or exclude in `rustfmt.toml`
- **Nightly-only rustfmt options**: If `rustfmt.toml` uses unstable options and the local toolchain is stable, `cargo fmt` silently ignores them but CI might use nightly
- **Style edition mismatch**: See 1d above
- **Macro-generated code**: `rustfmt` can't format inside macros -- expected and not a CI issue
- **Edition 2024 import sorting**: Edition 2024 changed import sorting order. `cargo fmt` follows the edition from Cargo.toml. If the project wants to opt out, set `style_edition = "2021"` in `rustfmt.toml`

---

## Phase 3: Fix Clippy

Skip if `$ARGUMENTS` is `fmt` or `workflow`.

Iterate over **every check-root from Phase 1a** -- workspace roots *and* standalone crates. Run the full clippy fix cycle for each. Always use **absolute paths** for `cd` and chain dependent commands with `&&` in a **single Bash call**. Remember: standalone crates do *not* take `--workspace`; workspace roots do.

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

**Attribute-edit footgun**: rustfmt has its own opinions about how long-form attributes break across lines, and they're not always derivable from `max_width`. In particular `#[cfg_attr(cond, inner_attr = "...")]` with a multi-argument inner attribute is reflowed onto multiple lines even when the single-line form fits in 100 cols. After hand-editing any attribute (or any line that came out of an Edit / replace_all), re-run `cargo fmt --check` for that root *before* declaring the warning fixed — don't trust eyeballed line lengths.

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

**Replay the exact CI commands** from Phase 1b for each CI-covered root, plus run the defaults below for any Phase 1a check-root not covered by CI. Use absolute paths and `&&` chains in single Bash calls. Run only the checks matching the current focus:
- **`fmt`**: fmt check commands only
- **`clippy`**: clippy check commands only
- **`all`**: both, e.g., `cd /absolute/path && cargo fmt --all --check && cargo clippy --all-targets [flags] -- -D warnings`

If no CI workflow was found, use the defaults above for every check-root.

Use extended timeouts (600000ms) for commands involving compilation.

**Every check-root from Phase 1a must pass.** Before leaving Phase 4, print a per-root pass/fail table -- roots attempted must equal roots in the list, and every row must be green. If any fails, return to the relevant phase for that specific root.

### 4b: Cross-target clippy when platform-cfg-skewed

Skip if Phase 1f did **not** flag the project as platform-cfg-skewed.

Otherwise, for each distinct CI runner OS that differs from your host:

1. Install the target if missing:
   ```bash
   rustup target add <ci-triple>
   ```
2. Replay the CI clippy command(s) for each affected check-root with `--target <ci-triple>`:
   ```bash
   cd /absolute/path && cargo clippy --workspace --all-targets --target <ci-triple> [flags] -- -D warnings
   ```

**Native-dep cross-compile may fail** (cc-rs / `libsqlite3-sys` / `openssl-sys` / similar need a C cross-toolchain you probably don't have on Windows). Two fallbacks:

- **Mini-repro**: extract the smallest possible snippet exhibiting the cfg pattern into a fresh crate with no native deps and run cross-target clippy there. Sufficient for attribute / lint-eval skew (`unused_attributes`, `dead_code`, etc.) since those don't depend on linking.
  ```bash
  mkdir -p /tmp/cfg-check/src && cat > /tmp/cfg-check/Cargo.toml <<EOF
  [package]
  name = "cfg-check"
  version = "0.0.0"
  edition = "2021"
  publish = false
  EOF
  # paste the suspect cfg pattern into src/lib.rs, then:
  cd /tmp/cfg-check && cargo clippy --all-targets --target <ci-triple> -- -D warnings
  ```
- **Symmetric-by-construction fix**: when the fix replaces a stacked `#[attr]` + `#[cfg_attr(not(X), attr)]` with two mutually exclusive `#[cfg_attr(X, attr)]` + `#[cfg_attr(not(X), attr)]`, exactly one attribute resolves on every target by construction. No cross-target run can prove or disprove this -- the symmetry does. Document the reasoning in the commit message.

If both fallbacks are blocked, document the gap in the Phase 6 report so the user knows host-target verification did not cover everything.

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
- **Cross-target**: triple(s) verified, or "host-only — no platform cfg detected", or "skipped — native deps blocked cross-compile, mini-repro used / symmetry argued"
- **Workflow**: created / updated / already correct
- **Badge**: added / fixed / already present
- **Toolchain**: aligned / recommended pinning / already consistent

If all checks already passed, say so and stop. Do not manufacture work.

**This skill does not commit changes.** Use `/rust-commit` after reviewing the fixes if you want to commit.

---

## Guardrails

- **Replay CI commands verbatim.** Use the exact commands, flags, `cd` paths, and `&&` chains from the CI workflow. Convert relative paths to absolute. Do not improvise your own command structure.
- **Never split `cd` and cargo into separate Bash calls.** The working directory does not persist between Bash tool invocations. Always chain with `&&` in a single call: `cd /abs/path && cargo ...`
- **Enumerate every `Cargo.toml` in the repo, do not skim.** Use `find` (pruning `target`, `node_modules`, `vendor`, `.git`) plus a `**/Cargo.toml` Glob fallback. Classify each as workspace root, workspace member, standalone, or `exclude`d. Print the final check-roots list before Phase 2.
- **Cover every check-root.** Workspace roots (via `--workspace`), standalone crates (direct `cargo` in their directory), and `exclude`d crates (treated as standalone) all need their own fmt + clippy pass. Not just what CI already checks -- every crate in the folder tree.
- **Crate count invariant.** Roots processed in Phase 2/3/4 must equal the Phase 1a check-roots list. Report the tally; do not declare success otherwise.
- **Host-target clippy is not enough when CI runs on a different OS.** Code under `#[cfg(not(target_os = "your_host"))]` is invisible to your clippy run; the same lint may fire only on the CI runner. If Phase 1f flagged platform-cfg skew, Phase 4b is mandatory before declaring CI green. Common smoking gun: `unused_attributes` on `#[ignore]` stacked with `#[cfg_attr(not(X), ignore)]`.
- **Re-verify after every hand edit.** Any time you Edit/Write a Rust file — even outside the main fix loop, even to fix a CI failure surfaced by the user — the *last* thing before reporting done must be re-running fmt and clippy on the file's check-root. Edits made after Phase 4's pass-table are the most common source of "fixed it, pushed it, CI red again" — usually rustfmt reflowing a hand-typed attribute. No exceptions: every edit cycle ends with `cargo fmt --check && cargo clippy --all-targets [flags] -- -D warnings` for the affected root.
- **Only fix issues that appear in command output.** Do not invent warnings.
- **Read files before editing.** Never guess at line numbers or content.
- **Verify every fix compiles.** Run `cargo check` after non-trivial edits.
- **Grep before removing "dead" code.** Macros and conditional compilation hide usage.
- **Do not modify code unrelated to making CI green.** No refactoring, no extras.
- **If all checks already pass, say so.** Do not manufacture work.
