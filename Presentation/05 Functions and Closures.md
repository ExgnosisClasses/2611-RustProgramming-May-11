# Module 5: Functions and Closures

## Module Overview

Functions are the basic unit of program organization. Closures are functions that capture values from the surrounding scope. Together they form the foundation of every nontrivial Rust program. This module covers both, plus the relationship between them.

The interesting parts of this module are not the syntax (which will look familiar) but the design choices behind it. Three ideas are central:

1. **Functions and expressions are deeply integrated.** A function body is an expression, the last expression is the return value, and the distinction between expressions and statements determines what is returned.
2. **Closures are categorized by what they do with captured values.** The compiler sorts every closure into one of three categories (`Fn`, `FnMut`, `FnOnce`) based on whether it reads, mutates, or consumes its environment. This categorization drives where closures can be used.
3. **Functions and closures can be passed around as values.** Higher-order functions and function pointers let you treat behavior the same way you treat data, which is essential for iterators and many design patterns.

By the end of this module, students will:

- Define functions with parameters and return types and understand where the type annotations are required.
- Distinguish expressions from statements and explain why Rust functions return values without an explicit `return`.
- Write closures that capture variables and choose the right capture mode.
- Recognize which of `Fn`, `FnMut`, or `FnOnce` a closure satisfies and how that limits where it can be used.
- Pass closures and function pointers as arguments to higher-order functions.
- Connect the closure concept to lambda expressions in other languages, recognizing what is the same and what is different.

The patterns in this module appear in every iterator chain, every callback, and every generic algorithm. Many later modules build on these foundations.

> **A note on forward references.** Several topics in this module touch on concepts that have full treatment in later modules:
>
> - **Borrowing and references** (`&`, `&mut`) appear in the closure capture discussion. Module 6 covers these in depth as part of ownership.
> - **Traits** (`Fn`, `FnMut`, `FnOnce`) are introduced here as named categories. Module 9 covers the trait system formally.
> - **Lifetimes** are mentioned briefly when discussing returned closures. Module 7 covers them.
>
> Where these topics come up, the explanations are deliberately at the level needed to understand closures, with a forward reference to the module that covers the concept fully. Treat them as named patterns for now; their underlying mechanics will become clear later.

---

## A. Defining and Calling Functions

### Basic Syntax

A function is declared with the `fn` keyword. Parameters are written `name: type`, the return type is written `-> type`, and the body is enclosed in braces.

```rust
fn greet(name: &str) -> String {
    format!("Hello, {name}")
}

fn main() {
    let message = greet("Alice");
    println!("{message}");
}
```

The function signature is a contract: the compiler checks that `greet` is called with a `&str` and that what it produces is a `String`. The body's job is to fulfill that contract.

### Naming Conventions

Functions use `snake_case`. The Rust community is consistent about this, and `rustfmt` will not change names but `clippy` will warn about non-conventional ones.

```rust
fn calculate_area(width: f64, height: f64) -> f64 { ... }     // good
fn calculateArea(width: f64, height: f64) -> f64 { ... }      // works but flagged
```

Function names should describe what the function does, ideally as a verb or verb phrase. Names like `parse_input`, `compute_total`, or `is_valid` are typical.

### Order of Declaration

Unlike C and C++, Rust does not require function declarations to appear before their use. The compiler reads the entire file before resolving names, so a function defined at the bottom of the file can be called from the top.

```rust
fn main() {
    let result = double(5);
    println!("{result}");
}

fn double(x: i32) -> i32 {
    x * 2
}
```

This compiles. There are no forward declarations or header files in Rust. The compiler handles the resolution.

### Functions Cannot Capture from Their Surroundings

You can declare a function inside another function, but the inner function cannot use variables from the outer one:

```rust
fn outer() {
    let x = 5;

    fn inner() {
        // println!("{x}");    // ERROR: cannot capture x
    }

    inner();
}
```

This is the compile error students most often hit when coming from JavaScript or Python. The fix is to use a closure (covered in Section D) or to pass the value explicitly as a parameter.

The reason for this restriction is performance: a nested function compiles to a regular standalone function with no hidden state. If it could capture, the compiler would have to insert hidden parameters or hidden allocations, contradicting the zero-cost abstractions principle. Closures handle the capturing case explicitly, with the cost made visible in the type.

---

## B. Parameters and Return Types

### Type Annotations Are Required

Every function parameter must have an explicit type annotation. The same applies to the return type. The compiler does not infer function signatures from their bodies.

```rust
fn add(a: i32, b: i32) -> i32 {     // every type is explicit
    a + b
}
```

This is in contrast to local variables (`let x = 5` infers `i32`) and closures (which can omit types). The reason for the asymmetry is documentation: the function signature is the contract that callers see. If the compiler inferred the signature, every change to the body could silently change the signature, breaking callers in nonlocal ways. Forcing annotations on signatures keeps the contract explicit and stable.

### Returning No Value

When a function does not return anything meaningful, you can omit the return type:

```rust
fn print_message(message: &str) {
    println!("{message}");
}
```

This is equivalent to declaring the return type as `()`, the unit type:

```rust
fn print_message(message: &str) -> () {
    println!("{message}");
}
```

Most Rust developers omit `-> ()` for clarity. Both forms compile to the same code.

### Returning Values

Inside the function body, the last expression with no trailing semicolon is the return value:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b           // no semicolon: this is the return value
}
```

You can also use `return` explicitly, which is required for early returns and optional for the final expression:

```rust
fn divide(numerator: f64, denominator: f64) -> f64 {
    if denominator == 0.0 {
        return 0.0;     // early return
    }
    numerator / denominator     // implicit return
}
```

The convention is to use `return` only for early exits and to use the implicit form for the final value. clippy will warn about the redundant `return result;` at the end of a function.

### Multiple Return Values

Rust functions return exactly one value. To return multiple values, use a tuple or a struct. Tuples are appropriate for short, position-obvious returns:

```rust
fn min_max(values: &[i32]) -> (i32, i32) {
    let mut min = values[0];
    let mut max = values[0];
    for &v in &values[1..] {
        if v < min { min = v; }
        if v > max { max = v; }
    }
    (min, max)
}

fn main() {
    let (lo, hi) = min_max(&[3, 1, 4, 1, 5, 9, 2, 6]);
    println!("min: {lo}, max: {hi}");
}
```

For more than three values, or when the values are easy to confuse, prefer a struct (covered in a later module).

### Mutable Parameters

Parameter bindings are immutable by default, just like `let`. To allow modification of the parameter inside the function, declare it `mut`:

```rust
fn truncate(mut text: String, max_len: usize) -> String {
    if text.len() > max_len {
        text.truncate(max_len);
    }
    text
}
```

The `mut` here only allows the function to modify its own copy of the parameter. Whether the caller's value is affected depends on whether ownership was transferred or a reference was passed. Module 6 covers this in detail.

---

## C. Expressions vs. Statements

This is one of the most important sections of the module. The expression-versus-statement distinction explains many things about how Rust code is written and why function returns work the way they do.

### Definitions

An **expression** evaluates to a value. Examples:

```rust
5                      // a literal expression
x + 1                  // an arithmetic expression
foo(bar)               // a function call expression
if x > 0 { 1 } else { -1 }    // an if expression
{ let y = 5; y * 2 }   // a block expression
```

A **statement** performs an action but does not produce a value. Statements end with a semicolon. Examples:

```rust
let x = 5;             // a let statement
x = 6;                 // an assignment statement (only valid inside other code)
return value;          // a return statement
```

### The Rule

In Rust, the last thing in a block determines what the block evaluates to:

- If the last item is an expression (no trailing semicolon), the block evaluates to that expression's value.
- If the last item is a statement (or an expression followed by a semicolon), the block evaluates to `()`.

This rule is the basis of implicit returns from functions and the basis of `if` and `loop` as expressions.

### A Side-by-Side Demonstration

```rust
fn good() -> i32 {
    let x = 5;
    x + 1           // expression: this is the return value (6)
}

fn broken() -> i32 {
    let x = 5;
    x + 1;          // statement (semicolon makes it one): block evaluates to ()
}                   // ERROR: expected i32, found ()
```

The first version returns 6. The second version does not compile because the trailing semicolon converts the expression into a statement, and the function ends up returning `()` instead of `i32`.

This is the most common bug from semicolons in Rust. The compiler error is helpful when it happens:

```
error[E0308]: mismatched types
  |
  | fn broken() -> i32 {
  |                --- expected `i32` because of return type
  |     ...
  |     x + 1;
  |          - help: remove this semicolon to return this value
```

### Blocks Are Expressions

A block (`{ ... }`) is itself an expression. You can put a block on the right side of `let`:

```rust
let result = {
    let x = 5;
    let y = 10;
    x + y           // last expression, no semicolon: block evaluates to 15
};

println!("{result}");
```

The block runs the inner statements, then evaluates to its final expression. This is how you create scoped temporary computations or apply some logic before binding a final value.

### Why This Design?

Rust inherits the expression-orientation from ML-family languages (OCaml, Haskell). The advantages are practical:

**Simplification.** Many constructs that are special in C-family languages are just expressions in Rust. There is no separate "ternary operator" because `if` is already an expression. There is no "block expression" syntax because blocks are already expressions. Less special-case syntax means less to learn and remember.

**Avoiding mutable state.** Instead of declaring a mutable variable and assigning to it from inside an `if`, you bind it to the value of the `if` directly. This eliminates a category of "uninitialized variable" bugs.

**Composability.** Expressions can appear anywhere a value is expected. You can put an `if` inside a function call, inside another expression, inside a `let` binding. Statements cannot be composed this way.

The cost is that semicolons matter more than in many other languages. Adding or removing one can change a function's behavior. The compiler error messages handle this well, but it does take some practice to internalize.

### Common Patterns

A few patterns that come from this design:

**Conditional binding.** Use `if` as an expression to bind one of several values:

```rust
let speed = if velocity > 100 { "fast" } else { "slow" };
```

**Block-scoped initialization.** Compute a value with intermediate steps, all in one binding:

```rust
let coordinates = {
    let raw = read_input();
    let trimmed = raw.trim();
    parse_coordinates(trimmed)
};
```

The intermediate variables `raw` and `trimmed` are scoped to the block and disappear after the binding completes.

**Last-expression returns.** Function bodies almost always end with an expression rather than `return`. This is the conventional Rust style.

---

## D. Closures: Syntax and Capturing Variables

### What Is a Closure?

A closure is an anonymous function that can capture values from its enclosing scope. In other words, it is a function plus the local context where it was defined.

```rust
fn main() {
    let multiplier = 3;

    let multiply = |x| x * multiplier;     // closure that captures multiplier

    println!("{}", multiply(5));     // 15
    println!("{}", multiply(7));     // 21
}
```

The closure `|x| x * multiplier` is defined inside `main`. It uses `multiplier`, which is a local variable of `main`. The closure "captures" `multiplier`, meaning it carries access to the value as part of itself. Even after `main` continues running past the closure definition, the closure still has access to `multiplier`.

### Syntax

Closures look different from functions:

```rust
let f1 = |x|         x + 1;              // single parameter, no type annotations
let f2 = |x: i32|    x + 1;              // single parameter, typed
let f3 = |x, y|      x + y;              // two parameters
let f4 = |x: i32| -> i32 { x + 1 };      // typed and explicit return type
let f5 = || println!("hello");           // no parameters
```

The parameter list is between `|` (vertical bars) instead of parentheses. Type annotations are optional; the compiler can usually infer them from how the closure is used. The body is a single expression for short closures, or a block for multi-statement bodies.

### Type Inference for Closures

This is a key difference from functions. Functions require explicit parameter and return type annotations. Closures usually do not.

```rust
fn double(x: i32) -> i32 {     // explicit
    x * 2
}

let double = |x| x * 2;         // inferred
```

The inference works because closures are typically used right where they are defined, or are passed to a function whose signature constrains the types. The compiler can usually figure out the parameter and return types from context.

When the compiler cannot, you can add annotations:

```rust
let identity = |x| x;           // does not compile if not used: type cannot be inferred
let identity = |x: i32| x;      // works
```

### Capture Modes

The interesting question about a closure is how it captures the values from its environment. There are three possibilities. Each one corresponds to a way of connecting to a value that is covered in detail in Module 6 (Ownership and Borrowing). For now, treat the three modes as named patterns: "borrow," "borrow mutably," and "take ownership."

#### 1. Capture by Borrowing (immutable)

The closure borrows the value. It can read but not modify. The original variable is still usable after the closure runs.

```rust
let name = String::from("Alice");
let greet = || println!("Hello, {name}");

greet();
greet();
println!("Original name still here: {name}");      // works
```

The `&` notation for references and the rules around them are covered in Module 6. The takeaway here: this is the cheapest and most flexible capture mode, and the compiler picks it when it can.

#### 2. Capture by Mutable Borrow

The closure borrows mutably. It can modify the captured value. While the closure exists, no one else can use the original variable.

```rust
let mut count = 0;
let mut increment = || count += 1;

increment();
increment();
println!("Count: {count}");          // 2
```

Note that the closure binding itself must be `mut` because calling it changes its captured state. The borrow checker rules that govern when this is allowed are covered in Module 6.

#### 3. Capture by Value (move)

The closure takes ownership of the captured values. The original variable is no longer usable after the closure is created.

```rust
let name = String::from("Alice");
let take_ownership = move || println!("Hello, {name}");

take_ownership();
// println!("{name}");      // ERROR: name was moved into the closure
```

The `move` keyword forces ownership transfer. This is necessary in cases where the closure needs to outlive the scope where it was created, or where it is sent to another thread.

### How the Compiler Chooses

The compiler picks the least restrictive capture mode that works. If the closure body only reads a value, it captures by borrow. If the body modifies the value, it captures by mutable borrow. If you write `move`, it captures by value regardless.

You generally let the compiler choose. Use `move` only when you specifically need to transfer ownership, most commonly when sending a closure to another thread:

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("{data:?}");
    });

    handle.join().unwrap();
}
```

The new thread might outlive the original `data`, so the closure must own its captured values rather than borrow them. `move` is required.

---

## E. `Fn`, `FnMut`, and `FnOnce`

The capture modes from the previous section are tied to three named categories called traits. Module 9 covers the trait system in detail; for this module, treat `Fn`, `FnMut`, and `FnOnce` as labels that the compiler attaches to a closure based on what the closure does. Every closure (and every function) gets one or more of these labels, and the labels determine where the closure can be used.

### The Three Categories

**`FnOnce`**: A closure that can be called at least once. Calling it may consume captured values, so it might not be callable again. Every closure qualifies as `FnOnce`.

**`FnMut`**: A closure that can be called multiple times and may mutate captured values. To call it, you need a mutable reference. Every `FnMut` is also `FnOnce`.

**`Fn`**: A closure that can be called multiple times without mutating captured values. Safe to call from multiple places. Every `Fn` is also `FnMut` and `FnOnce`.

There is a hierarchy: `Fn` is the most restrictive (the closure does the least), and `FnOnce` is the most permissive (the closure can do almost anything but only once).

### Examples

```rust
fn main() {
    // Fn: only reads from captures
    let name = String::from("Alice");
    let greeter = || println!("Hello, {name}");
    greeter();
    greeter();      // can call multiple times

    // FnMut: mutates captures
    let mut count = 0;
    let mut counter = || count += 1;
    counter();
    counter();      // can call multiple times, but the binding must be mut

    // FnOnce: consumes captures (here, takes ownership of `data`)
    let data = String::from("hello");
    let consume = move || { let owned = data; println!("{owned}"); };
    consume();
    // consume();   // ERROR: data has been moved out, cannot call again
}
```

The pattern: read-only closures are `Fn`, mutating closures are `FnMut`, consuming closures are `FnOnce`.

### Why Three Categories?

Functions that accept closures use these labels to declare what they need from the closure. This is one of the more important patterns in idiomatic Rust:

```rust
fn call_twice<F: Fn()>(f: F) {
    f();
    f();
}
```

The `F: Fn()` part says "I need a closure I can call multiple times without mutation." The detailed syntax (`<F: ...>`, `where`, etc.) is covered alongside generics and traits in Modules 8 and 9. For now, the takeaway is that functions can specify which kind of closure they accept, and the three-tier system lets a function ask for exactly what it needs.

A function that asks for `FnMut` is more flexible (it accepts both `FnMut` and `Fn` closures, since every `Fn` is also an `FnMut`). A function that asks for `FnOnce` is the most flexible (it accepts any closure) but can only call it once.

### A Practical Example

Consider a retry helper:

```rust
fn retry<F, T>(mut operation: F, max_attempts: u32) -> Result<T, &'static str>
where
    F: FnMut() -> Result<T, &'static str>,
{
    for _ in 0..max_attempts {
        if let Ok(result) = operation() {
            return Ok(result);
        }
    }
    Err("max attempts exceeded")
}
```

The bound `FnMut() -> Result<T, ...>` says the operation might mutate state between calls (perhaps incrementing an attempt counter). The function does not need ownership of the closure, just mutable access. This is the right requirement for "call this several times, possibly with side effects."

If the function needed to call the operation only once, `FnOnce` would be enough and would accept more closures. If the operation never had any side effects, `Fn` would be enough and would document the no-mutation guarantee to callers.

### Choosing the Right Category

A useful rule of thumb when writing a function that takes a closure:

| If the function...                                  | Use category |
|-----------------------------------------------------|--------------|
| Only reads state and might be called from anywhere  | `Fn`         |
| Needs to call the closure multiple times with side effects | `FnMut` |
| Calls the closure exactly once and is done          | `FnOnce`     |

The default choice is usually `FnMut` if you call the closure more than once, `FnOnce` otherwise. Use `Fn` when you specifically need the no-mutation guarantee.

---

## F. Closures Compared to Lambdas in Other Languages

Rust closures are very similar to what other languages call lambdas, lambda expressions, or anonymous functions. If you have used lambdas in Python, JavaScript, Java, C#, or C++, the basic mental model carries over directly. This section maps that mental model to Rust and identifies the differences that come from Rust's design priorities.

### What They Have in Common

The core concept is identical across all of these languages. In each one, you can:

- Define a function inline without giving it a name.
- Pass that function as an argument to another function.
- Store it in a variable.
- Have it remember values from where it was defined.

A "double the input" lambda looks similar across languages:

```python
# Python
double = lambda x: x * 2
print(double(5))    # 10
```

```javascript
// JavaScript
const double = x => x * 2;
console.log(double(5));    // 10
```

```java
// Java
Function<Integer, Integer> doubleIt = x -> x * 2;
System.out.println(doubleIt.apply(5));    // 10
```

```rust
// Rust
let double = |x| x * 2;
println!("{}", double(5));    // 10
```

The intent is identical. Each language has slightly different syntax for the parameter list and body, but the meaning is the same: a small anonymous function bound to a name.

The capture behavior is also conceptually similar. In all four languages, a lambda defined inside another function can refer to variables from the enclosing scope:

```rust
let multiplier = 3;
let multiply = |x| x * multiplier;     // captures multiplier
```

```python
multiplier = 3
multiply = lambda x: x * multiplier    # captures multiplier
```

If you understand lambdas in any of these languages, the basic shape of Rust closures will feel familiar.

### Where Rust Differs

The differences come from Rust's three core concerns: ownership, performance, and the type system. Each one shapes closures in a distinctive way.

#### 1. Closures Have Anonymous, Unique Types

In most languages, all lambdas with the same signature share a type:

- **Java**: `Function<Integer, Integer>` is the type of any lambda taking an `Integer` and returning an `Integer`.
- **Python**: All callables are essentially the same; lambdas have a `function` type.
- **JavaScript**: Functions are values of type `function`.
- **C#**: `Func<int, int>` is the type of any matching lambda.

In Rust, every closure has its own anonymous type. Two closures with identical signatures and bodies are still different types as far as the compiler is concerned:

```rust
let f1 = |x: i32| x + 1;
let f2 = |x: i32| x + 1;     // different type from f1, despite looking identical
```

This is because each closure carries its own captured environment, and the captures are part of the type. The compiler generates a unique struct for each closure that holds whatever it captured. Two closures cannot share a type because their captures might differ in size, layout, or contents.

The practical consequence is that you cannot easily store closures in variables with the same type, put them in arrays, or return them directly without using one of two workarounds (covered in Section H below). This is unusual among mainstream languages and is a direct consequence of the zero-cost abstractions principle: the compiler generates specialized code for each closure rather than using a uniform runtime representation.

#### 2. Capture Modes Are Explicit and Categorized

Most languages have one capture mode, sometimes two:

- **Python, JavaScript, Java**: Capture by reference. The lambda sees the current value of captured variables, which can change after capture.
- **C++**: You write the capture mode explicitly: `[x]` for capture by copy, `[&x]` for capture by reference, `[=]` for "copy everything used."

Rust has three capture modes (borrow, mutable borrow, by value) and the compiler picks the least restrictive one that makes the closure work. You usually let it choose.

These three modes correspond to the three categories you saw in Section E (`Fn`, `FnMut`, `FnOnce`). The category determines where the closure can be used: a function that requires `Fn` rejects a closure that mutates; a function that requires `FnMut` accepts both `Fn` and `FnMut` closures.

This system has no real parallel in other languages. C++ has capture modes but they do not constrain how the closure is used. Java, Python, and JavaScript all use one capture mode and one runtime type. Rust's three-tier system is what makes closures fit into the broader ownership model.

#### 3. No Hidden Performance Cost

Lambdas in many languages involve hidden allocations or runtime dispatch:

- **Java**: Lambdas allocate (the JVM optimizes some cases). Calling a lambda goes through a virtual method call.
- **Python**: Everything is a heap-allocated object. Function calls are dictionary lookups.
- **JavaScript**: Functions are first-class objects. Each lambda allocates.
- **C++**: Lambdas without capture are zero-cost (compile to function pointers). Lambdas with capture become structs (similar to Rust), but type erasure (`std::function`) reintroduces heap allocation and dispatch.

In Rust, closures used in generic contexts (the common case) compile to specialized code with no allocation and no dispatch. The closure is monomorphized into the function that uses it, just like generic types are. A `vec.iter().map(|n| n * 2).sum()` compiles to a tight loop with no closure object existing at runtime.

You only pay for dynamic dispatch and allocation when you specifically opt into them.

#### 4. Captured References Must Live Long Enough

If a closure captures a reference, that reference must live as long as the closure does. The borrow checker enforces this. In other languages, capturing a reference to a stack-local variable that goes out of scope is either a runtime crash (C++) or works because the language uses garbage collection (Java, Python, JavaScript).

```rust
fn make_closure() -> impl Fn() {
    let s = String::from("hello");
    let closure = || println!("{s}");        // captures &s
    closure       // ERROR: returning a closure that references local s
}
```

This does not compile because `s` would go out of scope when the function returns. The fix is to use `move`, which makes the closure own `s`:

```rust
fn make_closure() -> impl Fn() {
    let s = String::from("hello");
    move || println!("{s}")        // closure owns s; both can leave the function together
}
```

In garbage-collected languages, the equivalent code "just works" because the runtime keeps the captured value alive as long as the closure references it. Rust requires the same guarantee but checks it at compile time, which means you sometimes have to be explicit (with `move`) where other languages would handle it automatically. The lifetime rules involved are covered in Module 7.

### A Summary Table

| Property                          | Rust                  | Java                  | Python                | JavaScript            | C++                   |
|-----------------------------------|-----------------------|-----------------------|-----------------------|-----------------------|-----------------------|
| Inline syntax                     | Yes                   | Yes                   | Yes (limited)         | Yes                   | Yes                   |
| Captures from enclosing scope     | Yes                   | Yes (effectively final) | Yes                | Yes                   | Yes (explicit)        |
| Each closure has unique type      | Yes                   | No                    | No                    | No                    | Yes                   |
| Capture mode chosen by compiler   | Yes                   | No (by reference)     | No (by reference)     | No (by reference)     | No (programmer)       |
| Categorized by capability         | Yes (Fn/FnMut/FnOnce) | No                    | No                    | No                    | No                    |
| Heap allocation for closures      | Optional              | Yes                   | Yes                   | Yes                   | Optional              |
| Dynamic dispatch                  | Optional              | Yes                   | Yes                   | Yes                   | Optional              |
| Lifetimes checked at compile time | Yes                   | No (GC)               | No (GC)               | No (GC)               | No (manual)           |

### Common Use Cases

The use cases for closures in Rust are largely the same as for lambdas elsewhere. The most common patterns:

**Iterator transformations.** This is by far the most common use of closures in idiomatic Rust:

```rust
let numbers = vec![1, 2, 3, 4, 5];
let doubled: Vec<i32> = numbers.iter().map(|n| n * 2).collect();
let evens: Vec<&i32> = numbers.iter().filter(|n| **n % 2 == 0).collect();
```

If you have used `map` and `filter` in Python, JavaScript, Java streams, or C# LINQ, this is the same pattern.

**Callbacks for events.** GUI frameworks, web servers, and async runtimes use closures as event handlers. The closure captures application state and runs when an event occurs.

**Custom comparison and ordering.** Sort with a custom key:

```rust
let mut words = vec!["apple", "banana", "kiwi"];
words.sort_by_key(|w| w.len());     // sort by length
```

**Lazy evaluation.** A closure represents a computation that might happen later. The standard library uses this for things like `Option::unwrap_or_else`:

```rust
let value: Option<i32> = None;
let result = value.unwrap_or_else(|| compute_default());
```

`compute_default()` only runs if the option is `None`. The closure defers the work.

**Configuration and DSL-like APIs.** Builder patterns and configuration systems often accept closures to let the user describe behavior:

```rust
thread::spawn(move || {
    // code that runs in the new thread
});
```

The closure is the unit of work the new thread will run.

### The Practical Implication

The basic intuition from other languages carries over: a closure is a function with captured state. Most everyday uses (passing closures to `map`, `filter`, `find`, etc.) feel similar across languages.

Where Rust diverges is when you start trying to do unusual things with closures: storing them in collections, returning them from functions, sharing them between threads, mixing closures with different signatures. Each of these requires more attention in Rust than in garbage-collected languages because Rust forces you to make the type, ownership, and lifetime decisions explicit.

This is the same trade-off that runs through the rest of the language. You pay a small upfront cost in explicitness for a guarantee that your code will not crash or leak at runtime. For everyday use, closures in Rust feel like lambdas in any other language. For systems work, the explicit categorization is what makes them safe to use in low-level contexts where garbage collection is not an option.

---

## G. Higher-Order Functions

A higher-order function is one that takes a function or closure as a parameter, or returns one. This is fundamental to Rust's iterator API and to many common patterns.

### Functions That Take Closures

The standard library has many higher-order methods. The iterator methods are the most common:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let doubled: Vec<i32> = numbers.iter().map(|n| n * 2).collect();
    println!("{doubled:?}");        // [2, 4, 6, 8, 10]

    let even: Vec<&i32> = numbers.iter().filter(|n| **n % 2 == 0).collect();
    println!("{even:?}");           // [2, 4]

    let total: i32 = numbers.iter().sum();
    println!("{total}");            // 15
}
```

`map` takes a closure and applies it to each element. `filter` takes a closure that returns `bool` and keeps elements where it returns true. Both are higher-order: the closure is the parameter that controls the behavior.

### Writing Your Own Higher-Order Functions

You can write functions that accept closures using the `Fn` family of categories:

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

    println!("{}", apply(double, 5));        // 10
    println!("{}", apply(add_ten, 5));       // 15
    println!("{}", apply(|x| x * x, 5));     // 25
}
```

The `F: Fn(i32) -> i32` part says: "any closure that takes an `i32` and returns an `i32`." The full meaning of generic parameters and `where` clauses is covered alongside generics in Module 8.

### Returning Closures

Returning a closure is more involved than accepting one, because each closure has a unique anonymous type. The two common forms:

**Boxed trait object** (heap-allocated, runtime dispatch):

```rust
fn make_adder(n: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x + n)
}

fn main() {
    let add_five = make_adder(5);
    println!("{}", add_five(10));        // 15
}
```

**`impl Trait`** (compile-time dispatch, no heap allocation, but more restrictive):

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}
```

The `impl Trait` form is preferred when the function returns one specific closure. The `Box<dyn Fn>` form is necessary when the function might return different closures depending on its arguments. Module 9 (traits) covers both forms in detail.

### Why Take Closures?

Higher-order functions allow algorithms to be parameterized by behavior. Instead of writing one `filter_evens` and one `filter_positives`, you write `filter` and pass different closures.

The standard library's iterator API is an extreme example: a small number of methods (`map`, `filter`, `fold`, `find`, `any`, `all`, `take`, `skip`, etc.) compose into a vast number of programs. Each method takes one or more closures, and the closures contain the program-specific logic. The framework provides the iteration machinery; the closure provides the application-specific behavior.

This pattern is one reason iterator code in Rust is so concise. A complex data transformation that would require dozens of lines of imperative code often fits on one line as a chain of higher-order calls.

---

## H. Function Pointers

A closure has an anonymous type. Each closure is its own type, even if the body is the same. A function pointer, by contrast, is a regular type that can be named: `fn(i32) -> i32`.

### The `fn` Type

The lowercase `fn` is the function pointer type. It is similar to `Fn` (the closure category) in name but very different in semantics:

```rust
fn double(x: i32) -> i32 {
    x * 2
}

fn main() {
    let f: fn(i32) -> i32 = double;
    println!("{}", f(5));        // 10
}
```

The variable `f` has type `fn(i32) -> i32`: a pointer to a function with that signature. It can be assigned, copied, and passed around like any other value.

### When to Use `fn` vs `Fn`

The trade-off:

- **`fn`** is a concrete type. It has a fixed size (the size of a pointer), can be stored in arrays and structs without boxing, and can be copied freely. But it cannot represent closures that capture environment variables; only plain functions and non-capturing closures fit into the `fn` type.

- **`Fn` (and friends)** are categories that can be used as generic constraints. Any closure satisfies them, including closures that capture. But the resulting type is generic or boxed, and using it is slightly more involved than using `fn`.

Use `fn` when:
- You are passing functions, not closures, and want a simple concrete type.
- You need to interoperate with C (FFI), where `fn` is the native shape.
- You want to store function pointers in a struct without generics.

Use `Fn` (or `FnMut`, `FnOnce`) when:
- You want to accept any function-like thing, including capturing closures.
- You are writing generic code that should be flexible about its callable parameter.

### Functions Implement the Closure Categories

A regular function automatically satisfies all three of `Fn`, `FnMut`, and `FnOnce`. This is why a `fn` value can be passed to a function that expects an `Fn`:

```rust
fn double(x: i32) -> i32 { x * 2 }

fn apply_fn<F: Fn(i32) -> i32>(f: F, value: i32) -> i32 {
    f(value)
}

fn main() {
    let result = apply_fn(double, 5);     // works: double satisfies Fn(i32) -> i32
    println!("{result}");                  // 10
}
```

So in a higher-order function that uses the `Fn` categories, callers can pass either functions or closures interchangeably. This is the common case and is usually what you want.

### A Pattern: Function Tables

Function pointers are useful when you need a uniform type for a collection of functions. For example, a dispatch table:

```rust
fn add(a: i32, b: i32) -> i32 { a + b }
fn subtract(a: i32, b: i32) -> i32 { a - b }
fn multiply(a: i32, b: i32) -> i32 { a * b }

fn main() {
    let operations: [fn(i32, i32) -> i32; 3] = [add, subtract, multiply];

    for op in operations {
        println!("{}", op(10, 3));
    }
}
```

The array `[fn(i32, i32) -> i32; 3]` requires a concrete type. Closures (each of which has its own anonymous type) cannot be put into this array directly. The function pointer type lets the three functions share one type.

If the same array used `Box<dyn Fn(i32, i32) -> i32>`, it would accept closures too, at the cost of heap allocation and dynamic dispatch. Function pointers are zero-cost; trait objects are not. Choose based on whether you need the flexibility.

---

## Module Summary

- Functions are declared with `fn`, with required type annotations on parameters and return values. The function body is an expression; the last expression with no semicolon is the return value.
- Statements perform actions and end with semicolons. Expressions produce values. The distinction is what makes implicit returns work.
- Closures are anonymous functions that can capture values from their enclosing scope. The compiler infers their types and chooses the least restrictive capture mode.
- Every closure satisfies one or more of `Fn`, `FnMut`, `FnOnce`. The category determines whether the closure can be called multiple times and whether it can mutate or consume its captures.
- Closures in Rust correspond to lambdas in other languages, with the same core concept of an inline anonymous function that captures values. The differences are in how Rust handles types, capture modes, and lifetimes; the use cases (iterator transformations, callbacks, custom ordering, lazy evaluation, configuration APIs) are largely the same.
- Higher-order functions take or return functions and closures. They are the basis of the iterator API and many other patterns in idiomatic Rust.
- Function pointers (`fn`) are a concrete type for non-capturing functions. The `Fn` family is more flexible because any callable satisfies them, including capturing closures.

## Discussion Questions

1. Rust requires explicit type annotations on function signatures but allows them to be omitted on closures. Explain why this asymmetry makes sense given how each is typically used.
2. The expression-versus-statement distinction means that adding or removing a semicolon at the end of a function can change its return type. Identify a code pattern where this distinction is helpful, and one where it is a footgun.
3. Pick a language you know well and identify a closure use case from this module that would work the same way in that language, and one where Rust's stricter rules would force a different approach.

---

