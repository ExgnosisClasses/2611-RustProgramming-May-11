# Module 16: Testing and Documentation
## Lab 12 -- Unit Tests, Integration Tests, Mocking, and Documentation

> **Course:** Mastering Rust
> **Module:** 16 - Testing and Documentation
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 16: unit tests with `#[test]`, integration tests in the `tests/` directory, the assertion macros, panic tests with `#[should_panic]`, `Result`-returning tests, organizing and running subsets of tests, mocking with the `mockall` crate, documentation comments, doc tests, and generating documentation with `cargo doc`.

You will build a single Cargo project called `inventory_lib` that models a small inventory management library: items with quantities, a store that holds them, and operations to add, remove, and query stock. The project is structured as a library (not a binary) because libraries are where testing and documentation matter most. You will write the library's code, write tests at multiple levels (unit and integration), document everything with doc comments and runnable examples, and generate HTML documentation.

The focus is on judgment: when a unit test is appropriate versus an integration test, when assertions need custom messages, when mocking is the right tool versus a manual fake, and what makes documentation actually useful. Many decisions have multiple defensible answers; the lab guides you toward idiomatic Rust and asks you to compare alternatives.

By the end of this lab you will be able to:

- Write unit tests using `#[test]` and organize them in `#[cfg(test)] mod tests` blocks.
- Write integration tests in the `tests/` directory and understand what makes them different.
- Use `assert!`, `assert_eq!`, `assert_ne!`, and custom failure messages.
- Test for panics with `#[should_panic]` and write `Result`-returning tests with `?`.
- Run subsets of tests by name, mark slow tests as `#[ignore]`, and capture output with `--nocapture`.
- Use the `mockall` crate to generate mock implementations of traits.
- Write item-level (`///`) and module-level (`//!`) documentation comments.
- Include runnable code examples in documentation that are tested automatically.
- Generate HTML documentation with `cargo doc`.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** Exercise 6 uses the `mockall` crate, which is not part of the standard library. It will be added through `Cargo.toml`, and Cargo will download it on the first build. Make sure you have an internet connection when you reach Exercise 6.

> **A different project type.** Unlike most previous labs, this one creates a *library* crate rather than a binary. Libraries have their main code in `src/lib.rs` instead of `src/main.rs`. This is the right project type when the focus is on code that is meant to be tested and documented for other developers.

---

## Lab Project Setup

Create a new Cargo library project for this lab:

```bash
cd ~/rust-course/labs
cargo new inventory_lib --lib
cd inventory_lib
code .
```

The `--lib` flag tells Cargo to create a library crate (with `src/lib.rs`) instead of a binary crate (with `src/main.rs`).

Create a `lab12-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Unit Tests with `#[test]`

**Estimated time:** 10-15 minutes
**Topics covered:** test functions, the `#[cfg(test)] mod tests` pattern, access to private items, running tests

### Context

Unit tests in Rust live in the same file as the code they test, inside a special module. They have access to private items, run with `cargo test`, and are conditionally compiled (excluded from release builds). This exercise establishes the basic mechanics.

### Task 1.1 -- The starter library

Replace the contents of `src/lib.rs` with:

```rust
//! A small inventory management library.

/// Computes the value of an inventory item: quantity multiplied by unit price.
pub fn item_value(quantity: u32, unit_price: f64) -> f64 {
    quantity as f64 * unit_price
}

/// Computes the discount amount for a given purchase value.
/// Discount is 10% for purchases of $100 or more, 0% otherwise.
pub fn compute_discount(purchase_value: f64) -> f64 {
    if purchase_value >= 100.0 {
        purchase_value * 0.10
    } else {
        0.0
    }
}

/// Internal helper: rounds a value to two decimal places.
fn round_to_cents(value: f64) -> f64 {
    (value * 100.0).round() / 100.0
}
```

The library has two public functions and one private helper. Notice the `//!` comment at the top: that is module-level documentation (covered in Exercise 7).

Verify the library compiles:

```bash
cargo build
```

### Task 1.2 -- Add a basic unit test

At the bottom of `src/lib.rs`, add:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn item_value_simple() {
        let result = item_value(3, 5.0);
        assert_eq!(result, 15.0);
    }
}
```

Run the tests:

```bash
cargo test
```

Expected output:

```
running 1 test
test tests::item_value_simple ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Three things to understand:

- `#[cfg(test)]` means "compile this module only when running tests." In release builds (`cargo build --release`), this module is excluded entirely.
- The convention is to name the module `tests` (lowercase, plural). Inside it, `use super::*` brings the parent module's items (including private ones) into scope.
- Each function marked `#[test]` is automatically discovered and run by `cargo test`. The function must take no arguments and (typically) return `()`.

### Task 1.3 -- Add more tests

Expand the tests module:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn item_value_simple() {
        let result = item_value(3, 5.0);
        assert_eq!(result, 15.0);
    }

    #[test]
    fn item_value_zero_quantity() {
        let result = item_value(0, 5.0);
        assert_eq!(result, 0.0);
    }

    #[test]
    fn item_value_zero_price() {
        let result = item_value(100, 0.0);
        assert_eq!(result, 0.0);
    }

    #[test]
    fn discount_under_threshold() {
        assert_eq!(compute_discount(50.0), 0.0);
        assert_eq!(compute_discount(99.99), 0.0);
    }

    #[test]
    fn discount_at_threshold() {
        let discount = compute_discount(100.0);
        assert!((discount - 10.0).abs() < 0.001);
    }

    #[test]
    fn discount_over_threshold() {
        let discount = compute_discount(250.0);
        assert!((discount - 25.0).abs() < 0.001);
    }
}
```

Run the tests:

```bash
cargo test
```

Expected output (test order may vary):

```
running 6 tests
test tests::discount_at_threshold ... ok
test tests::discount_over_threshold ... ok
test tests::discount_under_threshold ... ok
test tests::item_value_simple ... ok
test tests::item_value_zero_price ... ok
test tests::item_value_zero_quantity ... ok

test result: ok. 6 passed; 0 failed
```

Notice that tests do not necessarily run in the order they are defined. Rust runs tests in parallel by default, so completion order is nondeterministic.

The test functions cover both the typical case (`item_value_simple`) and edge cases (zero quantity, zero price, boundary values for the discount). Good test coverage means thinking about boundaries and special cases, not just the happy path.

### Task 1.4 -- Tests can access private items

Add a test for the private `round_to_cents` function:

```rust
#[test]
fn round_to_cents_basic() {
    assert_eq!(round_to_cents(1.234), 1.23);
    assert_eq!(round_to_cents(1.236), 1.24);
    assert_eq!(round_to_cents(1.0), 1.0);
}
```

Run `cargo test`. The new test passes. This works because the tests module is inside the same file as the code; private items are accessible within the same module.

This is a key advantage of Rust's "tests in the same file" convention. Unit tests can verify implementation details, which is sometimes essential. Languages that put tests in separate files have to expose internal items publicly (loosening encapsulation) or use special "package-private" mechanisms.

### Task 1.5 -- A failing test

Add a deliberately wrong test:

```rust
#[test]
fn deliberately_wrong() {
    assert_eq!(item_value(2, 3.0), 999.0);    // wrong expected value
}
```

Run `cargo test`. Expected output:

```
running 8 tests
test tests::deliberately_wrong ... FAILED
test tests::discount_at_threshold ... ok
...

failures:

---- tests::deliberately_wrong stdout ----
thread 'tests::deliberately_wrong' panicked at 'assertion `left == right` failed
  left: 6.0
 right: 999.0'

failures:
    tests::deliberately_wrong

test result: FAILED. 7 passed; 1 failed; 0 ignored; 0 measured
```

Notice the failure output:

- The test name is shown clearly.
- The assertion's both sides are printed: `left` is what the code returned, `right` is what was expected.
- The exit code is non-zero, so CI systems and shell scripts can detect the failure.

Remove the deliberately wrong test before continuing.

### Checkpoints

1. The `#[cfg(test)]` attribute on the tests module means it is compiled only for tests. What is the practical benefit of this compared to always compiling the test code?
2. The unit tests in this exercise can access the private `round_to_cents` function because they are in the same module. Some languages put tests in separate files, requiring private items to be exposed for testing. What does each approach gain and lose?
3. The test order in the output was not the order the tests were defined. Tests run in parallel by default. What kinds of tests would break if run in parallel? What can you do about it?

---

## Exercise 2 -- Integration Tests in the tests/ Directory

**Estimated time:** 10-15 minutes
**Topics covered:** integration tests, the `tests/` directory, public-API-only testing, shared helpers

### Context

Unit tests live in the same file as the code. Integration tests live in their own directory and test the library from an external user's perspective. They cannot access private items; they use only the public API, just as a real consumer would.

### Task 2.1 -- Create an integration test

In the project root, create a `tests/` directory:

```bash
mkdir tests
```

Then create `tests/basic_usage.rs` with this content:

```rust
use inventory_lib::{item_value, compute_discount};

#[test]
fn item_value_works() {
    let value = item_value(10, 5.0);
    assert_eq!(value, 50.0);
}

#[test]
fn discount_combines_with_item_value() {
    let value = item_value(20, 10.0);        // 200.0
    let discount = compute_discount(value);   // 20.0 (10% of 200)

    assert_eq!(value, 200.0);
    assert!((discount - 20.0).abs() < 0.001);
}
```

Run all tests:

```bash
cargo test
```

Expected output:

```
running 7 tests
test tests::discount_at_threshold ... ok
...
test result: ok. 7 passed; 0 failed

     Running tests/basic_usage.rs

running 2 tests
test discount_combines_with_item_value ... ok
test item_value_works ... ok

test result: ok. 2 passed; 0 failed
```

`cargo test` runs both the unit tests (in `src/lib.rs`) and the integration tests (in `tests/`). The output is grouped by source: first the unit tests, then each integration test file separately.

### Task 2.2 -- Integration tests use only public items

Add this to `tests/basic_usage.rs`:

```rust
#[test]
fn cannot_access_private() {
    let result = inventory_lib::round_to_cents(1.234);     // ERROR
    assert_eq!(result, 1.23);
}
```

Run `cargo test`. The compiler rejects this:

```
error[E0603]: function `round_to_cents` is private
```

The function exists in the library but is not public. Integration tests are in a separate crate, so they see only what is exposed.

Remove the bad test. The integration test boundary is enforced by the compiler: you cannot bypass the public API even if you wanted to.

This is exactly what makes integration tests valuable. They verify that the library works for actual users (who only see the public API). If a refactoring breaks the public interface, integration tests fail. If it changes internal implementation while preserving the public API, integration tests still pass.

### Task 2.3 -- Add a second integration test file

Create `tests/discount_scenarios.rs`:

```rust
use inventory_lib::{item_value, compute_discount};

#[test]
fn small_purchase_no_discount() {
    let value = item_value(2, 25.0);        // 50.0
    let discount = compute_discount(value);
    assert_eq!(discount, 0.0);
}

#[test]
fn exactly_threshold_purchase() {
    let value = item_value(10, 10.0);        // 100.0
    let discount = compute_discount(value);
    assert!((discount - 10.0).abs() < 0.001);
}

#[test]
fn large_purchase_with_discount() {
    let value = item_value(50, 20.0);        // 1000.0
    let discount = compute_discount(value);
    let net = value - discount;

    assert!((discount - 100.0).abs() < 0.001);
    assert!((net - 900.0).abs() < 0.001);
}
```

Run `cargo test`. Both integration test files are compiled and run separately:

```
     Running tests/basic_usage.rs

running 2 tests
...

     Running tests/discount_scenarios.rs

running 3 tests
...
```

Each file in `tests/` is treated as its own crate. This means each has its own `main` function (generated by the test harness), its own dependencies, and its own scope.

### Task 2.4 -- Shared helpers between integration tests

Two integration test files cannot share code directly because each is its own crate. To share, you put helpers in a subdirectory:

```bash
mkdir tests/common
```

Create `tests/common/mod.rs`:

```rust
use inventory_lib::{item_value, compute_discount};

/// Helper for tests: computes the net cost of a purchase.
pub fn net_cost(quantity: u32, unit_price: f64) -> f64 {
    let value = item_value(quantity, unit_price);
    let discount = compute_discount(value);
    value - discount
}
```

Now use this from a test file. Update `tests/basic_usage.rs`:

```rust
mod common;

use inventory_lib::{item_value, compute_discount};

#[test]
fn item_value_works() {
    let value = item_value(10, 5.0);
    assert_eq!(value, 50.0);
}

#[test]
fn discount_combines_with_item_value() {
    let value = item_value(20, 10.0);
    let discount = compute_discount(value);

    assert_eq!(value, 200.0);
    assert!((discount - 20.0).abs() < 0.001);
}

#[test]
fn net_cost_helper_works() {
    let net = common::net_cost(20, 10.0);    // value 200, discount 20, net 180
    assert!((net - 180.0).abs() < 0.001);
}
```

Run `cargo test`. The new test using the helper passes.

The `mod common;` declaration tells the compiler about the helper module. The directory structure (`common/mod.rs` rather than `common.rs`) is what prevents Cargo from treating common as its own integration test file. If you used `tests/common.rs`, Cargo would try to compile and run it as a test, which is not what you want for helper code.

### Task 2.5 -- Looking at the test summary

When `cargo test` runs everything, the output structure tells you about each test target:

```
   Compiling inventory_lib v0.1.0
    Finished test [unoptimized + debuginfo] target(s) in ...
     Running unittests src/lib.rs

running 7 tests
[unit tests]

     Running tests/basic_usage.rs

running 3 tests
[basic_usage tests]

     Running tests/discount_scenarios.rs

running 3 tests
[discount_scenarios tests]

   Doc-tests inventory_lib

running 0 tests
```

Three things are tested:

- **`unittests src/lib.rs`**: the unit tests in the library.
- **`tests/...`**: each integration test file separately.
- **`Doc-tests inventory_lib`**: doc tests from doc comments (Exercise 8 will add these).

`cargo test` runs all three by default. You can run only one category by name:

```bash
cargo test --lib              # only unit tests
cargo test --tests            # only integration tests
cargo test --doc              # only doc tests
```

### Checkpoints

1. Integration tests can access only the public API. Unit tests can access private items. When would you write each kind of test for the same feature?
2. Why are integration test files compiled as separate crates rather than all together as one crate?
3. The shared helper module uses the path `tests/common/mod.rs` rather than `tests/common.rs`. What goes wrong if you use the second path?

---

## Exercise 3 -- The Assertion Macros and Custom Messages

**Estimated time:** 10 minutes
**Topics covered:** `assert!`, `assert_eq!`, `assert_ne!`, custom failure messages, floating-point comparisons

### Context

Tests verify expectations by asserting. The standard library provides three macros: `assert!` for boolean conditions, `assert_eq!` for equality, `assert_ne!` for inequality. All three accept custom messages that appear in failure output.

### Task 3.1 -- The three basic assertions

In `src/lib.rs`, replace your test module with this expanded version:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn assertion_with_assert() {
        let value = item_value(3, 5.0);
        assert!(value > 0.0);
        assert!(value < 100.0);
        assert!(value == 15.0);
    }

    #[test]
    fn assertion_with_assert_eq() {
        let value = item_value(3, 5.0);
        assert_eq!(value, 15.0);

        let discount = compute_discount(200.0);
        assert_eq!(discount, 20.0);
    }

    #[test]
    fn assertion_with_assert_ne() {
        let value = item_value(3, 5.0);
        assert_ne!(value, 0.0);
        assert_ne!(value, 999.0);
    }
}
```

Run `cargo test --lib`. All three tests pass.

The three macros serve different purposes:

- **`assert!(expr)`**: succeeds if `expr` is true. Generic; works with any boolean condition.
- **`assert_eq!(a, b)`**: succeeds if `a == b`. On failure, prints both values.
- **`assert_ne!(a, b)`**: succeeds if `a != b`. On failure, prints both values.

For equality checks, prefer `assert_eq!` over `assert!(a == b)`. The error message is much more useful: with `assert!`, you see only "assertion failed"; with `assert_eq!`, you see the actual and expected values.

### Task 3.2 -- The failure messages

Add a deliberately failing test:

```rust
#[test]
fn compare_failures() {
    let value = item_value(3, 5.0);

    // assert! failure: just tells you what failed
    assert!(value == 100.0);
}
```

Run `cargo test --lib compare_failures`. Expected output:

```
failures:

---- tests::compare_failures stdout ----
thread 'tests::compare_failures' panicked at 'assertion failed: value == 100.0'
```

The message tells you only that the assertion failed. It does not tell you what `value` actually was.

Now change to `assert_eq!`:

```rust
#[test]
fn compare_failures() {
    let value = item_value(3, 5.0);

    // assert_eq! failure: shows both values
    assert_eq!(value, 100.0);
}
```

Run the test. Expected output:

```
failures:

---- tests::compare_failures stdout ----
thread 'tests::compare_failures' panicked at 'assertion `left == right` failed
  left: 15.0
 right: 100.0'
```

Now you see both values: the actual (`15.0`) and the expected (`100.0`). When diagnosing the failure, this is invaluable.

Remove the failing test before continuing.

### Task 3.3 -- Custom failure messages

All three assertion macros accept additional arguments for a custom message:

```rust
#[test]
fn assertions_with_custom_messages() {
    let value = item_value(5, 4.0);

    assert_eq!(value, 20.0, "expected 5 units at $4 = $20, got {value}");

    let discount = compute_discount(150.0);
    assert!(
        discount > 0.0,
        "discount should be positive for purchases over $100, got {discount}"
    );

    let zero_value = item_value(0, 100.0);
    assert_ne!(
        zero_value, 1.0,
        "zero quantity should produce zero value, not {zero_value}"
    );
}
```

Run the test. It passes (we expect it to). To see what the message looks like on failure, change `value` to expect something wrong:

```rust
assert_eq!(value, 100.0, "expected 5 units at $4 = $20, got {value}");
```

Run the test. Expected output:

```
thread 'tests::assertions_with_custom_messages' panicked at 'assertion `left == right` failed: expected 5 units at $4 = $20, got 20
  left: 20.0
 right: 100.0'
```

The custom message is appended to the default failure message. It gives the reader context that the bare value comparison cannot.

Custom messages use `format!`-style syntax with `{}` placeholders. You can include any variables or expressions visible at the call site.

Restore the correct expected value (`20.0`) before continuing.

### Task 3.4 -- When custom messages are valuable

Three patterns where custom messages help:

**Multiple assertions in one test:** without messages, a failure tells you which assertion failed only by line number. With messages, you know what was being checked:

```rust
#[test]
fn multiple_assertions() {
    let value = item_value(2, 50.0);
    let discount = compute_discount(value);

    assert_eq!(value, 100.0, "item value calculation");
    assert!((discount - 10.0).abs() < 0.001, "discount at exactly threshold");
}
```

**Complex conditions:** `assert!(complex_predicate(x, y, z))` tells you nothing on failure. A message describes what was expected:

```rust
assert!(
    is_valid_order(quantity, price, currency),
    "order should be valid: qty={quantity}, price={price}, currency={currency:?}"
);
```

**Boundary conditions:** when testing boundaries, the message can document the rule being checked:

```rust
let discount = compute_discount(99.99);
assert_eq!(
    discount, 0.0,
    "discount should be zero just below the $100 threshold"
);
```

The general rule: write a message whenever the bare assertion would not be self-explanatory at the time of failure. If the test name plus the line number tells you everything you need, you can skip the message.

### Task 3.5 -- Floating-point equality

Floating-point arithmetic can produce surprising results. Try:

```rust
#[test]
fn float_equality_gotcha() {
    let result = 0.1 + 0.2;
    assert_eq!(result, 0.3);    // PROBABLY FAILS
}
```

Run the test. Expected output:

```
thread 'tests::float_equality_gotcha' panicked at 'assertion `left == right` failed
  left: 0.30000000000000004
 right: 0.3'
```

The sum is not exactly `0.3` due to floating-point representation. This is not a bug in Rust; it is how floating-point works in nearly every language.

The fix is to use a tolerance:

```rust
#[test]
fn float_equality_with_tolerance() {
    let result = 0.1 + 0.2;
    assert!(
        (result - 0.3).abs() < 1e-10,
        "expected ~0.3, got {result}"
    );
}
```

The `(result - 0.3).abs() < 1e-10` pattern is common in tests. The constant `1e-10` is a small tolerance; floats are equal if their difference is less than this.

For more elaborate float tests, the `approx` crate provides macros like `assert_relative_eq!` and `assert_abs_diff_eq!`. For simple cases, manual tolerance is fine.

### Checkpoints

1. `assert_eq!(a, b)` and `assert!(a == b)` are functionally equivalent (both fail if `a != b`). Why is `assert_eq!` preferred?
2. Task 3.5 showed that `0.1 + 0.2 != 0.3` in floating-point. Why does this happen, and what other operations might produce surprising results?
3. The custom messages in Task 3.4 used `format!`-style placeholders. The variables came from the test's scope. Could the custom message format strings access variables that did not exist at the call site? What would happen?

---

## Exercise 4 -- `#[should_panic]` and Result-Returning Tests

**Estimated time:** 10-15 minutes
**Topics covered:** testing for panics, expected panic messages, tests that use `?`

### Context

Some tests need different shapes. A test that verifies a function panics on bad input uses `#[should_panic]`. A test that uses `?` to handle errors returns `Result`. This exercise covers both.

### Task 4.1 -- Add a function that can panic

Add this to `src/lib.rs`:

```rust
/// Divides total cost by number of items.
/// Panics if `items` is 0 (cannot divide by zero).
pub fn cost_per_item(total_cost: f64, items: u32) -> f64 {
    if items == 0 {
        panic!("cannot compute cost per item for 0 items");
    }
    total_cost / items as f64
}
```

Test the happy path:

```rust
#[test]
fn cost_per_item_simple() {
    let result = cost_per_item(100.0, 5);
    assert_eq!(result, 20.0);
}
```

Run `cargo test --lib cost_per_item_simple`. It passes.

### Task 4.2 -- Test for the panic

To verify that the function panics on bad input, use `#[should_panic]`:

```rust
#[test]
#[should_panic]
fn cost_per_item_zero_panics() {
    cost_per_item(100.0, 0);
}
```

Run `cargo test --lib cost_per_item_zero_panics`. Expected output:

```
running 1 test
test tests::cost_per_item_zero_panics ... ok

test result: ok. 1 passed
```

The test passes because the function panicked (as expected). With `#[should_panic]`, the success condition is inverted: the test passes if the code panics, fails if it does not.

If you forgot the `#[should_panic]` attribute, the panic would cause the test to fail. The attribute is what signals "this panic is the expected behavior."

### Task 4.3 -- Specify the expected panic message

`#[should_panic]` matches any panic. For more precision, you can require a specific message:

```rust
#[test]
#[should_panic(expected = "cannot compute cost per item for 0 items")]
fn cost_per_item_zero_panics_with_message() {
    cost_per_item(100.0, 0);
}
```

Run the test. It passes because the actual panic message contains the expected substring.

The `expected = "..."` text is matched as a substring. The test passes only if the panic occurs AND the message contains the expected text. This catches a class of bug where the function panics, but for the wrong reason.

For example, suppose someone refactored the function to panic on division by negative numbers (changing the rule). The function might still panic for `cost_per_item(100.0, 0)`, but the message would be different ("cannot use negative count" or similar). With a bare `#[should_panic]`, the test would still pass (any panic counts). With the expected message, the test would fail, correctly identifying that the panic semantics changed.

### Task 4.4 -- When `#[should_panic]` is appropriate

`#[should_panic]` is appropriate when:

- The function uses `panic!` for invariant violations (bugs in the caller's input).
- The contract of the function explicitly says "panics on X."
- You want to verify that the panic is the expected one.

It is NOT appropriate when:

- The function returns `Result` for errors. You assert on the `Err` variant directly, not via `should_panic`.
- The function might panic in many places. You cannot tell which panic the test caught.
- Testing in production-like conditions where panics should be avoided altogether.

For functions that use `Result` for errors, the next task shows the right pattern.

### Task 4.5 -- A function that returns Result

Add this to `src/lib.rs`:

```rust
/// Parses a string like "qty:price" into a (quantity, price) pair.
/// Returns Err if the format is invalid or values cannot be parsed.
pub fn parse_inventory_entry(input: &str) -> Result<(u32, f64), String> {
    let parts: Vec<&str> = input.split(':').collect();

    if parts.len() != 2 {
        return Err(format!("expected 'qty:price', got '{input}'"));
    }

    let qty: u32 = parts[0].trim().parse()
        .map_err(|e| format!("invalid quantity '{}': {e}", parts[0]))?;

    let price: f64 = parts[1].trim().parse()
        .map_err(|e| format!("invalid price '{}': {e}", parts[1]))?;

    Ok((qty, price))
}
```

This function returns `Result<(u32, f64), String>`. The `?` operator inside the function propagates errors from `parse()`.

### Task 4.6 -- Result-returning tests

Test the success case using a `Result`-returning test:

```rust
#[test]
fn parse_inventory_entry_success() -> Result<(), String> {
    let (qty, price) = parse_inventory_entry("5:9.99")?;
    assert_eq!(qty, 5);
    assert!((price - 9.99).abs() < 0.001);
    Ok(())
}
```

Run `cargo test --lib parse_inventory_entry_success`. It passes.

The test function returns `Result<(), String>`. Inside, the `?` operator lets you use the parse function naturally; if it returns `Err`, the test returns the `Err` (which makes the test fail with that error message). On success, the test continues, asserts, and returns `Ok(())`.

This pattern is cleaner than:

```rust
let result = parse_inventory_entry("5:9.99");
let (qty, price) = result.expect("should parse");
```

The `Result`-returning form lets `?` do the work. For tests with multiple fallible operations, the simplification is significant.

### Task 4.7 -- Testing error cases

For error cases, you typically do not use `?` (you want the Err). Instead, match on the result:

```rust
#[test]
fn parse_invalid_format() {
    let result = parse_inventory_entry("no colon here");
    assert!(result.is_err());
}

#[test]
fn parse_invalid_quantity() {
    let result = parse_inventory_entry("abc:5.99");
    match result {
        Err(msg) => assert!(msg.contains("invalid quantity")),
        Ok(_) => panic!("expected error for non-numeric quantity"),
    }
}

#[test]
fn parse_invalid_price() {
    let result = parse_inventory_entry("5:not_a_number");
    assert!(matches!(result, Err(msg) if msg.contains("invalid price")));
}
```

Run `cargo test --lib parse_invalid`. All three tests pass.

Three different styles for error testing:

- **`result.is_err()`**: simplest, just checks that an error occurred.
- **`match`**: extracts the error data and asserts specific things about it.
- **`matches!`**: a one-liner pattern match with a guard.

The `matches!` macro is convenient for simple "matches this variant" checks. The `match` form is more verbose but gives you access to the inner data for detailed assertions.

### Task 4.8 -- Mixing styles in one test suite

A real test suite typically has all three forms: regular tests, `#[should_panic]` tests, and `Result`-returning tests:

```rust
#[test]
fn normal_test() {
    let value = item_value(3, 5.0);
    assert_eq!(value, 15.0);
}

#[test]
#[should_panic(expected = "cannot compute cost per item for 0 items")]
fn panic_test() {
    cost_per_item(100.0, 0);
}

#[test]
fn result_test() -> Result<(), String> {
    let (qty, _) = parse_inventory_entry("10:5.99")?;
    assert_eq!(qty, 10);
    Ok(())
}
```

Each style is appropriate for what it tests. Use whatever fits.

### Checkpoints

1. `#[should_panic]` accepts an `expected = "..."` argument for matching the panic message. What kind of bug does specifying the expected message catch that a bare `#[should_panic]` would miss?
2. The `Result`-returning test in Task 4.6 used `?` for cleaner error handling. Why does `?` only work in functions that return `Result`?
3. When testing a function that returns `Result`, you have three styles: `is_err()`, `match`, and `matches!`. When would you choose each? What does each give up?

---

## Exercise 5 -- Running Subsets and Test Organization

**Estimated time:** 10 minutes
**Topics covered:** running tests by name, `#[ignore]` for slow tests, parallelism, output capture

### Context

`cargo test` runs every test by default. For a real codebase, you often want to run only specific tests during development. This exercise covers the options.

### Task 5.1 -- Run tests by name pattern

`cargo test <name>` runs only tests whose name matches the pattern (as a substring):

```bash
cargo test --lib discount             # tests with "discount" in the name
cargo test --lib item_value           # tests with "item_value"
cargo test --lib parse                # tests starting with "parse"
```

Try each. Expected output for `cargo test --lib discount`:

```
running 4 tests
test tests::compute_discount_under_threshold ... ok
test tests::discount_at_threshold ... ok
test tests::discount_combines_with_item_value ... ok
test tests::discount_over_threshold ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 11 filtered out
```

Note the `11 filtered out` at the end: 11 other tests were skipped because they did not match the pattern.

The pattern matches the full test name (including the module path: `tests::discount_under_threshold` would match patterns "discount", "tests", and "tests::discount").

### Task 5.2 -- Mark a slow test as ignored

Some tests are slow (large datasets, external resources, etc.) and should not run on every build. Use `#[ignore]`:

```rust
#[test]
#[ignore]
fn expensive_computation_test() {
    // Simulate expensive work:
    let mut total = 0u64;
    for i in 1..=10_000_000 {
        total = total.wrapping_add(i);
    }
    assert!(total > 0);
}
```

Run `cargo test --lib`. The new test does not run; it appears in the summary as ignored:

```
running 15 tests
...

test result: ok. 14 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out
```

To run the ignored test explicitly:

```bash
cargo test --lib -- --ignored          # only ignored tests
cargo test --lib -- --include-ignored  # both regular AND ignored
```

The `--` separates `cargo test` options from arguments passed to the test binary. The `--ignored` flag tells the test binary to run only ignored tests.

`#[ignore]` is useful for:

- Tests that hit external services (database, network).
- Tests with very long runtimes.
- Tests that depend on system state (specific files, environment).
- Quarantined flaky tests under investigation.

The tests stay in the codebase and run on demand without slowing down everyday work.

### Task 5.3 -- Output capture

By default, `cargo test` captures stdout from passing tests (you see output only on failure). To see all output:

```rust
#[test]
fn test_with_output() {
    println!("this is from the test");
    let value = item_value(3, 5.0);
    println!("computed value: {value}");
    assert_eq!(value, 15.0);
}
```

Run normally:

```bash
cargo test --lib test_with_output
```

You see only "ok"; the println output is hidden.

Now with output:

```bash
cargo test --lib test_with_output -- --nocapture
```

Expected output:

```
running 1 test
this is from the test
computed value: 15
test tests::test_with_output ... ok
```

The `--nocapture` flag (passed to the test binary) makes test output visible. This is useful when debugging: you can add temporary `println!` calls to understand what is happening without removing the assertions.

### Task 5.4 -- Test threads

By default, tests run in parallel. This is fast but can cause problems if tests share state (e.g., a global counter, a single file). To run sequentially:

```bash
cargo test --lib -- --test-threads=1
```

This forces one thread, meaning tests run one at a time in deterministic order.

When is single-threaded testing needed?

- **Shared global state.** A test that sets a global value, then another that reads it.
- **Filesystem tests.** Multiple tests reading or writing the same file.
- **Environment variables.** Tests that set env vars; concurrent tests might see each other's values.

For most code, parallel tests are fine. When they cause problems, you have two options:

- Run sequentially with `--test-threads=1` (slower).
- Restructure tests to not share state (better).

Restructuring is usually preferable. Tests that depend on global state are fragile in many ways beyond parallelism (test order, test isolation, etc.).

### Task 5.5 -- The full test command reference

For reference, common cargo test invocations:

```bash
cargo test                              # everything
cargo test --lib                        # only unit tests
cargo test --tests                      # only integration tests
cargo test --doc                        # only doc tests

cargo test some_pattern                 # tests matching the pattern
cargo test --lib some_pattern           # unit tests matching

cargo test -- --ignored                 # only ignored tests
cargo test -- --include-ignored         # all tests including ignored

cargo test -- --nocapture               # show test output
cargo test -- --test-threads=1          # serial execution

cargo test -- --list                    # list all tests without running

cargo test --release                    # tests against release build
```

The combinations cover most needs. For unfamiliar scenarios, `cargo test --help` and `cargo test -- --help` document everything.

### Checkpoints

1. Tests run in parallel by default. What is the performance benefit? What is the cost?
2. `#[ignore]` marks tests that do not run by default. What is the practical difference between marking a test `#[ignore]` and just commenting it out?
3. The `cargo test discount` pattern matches all tests with "discount" in the name. What would `cargo test ::` match? (Hint: think about what names contain `::`.)

---

## Exercise 6 -- Mocking with mockall

**Estimated time:** 15 minutes
**Topics covered:** the `mockall` crate, `#[automock]`, configuring mock behavior

### Context

For unit tests, you sometimes need to substitute fake implementations of dependencies. The `mockall` crate generates mock implementations of traits. This exercise introduces the basic usage.

### Task 6.1 -- Set up mockall

Open `Cargo.toml`. Add `mockall` as a dev-dependency:

```toml
[dev-dependencies]
mockall = "0.12"
```

The `dev-dependencies` section is for dependencies used only during development and testing. `mockall` is not needed for the library's production code; only the tests use it.

Save and run `cargo build`. Cargo downloads `mockall`.

### Task 6.2 -- Define a trait to mock

Add this to `src/lib.rs`:

```rust
use mockall::automock;

/// A trait for looking up unit prices. Could be backed by a database, file, etc.
#[automock]
pub trait PriceLookup {
    fn unit_price(&self, item_code: &str) -> Option<f64>;
}

/// Computes the total value of an order using a price lookup.
pub fn order_total(
    items: &[(String, u32)],
    prices: &dyn PriceLookup,
) -> Result<f64, String> {
    let mut total = 0.0;
    for (code, quantity) in items {
        let price = prices.unit_price(code)
            .ok_or_else(|| format!("unknown item: {code}"))?;
        total += price * *quantity as f64;
    }
    Ok(total)
}
```

The `#[automock]` attribute generates a `MockPriceLookup` struct that implements the trait. The struct has methods to configure its behavior (what to return for each call).

`order_total` takes a slice of (code, quantity) pairs and a price lookup. For each item, it looks up the price, multiplies by quantity, and sums. If a lookup fails (unknown item), the function returns an error.

The function takes `&dyn PriceLookup`, a trait object. This makes it work with any implementation of the trait, including the mock generated by `mockall`.

### Task 6.3 -- Use the mock in a test

Add a test that uses the mock:

```rust
#[test]
fn order_total_with_mock() {
    let mut mock = MockPriceLookup::new();

    // Configure the mock: when called with "WIDGET", return Some(9.99).
    mock.expect_unit_price()
        .with(mockall::predicate::eq("WIDGET"))
        .times(1)
        .returning(|_| Some(9.99));

    // Same for "GIZMO":
    mock.expect_unit_price()
        .with(mockall::predicate::eq("GIZMO"))
        .times(1)
        .returning(|_| Some(4.50));

    let items = vec![
        (String::from("WIDGET"), 3),
        (String::from("GIZMO"), 5),
    ];

    let result = order_total(&items, &mock).unwrap();
    let expected = 3.0 * 9.99 + 5.0 * 4.50;     // 29.97 + 22.50 = 52.47
    assert!((result - expected).abs() < 0.001);
}
```

Run `cargo test --lib order_total_with_mock`. Expected output:

```
test tests::order_total_with_mock ... ok
```

What the mock setup does:

- `MockPriceLookup::new()` creates a fresh mock.
- `expect_unit_price()` says "I expect the `unit_price` method to be called."
- `.with(predicate::eq("WIDGET"))` constrains: when the argument matches "WIDGET."
- `.times(1)` says: exactly once.
- `.returning(|_| Some(9.99))` says: return this value.

When the test runs, the mock checks each call. If the calls match the expectations, the test passes. If something unexpected happens (wrong argument, wrong call count), the test fails.

### Task 6.4 -- Test the error path

Mocks make it easy to test failure cases:

```rust
#[test]
fn order_total_unknown_item() {
    let mut mock = MockPriceLookup::new();

    mock.expect_unit_price()
        .returning(|_| None);  // simulate "no price found" for any input

    let items = vec![(String::from("MYSTERY"), 1)];

    let result = order_total(&items, &mock);
    assert!(result.is_err());
    let err = result.unwrap_err();
    assert!(err.contains("unknown item: MYSTERY"));
}
```

Run the test. It passes: the mock always returns `None`, the function detects the missing price, returns an `Err`, and the test verifies the error message.

Without mocking, testing this error case would require a real `PriceLookup` implementation that happened to not know "MYSTERY." That is harder to set up reliably. With mocks, you configure the behavior directly.

This is a key advantage of mocking: you can simulate any behavior, including error conditions that are difficult to trigger naturally.

### Task 6.5 -- Verifying call counts

The mock can verify how many times a method is called:

```rust
#[test]
fn order_total_call_counts() {
    let mut mock = MockPriceLookup::new();

    // Setup: expect WIDGET to be looked up exactly twice (it appears twice in the order).
    mock.expect_unit_price()
        .with(mockall::predicate::eq("WIDGET"))
        .times(2)
        .returning(|_| Some(5.0));

    let items = vec![
        (String::from("WIDGET"), 1),
        (String::from("WIDGET"), 3),
    ];

    let result = order_total(&items, &mock).unwrap();
    assert_eq!(result, 5.0 + 15.0);    // 20.0
}
```

The mock asserts that `unit_price` is called exactly twice with "WIDGET." If the function were buggy and called it once, three times, or with the wrong argument, the test would fail.

This is the verification side of mocks: not just providing canned responses, but checking that the code interacts with the dependency correctly.

### Task 6.6 -- A manual fake as an alternative

For simple cases, a manual fake can be cleaner than a mock:

```rust
struct FakeLookup {
    prices: std::collections::HashMap<String, f64>,
}

impl FakeLookup {
    fn new() -> FakeLookup {
        FakeLookup { prices: std::collections::HashMap::new() }
    }

    fn with_price(mut self, code: &str, price: f64) -> Self {
        self.prices.insert(code.to_string(), price);
        self
    }
}

impl PriceLookup for FakeLookup {
    fn unit_price(&self, code: &str) -> Option<f64> {
        self.prices.get(code).copied()
    }
}

#[test]
fn order_total_with_manual_fake() {
    let lookup = FakeLookup::new()
        .with_price("WIDGET", 9.99)
        .with_price("GIZMO", 4.50);

    let items = vec![
        (String::from("WIDGET"), 3),
        (String::from("GIZMO"), 5),
    ];

    let result = order_total(&items, &lookup).unwrap();
    let expected = 3.0 * 9.99 + 5.0 * 4.50;
    assert!((result - expected).abs() < 0.001);
}
```

The manual fake is sometimes clearer for simple cases. It does not verify call counts (just simulates the behavior), but the setup is straightforward.

When to use manual fakes vs mockall:

- **Manual fakes** are simpler when the behavior is straightforward and you do not need call verification.
- **Mockall** is better when you need detailed expectations (specific arguments, exact counts, ordered calls).

For most tests, either works. The choice is stylistic.

### Checkpoints

1. The `mockall` crate is added as a `dev-dependency` rather than a regular `dependency`. What is the practical difference, and why does it matter?
2. The `order_total` function takes `&dyn PriceLookup` (a trait object). Could it have taken `impl PriceLookup` instead? What changes about how you would use it with the mock?
3. Mocks let you simulate specific behaviors, including error conditions. What is the risk of relying too heavily on mocks? When do mocks become a liability rather than a help?

---

## Exercise 7 -- Documentation Comments

**Estimated time:** 10 minutes
**Topics covered:** `///` for items, `//!` for modules, Markdown formatting, conventional section headers

### Context

Rust has special comment syntax for documentation. Comments starting with `///` document the item that follows; comments starting with `//!` document the enclosing item (file or module). This exercise adds documentation to your library.

### Task 7.1 -- Item-level documentation

You may have already noticed the `//!` at the top of `src/lib.rs`. Now add `///` comments to each public function. Update your `src/lib.rs`:

```rust
//! A small inventory management library.
//!
//! This library provides functions for working with inventory items:
//! computing values, applying discounts, and parsing entries.

/// Computes the value of an inventory item.
///
/// Multiplies the quantity by the unit price.
///
/// # Examples
///
/// ```
/// let value = inventory_lib::item_value(3, 5.0);
/// assert_eq!(value, 15.0);
/// ```
pub fn item_value(quantity: u32, unit_price: f64) -> f64 {
    quantity as f64 * unit_price
}

/// Computes the discount amount for a given purchase value.
///
/// The discount is 10% for purchases of $100 or more, 0% otherwise.
///
/// # Examples
///
/// ```
/// let small = inventory_lib::compute_discount(50.0);
/// assert_eq!(small, 0.0);
///
/// let large = inventory_lib::compute_discount(200.0);
/// assert!((large - 20.0).abs() < 0.001);
/// ```
pub fn compute_discount(purchase_value: f64) -> f64 {
    if purchase_value >= 100.0 {
        purchase_value * 0.10
    } else {
        0.0
    }
}

/// Divides a total cost by a number of items to get the per-item cost.
///
/// # Panics
///
/// Panics if `items` is 0.
///
/// # Examples
///
/// ```
/// let per_item = inventory_lib::cost_per_item(100.0, 4);
/// assert_eq!(per_item, 25.0);
/// ```
pub fn cost_per_item(total_cost: f64, items: u32) -> f64 {
    if items == 0 {
        panic!("cannot compute cost per item for 0 items");
    }
    total_cost / items as f64
}

/// Parses a string of the form "quantity:price" into a tuple.
///
/// # Errors
///
/// Returns an `Err` if:
/// - The input does not contain exactly one `:` separator.
/// - The quantity cannot be parsed as `u32`.
/// - The price cannot be parsed as `f64`.
///
/// # Examples
///
/// ```
/// let (qty, price) = inventory_lib::parse_inventory_entry("5:9.99").unwrap();
/// assert_eq!(qty, 5);
/// assert!((price - 9.99).abs() < 0.001);
/// ```
pub fn parse_inventory_entry(input: &str) -> Result<(u32, f64), String> {
    let parts: Vec<&str> = input.split(':').collect();

    if parts.len() != 2 {
        return Err(format!("expected 'qty:price', got '{input}'"));
    }

    let qty: u32 = parts[0].trim().parse()
        .map_err(|e| format!("invalid quantity '{}': {e}", parts[0]))?;

    let price: f64 = parts[1].trim().parse()
        .map_err(|e| format!("invalid price '{}': {e}", parts[1]))?;

    Ok((qty, price))
}
```

Notice the conventional sections in the doc comments:

- **`# Examples`**: usage examples (these are also tested, as you will see in Exercise 8).
- **`# Errors`**: when `Result`-returning functions return `Err`.
- **`# Panics`**: when functions panic.

These section headers are conventions established by the standard library. Following them makes your documentation feel familiar to other Rust developers.

The first sentence of each doc comment is the summary that appears in indexes and search results. Keep it short and complete.

### Task 7.2 -- Module-level documentation

The `//!` at the top of the file is module-level documentation. It describes what the crate or module does as a whole, rather than any specific item.

The module-level doc at the top of `src/lib.rs` should give a high-level description, possibly with a quick-start example:

```rust
//! A small inventory management library.
//!
//! This library provides functions for working with inventory items:
//! computing values, applying discounts, and parsing entries.
//!
//! # Quick Start
//!
//! ```
//! use inventory_lib::{item_value, compute_discount};
//!
//! let value = item_value(5, 20.0);            // 100.0
//! let discount = compute_discount(value);      // 10.0 (10% of 100)
//! let net = value - discount;                  // 90.0
//!
//! assert_eq!(value, 100.0);
//! assert!((discount - 10.0).abs() < 0.001);
//! assert!((net - 90.0).abs() < 0.001);
//! ```
```

Update your `src/lib.rs` with this enhanced module-level doc.

The module-level doc is what users see first when they look at your crate on docs.rs (Exercise 9). It should orient them: what does this crate do, what are the main entry points, how would I use it?

### Task 7.3 -- Documentation style

A few stylistic conventions:

- **Start with a one-line summary.** It appears in indexes; keep it focused.
- **Use full sentences.** "Computes the value of an item" rather than "computes value."
- **Be specific about behavior.** "Returns 0 if the slice is empty" is more useful than vague language.
- **Mention edge cases.** Zero, negative numbers, empty inputs, etc.
- **Use Markdown formatting.** Code blocks, lists, headers all work.

Compare two versions of a doc comment:

**Less useful:**

```rust
/// Compute the price.
pub fn compute_discount(purchase_value: f64) -> f64 { ... }
```

**More useful:**

```rust
/// Computes the discount amount for a given purchase value.
///
/// The discount is 10% for purchases of $100 or more, 0% otherwise.
/// The threshold is inclusive: a purchase of exactly $100 receives the discount.
///
/// # Examples
///
/// ```
/// // Below threshold:
/// assert_eq!(inventory_lib::compute_discount(50.0), 0.0);
///
/// // At threshold:
/// assert_eq!(inventory_lib::compute_discount(100.0), 10.0);
///
/// // Above threshold:
/// assert!((inventory_lib::compute_discount(200.0) - 20.0).abs() < 0.001);
/// ```
pub fn compute_discount(purchase_value: f64) -> f64 { ... }
```

The longer version tells the reader exactly what the function does, including the edge case (is the threshold inclusive or exclusive?). The examples cover the three cases.

Time spent on documentation pays off every time someone reads it. For library code, this happens often: every user of your crate, your colleagues, and future you.

### Task 7.4 -- Linking to other items

You can link to other items in your doc comments using `[name]` syntax. Add a link to compute_discount from the item_value doc:

```rust
/// Computes the value of an inventory item.
///
/// Multiplies the quantity by the unit price. To compute a discount on the result,
/// pass it to [`compute_discount`].
///
/// # Examples
///
/// ```
/// let value = inventory_lib::item_value(3, 5.0);
/// assert_eq!(value, 15.0);
/// ```
pub fn item_value(quantity: u32, unit_price: f64) -> f64 {
    quantity as f64 * unit_price
}
```

The `[`compute_discount`]` syntax creates a link to the `compute_discount` function in the generated HTML docs. The compiler resolves these links automatically.

You can link to functions, types, modules, and constants. Common patterns:

- `[`function_name`]` for functions.
- `[`Type`]` for types.
- `[`Type::method`]` for methods.
- `[`crate::module::item`]` for items in other modules.

Hover over a link in the generated docs to see a tooltip with the linked item's signature.

### Checkpoints

1. The doc comments use Markdown formatting. Why does it make sense for doc comments to use Markdown rather than a Rust-specific format?
2. The convention is "first sentence is the summary; following paragraphs are detail." What is the practical reason for this convention?
3. The doc comments include sections like `# Examples`, `# Errors`, and `# Panics`. These are not enforced by the language. Why does the convention matter if it is not required?

---

## Exercise 8 -- Doc Tests

**Estimated time:** 10 minutes
**Topics covered:** runnable code examples in docs, hiding setup, the `compile_fail` and `should_panic` annotations

### Context

A particularly useful feature: code blocks in documentation are extracted and run as tests. This means examples in docs cannot rot; if they break, `cargo test` catches them.

### Task 8.1 -- Doc tests automatically run

The doc comments you wrote in Exercise 7 already contain code examples. Run all tests:

```bash
cargo test
```

Look at the output for the `Doc-tests` section:

```
   Doc-tests inventory_lib

running 5 tests
test src/lib.rs - cost_per_item (line 41) ... ok
test src/lib.rs - parse_inventory_entry (line 63) ... ok
test src/lib.rs - compute_discount (line 27) ... ok
test src/lib.rs - item_value (line 11) ... ok
test src/lib.rs - (line 7) ... ok

test result: ok. 5 passed; 0 failed
```

Five doc tests ran: one per function with an `# Examples` section, plus one for the module-level quick-start example. Each ` ``` ` code block in a doc comment is treated as a test.

If you change a function in a way that breaks its example, the doc test fails. The documentation stays in sync with the code.

### Task 8.2 -- A breaking example

To see this in action, change a function temporarily:

```rust
pub fn item_value(quantity: u32, unit_price: f64) -> f64 {
    quantity as f64 * unit_price * 2.0      // BREAK: extra factor
}
```

Run `cargo test --doc`. Expected output:

```
running 5 tests
test src/lib.rs - item_value (line 11) ... FAILED
...

failures:

---- src/lib.rs - item_value (line 11) stdout ----
Test executable failed (exit status: 101).

stderr:
thread 'main' panicked at 'assertion `left == right` failed
  left: 30.0
 right: 15.0'
```

The doc test failed because the function no longer behaves as documented. This is a real safety net: documentation cannot become out of sync because the compiler verifies it.

Restore the function:

```rust
pub fn item_value(quantity: u32, unit_price: f64) -> f64 {
    quantity as f64 * unit_price
}
```

### Task 8.3 -- Hiding setup code

Sometimes a doc example needs setup that is not interesting to the reader. Lines starting with `#` are part of the test but hidden from the rendered docs:

```rust
/// Computes the total inventory value across multiple items.
///
/// # Examples
///
/// ```
/// # use inventory_lib::item_value;
/// let total = item_value(5, 10.0) + item_value(3, 20.0);
/// assert_eq!(total, 110.0);
/// ```
pub fn total_value(items: &[(u32, f64)]) -> f64 {
    items.iter().map(|(qty, price)| item_value(*qty, *price)).sum()
}
```

Add this new function to your library. The doc example uses an `#` line to hide the `use` statement. The reader sees only the interesting code; the test still has access to what it needs.

Wait. Actually the example above does not exercise `total_value`. Let me improve it:

```rust
/// Computes the total inventory value across multiple items.
///
/// Each item is a (quantity, unit_price) pair.
///
/// # Examples
///
/// ```
/// let items = vec![(5, 10.0), (3, 20.0)];
/// let total = inventory_lib::total_value(&items);
/// assert_eq!(total, 110.0);
/// ```
pub fn total_value(items: &[(u32, f64)]) -> f64 {
    items.iter().map(|(qty, price)| item_value(*qty, *price)).sum()
}
```

Run `cargo test --doc`. The new doc test passes.

The `#` hiding is useful when:

- The setup is uninteresting (imports, boilerplate).
- The example needs context that does not fit the narrative.
- You want to keep examples focused on the demonstrated behavior.

### Task 8.4 -- Doc tests that should fail to compile

Sometimes you want to show that something is NOT allowed. The `compile_fail` annotation says the example should not compile:

```rust
/// A helper type for typed quantities.
///
/// # Examples
///
/// ```
/// let q = inventory_lib::Quantity::new(5);
/// assert_eq!(q.value(), 5);
/// ```
///
/// Direct construction is not allowed:
///
/// ```compile_fail
/// // The `count` field is private:
/// let q = inventory_lib::Quantity { count: 5 };
/// ```
pub struct Quantity {
    count: u32,
}

impl Quantity {
    pub fn new(count: u32) -> Quantity {
        Quantity { count }
    }

    pub fn value(&self) -> u32 {
        self.count
    }
}
```

Run `cargo test --doc`. Expected output (with two new doc tests):

```
running 6 tests
test src/lib.rs - Quantity (line 102) ... ok
test src/lib.rs - Quantity (line 107) ... ok
...
```

The first example demonstrates valid use. The second example, with `compile_fail`, passes only because it does NOT compile (which is what we want to demonstrate).

This annotation is useful for documenting type-system guarantees. If the type stops enforcing the guarantee (someone accidentally makes `count` public, for example), the doc test starts compiling and the test fails. The annotation works as a regression test for the API design.

### Task 8.5 -- Doc tests that should panic

The `should_panic` annotation works in doc tests too:

```rust
/// Divides a total cost by a number of items.
///
/// # Examples
///
/// ```
/// let per_item = inventory_lib::cost_per_item(100.0, 4);
/// assert_eq!(per_item, 25.0);
/// ```
///
/// # Panics
///
/// Panics if `items` is 0:
///
/// ```should_panic
/// inventory_lib::cost_per_item(100.0, 0);
/// ```
pub fn cost_per_item(total_cost: f64, items: u32) -> f64 {
    if items == 0 {
        panic!("cannot compute cost per item for 0 items");
    }
    total_cost / items as f64
}
```

Update `cost_per_item` with this enhanced documentation, replacing your previous version.

Run `cargo test --doc`. The new doc test passes because the example panics (as the annotation expects). The example clearly demonstrates the panic behavior, complementing the `# Panics` section.

### Task 8.6 -- Skipping doc tests

Some code blocks are illustrative but not runnable. The `ignore` annotation skips them as tests:

```rust
/// # Examples
///
/// ```ignore
/// // This requires a connection to a server, not runnable in tests:
/// let prices = connect_to_server("https://example.com");
/// let total = inventory_lib::total_value(&items_from(prices));
/// ```
```

The code block is rendered as documentation but not run as a test. Use this for examples that require external resources (network, files, environment) or that are too elaborate to maintain as tests.

### Task 8.7 -- Why doc tests matter

Doc tests serve three purposes:

- **Documentation.** Worked examples of how to use the API.
- **Testing.** Verification that the API works as documented.
- **Maintenance pressure.** Examples cannot drift out of sync; if they do, builds fail.

The combination is what makes Rust documentation reliable. When you read a doc example, you can trust it because it has been verified to work.

In libraries with many functions, doc tests can outnumber regular tests. The combined "doc + test" form encourages writing good examples for every public API.

### Checkpoints

1. Doc tests are extracted from comments and run as part of `cargo test`. What kind of bug does this catch that regular tests might miss?
2. Task 8.3 used `#` to hide setup code in doc examples. What is the trade-off between using `#` and just writing the full setup as visible?
3. The `compile_fail` annotation makes a doc test pass when the code does NOT compile. What is this useful for in practice?

---

## Exercise 9 -- Generating Docs with cargo doc

**Estimated time:** 5-10 minutes
**Topics covered:** `cargo doc`, the generated HTML, docs.rs, `#![warn(missing_docs)]`

### Context

The `cargo doc` command generates HTML documentation from your code. The output is browsable and matches the format of the standard library docs.

### Task 9.1 -- Generate documentation

Run:

```bash
cargo doc --open
```

The `--open` flag opens the generated HTML in your default browser. You should see a navigation page with your crate, its modules, and its items listed.

If `--open` does not launch a browser, run without it and manually open `target/doc/inventory_lib/index.html`.

Browse around:

- The crate's main page shows the module-level documentation (the `//!` at the top of `src/lib.rs`).
- Each function has its own page with the signature, doc comments, and examples.
- The "All items" link shows everything in the crate.
- Click any linked item to navigate.

The look matches https://doc.rust-lang.org/std/, which is the same documentation system applied to the standard library.

### Task 9.2 -- What gets included

By default, `cargo doc` documents:

- Public items in your crate.
- Public items from your dependencies (so you can navigate from your code to library types).

To document only your crate (faster, smaller output):

```bash
cargo doc --no-deps --open
```

This skips the dependencies' documentation. For everyday use during development, this is usually what you want.

### Task 9.3 -- Hiding items from documentation

Sometimes you have public items that should not appear in the public documentation (for example, items intended only for use by tests or internal tooling). The `#[doc(hidden)]` attribute hides them:

```rust
#[doc(hidden)]
pub fn internal_helper(x: u32) -> u32 {
    x * 2
}
```

Add this to your library. Run `cargo doc --open` again. The function does not appear in the documentation, even though it is public.

`#[doc(hidden)]` is used sparingly. It is appropriate for:

- Items that are technically public but unstable (subject to change without notice).
- Test helpers exposed for integration tests but not meant for general use.
- Internal items needed for the crate's macros to work.

For most items, if it is public, it should be documented. If it should not be documented, it probably should not be public.

### Task 9.4 -- Enforcing documentation

For libraries that take documentation seriously, you can require every public item to have docs:

```rust
#![warn(missing_docs)]
//! A small inventory management library.
//! ...
```

The `#![warn(missing_docs)]` at the very top of `src/lib.rs` (before any other content) makes the compiler warn for any public item that lacks documentation.

Add this to your library and run `cargo build`. If you have undocumented public items, you will see warnings like:

```
warning: missing documentation for a function
  |
  | pub fn some_undocumented_function() { ... }
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

For published crates, this lint is essential. It catches the easy-to-forget cases (you added a new function but forgot to document it).

Some projects use `#![deny(missing_docs)]` to make missing docs an error rather than a warning, forcing them to be addressed before code can be merged.

### Task 9.5 -- docs.rs

When you publish a crate to crates.io (the Rust package registry), docs.rs automatically builds and hosts the generated documentation. Users see your `cargo doc` output without you needing to host it.

The URL format is:

```
https://docs.rs/<crate_name>/<version>
```

For example, `https://docs.rs/serde` shows the docs for serde's latest version.

This is a powerful community standard: every published Rust crate has online documentation by default. Users expect to find docs at `https://docs.rs/<crate>`; if your docs are good, they will find them and use your crate. If your docs are poor or missing, users will pass over you for a better-documented alternative.

The investment in documentation pays off in adoption. Well-documented crates are widely used; poorly-documented ones struggle for visibility regardless of their quality.

### Task 9.6 -- Customizing the generated output

A few options for customizing what `cargo doc` produces:

```toml
# Cargo.toml

[package.metadata.docs.rs]
all-features = true                              # build with all features
rustdoc-args = ["--cfg", "docsrs"]               # special cfg flag
```

The `[package.metadata.docs.rs]` section configures how docs.rs builds your documentation. These settings only apply to docs.rs; local `cargo doc` builds ignore them.

For individual items:

```rust
#[doc(alias = "stock")]
pub fn count_items(...) { ... }
```

The `#[doc(alias = "stock")]` attribute adds "stock" as a search alias. If someone searches for "stock" in the docs, they will find this function. Useful when an item has multiple natural names.

These are advanced features that you rarely need for typical projects. The default `cargo doc` output is usually sufficient.

### Checkpoints

1. The default `cargo doc` builds documentation for both your crate and all its dependencies. What is the practical benefit of including dependencies in the docs? When is `--no-deps` preferable?
2. The `#![warn(missing_docs)]` lint makes the compiler warn about undocumented public items. Why is this useful for libraries but rarely used for applications?
3. The docs.rs convention (`https://docs.rs/<crate>`) is a community standard, not a language feature. What does this kind of convention provide that a feature could not?

---

## Summary and Reflection

You have now used every concept from Module 16 in a working library.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Unit Tests | `#[cfg(test)] mod tests`, private items | Unit tests live with the code; they can access private items, run with `cargo test`. |
| 2 -- Integration Tests | `tests/` directory, public-API only | Integration tests verify the public interface; they cannot access private items. |
| 3 -- Assertions | `assert!`, `assert_eq!`, custom messages | `assert_eq!` shows both sides on failure; custom messages help diagnose what was expected. |
| 4 -- Panics and Results | `#[should_panic]`, `Result`-returning tests | Each test style has its place; choose based on whether the function panics or returns Result. |
| 5 -- Running Tests | name patterns, `#[ignore]`, output capture | `cargo test` has flexible filters and options for running subsets. |
| 6 -- Mocking | `mockall`, manual fakes | Mocks let you simulate any behavior of a dependency; manual fakes are simpler when sufficient. |
| 7 -- Doc Comments | `///`, `//!`, conventional sections | Documentation is a first-class concern; conventional sections make docs feel familiar. |
| 8 -- Doc Tests | runnable examples in docs | Examples cannot rot because they are tested; documentation and tests are unified. |
| 9 -- Generating Docs | `cargo doc`, docs.rs | The doc generation is part of the standard tooling; docs.rs hosts published crates' docs automatically. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab12-notes.md` before your next session.

1. Of the testing and documentation concepts in Module 16, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. Rust unifies documentation and testing via doc tests: code examples in documentation are run as part of the test suite. Identify a specific kind of bug or maintenance problem this unification solves compared to languages where documentation and tests are separate.

3. The lab introduced both unit tests (in `src/lib.rs`) and integration tests (in `tests/`). Many languages use one or the other; Rust supports both. Identify a specific situation where each kind is the right choice for a given feature. What characteristics push you toward each?

---

*End of Lab 12*
