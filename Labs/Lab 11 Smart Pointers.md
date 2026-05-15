# Module 15: Smart Pointers and Interior Mutability
## Lab 11 -- Working with Box, Rc, RefCell, Weak, and the Smart Pointer Traits

> **Course:** Mastering Rust
> **Module:** 15 - Smart Pointers and Interior Mutability
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 15: heap allocation with `Box`, reference counting with `Rc`, interior mutability with `RefCell` and `Cell`, breaking reference cycles with `Weak`, and the traits (`Deref` and `Drop`) that make smart pointers work. You will build small programs that exercise each pointer type, then a single integration program that combines them.

You will build a single Cargo project called `node_graph` that models a small linked data structure. Tree-like and graph-like structures are a natural fit for smart pointers because they typically require shared ownership (multiple parents pointing to children, parent-child relationships in trees) and interior mutability (modifying nodes after construction). The lab walks you through each smart pointer type with a small focused example, then combines them in a final integration program.

The focus is on judgment: when `Box` is necessary versus when a regular value suffices, when `Rc` makes sense versus designing around shared ownership, when `RefCell` is appropriate versus a workaround for the borrow checker, when `Weak` solves a real cycle problem. Many decisions have multiple defensible answers; the lab guides you toward idiomatic Rust and asks you to compare alternatives.

By the end of this lab you will be able to:

- Use `Box<T>` for heap allocation, recursive types, and trait objects.
- Use `Rc<T>` for shared ownership in single-threaded code.
- Recognize the difference between `Rc<T>` and `Arc<T>` and when each is needed.
- Use `RefCell<T>` to mutate data through shared references with runtime borrow checking.
- Use `Cell<T>` for `Copy` types where `RefCell` is overkill.
- Combine `Rc<RefCell<T>>` for shared, mutable, single-threaded data.
- Recognize reference cycles and break them with `Weak<T>`.
- Implement `Deref` to make a custom type act like a pointer.
- Implement `Drop` to run cleanup code automatically.
- Choose the right smart pointer for a given situation.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** This lab uses only the standard library; no third-party crates are needed. Several exercises produce deliberate compile errors and runtime panics to illustrate when smart pointers' rules are violated; these are teaching moments, not bugs to fix.

> **A note on when you'll use these.** Most everyday Rust code does not reach for smart pointers beyond the ones implicit in `Vec` and `String`. The patterns in this lab are common in specific situations: graph data structures, observers, certain GUI patterns, and shared resources. Knowing they exist and recognizing them when reading code is more important than using them often.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new node_graph
cd node_graph
code .
```

Create a `lab11-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Box for Heap Allocation

**Estimated time:** 10-15 minutes
**Topics covered:** `Box<T>` basics, recursive types, trait objects

### Context

`Box<T>` is the simplest smart pointer. It allocates a value on the heap and gives you a pointer to it. This exercise covers the three main reasons to use it.

### Task 1.1 -- Basic Box

Replace the contents of `src/main.rs` with:

```rust
fn main() {
    let value: Box<i32> = Box::new(42);
    println!("value: {value}");
    println!("dereferenced: {}", *value);

    // Box derefs to the inner value, so most operations work transparently:
    let doubled = *value * 2;
    println!("doubled: {doubled}");
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
value: 42
dereferenced: 42
doubled: 84
```

`Box::new(42)` allocates memory on the heap, moves 42 into it, and returns a `Box<i32>` that owns the allocation. When `value` goes out of scope, the heap memory is freed.

For an `i32` like this, `Box` is wasteful. The integer would fit fine on the stack; putting it in a box just adds an allocation. The next tasks show when `Box` is actually useful.

### Task 1.2 -- A recursive type

Try to define a linked list directly:

```rust
enum LinkedList {
    Node(i32, LinkedList),         // ERROR
    Empty,
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0072]: recursive type `LinkedList` has infinite size
  |
  | enum LinkedList {
  | ^^^^^^^^^^^^^^^ recursive type has infinite size
  |     Node(i32, LinkedList),
  |               ---------- recursive without indirection
  |
  = help: insert some indirection (e.g., a `Box`, `Rc`, or `&`) to break the cycle
```

The compiler explains the issue. To compute the size of `LinkedList`, it would need to know the size of `Node`, which contains a `LinkedList`, which contains a `Node`, which contains a `LinkedList`... infinite recursion.

The fix is `Box`:

```rust
#[derive(Debug)]
enum LinkedList {
    Node(i32, Box<LinkedList>),
    Empty,
}

fn main() {
    let list = LinkedList::Node(
        1,
        Box::new(LinkedList::Node(
            2,
            Box::new(LinkedList::Node(
                3,
                Box::new(LinkedList::Empty),
            )),
        )),
    );

    println!("list: {list:?}");
}
```

Run the program. Expected output:

```
list: Node(1, Node(2, Node(3, Empty)))
```

`Box<LinkedList>` has a known size (a pointer, typically 8 bytes). The actual `LinkedList` lives on the heap. The compiler can now compute the size of `LinkedList`: an enum with a fixed largest variant size.

This pattern (recursive types use Box for the recursive field) is essential for any self-referential data structure.

### Task 1.3 -- A binary tree

The same idea applies to trees:

```rust
#[derive(Debug)]
struct TreeNode {
    value: i32,
    left: Option<Box<TreeNode>>,
    right: Option<Box<TreeNode>>,
}

impl TreeNode {
    fn leaf(value: i32) -> TreeNode {
        TreeNode { value, left: None, right: None }
    }

    fn branch(value: i32, left: TreeNode, right: TreeNode) -> TreeNode {
        TreeNode {
            value,
            left: Some(Box::new(left)),
            right: Some(Box::new(right)),
        }
    }

    fn sum(&self) -> i32 {
        let left_sum = self.left.as_ref().map_or(0, |n| n.sum());
        let right_sum = self.right.as_ref().map_or(0, |n| n.sum());
        self.value + left_sum + right_sum
    }
}

fn main() {
    let tree = TreeNode::branch(
        1,
        TreeNode::branch(2, TreeNode::leaf(4), TreeNode::leaf(5)),
        TreeNode::branch(3, TreeNode::leaf(6), TreeNode::leaf(7)),
    );

    println!("tree sum: {}", tree.sum());
}
```

Run the program. Expected output:

```
tree sum: 28
```

The tree structure is `Option<Box<TreeNode>>` for each child. `None` means no child; `Some(Box<...>)` is the child wrapped in a Box for the recursion. The traversal (`sum`) recursively walks the tree.

This is the canonical Rust tree node. Real-world trees in production code often elaborate on this pattern (adding parent pointers, balancing logic, etc.), but the core structure is always recursive types with Box for the recursive fields.

### Task 1.4 -- Trait objects with Box

When you need to store different concrete types behind a common trait, you use `Box<dyn Trait>`:

```rust
trait Operation {
    fn name(&self) -> &str;
    fn apply(&self, value: i32) -> i32;
}

struct Add(i32);
struct Multiply(i32);
struct Negate;

impl Operation for Add {
    fn name(&self) -> &str { "add" }
    fn apply(&self, value: i32) -> i32 { value + self.0 }
}

impl Operation for Multiply {
    fn name(&self) -> &str { "multiply" }
    fn apply(&self, value: i32) -> i32 { value * self.0 }
}

impl Operation for Negate {
    fn name(&self) -> &str { "negate" }
    fn apply(&self, value: i32) -> i32 { -value }
}

fn main() {
    let ops: Vec<Box<dyn Operation>> = vec![
        Box::new(Add(5)),
        Box::new(Multiply(3)),
        Box::new(Negate),
        Box::new(Add(100)),
    ];

    let mut value = 10;
    for op in &ops {
        let new_value = op.apply(value);
        println!("{}({value}) = {new_value}", op.name());
        value = new_value;
    }

    println!("final: {value}");
}
```

Run the program. Expected output:

```
add(10) = 15
multiply(15) = 45
negate(45) = -45
add(-45) = 55
final: 55
```

The vector holds three different concrete types (`Add`, `Multiply`, `Negate`) behind a common trait (`Operation`). Without `Box<dyn Operation>`, you could not have them in the same vector because they have different sizes.

`Box<dyn Operation>` uniformly takes 16 bytes on a 64-bit system (one pointer to the data, one to the vtable). The actual types live on the heap. The trait method calls go through the vtable (dynamic dispatch).

### Task 1.5 -- Returning Box<dyn Trait>

A function returning different concrete types must return `Box<dyn Trait>`:

```rust
fn make_operation(kind: &str) -> Box<dyn Operation> {
    match kind {
        "add5" => Box::new(Add(5)),
        "double" => Box::new(Multiply(2)),
        "negate" => Box::new(Negate),
        _ => Box::new(Add(0)),  // identity
    }
}

fn main() {
    for kind in &["add5", "double", "negate"] {
        let op = make_operation(kind);
        let result = op.apply(10);
        println!("{}: {result}", op.name());
    }
}
```

Run the program. Expected output:

```
add: 15
multiply: 20
negate: -10
```

The function selects a concrete type based on the input and returns it wrapped in a Box. The caller does not need to know the concrete type; they just use it through the trait.

Without `Box<dyn Trait>`, you could not write this function: `impl Trait` in return position requires a single concrete type for all return paths.

### Checkpoints

1. The `Box<i32>` example in Task 1.1 was described as "wasteful." When would putting an `i32` (or any simple type) in a `Box` actually be necessary?
2. In Task 1.2, the compiler explained the recursive type problem. Could the language have made recursive types work automatically (without requiring `Box`)? What would the language have to do internally to support that?
3. The `Vec<Box<dyn Operation>>` in Task 1.4 holds three different types uniformly. What is the runtime cost compared to a `Vec<Add>` (a homogeneous vector of one type)?

---

## Exercise 2 -- Rc for Shared Ownership

**Estimated time:** 10-15 minutes
**Topics covered:** `Rc<T>`, multiple owners, `Rc::clone` convention, reference counting

### Context

`Rc<T>` ("reference counted") lets multiple parts of your program share ownership of a value. When the last `Rc` pointing to the value is dropped, the value is freed.

### Task 2.1 -- The basic problem

Without smart pointers, ownership is exclusive. Try this:

```rust
fn main() {
    let data = vec![1, 2, 3, 4, 5];

    let owner_a = data;
    let owner_b = data;        // ERROR: data was moved to owner_a
    println!("a: {owner_a:?}");
    println!("b: {owner_b:?}");
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0382]: use of moved value: `data`
```

`Vec<i32>` is not `Copy`, so assignment moves it. After `let owner_a = data`, `data` is no longer valid. The second assignment fails.

You could clone:

```rust
let owner_b = data.clone();
```

But cloning duplicates the data: two complete vectors, two independent heaps. If you wanted both to refer to the *same* data (so changes by one are seen by the other, or just to share without duplication), cloning is wrong.

### Task 2.2 -- Rc to the rescue

`Rc::new` creates a reference-counted pointer:

```rust
use std::rc::Rc;

fn main() {
    let data = Rc::new(vec![1, 2, 3, 4, 5]);

    let owner_a = Rc::clone(&data);
    let owner_b = Rc::clone(&data);

    println!("original: {data:?}");
    println!("a: {owner_a:?}");
    println!("b: {owner_b:?}");

    println!("reference count: {}", Rc::strong_count(&data));
}
```

Run the program. Expected output:

```
original: [1, 2, 3, 4, 5]
a: [1, 2, 3, 4, 5]
b: [1, 2, 3, 4, 5]
reference count: 3
```

Three things to notice:

- `Rc::new(value)` wraps the value in a reference-counted pointer.
- `Rc::clone(&rc)` produces another pointer to the same data, incrementing the reference count.
- `Rc::strong_count(&rc)` returns the current count.

All three variables (`data`, `owner_a`, `owner_b`) point to the same heap allocation. The vector is not duplicated. The reference count tracks how many `Rc` pointers exist.

### Task 2.3 -- Drop and reference counting

Watch the count change as Rcs are dropped:

```rust
use std::rc::Rc;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);
    println!("after creation: count = {}", Rc::strong_count(&data));

    {
        let _temp = Rc::clone(&data);
        println!("inside block: count = {}", Rc::strong_count(&data));
    }   // _temp goes out of scope here

    println!("after block: count = {}", Rc::strong_count(&data));

    let another = Rc::clone(&data);
    println!("after another clone: count = {}", Rc::strong_count(&data));

    drop(another);
    println!("after drop: count = {}", Rc::strong_count(&data));
}
// data is dropped here; count becomes 0; data is freed
```

Run the program. Expected output:

```
after creation: count = 1
inside block: count = 2
after block: count = 1
after another clone: count = 2
after drop: count = 1
```

When an `Rc` is dropped (going out of scope, or via `drop()`), the count decreases. When the count reaches zero, the underlying data is freed.

The `drop()` function from the standard library explicitly drops a value. It is equivalent to letting the value go out of scope but is occasionally useful for explicit control.

### Task 2.4 -- The Rc::clone convention

You can also write `data.clone()` instead of `Rc::clone(&data)`. Both work, but the explicit `Rc::clone` is preferred by convention:

```rust
use std::rc::Rc;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);

    // Both of these are equivalent:
    let a = Rc::clone(&data);
    let b = data.clone();

    println!("count: {}", Rc::strong_count(&data));
    println!("a: {a:?}");
    println!("b: {b:?}");
}
```

Run the program. Expected output:

```
count: 3
a: [1, 2, 3]
b: [1, 2, 3]
```

Both calls do the same thing. The convention exists for clarity:

- `Rc::clone(&data)` is clearly "increment the reference count, cheap."
- `data.clone()` could be confused with a deep clone of the contained `Vec`, which would be expensive.

Since `clone()` works on any `Clone` type and could potentially do a lot of work, the explicit `Rc::clone` form documents the intent. When you read it, you know exactly what is happening.

### Task 2.5 -- A practical use: shared configuration

A real scenario where `Rc` is useful:

```rust
use std::rc::Rc;

#[derive(Debug)]
struct Config {
    name: String,
    max_connections: u32,
    timeout_ms: u64,
}

struct Service {
    name: String,
    config: Rc<Config>,
}

impl Service {
    fn new(name: &str, config: Rc<Config>) -> Service {
        Service {
            name: name.to_string(),
            config,
        }
    }

    fn describe(&self) {
        println!(
            "service '{}': max_conn={}, timeout={}ms",
            self.name, self.config.max_connections, self.config.timeout_ms
        );
    }
}

fn main() {
    let config = Rc::new(Config {
        name: String::from("production"),
        max_connections: 100,
        timeout_ms: 5000,
    });

    let service_a = Service::new("api", Rc::clone(&config));
    let service_b = Service::new("web", Rc::clone(&config));
    let service_c = Service::new("worker", Rc::clone(&config));

    service_a.describe();
    service_b.describe();
    service_c.describe();

    println!("config count: {}", Rc::strong_count(&config));
}
```

Run the program. Expected output:

```
service 'api': max_conn=100, timeout=5000ms
service 'web': max_conn=100, timeout=5000ms
service 'worker': max_conn=100, timeout=5000ms
config count: 4
```

Three services share a single `Config`. Without `Rc`, each service would need its own clone of the config (duplicating data) or borrow from somewhere (constraining lifetimes). `Rc` lets each service own its share of the config without duplicating it.

The configuration lives until the last service is dropped. After main returns, all services are dropped, the count reaches zero, and the config is freed.

### Checkpoints

1. The `Rc::clone(&data)` pattern is preferred over `data.clone()`. Both do the same thing for `Rc`. Why is the convention preferred? What would `data.clone()` mean for a `Vec` versus an `Rc<Vec>`?
2. Try this experiment in your head: in Task 2.5, what happens if the program also stored each service in a Vec and the Vec lived longer than `config`? Would there be a problem?
3. `Rc` is described as single-threaded only. Why? What goes wrong if multiple threads share an `Rc`? (Hint: think about the reference count update.)

---

## Exercise 3 -- Arc for Multi-Threaded Sharing

**Estimated time:** 5-10 minutes
**Topics covered:** `Arc<T>`, the difference from `Rc`, when each is appropriate

### Context

`Arc<T>` is the multi-threaded version of `Rc<T>`. The "A" stands for "atomic": the reference count uses atomic operations, making it safe to share across threads. You have already used Arc in Lab 10.

### Task 3.1 -- Why Rc fails across threads

Try sharing an Rc across threads:

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);

    let data_clone = Rc::clone(&data);
    thread::spawn(move || {
        println!("{data_clone:?}");        // ERROR
    });
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0277]: `Rc<Vec<i32>>` cannot be sent between threads safely
  |
  = help: within `[closure]`, the trait `Send` is not implemented for `Rc<Vec<i32>>`
```

The error is the `Send` rule from Module 14: `Rc` is not `Send` and cannot be moved into a thread. This is what protects you from the data race that non-atomic reference counting would cause.

### Task 3.2 -- Arc fixes it

Switch to Arc:

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);

    let mut handles = vec![];
    for i in 0..3 {
        let data_clone = Arc::clone(&data);
        let handle = thread::spawn(move || {
            println!("thread {i}: {data_clone:?}");
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("final count: {}", Arc::strong_count(&data));
}
```

Run the program. Expected output (thread order may vary):

```
thread 0: [1, 2, 3]
thread 1: [1, 2, 3]
thread 2: [1, 2, 3]
final count: 1
```

`Arc` has the same API as `Rc`: `Arc::new`, `Arc::clone`, `Arc::strong_count`. The difference is internal: the count is updated with atomic instructions, which are safe across threads but slightly slower than non-atomic.

When the threads finish (and their `Arc` clones are dropped), the count drops back to 1 (the original `data` in main).

### Task 3.3 -- When to use which

The decision is simple:

- **Single-threaded code:** use `Rc<T>`. Slightly faster (non-atomic operations).
- **Multi-threaded code:** use `Arc<T>`. Atomic operations make it thread-safe.

The compiler enforces the choice: `Rc` is `!Send`, so the compiler refuses to move it into a thread. If you ever try to use `Rc` across threads, the error is clear.

Some code conventions use `Arc` universally, even in single-threaded contexts, for consistency. The performance overhead is small (a few extra CPU cycles per clone), and it future-proofs the code in case threading is added later. Most code distinguishes them; the performance difference is usually invisible.

### Checkpoints

1. The compiler error in Task 3.1 said "`Rc<Vec<i32>>` cannot be sent between threads safely." This is a `Send`-trait error. What specific bug is being prevented?
2. Atomic operations are slightly slower than non-atomic ones (perhaps 2-3x for simple operations like increments). Why isn't this a reason to always use `Rc` even in multi-threaded code? When would the performance difference actually matter?
3. The text mentions some code conventions use `Arc` universally for consistency. What are the trade-offs of this choice compared to using `Rc` and `Arc` situationally?

---

## Exercise 4 -- RefCell for Interior Mutability

**Estimated time:** 15 minutes
**Topics covered:** `RefCell<T>`, runtime borrow checking, `borrow` and `borrow_mut`, panics

### Context

`RefCell<T>` provides interior mutability: it lets you mutate a value through a shared reference. The borrow checker still enforces "one mutable OR many immutable," but at runtime instead of compile time.

### Task 4.1 -- The motivating problem

Try to add a method that mutates state through `&self`:

```rust
struct Logger {
    messages: Vec<String>,
}

impl Logger {
    fn new() -> Logger {
        Logger { messages: Vec::new() }
    }

    fn log(&self, message: String) {
        self.messages.push(message);        // ERROR: cannot mutate through &self
    }

    fn print_all(&self) {
        for msg in &self.messages {
            println!("{msg}");
        }
    }
}

fn main() {
    let logger = Logger::new();
    logger.log(String::from("first"));
    logger.print_all();
}
```

Run `cargo build`. The compiler rejects the `log` method:

```
error[E0596]: cannot borrow `self.messages` as mutable, as it is behind a `&` reference
```

The method takes `&self` (shared), but it tries to mutate `messages`. The compiler refuses.

You could change `log` to take `&mut self`, but that propagates: every method calling `log` would need `&mut self`. The signature would say "this method mutates the logger" even though the user does not see any mutation. This is the wrong abstraction.

### Task 4.2 -- RefCell fixes it

Wrap the field in `RefCell`:

```rust
use std::cell::RefCell;

struct Logger {
    messages: RefCell<Vec<String>>,
}

impl Logger {
    fn new() -> Logger {
        Logger { messages: RefCell::new(Vec::new()) }
    }

    fn log(&self, message: String) {
        self.messages.borrow_mut().push(message);
    }

    fn print_all(&self) {
        for msg in self.messages.borrow().iter() {
            println!("{msg}");
        }
    }

    fn count(&self) -> usize {
        self.messages.borrow().len()
    }
}

fn main() {
    let logger = Logger::new();

    logger.log(String::from("first message"));
    logger.log(String::from("second message"));
    logger.log(String::from("third message"));

    println!("logged {} messages:", logger.count());
    logger.print_all();
}
```

Run the program. Expected output:

```
logged 3 messages:
first message
second message
third message
```

`RefCell::borrow_mut()` returns a mutable reference to the inner value, even though the cell itself was accessed through `&self`. The cell is "internally mutable": the inner value can change while the cell appears unchanged from the outside.

`borrow()` returns an immutable reference. `borrow_mut()` returns a mutable one. Both follow the borrow rules; the difference from compile-time borrowing is that violations are caught at runtime instead.

### Task 4.3 -- The runtime borrow check

The borrow rules are still enforced. Violating them panics:

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);

    let r1 = cell.borrow();        // immutable borrow
    let r2 = cell.borrow();        // another immutable borrow: OK
    println!("r1: {r1}, r2: {r2}");

    // Now try to violate the rules:
    let r3 = cell.borrow_mut();    // PANIC at runtime
    println!("r3: {}", *r3);
}
```

Run the program. Expected output:

```
r1: 5, r2: 5
thread 'main' panicked at 'already borrowed: BorrowMutError', ...
```

The borrows are tracked at runtime. When `borrow_mut()` is called while immutable borrows exist, the program panics. The compile-time check would have rejected this at compile time; the runtime check catches it at the moment the rule is violated.

This is the cost of `RefCell`: invalid usage is a runtime panic, not a compile error. The compile-time check is absolute (caught before deployment); the runtime check can fail in production if a code path you did not test triggers a conflict.

The benefit is that some valid patterns (like the logger above) are accepted that the compile-time checker would reject.

### Task 4.4 -- Scoping borrows

The borrow ends when the guard goes out of scope:

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);

    {
        let r1 = cell.borrow();
        println!("r1: {r1}");
    }   // r1 dropped here; immutable borrow ends

    {
        let mut r2 = cell.borrow_mut();
        *r2 = 100;
    }   // r2 dropped here; mutable borrow ends

    println!("final: {}", cell.borrow());
}
```

Run the program. Expected output:

```
r1: 5
final: 100
```

The inner blocks scope the borrows. After each block, the borrow ends, and a new borrow can be created. This is the same scoping rule as compile-time borrows.

Holding borrows for the minimum scope is good practice. If you borrow widely, you risk runtime panics from other code that tries to borrow incompatibly.

### Task 4.5 -- A practical use: caching

A common use of `RefCell` is caching computed values:

```rust
use std::cell::RefCell;

struct Computer {
    inputs: Vec<i32>,
    cached_sum: RefCell<Option<i32>>,
}

impl Computer {
    fn new(inputs: Vec<i32>) -> Computer {
        Computer {
            inputs,
            cached_sum: RefCell::new(None),
        }
    }

    fn sum(&self) -> i32 {
        let mut cache = self.cached_sum.borrow_mut();
        match *cache {
            Some(value) => {
                println!("(cache hit)");
                value
            }
            None => {
                println!("(computing)");
                let computed: i32 = self.inputs.iter().sum();
                *cache = Some(computed);
                computed
            }
        }
    }
}

fn main() {
    let c = Computer::new(vec![1, 2, 3, 4, 5]);

    println!("first call: {}", c.sum());
    println!("second call: {}", c.sum());
    println!("third call: {}", c.sum());
}
```

Run the program. Expected output:

```
(computing)
first call: 15
(cache hit)
second call: 15
(cache hit)
third call: 15
```

The `sum` method takes `&self` because, from the caller's view, getting the sum is read-only. Internally, the cache is filled on the first call and read on subsequent calls. The `RefCell` allows this internal mutation through a shared reference.

This is the classic case for `RefCell`: a method that is logically read-only but maintains internal mutable state (a cache, a counter, a logger).

### Task 4.6 -- Cell for Copy types

For `Copy` types, `Cell` is simpler than `RefCell`:

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<u32>,
    max: u32,
}

impl Counter {
    fn new(max: u32) -> Counter {
        Counter {
            count: Cell::new(0),
            max,
        }
    }

    fn increment(&self) -> bool {
        let current = self.count.get();
        if current < self.max {
            self.count.set(current + 1);
            true
        } else {
            false
        }
    }

    fn value(&self) -> u32 {
        self.count.get()
    }
}

fn main() {
    let counter = Counter::new(5);

    for _ in 0..7 {
        let success = counter.increment();
        println!("count: {}, success: {success}", counter.value());
    }
}
```

Run the program. Expected output:

```
count: 1, success: true
count: 2, success: true
count: 3, success: true
count: 4, success: true
count: 5, success: true
count: 5, success: false
count: 5, success: false
```

`Cell` has only `get()` (returns a copy of the inner value) and `set()` (replaces it). There are no borrows; no runtime borrow check; no possibility of panicking from borrow conflicts.

`Cell` works for `Copy` types because returning a copy is straightforward. For non-Copy types like `Vec` or `String`, you need `RefCell`.

When choosing:

- `Cell` for simple `Copy` types (integers, booleans, simple structs of Copy types).
- `RefCell` for everything else.

### Checkpoints

1. The `RefCell` example in Task 4.5 used a cache. The cache is logically "internal state that should not be visible from outside." Why is `RefCell` better than just making the method take `&mut self`?
2. Task 4.3 demonstrated a runtime panic from invalid borrows. In production code, what kinds of bugs could trigger this panic? How would you test for it?
3. `Cell` and `RefCell` both provide interior mutability. Why are they separate types rather than one type that handles both Copy and non-Copy cases?

---

## Exercise 5 -- Combining Rc and RefCell

**Estimated time:** 10-15 minutes
**Topics covered:** `Rc<RefCell<T>>`, shared mutable state, common patterns

### Context

`Rc<T>` gives shared ownership but provides no mutation. `RefCell<T>` gives mutation but no sharing. Combined as `Rc<RefCell<T>>`, you get both: many owners, all of whom can mutate the underlying data.

### Task 5.1 -- The pattern

Replace `src/main.rs` with:

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

    let owner_a = Rc::clone(&shared);
    let owner_b = Rc::clone(&shared);

    println!("initial: {:?}", shared.borrow());

    owner_a.borrow_mut().push(4);
    owner_b.borrow_mut().push(5);
    shared.borrow_mut().push(6);

    println!("after pushes: {:?}", shared.borrow());
    println!("(seen through a): {:?}", owner_a.borrow());
    println!("(seen through b): {:?}", owner_b.borrow());

    println!("reference count: {}", Rc::strong_count(&shared));
}
```

Run the program. Expected output:

```
initial: [1, 2, 3]
after pushes: [1, 2, 3, 4, 5, 6]
(seen through a): [1, 2, 3, 4, 5, 6]
(seen through b): [1, 2, 3, 4, 5, 6]
reference count: 3
```

All three `Rc`s point to the same `RefCell<Vec<i32>>`. Each can mutate the vector through `borrow_mut()`. The mutations are visible through every Rc because they all point to the same data.

This is the "shared mutable state" pattern in single-threaded Rust. The multi-threaded analogue is `Arc<Mutex<T>>` (from Lab 10).

### Task 5.2 -- A practical use: a shared counter

Multiple parts of a program tracking a shared count:

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Department {
    name: String,
    request_count: Rc<RefCell<u32>>,
}

impl Department {
    fn new(name: &str, count: Rc<RefCell<u32>>) -> Department {
        Department {
            name: name.to_string(),
            request_count: count,
        }
    }

    fn handle_request(&self) {
        let mut count = self.request_count.borrow_mut();
        *count += 1;
        println!("{}: handled request, total now {}", self.name, *count);
    }
}

fn main() {
    let total_requests = Rc::new(RefCell::new(0));

    let sales = Department::new("Sales", Rc::clone(&total_requests));
    let support = Department::new("Support", Rc::clone(&total_requests));
    let billing = Department::new("Billing", Rc::clone(&total_requests));

    sales.handle_request();
    support.handle_request();
    support.handle_request();
    billing.handle_request();
    sales.handle_request();

    println!("\ngrand total: {}", total_requests.borrow());
}
```

Run the program. Expected output:

```
Sales: handled request, total now 1
Support: handled request, total now 2
Support: handled request, total now 3
Billing: handled request, total now 4
Sales: handled request, total now 5

grand total: 5
```

Three departments share a single counter. Each can read and increment it. The final count reflects all five increments.

Without `Rc<RefCell<u32>>`, you would need either:

- A single owner with method calls to update it (requires `&mut` access, more restrictive).
- Each department with its own counter, plus a way to aggregate (more bookkeeping).
- A truly shared atomic counter (more complex than needed for single-threaded code).

The combined pointer makes the design straightforward.

### Task 5.3 -- A graph node

A more substantial example, with nodes that can hold references to other nodes:

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct Node {
    value: i32,
    neighbors: RefCell<Vec<Rc<Node>>>,
}

impl Node {
    fn new(value: i32) -> Rc<Node> {
        Rc::new(Node {
            value,
            neighbors: RefCell::new(Vec::new()),
        })
    }

    fn connect(&self, other: Rc<Node>) {
        self.neighbors.borrow_mut().push(other);
    }

    fn neighbor_count(&self) -> usize {
        self.neighbors.borrow().len()
    }
}

fn main() {
    let a = Node::new(1);
    let b = Node::new(2);
    let c = Node::new(3);

    a.connect(Rc::clone(&b));
    a.connect(Rc::clone(&c));
    b.connect(Rc::clone(&c));

    println!("a (value {}) has {} neighbors", a.value, a.neighbor_count());
    println!("b (value {}) has {} neighbors", b.value, b.neighbor_count());
    println!("c (value {}) has {} neighbors", c.value, c.neighbor_count());

    println!("\nreference counts:");
    println!("  a: {}", Rc::strong_count(&a));
    println!("  b: {}", Rc::strong_count(&b));
    println!("  c: {}", Rc::strong_count(&c));
}
```

Run the program. Expected output:

```
a (value 1) has 2 neighbors
b (value 2) has 1 neighbors
c (value 3) has 0 neighbors

reference counts:
  a: 1
  b: 2
  c: 3
```

Each node has:

- A value.
- A `RefCell<Vec<Rc<Node>>>` of neighbors (mutable list of shared pointers to other nodes).

The neighbor list is wrapped in `RefCell` so the `connect` method can mutate it through `&self`. The neighbors themselves are `Rc<Node>` so multiple nodes can reference the same target.

The reference counts show how the structure is held together. Node `a` is referenced only by the local variable (`1` count). Node `b` is referenced by `a` (its neighbor) and the local variable (`2` count). Node `c` is referenced by `a`, by `b`, and by the local variable (`3` count).

When all the local variables go out of scope, all the counts drop, and the nodes are freed.

But notice: there is no cycle yet. Each connection is one-way. If we made the graph bidirectional (each node references its neighbors AND each neighbor back-references it), we would create cycles, which `Rc` cannot collect. The next exercise addresses this.

### Checkpoints

1. The `Rc<RefCell<T>>` pattern provides shared mutable state. What is the practical cost of this approach compared to just using `&mut T` references?
2. In Task 5.3, the graph is one-directional (each node knows its neighbors, but neighbors do not know who points to them). Could you implement a bidirectional graph with just `Rc<RefCell<T>>`? What would go wrong?
3. The combination `Rc<RefCell<T>>` is sometimes called "the workaround when ordinary Rust borrowing is not flexible enough." When should you suspect that you are using it inappropriately and reconsidering the design?

---

## Exercise 6 -- Reference Cycles and Weak

**Estimated time:** 15 minutes
**Topics covered:** reference cycles, memory leaks with `Rc`, `Weak` references, when to use weak

### Context

`Rc` cannot collect cycles. If two values hold `Rc`s to each other (a cycle), neither will ever be dropped, even after all external references are gone. `Weak` is the solution.

### Task 6.1 -- Creating a leak

Replace `src/main.rs` with:

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct Node {
    name: String,
    next: RefCell<Option<Rc<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("dropping node {}", self.name);
    }
}

fn main() {
    let a = Rc::new(Node {
        name: String::from("A"),
        next: RefCell::new(None),
    });

    let b = Rc::new(Node {
        name: String::from("B"),
        next: RefCell::new(None),
    });

    // Create a cycle: a -> b -> a
    *a.next.borrow_mut() = Some(Rc::clone(&b));
    *b.next.borrow_mut() = Some(Rc::clone(&a));

    println!("a count: {}", Rc::strong_count(&a));
    println!("b count: {}", Rc::strong_count(&b));

    println!("end of main");
}
```

Run the program. Expected output:

```
a count: 2
b count: 2
end of main
```

Notice what is missing: there are no "dropping node A" or "dropping node B" messages.

This is a memory leak. After main, `a` and `b` go out of scope, dropping the local `Rc`s. The count for each goes from 2 to 1. But each node still holds an `Rc` to the other:

- `a` has count 1 (from `b.next`).
- `b` has count 1 (from `a.next`).

Neither count reaches zero. The `Drop` implementation never runs. The memory is allocated forever (until the program exits).

This is the classic cycle problem with reference counting. `Rc` cannot detect cycles; it can only count direct ownership. Cycles defeat the counting.

### Task 6.2 -- Breaking the cycle with Weak

`Weak::new` (or `Rc::downgrade`) creates a non-owning reference:

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    name: String,
    next: RefCell<Option<Weak<Node>>>,   // changed from Rc to Weak
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("dropping node {}", self.name);
    }
}

fn main() {
    let a = Rc::new(Node {
        name: String::from("A"),
        next: RefCell::new(None),
    });

    let b = Rc::new(Node {
        name: String::from("B"),
        next: RefCell::new(None),
    });

    // Use Weak references to avoid creating ownership cycles:
    *a.next.borrow_mut() = Some(Rc::downgrade(&b));
    *b.next.borrow_mut() = Some(Rc::downgrade(&a));

    println!("a strong count: {}, weak count: {}",
        Rc::strong_count(&a), Rc::weak_count(&a));
    println!("b strong count: {}, weak count: {}",
        Rc::strong_count(&b), Rc::weak_count(&b));

    println!("end of main");
}
```

Run the program. Expected output (order of drops may vary):

```
a strong count: 1, weak count: 1
b strong count: 1, weak count: 1
end of main
dropping node B
dropping node A
```

Two important changes:

- The `next` field is now `RefCell<Option<Weak<Node>>>` instead of `RefCell<Option<Rc<Node>>>`.
- `Rc::downgrade(&b)` creates a `Weak<Node>` that points to `b` without contributing to the strong count.

The strong counts are now 1 each (just the local variables). Weak references show as a separate count (1 each, from the other node's `next`).

When the local variables go out of scope, the strong counts drop to 0, and the nodes are freed. The `Drop` implementations run. No leak.

### Task 6.3 -- Upgrading a Weak

A `Weak` cannot be used directly. You must "upgrade" it to a (possibly null) `Rc`:

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    name: String,
    next: RefCell<Option<Weak<Node>>>,
}

fn main() {
    let a = Rc::new(Node {
        name: String::from("A"),
        next: RefCell::new(None),
    });

    let b = Rc::new(Node {
        name: String::from("B"),
        next: RefCell::new(None),
    });

    *a.next.borrow_mut() = Some(Rc::downgrade(&b));

    // Try to follow the link from a:
    let weak = a.next.borrow().as_ref().unwrap().clone();
    if let Some(strong) = weak.upgrade() {
        println!("a points to: {}", strong.name);
    } else {
        println!("a's target is gone");
    }

    // Now drop b, then try to follow the link:
    drop(b);

    let weak = a.next.borrow().as_ref().unwrap().clone();
    if let Some(strong) = weak.upgrade() {
        println!("a still points to: {}", strong.name);
    } else {
        println!("a's target is gone (b was dropped)");
    }
}
```

Run the program. Expected output:

```
a points to: B
a's target is gone (b was dropped)
```

`weak.upgrade()` returns `Option<Rc<T>>`. If the target still exists, you get `Some(Rc)`. If it has been freed, you get `None`.

This is the key property of `Weak`: it does not keep the target alive. The target can be freed independently. When you want to follow the weak reference, you upgrade; if upgrade fails, the target is gone.

### Task 6.4 -- A practical tree with parent links

A common pattern is a tree where children can navigate back to their parent. The parent owns the children (strong reference); the children reference the parent weakly (so cycles do not form):

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct TreeNode {
    value: i32,
    parent: RefCell<Weak<TreeNode>>,
    children: RefCell<Vec<Rc<TreeNode>>>,
}

impl TreeNode {
    fn new(value: i32) -> Rc<TreeNode> {
        Rc::new(TreeNode {
            value,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(Vec::new()),
        })
    }

    fn add_child(parent: &Rc<TreeNode>, child: Rc<TreeNode>) {
        *child.parent.borrow_mut() = Rc::downgrade(parent);
        parent.children.borrow_mut().push(child);
    }

    fn parent_value(&self) -> Option<i32> {
        self.parent.borrow().upgrade().map(|p| p.value)
    }
}

fn main() {
    let root = TreeNode::new(1);
    let child_a = TreeNode::new(2);
    let child_b = TreeNode::new(3);
    let grandchild = TreeNode::new(4);

    TreeNode::add_child(&root, Rc::clone(&child_a));
    TreeNode::add_child(&root, Rc::clone(&child_b));
    TreeNode::add_child(&child_a, Rc::clone(&grandchild));

    println!("root: {}", root.value);
    println!("child_a: {} (parent: {:?})", child_a.value, child_a.parent_value());
    println!("child_b: {} (parent: {:?})", child_b.value, child_b.parent_value());
    println!("grandchild: {} (parent: {:?})", grandchild.value, grandchild.parent_value());

    println!("\nroot strong count: {}", Rc::strong_count(&root));
    println!("child_a strong count: {}", Rc::strong_count(&child_a));
    println!("child_a weak count: {}", Rc::weak_count(&child_a));
}
```

Run the program. Expected output:

```
root: 1
child_a: 2 (parent: Some(1))
child_b: 3 (parent: Some(1))
grandchild: 4 (parent: Some(1))

root strong count: 1
child_a strong count: 2
child_a weak count: 1
```

Each child can navigate to its parent via the weak reference. The parent does not keep the child alive (children's strong counts increase only from the parent's `children` vector); the child does not keep the parent alive (parent's strong counts come only from the root variable).

When the root variable goes out of scope, the entire tree is freed cleanly. No cycle, no leak.

### Task 6.5 -- The "weak parent" convention

The pattern from Task 6.4 is the canonical solution for parent-child relationships in trees:

- **Parent owns children** via `Rc`. When the parent is dropped, children are dropped.
- **Children reference parents** via `Weak`. The parent's lifetime is not tied to the children's.

The direction of "weak" is determined by ownership semantics: which side conceptually owns the lifetime?

For a tree, the parent owns the children: when you drop the root, the entire tree should go. So the parent → child direction is `Rc`, and the child → parent direction is `Weak`.

For other relationships, the choice depends on what makes sense:

- **Observer pattern:** subjects own observers via `Rc`; observers reference subjects via `Weak`.
- **Cache with back-pointers:** the cache owns entries (Rc); entries reference back to the cache (Weak).
- **Bidirectional iteration:** one direction is the "main" structure (Rc); the other is for navigation only (Weak).

The pattern recognition: if you have references in both directions, one must be weak to avoid a cycle. Decide which side's lifetime should drive the other; make that the strong direction.

### Checkpoints

1. The leak in Task 6.1 was silent. No error, no panic, just a missing Drop message. How would you detect this in production code where you might not notice the missing message?
2. `Weak::upgrade()` returns `Option<Rc<T>>`. The target might have been freed. What does this say about how you should design code that uses weak references? When can you assume upgrade will succeed?
3. The tree in Task 6.4 used the "weak parent, strong children" pattern. Could you reverse this (strong parent reference from children, weak references to children from parent)? What design would this represent? When might it be appropriate?

---

## Exercise 7 -- Implementing Deref and Drop

**Estimated time:** 10-15 minutes
**Topics covered:** the `Deref` trait, deref coercion, the `Drop` trait, automatic cleanup

### Context

Smart pointers work because of two traits: `Deref` (which makes a type act like a pointer) and `Drop` (which runs cleanup code when a value goes out of scope). This exercise builds a small custom smart pointer to see how these traits work.

### Task 7.1 -- A custom Box

Implement a simple Box-like type:

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(value: T) -> MyBox<T> {
        MyBox(value)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0
    }
}

fn main() {
    let x = MyBox::new(5);

    // The * operator works because of Deref:
    println!("*x = {}", *x);

    // Methods on T are available through deref coercion:
    let s = MyBox::new(String::from("hello"));
    println!("length: {}", s.len());          // String::len via deref
    println!("uppercase: {}", s.to_uppercase());
}
```

Run the program. Expected output:

```
*x = 5
length: 5
uppercase: HELLO
```

The `Deref` implementation has two key parts:

- `type Target = T;` says what type the dereference produces. For `MyBox<i32>`, the target is `i32`. For `MyBox<String>`, the target is `String`.
- `fn deref(&self) -> &T` returns a reference to the inner value. The `*` operator calls this.

With `Deref` implemented, `*my_box` gives you the inner value (or rather, a copy of it, since the target is what gets dereferenced).

### Task 7.2 -- Deref coercion

The compiler automatically calls `deref()` when types do not quite match. This is called "deref coercion":

```rust
fn print_str(s: &str) {
    println!("got: {s}");
}

fn main() {
    let owned = String::from("hello");
    print_str(&owned);                    // &String coerced to &str
    print_str("literal");                  // &str directly

    let boxed = Box::new(String::from("boxed"));
    print_str(&boxed);                     // &Box<String> -> &String -> &str

    let my_boxed = MyBox::new(String::from("custom"));
    print_str(&my_boxed);                  // &MyBox<String> -> &String -> &str
}
```

You will need to bring `MyBox` and its `Deref` impl into scope. Run the program:

```
got: hello
got: literal
got: boxed
got: custom
```

Each call works because of the chain of deref impls:

- `&String` derefs to `&str` (because `String` implements `Deref<Target = str>`).
- `&Box<String>` derefs to `&String` derefs to `&str`.
- `&MyBox<String>` derefs to `&String` derefs to `&str`.

The compiler walks the chain automatically. This is why string parameter types are so flexible: take `&str`, and callers can pass `&str`, `&String`, `&Box<String>`, `&MyBox<String>`, or anything else that derefs (chain-wise) to `str`.

### Task 7.3 -- Implementing Drop

The `Drop` trait runs code when a value is destroyed:

```rust
struct Resource {
    name: String,
}

impl Resource {
    fn new(name: &str) -> Resource {
        println!("creating: {name}");
        Resource { name: name.to_string() }
    }
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("dropping: {}", self.name);
    }
}

fn main() {
    println!("--- start ---");

    let r1 = Resource::new("first");

    {
        let r2 = Resource::new("second");
        println!("inside block");
    }   // r2 dropped here

    println!("after block");

    let r3 = Resource::new("third");

    println!("--- end ---");
}   // r3 dropped, then r1 dropped (reverse declaration order)
```

Run the program. Expected output:

```
--- start ---
creating: first
creating: second
inside block
dropping: second
after block
creating: third
--- end ---
dropping: third
dropping: first
```

Key observations:

- `drop()` is called automatically when each value goes out of scope.
- The order is reverse-declaration: values are dropped in the opposite order they were created.
- `r2` was dropped at the end of the inner block; `r3` and `r1` at the end of main.

The `Drop` trait is what gives Rust its deterministic destruction: every value's cleanup happens at a known point in the code, not at some indeterminate future time (as with garbage collection).

### Task 7.4 -- Drop for cleanup

A practical example: a temporary file that cleans itself up:

```rust
struct TempFile {
    path: String,
    file: std::fs::File,
}

impl TempFile {
    fn new(path: &str) -> std::io::Result<TempFile> {
        let file = std::fs::File::create(path)?;
        Ok(TempFile {
            path: path.to_string(),
            file,
        })
    }
}

impl Drop for TempFile {
    fn drop(&mut self) {
        println!("cleaning up: {}", self.path);
        if let Err(e) = std::fs::remove_file(&self.path) {
            eprintln!("failed to remove {}: {}", self.path, e);
        }
    }
}

fn main() -> std::io::Result<()> {
    {
        let _temp = TempFile::new("temporary.txt")?;
        println!("temp file exists");

        // Verify the file exists:
        println!("file exists: {}", std::path::Path::new("temporary.txt").exists());
    }   // _temp dropped here; file is deleted

    println!("after scope, file exists: {}", std::path::Path::new("temporary.txt").exists());

    Ok(())
}
```

Run the program. Expected output:

```
temp file exists
file exists: true
cleaning up: temporary.txt
after scope, file exists: false
```

The `TempFile` creates a file in its constructor and deletes it in its destructor. The user does not have to remember to clean up; the cleanup happens automatically when the `TempFile` goes out of scope.

This is the RAII (Resource Acquisition Is Initialization) pattern, which Rust enforces through `Drop`. Resources are tied to scopes; they are acquired when the object is created and released when it is destroyed. No explicit cleanup, no leaks.

Even if the function returns early (via `?` or other paths), Drop still runs as the value goes out of scope. The cleanup is guaranteed.

### Task 7.5 -- Cannot call drop directly

Try to drop a value yourself:

```rust
struct Resource;

impl Drop for Resource {
    fn drop(&mut self) {
        println!("dropping");
    }
}

fn main() {
    let r = Resource;
    r.drop();           // ERROR
    println!("after");
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0040]: explicit use of destructor method
```

You cannot call `drop()` directly. The reason is that Rust's automatic dropping happens when the value goes out of scope. If you also called `drop()` manually, the value would be dropped twice, causing memory bugs.

To drop a value before its scope ends (release a lock early, close a file early), use `std::mem::drop`:

```rust
fn main() {
    let r = Resource;
    std::mem::drop(r);              // OK: consumes r, calls Drop on it
    println!("after");
}
```

Run the program. Expected output:

```
dropping
after
```

`std::mem::drop` takes the value by value (consuming it), then lets it go out of scope inside the function, which triggers `Drop` exactly once. This is the safe way to "drop early."

### Checkpoints

1. The `Deref` trait makes `MyBox<T>` work with the `*` operator. What other operators or features become available because of `Deref`? (Hint: the lab demonstrated deref coercion.)
2. Drop order is reverse-declaration. Why does this make sense? What kinds of resources need a specific cleanup order?
3. The compiler refuses to let you call `drop()` directly. The `std::mem::drop` workaround consumes the value. Why is this distinction important? What goes wrong if you could call `drop()` twice?

---

## Exercise 8 -- Putting It Together

**Estimated time:** 15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from Module 15 into a single program: a small organizational tree that uses every pointer type. The structure has nodes (employees) connected by parent-child relationships, with shared metadata (a shared `Config`), tracked counts (via `RefCell`), and an explicit cleanup demonstration.

### Task 8.1 -- The full program

Replace your `src/main.rs` with this complete program:

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

// ============================================================
// Shared organizational metadata (Rc for shared ownership)
// ============================================================

#[derive(Debug)]
struct OrgConfig {
    name: String,
    headcount_target: u32,
}

// ============================================================
// Employee node with parent-child relationships
// ============================================================

struct Employee {
    name: String,
    title: String,
    config: Rc<OrgConfig>,
    manager: RefCell<Weak<Employee>>,
    reports: RefCell<Vec<Rc<Employee>>>,
}

impl Employee {
    fn new(name: &str, title: &str, config: Rc<OrgConfig>) -> Rc<Employee> {
        Rc::new(Employee {
            name: name.to_string(),
            title: title.to_string(),
            config,
            manager: RefCell::new(Weak::new()),
            reports: RefCell::new(Vec::new()),
        })
    }

    fn add_report(manager: &Rc<Employee>, report: Rc<Employee>) {
        *report.manager.borrow_mut() = Rc::downgrade(manager);
        manager.reports.borrow_mut().push(report);
    }

    fn manager_name(&self) -> Option<String> {
        self.manager.borrow().upgrade().map(|m| m.name.clone())
    }

    fn report_count(&self) -> usize {
        self.reports.borrow().len()
    }

    fn total_subtree_size(&self) -> u32 {
        let mut count = 1;        // self
        for report in self.reports.borrow().iter() {
            count += report.total_subtree_size();
        }
        count
    }
}

impl Drop for Employee {
    fn drop(&mut self) {
        println!("cleanup: removing employee record for {}", self.name);
    }
}

// ============================================================
// A counter that tracks activity across the org
// ============================================================

struct ActivityTracker {
    name: String,
    counts: RefCell<Vec<String>>,
}

impl ActivityTracker {
    fn new(name: &str) -> ActivityTracker {
        ActivityTracker {
            name: name.to_string(),
            counts: RefCell::new(Vec::new()),
        }
    }

    fn record(&self, event: &str) {
        self.counts.borrow_mut().push(event.to_string());
    }

    fn summary(&self) -> String {
        let events = self.counts.borrow();
        format!("{}: {} events ({})", self.name, events.len(), events.join(", "))
    }
}

impl Drop for ActivityTracker {
    fn drop(&mut self) {
        println!("cleanup: closing activity tracker '{}' with {} events",
            self.name, self.counts.borrow().len());
    }
}

// ============================================================
// Main: build the org structure and exercise the patterns
// ============================================================

fn main() {
    println!("=== Setting up organization ===");

    // Shared config used by every employee:
    let config = Rc::new(OrgConfig {
        name: String::from("Acme Corp"),
        headcount_target: 50,
    });

    // Build the org tree:
    let ceo = Employee::new("Alice", "CEO", Rc::clone(&config));
    let cto = Employee::new("Bob", "CTO", Rc::clone(&config));
    let cfo = Employee::new("Carol", "CFO", Rc::clone(&config));

    Employee::add_report(&ceo, Rc::clone(&cto));
    Employee::add_report(&ceo, Rc::clone(&cfo));

    let engineer1 = Employee::new("Dave", "Engineer", Rc::clone(&config));
    let engineer2 = Employee::new("Eve", "Engineer", Rc::clone(&config));
    Employee::add_report(&cto, Rc::clone(&engineer1));
    Employee::add_report(&cto, Rc::clone(&engineer2));

    let accountant = Employee::new("Frank", "Accountant", Rc::clone(&config));
    Employee::add_report(&cfo, Rc::clone(&accountant));

    // Use an activity tracker (RefCell) to record events:
    let tracker = ActivityTracker::new("hiring");
    tracker.record("Alice promoted to CEO");
    tracker.record("Bob hired as CTO");
    tracker.record("Carol hired as CFO");
    tracker.record("Dave hired as Engineer");

    // Report on the structure:
    println!("\n=== Organization Report ===");
    println!("Company: {}", config.name);
    println!("Headcount target: {}", config.headcount_target);
    println!("Config Rc count: {}", Rc::strong_count(&config));

    println!("\nReporting structure:");
    println!("  {} ({}): manages {}",
        ceo.name, ceo.title, ceo.report_count());
    println!("  {} ({}): reports to {:?}, manages {}",
        cto.name, cto.title, cto.manager_name(), cto.report_count());
    println!("  {} ({}): reports to {:?}, manages {}",
        cfo.name, cfo.title, cfo.manager_name(), cfo.report_count());
    println!("  {} ({}): reports to {:?}",
        engineer1.name, engineer1.title, engineer1.manager_name());
    println!("  {} ({}): reports to {:?}",
        engineer2.name, engineer2.title, engineer2.manager_name());
    println!("  {} ({}): reports to {:?}",
        accountant.name, accountant.title, accountant.manager_name());

    println!("\nTotal organization size: {}", ceo.total_subtree_size());

    println!("\nActivity: {}", tracker.summary());

    // Show reference counts:
    println!("\nReference counts:");
    println!("  CEO strong: {}", Rc::strong_count(&ceo));
    println!("  CTO strong: {}, weak: {}",
        Rc::strong_count(&cto), Rc::weak_count(&cto));
    println!("  CFO strong: {}, weak: {}",
        Rc::strong_count(&cfo), Rc::weak_count(&cfo));

    // Explicitly drop one local reference to show it does not free the data:
    println!("\n=== Dropping local reference to engineer2 ===");
    drop(engineer2);
    println!("(engineer2 not freed; CTO still holds an Rc to it)");

    println!("\n=== End of main; cleanup begins ===");
}
```

Run the program:

```bash
cargo run
```

Expected output (the cleanup order may vary slightly):

```
=== Setting up organization ===

=== Organization Report ===
Company: Acme Corp
Headcount target: 50
Config Rc count: 7

Reporting structure:
  Alice (CEO): manages 2
  Bob (CTO): reports to Some("Alice"), manages 2
  Carol (CFO): reports to Some("Alice"), manages 1
  Dave (Engineer): reports to Some("Bob")
  Eve (Engineer): reports to Some("Bob")
  Frank (Accountant): reports to Some("Carol")

Total organization size: 6

Activity: hiring: 4 events (Alice promoted to CEO, Bob hired as CTO, Carol hired as CFO, Dave hired as Engineer)

Reference counts:
  CEO strong: 1
  CTO strong: 2, weak: 0
  CFO strong: 2, weak: 0

=== Dropping local reference to engineer2 ===
(engineer2 not freed; CTO still holds an Rc to it)

=== End of main; cleanup begins ===
cleanup: closing activity tracker 'hiring' with 4 events
cleanup: removing employee record for Frank
cleanup: removing employee record for Eve
cleanup: removing employee record for Dave
cleanup: removing employee record for Carol
cleanup: removing employee record for Bob
cleanup: removing employee record for Alice
```

The cleanup messages at the end show each employee being dropped exactly once. The recursive structure unwinds: when `ceo` goes out of scope, its `reports` Vec is dropped, which drops each report's Rc, which (if the count reaches zero) drops the employee, which drops its own reports, and so on. The tree disassembles cleanly with no leaks.

### Task 8.2 -- Identify the patterns

In `lab11-notes.md`, identify each instance of the following patterns in the program above:

1. A `Rc<T>` used to share an immutable value across many owners.
2. A `Rc<Employee>` used to share a node across multiple references.
3. A `Weak<Employee>` used to avoid a reference cycle.
4. A `RefCell<T>` used to allow mutation through a `&self` method.
5. The `Rc::downgrade` function called to create a weak reference.
6. The `Weak::upgrade` method (via `Option::map`) to safely access a weak reference's target.
7. A `Drop` implementation that runs on each instance of a struct.
8. A recursive method (`total_subtree_size`) that walks the tree structure.
9. The `std::mem::drop` function used to release a reference early.
10. The "weak parent, strong children" pattern in the tree structure.

### Task 8.3 -- Predict and verify

For each of the following code changes, predict what will happen and verify. Write your predictions in `lab11-notes.md` BEFORE running.

**Change A:** Change the `manager: RefCell<Weak<Employee>>` field to `manager: RefCell<Option<Rc<Employee>>>` (strong reference instead of weak). What happens at the end of main? Do the cleanup messages still appear?

**Change B:** Add this line just before the "End of main" message in `main`:

```rust
let _borrow_test = cto.reports.borrow();
cto.reports.borrow_mut().push(Rc::clone(&accountant));
```

What happens?

**Change C:** Remove the `Drop` implementation on `Employee` entirely. Does anything change in the program's behavior, beyond the missing cleanup messages?

For each change, write your prediction first, then run and verify.

### Checkpoints

1. The `OrgConfig` is wrapped in `Rc` because every employee needs access to it. Could this have been done with a regular `&OrgConfig` reference? What would change about the program's structure?
2. The `manager` field uses `Weak<Employee>`. The `reports` field uses `Vec<Rc<Employee>>` (strong). Why this asymmetry? What would happen if both were strong?
3. The `Drop` implementations on `Employee` show that each employee is freed exactly once when the program ends. What does this tell you about the reference counting? Are any of the counts left non-zero?

---

## Summary and Reflection

You have now used every concept from Module 15 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Box | heap allocation, recursive types, trait objects | Box is the simplest smart pointer; needed for known-size requirements and uniform trait collections. |
| 2 -- Rc | shared ownership, reference counting | Rc lets multiple parts of a program share data without cloning or complicated lifetimes. |
| 3 -- Arc | thread-safe sharing | Arc is the multi-threaded version; the compiler enforces the choice. |
| 4 -- RefCell and Cell | interior mutability, runtime borrow checking | RefCell lets methods take &self while still mutating internal state, at the cost of runtime checks. |
| 5 -- Rc + RefCell | shared mutable state | The combination gives single-threaded shared mutable data; analogous to Arc<Mutex<T>> for threads. |
| 6 -- Weak | breaking cycles, non-owning references | Weak allows references that do not extend lifetime; essential for tree-with-parent and similar patterns. |
| 7 -- Deref and Drop | making types act like pointers, automatic cleanup | These traits are the foundation: Deref gives pointer-like behavior, Drop ensures cleanup happens. |
| 8 -- Integration | combining everything | Real Rust code combines these patterns; the structure of the program guides which to use where. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab11-notes.md` before your next session.

1. Of the smart pointer concepts in Module 15, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. The lab introduced `Rc<RefCell<T>>` for shared mutable state in single-threaded code, and Lab 10 introduced `Arc<Mutex<T>>` for the multi-threaded equivalent. These two patterns express the same idea (shared mutable data) but use different mechanisms. What are the trade-offs of each? When would the runtime cost of one outweigh its programming convenience?

3. The Drop trait gives Rust deterministic destruction: every value's cleanup happens at a known point in the source code. This is similar to C++ destructors and very different from Java/Python's garbage collection. Identify a specific kind of bug or pattern that deterministic destruction prevents or makes easier compared to garbage collection. What does Rust gain from this design? What does it lose?

---

*End of Lab 11*
