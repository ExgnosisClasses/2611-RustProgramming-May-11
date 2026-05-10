# Modules 6 and 7: Ownership, Borrowing, and Lifetimes
## Lab 5 -- Working with Ownership, Borrows, and Lifetime Annotations

> **Course:** Mastering Rust
> **Modules:** 6 - Ownership and Borrowing, 7 - Lifetimes
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with the most distinctive features of Rust: the ownership system, borrowing, and lifetimes. You will work through every concept from Modules 6 and 7: move semantics, the `Copy` and `Clone` traits, immutable and mutable references, the borrow rules, slices, and the lifetime annotations that make function and struct signatures explicit.

You will build a single Cargo project called `text_analyzer` that processes text data using all of the techniques. The project deliberately produces compile errors at several points so you can read the borrow checker's diagnostics and fix them; this is one of the most important skills for everyday Rust programming.

The focus is on the mechanics of moving values, borrowing them, and expressing the relationships between references. Many exercises ask you to predict what will happen before running the code; reading the borrow checker's errors is essential to learning the language.

By the end of this lab you will be able to:

- Recognize when a value is moved versus copied and predict whether a binding is still valid.
- Choose between `Clone` and references for sharing data.
- Use immutable and mutable references with the borrow rules.
- Read borrow checker errors and identify what is wrong.
- Use slices (`&[T]` and `&str`) as flexible parameter types.
- Add lifetime annotations to functions that return references derived from inputs.
- Add lifetime annotations to structs that hold references.
- Recognize when the lifetime elision rules apply and when explicit annotations are required.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** This lab is the most challenging so far. The ownership system and borrow checker take time to internalize. Several exercises deliberately produce compile errors; the goal is to read the diagnostic, understand it, and fix the code. This is exactly what real Rust development feels like. Do not skip the failing examples; they are the most valuable learning material in the lab.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new text_analyzer
cd text_analyzer
code .
```

Create a `lab5-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Move Semantics

**Estimated time:** 10 minutes
**Topics covered:** ownership rules, move on assignment, move on function call, the moved value error

### Context

Module 6 introduced ownership: every value has exactly one owner; when the owner goes out of scope, the value is dropped. Assigning a non-`Copy` value to a new binding moves ownership. After a move, the original binding is no longer valid. This exercise establishes the mechanics through working and failing examples.

### Task 1.1 -- The basic move

Replace the contents of `src/main.rs` with:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;

    println!("s2: {s2}");
    // println!("s1: {s1}");      // we will uncomment this in a moment
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
s2: hello
```

The line `let s2 = s1;` moved the `String`. Now `s2` owns the heap data and `s1` is no longer valid.

### Task 1.2 -- Trigger the move error

Uncomment the second `println!`:

```rust
println!("s1: {s1}");
```

Run `cargo build`. The compiler rejects this with:

```
error[E0382]: borrow of moved value: `s1`
```

The error message is unusually informative. It tells you:
- That `s1` was moved (and where: in the assignment).
- That `String` does not implement `Copy`, which is why the assignment is a move rather than a copy.
- That the use of `s1` after the move is what is illegal.

In `lab5-notes.md`, copy the exact compiler error message for reference. You will see this error often as you learn Rust; recognizing it instantly is valuable.

Re-comment the bad line so the program compiles.

### Task 1.3 -- Move into a function

Add this function below `main`:

```rust
fn take_ownership(s: String) {
    println!("inside take_ownership: {s}");
}
```

And modify `main` to call it:

```rust
fn main() {
    let s = String::from("hello");
    take_ownership(s);

    // println!("s after call: {s}");      // would also fail
}
```

Run the program. It works. The `String` was moved into `take_ownership`, where it was used and then dropped at the end of the function.

If you uncomment the last `println!`, you get the same `borrow of moved value` error. Passing a value to a function (when the parameter takes it by value) is just as much a move as `let s2 = s1`.

### Task 1.4 -- Return ownership

Add this function:

```rust
fn make_string() -> String {
    let s = String::from("hello from inside");
    s
}
```

The function creates a `String` and returns it. The value moves out of the function to the caller:

```rust
fn main() {
    let s = make_string();
    println!("{s}");
}
```

The `String` was created inside the function but is owned by `s` in `main` after the return. Ownership transferred. The function did not leak its local; it just moved the value out.

### Task 1.5 -- Move with the function and back

Add this function:

```rust
fn process(s: String) -> String {
    println!("processing: {s}");
    s
}
```

This function takes ownership and returns it. From the caller:

```rust
fn main() {
    let s = String::from("hello");
    let s = process(s);                   // takes s, returns it; we shadow with the same name
    println!("after process: {s}");
}
```

This works. The `String` is moved into `process`, then moved back out, and we shadow the original `s` with the returned value.

This pattern (passing ownership in and out) is sometimes useful but is usually a code smell. References (next exercise) are almost always a better choice.

### Checkpoints

1. The move on `let s2 = s1;` does not actually copy any heap data. What is copied? What does `s1` look like in memory after the move (even though the compiler will not let you use it)?
2. Why does `String` move on assignment while `i32` does not? What property of the type controls this behavior?
3. The pattern in Task 1.5 (move in, move back out) is described as a code smell. Why? What does it usually mean about the function's design?

---

## Exercise 2 -- Clone vs Copy

**Estimated time:** 10 minutes
**Topics covered:** the `Copy` trait, the `Clone` trait, when each applies, the cost of cloning

### Context

Some types implement `Copy`, which means assignment duplicates them rather than moving them. Other types implement `Clone`, which provides explicit duplication via `.clone()`. Knowing which is which matters for performance and for predicting whether a value remains valid after use.

### Task 2.1 -- Copy types

Replace the contents of `src/main.rs` with:

```rust
fn main() {
    let x = 5;
    let y = x;

    println!("x = {x}, y = {y}");        // both work: 5, 5

    let a = true;
    let b = a;
    println!("a = {a}, b = {b}");

    let c = 'A';
    let d = c;
    println!("c = {c}, d = {d}");

    let pair = (1, 2.5);
    let pair2 = pair;
    println!("{pair:?}, {pair2:?}");
}
```

Run the program. Expected output:

```
x = 5, y = 5
a = true, b = true
c = A, d = A
(1, 2.5), (1, 2.5)
```

Each assignment copies the value rather than moving it. The originals are still valid because `i32`, `bool`, `char`, and tuples of `Copy` types all implement `Copy`.

### Task 2.2 -- Cloning a String

Replace `main` with:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();

    println!("s1: {s1}");        // works
    println!("s2: {s2}");        // works
}
```

Run the program. Both `String`s are valid because `s1.clone()` produced an independent copy. The original `s1` was not moved.

The `.clone()` call did real work: it allocated new heap memory and copied the five bytes of `"hello"` into it. Cloning is not free.

### Task 2.3 -- A custom struct with Copy

Add this struct definition above `main`:

```rust
#[derive(Debug, Copy, Clone)]
struct Point {
    x: f64,
    y: f64,
}
```

The `#[derive(Copy, Clone)]` attribute makes `Point` a `Copy` type. (Recall from Module 8 that `Copy` requires `Clone`; the two go together.) Use it in `main`:

```rust
fn main() {
    let p1 = Point { x: 3.0, y: 4.0 };
    let p2 = p1;

    println!("{p1:?}, {p2:?}");          // both work
}
```

Run the program. Both points are valid because `Point` implements `Copy`. The assignment was a bitwise copy.

### Task 2.4 -- A struct that cannot be Copy

Now try the same with a struct that contains a `String`:

```rust
#[derive(Debug, Copy, Clone)]              // ERROR
struct Person {
    name: String,
    age: u32,
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0204]: the trait `Copy` may not be implemented for this type
  |
  | name: String,
  |       ------ this field does not implement `Copy`
```

A struct can derive `Copy` only if every field is `Copy`. `String` owns heap data, so it cannot be `Copy` (copying the bits would create two owners of the same heap allocation, violating the ownership rules).

Remove `Copy` from the derive (keep `Clone`):

```rust
#[derive(Debug, Clone)]
struct Person {
    name: String,
    age: u32,
}
```

The program compiles. `Person` can be cloned explicitly with `.clone()` but does not duplicate on assignment:

```rust
fn main() {
    let alice = Person { name: String::from("Alice"), age: 30 };
    let alice_copy = alice.clone();      // explicit clone; both valid

    let bob = Person { name: String::from("Bob"), age: 25 };
    let _other = bob;                    // moves bob; bob is no longer valid
    // println!("{bob:?}");              // would fail

    println!("{alice:?}, {alice_copy:?}");
}
```

### Task 2.5 -- The clone-everywhere antipattern

Module 6 mentioned that calling `.clone()` everywhere to silence the borrow checker is a code smell. Demonstrate this with a function that takes ownership unnecessarily:

```rust
fn print_length_owned(s: String) {        // takes ownership
    println!("length: {}", s.len());
}

fn main() {
    let message = String::from("hello world");
    print_length_owned(message.clone());     // clone to keep message valid
    print_length_owned(message.clone());     // clone again
    print_length_owned(message);              // last call: just move it
}
```

Run the program. It works, but two heap allocations happened just so we could pass the same string to a function three times. This is wasteful.

The right fix is references (next exercise). The function should take `&String` (or better, `&str`) so the caller does not have to clone or move.

### Checkpoints

1. The list of `Copy` types includes integers, floats, booleans, characters, and tuples of `Copy` types. Why is `String` deliberately not `Copy`? What rule would be violated if it were?
2. The compiler error in Task 2.4 said `Copy` may not be implemented because `String` does not implement `Copy`. Why does deriving `Copy` require all fields to be `Copy`? What invariant is the compiler protecting?
3. In Task 2.5, the clones were unnecessary. Explain in one sentence what was wrong with the function's signature, and what change to the signature would eliminate the need to clone.

---

## Exercise 3 -- Immutable References

**Estimated time:** 10 minutes
**Topics covered:** the `&` operator, function parameters, reading without taking ownership

### Context

References let you access a value without taking ownership. The owner retains the value; the borrower has temporary access. This is the most common way to pass data into functions in Rust.

### Task 3.1 -- The basic reference

Replace `src/main.rs` with:

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;

    println!("s = {s}");        // owner is still valid
    println!("r = {r}");        // borrower works too
}
```

Run the program. Both `s` and `r` work. The reference `r` borrows from `s` without taking ownership.

### Task 3.2 -- Functions that borrow

Replace the file with:

```rust
fn print_length(s: &String) {
    println!("length: {}", s.len());
}

fn main() {
    let s = String::from("hello world");

    print_length(&s);
    print_length(&s);                   // can borrow as many times as we want
    print_length(&s);

    println!("after calls: {s}");       // s is still valid
}
```

Run the program. The function takes a `&String` (a reference to a `String`), not a `String`. Calling it does not consume `s`. We can call it many times, and `s` is still valid afterward.

This is the canonical pattern for read-only access in Rust. Take references instead of owned values whenever you do not need to consume the data.

### Task 3.3 -- Why & makes the function flexible

A function that takes `&String` is restrictive: it accepts `&String` and not much else. A function that takes `&str` is more flexible. Replace `print_length` with:

```rust
fn print_length(s: &str) {
    println!("length: {}", s.len());
}

fn main() {
    let owned = String::from("hello world");
    let literal = "this is a literal";

    print_length(&owned);                // works
    print_length(literal);               // works (literals are &'static str)
    print_length("inline literal");      // works (passed directly)

    println!("owned: {owned}");
}
```

Run the program. The function now accepts both `&String` (via `&owned`, which deref-coerces to `&str`) and `&str` (literals directly). This is much more useful than `&String`.

The convention in idiomatic Rust: when a function only needs to read a string, take `&str`. When it needs to read a sequence, take `&[T]` instead of `&Vec<T>`.

### Task 3.4 -- Methods often borrow implicitly

Replace `main` with:

```rust
fn main() {
    let s = String::from("hello world");

    let length = s.len();           // s.len() takes &self; s is still valid
    let upper = s.to_uppercase();    // s.to_uppercase() takes &self; s is still valid

    println!("s: {s}");              // works
    println!("length: {length}");
    println!("upper: {upper}");
}
```

Run the program. Three method calls in a row, and `s` is still valid. The methods take `&self`, so they only borrow `s` temporarily. The dot-syntax automatically creates the reference.

If `to_uppercase` took `self` (by value) instead of `&self`, `s` would be moved into it and would not be available afterward. The convention of taking `&self` is what makes method chains and repeated calls possible.

### Task 3.5 -- Passing a reference to a function that needs to read

Add a more substantial function:

```rust
fn count_words(text: &str) -> usize {
    text.split_whitespace().count()
}

fn first_word(text: &str) -> &str {
    text.split_whitespace().next().unwrap_or("")
}
```

Use them from `main`:

```rust
fn main() {
    let sentence = String::from("the quick brown fox jumps");

    let count = count_words(&sentence);
    let first = first_word(&sentence);

    println!("words: {count}");
    println!("first: {first}");
    println!("original: {sentence}");
}
```

Run the program. Expected output:

```
words: 5
first: the
original: the quick brown fox jumps
```

The `first_word` function returns a slice that borrows from its input. The slice points into the same memory as the original `sentence`. We will revisit this in Exercise 7.

### Checkpoints

1. The function `print_length` was rewritten to take `&str` instead of `&String`. The change made the function accept more types. What types can be passed to a function taking `&str`?
2. In Task 3.4, the methods `len`, `to_uppercase`, and several others all take `&self` rather than `self`. What would change if they took `self` instead? Why is `&self` a better default?
3. The `first_word` function returns a `&str` that borrows from its input. What does this borrowing relationship tell the compiler about the lifetime of the returned slice?

---

## Exercise 4 -- Mutable References and the Borrow Rules

**Estimated time:** 15 minutes
**Topics covered:** `&mut`, the borrow rules, simultaneous-access errors, reading the diagnostics

### Context

Module 6 stated the borrow rules: at any time, you can have any number of immutable references OR exactly one mutable reference. Not both. This exercise demonstrates the rules with deliberate compile errors.

### Task 4.1 -- A mutable reference

Replace `src/main.rs` with:

```rust
fn main() {
    let mut s = String::from("hello");

    let r = &mut s;
    r.push_str(", world");

    println!("{r}");
}
```

Run the program. Expected output:

```
hello, world
```

Three things made this work:

- The owner `s` is `mut`. Without it, you cannot create a mutable reference.
- The reference is created with `&mut`, not just `&`.
- The reference is the only handle to `s` while it is being used to mutate.

### Task 4.2 -- The first borrow rule violation

Try this:

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &mut s;            // ERROR: cannot borrow as mutable while r1 exists

    println!("{r1}, {r2}");
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
```

The diagnostic is informative. It tells you exactly which borrows conflict and where. In `lab5-notes.md`, write down the full error message.

### Task 4.3 -- The second rule violation: two mutable

Replace `main` with:

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s;            // ERROR: cannot borrow as mutable more than once

    println!("{r1}, {r2}");
}
```

Run `cargo build`. The compiler rejects this with a different but related error:

```
error[E0499]: cannot borrow `s` as mutable more than once at a time
```

The rule is "exactly one mutable reference at a time," and this code tries to create two.

### Task 4.4 -- Multiple immutable references are fine

Replace `main` with:

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    let r3 = &s;

    println!("{r1}, {r2}, {r3}");
}
```

Run the program. Expected output:

```
hello, hello, hello
```

Three immutable references coexist without issue. The rule allows any number of immutable references; only mutable ones are restricted.

### Task 4.5 -- Sequential borrows work

Replace `main` with:

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{r1}, {r2}");
    // r1 and r2 are no longer used; their borrows end here.

    let r3 = &mut s;
    r3.push_str(", world");
    println!("{r3}");
}
```

Run the program. Expected output:

```
hello, hello
hello, world
```

The immutable borrows ended (their last use was in the first `println!`), so the mutable borrow that followed is legal. The compiler tracks where each borrow is actually used and ends the borrow at its last use, not at the end of the scope. This rule (called non-lexical lifetimes) makes the borrow checker much more flexible than a simple "borrow lasts until end of scope" model.

### Task 4.6 -- A function that mutates through a reference

Add this function:

```rust
fn append_exclamation(s: &mut String) {
    s.push_str("!");
}
```

Use it from `main`:

```rust
fn main() {
    let mut greeting = String::from("hello");

    append_exclamation(&mut greeting);
    append_exclamation(&mut greeting);
    append_exclamation(&mut greeting);

    println!("{greeting}");
}
```

Run the program. Expected output:

```
hello!!!
```

The function takes `&mut String`. Each call passes a fresh mutable reference. Between calls, the mutable borrow ends, so the next call can create its own.

This is the standard pattern for functions that modify their input: take a mutable reference, do the work, return.

### Task 4.7 -- A real bug the rules prevent

The borrow rules look like bureaucracy until you see the bugs they prevent. Try this:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];

    v.push(4);                       // ERROR: cannot borrow v as mutable
    println!("first: {first}");
}
```

Run `cargo build`. The compiler rejects this. Without the rule, here is what would happen:

- `&v[0]` creates a reference to the first element.
- `v.push(4)` might cause the vector to reallocate to a larger buffer (since 4 elements no longer fit in the original allocation).
- After reallocation, the original elements are at a new memory location.
- The reference `first` still points to the old memory, which is now freed.
- Reading `*first` would be a use-after-free bug.

The borrow rules detect this at compile time. The diagnostic tells you exactly what happened: you cannot have an immutable reference into the vector while doing something that might invalidate it.

### Checkpoints

1. The borrow rules are stated as "any number of immutable references OR exactly one mutable reference." What practical bug does the "or" prevent? In other words, why is it not safe to have both a mutable reference and an immutable reference simultaneously?
2. Task 4.5 worked because the borrows ended at the last use, not at the end of the scope. Why is this rule (non-lexical lifetimes) important for ordinary code?
3. Task 4.7 produced a compile error that prevented a use-after-free bug. In a language without compile-time borrow checking (like C++), this kind of bug usually shows up as a crash or memory corruption at runtime. Why is catching it at compile time better?

---

## Exercise 5 -- Slices

**Estimated time:** 10 minutes
**Topics covered:** array slices (`&[T]`), string slices (`&str`), slices as flexible parameter types

### Context

A slice is a reference to a contiguous range of elements. Slices are the idiomatic way to refer to part of a sequence without taking ownership and without copying. They appear constantly in Rust code.

### Task 5.1 -- Array slices

Replace `src/main.rs` with:

```rust
fn main() {
    let numbers = [10, 20, 30, 40, 50];

    let all: &[i32] = &numbers;
    let middle: &[i32] = &numbers[1..4];
    let first_two: &[i32] = &numbers[..2];
    let last_three: &[i32] = &numbers[2..];

    println!("all: {all:?}");
    println!("middle: {middle:?}");
    println!("first_two: {first_two:?}");
    println!("last_three: {last_three:?}");
}
```

Run the program. Expected output:

```
all: [10, 20, 30, 40, 50]
middle: [20, 30, 40]
first_two: [10, 20]
last_three: [30, 40, 50]
```

The slice syntax `&array[start..end]` produces a `&[T]`. The slice has a length and a pointer; it does not own the data. Slicing the same array four times produces four references into the same memory.

### Task 5.2 -- A function that takes a slice

Add this function:

```rust
fn sum(values: &[i32]) -> i32 {
    let mut total = 0;
    for v in values {
        total += v;
    }
    total
}
```

Use it from `main`:

```rust
fn main() {
    let array = [1, 2, 3, 4, 5];
    let vector = vec![10, 20, 30];

    println!("array sum: {}", sum(&array));
    println!("vector sum: {}", sum(&vector));
    println!("partial sum: {}", sum(&array[1..4]));
}
```

Run the program. Expected output:

```
array sum: 15
vector sum: 60
partial sum: 9
```

The single function works for arrays, vectors, and subranges. This is the value of `&[T]`: one parameter type accepts many concrete types.

### Task 5.3 -- The wrong way

For comparison, write a version that takes `Vec<i32>` instead:

```rust
fn sum_vec(values: Vec<i32>) -> i32 {
    let mut total = 0;
    for v in &values {
        total += v;
    }
    total
}
```

Try to call it with an array:

```rust
fn main() {
    let array = [1, 2, 3, 4, 5];
    let result = sum_vec(array);            // ERROR: expected Vec<i32>, found [i32; 5]
    println!("{result}");
}
```

Run `cargo build`. The compiler rejects this. The function only accepts `Vec<i32>`. To use it, you would have to convert: `sum_vec(array.to_vec())`, which allocates a new vector. Wasteful.

The slice version (`&[i32]`) is universally preferable when the function only needs to read a sequence. Take ownership only when you need it.

### Task 5.4 -- String slices

Replace `main` with:

```rust
fn main() {
    let s = String::from("hello world");

    let hello: &str = &s[0..5];
    let world: &str = &s[6..11];
    let entire: &str = &s;

    println!("hello: {hello}");
    println!("world: {world}");
    println!("entire: {entire}");
}
```

Run the program. Expected output:

```
hello: hello
world: world
entire: hello world
```

String slices are references into the bytes of a `String`. Like array slices, they have a pointer and a length; they do not own the data.

### Task 5.5 -- The byte-boundary rule

Module 6 mentioned that string slicing must occur at character boundaries. Try this:

```rust
fn main() {
    let s = String::from("héllo");
    let bad = &s[0..2];        // PANIC: byte 1 is in the middle of é
    println!("{bad}");
}
```

Run the program. It compiles, but at runtime:

```
thread 'main' panicked at 'byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3)'
```

The `é` is a two-byte character in UTF-8. Slicing across it would produce invalid UTF-8, so Rust panics. The compile-time check cannot catch this (the byte indices are runtime values), but the runtime check stops the program before it does something undefined.

For text-aware processing, use `chars()`, `split_whitespace()`, or other character-aware methods rather than byte slicing.

### Task 5.6 -- A function returning a slice

Add this function:

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
```

Use it from `main`:

```rust
fn main() {
    let sentence = String::from("hello world rust");
    let first = first_word(&sentence);

    println!("first word: {first}");
    println!("original: {sentence}");
}
```

Run the program. Expected output:

```
first word: hello
original: hello world rust
```

The function returns a slice that borrows from the input. The returned slice is valid as long as the input is valid. The compiler tracks this relationship through lifetimes (Exercise 6).

### Checkpoints

1. The function `sum` was rewritten to take `&[i32]` instead of `Vec<i32>`. List three concrete types of argument that the slice version accepts but the vector version does not.
2. In Task 5.5, the panic occurred at runtime, not compile time. Could the compiler have detected this case? Why or why not?
3. The `first_word` function returns a slice borrowing from its input. What is the relationship between the input's lifetime and the output's lifetime? (Anticipating Exercise 6.)

---

## Exercise 6 -- Lifetimes in Functions

**Estimated time:** 15 minutes
**Topics covered:** when annotations are required, the lifetime syntax, what an annotation tells the compiler

### Context

Most of the time, lifetimes are inferred. You write code without thinking about them, and the compiler infers the right relationships automatically. When the inference is not enough, the compiler asks for help; that is when you write explicit lifetime annotations.

This exercise shows when annotations are required and what they say.

### Task 6.1 -- A function that needs annotations

Replace `src/main.rs` with:

```rust
fn longer(s1: &str, s2: &str) -> &str {
    if s1.len() >= s2.len() { s1 } else { s2 }
}

fn main() {
    let a = String::from("longer string");
    let b = String::from("short");

    let result = longer(&a, &b);
    println!("longer: {result}");
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0106]: missing lifetime specifier
  |
  | fn longer(s1: &str, s2: &str) -> &str {
  |                                  ^ expected named lifetime parameter
```

The error explains that the return type contains a borrowed value, but the compiler cannot determine which input it borrows from. Is the result borrowed from `s1`? From `s2`? Both? The signature does not say.

### Task 6.2 -- Add the annotations

Fix the function:

```rust
fn longer<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() >= s2.len() { s1 } else { s2 }
}
```

Three changes:

- `<'a>` after the function name declares a lifetime parameter named `'a`.
- `&'a str` says each parameter is a reference with lifetime `'a`.
- `-> &'a str` says the return value is also a reference with lifetime `'a`.

The annotation tells the compiler: "both inputs and the output share a lifetime called `'a`." The function's contract is that the returned reference is valid for the same region as the inputs.

Run the program. Expected output:

```
longer: longer string
```

### Task 6.3 -- A use that fails the lifetime check

Try this:

```rust
fn main() {
    let a = String::from("longer string");
    let result;

    {
        let b = String::from("short");
        result = longer(&a, &b);
    }

    println!("longer: {result}");        // ERROR
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0597]: `b` does not live long enough
```

The function says "the result is valid for the lifetime that both inputs share." `a` lives for all of `main`. `b` lives only for the inner block. Their shared lifetime is the inner block. So `result` is valid only inside the inner block. Trying to use it outside the block is an error.

This is the value of lifetime annotations: the contract is enforced at the call site. The caller cannot violate it.

### Task 6.4 -- A function with one reference

Some functions have a single reference parameter and return a reference. The lifetime is straightforward:

```rust
fn first_word<'a>(text: &'a str) -> &'a str {
    text.split_whitespace().next().unwrap_or("")
}
```

But the compiler can figure this out without annotations:

```rust
fn first_word(text: &str) -> &str {
    text.split_whitespace().next().unwrap_or("")
}
```

Both versions compile. The second version uses lifetime elision: when there is exactly one input lifetime, it is automatically used for the output.

Add the un-annotated version to your file and confirm it compiles. The compiler is doing the work for you.

### Task 6.5 -- A function with independent lifetimes

Some functions have two reference parameters but return only one of them. The lifetimes can be independent:

```rust
fn first_param<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    println!("ignoring y: {y}");
    x
}
```

The two lifetime parameters `'a` and `'b` are independent. The return value is tied to `'a` only. This signature says: "the result lives as long as the first parameter, regardless of the second."

Use it:

```rust
fn main() {
    let long = String::from("a long-lived string");
    let result;

    {
        let short = String::from("short");
        result = first_param(&long, &short);
    }

    println!("{result}");                // works: tied to long, which is still alive
}
```

Run the program. It compiles and prints `a long-lived string`. The result is valid because it borrows from `long`, which lives long enough.

If we had used `longer` (where both inputs share a lifetime), the same call would fail. The independent lifetimes in `first_param` give the compiler more information.

### Task 6.6 -- The 'static lifetime

Replace `main` with:

```rust
fn main() {
    let literal: &'static str = "this is a literal";
    let static_value = literal;

    println!("{static_value}");
}
```

String literals have type `&'static str`. The `'static` lifetime means "valid for the entire program." Literals are baked into the binary, so they are always valid.

You will sometimes see `'static` as a bound:

```rust
fn store<'a>(s: &'a str) -> &'a str { s }

fn main() {
    let result = store("hello");
    println!("{result}");
}
```

The literal `"hello"` is `&'static str`, which is valid for any `'a` (because static is the longest possible lifetime). The function accepts it.

### Checkpoints

1. The compiler error in Task 6.1 said the function's return type contains a borrowed value but the signature does not say where it is borrowed from. Why is the function unable to compile without annotations? What information is missing?
2. In Task 6.5, two independent lifetime parameters were declared. What is the practical difference between this signature and one that uses a single lifetime for both inputs?
3. The `'static` lifetime represents the entire program duration. Apart from string literals, where else might values have `'static` lifetime?

---

## Exercise 7 -- Lifetimes in Structs

**Estimated time:** 10 minutes
**Topics covered:** structs that hold references, lifetime parameters on structs, lifetime parameters on impl blocks

### Context

A struct that holds a reference must declare the lifetime of that reference. The struct cannot outlive the data it borrows. This exercise shows the syntax and the constraints.

### Task 7.1 -- A struct that holds a reference

Replace `src/main.rs` with:

```rust
struct Excerpt<'a> {
    content: &'a str,
}

fn main() {
    let novel = String::from("It was the best of times. It was the worst of times.");

    let first_sentence = novel.split('.').next().expect("could not find a period");

    let excerpt = Excerpt { content: first_sentence };

    println!("excerpt: {}", excerpt.content);
}
```

Run the program. Expected output:

```
excerpt: It was the best of times
```

The struct `Excerpt<'a>` has a lifetime parameter `'a`. The `content` field has type `&'a str`. This means an `Excerpt` is valid only as long as the data its content borrows from is valid.

In this example, `excerpt` borrows (indirectly) from `novel`. As long as `novel` is alive, `excerpt` is valid.

### Task 7.2 -- The annotation is required

Try removing the lifetime:

```rust
struct Excerpt {
    content: &str,                       // ERROR
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0106]: missing lifetime specifier
  |
  | content: &str,
  |          ^ expected named lifetime parameter
```

A reference field requires an explicit lifetime parameter on the struct. The struct must declare what lifetime the reference is tied to.

Restore the lifetime:

```rust
struct Excerpt<'a> {
    content: &'a str,
}
```

### Task 7.3 -- The struct cannot outlive its borrow

Try this:

```rust
fn make_excerpt() -> Excerpt {        // ERROR: missing lifetime in return type
    let novel = String::from("the local data");
    Excerpt { content: &novel }
}
```

Run `cargo build`. The compiler rejects this with several errors. The deeper problem: the function tries to return a struct that borrows from a local variable. When the function returns, `novel` is dropped, and the `Excerpt` would have a dangling reference.

You cannot fix this by adding annotations. The fundamental issue is that the function tries to return data borrowed from data that no longer exists. The fix is to return owned data instead:

```rust
struct OwnedExcerpt {
    content: String,
}

fn make_owned_excerpt() -> OwnedExcerpt {
    let novel = String::from("the local data");
    OwnedExcerpt { content: novel }
}
```

The owned version compiles. It does not have lifetime parameters because it does not borrow.

This pattern (preferring owned structs unless borrowing is genuinely needed) is one of the most common in idiomatic Rust.

### Task 7.4 -- Methods on a struct with a lifetime

Add an impl block:

```rust
impl<'a> Excerpt<'a> {
    fn length(&self) -> usize {
        self.content.len()
    }

    fn first_word(&self) -> &str {
        self.content.split_whitespace().next().unwrap_or("")
    }
}
```

Use the methods:

```rust
fn main() {
    let novel = String::from("It was the best of times. It was the worst of times.");
    let first_sentence = novel.split('.').next().expect("no period");

    let excerpt = Excerpt { content: first_sentence };

    println!("length: {}", excerpt.length());
    println!("first word: {}", excerpt.first_word());
}
```

Run the program. Expected output:

```
length: 24
first word: It
```

The `impl<'a>` declares the lifetime parameter for the impl block. The methods themselves do not need additional lifetime annotations because of the elision rules: methods with `&self` automatically use `self`'s lifetime for any returned references.

### Checkpoints

1. The struct definition `struct Excerpt<'a> { content: &'a str }` declares a lifetime parameter. What does this tell users of the struct about its relationship to other data?
2. Task 7.3 showed that you cannot return a struct that borrows from local data. Why is this rejected? What invariant is being protected?
3. The methods on `Excerpt` did not need explicit lifetime annotations on themselves. Which elision rule is doing the work?

---

## Exercise 8 -- Putting It Together

**Estimated time:** 15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from Modules 6 and 7 into one program: ownership transfer, cloning, references (mutable and immutable), slices, lifetime annotations on functions and structs, and the borrow rules. The program processes text data through a small pipeline, demonstrating the patterns students will use in real code.

### Task 8.1 -- The full program

Replace `src/main.rs` with:

```rust
struct TextSnapshot<'a> {
    source: &'a str,
}

impl<'a> TextSnapshot<'a> {
    fn new(source: &'a str) -> TextSnapshot<'a> {
        TextSnapshot { source }
    }

    fn word_count(&self) -> usize {
        self.source.split_whitespace().count()
    }

    fn first_word(&self) -> &str {
        self.source.split_whitespace().next().unwrap_or("")
    }
}

fn longest_word<'a>(text: &'a str) -> &'a str {
    text.split_whitespace()
        .max_by_key(|w| w.len())
        .unwrap_or("")
}

fn append_period(text: &mut String) {
    if !text.ends_with('.') {
        text.push('.');
    }
}

fn count_chars(text: &str) -> usize {
    text.chars().count()
}

fn main() {
    let mut document = String::from("the quick brown fox jumps over the lazy dog");

    // Use immutable borrows for read-only analysis:
    let snapshot = TextSnapshot::new(&document);
    let word_count = snapshot.word_count();
    let first = snapshot.first_word();
    let longest = longest_word(&document);
    let char_count = count_chars(&document);

    println!("Original: '{document}'");
    println!("Words: {word_count}");
    println!("First word: '{first}'");
    println!("Longest word: '{longest}'");
    println!("Character count: {char_count}");

    // The snapshot's borrow ends here at its last use. Now we can mutate.

    // Use a mutable borrow to modify in place:
    append_period(&mut document);

    println!("\nAfter appending period: '{document}'");

    // Demonstrate ownership transfer with clone:
    let archived = document.clone();
    println!("Archived (clone): '{archived}'");

    // Show that both the original and the clone are valid:
    println!("Original still valid: '{document}'");
}
```

Run the program. Expected output:

```
Original: 'the quick brown fox jumps over the lazy dog'
Words: 9
First word: 'the'
Longest word: 'quick'
Character count: 43

After appending period: 'the quick brown fox jumps over the lazy dog.'
Archived (clone): 'the quick brown fox jumps over the lazy dog.'
Original still valid: 'the quick brown fox jumps over the lazy dog.'
```

### Task 8.2 -- Identify the patterns

In `lab5-notes.md`, identify each instance of the following patterns in the program above:

1. A struct with a lifetime parameter holding a reference.
2. An impl block that declares a lifetime parameter.
3. A function with explicit lifetime annotations on its signature.
4. A function that takes `&str` instead of `&String` for flexibility.
5. A function that takes `&mut String` to modify its argument.
6. A method that uses lifetime elision (no explicit annotations needed).
7. A use of `.clone()` to deliberately duplicate data.
8. A point in `main` where an immutable borrow ends so a mutable borrow can begin.

### Task 8.3 -- Predict and verify

For each of the following code changes, predict whether the program will compile, and run it to verify.

**Change A:** Add this line at the end of `main`, before the closing brace:

```rust
println!("Snapshot is still valid: {}", snapshot.first_word());
```

Will this compile? Why or why not?

**Change B:** Move the line `let snapshot = TextSnapshot::new(&document);` to BEFORE the line `append_period(&mut document);` and try to use `snapshot` after the mutation:

```rust
let snapshot = TextSnapshot::new(&document);
let words = snapshot.word_count();
println!("words: {words}");

append_period(&mut document);

println!("first: {}", snapshot.first_word());        // try to use snapshot after mutation
```

Will this compile? Why or why not?

**Change C:** Modify `longest_word` to remove the lifetime annotations:

```rust
fn longest_word(text: &str) -> &str {
    text.split_whitespace()
        .max_by_key(|w| w.len())
        .unwrap_or("")
}
```

Will this still compile? Why or why not?

For each change, write your prediction in `lab5-notes.md` BEFORE running the code. Then run it and write down what actually happened. The exercise of predicting and verifying is one of the best ways to internalize the borrow checker's rules.

### Checkpoints

1. The `TextSnapshot` struct holds a reference rather than an owned `String`. What is the trade-off between this design and an owned-data alternative? When would you choose each?
2. The `longest_word` function has explicit lifetime annotations, but it could compile without them due to elision. Why might a library author still write the annotations explicitly?
3. The program uses `.clone()` once, deliberately. In an earlier exercise, the lab described `.clone()` as a code smell when used reflexively. What makes the use here legitimate rather than a code smell?

---

## Summary and Reflection

You have now used every concept from Modules 6 and 7 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Move Semantics | ownership transfer, the moved-value error | Assignment moves non-Copy values. The original binding is invalid after the move. |
| 2 -- Clone vs Copy | implicit duplication for Copy types, explicit for Clone | `Copy` is for stack values; `Clone` is for explicit duplication of heap data. |
| 3 -- Immutable References | borrowing without taking ownership, `&str` flexibility | Take references unless you need ownership. `&str` is more flexible than `&String`. |
| 4 -- Mutable References | the borrow rules, runtime bugs prevented at compile time | Any number of immutable borrows OR exactly one mutable borrow at a time. |
| 5 -- Slices | `&[T]` and `&str` as flexible parameter types | Slices accept arrays, vectors, and subranges with one signature. |
| 6 -- Lifetimes in Functions | when annotations are required, what they say | Annotations express relationships between input and output references. |
| 7 -- Lifetimes in Structs | structs with reference fields | A struct with a reference cannot outlive its borrow. |
| 8 -- Integration | all of the above | Real Rust programs combine these patterns; the borrow checker enforces the contracts. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab5-notes.md` before your next session.

1. Of the concepts in Modules 6 and 7, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. Several exercises in this lab produced compile errors deliberately, asking you to read the diagnostic and understand it. What did you find about the borrow checker's error messages compared to compile errors you have seen in other languages? Are they more helpful, less helpful, or differently helpful?

3. The lab covered both the basic ownership rules (Module 6) and the lifetime annotations that make them explicit (Module 7). Now that you have used both, identify a specific situation where you initially thought lifetime annotations were unnecessary complexity, but came to see why they are valuable. What changed your mind?

---

*End of Lab 5*
