# Fearless Concurrency and Asynchronous Programming

Grounded in *The Rust Programming Language* Chapters 16, 17, and 21, plus modern `std` concurrency primitives.

## 1. Threads, Message Passing, and Global State (Ch 16)

- **Threads**: Use `std::thread::spawn(move || ...)` for detached/owned work, or `std::thread::scope` when threads borrow local stack data and join before scope exit (avoids `Arc`).
- **Message passing (`std::sync::mpsc`)**: Prefer channels over shared mutable state. Dropping all `Sender`s closes the channel cleanly so `for msg in rx` terminates automatically.
- **Shared-state concurrency**: Wrap shared mutable state in `Arc<Mutex<T>>` (or `Arc<RwLock<T>>` for read-heavy workloads). Keep critical sections tiny, drop guards immediately, and acquire multiple locks in a consistent global order to prevent deadlocks.
- **Global / Lazy State (Modern `std`)**:
  - **Do**: Use `std::sync::LazyLock<T>` and `std::sync::OnceLock<T>` (or `LazyCell`/`OnceCell` for single-threaded contexts).
  - **Don't**: Introduce `lazy_static!`, `once_cell`, or `static mut` (taking references to `static mut` is forbidden in Rust 2024).
- **`Send` and `Sync`**: `Rc<T>` is `!Send`; `RefCell<T>`/`Cell<T>` are `!Sync`. Never write `unsafe impl Send/Sync` unless building a verified low-level synchronization primitive.

## 2. Asynchronous Programming (`async`/`await`) (Ch 17)

- **Futures are lazy**: Calling an `async fn` returns `impl Future<Output = T>` and does nothing until `.await`ed or polled.
- **Concurrency & Racing**: Use `join!` / `try_join!` for concurrent independent futures; use `select!` to race futures (e.g., timeout vs. I/O).
- **Streams**: Process asynchronous sequences (`Stream` / `StreamExt`) with `.next().await` or stream combinators.
- **Critical async pitfalls**:
  1. **Never hold `std::sync::MutexGuard` across `.await`**: Scope sync locks inside `{ let mut g = lock.lock().unwrap(); ... }` before `.await`, or use an async-aware mutex only when holding across `.await` is strictly required.
  2. **Never block the executor**: Offload heavy CPU work or blocking sync I/O (`std::fs`, sync DB drivers) to a blocking thread pool (`spawn_blocking` / `rayon`).
  3. **Cancellation safety**: A `Future` can be dropped at any `.await` point (e.g., losing a `select!` branch). Keep state transitions atomic across `.await` boundaries.

## 3. `Pin` and `Unpin` Diagnostics (Ch 17.5)

Self-referential `async` state machines and streams are `!Unpin`. When polling a future/stream by mutable reference (e.g., in a `loop`, `select!`, or `.next().await`) triggers `E0277: ... cannot be unpinned`:
- **Prefer stack pinning** (zero heap allocation):
  ```rust
  use std::pin::pin;

  let mut fut = pin!(my_async_op());
  // Poll `fut` safely without boxing
  ```
- **Reserve `Box::pin(fut)`** for returning `Pin<Box<dyn Future<...>>>` from functions, recursive `async fn`s, or storing heterogeneous futures inside structs/collections.

## 4. Async Functions in Traits (AFIT vs `async-trait`)

- **Do**: Use native `async fn` in traits (stable since Rust 1.75) for static dispatch (`<T: MyTrait>` / `impl MyTrait`).
- **When `Send` bounds or `dyn Trait` are needed**: Native `async fn` in traits does not enforce `+ Send` on the returned future by default. If your runtime requires `Send` (e.g., `tokio::spawn`) or you need `Box<dyn MyTrait>`, use `trait-variant` (to generate a `Send` companion trait) or `#[async_trait]`.

## 5. Choosing Threads vs. Async (Ch 17.6, Ch 21)

- **OS threads / thread pools** (Ch 21 web server pattern): CPU-bound work, simple synchronous I/O, minimal dependencies.
- **`async`/`await`**: High-concurrency I/O-bound workloads (thousands of sockets, HTTP/RPC servers, multiplexed streams).
