# Module 14: Concurrent and Async Programming
## Lab 10 -- Threads, Channels, Locks, and Async/Await with Tokio

> **Course:** Mastering Rust
> **Module:** 14 - Concurrent and Async Programming
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 14: spawning threads, the `move` keyword for closures sent to threads, communicating with channels, sharing state with `Mutex` and `Arc`, the `Send` and `Sync` traits, and the async/await model with the Tokio runtime. You will build small programs that exercise each pattern, then a single integration program that combines them.

You will build a single Cargo project called `task_processor` that simulates a small concurrent task processing system. The project starts with simple thread spawning, adds channels for producer-consumer patterns, then shared state with `Arc<Mutex<T>>`, then async equivalents with Tokio. The final integration program processes tasks concurrently using async patterns. Each exercise builds on the previous one.

The focus is on judgment: when to use threads versus async, when channels are clearer than shared state, when to choose `Mutex` versus `RwLock`. Many decisions have multiple defensible answers; the lab guides you toward idiomatic Rust and asks you to compare alternatives.

By the end of this lab you will be able to:

- Spawn threads with `thread::spawn` and join them.
- Use `move` closures to send data into threads.
- Use channels (`mpsc::channel`) to send messages between threads.
- Share state across threads with `Arc<Mutex<T>>` and `Arc<RwLock<T>>`.
- Recognize the `Send` and `Sync` traits and what they require.
- Write async functions and call them with `.await`.
- Use the Tokio runtime to run async code.
- Spawn async tasks with `tokio::spawn` and coordinate them.
- Use `tokio::select!` to wait on multiple async operations.
- Choose between threads and async based on the workload.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** Exercises 5 through 8 use the Tokio runtime, which is not part of the standard library. It will be added through `Cargo.toml`, and Cargo will download it on the first build. Make sure you have an internet connection when you reach Exercise 5.

> **A note on concurrency programs.** Concurrent programs are inherently nondeterministic. The output order may differ between runs because threads or async tasks complete in unpredictable orders. The lab's expected outputs show one possible interleaving; yours may differ. The assertions in tests focus on what is deterministic (e.g., a final count is correct regardless of order, all tasks completed). When you see different output ordering, that is normal.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new task_processor
cd task_processor
code .
```

Create a `lab10-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Spawning Threads

**Estimated time:** 10 minutes
**Topics covered:** `thread::spawn`, `join`, thread identity, basic concurrency

### Context

A thread is an independent flow of execution running concurrently with the main program. Rust's standard library provides `thread::spawn` to create one. This exercise establishes the basic mechanics.

### Task 1.1 -- The basic spawn

Replace the contents of `src/main.rs` with:

```rust
use std::thread;
use std::time::Duration;

fn main() {
    println!("main: starting");

    thread::spawn(|| {
        for i in 1..=5 {
            println!("  thread: count {i}");
            thread::sleep(Duration::from_millis(100));
        }
    });

    for i in 1..=3 {
        println!("main: count {i}");
        thread::sleep(Duration::from_millis(150));
    }

    println!("main: done");
}
```

Run the program:

```bash
cargo run
```

Expected output (interleaving may vary):

```
main: starting
main: count 1
  thread: count 1
  thread: count 2
main: count 2
  thread: count 3
main: count 3
  thread: count 4
main: done
```

Notice two things:

- The main and spawned threads run concurrently, interleaving their output.
- The spawned thread may not finish before main does. When main returns, all threads are terminated abruptly. In this case, the spawned thread is cut off mid-count.

### Task 1.2 -- Waiting with join

To wait for a thread to finish, use the handle's `join` method:

```rust
use std::thread;
use std::time::Duration;

fn main() {
    println!("main: starting");

    let handle = thread::spawn(|| {
        for i in 1..=5 {
            println!("  thread: count {i}");
            thread::sleep(Duration::from_millis(100));
        }
        "thread done"
    });

    for i in 1..=3 {
        println!("main: count {i}");
        thread::sleep(Duration::from_millis(150));
    }

    let result = handle.join().expect("thread panicked");
    println!("main: thread returned '{result}'");
}
```

Run the program. Now the output ends with all 5 thread counts because main waits:

```
main: starting
main: count 1
  thread: count 1
  thread: count 2
main: count 2
  thread: count 3
main: count 3
  thread: count 4
  thread: count 5
main: thread returned 'thread done'
```

Two things to notice:

- `thread::spawn` returns a `JoinHandle`. The handle can be used to wait for the thread to finish.
- `join()` returns the value the thread returned (wrapped in `Result` to handle panics). The thread's return value was "thread done"; that string is what main printed.

### Task 1.3 -- Capturing values with move

Try to use a value from the surrounding scope in a thread:

```rust
fn main() {
    let message = String::from("hello from main");

    let handle = thread::spawn(|| {
        println!("  thread: {message}");        // ERROR
    });

    handle.join().expect("thread panicked");
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0373]: closure may outlive the current function, but it borrows `message`, which is owned by the current function
  |
  | thread::spawn(|| {
  |               ^^ may outlive borrowed value `message`
  |     println!("  thread: {message}");
  |                          ------- `message` is borrowed here
  |
help: to force the closure to take ownership of `message` (and any other referenced variables), use the `move` keyword
```

The error explains the problem. The closure borrows `message`, but `message` is local to `main`. The thread might outlive `main`, in which case the borrowed reference would be invalid.

The fix is `move`:

```rust
fn main() {
    let message = String::from("hello from main");

    let handle = thread::spawn(move || {
        println!("  thread: {message}");
    });

    handle.join().expect("thread panicked");
}
```

Run the program. Expected output:

```
  thread: hello from main
```

The `move` keyword forces the closure to take ownership of any captured variables. `message` is moved into the thread; the main function no longer has access to it. The thread owns `message` for its lifetime; the data outlives nothing the thread depends on.

This is the standard pattern: `thread::spawn(move || { ... })` for any thread that needs data from its surroundings.

### Task 1.4 -- Multiple threads

Spawn several threads at once:

```rust
fn main() {
    let mut handles = vec![];

    for i in 0..5 {
        let handle = thread::spawn(move || {
            println!("  thread {i}: starting");
            thread::sleep(Duration::from_millis(100));
            println!("  thread {i}: done");
            i * 10
        });
        handles.push(handle);
    }

    let results: Vec<i32> = handles.into_iter().map(|h| h.join().unwrap()).collect();

    println!("main: results = {results:?}");
}
```

Run the program. Expected output (order of starting and done may vary):

```
  thread 0: starting
  thread 1: starting
  thread 2: starting
  thread 3: starting
  thread 4: starting
  thread 0: done
  thread 1: done
  thread 2: done
  thread 3: done
  thread 4: done
main: results = [0, 10, 20, 30, 40]
```

The `results` vector is deterministic because we collect in the order the handles were created. Each `join` returns the thread's return value in that order.

The startup and completion order is nondeterministic; threads run concurrently, and the OS scheduler decides when each runs. Your output may have different interleaving than the example.

### Task 1.5 -- Threads that panic

If a thread panics, `join` returns `Err`:

```rust
fn main() {
    let handle = thread::spawn(|| {
        panic!("something went wrong");
    });

    match handle.join() {
        Ok(_) => println!("thread completed normally"),
        Err(_) => println!("thread panicked"),
    }
}
```

Run the program. Expected output:

```
thread 'thread::spawn' panicked at ...
thread panicked
```

The panic in the thread does not abort the program; the main thread receives an `Err` from `join` and continues. This is one reason `join` returns `Result`: thread panics are detected, not fatal.

For production code, you usually want to handle panics intentionally (log them, propagate them, restart the thread). The `Err` from `join` is your signal.

### Checkpoints

1. In Task 1.1, the program ended before the spawned thread finished its work, cutting off the count. Why did this happen? What does it tell you about thread lifetimes versus the main program's lifetime?
2. The `move` keyword in Task 1.3 forced the closure to take ownership of `message`. Why is `move` necessary specifically for thread closures? When you pass a closure to `Vec::iter().map(...)`, you typically do not need `move`. What is different about threads?
3. Task 1.5 showed that a thread panic does not crash the program; the main thread sees the panic via `join`. What is the practical benefit of this design compared to languages where any thread's crash takes down the whole process?

---

## Exercise 2 -- Sending Data Between Threads with Channels

**Estimated time:** 15 minutes
**Topics covered:** `mpsc::channel`, `Sender`, `Receiver`, producer-consumer patterns

### Context

A channel is a pipe that lets threads send messages to each other. One side sends; the other side receives. Channels are how Rust threads typically communicate, replacing the shared-memory-with-locks pattern common in other languages.

### Task 2.1 -- A simple channel

Replace `src/main.rs` with:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let message = String::from("hello from sender");
        tx.send(message).expect("receiver dropped");
    });

    let received = rx.recv().expect("sender dropped");
    println!("received: {received}");
}
```

Run the program. Expected output:

```
received: hello from sender
```

The pattern:

- `mpsc::channel()` creates a sender (`tx`) and a receiver (`rx`).
- The sender is moved into the spawned thread (note `move` on the closure).
- The sender calls `send(value)` to push a message.
- The receiver calls `recv()` in the main thread to pull the message.

The name `mpsc` stands for "multiple producer, single consumer": you can clone the sender to have multiple senders, but only one receiver exists.

### Task 2.2 -- Multiple sends

A sender can be used many times:

```rust
fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        for i in 1..=5 {
            tx.send(i).expect("receiver dropped");
            thread::sleep(std::time::Duration::from_millis(100));
        }
    });

    for received in rx {
        println!("received: {received}");
    }

    println!("channel closed");
}
```

Run the program. Expected output:

```
received: 1
received: 2
received: 3
received: 4
received: 5
channel closed
```

Two new patterns:

- The sender sends in a loop, with a small delay between sends.
- The receiver uses `for received in rx`, which iterates over messages until the channel closes.

The channel closes automatically when the sender is dropped (when the spawned thread finishes and its `tx` goes out of scope). The receiver's for-loop ends at that point, and execution continues past it.

This is the standard producer-consumer pattern: one thread produces values, another consumes them. The channel handles synchronization automatically.

### Task 2.3 -- Multiple producers

The "multiple producer" part of mpsc: clone the sender:

```rust
fn main() {
    let (tx, rx) = mpsc::channel();

    let tx_clone = tx.clone();
    thread::spawn(move || {
        for i in 1..=3 {
            tx_clone.send(format!("producer A: {i}")).unwrap();
            thread::sleep(std::time::Duration::from_millis(100));
        }
    });

    thread::spawn(move || {
        for i in 1..=3 {
            tx.send(format!("producer B: {i}")).unwrap();
            thread::sleep(std::time::Duration::from_millis(150));
        }
    });

    for received in rx {
        println!("{received}");
    }

    println!("all senders dropped, channel closed");
}
```

Run the program. Expected output (interleaving may vary):

```
producer A: 1
producer B: 1
producer A: 2
producer A: 3
producer B: 2
producer B: 3
all senders dropped, channel closed
```

Two threads send messages; one thread receives. The channel closes only when all senders are dropped. The receiver sees messages from both producers in whatever order they happen to arrive.

This pattern is useful for fan-in workflows: many sources sending to a single sink (e.g., multiple network connections all logging to one log writer).

### Task 2.4 -- Non-blocking try_recv

`recv()` blocks until a message arrives or the channel closes. Sometimes you want to check without blocking:

```rust
fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        thread::sleep(std::time::Duration::from_millis(500));
        tx.send("delayed message").unwrap();
    });

    for attempt in 1..=10 {
        match rx.try_recv() {
            Ok(message) => {
                println!("got message on attempt {attempt}: {message}");
                break;
            }
            Err(mpsc::TryRecvError::Empty) => {
                println!("attempt {attempt}: nothing yet");
                thread::sleep(std::time::Duration::from_millis(100));
            }
            Err(mpsc::TryRecvError::Disconnected) => {
                println!("channel closed");
                break;
            }
        }
    }
}
```

Run the program. Expected output:

```
attempt 1: nothing yet
attempt 2: nothing yet
attempt 3: nothing yet
attempt 4: nothing yet
attempt 5: nothing yet
got message on attempt 6: delayed message
```

`try_recv()` returns immediately. If a message is available, you get it. If not, you get `Err(Empty)` and can do something else. If the channel is closed, `Err(Disconnected)`.

This is useful for polling patterns where you want to check a channel as part of a larger loop (e.g., game loops, event-driven code).

### Task 2.5 -- Bounded channels with sync_channel

The default `mpsc::channel` is unbounded: senders can buffer arbitrarily many messages. `mpsc::sync_channel` has a fixed buffer:

```rust
fn main() {
    let (tx, rx) = mpsc::sync_channel(3);        // buffer of 3 messages

    thread::spawn(move || {
        for i in 1..=10 {
            println!("sender: sending {i}");
            tx.send(i).unwrap();
            println!("sender: sent {i}");
        }
        println!("sender: done");
    });

    thread::sleep(std::time::Duration::from_millis(500));

    println!("receiver: starting");
    for received in rx {
        println!("receiver: got {received}");
        thread::sleep(std::time::Duration::from_millis(200));
    }
}
```

Run the program. The output shows the bounded behavior:

```
sender: sending 1
sender: sent 1
sender: sending 2
sender: sent 2
sender: sending 3
sender: sent 3
sender: sending 4
            (sender blocks here, waiting for buffer space)
receiver: starting
receiver: got 1
sender: sent 4
sender: sending 5
            (continues...)
```

The first 3 sends succeed immediately because the buffer has room. Send 4 blocks until the receiver pulls one message, freeing space. The buffer of 3 limits how far ahead the sender can run.

Bounded channels are useful for backpressure: if the consumer is slow, the producer is throttled rather than building up an unbounded backlog. This prevents memory blowup.

### Checkpoints

1. The channel pattern (Task 2.1) is described as "replacing shared memory with locks" for thread communication. Why is message passing often preferable to shared state with locks?
2. The channel in Task 2.2 closes automatically when the sender is dropped. What would happen if the sender were never dropped (e.g., stored in a long-lived variable)? Why is the automatic close behavior useful?
3. The bounded channel in Task 2.5 caused the sender to block when the buffer was full. This is a form of backpressure. What problem does backpressure prevent? When is unbounded buffering preferable?

---

## Exercise 3 -- Shared State with Mutex

**Estimated time:** 15 minutes
**Topics covered:** `Mutex`, `Arc`, sharing state across threads, `MutexGuard`

### Context

Sometimes data does not fit a channel-based design: multiple threads need to read and write the same value. For this, Rust provides `Mutex` (mutual exclusion). Combined with `Arc` (atomic reference counting), it gives you safe shared state.

### Task 3.1 -- Mutex basics

Replace `src/main.rs` with:

```rust
use std::sync::Mutex;

fn main() {
    let counter = Mutex::new(0);

    {
        let mut value = counter.lock().unwrap();
        *value += 1;
        *value += 1;
        println!("inside lock: {}", *value);
    }

    println!("outside lock: {:?}", counter);
}
```

Run the program. Expected output:

```
inside lock: 2
outside lock: Mutex { data: 2, poisoned: false, .. }
```

`Mutex::new(value)` wraps a value in a mutex. To access the inner value, you call `lock()`. This returns a `MutexGuard`, a smart pointer that:

- Provides access to the inner value via deref (`*value`).
- Releases the lock when dropped (at the end of the scope).

The `{ ... }` block scopes the guard. When the block ends, the guard is dropped, releasing the lock. The outside `println!` shows the mutex with its current value.

`.lock().unwrap()` is the standard pattern. The `lock()` returns `Result` to handle the rare case of "poisoning" (a thread panicked while holding the lock). For most code, `unwrap()` is acceptable.

### Task 3.2 -- Sharing a Mutex across threads

To share a `Mutex` across threads, wrap it in an `Arc`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

Run the program. Expected output:

```
Result: 10
```

Three things to notice:

- The mutex is wrapped in an `Arc`. `Mutex` alone cannot be shared; you need `Arc` for the multiple-owner pattern.
- `Arc::clone(&counter)` produces a new `Arc` pointing to the same `Mutex`. The clone is cheap (just increments a reference count).
- Each thread locks the mutex, increments, and releases (when the guard goes out of scope at the end of the closure).

The result is deterministically 10 (ten threads each adding 1) regardless of the order in which they execute. The mutex serializes the increments; no two threads modify the value at the same time.

### Task 3.3 -- A counter with explicit unlock

Try this slightly more elaborate example:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![]));
    let mut handles = vec![];

    for i in 0..5 {
        let data = Arc::clone(&data);
        let handle = thread::spawn(move || {
            {
                let mut list = data.lock().unwrap();
                list.push(i * 10);
                println!("thread {i} added {}, list now has {} items", i * 10, list.len());
            }   // lock released here
            std::thread::sleep(std::time::Duration::from_millis(50));
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let final_list = data.lock().unwrap();
    println!("final list: {:?}", *final_list);
}
```

Run the program. Expected output (interleaving may vary):

```
thread 0 added 0, list now has 1 items
thread 1 added 10, list now has 2 items
thread 2 added 20, list now has 3 items
thread 3 added 30, list now has 4 items
thread 4 added 40, list now has 5 items
final list: [0, 10, 20, 30, 40]
```

The order may vary, but the final list always contains exactly the five values (in whatever order they were added).

The inner `{ ... }` block scopes the lock. Releasing the lock before the sleep means other threads can acquire it. If the lock were held during the sleep, threads would serialize on the sleep too, which would be wasteful.

The general principle: hold locks for as short a time as possible. Acquire, do the work that needs the lock, release. Other work happens outside the lock.

### Task 3.4 -- A lock that does too much

For comparison, try a version that holds the lock too long:

```rust
fn main() {
    let data = Arc::new(Mutex::new(vec![]));
    let mut handles = vec![];

    let start = std::time::Instant::now();

    for i in 0..5 {
        let data = Arc::clone(&data);
        let handle = thread::spawn(move || {
            let mut list = data.lock().unwrap();
            list.push(i * 10);
            std::thread::sleep(std::time::Duration::from_millis(100));    // sleep with lock held
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let elapsed = start.elapsed();
    println!("elapsed: {elapsed:?}");
    println!("final list: {:?}", *data.lock().unwrap());
}
```

Run the program. The elapsed time will be roughly 500ms (5 threads × 100ms each, serialized) instead of about 100ms (all running in parallel).

```
elapsed: 504ms
final list: [0, 10, 20, 30, 40]
```

The lock held during sleep forces the threads to run one at a time. They are not really concurrent; they are queued.

This is a real performance pitfall. Holding a lock during slow operations (I/O, computation, sleeps) destroys concurrency benefits. The previous version (Task 3.3) was much faster because the lock was released immediately.

### Task 3.5 -- RwLock for many readers

When the workload is "many threads read, few write," `RwLock` is more efficient than `Mutex`. It allows multiple simultaneous readers but only one writer:

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3, 4, 5]));
    let mut handles = vec![];

    // Spawn 5 reader threads:
    for i in 0..5 {
        let data = Arc::clone(&data);
        let handle = thread::spawn(move || {
            let reader = data.read().unwrap();
            println!("reader {i}: sum = {}", reader.iter().sum::<i32>());
        });
        handles.push(handle);
    }

    // Spawn 1 writer thread:
    let writer_data = Arc::clone(&data);
    let writer_handle = thread::spawn(move || {
        let mut writer = writer_data.write().unwrap();
        writer.push(100);
        println!("writer: added 100");
    });
    handles.push(writer_handle);

    for handle in handles {
        handle.join().unwrap();
    }

    println!("final: {:?}", *data.read().unwrap());
}
```

Run the program. Expected output (order may vary):

```
reader 0: sum = 15
reader 1: sum = 15
reader 2: sum = 15
writer: added 100
reader 3: sum = 115
reader 4: sum = 115
final: [1, 2, 3, 4, 5, 100]
```

`read()` returns a read guard; multiple read guards can exist simultaneously. `write()` returns a write guard; only one write guard can exist, and no read guards can coexist with it.

This is more efficient than `Mutex` when reads vastly outnumber writes. For workloads where every access modifies the data, `Mutex` is simpler and roughly as fast.

### Checkpoints

1. The mutex in Task 3.2 is wrapped in `Arc` before being shared across threads. Why is `Arc` needed? What goes wrong if you try to share a `Mutex` without it?
2. Task 3.4 showed a 5x performance regression from holding a lock during a sleep. Why is "hold the lock briefly" such an important rule? What other operations should typically be done outside the lock?
3. `RwLock` (Task 3.5) allows multiple simultaneous readers but only one writer. What kind of workload benefits from this versus a plain `Mutex`? What kind of workload would not benefit?

---

## Exercise 4 -- Send and Sync

**Estimated time:** 10 minutes
**Topics covered:** the marker traits, what types implement them, common compile errors

### Context

`Send` and `Sync` are marker traits that govern what can move between threads. Most types implement them automatically; understanding them helps you read compile errors when something does not.

### Task 4.1 -- Send: most types

Most types are `Send`, meaning they can be transferred to another thread:

```rust
use std::thread;

fn main() {
    let v: Vec<i32> = vec![1, 2, 3];
    let s: String = String::from("hello");
    let n: i32 = 42;

    let handle = thread::spawn(move || {
        println!("v: {v:?}");
        println!("s: {s}");
        println!("n: {n}");
    });

    handle.join().unwrap();
}
```

Run the program. Expected output:

```
v: [1, 2, 3]
s: hello
n: 42
```

`Vec`, `String`, and primitives are all `Send`. The `move` closure transfers ownership of them to the thread. The compiler verifies they are `Send` and accepts the code.

### Task 4.2 -- Rc is not Send

`Rc<T>` (single-threaded reference counting, from Module 15) is deliberately not `Send`:

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(42);

    let handle = thread::spawn(move || {
        println!("data: {data}");        // ERROR
    });

    handle.join().unwrap();
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0277]: `Rc<i32>` cannot be sent between threads safely
  |
  | thread::spawn(move || {
  |               ^^^^^^^ `Rc<i32>` cannot be sent between threads safely
  |
  = help: within `[closure]`, the trait `Send` is not implemented for `Rc<i32>`
```

The error explains why. `Rc` uses non-atomic reference counting; updating its count from multiple threads concurrently would cause data races. To prevent this, `Rc` is marked `!Send` (not Send), and the compiler refuses to move it across thread boundaries.

The fix is to use `Arc` instead:

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(42);

    let handle = thread::spawn(move || {
        println!("data: {data}");
    });

    handle.join().unwrap();
}
```

`Arc` uses atomic reference counting, which is safe to use across threads. It is `Send`.

This is one of the most important applications of the trait system: catching a class of concurrency bugs at compile time.

### Task 4.3 -- Sync and references

`Sync` says a type can be safely accessed from multiple threads simultaneously (via shared references). Most types are `Sync` automatically. A type is `Sync` if `&T` is `Send`.

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3, 4, 5]);
    let mut handles = vec![];

    for i in 0..3 {
        let data = Arc::clone(&data);
        let handle = thread::spawn(move || {
            let sum: i32 = data.iter().sum();
            println!("thread {i}: sum = {sum}");
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

Run the program. Expected output:

```
thread 0: sum = 15
thread 1: sum = 15
thread 2: sum = 15
```

The `Arc<Vec<i32>>` is shared across threads. Each thread reads the vector via the Arc. This works because `Vec<i32>` is `Sync` (it is safe to read concurrently from multiple threads).

If the threads wanted to *modify* the vector, you would need `Arc<Mutex<Vec<i32>>>` for synchronized mutation. Reading shared data needs only `Sync`; modifying needs a synchronization primitive.

### Task 4.4 -- Cell is not Sync

`Cell<T>` (from Module 15) provides interior mutability without synchronization. It is `!Sync`:

```rust
use std::cell::Cell;
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(Cell::new(0));

    let data_clone = Arc::clone(&data);
    let handle = thread::spawn(move || {
        data_clone.set(42);              // ERROR
    });

    handle.join().unwrap();
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0277]: `Cell<i32>` cannot be shared between threads safely
```

`Cell` is not `Sync` because its mutation is not synchronized. If two threads both called `set(...)` at the same time, the result would be a data race.

For shared mutation across threads, use `Arc<Mutex<T>>` or atomic types from `std::sync::atomic`.

### Checkpoints

1. The error in Task 4.2 said "Rc cannot be sent between threads safely." The compiler caught this at compile time. What kind of bug does this prevent? In a language without this check, when would the bug appear?
2. `Send` and `Sync` are usually auto-derived. Types are `Send` if all their fields are `Send`; `Sync` if all their fields are `Sync`. Why does this composition rule make sense?
3. The pattern `Arc<Mutex<T>>` combines two types: `Arc` for sharing, `Mutex` for synchronized mutation. Why are these two separate types rather than one combined "thread-safe value" type?

---

## Exercise 5 -- Async/Await with Tokio

**Estimated time:** 15 minutes
**Topics covered:** the `async` keyword, `.await`, the Tokio runtime, async functions

### Context

Async is a different model from threads. Instead of one OS thread per concurrent operation, async lets many operations share a small number of threads cooperatively. This is dramatically more efficient for I/O-bound workloads. This exercise introduces the async model using Tokio, the most common Rust async runtime.

### Task 5.1 -- Adding Tokio as a dependency

Open `Cargo.toml` and add:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

The `features = ["full"]` enables all Tokio features. For production code, you would enable only what you need; for learning, full is convenient.

Save the file. The next build will download Tokio.

### Task 5.2 -- A basic async function

Replace `src/main.rs` with:

```rust
use tokio::time::{sleep, Duration};

async fn say_hello() {
    sleep(Duration::from_millis(100)).await;
    println!("hello");
}

#[tokio::main]
async fn main() {
    say_hello().await;
    println!("done");
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
hello
done
```

Three new features:

- `async fn` declares an asynchronous function. It returns a `Future` rather than running immediately.
- `.await` runs an async function and waits for it to complete, yielding control to other tasks while waiting.
- `#[tokio::main]` is an attribute macro that transforms `main` into a function that starts the Tokio runtime and runs the async body on it.

The `sleep` is async: while it waits, other tasks can run. This is the central feature of async: efficient waiting that does not block a thread.

### Task 5.3 -- Sequential vs concurrent awaits

Without concurrency, awaits run sequentially:

```rust
use tokio::time::{sleep, Duration, Instant};

async fn task(name: &str, delay_ms: u64) {
    println!("starting {name}");
    sleep(Duration::from_millis(delay_ms)).await;
    println!("done {name}");
}

#[tokio::main]
async fn main() {
    let start = Instant::now();

    task("A", 100).await;
    task("B", 100).await;
    task("C", 100).await;

    println!("elapsed: {:?}", start.elapsed());
}
```

Run the program. Expected output:

```
starting A
done A
starting B
done B
starting C
done C
elapsed: 305ms
```

Each task runs to completion before the next starts. Total time is about 300ms (3 × 100ms). This is just synchronous code with extra syntax.

To run them concurrently, use `tokio::join!`:

```rust
#[tokio::main]
async fn main() {
    let start = Instant::now();

    tokio::join!(
        task("A", 100),
        task("B", 100),
        task("C", 100),
    );

    println!("elapsed: {:?}", start.elapsed());
}
```

Run the program. Expected output:

```
starting A
starting B
starting C
done A
done B
done C
elapsed: 102ms
```

All three tasks run concurrently. Total time is about 100ms (the longest single task). The `tokio::join!` macro awaits multiple futures concurrently, returning their results when all complete.

The output order may vary; the tasks run on the same thread but interleave at await points.

### Task 5.4 -- Spawning async tasks

For independent tasks that should run in the background, use `tokio::spawn`:

```rust
use tokio::time::{sleep, Duration};

async fn worker(id: u32) {
    println!("worker {id}: starting");
    sleep(Duration::from_millis(100)).await;
    println!("worker {id}: done");
}

#[tokio::main]
async fn main() {
    let mut handles = vec![];

    for i in 0..5 {
        let handle = tokio::spawn(worker(i));
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }

    println!("all workers done");
}
```

Run the program. Expected output (interleaving may vary):

```
worker 0: starting
worker 1: starting
worker 2: starting
worker 3: starting
worker 4: starting
worker 0: done
worker 1: done
worker 2: done
worker 3: done
worker 4: done
all workers done
```

`tokio::spawn(future)` runs the future on the Tokio runtime, returning a `JoinHandle` (similar to `thread::JoinHandle`). The handle is itself a future; awaiting it waits for the spawned task to complete.

This is parallel to the thread pattern from Exercise 1, but with key differences:

- Tasks are much cheaper than threads (kilobytes vs megabytes of memory).
- Many thousands of tasks can run on a small number of OS threads.
- Tasks share their threads cooperatively, yielding at await points.

For I/O-bound workloads (network requests, file I/O), async tasks are dramatically more efficient than threads.

### Task 5.5 -- Returning values from spawned tasks

Spawned tasks can return values:

```rust
async fn compute(n: u32) -> u32 {
    sleep(Duration::from_millis(100)).await;
    n * n
}

#[tokio::main]
async fn main() {
    let mut handles = vec![];

    for i in 1..=5 {
        let handle = tokio::spawn(compute(i));
        handles.push(handle);
    }

    let mut total = 0;
    for handle in handles {
        let result = handle.await.unwrap();
        total += result;
    }

    println!("total: {total}");
}
```

Run the program. Expected output:

```
total: 55
```

`compute(1) + compute(2) + compute(3) + compute(4) + compute(5)` = 1 + 4 + 9 + 16 + 25 = 55. The squares are computed concurrently (total elapsed time is about 100ms, not 500ms). The order of completion may vary, but the sum is deterministic.

### Checkpoints

1. The `async fn` keyword changes a function's return type: instead of returning the value directly, it returns a `Future` of the value. Why is this necessary? What does the change enable?
2. `tokio::join!` runs multiple futures concurrently on the same thread. They share execution by yielding at await points. How is this different from running the same operations as threads?
3. For an I/O-heavy workload (e.g., making 1000 HTTP requests), would you prefer 1000 threads or 1000 async tasks? Why? What kind of workload would prefer threads instead?

---

## Exercise 6 -- Async Coordination

**Estimated time:** 15 minutes
**Topics covered:** `tokio::select!`, async channels, async mutex, timeouts

### Context

Async code uses different versions of the primitives you have already seen. Channels are async; mutexes are async. The `tokio::select!` macro is the async equivalent of "do whichever happens first." This exercise introduces these patterns.

### Task 6.1 -- Async channels

Tokio provides async channels. They are similar to `std::sync::mpsc::channel` but their methods return futures:

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(10);

    tokio::spawn(async move {
        for i in 1..=5 {
            tx.send(i).await.unwrap();
            sleep(Duration::from_millis(100)).await;
        }
    });

    while let Some(message) = rx.recv().await {
        println!("received: {message}");
    }

    println!("channel closed");
}
```

Run the program. Expected output:

```
received: 1
received: 2
received: 3
received: 4
received: 5
channel closed
```

Notable points:

- `mpsc::channel(10)` creates a bounded channel with capacity 10. Tokio's channels are always bounded; the size is a deliberate design choice.
- `tx.send(value).await` is async: it returns a future that resolves when the message is sent (or fails if the receiver is dropped).
- `rx.recv().await` is async: it waits for a message or returns `None` when the channel closes.

For tasks coordinating within an async system, Tokio channels are preferred over the standard library's `mpsc` channels because they integrate with the async runtime.

### Task 6.2 -- `tokio::select!` for racing futures

`tokio::select!` waits for the first of several futures to complete:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let task1 = async {
        sleep(Duration::from_millis(100)).await;
        "task 1"
    };

    let task2 = async {
        sleep(Duration::from_millis(200)).await;
        "task 2"
    };

    let winner = tokio::select! {
        result = task1 => result,
        result = task2 => result,
    };

    println!("winner: {winner}");
}
```

Run the program. Expected output:

```
winner: task 1
```

Task 1 finishes first (100ms vs 200ms), so it wins the race. The `select!` returns the value from whichever arm completes first; the other future is cancelled.

This is useful for timeouts:

```rust
#[tokio::main]
async fn main() {
    let slow_task = async {
        sleep(Duration::from_millis(500)).await;
        "completed"
    };

    let timeout = async {
        sleep(Duration::from_millis(200)).await;
        "timed out"
    };

    let result = tokio::select! {
        v = slow_task => v,
        v = timeout => v,
    };

    println!("result: {result}");
}
```

Run the program. Expected output:

```
result: timed out
```

The timeout future finished first. The slow task was cancelled (it never completes; the future is dropped).

### Task 6.3 -- The timeout shortcut

For the common timeout pattern, Tokio provides a helper:

```rust
use tokio::time::{sleep, timeout, Duration};

async fn slow_operation() -> &'static str {
    sleep(Duration::from_millis(500)).await;
    "completed"
}

#[tokio::main]
async fn main() {
    let result = timeout(Duration::from_millis(200), slow_operation()).await;

    match result {
        Ok(value) => println!("got value: {value}"),
        Err(_) => println!("operation timed out"),
    }

    let result2 = timeout(Duration::from_millis(1000), slow_operation()).await;
    match result2 {
        Ok(value) => println!("got value: {value}"),
        Err(_) => println!("timed out"),
    }
}
```

Run the program. Expected output:

```
operation timed out
got value: completed
```

`timeout(duration, future)` runs the future with a time limit. If it completes within the limit, `Ok(value)`. If not, `Err(Elapsed)`.

This is the standard pattern for adding timeouts to operations. The underlying mechanism is `select!`; `timeout` is the convenience wrapper.

### Task 6.4 -- Async mutex

The standard library's `Mutex` is fine for short-held locks across threads. For locks that must be held across await points, Tokio provides `tokio::sync::Mutex`:

```rust
use std::sync::Arc;
use tokio::sync::Mutex;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for i in 0..5 {
        let counter = Arc::clone(&counter);
        let handle = tokio::spawn(async move {
            let mut value = counter.lock().await;
            *value += 1;
            sleep(Duration::from_millis(100)).await;
            *value += 10;
            println!("task {i}: counter is now {}", *value);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }

    let final_value = counter.lock().await;
    println!("final: {}", *final_value);
}
```

Run the program. Expected output:

```
task 0: counter is now 11
task 1: counter is now 22
task 2: counter is now 33
task 3: counter is now 44
task 4: counter is now 55
final: 55
```

The async mutex's `lock()` returns a future. The await yields control while waiting for the lock. The lock can be held across other await points (like the `sleep`); the standard `Mutex` would deadlock or perform poorly in that pattern.

For most async code, the standard `Mutex` is still preferred when the lock is held briefly and not across awaits. Use `tokio::sync::Mutex` when the lock must span await points.

### Checkpoints

1. The standard library has channels (`std::sync::mpsc`); Tokio has its own (`tokio::sync::mpsc`). Why does Tokio provide its own rather than reusing the standard one?
2. `tokio::select!` cancels futures that did not complete. What happens to side effects (changes to state, partial work) when a future is cancelled mid-execution? When does this matter?
3. Task 6.4 uses `tokio::sync::Mutex` because the lock is held across an `.await`. Why does the standard `std::sync::Mutex` not work well in this case? What specifically goes wrong if you use it?

---

## Exercise 7 -- Async Networking (Tokio HTTP Client)

**Estimated time:** 10 minutes
**Topics covered:** real async I/O, concurrent HTTP requests, the `reqwest` crate

### Context

The motivating use case for async is I/O-bound workloads. This exercise uses an HTTP client to demonstrate why async matters: making concurrent network requests is dramatically more efficient than doing them sequentially.

> **Note:** This exercise makes real HTTP requests to public APIs. Ensure you have an internet connection. If the API is unavailable, the exercise will fail with network errors; you can mock the URLs or skip to the next exercise.

### Task 7.1 -- Add reqwest as a dependency

Update `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
```

Save the file. The next build will download reqwest.

### Task 7.2 -- A single request

Replace `src/main.rs` with:

```rust
use tokio::time::Instant;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let start = Instant::now();

    let response = reqwest::get("https://httpbin.org/get").await?;
    let status = response.status();
    let body = response.text().await?;

    println!("status: {status}");
    println!("body length: {} bytes", body.len());
    println!("elapsed: {:?}", start.elapsed());

    Ok(())
}
```

Run the program. Expected output (numbers will vary):

```
status: 200 OK
body length: 397 bytes
elapsed: 245ms
```

One HTTP request, about a quarter second. Note that `main` returns `Result`; the `?` operator works for clean error propagation.

### Task 7.3 -- Sequential requests

Make several requests one at a time:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let urls = vec![
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
    ];

    let start = Instant::now();

    for url in &urls {
        let response = reqwest::get(*url).await?;
        let status = response.status();
        println!("{url}: {status}");
    }

    println!("elapsed: {:?}", start.elapsed());

    Ok(())
}
```

Run the program. The `httpbin.org/delay/1` endpoint deliberately delays 1 second. Five sequential requests take about 5 seconds:

```
https://httpbin.org/delay/1: 200 OK
https://httpbin.org/delay/1: 200 OK
https://httpbin.org/delay/1: 200 OK
https://httpbin.org/delay/1: 200 OK
https://httpbin.org/delay/1: 200 OK
elapsed: 5.12s
```

Each request waits for the previous one to finish before starting. Most of the time, the program is sitting idle waiting for the server.

### Task 7.4 -- Concurrent requests

Now the same requests in parallel:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let urls = vec![
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
    ];

    let start = Instant::now();

    let mut handles = vec![];
    for url in urls {
        let handle = tokio::spawn(async move {
            reqwest::get(url).await.map(|r| r.status())
        });
        handles.push(handle);
    }

    for handle in handles {
        match handle.await.unwrap() {
            Ok(status) => println!("got: {status}"),
            Err(e) => println!("error: {e}"),
        }
    }

    println!("elapsed: {:?}", start.elapsed());

    Ok(())
}
```

Run the program. Five concurrent requests take about 1 second (the longest single request):

```
got: 200 OK
got: 200 OK
got: 200 OK
got: 200 OK
got: 200 OK
elapsed: 1.05s
```

5x speedup from concurrency. The program spends most of its time waiting for network responses; doing all five waits in parallel means the total wait time is the longest single wait, not the sum.

This is the value proposition of async for I/O-bound workloads. The same speedup applies to database queries, file reads, and any other operations that mostly wait.

### Task 7.5 -- Using join_all for cleaner syntax

For the common "wait for all" pattern, `futures::future::join_all` is cleaner:

```rust
use futures::future::join_all;
```

Add this to `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
futures = "0.3"
```

Then:

```rust
use futures::future::join_all;
use tokio::time::Instant;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let urls = vec![
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/1",
    ];

    let start = Instant::now();

    let futures = urls.iter().map(|url| reqwest::get(*url));
    let results = join_all(futures).await;

    for result in results {
        match result {
            Ok(response) => println!("got: {}", response.status()),
            Err(e) => println!("error: {e}"),
        }
    }

    println!("elapsed: {:?}", start.elapsed());

    Ok(())
}
```

Run the program. Same 5x speedup, but with cleaner syntax. `join_all` takes an iterator of futures and returns a future that completes when all of them have completed.

This is the standard pattern for "run all these operations concurrently and collect results."

### Checkpoints

1. The concurrent version of Task 7.4 was 5x faster than the sequential version. Why? Where did the time savings come from?
2. For a CPU-bound workload (e.g., computing a million SHA hashes), running the same code concurrently with async would not provide a 5x speedup. Why? What would?
3. The HTTP requests in this exercise were inherently slow (1 second each due to the server). For fast requests (e.g., 10ms each), would async still provide a substantial speedup? What is the break-even point?

---

## Exercise 8 -- Putting It Together

**Estimated time:** 15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from Module 14 into a single program: an async task processor that takes a queue of work items, processes them concurrently with multiple workers, and aggregates the results. The program demonstrates the patterns students will use in real concurrent code.

### Task 8.1 -- The full program

Replace your `src/main.rs` with this complete program (and ensure `Cargo.toml` has `tokio = { version = "1", features = ["full"] }`):

```rust
use std::sync::Arc;
use tokio::sync::{mpsc, Mutex};
use tokio::time::{sleep, Duration, Instant};

// ============================================================
// Domain types
// ============================================================

#[derive(Debug, Clone)]
struct Task {
    id: u32,
    description: String,
    work_ms: u64,
}

#[derive(Debug)]
struct TaskResult {
    task_id: u32,
    worker_id: u32,
    elapsed_ms: u64,
    output: String,
}

// ============================================================
// Worker
// ============================================================

async fn process_task(task: Task, worker_id: u32) -> TaskResult {
    let start = Instant::now();
    println!("worker {worker_id}: starting task {} ({})", task.id, task.description);

    // Simulate work:
    sleep(Duration::from_millis(task.work_ms)).await;

    let result = TaskResult {
        task_id: task.id,
        worker_id,
        elapsed_ms: start.elapsed().as_millis() as u64,
        output: format!("processed: {}", task.description),
    };

    println!("worker {worker_id}: finished task {} in {}ms", task.id, result.elapsed_ms);
    result
}

// ============================================================
// Worker loop: pulls tasks from the channel until closed
// ============================================================

async fn worker_loop(
    worker_id: u32,
    receiver: Arc<Mutex<mpsc::Receiver<Task>>>,
    completed: Arc<Mutex<Vec<TaskResult>>>,
) {
    println!("worker {worker_id}: started");

    loop {
        // Pull next task (need to hold the lock briefly to recv):
        let task = {
            let mut rx = receiver.lock().await;
            rx.recv().await
        };

        match task {
            Some(task) => {
                let result = process_task(task, worker_id).await;
                let mut completed_list = completed.lock().await;
                completed_list.push(result);
            }
            None => {
                // Channel closed; no more tasks
                println!("worker {worker_id}: no more tasks, stopping");
                break;
            }
        }
    }
}

// ============================================================
// Main: orchestrate the workers
// ============================================================

#[tokio::main]
async fn main() {
    let start = Instant::now();

    // Create the task queue:
    let (tx, rx) = mpsc::channel::<Task>(100);
    let rx = Arc::new(Mutex::new(rx));

    // Shared list of completed results:
    let completed = Arc::new(Mutex::new(Vec::<TaskResult>::new()));

    // Spawn 3 workers:
    let mut worker_handles = vec![];
    for worker_id in 1..=3 {
        let rx = Arc::clone(&rx);
        let completed = Arc::clone(&completed);
        let handle = tokio::spawn(worker_loop(worker_id, rx, completed));
        worker_handles.push(handle);
    }

    // Spawn the task producer:
    let producer_handle = tokio::spawn(async move {
        let tasks = vec![
            Task { id: 1, description: String::from("fetch user data"), work_ms: 200 },
            Task { id: 2, description: String::from("validate input"), work_ms: 100 },
            Task { id: 3, description: String::from("compute statistics"), work_ms: 300 },
            Task { id: 4, description: String::from("send notification"), work_ms: 150 },
            Task { id: 5, description: String::from("update database"), work_ms: 250 },
            Task { id: 6, description: String::from("write log entry"), work_ms: 50 },
            Task { id: 7, description: String::from("generate report"), work_ms: 400 },
            Task { id: 8, description: String::from("cleanup temp files"), work_ms: 100 },
        ];

        for task in tasks {
            println!("producer: enqueueing task {}", task.id);
            tx.send(task).await.unwrap();
        }

        println!("producer: all tasks sent, closing channel");
        // tx is dropped here, closing the channel
    });

    // Wait for producer to finish:
    producer_handle.await.unwrap();

    // Wait for all workers to finish (they will exit when the channel closes):
    for handle in worker_handles {
        handle.await.unwrap();
    }

    let elapsed = start.elapsed();

    // Print summary:
    let completed_list = completed.lock().await;

    println!("\n=== Summary ===");
    println!("Total elapsed: {elapsed:?}");
    println!("Tasks completed: {}", completed_list.len());

    let total_work: u64 = completed_list.iter().map(|r| r.elapsed_ms).sum();
    println!("Total work performed: {total_work}ms (across all workers)");
    println!("Speedup ratio: {:.2}x", total_work as f64 / elapsed.as_millis() as f64);

    println!("\nResults by worker:");
    for worker_id in 1..=3 {
        let by_worker: Vec<&TaskResult> = completed_list.iter()
            .filter(|r| r.worker_id == worker_id)
            .collect();
        println!("  worker {worker_id}: {} tasks", by_worker.len());
        for result in by_worker {
            println!("    task {}: {}ms ({})", result.task_id, result.elapsed_ms, result.output);
        }
    }
}
```

Run the program:

```bash
cargo run
```

Expected output (interleaving and exact timings will vary, but the structure is consistent):

```
worker 1: started
worker 2: started
worker 3: started
producer: enqueueing task 1
producer: enqueueing task 2
producer: enqueueing task 3
worker 1: starting task 1 (fetch user data)
worker 2: starting task 2 (validate input)
worker 3: starting task 3 (compute statistics)
producer: enqueueing task 4
producer: enqueueing task 5
producer: enqueueing task 6
producer: enqueueing task 7
producer: enqueueing task 8
producer: all tasks sent, closing channel
worker 2: finished task 2 in 100ms
worker 2: starting task 4 (send notification)
worker 1: finished task 1 in 200ms
worker 1: starting task 5 (update database)
worker 2: finished task 4 in 150ms
worker 2: starting task 6 (write log entry)
worker 2: finished task 6 in 50ms
worker 2: starting task 7 (generate report)
worker 3: finished task 3 in 300ms
worker 3: starting task 8 (cleanup temp files)
worker 1: finished task 5 in 250ms
worker 1: no more tasks, stopping
worker 3: finished task 8 in 100ms
worker 3: no more tasks, stopping
worker 2: finished task 7 in 400ms
worker 2: no more tasks, stopping

=== Summary ===
Total elapsed: 612ms
Tasks completed: 8
Total work performed: 1550ms (across all workers)
Speedup ratio: 2.53x

Results by worker:
  worker 1: 2 tasks
    task 1: 200ms (processed: fetch user data)
    task 5: 250ms (processed: update database)
  worker 2: 4 tasks
    task 2: 100ms (processed: validate input)
    task 4: 150ms (processed: send notification)
    task 6: 50ms (processed: write log entry)
    task 7: 400ms (processed: generate report)
  worker 3: 2 tasks
    task 3: 300ms (processed: compute statistics)
    task 8: 100ms (processed: cleanup temp files)
```

The speedup ratio (~2.5x in this run) shows the value of concurrency: total elapsed time is much less than the sum of work performed because three workers operated in parallel.

### Task 8.2 -- Identify the patterns

In `lab10-notes.md`, identify each instance of the following patterns in the program above:

1. An async function declared with `async fn`.
2. A use of `.await` to wait for an async operation.
3. The `#[tokio::main]` attribute macro that sets up the runtime.
4. A use of `tokio::spawn` to run an async task in the background.
5. An async channel created with `mpsc::channel`.
6. An async mutex (`tokio::sync::Mutex`).
7. `Arc` used to share state across multiple async tasks.
8. A pattern where a producer task closes the channel by dropping the sender, signaling workers to stop.
9. The "hold the lock briefly" pattern using a scoped block.
10. A use of `match` on the `Option` returned by `recv()` to distinguish "got task" from "channel closed."

### Task 8.3 -- Predict and verify

For each of the following changes, predict what will happen and verify. Write your predictions in `lab10-notes.md` BEFORE running.

**Change A:** Change the number of workers from 3 to 1. What changes? How does total elapsed time change?

**Change B:** Add a `sleep(Duration::from_millis(50)).await` inside the `match Some(task)` arm of `worker_loop`, BEFORE acquiring the completed list lock. What happens? Does anything break?

**Change C:** Remove the inner block around the `recv()` call (so the receiver lock is held during the call to `process_task`). What happens? How does the program's behavior change?

For each change, write your prediction first, then run and verify.

### Checkpoints

1. The program uses `tokio::sync::Mutex` rather than `std::sync::Mutex`. Why is this the right choice here? When would using the standard library's `Mutex` cause problems?
2. The workers stop when the channel closes (when the producer drops the sender). What is the advantage of this design compared to other ways of signaling "no more work"?
3. The shared `completed` list is a `Arc<Mutex<Vec<TaskResult>>>`. Could it have been a channel instead? What would change if results were sent through a channel rather than pushed to a shared list?

---

## Summary and Reflection

You have now used every concept from Module 14 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Thread Spawning | thread::spawn, join, move | Threads run concurrently; `move` transfers ownership; `join` waits for completion. |
| 2 -- Channels | mpsc::channel, send/recv, bounded vs unbounded | Channels are the preferred way for threads to communicate; they replace shared-memory-with-locks for many cases. |
| 3 -- Mutex and RwLock | Arc<Mutex<T>>, holding locks briefly | Shared mutable state requires synchronization; hold locks briefly to maintain concurrency benefits. |
| 4 -- Send and Sync | marker traits, why Rc is not Send | The compiler enforces thread safety at compile time using these marker traits. |
| 5 -- Async/Await Basics | async fn, .await, tokio::main | Async is a different model: many tasks share few threads, yielding at await points. |
| 6 -- Async Coordination | select, timeout, async channels and mutex | Async has its own versions of the synchronization primitives, designed for the async runtime. |
| 7 -- Async I/O | reqwest, concurrent HTTP requests | Async shines for I/O-bound workloads: 5x speedup is common when operations are wait-dominated. |
| 8 -- Integration | all of the above | Real concurrent code combines workers, channels, shared state, and async coordination. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab10-notes.md` before your next session.

1. Of the concurrency concepts in Module 14, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. Rust offers two concurrency models: threads (with channels, mutexes, etc.) and async (with Tokio). Identify a specific kind of workload where each is the right choice. What characteristics of the workload led you to that choice?

3. The borrow checker prevents many concurrency bugs at compile time (data races, sending non-Send types across threads, etc.). Identify one specific bug that Rust prevents that you have either experienced or read about in another language. How does Rust's prevention work?

---

*End of Lab 10*
