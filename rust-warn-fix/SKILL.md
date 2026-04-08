---
name: rust-warn-fix
description: "Analyzes all Rust compiler and Clippy warnings, then intelligently fixes or silences each one using expert judgment. Use when user says 'fix warnings', 'clean up warnings', 'resolve clippy', 'zero warnings', 'handle warnings', 'warning cleanup', or when Rust build output shows warnings. Also use proactively when cargo build/clippy output contains warnings during development. Do NOT use for adding new lint rules, configuring CI pipelines, or Cargo.toml lint setup."
argument-hint: "[file path, module, or lint name to focus on]"
allowed-tools: "Bash Read Edit Grep Glob"
---

# Rust Warning Fixer

You are an expert Rust developer who resolves compiler and Clippy warnings with forward-thinking judgment. ultrathink about each warning before acting — your goal is zero warnings through intelligent fixes, not blanket suppression.

## Core Philosophy

**Fix first, silence only with justification.** Every silenced warning is technical debt with an expiration date. Every fixed warning is permanent improvement. When you silence, use `#[expect]` so the suppression self-destructs when no longer needed.

## Input

The optional argument is: `$ARGUMENTS`

If provided, it narrows scope to a specific file path, module name, or lint name. If empty, process all warnings in the project.

## Workflow

### Step 0: Pre-flight

Check project structure before running anything:

```bash
# Detect workspace vs single crate, check MSRV
grep -E 'rust-version|edition|\[workspace\]' Cargo.toml
```

- If `[workspace]` is present and `$ARGUMENTS` names a specific package, use `cargo clippy -p <package>` throughout.
- If the project has mutually exclusive features (look for `compile_error!` in feature gates), do NOT use `--all-features`. Use default features or the feature set documented in the project's CI/README.

### Step 1: Capture Warnings

```bash
cargo clippy --all-targets 2>&1
```

Add `--all-features` only if the project supports enabling all features simultaneously. Add `--workspace` for workspace projects when no specific package is targeted.

If clippy **fails with compilation errors** (not warnings), stop. Report the errors to the user — you cannot fix warnings in code that doesn't compile.

If `$ARGUMENTS` specifies a file or module, focus on warnings from that scope. If it specifies a lint name, filter to that lint.

### Step 2: Try Automatic Fixes First

Let Clippy fix what it can machine-apply:

```bash
cargo clippy --fix --allow-dirty --allow-staged --all-targets 2>&1 || true
```

**Verify the auto-fix didn't break anything** — `clippy --fix` has known bugs where it generates code that doesn't compile:

```bash
cargo check 2>&1
```

If `cargo check` fails after auto-fix, revert the broken changes (`git checkout -- <file>`) and fix those warnings manually instead.

Then re-run to see what remains:

```bash
cargo clippy --all-targets 2>&1
```

### Step 3: Categorize Each Remaining Warning

For every warning, classify it using the decision framework below, then apply the appropriate action.

### Step 4: Verify

After all changes, confirm zero warnings:

```bash
cargo clippy --all-targets 2>&1
```

If new warnings appeared (fixing one can reveal another), repeat from Step 3. Continue until clean.

---

## Decision Framework

For each warning, follow this priority order.

### Tier 1: ALWAYS FIX (never silence these)

These have mechanical, objectively correct fixes. Apply directly.

| Warning | Fix |
|---------|-----|
| `unused_imports` | Delete the import line |
| `unused_mut` | Remove `mut` keyword |
| `unused_must_use` | Handle the `Result`/`#[must_use]` return — do not discard silently |
| `while_true` | Replace `while true` with `loop` |
| `for_loops_over_fallibles` | Convert to `if let Some(x) = ...` (Option) or `if let Ok(x) = ...` (Result) |
| `path_statements` | Remove the bare path or use the value |
| `unreachable_code` | Remove dead code after the unreachable point |
| `redundant_semicolons` | Remove extra semicolons |
| `non_snake_case` / `non_camel_case_types` / `non_upper_case_globals` | Rename to follow convention. Grep for all usages and update them. If it's a public API, check if renaming is safe. |
| `clippy::needless_return` | Remove explicit `return`, use tail expression |
| `clippy::let_and_return` | Collapse into tail expression |
| `clippy::manual_map` | Use `.map()` |
| `clippy::single_match` | Convert to `if let` |
| `clippy::needless_borrow` | Remove `&` |
| `clippy::redundant_closure` | Pass the function directly (e.g., `\|x\| foo(x)` becomes `foo`) |
| `clippy::bool_assert_comparison` | Use `assert!(x)` or `assert!(!x)` |
| `clippy::clone_on_copy` | Remove `.clone()` on Copy types |
| `clippy::map_clone` | Use `.copied()` or `.cloned()` |
| `clippy::collapsible_if` | Merge nested `if` into single condition |
| `clippy::len_zero` | Use `.is_empty()` instead of `.len() == 0` |
| `clippy::needless_range_loop` | Use iterator instead of index loop |

### Tier 2: FIX WHEN SAFE (use judgment)

These require context awareness. Read the surrounding code before deciding.

| Warning | When to Fix | When to Silence |
|---------|------------|-----------------|
| `dead_code` | Remove if truly unreachable and not part of a public API being built incrementally | Silence if: public library API, FFI export, or incomplete feature under active development |
| `unused_variables` | Prefix with `_` if binding is structurally required (destructuring, trait impls). Remove if it serves no purpose | Silence only in trait impls where the signature is fixed and `_name` would be confusing |
| `deprecated` | Migrate to replacement API if straightforward | Silence with reason if migration requires significant refactoring or replacement is unstable |
| `unused_assignments` | Remove if value is truly never read | Silence if assignment exists for side effects or clarity |
| `clippy::redundant_clone` | Remove `.clone()` **and verify compilation** — this lint has known false positives (it's in the nursery category). Always run `cargo check` after removing a clone | Silence if removal causes borrow checker errors |
| `clippy::too_many_arguments` | Refactor into config/builder struct if there's a natural grouping | Silence if function mirrors an external API or FFI boundary |
| `clippy::type_complexity` | Create a type alias | Silence if type is used once and an alias would reduce readability |
| `clippy::cast_lossless` | Use `.into()` or `Type::from()` | Silence in FFI or numeric-heavy code where cast chains are clearer |
| `clippy::module_name_repetitions` | Rename if stutter is unhelpful (e.g., `user::UserConfig` to `user::Config`) | Silence if full name is more discoverable or module is re-exported |

### Tier 3: SILENCE WITH `#[expect]` (legitimate suppressions)

Valid reasons to suppress rather than fix. Always use `#[expect]` with a `reason`.

- **FFI boundaries**: Function signatures must match C headers exactly
- **Trait implementations**: Required method signatures where params are intentionally unused
- **Serialization fields**: Fields used only by serde/derive but appear "dead" to the compiler
- **Platform-specific code**: Variables used only under certain `#[cfg]` conditions
- **Performance-critical hot paths**: Where Clippy's "cleaner" suggestion would add overhead
- **Deliberate patterns**: Intentional unsafe, keeping code for clarity, matching external API naming

### Tier 4: NEVER SILENCE (always investigate and fix)

These indicate bugs or potential undefined behavior. Silencing hides real problems.

- `unconditional_recursion` — infinite recursion bug
- `unused_must_use` — discarded error/value that must be handled
- `invalid_value` — creating invalid memory states
- `dangling_pointers_from_locals` / `dangling_pointers_from_temporaries` — use-after-free
- `const_item_mutation` — mutation that has no effect
- `invalid_nan_comparisons` — comparison that is always false
- `dropping_copy_types` — drop that does nothing (logic error)
- `dropping_references` — usually a logic error (intended to drop the owned value, not the reference). Investigate whether the code intended `drop(owned_value)` instead
- Any warning in Clippy's `correctness` category
- `suspicious_double_ref_op` — likely logic error with double references

If thorough analysis confirms a false positive, document why with a `// SAFETY:` or `// NOTE:` comment and use `#[expect]` with a detailed reason.

---

## Suppression Syntax

**Always prefer `#[expect]` over `#[allow]`.** The `expect` attribute (stabilized in Rust 1.81) is strictly superior: it suppresses the warning like `allow`, but emits `unfulfilled_lint_expectations` when the warning disappears — meaning stale suppressions self-clean.

### Correct form

```rust
#[expect(dead_code, reason = "public API surface — callers exist in downstream crates")]
pub fn exported_helper() { ... }

#[expect(clippy::too_many_arguments, reason = "mirrors the C API signature from libfoo.h")]
fn ffi_wrapper(a: i32, b: i32, c: i32, d: i32, e: i32, f: i32, g: i32) { ... }
```

### Scoping rules

1. **Narrowest scope possible.** Put `#[expect]` on the specific item, not the module or crate.
2. **Never use `#![allow(warnings)]` or `#![allow(clippy::all)]` at crate root.** This hides real problems.
3. **For multiple items**: if 3+ items in the same module need the same suppression for the same reason, module-level is acceptable. Prefer per-item.

### MSRV below 1.81

If the project's MSRV is below 1.81 (check `rust-version` in Cargo.toml), `#[expect]` and the `reason` parameter are unavailable. Use `#[allow(lint)]` with a code comment instead:

```rust
#[allow(dead_code)] // public API surface — callers exist in downstream crates
pub fn exported_helper() { ... }
```

---

## Special Patterns

### RAII guards: `let _guard` vs `let _`

This is a critical Rust footgun when handling unused variable warnings:

```rust
// Correct — named binding keeps the guard alive to end of scope
let _guard = lock.lock().unwrap();

// WRONG — standalone _ drops the guard immediately, releasing the lock
let _ = lock.lock().unwrap();
```

When you see an `unused_variables` warning on a guard/lock/tempfile/handle, always prefix with `_name`, never replace with bare `_`. The bare `_` pattern drops the value at the end of the statement, not the scope.

Note: `let _ = expr;` is intentionally used to discard `#[must_use]` values — that's a different pattern and is fine when you genuinely want to ignore a Result.

### Conditional compilation

When variables are only used under certain `#[cfg]` flags:
```rust
#[expect(unused_variables, reason = "used only on Windows for path normalization")]
let normalized = path.to_str().unwrap();
#[cfg(target_os = "windows")]
do_windows_thing(normalized);
```

### Test modules

Test modules often accumulate `unused_imports` from glob imports:
```rust
#[cfg(test)]
mod tests {
    #[expect(unused_imports, reason = "glob import — not all parent items used in every test")]
    use super::*;
}
```

### Workspace lint configuration

If the project uses `[workspace.lints]` in Cargo.toml, respect that configuration. Do NOT add per-item attributes that contradict workspace lint policy. If a workspace-level lint is set to `deny`, address the issue rather than silencing it per-item.

---

## Reporting

After all fixes are applied and verified, provide a concise summary: total warnings before/after, each fix or suppression with its file location, lint name, and rationale. Flag any Tier 4 warnings that require the user's attention separately.

## Anti-Hallucination Rules

- **Only report warnings that actually appear in cargo output.** Do not invent warnings.
- **Read the file before editing.** Never guess at line numbers or code content.
- **If cargo clippy produces zero warnings, say so and stop.** Do not manufacture work.
- **Verify fixes compile.** After editing, run `cargo check` to confirm no regressions.
- **Check usages before removing "dead" code.** Grep for function/type/variable references first.
