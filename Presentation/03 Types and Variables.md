# Module 3: Rust Types and Variables

## Module Overview

This module introduces Rust's primitive type system and the rules that govern variable bindings. Most of the syntax will look familiar to anyone with experience in C, Java, or Go. The interesting parts are the design choices: why integers have explicit widths, why variables are immutable by default, why shadowing is allowed when reassignment is not, and why constants are split into two distinct concepts.

By the end of this module, students will:

- Recognize and choose appropriate scalar types for numeric, boolean, and character data.
- Understand when type inference suffices and when explicit annotations are required.
- Use immutability and the `mut` keyword to express intent clearly.
- Distinguish between `const` and `static` and know when each is appropriate.
- Use shadowing as a deliberate idiom rather than confusing it with mutation.
- Build and pattern-match on tuples and fixed-size arrays.

The goal is more than memorizing syntax. Students should leave understanding why Rust made each design decision, so the type system feels like a tool rather than a tax.

---

## A. Scalar Types: Integers, Floats, Booleans, Characters

A scalar type represents a single value. Rust has four scalar categories: integers, floating-point numbers, booleans, and characters. All four are stored on the stack, copied by value, and have known sizes at compile time.

### Integers

Rust integers come with explicit width and signedness in the type name. There is no plain `int`.

| Length     | Signed   | Unsigned |
|------------|----------|----------|
| 8-bit      | `i8`     | `u8`     |
| 16-bit     | `i16`    | `u16`    |
| 32-bit     | `i32`    | `u32`    |
| 64-bit     | `i64`    | `u64`    |
| 128-bit    | `i128`   | `u128`   |
| arch-size  | `isize`  | `usize`  |

The `isize` and `usize` types are the size of a pointer on the target architecture: 32 bits on a 32-bit platform, 64 bits on a 64-bit platform. They are used for indexing collections and for any quantity that must hold a memory address.

#### Why explicit widths?

Languages like C have an `int` whose size depends on the platform. This causes portability bugs: code that works on a 64-bit machine fails on a 32-bit microcontroller because integer overflow happens at a smaller value. Rust avoids this by making the size visible in the type. If you need a 32-bit integer, you write `i32`. The same code behaves identically on every platform.

#### Default integer type

When the compiler cannot determine the integer type from context, it falls back to `i32`. This is a deliberate choice: 32 bits is large enough for typical counts and small enough to be efficient on every modern architecture.

```rust
let count = 42;        // inferred as i32
let big: u64 = 42;     // explicitly typed
let small = 42i8;      // type suffix on the literal
```

#### Numeric literals

Rust supports several literal formats:

```rust
let decimal = 98_222;          // underscores for readability
let hex = 0xff;                // hexadecimal
let octal = 0o77;              // octal
let binary = 0b1111_0000;      // binary
let byte = b'A';               // byte literal (u8 only)
```

The underscore is purely visual. `1_000_000` and `1000000` are the same literal.

#### Integer overflow

In debug builds, integer overflow causes a panic. In release builds, overflow wraps using two's complement. This may surprise developers used to C semantics where overflow is undefined behavior.

```rust
fn main() {
    let mut x: u8 = 255;
    x += 1;     // panics in debug, wraps to 0 in release
    println!("{x}");
}
```

#### The Design Reasoning

The split behaviour between debug and release is a deliberate compromise that reveals several of Rust's core design principles. When the language designers had to choose what happens on overflow, they faced three options, each with serious problems.

**Option 1: Undefined behavior** (C and C++ for signed integers). The compiler may assume overflow never happens and optimize accordingly. Rust rejected this on principle. Undefined behavior in safe code violates Rust's central safety promise. Code that could silently produce wrong results, or that the optimizer could turn into arbitrary instructions, is exactly what Rust was created to prevent.

**Option 2: Always wrap** (Java, Go). Wrapping is fast and well-defined. But it has a subtle problem: most overflow is a bug. When a programmer writes `count + 1`, they almost always mean "the next integer," not "the next integer, or possibly a large negative number if count happened to be at the maximum." Silent wrapping turns logic bugs into hard-to-find production failures. The 2014 incident where Boeing 787s had to be rebooted every 248 days because a counter overflowed is a famous example.

**Option 3: Always panic.** Correct but slow. Every arithmetic operation would need a runtime check, producing 5 to 30 percent slowdowns on numeric code. Worse, it would break legitimate code: hash functions, random number generators, and many cryptographic primitives deliberately rely on wrapping arithmetic.

The Rust solution optimizes for two different audiences:

- **During development**, the developer wants to find bugs immediately. A panic on overflow is loud and obvious. The cost of the runtime check does not matter because nobody benchmarks debug builds.
- **In production**, the program must run fast and behave deterministically. Crashing on overflow in production would turn latent bugs into outages. Release builds wrap, matching what the hardware already does, at zero runtime cost.

The reasoning is that overflow checks are a debugging aid, not a security feature. They help you find the bug while testing. Once the bug is fixed, the check has done its job.

This decision is a good window into Rust's design philosophy generally. The language tries to refuse silent failure modes, make the dangerous operation more verbose than the safe operation, pay performance costs only when the developer asks for them, and treat development and production as different environments with different needs. You will see this same reasoning in bounds checks on arrays, in `Option` and `Result` instead of null and exceptions, and in the borrow checker itself.

#### Explicit Overflow Methods

When you genuinely care about overflow behaviour, the standard library provides four families of methods that communicate intent at the call site:

- `wrapping_add`, `wrapping_sub`: silently wrap. Use when wrapping is the correct semantics (cryptography, hash functions).
- `checked_add`, `checked_sub`: return `Option`, with `None` on overflow. Use when you want to handle the overflow case explicitly.
- `saturating_add`, `saturating_sub`: clamp to the type's bounds. Use when overflow should produce the maximum or minimum value rather than wrap.
- `overflowing_add`, `overflowing_sub`: return the wrapped value and a boolean flag. Use when you need both the result and whether overflow occurred.

```rust
let a: u8 = 200;
let b: u8 = 100;

let wrapped = a.wrapping_add(b);                  // 44
let checked = a.checked_add(b);                   // None
let saturated = a.saturating_add(b);              // 255
let (result, overflowed) = a.overflowing_add(b);  // (44, true)
```

Code that uses these methods is self-documenting. Code that uses plain `+` is saying "I have established that this cannot overflow." Both are valid; the choice depends on what you can prove about your inputs.

#### Configuring the Default

The default behaviour is a knob, not a law. If you want release builds to also panic on overflow (for example, in a financial application where any overflow indicates a serious bug), set this in `Cargo.toml`:

```toml
[profile.release]
overflow-checks = true
```

The reverse is also possible: turn off checks in debug builds to match production behaviour during specific test runs. Most projects use the defaults.

### Floating-Point Numbers

Rust has two floating-point types, both following the IEEE 754 standard:

- `f32`: 32-bit single precision.
- `f64`: 64-bit double precision.

The default is `f64`. On modern CPUs, `f64` is roughly the same speed as `f32` and has significantly more precision. Use `f32` only when memory matters (large arrays, GPU code, embedded systems).

```rust
let x = 2.0;          // inferred as f64
let y: f32 = 3.14;    // explicitly f32
let z = 1e6;          // 1,000,000.0, inferred as f64
```

All arithmetic operators work on floats:

```rust
let sum = 5.0 + 10.0;
let difference = 95.5 - 4.3;
let product = 4.0 * 30.0;
let quotient = 56.7 / 32.2;
let remainder = 43.5 % 5.0;
```

Floating-point operations follow IEEE 754 rules, including `NaN` and infinity:

```rust
let nan = 0.0_f64 / 0.0;
println!("{}", nan.is_nan());       // true
println!("{}", nan == nan);          // false (NaN is never equal to anything)
```

The fact that `NaN != NaN` is an IEEE 754 property, not a Rust quirk. It is one reason floating-point comparisons require care.

### Booleans

The `bool` type has exactly two values: `true` and `false`. It occupies one byte, not one bit, because individual bits are not addressable on most hardware.

```rust
let is_active = true;
let is_admin: bool = false;
```

Rust does not implicitly convert other types to `bool`. This compiles in C but not in Rust:

```rust
// This is a compile error in Rust:
// if 1 { ... }
//
// You must write:
if 1 != 0 { /* ... */ }
```

This explicitness is intentional. Implicit truthiness is a common source of bugs. In Rust, anything that wants to be tested for truth must return a `bool` directly.

### Characters

The `char` type represents a single Unicode scalar value, four bytes wide. This is wider than the C `char` (one byte) and wider than Java's `char` (two bytes, UTF-16 code unit).

```rust
let letter = 'A';
let emoji_char = '\u{1F600}';     // unicode escape
let chinese = '中';
let newline = '\n';
```

A `char` can hold any Unicode scalar value, including those in the supplementary planes that require surrogate pairs in UTF-16. This is one reason Rust strings are simpler to reason about than Java strings: there is no concept of a "low surrogate" character.

The fact that `char` is four bytes also explains why Rust strings (covered in detail in a later module) are not indexed by character. A `String` is stored as UTF-8, where one character occupies one to four bytes. Indexing into the middle of a multi-byte sequence would not be safe, so Rust does not allow it.

### Type Sizes Summary

```rust
fn main() {
    println!("i8:    {} bytes", std::mem::size_of::<i8>());      // 1
    println!("i32:   {} bytes", std::mem::size_of::<i32>());     // 4
    println!("i64:   {} bytes", std::mem::size_of::<i64>());     // 8
    println!("usize: {} bytes", std::mem::size_of::<usize>());   // 8 on 64-bit
    println!("f32:   {} bytes", std::mem::size_of::<f32>());     // 4
    println!("f64:   {} bytes", std::mem::size_of::<f64>());     // 8
    println!("bool:  {} bytes", std::mem::size_of::<bool>());    // 1
    println!("char:  {} bytes", std::mem::size_of::<char>());    // 4
}
```

---

## B. Type Inference and Explicit Annotations

Rust is statically typed. Every variable has a type that the compiler must know. However, the compiler will infer most types automatically, so explicit annotations are usually optional.

### When Inference Works

The compiler infers a variable's type from the value assigned to it and from how the variable is used afterward.

```rust
let x = 42;                     // i32 (default integer)
let y = 3.14;                   // f64 (default float)
let s = "hello";                // &str
let v = vec![1, 2, 3];          // Vec<i32>
let pair = (1, "one");          // (i32, &str)
```

In each case, the right-hand side has a known type, and that type flows to the variable.

### When Inference Needs Help

There are cases where the value alone is not enough information.

#### Case 1: A literal that could be many types

```rust
// This compiles, x is i32 by default:
let x = 42;

// This also compiles, but the function parameter forces a specific type:
fn takes_u8(_: u8) {}
let x = 42;
takes_u8(x);    // x is now inferred as u8
```

The compiler looks at how the variable is used to refine the type.

#### Case 2: Generic function results

```rust
// This does not compile:
let parsed = "42".parse();
//  ^^^^^^ cannot infer type

// You must annotate:
let parsed: i32 = "42".parse().unwrap();

// Or use the turbofish syntax:
let parsed = "42".parse::<i32>().unwrap();
```

`str::parse` is generic over the result type. Without context, the compiler does not know whether you want `i32`, `f64`, `bool`, or anything else parseable from a string. The annotation tells it.

#### Case 3: Empty collections

```rust
// This does not compile:
let v = Vec::new();
//  ^ cannot infer type

// You must annotate, either on the variable or with turbofish:
let v: Vec<i32> = Vec::new();
let v = Vec::<i32>::new();
```

An empty `Vec` could be a vector of anything. The compiler refuses to guess.

### The Turbofish

The `::<T>` syntax is officially called the turbofish. It appears anywhere a generic function or method needs an explicit type argument:

```rust
let nums: Vec<i32> = (1..=5).collect();          // annotation on the binding
let nums = (1..=5).collect::<Vec<i32>>();        // annotation on collect
let nums = (1..=5).collect::<Vec<_>>();           // _ lets the compiler infer the inner type
```

The third form is common: tell the compiler "I want a `Vec`, you figure out what is in it." This is useful when the element type is obvious from context.

### When to Annotate Explicitly

Even when inference works, an explicit annotation can be valuable:

- **Documenting public APIs.** Function signatures must always be explicit. The annotation is part of the contract.
- **Making the code easier to read.** A nontrivial expression chain can produce a type that is not obvious to a human reader.
- **Catching mistakes early.** If you intended `i32` but the inferred type is `i64`, an annotation will catch the discrepancy at the binding site rather than at some distant use.
- **Working with type-level numeric literals.** Library APIs that require a specific integer width are easier to call with an explicit annotation.

The community convention is: annotate function signatures always, annotate local variables only when the type is non-obvious or when inference fails.

---

## C. Immutability by Default and the `mut` Keyword

In Rust, `let` produces an immutable binding. To make a binding mutable, you must explicitly opt in with `mut`.

### The Default

```rust
let x = 5;
x = 6;     // compile error: cannot assign twice to immutable variable `x`
```

The compiler refuses to allow the reassignment. To make the binding mutable:

```rust
let mut x = 5;
x = 6;     // ok
```

### Why Immutable by Default?

This is one of Rust's most important defaults. Three reasons drive the choice.

#### Reason 1: Reasoning about code

When a variable cannot change, you can read its value at the declaration and trust that it will not be different elsewhere. This dramatically simplifies reading unfamiliar code. In a 200-line function, immutable bindings can be understood without scrolling.

#### Reason 2: Preventing bugs

A surprising fraction of bugs come from variables changing unexpectedly. By making mutability opt-in, Rust forces the developer to acknowledge each mutable binding as a deliberate choice. This often uncovers code that was mutating values without needing to.

#### Reason 3: Concurrency safety

Mutable data shared between threads is the source of data races. Immutable data, in contrast, is safe to share. Making immutability the default aligns with the language's broader goal of fearless concurrency.

### A Demo

The following short example illustrates the pattern. The `mut` keyword is reserved for variables that genuinely need to change.

```rust
fn main() {
    // Configuration values that should never change after initialization.
    let max_connections = 100;
    let timeout_seconds = 30;

    // A counter that must change as work is performed.
    let mut connections_handled = 0;

    for _ in 0..max_connections {
        connections_handled += 1;
    }

    println!("Limit: {max_connections}, processed: {connections_handled}");
    println!("Timeout: {timeout_seconds} seconds");
}
```

The reader can see at a glance which values are configuration (immutable) and which are state (mutable). The intent is encoded in the source.

### Mutability Versus Reassignment

It is worth noting that `mut` only matters when you want to change the value bound to the same name. If you are willing to introduce a new binding, you can do so without `mut`:

```rust
let x = 5;
let x = x + 1;       // not mutation; this is shadowing (covered in Section E)
```

This distinction matters. Many cases that look like mutation are better expressed as shadowing.

---

## D. Constants and Static Variables

Rust has two ways to declare values that live for the entire program: `const` and `static`. Both compile to compile-time constants. Both are uppercase by convention. They differ in subtle but important ways.

### `const`: Compile-Time Constants

A `const` is inlined at every point of use. There is no single memory location for a `const`; the compiler substitutes the value directly.

```rust
const MAX_USERS: u32 = 100_000;
const PI: f64 = 3.141592653589793;
const GREETING: &str = "Hello, world";

fn main() {
    println!("Max users: {MAX_USERS}");
    println!("Pi: {PI}");
}
```

Rules for `const`:

- The type must be explicit. There is no inference.
- The value must be a constant expression: known at compile time.
- The value cannot involve any runtime computation.
- The name must be `SCREAMING_SNAKE_CASE` by convention.

Constants can appear in any scope, including inside a function:

```rust
fn calculate_circle_area(radius: f64) -> f64 {
    const PI: f64 = 3.14159265358979;
    PI * radius * radius
}
```

### `static`: A Single Memory Location

A `static` has a fixed memory address. Every reference to a `static` points to the same location.

```rust
static APPLICATION_NAME: &str = "MyApp";
static MAX_CONNECTIONS: u32 = 1000;
```

The syntax looks similar to `const`, but the semantics differ. With `static`, the compiler creates one location in the program's data segment. References to `APPLICATION_NAME` all point to that single location.

When does the difference matter?

- A `const` cannot be referenced as `&CONSTANT_NAME`, because there is no single address to reference.
- A `static` can have a stable address, so it can be referenced.
- A `static` can have interior mutability or be `mut` (rare and unsafe).

In practice, almost all program-level constants should use `const`. Use `static` only when you specifically need a fixed memory address, for instance for FFI to C code or for lock-free data structures.

#### Mutable Statics

`static mut` exists but is `unsafe` to access:

```rust
static mut COUNTER: u32 = 0;

fn main() {
    unsafe {
        COUNTER += 1;
        println!("{COUNTER}");
    }
}
```

Modifying a global mutable variable from multiple threads is a data race, which Rust normally prevents at compile time. `static mut` exists for the rare cases where you genuinely need this, but in idiomatic Rust the answer is almost always to use `OnceLock`, `Mutex`, or `AtomicU32` instead.

### Choosing Between Them

Most code should use `const`. The decision tree:

| If you need...                                               | Use      |
|--------------------------------------------------------------|----------|
| A named compile-time value used by reference or by value     | `const`  |
| A program-wide value with a stable address                   | `static` |
| A program-wide value initialized at runtime                  | `OnceLock` (covered in a later module) |
| A program-wide mutable value                                 | `Mutex` or `AtomicU32` |

### Constants vs. Immutable `let` Bindings

A common question is why `const` exists when `let` already produces immutable bindings. The differences are:

| Property                            | `const`                  | `let`                          |
|-------------------------------------|--------------------------|--------------------------------|
| Scope                               | Any (including module)   | Function-local only            |
| Type annotation                     | Required                 | Optional                       |
| Initialization                      | Compile-time only        | Runtime allowed                |
| Inlined at use sites                | Yes                      | No                             |
| Can be `mut`                        | No                       | Yes (with `let mut`)           |
| Naming convention                   | `SCREAMING_SNAKE_CASE`   | `snake_case`                   |

A `const` is for values that are known when the program is written and that should be visible across the codebase. A `let` is for values computed during program execution, even if they happen not to change.

---

## E. Shadowing

Rust allows you to declare a new variable with the same name as an existing one. The new binding "shadows" the previous one. The previous binding is hidden but not destroyed.

```rust
fn main() {
    let x = 5;
    let x = x + 1;          // new binding, shadows the old x
    let x = x * 2;          // another new binding
    println!("{x}");        // prints 12
}
```

This is not mutation. Each `let` creates a new variable that happens to share a name with an old one. The old variable still exists in memory until its scope ends, but you can no longer refer to it by name.

### Shadowing vs. Mutation

The two look similar but are different operations:

```rust
// Shadowing: each let creates a new binding
let value = "5";              // value is &str
let value: i32 = value.parse().unwrap();   // value is i32
println!("{value}");

// Mutation: one binding, value reassigned
let mut value = 5;            // value is i32
value = 6;                    // still i32
// value = "five";            // compile error: cannot change type
```

The key difference is the type. Shadowing allows you to change the type; mutation does not. The first example is idiomatic Rust. The second cannot be expressed with mutation alone.

### Why Allow Shadowing?

Three concrete reasons.

#### Reason 1: Cleaner type conversions

A common pattern is taking a string from input, parsing it to a number, and using the number afterward. Without shadowing, you would need two different names:

```rust
// Without shadowing
let raw_input = "42";
let parsed_value: i32 = raw_input.parse().unwrap();
let doubled = parsed_value * 2;

// With shadowing
let value = "42";
let value: i32 = value.parse().unwrap();
let value = value * 2;
```

Shadowing lets you keep the same conceptual name as the value goes through transformations.

#### Reason 2: Limiting the lifetime of intermediate values

```rust
let user_input = read_line();                // String, contains whitespace
let user_input = user_input.trim();          // &str, trimmed
let user_input: u32 = user_input.parse()     // u32, parsed
    .expect("not a valid number");
```

Each successive shadow makes the previous form unreachable. The original `String` cannot be accidentally used after parsing. This is a small form of state machine: once a value has been transformed, the previous form is gone.

#### Reason 3: Cleaner control flow

Shadowing inside a block creates a binding that exists only for that block:

```rust
let x = 5;
{
    let x = x * 2;        // new x, only visible in this block
    println!("{x}");      // 10
}
println!("{x}");          // 5, the outer x is restored
```

This is useful for temporary computations that should not leak out of their context.

### When NOT to Shadow

Shadowing can be confusing when overused. A few guidelines:

- Shadow when the new value is a transformation of the old one and the same conceptual variable is intended.
- Do not shadow when the values are conceptually unrelated. Two different things deserve two different names.
- Do not shadow inside long functions where the original binding might still be relevant in a reader's mind.

The community convention is to use shadowing for type conversions and short pipelines, and to use distinct names for distinct concepts.

---

## F. Compound Types: Tuples and Arrays

Rust has two primitive compound types: tuples and arrays. Both have a fixed size known at compile time. Both are stored on the stack. Both copy by value when their elements are copyable.

### Tuples

A tuple groups a fixed number of values, possibly of different types, into one compound value.

```rust
let person: (String, u32, bool) = (String::from("Alice"), 30, true);
let pair = (1, "one");                     // type inferred as (i32, &str)
let single = (42,);                        // single-element tuple, the comma is required
let unit: () = ();                          // the empty tuple, called "unit"
```

The unit type `()` is significant. It is the implicit return type of functions that do not return anything meaningful. It has exactly one value, also written `()`.

#### Accessing Tuple Elements

There are two ways to extract values from a tuple.

**Direct field access** uses dot notation with a numeric index:

```rust
let point = (3.0, 4.0);
let x = point.0;
let y = point.1;
println!("({x}, {y})");
```

**Destructuring** is the more idiomatic approach:

```rust
let point = (3.0, 4.0);
let (x, y) = point;
println!("({x}, {y})");
```

Destructuring is a form of pattern matching. It works in `let` bindings, function parameters, `match` arms, and elsewhere. The pattern's shape must match the value's shape. The compiler checks this at compile time.

#### Common Uses

Tuples are useful for short-lived groupings, particularly returning multiple values from a function:

```rust
fn parse_coordinates(input: &str) -> (f64, f64) {
    let parts: Vec<&str> = input.split(',').collect();
    let x: f64 = parts[0].trim().parse().unwrap();
    let y: f64 = parts[1].trim().parse().unwrap();
    (x, y)
}

fn main() {
    let (x, y) = parse_coordinates("3.5, 7.2");
    println!("x = {x}, y = {y}");
}
```

When a tuple has more than three or four elements, or when the meaning of each position is not obvious from context, prefer a `struct` (covered in a later module). A `struct` gives names to fields, which is more readable than positional access.

### Arrays

An array is a fixed-size sequence of elements, all of the same type.

```rust
let numbers: [i32; 5] = [1, 2, 3, 4, 5];
let zeros = [0; 100];                     // 100 zeros: [0, 0, 0, ..., 0]
let words = ["one", "two", "three"];
```

The type `[i32; 5]` reads as "array of five `i32` values." The size is part of the type. An array of three integers is a different type from an array of four integers, and they cannot be assigned to the same variable.

#### Why fixed size?

Arrays have their size known at compile time. This means:

- They are stored entirely on the stack with no heap allocation.
- The compiler can check bounds statically in many cases.
- Memory layout is predictable, which is critical for embedded and systems work.

When you need a sequence whose size is determined at runtime, use `Vec<T>` instead. `Vec` is a heap-allocated, growable array. It is the correct choice for almost all general-purpose work. Arrays are for cases where the size is genuinely fixed and known.

#### Accessing Array Elements

Indexing uses square brackets:

```rust
let numbers = [10, 20, 30, 40, 50];
let first = numbers[0];
let third = numbers[2];

println!("First: {first}, third: {third}");
```

Out-of-bounds access is a runtime panic, not a memory safety violation:

```rust
let numbers = [1, 2, 3];
let item = numbers[10];     // panic: index out of bounds
```

This is one of the central safety guarantees: a Rust array index can never read past the end of the array, regardless of whether the index is a constant or a runtime value. The compiler inserts bounds checks where it cannot prove safety statically. In release builds with the optimizer, many of these checks are eliminated when the optimizer can prove the index is in range.

#### Iterating Over Arrays

The idiomatic way to walk an array is with an iterator:

```rust
let numbers = [1, 2, 3, 4, 5];
for n in numbers {
    println!("{n}");
}

let sum: i32 = numbers.iter().sum();
let doubled: Vec<i32> = numbers.iter().map(|n| n * 2).collect();
```

You should rarely write `for i in 0..numbers.len()`. Iterators are equally fast and harder to misuse. clippy will flag manual index loops, as you saw in Module 2's lab.

### Slices

Closely related to arrays is the slice type `&[T]`. A slice is a reference to a contiguous sequence of elements, without owning them. A slice does not have a compile-time-known size; it has a runtime length.

```rust
let array = [1, 2, 3, 4, 5];
let whole: &[i32] = &array;
let middle: &[i32] = &array[1..4];      // [2, 3, 4]

println!("Length: {}", middle.len());
```

Slices are how functions usually take array-like input. A function that accepts `&[i32]` works with arrays of any size, with `Vec<i32>`, and with sub-ranges of either:

```rust
fn sum_all(values: &[i32]) -> i32 {
    values.iter().sum()
}

fn main() {
    let array = [1, 2, 3, 4, 5];
    let vector = vec![10, 20, 30];

    println!("{}", sum_all(&array));         // 15
    println!("{}", sum_all(&vector));        // 60
    println!("{}", sum_all(&array[1..3]));   // 5
}
```

This pattern is one of the first places Rust's reference system pays off in practice. Slices are covered in greater depth in the ownership and borrowing module.

### Tuples vs. Arrays

A summary of when to use each:

| Scenario                                              | Use      |
|-------------------------------------------------------|----------|
| Fixed number of values, different types               | Tuple    |
| Fixed number of values, same type                     | Array    |
| Many values, same type, fixed size known at compile   | Array    |
| Many values, same type, size determined at runtime    | `Vec<T>` |
| Two or three return values from a function           | Tuple    |
| More than three named fields                          | `struct` |

---

## Module Summary

- Rust scalar types are integers (eleven variants), floats (`f32` and `f64`), `bool`, and `char` (a four-byte Unicode scalar value).
- Type inference handles most variable declarations. Annotations are required when the compiler cannot determine the type from context, such as with empty collections or generic function results.
- Variables are immutable by default. The `mut` keyword is the explicit opt-in for reassignment.
- `const` defines compile-time constants that are inlined at every use site. `static` defines values with fixed memory addresses. Use `const` for almost all program-wide values.
- Shadowing creates a new binding with an existing name and is the idiomatic way to transform values, including across type changes.
- Tuples group fixed numbers of differently-typed values. Arrays hold a fixed number of same-typed values, with bounds checked at runtime. Slices reference contiguous ranges without owning them.

## Discussion Questions

1. The default integer type is `i32` and the default float type is `f64`. Why are the defaults different widths? What does each choice optimize for?
2. Mutable bindings require an explicit `mut` keyword, but immutable bindings are free. What if Rust had made the choice the other way: implicit mutability with an explicit `let immut`? Identify two specific code patterns that would become more error-prone.
3. Shadowing and mutation can both be used to "change" a variable. Construct a small example where shadowing produces clearer code than mutation, and a second example where mutation is clearer than shadowing.

---
