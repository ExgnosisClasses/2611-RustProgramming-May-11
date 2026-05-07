# Module 5: Functions and Closures
## Lab 4 -- Working with Functions, Closures, and Higher-Order Functions

> **Course:** Mastering Rust
> **Module:** 5 - Functions and Closures
> **Estimated time:** 75-90 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 5: function definitions, parameters and return types, the expression-versus-statement distinction, closures and their capture modes, the `Fn`/`FnMut`/`FnOnce` categories, higher-order functions, and function pointers. You will build a single Cargo project called `data_processor` that processes streams of numeric data with reusable, composable transformations.

The focus is on the mechanics of building functions and closures and on choosing between them. Many exercises have multiple correct solutions; the lab guides you toward the idiomatic choice and asks you to compare alternatives.

By the end of this lab you will be able to:

- Write function signatures with appropriate parameter and return types.
- Recognize where the semicolon at the end of a line determines whether something is an expression or a statement.
- Write closures with and without type annotations and choose between them.
- Identify which capture mode the compiler picks for a given closure and force the `move` capture when needed.
- Determine whether a closure is `Fn`, `FnMut`, or `FnOnce` based on what it does.
- Pass functions and closures to higher-order functions interchangeably.
- Use `fn` (the function pointer type) to store regular functions in a uniform collection.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** This lab assumes familiarity with the type system from Lab 2 and the control flow from Lab 3. The closure exercises briefly use references (`&`, `&mut`) and traits (`Fn`, `FnMut`, `FnOnce`). Both topics will be covered in detail in later modules; for this lab, treat them as named patterns introduced in Module 5.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new data_processor
cd data_processor
code .
```

Create a `lab4-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Defining and Calling Functions

**Estimated time:** 10 minutes
**Topics covered:** function syntax, parameters, return types, naming conventions

### Context

Functions are the foundation of program organization in any language. Rust's syntax is similar to C and Java with a few differences: the `fn` keyword, type annotations after the parameter name, and the `->` arrow before the return type. This exercise establishes the basic mechanics.

### Task 1.1 -- Write your first functions

Replace the contents of `src/main.rs` with:

```rust
fn double(value: i32) -> i32 {
    value * 2
}

fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn print_result(label: &str, value: i32) {
    println!("{label}: {value}");
}

fn main() {
    let x = 5;
    let doubled = double(x);
    let sum = add(x, doubled);

    print_result("doubled", doubled);
    print_result("sum", sum);
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
doubled: 10
sum: 15
```

Note three things:

- Each parameter has its type written after the name: `value: i32`, `label: &str`.
- The return type appears after `->`. The function `print_result` has no return type because it does not return a meaningful value.
- The body of `double` and `add` ends with an expression (no semicolon), which becomes the return value.

### Task 1.2 -- Function declaration order

Add this function below `main`:

```rust
fn used_before_declared(x: i32) -> i32 {
    x * x
}
```

Then call it from `main`:

```rust
    let squared = used_before_declared(4);
    println!("squared: {squared}");
```

Run the program. The function is declared after `main` but called from inside `main`. Unlike C, Rust does not require forward declarations. The compiler reads the entire file before resolving names.

### Task 1.3 -- Convention check

Replace `double` with this version that uses non-conventional naming:

```rust
fn Double(Value: i32) -> i32 {
    Value * 2
}
```

Update the call sites accordingly. Run `cargo build`. The program compiles, but with warnings:

```
warning: function `Double` should have a snake case name
warning: variable `Value` should have a snake case name
```

Rust enforces `snake_case` for function and variable names through compiler warnings. Restore the lowercase names before continuing.

### Checkpoints

1. The function `print_result` has no `-> ReturnType` annotation. What is its actual return type, and how can you verify this?
2. The compiler accepted the function `used_before_declared` even though it was defined after `main`. Why is this safe in Rust when it would be a compile error in C?
3. The naming convention is enforced through warnings, not errors. Why might the language choose to warn rather than reject? What would change if it were a hard error?

---

## Exercise 2 -- Expressions vs. Statements

**Estimated time:** 15 minutes
**Topics covered:** the expression/statement distinction, implicit returns, the semicolon footgun, blocks as expressions

### Context

This is the most unusual aspect of Rust functions for developers from C-family languages. The last expression in a function body is the return value. Adding a semicolon turns the expression into a statement, which changes the function's behavior. This exercise demonstrates the rule and the compiler errors that come from misunderstanding it.

### Task 2.1 -- The implicit return

Replace `main.rs` with:

```rust
fn double(value: i32) -> i32 {
    value * 2       // expression: this is the return value
}

fn main() {
    let result = double(5);
    println!("{result}");
}
```

Run the program. Expected output:

```
10
```

The body of `double` is a single expression. There is no `return` keyword. The expression's value flows out of the function as the return value.

### Task 2.2 -- The semicolon footgun

Add a semicolon to the end of the expression:

```rust
fn double(value: i32) -> i32 {
    value * 2;       // semicolon makes this a statement
}
```

Run `cargo build`. The compiler rejects this with:

```
error[E0308]: mismatched types
  |
  | fn double(value: i32) -> i32 {
  |                          --- expected `i32` because of return type
  |     value * 2;
  |              - help: remove this semicolon to return this value
```

The semicolon converted the expression into a statement. Statements have no value. The function body now ends with a statement, so the function returns `()` (the unit type). The compiler is helpful: it identifies the semicolon as the problem and tells you exactly how to fix it.

Remove the semicolon and confirm the program compiles.

### Task 2.3 -- Mixing return and implicit return

Replace `double` with this version that returns explicitly only on a special case:

```rust
fn safe_divide(numerator: f64, denominator: f64) -> f64 {
    if denominator == 0.0 {
        return 0.0;       // early return, semicolon is required here
    }
    numerator / denominator       // implicit return, no semicolon
}

fn main() {
    let a = safe_divide(10.0, 2.0);
    let b = safe_divide(10.0, 0.0);
    println!("a = {a}, b = {b}");
}
```

Run the program. Expected output:

```
a = 5, b = 0
```

The convention is to use `return` only for early exits and to use the implicit form for the final value. Both produce the same compiled code; the choice is stylistic.

### Task 2.4 -- Blocks as expressions

Replace `main` with:

```rust
fn main() {
    let value = {
        let x = 5;
        let y = 10;
        x + y       // last expression is the value of the block
    };

    println!("value: {value}");
}
```

Run the program. Expected output:

```
value: 15
```

The block `{ ... }` is itself an expression. It evaluates to its last expression. The intermediate variables `x` and `y` are scoped to the block; they do not exist outside it.

This pattern is useful for grouping intermediate computations together when binding a final value.

### Task 2.5 -- if as an expression in a function body

Replace `main.rs` with:

```rust
fn classify(temperature: f64) -> &'static str {
    if temperature > 30.0 {
        "hot"
    } else if temperature > 20.0 {
        "warm"
    } else {
        "cool"
    }
}

fn main() {
    println!("{}", classify(25.0));
    println!("{}", classify(35.0));
    println!("{}", classify(15.0));
}
```

Run the program. Expected output:

```
warm
hot
cool
```

The function body is a single `if` expression. Each branch produces a `&'static str`, and the `if` expression as a whole is the function's return value. There is no `return` keyword anywhere.

This is the most common idiom for short classification functions in Rust.

### Checkpoints

1. Task 2.2 showed that a misplaced semicolon converts an expression into a statement. Why is this a deliberate language design rather than a quirk to be worked around?
2. Task 2.4 used a block as an expression to compute a value with intermediate steps. Identify a code pattern where this is cleaner than declaring the intermediate variables in the surrounding scope.
3. Task 2.5 used `if` as the only expression in a function body. Could `loop` or `while` be used the same way? Why or why not? (Hint: review Module 4.)

---

## Exercise 3 -- Closure Syntax and Type Inference

**Estimated time:** 10-15 minutes
**Topics covered:** closure syntax, type inference, when annotations are required

### Context

Closures are anonymous functions that can capture values from their surrounding scope. The syntax is different from regular functions: parameters go between vertical bars rather than parentheses, and types are usually inferred. This exercise establishes the syntax and the inference rules.

### Task 3.1 -- Basic closure syntax

Replace `main.rs` with:

```rust
fn main() {
    // Single-parameter closure with no annotations
    let double = |x| x * 2;
    println!("double(5) = {}", double(5));

    // Two parameters
    let add = |x, y| x + y;
    println!("add(3, 7) = {}", add(3, 7));

    // No parameters
    let greeting = || "Hello, world!";
    println!("{}", greeting());

    // Multi-statement body in braces
    let format_pair = |label: &str, value: i32| {
        let result = format!("{label}: {value}");
        result      // last expression: the return value
    };
    println!("{}", format_pair("count", 42));
}
```

Run the program. Expected output:

```
double(5) = 10
add(3, 7) = 10
Hello, world!
count: 42
```

Note four things:

- Parameters go between `|` and `|`, not between `(` and `)`.
- Type annotations are usually optional. The compiler infers them from how the closure is used.
- The body can be a single expression (no braces) or a block (braces).
- For multi-statement bodies, the last expression with no semicolon is the return value, just like functions.

### Task 3.2 -- Inference from usage

Replace `main` with:

```rust
fn main() {
    let identity = |x| x;       // type cannot yet be determined
    println!("{}", identity(5));
}
```

Run the program. It compiles and prints `5`. The compiler inferred `i32` from the call site `identity(5)`.

But try to call it with a different type after that:

```rust
fn main() {
    let identity = |x| x;
    println!("{}", identity(5));
    println!("{}", identity("hello"));      // error
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0308]: mismatched types
  |
  |     println!("{}", identity("hello"));
  |                             ^^^^^^^ expected integer, found `&str`
```

Once the closure is "used" once with an `i32`, its type is locked in. It is not generic. The compiler infers a single signature based on the first use.

If you want a closure that works for multiple types, you have to annotate or use generics, which goes beyond this lab.

### Task 3.3 -- Required annotations

Some closures cannot be inferred. Replace `main` with:

```rust
fn main() {
    let untyped = |x| x;        // never used; type cannot be inferred
    println!("done");
}
```

Run `cargo build`. The compiler rejects this with an error about a type that cannot be inferred. The closure is never called, so the compiler has no way to determine the parameter type.

Fix it by annotating:

```rust
    let untyped = |x: i32| x;
    println!("done");
```

The program now compiles. Once the parameter type is known, the return type is inferred to be the same.

### Task 3.4 -- Comparing functions and closures

Replace `main.rs` with:

```rust
fn double_fn(x: i32) -> i32 {
    x * 2
}

fn main() {
    let double_cl = |x| x * 2;

    println!("function: {}", double_fn(5));
    println!("closure:  {}", double_cl(5));
}
```

Run the program. Both forms produce `10`. They are functionally equivalent here, but they are not interchangeable in all contexts (Section G of Module 5 explained this). The function has explicit type annotations because they are required for functions; the closure has none because the compiler infers them.

### Checkpoints

1. The compiler infers closure types from usage. The first call to a closure determines its types for the rest of the program. Why is this design choice reasonable? What would be lost if closures were generic by default?
2. Task 3.3 showed that an unused closure with no annotations cannot be compiled. Why is this an error rather than simply leaving the type undetermined?
3. Compare the closure form `|x| x * 2` with the function form `fn double(x: i32) -> i32 { x * 2 }`. Identify two specific situations where the closure form is preferable, and one where the function form is preferable.

---

## Exercise 4 -- Capturing Variables

**Estimated time:** 15 minutes
**Topics covered:** the three capture modes, when each is used, the `move` keyword

### Context

The most important feature of closures is that they can capture values from their enclosing scope. Rust has three capture modes: by borrow (read-only), by mutable borrow (read-write), and by value (move). The compiler picks the least restrictive mode that lets the closure work. The `move` keyword forces capture by value.

> **Note:** This exercise touches on borrowing and references, which are covered in detail in a later module. For now, treat the three capture modes as named patterns: "borrow," "borrow mutably," and "take ownership."

### Task 4.1 -- Capture by borrow (read-only)

Replace `main.rs` with:

```rust
fn main() {
    let name = String::from("Alice");

    let greet = || println!("Hello, {name}");

    greet();
    greet();

    // The original name is still usable here:
    println!("Original name: {name}");
}
```

Run the program. Expected output:

```
Hello, Alice
Hello, Alice
Original name: Alice
```

The closure captured `name` by borrow. It can read but not modify. The original `name` is still usable after the closure runs because the closure only borrowed it.

### Task 4.2 -- Capture by mutable borrow

Replace `main` with:

```rust
fn main() {
    let mut count = 0;

    let mut increment = || {
        count += 1;
        println!("count is now {count}");
    };

    increment();
    increment();
    increment();
}
```

Run the program. Expected output:

```
count is now 1
count is now 2
count is now 3
```

The closure captured `count` by mutable borrow. Note two things:

- The closure binding itself must be `mut`. The line is `let mut increment = ...`. This is because calling the closure modifies its captured state.
- Inside the closure, you write `count += 1` even though `count` is captured by mutable reference. The compiler dereferences automatically.

Try removing `mut` from the closure binding:

```rust
    let increment = || { count += 1; ... };
```

Run `cargo build`. The compiler rejects this, indicating that the closure is `FnMut` and the binding must be mutable to call it. Restore the `mut` keyword.

### Task 4.3 -- Borrow restrictions while a closure is active

Replace `main` with:

```rust
fn main() {
    let mut count = 0;

    let mut increment = || count += 1;

    // While increment exists and might be called, count is borrowed:
    println!("count: {count}");      // ERROR: cannot borrow count

    increment();
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0502]: cannot borrow `count` as immutable because it is also borrowed as mutable
```

While the closure has `count` borrowed mutably, no one else can use `count` (not even to read it). This is part of Rust's borrow rules, covered fully in Module 6. The fix is to call the closure before reading count, or to scope the closure tightly:

```rust
    increment();
    println!("count: {count}");      // ok: closure no longer in use
```

Confirm the fix works.

### Task 4.4 -- Capture by value with move

Replace `main` with:

```rust
fn main() {
    let name = String::from("Alice");

    let take_ownership = move || println!("Hello, {name}");

    take_ownership();

    // println!("{name}");      // ERROR: name was moved into the closure
}
```

Run the program. Expected output:

```
Hello, Alice
```

The `move` keyword forces the closure to take ownership of `name`. After the closure is created, `name` no longer exists in `main`'s scope. Try uncommenting the last line:

```rust
    println!("{name}");
```

Run `cargo build`. The compiler rejects this:

```
error[E0382]: borrow of moved value: `name`
```

The original `name` is gone. It is now owned by the closure.

Remove the bad line and confirm the program compiles again.

### Task 4.5 -- A practical use of move: thread spawning

Replace `main.rs` with:

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5];

    let handle = thread::spawn(move || {
        println!("Inside the thread: {data:?}");
        let total: i32 = data.iter().sum();
        println!("Sum: {total}");
    });

    handle.join().unwrap();
    println!("Main thread is done");
}
```

Run the program. Expected output:

```
Inside the thread: [1, 2, 3, 4, 5]
Sum: 15
Main thread is done
```

The `thread::spawn` function requires a closure that takes ownership of its captures. The reason is that the new thread might outlive the calling function, so the closure cannot borrow from local variables that might no longer exist. The `move` keyword transfers `data` into the closure, and the closure carries it into the new thread.

This is by far the most common practical use of `move`.

### Checkpoints

1. Task 4.1 used a closure that only reads its capture. Task 4.2 used one that mutates its capture. The compiler chose the capture mode automatically. Why is this preferable to making the developer specify the mode explicitly?
2. Task 4.3 showed that you cannot read a variable while a closure has it borrowed mutably. The same rule applies to ordinary mutable borrows. What kind of bug does this rule prevent?
3. Task 4.5 used `move` to send data to a new thread. Without `move`, the thread would borrow from the calling function. Why is this dangerous? What goes wrong if the calling function returns while the thread is still running?

---

## Exercise 5 -- The `Fn`, `FnMut`, and `FnOnce` Categories

**Estimated time:** 15 minutes
**Topics covered:** the three closure categories, when each applies, accepting closures in functions

### Context

Every closure satisfies one or more of three categories: `Fn` (read-only, callable any number of times), `FnMut` (mutating, callable multiple times), and `FnOnce` (might consume captures, callable at most once). The category determines where the closure can be used. A function that accepts an `Fn` will reject a closure that mutates; a function that accepts `FnOnce` will accept any closure but can call it only once.

> **Note:** The mechanics of using these as generic constraints (the `<F: Fn(...)>` syntax) involve traits and generics, which are covered in later modules. For this exercise, treat the categories as labels the compiler attaches to closures, and treat the function signatures as patterns for accepting closures.

### Task 5.1 -- An Fn closure

Replace `main.rs` with:

```rust
fn call_twice<F: Fn()>(f: F) {
    f();
    f();
}

fn main() {
    let name = String::from("Alice");
    let greet = || println!("Hello, {name}");

    call_twice(greet);
}
```

Run the program. Expected output:

```
Hello, Alice
Hello, Alice
```

The closure `greet` only reads its capture. It satisfies `Fn`, which means it can be called any number of times. The function `call_twice` calls it twice without issue.

### Task 5.2 -- An FnMut closure

Replace `main.rs` with:

```rust
fn call_three_times<F: FnMut()>(mut f: F) {
    f();
    f();
    f();
}

fn main() {
    let mut count = 0;
    let increment = || count += 1;

    call_three_times(increment);

    // Note: after passing to the function, we cannot use the closure here.
    // But we will see count's final value later in this exercise.
}
```

Run the program. Hmm, there is no output. The `println!` was not in the closure or the function. Add a print to verify the count was modified:

This exercise is tricky because once `count` is captured, we cannot easily print it from `main`. Try this version instead:

```rust
fn call_three_times<F: FnMut()>(mut f: F) {
    f();
    f();
    f();
}

fn main() {
    let mut count = 0;

    {
        let increment = || count += 1;
        call_three_times(increment);
    }

    // Now the closure is gone, and we can use count again
    println!("Count: {count}");
}
```

Run the program. Expected output:

```
Count: 3
```

The closure satisfied `FnMut` because it modifies its capture. The function accepted it and called it three times. After the closure scope ends, the borrow on `count` is released and `main` can read it again.

### Task 5.3 -- An FnOnce closure

Replace `main.rs` with:

```rust
fn call_once<F: FnOnce()>(f: F) {
    f();
}

fn main() {
    let name = String::from("Alice");

    let consume = move || {
        println!("Hello, {name}");
        drop(name);      // explicitly destroy name
    };

    call_once(consume);
}
```

Run the program. Expected output:

```
Hello, Alice
```

The closure consumes `name` (the `drop` explicitly destroys it). After running once, the closure cannot be called again because the captured value is gone. This is `FnOnce`. The function accepts it but can only call it once.

### Task 5.4 -- The hierarchy

Replace `main.rs` with:

```rust
fn call_three_times<F: FnMut()>(mut f: F) {
    f();
    f();
    f();
}

fn main() {
    let name = String::from("Alice");
    let greet = || println!("Hello, {name}");

    // greet is Fn, but Fn is also FnMut, so it can be passed here.
    call_three_times(greet);
}
```

Run the program. Expected output:

```
Hello, Alice
Hello, Alice
Hello, Alice
```

The closure `greet` only reads its capture, so it satisfies `Fn`. But `Fn` is a more restrictive form of `FnMut` (every `Fn` is also a valid `FnMut`). The function `call_three_times` accepts `FnMut`, so the more specific `Fn` closure is acceptable.

### Task 5.5 -- A practical retry pattern

Replace `main.rs` with:

```rust
fn retry<F>(mut operation: F, max_attempts: u32) -> Result<i32, String>
where
    F: FnMut() -> Result<i32, String>,
{
    for attempt in 1..=max_attempts {
        match operation() {
            Ok(value) => return Ok(value),
            Err(message) => {
                println!("Attempt {attempt} failed: {message}");
            }
        }
    }
    Err(format!("Failed after {max_attempts} attempts"))
}

fn main() {
    let mut attempt_count = 0;

    let operation = || {
        attempt_count += 1;
        if attempt_count < 3 {
            Err(format!("not ready (attempt {attempt_count})"))
        } else {
            Ok(42)
        }
    };

    let result = retry(operation, 5);
    println!("Result: {result:?}");
}
```

Run the program. Expected output:

```
Attempt 1 failed: not ready (attempt 1)
Attempt 2 failed: not ready (attempt 2)
Result: Ok(42)
```

The closure mutates `attempt_count` between calls, so it satisfies `FnMut`. The `retry` function accepts an `FnMut() -> Result<i32, String>` (a closure with no parameters that returns a result). It calls the closure up to `max_attempts` times, returning the first success.

This is one of the most common production uses of closures with the `FnMut` category.

### Checkpoints

1. Task 5.4 showed that an `Fn` closure can be passed to a function that accepts `FnMut`. Why does this work, and why does the reverse (passing an `FnMut` to a function expecting `Fn`) not work?
2. The function `retry` in Task 5.5 was declared with `FnMut`. What would change if it were declared with `FnOnce` instead? What would change with `Fn`?
3. The compiler decides which categories a closure satisfies based on what the body does, not on what the developer says. What does this design tell you about the relationship between intent and verification in Rust?

---

## Exercise 6 -- Higher-Order Functions

**Estimated time:** 10-15 minutes
**Topics covered:** functions that take closures, iterator methods, returning closures

### Context

A higher-order function is one that takes a function or closure as a parameter, or returns one. The standard library's iterator API is built almost entirely from higher-order functions. This exercise demonstrates the most common patterns.

### Task 6.1 -- Standard library higher-order methods

Replace `main.rs` with:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    let doubled: Vec<i32> = numbers.iter().map(|n| n * 2).collect();
    println!("Doubled: {doubled:?}");

    let even: Vec<&i32> = numbers.iter().filter(|n| **n % 2 == 0).collect();
    println!("Even: {even:?}");

    let total: i32 = numbers.iter().sum();
    println!("Total: {total}");

    let max = numbers.iter().max();
    println!("Max: {max:?}");

    let first_over_5 = numbers.iter().find(|&&n| n > 5);
    println!("First over 5: {first_over_5:?}");
}
```

Run the program. Expected output:

```
Doubled: [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
Even: [2, 4, 6, 8, 10]
Total: 55
Max: Some(10)
First over 5: Some(6)
```

Each method takes one or more closures as arguments. The framework provides the iteration machinery; the closures provide the program-specific logic. This is why iterator code in Rust is so concise: a small number of methods compose into a vast number of programs.

The double-ampersand in `find(|&&n| n > 5)` is because `iter()` produces `&i32`, and `find` produces an `&` of that, so the closure gets `&&i32`. The pattern `&&n` destructures both layers, leaving `n` as a plain `i32`. The `&` patterns appear often when working with iterators; we will return to them in the iterator module.

### Task 6.2 -- Write your own higher-order function

Replace `main.rs` with:

```rust
fn apply<F>(f: F, value: i32) -> i32
where
    F: Fn(i32) -> i32,
{
    f(value)
}

fn main() {
    let double = |x| x * 2;
    let add_ten = |x| x + 10;
    let square = |x| x * x;

    println!("apply(double, 5)  = {}", apply(double, 5));
    println!("apply(add_ten, 5) = {}", apply(add_ten, 5));
    println!("apply(square, 5)  = {}", apply(square, 5));

    // You can also pass a closure inline:
    println!("inline:          = {}", apply(|x| x + 100, 5));
}
```

Run the program. Expected output:

```
apply(double, 5)  = 10
apply(add_ten, 5) = 15
apply(square, 5)  = 25
inline:          = 105
```

The function `apply` takes two arguments: a closure that takes `i32` and returns `i32`, and a value to apply it to. The `<F>` and `where F: Fn(i32) -> i32` syntax is generic-and-trait-bound code that we will cover formally in later modules; for now, treat it as the pattern for "this function accepts any closure with this signature."

### Task 6.3 -- Compose with multiple operations

Add this function to your file:

```rust
fn apply_all<F>(values: &[i32], f: F) -> Vec<i32>
where
    F: Fn(i32) -> i32,
{
    values.iter().map(|&v| f(v)).collect()
}
```

Use it from `main`:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let doubled = apply_all(&numbers, |x| x * 2);
    let squared = apply_all(&numbers, |x| x * x);

    println!("Original: {numbers:?}");
    println!("Doubled:  {doubled:?}");
    println!("Squared:  {squared:?}");
}
```

Run the program. Expected output:

```
Original: [1, 2, 3, 4, 5]
Doubled:  [2, 4, 6, 8, 10]
Squared:  [1, 4, 9, 16, 25]
```

The `apply_all` function is a small reusable abstraction. The same code handles any transformation that fits the signature `Fn(i32) -> i32`. The standard library's `map` is a more general version of the same idea.

### Task 6.4 -- Returning a closure

Add this function to your file:

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}
```

Use it from `main`:

```rust
fn main() {
    let add_five = make_adder(5);
    let add_hundred = make_adder(100);

    println!("add_five(10)    = {}", add_five(10));
    println!("add_hundred(10) = {}", add_hundred(10));
}
```

Run the program. Expected output:

```
add_five(10)    = 15
add_hundred(10) = 110
```

The function `make_adder` returns a closure that captures `n` by value (because of `move`). Each call produces a new closure that adds a different number.

The return type `impl Fn(i32) -> i32` says "I return some closure that satisfies `Fn(i32) -> i32`; the caller does not need to know the exact type." This is the modern Rust way to return closures. A later module covers it in detail along with the alternative `Box<dyn Fn>` form.

### Checkpoints

1. Task 6.1 used iterator methods like `map`, `filter`, `find`, and `sum`. Each accepts a closure that controls its behavior. Identify a different higher-order method from the iterator API (consult the documentation if needed) and describe what it does.
2. Task 6.4 returned a closure with `impl Fn(i32) -> i32`. Why does this work for `make_adder` even though every closure has its own anonymous type? (Hint: think about what `impl Trait` means.)
3. The function `apply_all` in Task 6.3 takes both a slice (`&[i32]`) and a closure (`F`). Why are these the right parameter types? What would change if it took `Vec<i32>` instead of `&[i32]`?

---

## Exercise 7 -- Function Pointers

**Estimated time:** 10-15 minutes
**Topics covered:** the `fn` type, function pointers vs closures, function tables

### Context

A closure has an anonymous type unique to itself. A function, by contrast, has a concrete type called a function pointer (`fn(...) -> ...`). The two interact in specific ways: every regular function can be used as a function pointer, and every function pointer can be used wherever an `Fn`, `FnMut`, or `FnOnce` is expected.

### Task 7.1 -- Function pointers

Replace `main.rs` with:

```rust
fn double(x: i32) -> i32 { x * 2 }
fn add_ten(x: i32) -> i32 { x + 10 }
fn square(x: i32) -> i32 { x * x }

fn main() {
    let f: fn(i32) -> i32 = double;

    println!("f(5) = {}", f(5));

    // Reassign to a different function with the same signature:
    let f: fn(i32) -> i32 = add_ten;
    println!("f(5) = {}", f(5));

    let f: fn(i32) -> i32 = square;
    println!("f(5) = {}", f(5));
}
```

Run the program. Expected output:

```
f(5) = 10
f(5) = 15
f(5) = 25
```

The variable `f` has type `fn(i32) -> i32`: a pointer to any function with that signature. The three reassignments use shadowing (Module 3) to bind `f` to different functions in turn.

### Task 7.2 -- A function pointer cannot hold a capturing closure

Add this to `main`:

```rust
    let multiplier = 3;
    let multiply: fn(i32) -> i32 = |x| x * multiplier;
```

Run `cargo build`. The compiler rejects this:

```
error[E0308]: mismatched types
  |
  |     let multiply: fn(i32) -> i32 = |x| x * multiplier;
  |                   --------------   ^^^^^^^^^^^^^^^^^^^ expected fn pointer, found closure
```

A closure that captures cannot fit into the `fn` type because each closure has its own anonymous type. Function pointers are the simplest concrete type for callables; they cannot represent captures.

### Task 7.3 -- Non-capturing closures can be function pointers

Replace the bad line with a non-capturing closure:

```rust
    let multiply: fn(i32) -> i32 = |x| x * 3;       // no capture
```

Run the program. The closure is just a function expressed with closure syntax; since it captures nothing, it can coerce to a function pointer.

### Task 7.4 -- A function table

Replace `main.rs` with:

```rust
fn add(a: i32, b: i32) -> i32 { a + b }
fn subtract(a: i32, b: i32) -> i32 { a - b }
fn multiply(a: i32, b: i32) -> i32 { a * b }

fn main() {
    let operations: [(&str, fn(i32, i32) -> i32); 3] = [
        ("add", add),
        ("subtract", subtract),
        ("multiply", multiply),
    ];

    let a = 12;
    let b = 4;

    for (name, op) in operations {
        println!("{a} {name} {b} = {}", op(a, b));
    }
}
```

Run the program. Expected output:

```
12 add 4 = 16
12 subtract 4 = 8
12 multiply 4 = 48
```

The array stores three function pointers along with their names. Each iteration of the loop calls the corresponding operation.

This pattern only works because functions have a uniform type (`fn(i32, i32) -> i32`). Closures, even closures with the same signature, would each have a different anonymous type and could not be stored in this array.

### Task 7.5 -- Functions in higher-order functions

Replace `main.rs` with:

```rust
fn double(x: i32) -> i32 { x * 2 }

fn apply<F: Fn(i32) -> i32>(f: F, value: i32) -> i32 {
    f(value)
}

fn main() {
    // A function passed where Fn is expected:
    println!("function: {}", apply(double, 5));

    // A closure passed where Fn is expected:
    println!("closure:  {}", apply(|x| x * 2, 5));
}
```

Run the program. Expected output:

```
function: 10
closure:  10
```

The function `apply` accepts anything that satisfies `Fn(i32) -> i32`. Both regular functions and closures satisfy this, so they are interchangeable at the call site. This is the most common case in Rust: write higher-order functions to accept the `Fn` family, and callers can pass either functions or closures.

### Checkpoints

1. Task 7.4 used `fn(i32, i32) -> i32` to type a function table. Could the same array hold both regular functions and capturing closures? Why or why not?
2. Task 7.5 showed that a regular function can be passed where an `Fn` is expected. The reverse is not always true: not every `Fn` is a function. Identify the specific situations where you can convert one direction but not the other.
3. Function pointers are described as "zero-cost." What does this mean concretely? What is the alternative cost paid by `Box<dyn Fn>`? (A later module covers `dyn` formally; describe the cost in general terms.)

---

## Exercise 8 -- Putting It Together

**Estimated time:** 10-15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from the module into one program: function definitions with deliberate types, expression-form bodies, closures with each capture mode, the `Fn` and `FnMut` categories, higher-order functions, and a function pointer for a uniform dispatch table. The program processes a stream of numeric data using composable transformations.

### Task 8.1 -- The full program

Replace `main.rs` with:

```rust
fn classify(value: i32) -> &'static str {
    if value > 100 {
        "high"
    } else if value > 50 {
        "medium"
    } else {
        "low"
    }
}

fn count_above<F>(values: &[i32], predicate: F) -> usize
where
    F: Fn(i32) -> bool,
{
    values.iter().filter(|&&v| predicate(v)).count()
}

fn make_threshold_predicate(threshold: i32) -> impl Fn(i32) -> bool {
    move |value| value > threshold
}

fn double(x: i32) -> i32 { x * 2 }
fn halve(x: i32) -> i32 { x / 2 }
fn square(x: i32) -> i32 { x * x }

fn main() {
    let values = vec![10, 25, 47, 88, 132, 156, 30, 75];

    // Classify each value
    println!("Classifications:");
    for &v in &values {
        println!("  {v} = {}", classify(v));
    }

    // Use a higher-order function with an inline closure
    let above_50 = count_above(&values, |v| v > 50);
    println!("\nValues above 50: {above_50}");

    // Use a higher-order function with a closure produced by another function
    let above_100 = make_threshold_predicate(100);
    let count = count_above(&values, above_100);
    println!("Values above 100: {count}");

    // A counter closure that uses FnMut
    let mut total_counted = 0;
    {
        let mut count_with_side_effect = |v: i32| {
            total_counted += 1;
            v > 50
        };

        let above_50_again = values.iter().filter(|&&v| count_with_side_effect(v)).count();
        println!("\nWith side-effect counting:");
        println!("  Above 50: {above_50_again}");
    }
    println!("  Total checks performed: {total_counted}");

    // A function table indexed by name
    let transforms: [(&str, fn(i32) -> i32); 3] = [
        ("double", double),
        ("halve", halve),
        ("square", square),
    ];

    println!("\nApplying each transform to {}:", values[0]);
    for (name, transform) in transforms {
        println!("  {name}: {}", transform(values[0]));
    }
}
```

Run the program. Expected output:

```
Classifications:
  10 = low
  25 = low
  47 = low
  88 = medium
  132 = high
  156 = high
  30 = low
  75 = medium

Values above 50: 4
Values above 100: 2

With side-effect counting:
  Above 50: 4
  Total checks performed: 8

Applying each transform to 10:
  double: 20
  halve: 5
  square: 100
```

### Task 8.2 -- Identify the patterns

In `lab4-notes.md`, identify each instance of the following patterns in the program above:

1. A function whose body is a single expression with no `return` keyword.
2. A function with an early `return` and a final implicit return.
3. A higher-order function that accepts an `Fn` closure.
4. A function that returns a closure using `impl Fn`.
5. A closure that captures a variable by `move`.
6. A closure that satisfies `FnMut` because it mutates a captured variable.
7. A function pointer used to type-erase three different function names.

### Checkpoints

1. The function `count_above` uses `Fn` rather than `FnMut`. Why is `Fn` the right choice? What would happen at the call site in `main` if the bound were changed to `FnMut`?
2. The function `make_threshold_predicate` returns `impl Fn(i32) -> bool`, which captures `threshold` by move. Could `threshold` be captured by borrow instead? What would the consequences be?
3. The `FnMut` closure in Task 8.1 was wrapped in a block scope. Why? What would happen if the closure outlived the surrounding scope's use of `total_counted`?

---

## Summary and Reflection

You have now used every concept from Module 5 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Function Definitions | syntax, types, naming | Functions require explicit signatures. The signature is the contract callers depend on. |
| 2 -- Expressions vs. Statements | implicit returns, semicolon footgun | The last expression in a block is its value. Adding a semicolon converts an expression to a statement. |
| 3 -- Closure Syntax | inference, type locking | Closures usually do not need annotations. The first call locks in their types. |
| 4 -- Capturing Variables | borrow, mutable borrow, move | The compiler picks the least restrictive capture. `move` is needed for cross-thread or returned closures. |
| 5 -- Fn / FnMut / FnOnce | the closure hierarchy | Functions specify which category they accept. More restrictive closures can be passed to more permissive parameters. |
| 6 -- Higher-Order Functions | map/filter/etc., custom higher-order code | Closures parameterize behavior. The standard library iterator API is built on this pattern. |
| 7 -- Function Pointers | `fn` vs `Fn` | Functions can be stored in a uniform type. Closures with captures cannot. |
| 8 -- Integration | all of the above | Idiomatic Rust uses these constructs together; reading code closely is how you learn the patterns. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab4-notes.md` before your next session.

1. Of the closure-related concepts in this module, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. The lab compared closures to lambdas in other languages. Pick one feature of Rust closures that does not exist in another language you know (the unique anonymous type, the `Fn`/`FnMut`/`FnOnce` hierarchy, the explicit `move` keyword, etc.). Explain in your own words why Rust has this feature and what it gains the language.

3. The lab exercises asked you to apply judgment several times: when to use a function vs a closure, when to use `move`, which `Fn` category to require in a function signature. Identify one case from this lab where you initially picked one approach and then changed your mind after considering the alternative. What changed your decision?

---

*End of Lab 4*
