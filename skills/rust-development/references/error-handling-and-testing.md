# Error Handling, Testing, and Project Organization

Grounded in *The Rust Programming Language* Chapters 7, 9, 11, 12, and 14, plus Rust 2024 Edition rules.

## 1. Error Handling Strategy (Ch 9)

- **`panic!` vs `Result<T, E>`**: Use `panic!` only for bugs or violated internal invariants; return `Result<T, E>` for any expected failure (I/O, parsing, network, invalid user input) and propagate with `?`.
- **Library vs. Application boundary**:
  - **Libraries**: Return structured, typed errors (`enum MyError` implementing `std::error::Error + Display`, or via `thiserror`). Mark public error enums `#[non_exhaustive]` so adding variants in minor releases is not a breaking change.
  - **Binaries / CLIs**: May use `Result<(), Box<dyn Error>>` (or `anyhow::Result`) at the top level (`main`) to attach context and exit cleanly.
- **Collecting Iterators of `Result` (Ch 13 / Ch 9)**: Prefer `.collect::<Result<Vec<_>, _>>()?` over manual `Vec::new()` + `for` loops:
  ```rust
  let parsed: Vec<i32> = inputs
      .iter()
      .map(|s| s.parse::<i32>())
      .collect::<Result<Vec<_>, _>>()?;
  ```
- **When `.expect()` is acceptable**: In `#[test]` functions, or when a compile-time invariant guarantees `Some`/`Ok`—always state the invariant in `.expect("...")`.

## 2. Automated Testing (Ch 11)

- **Unit tests**: `#[cfg(test)] mod tests { use super::*; ... }` in the same file to test private helpers without widening `pub` visibility.
- **Integration tests**: Top-level `tests/*.rs` exercising the crate strictly through its `pub` API.
- **Doc tests**: Runnable `///` examples verified by `cargo test --doc`.
- **Test hygiene**:
  - Return `Result<(), Box<dyn Error>>` from tests when convenient (`?` works inside tests).
  - Always include `expected = "..."` in `#[should_panic(expected = "...")]` so unrelated panics don't falsely pass.
  - Mark slow/networked tests with `#[ignore]`.
- **Rust 2024 Pitfall — Environment Variables in Tests**: `std::env::set_var` and `std::env::remove_var` are **`unsafe` in Rust 2024** because Cargo runs tests concurrently in multiple threads and mutating process env is a data race in POSIX.
  - **Do**: Refactor code under test to accept config structs, `impl Fn(&str) -> Option<String>`, or `HashMap` overrides instead of mutating process-wide env vars.

## 3. Binary & Library Separation (Ch 12)

Following the *minigrep* pattern from Chapter 12:
- Keep `src/main.rs` thin: parse CLI args / env vars, call `my_crate::run(config)`, print errors to `stderr` (`eprintln!`), and exit non-zero on `Err`.
- Put all domain logic, parsing, and testable functions in `src/lib.rs` (and submodules).

## 4. Cargo, Modules, and Workspaces (Ch 7, 14)

- **Visibility**: Default to private; use `pub(crate)` for internal cross-module helpers and `pub` only for intentional public API. Use `pub use` in `lib.rs` to re-export ergonomic paths.
- **Workspaces & Lints**: For multi-crate repos, use `[workspace]` with `resolver = "3"` (Edition 2024 default), `[workspace.dependencies]`, and `[workspace.lints]` inherited via `[lints] workspace = true`. See [Cargo-lints.toml](../resources/Cargo-lints.toml) for the full template.
- **Release profiles**: Keep default `dev` (`opt-level = 0`) and `release` (`opt-level = 3`); tune `lto` / `codegen-units` only when benchmarks justify it.
