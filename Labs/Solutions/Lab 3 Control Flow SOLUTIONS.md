# Module 4: Control Flow
## Lab 3 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- `if` as a Statement

### Checkpoints

**1. Two specific bugs prevented by the `bool` rule.**

The first is the classic confusion between assignment and equality. In C, `if (x = 0)` compiles, assigns 0 to `x`, and tests the assigned value (which is always falsy). The bug is silent: the program runs, but the condition never matches. In Rust, `if x = 0` is a syntax error, and `if x == 0` is the only way to test for equality. The `bool` rule helps catch this kind of typo because the assignment expression has type `()` rather than `bool`.

The second is integer-as-boolean confusion. In C, `if (count)` is a common idiom meaning "if count is non-zero," but it is ambiguous: is the test "is count non-zero" or "is count true"? In a codebase that uses `int` for both numeric values and flags, the same code pattern can have different intents in different places. Rust forces the developer to write `if count > 0` or `if count != 0`, which is unambiguous.

**2. What bug class does the brace requirement eliminate?**

The single best-known example is the Apple "goto fail" SSL bug from 2014. The original C code looked roughly like this:

```c
if (some_check())
    goto fail;
    goto fail;        // duplicate line, always executed
```

The second `goto fail` was outside the `if` because there were no braces. The bug bypassed certificate validation in Apple's SSL library and went undetected for months. Rust's mandatory braces make this impossible: the second `goto fail` would not be part of the `if` body whether the braces were there or not, but in Rust you cannot write the bug-prone form at all.

More generally, mandatory braces eliminate any ambiguity about scope. A reviewer can see the body's extent at a glance, and a developer adding a second statement cannot accidentally place it outside the conditional.

**3. Parentheses around conditions: easier to read?**

This is genuinely subjective. Some readers find parentheses clarifying because they delimit the condition visually. Others find them unnecessary noise because the braces already serve as a delimiter, and the visual separation between condition and body is clear without them.

The Rust choice (no parentheses) signals a design priority of minimizing required syntax. Rust includes a lot of mandatory punctuation in some places (turbofish, lifetimes, mandatory braces) but tries to avoid it where it adds nothing. The condition is bounded by the `if` keyword on one side and the opening brace on the other; parentheses would be redundant.

The same principle appears elsewhere in the language: no semicolons after function definitions, no parentheses around the `for` head, no required parens around tuple destructuring patterns in many contexts.

---

## Exercise 2 -- `if` as an Expression

### Checkpoints

**1. Why is the first branch the "expected" type?**

The compiler walks branches in source order. When it encounters the first branch, it has no constraints on the type, so it accepts whatever the branch produces and uses that as the expected type for everything that follows. When subsequent branches produce different types, the compiler reports them as not matching what was already established.

This is a quality-of-error-message decision more than a fundamental rule. The compiler could equally pick the last branch, or attempt to unify all branches into a common supertype. The "first branch sets the expected type" approach produces clearer error messages because the source-order convention is the same one developers use when reading the code.

If the developer wants to specify the type explicitly, they can annotate the binding (`let category: &str = ...`), and the compiler will then check every branch against that explicit type rather than against the first branch's inferred type.

**2. Why is `else` required when the value is used?**

If `else` were optional, the value of `category` would be undefined when no condition matched. Rust does not have implicit null, default, or undefined values to fill that gap.

The compiler enforces this by tracking whether the `if` expression has an "exit path" with no value. An `if` without `else` has two paths: the `if` branch (produces a value) and the implicit "neither" branch (produces nothing). Without `else`, the implicit branch is treated as producing the unit type `()`. So `if x > 0 { 42 }` actually has type `()` because the two branches do not unify.

When the value is discarded (`if x > 0 { println!("..."); }`), this works because both branches produce `()` (the `println!` returns unit, and the implicit branch also produces unit). But when the value is used (`let v = if ...`), the type mismatch becomes an error.

The rule is consistent and follows from the type system: every branch must produce a value of the same type, including the implicit "no condition matched" branch. Adding `else` makes the implicit branch explicit.

**3. A pattern where the expression form does not save much.**

When the conditional has side effects in addition to (or instead of) producing a value, the expression form is no shorter:

```rust
let result = if condition {
    println!("logging the true case");
    perform_work_a()
} else {
    println!("logging the false case");
    perform_work_b()
};
```

versus the statement form:

```rust
let result;
if condition {
    println!("logging the true case");
    result = perform_work_a();
} else {
    println!("logging the false case");
    result = perform_work_b();
}
```

The expression form is still slightly cleaner, but the savings are smaller because the side-effect lines are present in both versions. The expression form really shines when the branches are pure value-producing expressions.

Another case: when each branch is its own multi-line block of complex logic, the expression form is no shorter than the statement form, because the body is dominant. The expression form provides a guarantee (every branch produces a value) but does not compress the code.

---

## Exercise 3 -- Choosing Between `loop`, `while`, and `for`

### Task 3.6 -- Loop choice answers

1. **Process every element in an array of sensor readings:** `for`. The natural fit; you are walking a sequence.

2. **Repeatedly try to connect to a server until success:** `loop`. The exit condition (success) is computed inside the body, not at the top. `loop` with `break value` (covered in Exercise 5) is the natural shape.

3. **Read input until the user types "quit":** Either `loop` or `while` is reasonable. `loop` is slightly more idiomatic because the exit logic depends on the input read inside the body. `while` would require either reading input twice or initializing a sentinel value before the loop, both of which are awkward.

4. **Compute the factorial of n:** `for`. You are walking the range `1..=n` and multiplying. A `for` loop with a range is the natural shape. (Or, more idiomatically, `(1..=n).product()`.)

5. **Run a graphics event loop:** `loop`. The exit condition (window closed) is signaled by an internal event, not by a single condition checked at the top. Some game engines use `while running { ... }` instead, which is also reasonable.

### Checkpoints

**1. Why does the community care about `loop` vs `while true` if the runtime is identical?**

Three reasons:

- **Honesty about intent.** `loop` says "this loop runs until something inside breaks it." `while true` says "this loop runs while a condition is true," and the condition is meaningless. Reading `loop` is faster because the reader does not have to evaluate the trivial condition.
- **Optimizer friendliness.** The compiler treats `loop` as an unconditional loop and can prove certain properties (the loop body is reached unconditionally, certain optimizations are valid). With `while true`, the compiler usually figures it out, but treating it as conditional may inhibit some analyses.
- **Consistency with the language's expression model.** Only `loop` can produce values via `break`. Even when no value is needed, using `loop` keeps the option open for refactoring.

The deeper point is that style consistency in a community matters. When all Rust code uses `loop` for infinite loops, every reader recognizes the pattern instantly. When some uses `loop` and some uses `while true`, every reader has to wonder if there is a subtle reason for the choice.

**2. `continue` vs `if/else` for skipping values.**

Both work. Compare:

```rust
// With continue (early exit pattern)
for reading in readings {
    if reading == -999.0 {
        continue;
    }
    total += reading;
    count += 1;
}

// With nested if (positive condition pattern)
for reading in readings {
    if reading != -999.0 {
        total += reading;
        count += 1;
    }
}
```

The `continue` form is preferable when:
- There are multiple skip conditions (each gets its own early-exit, no nesting).
- The "happy path" is the dominant code in the loop body.
- The skip condition is exceptional (sentinel values, error cases).

The nested-`if` form is preferable when:
- There is a single condition and a single action.
- The condition is the natural expression of intent ("when X, do Y").

For the lab's example with one skip condition, both are reasonable. In real code with several validation checks (skip if negative, skip if out of range, skip if missing timestamp), the `continue` form scales better because each check becomes a single line at the top of the loop.

**3. Three constructs vs one general construct.**

What you gain by having more constructs:
- Each construct's intent is encoded in its keyword. The reader knows immediately what kind of loop they are looking at.
- The compiler can apply different rules to each. `loop` can produce values; `while` and `for` cannot. This rule would have to be expressed differently if there were one general loop construct.
- Idioms become easier to write. There is one obviously-right keyword for each kind of loop.

What you give up:
- Consistency. A language with one loop construct has fewer keywords to learn.
- Flexibility. A language might want to combine features (a `for` that produces a value, perhaps), which is harder when the constructs are separate.

The choice reflects Rust's general philosophy: prefer many specific constructs that capture intent over fewer general constructs that require the developer to express intent through composition. This is also why Rust has `Option`, `Result`, `Vec`, `HashMap`, and `HashSet` as separate types rather than one general "container" type. Specificity at the type level is a virtue.

---

## Exercise 4 -- Iterating with `for` and Ranges

### Checkpoints

**1. Why is the half-open range form more common?**

Half-open ranges match the conventions of array indexing. An array of length 5 has valid indices 0, 1, 2, 3, 4 (which is exactly `0..5`). When you write `for i in 0..arr.len()`, the half-open form gives you exactly the valid indices, no more and no less. With the closed form `0..=arr.len()`, you would get one extra iteration with an out-of-bounds index.

More generally, half-open ranges compose better in mathematical reasoning. The size of `start..end` is `end - start`. The start of the next adjacent range is just `end`. With closed ranges, both the size formula and the adjacency rule require adjustments by one, which is the classic source of off-by-one bugs.

The closed form `..=` exists because it is occasionally the right tool: when the upper bound is genuinely part of the range (like a temperature scale `0..=100`), the inclusive form expresses the intent more clearly. For loops over collection indices, half-open is almost always correct.

**2. Why does clippy flag the index-loop form even when the optimizer removes the bounds check?**

Clippy lints are about readability and idiom, not just performance. The reasons for flagging the index form include:

- **It is more verbose.** Three components in `enumerate` (`labels.iter()`, `.enumerate()`, the destructuring) versus three in the index form (`0..labels.len()`, `i`, `labels[i]`), but the `enumerate` form expresses intent ("iterate over each item, with index") while the index form expresses mechanism ("iterate from 0 to length, then look up").
- **It is easier to misuse.** A typo like `labels[i + 1]` accesses one past the index, which compiles, runs, and may panic. With `enumerate`, you get the value directly and there is no temptation to compute fancy indices.
- **It does not generalize.** `for i in 0..vec.len()` works for vectors and arrays but not for `HashMap`, `HashSet`, or any other collection that is not indexed by integer position. `for (key, value) in &map` works uniformly across all iterable types. Habituating to `enumerate` makes the same idiom work everywhere.

The runtime cost is one factor among several, and not the dominant one. Most clippy lints are about catching code that is correct but unidiomatic.

**3. Removing `mut` from the maximum-finding code.**

One option uses `iter().enumerate()` with `max_by`:

```rust
let (max_index, &max_so_far) = readings.iter()
    .enumerate()
    .max_by(|(_, a), (_, b)| a.partial_cmp(b).unwrap())
    .unwrap();
```

The `max_by` method walks the iterator and returns the element for which the comparison is greatest. The `partial_cmp` is needed because `f64` does not implement `Ord` (because of NaN), so we have to provide a comparison closure.

Another option uses `fold`:

```rust
let (max_index, max_so_far) = readings.iter()
    .copied()
    .enumerate()
    .fold((0, f64::NEG_INFINITY), |(best_idx, best), (idx, val)| {
        if val > best { (idx, val) } else { (best_idx, best) }
    });
```

Both versions have no `mut`. The `max_by` version is shorter and clearer for this specific task. The `fold` version is more general and shows how more complex accumulations can be expressed.

The point is not that iterator methods are always shorter (sometimes they are not), but that they can express the same thing without mutable state. Mutability is a tool; reducing it where natural is part of writing idiomatic Rust.

---

## Exercise 5 -- `loop` as an Expression with `break` Values

### Checkpoints

**1. Two advantages of `break value` over a mutable variable.**

First, the value cannot be observed before the loop completes. With a mutable variable initialized before the loop:

```rust
let mut result = String::new();    // initial value, sometimes wrong
loop {
    result = compute();
    if condition_met() { break; }
}
```

If something inside the loop reads `result` between iterations, it sees the previous (possibly invalid) value. With `break value`, the binding does not exist until the loop has completed:

```rust
let result = loop {
    let value = compute();
    if condition_met() { break value; }
};
```

Second, the type of the result is whatever `break` provides, with no need for a sentinel initial value. Compare:

```rust
let mut result: Option<String> = None;     // need Option to express "not yet"
loop {
    if let Some(value) = try_get() {
        result = Some(value);
        break;
    }
}
let result = result.unwrap();              // unwrap the Option

// vs

let result = loop {
    if let Some(value) = try_get() {
        break value;                       // unwraps inline
    }
};
```

The `break value` form removes the need for the `Option` wrapper and the `unwrap` call. It is shorter, has no risk of misuse, and removes one mutable variable.

**2. Why can `break` not return a value from `for`?**

A `for` loop ends in one of two ways:
- A `break` is executed inside the body.
- The iterator is exhausted (no more elements).

If `break value` were allowed in `for`, only the first exit path would have a value. The second exit path (normal exhaustion) would have no value. The compiler would have to either reject programs where the second path is possible, force the developer to provide a "default" value somehow, or use some sentinel. None of these is clean.

`loop` does not have this problem because the only way out is `break`. Every exit path produces a value. The type discipline works without exceptions.

The pragmatic answer for `for` is to use iterator methods that already model "exit on first match with a value or return None": `find`, `position`, `any`, `all`. These methods return `Option` or `bool`, expressing the "no match" case explicitly.

**3. When to use a manual `loop` over an iterator method.**

When the search has side effects beyond just finding a match. For example, logging, updating external state, or performing I/O during the iteration:

```rust
let result = loop {
    let attempt = next_attempt();
    log::info!("Trying attempt {attempt}");
    if let Ok(value) = perform(attempt) {
        log::info!("Succeeded after {attempt}");
        break value;
    }
    metrics::record_failure();
    if attempts_exhausted() { panic!("..."); }
};
```

The loop has multiple side-effects per iteration (logging, metrics, retry-limit checking) that would be awkward to fold into iterator method chains. The manual `loop` is the natural form.

When the search produces multiple values that need to be tracked together (best-so-far, count of candidates seen, etc.), and the accumulator is more complex than what `fold` cleanly expresses.

When the iteration itself is conditional and depends on dynamic computation that does not fit a fixed iterator. Retry loops with adaptive backoff are a classic case.

In short: iterator methods first, manual `loop` when the iterator methods do not naturally fit.

---

## Exercise 6 -- Labeled Loops and Nested Control

### Checkpoints

**1. How is the labeled-break form harder to misread than the flag-variable form?**

In the flag-variable form, the inner break and the outer break are separated. A reader has to:

1. See the `break` in the inner loop.
2. Remember that this only exits the inner loop.
3. Find the post-inner-loop check that re-tests the flag.
4. Verify that the flag-and-recheck logic is correct.

If the recheck is missing, the outer loop continues and does extra work. If the flag is checked but not reset between outer iterations, the loop behaves wrongly the second time around. Both bugs are easy to introduce when modifying the code later.

The labeled form has a single point of control. `break 'outer` is the explicit, named instruction to exit the outer loop. There is no flag to forget to set, no recheck to maintain. The reader sees one exit path and understands it immediately.

The general principle: control flow is easier to read when its mechanism is explicit. Flags are an implicit mechanism (they encode "the state of the search," which the reader has to reconstruct). Labels are explicit (they say "exit this specific loop").

**2. Why is function extraction usually preferred when feasible?**

Function extraction provides three benefits:

- **`return` is more universal than labels.** Any reader of any language understands `return`. Labels are unfamiliar to newcomers and look unusual.
- **The function gets a name.** `find_alarm(grid, threshold)` is self-documenting. The labeled-loop form requires the reader to understand the loop's purpose from context.
- **The function is reusable.** Once the search is a function, it can be called from anywhere. The labeled loop is tied to its surrounding code.

Function extraction is NOT feasible when:

- The loop body shares mutable state with surrounding code that would be awkward to pass as parameters and return.
- The early exit must update multiple local variables in the surrounding scope.
- The loop is genuinely one-off and naming it would add ceremony without clarity.

In practice, most nested loops where you reach for labels turn out to be search-shaped and benefit from extraction. Labels are most useful for state-machine-like loops where the iteration shares a lot of context with the surrounding function.

**3. A 50-line outer loop with nested continue points: still labels?**

Probably not. By the time the outer loop is 50 lines with multiple continue points, the function is already complicated. The fix is usually to break it up, not to add more labels.

Common refactors:

- **Extract the inner loop into its own function** that returns a status (e.g., `enum InnerResult { Continue, Skip, Break }`). The outer loop dispatches on the status.
- **Extract a helper function** that performs the operation that triggers the `continue 'outer`. The function returns early, and the outer loop only needs to handle a clean signal.
- **Restructure the algorithm.** Often a complicated nested loop indicates an algorithm that could be expressed differently (perhaps with iterator combinators or a state machine).

The labels work, but they tend to be a sign that the code is doing too much in one place. They are a tool for tactical complexity, not a substitute for good structure.

---

## Exercise 7 -- Putting It Together

### Task 7.2 -- Pattern identification

1. **`if` chain as an expression:** the `classify` function. The body is a single `if`/`else if`/`else` chain that produces a `&'static str`.

2. **`for` over a range:** `for second in (1..=3).rev()` in the countdown.

3. **`for` with `enumerate`:** `for (day, hourly) in daily_readings.iter().enumerate()` and the inner `for (hour, &reading) in hourly.iter().enumerate()`.

4. **`loop` with `break value`:** `let first_alarm = 'search: loop { ... break 'search Some(...) ... break 'search None };`

5. **Labeled loop:** `'search: loop { ... }`. The label is needed because the inner `for` loops would otherwise be the target of `break`, but we want to exit the outer `loop`.

6. **Reversed range:** `(1..=3).rev()` in the countdown.

### Checkpoints

**1. Why `&'static str` rather than `String`?**

The function `classify` returns one of four hard-coded string literals: `"alarm"`, `"warm"`, `"comfortable"`, `"cool"`. These are stored in the program's read-only data segment and exist for the entire lifetime of the program. Their type is `&'static str`: a borrowed reference to a string slice that lives for the static lifetime.

Returning `String` would require allocating a new `String` on every call: `String::from("alarm")`, etc. This involves a heap allocation, a copy of the bytes, and eventual deallocation when the `String` is dropped. For string literals that already exist in static memory, the allocation is wasted work.

`&'static str` lets the caller use the value without owning it. The caller can store it in a variable, pass it to other functions, or print it, all without taking ownership. The cost is zero: no allocation, no copy, just a pointer and a length.

The general rule: if the function is returning a constant value chosen from a fixed set, prefer `&'static str`. If the function is constructing a new string from runtime data (like formatting), `String` is the right return type.

**2. When to extract `find_first_alarm` into its own function.**

Push toward function extraction when:

- The search is a discrete operation that has a clear name. `find_first_alarm` is a perfectly reasonable function name.
- The search can be reused. If multiple parts of the program need to find the first alarm, the function is the natural unit of reuse.
- The surrounding `main` function is becoming too long. Extracting reduces the cognitive load of reading `main`.
- The function would have a reasonable type signature without too many parameters.

The labeled-loop form is preferable when:

- The search is so simple and one-off that extracting it adds more lines than it saves.
- The search uses local context that would be awkward to pass as parameters (less common in this example, where the inputs are clean).
- The code is a prototype or exploration where the structure is still being figured out.

For production code, the function form is almost always better. For a teaching example demonstrating labeled loops, the inline form serves the pedagogical purpose. The lab uses the labeled form to demonstrate the technique even though a real codebase would refactor it.

**3. Computing the average without `mut`.**

Yes, with iterator chaining. The grid is `[[f64; 7]; 3]`. To compute its average, flatten and sum:

```rust
let total: f64 = daily_readings.iter().flatten().sum();
let count = daily_readings.iter().flatten().count();
let average = total / count as f64;
```

The `flatten()` method takes an iterator of iterators (each row is iterable, and we have an iterator of rows) and produces a single iterator over all elements. Then `sum()` and `count()` work on the flattened sequence.

A more efficient version avoids the second pass:

```rust
let (total, count) = daily_readings.iter()
    .flatten()
    .fold((0.0, 0_u32), |(t, c), &r| (t + r, c + 1));
let average = total / count as f64;
```

`fold` walks the iterator once and accumulates both values together. This trades one mutable accumulator for an explicit fold function, which is a stylistic preference; both are correct and equally efficient (after optimization).

The simpler iterator chain (`.flatten().sum()` and a known count from the array dimensions, since the grid is fixed-size) would be even cleaner:

```rust
let total: f64 = daily_readings.iter().flatten().sum();
let count = (daily_readings.len() * daily_readings[0].len()) as f64;
let average = total / count;
```

There are several correct answers. The point of the exercise is to recognize that the imperative loop is one option among many, and to develop intuition for which one fits the situation.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept**

A common pattern: developers from C-family languages find the basic `if` and `while` immediately natural but struggle with `if` as an expression and `break value`. Both features are unusual enough that they require deliberate practice to internalize.

Iterator-based `for` loops can also be a stumbling block. Developers used to `for (int i = 0; i < n; i++)` need to retrain their reflexes to use `enumerate` and iterator methods.

The reverse pattern: developers from functional languages (Haskell, OCaml, Scala, F#) find `if` as an expression and value-producing loops natural, since their languages already work this way. They sometimes struggle with the imperative-style options because they have not used them in years.

**2. Three loop constructs vs one general construct**

Compared to Python, which has `for` and `while` (no `loop`):

- Rust gains: a clear keyword for infinite loops, the ability to produce values from a loop, more honest expression of intent.
- Rust costs: one more keyword to learn, slightly more decision-making at every loop site.

Compared to Java, which has `for`, `while`, `do-while`, and labeled break/continue (similar to Rust):

- The constructs are roughly equivalent. Rust adds `loop` for clarity and `break value` for expressiveness.
- Rust's `for` is built on iterators (more general); Java's traditional `for` is index-based and the enhanced `for` is collection-based.

The lab's lesson is that more constructs let each one express its intent more precisely. The cost is paid once, when learning the language; the benefit is paid every time a developer reads code and instantly knows which kind of loop they are looking at.

**3. A decision the reader changed**

This is necessarily personal. Common patterns:

- "I initially wrote a manual `loop` to find the first alarm in a list, then realized `iter().position()` does it in one line."
- "I initially used a flag variable to break out of nested loops, then refactored to a label after seeing the labeled form in another exercise."
- "I initially used `if/else` to skip sentinel values inside a loop, then realized `continue` makes the happy path more prominent."

The reflection is most valuable when the reader can identify a specific case where they switched approaches and articulate why.

---

*End of Lab 3 Solutions*
