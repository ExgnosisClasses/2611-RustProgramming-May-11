# Module 14: Concurrent and Async Programming
## Lab 10 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Spawning Threads

### Checkpoints

**1. Why did the program in Task 1.1 end before the spawned thread finished?**

When `main` returns, the entire process exits, and all threads are terminated abruptly. The OS does not wait for spawned threads to finish before tearing down the program.

This is a deliberate design choice. The main thread has a special role: it is the lifetime anchor for the program. Other threads are "background" by default; they run for as long as the program runs, but they do not extend its life.

The practical implication: if you spawn a thread and want it to finish its work, you must explicitly join it. Calling `join()` on the handle blocks until the thread completes. Without join, the thread might be cut off mid-work when the program exits.

This is different from some languages (Java, for instance) where the runtime tracks "non-daemon" threads and waits for them. Rust's model is simpler: the main thread is special, others are not. If you want a thread to extend the program's lifetime, join it explicitly.

For long-running background tasks (server workers, monitoring threads), this can be a footgun. The common patterns: explicit join at the end, or sending a shutdown signal and waiting for clean termination. Production code rarely relies on "let the thread run; we'll catch it later" patterns because they are unreliable.

**2. Why is `move` necessary for thread closures specifically?**

The thread might outlive any local data borrowed from the surrounding scope. Consider:

```rust
fn main() {
    let message = String::from("hello");

    thread::spawn(|| {
        println!("{message}");      // borrows message
    });

    // What if main returns here?
}
```

If the closure borrowed `message`, the thread would hold a reference into `main`'s stack frame. When `main` returns, that stack frame is gone, and the reference would be invalid. The thread might still be running, trying to read freed memory.

Rust's lifetime system catches this: it sees the borrow and computes that the thread (which might run for an arbitrarily long time) would outlive the borrow's source. The compiler refuses the borrow and tells you to use `move`.

`move` transfers ownership into the closure. The thread owns the data; the thread's data lives as long as the thread. The borrow problem is gone.

For closures passed to iterator methods like `map`, the closure is called synchronously within the function. There is no "outlives" concern because the function call ends before any of the borrowed data could be invalidated. The closure can borrow safely; `move` is not required.

The distinction is about lifetime relationships. Threads escape the current scope; iterator closures do not. The borrow checker recognizes the difference and applies different rules.

**3. What is the benefit of thread panics being non-fatal?**

A process should not necessarily die because one thread had a problem. Consider a server handling many concurrent connections in threads. If one thread panics on a malformed request, the server should log the issue and keep serving other clients, not crash entirely.

Rust's design makes this possible: a panic in a spawned thread unwinds that thread's stack and ends the thread, but the rest of the program continues. The main thread (or whoever has the join handle) can detect the panic and respond.

Compare with languages where any uncaught exception in any thread tears down the process. Those have to either crash a lot or wrap every thread body in elaborate exception handling. Rust's design makes the cleanup automatic: the thread dies cleanly, the program continues, and you discover the death via `join`.

For production code, the pattern is usually: thread panics are logged, possibly trigger a restart (for worker pools), and may flag a metric. The fact that the program survives the panic is what enables this pattern.

There is one important caveat: a panic in a thread holding a `Mutex` "poisons" the mutex. Other threads acquiring the mutex see `Err(PoisonError)`. This is a signal that "the state behind this mutex may be inconsistent." Handling it correctly takes care; the default `unwrap()` will propagate the panic.

---

## Exercise 2 -- Sending Data Between Threads with Channels

### Checkpoints

**1. Why prefer message passing over shared state with locks?**

Several reasons:

- **Ownership clarity.** When you send a value through a channel, you transfer ownership. Only one thread can own a given piece of data at a time. This eliminates a class of concurrency bugs (using data after another thread has modified it) by design.

- **No lock contention.** Locks are cheap when uncontended but expensive when many threads compete. Channels avoid this: each send/recv is a single operation; there is no "everyone waiting for the lock" scenario.

- **Simpler reasoning.** Code that uses channels is easier to read: "this thread produces values; that thread consumes them." Shared state with locks requires you to remember which locks protect what and in what order to acquire them. The mental load is higher.

- **Avoids common bugs.** Deadlocks (two threads each holding a lock the other needs) cannot happen with simple channels. Order-of-acquisition bugs disappear. Each thread independently sends or receives.

The slogan in the Rust community is "share by communicating, not communicate by sharing." This was inherited from Go, but applies to Rust too.

When does shared state with locks make sense? When you genuinely have a single piece of data that multiple threads need to read and write (a counter, a cache, a shared collection). Channels do not fit well for that. The combination `Arc<Mutex<T>>` is the right tool.

For most other concurrent patterns (producer-consumer, fan-out, fan-in, work distribution), channels are clearer and safer.

**2. What happens if the sender is never dropped?**

The channel remains open. The receiver's `recv()` calls block forever (or `for received in rx` loops forever). The program hangs.

This sounds bad but is actually desirable. Consider a server: it processes requests on a channel. The sender (the part that receives connections and enqueues work) stays alive for the program's lifetime. The receiver (the worker) wants to keep running too. The channel staying open is exactly what you want.

For programs that should terminate when work is done, the sender must be dropped to signal completion. Standard patterns:

- **The sender is in a producer task that exits.** When the task completes, its locals (including the sender) are dropped.
- **Explicit drop.** `drop(tx)` releases the sender.
- **Move and forget.** The sender is moved into a finite-lifetime context that ends naturally.

If you find your program hanging in `recv()`, "did all senders get dropped?" is the question to ask. Usually the answer is "no, one is still alive somewhere unexpectedly."

**3. What problem does backpressure prevent?**

Unbounded memory growth. Without backpressure, a fast producer feeding a slow consumer would buffer up arbitrarily many messages. The channel's internal buffer grows without limit. Eventually the program runs out of memory.

This is a real problem in production systems. A backend service that accepts incoming requests faster than it can process them might queue up millions of pending requests, consuming all available RAM and dying. The right behavior is usually to reject requests when the queue is full, signaling to upstream clients to slow down or fail fast.

Backpressure (via bounded channels) makes this signaling automatic. When the buffer fills, `send()` blocks, throttling the producer. If the producer is itself receiving work from somewhere, the throttling propagates backward. Eventually, the original source of work either slows down or is forced to drop work.

When is unbounded buffering preferable?

- **Low-volume signaling.** A control-plane channel that carries occasional commands (shutdown, reload config) does not need backpressure; the volume is too low to matter.
- **Latency-sensitive work where blocking the sender is worse than queueing.** If your producer thread cannot afford to block, you accept the memory risk to keep it running.
- **Memory is genuinely abundant.** For small in-process queues, the memory cost is negligible.

In production, the conventional wisdom is "use bounded channels by default; reach for unbounded only when you know why." Tokio's channels are bounded by default for exactly this reason.

---

## Exercise 3 -- Shared State with Mutex

### Checkpoints

**1. Why is `Arc` needed when sharing a Mutex across threads?**

Two reasons combine.

First, you need shared ownership: multiple threads each hold a reference to the same Mutex. `Mutex<T>` itself is just a value; it does not provide a way to share it. You could use a regular reference (`&Mutex<T>`), but that has lifetime constraints that do not work for threads (the thread might outlive the borrow).

Second, you need atomic reference counting because the sharing crosses thread boundaries. `Rc` would not work (it is not `Send`). `Arc` is the thread-safe version: atomic ref counting that is safe across threads.

`Arc<Mutex<T>>` is the canonical combination: `Arc` for shared ownership across threads, `Mutex` for synchronized access to the inner value. Try to use either alone and you hit a compile error:

- `Mutex<T>` alone: cannot be cloned, so cannot be given to multiple threads.
- `Arc<T>` alone: gives you shared read-only access (T must be `Sync`); does not provide mutation.

Together, they cover the common case: multiple owners, synchronized mutation. The pattern appears so often that you can recognize it on sight.

**2. Why is "hold the lock briefly" such an important rule?**

A lock serializes access. While one thread holds the lock, no other thread can acquire it; they wait. The longer you hold the lock, the longer other threads wait. The longer they wait, the less concurrent your program is.

In the extreme case (Task 3.4), holding the lock during a 100ms sleep means each thread waits 100ms for the previous one. Five threads × 100ms = 500ms total, even though the actual work per thread is essentially zero. The threads are serialized; concurrency was lost.

The rule applies to many operations beyond sleep:

- **I/O.** Reading a file, writing to a database, making a network call. All slow; holding the lock through these is wasteful.
- **Computation.** Anything CPU-intensive that does not need shared state. Move it outside the lock.
- **Allocation.** Even ordinary heap allocations can be slow under memory pressure; keep them out of the lock when possible.
- **Other locks.** Acquiring multiple locks while holding a first one is a recipe for deadlock and contention.

The pattern:

```rust
// Bad: hold lock during slow work
let mut data = shared.lock().unwrap();
data.process();                    // slow!
*data += 1;
```

```rust
// Good: lock only for shared access
let result = compute_slow();        // outside the lock
let mut data = shared.lock().unwrap();
*data += result;
// data is dropped here; lock released
```

The good version computes the slow part without holding the lock, then briefly acquires the lock to apply the result.

This is sometimes called "critical section design." The critical section is the code that must hold the lock. Keep it small; everything else stays outside.

**3. When does RwLock benefit you over Mutex?**

When reads vastly outnumber writes. The classic example is a configuration object that is loaded once at startup and read frequently by many threads. With `Mutex`, every read takes the lock, serializing access. With `RwLock`, many threads can read simultaneously; only writers are serialized.

For workloads like:

- A cache that is mostly read.
- A configuration that changes rarely.
- A lookup table populated at startup.

`RwLock` is the better choice.

For workloads like:

- A counter that every thread increments.
- A queue where threads both push and pop.
- A state machine where every transition modifies state.

`Mutex` is simpler and roughly as fast (the read-write distinction does not buy you much when most operations write).

A practical caveat: `RwLock` is more complex than `Mutex` internally. The cost of acquiring an RwLock is higher than acquiring a `Mutex`. For truly hot paths with simple operations, `Mutex` can outperform `RwLock` even on read-heavy workloads. As always, measure before optimizing.

A subtle pitfall: `RwLock` can starve writers if there is a constant stream of readers. The exact behavior depends on the implementation, but you should not assume "many readers, occasional writer" gets fair treatment. For critical writer fairness, use a different primitive or a `Mutex` with appropriate scheduling.

---

## Exercise 4 -- Send and Sync

### Checkpoints

**1. What kind of bug does `Send` prevent?**

Data races on non-atomic operations. The most concrete example is `Rc`'s reference counter.

When you clone an `Rc`, the implementation increments a counter. In a single-threaded context, this is just `count += 1`. In a multi-threaded context, this becomes a race: if two threads clone simultaneously, both might read the same count, both write count+1, and the count is now off by one.

The consequences:

- Eventually, the count reaches zero "too early," freeing the data while clones still exist. Use-after-free.
- Or the count never reaches zero, leaking memory.

These bugs are notoriously hard to catch in testing (they depend on timing) and devastating in production (crashes or memory leaks).

In a language without compile-time Send checks (C++, Java, Python, etc.), you can write code like this:

```cpp
// C++ pseudocode
std::shared_ptr<int> data = make_shared(42);
std::thread t([&]() {
    auto local = data;        // increment non-atomically
});
auto local2 = data;            // another non-atomic increment
```

This compiles but races. The fix is to use `std::atomic` reference counting (Boost's `intrusive_ptr` or careful std::shared_ptr usage). The compiler does not enforce this; you have to remember.

In Rust, `Rc` is `!Send`. You cannot move it across threads. The compiler refuses. To share across threads, you use `Arc` (atomic). The right tool is enforced by the type system; choosing the wrong one is a compile error, not a runtime crash.

This is one of Rust's most distinctive safety guarantees. The class of bugs is real and common in other languages; Rust eliminates it at compile time.

**2. Why does the "all fields must be Send" composition rule make sense?**

A struct's safety properties are determined by its fields. If a struct has a `Rc` field, the struct's bits include a pointer to data that uses non-atomic reference counting. Moving the struct moves the pointer. If you moved it to a thread that also has a clone of the same `Rc`, you would have non-atomic counts across threads, recreating the bug `Send` is supposed to prevent.

The rule "all fields must be `Send` for the struct to be `Send`" propagates safety automatically. You do not have to mark each composite type manually; the compiler determines it from the constituents.

This is the same auto-derivation pattern used for `Copy`, `Send`, `Sync`, and several other marker traits. The properties propagate through composition.

There are exceptions. Sometimes a type has fields that would prevent automatic `Send`/`Sync` but is actually safe to send (because of synchronization the user cannot see, for instance). In those cases, the type uses `unsafe impl Send` to manually declare itself as `Send`. This is rare and requires careful justification; most types let the automatic derivation handle it.

**3. Why are `Arc` and `Mutex` separate types?**

They have different responsibilities. `Arc` handles shared ownership: many places hold a pointer to the same data, with reference counting to know when to free. `Mutex` handles synchronization: ensure only one thread reads or writes at a time.

These are orthogonal concerns. You might want:

- `Arc<T>` without `Mutex`: shared read-only access to immutable data. Multiple threads read the same `T`; no mutation, no synchronization needed.
- `Mutex<T>` without `Arc`: synchronization within a single owner. Useful in some single-threaded patterns or when the Mutex is owned by a long-lived structure.
- `Arc<Mutex<T>>`: the combined case. Shared ownership AND synchronized mutation.

By separating the two, the language gives you compositional flexibility. You combine them as needed.

A single "thread-safe shared value" type would have to pick a synchronization strategy: probably `Mutex`-like. But `RwLock` is also useful, and `RefCell` (for single-threaded interior mutability), and atomic types for simple cases. Forcing one combination would be restrictive.

The composition is verbose (`Arc<Mutex<T>>` is more typing than a hypothetical `ThreadSafe<T>`) but it makes the design explicit. Reading `Arc<Mutex<T>>` tells you immediately: shared ownership, synchronized mutation. Reading `ThreadSafe<T>` would tell you only "thread-safe somehow."

Rust generally prefers explicitness over convenience. The composition pattern reflects this philosophy.

---

## Exercise 5 -- Async/Await with Tokio

### Checkpoints

**1. Why does `async fn` return a Future rather than the value directly?**

An async function is suspended at every `.await` point. While suspended, it does no work; it yields control to the runtime. Other tasks run. When the awaited operation is ready, the function resumes.

For this to work, the function cannot just "run and return a value." It must produce something that the runtime can manage: a state machine that can be advanced step by step. That state machine is the `Future`.

The Future is essentially "a function call that may take many steps to complete." Calling `async fn foo()` returns the Future, which represents "the work needed to compute foo's result." Awaiting the Future runs the work, with the runtime managing suspensions and resumptions.

This is fundamentally different from how synchronous functions work. A sync function runs to completion before returning. An async function is more like a generator: it has state, can pause, can resume, eventually produces a final value.

The mechanics are complex but the user-visible effect is simple. You write `async fn` to declare an async function. You call it and `.await` the result. The runtime handles the state-machine work.

**2. How is `tokio::join!` different from running operations as threads?**

`tokio::join!` runs futures concurrently on the same thread (or possibly multiple threads, depending on the runtime). The futures yield at await points, allowing other futures to run. The yield points are explicit (`.await`); the runtime cannot interrupt at arbitrary points.

Threads, by contrast, run truly in parallel (on multiple CPU cores) and can be preempted by the OS at any point. The OS scheduler decides when to switch between threads; the program does not control it.

Practical implications:

- **Cooperative scheduling:** async tasks yield only at await points. If a task does CPU-intensive work without awaiting, it blocks all other tasks on its thread. Threads do not have this issue; the OS preempts them.
- **Cheap tasks:** an async task is much cheaper than a thread. You can have thousands of async tasks; you would struggle to have thousands of threads.
- **No real parallelism (single-threaded runtime):** by default, Tokio uses a multi-threaded runtime, so async tasks do get some parallelism. But the runtime can run as single-threaded too, in which case tasks share one thread cooperatively.

The two models are complementary, not competing. Use threads for CPU-bound parallelism. Use async for many-concurrent-things-mostly-waiting.

**3. 1000 HTTP requests: threads or async tasks?**

Async tasks, by a wide margin.

Each thread requires its own stack (typically 1MB or more), substantial OS bookkeeping, and is expensive to create and destroy. 1000 threads is 1GB+ of stack memory, slow startup, and heavy scheduler load.

Each async task requires kilobytes of memory and is created in microseconds. 1000 tasks is a few megabytes total; the runtime handles them on a handful of OS threads.

For an HTTP-request workload, most of the time is spent waiting for the network. Async excels here: tasks yield to the runtime while waiting, allowing other tasks to make progress.

CPU-bound workloads are different. If you wanted to compute 1000 large matrix multiplications, async would not help: the work is not waiting; it is computing. Threads (limited to roughly the number of CPU cores) would give you actual parallelism. More than that becomes wasteful because the cores are saturated.

The decision criterion:

- **Waiting-bound workloads** (network I/O, file I/O, database queries, sleeping): async.
- **Compute-bound workloads** (matrix multiplication, encryption, parsing): threads (with the count matching CPU cores).
- **Mixed workloads:** typically async for the wait-heavy parts, with `spawn_blocking` to offload CPU-heavy tasks to a thread pool.

For most server applications (web servers, API gateways, message processors), the answer is async. The work is mostly I/O.

---

## Exercise 6 -- Async Coordination

### Checkpoints

**1. Why does Tokio provide its own channels rather than reusing std?**

`std::sync::mpsc` is synchronous: `recv()` blocks the calling thread until a message arrives. In an async context, blocking the thread is exactly what you want to avoid. Blocking holds onto the thread, preventing other tasks from running on it.

Tokio's channels are async: `recv().await` yields control while waiting. When a message arrives, the runtime wakes the waiting task. The thread is free to run other tasks during the wait.

For an async program with hundreds of tasks coordinating through channels, using sync channels would force most tasks to block their threads. The runtime's efficiency would collapse.

There are other differences. Tokio's channels are bounded by default (with explicit capacity). They integrate with `select!` for waiting on multiple channels. They support patterns specific to async (e.g., `oneshot::channel` for single-use signaling).

The general pattern: in an async program, use async-aware primitives (async channels, async mutexes, async I/O). Sync primitives still work but introduce subtle performance issues by blocking threads.

**2. What happens to side effects when a future is cancelled?**

The future stops executing at its current point. Any work it has not yet done is abandoned. Any work it has already done remains, in whatever state it was in.

This can cause problems:

- **Partial mutations.** If the future was in the middle of updating a shared data structure (e.g., between two `lock().await` calls), the structure could be in an inconsistent state. Other tasks may see this inconsistency.
- **Unfinished cleanup.** If the future was responsible for releasing a resource (closing a file, returning a connection to a pool), the cleanup may not happen. The resource leaks.
- **Half-done network operations.** Sending part of a request, then being cancelled, may leave the server in an unexpected state.

To handle this properly:

- **Make operations atomic where possible.** Each await point is a potential cancellation boundary; design so that any cancellation point leaves the system in a consistent state.
- **Use `Drop` for cleanup.** Cleanup that runs on Drop happens even on cancellation. RAII-style cleanup is automatic.
- **Avoid spanning await points with critical state.** If you must update multiple things atomically, do it without awaits between them.

Some patterns handle cancellation gracefully:

```rust
async fn with_cleanup() {
    let resource = acquire().await;
    do_work(&resource).await;
    cleanup(resource).await;     // might be cancelled
}
```

If cancelled between `do_work` and `cleanup`, the cleanup never runs. The fix is to use Drop:

```rust
async fn with_cleanup() {
    let resource = acquire().await;
    // resource's Drop handles cleanup
    do_work(&resource).await;
}
```

Now cancellation triggers Drop, which handles cleanup. The pattern is more robust.

**3. Why does std::sync::Mutex not work well across await points?**

Two reasons.

First, holding a `std::sync::Mutex` across an `.await` is a concrete deadlock risk. The Mutex blocks the thread; the async runtime might schedule another task on the same thread; that task might try to acquire the same Mutex; deadlock. The two locks "deadlock" because both are waiting, neither can proceed.

Second, even when not deadlocking, the standard Mutex blocks the thread. Async runtimes have a small number of worker threads; blocking one means that thread cannot run other tasks. If many tasks block on the standard Mutex, throughput collapses.

Tokio's async Mutex (`tokio::sync::Mutex`) solves both:

- It yields while waiting, allowing other tasks to run on the thread.
- It does not block the thread; it uses the runtime's wait/notify mechanism.

The rule: if you might hold a lock across an `.await`, use `tokio::sync::Mutex`. If the lock is acquired and released without any awaits, the standard Mutex is fine (and slightly faster due to less overhead).

Many real bugs come from violating this rule. A `std::sync::Mutex` held across an `.await` looks innocuous but creates subtle latency or deadlock issues. The Tokio docs and the `tokio::sync::Mutex` documentation explicitly warn about this.

---

## Exercise 7 -- Async Networking

### Checkpoints

**1. Why was the concurrent version 5x faster?**

The sequential version spent each second waiting for the network response, then started the next request. Five requests × 1 second of waiting = 5 seconds total.

The concurrent version started all five requests roughly simultaneously. While one was waiting, the others were also waiting. The bottleneck was the single longest request (about 1 second). Total time ≈ 1 second.

The savings came from overlapping wait times. The wall-clock time was dominated by the longest single request rather than the sum.

This is the central value of async I/O. When your program is mostly waiting (for network, disk, database), parallelizing the waits gives near-linear speedup up to the number of concurrent operations the server can handle.

The same speedup is achievable with threads (running 5 threads, each making a sync request, would also take about 1 second). The async advantage is that you can do this with much less memory and overhead, supporting thousands of concurrent operations rather than dozens.

**2. Why would async not provide 5x speedup for CPU-bound work?**

CPU-bound work does not wait. Computing a hash uses the CPU continuously. There is no "idle time" during which other tasks could run.

If you have 1 CPU core and 5 tasks each needing 1 second of CPU work, you cannot do them in less than 5 seconds total. The work is sequential by physical constraint.

Async does not change this. The async runtime runs tasks on threads; the threads run on CPU cores. If all 5 tasks are CPU-bound, they compete for the same cores. Even if they all "yield" at await points, they still need to actually do their work, and the work has to fit in the available CPU time.

What does provide speedup for CPU-bound work? Multiple CPU cores. With 5 cores, you can run 5 tasks in parallel, each on its own core. Threads or async tasks both work for this, but the parallelism is limited by core count, not by how many tasks you have.

For very CPU-bound work, the right pattern is often:

- Use threads (or async with `spawn_blocking`) to dispatch work to CPU cores.
- Match the number of workers to the number of cores (typically `num_cpus::get()`).
- Don't create more workers than cores; the extra workers just compete with each other for CPU time without providing more throughput.

**3. Break-even point for async with fast requests?**

The break-even point depends on the overhead of async versus the time saved by concurrency.

For a request that takes 10ms:

- Sequential: 5 × 10ms = 50ms.
- Concurrent: ~10ms (the longest single request), plus a small overhead for spawning tasks (microseconds per task).

5x speedup, similar to the slow case. The relative speedup is the same regardless of request duration; only the absolute savings differ.

For a request that takes 100μs (100 microseconds):

- Sequential: 5 × 100μs = 500μs.
- Concurrent: 100μs plus overhead.

If task spawning costs 50μs per task, you have 50μs × 5 = 250μs of overhead. The concurrent version is 100μs + 250μs = 350μs, vs sequential 500μs. Still faster, but the gap is smaller.

For requests that take 10μs, the spawn overhead could exceed the work itself. At some point, the overhead of concurrency outweighs the savings.

In practice, async is appropriate when:

- Each operation involves measurable I/O wait time (>100μs typically).
- You have many concurrent operations (>10 typically).
- The total throughput is dominated by waits, not work.

For very fast operations or low concurrency, the overhead of async may not pay off. Synchronous code with a thread pool is often simpler and just as fast.

---

## Exercise 8 -- Putting It Together

### Task 8.2 -- Pattern identification

1. **Async function:** `process_task`, `worker_loop`, and the closures inside `tokio::spawn` blocks are all declared with `async fn` or `async move`.

2. **`.await`:** Many places: `sleep(...).await`, `rx.recv().await`, `tx.send(task).await`, `producer_handle.await`, `handle.await`.

3. **`#[tokio::main]`:** On the `main` function. Transforms it into a function that creates and runs a Tokio runtime.

4. **`tokio::spawn`:** Multiple places. Each worker is spawned with `tokio::spawn(worker_loop(...))`. The producer is spawned with `tokio::spawn(async move { ... })`.

5. **Async channel:** `mpsc::channel::<Task>(100)` creates a bounded async channel with capacity 100.

6. **Async mutex:** `Arc<Mutex<...>>` where the `Mutex` is `tokio::sync::Mutex` (imported as `use tokio::sync::{mpsc, Mutex}`).

7. **`Arc` for sharing:** `Arc::new(Mutex::new(rx))` and `Arc::new(Mutex::new(Vec::new()))`. The Arcs are cloned for each worker thread.

8. **Producer drops sender to close channel:** The producer task ends, dropping `tx`. When all senders are dropped, the channel closes, and `recv()` returns `None`, signaling workers to exit.

9. **Hold the lock briefly:** The `task = { let mut rx = receiver.lock().await; rx.recv().await }` pattern. The lock is acquired in a block; when the block ends, the lock is released. The lock is held only long enough to call `recv()`.

10. **Match on Option from recv:** `match task { Some(task) => ..., None => break }` distinguishes "got a task" from "channel closed."

### Task 8.3 -- Predict and verify

**Change A:** Reduce workers from 3 to 1.

**Prediction:** With only one worker, tasks are processed sequentially. Total elapsed time will be approximately the sum of all task durations: 200 + 100 + 300 + 150 + 250 + 50 + 400 + 100 = 1550ms (about 1.5 seconds), compared to ~600ms with 3 workers.

**Result:** As predicted. Total elapsed time will be roughly 1550ms, matching the total work. No concurrency benefit because only one worker processes tasks. The speedup ratio will be ~1.0x.

The lesson: the level of concurrency directly affects throughput. With more workers, more tasks run in parallel, finishing the queue faster. With fewer workers, the queue serializes.

**Change B:** Add `sleep(Duration::from_millis(50)).await` before acquiring the completed list lock.

**Prediction:** This will compile and run. The extra sleep adds 50ms to each task's processing time. Total elapsed time will increase by approximately 50ms × (tasks per worker, on average). The async runtime handles the sleep without blocking; the worker yields during the sleep, allowing other tasks to use the thread.

**Result:** The program runs without error. Total elapsed time increases by roughly 50ms × (longest worker's task count). The output shows each task taking 50ms longer.

The lesson: adding work that yields (like `sleep`) is handled gracefully by async. The runtime swaps in other tasks during the wait. The added latency is real, but there are no concurrency-breaking effects.

**Change C:** Remove the inner block around `recv()`, holding the receiver lock across `process_task`.

**Prediction:** The program will compile but will lose all concurrency benefits. Each worker, once it gets a task, holds the receiver lock while processing. Other workers cannot pick up tasks. The result is effectively single-worker performance: roughly 1550ms total.

**Result:** As predicted. The output shows that only one worker is active at a time. Tasks are processed sequentially, not in parallel.

The lesson: holding locks across long operations destroys concurrency. The "hold briefly" rule applies to async just as much as to threaded code. The cure is the inner scope that releases the lock as soon as the recv completes.

### Checkpoints

**1. Why `tokio::sync::Mutex` rather than `std::sync::Mutex`?**

The worker holds the receiver lock through an `.await` (during `rx.recv().await`). A `std::sync::Mutex` would block the thread while waiting for the recv to complete. With multiple workers on a small number of OS threads, this would cascade: blocked workers prevent the runtime from scheduling other work on their threads.

`tokio::sync::Mutex` yields during the lock acquisition, allowing other tasks to use the thread while waiting. The async runtime maintains throughput.

The completed list lock is similar: it is held in an async function and could conceivably span an `.await` (though in this code it does not). Using `tokio::sync::Mutex` consistently avoids the risk.

For locks that are guaranteed not to span awaits, `std::sync::Mutex` is fine and slightly more efficient. The judgment is: does the code path ever hold the lock across an `.await`? If yes (or if you cannot easily verify "no"), use the async mutex.

**2. Why is "channel close signals stop" a good design?**

Several advantages:

- **No special "stop" message.** You do not need to define a "shutdown" variant of your task type. The channel's natural closing acts as the signal.
- **Automatic for finite work.** When the producer's work is done, dropping the sender is the natural conclusion. The workers see the closure and exit.
- **Robust to producer failure.** If the producer crashes, the sender is dropped (via stack unwinding or process exit). The workers see the channel close and exit cleanly.
- **Composes with multiple producers.** If you have multiple senders, the channel closes when all of them are dropped. Workers see the channel close only when all producers are done.

Alternative designs:

- **Sentinel value:** "When you receive `Task::Shutdown`, stop." Requires extra logic in workers; can be missed if not handled.
- **Separate shutdown channel:** "Listen on the work channel and a shutdown channel; stop when shutdown signal arrives." More complex, but useful when you need to interrupt work mid-process.
- **Cancellation tokens:** Pass a token to each worker; check it periodically. Useful for cooperative cancellation but more complex to wire up.

For "finish all work, then stop," the channel-close pattern is the simplest and most idiomatic. For "stop NOW, abandon remaining work," cancellation tokens or `select!` with a shutdown channel are better.

**3. Could `completed` have been a channel instead?**

Yes, and arguably should be. A channel-based design:

```rust
let (results_tx, mut results_rx) = mpsc::channel::<TaskResult>(100);

// Each worker pushes results via the sender:
worker.send_result(results_tx.clone(), result).await;

// Main collects results from the receiver:
while let Some(result) = results_rx.recv().await {
    // process result
}
```

Advantages of the channel approach:

- **No shared mutable state.** Each worker has its own sender; no Mutex contention.
- **No locking overhead.** Channels are slightly cheaper than locks for high-frequency operations.
- **Natural backpressure.** A bounded channel forces workers to slow down if the consumer is slow.
- **More idiomatic async Rust.** "Communicate by channels" is the common pattern.

Disadvantages:

- **More setup.** You need both the task channel and the result channel.
- **Order is not preserved per-task.** Results arrive in completion order, not task-submission order.
- **Slightly more complex shutdown.** Need to ensure the result channel closes after all senders drop.

For this lab, the shared list with `Mutex` was chosen for didactic reasons (to demonstrate the pattern). In production code, the channel-based version is more idiomatic, but both work.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

Common patterns:

- Developers from C find threads familiar (similar to pthreads) but `Send`/`Sync` enforcement surprising. Channels feel familiar from Go.
- Developers from Java find threads familiar (similar to `Thread`) and channels somewhat familiar (similar to `BlockingQueue`). Async is the steepest part because it is conceptually different from Java's "every concurrent operation is a thread" model.
- Developers from Python find async familiar (similar to asyncio) but the rigorous error handling at every await point is more involved than Python's exception model.
- Developers from JavaScript find async very familiar (it follows the same Promise model) but threads less so.

The most common reflection: "The Mutex-poisoning behavior was surprising at first; I'm used to a panicked thread just dying and the lock being released. Once I understood that Rust treats lock holders specially (they might have left state inconsistent), it made sense."

Another common reflection: "I had to mentally adjust to async being cooperative. Coming from threads, I assumed I could do anything in any task, but tight CPU loops without awaits would block the runtime. The `spawn_blocking` escape hatch was important to find."

**2. When threads vs when async.**

Sample answer:

"For a CPU-bound batch processing job (compute statistics over millions of records, render high-resolution images, train a small ML model), threads are the right choice. The work is computation, not waiting. Threading gives true parallelism across CPU cores. Async would provide no benefit because there is nothing to wait for.

For a web server handling thousands of concurrent client connections, async is the right choice. Each connection mostly waits: for the client to send data, for the database to respond, for downstream services to reply. Async tasks let me handle 10,000 concurrent connections with a handful of OS threads. Threads would require 10,000 OS threads, which is impractical.

For a system that does both (e.g., a web service that does some expensive computation per request), the answer is async with `spawn_blocking` for the CPU-heavy parts. The HTTP-handling code is async; when it needs to do expensive work, it offloads to a thread pool. The async runtime is freed to handle other connections during the offloaded work.

The decision criterion I use: 'What is the program doing most of the time?' If waiting, async. If computing, threads (with parallelism). If both, async with selective offloading."

**3. A bug Rust prevents that you have seen elsewhere.**

Sample answer:

"In a Java codebase I worked on, we had a bug where multiple threads were modifying a `HashMap` without synchronization. The map was passed around between threads, with the assumption that since the writes were 'atomic' (single put operations), no synchronization was needed.

This is wrong: `HashMap` is not thread-safe. Internal state (rehash tables, etc.) can be corrupted by concurrent writes, leading to infinite loops, missing entries, or NullPointerExceptions in apparently sequential code paths. The bug appeared randomly under load, took weeks to track down, and required restructuring the threading model.

Rust would have prevented this at compile time. `HashMap<K, V>` is `Send` but not `Sync` (you cannot share `&HashMap` across threads when one might mutate). To share a HashMap, you would need `Arc<Mutex<HashMap<K, V>>>` (or `Arc<RwLock<...>>`). The compiler refuses the bare-HashMap-passed-between-threads pattern, forcing you to choose your synchronization explicitly.

The bug class is real and common. Rust's `Send`/`Sync` system makes it compile-time impossible. The cost is a slightly more verbose initial setup (explicit `Arc<Mutex<T>>` instead of bare `HashMap`); the benefit is that the class of bugs vanishes entirely. The trade-off is worth it in any concurrent codebase."

---

*End of Lab 10 Solutions*
