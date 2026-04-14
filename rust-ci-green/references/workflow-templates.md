# GitHub Actions Workflow Templates for Rust CI

Reference templates for creating or auditing Rust CI workflows. Adapt to each project's specific flags.

## Standard Template (Single Crate)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always

jobs:
  fmt:
    name: Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      - run: cargo fmt --all --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo clippy --all-targets -- -D warnings

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test

  docs:
    name: Docs
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo doc --no-deps
        env:
          RUSTDOCFLAGS: "-D warnings"
```

## Workspace Template

Add `--workspace` to all cargo commands:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always

jobs:
  fmt:
    name: Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      - run: cargo fmt --all --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo clippy --workspace --all-targets -- -D warnings

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --workspace

  docs:
    name: Docs
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo doc --workspace --no-deps
        env:
          RUSTDOCFLAGS: "-D warnings"
```

## With All Features

Only use when the project supports enabling all features simultaneously (no mutually exclusive features):

```yaml
      - run: cargo clippy --workspace --all-targets --all-features -- -D warnings
      - run: cargo test --workspace --all-features
```

## With MSRV Testing

Add a separate job when the project declares `rust-version` in Cargo.toml:

```yaml
  msrv:
    name: MSRV Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@1.70.0  # Match rust-version in Cargo.toml
      - uses: Swatinem/rust-cache@v2
      - run: cargo check --all-targets
```

## With Pinned Toolchain

If the project has `rust-toolchain.toml`, `dtolnay/rust-toolchain` respects it automatically:

```yaml
      - uses: dtolnay/rust-toolchain@stable
        # Reads rust-toolchain.toml if present -- no explicit toolchain needed
        with:
          components: clippy, rustfmt
```

## Design Principles

### Separate jobs for clarity
Split fmt, clippy, test, and docs into separate jobs:
- **Parallel execution** -- faster CI
- **Clear failure signals** -- know exactly what broke
- **Cheaper reruns** -- only failed jobs rerun

### No caching for fmt
The `fmt` job doesn't need `Swatinem/rust-cache` -- `cargo fmt --check` doesn't compile anything.

### Cache after toolchain
`Swatinem/rust-cache` **must** come after `dtolnay/rust-toolchain` because it uses the rustc version as part of its cache key. If the cache step runs first, it can't determine the correct key.

### Cache invalidation warning
Never use `RUSTFLAGS: "-Dwarnings"` as a job-level env var when also running clippy. It changes the cargo cache key, causing full rebuilds of dependencies. Use `-- -D warnings` after the clippy command.

Exception: `RUSTDOCFLAGS: "-D warnings"` is required for `cargo doc` -- it doesn't accept `-- -D warnings`. Set it at the step level, not job level, to avoid affecting other steps.

### Badge syntax
```markdown
[![CI](https://github.com/OWNER/REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/OWNER/REPO/actions/workflows/ci.yml)
```

Derive OWNER/REPO from `git remote get-url origin`.
