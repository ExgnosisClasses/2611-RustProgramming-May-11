# Module 10: Error Handling

## Module Overview

Error handling in Rust is built on the foundations from Module 9. The `Result<T, E>` enum and pattern matching together form a complete system for handling failures, and the `?` operator makes that system ergonomic. This module covers the system in depth, plus the conventions and libraries that the Rust community has standardized around.

The interesting parts of this module are not the syntax but the design philosophy. Three ideas are central:

1. **Errors are values, not control flow.** A function that can fail returns a `Result<T, E>`. The error is data the caller can inspect, transform, log, or propagate. There is no separate exception channel.
2. **The compiler enforces error handling.** A `Result` cannot be silently ignored. The caller must either handle the error or explicitly propagate it. This eliminates the "forgot to check" class of bugs.
3. **Different code has different error needs.** Library code typically defines specific error types so callers can react precisely. Application code typically wants to bubble errors up to a central handler that logs and reports them. The same language supports both styles, with different libraries for each.

By the end of this module, students will:

- Distinguish recoverable errors (`Result`) from unrecoverable ones (`panic!`) and choose the right one for each situation.
- Use the `?` operator to propagate errors through functions cleanly.
- Define custom error types using enums and the standard `Error` trait.
- Use the `thiserror` crate to reduce boilerplate when defining library error types.
- Use the `anyhow` crate for application-level error handling where specific error types are not needed.
- Apply error handling best practices: choose the right error type, attach context, and propagate appropriately.

This module is short on new syntax but long on judgment. Most decisions in error handling are about which approach fits the situation, not about which feature to use.

> **A note on third-party libraries.** Sections E and F introduce `thiserror` and `anyhow`, which are not part of the standard library. They are external crates that the community has converged on as best practice. The standard library provides the `Error` trait and the `Result` type; `thiserror` and `anyhow` build on top of these to make common patterns less verbose. Knowing both the manual approach and the library approach is valuable: the manual approach shows what is happening, and the library approach is what you will use in real code.

---

## A. Unrecoverable Errors with `panic!`

Some errors cannot be handled meaningfully. They indicate bugs in the program rather than expected runtime conditions. For these, Rust uses panics.

### What `panic!` Does

The `panic!` macro stops the current thread, prints an error message, unwinds the stack (running destructors), and either terminates the program or returns control to a parent thread.

```rust
fn main() {
    panic!("something went very wrong");
}
```

Output:

```
thread 'main' panicked at src/main.rs:2:5:
something went very wrong
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The program exits with a non-zero status code.

### When the Standard Library Panics

Many common operations can panic when their preconditions are violated:

```rust
let v = vec![1, 2, 3];
let x = v[10];                    // panic: index out of bounds

let s = "12abc";
let n: i32 = s.parse().unwrap();  // panic: not a valid integer

let opt: Option<i32> = None;
let value = opt.unwrap();         // panic: called Option::unwrap on a None value
```

Each of these has a `Result`- or `Option`-returning alternative that lets you handle the failure: `Vec::get` returns `Option<&T>`, `parse` returns `Result<T, _>`, and `Option::unwrap_or` returns a default. The panicking versions are convenient when you have already proven the operation cannot fail.

### When to Panic

Use `panic!` (or `unwrap`, or `expect`) when:

- The condition indicates a programming bug, not a runtime condition.
- Continuing past the failure would put the program in an invalid state.
- You have invariants that the type system cannot express, and violating them is genuinely impossible if the code is correct.

Use `Result` instead when:

- The failure is something the caller might reasonably want to handle.
- The failure is data-dependent (bad input, missing file, network error).
- The function is part of a public API.

A common phrasing: panic on bugs, return `Result` on errors. A bug is something that should never happen if the code is correct. An error is something that can happen for reasons outside the program's control.

### `expect` for Better Panic Messages

`unwrap` panics with a generic message. `expect` lets you supply a more useful one:

```rust
let value = parse_config().expect("config file is required at startup");
```

If `parse_config` returns `Err`, the panic message includes "config file is required at startup," which is more informative than the default. Use `expect` over `unwrap` when you do choose to panic; the message helps debugging.

### Panic Behavior: Unwind vs. Abort

By default, a panic unwinds the stack: as it propagates up the call stack, it runs destructors for local variables, releasing resources cleanly. This is the default in development.

For production binaries that prioritize size and speed, you can configure panics to abort instead:

```toml
# Cargo.toml
[profile.release]
panic = "abort"
```

In abort mode, a panic immediately terminates the process without running destructors. This produces smaller binaries and faster compilation but means resources held by the panicking thread are not cleaned up gracefully.

For most applications, the default unwinding behavior is correct. The abort option is for embedded systems, kernels, or other environments where the runtime overhead of unwinding matters.

---

## B. Recoverable Errors with `Result<T, E>`

You met `Result` in Module 9. This section goes into more depth on how to use it in practice.

### The Type, Recapped

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

A function that can fail returns `Result<T, E>`. The caller cannot use the success value without first matching on the result.

### Reading and Writing a File

A canonical example: reading a file's contents into a string. Several things can go wrong (file does not exist, permission denied, not valid UTF-8), so the function returns `Result`:

```rust
use std::fs::File;
use std::io::Read;

fn read_file(path: &str) -> Result<String, std::io::Error> {
    let mut file = File::open(path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}
```

Two things to note:

- The return type is `Result<String, std::io::Error>`. Success produces a `String`; failure produces an `io::Error`.
- The `?` operator (covered in Section C) propagates errors up the call chain without explicit matches.

### Handling the Result at the Caller

The caller deals with the `Result`:

```rust
fn main() {
    match read_file("config.toml") {
        Ok(contents) => {
            println!("read {} bytes", contents.len());
        }
        Err(e) => {
            eprintln!("failed to read file: {e}");
        }
    }
}
```

Or, if there is a sensible default:

```rust
let contents = read_file("config.toml").unwrap_or_default();
```

`unwrap_or_default` returns the type's default value (an empty `String` here) on error.

Or, to convert one error into another:

```rust
let contents = read_file("config.toml")
    .map_err(|e| format!("config error: {e}"))?;
```

`map_err` transforms the error variant without affecting the success variant. This is useful when you want to wrap one error type in another or add context.

### Common Result Methods

`Result<T, E>` has many methods that handle common patterns:

```rust
let r: Result<i32, String> = Ok(5);

r.is_ok();                       // true
r.is_err();                      // false

r.unwrap();                      // 5; PANICS if Err
r.expect("must be Ok");          // 5; PANICS with message if Err
r.unwrap_or(0);                  // 5 if Ok, 0 if Err
r.unwrap_or_else(|e| e.len() as i32);   // computes default from error

r.map(|n| n * 2);                // Ok(10)
r.map_err(|e| e.to_uppercase()); // Ok(5), but error would be uppercased

r.ok();                          // Some(5); converts Result to Option
r.err();                         // None; converts to Option<E>

r.and_then(|n| Ok(n + 1));       // chains operations that might fail
```

These methods let you avoid explicit `match` for many common cases. `map` and `and_then` are particularly useful for chains of operations.

### The Difference from Exceptions

In a language with exceptions, errors are raised and caught through a separate mechanism. The function's return type does not document them. Catching is done with try/catch blocks.

In Rust, errors are values. The function's type signature documents them. Handling is done with the same constructs (`match`, `if let`, `?`) used for any other data.

The implications:

- A function that can fail says so in its type. You cannot accidentally ignore an error from a function that returns `Result`; the compiler enforces handling.
- Errors compose like values. You can `map` over them, collect them into vectors, or pass them to functions.
- Stack unwinding does not happen invisibly. Your code's control flow is visible in the source.

The cost: more verbose at the call site than exception-based error handling. The `?` operator (next section) reduces this verbosity dramatically.

---

## C. The `?` Operator and Error Propagation

Writing `match` for every fallible call is verbose. The `?` operator is a shortcut that handles the common case: "if this is Ok, give me the value; if this is Err, return it from the current function."

### The Basic Idea

Without `?`:

```rust
fn process_file(path: &str) -> Result<String, std::io::Error> {
    let mut file = match File::open(path) {
        Ok(f) => f,
        Err(e) => return Err(e),
    };

    let mut contents = String::new();
    match file.read_to_string(&mut contents) {
        Ok(_) => {}
        Err(e) => return Err(e),
    }

    Ok(contents)
}
```

With `?`:

```rust
fn process_file(path: &str) -> Result<String, std::io::Error> {
    let mut file = File::open(path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}
```

Each `?` does the same thing: if the result is `Ok`, unwrap it and continue; if it is `Err`, return it from the current function.

### What `?` Requires

The `?` operator only works in functions that return a compatible `Result`. Specifically:

- The function must return a `Result<T, E>` (or `Option<T>`, see below).
- The error type produced by `?` must be convertible to the function's error type.

Conversion happens automatically through the `From` trait (covered in Module 9 informally; covered in Module 13 fully). For example, if a function returns `Result<T, MyError>` and you call `something_that_returns_io_error()?`, Rust will try to convert the `io::Error` to `MyError` if there is a `From<io::Error> for MyError` implementation.

### Chaining

The most powerful use of `?` is chaining multiple fallible calls:

```rust
fn process_user_input(input: &str) -> Result<i32, MyError> {
    let trimmed = input.trim();
    let parsed: i32 = trimmed.parse()?;
    let validated = validate_in_range(parsed)?;
    let computed = compute_result(validated)?;
    Ok(computed)
}
```

Each step is a potential failure point. The function reads as a linear sequence of operations because `?` handles the "what if this fails" case. The happy path is what you see; the error paths are implied by the `?`.

### `?` with `Option`

The `?` operator also works with `Option`. In a function returning `Option<T>`, `?` says "if this is Some, unwrap; if this is None, return None from the current function":

```rust
fn first_word_length(text: Option<&str>) -> Option<usize> {
    let s = text?;
    let first = s.split_whitespace().next()?;
    Some(first.len())
}
```

The two `?` operators handle the two failure points. If `text` is `None`, the function returns `None` immediately. If `split_whitespace().next()` is `None` (empty string), the function returns `None` immediately.

You cannot mix `Result` and `Option` with `?` directly. If you need to interoperate, convert with `.ok()` (Result to Option) or `.ok_or(error)` (Option to Result).

### Why It Looks Strange

The `?` operator is unusual in mainstream languages. The closest analogue is the `try` keyword in Swift, which serves a similar purpose. The choice of `?` (a single character) is deliberate: it makes error propagation lightweight at the call site, so you are not penalized syntactically for handling errors carefully.

Once you have used it for a while, the `?` becomes nearly invisible. It is the visual equivalent of "this might fail, and that is fine."

### Where `?` Cannot Be Used

The `?` operator only works in functions that return `Result` or `Option`. In particular, you cannot use `?` directly in `main` if `main` returns nothing:

```rust
fn main() {
    let contents = read_file("config.toml")?;     // ERROR: main returns ()
}
```

The fix is to give `main` a `Result` return type:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let contents = read_file("config.toml")?;
    println!("{contents}");
    Ok(())
}
```

The `Box<dyn std::error::Error>` is a generic error type that accepts any `Error`. The `Ok(())` returns the unit value wrapped in `Ok` to indicate success. With this signature, `?` works in `main`.

---

## D. Defining Custom Error Types

For library code, returning `String` or `io::Error` from every function is rarely the right answer. Libraries typically define their own error types, usually as enums with one variant per kind of failure.

### A Simple Custom Error Enum

```rust
#[derive(Debug)]
enum ParseError {
    EmptyInput,
    NotANumber(String),
    OutOfRange { value: i64, min: i64, max: i64 },
}

fn parse_in_range(input: &str, min: i64, max: i64) -> Result<i64, ParseError> {
    if input.is_empty() {
        return Err(ParseError::EmptyInput);
    }

    let value: i64 = input.parse()
        .map_err(|_| ParseError::NotANumber(input.to_string()))?;

    if value < min || value > max {
        return Err(ParseError::OutOfRange { value, min, max });
    }

    Ok(value)
}
```

The `ParseError` enum has one variant per failure mode. Each variant carries the data that would help the caller understand what went wrong. The `#[derive(Debug)]` lets the error be printed with `{:?}`.

### The Standard `Error` Trait

The standard library has a trait called `Error` that error types are expected to implement. It provides:

- A `source()` method that returns the underlying cause, if any (for error chains).
- An obligation to implement `Display`, providing a user-facing message.

To make `ParseError` a proper error type, implement `Display` and `Error`:

```rust
use std::fmt;

#[derive(Debug)]
enum ParseError {
    EmptyInput,
    NotANumber(String),
    OutOfRange { value: i64, min: i64, max: i64 },
}

impl fmt::Display for ParseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ParseError::EmptyInput => write!(f, "input cannot be empty"),
            ParseError::NotANumber(s) => write!(f, "'{s}' is not a number"),
            ParseError::OutOfRange { value, min, max } => {
                write!(f, "{value} is out of range [{min}, {max}]")
            }
        }
    }
}

impl std::error::Error for ParseError {}
```

The `Display` implementation produces the user-facing message. The `Error` trait has no required methods (the defaults are fine for most cases), so the impl block is empty.

With these in place, `ParseError` is a fully-featured error type. It can be:

- Printed with `{}` (showing the user-facing message) or `{:?}` (showing the variant and data).
- Converted into a generic `Box<dyn Error>`.
- Used as the error type in any function that returns `Result`.

### The `From` Trait for Conversions

When a function calls another function with a different error type, you typically want to convert from the inner error to your error. The `From` trait makes this work with `?`:

```rust
impl From<std::io::Error> for ParseError {
    fn from(err: std::io::Error) -> ParseError {
        ParseError::NotANumber(err.to_string())
    }
}

fn read_and_parse(path: &str) -> Result<i64, ParseError> {
    let contents = std::fs::read_to_string(path)?;       // io::Error -> ParseError
    parse_in_range(&contents, 0, 100)
}
```

Because `From<io::Error>` is implemented for `ParseError`, the `?` operator automatically converts the `io::Error` from `read_to_string` into a `ParseError`. The function's return type is satisfied.

This is how error types compose. As long as you define `From` implementations between error types, `?` does the right thing.

### Why This Pattern?

Custom error types let library users:

- Match on specific error variants to handle them differently.
- Get rich, structured information about failures.
- Convert between error types as needed using `From`.

The cost is the boilerplate of implementing `Debug`, `Display`, `Error`, and `From` for every variant. The next two sections show libraries that reduce this boilerplate.

---

## E. Using `thiserror` for Ergonomic Error Types

The `thiserror` crate is a procedural macro that generates the boilerplate for custom error types. It is the de facto standard for defining error types in Rust libraries.

### Adding the Dependency

In `Cargo.toml`:

```toml
[dependencies]
thiserror = "1"
```

Then in your code:

```rust
use thiserror::Error;
```

### A Custom Error Type with `thiserror`

The same `ParseError` from the previous section, written with `thiserror`:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum ParseError {
    #[error("input cannot be empty")]
    EmptyInput,

    #[error("'{0}' is not a number")]
    NotANumber(String),

    #[error("{value} is out of range [{min}, {max}]")]
    OutOfRange { value: i64, min: i64, max: i64 },
}
```

The `#[derive(Error)]` attribute generates implementations of `Display` and `std::error::Error`. The `#[error("...")]` attribute on each variant supplies the format string for that variant's `Display` output.

The format strings can reference:

- Tuple variant fields by position: `{0}`, `{1}`, etc.
- Named fields by name: `{value}`, `{min}`, `{max}`.

This is significantly less boilerplate than the manual approach. The same trait implementations are produced; only the syntax for declaring them is shorter.

### Wrapping Other Errors

A common pattern is wrapping errors from other libraries:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum MyError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("invalid configuration: {0}")]
    Config(String),
}
```

Three things happen here:

- Each `#[from]` attribute generates a `From<T> for MyError` implementation, so `?` automatically converts the wrapped error to `MyError`.
- The `Display` formatter for the `Io` variant uses `{0}` to display the wrapped `io::Error`.
- The `source()` method returns the wrapped error, so error chains work correctly.

With this pattern, you can write:

```rust
fn load_config(path: &str) -> Result<Config, MyError> {
    let contents = std::fs::read_to_string(path)?;       // io::Error -> MyError
    let value: i32 = contents.trim().parse()?;            // ParseIntError -> MyError
    Ok(Config { value })
}
```

Both `?` operators work because `MyError` implements `From<io::Error>` and `From<ParseIntError>`, both generated by `thiserror`.

### When to Use `thiserror`

Use `thiserror` for library code where you want:

- Specific error variants that callers can match on.
- Clean conversion from other error types.
- Standard error trait implementations (`Display`, `Error`, `source`).

`thiserror` does not change the underlying model; it just removes the boilerplate. The code it generates is what you would write by hand, just compressed.

For application code (covered next), `thiserror` is sometimes overkill because you do not need fine-grained error types.

---

## F. Using `anyhow` for Application-Level Error Handling

Application code often does not need to distinguish between error types. When a database query fails, when a file is missing, or when input is malformed, the application typically wants to:

- Log the error.
- Return a user-friendly message.
- Exit or move on to the next request.

The application does not need to recover differently from each kind of error. It just needs a uniform way to handle "something went wrong." For this, the `anyhow` crate provides a generic error type.

### Adding the Dependency

In `Cargo.toml`:

```toml
[dependencies]
anyhow = "1"
```

### The `anyhow::Result` Type

`anyhow::Result<T>` is shorthand for `Result<T, anyhow::Error>`. The `anyhow::Error` type wraps any error implementing `std::error::Error`:

```rust
use anyhow::Result;

fn load_config(path: &str) -> Result<Config> {
    let contents = std::fs::read_to_string(path)?;       // io::Error -> anyhow::Error
    let value: i32 = contents.trim().parse()?;            // ParseIntError -> anyhow::Error
    Ok(Config { value })
}
```

The `?` operator works automatically because `anyhow::Error` accepts any error type. There is no need to define a custom error type or implement `From` for each underlying error.

### Adding Context

The most useful feature of `anyhow` is the `context` method, which attaches information to an error as it propagates:

```rust
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let contents = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config file: {path}"))?;

    let value: i32 = contents.trim().parse()
        .with_context(|| format!("config file does not contain a valid integer: {path}"))?;

    Ok(Config { value })
}
```

If `read_to_string` fails, the error message includes both the underlying I/O error and the context "failed to read config file: /path/to/config.toml". This produces multi-line error reports that are much easier to debug than a bare error message.

When printed, the error chain looks like:

```
Error: failed to read config file: config.toml

Caused by:
    No such file or directory (os error 2)
```

The original error and the added context are both visible. For nested operations, you can have many layers of context:

```
Error: failed to start server

Caused by:
    0: failed to load configuration
    1: failed to read config file: /etc/myapp/config.toml
    2: No such file or directory (os error 2)
```

Each layer of context tells the reader something about why the operation was happening. This is invaluable for debugging production issues.

### When to Use `anyhow`

Use `anyhow` for:

- Application-level code (the top of the call stack).
- Quick prototypes where defining error types is overkill.
- Functions that combine errors from many sources and do not need to differentiate them.

Avoid `anyhow` for:

- Library code where callers might want to match on specific error types.
- Functions where the caller is expected to recover from specific failure modes.

The convention is: libraries return specific error types (often via `thiserror`), and applications convert those errors to `anyhow::Error` as they bubble up.

### `thiserror` and `anyhow` Together

The two libraries are complementary, not competing. A typical project structure:

```rust
// Library crate uses thiserror for specific types
use thiserror::Error;

#[derive(Debug, Error)]
pub enum LibError {
    #[error("not found: {0}")]
    NotFound(String),
    #[error("invalid input: {0}")]
    InvalidInput(String),
}

// Application crate uses anyhow for uniform handling
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let data = lib::fetch("user_42")
        .context("failed to fetch user data")?;     // LibError -> anyhow::Error
    println!("{data}");
    Ok(())
}
```

The library exposes specific errors so its users can match on them. The application calls the library and converts errors to `anyhow::Error` for uniform handling at the top level.

---

## G. Error Handling Best Practices

This section is judgment-heavy. The right approach depends on what you are building, and the wrong approach is rarely catastrophic but often tedious.

### Choose the Right Granularity

Three rough categories of error handling:

**Library code.** Define specific error types using `thiserror`. Each variant should represent a meaningful failure mode that callers might react to. Aim for small, focused error enums (3-10 variants) rather than monolithic ones.

**Application code.** Use `anyhow` for most of the call stack. Add context with `.context()` or `.with_context()` as errors propagate. Convert library errors to `anyhow::Error` automatically through `?`.

**Quick scripts and prototypes.** Use `anyhow` everywhere, or even just `Result<_, Box<dyn Error>>`. Optimize for readability over correctness; you can always refine later.

### Always Add Context

A bare error like "No such file or directory" is nearly useless in a production log. Adding context (with `anyhow::Context` or by wrapping in a custom error variant) tells the reader what was being attempted:

```rust
let config = load_config(&args.config_path)
    .with_context(|| format!("failed to start with config file {}", args.config_path))?;
```

The context line answers "what was the program trying to do?" The underlying error answers "why did it fail?" Both are needed for diagnosis.

### Be Honest About Panics

Use `.unwrap()` and `.expect()` only when:

- You can prove the operation cannot fail (a constant value, a known-good input).
- A failure indicates a bug in your code that you cannot recover from.

A common antipattern is using `.unwrap()` on user input or external data. This turns runtime errors into program crashes:

```rust
let port: u16 = std::env::var("PORT").unwrap().parse().unwrap();    // crashes on bad input
```

The right way:

```rust
let port: u16 = std::env::var("PORT")
    .context("PORT environment variable is required")?
    .parse()
    .context("PORT must be a valid u16")?;
```

This produces useful error messages instead of panics. The user sees what went wrong and can fix it.

### Distinguish Errors from Panics

Panic is for bugs. `Result` is for errors. The distinction matters because:

- Panics in production are emergencies that should be investigated.
- Errors are normal occurrences that should be logged and handled.

If you find yourself wrapping `Result` returns in `.unwrap()` everywhere, you are using `Result` as a panic mechanism. Either change the function to return what it actually computes (avoiding the `Result`) or handle the errors properly.

### Do Not Stringify Too Early

A common mistake is converting errors to strings as soon as they appear:

```rust
fn load() -> Result<Config, String> {        // bad: stringified too early
    let contents = std::fs::read_to_string("config.toml")
        .map_err(|e| e.to_string())?;
    // ...
}
```

This loses information. The original `io::Error` had a kind, an OS error code, and structural information. The string version has none of that.

Better:

```rust
fn load() -> Result<Config, anyhow::Error> {  // or thiserror enum
    let contents = std::fs::read_to_string("config.toml")
        .context("failed to read config")?;
    // ...
}
```

Now the original error is preserved (in `anyhow`'s error chain or in a wrapper variant). Callers can introspect it if they need to. The string representation is generated only when displayed.

### Avoid Catch-All Variants

A common antipattern in custom error types:

```rust
#[derive(Debug, Error)]
enum MyError {
    #[error("I/O error: {0}")]
    Io(#[from] io::Error),

    #[error("other: {0}")]
    Other(String),                    // catch-all for everything else
}
```

The `Other(String)` variant tempts you to stuff random strings into it whenever you have an error you do not want to handle properly. Over time, the variant becomes a dumping ground for unstructured errors.

Better: define specific variants for each failure mode. If you genuinely have many sources of error, use `anyhow::Error` (which is essentially "any error" but preserves structure) instead of a string.

### Test Error Paths

Error handling is code. Code without tests is unverified. Make sure your tests cover the cases where things fail, not just the happy path.

```rust
#[test]
fn returns_error_for_invalid_input() {
    let result = parse_in_range("abc", 0, 100);
    assert!(matches!(result, Err(ParseError::NotANumber(_))));
}

#[test]
fn returns_error_for_out_of_range() {
    let result = parse_in_range("200", 0, 100);
    assert!(matches!(result, Err(ParseError::OutOfRange { .. })));
}
```

The `matches!` macro is useful for testing that an error is of a specific variant without writing a full match. Both happy and error paths are exercised.

### Summary of Best Practices

A short list:

1. **Use `Result` for things that can fail; use `panic!` for bugs.**
2. **Define specific error types in libraries (`thiserror`); use generic ones in applications (`anyhow`).**
3. **Add context to errors as they propagate.**
4. **Avoid stringifying errors prematurely.**
5. **Avoid catch-all error variants.**
6. **Test error paths, not just happy paths.**
7. **Reserve `unwrap` and `expect` for cases where failure is impossible by construction.**

These are conventions, not rules. Most projects converge on something like this pattern, with adaptations for their specific needs.

---

## Module Summary

- Errors are values in Rust, not control flow. `Result<T, E>` represents recoverable failures; `panic!` represents bugs.
- The `?` operator propagates errors through a function with minimal syntactic overhead.
- Custom error types are usually enums with one variant per failure mode. Implementing `Display` and `Error` makes them usable everywhere errors are expected.
- The `thiserror` crate generates the boilerplate for custom error types. It is standard in library code.
- The `anyhow` crate provides a generic error type that wraps any error. It is standard in application code.
- Best practices: choose the right granularity (specific in libraries, generic in apps), add context, avoid stringifying errors, and test error paths.

## Discussion Questions

1. Rust uses `Result` instead of exceptions. Identify a specific concrete benefit of values-as-errors compared to exceptions, and a specific concrete cost. On balance, is the trade-off worth it for the kinds of programs you write?
2. Libraries typically use `thiserror` and applications typically use `anyhow`. Why does this division of labor make sense? What goes wrong if a library uses `anyhow` everywhere or an application defines an exhaustive error enum?
3. The `?` operator was added to Rust in 2016 (replacing an earlier `try!` macro). It is now one of the most-used operators in the language. What does this tell you about how design choices about ergonomics affect adoption of safety features?

#