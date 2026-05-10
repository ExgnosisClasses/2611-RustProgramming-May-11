# Module 16: Testing and Documentation

## Module Overview

This module covers two topics that often go together in practice: testing and documentation. Both are about making code trustworthy. Tests verify that the code does what it claims; documentation tells future readers (including you, six months from now) what the code is supposed to do and how to use it. Rust treats both as first-class concerns, with tools built into the standard development experience.

The interesting parts of this module are not the syntax but the level of integration. Three ideas are central:

1. **Tests live next to the code they test.** Unit tests are typically in the same file as the code, separated by a module convention. Integration tests are in their own directory. Both are run by a single command: `cargo test`.
2. **Documentation comments are also tests.** Code examples in documentation are extracted and run as part of the test suite. This means examples in docs cannot rot: if they break, the build fails.
3. **Documentation is auto-generated.** `cargo doc` produces searchable HTML documentation from comments in the source. The Rust ecosystem's docs.rs hosts these for every published crate. Good docs are part of the culture.

By the end of this module, students will:

- Write unit tests using `#[test]` and run them with `cargo test`.
- Organize integration tests in the `tests/` directory.
- Use the various assertion macros and provide custom failure messages.
- Test for panics with `#[should_panic]` and use `Result`-returning tests for clean error checking.
- Run subsets of tests by name and use the `--ignored` flag for slow tests.
- Use the `mockall` crate to create test doubles for traits.
- Write documentation comments at the item level (`///`) and the module level (`//!`).
- Write runnable code examples in documentation that are tested automatically.
- Generate and view documentation with `cargo doc`.
- Measure code coverage with `cargo-tarpaulin`.

This module is heavy on practical patterns. Testing and documentation are both habits more than skills; the patterns introduced here will appear constantly in real Rust code.

> **A note on third-party libraries.** Sections F and J introduce the `mockall` and `cargo-tarpaulin` crates respectively. Neither is part of the standard library, but both are widely used in the Rust ecosystem. The standard library's testing infrastructure is rich enough that you can write good tests without these crates; they fill specific gaps that come up frequently in production work.

---

## A. Unit Tests with `#[test]`

A unit test is a function that exercises a small piece of code in isolation. In Rust, unit tests live in the same file as the code they test, inside a special module.

### The Standard Pattern

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_works() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn add_handles_zero() {
        assert_eq!(add(0, 5), 5);
        assert_eq!(add(5, 0), 5);
    }
}
```

Three things happen:

- The `#[cfg(test)]` attribute makes the `tests` module compile only during testing. In release builds, this entire module is omitted.
- The `use super::*` brings the parent module's items into scope, so the tests can call `add` directly.
- Each `#[test]` function is automatically discovered and run by `cargo test`.

The test module is conventionally named `tests` (or sometimes by the function being tested). It can have any name; the `#[cfg(test)]` attribute is what matters.

### Running Tests

```bash
cargo test
```

This compiles the project (including the test module), runs all tests, and reports results:

```
running 2 tests
test tests::add_handles_zero ... ok
test tests::add_works ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Each test runs in its own thread by default. Failed tests print their failure message; passed tests just print "ok."

### What `#[test]` Does

The `#[test]` attribute marks a function as a test. The function:

- Must take no arguments.
- Must return `()` (or `Result<(), E>` for `?`-using tests, covered in Section D).
- Is run only when the crate is built with `cargo test`.

Behind the scenes, `cargo test` builds the crate with the `test` cfg flag enabled. The compiler generates a small test runner that calls each `#[test]` function and reports the results.

### Why Unit Tests Live in the Source File

Most languages put tests in separate files or directories. Rust's convention of putting unit tests in the same file has practical advantages:

- **Tests have access to private items.** The test module can call private functions that external code cannot. This is essential for testing implementation details.
- **Tests are right next to the code.** When you change a function, its tests are visible. This encourages keeping tests up to date.
- **No file path bookkeeping.** You do not have to navigate between `src/foo.rs` and `tests/foo_test.rs` to update both.

The `#[cfg(test)]` module pattern keeps test code from bloating release builds. The compiled binary contains only the production code; the tests exist only when running `cargo test`.

---

## B. Integration Tests in the `tests/` Directory

Unit tests verify individual functions. Integration tests verify that the whole library works together from an external user's perspective. They live in a separate directory.

### The Setup

```
my_lib/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    ├── basic_usage.rs
    └── error_handling.rs
```

Each file in `tests/` is compiled as a separate crate that depends on your library:

```rust
// tests/basic_usage.rs
use my_lib::process;

#[test]
fn processes_valid_input() {
    let result = process("hello");
    assert_eq!(result, "HELLO");
}

#[test]
fn processes_empty_input() {
    let result = process("");
    assert_eq!(result, "");
}
```

These tests use only the library's public API. They cannot access private items because they are in a different crate. This is exactly what users of the library can do, so integration tests verify the library from a user's perspective.

`cargo test` runs both unit and integration tests automatically.

### Why a Separate Directory?

Putting integration tests outside `src/` enforces an important constraint: integration tests use only the public API. If a test passes by reaching into private internals, it might continue to pass after a refactoring that breaks the actual public interface.

The `tests/` directory separation is a structural way to ensure tests reflect what users see.

### Sharing Code Between Integration Tests

Two integration test files cannot share code directly because each is its own crate. The convention for shared test helpers is:

```
tests/
├── common/
│   └── mod.rs           (shared helpers)
├── basic_usage.rs
└── error_handling.rs
```

In `tests/common/mod.rs`:

```rust
pub fn setup_test_data() -> Vec<i32> {
    vec![1, 2, 3, 4, 5]
}
```

In each integration test file:

```rust
mod common;
use common::setup_test_data;

#[test]
fn uses_helpers() {
    let data = setup_test_data();
    // ...
}
```

The `mod common;` line declares the module. The directory structure (with `mod.rs`) prevents Cargo from treating `common.rs` as its own integration test file.

### Integration Tests for Binary Crates

Integration tests work straightforwardly for library crates. For binary crates (those with only `main.rs`), they are more limited because the binary's items are not exposed for testing.

The recommended pattern for binary crates is to keep most of the logic in a library (`lib.rs`) and have `main.rs` be a thin wrapper. Integration tests then exercise the library, and `main.rs` is mostly just argument parsing and library calls. Many production Rust binaries are organized this way.

---

## C. `assert!`, `assert_eq!`, `assert_ne!`, and Custom Messages

The standard library provides three core assertion macros. They are how tests check expectations.

### `assert!`

`assert!(condition)` panics if the condition is false:

```rust
#[test]
fn list_is_not_empty() {
    let list = vec![1, 2, 3];
    assert!(!list.is_empty());
    assert!(list.len() > 0);
}
```

The macro accepts any expression returning `bool`. If the expression is false, the test fails with a default message that includes the source file and line.

### `assert_eq!` and `assert_ne!`

`assert_eq!(left, right)` panics if the two values are not equal. `assert_ne!(left, right)` panics if they are equal:

```rust
#[test]
fn arithmetic() {
    assert_eq!(2 + 2, 4);
    assert_eq!("hello".to_uppercase(), "HELLO");
    assert_ne!(0, 1);
}
```

These macros print both values when they fail, which is much more useful than a bare `assert!(left == right)` would be:

```
assertion `left == right` failed
  left: 4
 right: 5
```

You can see exactly what was expected and what was produced. This is one of the small features that makes Rust testing pleasant.

The values must implement `Debug` for the failure message to format them. Most types do (often via `#[derive(Debug)]`). For your own types, deriving `Debug` is the easiest way to make them work with assertions.

### Custom Failure Messages

All three macros accept additional arguments for a custom message:

```rust
#[test]
fn temperature_is_reasonable() {
    let temp = read_temperature();
    assert!(
        temp > -50.0 && temp < 150.0,
        "temperature {temp} is outside the reasonable range"
    );
}

#[test]
fn user_count_matches() {
    let count = count_users();
    assert_eq!(
        count, 100,
        "expected 100 users in test database, found {count}"
    );
}
```

The custom message uses `format!`-style syntax. It appears in the failure output along with the standard message. Custom messages are particularly valuable when:

- The default failure message is unclear (e.g., `assert!(some_complex_expression)`).
- You want to show contextual information that helps diagnose the failure.
- Multiple assertions in one test could fail; the message identifies which one.

### Floating-Point Equality

A common gotcha: `assert_eq!` for floats does an exact comparison, which often fails due to floating-point rounding:

```rust
#[test]
fn float_math() {
    assert_eq!(0.1 + 0.2, 0.3);          // FAILS: 0.1 + 0.2 = 0.30000000000000004
}
```

For floats, use a tolerance:

```rust
#[test]
fn float_math() {
    let result = 0.1 + 0.2;
    assert!((result - 0.3).abs() < 1e-10);
}
```

The `approx` crate provides macros like `assert_relative_eq!` and `assert_abs_diff_eq!` for cleaner float comparisons. For most code, manual tolerance checks are sufficient.

### When to Use Which

A rough decision guide:

- **`assert_eq!`** for any equality check. The dual output makes failures clear.
- **`assert_ne!`** for inequality. Less common but useful.
- **`assert!`** for boolean conditions that are not equality (e.g., "is greater than," "contains").
- **Custom messages** when the failure output would not be self-explanatory.

For collection comparisons (vectors, hashmaps, etc.), `assert_eq!` works as long as the types implement `PartialEq` and `Debug`, which they typically do.

---

## D. `#[should_panic]` and `Result`-Returning Tests

Some tests need different shapes than "assert and pass."

### Testing for Panics

A function might be expected to panic on invalid input. To test that, use `#[should_panic]`:

```rust
fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("division by zero");
    }
    a / b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn divide_by_zero_panics() {
        divide(10, 0);
    }
}
```

The `#[should_panic]` attribute reverses the test's success condition: the test passes if the function panics, fails if it does not.

For more precision, you can require a specific panic message:

```rust
#[test]
#[should_panic(expected = "division by zero")]
fn divide_by_zero_panics_with_correct_message() {
    divide(10, 0);
}
```

The `expected` string is matched as a substring of the actual panic message. The test passes only if the panic occurs and the message contains the expected text.

### When to Use `#[should_panic]`

`#[should_panic]` is appropriate when:

- The function under test uses `panic!` for unrecoverable errors.
- You are verifying that invalid input is rejected.
- The contract of the function explicitly says "panics on X."

For functions that use `Result` for errors, `#[should_panic]` is the wrong tool. You want to assert that the function returns `Err(...)`, which is straightforward without it:

```rust
#[test]
fn divide_by_zero_returns_error() {
    let result = safe_divide(10, 0);
    assert!(result.is_err());
}
```

### `Result`-Returning Tests

Tests can return `Result<(), E>` instead of `()`. This lets you use the `?` operator inside the test for clean error handling:

```rust
#[test]
fn parses_valid_input() -> Result<(), Box<dyn std::error::Error>> {
    let input = "42";
    let parsed: i32 = input.parse()?;
    assert_eq!(parsed, 42);
    Ok(())
}
```

If the test returns `Ok(())`, it passes. If it returns `Err(...)`, it fails with the error displayed.

The advantage is that you can use `?` to short-circuit on errors, treating the test like a normal function. Without this, every fallible operation would need to be unwrapped or matched explicitly.

The boxed error type `Box<dyn std::error::Error>` accepts any error. For tests, this is usually convenient enough. If you want a specific error type, you can use it instead.

### Mixing Approaches

A real test suite usually has all three styles:

```rust
#[test]
fn computes_correctly() {                                // standard test
    assert_eq!(compute(5), 25);
}

#[test]
#[should_panic(expected = "negative input")]
fn rejects_negative() {                                  // panic test
    compute(-1);
}

#[test]
fn parses_config() -> Result<(), ConfigError> {          // result test
    let cfg = Config::from_str("key=value")?;
    assert_eq!(cfg.key, "value");
    Ok(())
}
```

Each style fits its purpose. The standard form is most common; the others are used when they fit better.

---

## E. Test Organization and Running Subsets

`cargo test` runs every test by default. For large projects, you often want to run subsets.

### Running Tests by Name

`cargo test <pattern>` runs only tests whose names contain the pattern:

```bash
cargo test add_works                # only the add_works test
cargo test add                      # tests with "add" in the name
cargo test tests::add               # tests in the tests module starting with add
```

The pattern is a substring match on the full test name (including its module path). This is the simplest way to focus on a specific test or group during development.

### Running Tests in a Module

```bash
cargo test math::tests              # all tests in the math::tests module
```

Module paths are useful when tests are organized by feature.

### Ignored Tests

Tests can be marked `#[ignore]` to skip them by default:

```rust
#[test]
fn fast_test() {
    // runs by default
}

#[test]
#[ignore]
fn slow_test() {
    // skipped by default; marked as "ignored" in the output
}
```

To run only ignored tests:

```bash
cargo test -- --ignored             # only ignored tests
cargo test -- --include-ignored     # both regular and ignored
```

`#[ignore]` is useful for tests that are slow, require external services, or are flaky. They stay in the codebase and run on demand without slowing down everyday development.

### Showing Output

By default, `cargo test` captures stdout from passing tests (only failed tests show their output). To see all output:

```bash
cargo test -- --nocapture           # show stdout from all tests
```

This is useful when debugging tests or when you want to see how a test reaches its result.

### Running Tests in Parallel

By default, tests run in parallel across multiple threads. This is fast but can cause problems for tests that share global state (databases, files, environment variables).

To run tests serially:

```bash
cargo test -- --test-threads=1      # one thread; tests run sequentially
```

For most code, parallel tests are fine. For tests that genuinely conflict (e.g., both modify the same file), either serialize them with `--test-threads=1` or use a mutex inside the tests.

### Running Tests for Specific Crates in a Workspace

In a workspace (Module 15), `cargo test -p crate_name` runs only tests in that crate:

```bash
cargo test -p my_core               # tests in the my_core crate only
cargo test -p my_core --lib         # only unit tests in my_core
cargo test -p my_core --test basic  # only the basic.rs integration test
```

These options give fine-grained control over which tests run, which matters in large projects where the full test suite takes minutes.

### Test Output Formatting

The default output is human-readable. For CI systems and tooling, JSON output is available:

```bash
cargo test -- --format json         # JSON output for tooling
```

This is useful for integrating tests into reporting dashboards, code coverage tools, and continuous integration pipelines.

---

## F. Mocking with `mockall`

For unit tests, you sometimes need to substitute fake implementations of dependencies. The `mockall` crate is the standard mocking library in Rust.

### When You Need Mocks

Consider a function that depends on an HTTP client:

```rust
trait HttpClient {
    fn get(&self, url: &str) -> Result<String, String>;
}

struct UserService<C: HttpClient> {
    client: C,
}

impl<C: HttpClient> UserService<C> {
    fn fetch_user_name(&self, id: u64) -> Result<String, String> {
        let response = self.client.get(&format!("https://api.example.com/users/{id}"))?;
        // parse response, extract name
        Ok(response.lines().next().unwrap_or("").to_string())
    }
}
```

Testing `fetch_user_name` with a real HTTP client is slow and unreliable (network failures, rate limits, etc.). You want to substitute a fake `HttpClient` that returns canned responses.

### Adding `mockall`

```toml
[dev-dependencies]
mockall = "0.12"
```

Note `dev-dependencies`, not `dependencies`: mockall is needed only for tests.

### Generating a Mock

The `#[automock]` attribute generates a mock implementation for a trait:

```rust
use mockall::automock;

#[automock]
trait HttpClient {
    fn get(&self, url: &str) -> Result<String, String>;
}
```

This generates a `MockHttpClient` struct with the same methods, plus methods to configure the mock's behavior.

### Using the Mock in a Test

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fetches_user_name_correctly() {
        let mut mock = MockHttpClient::new();

        // Configure: when get() is called with this URL, return this response
        mock.expect_get()
            .with(mockall::predicate::eq("https://api.example.com/users/42"))
            .times(1)
            .returning(|_| Ok("Alice\n".to_string()));

        let service = UserService { client: mock };
        let name = service.fetch_user_name(42).unwrap();

        assert_eq!(name, "Alice");
    }
}
```

The mock is configured before use:

- `expect_get()` says "I expect the get() method to be called."
- `with(...)` specifies what argument to expect.
- `times(1)` says "expect exactly one call."
- `returning(...)` specifies what to return.

When the test runs the service, the mock returns the configured response. After the test, mockall verifies that the expected calls were made.

### Mocking Without Modifying the Original Trait

If the trait is not yours (it is in another crate), you cannot put `#[automock]` on it. The `mock!` macro generates a mock for an external trait:

```rust
mockall::mock! {
    pub MyHttpClient {}
    impl HttpClient for MyHttpClient {
        fn get(&self, url: &str) -> Result<String, String>;
    }
}
```

This generates `MockMyHttpClient` with the same configuration methods. The trait must be declared, but `mock!` does not need to be in the same crate as the trait definition.

### Limitations

`mockall` works for trait-based code. To use it, the dependencies you want to mock must be expressed as traits, not concrete types. This is good design generally (depending on traits rather than concrete types makes code testable), but it means you sometimes need to refactor before you can mock.

For simple cases, manually writing a fake implementation is just as good as using `mockall`:

```rust
struct FakeClient {
    responses: HashMap<String, String>,
}

impl HttpClient for FakeClient {
    fn get(&self, url: &str) -> Result<String, String> {
        self.responses.get(url).cloned().ok_or_else(|| "not found".to_string())
    }
}
```

For complex tests with detailed expectations, `mockall`'s configuration is much more concise than manual fakes.

### When to Mock

Mock when:

- The dependency is slow (network, disk, database).
- The dependency is unreliable (third-party APIs).
- You want to test error paths that are hard to trigger naturally.
- You want to verify specific interactions (e.g., "the cache was checked before the database").

Avoid mocking when:

- The dependency is fast and deterministic (a pure function).
- Mocking would essentially duplicate the original implementation.
- The test would be more accurate with the real dependency.

A common antipattern is mocking everything in unit tests, then having the integration tests fail because the mocks did not reflect the real behavior. Use mocks judiciously.

---

## G. Documentation Comments: `///` and `//!`

Rust has special comment syntax for documentation. These comments are extracted by `cargo doc` to produce HTML documentation.

### Item Documentation: `///`

Three slashes mean "documentation for the next item":

```rust
/// Adds two integers and returns the sum.
///
/// # Examples
///
/// ```
/// let result = my_lib::add(2, 3);
/// assert_eq!(result, 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

The first sentence is a summary that appears in indices and search results. Following paragraphs go into the detailed documentation. Markdown formatting is fully supported: lists, headings, links, code blocks.

The convention for sections:

- `# Examples`: usage examples (these are tested; see Section H).
- `# Errors`: when this returns `Err` (for `Result`-returning functions).
- `# Panics`: when this panics.
- `# Safety`: invariants for `unsafe` functions.

These section headers are conventions, not requirements. The Rust standard library uses them consistently, and your code will look more familiar to Rust developers if you follow them.

### Module Documentation: `//!`

Double slash and exclamation point mean "documentation for the enclosing item":

```rust
//! # My Library
//!
//! This library provides utilities for processing text data.
//! It supports tokenization, normalization, and pattern matching.

pub mod tokenize;
pub mod normalize;
pub mod patterns;
```

The `//!` comments appear at the top of a file or module and document the module itself rather than the next item. They are typical at the top of `lib.rs` to provide a crate-level overview.

### Documentation Style

A few conventions:

- **Start with a one-line summary.** It appears in indices and listings.
- **Use full sentences with proper punctuation.**
- **Link to related items.** Use `[Type]` or `[function]` syntax for inline links.
- **Show examples.** They serve as both documentation and tests.
- **Note when behavior is surprising.** "This returns 0 if the input is empty" is helpful even if obvious.

Compare two versions of the same documentation:

```rust
/// gets value
pub fn get(&self, key: &str) -> Option<&Value> { /* ... */ }
```

versus:

```rust
/// Returns the value associated with `key`, or `None` if the key is not present.
///
/// The lookup is case-sensitive. For case-insensitive lookup, use [`get_ignore_case`].
///
/// # Examples
///
/// ```
/// # use my_lib::Map;
/// let mut map = Map::new();
/// map.insert("hello", 42);
/// assert_eq!(map.get("hello"), Some(&42));
/// assert_eq!(map.get("HELLO"), None);
/// ```
pub fn get(&self, key: &str) -> Option<&Value> { /* ... */ }
```

The second version takes more time to write but pays off every time someone reads it.

### Linking to Items

Within doc comments, you can link to other items in the crate or in other crates:

```rust
/// See also [`Result`] and [`Option`].
///
/// [`Result`]: std::result::Result
/// [`Option`]: std::option::Option
```

Or with the simpler intra-doc link syntax (Rust 2018 and later):

```rust
/// See also [`Result`] and [`Option`].
///
/// This is related to the [`process`](crate::processor::process) function.
```

`cargo doc` resolves these links automatically. Hovering over a linked item in the generated HTML shows its signature and brief description.

---

## H. Doc Tests: Runnable Examples in Docs

A particularly useful feature: code blocks in documentation are extracted and run as tests.

### How Doc Tests Work

```rust
/// Doubles the input.
///
/// # Examples
///
/// ```
/// let result = my_lib::double(5);
/// assert_eq!(result, 10);
/// ```
pub fn double(n: i32) -> i32 {
    n * 2
}
```

The code block under "Examples" is automatic test. When you run `cargo test`, it:

1. Extracts the code from the doc comment.
2. Compiles it as a separate test.
3. Runs it.
4. Reports success or failure alongside the regular tests.

This means examples in documentation cannot rot. If the example uses an outdated API, the test fails and the developer is forced to update both the code and the documentation.

### What Goes in a Doc Test

A doc test runs in its own scope, but with the standard library and your crate already imported. You can use the public API directly:

```rust
/// ```
/// let s = String::from("hello");        // standard library is available
/// let upper = my_lib::uppercase(&s);     // your crate is available
/// assert_eq!(upper, "HELLO");
/// ```
```

The test is wrapped in an implicit `fn main() { ... }`, so you can use `let` bindings, function calls, and `assert!` macros directly.

### Hiding Setup Code

Sometimes you need setup code that is not interesting to the reader. Lines starting with `#` are part of the test but hidden from the rendered documentation:

```rust
/// ```
/// # use my_lib::Database;
/// # let db = Database::test_instance();
/// let user = db.fetch_user("alice");
/// assert_eq!(user.name, "alice");
/// ```
```

The reader sees only the `let user = ...` and `assert_eq!` lines. The hidden setup is run but not displayed.

### Doc Tests That Should Fail to Compile

Sometimes you want to demonstrate that the API rejects certain things. The `compile_fail` annotation does this:

```rust
/// ```compile_fail
/// let s: my_lib::Sealed = my_lib::Sealed::new();
/// s.private_method();      // should not compile
/// ```
```

The test passes only if the code does NOT compile. This is useful for demonstrating type-system guarantees.

### Doc Tests That Should Panic

The `should_panic` annotation works in doc tests too:

```rust
/// ```should_panic
/// my_lib::divide(10, 0);    // panics
/// ```
```

The test passes only if the code panics.

### Skipping Doc Tests

Sometimes a code block is illustrative but not runnable. The `ignore` annotation skips it:

```rust
/// ```ignore
/// let stream = open_network_stream("example.com");      // requires network
/// stream.process();
/// ```
```

The code block is rendered as documentation but not run as a test.

### Why This Is Powerful

Doc tests serve three purposes:

- **Documentation.** The code is a worked example of how to use the API.
- **Testing.** The code verifies that the API works as documented.
- **Maintenance pressure.** Examples cannot drift out of sync with the API; if they do, tests fail.

The combination of forms is what makes Rust documentation reliable. When you read a doc example, you can trust it because it has been verified to work with the current code.

---

## I. Generating Docs with `cargo doc`

The `cargo doc` command produces HTML documentation from the comments in your code.

### Basic Usage

```bash
cargo doc                       # generate docs for your crate and dependencies
cargo doc --open                # generate and open in browser
cargo doc --no-deps             # only your crate, not dependencies
```

The output goes to `target/doc/`. The `--open` flag launches a browser to view the result. You see your crate's items, organized by module, with all the documentation comments rendered as HTML.

### What `cargo doc` Generates

For each module, struct, enum, function, and trait, the documentation includes:

- A summary line and detailed description.
- Type signatures.
- Implementation methods.
- Trait implementations.
- Source code (linked from the documentation).
- Cross-links to related items.
- A search box for finding items.

The output is a complete browsable reference for the crate. It is the same format as the standard library documentation at https://doc.rust-lang.org/std/.

### docs.rs

When you publish a crate to crates.io, the documentation is automatically built and hosted at https://docs.rs/<crate_name>. Users see your `cargo doc` output without you needing to host it.

This makes good documentation a community expectation: every crate has online documentation by default, so users assume it exists and look for it.

### Customizing the Output

Several options affect the generated documentation:

```toml
# Cargo.toml
[package.metadata.docs.rs]
all-features = true                              # build with all features for docs.rs
rustdoc-args = ["--cfg", "docsrs"]               # special cfg for docs

[features]
nightly_only = []                                 # available with --features nightly_only
```

The `[package.metadata.docs.rs]` section configures how docs.rs builds your documentation. The `rustdoc-args` line adds custom flags to the documentation build.

The `#[doc(...)]` attribute on items provides per-item customization:

```rust
#[doc(hidden)]                          // hide from documentation
fn internal_helper() { }

#[doc(alias = "size")]                  // search alias
pub fn length(&self) -> usize { /* ... */ }

#[doc = "Custom string used as documentation."]
pub fn some_function() { }
```

These are advanced and rarely needed for everyday code. `#[doc(hidden)]` is the most common, used to keep public-but-unstable items out of the documentation.

### Documentation Quality Standards

The Rust community has high expectations for documentation:

- **Every public item should have at least a one-line summary.**
- **Public functions should describe parameters, return values, and errors.**
- **Examples are expected for non-trivial APIs.**
- **The crate-level documentation should explain the overall purpose.**

You can check that everything is documented with the `#![warn(missing_docs)]` lint at the top of `lib.rs`. This makes the compiler warn about any public item without documentation.

For libraries published to crates.io, well-documented crates are adopted; poorly-documented ones are not. The investment in documentation pays off in user adoption.

---

## J. `cargo-tarpaulin` for Code Coverage

Code coverage measures which parts of your code are exercised by tests. It is a useful (but imperfect) signal of test quality.

### Installing tarpaulin

`cargo-tarpaulin` is a third-party tool, installed once per system:

```bash
cargo install cargo-tarpaulin
```

It supports Linux out of the box. On macOS and Windows, it has limitations; for cross-platform CI, an alternative like `cargo-llvm-cov` may be a better fit.

### Running Coverage

```bash
cargo tarpaulin                         # run tests with coverage analysis
cargo tarpaulin --out Html              # produce an HTML report
```

The output shows:

- The percentage of lines covered.
- Per-file coverage breakdowns.
- Which lines were and were not exercised by tests.

The HTML report is browsable, with each line of source colored green (covered) or red (not covered).

### What Coverage Means

A high coverage percentage (say, 90%) means most lines were executed during tests. It does NOT mean:

- The code is correct.
- All edge cases were tested.
- The tests assert the right things.

Coverage measures execution, not correctness. A line of code that crashes silently can still count as "covered" if the tests do not assert anything about it.

A common mistake is treating coverage as a goal in itself. Writing tests purely to increase the coverage number leads to weak tests: tests that exercise code without verifying it. The right use of coverage is to find untested areas: lines that are red in the report deserve a closer look.

### Reasonable Coverage Targets

Rough guidelines:

- **80-90% line coverage** is achievable for most libraries.
- **100% coverage is rarely worth pursuing.** The last 10% often consists of error paths, defensive checks, or `unreachable!` calls that are hard to trigger.
- **Critical code (security, financial, safety) deserves higher targets.**
- **For application code, 60-70% is often enough.** The integration tests and real usage cover what matters.

The right number depends on the project. The point of measuring coverage is to find the gaps, not to maximize a metric.

### Coverage in CI

Many projects run coverage as part of continuous integration:

```yaml
# Example GitHub Actions step (illustrative)
- name: Run coverage
  run: cargo tarpaulin --out Xml

- name: Upload coverage
  uses: codecov/codecov-action@v3
```

Services like Codecov and Coveralls track coverage over time and flag pull requests that lower it. This catches accidental regressions in test coverage.

### What NOT to Use Coverage For

A few warnings:

- **Do not block merges on tiny coverage decreases.** A 0.5% drop is usually not meaningful.
- **Do not test getters and setters just for coverage.** They are rarely worth testing if they are trivial.
- **Do not write tests purely to cover error paths.** Some error paths exist only for defense in depth and are hard to test naturally.

Coverage is a tool, not a goal. Use it to inform what to test, not as a target to optimize.

---

## Module Summary

- Unit tests live in the same file as the code they test, inside a `#[cfg(test)] mod tests` block. They have access to private items.
- Integration tests live in the `tests/` directory at the crate root. Each file is its own crate that uses only the public API.
- The assertion macros `assert!`, `assert_eq!`, `assert_ne!` cover most checks. Custom messages improve failure output.
- `#[should_panic]` tests verify that code panics. `Result`-returning tests let you use `?` for clean error handling.
- `cargo test` runs all tests; subsets can be selected by name pattern, module path, or via `--ignored` for slow tests.
- The `mockall` crate generates mock implementations of traits for unit tests. Use it when dependencies are slow or unreliable.
- Documentation comments use `///` for items and `//!` for the enclosing module. They support Markdown and intra-doc links.
- Doc tests are runnable code examples in documentation. They are extracted and tested by `cargo test`, ensuring examples stay correct.
- `cargo doc` generates HTML documentation; published crates have docs hosted at docs.rs automatically.
- `cargo-tarpaulin` measures code coverage. Use it to find untested areas, not as a metric to maximize.

## Discussion Questions

1. Rust puts unit tests in the same file as the code they test, inside a `#[cfg(test)]` module. Most languages put tests in separate files. What does the same-file convention encourage, and what does it cost?
2. Doc tests are extracted from comments and run as part of the test suite. This means examples in docs cannot drift out of sync with the API. What are the implications for the rate at which APIs can be updated, especially in libraries with many examples?
3. Code coverage measures which lines were executed by tests. A high coverage percentage does not mean the code is correct, only that it ran. Identify a specific category of bug that tests with 95% coverage might still miss, and explain why.

