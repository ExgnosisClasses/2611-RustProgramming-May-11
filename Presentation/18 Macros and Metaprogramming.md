# Module 18: Macros and Metaprogramming

## Module Overview

Macros are Rust's way of generating code at compile time. You have been using them throughout the course without thinking much about them: `println!`, `vec!`, `format!`, `assert_eq!`, `#[derive(Debug)]`, `#[test]` — all of these are macros. This module explains what they are, how they work, and how to write your own.

There are two distinct kinds of macros in Rust. **Declarative macros** (written with `macro_rules!`) are pattern-based: you describe what input should look like and what code to generate for each pattern. They are useful for simple cases like variadic constructors or DSLs. **Procedural macros** are full Rust programs that run at compile time, taking code as input and producing code as output. They are how `#[derive(Debug)]`, `#[tokio::main]`, and `sqlx::query!` work.

Three ideas are central:

1. **Macros generate code; functions process values.** A function takes runtime values and computes runtime results. A macro takes source code as input and produces source code as output, before the program is compiled. This distinction shapes everything else about them.
2. **Macros are not free.** They expand to ordinary Rust code that the compiler then checks. Their generated code follows all the normal rules (types, ownership, lifetimes). The cost is compile time and a sometimes-mysterious error experience when generated code fails to compile.
3. **Prefer functions and generics when they suffice.** Macros solve specific problems that functions cannot: variadic arguments, taking expressions as input, generating implementations across many types. Reaching for a macro before exhausting non-macro options is usually overreach.

By the end of this module, students will:

- Read and write declarative macros with `macro_rules!`.
- Understand what the common built-in macros (`vec!`, `println!`, `format!`, `todo!`, `unimplemented!`) do under the hood.
- Recognize the three kinds of procedural macros and know when each is used.
- Understand how derive macros (`#[derive(...)]`) generate trait implementations.
- Recognize attribute macros (`#[tokio::main]`, `#[test]`) and function-like macros (`sqlx::query!`).
- Decide when a problem genuinely calls for a macro versus a function, generic, or trait.

This module is mostly about reading and understanding macros rather than writing them. Most production Rust code uses macros constantly but defines them rarely. Knowing how they work is essential for reading any nontrivial codebase; writing them is a more specialized skill that comes up in library development.

> **A note on prerequisites.** This module touches on syntax tree concepts (tokens, parsing) that are not part of the language proper but help in understanding what macros do. Procedural macros also require a separate crate type, which is a deviation from the rest of Rust's build system. The module focuses on conceptual understanding; writing custom procedural macros is a substantial separate topic that real libraries (`thiserror`, `serde`, `tokio`) handle for you.

---

## A. Declarative Macros with `macro_rules!`

Declarative macros are the simpler of the two kinds. They work by pattern matching on the source code passed to them.

### A First Example

Here is a tiny macro that prints a banner:

```rust
macro_rules! banner {
    () => {
        println!("======================");
        println!("=== HERE IS A BANNER ===");
        println!("======================");
    };
}

fn main() {
    banner!();
    println!("normal code in between");
    banner!();
}
```

Output:

```
======================
=== HERE IS A BANNER ===
======================
normal code in between
======================
=== HERE IS A BANNER ===
======================
```

The macro takes no arguments (the `()` matches an empty invocation) and expands to three `println!` calls. At each call site, the compiler replaces `banner!();` with the three lines from the macro body.

The macro is invoked with the `!` suffix: `banner!()`. This is how Rust distinguishes macros from function calls.

### The Anatomy of `macro_rules!`

The general structure:

```rust
macro_rules! macro_name {
    (matcher) => { expansion };
    (matcher) => { expansion };
    // ... more rules
}
```

A macro can have multiple rules. Each rule has:

- A **matcher** in parentheses or brackets that describes what the macro accepts.
- A **fat arrow** `=>`.
- An **expansion** in braces that describes what to generate.

When you invoke the macro, Rust tries each rule in order and uses the first matcher that fits.

### Macros with Arguments

A macro can take arguments, called *metavariables*:

```rust
macro_rules! say_hello {
    ($name:expr) => {
        println!("Hello, {}!", $name);
    };
}

fn main() {
    say_hello!("Alice");
    say_hello!(String::from("Bob"));
}
```

Output:

```
Hello, Alice!
Hello, Bob!
```

The matcher `$name:expr` says: "capture an expression and bind it to the name `name`." The expansion uses `$name` to substitute the captured expression.

The `:expr` part is a *fragment specifier*. It says what kind of syntax fragment to expect. Common fragment specifiers include:

- `expr` — an expression (`5`, `x + 1`, `vec![1, 2, 3]`).
- `ident` — an identifier (`foo`, `my_variable`, but not `5` or `x + 1`).
- `ty` — a type (`i32`, `Vec<String>`, `&str`).
- `pat` — a pattern (`Some(x)`, `(a, b)`, `0..=9`).
- `stmt` — a statement.
- `block` — a brace-delimited block.
- `item` — an item like a function or struct definition.
- `path` — a path like `std::collections::HashMap`.
- `tt` — a single token tree (the most flexible).

Each specifier constrains what kind of input the macro accepts. If you write `($name:expr)` and the user passes something that is not an expression, the macro fails to expand.

### Multiple Patterns

A macro with multiple rules handles different argument shapes:

```rust
macro_rules! greet {
    () => {
        println!("Hello!");
    };
    ($name:expr) => {
        println!("Hello, {}!", $name);
    };
    ($greeting:expr, $name:expr) => {
        println!("{}, {}!", $greeting, $name);
    };
}

fn main() {
    greet!();
    greet!("Alice");
    greet!("Hi", "Bob");
}
```

Output:

```
Hello!
Hello, Alice!
Hi, Bob!
```

Three different invocations match three different rules. The rules are tried in order; the first matching rule wins.

This is how macros can have "optional" arguments or variable-arity behavior, which Rust's normal function syntax does not allow.

### Repetition

The killer feature of declarative macros is repetition. A matcher can capture a variable number of inputs:

```rust
macro_rules! print_all {
    ( $( $item:expr ),* ) => {
        $(
            println!("{}", $item);
        )*
    };
}

fn main() {
    print_all!(1, 2, 3);
    print_all!("hello", "world");
    print_all!();
}
```

Output:

```
1
2
3
hello
world
```

The matcher `$( $item:expr ),*` reads as: "zero or more `$item:expr` expressions, separated by commas." The expansion `$( println!("{}", $item); )*` reads as: "for each captured `$item`, generate this code."

The `*` means "zero or more." Other repetition operators are:

- `+` for "one or more."
- `?` for "zero or one" (optional).

The separator (here, `,`) can be any token. Common separators are commas and semicolons.

### A Practical Example: A Min Macro

A macro that finds the minimum of any number of arguments:

```rust
macro_rules! min {
    ($first:expr) => {
        $first
    };
    ($first:expr, $( $rest:expr ),+) => {
        {
            let f = $first;
            let r = min!($( $rest ),+);
            if f < r { f } else { r }
        }
    };
}

fn main() {
    let m = min!(3, 7, 1, 8, 4);
    println!("min: {m}");
}
```

Output:

```
min: 1
```

The macro has two rules:

- For a single argument, the result is that argument.
- For multiple arguments, take the minimum of the first and the (recursive) minimum of the rest.

This recursive structure is common in declarative macros: a base case and a recursive case that reduces the input.

### Hygiene

Declarative macros are *hygienic*: names introduced inside the macro do not conflict with names in the surrounding code, and vice versa. This prevents a class of bugs that plague macros in languages like C.

Consider:

```rust
macro_rules! double {
    ($x:expr) => {
        {
            let x = 2;        // local x inside the macro
            $x * x             // ERROR: x here refers to... what?
        }
    };
}

fn main() {
    let x = 5;
    let result = double!(x);
    println!("{result}");
}
```

In C-style macros, the `let x = 2` inside the macro would clash with the `$x` substitution (also `x` from the caller). Rust's hygiene prevents this: the `x` introduced by the macro and the `x` substituted from the caller are different variables, treated as if they had different names.

Hygiene makes macros safer to write and use. You do not have to worry about your macro's internal variable names colliding with the caller's names.

### When to Use Declarative Macros

Declarative macros are appropriate when:

- You need variadic arguments (any number of arguments of the same kind).
- You want a syntactic shortcut that does not fit a function (e.g., taking patterns or types as input).
- The macro is small and the pattern is clear.

They are less appropriate for:

- Complex code generation. Procedural macros (Section C) handle complexity better.
- When a function or generic would suffice. Macros add cognitive load; use them when they pull their weight.
- When error messages matter. Macro errors are notoriously confusing; generated code that fails to compile produces errors that point inside the macro expansion, often confusingly.

The standard library uses declarative macros for `vec!`, `println!`, `format!`, and similar utilities. These all benefit from variadic syntax that functions cannot provide.

---

## B. Common Built-In Macros

The standard library provides a small set of macros that appear in nearly every Rust program. Knowing what each one does (and why it is a macro rather than a function) helps in reading code.

### `println!`, `print!`, `eprintln!`, `eprint!`

The print family writes formatted text to stdout (or stderr for the `e` variants):

```rust
println!("hello");
println!("count: {}", 42);
println!("{name}: {age}", name = "Alice", age = 30);
println!("{:.2}", 3.14159);             // 3.14

eprintln!("error: file not found");
```

Why these are macros rather than functions: the format string `"count: {}"` is parsed at compile time. The compiler checks that the number of `{}` placeholders matches the number of arguments and that each argument's type implements the appropriate formatting trait. If you misuse `{}`, you get a compile error.

A function-based equivalent would have to accept variadic arguments (which Rust does not support) and would defer the format-string check to runtime. The macro version catches mistakes earlier and is more efficient.

### `format!`

Like `println!` but returns a `String` instead of printing:

```rust
let name = "Alice";
let age = 30;
let greeting = format!("Hello, {name}! You are {age}.");
println!("{greeting}");
```

`format!` is the standard way to construct strings from multiple pieces. It is more flexible than the `+` operator and more efficient than building a `Vec<String>` and joining it.

The same compile-time checks apply: invalid format strings or type mismatches are caught early.

### `vec!`

The standard way to create a `Vec`:

```rust
let v = vec![1, 2, 3, 4, 5];
let zeros = vec![0; 100];                // 100 zeros
let empty: Vec<i32> = vec![];
```

`vec!` accepts three forms:

- A comma-separated list of expressions, producing a `Vec` containing them.
- An expression, a semicolon, and a count, producing a `Vec` with the expression repeated.
- An empty pair of brackets, producing an empty `Vec`.

Why this is a macro: the first form is variadic (any number of expressions). The second form has a non-function-like syntax (`expr; count`). Neither fits a function signature in Rust.

The macro expands to code like:

```rust
// vec![1, 2, 3] expands roughly to:
{
    let mut v = Vec::new();
    v.push(1);
    v.push(2);
    v.push(3);
    v
}
```

(The actual expansion is more sophisticated; it pre-allocates the right capacity.)

### `assert!`, `assert_eq!`, `assert_ne!`

Testing assertions:

```rust
assert!(condition);
assert!(condition, "custom message: {value}");

assert_eq!(actual, expected);
assert_ne!(actual, unexpected);
```

Why these are macros: they capture the source code of their arguments for error messages. When `assert_eq!(x, y)` fails, the message includes both the literal source ("assertion failed: x == y") and the runtime values. A function would only have access to the values, not the source.

The `assert_eq!` failure message is one of the small features that makes Rust testing pleasant:

```
assertion `left == right` failed
  left: 4
 right: 5
```

You see both what was expected and what was produced. Each comes with the source expression that produced it.

### `todo!` and `unimplemented!`

Placeholders for code you have not written yet:

```rust
fn compute_taxes(income: f64) -> f64 {
    todo!()
}

fn rare_case() -> String {
    unimplemented!("not yet supported")
}
```

Both panic at runtime with a message indicating that the code is missing. The difference is intent:

- `todo!()` says "I plan to write this; remind me." It is for work in progress.
- `unimplemented!()` says "this case is deliberately not handled (yet)." It is for cases that may or may not get implemented.

The practical difference is minor; both panic. The semantic difference is what you communicate to readers (including yourself).

These macros are useful for sketching out an API or implementation. You declare the function signatures, fill in `todo!()` bodies, and the code compiles. You can then implement each body one at a time, with the compiler reminding you (via panic at test time, or via warnings) about the ones still missing.

Why these are macros: they return `!` (the never type) so the function signature compiles even though no value is returned. A function returning `!` would be possible, but the macros also accept an optional message and capture the source location for the panic, both of which need macro-level features.

### `panic!`

The primitive for unrecoverable errors:

```rust
panic!("something went very wrong");
panic!("expected positive, got {value}");
```

`panic!` aborts the program (or unwinds the stack, depending on configuration) with the given message. It is the foundation under `assert!`, `unreachable!`, and the other panic-related macros.

Why it is a macro: it captures source location (file, line) and supports format-string syntax. A function could do the formatting, but accessing the source location from inside a function would be awkward.

### `dbg!`

A debugging convenience:

```rust
let x = 5;
let y = dbg!(x * 2);            // prints "[src/main.rs:5] x * 2 = 10"
                                  // and returns 10
```

`dbg!` prints its argument's value (with source location) and returns the value unchanged. This makes it easy to inspect intermediate values without permanently restructuring the code:

```rust
let total = dbg!(prices.iter().sum::<f64>()) * 1.1;
```

The `dbg!` is removed when no longer needed. The pattern (insert, observe, remove) is much smoother than manual print-debugging.

Why it is a macro: it captures the source code of its argument for the printed label. The output "x * 2 = 10" includes the literal text "x * 2"; a function would only see the value 10 and could not produce that label.

### `include_str!` and `include_bytes!`

Compile-time file inclusion:

```rust
const SHADER_SOURCE: &str = include_str!("shader.glsl");
const ICON_DATA: &[u8] = include_bytes!("icon.png");
```

These read a file at compile time and embed its contents as a `&'static str` or `&'static [u8]`. The file must exist when the program is compiled; missing files produce compile errors.

The macros are useful for bundling assets into a binary: configuration templates, shader source, small images, license text. The data ends up in the executable; no external files need to ship.

Why these are macros: they read files at compile time, before the program runs. Functions execute at runtime. The compile-time nature is essential.

### `env!`, `option_env!`

Compile-time environment variable access:

```rust
const VERSION: &str = env!("CARGO_PKG_VERSION");
const BUILD_INFO: Option<&str> = option_env!("BUILD_INFO");
```

`env!` reads an environment variable at compile time and embeds its value. Missing variables produce compile errors. `option_env!` returns `Some(value)` or `None` based on whether the variable is set.

These are useful for embedding build-time information into the binary: version numbers (typically set by Cargo), build identifiers (set by CI), feature flags.

---

## C. Introduction to Procedural Macros

Procedural macros are full Rust programs that run at compile time. They take Rust syntax as input and produce Rust syntax as output. There are three kinds:

- **Derive macros:** `#[derive(MyTrait)]` on a struct or enum. The macro generates a trait implementation.
- **Attribute macros:** `#[my_attr]` on items. The macro transforms the item.
- **Function-like macros:** `my_macro!(input)`. Like declarative macros but with arbitrary code generation logic.

### How They Differ From Declarative Macros

A declarative macro is just pattern matching: input syntax → output syntax via templates. A procedural macro is a Rust function that takes a `TokenStream` (a representation of source code) and returns a `TokenStream`.

The function can do anything Rust can do: parse the input syntactically, analyze it, generate new syntax based on what it found. This is much more powerful than `macro_rules!`:

- Generate code that depends on field types (e.g., `#[derive(Serialize)]` looks at each field's type).
- Validate input and emit good error messages.
- Apply complex transformations that pattern matching cannot express.

The cost is complexity. Writing a procedural macro requires understanding the Rust syntax tree (the `syn` crate is the standard parser), generating code (the `quote` crate is the standard tool), and managing a separate crate type (`proc-macro = true` in Cargo.toml).

### A Procedural Macro Crate

A procedural macro lives in its own crate with a special configuration:

```toml
# Cargo.toml
[package]
name = "my_macros"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
syn = "2"
quote = "1"
proc-macro2 = "1"
```

The `proc-macro = true` line marks this as a procedural macro crate. The crate can only export procedural macros; it cannot contain regular code that other crates call.

A user of the macro adds it as a normal dependency:

```toml
[dependencies]
my_macros = { path = "../my_macros" }
```

And uses the macros it exports. The build system handles the rest.

### The Three Kinds in More Detail

The next three sections cover each kind. Each follows the same general pattern (a function that takes and returns `TokenStream`) but has a distinct call syntax and use case.

You will use procedural macros (almost) constantly without writing them; libraries provide them and you apply them with attributes or derives. The point of this section is to understand what is happening when you see them.

---

## D. Derive Macros: `#[derive(...)]`

Derive macros are the most common procedural macros in practice. You have used them throughout the course:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct UserId(u64);

#[derive(Serialize, Deserialize)]
struct Config {
    name: String,
    port: u16,
}

#[derive(Error)]
enum MyError {
    NotFound,
    Other(String),
}
```

Each entry in the `derive(...)` list invokes a derive macro that generates a trait implementation for the struct or enum.

### What a Derive Macro Sees

A derive macro receives the entire item it is applied to. For `#[derive(Debug)] struct Point { x: i32, y: i32 }`, the macro receives a token stream representing:

```rust
struct Point {
    x: i32,
    y: i32,
}
```

It can inspect the struct's name, fields, field types, attributes, generic parameters, and so on. Based on this, it generates a trait implementation.

For `#[derive(Debug)]`, the generated code looks roughly like:

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

The macro walked the struct's fields and generated a corresponding `.field(...)` call for each. Different structs produce different generated code; the macro adapts to the input.

### Standard-Library Derive Macros

The standard library provides derive macros for:

- `Debug` — produces `{:?}` formatting.
- `Clone` — duplicates the value via cloning each field.
- `Copy` — opt-in to bitwise copy semantics (requires all fields to be `Copy`).
- `PartialEq` and `Eq` — equality comparison.
- `PartialOrd` and `Ord` — ordering comparison.
- `Hash` — hashing for use as a HashMap key.
- `Default` — default value via each field's default.

These derives work for any type whose fields support the underlying trait. For example, `#[derive(Clone)]` works only if every field is `Clone`. The compiler checks this and emits an error if a field does not satisfy the requirement.

### Third-Party Derive Macros

Libraries provide their own derive macros. Common ones you have seen:

- `#[derive(Serialize, Deserialize)]` from `serde` — generates serialization code for many formats (JSON, TOML, MessagePack, etc.).
- `#[derive(Error)]` from `thiserror` — generates `Display` and `Error` implementations for error enums.
- `#[derive(Parser)]` from `clap` — generates command-line argument parsing code.
- `#[derive(StructOpt)]` (older alternative to `clap::Parser`).
- `#[derive(sqlx::FromRow)]` — generates code to construct a struct from a database row.

Each of these takes the struct or enum it is applied to and generates substantial amounts of code. The user writes one line; the macro generates dozens or hundreds.

### Helper Attributes

Many derive macros accept "helper attributes" that customize generation:

```rust
#[derive(Serialize, Deserialize)]
struct User {
    #[serde(rename = "user_id")]
    id: u64,

    #[serde(default)]
    email: Option<String>,

    #[serde(skip_serializing_if = "Vec::is_empty")]
    tags: Vec<String>,
}
```

The `#[serde(...)]` attributes on individual fields configure how each field is serialized. The derive macro reads these attributes and adjusts its output accordingly.

This is one of the most powerful patterns in modern Rust: declarative configuration via attributes, with the implementation generated by the macro.

### When Derive Macros Fail

Derive macros can produce compile errors if the input does not satisfy their requirements:

```rust
#[derive(Copy, Clone)]
struct User {
    name: String,        // ERROR: String is not Copy
}
```

```
error[E0204]: the trait `Copy` may not be implemented for this type
  |
  | name: String,
  |       ------ this field does not implement `Copy`
```

The error comes from the generated code. The macro generated `impl Copy for User`, and the compiler checks the implementation. Since `String` does not implement `Copy`, the implementation is invalid, and the compiler emits an error.

The error message can sometimes feel obscure because it refers to code you did not write. With practice, you learn to recognize "this comes from the derive" and check the relevant trait's requirements.

### How `thiserror`'s Derive Works

A concrete example. The `thiserror` crate provides `#[derive(Error)]`:

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
}
```

The macro generates two impls. First, `Display`:

```rust
impl std::fmt::Display for ParseError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ParseError::Empty => write!(f, "file is empty"),
            ParseError::NotANumber(s) => write!(f, "expected number, got '{}'", s),
            ParseError::OutOfRange { value, max } => {
                write!(f, "number {} out of range (max {})", value, max)
            }
        }
    }
}
```

Second, `std::error::Error`:

```rust
impl std::error::Error for ParseError {}
```

The macro parses each variant's `#[error("...")]` attribute and generates the appropriate match arm. The format string references fields by name (`{value}`) or by index (`{0}`); the macro emits the corresponding access in the generated code.

This is a real procedural macro doing real work. The user writes a few lines of declarative attributes; the macro produces the trait implementations that would otherwise be many lines of boilerplate.

---

## E. Attribute Macros

Attribute macros are applied to items using `#[attribute_name]` (or `#[attribute_name(args)]`). The macro can transform the item in any way it chooses.

### How They Look in Use

You have seen attribute macros throughout the course:

```rust
#[tokio::main]
async fn main() {
    // ...
}

#[test]
fn my_test() {
    // ...
}

#[wasm_bindgen]
pub fn js_callable() {
    // ...
}
```

Each `#[...]` is an attribute macro that transforms the function it precedes.

### What an Attribute Macro Does

An attribute macro receives two token streams:

- The arguments to the attribute (the part inside the parentheses, if any).
- The item the attribute is applied to.

It returns a single token stream that replaces the original item. The replacement can be the original, a modified version, or something entirely different.

### Example: `#[tokio::main]`

Async `main` functions are not directly supported by Rust. The compiler does not know how to run an async function as the program's entry point. `#[tokio::main]` solves this by transforming the function:

You write:

```rust
#[tokio::main]
async fn main() {
    let result = fetch_data().await;
    println!("{result:?}");
}
```

The macro transforms this into roughly:

```rust
fn main() {
    let runtime = tokio::runtime::Runtime::new().unwrap();
    runtime.block_on(async {
        let result = fetch_data().await;
        println!("{result:?}");
    });
}
```

The async function becomes a synchronous wrapper that creates a Tokio runtime and runs the async body on it. The user writes async-looking code; the macro generates the runtime setup.

This is what makes the async `main` pattern work. Without the attribute macro, you would write all this boilerplate yourself.

### Example: `#[test]`

The `#[test]` attribute is built into the compiler (not technically a procedural macro, but it behaves like one). It marks a function as a test:

You write:

```rust
#[test]
fn it_works() {
    assert_eq!(2 + 2, 4);
}
```

The compiler, when building in test mode (`cargo test`), collects all `#[test]`-marked functions into a generated test harness that runs them and reports results. In a non-test build, the marked function is omitted entirely.

The transformation is more elaborate than a simple wrapper, but the principle is the same: an attribute that triggers code generation around the marked item.

### Example: Web Framework Routes

Many web frameworks use attribute macros for route handlers:

```rust
#[get("/users/{id}")]
async fn get_user(id: u64) -> User {
    // ...
}
```

The `#[get(...)]` attribute generates the routing registration code: a hidden function that registers `get_user` to handle GET requests to `/users/{id}`. The framework collects these registrations at startup.

This is much cleaner than the alternative (manually calling `router.add_route("/users/{id}", get_user)` for every handler). The attribute keeps the route definition next to the handler.

### When Attribute Macros Are Appropriate

Attribute macros are appropriate when:

- The transformation is large enough that requiring users to write it manually would be burdensome.
- The transformation can be described declaratively (the user provides annotations; the macro generates code).
- The transformation is item-level (it changes how a function or struct works, not just inserts code at a call site).

Common uses:

- Framework integration (async runtimes, web servers, GUI frameworks).
- Aspect-oriented features (logging, timing, retries).
- DSLs that fit the "transformation of items" model.

### When They Become Problematic

Attribute macros can be heavy:

- They are invisible. Looking at `#[my_attr] fn foo()`, you see only the original function. The transformed code is not in the source. This can make debugging hard.
- Errors from generated code can be confusing.
- They slow compilation. Procedural macros run as part of compilation; complex ones add real time.

For most code, you use attribute macros provided by libraries; you do not write them yourself. The libraries' macros are tested and documented. Writing your own is a substantial project.

---

## F. Function-Like Procedural Macros

Function-like procedural macros look like declarative macros at the call site (with `!`) but are implemented as procedural macros:

```rust
let query = sqlx::query!("SELECT id, name FROM users WHERE active = ?", true);
let html = html! { <div>Hello, {name}!</div> };
```

The `query!` macro from `sqlx` parses the SQL string, validates it against your database schema at compile time, and generates code that executes the query. The `html!` macro (from various HTML-templating libraries) parses HTML-like syntax and generates the equivalent Rust code.

### Why They Are Procedural

A declarative macro could not do what these do. Declarative macros are limited to pattern-based template expansion. To parse SQL and validate against a schema, the macro needs to:

1. Run Rust code at compile time.
2. Connect to a database.
3. Execute the query as a prepared statement.
4. Inspect the resulting types.
5. Generate Rust code that materializes those types.

That is a full program's worth of logic, not a pattern. Procedural macros enable it.

### When You See One

The syntax `name!(...)` could be either a declarative macro or a function-like procedural macro. The difference is invisible at the call site; both look the same. The implementation is what differs.

You can tell by looking at the macro's source (if available): a declarative macro uses `macro_rules!`, a procedural one uses `#[proc_macro]` and friends. For users, the distinction usually does not matter; both are macros that take input and produce code.

### Common Examples

- `sqlx::query!`, `sqlx::query_as!` — compile-time SQL validation.
- `regex!` (from various crates) — compile-time regex validation.
- `lazy_static!` (declarative, but similar in feel) — lazy static initialization.
- `format_args!` — used internally by `format!`, `println!`, etc.
- `include_str!`, `include_bytes!`, `env!`, `option_env!` — built-in compile-time data.

### A Practical Example: `sqlx::query!`

The motivating use case: SQL queries with compile-time validation.

```rust
let user_id = 42;
let result = sqlx::query!(
    "SELECT name, email FROM users WHERE id = ?",
    user_id
).fetch_one(&pool).await?;

println!("name: {}, email: {}", result.name, result.email);
```

The `query!` macro:

1. Parses the SQL string at compile time.
2. Connects to the database (using DATABASE_URL environment variable).
3. Validates that the query is syntactically correct.
4. Determines the types of the columns returned (`name: String`, `email: String`).
5. Generates a struct with those field types.
6. Generates code to execute the query and parse the result into the struct.

If the query has a typo (`SELCT` instead of `SELECT`), compile error. If a column does not exist, compile error. If a parameter is the wrong type, compile error.

This is impossible with a regular function or declarative macro. The compile-time database connection and type inference require procedural macro power.

The cost is that compilation requires a working database, which complicates some workflows. The benefit is that SQL bugs that would normally appear at runtime are caught at compile time.

---

## G. When to Use Macros vs. Functions or Generics

Macros are powerful but have real costs. The question to ask before writing one: "could a function or generic do this?"

### When a Function Suffices

Most code does not need a macro. A function can:

- Take arguments and return a value.
- Use generics to work with multiple types.
- Use traits to abstract over behavior.
- Be reused across modules and crates.

If your problem fits this shape, write a function. Functions are easier to read, easier to debug, easier to test, and produce clearer error messages.

A common antipattern: writing a macro for what is essentially a function. For example:

```rust
// Antipattern: macro for a simple computation
macro_rules! double {
    ($x:expr) => { $x * 2 };
}

// Better: a function
fn double(x: i32) -> i32 {
    x * 2
}
```

The macro version offers no benefit over the function. The function is clearer, has a type signature, integrates with IDEs, and works in any context. The macro is just noise.

### When a Generic Suffices

If you want one piece of code to work for multiple types, use a generic function (or struct, or trait). Macros are not the right tool for type-level generality:

```rust
// Antipattern: macro to swap two values
macro_rules! swap {
    ($a:expr, $b:expr) => {
        let temp = $a;
        $a = $b;
        $b = temp;
    };
}

// Better: a generic function
fn swap<T>(a: &mut T, b: &mut T) {
    std::mem::swap(a, b);
}
```

The generic function handles any type. The macro just generates the same swap code repeatedly without giving you the benefits of types (signature documentation, IDE hints, error messages).

The standard library's `std::mem::swap` is exactly this generic function; you do not need a macro for swap.

### When Macros Are the Right Tool

Macros are appropriate when:

- **Variadic arguments.** `println!`, `vec!`, `format!`. Rust does not support variadic functions; macros fill the gap.
- **Compile-time information.** `include_str!`, `env!`. Functions execute at runtime; you need a macro to act at compile time.
- **Source-code capture.** `dbg!`, `assert_eq!`. The macro needs to see the literal source code of arguments to produce useful output.
- **Code generation across many types.** `#[derive(Debug)]`. Implementing `Debug` for every struct by hand is tedious; the macro generates the boilerplate.
- **DSLs.** SQL, HTML templates, query builders. The macro accepts syntax that does not fit normal Rust.
- **Cross-cutting transformations.** `#[tokio::main]`, `#[test]`. The transformation applies to items, not values.

When you find yourself wanting one of these features, a macro is the appropriate tool. Otherwise, prefer non-macro solutions.

### A Decision Tree

When tempted to write a macro, ask:

1. **Could a function do this?** If yes, write the function. Done.
2. **Could a generic function do this?** If yes, write the generic. Done.
3. **Could a trait abstract this?** Define the trait and implement it.
4. **Does this need variadic arguments or compile-time evaluation?** A macro might be necessary.
5. **Is the macro replacing a single line of code?** Probably overengineering. Reconsider.
6. **Does the macro need full code-analysis power?** Use a procedural macro. Reach for `thiserror` or `serde`-style designs.
7. **Is the macro just pattern-based substitution?** Use `macro_rules!`.

Most of the time, the answer is "use a function" or "use a generic." Macros solve specific problems; do not reach for them as a default.

### The Cost of Macros

Tangible costs:

- **Compile time.** Every macro invocation runs at compile time. Heavy macro usage slows builds.
- **Error messages.** Errors in generated code often point inside the macro expansion, which can be confusing.
- **Tooling.** IDEs sometimes struggle with macro-heavy code (autocomplete, go-to-definition).
- **Readability.** Macros are less familiar than functions; readers must learn their conventions.

Intangible costs:

- **Surprises.** A macro might do something unexpected (introducing variables, capturing scope unexpectedly, generating multiple statements where one was expected).
- **Maintenance.** Macros are harder to modify than functions. Refactoring a macro often requires understanding every invocation site.

These costs are real but acceptable for macros that genuinely solve a problem. The standard library's macros (`vec!`, `println!`, etc.) are universally familiar and well-documented. The cost of using them is minimal; the benefit is substantial.

For custom macros, the cost-benefit calculation is harder. A macro that saves a few lines but introduces confusion is rarely worth it. A macro that eliminates substantial boilerplate (like `thiserror`'s `Error` derive) can be transformative.

### A Closing Note on Procedural Macros

Procedural macros are particularly costly to write. They require:

- A separate crate type.
- Dependencies on `syn`, `quote`, `proc-macro2`.
- Understanding of the Rust syntax tree.
- Careful attention to error messages (since errors in generated code are confusing).
- Documentation, because users cannot easily inspect the generated code.

For most projects, the right approach is to use procedural macros that already exist (from `serde`, `thiserror`, `tokio`, `clap`, etc.) rather than writing new ones. The existing ones have been refined over time and handle edge cases that a new macro would miss.

If you do need a custom procedural macro, expect it to be a substantial project: think days, not hours. The result is often worth it for the right problem, but the investment is real.

---

## Module Summary

- Macros generate code at compile time. Functions process values at runtime. The distinction shapes everything else.
- Declarative macros (`macro_rules!`) work by pattern matching: input syntax to output syntax via templates.
- Common built-in macros (`println!`, `vec!`, `format!`, `assert_eq!`, `dbg!`, `todo!`) exist because they need variadic arguments, compile-time format checking, or source-code capture.
- Procedural macros are Rust programs that run at compile time. There are three kinds: derive, attribute, and function-like.
- Derive macros (`#[derive(...)]`) generate trait implementations from struct or enum definitions.
- Attribute macros (`#[my_attr]`) transform items: functions, structs, modules.
- Function-like macros (`name!(...)`) look like declarative macros but can use arbitrary code generation logic.
- Prefer functions and generics when they suffice. Macros are for specific problems: variadic arguments, compile-time evaluation, source capture, code generation across many types, DSLs.

## Discussion Questions

1. The standard library uses `println!` and `vec!` as macros rather than functions. Identify a specific feature each macro provides that a function could not. Could the language have been designed differently to make these features available to functions?
2. Procedural macros run at compile time and can do anything Rust can do (read files, connect to databases, generate code). This power has trade-offs. Identify a specific risk or downside of compile-time code generation that does not apply to runtime computation.
3. Many derive macros (`Serialize`, `Deserialize`, `Error`) use helper attributes (`#[serde(rename = "...")]`) to customize their behavior. What does this design pattern give you compared to alternatives like trait methods or builder functions? When is it the right approach?

#