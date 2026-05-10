# Module 6: Ownership and Borrowing

## Module Overview

Ownership is the central feature of Rust. It is what makes the language safe without a garbage collector and fast without a runtime. Every value in a Rust program has exactly one owner; when the owner goes out of scope, the value is destroyed. Borrowing lets you access a value without taking ownership, with rules that prevent both memory unsafety and data races.

This is the most distinctive part of Rust, and it is the part that takes the longest to internalize. Most students arrive with one of two backgrounds:

- They come from C or C++, where memory management is manual and bugs are common.
- They come from Java, Python, JavaScript, Go, or another garbage-collected language, where memory management is automatic and rarely visible.

Rust offers a third path: memory management is automatic, but the rules are visible in the source code and checked at compile time. There is no garbage collector. There is no manual `free`. The compiler tracks ownership statically and inserts the right cleanup code automatically.

By the end of this module, students will:

- State the three ownership rules and explain why each one exists.
- Recognize when a value is moved from one binding to another and when it is copied.
- Distinguish `Clone` from `Copy` and choose the right one for a given type.
- Use references (`&` and `&mut`) to pass data without transferring ownership.
- Apply the borrow rules: any number of immutable borrows OR exactly one mutable borrow at a time.
- Read borrow checker errors and understand what they are telling you.
- Use slices to refer to contiguous regions of arrays, vectors, and strings.

This module is dense and the rules will not feel natural at first. The second half of the course (and most of your future Rust code) builds on these rules. Students who internalize them now will find later modules much smoother.

> **A note on the borrow checker.** The borrow checker is the compiler component that enforces all the rules in this module. It is not a separate tool; it runs every time you compile. Errors from it can be intimidating at first because they often involve unfamiliar terminology (lifetimes, moves, borrows). The error messages are unusually good, and they almost always tell you what to do. Read them carefully. Most students go through a phase where they fight the borrow checker, then a phase where they just write code and the borrow checker stays out of their way.

---

## A. The Ownership Model: Rules and Motivation

### The Problem Ownership Solves

Memory in a running program comes from one of two places: the stack or the heap.

The **stack** is fast and predictable. Values are pushed when they are needed and popped when their scope ends. The size of every stack value must be known at compile time. Integer types, booleans, characters, fixed-size arrays, and references are all stack-allocated.

The **heap** is slower and more flexible. The program asks the allocator for a block of memory, gets back a pointer, and is responsible for releasing the memory when finished. Heap-allocated values can be any size, can grow, and can outlive the function that created them. `String`, `Vec`, `HashMap`, and `Box` are all heap-allocated.

Heap allocation introduces a difficult question: when is each piece of heap memory no longer needed?

Three categories of bugs come from getting this wrong:

- **Memory leaks**: memory is allocated but never freed. The program slowly consumes more memory until it crashes or the operating system kills it.
- **Use-after-free**: memory is freed but the program continues to use a pointer to it. The contents are now garbage, and reading or writing through the pointer corrupts unrelated data.
- **Double-free**: memory is freed twice. The allocator's internal data structures are corrupted, often leading to crashes much later in the program.

C and C++ handle this with manual `malloc` and `free`. Memory bugs are pervasive: Microsoft and Google independently report that around 70 percent of their security vulnerabilities are memory safety issues in C and C++ code.

Garbage-collected languages handle this with a runtime that tracks references and frees memory automatically. The cost is performance overhead, GC pauses, and the inability to use the language in contexts where a runtime is unavailable (kernels, embedded systems, very-low-latency code).

Rust handles this with ownership: a compile-time system that tracks who owns each piece of memory and automatically inserts cleanup code when the owner goes out of scope. The rules are stricter than what most languages enforce, but the compiler verifies them, so the entire class of memory bugs is impossible in safe Rust.

### The Three Rules

The ownership system is governed by three rules:

1. **Every value has a single owner.**
2. **There can only be one owner at a time.**
3. **When the owner goes out of scope, the value is dropped.**

These rules apply to every value in the program. They are checked at compile time. If your code violates them, it does not compile.

### Rules in Action

```rust
fn main() {
    let s = String::from("hello");      // s is the owner of the String
    println!("{s}");
}                                        // s goes out of scope; the String is dropped
```

The `String` is allocated on the heap when `String::from` is called. The variable `s` owns it. When `main` returns, `s` goes out of scope and Rust automatically calls the `String`'s destructor (called `Drop` in Rust terminology), which releases the heap memory.

There is no `free`. There is no garbage collector. The compiler inserts the cleanup code at the right place because it can see exactly when `s`'s scope ends.

### Why "Single Owner"?

The single-owner rule is what prevents double-frees. If two variables owned the same value, both would try to drop it, and you would have a double-free. By forbidding shared ownership at the basic level, the rule eliminates the possibility entirely.

Shared ownership is still possible in Rust through specific types like `Rc` and `Arc` (covered in a later module), which use reference counting. But these are explicit opt-ins; the default is single ownership.

### Scope and Drop

Scope in Rust is determined by braces. A value's owner is in scope from the line of declaration to the closing brace of the enclosing block.

```rust
fn main() {
    {
        let s = String::from("hello");
        println!("{s}");
    }                                    // s goes out of scope here; String is dropped
    
    // s is no longer accessible here
}
```

The block `{ ... }` introduces a new scope. When the block ends, every value owned within it is dropped. This is identical in spirit to C++ destructors and RAII; in fact, Rust's ownership system was directly influenced by RAII.

### A Working Mental Model

For the rest of this module, hold this picture in mind:

- A binding (`let x = ...`) owns the value it points to.
- The compiler tracks which binding owns which value.
- When a binding goes out of scope, its value is destroyed.
- Operations like assignment, function calls, and returns either transfer ownership (move) or borrow (with `&` or `&mut`).
- The compiler verifies at every step that the rules hold.

The next sections cover what these operations look like in code.

---

## B. Move Semantics

### The Problem

Consider this code that would work in many languages:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
    println!("{s1}");      // ERROR
    println!("{s2}");
}
```

The first `println!` does not compile. The compiler says:

```
error[E0382]: borrow of moved value: `s1`
  |
  | let s1 = String::from("hello");
  |     -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
  | let s2 = s1;
  |          -- value moved here
  | println!("{s1}");
  |           ^^ value borrowed here after move
```

The assignment `let s2 = s1` did not copy the string. It moved it. After the move, `s1` is no longer valid.

### What Happens in Memory

A `String` is a small struct on the stack with three fields: a pointer to the heap-allocated data, a length, and a capacity. When you assign a `String` to a new binding, the simple operation copies these three fields.

```
After let s1 = String::from("hello"):

    s1: [ptr | len: 5 | cap: 5]
            |
            v
            heap: ['h', 'e', 'l', 'l', 'o']

After let s2 = s1:

    s1: [ptr | len: 5 | cap: 5]      <- pointer still here, but binding is "moved"
    s2: [ptr | len: 5 | cap: 5]      <- same heap data
            |                |
            +----------------+
                    v
                    heap: ['h', 'e', 'l', 'l', 'o']
```

Now both stack records point to the same heap data. If both `s1` and `s2` were valid, when `s1` went out of scope, it would free the heap memory. When `s2` went out of scope, it would try to free the already-freed memory: a double-free.

Rust prevents this by treating the assignment as a move. Conceptually, `s1` no longer owns the data; `s2` does. The compiler refuses any attempt to use `s1` after the move.

### The Rule

When a value is assigned to a new binding, passed to a function, or returned from a function, ownership moves. After the move, the original binding is no longer valid.

```rust
fn takes_ownership(s: String) {
    println!("{s}");
}                                    // s goes out of scope; String dropped here

fn main() {
    let s = String::from("hello");
    takes_ownership(s);
    
    // println!("{s}");              // ERROR: s was moved into takes_ownership
}
```

The function `takes_ownership` takes its parameter by value, which means ownership transfers when the function is called. After the call, `s` is no longer valid in `main`.

### Returning Values

Functions can transfer ownership back to the caller via the return value:

```rust
fn make_string() -> String {
    let s = String::from("hello");
    s                                // ownership moves to the caller
}

fn main() {
    let owned = make_string();
    println!("{owned}");
}
```

The string created in `make_string` would normally be dropped when the function returns. But because `s` is the return value, ownership transfers to the caller, and the value is not dropped. Now `owned` is the new owner.

### Why This Is Reasonable

Move semantics may feel restrictive, but it produces predictable, fast code:

- **No hidden cost.** Assignment is a few stack-pointer copies. There is no implicit deep copy of heap data.
- **No double-free.** Only one binding owns a value at any time.
- **No use-after-free.** The compiler refuses to compile code that uses a moved value.

Compare this to C++, where a similar assignment makes a deep copy by default (slow), and to Python or Java, where every variable is a pointer with shared ownership tracked by the garbage collector (overhead).

Rust's choice is "move by default, copy or clone explicitly when you need to." The cost of each operation is visible in the source.

---

## C. Clone and Copy Traits

Move semantics applies to all types, but there are two categories of types that have special behavior: `Copy` types (which are duplicated implicitly) and `Clone` types (which can be duplicated explicitly).

> **Note:** The full machinery of traits is covered in Module 9. For this section, treat `Copy` and `Clone` as labels that tell you how the type behaves: `Copy` types are duplicated automatically; `Clone` types can be duplicated by calling `.clone()`.

### `Copy`: Implicit Duplication for Stack Values

Some types are so simple that copying them is just as cheap as moving them. Integer types, floating-point types, booleans, characters, and arrays of `Copy` types all fit this description. They are entirely on the stack with no heap data.

For these types, the compiler treats assignment as a copy, not a move:

```rust
fn main() {
    let x = 5;
    let y = x;
    println!("x = {x}, y = {y}");        // both work: 5, 5
}
```

The `Copy` types are:

- All integer types (`i8`, `i32`, `u64`, etc.)
- Floating-point types (`f32`, `f64`)
- `bool`
- `char`
- Tuples of `Copy` types
- Arrays of `Copy` types
- Shared references (`&T`, where `T` is any type)

A type is `Copy` if it can be duplicated by simply copying its bits. Anything that owns heap memory cannot be `Copy`, because duplicating the bits would mean two owners of the same heap allocation, which violates the ownership rules.

Function parameters and return values follow the same rule. A function that takes an `i32` does not consume the caller's value:

```rust
fn double(x: i32) -> i32 {
    x * 2
}

fn main() {
    let n = 5;
    let result = double(n);
    println!("n = {n}, result = {result}");      // both work
}
```

The `n` is copied into `double`. The original `n` in `main` is unaffected.

### `Clone`: Explicit Duplication for Anything

Some types own heap memory and cannot be `Copy`. The most common are `String`, `Vec`, `HashMap`, `Box`, and any user-defined struct that contains them.

For these types, you can request a duplicate by calling `.clone()`:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();
    
    println!("s1 = {s1}");        // works: s1 still owns its String
    println!("s2 = {s2}");        // works: s2 owns a separate copy
}
```

The `.clone()` method allocates new heap memory, copies the data, and produces an independent owned value. Both `s1` and `s2` are now valid and independent.

Cloning is **not** free. For a `String`, it allocates and copies every byte. For a `Vec<i32>` with a million elements, it allocates and copies four million bytes. The visible `.clone()` call makes this cost explicit.

### When to Clone

Cloning is the right answer when:

- You need two independent copies of the data.
- The cost is acceptable for your use case.
- You have already considered borrowing instead.

Cloning is the wrong answer when:

- The data is large and you only need to read it (use a reference instead).
- You are calling `.clone()` reflexively to silence the compiler (this is a code smell).

A common antipattern is calling `.clone()` everywhere to get rid of borrow checker errors. This works but defeats the purpose of using Rust. The right approach is usually to add references and let the borrow checker prove that no clone is needed. The next sections show how.

### A Side-by-Side Comparison

```rust
// Copy: implicit duplication
let a = 5;
let b = a;
println!("a = {a}, b = {b}");        // both valid

// Move: implicit ownership transfer (no Copy)
let s1 = String::from("hello");
let s2 = s1;
// println!("{s1}");                 // ERROR: s1 was moved

// Clone: explicit duplication
let s3 = String::from("world");
let s4 = s3.clone();
println!("s3 = {s3}, s4 = {s4}");    // both valid
```

The three cases form a hierarchy: `Copy` for cheap stack values (implicit), move for owned heap values (default), `Clone` for heap values when you genuinely need a duplicate (explicit).

---

## D. References and Borrowing

Move semantics and cloning both transfer or duplicate ownership. Sometimes you need to access a value without doing either: just look at it (or modify it) without taking it. References solve this problem.

### Creating a Reference

A reference is created with the `&` operator. Its type is `&T` for some type `T`:

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;
    
    println!("s = {s}");        // s still works
    println!("r = {r}");        // r works too
}
```

The variable `r` is a reference to the `String` owned by `s`. The reference does not own the `String`; it just points to it. Both `s` and `r` can be used.

### What Borrowing Means

Creating a reference is called "borrowing." The reference holder borrows access to the value from the owner. The borrow is temporary; the owner retains ownership and the borrowed value will not be dropped while a reference to it exists.

In the example above:

- `s` owns the `String`.
- `r` borrows from `s`.
- When `r` goes out of scope, the borrow ends.
- When `s` goes out of scope, the `String` is dropped (no one was preventing that).

The owner is responsible for the value's lifetime. References are second-class: they exist as long as the owner does.

### Functions That Borrow

Functions can take parameters by reference, which is how you read data without taking ownership:

```rust
fn print_length(s: &String) {
    println!("length: {}", s.len());
}

fn main() {
    let s = String::from("hello");
    print_length(&s);
    print_length(&s);             // can borrow as many times as we want
    println!("s = {s}");          // s is still owned by main
}
```

The function takes `&String` (a reference to a `String`) instead of `String` (an owned `String`). Calling the function with `&s` creates a temporary reference; the reference goes out of scope when the function returns. The original `String` is unaffected.

This is the most common way functions accept data they only need to read. You will see it constantly in idiomatic Rust.

### Why Functions Should Usually Take References

Compare two ways of writing a length-printing function:

```rust
// Takes ownership:
fn print_length_owned(s: String) {
    println!("length: {}", s.len());
}

// Borrows:
fn print_length_borrowed(s: &String) {
    println!("length: {}", s.len());
}
```

The first form takes ownership. After calling it, the caller can no longer use the original `String`. This is rarely what you want for a function that just reads.

The second form borrows. The caller retains ownership and can use the `String` again. This is almost always the right form for a function that only needs to read.

The convention in idiomatic Rust: **take references unless you specifically need to take ownership.** The exception is when the function consumes the value (stores it somewhere, transforms it into something else, or moves it elsewhere).

### Reference Types

There are two kinds of references in Rust:

- **`&T`**: a shared (immutable) reference. You can read but not modify.
- **`&mut T`**: a mutable reference. You can read and modify.

The next section covers mutable references in detail. Most references you create are immutable.

### Method Calls Often Borrow Implicitly

When you call a method on a value, Rust often takes a reference automatically:

```rust
let s = String::from("hello");
let length = s.len();             // s.len() takes &self under the hood
println!("{s}");                   // s is still valid
```

The `len` method's signature is `fn len(&self) -> usize`. It takes a reference, not ownership. The dot-syntax handles the reference creation transparently.

This is one reason method calls feel natural even though Rust has explicit references: most methods borrow rather than consume.

---

## E. Mutable References and Borrow Rules

A mutable reference (`&mut T`) lets you modify the value through the reference. The borrow checker enforces specific rules about when mutable references can exist.

### Creating a Mutable Reference

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r = &mut s;
    r.push_str(", world");
    
    println!("{r}");
}
```

Three conditions are needed:

- The original binding must be mutable (`let mut s`).
- The reference must be created with `&mut`.
- The reference itself can be used to modify the value, including calling methods that mutate.

### The Borrow Rules

The borrow checker enforces two rules. They are simple to state and consequential in practice.

**Rule 1: At any time, you can have either:**
- **Any number of immutable references (`&T`)**, OR
- **Exactly one mutable reference (`&mut T`)**.

You cannot mix the two. While a mutable reference exists, no other references (mutable or immutable) can exist to the same data. While any immutable references exist, no mutable reference can exist.

**Rule 2: References must always be valid.** A reference cannot outlive the value it points to. (Covered in Section F.)

### Why These Rules?

Rule 1 prevents data races at the language level. A data race needs three ingredients:

- Two or more pointers to the same data.
- At least one of them is a write.
- The accesses are not synchronized.

The borrow rules eliminate the first two together: if writes can happen (a mutable reference exists), then no other access can happen (no other references allowed). Rust's compiler enforces this statically. Data races, which plague C and C++ programs and require runtime tools to detect, are simply impossible in safe Rust.

The same rules also prevent more subtle bugs that arise even in single-threaded code. Consider:

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];          // immutable reference into v
v.push(4);                   // modifies v, possibly reallocating
println!("{first}");        // would be reading freed memory
```

If the borrow rules allowed this, `first` could be a dangling pointer after `v.push(4)` reallocates the vector's backing storage. The rules prevent this by refusing to compile: the immutable reference and the mutating call cannot coexist.

### A Simple Demonstration

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &s;              // immutable borrow
    let r2 = &s;              // another immutable borrow: OK
    println!("{r1}, {r2}");
    
    // Now r1 and r2 are no longer used; their borrows end here.
    
    let r3 = &mut s;          // mutable borrow: OK because no other borrows are active
    r3.push_str(" world");
    println!("{r3}");
}
```

This works because the immutable borrows (`r1`, `r2`) end before the mutable borrow (`r3`) begins. The compiler tracks where each borrow is actually used and ends the borrow at its last use, not at the end of the scope. This rule (called non-lexical lifetimes) makes the borrow checker much more flexible than the simple "borrow lasts until end of scope" model would suggest.

### A Failing Case

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &s;
    let r2 = &mut s;          // ERROR: cannot borrow as mutable while r1 exists
    
    println!("{r1}, {r2}");
}
```

The compiler rejects this with:

```
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
```

The immutable borrow `r1` is used after the mutable borrow attempt, so its borrow span overlaps with the mutable one. The rules forbid overlap.

### Reading the Errors

Borrow checker errors are unusually informative. They typically:

- Identify the conflicting borrows by line.
- Explain which borrow is which.
- Sometimes suggest a fix (such as cloning, or restructuring the code).

Read every diagnostic in full. The message almost always tells you what to do.

### A Common Pattern: Sequential Borrows

When you need both reads and writes, the typical pattern is to do all the reading first, then do the writing:

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    
    // Read phase: collect what we need
    let first = v[0];
    let last = v[v.len() - 1];
    
    // Write phase: now v is free for modification
    v.push(first + last);
    
    println!("{v:?}");        // [1, 2, 3, 4, 5, 6]
}
```

By copying the values we need (`v[0]` and `v[v.len() - 1]` are `i32`, which is `Copy`), the borrows end immediately. The mutating `push` can then proceed without conflict.

If the values were not `Copy`, you would clone them or restructure the code. Sometimes the right answer is to use a different data structure entirely.

---

## F. Dangling References and the Borrow Checker

A dangling reference is a reference that points to memory that has already been freed. In C, dangling pointers are a leading source of bugs:

```c
char* dangling() {
    char buffer[10] = "hello";
    return buffer;       // returning a pointer to stack memory that is about to be freed
}
```

The function returns a pointer to a local buffer. When the function returns, the buffer is freed, and the pointer is dangling. Reading or writing through it causes undefined behavior.

Rust prevents dangling references at compile time. The borrow checker tracks how long each reference must remain valid and refuses to compile code where a reference could outlive what it points to.

### A Failing Example

```rust
fn dangle() -> &String {
    let s = String::from("hello");
    &s                          // ERROR: returning a reference to local s
}
```

This does not compile. The compiler says:

```
error[E0106]: missing lifetime specifier
  |
  | fn dangle() -> &String {
  |                ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but
          there is no value for it to be borrowed from
```

The error message is technical but the intent is clear: the function tries to return a reference to a local that is about to be dropped. Rust will not allow it.

### The Fix: Return the Value, Not a Reference

The simplest fix is to return the owned value:

```rust
fn no_dangle() -> String {
    let s = String::from("hello");
    s                            // ownership moves to the caller
}
```

The `String` moves out of the function. The caller now owns it. There is no reference, so there is no possibility of a dangling reference.

### Lifetimes

The borrow checker uses a system called **lifetimes** to track how long each reference must remain valid. A lifetime is a compile-time region of code during which a reference is guaranteed to be valid.

Lifetimes are mostly invisible. The compiler infers them in the vast majority of cases. You only see explicit lifetime syntax in function signatures that involve multiple references with relationships between them. Lifetimes are the topic of Module 7; for this module, you do not need to write any explicit lifetime annotations.

The takeaway for now: the compiler tracks the validity of every reference. If a reference might outlive its target, the code does not compile. This guarantees that every reference in a running Rust program points to valid data.

### Why This Matters

The combination of the ownership rules and the borrow checker eliminates entire categories of bugs at compile time:

- **No use-after-free**: a reference cannot outlive its target.
- **No dangling pointers**: the borrow checker rejects them.
- **No data races in safe code**: the borrow rules prevent simultaneous reads and writes.
- **No double-frees**: the single-owner rule makes them impossible.
- **No memory leaks (in most cases)**: scope-based cleanup runs reliably.

The price is that you have to think about ownership and borrowing as you write. The reward is that the compiler catches mistakes that other languages would discover only at runtime, often in production.

---

## G. Slices: String Slices and Array Slices

A slice is a reference to a contiguous range of elements. Slices are how you refer to part of a sequence without taking ownership and without copying data.

Two slice types appear constantly in Rust code:

- **`&[T]`**: a slice of a sequence of `T`. Used for arrays, vectors, and ranges of either.
- **`&str`**: a slice of a string. Used for parts of `String`s and for string literals.

### Array and Vector Slices

A slice of a sequence is created with the `&array[start..end]` syntax:

```rust
fn main() {
    let numbers = [10, 20, 30, 40, 50];
    
    let all: &[i32] = &numbers;              // whole array as slice
    let middle: &[i32] = &numbers[1..4];     // [20, 30, 40]
    let first_two: &[i32] = &numbers[..2];   // [10, 20]
    let last_three: &[i32] = &numbers[2..];  // [30, 40, 50]
    
    println!("all: {all:?}");
    println!("middle: {middle:?}");
    println!("first_two: {first_two:?}");
    println!("last_three: {last_three:?}");
}
```

The slice has three useful properties:

- It is a reference, so it does not take ownership.
- It carries a length, so out-of-bounds access is prevented at runtime.
- It does not copy the underlying data; it just refers to a range.

### Slices in Function Signatures

The most important use of slices is in function parameters. A function that accepts `&[i32]` can be called with arrays of any size, with `Vec<i32>`, and with subranges of either:

```rust
fn sum(values: &[i32]) -> i32 {
    let mut total = 0;
    for v in values {
        total += v;
    }
    total
}

fn main() {
    let array = [1, 2, 3, 4, 5];
    let vector = vec![10, 20, 30];
    
    println!("{}", sum(&array));            // 15
    println!("{}", sum(&vector));           // 60
    println!("{}", sum(&array[1..4]));      // 9
}
```

The function does not care whether the data lives in an array, a vector, or a subrange. It just needs a contiguous range of `i32` values to read.

This pattern is so common that you should default to `&[T]` for any function parameter that needs to read a sequence. Use `Vec<T>` only when the function needs to take ownership of the vector (because it will store, modify, or transform it).

### String Slices

Strings have their own slice type, `&str`. Like array slices, a string slice is a reference to a range of bytes within a `String` (or string literal).

```rust
fn main() {
    let s = String::from("hello world");
    
    let hello: &str = &s[0..5];
    let world: &str = &s[6..11];
    let entire: &str = &s;        // whole string as slice
    
    println!("{hello}");
    println!("{world}");
    println!("{entire}");
}
```

String slices are immutable references into the original string's data. They are the type used to share string data without copying.

### `&str` vs `String`

This is one of the most important distinctions in Rust:

- **`String`**: an owned, growable, heap-allocated string. Use this when you need to own and modify string data.
- **`&str`**: a borrowed string slice. Use this for function parameters, string literals, and any case where you only need to read.

String literals like `"hello"` have type `&'static str`: a slice of bytes baked into the program's binary. They are not `String`s.

The convention parallels that of slices generally: **take `&str` parameters unless you need ownership; return `String` (owned) when constructing new string data.**

### A Function That Takes `&str`

```rust
fn first_word(text: &str) -> &str {
    let bytes = text.as_bytes();
    
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &text[0..i];
        }
    }
    
    text
}

fn main() {
    let s = String::from("hello world");
    let first = first_word(&s);
    println!("first word: {first}");
    
    let literal = "hello rust";
    let first = first_word(literal);
    println!("first word: {first}");
}
```

The function returns a slice that references the input. The caller can pass either a `String` (with `&s`) or a `&str` directly (string literals work).

The interesting part is the relationship between the input and the output. The returned slice borrows from the input. As long as the input remains valid, the slice is valid. The borrow checker enforces this; you do not have to track it manually.

### UTF-8 and String Slicing

A `String` is stored as UTF-8, where one character may take one to four bytes. Slicing a string by byte position is allowed, but slicing in the middle of a multi-byte character causes a runtime panic:

```rust
let s = String::from("hello");
let valid = &s[0..2];        // "he", fine

let multibyte = String::from("héllo");
let invalid = &multibyte[0..2];      // PANIC: byte index 2 falls inside a character
```

The byte at position 2 of `"héllo"` is the second byte of the `é` character, not a character boundary. Slicing across a character is undefined for human readers and would produce invalid UTF-8, so Rust panics rather than allowing it.

For most common cases (slicing at known boundaries, slicing entire strings, slicing strings of pure ASCII), this is not an issue. When working with arbitrary text, use methods like `.chars()`, `.char_indices()`, or `.split()` to walk a string by character or by delimiter rather than by byte position.

### Why Slices Matter

Slices are the type that makes ownership work in practice. Without them, every function that wanted to read part of a sequence would either need to take ownership (and give it back) or take a more specific type. Slices let one function signature work with many concrete types.

The overall pattern in idiomatic Rust:

| If you need to...                          | Use            |
|--------------------------------------------|----------------|
| Read part of a sequence                    | `&[T]`         |
| Read part of a string                      | `&str`         |
| Read and modify a sequence                 | `&mut [T]`     |
| Take ownership of a sequence               | `Vec<T>` or `String` |
| Hold a fixed-size sequence on the stack    | `[T; N]`       |

References and slices are how Rust achieves the "single-owner with broad sharing" balance. Ownership is strict; borrowing is flexible. Most code spends its time borrowing.

---

## Module Summary

- Every value in Rust has exactly one owner. When the owner goes out of scope, the value is dropped automatically.
- Assigning a non-`Copy` value to a new binding moves ownership. The original binding is no longer valid.
- `Copy` types (small stack values) are duplicated implicitly. `Clone` types can be duplicated explicitly via `.clone()`, which performs a real copy of any heap data.
- References (`&T` and `&mut T`) let code access a value without taking ownership. The reference holder is said to "borrow" the value.
- The borrow rules: at any time, you can have any number of immutable references OR exactly one mutable reference. References must always point to valid data.
- The borrow checker enforces all of this at compile time. It rejects programs that would have memory bugs at runtime.
- Slices (`&[T]` and `&str`) are references to contiguous ranges. They are the idiomatic parameter type for functions that read sequences.

## Discussion Questions

1. The single-owner rule prevents double-frees but seems restrictive. Identify a code pattern that would be natural in C++ or Java but requires explicit cloning, borrowing, or restructuring in Rust. Is the cost worth the safety guarantee?
2. The borrow rules forbid simultaneous reads and writes. This prevents data races in concurrent code, but the rules apply even in single-threaded code. Identify a single-threaded bug that the rules prevent and explain why it is hard to find without compile-time enforcement.
3. Most function parameters in idiomatic Rust are references rather than owned values. What does this signal about how Rust thinks about data flow? Compare this to a language you know where parameters are typically passed by value (C, Python's primitive types) or by reference (Java's objects, Python's everything-else).

## Looking Ahead

Module 7 covers lifetimes, the syntax that makes the borrow checker's reasoning visible. You have already been using lifetimes in this module without writing them: every reference you created had a lifetime that the compiler inferred. Module 7 covers when the inference fails and how to write explicit lifetime annotations to help the compiler. After lifetimes, the rest of the language (generics, traits, error handling, concurrency) builds on the ownership and borrowing foundations you have now learned.
