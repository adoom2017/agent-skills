# Ownership, Types, Generics, and Smart Pointers

Grounded in *The Rust Programming Language* Chapters 3–6, 8, 10, 15, and 19, plus Rust 2024 Edition rules.

## 1. Borrowing & Signature Hygiene (Ch 4, 8)

- **Do**: Accept `&str`, `&[T]`, and `Option<&T>` (`.as_ref()`) in function parameters.
  **Don't**: Accept `&String`, `&Vec<T>`, or `&Option<T>` (forces caller allocation or awkward derefs).
- **Do**: Pre-allocate with `Vec::with_capacity(n)` when size is known; use `HashMap::entry(k).or_insert_with(...)` to avoid double lookups; use `BTreeMap` when deterministic key order or range queries are needed.
- **Do**: Iterate UTF-8 explicitly with `.chars()` or `.bytes()`.
  **Don't**: Slice strings at arbitrary byte offsets without `.is_char_boundary(idx)` check.

## 2. Structs, Enums, and Pattern Matching (Ch 5, 6, 19)

- **Make invalid states unrepresentable**: Model mutually exclusive states as `enum` variants carrying their own data rather than structs full of `Option<T>` flags.
- **Exhaustive `match` vs `let ... else`**: Prefer `match` when all variants matter so new variants cause compile errors; use `let Some(v) = opt else { return Err(...); };` for clean early returns.
- **Method receivers**: `&self` for inspection, `&mut self` for in-place mutation, `self` only when consuming/transforming (e.g., `.build()`, `.into_inner()`).

## 3. Numeric Safety & Safe Indexing

- **No blind `as` narrowing/sign casts**: `u64 as u32` or `i32 as usize` silently truncates or wraps.
  - **Do**: `let idx = usize::try_from(raw_id)?;`
  - **Do**: Use `checked_add` / `saturating_sub` / `wrapping_add` explicitly when overflow semantics matter.
- **No unchecked indexing on untrusted indices**:
  - **Do**: `let item = slice.get(idx).ok_or(Error::OutOfBounds)?;`
  - **Don't**: `slice[idx]` when `idx` comes from external input or arithmetic.

## 4. Generics, Traits, and Rust 2024 RPIT Lifetimes (Ch 10)

- **Zero-cost generics**: Prefer static dispatch (`impl Trait` or `<T: Trait>` with `where` clauses) by default.
- **Lifetime elision**: Elide lifetimes whenever possible; annotate `'a` only when linking an output reference to specific input references or storing references in structs (`struct Parser<'a>`). Avoid `'static` bounds unless data truly lives for the entire program or crosses `thread::spawn` / `tokio::spawn`.
- **Rust 2024 RPIT Lifetime Capture (`use<...>`)**: In Edition 2024, `-> impl Trait` captures **all** in-scope generic type and lifetime parameters by default. If a returned iterator/future does not actually borrow an input reference, exclude that lifetime with `+ use<...>` so callers aren't unnecessarily borrowed:
  ```rust
  // Captures T only, not '_ from `label`
  fn make_iter<T: Clone>(items: Vec<T>, _label: &str) -> impl Iterator<Item = T> + use<T> {
      items.into_iter()
  }
  ```

## 5. Smart Pointers & Interior Mutability (Ch 15)

Choose the weakest pointer that satisfies ownership requirements:

| Type | Ownership | Thread-Safe | Mutability | When to Use |
|---|---|---|---|---|
| `Box<T>` | Single | Yes (if `T: Send/Sync`) | Inherited | Recursive types, large stack values, `Box<dyn Trait>` |
| `Rc<T>` | Shared | **No** | Immutable | Single-threaded DAGs / shared read-only state |
| `Arc<T>` | Shared | **Yes** | Immutable | Multi-threaded / async shared read-only state |
| `RefCell<T>` | Single | **No** | Runtime-checked (`borrow_mut`) | Single-threaded interior mutability (e.g., mock objects) |
| `Cow<'a, T>` | Borrowed or Owned | Inherited | Clone-on-write | Avoids allocation when mutation is only occasional |

- **Prevent reference cycles**: Use `Weak<T>` (`Rc::downgrade` / `Arc::downgrade`) for parent or back-references in cyclic graphs/trees.
- **Smart Pointer Anti-Pattern (Ch 15)**: **Never** implement `Deref` or `DerefMut` on domain structs solely to simulate OOP inheritance or forward methods. Reserve `Deref` strictly for smart pointers and transparent newtype views.
