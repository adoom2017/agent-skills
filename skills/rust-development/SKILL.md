---
name: rust-development
description: >-
  Write, review, refactor, and debug idiomatic, safe, and performant Rust code
  grounded in The Rust Programming Language (stable book) and modern Rust
  engineering practices. Use when creating Cargo projects, designing traits/types,
  resolving ownership/lifetime/borrow-checker errors, implementing error handling,
  writing concurrent or async Rust, reviewing unsafe blocks, or running Cargo
  verification gates.
metadata:
  source: https://doc.rust-lang.org/stable/book/title-page.html
  edition: "2024"
  tags:
    - rust
    - cargo
    - ownership
    - lifetimes
    - async
    - concurrency
    - error-handling
---

# Rust Development Skill

Synthesized from *The Rust Programming Language* (stable book, Chapters 1–21 + Appendices), Rust 2024 Edition rules, the Rust API Guidelines, and `rust-skills` engineering practices.

## Agent Behavior Contract

When this skill is active, follow these rules:

1. **Borrow before owning, slice before allocating**: Accept `&str` over `&String`, `&[T]` over `&Vec<T>`, and `&T` over `T` when ownership transfer is unnecessary (Book Ch 4). Avoid `.clone()` solely to silence the borrow checker.
2. **Never use `.unwrap()` or `.expect()` in library or production paths** without a documented invariant: Return `Result<T, E>` or `Option<T>`, propagate with `?`, and reserve `panic!` for unrecoverable contract violations or tests (Book Ch 9).
3. **Prefer compile-time guarantees over runtime checks**: Use enums, newtypes, `Option`, and trait bounds (`T: Trait`) instead of sentinel values, stringly-typed flags, or `RefCell` when static borrowing works (Book Ch 5, 6, 10, 15).
4. **Enforce numeric & indexing safety**: Avoid lossy `as` casts (`u64 as u32`, `i32 as usize`); prefer `TryFrom` / `checked_*` / `saturating_*`. On untrusted indices, use `slice.get(i)` instead of `slice[i]`.
5. **Favor iterators and combinators over manual index loops** when transforming collections — they are zero-cost and eliminate bounds-check bugs (Book Ch 13).
6. **Follow Rust 2024 Edition & `unsafe` rules**: Use `unsafe extern "C"`, `#[unsafe(no_mangle)]`, explicit inner `unsafe { ... }` blocks inside `unsafe fn` with `// SAFETY: ...` comments, and `LazyLock`/`OnceLock` instead of `static mut`. Note that `std::env::set_var` / `remove_var` are `unsafe` in 2024 (Book Ch 20).
7. **Prefer modern `std` over legacy crates**: Use `std::sync::LazyLock` / `OnceLock` instead of `lazy_static!` / `once_cell`, native `async fn` in traits (1.75+) before reaching for `#[async_trait]`, and `c"..."` literals for C strings.
8. **Respect async vs. thread boundaries**: Never hold a `std::sync::MutexGuard` across `.await` or run blocking I/O / CPU-heavy loops on an async executor thread (Book Ch 16, 17).
9. **Always verify with Cargo gates** after edits: run `cargo fmt`, `cargo clippy`, and `cargo test` before concluding.

---

## Diagnostic & Error-Routing Table

| Symptom / Compiler Error | First Check | Smallest Fix | Deep Dive |
|---|---|---|---|
| `E0382` (use of moved value) | Did ownership move unnecessarily? | Borrow (`&T` / `&mut T`) or return ownership | [ownership-and-types.md](./references/ownership-and-types.md) |
| `E0502` / `E0499` (borrow conflict) | Overlapping `&mut` and `&` scopes | Narrow borrow scope, split struct fields, or use `RefCell` | [ownership-and-types.md](./references/ownership-and-types.md) |
| `E0106` / lifetime mismatch | Output reference tied to multiple inputs | Elide if single input, or bind `'a` explicitly | [ownership-and-types.md](./references/ownership-and-types.md) |
| RPIT lifetime over-capture (`-> impl Trait`) | Edition 2024 captures all in-scope params | Add precise capture bound `+ use<T>` | [ownership-and-types.md](./references/ownership-and-types.md) |
| `E0277` (`Send`/`Sync` not met) | Non-thread-safe type across thread/`.await` | Replace `Rc`/`RefCell` with `Arc`/`Mutex`; drop guard before `.await` | [concurrency-and-async.md](./references/concurrency-and-async.md) |
| `E0277` (`Unpin` not met on `Future`/`Stream`) | Polling unpinned `async` state machine | Stack-pin with `std::pin::pin!(fut)` (or `Box::pin` if heap needed) | [concurrency-and-async.md](./references/concurrency-and-async.md) |
| `static_mut_refs` / 2024 `unsafe` errors | `static mut`, `extern`, `#[no_mangle]`, `env::set_var` | Use `LazyLock`/`Mutex`, `unsafe extern`, `#[unsafe(no_mangle)]` | [idioms-and-advanced.md](./references/idioms-and-advanced.md) |
| Excessive `.clone()` or `String` allocs | Function signature takes owned types | Switch parameters to `&str`, `&[T]`, `impl AsRef<Path>`, or `Cow<'_, str>` | [idioms-and-advanced.md](./references/idioms-and-advanced.md) |
| Messy error types / `.unwrap()` chains | Mixing `panic!` and recoverable errors | Define domain error enum (`Result<T, E>`), propagate with `?` | [error-handling-and-testing.md](./references/error-handling-and-testing.md) |
| Slow or bloated trait dispatch | Unnecessary `Box<dyn Trait>` in hot path | Use static dispatch (`impl Trait` or `<T: Trait>`) | [idioms-and-advanced.md](./references/idioms-and-advanced.md) |
| Async task hangs or blocks executor | Blocking call or lock held across `.await` | Scope sync lock before `.await`; offload CPU/blocking work | [concurrency-and-async.md](./references/concurrency-and-async.md) |

---

## Code Review Workflow

When reviewing or refactoring Rust code, audit in this order:

1. **Unsafety & Edition 2024 Compliance**: Check every `unsafe` block for `// SAFETY:` invariants, `unsafe extern` / `#[unsafe(...)]` syntax, and absence of `static mut` refs.
2. **Ownership & Zero-Copy Signatures**: Replace `&String`/`&Vec<T>` with `&str`/`&[T]`, eliminate redundant `.clone()`, and verify `use<...>` on RPITs if lifetimes are over-captured.
3. **Error Handling & Bounds Safety**: Remove production `.unwrap()`/`.expect()`, replace lossy `as` numeric casts with `TryFrom`/`checked_*`, and guard untrusted slice indexing with `.get()`.
4. **Concurrency & Async Hygiene**: Ensure no `std::sync::MutexGuard` crosses `.await`, blocking work is offloaded, and `!Unpin` futures/streams use `std::pin::pin!`.
5. **API Guidelines & Cargo Gates**: Check naming (`as_`/`to_`/`into_`, no `get_` prefix), `#[must_use]` / `#[non_exhaustive]`, and run Clippy + tests.

---

## Book-to-Practice Core Map (Chapters 1–21)

| Book Chapters | Core Concept | Key Engineering Rule | Reference |
|---|---|---|---|
| Ch 1–3, 7, 14 | Cargo, Types, Modules, Workspaces | Keep modules small; expose minimal `pub` API; use `resolver = "3"` workspaces | [error-handling-and-testing.md](./references/error-handling-and-testing.md) |
| Ch 4–6, 8, 10, 15, 19 | Ownership, Enums, Generics, Smart Pointers, Patterns | Model states with enums; exhaustively `match`; avoid `Deref` OOP abuse | [ownership-and-types.md](./references/ownership-and-types.md) |
| Ch 9, 11, 12 | Error Handling, Testing, CLI I/O | Separate `lib.rs` logic from `main.rs` I/O; test without mutating global env | [error-handling-and-testing.md](./references/error-handling-and-testing.md) |
| Ch 13, 18, 20 | Closures, Iterators, Trait Objects, Advanced & Unsafe | Prefer `impl Iterator` / generic bounds; follow API Guidelines & 2024 `unsafe` rules | [idioms-and-advanced.md](./references/idioms-and-advanced.md) |
| Ch 16, 17, 21 | Threads, Channels, `Send`/`Sync`, `async`/`await`, `Pin`, Streams | Prefer message passing (`mpsc`) or scoped shared state (`Arc<Mutex<T>>`); use `async` for I/O concurrency | [concurrency-and-async.md](./references/concurrency-and-async.md) |

---

## Cargo Verification & Setup Gate

For new or refactored crates, apply the lint configuration from [Cargo-lints.toml](./resources/Cargo-lints.toml), then run:

```bash
cargo fmt --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test
```

For documentation and workspace checks when publishing or refactoring public APIs:

```bash
cargo doc --no-deps
cargo test --doc
```
