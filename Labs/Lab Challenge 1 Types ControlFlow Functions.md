# Modules 3, 4, and 5: Types, Control Flow, and Functions
## Challenge Lab -- Implement a Numeric Sequence Analyzer

> **Course:** Mastering Rust
> **Modules:** 3 - Types and Variables, 4 - Control Flow, 5 - Functions and Closures
> **Estimated time:** 60 minutes
> **Type:** Challenge lab (write your own implementation against a spec)

---

## Overview

This is a different kind of lab. Instead of guided tutorial-style exercises that walk you through each step, you are given a problem statement and starter code. Your job is to fill in the implementations and make the program work. The starter code includes function signatures, test assertions, and a `main` function that exercises everything; you write the bodies.

The problem is to build a **numeric sequence analyzer**: a small program that takes a list of integers and computes a variety of statistics and transformations on them. Each function exercises specific concepts from Modules 3, 4, and 5. You can implement them in any order, but later functions sometimes use earlier ones, so working from top to bottom usually makes sense.

There is no single right answer; many implementations will pass the tests. The goal is to write idiomatic Rust that demonstrates understanding of the three modules. The solution file shows one good implementation; your version may differ.

### What you will demonstrate

By completing this lab you will show that you can:

- Choose appropriate scalar types and use mutability correctly (Module 3).
- Write functions with correct signatures and return types (Module 5).
- Use `if`, `match`, `for`, `while`, and `loop` for different control-flow needs (Module 4).
- Use ranges and iterator methods to traverse data (Modules 4 and 11 preview).
- Write closures and pass them to higher-order functions (Module 5).
- Apply the `Fn`/`FnMut` distinction correctly when writing closure-accepting functions.
- Use shadowing, tuple destructuring, and array literals when they fit (Module 3).
- Use early returns, labeled loops, and `break` with values where appropriate (Module 4).

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

> **Constraints you must respect.** This lab uses only material from Modules 3, 4, and 5. Do **not** use:
> - The `?` operator (Module 10).
> - Defining your own structs or enums (Modules 8, 9).
> - `HashMap` or `HashSet` (Module 11). You may use `Vec` because it appears in starter code.
> - Custom error types or `Result` (Module 10). The functions return plain values or `Option` for the "might fail" cases.
> - Pattern matching beyond what Module 4 covers (`if let`, simple `match` on integers/booleans). Avoid destructuring complex types.
>
> If you find yourself reaching for something on this list, you are probably overcomplicating the solution. Each function in the spec can be implemented using only Modules 3, 4, and 5.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new sequence_analyzer
cd sequence_analyzer
code .
```

---

## The Starter Code

Replace the contents of `src/main.rs` with the starter code below. The function signatures are complete; the bodies are stubbed with `todo!()` macros that will compile but panic at runtime. Replace each `todo!()` with your implementation.

```rust
// ============================================================
// Sequence Analyzer -- Challenge Lab for Modules 3, 4, 5
//
// Implement each of the functions below so that the assertions
// in `main` pass. You may add helper functions if useful.
// You may NOT add struct or enum definitions or use HashMap/HashSet.
// ============================================================

// ------------------------------------------------------------
// PART 1: Basic statistics
// ------------------------------------------------------------

/// Return the sum of all elements. Sum of an empty slice is 0.
fn sum(values: &[i32]) -> i32 {
    todo!()
}

/// Return the arithmetic mean of all elements as f64.
/// Return None if the slice is empty (cannot divide by zero).
fn mean(values: &[i32]) -> Option<f64> {
    todo!()
}

/// Return the minimum and maximum values as a tuple (min, max).
/// Return None if the slice is empty.
fn min_max(values: &[i32]) -> Option<(i32, i32)> {
    todo!()
}

/// Return the range (max - min) of the slice.
/// Return None if the slice is empty.
fn range(values: &[i32]) -> Option<i32> {
    todo!()
}

// ------------------------------------------------------------
// PART 2: Counting and filtering
// ------------------------------------------------------------

/// Return how many elements in the slice are positive (> 0).
fn count_positive(values: &[i32]) -> usize {
    todo!()
}

/// Return how many elements in the slice satisfy the predicate.
/// This function is generic over any predicate closure.
fn count_where<F>(values: &[i32], predicate: F) -> usize
where
    F: Fn(i32) -> bool,
{
    todo!()
}

/// Return a new Vec containing only the elements that satisfy the predicate.
/// Order must be preserved.
fn filter_values<F>(values: &[i32], predicate: F) -> Vec<i32>
where
    F: Fn(i32) -> bool,
{
    todo!()
}

// ------------------------------------------------------------
// PART 3: Transformations
// ------------------------------------------------------------

/// Return a new Vec where each element has been doubled.
fn doubled(values: &[i32]) -> Vec<i32> {
    todo!()
}

/// Return a new Vec where each element has been transformed by the function.
/// This function is generic over any unary transformation.
fn map_values<F>(values: &[i32], transform: F) -> Vec<i32>
where
    F: Fn(i32) -> i32,
{
    todo!()
}

/// Return a new Vec of running sums.
/// For input [1, 2, 3, 4], output is [1, 3, 6, 10].
/// For an empty input, return an empty Vec.
fn running_sum(values: &[i32]) -> Vec<i32> {
    todo!()
}

// ------------------------------------------------------------
// PART 4: Searching
// ------------------------------------------------------------

/// Return the index of the first element equal to target.
/// Return None if not found.
fn find_first(values: &[i32], target: i32) -> Option<usize> {
    todo!()
}

/// Return the index of the first element that satisfies the predicate.
/// Return None if no element satisfies it.
fn find_first_where<F>(values: &[i32], predicate: F) -> Option<usize>
where
    F: Fn(i32) -> bool,
{
    todo!()
}

// ------------------------------------------------------------
// PART 5: Reduction with closures
// ------------------------------------------------------------

/// Apply a binary operation to combine all elements, starting from `initial`.
/// For example, reduce(&[1, 2, 3], 0, |acc, x| acc + x) returns 6.
/// For an empty input, returns initial unchanged.
fn reduce<F>(values: &[i32], initial: i32, combine: F) -> i32
where
    F: Fn(i32, i32) -> i32,
{
    todo!()
}

// ------------------------------------------------------------
// PART 6: Closure-returning function (advanced)
// ------------------------------------------------------------

/// Return a closure that adds `n` to its argument.
/// For example, let add5 = make_adder(5); add5(10) returns 15.
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    todo!()
}

/// Return a closure that, when called, increments an internal counter
/// and returns the new count. Note this closure mutates state and must be FnMut.
/// For example: let mut c = make_counter(); c(); c(); c() returns 3.
fn make_counter() -> impl FnMut() -> i32 {
    todo!()
}

// ------------------------------------------------------------
// PART 7: A small classification challenge
// ------------------------------------------------------------

/// Classify a value as "negative", "zero", "small" (1-9), "medium" (10-99),
/// or "large" (100 or more).
fn classify(value: i32) -> &'static str {
    todo!()
}

/// Return a count of how many elements fall into each classification.
/// Return the counts in the order: (negative, zero, small, medium, large).
fn classification_counts(values: &[i32]) -> (usize, usize, usize, usize, usize) {
    todo!()
}

// ============================================================
// Main: tests all the functions
// ============================================================

fn main() {
    // PART 1: Basic statistics
    assert_eq!(sum(&[]), 0);
    assert_eq!(sum(&[1, 2, 3, 4, 5]), 15);
    assert_eq!(sum(&[-1, -2, -3]), -6);
    assert_eq!(sum(&[100, -50, 25]), 75);

    assert_eq!(mean(&[]), None);
    assert_eq!(mean(&[10]), Some(10.0));
    assert_eq!(mean(&[2, 4, 6, 8]), Some(5.0));
    assert_eq!(mean(&[1, 2, 3]), Some(2.0));

    assert_eq!(min_max(&[]), None);
    assert_eq!(min_max(&[5]), Some((5, 5)));
    assert_eq!(min_max(&[3, 1, 4, 1, 5, 9, 2, 6]), Some((1, 9)));
    assert_eq!(min_max(&[-10, 0, 10]), Some((-10, 10)));

    assert_eq!(range(&[]), None);
    assert_eq!(range(&[5]), Some(0));
    assert_eq!(range(&[3, 1, 4, 1, 5, 9, 2, 6]), Some(8));

    println!("PART 1 passed");

    // PART 2: Counting and filtering
    assert_eq!(count_positive(&[]), 0);
    assert_eq!(count_positive(&[-1, 0, 1, 2, -3]), 2);
    assert_eq!(count_positive(&[1, 2, 3, 4, 5]), 5);

    assert_eq!(count_where(&[1, 2, 3, 4, 5], |x| x > 2), 3);
    assert_eq!(count_where(&[1, 2, 3, 4, 5], |x| x % 2 == 0), 2);
    assert_eq!(count_where(&[], |x| x > 0), 0);

    assert_eq!(filter_values(&[1, 2, 3, 4, 5], |x| x > 2), vec![3, 4, 5]);
    assert_eq!(filter_values(&[1, 2, 3, 4, 5], |x| x % 2 == 0), vec![2, 4]);
    assert_eq!(filter_values(&[], |x| x > 0), Vec::<i32>::new());

    println!("PART 2 passed");

    // PART 3: Transformations
    assert_eq!(doubled(&[]), Vec::<i32>::new());
    assert_eq!(doubled(&[1, 2, 3]), vec![2, 4, 6]);
    assert_eq!(doubled(&[-5, 0, 5]), vec![-10, 0, 10]);

    assert_eq!(map_values(&[1, 2, 3], |x| x + 10), vec![11, 12, 13]);
    assert_eq!(map_values(&[1, 2, 3], |x| x * x), vec![1, 4, 9]);
    assert_eq!(map_values(&[], |x| x + 1), Vec::<i32>::new());

    assert_eq!(running_sum(&[]), Vec::<i32>::new());
    assert_eq!(running_sum(&[1, 2, 3, 4]), vec![1, 3, 6, 10]);
    assert_eq!(running_sum(&[10, -5, 3]), vec![10, 5, 8]);

    println!("PART 3 passed");

    // PART 4: Searching
    assert_eq!(find_first(&[1, 2, 3, 4, 5], 3), Some(2));
    assert_eq!(find_first(&[1, 2, 3, 4, 5], 99), None);
    assert_eq!(find_first(&[], 5), None);
    assert_eq!(find_first(&[5, 5, 5], 5), Some(0));

    assert_eq!(find_first_where(&[1, 2, 3, 4, 5], |x| x > 3), Some(3));
    assert_eq!(find_first_where(&[1, 2, 3], |x| x > 10), None);
    assert_eq!(find_first_where(&[2, 4, 6], |x| x % 2 == 1), None);

    println!("PART 4 passed");

    // PART 5: Reduction
    assert_eq!(reduce(&[1, 2, 3, 4], 0, |acc, x| acc + x), 10);
    assert_eq!(reduce(&[1, 2, 3, 4], 1, |acc, x| acc * x), 24);
    assert_eq!(reduce(&[], 42, |acc, x| acc + x), 42);
    assert_eq!(reduce(&[3, 7, 2, 8, 5], i32::MIN, |acc, x| if x > acc { x } else { acc }), 8);

    println!("PART 5 passed");

    // PART 6: Closure-returning functions
    let add5 = make_adder(5);
    assert_eq!(add5(10), 15);
    assert_eq!(add5(-3), 2);

    let add100 = make_adder(100);
    assert_eq!(add100(0), 100);

    let mut counter = make_counter();
    assert_eq!(counter(), 1);
    assert_eq!(counter(), 2);
    assert_eq!(counter(), 3);
    assert_eq!(counter(), 4);

    let mut another_counter = make_counter();
    assert_eq!(another_counter(), 1);          // independent counter

    println!("PART 6 passed");

    // PART 7: Classification
    assert_eq!(classify(-5), "negative");
    assert_eq!(classify(0), "zero");
    assert_eq!(classify(7), "small");
    assert_eq!(classify(50), "medium");
    assert_eq!(classify(500), "large");
    assert_eq!(classify(99), "medium");
    assert_eq!(classify(100), "large");

    let counts = classification_counts(&[-5, -2, 0, 3, 7, 15, 50, 99, 100, 500]);
    assert_eq!(counts, (2, 1, 2, 3, 2));
    //                  ^  ^  ^  ^  ^
    //              negative ze sm md large

    assert_eq!(classification_counts(&[]), (0, 0, 0, 0, 0));

    println!("PART 7 passed");

    println!("\nAll tests passed!");
}
```

---

## How to Approach This Lab

You can implement the functions in any order, but a few notes will help.

### General strategy

Most of these functions involve walking through the slice and computing something. You have several tools from the modules:

- **A `for` loop with `for x in values`** iterates over references to elements (Module 4). You will typically need to dereference (`*x`) to get the value out, since `i32` is `Copy`.
- **A `for` loop with `for i in 0..values.len()`** iterates over indices, useful when you need the position.
- **Iterator methods like `.iter().sum()`, `.iter().filter(...)`, `.iter().any(...)`** are mentioned in Module 4 and 5 and are often the most concise. The `iter()` returns references; remember the `**x` pattern in closures from Module 5.

For functions that take a closure (like `count_where`), the closure parameter has type `Fn(i32) -> bool`. When you call `predicate(x)` you pass an `i32` (not a reference). When iterating, you may need to dereference the iterator's reference.

### Tips for specific parts

- **Part 1 (statistics):** All four functions need to handle the empty case. `sum` returns 0; the others return `None`. You can use direct loops or iterator methods.
- **Part 2 (counting and filtering):** The generic versions parallel the standard library's `filter` and `count` methods. You can implement them with explicit loops if you prefer, or by calling the iterator methods.
- **Part 3 (transformations):** `doubled` is just `map_values` with a fixed closure; you may want to implement `map_values` first and call it from `doubled`.
- **Part 4 (searching):** Both `find_first` functions return `Option<usize>`. You can use a loop with early return when found.
- **Part 5 (reduction):** This is the classic "fold" pattern. You accumulate a value as you walk the slice.
- **Part 6 (closures):** `make_adder` returns a closure that captures `n` by value. `make_counter` returns a closure that owns and mutates an internal counter; declare it as `FnMut` since it changes state on each call. Notice the use of `let mut counter` in the test.
- **Part 7 (classification):** `classify` is a good case for `if/else if/else` or `match` with ranges (Module 4 covers both). `classification_counts` walks the slice once and counts each category.

### Avoiding over-engineering

Each function can be implemented in 1-5 lines of straightforward Rust. If you find yourself writing 15-line functions or reaching for advanced features, stop and reconsider.

The solution file uses iterators where they fit and explicit loops where they are clearer. Either style is acceptable; the goal is correct, readable code.

---

## Running Your Implementation

Build and run:

```bash
cargo run
```

Each `assert_eq!` panics if the assertion fails, telling you the line and the mismatch. If you implement everything correctly, the final output is:

```
PART 1 passed
PART 2 passed
PART 3 passed
PART 4 passed
PART 5 passed
PART 6 passed
PART 7 passed

All tests passed!
```

If you see a panic message like:

```
thread 'main' panicked at 'assertion `left == right` failed
  left: 0
  right: 15'
```

Look at the line number to identify which assertion failed. The `left` is what your function returned; the `right` is what was expected.

If you see `not yet implemented` panics, those are from `todo!()` macros that you have not replaced yet. Find the function and implement it.

---

## Stretch Goals (if you finish early)

If you finish before the hour is up, try these additional challenges. They are not required, but each exercises additional concepts from the three modules.

### Stretch 1 -- Median

Add a function that returns the median of the slice:

```rust
fn median(values: &[i32]) -> Option<f64> {
    todo!()
}
```

The median is the middle value of a sorted sequence. For an even-length sequence, it is the average of the two middle values. You may use `Vec::sort` on a clone of the input.

Test cases to add to `main`:

```rust
assert_eq!(median(&[]), None);
assert_eq!(median(&[5]), Some(5.0));
assert_eq!(median(&[1, 2, 3]), Some(2.0));
assert_eq!(median(&[1, 2, 3, 4]), Some(2.5));
assert_eq!(median(&[5, 1, 3, 9, 7]), Some(5.0));
```

### Stretch 2 -- Variance and standard deviation

```rust
fn variance(values: &[i32]) -> Option<f64> {
    todo!()
}

fn standard_deviation(values: &[i32]) -> Option<f64> {
    todo!()
}
```

Variance is the average of `(x - mean)²`. Standard deviation is the square root of variance. Use the population formula (divide by N, not N-1). Both return None for empty input. `standard_deviation` can be a one-liner that calls `variance`.

Test:

```rust
assert_eq!(variance(&[2, 4, 4, 4, 5, 5, 7, 9]), Some(4.0));
assert_eq!(standard_deviation(&[2, 4, 4, 4, 5, 5, 7, 9]), Some(2.0));
```

### Stretch 3 -- A more interesting closure-returner

Write a function `make_threshold_filter` that takes a threshold and returns a closure that, given a slice, returns the count of elements greater than the threshold:

```rust
fn make_threshold_filter(threshold: i32) -> impl Fn(&[i32]) -> usize {
    todo!()
}

// Test:
let above_5 = make_threshold_filter(5);
assert_eq!(above_5(&[1, 6, 3, 8, 5, 7]), 3);
```

### Stretch 4 -- Use labeled loops

Replace any of your search functions (like `find_first`) with a version that uses a labeled loop and `break 'label`. The behavior should be identical, but the implementation demonstrates Module 4's labeled-loop feature.

---

## Submission and Self-Assessment

When you have all assertions passing, look back at your code with these questions:

1. Are your functions doing what they say? Could a reader understand the function from its signature and body alone?
2. Did you use iterators or explicit loops? Was the choice deliberate?
3. Did you handle the empty-slice case explicitly, or did your iterator methods handle it naturally?
4. For functions with closure parameters, are the trait bounds (`Fn`, `FnMut`) correct?
5. For `make_counter`, did you understand why it needed `FnMut` instead of `Fn`?

If you can answer all of these, you have demonstrated understanding of the three modules. If any feel uncertain, revisit the relevant section before moving on to Module 6.

---

*End of Challenge Lab*
