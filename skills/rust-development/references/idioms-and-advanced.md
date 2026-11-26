# Idioms, Rust 2024 Guardrails, API Guidelines, and Unsafe

Grounded in *The Rust Programming Language* Chapters 13, 18, 20, Appendices, Rust 2024 Edition, and the Rust API Guidelines.

## 1. Closures & Iterators (Ch 13)

- **Accept the most permissive closure bound**: `FnOnce` > `FnMut` > `Fn`.
- **Lazy iterator chains**: Prefer `.iter()` (borrows `&T`) over `.into_iter()` (moves `T`) when the collection is still needed. Chain `.filter()`, `.map()`, `.flat_map()` directly without intermediate `.collect::<Vec<_>>()`.

## 2. Trait Objects vs. Generics & Advanced Types (Ch 18, 20)

- **Static dispatch (`<T: Trait>` / `impl Trait`)**: Default choice—monomorphized, inlinable, zero vtable overhead.
- **Dynamic dispatch (`Box<dyn Trait>`, `&dyn Trait`)**: Use for heterogeneous collections (`Vec<Box<dyn Draw>>`) or to curb monomorphization bloat. Requires dyn-compatibility (no `Self` return, no generic methods).
- **Associated types vs. generics**: Associated type (`trait Iterator { type Item; }`) when one logical impl exists per type; generic parameter (`trait Add<Rhs = Self>`) when multiple impls per type make sense.
- **Newtype pattern**: Wrap external types (`struct UserId(u64);`) to enforce domain type safety and satisfy the orphan rule.

## 3. Rust 2024 Edition Guardrails & `unsafe` Checklist (Ch 20.1)

`unsafe` enables **only five** operations (the borrow checker still runs): dereferencing raw pointers, calling `unsafe fn`/FFI, accessing `static mut`, implementing `unsafe trait`, and reading `union` fields.

**Mandatory Rust 2024 & safety rules**:
- **`unsafe extern` blocks**: External FFI blocks must be declared `unsafe extern "C" { ... }`.
- **Unsafe attributes**: Use `#[unsafe(no_mangle)]`, `#[unsafe(export_name = "...")]`, and `#[unsafe(link_section = "...")]`.
- **`unsafe_op_in_unsafe_fn`**: Inside an `unsafe fn`, the body is treated as safe code by default. Isolate each unsafe operation in a minimal inner `unsafe { ... }` block with a `// SAFETY: ...` comment, and document caller preconditions under `# Safety` in rustdoc.
- **No `static mut` references**: Taking `&STATIC_MUT` or `&mut STATIC_MUT` is a hard error (`static_mut_refs`). Replace with `LazyLock<Mutex<T>>` or atomics (use `&raw const` / `&raw mut` strictly for low-level FFI).
- **Modern `std` literals**: Use `c"hello"` (`&'static CStr`) instead of `CString::new("hello").unwrap()`; iterate `Box<[T]>` directly by value (`into_iter()`).
- **Prefer safe `std` helpers**: Use `slice::split_at_mut`, `NonNull`, and `MaybeUninit` over manual pointer arithmetic.

## 4. Rust API Guidelines Checklist

- **Conversion & getter naming**:
  - `as_*(&self)` — free borrowed view (`as_bytes`, `as_str`).
  - `to_*(&self)` — potentially allocating/expensive conversion (`to_string`, `to_vec`).
  - `into_*(self)` — consumes `self`, transferring ownership (`into_inner`).
  - **No `get_` prefix on getters**: Write `fn len(&self)` and `fn name(&self)`, not `fn get_len(&self)`.
- **Conversion traits**: Implement `From<A> for B` / `TryFrom<A> for B` (never implement `Into` or `TryInto` directly—they are blanket-implemented). Accept `impl AsRef<Path>` for file paths; accept `impl Into<String>` only when ownership is unconditionally stored.
- **Derives & evolution**:
  - Derive `#[derive(Debug, Clone, PartialEq, Eq)]` (plus `Hash`, `Default`, `Copy` when semantically valid) on public types.
  - Add `#[must_use]` on pure functions, builders, and RAII guards whose return value must not be silently dropped.
  - Add `#[non_exhaustive]` on public structs/enums expected to gain fields/variants without breaking semver.
  - Use a private `mod private { pub trait Sealed {} }` supertrait when downstream crates must not implement a public trait.

## 5. Macros & Anti-Patterns to Avoid (`rust-skills`)

- **Prefer functions/generics over macros** unless variable arity, compile-time syntax, or trait boilerplate reduction (`macro_rules!`) is genuinely needed.
- **Clone-driven development**: Calling `.clone()` everywhere instead of fixing lifetimes or borrowing `&T`.
- **Stringly-typed APIs**: Passing `String` where an `enum` or newtype belongs.
- **Unbounded `Arc<Mutex<T>>` soup**: Wrapping entire structs in `Mutex` when message passing or fine-grained ownership works.
