# Module 10: Error Handling
## Lab 7 -- Working with panic!, Result, the ? Operator, and Custom Error Types

> **Course:** Mastering Rust
> **Module:** 10 - Error Handling
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 10: unrecoverable errors with `panic!`, recoverable errors with `Result<T, E>`, the `?` operator for error propagation, defining custom error types manually, using the `thiserror` crate to reduce boilerplate, using the `anyhow` crate for application-level error handling, and the conventions that Rust developers follow when designing error-handling code.

You will build a single Cargo project called `config_parser` that loads, validates, and processes configuration data. Configuration parsing is a natural fit for error handling: there are many ways the input can be invalid, and each kind of failure carries different information. The lab walks you through writing the same logic three times: with manual error types, with `thiserror`, and with `anyhow`. By doing each, you will see what each approach optimizes for.

The focus is on judgment: when to panic versus return a `Result`, when to define a specific error type versus use a generic one, and when each library makes sense. Error handling is more about design choices than syntax; the lab guides you toward idiomatic Rust and asks you to compare alternatives.

By the end of this lab you will be able to:

- Distinguish unrecoverable errors (`panic!`) from recoverable errors (`Result`) and choose appropriately.
- Use `Result<T, E>` to represent operations that can fail and the methods that work with it.
- Use the `?` operator to propagate errors through function chains.
- Define custom error enums manually with `Display`, `Error`, and `From` implementations.
- Use the `thiserror` crate to generate the boilerplate for error types automatically.
- Use the `anyhow` crate for application-level error handling with context.
- Apply best practices: choose granularity, add context, avoid stringification.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** This lab uses two third-party crates, `thiserror` and `anyhow`. Both are widely used in production Rust. They will be added through `Cargo.toml`, and Cargo will download them on the first build. Make sure you have an internet connection when you run the build for Exercises 5 and 6.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new config_parser
cd config_parser
code .
```

Create a `lab7-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Unrecoverable Errors with panic!

**Estimated time:** 10 minutes
**Topics covered:** the `panic!` macro, when the standard library panics, `unwrap` and `expect`, panic vs. error

### Context

Some failures are bugs rather than conditions to handle. A function called with invalid arguments that should never have been given those arguments is a programming error; the program cannot meaningfully continue. For these cases, Rust uses `panic!`. This exercise establishes when panicking is the right choice.

### Task 1.1 -- Calling panic! directly

Replace the contents of `src/main.rs` with:

```rust
fn main() {
    println!("about to panic");
    panic!("something went very wrong");
    println!("this line never runs");
}
```

Run the program:

```bash
cargo run
```

Expected output (truncated):

```
about to panic
thread 'main' panicked at src/main.rs:3:5:
something went very wrong
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The program exits with a non-zero status code. The line after `panic!` never runs.

The panic message includes the file and line, the message you provided, and a hint to set `RUST_BACKTRACE=1` for more detail. Try it:

```bash
RUST_BACKTRACE=1 cargo run
```

The output now includes a stack trace showing every function call leading to the panic. This is invaluable for debugging.

### Task 1.2 -- Standard library panics

Many standard operations panic on invalid input. Replace `main` with:

```rust
fn main() {
    let v = vec![1, 2, 3];
    let x = v[10];        // panic: index out of bounds
    println!("{x}");
}
```

Run the program. Expected output:

```
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 10', src/main.rs:3:13
```

Vector indexing panics if the index is out of bounds. The program cannot meaningfully continue (there is no element at that position), so it stops.

The standard library has a non-panicking alternative: `Vec::get` returns `Option<&T>`:

```rust
fn main() {
    let v = vec![1, 2, 3];

    match v.get(10) {
        Some(value) => println!("got {value}"),
        None => println!("no element at index 10"),
    }
}
```

Run this version. Expected output:

```
no element at index 10
```

The `get` method handles the case explicitly. It is the right choice when the index might be out of bounds for legitimate reasons.

### Task 1.3 -- unwrap and expect

`Option` and `Result` have an `unwrap` method that panics if the value is `None` or `Err`:

```rust
fn main() {
    let s = "42";
    let parsed: i32 = s.parse().unwrap();
    println!("got {parsed}");

    let bad = "abc";
    let panics: i32 = bad.parse().unwrap();    // panic
    println!("never runs");
}
```

Run the program. The first parse succeeds; the second panics with a generic message about the failed conversion.

`expect` is similar but lets you provide a custom message:

```rust
fn main() {
    let bad = "abc";
    let panics: i32 = bad.parse().expect("the input must be a valid integer");
    println!("never runs");
}
```

Run the program. The panic message now includes the custom text:

```
thread 'main' panicked at 'the input must be a valid integer: ParseIntError { kind: InvalidDigit }', ...
```

The `expect` form is preferred over `unwrap` when you do choose to panic. The custom message helps anyone debugging the failure understand the developer's intent.

### Task 1.4 -- When to panic

Different operations have different appropriate handling. Some examples:

```rust
fn main() {
    // Programming bug: indexing a known empty array
    let arr: [i32; 0] = [];
    // arr[0]                          // panic: this would be a bug

    // Reasonable to unwrap: we KNOW this parses (constant)
    let pi: f64 = "3.14159".parse().unwrap();
    println!("pi = {pi}");

    // NOT reasonable to unwrap: user input might fail
    // let user_input = read_input();
    // let n: i32 = user_input.parse().unwrap();    // BAD: should be handled

    // Better: handle it
    let user_input = "42";       // pretend this came from a user
    match user_input.parse::<i32>() {
        Ok(n) => println!("user typed {n}"),
        Err(e) => println!("invalid input: {e}"),
    }
}
```

Run the program. The good cases work; the bad case (unwrapping user input) is commented out because it would represent a design flaw.

The rule of thumb:

- **Panic** when the failure indicates a bug that the program cannot meaningfully handle.
- **Use `unwrap` or `expect`** when you can prove the operation cannot fail (parsing a constant, accessing a known-valid index).
- **Return `Result`** when the failure is something the caller might handle (parsing user input, reading files, network operations).

Reaching for `unwrap` on something that could legitimately fail is a code smell. The next exercises show what to do instead.

### Checkpoints

1. The panic in Task 1.2 occurred on `v[10]`. The program could have returned a default value (like 0) or wrapped around (modulo length). Why does Rust panic instead? What kind of bug is the language trying to prevent?
2. `unwrap` and `expect` both panic on failure. The difference is just the message. Why is `expect` preferred over `unwrap` when you do choose to panic?
3. The "rule of thumb" lists three cases for choosing between panic and Result. Identify a specific scenario in real code (file handling, network requests, parsing, etc.) for each of the three cases.

---

## Exercise 2 -- Result and the Match Pattern

**Estimated time:** 10-15 minutes
**Topics covered:** `Result<T, E>` mechanics, matching on Result, common methods

### Context

`Result<T, E>` represents an operation that might succeed (with a `T`) or fail (with an `E`). Unlike `panic!`, the failure can be handled. The compiler enforces handling: you cannot use the success value without first checking which variant the result is.

### Task 2.1 -- A function returning Result

Replace `src/main.rs` with:

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    let result1 = divide(10.0, 2.0);
    let result2 = divide(10.0, 0.0);

    match result1 {
        Ok(value) => println!("result1: {value}"),
        Err(e) => println!("result1 error: {e}"),
    }

    match result2 {
        Ok(value) => println!("result2: {value}"),
        Err(e) => println!("result2 error: {e}"),
    }
}
```

Run the program. Expected output:

```
result1: 5
result2 error: division by zero
```

The function returns `Result<f64, String>`. The caller has to handle both variants because there is no way to extract the `f64` without a match (or the equivalent).

### Task 2.2 -- Common Result methods

Add this code to `main` (after the matches above):

```rust
fn main() {
    let r1 = divide(10.0, 2.0);
    let r2 = divide(10.0, 0.0);

    // Direct queries about variant:
    println!("r1 is_ok: {}", r1.is_ok());
    println!("r2 is_err: {}", r2.is_err());

    // Unwrap with default:
    let v1 = r1.clone().unwrap_or(0.0);
    let v2 = r2.clone().unwrap_or(0.0);
    println!("v1: {v1}, v2: {v2}");

    // Computed default:
    let v3 = divide(10.0, 0.0).unwrap_or_else(|e| {
        eprintln!("computing default after error: {e}");
        -1.0
    });
    println!("v3: {v3}");

    // Transform the error:
    let r3: Result<f64, String> = divide(1.0, 0.0).map_err(|e| format!("custom: {e}"));
    println!("r3: {r3:?}");

    // Transform the success:
    let r4: Result<i32, String> = divide(10.0, 2.0).map(|v| v as i32);
    println!("r4: {r4:?}");
}
```

Replace the previous `main` content with this and recompile. Expected output:

```
r1 is_ok: true
r2 is_err: true
v1: 5, v2: 0
computing default after error: division by zero
v3: -1
r3: Err("custom: division by zero")
r4: Ok(5)
```

Several methods are useful:

- `is_ok` / `is_err` return `bool`.
- `unwrap_or(default)` returns the success value or the default.
- `unwrap_or_else(closure)` computes the default from the error.
- `map(f)` transforms the success value, leaving Err unchanged.
- `map_err(f)` transforms the error, leaving Ok unchanged.

These methods let you avoid explicit `match` for common patterns. The closure-based ones (`unwrap_or_else`, `map_err`) accept any closure that fits the signature.

### Task 2.3 -- Chaining with and_then

`and_then` chains operations that each return `Result`. If any step fails, the chain stops:

```rust
fn parse_int(s: &str) -> Result<i32, String> {
    s.parse::<i32>().map_err(|e| format!("parse error: {e}"))
}

fn check_positive(n: i32) -> Result<i32, String> {
    if n > 0 {
        Ok(n)
    } else {
        Err(format!("expected positive, got {n}"))
    }
}

fn double(n: i32) -> Result<i32, String> {
    n.checked_mul(2)
        .ok_or_else(|| format!("overflow doubling {n}"))
}

fn main() {
    let inputs = vec!["42", "0", "abc", "999999999999"];

    for input in inputs {
        let result = parse_int(input)
            .and_then(check_positive)
            .and_then(double);

        match result {
            Ok(value) => println!("{input}: ok {value}"),
            Err(e) => println!("{input}: error: {e}"),
        }
    }
}
```

Run the program. Expected output:

```
42: ok 84
0: error: expected positive, got 0
abc: error: parse error: invalid digit found in string
999999999999: error: parse error: number too large to fit in target type
```

`and_then` is the chaining operator: "if Ok, pass the value to this next function and use its result; if Err, propagate the error." Each step in the chain can fail; the first failure short-circuits the rest.

This is functional-style error handling. It works, but for chains longer than two or three steps, the `?` operator (next exercise) is more readable.

### Task 2.4 -- A practical chain with collect

Iterators of `Result` can be collected into a `Result<Vec<T>, E>`:

```rust
fn main() {
    let inputs = vec!["1", "2", "3", "4", "5"];

    let parsed: Result<Vec<i32>, _> = inputs.iter().map(|s| s.parse::<i32>()).collect();

    match parsed {
        Ok(values) => println!("all parsed: {values:?}"),
        Err(e) => println!("error: {e}"),
    }

    let with_bad = vec!["1", "2", "abc", "4"];
    let parsed: Result<Vec<i32>, _> = with_bad.iter().map(|s| s.parse::<i32>()).collect();

    match parsed {
        Ok(values) => println!("all parsed: {values:?}"),
        Err(e) => println!("error: {e}"),
    }
}
```

Run the program. Expected output:

```
all parsed: [1, 2, 3, 4, 5]
error: invalid digit found in string
```

`collect` on an iterator of `Result<T, E>` produces a `Result<Vec<T>, E>`. If all elements are `Ok`, you get the vector. If any is `Err`, you get the first error and the rest are not processed.

This is the canonical way to parse a list of values: split, parse each, collect with error short-circuiting.

### Checkpoints

1. In Task 2.2, several methods (`unwrap_or`, `unwrap_or_else`, `map_err`) avoid explicit match. Why might explicit match still be the right choice in some cases?
2. The `and_then` chaining in Task 2.3 short-circuits on the first error. What practical benefit does this short-circuiting provide compared to running all steps and collecting the errors?
3. In Task 2.4, `collect` returned the *first* error, not all errors. When would you want to collect all errors instead? What method or pattern would you use?

---

## Exercise 3 -- The ? Operator

**Estimated time:** 15 minutes
**Topics covered:** `?` for error propagation, requirements for `?`, `?` with Option

### Context

Writing `match` for every fallible call is verbose. The `?` operator is a shortcut: if the result is `Ok`, unwrap and continue; if it is `Err`, return the error from the current function. This makes error-propagating code read almost like ordinary linear code.

### Task 3.1 -- Without the ? operator

Replace `src/main.rs` with this verbose version:

```rust
fn parse_age(input: &str) -> Result<u32, String> {
    let trimmed = input.trim();

    let parsed: u32 = match trimmed.parse() {
        Ok(n) => n,
        Err(e) => return Err(format!("parse error: {e}")),
    };

    if parsed > 150 {
        return Err(format!("age {parsed} is unrealistic"));
    }

    Ok(parsed)
}

fn main() {
    let inputs = vec!["42", "  30  ", "abc", "200"];

    for input in inputs {
        match parse_age(input) {
            Ok(age) => println!("'{input}' -> age {age}"),
            Err(e) => println!("'{input}' -> error: {e}"),
        }
    }
}
```

Run the program. Expected output:

```
'42' -> age 42
'  30  ' -> age 30
'abc' -> error: parse error: invalid digit found in string
'200' -> error: age 200 is unrealistic
```

The function works but the match is intrusive. Every fallible call needs the same boilerplate.

### Task 3.2 -- The ? operator

Replace `parse_age` with the cleaner version:

```rust
fn parse_age(input: &str) -> Result<u32, String> {
    let trimmed = input.trim();

    let parsed: u32 = trimmed.parse().map_err(|e: std::num::ParseIntError| format!("parse error: {e}"))?;

    if parsed > 150 {
        return Err(format!("age {parsed} is unrealistic"));
    }

    Ok(parsed)
}
```

Run the program. The output is identical. The `?` after `map_err(...)` does the same thing as the explicit match: unwrap on Ok, return the error on Err.

The single line `let parsed: u32 = trimmed.parse().map_err(...)?;` is shorter, reads linearly, and makes the success path obvious. The error path is implied by the `?`.

### Task 3.3 -- Chain with ?

The real value of `?` shows up in chains. Add a more complex function:

```rust
fn parse_user_data(input: &str) -> Result<(String, u32), String> {
    let parts: Vec<&str> = input.split(',').collect();

    if parts.len() != 2 {
        return Err(format!("expected 'name,age', got '{input}'"));
    }

    let name = parts[0].trim().to_string();
    if name.is_empty() {
        return Err(String::from("name cannot be empty"));
    }

    let age = parse_age(parts[1])?;        // propagate the error from parse_age

    Ok((name, age))
}

fn main() {
    let inputs = vec![
        "Alice,42",
        "Bob, 30",
        "Carol,abc",
        ",100",
        "Dave,200",
        "Eve",
    ];

    for input in inputs {
        match parse_user_data(input) {
            Ok((name, age)) => println!("'{input}' -> {name} is {age}"),
            Err(e) => println!("'{input}' -> error: {e}"),
        }
    }
}
```

Run the program. Expected output:

```
'Alice,42' -> Alice is 42
'Bob, 30' -> Bob is 30
'Carol,abc' -> error: parse error: invalid digit found in string
',100' -> error: name cannot be empty
'Dave,200' -> error: age 200 is unrealistic
'Eve' -> error: expected 'name,age', got 'Eve'
```

The `?` after `parse_age(parts[1])` propagates the error directly. If parsing the age fails, the function returns the error without further code. The success path continues.

For functions that call several fallible operations, the `?` operator makes the happy path clear. The error paths are implied.

### Task 3.4 -- ? on different error types: from std::io

Add a function that reads from a file:

```rust
use std::fs;

fn read_config(path: &str) -> Result<String, std::io::Error> {
    let contents = fs::read_to_string(path)?;
    Ok(contents)
}

fn main() {
    match read_config("nonexistent.txt") {
        Ok(s) => println!("contents: {s}"),
        Err(e) => println!("error: {e}"),
    }
}
```

Run the program. Expected output:

```
error: No such file or directory (os error 2)
```

The `?` works because the function returns `Result<_, std::io::Error>` and `read_to_string` returns `Result<_, std::io::Error>`. The error types match.

But what if the function does multiple things, each with different error types? The next subtask shows the issue.

### Task 3.5 -- The ? operator and From

Add a function that combines I/O and parsing:

```rust
fn read_config_age(path: &str) -> Result<u32, ???> {
    let contents = fs::read_to_string(path)?;        // returns std::io::Error
    let age: u32 = contents.trim().parse()?;          // returns std::num::ParseIntError
    Ok(age)
}
```

The function has two `?` operators that produce two different error types. What should the function's error type be?

Option 1: Use `Box<dyn std::error::Error>`, which accepts any error implementing the `Error` trait:

```rust
fn read_config_age(path: &str) -> Result<u32, Box<dyn std::error::Error>> {
    let contents = fs::read_to_string(path)?;
    let age: u32 = contents.trim().parse()?;
    Ok(age)
}

fn main() {
    match read_config_age("config.txt") {
        Ok(age) => println!("age from config: {age}"),
        Err(e) => println!("error: {e}"),
    }
}
```

Run the program. The function compiles. The `?` works because both `std::io::Error` and `std::num::ParseIntError` can be converted to `Box<dyn std::error::Error>` via the `From` trait.

Option 2 (covered in Exercise 4): define a custom error type and implement `From` from each underlying error type.

### Task 3.6 -- ? in main

A program where `main` itself uses `?` needs to return a `Result`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let age = read_config_age("config.txt")?;
    println!("age: {age}");
    Ok(())
}
```

Try this. The function will fail (assuming `config.txt` does not exist), and the program will exit with the error message. The `?` in `main` returns the error, which Rust prints automatically.

This is a clean way to write small programs where errors should just be printed and the program should exit. For larger programs, you usually want explicit error handling.

### Task 3.7 -- ? with Option

The `?` operator works on `Option` too. In a function returning `Option`, `?` says "if Some, unwrap; if None, return None":

```rust
fn first_word_length(text: Option<&str>) -> Option<usize> {
    let s = text?;
    let first = s.split_whitespace().next()?;
    Some(first.len())
}

fn main() {
    println!("{:?}", first_word_length(Some("hello world")));    // Some(5)
    println!("{:?}", first_word_length(None));                    // None
    println!("{:?}", first_word_length(Some("")));                 // None (no words)
}
```

Run the program. Expected output:

```
Some(5)
None
None
```

You cannot mix `Result` and `Option` with `?` directly. If you need to interoperate, use `.ok()` (Result to Option) or `.ok_or(error)` (Option to Result).

### Checkpoints

1. The `?` operator in Task 3.2 replaced six lines of explicit match handling with one line. Beyond brevity, what does this give you in terms of readability?
2. Task 3.5 used `Box<dyn std::error::Error>` to handle multiple error types. What is the trade-off compared to defining a single concrete error type?
3. The `?` operator does not work outside functions that return `Result` or `Option`. Why is this restriction necessary? What would go wrong if `?` could be used anywhere?

---

## Exercise 4 -- Defining Custom Error Types

**Estimated time:** 15 minutes
**Topics covered:** error enums, `Display` and `Error` traits, `From` for conversions

### Context

For a real application, returning `String` errors loses information. A custom error type with one variant per failure mode lets callers handle each case appropriately. This exercise walks through implementing one manually, before showing the library shortcut in Exercise 5.

### Task 4.1 -- A simple error enum

Replace `src/main.rs` with:

```rust
use std::fmt;

#[derive(Debug)]
enum ConfigError {
    EmptyFile,
    InvalidLine { line_number: usize, content: String },
    MissingField(String),
    InvalidValue { field: String, value: String },
}
```

This declares an error type with four variants. Each variant carries the data relevant to that kind of failure. The `#[derive(Debug)]` lets us print errors with `{:?}`.

### Task 4.2 -- Implementing Display

Errors should implement `Display` for user-facing messages. Add this:

```rust
impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::EmptyFile => {
                write!(f, "configuration file is empty")
            }
            ConfigError::InvalidLine { line_number, content } => {
                write!(f, "line {line_number}: invalid format '{content}'")
            }
            ConfigError::MissingField(field) => {
                write!(f, "missing required field: {field}")
            }
            ConfigError::InvalidValue { field, value } => {
                write!(f, "invalid value '{value}' for field '{field}'")
            }
        }
    }
}
```

The `Display` implementation provides a user-facing string for each variant. The `match` extracts the variant's data and `write!` produces the formatted message.

### Task 4.3 -- Implementing the Error trait

Add the standard `Error` trait implementation:

```rust
impl std::error::Error for ConfigError {}
```

That is the entire implementation. The trait has no required methods (the defaults are fine for most cases). Implementing it formally marks `ConfigError` as a real error type, which lets it work with code that expects any `Error`-implementing type (like `Box<dyn Error>`).

### Task 4.4 -- Using the custom error

Add a parsing function:

```rust
struct Config {
    name: String,
    port: u16,
    debug: bool,
}

fn parse_config(text: &str) -> Result<Config, ConfigError> {
    if text.trim().is_empty() {
        return Err(ConfigError::EmptyFile);
    }

    let mut name = None;
    let mut port = None;
    let mut debug = None;

    for (i, line) in text.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }

        let parts: Vec<&str> = line.splitn(2, '=').collect();
        if parts.len() != 2 {
            return Err(ConfigError::InvalidLine {
                line_number: i + 1,
                content: line.to_string(),
            });
        }

        let key = parts[0].trim();
        let value = parts[1].trim();

        match key {
            "name" => name = Some(value.to_string()),
            "port" => {
                let p: u16 = value.parse().map_err(|_| ConfigError::InvalidValue {
                    field: String::from("port"),
                    value: value.to_string(),
                })?;
                port = Some(p);
            }
            "debug" => match value {
                "true" => debug = Some(true),
                "false" => debug = Some(false),
                _ => {
                    return Err(ConfigError::InvalidValue {
                        field: String::from("debug"),
                        value: value.to_string(),
                    });
                }
            },
            _ => {
                return Err(ConfigError::InvalidLine {
                    line_number: i + 1,
                    content: line.to_string(),
                });
            }
        }
    }

    Ok(Config {
        name: name.ok_or_else(|| ConfigError::MissingField(String::from("name")))?,
        port: port.ok_or_else(|| ConfigError::MissingField(String::from("port")))?,
        debug: debug.ok_or_else(|| ConfigError::MissingField(String::from("debug")))?,
    })
}

fn main() {
    let inputs = vec![
        "",
        "name = web-server\nport = 8080\ndebug = true",
        "name = web-server\nport = 8080",
        "name = web-server\nport = abc\ndebug = true",
        "this is not a config",
    ];

    for input in inputs {
        match parse_config(input) {
            Ok(c) => println!(
                "OK: name={}, port={}, debug={}",
                c.name, c.port, c.debug
            ),
            Err(e) => println!("ERROR: {e}"),
        }
    }
}
```

Run the program. Expected output:

```
ERROR: configuration file is empty
OK: name=web-server, port=8080, debug=true
ERROR: missing required field: debug
ERROR: invalid value 'abc' for field 'port'
ERROR: line 1: invalid format 'this is not a config'
```

Note how each error type produces a meaningful message. A caller can also pattern-match on the variant to handle each kind specifically:

```rust
match parse_config(input) {
    Ok(c) => println!("ok: {}", c.name),
    Err(ConfigError::EmptyFile) => println!("file was empty; using defaults"),
    Err(ConfigError::MissingField(f)) => println!("you forgot the {f} field"),
    Err(e) => println!("other error: {e}"),
}
```

This is the value of structured error types: callers can choose which errors to handle specifically and which to handle generically.

### Task 4.5 -- The From trait for conversion

Real applications often combine multiple error types. The `From` trait makes `?` automatically convert. Add a wrapping function:

```rust
use std::fs;

#[derive(Debug)]
enum AppError {
    IoError(std::io::Error),
    ConfigError(ConfigError),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::IoError(e) => write!(f, "I/O error: {e}"),
            AppError::ConfigError(e) => write!(f, "config error: {e}"),
        }
    }
}

impl std::error::Error for AppError {}

impl From<std::io::Error> for AppError {
    fn from(err: std::io::Error) -> AppError {
        AppError::IoError(err)
    }
}

impl From<ConfigError> for AppError {
    fn from(err: ConfigError) -> AppError {
        AppError::ConfigError(err)
    }
}

fn read_and_parse(path: &str) -> Result<Config, AppError> {
    let contents = fs::read_to_string(path)?;        // io::Error -> AppError via From
    let config = parse_config(&contents)?;            // ConfigError -> AppError via From
    Ok(config)
}

fn main() {
    match read_and_parse("nonexistent.toml") {
        Ok(c) => println!("ok: {}", c.name),
        Err(e) => println!("error: {e}"),
    }
}
```

Run the program. Expected output:

```
error: I/O error: No such file or directory (os error 2)
```

The `?` after `read_to_string(path)` automatically converts the `std::io::Error` into an `AppError::IoError` via the `From<std::io::Error> for AppError` implementation. Similarly for `ConfigError`. The function reads cleanly because the conversions are automatic.

This pattern (a wrapper enum with `From` implementations for each underlying error) is one of the most common in Rust. The next exercise shows how to generate the boilerplate automatically.

### Checkpoints

1. The `ConfigError` enum has variants with different shapes (one with a string field, one with a struct-style field). Why does Rust let variants have different shapes rather than requiring a uniform structure?
2. The `Error` trait implementation in Task 4.3 was a single empty line: `impl std::error::Error for ConfigError {}`. The trait has no required methods. Why does the trait exist at all if it has no requirements?
3. Task 4.5 wrapped `std::io::Error` in `AppError::IoError` rather than just storing a `String` description. What information is preserved by wrapping that would be lost by stringifying?

---

## Exercise 5 -- Using thiserror

**Estimated time:** 10 minutes
**Topics covered:** the `thiserror` crate, derive macros for error types, `#[from]`

### Context

The manual error type from Exercise 4 required a lot of boilerplate: `Display` implementation with a match for every variant, `Error` trait, `From` implementations. The `thiserror` crate generates all of this from a single derive. This exercise rewrites the same logic using the crate.

### Task 5.1 -- Adding thiserror as a dependency

Open `Cargo.toml` in the project root. Add `thiserror` to the dependencies:

```toml
[dependencies]
thiserror = "1"
```

Save the file. The next `cargo build` will download the crate.

### Task 5.2 -- Rewriting ConfigError with thiserror

Replace the `ConfigError` definition (and its `Display` and `Error` impls) with this single block:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("configuration file is empty")]
    EmptyFile,

    #[error("line {line_number}: invalid format '{content}'")]
    InvalidLine { line_number: usize, content: String },

    #[error("missing required field: {0}")]
    MissingField(String),

    #[error("invalid value '{value}' for field '{field}'")]
    InvalidValue { field: String, value: String },
}
```

Remove the manual `impl Display` and `impl Error` blocks for `ConfigError`. Also remove the `use std::fmt;` line if it is no longer needed.

The `#[derive(Error)]` and `#[error("...")]` attributes do all the work:

- `#[derive(Error)]` generates the `std::error::Error` implementation.
- `#[error("...")]` generates the `Display` implementation. The format string can reference fields by name (`{field}`) or by index (`{0}`).

Run `cargo run`. The program should compile and produce the same output as before.

### Task 5.3 -- Rewriting AppError with thiserror

Replace `AppError` (and its `Display`, `Error`, and `From` implementations) with:

```rust
#[derive(Debug, Error)]
enum AppError {
    #[error("I/O error: {0}")]
    IoError(#[from] std::io::Error),

    #[error("config error: {0}")]
    ConfigError(#[from] ConfigError),
}
```

Remove the manual `impl Display`, `impl Error`, and `impl From` blocks for `AppError`. The `#[from]` attribute on each variant generates the `From` implementation automatically.

Run `cargo run`. The program compiles and produces the same output. The `?` operator still works because `From` is still implemented; it is just generated by the macro instead of written by hand.

### Task 5.4 -- Side by side comparison

Write the manual version's lines and the thiserror version's lines in `lab7-notes.md`:

**Manual version:**

```rust
// ConfigError
#[derive(Debug)]
enum ConfigError { /* 4 variants */ }
impl fmt::Display for ConfigError { /* match with 4 arms */ }
impl std::error::Error for ConfigError {}

// AppError
#[derive(Debug)]
enum AppError { /* 2 variants */ }
impl fmt::Display for AppError { /* match with 2 arms */ }
impl std::error::Error for AppError {}
impl From<std::io::Error> for AppError { /* */ }
impl From<ConfigError> for AppError { /* */ }
```

**thiserror version:**

```rust
#[derive(Debug, Error)]
enum ConfigError { /* 4 variants with #[error("...")] */ }

#[derive(Debug, Error)]
enum AppError { /* 2 variants with #[error] and #[from] */ }
```

The thiserror version is roughly half the lines and significantly more readable. The behavior is identical: the same traits are implemented, the `?` operator works the same way, callers see the same error messages.

This is the practical use of macro-based code generation: when there is a uniform pattern that you would write the same way every time, a macro can generate it. You write the high-level intent (variant names and message templates); the macro produces the boilerplate.

### Task 5.5 -- The full revised program

Your `src/main.rs` should now look like this complete file. Verify yours matches:

```rust
use std::fs;
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("configuration file is empty")]
    EmptyFile,

    #[error("line {line_number}: invalid format '{content}'")]
    InvalidLine { line_number: usize, content: String },

    #[error("missing required field: {0}")]
    MissingField(String),

    #[error("invalid value '{value}' for field '{field}'")]
    InvalidValue { field: String, value: String },
}

#[derive(Debug, Error)]
enum AppError {
    #[error("I/O error: {0}")]
    IoError(#[from] std::io::Error),

    #[error("config error: {0}")]
    ConfigError(#[from] ConfigError),
}

struct Config {
    name: String,
    port: u16,
    debug: bool,
}

fn parse_config(text: &str) -> Result<Config, ConfigError> {
    // ... same as Exercise 4
    if text.trim().is_empty() {
        return Err(ConfigError::EmptyFile);
    }

    let mut name = None;
    let mut port = None;
    let mut debug = None;

    for (i, line) in text.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }

        let parts: Vec<&str> = line.splitn(2, '=').collect();
        if parts.len() != 2 {
            return Err(ConfigError::InvalidLine {
                line_number: i + 1,
                content: line.to_string(),
            });
        }

        let key = parts[0].trim();
        let value = parts[1].trim();

        match key {
            "name" => name = Some(value.to_string()),
            "port" => {
                let p: u16 = value.parse().map_err(|_| ConfigError::InvalidValue {
                    field: String::from("port"),
                    value: value.to_string(),
                })?;
                port = Some(p);
            }
            "debug" => match value {
                "true" => debug = Some(true),
                "false" => debug = Some(false),
                _ => {
                    return Err(ConfigError::InvalidValue {
                        field: String::from("debug"),
                        value: value.to_string(),
                    });
                }
            },
            _ => {
                return Err(ConfigError::InvalidLine {
                    line_number: i + 1,
                    content: line.to_string(),
                });
            }
        }
    }

    Ok(Config {
        name: name.ok_or_else(|| ConfigError::MissingField(String::from("name")))?,
        port: port.ok_or_else(|| ConfigError::MissingField(String::from("port")))?,
        debug: debug.ok_or_else(|| ConfigError::MissingField(String::from("debug")))?,
    })
}

fn read_and_parse(path: &str) -> Result<Config, AppError> {
    let contents = fs::read_to_string(path)?;
    let config = parse_config(&contents)?;
    Ok(config)
}

fn main() {
    match read_and_parse("nonexistent.toml") {
        Ok(c) => println!("ok: {}", c.name),
        Err(e) => println!("error: {e}"),
    }

    let inputs = vec![
        "",
        "name = web-server\nport = 8080\ndebug = true",
        "name = web-server\nport = 8080",
        "name = web-server\nport = abc\ndebug = true",
    ];

    for input in inputs {
        match parse_config(input) {
            Ok(c) => println!("OK: name={}, port={}", c.name, c.port),
            Err(e) => println!("ERROR: {e}"),
        }
    }
}
```

Run the program. The output should match the previous exercises, demonstrating that the behavior is unchanged; only the source code is shorter.

### Checkpoints

1. The `#[from]` attribute generated `From` implementations automatically. What is the requirement for `#[from]` to work? When can you not use it?
2. The `#[error("...")]` format string can reference fields. What field syntaxes does the format string support? What about for tuple variants?
3. `thiserror` is a procedural macro. The code is generated at compile time and ends up identical to what you would write by hand. What does this tell you about the cost of using `thiserror` versus writing the impls manually?

---

## Exercise 6 -- Using anyhow for Application-Level Errors

**Estimated time:** 10-15 minutes
**Topics covered:** the `anyhow` crate, `anyhow::Result`, adding context

### Context

For library code, specific error types (using `thiserror`) are valuable. For application code, the program often does not need to distinguish between many error types; it just needs to log the error, return an appropriate exit code, and move on. The `anyhow` crate provides a generic error type that wraps any error implementing `Error`, with extra features for adding context.

### Task 6.1 -- Adding anyhow as a dependency

Open `Cargo.toml` and add `anyhow`:

```toml
[dependencies]
thiserror = "1"
anyhow = "1"
```

Save the file. The next build will download `anyhow`.

### Task 6.2 -- Rewriting main with anyhow

Modify your program to use `anyhow` in `main`:

```rust
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let config = read_and_parse("nonexistent.toml")
        .context("failed to load configuration")?;

    println!("loaded: name={}, port={}", config.name, config.port);

    Ok(())
}
```

Run the program (with `cargo run`). Expected output:

```
Error: failed to load configuration

Caused by:
    0: I/O error: No such file or directory (os error 2)
    1: No such file or directory (os error 2)
```

Two notable changes:

- `main` now returns `Result<()>` (which is `anyhow::Result<()>`, equivalent to `Result<(), anyhow::Error>`).
- The `.context("failed to load configuration")` adds context to the error, producing a chain when displayed.

The output shows the chain: the high-level context ("failed to load configuration"), then the wrapped `AppError`, then the underlying I/O error. Each layer adds information.

### Task 6.3 -- Adding context at multiple levels

Add context to multiple operations:

```rust
fn read_and_parse(path: &str) -> anyhow::Result<Config> {
    let contents = fs::read_to_string(path)
        .with_context(|| format!("failed to read config file: {path}"))?;

    let config = parse_config(&contents)
        .with_context(|| format!("failed to parse config from {path}"))?;

    Ok(config)
}

fn main() -> anyhow::Result<()> {
    let config = read_and_parse("config.toml")
        .context("failed to load configuration during startup")?;

    println!("ok: {}", config.name);
    Ok(())
}
```

Note: the function's return type changed to `anyhow::Result<Config>`. This is what allows context to be added; the underlying `AppError` is wrapped automatically.

Run the program. Expected output (when config.toml does not exist):

```
Error: failed to load configuration during startup

Caused by:
    0: failed to read config file: config.toml
    1: No such file or directory (os error 2)
```

Each layer of context is visible in the chain. The reader can see what was being attempted at each level: the program was loading configuration, which involved reading the file, which failed.

This kind of error chain is invaluable for debugging production issues. The bare error "No such file or directory" tells you what failed but not what was being attempted; the chain tells you both.

### Task 6.4 -- with_context vs context

Two methods exist for adding context:

```rust
.context("static message")
.with_context(|| format!("dynamic message: {variable}"))
```

`context` takes a fixed string; `with_context` takes a closure that produces the message. Use `with_context` when:

- The message includes runtime data (variable values, computed paths).
- The message is expensive to compute and you want to skip the cost on the success path.

Use `context` when:

- The message is a fixed string.

The closure form is slightly more code but more flexible. Most production code uses `with_context` regularly.

### Task 6.5 -- Use ? without explicit error type

`anyhow::Result<T>` is shorthand for `Result<T, anyhow::Error>`. The `anyhow::Error` type accepts any error implementing `std::error::Error`. This means `?` works without `From` implementations:

```rust
fn read_anything(path: &str) -> anyhow::Result<i32> {
    let contents = fs::read_to_string(path)?;        // io::Error
    let n: i32 = contents.trim().parse()?;            // ParseIntError
    Ok(n)
}
```

No custom error enum, no `From` impls, no boilerplate. The function reads cleanly. Both error types are wrapped into `anyhow::Error` automatically.

This is the trade-off: `anyhow` is much less code but provides less structure. Callers cannot match on specific error variants because the type is opaque. `anyhow` is for application code where you do not need to distinguish; libraries should still use specific error types.

### Task 6.6 -- The two libraries together

The recommended pattern is to use both:

- **`thiserror`** in libraries to define specific error types.
- **`anyhow`** in applications to combine errors from multiple libraries with added context.

Your program already uses both:

- `ConfigError` and `AppError` (defined with `thiserror`) are used by `parse_config` and `read_and_parse`. They are library-level types.
- `main` uses `anyhow::Result<()>` to combine everything with context. It is application-level.

If `read_and_parse` were a library function, it would return `Result<Config, AppError>` (using `thiserror`). Application code would then convert to `anyhow::Error` at the top level, adding context.

The two crates are not competitors; they fill complementary roles. Most production Rust projects use both.

### Checkpoints

1. The `anyhow::Error` type accepts any error implementing `std::error::Error`. What is the cost of this generality compared to a specific error type? What is the benefit?
2. The `with_context` method takes a closure rather than a string. Why is the closure form sometimes preferable, even though `context(string)` looks simpler?
3. The "use thiserror in libraries, anyhow in applications" guideline is a community convention, not a language requirement. What goes wrong if a library uses `anyhow` everywhere, or an application defines exhaustive error enums?

---

## Exercise 7 -- Error Handling Best Practices

**Estimated time:** 10-15 minutes
**Topics covered:** preserving error information, avoiding stringification, testing error paths

### Context

This exercise demonstrates several best practices through small examples. Each illustrates a common antipattern and the better alternative.

### Task 7.1 -- Preserve error structure

Replace `src/main.rs` with this antipattern (do not commit it):

```rust
use std::fs;

// ANTIPATTERN: stringified errors lose information
fn read_count(path: &str) -> Result<i32, String> {
    let contents = fs::read_to_string(path)
        .map_err(|e| e.to_string())?;            // converts io::Error to String

    let n: i32 = contents.trim().parse()
        .map_err(|e| e.to_string())?;            // converts ParseIntError to String

    Ok(n)
}

fn main() {
    match read_count("nonexistent.txt") {
        Ok(n) => println!("count: {n}"),
        Err(e) => println!("error: {e}"),
    }
}
```

Run the program. The output is fine for display, but the original error structure is gone:

```
error: No such file or directory (os error 2)
```

The caller cannot tell that this was an I/O error specifically. They cannot extract the OS error code, the path, or any other structured information. They just have a string.

### Task 7.2 -- Preserve structure with thiserror

The right way (using thiserror, as in Exercise 5):

```rust
use std::fs;
use thiserror::Error;

#[derive(Debug, Error)]
enum CountError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("not a number: {0}")]
    Parse(#[from] std::num::ParseIntError),
}

fn read_count(path: &str) -> Result<i32, CountError> {
    let contents = fs::read_to_string(path)?;
    let n: i32 = contents.trim().parse()?;
    Ok(n)
}

fn main() {
    match read_count("nonexistent.txt") {
        Ok(n) => println!("count: {n}"),
        Err(CountError::Io(e)) => {
            // The caller can inspect the io::Error directly.
            println!("I/O error: kind={:?}, message={e}", e.kind());
        }
        Err(CountError::Parse(e)) => {
            println!("parse error: {e}");
        }
    }
}
```

Run the program. The error output is similar, but the caller can pattern-match on the variant and inspect the underlying error. The `e.kind()` call returns an `io::ErrorKind` enum that the caller could use to handle different I/O errors specifically (file not found, permission denied, etc.).

The lesson: prefer wrapping errors over stringifying them. Wrapped errors preserve information; stringified errors discard it.

### Task 7.3 -- Avoid catch-all error variants

Another antipattern is the "catch-all" error variant:

```rust
#[derive(Debug, Error)]
enum AntiError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("other: {0}")]
    Other(String),                     // anti-pattern: dumping ground
}
```

The `Other(String)` variant becomes a temptation: any time you have an error you do not want to handle properly, you stuff it into `Other`. Over time, the variant becomes a bag of unstructured strings.

The better approach: each kind of failure gets its own variant. If you genuinely need a generic catch-all (and have many uncategorizable errors), use `anyhow::Error` instead of a string-based variant. `anyhow` preserves structure even when the type is generic.

### Task 7.4 -- Add context, but not too much

The `anyhow` context pattern is valuable, but you can overdo it. A balance:

**Too little context:**

```rust
let user = db.get_user(id)?;        // bare error from db; no context
```

If this fails, the error message is whatever the database driver produced, with no indication of what was being attempted.

**Right amount:**

```rust
let user = db.get_user(id)
    .with_context(|| format!("failed to fetch user {id}"))?;
```

Now the chain has the context (what was being attempted) and the underlying error (why it failed). Both are visible in the error message.

**Too much:**

```rust
let user = db.get_user(id)
    .with_context(|| format!("getting user from database"))?
    .map_err(|e| /* more wrapping */)?
    .with_context(|| format!("during request handling"))?;
```

Every line adds context. The error chain becomes a verbose list of phrases that each repeat the obvious. Aim for one layer of context per logical operation, not per line.

The rule of thumb: when you find yourself adding context, ask "would this help someone debugging this issue?" If yes, add it. If it just repeats what is already obvious from the call site, skip it.

### Task 7.5 -- Do not use unwrap on user input

A specific case of the "panic only for bugs" rule:

```rust
// ANTIPATTERN: unwrap on user input
fn parse_port_bad(input: &str) -> u16 {
    input.parse().unwrap()         // panics on bad input; user crashes
}

// CORRECT: return Result
fn parse_port_good(input: &str) -> Result<u16, String> {
    input.parse().map_err(|e: std::num::ParseIntError| {
        format!("invalid port '{input}': {e}")
    })
}

fn main() {
    let result = parse_port_good("80");
    println!("good port: {result:?}");

    let result = parse_port_good("abc");
    println!("bad port: {result:?}");

    // Don't run this:
    // let port = parse_port_bad("abc");    // panics
}
```

Run the program. The good function returns `Err` on bad input, which the caller can handle. The bad function would panic, taking down the entire program.

User input, file contents, network responses, environment variables, and command-line arguments are all "untrusted." They might be wrong. Code that processes them should return `Result`, not panic.

`unwrap` is appropriate when:

- You can prove the value will be valid (a constant, a value you just validated).
- A failure indicates a bug in your own code that you cannot recover from.

For everything else, return `Result` and let the caller decide.

### Task 7.6 -- Test the error paths

Add tests for the parser. Update the bottom of `src/main.rs` (or move the parsing function and types to `src/lib.rs` for proper testing):

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn empty_file_returns_error() {
        let result = parse_config("");
        assert!(matches!(result, Err(ConfigError::EmptyFile)));
    }

    #[test]
    fn missing_field_returns_specific_error() {
        let input = "name = web\nport = 80";    // missing debug
        let result = parse_config(input);
        match result {
            Err(ConfigError::MissingField(field)) => assert_eq!(field, "debug"),
            other => panic!("unexpected result: {other:?}"),
        }
    }

    #[test]
    fn invalid_port_returns_specific_error() {
        let input = "name = web\nport = abc\ndebug = true";
        let result = parse_config(input);
        assert!(matches!(
            result,
            Err(ConfigError::InvalidValue { field, .. }) if field == "port"
        ));
    }

    #[test]
    fn valid_config_parses() {
        let input = "name = web\nport = 8080\ndebug = false";
        let result = parse_config(input);
        let config = result.expect("should succeed");
        assert_eq!(config.name, "web");
        assert_eq!(config.port, 8080);
        assert_eq!(config.debug, false);
    }
}
```

Run the tests:

```bash
cargo test
```

All four tests should pass. Three of them exercise error paths; one exercises the success path. This is the right ratio: error paths often outnumber success paths and need just as much testing.

The `matches!` macro is useful for testing the variant of an error without writing a full match. The pattern `matches!(result, Err(ConfigError::EmptyFile))` returns `true` if the result is exactly that variant.

For more specific assertions (extracting the inner data), use a `match` or pattern-match in `if let`.

### Checkpoints

1. Stringifying errors (Task 7.1) loses structural information. Identify a specific scenario where the original `io::Error` would be useful but a `String` would not be enough.
2. The "catch-all" variant antipattern (Task 7.3) tempts you to put unstructured errors into a generic bucket. What is the long-term effect on the codebase if you give in to this temptation?
3. The error path tests in Task 7.6 used `matches!` and patterns. Could they have used `assert!(result.is_err())` instead? What would that give up?

---

## Exercise 8 -- Putting It Together

**Estimated time:** 10-15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise puts everything from the lab into a single small but realistic program. It demonstrates the patterns students will use in real Rust code: a library-level error type with `thiserror`, application-level handling with `anyhow`, validation, propagation with `?`, and tests for error paths.

### Task 8.1 -- The full program

Replace your `src/main.rs` with this complete program. (Make sure `Cargo.toml` has both `thiserror = "1"` and `anyhow = "1"`.)

```rust
use std::fs;
use thiserror::Error;
use anyhow::{Context, Result};

// ============================================================
// Library-level error type with thiserror
// ============================================================

#[derive(Debug, Error)]
enum ConfigError {
    #[error("configuration is empty")]
    Empty,

    #[error("line {line}: invalid format '{content}'")]
    InvalidLine { line: usize, content: String },

    #[error("missing required field: {0}")]
    MissingField(String),

    #[error("invalid value '{value}' for field '{field}': {reason}")]
    InvalidValue {
        field: String,
        value: String,
        reason: String,
    },

    #[error("port {port} is out of valid range (1-65535)")]
    PortOutOfRange { port: u32 },
}

// ============================================================
// Domain types
// ============================================================

#[derive(Debug)]
struct Config {
    name: String,
    port: u16,
    workers: u32,
    log_level: LogLevel,
}

#[derive(Debug, PartialEq)]
enum LogLevel {
    Error,
    Warn,
    Info,
    Debug,
}

impl LogLevel {
    fn from_str(s: &str) -> Option<LogLevel> {
        match s.to_lowercase().as_str() {
            "error" => Some(LogLevel::Error),
            "warn" => Some(LogLevel::Warn),
            "info" => Some(LogLevel::Info),
            "debug" => Some(LogLevel::Debug),
            _ => None,
        }
    }
}

// ============================================================
// Library function: parse with structured errors
// ============================================================

fn parse_config(text: &str) -> std::result::Result<Config, ConfigError> {
    if text.trim().is_empty() {
        return Err(ConfigError::Empty);
    }

    let mut name = None;
    let mut port: Option<u16> = None;
    let mut workers: Option<u32> = None;
    let mut log_level = None;

    for (i, line) in text.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }

        let parts: Vec<&str> = line.splitn(2, '=').collect();
        if parts.len() != 2 {
            return Err(ConfigError::InvalidLine {
                line: i + 1,
                content: line.to_string(),
            });
        }

        let key = parts[0].trim();
        let value = parts[1].trim();

        match key {
            "name" => name = Some(value.to_string()),
            "port" => {
                let parsed: u32 = value.parse().map_err(|e: std::num::ParseIntError| {
                    ConfigError::InvalidValue {
                        field: String::from("port"),
                        value: value.to_string(),
                        reason: e.to_string(),
                    }
                })?;
                if parsed == 0 || parsed > 65535 {
                    return Err(ConfigError::PortOutOfRange { port: parsed });
                }
                port = Some(parsed as u16);
            }
            "workers" => {
                let parsed: u32 = value.parse().map_err(|e: std::num::ParseIntError| {
                    ConfigError::InvalidValue {
                        field: String::from("workers"),
                        value: value.to_string(),
                        reason: e.to_string(),
                    }
                })?;
                workers = Some(parsed);
            }
            "log_level" => {
                log_level = Some(LogLevel::from_str(value).ok_or_else(|| {
                    ConfigError::InvalidValue {
                        field: String::from("log_level"),
                        value: value.to_string(),
                        reason: String::from("expected error/warn/info/debug"),
                    }
                })?);
            }
            _ => {
                return Err(ConfigError::InvalidLine {
                    line: i + 1,
                    content: line.to_string(),
                });
            }
        }
    }

    Ok(Config {
        name: name.ok_or_else(|| ConfigError::MissingField(String::from("name")))?,
        port: port.ok_or_else(|| ConfigError::MissingField(String::from("port")))?,
        workers: workers.ok_or_else(|| ConfigError::MissingField(String::from("workers")))?,
        log_level: log_level.ok_or_else(|| ConfigError::MissingField(String::from("log_level")))?,
    })
}

// ============================================================
// Application function: read and parse with anyhow context
// ============================================================

fn load_config(path: &str) -> Result<Config> {
    let contents = fs::read_to_string(path)
        .with_context(|| format!("failed to read config file: {path}"))?;

    let config = parse_config(&contents)
        .with_context(|| format!("failed to parse config from {path}"))?;

    Ok(config)
}

// ============================================================
// Main: top-level error handling
// ============================================================

fn main() -> Result<()> {
    // Run the parser against several inline samples (no actual file I/O):
    let samples = vec![
        ("valid", "name = web\nport = 8080\nworkers = 4\nlog_level = info"),
        ("empty", ""),
        ("missing field", "name = web\nport = 8080"),
        ("invalid port", "name = web\nport = abc\nworkers = 4\nlog_level = info"),
        ("port out of range", "name = web\nport = 70000\nworkers = 4\nlog_level = info"),
        ("bad log level", "name = web\nport = 80\nworkers = 4\nlog_level = silly"),
    ];

    for (label, text) in samples {
        match parse_config(text) {
            Ok(config) => println!("[{label}] OK: {config:?}"),
            Err(e) => println!("[{label}] ERR: {e}"),
        }
    }

    // Demonstrate anyhow context for file-based loading:
    println!("\n--- File loading test ---");
    match load_config("nonexistent.toml") {
        Ok(config) => println!("loaded: {config:?}"),
        Err(e) => {
            println!("Error: {e}");
            for cause in e.chain().skip(1) {
                println!("  caused by: {cause}");
            }
        }
    }

    Ok(())
}

// ============================================================
// Tests
// ============================================================

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn empty_input_is_empty_error() {
        let result = parse_config("");
        assert!(matches!(result, Err(ConfigError::Empty)));
    }

    #[test]
    fn whitespace_only_is_empty_error() {
        let result = parse_config("   \n  \t\n");
        assert!(matches!(result, Err(ConfigError::Empty)));
    }

    #[test]
    fn missing_field_specific_error() {
        let result = parse_config("name = web\nport = 80\nworkers = 1");
        match result {
            Err(ConfigError::MissingField(field)) => assert_eq!(field, "log_level"),
            other => panic!("unexpected: {other:?}"),
        }
    }

    #[test]
    fn invalid_port_value() {
        let result = parse_config("name = web\nport = abc\nworkers = 1\nlog_level = info");
        assert!(matches!(
            result,
            Err(ConfigError::InvalidValue { ref field, .. }) if field == "port"
        ));
    }

    #[test]
    fn port_out_of_range() {
        let result = parse_config("name = web\nport = 70000\nworkers = 1\nlog_level = info");
        assert!(matches!(result, Err(ConfigError::PortOutOfRange { port: 70000 })));
    }

    #[test]
    fn valid_config_succeeds() {
        let result = parse_config("name = web\nport = 80\nworkers = 4\nlog_level = info");
        let config = result.expect("should succeed");
        assert_eq!(config.name, "web");
        assert_eq!(config.port, 80);
        assert_eq!(config.workers, 4);
        assert_eq!(config.log_level, LogLevel::Info);
    }

    #[test]
    fn comments_are_ignored() {
        let result = parse_config(
            "# this is a comment\nname = web\n# another comment\nport = 80\nworkers = 4\nlog_level = info"
        );
        assert!(result.is_ok());
    }
}
```

Run the program:

```bash
cargo run
```

Expected output (the file-loading section may vary based on your environment):

```
[valid] OK: Config { name: "web", port: 8080, workers: 4, log_level: Info }
[empty] ERR: configuration is empty
[missing field] ERR: missing required field: workers
[invalid port] ERR: invalid value 'abc' for field 'port': invalid digit found in string
[port out of range] ERR: port 70000 is out of valid range (1-65535)
[bad log level] ERR: invalid value 'silly' for field 'log_level': expected error/warn/info/debug

--- File loading test ---
Error: failed to read config file: nonexistent.toml
  caused by: No such file or directory (os error 2)
```

Run the tests:

```bash
cargo test
```

Expected output:

```
running 7 tests
test tests::comments_are_ignored ... ok
test tests::empty_input_is_empty_error ... ok
test tests::invalid_port_value ... ok
test tests::missing_field_specific_error ... ok
test tests::port_out_of_range ... ok
test tests::valid_config_succeeds ... ok
test tests::whitespace_only_is_empty_error ... ok

test result: ok. 7 passed; 0 failed
```

### Task 8.2 -- Identify the patterns

In `lab7-notes.md`, identify each instance of the following patterns in the program above:

1. A custom error enum defined with `thiserror`.
2. A variant with named fields holding structured error data.
3. A `#[error("...")]` attribute that references a field by name in the format string.
4. An error variant that wraps another error type using `#[from]` (Note: this version doesn't use `#[from]` because it converts via map_err. Identify a place where `#[from]` could replace the manual conversion).
5. A use of `?` to propagate errors through several operations.
6. A use of `map_err` to convert one error type to another with custom logic.
7. A function that returns `anyhow::Result<T>`.
8. A use of `with_context` to add context to a propagated error.
9. A test that uses `matches!` to check the error variant.
10. A test that extracts inner data from an error using a `match`.

### Task 8.3 -- Predict and verify

For each of the following code changes, predict whether the program will compile and what behavior changes you expect. Write your predictions in `lab7-notes.md` BEFORE running.

**Change A:** In `parse_config`, replace `.map_err(|e: std::num::ParseIntError| ConfigError::InvalidValue { ... })?` with just `?` (no `map_err`). What error message do you expect?

**Change B:** Add a new error variant `ConfigError::DuplicateField(String)` to the enum but do not actually use it. Will the program still compile? Will tests still pass?

**Change C:** Change `load_config` to return `std::result::Result<Config, ConfigError>` instead of `anyhow::Result<Config>`. What changes are required at the call site? What error context features are lost?

For each change, write your prediction first, then run and verify.

### Checkpoints

1. The program uses `thiserror` for `ConfigError` and `anyhow` for `main` and `load_config`. Why is this division of labor better than using one library throughout?
2. The `parse_config` function's signature is `fn parse_config(text: &str) -> std::result::Result<Config, ConfigError>` (with the explicit `std::result::` prefix). Why was this fully-qualified path needed? What conflict was being avoided?
3. The tests in the lab use `matches!` for some error checks and full `match` for others. When is each appropriate? What is the trade-off?

---

## Summary and Reflection

You have now used every concept from Module 10 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- panic! and unwrap | unrecoverable errors | Panics are for bugs, not for handling input. Use `expect` over `unwrap` for better diagnostics. |
| 2 -- Result methods | match, common combinators | `Result` makes errors part of the type system. Methods like `map_err` and `and_then` avoid explicit match. |
| 3 -- The ? operator | error propagation, requirements | `?` makes the success path linear and pushes errors up automatically. |
| 4 -- Custom error types | enums with Display, Error, From | Specific error types preserve information; callers can match on variants to handle each kind. |
| 5 -- thiserror | derive-based error generation | `thiserror` produces the same code as the manual approach with much less boilerplate. |
| 6 -- anyhow | application-level errors with context | `anyhow` accepts any error and lets you add context as it propagates. Use it at the application level. |
| 7 -- Best practices | preserving structure, testing errors | Avoid stringifying errors, avoid catch-all variants, test error paths as carefully as success paths. |
| 8 -- Integration | all of the above | Real Rust uses thiserror in libraries, anyhow in applications. Each plays its role. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab7-notes.md` before your next session.

1. Of the error-handling concepts in Module 10, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. Rust's `Result<T, E>` makes errors values rather than control flow. This is unusual; most mainstream languages use exceptions. Identify a specific concrete benefit of values-as-errors compared to exceptions in a language you know, and a specific concrete cost. On balance, is the trade-off worth it?

3. The lab covered both manual error type definition (Exercise 4) and the library-based shortcut (Exercise 5). Now that you have written each, identify one thing that the manual version teaches you that the library version hides, and one thing the library version handles better than the manual version would in a real codebase.

---

*End of Lab 7*
