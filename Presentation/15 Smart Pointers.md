# Module 15: Smart Pointers and Interior Mutability

## Module Overview

Smart pointers are types that act like pointers but provide additional capabilities. You have already used several without thinking about them: `String` is a smart pointer (it owns a heap buffer), `Vec<T>` is a smart pointer (it owns a heap array), and `Arc<T>` from Module 14 is a smart pointer (it manages shared ownership across threads). This module covers the broader family.

Most smart pointers exist because the basic ownership rules from Module 6 are sometimes too restrictive. The rules say "exactly one owner, references must always be valid," which works for the vast majority of code. But there are real situations where you need shared ownership (multiple parts of a program holding the same data), or interior mutability (modifying data through a shared reference), or runtime-checked borrowing instead of compile-time. Smart pointers provide these capabilities while preserving safety.

Three ideas are central:

1. **Smart pointers are types, not language features.** They are defined in the standard library using normal Rust code, with traits like `Deref` and `Drop` giving them pointer-like behavior. You can write your own smart pointers if you need to.
2. **Each smart pointer trades something for something.** `Box` adds heap allocation. `Rc` and `Arc` add reference counting overhead. `RefCell` moves borrow checking to runtime. The right pointer depends on what you need.
3. **Interior mutability is the safety valve.** When the borrow checker is too strict, types like `Cell` and `RefCell` provide a controlled way to mutate data through shared references, with safety enforced by other means.

By the end of this module, students will:

- Use `Box<T>` to put values on the heap and recognize when this is necessary.
- Use `Rc<T>` for shared ownership in single-threaded code and `Arc<T>` for multi-threaded code.
- Use `RefCell<T>` to mutate data through shared references with runtime borrow checking.
- Use `Cell<T>` for `Copy` types where `RefCell` is overkill.
- Use `Weak<T>` to break reference cycles that would otherwise leak memory.
- Recognize the `Deref` and `Drop` traits as the foundations of smart pointer behavior.
- Choose the right smart pointer for a given situation.

This module fills in concepts that were referenced earlier (Modules 6, 13, and 14) but not fully covered. After it, the family of smart pointers is no longer mysterious; it is a toolbox of specific tools for specific needs.

> **A note on when you will use these.** Most everyday Rust code uses very few smart pointers beyond the implicit ones in `Vec` and `String`. `Box` shows up occasionally for trait objects and recursive types. `Arc` is common in multi-threaded code. `Rc`, `RefCell`, `Cell`, and `Weak` are more specialized; they solve specific problems and you will reach for them when those problems appear. Knowing they exist is more important than using them often.

---

## A. `Box<T>`: Heap Allocation

`Box<T>` is the simplest smart pointer. It allocates a value on the heap and gives you a pointer to it.

### The Basics

```rust
fn main() {
    let value: Box<i32> = Box::new(42);
    println!("{value}");                // 42; the Box dereferences automatically
}
```

`Box::new(value)` allocates memory on the heap, moves the value into it, and returns a `Box<T>` that owns that allocation. When the box goes out of scope, the heap memory is freed.

For most code, you do not need `Box` for simple values. An `i32` lives fine on the stack, and putting it in a box just adds an allocation. The cases where `Box` is necessary involve specific patterns.

### Use Case 1: Recursive Types

A struct or enum cannot directly contain a value of itself, because the size would be infinite (a `T` contains a `T` contains a `T`...). Wrapping the recursive field in `Box` solves this:

```rust
// Does NOT compile:
enum LinkedList {
    Node(i32, LinkedList),       // ERROR: recursive type has infinite size
    Empty,
}

// Compiles:
enum LinkedList {
    Node(i32, Box<LinkedList>),  // OK: Box has known size (a pointer)
    Empty,
}
```

A `Box<LinkedList>` is just a pointer; its size is known (8 bytes on a 64-bit system). The actual `LinkedList` it points to lives on the heap. The compiler can compute the size of the enum because the pointer's size is fixed.

This pattern is common for tree structures, linked lists, and any other self-referential data.

### Use Case 2: Trait Objects

When you need to store values of different types behind a single trait, you use `Box<dyn Trait>`:

```rust
trait Animal {
    fn sound(&self) -> &str;
}

struct Dog;
impl Animal for Dog {
    fn sound(&self) -> &str { "woof" }
}

struct Cat;
impl Animal for Cat {
    fn sound(&self) -> &str { "meow" }
}

fn main() {
    let animals: Vec<Box<dyn Animal>> = vec![
        Box::new(Dog),
        Box::new(Cat),
    ];

    for animal in &animals {
        println!("{}", animal.sound());
    }
}
```

The vector holds `Dog` and `Cat` instances behind a uniform `Box<dyn Animal>` type. Each call to `sound()` goes through dynamic dispatch (Module 13 covered this).

You cannot put `Dog` and `Cat` directly in the same `Vec` because they have different types and sizes. The `Box<dyn Animal>` makes them uniform: each box has the same size (a pointer plus a vtable), and the actual values live on the heap.

### Use Case 3: Large Values

If you have a value that is large to copy or move, `Box` lets you keep the value on the heap and pass around just the pointer:

```rust
let big_data: Box<[u8; 1_000_000]> = Box::new([0u8; 1_000_000]);
let same_data = big_data;            // moves only the pointer, not the megabyte
```

Without the box, moving the array would copy a million bytes. With the box, only the pointer (8 bytes) moves; the data stays where it is. This is rarely needed in practice; vectors handle most large-data cases.

### `Box` and Ownership

`Box<T>` follows the standard ownership rules. There is exactly one owner; when it goes out of scope, the heap allocation is freed:

```rust
fn main() {
    let b = Box::new(42);
    let c = b;               // ownership moves; b is no longer valid
    // println!("{b}");      // ERROR: b was moved
    println!("{c}");         // OK
}                            // c goes out of scope; heap memory freed
```

This is the same as moving any other owned value. The box is a normal Rust value that happens to point to heap memory.

### The Cost

`Box::new` performs one heap allocation. Allocations are not free, but they are also not expensive in absolute terms (a few hundred nanoseconds on modern hardware). For occasional use, the cost is negligible. For tight loops that allocate many boxes, you might want to think about whether the allocations are necessary.

---

## B. `Rc<T>` and `Arc<T>`: Reference Counting

`Rc<T>` and `Arc<T>` provide shared ownership. Multiple variables can hold pointers to the same data; the data is freed when the last pointer is dropped.

### `Rc<T>`: Single-Threaded Reference Counting

```rust
use std::rc::Rc;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);

    let a = Rc::clone(&data);
    let b = Rc::clone(&data);

    println!("data: {data:?}");
    println!("a: {a:?}");
    println!("b: {b:?}");

    println!("reference count: {}", Rc::strong_count(&data));    // 3
}
```

`Rc::new` wraps a value, returning a reference-counted pointer. `Rc::clone` produces another pointer to the same data, incrementing the reference count. When all `Rc`s are dropped, the count reaches zero and the data is freed.

The "Rc" stands for "reference counted." It is single-threaded only: the reference count is updated with normal arithmetic, not atomic operations. Trying to send an `Rc` to another thread is rejected by the compiler:

```rust
let data = Rc::new(vec![1, 2, 3]);
std::thread::spawn(move || {
    println!("{data:?}");        // ERROR: Rc is not Send
});
```

For multi-threaded code, use `Arc` instead.

### `Rc::clone` vs `data.clone()`

A small but important convention: when cloning an `Rc`, write `Rc::clone(&data)` rather than `data.clone()`. Both work, but the explicit form makes it clear that you are cheaply cloning a pointer (incrementing a counter), not deeply cloning the data.

This matters because `data.clone()` works on any `Clone` type. If `data` were a `Vec` instead of an `Rc<Vec>`, the clone would duplicate the entire vector. The explicit `Rc::clone` syntax makes the intent unambiguous.

### Why Reference Counting?

Without `Rc`, you cannot have multiple owners of the same data. If two parts of your program both need to hold onto a value, you have three options:

- One owns it; the other borrows. This requires careful lifetime management.
- One owns it; the other clones it. This duplicates the data.
- Neither owns it; both reference it from somewhere else. This may not be feasible.

`Rc` adds a fourth option: shared ownership. Both parts hold an `Rc` to the same data. The data lives until both `Rc`s are dropped.

### `Arc<T>`: Atomic Reference Counting

`Arc<T>` is the thread-safe version of `Rc<T>`. The "A" stands for "atomic": the reference count is updated using atomic instructions, making it safe to share `Arc`s across threads.

You saw this in Module 14:

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);

    for i in 0..3 {
        let data = Arc::clone(&data);
        thread::spawn(move || {
            println!("thread {i}: {data:?}");
        });
    }
}
```

Arc has the same API as Rc but with atomic operations. The atomic operations are slightly slower than non-atomic ones (the cost is small but real), so you use `Rc` in single-threaded contexts and `Arc` only when you actually need cross-thread sharing.

### The Limit: Both Are Read-Only

Neither `Rc<T>` nor `Arc<T>` lets you modify the underlying value. Both give you shared, read-only access. If you want shared, mutable access, you combine them with the interior mutability types covered next: `Rc<RefCell<T>>` for single-threaded, `Arc<Mutex<T>>` for multi-threaded.

This is a deliberate split. Reference counting handles ownership; the interior mutability types handle synchronization. Combining them gives you both.

---

## C. `RefCell<T>` and the Interior Mutability Pattern

`RefCell<T>` lets you mutate a value through a shared reference. It moves Rust's borrow checking from compile time to runtime.

### The Problem

The borrow rules say: a value is either shared (multiple `&T`) or exclusively borrowed (one `&mut T`), never both at the same time. The compiler enforces this statically by tracking borrows in the source code.

Sometimes this is too restrictive. Consider a logger that needs to record events:

```rust
struct Logger {
    messages: Vec<String>,
}

impl Logger {
    fn log(&self, message: String) {       // takes &self (shared)
        self.messages.push(message);        // ERROR: cannot mutate through &self
    }
}
```

The method takes `&self` because logging conceptually does not change the logger from the caller's perspective. But internally, it modifies `messages`. The compiler rejects the code because you cannot mutate a field through a shared reference.

You could change the method to take `&mut self`, but that propagates: every method that calls `log` would also need `&mut self`, even if the caller does not conceptually mutate anything. The signature would lie about what the method does.

### The Solution: `RefCell`

`RefCell<T>` provides interior mutability: it lets you mutate the inner value through a shared reference to the cell. The borrow rules are still enforced, just at runtime instead of compile time.

```rust
use std::cell::RefCell;

struct Logger {
    messages: RefCell<Vec<String>>,
}

impl Logger {
    fn new() -> Logger {
        Logger { messages: RefCell::new(Vec::new()) }
    }

    fn log(&self, message: String) {       // still takes &self
        self.messages.borrow_mut().push(message);
    }

    fn print_all(&self) {
        for msg in self.messages.borrow().iter() {
            println!("{msg}");
        }
    }
}

fn main() {
    let logger = Logger::new();
    logger.log("first message".to_string());
    logger.log("second message".to_string());
    logger.print_all();
}
```

`RefCell::borrow_mut()` returns a mutable reference to the inner value, even though the cell itself was accessed through `&self`. The cell is "internally mutable": its inner value can change while it appears unchanged from the outside.

### Runtime Borrow Checking

`RefCell` enforces the borrow rules at runtime instead of compile time. The cell tracks active borrows. If you try to violate the rules, the program panics:

```rust
let cell = RefCell::new(5);

let r1 = cell.borrow();              // immutable borrow
let r2 = cell.borrow();              // another immutable borrow: OK
let r3 = cell.borrow_mut();          // PANIC: cannot borrow mutably while immutable borrows exist
```

The compile-time borrow checker would have rejected this. The runtime check catches it instead, with a panic at the moment the rules are violated.

This is a real trade-off. Compile-time checks are absolute; runtime checks can fail in production if a code path you did not test triggers a borrow conflict. The benefit is that some valid patterns (like the logger above) are accepted that the compile-time checker would reject.

### `borrow()` vs `borrow_mut()`

The two methods return different types:

```rust
let cell = RefCell::new(5);

let r: std::cell::Ref<i32> = cell.borrow();          // shared borrow
let r: std::cell::RefMut<i32> = cell.borrow_mut();    // exclusive borrow
```

`Ref` and `RefMut` are smart pointer guards. As long as a `Ref` exists, the cell is immutably borrowed; as long as a `RefMut` exists, it is exclusively borrowed. When the guards are dropped, the borrow ends.

This is the same pattern as `MutexGuard` from Module 14. The guard's scope determines the borrow's scope.

### When to Use `RefCell`

`RefCell` is appropriate when:

- You have a logical contract that takes `&self` but needs to mutate internal state.
- You are implementing a pattern that the compile-time borrow checker rejects but is genuinely safe.
- You can verify (by reasoning or testing) that runtime borrow conflicts will not happen.

`RefCell` is NOT a workaround to use whenever the compiler complains. The compiler is usually right. Reach for `RefCell` only when the design genuinely calls for interior mutability.

### `Rc<RefCell<T>>`: Shared Mutable State

The combination `Rc<RefCell<T>>` gives you shared, mutable, single-threaded data:

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

    let a = Rc::clone(&shared);
    let b = Rc::clone(&shared);

    a.borrow_mut().push(4);
    b.borrow_mut().push(5);

    println!("{:?}", shared.borrow());        // [1, 2, 3, 4, 5]
}
```

`Rc` provides shared ownership; `RefCell` provides shared mutation. Both `a` and `b` can modify the underlying vector through their shared pointer. The borrow checker, at runtime, ensures only one mutation happens at a time.

This is the single-threaded analogue to `Arc<Mutex<T>>` from Module 14.

---

## D. `Cell<T>` for Copy Types

`Cell<T>` is a simpler form of interior mutability for `Copy` types.

### How It Differs from `RefCell`

`RefCell` lets you take references to the inner value. `Cell` does not; it only lets you `get` and `set` the value:

```rust
use std::cell::Cell;

fn main() {
    let counter = Cell::new(0);

    counter.set(counter.get() + 1);
    counter.set(counter.get() + 1);
    counter.set(counter.get() + 1);

    println!("count: {}", counter.get());        // 3
}
```

`get()` returns a copy of the inner value (which is why `Cell` requires `Copy`). `set()` replaces the inner value with a new one. There is no concept of "borrowing" the cell's interior.

Because `Cell` returns copies, there is no possibility of borrow conflicts. Multiple readers cannot conflict with a writer because every reader gets its own copy. This means `Cell` has no runtime borrow check and no panics for borrow violations.

### When to Use `Cell` over `RefCell`

`Cell` is appropriate when:

- The inner type is `Copy` (integers, booleans, simple structs).
- You only need to read and write the value, not borrow into it.
- You want to avoid the runtime overhead of `RefCell`'s borrow tracking.

`RefCell` is necessary when:

- The inner type is not `Copy` (like `String` or `Vec`).
- You need to call methods on the inner value that take `&self` or `&mut self`.

For an integer counter or a boolean flag inside a struct that takes `&self` methods, `Cell` is the lightweight choice. For anything more complex, you usually need `RefCell`.

### A Practical Example

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<u32>,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: Cell::new(0) }
    }

    fn increment(&self) {                // takes &self, but mutates internally
        self.count.set(self.count.get() + 1);
    }

    fn value(&self) -> u32 {
        self.count.get()
    }
}

fn main() {
    let counter = Counter::new();
    counter.increment();
    counter.increment();
    counter.increment();
    println!("{}", counter.value());     // 3
}
```

The `Counter` exposes `&self` methods because, from the caller's perspective, incrementing does not violate any invariant of the counter; it just bumps a number. `Cell` makes the implementation possible without changing the public API.

---

## E. `Weak<T>` References and Preventing Cycles

`Rc` and `Arc` are great for shared ownership but have a known weakness: if two values hold `Rc`s to each other (a cycle), neither will ever be dropped, even after all external references are gone. This is a memory leak.

### How Cycles Happen

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
    value: i32,
    next: RefCell<Option<Rc<Node>>>,
}

fn main() {
    let a = Rc::new(Node { value: 1, next: RefCell::new(None) });
    let b = Rc::new(Node { value: 2, next: RefCell::new(None) });

    *a.next.borrow_mut() = Some(Rc::clone(&b));
    *b.next.borrow_mut() = Some(Rc::clone(&a));      // cycle: b -> a -> b -> ...
}
```

After this, `a` holds an `Rc` to `b`, and `b` holds an `Rc` to `a`. Even when `a` and `b` go out of scope, their reference counts are still 1 (each held by the other). The reference counts never reach zero, so the data is never freed. The memory leaks.

This is the classic problem with reference counting: it cannot collect cycles. Garbage collectors handle cycles automatically; reference counting does not.

### `Weak<T>`: Non-Owning References

`Weak<T>` is a reference-counted pointer that does NOT contribute to the reference count. It points to data owned by an `Rc` (or `Arc`) but does not keep that data alive on its own.

```rust
use std::rc::{Rc, Weak};

fn main() {
    let strong = Rc::new(42);
    let weak: Weak<i32> = Rc::downgrade(&strong);

    println!("strong count: {}", Rc::strong_count(&strong));        // 1
    println!("weak count: {}", Rc::weak_count(&strong));            // 1

    // Use the weak reference: try to get back to a strong one
    if let Some(value) = weak.upgrade() {
        println!("value: {value}");          // 42
    }

    drop(strong);

    // After the strong reference is dropped, the data is gone
    if weak.upgrade().is_none() {
        println!("data was already freed");
    }
}
```

`Rc::downgrade(&rc)` produces a `Weak`. `weak.upgrade()` tries to get a strong `Rc` back; it returns `Option<Rc<T>>`, with `Some` if the data still exists and `None` if it has been freed.

The `Weak` does not keep the data alive. When all strong references are dropped, the data is freed regardless of how many weak references exist.

### Breaking Cycles with `Weak`

The pattern: when you have a parent-child or A-references-B-references-A relationship, one direction uses `Rc` (strong, owning) and the other uses `Weak` (weak, non-owning):

```rust
struct Tree {
    value: i32,
    parent: RefCell<Weak<Tree>>,        // weak: parent does not keep child alive
    children: RefCell<Vec<Rc<Tree>>>,    // strong: parent owns children
}
```

The child has a `Weak` reference to its parent. The parent has `Rc` references to its children. When the root tree is dropped, the children are freed (their `Rc` count reaches zero), and any weak references the children held are now invalidated. No cycle, no leak.

The choice of which direction is weak depends on the semantics:

- **Parent owns children**: parent has `Rc<Child>`, child has `Weak<Parent>`.
- **Children own parent**: parent has `Weak<Child>`, child has `Rc<Parent>`.

The owning direction is whichever side conceptually "owns" the lifetime. For a tree, the parent owns the children: when the parent is gone, the children should go too.

### When You Need This

For most data structures, cycles do not arise naturally. Trees with parent pointers, doubly-linked lists, observer patterns, and graph structures are the common cases. If you are not building one of these, you probably do not need `Weak`.

If you do build a structure with potential cycles, the `Rc + Weak` (or `Arc + Weak` for multi-threaded) pattern is the standard solution. The alternative, an arena allocator that owns all nodes by index rather than pointer, is sometimes simpler for graph-like data; the `slotmap` and `petgraph` crates provide such structures.

---

## F. `Deref` and `Drop` Traits

Smart pointers work because of two traits: `Deref` (which makes a type act like a pointer) and `Drop` (which runs cleanup code when a value goes out of scope).

### `Deref`: Making a Type Act Like a Pointer

The `Deref` trait lets you customize what `*` does on a type:

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
    println!("{}", *x);                  // dereferences to 5
}
```

`Deref::deref` returns a reference to the inner value. The `*` operator calls this method and gives you the inner value back.

This is what makes `Box`, `Rc`, `Arc`, `String`, and `Vec` feel like the things they wrap. When you write `s.len()` on a `String`, the dot operator implicitly calls `*` to get to the underlying `str`, which has the `len` method. The whole thing works because `String` implements `Deref<Target = str>`.

### Deref Coercion

A related feature: the compiler automatically converts between `&SomeType` and `&Target` when needed:

```rust
fn print_str(s: &str) {
    println!("{s}");
}

fn main() {
    let owned = String::from("hello");
    print_str(&owned);                   // &String coerced to &str
}
```

The function expects `&str`. We pass `&String`. The compiler sees that `String` implements `Deref<Target = str>` and inserts a deref to convert. This is called "deref coercion" and is what makes string parameter types so flexible: you can pass `&str`, `&String`, `&Box<str>`, or `&Box<String>` interchangeably.

The same applies to `Vec` and slices: a `&Vec<T>` coerces to `&[T]`, which is why functions take `&[T]` parameters and accept vectors.

### `Drop`: Custom Cleanup

The `Drop` trait runs code when a value goes out of scope:

```rust
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("releasing {}", self.name);
    }
}

fn main() {
    let r1 = Resource { name: "first".to_string() };
    {
        let r2 = Resource { name: "second".to_string() };
        println!("inside block");
    }                                    // r2 goes out of scope here
    println!("outside block");
}                                        // r1 goes out of scope here
```

Output:

```
inside block
releasing second
outside block
releasing first
```

The `drop` method runs automatically when the value goes out of scope. The order is reverse-declaration: values are dropped in the opposite order from which they were created.

### When `Drop` Is Used

`Drop` is essential for types that hold resources beyond memory:

- File handles: close the file when the handle is dropped.
- Database connections: close the connection.
- Locks (`MutexGuard`): release the lock.
- Network sockets: close the socket.
- Reference-counted pointers: decrement the count.

The standard library types implement `Drop` for you. You implement it yourself when you write a type that wraps an OS resource or has some custom cleanup logic.

### `Drop` Cannot Be Called Manually

A subtle rule: you cannot call `drop()` on a value yourself. Rust runs it automatically and prevents you from running it twice:

```rust
let r = Resource { name: "test".to_string() };
r.drop();                                // ERROR: cannot call drop directly
```

If you really need to drop a value early (release a lock, close a file before scope ends), use the `std::mem::drop` function:

```rust
use std::mem;

let r = Resource { name: "test".to_string() };
mem::drop(r);                            // OK: takes the value, calls Drop on it
```

`mem::drop` takes the value by value (consuming it), then lets it go out of scope inside the function, which triggers `Drop`. This is the safe way to "drop early."

---

## G. When to Use Which Smart Pointer

A summary of when each pointer is appropriate.

### Decision Guide

| Need                                              | Use                       |
|---------------------------------------------------|---------------------------|
| Just put a value on the heap                      | `Box<T>`                  |
| Recursive type (linked list, tree)                | `Box<T>` for the recursion|
| Trait object (multiple types behind one trait)    | `Box<dyn Trait>`          |
| Shared ownership, single-threaded                 | `Rc<T>`                   |
| Shared ownership, multi-threaded                  | `Arc<T>`                  |
| Mutate through `&self`, single-threaded, non-Copy | `RefCell<T>`              |
| Mutate through `&self`, single-threaded, Copy     | `Cell<T>`                 |
| Mutate through `&self`, multi-threaded            | `Mutex<T>` (Module 14)    |
| Shared, mutable, single-threaded                  | `Rc<RefCell<T>>`          |
| Shared, mutable, multi-threaded                   | `Arc<Mutex<T>>`           |
| Avoid a reference cycle                           | `Weak<T>`                 |

### A Mental Algorithm

When deciding what pointer to use, walk through these questions:

1. **Do you need shared ownership?** If only one place owns the data, use a regular owned value (or `Box` if you specifically need the heap). If multiple places need ownership, you need `Rc` or `Arc`.

2. **Is the code multi-threaded?** If yes, use the `Arc` family (`Arc`, `Mutex`, `RwLock`). If no, use the `Rc` family (`Rc`, `RefCell`, `Cell`). Mixing them does not work; the compiler will reject `Rc` in multi-threaded contexts.

3. **Do you need to mutate the shared data?** If only reading, plain `Rc` or `Arc` is enough. If mutating, you need interior mutability (`RefCell` for single-threaded, `Mutex` for multi-threaded).

4. **Is the inner type `Copy`?** For simple `Copy` types in single-threaded code, `Cell` is lighter than `RefCell`. For non-`Copy` types, you need `RefCell`.

5. **Can the structure form a cycle?** If yes, identify which direction should be non-owning and use `Weak` for that direction.

### Common Combinations

A few patterns that come up enough to memorize:

- **`Rc<RefCell<T>>`**: shared, mutable data in single-threaded code. The classic "graph node" or "linked tree" pattern.
- **`Arc<Mutex<T>>`**: shared, mutable data across threads. The classic "concurrent counter" or "shared resource" pattern.
- **`Arc<RwLock<T>>`**: shared, mutable data across threads with many readers and few writers. Better than `Arc<Mutex<T>>` for read-heavy workloads.
- **`Box<dyn Trait>`**: returning or storing a trait object when the concrete type is determined at runtime.
- **`Vec<Box<dyn Trait>>`**: a heterogeneous collection of items behind a common trait.

### A Warning About Overuse

It is easy to reach for smart pointers when the basic ownership rules are inconvenient. This is usually the wrong move. Most code does not need `Rc` or `RefCell`; the borrow checker is right that the design is unsafe or unnecessary. Cloning data, restructuring code, or passing references explicitly are usually better solutions.

Smart pointers are tools for specific problems: shared ownership when the design genuinely calls for it, interior mutability when an API contract requires `&self`, recursive or polymorphic types that need indirection. Used judiciously, they extend Rust's capabilities. Used reflexively, they obscure the design and add overhead.

The Rust convention is "start without smart pointers, add them when you need them." The compiler will tell you when the basic rules are insufficient. At that point, the right smart pointer is usually obvious from the situation.

---

## Module Summary

- `Box<T>` puts a value on the heap. Used for recursive types, trait objects, and large values.
- `Rc<T>` provides shared ownership in single-threaded code. `Arc<T>` does the same across threads.
- `RefCell<T>` provides interior mutability with runtime borrow checking. It lets you mutate through a shared reference, panicking if the borrow rules are violated.
- `Cell<T>` is a simpler form of interior mutability for `Copy` types. It exposes only `get` and `set`, with no possibility of borrow conflicts.
- `Weak<T>` is a non-owning reference that does not contribute to the count. Used to break cycles in data structures.
- The `Deref` trait makes types act like pointers, including the deref coercion that converts `&String` to `&str` automatically.
- The `Drop` trait runs cleanup code when a value goes out of scope. It is the foundation of resource management.
- The right smart pointer depends on what you need: ownership pattern, mutation needs, threading, and whether cycles are possible.

## Discussion Questions

1. `RefCell<T>` moves borrow checking from compile time to runtime. The compile-time check is absolute (caught before deployment), but the runtime check can panic in production. Identify a situation where the trade-off is worth it, and one where it would be a bad choice.
2. Reference counting (`Rc<T>` and `Arc<T>`) has a fundamental limitation: it cannot collect cycles. Garbage-collected languages handle this automatically. What does Rust gain by using reference counting instead of garbage collection, and what is the cost in terms of programmer responsibility?
3. The `Drop` trait runs automatically when values go out of scope. This is similar to C++ destructors and unlike Java finalizers (which are unreliable). Why is the deterministic, scope-based cleanup model so important for resource management? What kinds of bugs does it prevent?
