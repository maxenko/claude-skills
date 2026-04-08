# Rust Test Module Reference

Supplementary reference for edge cases and visibility rules. The main workflow is in SKILL.md — this file covers details needed only when handling unusual situations.

## Empirical Data: How Major Projects Organize Tests

| Project | Inline `mod tests` | Separate `mod tests;` | Integration `tests/` |
|---------|--------------------|-----------------------|---------------------|
| tokio | 14 files | 7 files | 164 files |
| axum | 28 files | 1 file | 1 file |
| bevy | 233 files | 2 files | ~83 files |
| ripgrep | 36 files | 0 files | 15 files |
| serde | 0 (separate crate) | 0 | 26 (test_suite crate) |

Inline is the overwhelming default (~30:1). Extraction happens only when test blocks are large (tokio extracts at ~900+ impl lines / large test suites).

## Visibility Rules After Extraction

| Item visibility | From inline `mod tests`? | From `foo/tests.rs`? | From `tests/` (integration)? |
|----------------|--------------------------|---------------------|------------------------------|
| `fn private()` | Yes | Yes (still child module) | No |
| `pub(crate) fn` | Yes | Yes | No |
| `pub fn` | Yes | Yes | Yes |
| `pub(super) fn` | Yes | Yes* | No |

*`pub(super)` items in `foo` are accessible from `foo::tests` because child modules can access all parent items regardless of declared visibility. The `pub(super)` annotation is irrelevant from the child's perspective.

## `#[cfg(test)]` Scoping Rules

- `#[cfg(test)]` on `mod tests;` — entire file only compiled during `cargo test`
- `#[cfg(test)]` on `use` statements — import only during testing
- `#[cfg(test)]` on functions outside `mod tests` — compiled only during test, but unusual pattern
- `cfg(test)` is **per-crate**: a `#[cfg(test)]` item in crate A is NOT available when testing crate B, even in the same workspace

## Multi-File Test Module Pattern

For very large test suites (test file itself exceeds ~500 lines), tokio and axum use a directory-based test module:

```
src/routing/
  mod.rs                # #[cfg(test)] mod tests;
  tests/
    mod.rs              # shared setup + some tests
    fallback.rs         # use super::*;
    merge.rs
    nest.rs
```

This is rare. Only use it when a single `tests.rs` becomes unwieldy.

## Test Framework Macros

Test modules may use macro-heavy frameworks that generate items. Preserve these exactly during extraction:

- **`rstest`**: `#[rstest]`, `#[fixture]`, and `#[case]` attributes generate test functions and fixture constructors. Move them with the test module as-is — they resolve at compile time and don't depend on module position.
- **`proptest`**: `proptest! { ... }` macro blocks generate `#[test]` functions. Treat the entire `proptest!` invocation as a single item — don't split it across files.
- **`test-case`**: `#[test_case(...)]` attributes on functions. Move with the function they annotate.
- **`tokio::test` / `async-std::test`**: `#[tokio::test]` and similar async test macros. These are drop-in `#[test]` replacements and need no special handling.

If a test module has `use rstest::*;` or similar framework imports, these move with the test module contents (they were already scoped to the test module).

## Additional Anti-Patterns

These supplement the critical principles in SKILL.md:

- **Using `#[path]` attribute for test modules** — works but non-standard. No major project (tokio, serde, bevy, axum, ripgrep) uses it for tests.
- **Creating `tests/common.rs`** — Cargo treats it as a test crate and shows `running 0 tests`. Use `tests/common/mod.rs` instead.
- **`pub(crate)` does not help integration tests** — `tests/` files are separate crates. Only fully `pub` items are visible to them.
