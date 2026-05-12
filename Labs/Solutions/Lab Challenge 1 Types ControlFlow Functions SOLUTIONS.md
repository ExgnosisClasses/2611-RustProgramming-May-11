# Modules 3, 4, and 5: Types, Control Flow, and Functions
## Challenge Lab -- Complete Solution

> **Course:** Mastering Rust
> **Purpose:** This file contains one complete, working solution for every function
> in the challenge lab, along with notes on the choices made. Many other implementations
> are equally valid. The point is not to match this solution exactly but to understand
> why each piece works.

---

## Reading Order

This file has three parts:

1. **The complete `src/main.rs` solution** — one valid implementation of every function, ready to copy and run.
2. **Discussion of each function** — what concepts it exercises, why the implementation is the way it is, and alternative approaches.
3. **Stretch goal solutions** — for students who finished early.

If you finished the lab and want to verify your work, read Part 1. If you got stuck on specific functions, jump to those in Part 2. If you tackled the stretch goals, see Part 3.

---

## Part 1: The Complete Solution

```rust
// ============================================================
// Sequence Analyzer -- Solution
// ============================================================

fn sum(values: &[i32]) -> i32 {
    values.iter().sum()
}

fn mean(values: &[i32]) -> Option<f64> {
    if values.is_empty() {
        None
    } else {
        Some(sum(values) as f64 / values.len() as f64)
    }
}

fn min_max(values: &[i32]) -> Option<(i32, i32)> {
    if values.is_empty() {
        return None;
    }

    let mut min = values[0];
    let mut max = values[0];

    for &v in values.iter().skip(1) {
        if v < min {
            min = v;
        }
        if v > max {
            max = v;
        }
    }

    Some((min, max))
}

fn range(values: &[i32]) -> Option<i32> {
    match min_max(values) {
        Some((min, max)) => Some(max - min),
        None => None,
    }
}

fn count_positive(values: &[i32]) -> usize {
    count_where(values, |x| x > 0)
}

fn count_where<F>(values: &[i32], predicate: F) -> usize
where
    F: Fn(i32) -> bool,
{
    let mut count = 0;
    for &v in values {
        if predicate(v) {
            count += 1;
        }
    }
    count
}

fn filter_values<F>(values: &[i32], predicate: F) -> Vec<i32>
where
    F: Fn(i32) -> bool,
{
    let mut result = Vec::new();
    for &v in values {
        if predicate(v) {
            result.push(v);
        }
    }
    result
}

fn doubled(values: &[i32]) -> Vec<i32> {
    map_values(values, |x| x * 2)
}

fn map_values<F>(values: &[i32], transform: F) -> Vec<i32>
where
    F: Fn(i32) -> i32,
{
    let mut result = Vec::with_capacity(values.len());
    for &v in values {
        result.push(transform(v));
    }
    result
}

fn running_sum(values: &[i32]) -> Vec<i32> {
    let mut result = Vec::with_capacity(values.len());
    let mut running = 0;
    for &v in values {
        running += v;
        result.push(running);
    }
    result
}

fn find_first(values: &[i32], target: i32) -> Option<usize> {
    find_first_where(values, |x| x == target)
}

fn find_first_where<F>(values: &[i32], predicate: F) -> Option<usize>
where
    F: Fn(i32) -> bool,
{
    for i in 0..values.len() {
        if predicate(values[i]) {
            return Some(i);
        }
    }
    None
}

fn reduce<F>(values: &[i32], initial: i32, combine: F) -> i32
where
    F: Fn(i32, i32) -> i32,
{
    let mut acc = initial;
    for &v in values {
        acc = combine(acc, v);
    }
    acc
}

fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}

fn make_counter() -> impl FnMut() -> i32 {
    let mut count = 0;
    move || {
        count += 1;
        count
    }
}

fn classify(value: i32) -> &'static str {
    if value < 0 {
        "negative"
    } else if value == 0 {
        "zero"
    } else if value < 10 {
        "small"
    } else if value < 100 {
        "medium"
    } else {
        "large"
    }
}

fn classification_counts(values: &[i32]) -> (usize, usize, usize, usize, usize) {
    let mut negative = 0;
    let mut zero = 0;
    let mut small = 0;
    let mut medium = 0;
    let mut large = 0;

    for &v in values {
        match classify(v) {
            "negative" => negative += 1,
            "zero" => zero += 1,
            "small" => small += 1,
            "medium" => medium += 1,
            "large" => large += 1,
            _ => unreachable!(),
        }
    }

    (negative, zero, small, medium, large)
}

// (main function unchanged from starter code)
```

When run with `cargo run`, this prints:

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

---

## Part 2: Discussion of Each Function

### `sum` -- Module 5 (functions), Module 11 preview (iterator method)

```rust
fn sum(values: &[i32]) -> i32 {
    values.iter().sum()
}
```

The iterator method `sum()` walks the slice and adds all elements. For an empty slice, it returns 0 (the identity element for addition). This is the most concise form.

An explicit-loop alternative:

```rust
fn sum(values: &[i32]) -> i32 {
    let mut total = 0;
    for &v in values {
        total += v;
    }
    total
}
```

Both work. The iterator version is more idiomatic for this kind of reduction; the explicit loop makes the algorithm obvious to readers unfamiliar with iterators.

Note the `&v` pattern in the for loop. The iterator yields `&i32`; `&v` destructures the reference to bind `v` to the integer value directly. This avoids needing `*v` later.

### `mean` -- Module 3 (numeric types, casts), Module 5 (Option)

```rust
fn mean(values: &[i32]) -> Option<f64> {
    if values.is_empty() {
        None
    } else {
        Some(sum(values) as f64 / values.len() as f64)
    }
}
```

Two interesting points:

- **The empty-case check.** Without it, division by zero would produce `f64::NaN` rather than a panic, but returning `None` makes the empty case explicit. The function's signature (`Option<f64>`) tells callers to expect possible absence.
- **The `as` casts.** `sum(values)` returns `i32`; `values.len()` returns `usize`. Both need to be converted to `f64` before the division. The `as` operator performs the conversion. Without the casts, the compiler would reject the `/` because `i32` and `usize` are not directly compatible.

An alternative using `then`:

```rust
fn mean(values: &[i32]) -> Option<f64> {
    (!values.is_empty()).then(|| sum(values) as f64 / values.len() as f64)
}
```

This is more concise but uses a method (`bool::then`) that students may not yet have seen. The if/else form is clearer for this stage of the course.

### `min_max` -- Module 3 (tuples), Module 4 (loops, comparisons)

```rust
fn min_max(values: &[i32]) -> Option<(i32, i32)> {
    if values.is_empty() {
        return None;
    }

    let mut min = values[0];
    let mut max = values[0];

    for &v in values.iter().skip(1) {
        if v < min {
            min = v;
        }
        if v > max {
            max = v;
        }
    }

    Some((min, max))
}
```

A few points:

- **Initialize from the first element.** Starting with `i32::MAX` for min and `i32::MIN` for max would also work, but initializing from `values[0]` is cleaner and works correctly for single-element slices.
- **`iter().skip(1)`** iterates from the second element onward. The first element is already in min and max.
- **Two separate `if`s** rather than `if/else if`. A single element could be both the new min and the new max (compared to themselves on first iteration after skip), but more importantly, the two checks are independent and clearer this way.
- **Early return** with `return None` for the empty case. An alternative is to wrap the whole body in `if !values.is_empty() { ... } else { None }`, but early return reads more linearly.

The iterator method `iter().min()` and `iter().max()` exist and would each take O(n), so calling both separately would be O(2n). The combined loop is O(n). For small slices the difference is negligible; the combined form just feels more honest about what the function does.

### `range` -- Module 5 (Option matching)

```rust
fn range(values: &[i32]) -> Option<i32> {
    match min_max(values) {
        Some((min, max)) => Some(max - min),
        None => None,
    }
}
```

The function delegates to `min_max` and computes the difference. The `match` handles both cases: if `min_max` returned the pair, compute and wrap; if it returned `None`, propagate.

A more concise form uses `Option::map`:

```rust
fn range(values: &[i32]) -> Option<i32> {
    min_max(values).map(|(min, max)| max - min)
}
```

`map` transforms the inner value if present and propagates `None` otherwise. Both forms are valid; the explicit `match` is fine for this stage. The `?` operator would also work but is from Module 10.

### `count_positive` and `count_where` -- Module 5 (closures, higher-order functions)

```rust
fn count_positive(values: &[i32]) -> usize {
    count_where(values, |x| x > 0)
}

fn count_where<F>(values: &[i32], predicate: F) -> usize
where
    F: Fn(i32) -> bool,
{
    let mut count = 0;
    for &v in values {
        if predicate(v) {
            count += 1;
        }
    }
    count
}
```

`count_positive` delegates to `count_where` with a specific predicate. This shows the value of generic higher-order functions: one general implementation handles many cases via different closures.

The `where F: Fn(i32) -> bool` bound says "any type that can be called like a function taking `i32` and returning `bool`." Closures satisfy this; function pointers satisfy this; anything else implementing `Fn` does too.

The predicate is called with `predicate(v)`, passing an `i32` by value. This works because `i32` is `Copy`. For a non-`Copy` type, you would pass `&v` or accept a different signature.

An iterator-method version:

```rust
fn count_where<F>(values: &[i32], predicate: F) -> usize
where
    F: Fn(i32) -> bool,
{
    values.iter().filter(|&&x| predicate(x)).count()
}
```

The `|&&x|` is two levels of dereferencing: one for the iterator's `&i32`, one for the filter's reference to the iterator's reference. It works but is fiddly. The explicit loop is clearer.

### `filter_values` -- Module 5 (closures returning Vec)

```rust
fn filter_values<F>(values: &[i32], predicate: F) -> Vec<i32>
where
    F: Fn(i32) -> bool,
{
    let mut result = Vec::new();
    for &v in values {
        if predicate(v) {
            result.push(v);
        }
    }
    result
}
```

Parallel to `count_where` but accumulates a vector instead of a count. The pattern (walk, test, conditionally push) is one of the most common in Rust.

An iterator version with collect:

```rust
fn filter_values<F>(values: &[i32], predicate: F) -> Vec<i32>
where
    F: Fn(i32) -> bool,
{
    values.iter().copied().filter(|&x| predicate(x)).collect()
}
```

`copied()` turns the iterator of `&i32` into an iterator of `i32` (copying each, which is free for `Copy` types). The result is more concise but uses more iterator methods.

### `doubled`, `map_values` -- Module 5 (closures)

```rust
fn doubled(values: &[i32]) -> Vec<i32> {
    map_values(values, |x| x * 2)
}

fn map_values<F>(values: &[i32], transform: F) -> Vec<i32>
where
    F: Fn(i32) -> i32,
{
    let mut result = Vec::with_capacity(values.len());
    for &v in values {
        result.push(transform(v));
    }
    result
}
```

The same delegation pattern as `count_positive` to `count_where`. The closure type is `Fn(i32) -> i32`: takes an i32, returns an i32.

`Vec::with_capacity(values.len())` is a minor optimization. We know exactly how many elements the result will have, so we pre-allocate to avoid reallocating as the vector grows. For small slices this does not matter; for large ones it can be measurably faster.

### `running_sum` -- Module 3 (mutability), Module 4 (loops)

```rust
fn running_sum(values: &[i32]) -> Vec<i32> {
    let mut result = Vec::with_capacity(values.len());
    let mut running = 0;
    for &v in values {
        running += v;
        result.push(running);
    }
    result
}
```

A classic accumulator pattern. The `running` variable holds the sum so far; we update it and record the current value at each step.

Two mutable variables: `result` (the output) and `running` (the accumulator). Both need `mut` because they change inside the loop. Notice that the `running` variable is a *shadowing-friendly* name; if we wanted to make this point-free, we could use a shadowed binding in each iteration, but the mutable approach is much clearer here.

The iterator version uses `scan`:

```rust
fn running_sum(values: &[i32]) -> Vec<i32> {
    values
        .iter()
        .scan(0, |acc, &x| {
            *acc += x;
            Some(*acc)
        })
        .collect()
}
```

`scan` is `fold` that yields intermediate results. The closure receives a mutable reference to the accumulator and returns `Option`; `None` stops the iteration. This is concise but introduces a method students may not yet have seen.

### `find_first` and `find_first_where` -- Module 4 (early return), Module 5 (closures)

```rust
fn find_first(values: &[i32], target: i32) -> Option<usize> {
    find_first_where(values, |x| x == target)
}

fn find_first_where<F>(values: &[i32], predicate: F) -> Option<usize>
where
    F: Fn(i32) -> bool,
{
    for i in 0..values.len() {
        if predicate(values[i]) {
            return Some(i);
        }
    }
    None
}
```

The `for i in 0..values.len()` form iterates over indices. We need the index for the return value, so this is appropriate.

The early `return Some(i)` short-circuits at the first match. The function does not continue after finding the answer; it returns immediately. The final `None` runs only if the loop completes without finding a match.

An alternative uses `enumerate` to get index-value pairs:

```rust
fn find_first_where<F>(values: &[i32], predicate: F) -> Option<usize>
where
    F: Fn(i32) -> bool,
{
    for (i, &v) in values.iter().enumerate() {
        if predicate(v) {
            return Some(i);
        }
    }
    None
}
```

This is slightly more idiomatic. The `enumerate` adapter pairs each element with its index, and the loop destructures both.

The iterator method `position` does exactly this:

```rust
fn find_first_where<F>(values: &[i32], predicate: F) -> Option<usize>
where
    F: Fn(i32) -> bool,
{
    values.iter().position(|&x| predicate(x))
}
```

But it uses `position` from Module 11, so the explicit loop is appropriate for this stage.

### `reduce` -- Module 5 (closure with two arguments)

```rust
fn reduce<F>(values: &[i32], initial: i32, combine: F) -> i32
where
    F: Fn(i32, i32) -> i32,
{
    let mut acc = initial;
    for &v in values {
        acc = combine(acc, v);
    }
    acc
}
```

The classic fold pattern. The `combine` closure takes the accumulator and the current value, returns the new accumulator. The function walks the slice updating the accumulator.

The signature `Fn(i32, i32) -> i32` says "a closure taking two `i32`s and returning an `i32`." This is what `combine(acc, v)` requires.

For the empty slice, the loop runs zero times, and `acc` is still `initial`. This is what the assertion `reduce(&[], 42, |acc, x| acc + x) == 42` requires.

The interesting test case is the one that uses `reduce` to compute the maximum:

```rust
reduce(&[3, 7, 2, 8, 5], i32::MIN, |acc, x| if x > acc { x } else { acc })
```

Starting from `i32::MIN`, the first comparison will always update to the actual first element. After that, the accumulator holds the largest value seen. This is `max` written as a fold.

### `make_adder` -- Module 5 (closures returning closures, capture by value)

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}
```

The `move` keyword forces the closure to take ownership of `n` (rather than borrowing it). This is necessary because the closure outlives the function `make_adder`; without `move`, the closure would hold a reference to `n`, but `n` would go out of scope when `make_adder` returns.

For `Copy` types like `i32`, "moving" is just copying. Each closure produced by `make_adder` gets its own copy of `n`. Multiple calls produce closures with different captured values:

```rust
let add5 = make_adder(5);
let add100 = make_adder(100);
// add5 and add100 are independent closures with different captured n
```

The return type `impl Fn(i32) -> i32` says "some specific type that implements `Fn(i32) -> i32`." The compiler picks the concrete type (a unique anonymous type for each closure expression). Callers do not need to know the specific type; they only need to know it can be called like a function.

### `make_counter` -- Module 5 (FnMut, mutable captured state)

```rust
fn make_counter() -> impl FnMut() -> i32 {
    let mut count = 0;
    move || {
        count += 1;
        count
    }
}
```

Two important details:

- **`impl FnMut() -> i32`** rather than `impl Fn() -> i32`. The closure modifies its captured `count`, so it cannot be `Fn` (which requires `&self`). `FnMut` is `&mut self`, which allows modification.
- **`move`** moves `count` into the closure. The closure owns `count` and modifies it across calls.

Each call to `make_counter` produces an independent closure with its own `count`. The test verifies this:

```rust
let mut counter = make_counter();
counter();    // 1
counter();    // 2
let mut another = make_counter();
another();    // 1 (independent)
```

Note the `let mut counter` at the call site. To call an `FnMut` closure, the caller's binding must be mutable. Without the `mut`, the call fails with "cannot borrow as mutable."

This is one of the most common closure mistakes for students new to Rust. The trait bound `FnMut` requires the caller to allow mutation; the closure can hold mutable state.

### `classify` -- Module 4 (if/else if chain)

```rust
fn classify(value: i32) -> &'static str {
    if value < 0 {
        "negative"
    } else if value == 0 {
        "zero"
    } else if value < 10 {
        "small"
    } else if value < 100 {
        "medium"
    } else {
        "large"
    }
}
```

The `if/else if` chain handles ordered ranges naturally. Each condition is checked in order; the first true one wins.

An alternative using `match` with ranges:

```rust
fn classify(value: i32) -> &'static str {
    match value {
        v if v < 0 => "negative",
        0 => "zero",
        1..=9 => "small",
        10..=99 => "medium",
        _ => "large",
    }
}
```

This works and is more declarative. The first arm uses a guard (`if v < 0`) because `match` cannot directly express "anything less than 0" with the `1..=9` style syntax for negative ranges that vary by lower bound. Both forms are idiomatic; if/else if is simpler when the categories are sequential ranges.

The return type `&'static str` is a string slice with the `'static` lifetime: it lives for the entire program. String literals like `"negative"` are baked into the binary and have `&'static` lifetime, so returning them is straightforward. This is the right return type for fixed string responses.

### `classification_counts` -- Module 3 (tuples), Module 4 (match), Module 5 (functions)

```rust
fn classification_counts(values: &[i32]) -> (usize, usize, usize, usize, usize) {
    let mut negative = 0;
    let mut zero = 0;
    let mut small = 0;
    let mut medium = 0;
    let mut large = 0;

    for &v in values {
        match classify(v) {
            "negative" => negative += 1,
            "zero" => zero += 1,
            "small" => small += 1,
            "medium" => medium += 1,
            "large" => large += 1,
            _ => unreachable!(),
        }
    }

    (negative, zero, small, medium, large)
}
```

Five mutable accumulators, one per category. The loop walks the slice and increments the appropriate counter based on `classify`'s return value.

The `match` arms are string literals. The `_ => unreachable!()` handles the "no other classification exists" case; `unreachable!()` panics if reached, signaling a bug. Since `classify` only returns one of the five values, the unreachable arm should never run.

The return type `(usize, usize, usize, usize, usize)` is a tuple of five elements. The final expression `(negative, zero, small, medium, large)` constructs the tuple.

A more "scalable" version might use a `HashMap`, but that is from Module 11. For five fixed categories, separate variables and a tuple work fine.

An alternative without going through string-matching would replicate the classify logic inline:

```rust
fn classification_counts(values: &[i32]) -> (usize, usize, usize, usize, usize) {
    let mut counts = (0, 0, 0, 0, 0);

    for &v in values {
        if v < 0 {
            counts.0 += 1;
        } else if v == 0 {
            counts.1 += 1;
        } else if v < 10 {
            counts.2 += 1;
        } else if v < 100 {
            counts.3 += 1;
        } else {
            counts.4 += 1;
        }
    }

    counts
}
```

This avoids the string comparison and the `unreachable!` arm. The tuple-field access `counts.0`, `counts.1`, etc. is the syntax for tuple elements. Both versions work; the named-variables version is slightly more readable for humans, the inline-if version slightly more efficient.

---

## Part 3: Stretch Goal Solutions

### Stretch 1 -- Median

```rust
fn median(values: &[i32]) -> Option<f64> {
    if values.is_empty() {
        return None;
    }

    let mut sorted = values.to_vec();
    sorted.sort();

    let n = sorted.len();
    if n % 2 == 1 {
        Some(sorted[n / 2] as f64)
    } else {
        let lower = sorted[n / 2 - 1] as f64;
        let upper = sorted[n / 2] as f64;
        Some((lower + upper) / 2.0)
    }
}
```

The implementation:

- **Clone the input** to avoid modifying the caller's data. `values.to_vec()` creates an owned `Vec<i32>` containing the same values.
- **Sort the copy.** `sort()` requires the elements to implement `Ord`, which `i32` does.
- **Odd length:** the middle element is at index `n/2` (integer division).
- **Even length:** the two middle elements are at `n/2 - 1` and `n/2`; their average is the median.

Test cases:

```rust
assert_eq!(median(&[]), None);
assert_eq!(median(&[5]), Some(5.0));
assert_eq!(median(&[1, 2, 3]), Some(2.0));
assert_eq!(median(&[1, 2, 3, 4]), Some(2.5));
assert_eq!(median(&[5, 1, 3, 9, 7]), Some(5.0));    // sorts to [1, 3, 5, 7, 9], median 5
```

### Stretch 2 -- Variance and Standard Deviation

```rust
fn variance(values: &[i32]) -> Option<f64> {
    let m = mean(values)?;
    let n = values.len() as f64;
    let sum_squared_diffs: f64 = values
        .iter()
        .map(|&x| {
            let diff = x as f64 - m;
            diff * diff
        })
        .sum();
    Some(sum_squared_diffs / n)
}

fn standard_deviation(values: &[i32]) -> Option<f64> {
    variance(values).map(|v| v.sqrt())
}
```

Wait — this uses `?` and `Option::map`, which the stretch goal doesn't prohibit but the main lab does. An alternative without `?`:

```rust
fn variance(values: &[i32]) -> Option<f64> {
    if values.is_empty() {
        return None;
    }

    let m = match mean(values) {
        Some(m) => m,
        None => return None,
    };

    let n = values.len() as f64;
    let mut sum_squared_diffs = 0.0;
    for &x in values {
        let diff = x as f64 - m;
        sum_squared_diffs += diff * diff;
    }

    Some(sum_squared_diffs / n)
}

fn standard_deviation(values: &[i32]) -> Option<f64> {
    match variance(values) {
        Some(v) => Some(v.sqrt()),
        None => None,
    }
}
```

For `[2, 4, 4, 4, 5, 5, 7, 9]`: mean is 5, squared differences are `[9, 1, 1, 1, 0, 0, 4, 16]`, sum is 32, divided by 8 is 4. Standard deviation is `sqrt(4) = 2`. ✓

### Stretch 3 -- Threshold filter closure

```rust
fn make_threshold_filter(threshold: i32) -> impl Fn(&[i32]) -> usize {
    move |slice| {
        let mut count = 0;
        for &v in slice {
            if v > threshold {
                count += 1;
            }
        }
        count
    }
}
```

The function returns a closure that takes a slice and returns a count. The threshold is captured by value (via `move`). Each call to `make_threshold_filter` produces an independent closure with its own captured threshold.

The closure could also delegate to `count_where`:

```rust
fn make_threshold_filter(threshold: i32) -> impl Fn(&[i32]) -> usize {
    move |slice| count_where(slice, |x| x > threshold)
}
```

This is more concise but introduces nested closures: the outer one captures `threshold`, the inner captures `threshold` again from the outer closure's scope. Both work; the explicit loop version is clearer for students new to closures.

Test:

```rust
let above_5 = make_threshold_filter(5);
assert_eq!(above_5(&[1, 6, 3, 8, 5, 7]), 3);    // 6, 8, 7
```

### Stretch 4 -- Labeled loops

The `find_first_where` function with a labeled loop:

```rust
fn find_first_where<F>(values: &[i32], predicate: F) -> Option<usize>
where
    F: Fn(i32) -> bool,
{
    let result = 'search: loop {
        for i in 0..values.len() {
            if predicate(values[i]) {
                break 'search Some(i);
            }
        }
        break 'search None;
    };
    result
}
```

The `'search:` label names the outer loop. `break 'search value` exits the labeled loop with the value as the loop's result. The outer `loop` runs at most once (we always break out of it); the inner `for` loop does the actual searching.

This is contrived for a single-loop search; labeled loops are more useful when you have nested loops and need to break out of an outer one from an inner one. But it does demonstrate the syntax.

A cleaner use of labeled loops would be when searching a 2D structure:

```rust
'outer: for row in matrix {
    for &cell in row {
        if cell == target {
            break 'outer;       // breaks the outer loop, not just inner
        }
    }
}
```

For the single-loop case, the original implementation is clearer.

---

## Common Mistakes and Recovery

Things students typically get wrong on this lab:

**Forgetting `mut` on the counter binding.** The test code has `let mut counter = make_counter()`. If students write `let counter = make_counter()`, the compiler will reject `counter()` with "cannot borrow as mutable." This is the most common confusion around `FnMut`.

**Forgetting `move` on closures returning from functions.** `make_adder(n: i32)` and `make_counter()` both need `move` on their inner closures. Without it, the closures try to borrow from the function's local scope, which has ended by the time the closure is called. The compiler usually catches this clearly, but the error message can confuse beginners.

**Using `*x` versus `&v` versus just `x`.** Different iteration forms produce different things:

- `for x in &vec` yields references; you typically need `*x` to use the value (for `Copy` types).
- `for &v in &vec` destructures the reference, binding `v` to the value directly.
- `for v in vec` (no `&`) consumes the vector; rare and usually not what you want.

The `&v` form is preferred in this lab because the function bodies usually want the value directly.

**Off-by-one in `running_sum`.** Some implementations push the running total before adding the current element, producing `[0, 1, 3, 6]` instead of `[1, 3, 6, 10]`. Read the test cases carefully; the expected output is the sum *after* including each element.

**Returning the wrong tuple order in `classification_counts`.** The order is fixed in the signature: `(negative, zero, small, medium, large)`. Returning in a different order will fail the assertions even if the counts are correct.

**Using `Vec<i32>` instead of `&[i32]` for slice parameters.** Some students reach for `Vec` because it is more familiar. The lab specifies `&[i32]` because slices are the idiomatic parameter type, but changing the signatures is technically allowed if the assertions still pass. (They will not pass with literals like `sum(&[1, 2, 3])` if the function takes `Vec<i32>` instead.)

---

## What This Lab Demonstrates

If you wrote your own implementations and made the tests pass, you have demonstrated:

- Understanding of Rust's basic types and casting (Module 3).
- Comfortable use of mutable variables with explicit `mut`.
- Tuple construction, return, and destructuring.
- Function definition with parameters, return types, and bodies.
- The expression-vs-statement distinction (functions return their last expression).
- Use of `Option<T>` to represent optional results.
- Use of `if`, `else if`, `else` expressions and chains.
- Use of `for` loops with various iteration styles.
- Early return from a function inside a loop.
- The use of `match` on simple types.
- Generic functions with trait bounds (`F: Fn(i32) -> bool`).
- Closures that capture by value, including the `move` keyword.
- The distinction between `Fn` and `FnMut`.
- Returning closures from functions using `impl Trait`.

This is the foundation for everything that comes after. Modules 6 (ownership) and 7 (lifetimes) build on this with explicit memory management; the rest of the course extends it with traits, generics, error handling, concurrency, and the broader ecosystem.

---

*End of Challenge Lab Solution*
