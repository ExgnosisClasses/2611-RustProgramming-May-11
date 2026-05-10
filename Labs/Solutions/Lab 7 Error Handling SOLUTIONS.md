# Module 10: Error Handling
## Lab 7 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Unrecoverable Errors with panic!

### Checkpoints

**1. Why does Rust panic on out-of-bounds indexing rather than returning a default or wrapping around?**

Returning a default would silently mask a bug. If `v[10]` returned 0 when the vector has 3 elements, the program would continue with wrong data. The rest of the program might do calculations on this 0, write it to a database, or display it to a user, all without indication that something is wrong. The bug would propagate, potentially producing wildly incorrect output far from where the actual mistake was made.

Wrapping around (modulo length) would have the same effect plus the additional confusion of producing a "valid-looking" element that is not the one the programmer intended. `v[10]` returning `v[1]` would be even more misleading.

Panicking immediately exposes the bug at the moment it occurs. The program stops, the developer sees the error, and the problem is debugged at the source rather than three steps later.

The principle is broader: Rust prefers loud failures over silent ones for programming errors. The `Vec::get` method exists for cases where out-of-bounds access is a legitimate possibility (returning `Option<&T>`). Direct indexing (`v[i]`) is for cases where the index is known to be valid. Mixing them up is a programming error, and panicking is the appropriate response.

**2. Why is `expect` preferred over `unwrap` for better diagnostics?**

The custom message in `expect` tells the developer (or anyone reading logs) what the programmer believed about the value. When a panic happens, the message answers the question "what did the developer think would be true here?"

For example:

```rust
let value = config.timeout_ms.expect("config.toml should always have timeout_ms after loading");
```

If this panics, the message immediately tells you that the program assumed timeout_ms was set after loading. The developer reading the log can think: "ah, loading must have failed silently, or there's a code path that bypasses loading."

The bare `unwrap()` would just say "called Option::unwrap on a None value" with the file and line. You would have to dig into the code to figure out what was supposed to be there.

In production, panics are rare but expensive to debug. Anything that helps the on-call engineer understand the failure faster is worth the few extra characters of typing. The convention is: use `expect` over `unwrap`, and write the message as if explaining your assumption to someone reading a log.

**3. Three scenarios for the rule of thumb (panic vs unwrap vs Result).**

**Panic appropriate (bug, cannot continue):**

- A function that takes a sorted slice and binary-searches it. If you call it with an unsorted slice, the function's behavior is undefined; panicking on a debug-build assertion is appropriate. The caller violated the function's precondition; the program cannot meaningfully continue.

**`unwrap`/`expect` appropriate (failure is impossible by construction):**

- Parsing a string literal you wrote yourself: `let pi: f64 = "3.14159".parse().unwrap();`. You know the literal parses; the parse cannot fail unless someone edits the source code in a way that obviously breaks. Using `unwrap` here is reasonable and mainstream.

- Indexing into a vector immediately after pushing to it: `v.push(x); let last = &v[v.len() - 1];`. The push guarantees the index is valid.

**`Result` appropriate (might fail at runtime, caller should handle):**

- Reading a file (`fs::read_to_string`). The file might not exist, might lack permissions, or might have I/O errors. The caller is in the best position to decide what to do (try a backup file, log and exit, prompt the user, etc.).

- Parsing user input (`input.parse::<i32>()`). The user might type anything. The function should return `Result` so the caller can show an error message and ask again.

- Network operations, database queries, anything involving external systems.

The pattern: panic for bugs in your own code, `unwrap`/`expect` when you can prove success, `Result` for everything that might genuinely fail at runtime.

---

## Exercise 2 -- Result and the Match Pattern

### Checkpoints

**1. When is explicit match preferred over `unwrap_or` and similar methods?**

When the success and error cases require non-trivial different logic. Methods like `unwrap_or` are convenient when the error case is just "use a default value." But when the error case needs to log, retry, return a different error, or perform other actions, an explicit match is clearer:

```rust
match read_config(path) {
    Ok(c) => process(c),
    Err(ConfigError::FileNotFound(_)) => {
        eprintln!("no config; using defaults");
        process(default_config())
    }
    Err(ConfigError::ParseError { line, .. }) => {
        eprintln!("syntax error on line {line}; aborting");
        std::process::exit(1);
    }
    Err(e) => {
        eprintln!("config error: {e}");
        std::process::exit(2);
    }
}
```

This kind of multi-branch handling does not fit `unwrap_or`. The match expresses each case clearly.

The rule of thumb: use `unwrap_or`, `map`, `and_then` for simple, uniform handling. Use match when the cases differ enough to need their own logic.

**2. Why is short-circuiting useful in `and_then` chains?**

Short-circuiting saves work and produces clearer error messages. Consider a chain that:

1. Parses a string into a number.
2. Looks up a user by that number.
3. Fetches the user's preferences.
4. Returns the preferred theme.

If parsing fails, there is no point in trying to look up a user. If the user does not exist, there is no point in fetching preferences. Short-circuiting means each step only runs if the previous one succeeded; an early failure stops the chain.

The error message is also clearer. If you ran all four steps regardless and collected errors, you would see "parse failed, lookup failed (because parse failed), fetch failed (because lookup failed), preference missing." The cascade obscures the actual cause. Short-circuiting gives you just the first error: "parse failed."

For independent operations (each step does not depend on the previous), short-circuiting may not be what you want. There you would use other patterns: `Vec<Result>` and inspect each, or libraries that collect multiple errors.

**3. When would you collect all errors instead of just the first?**

When validating user input where the user wants to see every problem at once, not fix one error and resubmit just to see the next. Form validation is the canonical case:

```rust
let mut errors = Vec::new();
if name.is_empty() { errors.push(String::from("name is required")); }
if age < 0 { errors.push(String::from("age must be non-negative")); }
if !email.contains('@') { errors.push(String::from("email looks invalid")); }

if !errors.is_empty() {
    return Err(errors);
}
```

The pattern is to accumulate errors in a `Vec` rather than returning on the first one. The user fills out the form, sees all problems, and fixes them all in one round trip.

The standard library does not directly support this pattern with iterator methods. You can write it manually, or use crates like `validator` that handle accumulation. For most applications (CLI tools, server requests, configuration parsing), short-circuiting on the first error is appropriate; for user-facing forms, accumulation is.

---

## Exercise 3 -- The ? Operator

### Checkpoints

**1. Beyond brevity, what does `?` give you in terms of readability?**

The `?` operator preserves the linearity of the success path. Code with explicit match for every fallible call interleaves the happy path and the error path:

```rust
let a = match step1() {
    Ok(v) => v,
    Err(e) => return Err(e),
};
let b = match step2(a) {
    Ok(v) => v,
    Err(e) => return Err(e),
};
let c = match step3(b) {
    Ok(v) => v,
    Err(e) => return Err(e),
};
```

The reader has to mentally filter out the error-handling boilerplate to follow the logic. With `?`:

```rust
let a = step1()?;
let b = step2(a)?;
let c = step3(b)?;
```

The success path is obvious. The error path is implied by the `?`s. A reader scanning this code immediately sees "do step1, then step2, then step3, with each potentially failing."

This is similar to how exception-using languages let you write linear code that handles errors implicitly. The difference is that `?` is visible in the source (you can see exactly where errors might propagate), while exceptions are invisible. Rust's choice gives you the readability benefit of implicit handling with the explicitness of seeing each propagation point.

**2. Trade-off of `Box<dyn std::error::Error>` vs custom error types.**

`Box<dyn std::error::Error>` accepts any error type, requiring no conversion code. It is convenient for quick prototypes, application-level code, and the experimental phase of a project.

The trade-offs:

- **Loss of structure.** The caller cannot match on specific error variants; they just get "some error." This makes recovery code harder; you can log the error or exit, but it is awkward to handle different error kinds differently.
- **Heap allocation.** Each error is boxed, adding allocation overhead. For most applications this is negligible; for very hot paths, it can matter.
- **Less idiomatic for libraries.** Library users expect specific error types they can match on. A library that returns `Box<dyn Error>` everywhere is harder to use programmatically.

A custom error type (manual or via thiserror) gives you:

- **Pattern matching on variants.** Callers can handle each kind specifically.
- **No allocation.** Error values are stack-allocated.
- **Documentation in the type.** The error type lists every possible failure, serving as a reference for users.

The convention: use `Box<dyn Error>` for application-level glue (the closer you are to `main`, the more acceptable it is). Use specific types for library APIs. The `anyhow::Error` crate provides a similar generic type with extra features (context, chain printing) that makes it more useful than bare `Box<dyn Error>` for application code.

**3. Why does `?` only work in functions returning `Result` or `Option`?**

The `?` operator is short for "if Err, return this error from the current function." For "return this error" to make sense, the current function must be capable of returning errors. A function returning `()` cannot return an error; it just returns nothing.

If `?` were allowed in any function, what would happen? The compiler would have to either:

- Convert errors to panics (changing behavior dramatically based on context).
- Discard errors silently (defeating the purpose of error handling).
- Pick one of these as the default and let users override it (adding complexity).

None of these is satisfying. The current rule (require the enclosing function to return `Result` or `Option`) is simple and matches the intent: error propagation only makes sense in a function that propagates errors.

The fix when you want `?` in `main` is to change `main`'s signature: `fn main() -> Result<(), Box<dyn std::error::Error>>`. Now `main` returns `Result`, and `?` works. Rust automatically prints the error and exits with a non-zero status if `main` returns `Err`.

The same applies to closures; the `?` works only in closures whose return type is `Result` or `Option`. This is a less common case but follows the same principle.

---

## Exercise 4 -- Defining Custom Error Types

### Checkpoints

**1. Why do enum variants have different shapes?**

Different errors carry different information. A "file not found" error needs to know the path. A "parse error on line N" needs the line number and content. A "value out of range" needs the field, value, and range. Forcing all variants to have the same shape would either:

- Use the largest possible shape for everything, wasting space and making variants carry meaningless data.
- Use `Option` for fields that some variants do not use, which adds runtime checks and obscures the intent.
- Stringify everything into a single message field, losing structured information.

By letting each variant have its own shape, the type captures exactly the relevant information. A `MissingField(String)` carries just the field name; a `ParseError { line, content, expected }` carries the multiple pieces relevant to a parse error. The type system enforces that you provide the right data for each variant; the compiler catches mistakes at construction time.

This is the same flexibility that makes `Option`, `Result`, and other standard enums useful. Different variants need different data; the language supports this directly.

**2. Why does `std::error::Error` exist if it has no required methods?**

The trait serves as a marker and as a hook for the ecosystem. A type implementing `Error` signals that it is intended as an error type and can be used wherever errors are expected.

Specifically:

- **`Box<dyn Error>` and similar.** Code that wants to accept "any error" uses `Box<dyn std::error::Error>`. This works because `Error` is the trait used to define what "any error" means. Without the trait, there would be no way to express this generality.
- **Error chain support.** The `Error` trait has a default-implemented `source` method for getting the underlying cause. Errors that wrap other errors override this; tools like `anyhow` use it to build chains.
- **Future extensibility.** The trait can be extended with new default-implemented methods without breaking existing implementations.
- **Convention.** Even with no required methods, implementing `Error` documents intent. Code reviewers see `impl Error for MyType` and understand "this is meant to be an error."

The pattern of "marker trait with optional methods" appears elsewhere in Rust (e.g., `Send`, `Sync`, `Sized`). The trait carries semantic meaning even when it has no implementation requirements.

**3. What information is preserved by wrapping I/O errors instead of stringifying them?**

Quite a lot:

- **Error kind.** `io::Error::kind()` returns an `io::ErrorKind` enum: `NotFound`, `PermissionDenied`, `TimedOut`, etc. Callers can use this to decide how to handle the error (retry on timeout, fail immediately on permission denied).
- **OS error code.** On Linux, the underlying errno value is preserved. This can be useful for OS-specific handling.
- **Error chain.** If the I/O error wraps something deeper (a file system error wraps a kernel error, etc.), the chain is preserved.
- **Path information.** Some I/O errors include the path that was being accessed.

Stringifying with `e.to_string()` produces a single human-readable message that loses all of this. The caller gets "No such file or directory (os error 2)" but cannot tell whether it was specifically a "not found" error (versus, say, a permission error, which has the same kind of string).

A real-world consequence: if the application wants to distinguish "config not found, use defaults" from "config exists but couldn't read it, error out," stringified errors make this hard. Wrapped errors make it easy: match on `ErrorKind::NotFound`.

The general rule: prefer wrapping over stringification. Convert to a string only at the end, when you actually need to display the error.

---

## Exercise 5 -- Using thiserror

### Checkpoints

**1. What does `#[from]` require?**

The `#[from]` attribute generates a `From<T> for YourError` implementation. For it to work:

- The variant must hold the wrapped error as its only field (or the only non-`source` field).
- The variant can be either tuple-style (`Io(#[from] io::Error)`) or struct-style with a `source` field.
- There can be at most one `#[from]` per variant.
- Different variants can use `#[from]` for different types, but each type can be used only once across the enum (otherwise the `From` impl would be ambiguous).

The restriction "each type used at most once" is the most common limitation. If you have two variants that both wrap `io::Error`, you cannot put `#[from]` on both; the compiler would not know which variant to convert into. You would need to use `#[from]` on one and convert the other manually.

In practice, the rules rarely cause problems. Most error enums have one variant per wrapped error type, which fits the `#[from]` pattern naturally.

**2. Format string syntax in `#[error]`.**

The `#[error("...")]` format string supports:

- **Named field references:** `{field_name}` for struct-style variants. `#[error("missing {0}")]` would not work for `MissingField { field: String }`; you need `#[error("missing {field}")]`.
- **Positional references:** `{0}`, `{1}`, etc. for tuple-style variants. `#[error("io error: {0}")]` for `Io(io::Error)`.
- **Format specifiers:** Standard format specifiers work: `{field:?}` for Debug formatting, `{field:.2}` for two decimal places, etc.
- **Expressions in named fields:** Some thiserror versions support expressions: `{value:?}`, `{0:?}`. Check the version's documentation.

The format string is processed at macro expansion time, which means errors in the format string become compiler errors. If you reference a field that does not exist, the compiler will tell you.

For more complex formatting (multi-line messages, conditional content, etc.), you can implement `Display` manually instead of using `#[error]`. The two approaches can coexist: use `#[error]` for variants where it suffices, write `Display` for variants that need more.

**3. What does macro-based code generation tell you about cost?**

The generated code is identical to what you would write by hand. There is no runtime cost: no extra dispatch, no allocation, no overhead. The macro just saves you typing.

This is the same property as Rust's other zero-cost abstractions (generics, iterators, closures): the abstraction exists at the source level for the developer's benefit; the compiler reduces it to the same machine code that hand-written specialized code would produce.

For procedural macros like `thiserror`, the cost is at compile time: the macro runs during compilation, producing the expanded code. This adds compilation time but does not affect runtime. For most projects, the compile-time cost is negligible.

The lesson: macro-based code generation in Rust is a way to remove boilerplate without paying runtime cost. When a pattern is common enough to have a macro library, using the library is usually the right choice. The exception is when you need behavior that the macro does not support; then you fall back to manual implementation.

---

## Exercise 6 -- Using anyhow for Application-Level Errors

### Checkpoints

**1. What is the cost of `anyhow::Error`'s generality?**

The cost is the loss of compile-time type information about which errors are possible. With a specific error type, callers can match on variants and handle each case differently. With `anyhow::Error`, the type is opaque; callers can only display the error or downcast to a specific type via `error.downcast_ref::<T>()`, which is verbose.

Specifically:

- **Pattern matching is harder.** You cannot write `match error { MyError::A => ..., MyError::B => ... }` because the type does not have visible variants.
- **Error recovery is awkward.** Code that wants to retry on certain errors but not others has to use `downcast_ref` or string matching, both of which are fragile.
- **API documentation is less clear.** A function returning `anyhow::Result<T>` does not document what errors it might produce.

The benefit is convenience: no need to define error types, no `From` impls, automatic conversion of any error via `?`, and the context-adding feature.

The trade-off: use `anyhow::Error` at application boundaries where you do not need type-level discrimination. Use specific types at library boundaries where users need to match. Most non-trivial projects use both.

**2. Why is `with_context(closure)` sometimes preferable to `context(string)`?**

Two reasons.

**Cost on the success path.** The closure form only runs the format expression if there is an error. The string form formats the message regardless. If the message is expensive to compute (e.g., includes a large struct's debug representation), the closure form avoids the cost when no error occurs.

```rust
// Always formats the message, even if there is no error:
.context(format!("processing {}", expensive_to_format(&data)))

// Formats the message only if there is an error:
.with_context(|| format!("processing {}", expensive_to_format(&data)))
```

For most messages, the cost difference is negligible. For very hot paths, it can matter.

**Capturing context.** The closure captures variables at the call site, which makes it easy to include runtime data:

```rust
let path = "/etc/myapp.toml";
fs::read_to_string(path).with_context(|| format!("failed to read {path}"))?;
```

The string form would require pre-formatting the message. The closure form composes naturally.

In practice, `with_context` is more common in real code because the messages usually include data. `context` is used when the message is genuinely fixed.

**3. What goes wrong if a library uses `anyhow` everywhere or an application defines exhaustive enums?**

**Library using anyhow everywhere:**

- Users cannot match on specific errors. They can only display them or downcast, both of which are awkward.
- The library's API is less documented; users do not know what failures to expect.
- Refactoring is harder because there are no compile-time checks for which errors callers depend on.
- The library becomes harder to use programmatically (e.g., for retry logic, fallback behavior).

The library is not unusable, but it is harder to use well. Users either accept the loss of structure or have to wrap the library's errors themselves.

**Application defining exhaustive enums:**

- Verbose and time-consuming to maintain.
- Each new dependency requires a new `From` impl in your error type.
- Your error type becomes a kitchen sink, with variants for every possible source of failure across the codebase.
- The exhaustiveness checking that makes specific types valuable is undermined; you end up with one giant enum.

The application is not broken, but the error type does not provide much value. Users (or the application's own code) cannot meaningfully handle each variant differently; in practice, all errors get logged and the program exits or returns a generic error.

The convention exists for good reason: libraries benefit from specific errors (their users want to handle them); applications benefit from generic errors with context (they just want to log and continue or exit).

---

## Exercise 7 -- Error Handling Best Practices

### Checkpoints

**1. When would the original `io::Error` matter but a `String` would not?**

The most common case: handling specific I/O failures differently. For example, a configuration loader that should use defaults if the file does not exist but error out on permission denied:

```rust
match fs::read_to_string(path) {
    Ok(contents) => parse(&contents),
    Err(e) if e.kind() == io::ErrorKind::NotFound => Ok(default_config()),
    Err(e) => Err(e.into()),
}
```

With `e.kind()`, the code distinguishes "file does not exist" (use defaults) from "permission denied" (real error). With a stringified error, the same logic would require parsing the message string, which is fragile (the OS might phrase the error differently across platforms or versions).

Other cases:

- **Retry logic.** Retry on timeout but not on permission denied. The kind distinguishes them.
- **User feedback.** Show "file not found" with a helpful message but "permission denied" with a different one.
- **Logging detail.** Different error kinds get different log levels.

In all these cases, the structured error is essential. Stringification would make the logic either fragile or impossible.

**2. Long-term effects of catch-all error variants.**

The variant becomes a dumping ground. Whenever a developer encounters an error they do not want to add a specific variant for, they put it in `Other(String)` or `Generic(String)`. Over time:

- The percentage of errors going into the catch-all grows. The structured variants are used only for the original few cases.
- Pattern matching on the error type becomes useless. Every callsite ends up handling the catch-all generically (logging the string and continuing or exiting).
- The error type's value disappears. It conveys little more than `String` would have.

The fix is discipline: when you need a new error case, add a new variant rather than reaching for the catch-all. Code review can catch the antipattern.

A related antipattern is using `String` as the error type entirely: `Result<T, String>`. This is sometimes acceptable for internal helpers or quick prototypes but is wrong for any persistent code. The `anyhow::Error` type fills the same role with much more information preserved.

**3. Could `assert!(result.is_err())` replace `matches!` and patterns?**

It could, but it would test less. `assert!(result.is_err())` only verifies that the result is some error. The test would pass even if a refactoring changed the error variant, the message, or the wrapped data, as long as the result is still an error of some kind.

`matches!(result, Err(MyError::SpecificVariant))` verifies the exact variant. If the code changes to return a different error variant (intentionally or by mistake), the test fails. The test pins down the contract: not just "this fails" but "this fails specifically because of X."

The trade-off is over-specification. A test that asserts "the error is exactly `MyError::ParseError { line: 5, content: \"abc\" }`" might break for unrelated reasons (e.g., the line counting changed in a refactor). Aim for the right level of specificity:

- Assert the variant (catches changes in failure category).
- Optionally assert key fields (catches incorrect data).
- Avoid asserting the entire error in detail (creates brittle tests).

The `matches!` macro with a partial pattern is the right tool: `matches!(result, Err(MyError::ParseError { .. }))` checks the variant without pinning down internal data.

---

## Exercise 8 -- Putting It Together

### Task 8.2 -- Pattern identification

1. **Custom error enum with thiserror:** `enum ConfigError` with `#[derive(Debug, Error)]` and `#[error("...")]` attributes on variants.

2. **Variant with named fields:** `ConfigError::InvalidLine { line, content }`, `ConfigError::InvalidValue { field, value, reason }`, `ConfigError::PortOutOfRange { port }`.

3. **Format string referencing field by name:** `#[error("line {line}: invalid format '{content}'")]` and `#[error("invalid value '{value}' for field '{field}': {reason}")]`.

4. **Where #[from] could replace manual conversion:** The `parse_config` function uses `map_err` to convert `ParseIntError` into `ConfigError::InvalidValue`. To use `#[from]`, you would need a separate variant like `ParseError(#[from] std::num::ParseIntError)` and the function would propagate that variant directly. The current design uses `map_err` because the resulting `ConfigError::InvalidValue` carries more context (which field, what value) than just the inner ParseIntError.

5. **`?` propagating through several operations:** The `parse_config` function uses `?` after `map_err` and after `ok_or_else` calls. The `load_config` function uses `?` after `read_to_string` and `parse_config`.

6. **`map_err` for custom conversion:** The `port` parsing in `parse_config`: `value.parse().map_err(|e: std::num::ParseIntError| ConfigError::InvalidValue { ... })?`.

7. **Function returning `anyhow::Result<T>`:** `load_config(path: &str) -> Result<Config>` (the `Result` here is `anyhow::Result`) and `main() -> Result<()>`.

8. **`with_context`:** In `load_config`: `.with_context(|| format!("failed to read config file: {path}"))?` and `.with_context(|| format!("failed to parse config from {path}"))?`.

9. **Test using `matches!`:** `tests::empty_input_is_empty_error`, `tests::whitespace_only_is_empty_error`, `tests::invalid_port_value`, `tests::port_out_of_range`.

10. **Test using full `match`:** `tests::missing_field_specific_error` extracts the inner `field` value to assert against. The `matches!` macro could not do that directly without more advanced patterns.

### Task 8.3 -- Predict and verify

**Change A:** Replace `.map_err(...)?` with just `?` for parsing the port.

**Prediction:** This will fail to compile. The `parse()` returns `Result<u32, ParseIntError>`. Using `?` would propagate a `ParseIntError`, but `parse_config` returns `Result<Config, ConfigError>`. There is no `From<ParseIntError> for ConfigError` implementation, so the conversion fails.

**Result:** As predicted, the compiler rejects this with:

```
error[E0277]: the trait bound `ConfigError: From<ParseIntError>` is not satisfied
```

The fix is either to add a `From` impl (or `#[from]` attribute on a new variant) or keep the `map_err` to provide the structured `InvalidValue` error. The latter is preferred here because the `InvalidValue` variant carries more information than a simple wrapped `ParseIntError` would.

The lesson: `?` requires a conversion path between error types. When the conversion is structural (preserving information), use `From`. When the conversion needs to add context (like which field had the bad value), use `map_err` with explicit construction.

**Change B:** Add `DuplicateField(String)` variant without using it.

**Prediction:** The program will compile. Tests will pass. The unused variant generates a "variant is never constructed" warning (because `#[derive(Debug, Error)]` produces some uses, but the variant is never constructed in your code).

**Result:** As predicted, the program compiles with a warning. Tests pass without modification because none of them produces or expects `DuplicateField`. The warning suggests adding `#[allow(dead_code)]` or actually using the variant.

The lesson: error enums can be extended without breaking existing code, as long as no `match` expression on the type was non-exhaustive. The `match` expressions in this lab all use `matches!` or specific variants, so they handle the extension gracefully. If a `match` had used `_` as a catch-all, the new variant would be silently caught by the default; if a `match` listed every variant explicitly, adding a new one would force the compiler to flag it.

This is one of the practical benefits of exhaustive matching: adding variants forces you to consider each consumer.

**Change C:** Change `load_config` to return `Result<Config, ConfigError>` instead of `anyhow::Result<Config>`.

**Prediction:** Several things change:

- The return type is no longer compatible with `?` propagating an `io::Error` from `read_to_string`. You would need to either add a `From<io::Error> for ConfigError` impl (which would require a new variant) or use `map_err` to convert manually.
- `with_context` no longer works because that is an `anyhow` feature. The error chain support is lost.
- `main` would need to be adjusted: either change its return type to `Result<(), ConfigError>` (which works but loses the ability to handle the file-loading test's general error), or convert the error at the call site.

**Result:** As predicted, multiple compilation errors. The fixes require either:

- Adding an `Io(#[from] std::io::Error)` variant to `ConfigError` and removing the `with_context` calls.
- Or, restructuring to keep `load_config` returning a wrapper error type like the original `AppError`, while keeping `main` using `anyhow`.

The lesson: `anyhow` and `thiserror` complement each other. `thiserror` is for specific library-level errors. `anyhow` is for application-level glue with context. Mixing them at the wrong level (using `thiserror` where `anyhow` would be appropriate) causes friction. The standard pattern in real codebases is to use `thiserror` for the library-style functions like `parse_config` and `anyhow` for the application-level coordination like `load_config` and `main`.

### Checkpoints

**1. Why use both thiserror and anyhow?**

The two libraries optimize for different use cases:

- **`thiserror`** is for library-style code: specific error types with named variants, structured information per variant, ability for callers to match on specific failures.
- **`anyhow`** is for application-style code: combining errors from multiple sources, adding context as errors propagate up, displaying error chains for debugging.

Using only thiserror everywhere means you cannot easily add context as errors propagate (you would have to define wrapper variants for each level of context). Using only anyhow means you lose the ability to match on specific error variants (the type is opaque).

The combination matches the natural structure of programs: leaf functions and library code use thiserror (precise errors), application-level code uses anyhow (combined errors with context). The two crates are designed to work together; `anyhow::Error` accepts any `thiserror`-defined error automatically.

**2. Why was `std::result::Result<Config, ConfigError>` written with a fully-qualified path?**

Because `Result` was already brought into scope via `use anyhow::{Context, Result};`. That `Result` is `anyhow::Result<T>`, which is `Result<T, anyhow::Error>` with the error type implicit. If you write `Result<Config, ConfigError>` after that import, the compiler interprets `Result` as `anyhow::Result`, which would be `Result<Config, anyhow::Error>` with a single type parameter, and it would complain about being given two.

Using `std::result::Result` explicitly disambiguates: this is the standard library's `Result`, not `anyhow`'s alias.

The alternative is to not bring `anyhow::Result` into scope. You can use it explicitly as `anyhow::Result<T>` everywhere, leaving the bare name `Result` to mean `std::result::Result`. This is more verbose but avoids the conflict.

In practice, programs that use anyhow heavily import `anyhow::Result` for convenience and use the std `Result` only when they specifically need a custom error type, typically with the fully qualified path.

**3. When is `matches!` appropriate vs full match?**

Use `matches!` when:

- You only need to verify the variant.
- You do not need to extract data from the matched value.
- The pattern is simple enough to fit on one line.

Use full `match` when:

- You need to extract and assert on inner data.
- The patterns are complex (nested, with multiple bindings).
- You want to handle multiple variants with different actions in a test.

The trade-off:

- `matches!` is concise but limited.
- `match` is verbose but flexible.

Examples:

```rust
// matches! is fine here:
assert!(matches!(result, Err(MyError::NotFound)));

// match is better here (need to assert inner data):
match result {
    Err(MyError::ParseError { line, .. }) => assert_eq!(line, 5),
    other => panic!("unexpected: {other:?}"),
}
```

The pattern with `panic!("unexpected: ...")` for a final catch-all is a common test idiom: it provides a useful failure message that includes the actual unexpected value, which helps debugging when the test fails.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

Common patterns:

- Developers from C find the lack of return-code-based errors initially uncomfortable. They are used to checking integer return values after each call. The `?` operator alleviates this once they understand it, but the early experience often feels verbose ("why do I have to handle every error explicitly?").
- Developers from Java/C# find `Result` more natural than they expected, because it is similar to checked exceptions in concept (errors as part of the function's signature). The difference is that `Result` is structural (a type) rather than declarative (a thrown clause), which they often appreciate.
- Developers from Python/JavaScript find `Result` initially burdensome because they are used to exceptions handling errors implicitly. The mandatory handling at every call site feels intrusive. Most warm up to it within a few weeks because of the visibility (errors are documented in types, not hidden in catches).
- The `?` operator is universally appreciated once understood. The transition from "this is verbose" to "this is much cleaner than I expected" usually happens fast.

The most common reflection: "Once I understood the pattern (define specific errors with thiserror, glue them with anyhow, propagate with ?), error handling became second nature. The first week was rough, but now I cannot imagine using exceptions instead."

**2. Cross-language comparison: values vs exceptions.**

Sample answer:

"In Java, exceptions propagate automatically without any indication at the call site. This is convenient: error handling code does not clutter the happy path. But it also makes errors invisible: a method that throws `IOException` looks identical to one that does not, unless you check the throws clause. This invisibility leads to surprises when an unexpected exception escapes a method.

Rust's `Result` makes errors part of the signature. The `?` operator gives you the same convenience (errors propagate with minimal syntax), but the propagation is visible: every `?` is a potential exit point. Code review can spot 'this might fail' just by looking at the source.

The cost is that the function's signature documents every error type. Adding a new error to a function changes its signature, potentially breaking callers. In Java, you can add a runtime exception without changing the throws clause; in Rust, you would have to add a new variant or change the error type. This is more work but produces more reliable code: callers know exactly what failures to expect.

For programs where error handling is critical (servers, infrastructure, security-sensitive code), Rust's approach catches bugs that Java's would let through. For quick scripts or business logic where any error means 'log and exit,' the difference is smaller and Java's convenience may win.

I think the trade-off is worth it for serious software but not for everything. Rust's approach pairs well with strict typing more generally; the 'errors are values' style fits the language's overall philosophy."

**3. What manual error types teach vs library version.**

Sample answer:

"The manual version (Exercise 4) made me understand what `Display`, `Error`, and `From` are doing. When I write the trait implementations explicitly, I see that `Display` is the user-facing message, `Error` is a marker for the ecosystem, and `From` is what makes `?` work. Each piece serves a specific purpose; understanding them helps me debug when something goes wrong (e.g., a missing `From` impl manifests as a confusing 'cannot use ? operator' error).

The thiserror version handles all of this automatically with attributes. For real codebases, this is much better. The macro generates the same code I would have written, with no runtime cost. It also keeps the format messages near the variants, where they are easier to maintain. When I want to change a message, I edit one line; with the manual version, I would edit a match arm, possibly far from the variant definition.

What thiserror handles better than I would: consistency. If I were writing the implementations by hand, I would probably make small inconsistencies (one variant uses 'invalid value' while another uses 'bad value') without noticing. The macro keeps the format strings adjacent to the variants, so consistency is easier to maintain.

The lesson is general: writing things manually first teaches the underlying concepts; using libraries afterward removes the boilerplate without giving up understanding. I think this is the right learning order: learn the pattern, then learn the tool that automates it."

---

*End of Lab 7 Solutions*
