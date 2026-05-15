# Module 18: Macros and Metaprogramming
## Lab 14 -- Writing Declarative Macros, Using Built-Ins, and Working with Derives

> **Course:** Mastering Rust
> **Module:** 18 - Macros and Metaprogramming
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 18: writing declarative macros with `macro_rules!`, using the common built-in macros (`vec!`, `println!`, `format!`, `assert_eq!`, `todo!`), recognizing and using procedural macros, applying derive macros to your own types, observing what attribute macros do, and judging when a macro is the right tool versus a function or generic.

You will build a single Cargo project called `event_dsl` that progressively explores macros: first writing your own declarative macros, then using the standard library's built-in macros in non-trivial ways, then applying procedural macros from popular crates (`serde`, `thiserror`). The project ends with an integration exercise that combines several macro types in one realistic scenario. The lab does NOT cover writing procedural macros from scratch (that is a substantial separate topic); instead, it focuses on reading, understanding, and using them.

The focus is on judgment: when a declarative macro is appropriate versus a function, when a procedural macro is genuinely necessary, what compile errors from macros mean, how to recognize macro expansion in error messages. Many decisions have multiple defensible answers; the lab guides you toward idiomatic Rust and asks you to compare alternatives.

By the end of this lab you will be able to:

- Read and write declarative macros with `macro_rules!`.
- Use fragment specifiers (`expr`, `ident`, `ty`, `pat`, etc.) appropriately.
- Write macros with repetition (`$( ... )*`, `$( ... ),*`).
- Recognize hygiene in macro-generated code.
- Use common built-in macros effectively.
- Apply standard derive macros (`Debug`, `Clone`, `PartialEq`, etc.) to your types.
- Use third-party derive macros (from `serde`, `thiserror`).
- Recognize attribute macros and function-like procedural macros.
- Read compiler error messages that point inside macro expansions.
- Choose between a macro, a function, and a generic for a given problem.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** Several exercises use third-party crates (`serde`, `thiserror`). These will be added through `Cargo.toml`, and Cargo will download them on the first build. Make sure you have an internet connection when you reach those exercises.

> **A note about macro errors.** Macro errors are notoriously confusing. The compiler often points inside the macro's expansion, showing generated code you did not write. This is a feature of how macros work; you will see examples in this lab. Reading these errors patiently is part of the skill.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new event_dsl
cd event_dsl
code .
```

Create a `lab14-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Your First Declarative Macros

**Estimated time:** 15 minutes
**Topics covered:** `macro_rules!`, basic patterns, the macro invocation `!` syntax

### Context

A declarative macro is defined with `macro_rules!`. It pattern-matches on the syntax passed to it and expands to new code. This exercise establishes the basic mechanics.

### Task 1.1 -- A trivial macro

Replace the contents of `src/main.rs` with:

```rust
macro_rules! say_hello {
    () => {
        println!("Hello from a macro!");
    };
}

fn main() {
    say_hello!();
    say_hello!();
    say_hello!();
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
Hello from a macro!
Hello from a macro!
Hello from a macro!
```

The macro `say_hello!` takes no arguments and expands to a single `println!` call. Each invocation is replaced at compile time with the macro's body.

Three things to notice:

- The macro is invoked with `say_hello!()`. The `!` distinguishes a macro call from a function call.
- The macro is defined before it is used. Macros must be defined or imported before any use.
- The macro's body has its own scope. Inside `{ ... }` after `=>`, you write Rust code (or code that becomes Rust code after expansion).

### Task 1.2 -- A macro with arguments

A macro can accept "metavariables" that capture parts of the input:

```rust
macro_rules! greet {
    ($name:expr) => {
        println!("Hello, {}!", $name);
    };
}

fn main() {
    greet!("Alice");
    greet!(String::from("Bob"));

    let name = "Charlie";
    greet!(name);
}
```

Run the program. Expected output:

```
Hello, Alice!
Hello, Bob!
Hello, Charlie!
```

The matcher `$name:expr` captures an expression and binds it to the name `name`. Inside the macro body, `$name` is replaced with the captured expression.

The `:expr` part is a *fragment specifier*. It tells the compiler what kind of syntax to expect. Other common specifiers include:

- `expr` (an expression like `1 + 2`, `f()`, `vec![1, 2]`).
- `ident` (an identifier like `foo`, `my_var`).
- `ty` (a type like `i32`, `Vec<String>`).
- `pat` (a pattern like `Some(x)`, `_`).
- `stmt` (a statement).
- `block` (a `{ ... }` block).
- `path` (a path like `std::io::Read`).
- `tt` (a "token tree"; the most flexible).

Each specifier constrains what the macro accepts. Try passing something invalid:

```rust
greet!(let x = 5);    // ERROR: not an expression
```

Run `cargo build`. The compiler rejects this with a parse error inside the macro.

The `:expr` specifier accepts most useful inputs (literals, function calls, variable references) but rejects statements and other non-expression syntax.

### Task 1.3 -- A macro with multiple patterns

A macro can have multiple rules, tried in order:

```rust
macro_rules! describe {
    () => {
        println!("nothing");
    };
    ($value:expr) => {
        println!("one value: {}", $value);
    };
    ($first:expr, $second:expr) => {
        println!("two values: {} and {}", $first, $second);
    };
}

fn main() {
    describe!();
    describe!(42);
    describe!("hello", "world");
}
```

Run the program. Expected output:

```
nothing
one value: 42
two values: hello and world
```

The compiler tries each rule in order. The first one that matches wins. This is how macros achieve "variable-arity" behavior that functions cannot.

If no rule matches, the compiler produces an error:

```rust
describe!(1, 2, 3);    // ERROR: no matching rule
```

Try this. Expected output:

```
error: no rules expected the token `,`
```

The compiler tells you the macro could not match what you passed. With three values, no rule applies.

### Task 1.4 -- Repetition for variadic macros

For genuinely variable-arity macros, use repetition. The syntax `$( ... ),*` says "zero or more, separated by commas":

```rust
macro_rules! print_all {
    ($($item:expr),*) => {
        $(
            println!("{}", $item);
        )*
    };
}

fn main() {
    print_all!(1, 2, 3, 4, 5);
    println!("---");
    print_all!("a", "b", "c");
    println!("---");
    print_all!();
}
```

Run the program. Expected output:

```
1
2
3
4
5
---
a
b
c
---
```

The matcher `$($item:expr),*` captures zero or more comma-separated expressions, each bound to `item`. The body `$( println!("{}", $item); )*` expands to one `println!` per captured item.

The pattern is the heart of declarative macros' power. It is what makes `vec![1, 2, 3]`, `println!("{}, {}", a, b)`, and similar variadic syntax possible. Functions cannot have variable-arity arguments in Rust; macros can.

The separator can be other characters:

- `$( ... );*` for semicolon-separated.
- `$( ... )*` for no separator (just consecutive items).
- `$( ... )+` for "one or more" (at least one required).

Try the semicolon variant:

```rust
macro_rules! sum_of {
    ($($item:expr);*) => {{
        let mut total = 0;
        $(
            total += $item;
        )*
        total
    }};
}

fn main() {
    let result = sum_of!(1; 2; 3; 4; 5);
    println!("sum: {}", result);
}
```

Run the program. Expected output:

```
sum: 15
```

The semicolon separator is unusual; you would not pick this in real code. But it shows that the separator is configurable.

### Task 1.5 -- Building a simple DSL

Combine these techniques to build a small DSL for creating vectors with labels:

```rust
macro_rules! labeled_vec {
    ($label:literal: $($item:expr),*) => {{
        let v = vec![$($item),*];
        println!("{}: {:?}", $label, v);
        v
    }};
}

fn main() {
    let numbers = labeled_vec!("primes": 2, 3, 5, 7, 11);
    let names = labeled_vec!("names": "Alice", "Bob", "Charlie");

    println!();
    println!("(numbers later: {numbers:?})");
    println!("(names later: {names:?})");
}
```

Run the program. Expected output:

```
primes: [2, 3, 5, 7, 11]
names: ["Alice", "Bob", "Charlie"]

(numbers later: [2, 3, 5, 7, 11])
(names later: ["Alice", "Bob", "Charlie"])
```

The macro `labeled_vec!` accepts a label (a literal string) followed by a colon and a comma-separated list of items. It builds a vector, prints it with the label, and returns the vector.

Two interesting bits:

- **`$label:literal`** uses the `literal` fragment specifier, which matches string literals, number literals, etc.
- **The double braces `{{ ... }}`** create a block expression that evaluates to the vector. The outer braces are part of the macro syntax; the inner block produces the value.

This is the kind of mini-DSL that macros enable. The syntax is non-standard (a label, then a colon, then values), but it expands to ordinary Rust code that the compiler can verify.

### Checkpoints

1. The `say_hello!()` macro has parentheses with nothing inside. What is the purpose of the parentheses? Could the macro be invoked as `say_hello!` (no parens)?
2. Task 1.3 had three rules, tried in order. What happens if two rules could match the same input? Which one is chosen, and is that deterministic?
3. The repetition syntax `$( ... ),*` separates items with commas. Could you use `$( ... ),+` (one or more)? When would `+` be preferable to `*`?

---

## Exercise 2 -- Macro Hygiene

**Estimated time:** 10 minutes
**Topics covered:** how Rust macros avoid name collisions, why this matters

### Context

A common bug in C-style macros: the macro's internal variables collide with the caller's variables. Rust macros are "hygienic": names introduced inside the macro do not collide with names outside. This exercise demonstrates this.

### Task 2.1 -- A potential collision

Try this:

```rust
macro_rules! double_with_temp {
    ($value:expr) => {{
        let temp = $value;
        temp * 2
    }};
}

fn main() {
    let temp = 100;
    let result = double_with_temp!(temp);
    println!("input was {temp}, doubled is {result}");
}
```

Run the program. Expected output:

```
input was 100, doubled is 200
```

What happened:

- The caller has a variable `temp` with value 100.
- The macro also has a variable named `temp` inside its body.
- The macro is invoked with `double_with_temp!(temp)`, passing the caller's `temp`.
- The macro's `temp` (internal) is bound to 100 (the value of the caller's `temp`).
- The macro returns `temp * 2 = 200`.
- The caller's `temp` is still 100; the macro's `temp` did not affect it.

In a C-style macro, this would NOT work. The macro's `temp = $value` would expand to `temp = temp`, which (without hygiene) would not behave correctly.

Rust's hygiene mechanism makes the two `temp` variables effectively different names, even though they look the same in the source. The macro's internal `temp` is distinct from any caller's `temp`.

### Task 2.2 -- Hygiene in detail

To see the mechanism more clearly:

```rust
macro_rules! increment_locally {
    () => {
        let local_var = 42;
        println!("inside macro: local_var = {local_var}");
    };
}

fn main() {
    let local_var = 100;
    println!("before macro: local_var = {local_var}");

    increment_locally!();

    println!("after macro: local_var = {local_var}");
}
```

Run the program. Expected output:

```
before macro: local_var = 100
inside macro: local_var = 42
after macro: local_var = 100
```

The caller's `local_var = 100` and the macro's `local_var = 42` coexist without collision. The macro's `local_var` is in a separate "syntax context" from the caller's.

This is a feature, not a bug. Hygiene means you can write macros without worrying about variable name conflicts in your callers' code.

### Task 2.3 -- The escape hatch (caller variables)

When the macro DOES want to introduce a variable visible to the caller, it must take the variable name as a parameter:

```rust
macro_rules! make_var {
    ($name:ident, $value:expr) => {
        let $name = $value;
    };
}

fn main() {
    make_var!(x, 42);
    make_var!(message, String::from("hello"));

    println!("x = {x}");
    println!("message = {message}");
}
```

Run the program. Expected output:

```
x = 42
message = hello
```

The `$name:ident` fragment captures an identifier (a variable name) from the caller. The macro's body `let $name = $value;` then becomes `let x = 42;` or `let message = String::from("hello")` at the call site.

The new variable IS visible to the caller because the identifier came from the caller's syntax context. This is the "controlled" way to introduce variables: by accepting the name as input rather than hard-coding it.

This pattern is rare in practice. Most macros do not introduce variables visible to callers; the hygiene rules make it the default. Macros that DO introduce visible variables (like `vec!`) take the names from the caller's perspective.

### Task 2.4 -- A practical use of hygiene

A macro that creates a "scoped variable" for timing:

```rust
use std::time::Instant;

macro_rules! time_block {
    ($label:expr, $body:block) => {{
        let start = Instant::now();
        let result = $body;
        let elapsed = start.elapsed();
        println!("[{}] took {:?}", $label, elapsed);
        result
    }};
}

fn main() {
    let start = Instant::now();   // caller's `start`

    let total = time_block!("sum 1..1_000_000", {
        (1u64..=1_000_000).sum::<u64>()
    });

    println!("caller's start was {:?} ago", start.elapsed());
    println!("total = {total}");
}
```

Run the program. Expected output (timings will vary):

```
[sum 1..1_000_000] took 3.45ms
caller's start was 3.46ms ago
total = 500000500000
```

The macro creates its own `start` variable (and `elapsed`, `result`). These do not collide with the caller's `start` variable. The macro reports its own elapsed time; the caller can still use its own `start`.

Without hygiene, the macro's `let start = Instant::now();` would shadow the caller's `start`, breaking the caller's subsequent code. Rust's hygiene prevents this.

### Checkpoints

1. Hygiene means the macro's internal names do not collide with the caller's names. Why is this a problem in C-style macros that Rust solves?
2. The escape hatch in Task 2.3 takes the variable name as an `$name:ident` parameter. Why does this work to "break" hygiene, when the macro's internal `let temp = ...` does not?
3. Hygiene has a cost: macros cannot easily "share" variables with their callers, even when this would be convenient. What is gained that justifies this restriction?

---

## Exercise 3 -- The Common Built-In Macros

**Estimated time:** 10-15 minutes
**Topics covered:** `vec!`, `println!`, `format!`, `assert_eq!`, `dbg!`, `todo!`, `unimplemented!`

### Context

The standard library provides several built-in macros that appear in almost every Rust program. This exercise covers what each does and why it must be a macro rather than a function.

### Task 3.1 -- vec!

The `vec!` macro creates `Vec<T>` instances:

```rust
fn main() {
    // Three forms:
    let list1: Vec<i32> = vec![1, 2, 3, 4, 5];          // comma-separated values
    let list2: Vec<i32> = vec![0; 10];                   // 10 zeros
    let empty: Vec<i32> = vec![];                        // empty

    println!("list1: {list1:?}");
    println!("list2: {list2:?}");
    println!("empty: {empty:?}");
}
```

Run the program. Expected output:

```
list1: [1, 2, 3, 4, 5]
list2: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
empty: []
```

`vec!` accepts three forms:

- `vec![a, b, c]`: a list of comma-separated values.
- `vec![value; count]`: a repeated value (semicolon syntax).
- `vec![]`: empty.

Why is `vec!` a macro rather than a function?

- **Variable arity.** Rust functions cannot accept any number of arguments. A macro can.
- **The semicolon form.** `vec![0; 10]` uses a non-function-like syntax. A function call cannot have a semicolon between arguments.
- **Compile-time type inference.** The macro can determine the element type from the values; a function's signature would need to be more specific.

You could write a function for one of these forms (e.g., `Vec::from(&[1, 2, 3])`), but you would lose the unified syntax.

### Task 3.2 -- println! and format!

These are closely related; one prints, the other returns a `String`:

```rust
fn main() {
    let name = "Alice";
    let age = 30;

    // println!: print to stdout with formatting:
    println!("Hello, {name}! You are {age} years old.");

    // format!: same syntax, returns a String:
    let message: String = format!("Hello, {name}! You are {age} years old.");
    println!("the message is: {message}");

    // Various format specifiers:
    println!("hex: {:#x}", 255);                  // 0xff
    println!("padded: {:>10}", "hi");              // (right-aligned in 10 chars)
    println!("float precision: {:.3}", 3.14159);   // 3.142
}
```

Run the program. Expected output:

```
Hello, Alice! You are 30 years old.
the message is: Hello, Alice! You are 30 years old.
hex: 0xff
padded:         hi
float precision: 3.142
```

These are macros because:

- **Compile-time format string checking.** The compiler verifies that `{}` placeholders match the number of arguments and their types. With `println!("{} {}", x)`, you get a compile error: too few arguments.
- **Variable arity.** Any number of arguments can be passed.
- **Specialized formatting.** Each placeholder can have format specifiers (`{:?}`, `{:.2}`, `{:#x}`); these are parsed at compile time.

A function-based equivalent would have to defer everything to runtime, losing the compile-time checks.

Try this:

```rust
println!("Hello, {name}! Bye, {missing}!");      // ERROR
```

The compiler catches the missing variable at compile time:

```
error: cannot find value `missing` in this scope
```

Without macros, this kind of check would happen at runtime, and the error message would be confusing or absent.

### Task 3.3 -- assert_eq! and dbg!

Two macros that appear in tests and debugging:

```rust
fn main() {
    let x = 5;
    let y = 5;
    assert_eq!(x, y);                              // passes silently

    // dbg! prints and returns its argument:
    let z = dbg!(x * 2);
    println!("z = {z}");

    // Often used as a quick-inspect tool:
    let total: i32 = (1..=10).map(|n| dbg!(n * n)).sum();
    println!("total of squares: {total}");
}
```

Run the program. Expected output (your line numbers will vary):

```
[src/main.rs:7] x * 2 = 10
z = 10
[src/main.rs:11] n * n = 1
[src/main.rs:11] n * n = 4
[src/main.rs:11] n * n = 9
...
[src/main.rs:11] n * n = 100
total of squares: 385
```

`assert_eq!` is a macro because it needs to:

- Take two values of any type implementing `PartialEq + Debug`.
- Print BOTH values on failure (using the source code as labels).

`dbg!` is a macro because it needs to:

- Capture the source text of its argument (to print as the label).
- Print AND return the value, so it can be used inline in expressions.

Both features require source-code access, which only macros have. A function has only the value, not the expression that produced it.

### Task 3.4 -- todo! and unimplemented!

Placeholders for code you have not written:

```rust
fn process_input(input: i32) -> String {
    if input < 0 {
        unimplemented!("negative inputs not handled yet")
    } else if input == 0 {
        todo!()
    } else {
        format!("positive: {input}")
    }
}

fn main() {
    let positive = process_input(5);
    println!("{positive}");

    // The next line would panic at runtime:
    // let zero = process_input(0);
}
```

Run the program. Expected output:

```
positive: 5
```

Both macros are placeholders that compile but panic if executed. They satisfy the type checker without requiring you to write working code.

The semantic difference:

- `todo!()` says "I plan to write this; the work is in progress."
- `unimplemented!()` says "this case is deliberately not handled (yet)."

Both panic with informative messages. Both return `!` (the never type), so they work in any return-value context.

Try uncommenting the `process_input(0)` line. The program panics:

```
thread 'main' panicked at 'not yet implemented'
```

These are essential during incremental development. You sketch out an API with `todo!()` bodies, the code compiles, and you fill in the bodies one at a time.

### Task 3.5 -- include_str! and env!

Compile-time data inclusion:

```rust
const VERSION: &str = env!("CARGO_PKG_VERSION");

fn main() {
    println!("event_dsl version: {VERSION}");

    // include_str! embeds a file's contents at compile time.
    // Not run for this lab (we have no file to include), but the pattern is:
    //   const README: &str = include_str!("../README.md");
    //   println!("{README}");
}
```

Run the program. Expected output:

```
event_dsl version: 0.1.0
```

`env!` reads an environment variable AT COMPILE TIME and embeds its value into the binary. `CARGO_PKG_VERSION` is set by Cargo to your package's version (from `Cargo.toml`).

`include_str!` is similar but reads a file. Both are macros because they execute at compile time, not at runtime. Functions cannot do compile-time work; macros can.

### Checkpoints

1. The `vec![value; count]` syntax uses a semicolon between the value and the count. Why is this not just `vec_repeat(value, count)`?
2. `println!` checks at compile time that `{}` placeholders match arguments. What kind of bug does this prevent? In a language without this check, when would the bug appear?
3. `dbg!` requires the source text of its argument to produce useful output ("`x * 2 = 10`"). Why can a macro do this but a function cannot?

---

## Exercise 4 -- Derive Macros for Your Own Types

**Estimated time:** 10-15 minutes
**Topics covered:** `#[derive(...)]`, what derives generate, derive limitations

### Context

You have used `#[derive(Debug)]` and others throughout the course. These are procedural macros that generate trait implementations for your type. This exercise looks more closely at what they produce and when they fail.

### Task 4.1 -- The basic derives

Replace `src/main.rs` with:

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 3, y: 4 };

    println!("{p:?}");        // uses Debug
    println!("{p:#?}");        // uses pretty-printed Debug
}
```

Run the program. Expected output:

```
Point { x: 3, y: 4 }
Point {
    x: 3,
    y: 4,
}
```

The `#[derive(Debug)]` attribute generated an `impl Debug for Point` that prints the struct's fields. Without the derive, you would have to write the impl manually:

```rust
impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

The derive generates exactly this code. Twelve lines saved per struct.

### Task 4.2 -- Multiple derives

Several derives can be combined:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct UserId(u64);

fn main() {
    let a = UserId(42);
    let b = a;                                    // Copy
    let c = a.clone();                            // Clone

    println!("a == b: {}", a == b);               // PartialEq
    println!("a: {a:?}");                          // Debug

    // Use as a HashMap key (requires Hash + Eq):
    let mut map = std::collections::HashMap::new();
    map.insert(a, "first");
    map.insert(UserId(100), "second");

    println!("a is in map: {:?}", map.get(&a));
}
```

Run the program. Expected output:

```
a == b: true
a: UserId(42)
a is in map: Some("first")
```

Each derive generates the corresponding trait implementation:

- `Debug` for `{:?}` printing.
- `Clone` for `.clone()` (duplicating).
- `Copy` for implicit copying on assignment (only valid when all fields are `Copy`).
- `PartialEq` for `==` and `!=`.
- `Eq` for "this type satisfies equivalence laws" (required for `HashMap` keys).
- `Hash` for hashing (required for `HashMap` keys).

This combination (the "standard derive set") is common for any value-like type. Together they give you a type that can be printed, copied, compared, and used as a HashMap key.

### Task 4.3 -- When derives fail

Some derives have requirements. `Copy` requires all fields to be `Copy`:

```rust
#[derive(Debug, Clone, Copy)]
struct User {
    id: u64,
    name: String,           // ERROR: String is not Copy
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0204]: the trait `Copy` may not be implemented for this type
  |
  | name: String,
  |       ------ this field does not implement `Copy`
```

The error mentions the field that violates the requirement. `String` owns heap data; copying it would create two owners of the same heap buffer, which is incorrect.

The fix is to either remove `Copy` (keep `Clone` only) or change the field. For most struct designs, `Clone` is sufficient; only types with truly trivial bit-by-bit copies should be `Copy`.

Remove the `Copy` derive:

```rust
#[derive(Debug, Clone)]
struct User {
    id: u64,
    name: String,
}

fn main() {
    let u1 = User { id: 1, name: String::from("Alice") };
    let u2 = u1.clone();    // explicit clone

    println!("u1: {u1:?}");
    println!("u2: {u2:?}");
}
```

Run the program. Expected output:

```
u1: User { id: 1, name: "Alice" }
u2: User { id: 1, name: "Alice" }
```

`Clone` works because each field implements `Clone`. The derive walks the fields and generates the appropriate `clone()` implementation.

### Task 4.4 -- Enums and derives

Enums also support derives:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
enum Color {
    Red,
    Green,
    Blue,
    Custom(u8, u8, u8),
}

fn main() {
    let c1 = Color::Red;
    let c2 = Color::Custom(255, 128, 0);

    println!("{c1:?}");
    println!("{c2:?}");

    println!("c1 == Color::Red: {}", c1 == Color::Red);
    println!("c1 == c2: {}", c1 == c2);
}
```

Run the program. Expected output:

```
Red
Custom(255, 128, 0)
c1 == Color::Red: true
c1 == c2: false
```

For enums, the derive handles each variant. `Debug` shows the variant name and data. `PartialEq` compares variants (same variant with same data = equal).

The variant data must satisfy the same requirements as struct fields. `Color::Custom(u8, u8, u8)` is `Copy` because `u8` is `Copy`. An enum with a `String` variant would not be `Copy`.

### Task 4.5 -- A non-standard derive

Different types might need different derive combinations:

```rust
#[derive(Debug, Clone, PartialOrd, PartialEq)]
struct Score(f64);

fn main() {
    let s1 = Score(85.5);
    let s2 = Score(92.3);

    println!("s1: {s1:?}");
    println!("s2 > s1: {}", s2 > s1);
    println!("s2 == Score(92.3): {}", s2 == Score(92.3));
}
```

Run the program. Expected output:

```
s1: Score(85.5)
s2 > s1: true
s2 == Score(92.3): true
```

`PartialOrd` enables comparison operators (`<`, `>`, `<=`, `>=`). It is "partial" because some comparisons might not be defined (`f64` has NaN, which is not comparable with anything). For `f64` types, `PartialOrd` is appropriate; `Ord` is not (it requires total ordering).

For integer-based types, you would typically use `Ord` instead. The choice between `PartialOrd` and `Ord` depends on whether the type has well-behaved ordering.

### Task 4.6 -- What derives cannot do

Some traits cannot be derived. `Display`, for example:

```rust
#[derive(Display)]              // ERROR: Display cannot be derived
struct Greeting {
    name: String,
}
```

Run `cargo build`. The compiler rejects this:

```
error: cannot find derive macro `Display` in this scope
```

`Display` requires user-facing formatting; there is no canonical format for an arbitrary type. The author must decide: should `Greeting { name: "Alice" }` print as "Alice", "Hello, Alice!", or something else? The standard library does not guess.

You must write the impl manually:

```rust
use std::fmt;

struct Greeting {
    name: String,
}

impl fmt::Display for Greeting {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Hello, {}!", self.name)
    }
}

fn main() {
    let g = Greeting { name: String::from("Alice") };
    println!("{g}");
}
```

Run the program. Expected output:

```
Hello, Alice!
```

The general rule: traits with canonical implementations can be derived; traits with semantic choices (like display formatting) cannot.

### Checkpoints

1. The standard derive set (`Debug, Clone, Copy, PartialEq, Eq, Hash`) is common for "value types." Why is this combination so frequent? When would you NOT include `Copy`?
2. `#[derive(Copy)]` requires all fields to be `Copy`. The compiler enforces this. Why is this rule strict? What goes wrong if you could derive `Copy` for a type with a `String` field?
3. `Display` cannot be derived because there is no canonical format. `Debug` can be derived. What is different about Debug that allows derivation? What does this say about the design intent of each trait?

---

## Exercise 5 -- Third-Party Derive Macros

**Estimated time:** 15 minutes
**Topics covered:** `serde`, `thiserror`, helper attributes, what procedural macros can do that declarative ones cannot

### Context

The standard library provides a few derives; third-party crates provide many more. This exercise uses `serde` (for serialization) and `thiserror` (for error types) to see what custom derives look like in practice.

### Task 5.1 -- Add serde and thiserror

Update `Cargo.toml`:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
```

Save and run `cargo build`. Cargo downloads the crates.

### Task 5.2 -- Serialize with serde

Replace `src/main.rs` with:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Event {
    timestamp: u64,
    kind: String,
    data: EventData,
}

#[derive(Debug, Serialize, Deserialize)]
struct EventData {
    user_id: u64,
    action: String,
}

fn main() {
    let event = Event {
        timestamp: 1700000000,
        kind: String::from("user.action"),
        data: EventData {
            user_id: 42,
            action: String::from("login"),
        },
    };

    // Serialize to JSON:
    let json = serde_json::to_string_pretty(&event).unwrap();
    println!("Serialized:");
    println!("{json}");

    // Deserialize back:
    let parsed: Event = serde_json::from_str(&json).unwrap();
    println!("\nDeserialized:");
    println!("{parsed:?}");
}
```

Run the program. Expected output:

```
Serialized:
{
  "timestamp": 1700000000,
  "kind": "user.action",
  "data": {
    "user_id": 42,
    "action": "login"
  }
}

Deserialized:
Event { timestamp: 1700000000, kind: "user.action", data: EventData { user_id: 42, action: "login" } }
```

The `#[derive(Serialize, Deserialize)]` attributes generate the necessary code to convert between Rust types and JSON (or any other format serde supports).

What this derive does is substantial. For `Event`, it generates:

- An `impl Serialize for Event` that visits each field and asks serde to serialize it.
- An `impl Deserialize for Event` that parses each field from the input format.

The generated code is dozens to hundreds of lines per struct. Writing it by hand would be tedious and error-prone. The derive does it perfectly.

You would not see this with a declarative macro; serde's derive is procedural. The macro inspects the struct's fields, figures out their types, and generates type-specific serialization code.

### Task 5.3 -- Helper attributes

`serde` has many "helper attributes" for customizing serialization:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    #[serde(rename = "userId")]
    id: u64,

    name: String,

    #[serde(default)]
    age: Option<u32>,

    #[serde(skip_serializing_if = "Vec::is_empty")]
    tags: Vec<String>,
}

fn main() {
    // A user with all fields:
    let full = User {
        id: 1,
        name: String::from("Alice"),
        age: Some(30),
        tags: vec![String::from("admin"), String::from("user")],
    };

    // A user with no tags:
    let sparse = User {
        id: 2,
        name: String::from("Bob"),
        age: None,
        tags: vec![],
    };

    println!("Full user:");
    println!("{}", serde_json::to_string_pretty(&full).unwrap());

    println!("\nSparse user:");
    println!("{}", serde_json::to_string_pretty(&sparse).unwrap());

    // Deserialize JSON missing the `age` field:
    let partial_json = r#"{"userId": 3, "name": "Charlie", "tags": []}"#;
    let charlie: User = serde_json::from_str(partial_json).unwrap();
    println!("\nDeserialized partial:");
    println!("{charlie:?}");
}
```

Run the program. Expected output:

```
Full user:
{
  "userId": 1,
  "name": "Alice",
  "age": 30,
  "tags": [
    "admin",
    "user"
  ]
}

Sparse user:
{
  "userId": 2,
  "name": "Bob",
  "age": null
}

Deserialized partial:
User { id: 3, name: "Charlie", age: None, tags: [] }
```

Three helper attributes demonstrated:

- **`#[serde(rename = "userId")]`** changes the field's name in the output (Rust uses `snake_case`; many APIs use `camelCase`).
- **`#[serde(default)]`** on `age` means "if missing from input, use the type's default" (`Option::default()` is `None`).
- **`#[serde(skip_serializing_if = "Vec::is_empty")]`** on `tags` means "do not include this field in output if it is empty." The `sparse` user's `tags` field is missing from the JSON.

These attributes are processed by the `serde` derive macro. They are not generic Rust syntax; they are specific to serde. Each derive macro defines its own helper attributes.

### Task 5.4 -- thiserror for error types

`thiserror` provides a `#[derive(Error)]` that generates `Display` and `std::error::Error` implementations:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum ParseError {
    #[error("file is empty")]
    Empty,

    #[error("expected number, got '{0}'")]
    NotANumber(String),

    #[error("number {value} out of range (max {max})")]
    OutOfRange { value: i64, max: i64 },

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}

fn parse_value(s: &str) -> Result<i64, ParseError> {
    if s.is_empty() {
        return Err(ParseError::Empty);
    }

    let value: i64 = s.parse().map_err(|_| ParseError::NotANumber(s.to_string()))?;

    if value > 100 {
        return Err(ParseError::OutOfRange { value, max: 100 });
    }

    Ok(value)
}

fn main() {
    let inputs = ["42", "", "hello", "1000"];

    for input in &inputs {
        match parse_value(input) {
            Ok(value) => println!("'{input}' -> {value}"),
            Err(e) => println!("'{input}' -> error: {e}"),
        }
    }
}
```

Run the program. Expected output:

```
'42' -> 42
'' -> error: file is empty
'hello' -> error: expected number, got 'hello'
'1000' -> error: number 1000 out of range (max 100)
```

The `#[derive(Error)]` and `#[error("...")]` attributes generate the `Display` implementation. The format strings use the variant's data (named fields like `value`, or positional like `0`).

What thiserror's derive generates is roughly:

```rust
impl std::fmt::Display for ParseError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ParseError::Empty => write!(f, "file is empty"),
            ParseError::NotANumber(s) => write!(f, "expected number, got '{}'", s),
            ParseError::OutOfRange { value, max } => write!(f, "number {} out of range (max {})", value, max),
            ParseError::Io(err) => write!(f, "I/O error: {}", err),
        }
    }
}

impl std::error::Error for ParseError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ParseError::Io(err) => Some(err),
            _ => None,
        }
    }
}
```

Writing this by hand for every error type would be tedious. `thiserror` handles it with a few attributes per variant.

The `#[from]` attribute on the `Io` variant is also significant. It generates `From<std::io::Error> for ParseError`, enabling `?` to convert IO errors into `ParseError::Io` automatically.

### Task 5.5 -- The procedural macro generates real code

To see how much code `#[derive(Serialize)]` generates, you can use the `cargo-expand` tool. (Installation: `cargo install cargo-expand`. This tool is optional; the lab does not require it.)

If you run `cargo expand` on a project, you see the full expanded code, including everything that derive macros generated. For even a small struct with `#[derive(Serialize, Deserialize)]`, the expansion is typically 50-100 lines.

This visibility is useful when debugging mysterious errors. If a derive's generated code does not compile, `cargo expand` shows you what it actually generated, so you can read it.

For this lab, just be aware that `cargo expand` exists. You will encounter situations later in your Rust career where it is essential.

### Checkpoints

1. The `#[derive(Serialize, Deserialize)]` on a struct generates substantial code. A declarative macro (`macro_rules!`) could not do this. What specifically about serde's needs requires a procedural macro?
2. Helper attributes like `#[serde(rename = "...")]` configure the derive's output. The attribute is `serde`-specific; the compiler does not enforce it. What does this tell you about how procedural macros work?
3. `thiserror`'s `#[error("{0}")]` and `#[from]` attributes are small but powerful. They replace what would be lines of `Display` and `From` impls. What is the trade-off between using `thiserror` versus writing the impls by hand?

---

## Exercise 6 -- Attribute Macros and Function-Like Macros

**Estimated time:** 10 minutes
**Topics covered:** `#[tokio::main]`, `#[test]`, function-like macros that look like declarative ones

### Context

Two more kinds of procedural macros: attribute macros (applied to items with `#[name]`) and function-like macros (called with `name!(...)`). You have used both throughout the course.

### Task 6.1 -- Attribute macros: `#[test]`

The `#[test]` attribute on a function marks it as a test:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add_basic() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn test_add_zero() {
        assert_eq!(add(5, 0), 5);
    }
}
```

Replace `src/main.rs` with the above (keep your imports but otherwise replace the body). Wait — this needs a `main` function. Add:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    println!("normal run: add(2, 3) = {}", add(2, 3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add_basic() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn test_add_zero() {
        assert_eq!(add(5, 0), 5);
    }
}
```

Run:

```bash
cargo run
```

Expected output:

```
normal run: add(2, 3) = 5
```

Then:

```bash
cargo test
```

Expected output:

```
running 2 tests
test tests::test_add_basic ... ok
test tests::test_add_zero ... ok

test result: ok. 2 passed; 0 failed; 0 ignored
```

The `#[test]` attribute does NOT modify the function's code. It marks the function for collection by the test harness. When you run `cargo test`, the test harness generates a `main` function that calls all the test-marked functions.

This is exactly what a procedural attribute macro does: it transforms or annotates the item it is applied to. For `#[test]`, the transformation is "register this function with the test harness."

### Task 6.2 -- Attribute macros: tokio::main

A more dramatic example. The `#[tokio::main]` attribute transforms an async `main` function into a synchronous one that runs on the Tokio runtime.

You used this in Lab 10. The user writes:

```rust
#[tokio::main]
async fn main() {
    // async code
}
```

The macro transforms this into:

```rust
fn main() {
    let runtime = tokio::runtime::Runtime::new().unwrap();
    runtime.block_on(async {
        // async code
    });
}
```

The async function becomes a synchronous wrapper that creates a Tokio runtime and runs the async body on it.

You will not write this transformation yourself; the `tokio` crate provides it. But understanding what the attribute does is important: it is not just a "label" on the function; it actively rewrites the function's body.

You can see this in Lab 10's solutions if you have it open. The async `main` works because of the macro transformation, not because Rust supports async main natively (it does not).

### Task 6.3 -- Function-like procedural macros

These look identical to declarative macros at the call site (with `!`) but are implemented as procedural macros. Common examples:

- **`sqlx::query!("SELECT ...")`** validates the SQL at compile time against your database schema.
- **`tracing::info!("message")`** logs structured data with compile-time format checking.
- **`html! { <div>...</div> }`** (from various HTML-templating libraries) parses HTML-like syntax.

You cannot tell at the call site whether `name!(...)` is declarative or procedural. The difference is in how the macro is implemented (`macro_rules!` for declarative; a procedural macro crate for the other).

For users, the distinction usually does not matter. Both are macros that take input and produce code. The procedural ones can do more (parse arbitrary syntax, run programs at compile time, etc.), but the call-site syntax is the same.

To illustrate, consider this declarative macro:

```rust
macro_rules! print_section {
    ($title:literal: $($line:expr);*) => {
        println!("=== {} ===", $title);
        $(
            println!("  {}", $line);
        )*
    };
}

fn main() {
    print_section!("First Section":
        "Item 1";
        "Item 2";
        "Item 3"
    );

    print_section!("Empty Section":);

    print_section!("Single":
        "Only one"
    );
}
```

Run the program. Expected output:

```
=== First Section ===
  Item 1
  Item 2
  Item 3
=== Empty Section ===
=== Single ===
  Only one
```

This is a declarative macro that creates a small DSL. A procedural macro could do something more complex (validate the title against a database, generate boilerplate based on input syntax, etc.), but for simple DSLs, declarative is sufficient and simpler.

### Task 6.4 -- Recognizing macro kinds in the wild

When reading Rust code, you will encounter all three kinds:

**Declarative macros (defined with `macro_rules!`):**

- Usually small, defined in the same crate or in `std`.
- Examples: `vec!`, `assert_eq!`, `format!`, custom application-specific macros.
- Limitations: pattern-based; cannot inspect types or generate complex code.

**Procedural macros (in their own crate with `proc-macro = true`):**

- Defined in a separate crate.
- Examples: `#[derive(Serialize)]`, `#[tokio::main]`, `sqlx::query!`.
- Powers: can inspect AST, generate arbitrary code, even run programs at compile time.

**Attribute macros (a procedural macro applied as an attribute):**

- Used with `#[name]` or `#[name(args)]` syntax.
- Transforms the item they are applied to.
- Examples: `#[test]`, `#[tokio::main]`, `#[async_trait]`.

**Function-like macros (procedural macros called like declarative ones):**

- Used with `name!(...)` syntax.
- Take a token stream as input, return a token stream.
- Examples: `sqlx::query!`, `html!` from various crates.

You do not need to write these distinctions on every macro you encounter. But knowing the categories helps when you read documentation or error messages.

### Checkpoints

1. `#[test]` is an attribute macro that marks functions for the test harness. The function looks unchanged when you read the source. What is happening behind the scenes that makes the test "discoverable"?
2. `#[tokio::main]` actively rewrites the function body. Reading `async fn main() { ... }`, you cannot tell that the function will not run as written. Why is this transparent rewriting acceptable for `#[tokio::main]` but might be concerning in other attribute macros?
3. A function-like procedural macro looks identical to a declarative macro at the call site. From a user's perspective, the distinction is invisible. Why does the distinction exist at all? Why not have just one kind?

---

## Exercise 7 -- Macros vs. Functions vs. Generics

**Estimated time:** 10-15 minutes
**Topics covered:** when to use macros, when to use functions, when to use generics

### Context

Macros, functions, and generics all let you write code that is reused. They have different trade-offs. This exercise builds the same conceptual operation three ways to compare them.

### Task 7.1 -- The function approach

Suppose you want to print any value of a specific type with a label:

```rust
fn print_labeled(label: &str, value: i32) {
    println!("[{label}] {value}");
}

fn main() {
    print_labeled("count", 42);
    print_labeled("total", 1000);
}
```

Run the program. Expected output:

```
[count] 42
[total] 1000
```

This works, but only for `i32`. To print labels for other types, you would need separate functions:

```rust
fn print_labeled_str(label: &str, value: &str) {
    println!("[{label}] {value}");
}

fn print_labeled_f64(label: &str, value: f64) {
    println!("[{label}] {value}");
}
```

Writing one function per type does not scale.

### Task 7.2 -- The generic approach

A generic function takes any type that implements a required trait:

```rust
use std::fmt::Display;

fn print_labeled<T: Display>(label: &str, value: T) {
    println!("[{label}] {value}");
}

fn main() {
    print_labeled("count", 42);                    // T = i32
    print_labeled("total", 1000_u64);              // T = u64
    print_labeled("ratio", 3.14);                  // T = f64
    print_labeled("name", "Alice");                // T = &str
    print_labeled("flag", true);                   // T = bool
}
```

Run the program. Expected output:

```
[count] 42
[total] 1000
[ratio] 3.14
[name] Alice
[flag] true
```

The function works for any type implementing `Display`. The compiler generates specialized versions for each type used (this is monomorphization). The trait bound enforces that the type can be printed.

Generics handle the "any type satisfying a trait" case elegantly. For a single value of a known shape, generics are usually the right tool.

### Task 7.3 -- The macro approach

A macro can do something functions cannot: take multiple values with labels in one call:

```rust
macro_rules! print_labeled {
    ($($label:literal: $value:expr),* $(,)?) => {
        $(
            println!("[{}] {}", $label, $value);
        )*
    };
}

fn main() {
    print_labeled! {
        "count": 42,
        "total": 1000,
        "name": "Alice",
        "ratio": 3.14,
    };
}
```

Run the program. Expected output:

```
[count] 42
[total] 1000
[name] Alice
[ratio] 3.14
```

The macro accepts a variable number of `label: value` pairs. Each pair is printed. The `$(,)?` at the end is a trailing-comma allowance (zero or one comma after the last item).

What the macro provides that the function cannot:

- **Multiple values in one call.** A function takes a fixed number of arguments; a macro can take any number.
- **Mixed types in one call.** The macro accepts `42` (i32), `"Alice"` (&str), and `3.14` (f64) in the same call. A generic function with `T: Display` could not (each call has one `T`).

This is the variadic-arguments use case where macros are the right tool.

### Task 7.4 -- Trade-off summary

For the "print a label and value" operation:

| Approach | Pros | Cons |
|---|---|---|
| Function (per type) | Simple, clear, fast | One function per type; tedious to maintain |
| Generic function | Works for any `Display` type; fast; clear errors | One call = one type |
| Declarative macro | Variadic, mixed types in one call | More complex syntax; less clear errors |

The right choice depends on what you actually need.

- For "one value, one type": a generic function (or even a regular function if only one type is involved).
- For "multiple values, mixed types, one call": a macro is necessary.

The standard library follows this rule strictly. `println!` is a macro because it must accept variadic arguments and mixed types. `to_string` is a method (defined by the `ToString` trait, which has a blanket impl from `Display`) because it operates on a single value.

### Task 7.5 -- A more substantial comparison

A larger example: a function that finds the maximum of a series of values.

**Generic function (one type only):**

```rust
fn max<T: PartialOrd>(values: &[T]) -> Option<&T> {
    if values.is_empty() {
        return None;
    }
    let mut largest = &values[0];
    for v in &values[1..] {
        if v > largest {
            largest = v;
        }
    }
    Some(largest)
}

fn main() {
    let nums = vec![3, 1, 4, 1, 5, 9, 2, 6];
    println!("max: {:?}", max(&nums));

    let words = vec!["banana", "apple", "cherry"];
    println!("max: {:?}", max(&words));
}
```

This works for any `T: PartialOrd`. The values must all be the same type.

**Macro (variadic, mixed types... but with limitations):**

```rust
macro_rules! max_all {
    ($first:expr $(, $rest:expr)*) => {{
        let mut largest = $first;
        $(
            if $rest > largest {
                largest = $rest;
            }
        )*
        largest
    }};
}

fn main() {
    let result = max_all!(3, 1, 4, 1, 5, 9, 2, 6);
    println!("max: {result}");

    // This works because all values are the same type:
    let words = max_all!("banana", "apple", "cherry");
    println!("max word: {words}");
}
```

This is more convenient at the call site (no need to create a vector first), but it has limitations:

- All values must still be the same type (mixing produces compile errors).
- Errors point inside the macro expansion, which is confusing.
- The macro's behavior is opaque (a function with a clear signature is more readable).

The macro is appropriate only if the calling convenience is worth the trade-offs. For most cases, the generic function is preferable.

### Task 7.6 -- A decision framework

When deciding between a macro and a function:

**Use a function (possibly generic) when:**

- A fixed number of arguments works.
- All inputs are the same type, or you can express the relationship with traits.
- You want clear error messages and IDE support.
- You want the code to be testable and refactorable normally.

**Consider a macro when:**

- You need variadic arguments.
- You need source-code capture (for debugging, formatting, or compile-time analysis).
- You need compile-time evaluation (reading files, environment variables, etc.).
- You are generating boilerplate across many types (consider a derive macro, even).
- You are building a DSL where Rust's normal syntax does not fit.

**Reach for a macro reluctantly.** Functions are easier to read, test, and debug. Macros add cognitive load; they should pull their weight.

A common antipattern: writing a macro for what is really a function. For example:

```rust
// Antipattern: macro for a simple computation
macro_rules! double {
    ($x:expr) => { $x * 2 };
}

// Better: a generic function
fn double<T: std::ops::Mul<i32, Output = T>>(x: T) -> T {
    x * 2
}
```

The macro version offers no benefit. The function works, has a clear signature, integrates with tooling, and produces clear errors. The macro is just noise.

### Checkpoints

1. The macro version of `max_all!` in Task 7.5 was more convenient at the call site than the function version. What did the convenience cost in terms of code quality?
2. The decision framework in Task 7.6 suggested using a macro "reluctantly." Why are macros recommended only as a last resort? What does a function give you that a macro does not?
3. The standard library has both `vec!` (macro) and `Vec::from(&[...])` (associated function). Both can build a vector. Why does the macro form exist alongside the function?

---

## Exercise 8 -- Putting It Together

**Estimated time:** 15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from Module 18 into a single program: a custom declarative macro alongside standard library macros, third-party derive macros, an attribute macro, and a function-like procedural macro from a third-party crate. The program is a small event recording system that demonstrates each macro type in context.

### Task 8.1 -- The full program

Replace your `src/main.rs` with this complete program (and ensure `Cargo.toml` has `serde`, `serde_json`, and `thiserror`):

```rust
use serde::{Serialize, Deserialize};
use thiserror::Error;

// ============================================================
// Custom declarative macro: builds event records
// ============================================================

macro_rules! event {
    // Basic form: event!(kind, user, action)
    ($kind:expr, $user:expr, $action:expr) => {
        Event::new($kind, $user, $action)
    };

    // Detailed form with extra fields:
    // event!(kind, user, action, metadata: { key1 => value1, key2 => value2 })
    ($kind:expr, $user:expr, $action:expr, metadata: { $($key:expr => $value:expr),* $(,)? }) => {{
        let mut event = Event::new($kind, $user, $action);
        $(
            event.metadata.insert($key.to_string(), $value.to_string());
        )*
        event
    }};
}

// ============================================================
// Domain types using third-party derive macros
// ============================================================

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Event {
    kind: String,
    user: String,
    action: String,
    metadata: std::collections::HashMap<String, String>,
}

impl Event {
    fn new(kind: &str, user: &str, action: &str) -> Event {
        Event {
            kind: kind.to_string(),
            user: user.to_string(),
            action: action.to_string(),
            metadata: std::collections::HashMap::new(),
        }
    }
}

// Custom error type using thiserror's derive:
#[derive(Debug, Error)]
enum EventError {
    #[error("event has no kind specified")]
    NoKind,

    #[error("user '{0}' is not authorized for action '{1}'")]
    Unauthorized(String, String),

    #[error("serialization error: {0}")]
    SerializeError(#[from] serde_json::Error),
}

// ============================================================
// Application code that uses everything
// ============================================================

fn validate_event(event: &Event) -> Result<(), EventError> {
    if event.kind.is_empty() {
        return Err(EventError::NoKind);
    }

    if event.action == "admin" && event.user != "root" {
        return Err(EventError::Unauthorized(event.user.clone(), event.action.clone()));
    }

    Ok(())
}

fn process_event(event: &Event) -> Result<String, EventError> {
    validate_event(event)?;
    let json = serde_json::to_string_pretty(event)?;
    Ok(json)
}

fn main() {
    println!("=== Event Recording System ===\n");

    // Use the custom event! macro:
    let login = event!("user.action", "alice", "login");

    let upload = event!("user.action", "bob", "upload", metadata: {
        "file" => "document.pdf",
        "size_bytes" => "1024",
        "checksum" => "abc123",
    });

    let unauthorized = event!("admin.action", "alice", "admin");

    let events = vec![login, upload, unauthorized];

    // Use vec! and various standard library macros:
    println!("Processing {} events:\n", events.len());

    for (i, event) in events.iter().enumerate() {
        println!("--- Event {} ---", i + 1);
        println!("{:?}", event);

        // Use debug-print and process:
        let result = dbg!(process_event(event));

        match result {
            Ok(json) => {
                println!("\nJSON output:");
                println!("{json}");
            }
            Err(e) => {
                // Use println! with formatted error:
                println!("\nValidation failed: {e}");
            }
        }
        println!();
    }

    // Verify assertions about our system:
    let test_event = event!("test", "user", "ping");
    assert_eq!(test_event.kind, "test");
    assert_eq!(test_event.user, "user");

    // Use todo! for unfinished functionality:
    fn future_feature(_event: &Event) -> Result<String, EventError> {
        todo!("event aggregation across multiple events")
    }

    let _ = future_feature; // suppress unused warning

    println!("=== Done ===");
}
```

Run the program:

```bash
cargo run
```

Expected output (your line numbers and JSON formatting may vary):

```
=== Event Recording System ===

Processing 3 events:

--- Event 1 ---
Event { kind: "user.action", user: "alice", action: "login", metadata: {} }
[src/main.rs:NN] process_event(event) = Ok(
    "{\n  \"kind\": \"user.action\",\n  \"user\": \"alice\",\n  \"action\": \"login\",\n  \"metadata\": {}\n}",
)

JSON output:
{
  "kind": "user.action",
  "user": "alice",
  "action": "login",
  "metadata": {}
}

--- Event 2 ---
Event { kind: "user.action", user: "bob", action: "upload", metadata: {"checksum": "abc123", "file": "document.pdf", "size_bytes": "1024"} }
[src/main.rs:NN] process_event(event) = Ok(...)

JSON output:
{
  "kind": "user.action",
  "user": "bob",
  "action": "upload",
  "metadata": {
    "checksum": "abc123",
    "file": "document.pdf",
    "size_bytes": "1024"
  }
}

--- Event 3 ---
Event { kind: "admin.action", user: "alice", action: "admin", metadata: {} }
[src/main.rs:NN] process_event(event) = Err(
    Unauthorized(
        "alice",
        "admin",
    ),
)

Validation failed: user 'alice' is not authorized for action 'admin'

=== Done ===
```

The program exercises every macro type: a custom declarative macro (`event!`), standard library macros (`vec!`, `println!`, `assert_eq!`, `dbg!`, `todo!`), and third-party derive macros (`Serialize`, `Deserialize`, `Error`).

### Task 8.2 -- Identify the patterns

In `lab14-notes.md`, identify each instance of the following patterns in the program above:

1. A custom declarative macro defined with `macro_rules!`.
2. A declarative macro with multiple rules (matched in order).
3. A declarative macro using repetition syntax (`$( ... )*`).
4. A third-party derive macro on a struct (`Serialize`, `Deserialize`).
5. A third-party derive macro on an enum (`Error`).
6. Helper attributes that customize a derive (`#[error("...")]`, `#[from]`).
7. The `vec!` built-in macro for constructing a vector.
8. The `assert_eq!` built-in macro for verifying expectations.
9. The `dbg!` macro for inspecting an expression's value.
10. The `todo!` macro as a placeholder for unfinished code.

### Task 8.3 -- Predict and verify

For each of the following changes, predict what will happen and verify. Write your predictions in `lab14-notes.md` BEFORE running.

**Change A:** Remove the `#[derive(Serialize, Deserialize)]` from the `Event` struct. What changes? Where does the compiler error point?

**Change B:** In the `event!` macro, change `$kind:expr` to `$kind:ident` (identifier instead of expression). Try to call `event!("user.action", "alice", "login")`. What happens?

**Change C:** Add a new variant to `EventError` (e.g., `#[error("missing field")] MissingField`) but do NOT update the format string with field references. Does the code still compile? Why or why not?

For each change, write your prediction first, then run and verify. Remember to revert each change before doing the next.

### Checkpoints

1. The custom `event!` macro takes structured input with the keyword `metadata:` and a block of `key => value` pairs. This is a small DSL. What does the macro provide that ordinary Rust function calls do not?
2. The `Event` struct uses three derives: `Debug`, `Clone`, `Serialize`, `Deserialize`. The first two are from the standard library; the others are from `serde`. Could they all have been implemented manually? Why use the derives instead?
3. The `EventError` enum uses `thiserror`'s `#[derive(Error)]` with `#[error("...")]` attributes per variant. The result is essentially a `Display` implementation. What is gained by using `thiserror` versus writing the impl manually?

---

## Summary and Reflection

You have now used every concept from Module 18 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- First Declarative Macros | `macro_rules!`, fragment specifiers, repetition | Macros pattern-match on syntax and expand to code. The `!` distinguishes them from function calls. |
| 2 -- Macro Hygiene | name collision prevention | Rust macros are hygienic: internal variables do not collide with caller variables. |
| 3 -- Built-In Macros | `vec!`, `println!`, `assert_eq!`, `dbg!`, `todo!` | Each built-in exists because functions cannot do what it does (variadic args, format checking, source capture). |
| 4 -- Standard Derives | `Debug`, `Clone`, `Copy`, `PartialEq`, etc. | Derives generate trait implementations from struct or enum definitions. |
| 5 -- Third-Party Derives | `serde`, `thiserror` | Procedural macros generate substantial code that declarative macros cannot produce. |
| 6 -- Attribute and Function-Like Macros | `#[test]`, `#[tokio::main]`, `sqlx::query!` | Three kinds of procedural macros, each with distinct call syntax. |
| 7 -- Macros vs Functions vs Generics | choosing the right tool | Functions and generics first; macros for genuine reasons (variadic, compile-time, source capture). |
| 8 -- Integration | combining everything | Real Rust code uses many macros: a few custom ones, many built-in, many third-party derives. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab14-notes.md` before your next session.

1. Of the macro concepts in Module 18, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. The Module 18 reading argued that macros should be reached for "reluctantly" — only when functions and generics cannot do the job. After working through this lab, identify one specific situation from the lab where a macro was genuinely the right tool. What characteristics made a macro the right choice?

3. Rust's procedural macros (derives, attributes, function-like) can do almost arbitrary code generation at compile time. This power has trade-offs. Identify a specific risk or downside of relying heavily on procedural macros in a codebase. How would you mitigate it?

---

*End of Lab 14*
