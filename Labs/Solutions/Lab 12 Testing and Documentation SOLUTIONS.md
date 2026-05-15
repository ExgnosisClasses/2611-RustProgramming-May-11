# Module 16: Testing and Documentation
## Lab 12 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Unit Tests with `#[test]`

### Checkpoints

**1. What is the practical benefit of `#[cfg(test)]` versus always compiling test code?**

Several benefits:

- **Binary size.** Release builds do not include test code, test helpers, or test-only dependencies. For a library that has hundreds of tests, this can mean a substantially smaller release binary.
- **Compile time.** Release builds skip the test module entirely. For codebases with extensive test suites, the savings on release builds add up.
- **Dependency separation.** Test-only dependencies (like `mockall` in Exercise 6) are listed under `dev-dependencies` and excluded from release builds. Without `#[cfg(test)]` (or its conceptual equivalent), every test dependency would be a runtime dependency, bloating the binary.
- **Encapsulation.** Test helpers can live alongside the code without polluting the public API. The `tests` module is private (to the file); test-only types and functions are not visible to library users.
- **Clear separation.** Reading the source, you immediately see which code is production and which is for testing. The `#[cfg(test)] mod tests` block is a visual marker.

The general principle: keep production builds lean. Test code, test dependencies, and test helpers are necessary during development but become noise at deployment time. Conditional compilation makes the separation crisp.

**2. Tests in the same file versus tests in separate files: trade-offs?**

**Same file (Rust's unit test convention):**

- **Pros:** Tests can access private items, encapsulation is preserved. Tests are closer to the code they verify, making it easier to maintain both together. Adding a function and its tests can happen in one place. No need to expose internals just for testing.

- **Cons:** Source files become longer (function plus tests). For complex modules, tests can dwarf production code in line count. Test-related code is mixed with production code in the same file, even if logically separated by the `#[cfg(test)] mod tests` block.

**Separate files (Java, Python convention):**

- **Pros:** Source files contain only production code. Tests are easy to locate (one file maps to another). Test discovery tools have clear conventions for where to look.

- **Cons:** Private items cannot be tested directly. To test internal behavior, you must expose it (loosening encapsulation) or use special "package-private" access (Java) or naming conventions (Python `_private`). This is a real loss: some internal invariants are hard to verify through public APIs alone.

When does each approach matter?

For libraries with clear public APIs, the language difference is small. You test mostly through the public API in both cases. Integration tests handle the rest.

For complex internal modules with intricate algorithms, the same-file convention shines. You can write detailed tests of private functions without disturbing the public surface.

For very large source files, the separate-file convention has advantages: each file stays focused. Rust mitigates this with the convention that test modules go at the bottom of the file, so scrolling past the production code reaches the tests naturally.

The Rust approach is a reasonable compromise: tests are in the same file but in a clearly separate module. You get most of the benefits of close-coupling without much of the file-size cost.

**3. What kinds of tests would break under parallel execution?**

Tests that share mutable state across the boundary that the test framework cannot see:

- **Tests that use global mutable variables (statics).** Two tests setting the same `static mut COUNTER` race; results depend on timing.
- **Tests that modify the filesystem in unsynchronized ways.** Two tests writing to the same file produce nondeterministic results.
- **Tests that modify environment variables.** Process-wide; one test setting `FOO=bar` is visible to another that runs concurrently.
- **Tests that use shared databases or caches.** Without isolation per test, results can leak between tests.
- **Tests that listen on a fixed port.** Only one test can bind the port; others fail.

The fundamental issue: parallelism assumes test isolation. When tests share state, the parallelism reveals the dependency.

What can you do about it?

- **Run sequentially:** `cargo test -- --test-threads=1`. Easy but slow.
- **Restructure to avoid shared state:** use unique resources per test (temp files with unique names, ephemeral ports, etc.).
- **Synchronize explicitly:** use a `Mutex` to serialize access to shared resources. Tests can run in parallel except for the critical sections.
- **Group related tests:** mark a small number of tests with `#[serial]` (from the `serial_test` crate) so only they serialize, while the rest run in parallel.

The right answer depends on the situation. For a few problematic tests, single-thread mode is acceptable. For many, restructuring is worth the effort.

---

## Exercise 2 -- Integration Tests in the tests/ Directory

### Checkpoints

**1. When to write a unit test versus an integration test for the same feature?**

The two test kinds serve different purposes.

**Unit tests** are for verifying internal implementation:

- Edge cases of small functions (boundary conditions, weird inputs).
- Private helpers that are not exposed.
- Branch coverage for internal logic.
- Performance characteristics or invariants that only make sense at the implementation level.

**Integration tests** are for verifying the public API:

- The "happy path" usage of the public API works.
- Combinations of public functions interact correctly.
- The library can be used the way a real consumer would use it.
- Refactoring does not change observable behavior.

For a given feature, both kinds add value. A typical pattern:

- **Unit tests** verify each piece of the implementation works in isolation. They cover edge cases comprehensively.
- **Integration tests** verify the pieces fit together for realistic use cases.

If you can only write one, the choice depends on what you are most worried about. For a library with subtle internal logic, unit tests catch implementation bugs. For a library with a complex public API, integration tests catch surface-level bugs.

Most real codebases have many more unit tests than integration tests. Integration tests are expensive (set up, tear down, larger surface area); unit tests are cheap. The pyramid (many unit tests, fewer integration tests, even fewer end-to-end tests) is the standard advice.

**2. Why are integration test files compiled as separate crates?**

Several reasons:

- **API enforcement.** Each integration test file is essentially "what would an external consumer of this library do?" Compiling them as separate crates enforces this: they can only access public items, just as a real consumer would.
- **Independence.** Tests in one file should not affect tests in another. Separate compilation means separate namespaces, separate `main` functions, separate test discovery. The boundary is strict.
- **Parallelism.** Test binaries can run in parallel. Each crate compiles independently and runs as its own process.
- **Specific to integration tests.** This setup is unique to the `tests/` directory; unit tests in `src/` share a single test binary because they live inside the same crate.

The cost is some duplication: each integration test file has its own `use` statements, its own helper definitions, and so on. The `tests/common/mod.rs` workaround mitigates this for shared helpers.

For most projects, the separation works well. You write a small number of focused integration test files; each verifies a coherent slice of the API. The strict boundary keeps tests from accidentally depending on each other.

**3. What goes wrong with `tests/common.rs` versus `tests/common/mod.rs`?**

`tests/common.rs` is treated by Cargo as an integration test file: a complete test crate with its own `main` function (generated by the test harness). Cargo will try to compile and run it.

If `common.rs` contains only helper functions (no `#[test]` annotations), Cargo runs it and reports:

```
running 0 tests
test result: ok. 0 passed; 0 failed
```

This is harmless but cluttered. Worse, if `common.rs` has any issues (missing imports, compile errors), it produces failure messages that have nothing to do with actual tests.

The `tests/common/mod.rs` path puts the file in a subdirectory. Cargo does not treat subdirectories as integration test crates; it only looks at top-level files in `tests/`. The subdirectory's `mod.rs` is a regular module file, included with `mod common;` from other test files.

This pattern is documented in the Cargo book and is the standard way to share code between integration tests. The naming is a bit awkward (the directory exists just to hide the file from Cargo's test discovery), but it works reliably.

An alternative is `tests/common/helpers.rs` (or any non-`mod.rs` filename inside the subdirectory). Test files would use `mod common;` and then `common::helpers::function()`. This works but adds a layer of nesting that is usually not needed.

The standard convention is `tests/common/mod.rs` with helpers directly in it. Stick with this unless you have a specific reason to deviate.

---

## Exercise 3 -- The Assertion Macros and Custom Messages

### Checkpoints

**1. Why is `assert_eq!` preferred over `assert!(a == b)`?**

The key difference is failure output. When `assert!(a == b)` fails, the message is:

```
thread '...' panicked at 'assertion failed: a == b'
```

You know that the comparison failed but not what the values were. For nontrivial expressions, this is not enough information to diagnose the failure.

When `assert_eq!(a, b)` fails, the message is:

```
thread '...' panicked at 'assertion `left == right` failed
  left: 15.0
 right: 100.0'
```

You see both values. For a moment-of-failure diagnosis, this is much more useful: you immediately know what the code returned versus what was expected.

For complex types (`Debug`-printed), `assert_eq!` shows the full representation. A `Vec<i32>` or a `HashMap` is fully visible. With `assert!`, you would see only "failed."

The same applies to `assert_ne!`: on failure, both values are printed, including the one that should have been different.

For boolean conditions that are not equality (like `assert!(x > 0)` or `assert!(is_valid(x))`), `assert!` is appropriate. There is nothing to compare; the boolean is the assertion. For equality and inequality, the specific macros are clearer.

**2. Why does `0.1 + 0.2 != 0.3` in floating-point?**

Floating-point numbers cannot represent every decimal exactly. They use a binary fraction internally; some decimal fractions (like 0.1, 0.2, 0.3) have no exact binary representation. They are stored as the nearest representable approximation.

`0.1` is stored as approximately `0.1000000000000000055511151231257827021181583404541015625`. Similarly for `0.2` and `0.3`. The exact stored values differ from each other in the low-order bits.

Adding `0.1 + 0.2` produces approximately `0.30000000000000004`, which is NOT the same as the stored representation of `0.3` (which is approximately `0.299999999999999988897769753748434595763683319091796875`).

The result depends on the specific stored representations, which depend on the IEEE 754 standard. Every language using IEEE 754 floats has this behavior: Python, Java, C, JavaScript, and Rust all give `0.30000000000000004` for `0.1 + 0.2`.

Other surprising results:

- `1.0 / 3.0 * 3.0 != 1.0` in many cases.
- Summing many small floats can lose precision; the order of additions matters.
- Comparing computed values for equality is generally unreliable.

The remedies:

- **Use tolerance for comparisons:** `(a - b).abs() < epsilon`. The right `epsilon` depends on the magnitudes involved.
- **Use rational types** when exactness is needed. The `num-rational` crate provides exact arbitrary-precision rationals.
- **Use decimal types** for financial computations. The `rust_decimal` crate provides decimal arithmetic that mirrors what humans expect.
- **Be wary of equality.** Treat float equality as a smell. Most comparisons should use `<`, `>`, or tolerance-based equality.

For tests, the tolerance pattern (`(result - expected).abs() < 1e-10`) is the common solution. The `approx` crate provides convenience macros like `assert_relative_eq!` for more elaborate tolerance handling.

**3. Custom message format strings: scope rules?**

The format strings work like `format!`: they use `{}` for positional args and `{name}` for named-capture (variables in scope).

A format string can only reference variables that exist at the call site. If you reference a variable that does not exist, you get a compile error:

```rust
assert!(condition, "value is {missing_variable}");    // ERROR
```

The compiler tells you that `missing_variable` is not in scope. The message is checked at compile time, just like any other `format!` use.

This is a feature: it prevents broken messages that reference variables that no longer exist (due to a refactoring) or that were never meant to be referenced. The compiler catches the mistake.

The variables captured can be of any type that implements `Display` (for `{}`) or `Debug` (for `{:?}`). You can use either:

```rust
assert!(condition, "got {value}");          // Display
assert!(condition, "got {value:?}");        // Debug
assert!(condition, "got {value:#?}");       // Debug, pretty-printed
```

For complex types, `Debug` is often more useful in test messages. It shows structure that `Display` might hide.

---

## Exercise 4 -- `#[should_panic]` and Result-Returning Tests

### Checkpoints

**1. What does the `expected = "..."` argument to `#[should_panic]` catch?**

It catches the case where the code panics for the wrong reason.

Consider a function that should panic for invalid input:

```rust
pub fn process(input: i32) -> i32 {
    if input < 0 {
        panic!("input must be non-negative");
    }
    input * 2
}
```

A simple `#[should_panic]` test passes if the function panics at all:

```rust
#[test]
#[should_panic]
fn negative_panics() {
    process(-1);
}
```

Now suppose someone changes the function:

```rust
pub fn process(input: i32) -> i32 {
    let result = input.checked_mul(2).expect("overflow in process");
    result
}
```

The function no longer panics on negative numbers; it returns a doubled negative. For very large inputs, it panics on overflow.

If the test calls `process(-1)`, the function returns `-2` without panicking. The `#[should_panic]` test FAILS, correctly catching the regression. Good.

But what if the test calls `process(i32::MAX)`?

```rust
#[test]
#[should_panic]
fn negative_panics() {
    process(i32::MAX);   // panics due to overflow, not negative-input
}
```

The function panics (due to overflow). The `#[should_panic]` test PASSES, incorrectly suggesting that "the function panics on bad input" is still true. The panic is real, but it is the wrong panic.

`#[should_panic(expected = "...")]` would catch this:

```rust
#[test]
#[should_panic(expected = "input must be non-negative")]
fn negative_panics() {
    process(i32::MAX);
}
```

With the expected message, the test checks that the panic message contains the expected text. The overflow panic message ("overflow in process") does not contain "input must be non-negative," so the test fails. The discrepancy is caught.

The bug being prevented: code panics for the right input, but for the wrong reason. Specifying the expected message tightens the test.

**2. Why does `?` only work in functions returning `Result`?**

The `?` operator's semantics are: "if this is `Err`, return early from the current function with that error; otherwise, unwrap to the inner value."

For "return early with that error" to make sense, the function must be able to return an error. A function returning a plain `i32` cannot return `Err(...)`; the type does not allow it.

Rust enforces this at compile time. Using `?` in a non-`Result` function produces:

```
error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option`
```

The compiler tells you the function signature is wrong. You either change the function to return `Result`, or you handle the error explicitly without `?`.

This is part of Rust's "errors are values" philosophy. Error propagation is explicit: every function that might propagate an error must say so in its signature. There is no equivalent of unchecked exceptions where errors flow invisibly.

For test functions, this means `Result`-returning tests need `-> Result<(), SomeError>` in their signature. The test framework recognizes the `Result` return type and treats `Err` as a test failure.

The convention: use plain tests (`-> ()`) for simple cases; use `Result`-returning tests for tests with multiple fallible operations where `?` improves readability.

**3. Three styles for testing Result: when to choose each?**

**`result.is_err()`:** simplest. Tells you whether the function returned an error, but not which error.

- **Use when:** the test is "this should fail" and you do not care which way. Quick exhaustive check that error paths are reached.
- **Loses:** any detail about the error. If the function suddenly returns the wrong error variant, the test still passes.

**`match` on the result:** verbose but flexible. Lets you destructure the error and assert specific things about it.

- **Use when:** you need to verify the error has specific properties (correct variant, correct message, correct data).
- **Loses:** conciseness. Two arms of pattern matching where one operation would do.

**`matches!(result, Err(...))`:** middle ground. Pattern matching as a single expression, with optional guard for additional checks.

- **Use when:** you want to verify the variant but the structure is simple. Good for "is this the right kind of error?"
- **Loses:** access to the captured values. You cannot easily inspect them after the match.

Practical choices:

- For most tests, `assert!(result.is_err())` is fine. The presence of an error is what matters.
- For tests that check specific error variants in an enum, `matches!` is concise.
- For tests that need detailed assertions about the error data, `match` gives you that.

A common pattern in practice:

```rust
// Quick check: did we get an error?
let err = result.unwrap_err();

// Detailed checks on the error:
assert!(err.to_string().contains("expected substring"));
assert!(matches!(err, MyError::InvalidInput(_)));
```

`unwrap_err()` panics if the result was `Ok`, but for tests where you have already verified the error case, it gives you direct access to the error data.

---

## Exercise 5 -- Running Subsets and Test Organization

### Checkpoints

**1. Parallel tests: performance benefit and cost?**

**Performance benefit:** for a test suite with hundreds of tests and modern multi-core CPUs, parallel execution can be many times faster than sequential. A laptop with 8 cores can run 8 tests at once; the wall-clock time is roughly `total_test_time / 8` for evenly distributed tests.

For unit tests (typically fast, isolated), the speedup is essentially linear with core count. For integration tests (sometimes slower, sometimes interdependent), the speedup is less reliable.

**Cost:** parallel tests require isolation. Any shared state between tests creates flakiness or wrong results. Tests that share resources (files, sockets, environment variables, global state) need careful synchronization or sequential execution.

The cost manifests as flaky tests: tests that pass sometimes and fail other times depending on timing. These are very expensive to diagnose; engineers waste significant time chasing them.

The right approach:

- **Design tests for isolation by default.** Each test creates its own resources, cleans up after itself, does not share state.
- **Use unique names/IDs for resources.** Temp files with random suffixes, ephemeral ports, separate databases.
- **Serialize the few that genuinely cannot run in parallel.** Use crates like `serial_test` for this, marking specific tests with `#[serial]`.

For most projects, the parallelism is worth the discipline. The speedup makes the test suite usable; without it, slow tests become tolerated, and tests are not run as often.

**2. `#[ignore]` versus commented-out tests: practical difference?**

`#[ignore]` keeps the test in the codebase, compiled, and visible:

- The test still compiles. Refactorings that affect the test still trigger compile errors.
- The test appears in `cargo test --list` (the test discovery output).
- You can run it explicitly with `cargo test -- --ignored`.
- The test name is recorded; CI can report on ignored tests.

A commented-out test is gone:

- Code may not compile; you would not know without uncommenting.
- The test is invisible to tooling.
- Future developers may not notice that it exists.
- It can rot silently as the surrounding code changes.

Practical uses of `#[ignore]`:

- **Slow tests:** mark to skip by default but available on demand.
- **External-dependency tests:** require database, network, or specific environment.
- **Quarantined tests:** known flaky, under investigation. Better than deleting.
- **Future tests:** scaffolding for features not yet ready.

The general rule: prefer `#[ignore]` over commenting out. Keeping the test code alive helps with maintenance even when the test is not running.

A more aggressive variant is to use feature flags:

```rust
#[cfg(feature = "slow-tests")]
#[test]
fn very_slow_test() { ... }
```

The test compiles only when the `slow-tests` feature is enabled. CI can run regular tests by default and slow tests with `cargo test --features slow-tests`. This is a cleaner separation than `#[ignore]` but adds Cargo configuration.

**3. `cargo test ::` matches what?**

The substring "::" appears in test names that include their module path. For tests in `src/lib.rs`, the names are `tests::test_name` (with the `::` separator).

So `cargo test ::` matches every test inside any module (because every such test name contains `::`). For integration tests, the test name is just `test_name` (no module path), so they would NOT match.

The result: `cargo test ::` runs all unit tests but skips all integration tests.

Of course, the cleaner way to do this is `cargo test --lib` for unit tests only. The `::` pattern is more of a curiosity than a tool.

The general pattern: `cargo test <substring>` is fuzzy matching on test names. Useful substrings depend on naming conventions:

- `cargo test foo` for tests with "foo" in their name.
- `cargo test tests::` for tests in a module called `tests`.
- `cargo test integration` for tests with "integration" in their name (if you adopt that convention).

For more elaborate filtering, the `cargo nextest` tool (a third-party alternative to `cargo test`) has more sophisticated filtering syntax.

---

## Exercise 6 -- Mocking with mockall

### Checkpoints

**1. `dev-dependencies` versus regular `dependencies`?**

`dev-dependencies` are dependencies used only during development:

- Test code.
- Benchmarks.
- Examples.

They are NOT compiled into the production build. Users of your library do not download them; they are not linked into the final binary.

Regular `dependencies` are compiled into the production build. Users of your library depend on them; the library cannot be used without them.

Why this matters:

- **Binary size.** Production builds skip dev-dependencies. `mockall` is a large crate (with `syn`, `quote`, and others); making it a runtime dependency would bloat every binary using your library.
- **Compile time for users.** A user of your library compiles your library plus its dependencies. If `mockall` is a regular dependency, every user pays its compile cost. As a dev-dependency, only your tests do.
- **Semantic correctness.** `mockall` is for testing, not production. Marking it as a dev-dependency is honest about its purpose.

The general rule: anything used only for testing, benchmarking, or examples goes in `dev-dependencies`. Anything the library itself uses goes in `dependencies`.

A common mistake: putting a test-related dependency in regular `dependencies` because the developer didn't know about `dev-dependencies`. This is harmless but wasteful; clean it up when you notice.

**2. `&dyn PriceLookup` versus `impl PriceLookup`?**

Both work, with different semantics.

**`&dyn PriceLookup`:** trait object, dynamic dispatch. The function can be called with any type implementing the trait. At runtime, method calls go through a vtable.

**`impl PriceLookup`:** generic, static dispatch. The compiler generates a specialized version for each concrete type. Method calls are direct (no vtable).

Practical differences:

- **Performance:** `impl` is slightly faster (direct calls, inlinable). `dyn` has one extra memory indirection per call.
- **Binary size:** `impl` produces more code (one specialized version per concrete type). `dyn` produces less.
- **Heterogeneous collections:** `dyn` lets you have `Vec<Box<dyn PriceLookup>>` mixing types. `impl` does not.

For the mock test, either works. With `impl`:

```rust
pub fn order_total<P: PriceLookup>(
    items: &[(String, u32)],
    prices: &P,
) -> Result<f64, String> {
    // same body
}

// In test:
order_total(&items, &mock)    // mock is concrete MockPriceLookup
```

For most code, `impl` is the better default: faster, more flexible at the call site, and "fails to compile" if the type does not match (rather than panicking through a vtable). Reach for `dyn` when you need runtime polymorphism (multiple types in one collection, returning different types from a function).

For library APIs, `impl Trait` is also slightly nicer for users: they pass whatever type they have without needing `Box::new(...)`.

**3. When do mocks become a liability?**

When tests verify mock interactions rather than real behavior. Some patterns to watch for:

- **Tests that pass even when the production code is broken.** If mocks return what the test expects regardless of input, the test does not catch real bugs.
- **Tests that mirror the production code structure.** When tests assert "this method calls that method with those arguments," the test becomes a tautology: it verifies the code does what it does, not that it does the right thing.
- **Brittle tests that break on every refactoring.** Mocks set up specific call expectations; refactoring that preserves behavior but changes calls breaks the tests.
- **Mocks of types you control.** If a type has a real implementation that is easy to use, prefer it over a mock. Mocks shine for slow, external, or stateful dependencies.

Better alternatives often:

- **Real implementations.** If the dependency is fast and deterministic, use it directly.
- **Fakes (manual implementations).** A simple struct that behaves like the real thing but with in-memory state. Cleaner than mocks for many cases.
- **Test doubles for specific behaviors.** A "stub" that returns predetermined values without verifying calls.

The mockall crate's strength is in cases where:

- The trait has many methods, and writing a full fake would be tedious.
- You want to verify specific interactions (call counts, argument values).
- The dependency is genuinely external (database, network) and a real implementation in tests would be impractical.

For simple cases (a trait with one or two methods, behavior easy to fake), a manual fake is often clearer. The lab's `FakeLookup` in Task 6.6 is an example.

Use mocks deliberately, not reflexively. Ask: "what is this test really verifying?" If the answer is "the mock returns what I told it to," the test is not verifying anything useful.

---

## Exercise 7 -- Documentation Comments

### Checkpoints

**1. Why Markdown for doc comments?**

Markdown is a widely-known format. Most developers can read and write it without learning new syntax. By using Markdown:

- **Authors do not need to learn a custom format.** They use the same syntax they know from README files, GitHub, Stack Overflow, etc.
- **Tools work naturally.** Editors with Markdown support (VS Code, IntelliJ, vim plugins) provide syntax highlighting and preview for doc comments.
- **Output is web-friendly.** Markdown converts cleanly to HTML, which is what `cargo doc` produces. The same source serves both reading-in-source and reading-on-web.

A Rust-specific format would have several disadvantages:

- Authors learn a new syntax.
- Tools need Rust-specific support, which is rare.
- The output is constrained by what the format can express.

Other languages have made similar choices. Python docstrings often use reStructuredText or Markdown. JavaScript's JSDoc allows Markdown in descriptions. The pattern is established: text in source files should be readable and writable using familiar formats.

The choice of Markdown specifically (rather than reStructuredText or something else) reflects its dominance in the developer world. GitHub, Stack Overflow, Slack, and many other tools default to Markdown; making doc comments Markdown leverages this familiarity.

**2. Why "first sentence is the summary" convention?**

Many tools display only the first sentence in indexes, search results, and tooltips:

- The docs.rs item index shows the first sentence next to the item name.
- Editor tooltips when hovering over a function show the first sentence.
- The `cargo doc` search results show the first sentence.

If your first sentence is informative, these displays are useful. If it is "TODO" or generic ("Helper function"), the displays are useless.

The convention forces good writing discipline. The first sentence must be a self-contained summary: "Computes the value of an inventory item" rather than "This function works with items." It must be complete (with a period); tools may strip everything after the first period.

A common pattern:

```rust
/// Returns the count of active users.       // first sentence: summary
///
/// The count excludes users who have been   // detail follows
/// inactive for more than 90 days.
pub fn active_user_count() -> usize { ... }
```

The summary works in isolation (in indexes, tooltips). The detail provides context for the in-depth reader.

This convention is not enforced by the language, but tools assume it. Following it makes your documentation work as expected with the broader ecosystem.

**3. Why does the section convention matter if not enforced?**

Conventions matter because they create predictability. When every library follows the same conventions:

- **Users learn one set of patterns.** Reading a new library, they know to look for `# Examples` and `# Errors` sections.
- **Searches work naturally.** Users searching for "panics" in docs find the `# Panics` sections.
- **Tools can build on the conventions.** Documentation aggregators, IDE features, and linters can rely on the standard sections.
- **Authors do not have to invent.** Conventions remove decisions: you know where to put what.

The convention specifically:

- `# Examples` for usage examples (which become doc tests).
- `# Errors` for `Result`-returning functions, explaining when `Err` is returned.
- `# Panics` for functions that can panic, explaining when.
- `# Safety` for `unsafe` functions, explaining invariants the caller must uphold.

The Rust standard library uses these consistently. Major crates adopt them. The result is documentation that feels familiar across the ecosystem; users browsing different libraries see the same structure.

If everyone invented their own conventions, browsing libraries would require learning each one's patterns. The shared convention reduces friction; that is why it matters even without enforcement.

You can add your own sections too:

```rust
/// # Examples
/// ...
/// # Performance
/// O(n) in the length of the input.
/// # Compatibility
/// Available since version 1.2.
```

The standard sections cover the most common needs; project-specific sections add detail. The convention is "common things use the standard names; specific things use whatever fits."

---

## Exercise 8 -- Doc Tests

### Checkpoints

**1. What kind of bug do doc tests catch that regular tests might miss?**

Documentation rot: the gradual drift between what the documentation says and what the code actually does.

Without doc tests:

- A developer changes a function.
- The function's documentation includes an example showing the old behavior.
- The developer does not update the example (does not notice, forgot, deferred).
- The documentation is now wrong; users following the example will be confused or broken.

This is extremely common in projects without doc tests. The documentation includes outdated examples, deprecated patterns, or behaviors that no longer exist. Reading the docs becomes unreliable: are these examples current?

With doc tests:

- Examples are run as part of `cargo test`.
- If the example no longer works (compile error or assertion failure), the test fails.
- Developers see the failure and either fix the example or fix the function.
- Documentation stays in sync with code.

The mechanism is simple but powerful. Every doc example is also a test; the test forces the example to be current.

The bug being prevented is not in the production code; it is in the documentation. But documentation bugs are real bugs from the user's perspective. A library with wrong examples is harder to use than one with no examples.

This is one of Rust's most distinctive testing features. Many languages have generated documentation, but few automatically verify that the examples in the documentation work. Rust's approach is unusual and effective.

**2. Hidden `#` lines: trade-offs?**

**Benefits of hiding setup:**

- The reader sees only the interesting code. Examples focus on the demonstrated behavior.
- Doc examples can be more elaborate without being more verbose to read.
- Boilerplate (imports, scaffolding) does not distract from the point.

**Costs:**

- The example is not literally copy-pasteable. If a user copies the visible code, it might not compile because of the hidden setup.
- Surprise: hidden code can do things the reader does not expect (set up state, define functions, etc.).
- Maintenance: the visible code and the hidden code must stay in sync.

The right balance:

- **Hide truly trivial setup.** `use crate_name::ItemType;` is rarely interesting; hiding it cleans up the example.
- **Show interesting setup.** If users need to know how to construct a complex object, do not hide the construction.
- **Keep the visible example self-contained-ish.** A user reading should understand the example without the hidden parts.

A test for whether to hide a line: "would I want this in a tutorial?" If yes, show it. If it is just necessary plumbing, hide it.

For complex APIs, an alternative to extensive hiding is to write a longer module-level example (which can include all the setup) and have the per-function examples focus on just that function's use. The setup is established once, then referred to.

**3. `compile_fail`: practical uses?**

Demonstrating type-system guarantees. Some APIs are designed to prevent certain misuses at compile time; `compile_fail` examples verify the prevention.

Common cases:

- **Encapsulation.** A type with private fields cannot be constructed externally. The `compile_fail` example shows the failure.
- **Lifetime constraints.** A reference cannot outlive what it references. An example demonstrating the constraint shows the type system catching it.
- **Type-state patterns.** A builder that requires methods to be called in order. Calling them in the wrong order is a compile error.
- **Send/Sync requirements.** Types that cannot cross thread boundaries. An example showing the constraint.

For example:

```rust
/// A connection that must be closed before being reused.
///
/// ```compile_fail
/// let conn = Connection::open();
/// // Attempting to use after explicit close:
/// conn.close();
/// conn.send_data();   // ERROR: conn was consumed by close()
/// ```
```

The example documents the API behavior AND tests that the compile error happens. If a refactoring accidentally allows the disallowed pattern (the `close()` method now takes `&self` instead of `self`), the `compile_fail` doc test will start compiling, and the test will fail.

This is a regression test for the API design. The test ensures the constraint stays in place.

Use `compile_fail` for documented invariants that the type system enforces. It's a small but powerful tool: zero runtime cost (the example never runs), strong guarantee (the test catches design regressions), and good documentation (users see what is NOT allowed).

---

## Exercise 9 -- Generating Docs with cargo doc

### Checkpoints

**1. `cargo doc` with versus without dependency docs?**

**With dependency docs (default):**

- Generates documentation for your crate AND all transitive dependencies.
- Navigation links to dependency types work (clicking a return type like `serde_json::Value` opens that type's docs).
- Large output (potentially hundreds of MB for crates with many dependencies).
- Slow to generate.

**Without (`--no-deps`):**

- Generates documentation for only your crate.
- Links to dependency types may not resolve (no destination to link to).
- Small output.
- Fast.

When to choose each:

- **Use the default during development.** When you are reading and writing docs, having links to dependency types is useful. The slow first generation is amortized over many quick reads.
- **Use `--no-deps` for fast iteration.** If you are repeatedly running `cargo doc` to check your own docs' rendering, skipping deps speeds up the cycle.
- **Use the default before publishing.** Make sure all the cross-links work for users who will see them.

For `docs.rs`, the default behavior is to build with dependencies. Published crates have full navigation working when users view them online.

**2. `#![warn(missing_docs)]` for libraries versus applications?**

**Libraries:** documentation is critical because users learn the library from its docs. A library with undocumented public items is hard to use. The lint catches the easy-to-forget cases (you added a function, forgot to document it before publishing).

The cost of documentation is paid once (by the author) and amortizes across all users. The benefit is enormous: every user can understand the library faster.

**Applications:** the "users" are the development team, who can read the source. Documentation matters less because the code IS the audience. Internal applications typically have light documentation; intensive documentation effort is reserved for public APIs.

For applications, the lint produces noise: warnings on private items, helpers, etc. that do not benefit from being documented. The team focuses documentation effort on what matters (top-level architecture, key abstractions) rather than every function.

A reasonable middle ground for applications: document module-level structure and key types, do not require docs on every function. The `#![warn(missing_docs)]` lint applied selectively (to a `pub` API module) gives you the discipline without the noise.

For library authors, treat `#![warn(missing_docs)]` as standard. It is part of the discipline of writing libraries that others can use.

**3. The docs.rs convention: what does a community convention give you?**

A convention that "every published crate has documentation at `https://docs.rs/<crate>`" provides:

- **Discoverability.** Users know where to find docs. No searching for project websites, GitHub pages, or custom doc sites.
- **Uniform format.** All docs look the same (cargo doc's output). Users know the navigation, search, and structure.
- **Versioned URLs.** `https://docs.rs/serde/1.0.193` always shows that specific version. Permanent, citable links.
- **No author burden.** Crate authors do not have to host, build, or maintain documentation infrastructure. Publish to crates.io, and docs appear automatically.
- **Verification.** docs.rs only builds if the code compiles cleanly. Broken docs are visible (the build fails, the page shows the failure).

A language feature could not provide all of this. The combination of:

- A central package registry (crates.io).
- An automatic documentation host (docs.rs).
- A standard documentation tool (cargo doc).
- A consistent format (the cargo doc output).

forms an ecosystem. Each piece is small individually; together they make the Rust documentation experience uniformly good.

Other languages have similar conventions (pkg.go.dev for Go, Hex.pm for Elixir), and each contributes to that language's ecosystem culture. Conventions that "just work" reduce friction; they let developers focus on writing code rather than configuring infrastructure.

For Rust specifically, the convention is one of the strongest selling points to new users: "every crate has docs at a predictable URL, and they are usually good." This builds trust and adoption.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

Common patterns:

- Developers from Java find unit tests familiar (similar to JUnit's `@Test`). The placement of tests in the same file feels unusual at first; tests in a separate `src/test/java/` tree is the Java convention. The integration test directory pattern feels familiar.

- Developers from Python find the tooling integration impressive. `pytest` discovers tests automatically; `cargo test` does the same. The doc tests feature is unusual: Python's doctests exist but are less prominent. Markdown in docstrings is similar across the languages.

- Developers from C++ find the lack of build configuration refreshing. `cargo test` just works; no build system to configure, no test runner to install separately. The integration test pattern feels lighter-weight than Google Test or Catch2.

- Developers from JavaScript find the parallelism by default unusual. `jest` runs tests in parallel by default for files, but tests within a file are serial. Rust's per-test parallelism within files requires more care to avoid shared state.

The most common reflection: "Doc tests were the biggest surprise. Examples in documentation that get tested? I had not seen this before. After a few weeks of writing Rust, I realized how valuable it was: I cannot remember the last time I saw a Rust library's docs with broken examples. In other languages, broken examples are a normal hazard."

Another common reflection: "Mocking with `mockall` felt more elaborate than I expected. I am used to dynamic-language mocking (Python's `unittest.mock`, JavaScript's `jest.fn()`) where mocks are loose and easy. The trait-based approach in Rust is more rigorous but also more verbose. Manual fakes ended up being my preferred approach for most cases."

**2. What does unifying docs and tests via doc tests solve?**

Sample answer:

"The bug it solves is 'documentation rot.' In every project I have worked on (in Python, Java, C#), documentation examples become wrong over time. Someone refactors a function; the docstring still shows the old usage; nobody notices for months. New users encounter the broken example, struggle, ask why the docs are wrong, and lose trust.

The root cause is that documentation and code are separate artifacts. The compiler does not check the docs; the tests do not check the docs; only humans check the docs, and humans forget. The drift is inevitable given enough time.

Doc tests solve this by making the examples part of the test suite. The compiler checks they compile; the test framework checks they pass. If a function's behavior changes, the docs cannot silently lie about it. Either:

- The doc example is updated as part of the change. Documentation stays current.
- The doc example breaks. The test suite reports a failure. The author addresses it.

Either way, documentation cannot drift.

In Python, I have seen `doctest` (the equivalent feature) but it is not the default. You opt in by writing `if __name__ == '__main__': doctest.testmod()` or running `pytest --doctest-modules`. Many projects do not bother. In Rust, doc tests are part of `cargo test`'s default behavior; you opt out (explicitly mark examples as `ignore`) rather than opt in. The default matters: people use what is easy.

The combined effect: Rust documentation has higher reliability than equivalent docs in most languages. When I read a Rust example, I trust it. When I read a Python example, I check the modification date and the code to be sure. The trust comes from the integrated tooling."

**3. Unit tests in `src/` versus integration tests in `tests/`: when each?**

Sample answer:

"For a function that converts internal data structures or implements a private algorithm, I write a unit test. Example: a helper function `normalize_inputs` that is used inside the library but not exposed. The test goes in `#[cfg(test)] mod tests` in the same file. I can verify edge cases of the helper without exposing it publicly.

For a feature that users will use through the public API, I write an integration test. Example: a database client library's `connect_and_query` workflow. The integration test mimics what a user would do: import the public types, call public methods, verify the result. This catches API design issues (missing methods, awkward signatures) that unit tests cannot.

The characteristics pushing me toward each:

- **Unit test:** the thing being tested is small, internal, and has specific edge cases I want to verify in isolation.
- **Integration test:** the thing being tested is a user-facing workflow that involves multiple public items working together.

I also write integration tests for examples: scenarios that look like documentation but are full programs. These verify that idiomatic usage of the library works.

In a typical project, the ratio is probably 5-10 unit tests per integration test. Most testing effort goes into edge cases of small functions; integration tests verify that the big-picture usage works.

One pattern I have learned: if I find myself writing many integration tests for one function, I should probably write more unit tests instead. Integration tests are expensive (setup overhead, slower, broader scope). Unit tests are cheap (focused, fast, narrow scope). The right balance leans heavily toward unit tests, with integration tests for the higher-level checks."

---

*End of Lab 12 Solutions*
