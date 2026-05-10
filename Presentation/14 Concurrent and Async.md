# Module 14: Concurrent and Async Programming

## Module Overview

Concurrency is where Rust's design pays its biggest dividends. The same ownership and borrowing rules that prevent memory bugs in single-threaded code also prevent data races in concurrent code. The phrase "if it compiles, it usually works" is most strongly true here: the entire class of concurrency bugs that plagues C, C++, and Java programs is rejected at compile time in safe Rust.

This module covers two distinct but related topics: traditional thread-based concurrency and async programming. They serve different purposes and are sometimes used together. Both build on the same ownership foundations.

Three ideas are central:

1. **Threads share ownership through specific types.** A value cannot just "be shared" between threads; you must opt in by using `Arc` (shared ownership) and synchronization primitives like `Mutex`. The compiler ensures these are used correctly.
2. **The compiler categorizes types by thread safety.** The traits `Send` (safe to transfer between threads) and `Sync` (safe to share by reference between threads) are checked automatically. If a type is not `Send`, the compiler will refuse to send it.
3. **Async is a different model with similar guarantees.** Async functions return `Future`s that a runtime polls to completion. Async code in Rust uses the same ownership rules, plus some specific patterns for the async context.

By the end of this module, students will:

- Spawn threads and pass data between them safely.
- Use channels for message-passing concurrency.
- Use `Mutex`, `RwLock`, and `Arc` for shared-state concurrency.
- Understand `Send` and `Sync` and why some types lack them.
- Write async functions and call them with `await`.
- Use the Tokio runtime to execute async code.
- Spawn async tasks and use `select!` to wait on multiple futures.
- Make HTTP requests with the `reqwest` crate as a representative async API.

This module is long because there are two parallel topics. Students who only need one or the other can skim the section that does not apply. In practice, most modern Rust code uses async for I/O-heavy work and threads for CPU-heavy work; both are worth knowing.

> **A note on third-party libraries.** Async Rust depends on external runtimes. The standard library provides the `Future` trait but no executor. The `tokio` crate is the dominant runtime in the ecosystem and is what most production Rust uses. The `reqwest` crate (Section K) is a high-level HTTP client built on top of `tokio`. Sections F through K assume both are available.

---

## A. Rust's Fearless Concurrency Model

### The Problem

Concurrent programs are difficult because shared mutable state can be modified from multiple threads simultaneously. The classic problems:

- **Data races**: two threads access the same data, at least one writes, and there is no synchronization. The result is undefined behavior.
- **Deadlocks**: threads wait for resources that other threads hold, and nobody can proceed.
- **Race conditions**: the program's correctness depends on the timing of operations, leading to bugs that appear only in production.

Most languages handle these problems with discipline and tools: documentation conventions, code review, runtime detectors, careful use of synchronization primitives. In practice, large concurrent codebases in C++, Java, and Go ship with these bugs constantly.

### Rust's Approach

Rust's ownership system extends to concurrent code. The borrow rules from Module 6 (any number of immutable references OR exactly one mutable reference) prevent data races by construction. The compiler tracks which thread owns which data and refuses to compile code that would allow simultaneous access.

The result is what the Rust community calls "fearless concurrency": you can write threaded code with confidence that data races are impossible. The compile-time guarantees mean entire categories of bugs simply cannot exist in safe Rust.

The remaining concurrent bugs (deadlocks, logical race conditions) still require care. The compiler does not solve all concurrency problems, but it does solve the ones most responsible for production crashes.

### Two Concurrency Models

This module covers two distinct concurrent programming models:

**Threads** (Sections B-E): traditional OS threads. Good for CPU-bound work where you want to use multiple cores in parallel.

**Async/await** (Sections F-K): cooperative multitasking. Good for I/O-bound work where many tasks wait on network, disk, or other slow operations.

The two are not competitors. Most real applications use both. Rust supports both with the same ownership rules and the same level of safety.

---

## B. Threads: `spawn`, `join`, and `move` Closures

The standard library provides traditional threads through `std::thread`.

### Spawning a Thread

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        println!("hello from a new thread");
    });

    println!("hello from the main thread");

    handle.join().unwrap();
}
```

`thread::spawn` takes a closure and runs it on a new thread. It returns a `JoinHandle` that the parent can use to wait for the thread to finish.

`handle.join()` blocks the current thread until the spawned thread completes. It returns `Result<T, ...>`: `Ok(value)` if the thread finished normally with that return value, or `Err(...)` if the thread panicked.

### Why `move` Is Usually Needed

Threads can outlive the function that spawned them. If the closure borrows from local variables, the references could become dangling when the parent returns. Rust prevents this:

```rust
fn main() {
    let data = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("{data:?}");        // ERROR: borrows data
    });

    handle.join().unwrap();
}
```

The compiler rejects this. The spawned thread's closure tries to borrow `data`, but the compiler cannot prove that `data` will live long enough.

The fix is `move`:

```rust
fn main() {
    let data = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("{data:?}");
    });

    handle.join().unwrap();
}
```

The `move` keyword forces the closure to take ownership of `data`. Now the closure owns the vector, and the new thread carries it. The parent thread no longer has access to `data`, but the spawned thread has it for as long as it runs.

This is the same `move` pattern from Module 5, applied to threads. Almost every thread spawn in Rust uses `move` because the values are typically owned by the parent and need to transfer.

### Returning Values from a Thread

A thread's closure can return a value, which `join` retrieves:

```rust
fn main() {
    let handle = thread::spawn(|| {
        let total: i32 = (1..=100).sum();
        total
    });

    let result = handle.join().unwrap();
    println!("sum: {result}");
}
```

The closure returns `i32`, so `join` returns `Result<i32, ...>`. The `.unwrap()` is safe here because the closure does not panic.

### CPU Parallelism with Threads

Spawning multiple threads is the standard way to parallelize CPU-bound work:

```rust
fn main() {
    let chunks = vec![
        vec![1, 2, 3, 4, 5],
        vec![6, 7, 8, 9, 10],
        vec![11, 12, 13, 14, 15],
    ];

    let handles: Vec<_> = chunks
        .into_iter()
        .map(|chunk| {
            thread::spawn(move || {
                chunk.iter().sum::<i32>()
            })
        })
        .collect();

    let total: i32 = handles
        .into_iter()
        .map(|h| h.join().unwrap())
        .sum();

    println!("total: {total}");          // 120
}
```

Each chunk is processed on its own thread. The main thread waits for all of them and sums the partial results. On a multi-core machine, this runs in parallel.

For more sophisticated parallel patterns, the `rayon` crate provides parallel iterators that handle the threading automatically. For everyday work-distribution, manual `spawn`/`join` is sufficient.

---

## C. Message Passing with Channels (mpsc)

One way to coordinate between threads is to send messages through channels. Rust's standard library provides multi-producer, single-consumer (mpsc) channels.

### Creating a Channel

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send("hello from thread").unwrap();
    });

    let msg = rx.recv().unwrap();
    println!("{msg}");
}
```

`mpsc::channel()` returns a tuple: a transmitter `tx` and a receiver `rx`. The transmitter can be cloned to allow multiple senders; the receiver cannot be cloned because the channel is single-consumer.

The spawned thread sends a message via `tx.send(value)`. The main thread receives via `rx.recv()`, which blocks until a message arrives or the channel is closed.

### Multiple Producers

The transmitter can be cloned, allowing multiple threads to send to the same receiver:

```rust
fn main() {
    let (tx, rx) = mpsc::channel();

    for i in 0..3 {
        let tx = tx.clone();
        thread::spawn(move || {
            tx.send(format!("from thread {i}")).unwrap();
        });
    }

    drop(tx);                            // close the original sender

    for received in rx {
        println!("{received}");
    }
}
```

Each thread sends one message. The original `tx` is dropped after the threads are spawned, so when all the cloned senders go out of scope, the channel closes. The receiver implements `Iterator`, so `for received in rx` walks all messages until the channel closes.

### Sending Owned Data

Messages sent through a channel must be `Send` (covered in Section E). For owned data, the value is moved into the channel:

```rust
let (tx, rx) = mpsc::channel();

let data = vec![1, 2, 3];
thread::spawn(move || {
    tx.send(data).unwrap();              // data is moved into the channel
});

let received = rx.recv().unwrap();
println!("{received:?}");
```

After `send`, the sending thread no longer has access to `data`. The receiving thread now owns it. This ownership transfer through the channel is what makes message-passing concurrency safe in Rust.

### When to Use Channels

Channels are appropriate when:

- Threads have well-defined producer/consumer roles.
- Data flows in one direction.
- You want to avoid shared mutable state.

The Go language's mantra "do not communicate by sharing memory; share memory by communicating" applies in Rust. Channel-based concurrency is often clearer than lock-based concurrency.

For more complex patterns (multiple consumers, broadcast, request/response), other channel implementations exist. The `crossbeam` and `flume` crates provide more flexible options.

---

## D. Shared State: `Mutex<T>`, `RwLock<T>`, and `Arc<T>`

When multiple threads need to read or modify the same data, you need shared state. Rust provides safe shared state through `Arc` (for ownership) and `Mutex` or `RwLock` (for synchronization).

### `Arc<T>`: Shared Ownership

`Arc<T>` is a thread-safe reference-counted pointer. It allows multiple threads to share ownership of the same data:

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3, 4, 5]);

    let mut handles = vec![];
    for i in 0..3 {
        let data = Arc::clone(&data);
        let handle = thread::spawn(move || {
            println!("thread {i} sees: {data:?}");
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

`Arc::new` wraps a value in an `Arc`. `Arc::clone` increments the reference count and produces a new `Arc` that points to the same data. When all `Arc`s are dropped, the underlying data is deallocated.

The "A" in `Arc` stands for "atomic": the reference count is updated atomically, making it safe to use across threads. There is also `Rc<T>` (without the atomic operations) for single-threaded use, but `Arc` is what you need for multi-threaded code.

### `Mutex<T>`: Exclusive Access

`Arc<T>` alone gives shared read access. To allow modification, wrap the data in a `Mutex`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let mut handles = vec![];
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut count = counter.lock().unwrap();
            *count += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("counter: {}", *counter.lock().unwrap());     // 10
}
```

The pattern is `Arc<Mutex<T>>`: shared ownership of mutex-protected data. To modify the inner value:

1. Call `lock()` to acquire the mutex. This returns a `LockGuard`.
2. Dereference the guard to access the data: `*count += 1`.
3. The guard is automatically released when it goes out of scope.

The `lock()` method returns `Result<MutexGuard, ...>`. It only fails if another thread panicked while holding the lock (a "poisoned" mutex). For most code, `.unwrap()` is acceptable.

### Why `Arc<Mutex<T>>` Together?

`Arc` provides shared ownership: multiple threads can have a handle to the same data. `Mutex` provides synchronized access: only one thread at a time can modify the data.

Without `Arc`, you cannot have multiple owners. Without `Mutex`, the borrow checker forbids simultaneous mutation. Together, they let multiple threads safely modify shared data, with the synchronization enforced by the lock.

This combination is so common that you will see `Arc<Mutex<T>>` constantly in concurrent Rust code.

### `RwLock<T>`: Multiple Readers or One Writer

`Mutex` allows only one thread at a time, even for reads. When most accesses are reads and only a few are writes, `RwLock` is more efficient:

```rust
use std::sync::{Arc, RwLock};

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));

    // Multiple readers can hold the lock simultaneously:
    let readers: Vec<_> = (0..3).map(|i| {
        let data = Arc::clone(&data);
        thread::spawn(move || {
            let read = data.read().unwrap();
            println!("reader {i}: {:?}", *read);
        })
    }).collect();

    for r in readers {
        r.join().unwrap();
    }

    // Only one writer at a time, and no readers while writing:
    {
        let mut write = data.write().unwrap();
        write.push(4);
    }

    println!("after write: {:?}", *data.read().unwrap());
}
```

The trade-off: `RwLock` is slightly more expensive per operation than `Mutex` because it tracks reader count. Use it when reads vastly outnumber writes; otherwise, `Mutex` is fine.

### Avoiding Deadlocks

The compiler prevents data races but does not prevent deadlocks. A deadlock typically arises when:

- Thread A holds lock X and tries to acquire lock Y.
- Thread B holds lock Y and tries to acquire lock X.
- Both threads block forever.

The standard advice for avoiding deadlocks:

- Acquire locks in a consistent order across the codebase.
- Hold locks for as short a time as possible.
- Avoid calling unknown functions while holding a lock.
- Prefer message passing over shared state when feasible.

These are programming discipline issues, not language features. The borrow checker cannot help here.

---

## E. The `Send` and `Sync` Traits

Two traits in the standard library categorize types by their thread-safety properties.

### `Send`: Safe to Transfer Between Threads

A type is `Send` if it is safe to move ownership of a value of that type to another thread. Most types are `Send`:

- Primitive types (integers, floats, bools, chars).
- Owned values like `String`, `Vec<T>` (if `T: Send`).
- `Arc<T>` (if `T: Send + Sync`).

A few types are not `Send`:

- `Rc<T>` (the non-atomic reference-counted pointer). Its reference count is not atomic, so concurrent updates would corrupt it.
- Raw pointers in some configurations.
- Types that wrap thread-local resources.

When you try to send a non-`Send` type to a thread, the compiler rejects it:

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);

    thread::spawn(move || {
        println!("{data:?}");        // ERROR: Rc is not Send
    });
}
```

The fix is to use `Arc` instead of `Rc`. `Arc` has the same purpose but is thread-safe.

### `Sync`: Safe to Share by Reference Across Threads

A type `T` is `Sync` if `&T` is `Send`. In other words, you can share references to a `Sync` value across threads.

Most types are `Sync` if their inner state is also thread-safe:

- Primitive types.
- `Arc<T>` (if `T: Sync`).
- `Mutex<T>` (if `T: Send`).
- Read-only collections of `Sync` types.

Some types are not `Sync`:

- `Cell<T>` and `RefCell<T>` (interior mutability without synchronization).
- `Rc<T>` (not thread-safe).

When you try to share a non-`Sync` type across threads, the compiler rejects it.

### How These Are Used

You usually do not implement `Send` or `Sync` manually. They are automatically implemented for any type whose components are all `Send` (or `Sync`). This is called "automatic" or "auto-derive" trait implementation.

You see them in trait bounds on concurrent APIs:

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
{
    // ...
}
```

The bounds say: the closure and its return value must both be `Send` (transferable between threads) and `'static` (no borrowed references). The compiler enforces these bounds, refusing to compile code that would send non-thread-safe data to another thread.

### When You See `Send`/`Sync` Errors

A common error message looks like:

```
error[E0277]: `Rc<...>` cannot be sent between threads safely
```

The fix is to replace the non-`Send` type with its thread-safe equivalent: `Rc` to `Arc`, `RefCell` to `Mutex` or `RwLock`, etc.

For most code, you will not think about `Send` and `Sync` directly. They work in the background, ensuring that types you create are correctly classified. When the compiler does flag a problem, the fix is usually to change the type rather than to think hard about why.

---

## F. Introduction to Async/Await

Threads are good for CPU-bound work. For I/O-bound work (network requests, database queries, file operations), running thousands of threads is wasteful: most threads spend most of their time waiting. Async programming addresses this by letting many tasks share a small number of threads.

### The Basic Idea

In synchronous code, a function that performs I/O blocks the thread until the I/O completes:

```rust
fn fetch(url: &str) -> String {
    // blocks the thread until the response arrives
    http_get(url)
}
```

If you have 1000 URLs to fetch, you need 1000 threads or you process them sequentially.

In async code, a function returns a `Future` that represents work to be done eventually:

```rust
async fn fetch(url: &str) -> String {
    http_get(url).await        // suspends, does not block
}
```

When the function reaches `.await`, it can suspend itself, letting other tasks run on the same thread. When the I/O completes, the runtime resumes the function. With this model, 1000 tasks can share 4 threads.

### What `async` Does

The `async` keyword turns a function (or block) into one that returns a `Future`:

```rust
async fn greet(name: &str) -> String {
    format!("hello, {name}")
}

fn main() {
    let future = greet("alice");
    // future is a Future; nothing has run yet.
}
```

Calling an async function does not run it. It returns a `Future` that, when polled by a runtime, eventually produces the result.

To actually run the future, you need a runtime (covered in Section G) and you call `.await` on it inside another async function:

```rust
async fn main_async() {
    let message = greet("alice").await;
    println!("{message}");
}
```

The `.await` suspends `main_async` until `greet` completes, then resumes with the result. From the writer's perspective, it reads almost like synchronous code; the runtime handles the suspension and resumption.

### Why Two Models?

Threads and async serve different needs:

| Aspect              | Threads                   | Async                            |
|---------------------|---------------------------|----------------------------------|
| Best for            | CPU-bound work            | I/O-bound work                   |
| Number of tasks     | Tens or hundreds          | Thousands or more                |
| Resource per task   | OS thread (~MB stack)     | Lightweight task (~KB)           |
| Scheduling          | Preemptive (OS)           | Cooperative (runtime)            |
| Blocking call cost  | Blocks the thread         | Blocks the runtime (problem)     |
| Library ecosystem   | Standard library          | Third-party (tokio, async-std)   |

Most modern Rust code uses async for I/O-heavy work (web servers, network clients) and threads for CPU-heavy work (data processing, computation). The two are complementary, not competing.

---

## G. The Tokio Runtime

The Rust standard library provides the `Future` trait and the `async`/`await` syntax but does not provide a runtime. To execute futures, you need an async runtime. The dominant choice in production Rust is `tokio`.

### Adding Tokio

In `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

The `features = ["full"]` enables all of Tokio's components. For production code, you typically enable only what you need (e.g., `["rt", "macros", "net"]`), but `"full"` is fine for learning.

### A Tokio Main

The simplest way to run async code is the `#[tokio::main]` attribute:

```rust
#[tokio::main]
async fn main() {
    println!("hello from async main");
    let result = greet("alice").await;
    println!("{result}");
}

async fn greet(name: &str) -> String {
    format!("hello, {name}")
}
```

The `#[tokio::main]` attribute transforms the async `main` into a synchronous one that creates a runtime and runs the future to completion. This is the standard entry point for async Rust applications.

### What the Runtime Does

A runtime is responsible for:

1. **Polling futures.** It calls `.poll()` on each scheduled future to advance it.
2. **Suspending and resuming.** When a future returns "not ready," the runtime moves on to other work and comes back later.
3. **Managing I/O readiness.** It tracks which sockets, files, and timers are ready and which are still waiting.
4. **Distributing tasks.** With multiple threads, it can run tasks on multiple cores.

Tokio's runtime is multi-threaded by default, balancing tasks across worker threads. You can configure it for single-threaded operation if you need that:

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() { /* ... */ }
```

The single-threaded flavor is useful for embedded systems or for tasks that need predictable scheduling.

### Spawning Tasks

Within an async context, you spawn additional tasks with `tokio::spawn`:

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // some async work
        42
    });

    let result = handle.await.unwrap();
    println!("{result}");
}
```

This is similar to `thread::spawn` but creates a lightweight async task instead of an OS thread. Tokio can run thousands of tasks concurrently on a small number of threads.

`tokio::spawn` returns a `JoinHandle` (similar to `thread::JoinHandle`). Calling `.await` on it waits for the task to complete and returns its result.

---

## H. Async Functions and Futures

This section drills into the mechanics of how async functions work.

### Async Function Signatures

An async function returns a `Future`:

```rust
async fn fetch_count() -> u32 {
    42
}
```

The actual return type is `impl Future<Output = u32>`, but the `async` keyword lets you write the return type as the eventual value. The compiler handles the wrapping.

You can call `fetch_count()` to get a future, but the future does nothing until awaited:

```rust
let future = fetch_count();          // not running yet
let count = future.await;            // now it runs
```

### Await Points

A `.await` is a "yield point" where the current task can suspend. The runtime can switch to another task during the suspension:

```rust
async fn process() {
    let a = step_one().await;        // can suspend here
    let b = step_two(a).await;       // can suspend here
    let c = step_three(b).await;     // can suspend here
    println!("{c}");
}
```

While `step_one` is waiting for I/O, the runtime can run other tasks. This is what makes async efficient for many concurrent operations.

### Async in Closures and Methods

The `async` keyword works in many places:

```rust
async fn function() { /* ... */ }

let closure = || async { /* ... */ };       // returns a future when called

// Async method:
struct Service;
impl Service {
    async fn handle_request(&self) -> String {
        format!("processed")
    }
}
```

The `async {}` block is an expression that produces a future. This is useful for adapting non-async closures to async contexts.

### Running Multiple Futures Concurrently

To run two futures concurrently (and wait for both), use `tokio::join!`:

```rust
async fn fetch_a() -> String { String::from("a") }
async fn fetch_b() -> String { String::from("b") }

#[tokio::main]
async fn main() {
    let (a, b) = tokio::join!(fetch_a(), fetch_b());
    println!("{a}, {b}");
}
```

`join!` polls both futures to completion in parallel (in the cooperative sense). The total time is the maximum of the two, not the sum. This is one of the most useful patterns for I/O-heavy code.

### Common Pitfall: Accidentally Sequential

A common mistake is awaiting one future before starting the next:

```rust
// Sequential, slow:
let a = fetch_a().await;
let b = fetch_b().await;       // does not start until a completes
```

vs

```rust
// Concurrent, fast:
let (a, b) = tokio::join!(fetch_a(), fetch_b());
```

The first version waits for `fetch_a` to complete before starting `fetch_b`. The second version starts both immediately and waits for both. For independent operations, the second form is what you want.

---

## I. `async move` Closures and Spawning Tasks

Spawning tasks in Tokio works much like spawning threads but with async closures.

### `async move` Closures

The `move` keyword applies to async closures the same way it does to regular closures (Module 5):

```rust
#[tokio::main]
async fn main() {
    let data = vec![1, 2, 3];

    let handle = tokio::spawn(async move {
        for n in &data {
            println!("{n}");
        }
    });

    handle.await.unwrap();
}
```

The `async move` block takes ownership of `data`, just as a regular `move` closure would. This is necessary because the spawned task may outlive the function that created it.

### Why `'static` Bounds Appear

`tokio::spawn` requires its future to be `'static`, meaning it cannot borrow from local variables that might go out of scope. The `move` keyword satisfies this by taking ownership.

If you forget `move`:

```rust
let data = vec![1, 2, 3];
tokio::spawn(async {
    for n in &data { /* ... */ }     // ERROR: borrows data
});
```

The compiler rejects this. The error mentions `'static`, which is the lifetime requirement for spawned tasks.

### Sharing Data Across Tasks

If multiple tasks need the same data, use `Arc`:

```rust
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let shared = Arc::new(vec![1, 2, 3]);

    let mut handles = vec![];
    for i in 0..3 {
        let data = Arc::clone(&shared);
        let handle = tokio::spawn(async move {
            println!("task {i}: {data:?}");
        });
        handles.push(handle);
    }

    for h in handles {
        h.await.unwrap();
    }
}
```

Each task gets its own `Arc` (a clone of the shared one). Each `Arc` is moved into the task. The underlying data is shared via reference counting, the same way it works with threads.

For mutable shared state in async code, use `tokio::sync::Mutex` rather than `std::sync::Mutex`. The async version's `lock()` returns a future, which can be awaited without blocking other tasks. The standard `Mutex` would block the thread, defeating the purpose of async.

### Channels in Async Code

Tokio provides async-aware channels in `tokio::sync::mpsc`:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(100);

    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(format!("message {i}")).await.unwrap();
        }
    });

    while let Some(msg) = rx.recv().await {
        println!("{msg}");
    }
}
```

The async versions of `send` and `recv` return futures that suspend when the channel is full or empty, instead of blocking the thread. This is the appropriate choice in async code.

---

## J. Selecting with `tokio::select!`

When you need to wait on multiple futures and act when the first one completes, use `tokio::select!`.

### The Basic Pattern

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let timeout = sleep(Duration::from_secs(2));
    let work = async {
        sleep(Duration::from_secs(5)).await;
        "work completed"
    };

    tokio::select! {
        _ = timeout => {
            println!("timed out");
        }
        result = work => {
            println!("got: {result}");
        }
    }
}
```

The `select!` macro polls all the futures concurrently. The first one to complete wins; the others are cancelled. The arm corresponding to the winning future runs its block.

In this example, `timeout` completes first (2 seconds is shorter than 5), so the program prints "timed out."

### Common Use Cases

`select!` is useful for:

- **Timeouts.** Race a future against a timer; whichever completes first determines the outcome.
- **Multiple sources.** Wait on multiple channels or events; act on whichever arrives first.
- **Cancellation.** Provide an early-exit signal alongside long-running work.

### Cancellation Semantics

When a `select!` arm wins, the futures from the other arms are dropped. This cancels them: they stop running. For most futures, this is safe; the cancellation point is wherever they were last awaiting.

For futures that hold resources (locks, file handles), cancellation can cause subtle issues. The general rule: keep `select!` arms focused, and avoid awaiting state-modifying operations across `select!` if you can.

### A Realistic Example

A simple HTTP server that processes requests with a timeout:

```rust
async fn handle_request(request: Request) -> Result<Response, Error> {
    let timeout = tokio::time::sleep(Duration::from_secs(30));

    tokio::select! {
        _ = timeout => {
            Err(Error::Timeout)
        }
        result = process_request(request) => {
            result
        }
    }
}
```

If `process_request` does not complete within 30 seconds, the timeout wins and the request is cancelled with a timeout error. This pattern is fundamental to robust networked services.

---

## K. Common Async Patterns: HTTP Clients with `reqwest`

Most async code in production deals with network I/O. The `reqwest` crate is the standard HTTP client in Rust.

### Adding `reqwest`

In `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = "0.12"
```

### A Simple GET Request

```rust
#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let body = reqwest::get("https://www.rust-lang.org")
        .await?
        .text()
        .await?;

    println!("body: {}", &body[..200]);
    Ok(())
}
```

`reqwest::get(url)` returns a future. Awaiting it gives you a `Response`. Calling `.text()` on the response returns another future that produces the body as a `String`. The `?` operator handles errors at each step (Module 10).

### Multiple Concurrent Requests

The whole point of async is to make many I/O operations efficient. To fetch multiple URLs concurrently:

```rust
use futures::future::join_all;

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let urls = vec![
        "https://www.rust-lang.org",
        "https://crates.io",
        "https://docs.rs",
    ];

    let client = reqwest::Client::new();

    let bodies = join_all(urls.into_iter().map(|url| {
        let client = client.clone();
        async move {
            client.get(url).send().await?.text().await
        }
    })).await;

    for (i, body) in bodies.iter().enumerate() {
        match body {
            Ok(text) => println!("URL {i}: {} bytes", text.len()),
            Err(e) => println!("URL {i}: error: {e}"),
        }
    }

    Ok(())
}
```

`join_all` (from the `futures` crate, often pulled in as a dependency of `tokio`) waits for all the futures to complete. The requests run concurrently, so the total time is the slowest request, not the sum.

### POST Requests with JSON

For APIs that accept JSON:

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Deserialize, Debug)]
struct User {
    id: u64,
    name: String,
    email: String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = reqwest::Client::new();

    let new_user = CreateUser {
        name: String::from("Alice"),
        email: String::from("alice@example.com"),
    };

    let response: User = client
        .post("https://api.example.com/users")
        .json(&new_user)
        .send()
        .await?
        .json()
        .await?;

    println!("created: {response:?}");
    Ok(())
}
```

The `serde` crate provides serialization to and from JSON (and many other formats). Deriving `Serialize` and `Deserialize` on structs lets `reqwest` automatically convert them.

### Error Handling

Network requests can fail in many ways. The `reqwest::Error` type represents these failures:

- Connection failures (DNS resolution, TCP connection refused).
- Timeouts.
- TLS errors.
- Non-success status codes (if you call `.error_for_status()`).
- JSON parsing errors (if you used `.json()`).

For robust code, propagate errors with `?` and handle them at the appropriate level. The error handling patterns from Module 10 apply directly.

### When Async Pays Off

The `reqwest` example shows where async shines: many independent I/O operations. Fetching 100 URLs sequentially takes the sum of all request times. Fetching them concurrently takes the maximum. For network-heavy applications (web crawlers, API aggregators, microservice clients), this is often a 10x to 100x speedup.

For CPU-bound work (sorting, computation, image processing), async does not help. Threads (or thread pools via `rayon`) are the right tool.

---

## Module Summary

- Rust's ownership system extends to concurrent code. The borrow rules prevent data races at compile time, regardless of how many threads or tasks are involved.
- Threads are spawned with `thread::spawn` and joined with `JoinHandle::join`. Closures usually need `move` because spawned threads may outlive their parent.
- Channels (`mpsc`) provide message-passing concurrency. Senders can be cloned for multiple producers; the receiver is single-consumer.
- Shared state uses `Arc<T>` for ownership, plus `Mutex<T>` or `RwLock<T>` for synchronization. The combination `Arc<Mutex<T>>` is one of the most common patterns in concurrent Rust.
- `Send` and `Sync` are traits that classify types by thread safety. Most types are `Send`; thread-unsafe types like `Rc` and `RefCell` are not.
- Async programming uses `async`/`await` syntax to write code that suspends at I/O points instead of blocking. Async functions return futures.
- The Tokio runtime is the standard executor. `#[tokio::main]` provides the entry point; `tokio::spawn` creates tasks.
- `tokio::join!` runs futures concurrently. `tokio::select!` waits for the first of several futures to complete.
- The `reqwest` crate is the standard HTTP client and represents typical async API design: a client struct, methods returning futures, and error handling via `Result`.

## Discussion Questions

1. Rust's ownership rules prevent data races at compile time. This is unusual; most languages either allow data races (C, C++) or prevent them at runtime (Go, Java with `synchronized`). What does Rust gain from compile-time prevention, and what does it cost in code that would compile easily in those languages?
2. Async and threads serve different purposes. Identify a specific real-world workload where async is the right choice, one where threads are the right choice, and one where you might use both together.
3. The `Send` and `Sync` traits are checked automatically by the compiler. You rarely implement them yourself, but you sometimes have to fight them when the compiler refuses to send something. Identify a category of bug that this prevents, and explain why catching it at compile time is more useful than catching it at runtime.
