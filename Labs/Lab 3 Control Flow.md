# Module 4: Control Flow
## Lab 3 -- Working with Conditionals, Loops, and Iterators

> **Course:** Mastering Rust
> **Module:** 4 - Control Flow
> **Estimated time:** 75-90 minutes

---

## Overview

This lab gives you hands-on experience with every control flow construct from Module 4: `if` expressions, the three loop forms, ranges, `break` with values, and labeled loops. You will build a single Cargo project called `weather_monitor` that processes streams of temperature readings and produces summaries. The project starts with simple conditionals and grows to use nested loops, search patterns, and value-producing loops.

The focus is on the constructs themselves and on choosing the right one for each situation. Many of the exercises have multiple correct solutions; the lab guides you toward the idiomatic choice and asks you to compare alternatives.

By the end of this lab you will be able to:

- Use `if` and `else if` chains as expressions and as statements.
- Choose between `loop`, `while`, and `for` based on what the code expresses.
- Iterate over ranges, arrays, and slices using each of the three reference forms.
- Replace C-style index loops with `enumerate()` and other iterator patterns.
- Return values from `loop` blocks using `break value`.
- Use loop labels to control nested control flow when no clean alternative exists.
- Read compiler errors to identify type mismatches, missing branches, and the rules of expression-style control flow.

---

## Before You Start

This lab requires the environment from Lab 1 and assumes familiarity with the type system from Lab 2. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** The lab uses small but realistic-looking weather data. The numbers are not from any particular location; they are illustrative.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new weather_monitor
cd weather_monitor
code .
```

Create a `lab3-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- `if` as a Statement

**Estimated time:** 10 minutes
**Topics covered:** basic `if`/`else if`/`else`, the `bool` requirement, mandatory braces

### Context

The familiar `if` statement is the natural starting point. The Rust syntax is similar to C and Java, with two important differences: parentheses around the condition are optional (and conventionally omitted), and braces around the body are required, even for single-statement bodies. This exercise establishes the syntax and the strict `bool` rule.

### Task 1.1 -- Categorize a temperature

Replace the contents of `src/main.rs` with:

```rust
fn main() {
    let temperature = 22.5;

    if temperature > 30.0 {
        println!("Hot");
    } else if temperature > 20.0 {
        println!("Warm");
    } else if temperature > 10.0 {
        println!("Cool");
    } else {
        println!("Cold");
    }
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
Warm
```

Try a few different values for `temperature` and confirm that each branch is reachable. Record the values you tried and the categories they produced in `lab3-notes.md`.

### Task 1.2 -- Discover the bool requirement

Add the following inside `main`:

```rust
    let count = 5;

    if count {
        println!("there are some");
    }
```

Run `cargo build`. The compiler rejects this with:

```
error[E0308]: mismatched types
  |
  |     if count {
  |        ^^^^^ expected `bool`, found integer
```

Rust does not have truthiness. The condition must produce a `bool` directly. Fix the code by writing the comparison explicitly:

```rust
    let count = 5;

    if count > 0 {
        println!("there are some");
    }
```

Run again. The program now compiles and prints both messages.

### Task 1.3 -- The brace requirement

Try this single-line form:

```rust
    let temperature = 22.5;

    if temperature > 20.0 println!("Warm");
```

Run `cargo build`. The compiler rejects this. The error message points to the missing braces. Unlike C, Java, and JavaScript, Rust requires braces even when the body is a single statement. This eliminates a class of bugs where adding a second statement to an `if` body silently changes the program's behavior because the new statement is not actually inside the conditional.

Fix it by adding braces:

```rust
    if temperature > 20.0 {
        println!("Warm");
    }
```

### Checkpoints

1. Rust requires conditions to be `bool` and rejects integer truthiness. Identify two specific bugs that this rule prevents in everyday code.
2. The braces around `if` bodies are mandatory. Many languages make them optional for single statements. What real-world bug class does Rust eliminate by requiring them?
3. Rust does not require parentheses around the condition. Some languages require them. Which is easier to read in your opinion, and what does the choice signal about the language's design priorities?

---

## Exercise 2 -- `if` as an Expression

**Estimated time:** 10-15 minutes
**Topics covered:** `if` expressions, type unification across branches, mandatory `else`

### Context

In Rust, an `if` block produces a value. This is one of the language's more useful features and replaces a common imperative pattern from other languages. Once you internalize it, you will use it more often than the statement form.

### Task 2.1 -- Use `if` in a `let` binding

Replace `main` with:

```rust
fn main() {
    let temperature = 22.5;

    let category = if temperature > 30.0 {
        "Hot"
    } else if temperature > 20.0 {
        "Warm"
    } else if temperature > 10.0 {
        "Cool"
    } else {
        "Cold"
    };

    println!("Today is {category}");
}
```

Run the program. Expected output:

```
Today is Warm
```

Notice three things:

- The entire `if` chain is one expression that produces a single string slice.
- Inside each branch, `"Warm"` has no semicolon; the value of the branch is the value of its last expression.
- The semicolon at the end of the `let` statement is at the very end, after the closing brace of the final `else`.

In `lab3-notes.md`, write down the equivalent code in another language you know (Python, Java, C, or JavaScript). How does the line count and clarity compare?

### Task 2.2 -- Discover the type rule

Try this version, where one branch produces a different type:

```rust
    let category = if temperature > 30.0 {
        "Hot"
    } else if temperature > 20.0 {
        "Warm"
    } else {
        42
    };
```

Run `cargo build`. The compiler produces an error similar to:

```
error[E0308]: `if` and `else` have incompatible types
  |
  |         "Warm"
  |         ------ expected because of this
  |     } else {
  |         42
  |         ^^ expected `&str`, found integer
```

When `if` is an expression, every branch must produce the same type. The compiler must assign one type to the whole expression and refuses to mix incompatible branches.

Fix the code by making all branches return strings, then continue.

### Task 2.3 -- Discover the `else` requirement

Try this version with no `else`:

```rust
    let category = if temperature > 30.0 {
        "Hot"
    };
```

Run `cargo build`. The compiler produces:

```
error[E0317]: `if` may be missing an `else` clause
```

When the value of `if` is used (here, bound to `category`), an `else` is mandatory. Without it, the value would be undefined when the condition is false. Rust does not have an implicit "default" value to fill in.

If you discard the value (use `if` purely for its side effects), `else` is optional:

```rust
    if temperature > 30.0 {
        println!("Hot day ahead");
    }
```

Restore the working version from Task 2.1 before continuing.

### Task 2.4 -- A practical use of `if` as expression

Replace `main` with:

```rust
fn main() {
    let readings = [21.5, 22.0, 22.8, 23.1, 22.5];

    let total: f64 = readings.iter().sum();
    let average = total / readings.len() as f64;

    let trend = if readings[readings.len() - 1] > readings[0] {
        "rising"
    } else if readings[readings.len() - 1] < readings[0] {
        "falling"
    } else {
        "steady"
    };

    let comfort = if average > 25.0 {
        "warm"
    } else if average > 18.0 {
        "comfortable"
    } else {
        "cool"
    };

    println!("Average: {average:.2}, trend: {trend}, comfort: {comfort}");
}
```

Run the program. Expected output:

```
Average: 22.38, trend: rising, comfort: comfortable
```

Both `trend` and `comfort` are computed in single expressions with no mutable state. Compare this to the imperative alternative, which would require declaring `let mut trend = ""` before the conditional and assigning inside each branch.

### Checkpoints

1. Task 2.2 showed that branches must agree on type. The compiler error pointed at the type of the first branch and called it "expected." Why does the first branch determine the expected type rather than the last?
2. Why is `else` required when the value of `if` is used, but optional when it is not? Explain the rule in terms of what value `category` would have if no branch matched.
3. The expression form replaces a pattern where you would declare a mutable variable and assign to it from inside each branch. Identify a code pattern from another language where the expression form does NOT save much, even though it is technically applicable.

---

## Exercise 3 -- Choosing Between `loop`, `while`, and `for`

**Estimated time:** 15 minutes
**Topics covered:** the three loop constructs, when each is the right choice, `break` and `continue`

### Context

Rust has three loop keywords, and they are not interchangeable. Each one expresses a different intent, and choosing the right one makes the code's purpose clearer. This exercise walks through each construct and asks you to apply judgment about which one fits.

### Task 3.1 -- The `loop` construct

Replace `main` with:

```rust
fn main() {
    let mut attempts = 0;

    loop {
        attempts += 1;
        println!("Attempt {attempts}");

        if attempts >= 3 {
            println!("Giving up after {attempts} attempts");
            break;
        }
    }

    println!("Done");
}
```

Run the program. Expected output:

```
Attempt 1
Attempt 2
Attempt 3
Giving up after 3 attempts
Done
```

The `loop` construct repeats forever until something inside it executes `break`. This shape is appropriate when the exit condition is computed inside the body, not at the top.

### Task 3.2 -- The `while` construct

Replace `main` with:

```rust
fn main() {
    let mut countdown = 5;

    while countdown > 0 {
        println!("{countdown}");
        countdown -= 1;
    }

    println!("Liftoff");
}
```

Run the program. Expected output:

```
5
4
3
2
1
Liftoff
```

The `while` construct checks a condition before each iteration. It is the right choice when there is a single, clean condition that controls whether to continue.

### Task 3.3 -- The `for` construct

Replace `main` with:

```rust
fn main() {
    let temperatures = [21.5, 22.0, 22.8, 23.1, 22.5];

    for temp in temperatures {
        println!("Reading: {temp}");
    }
}
```

Run the program. Expected output:

```
Reading: 21.5
Reading: 22.0
Reading: 22.8
Reading: 23.1
Reading: 22.5
```

The `for` construct walks every element produced by an iterator. For sequences (arrays, vectors, slices), this is by far the most common loop in idiomatic Rust.

### Task 3.4 -- The non-idiomatic infinite loop

Replace `main` with:

```rust
fn main() {
    let mut counter = 0;

    while true {
        counter += 1;
        if counter == 3 {
            break;
        }
        println!("Iteration {counter}");
    }

    println!("Counter ended at {counter}");
}
```

The program compiles and runs. But run `cargo clippy`. Clippy reports:

```
warning: denote infinite loops with `loop { ... }`
```

`loop` is the idiomatic way to express an infinite loop. It is more honest about the intent and gives the optimizer better information. Fix the code by replacing `while true` with `loop`:

```rust
    loop {
        counter += 1;
        if counter == 3 {
            break;
        }
        println!("Iteration {counter}");
    }
```

Run `cargo clippy` again. The warning is gone.

### Task 3.5 -- Using `continue`

`continue` skips the rest of the current iteration and starts the next one. Replace `main` with:

```rust
fn main() {
    let readings = [21.5, -999.0, 22.0, 22.8, -999.0, 23.1, 22.5];

    let mut total = 0.0;
    let mut count = 0;

    for reading in readings {
        if reading == -999.0 {
            continue;       // skip sentinel values
        }
        total += reading;
        count += 1;
    }

    let average = total / count as f64;
    println!("Average of {count} valid readings: {average:.2}");
}
```

Run the program. Expected output:

```
Average of 5 valid readings: 22.38
```

The `-999.0` sentinel values are skipped; the calculation only counts real readings.

### Task 3.6 -- Apply judgment

For each scenario below, decide whether `loop`, `while`, or `for` is the most appropriate choice. Record your answers in `lab3-notes.md`.

1. Process every element in an array of sensor readings.
2. Repeatedly try to connect to a server until the connection succeeds.
3. Read input from the user until they type "quit."
4. Compute the factorial of `n` (multiply 1 * 2 * 3 * ... * n).
5. Run an event loop in a graphics application, exiting only when the user closes the window.

### Checkpoints

1. `while true` and `loop` produce the same machine code. Clippy still flags one. Why does the language community care about this distinction if the runtime behavior is identical?
2. The Task 3.5 program uses `continue` to skip sentinel values. Could the same effect be achieved with `if` and `else`? Compare the two approaches for readability.
3. Some languages (Python, Go) have a `for` loop that is general enough to subsume `while` and `loop`. Rust has three separate constructs. What do you gain by having more constructs, and what do you give up?

---

## Exercise 4 -- Iterating with `for` and Ranges

**Estimated time:** 15 minutes
**Topics covered:** range operators, iteration by value/reference/mutable reference, `enumerate`, `rev`

### Context

The `for` loop in Rust does not work directly with integer indices. To loop over a range of numbers, you create a range iterator with `..` or `..=`. To loop over a collection while still seeing indices, you use `.enumerate()`. These patterns replace the C-style `for (int i = 0; ...)` loop entirely.

### Task 4.1 -- Range operators

Replace `main` with:

```rust
fn main() {
    println!("Half-open range 0..5:");
    for i in 0..5 {
        print!("{i} ");
    }
    println!();

    println!("Closed range 0..=5:");
    for i in 0..=5 {
        print!("{i} ");
    }
    println!();

    println!("Reversed 1..=5:");
    for i in (1..=5).rev() {
        print!("{i} ");
    }
    println!();
}
```

Run the program. Expected output:

```
Half-open range 0..5:
0 1 2 3 4 
Closed range 0..=5:
0 1 2 3 4 5 
Reversed 1..=5:
5 4 3 2 1 
```

Note three things:

- `0..5` includes 0 but excludes 5. This matches array indexing conventions.
- `0..=5` includes both endpoints. Used when the upper bound is part of the range conceptually.
- `(1..=5).rev()` requires the parentheses, otherwise `.rev()` would apply to just `5`.

### Task 4.2 -- Three iteration forms

Replace `main` with:

```rust
fn main() {
    let temperatures = [21.5, 22.0, 22.8, 23.1, 22.5];

    println!("By value:");
    for t in temperatures {
        println!("  {t}");
    }

    println!("By reference:");
    for t in &temperatures {
        println!("  {t}");
    }

    let mut adjustable = [21.5, 22.0, 22.8];

    println!("By mutable reference (before): {adjustable:?}");
    for t in &mut adjustable {
        *t += 1.0;
    }
    println!("By mutable reference (after):  {adjustable:?}");
}
```

Run the program. Expected output:

```
By value:
  21.5
  22.0
  22.8
  23.1
  22.5
By reference:
  21.5
  22.0
  22.8
  23.1
  22.5
By mutable reference (before): [21.5, 22.0, 22.8]
By mutable reference (after):  [22.5, 23.0, 23.8]
```

The three forms differ in what the loop variable is:

- `for t in temperatures` produces `t` of type `f64`. For `Copy` types like `f64`, this is fine. For non-`Copy` types (like `String`), this would consume the collection.
- `for t in &temperatures` produces `t` of type `&f64`, a reference. The collection is unchanged.
- `for t in &mut temperatures` produces `t` of type `&mut f64`, a mutable reference. You can modify each element via `*t = ...`.

The dereference operator `*t += 1.0` is needed because `t` is a reference, not the value itself. We will return to references in detail in the ownership module.

### Task 4.3 -- Enumerate for indices

Replace `main` with:

```rust
fn main() {
    let labels = ["morning", "afternoon", "evening", "night"];

    for (index, label) in labels.iter().enumerate() {
        println!("Period {index}: {label}");
    }
}
```

Run the program. Expected output:

```
Period 0: morning
Period 1: afternoon
Period 2: evening
Period 3: night
```

`.enumerate()` produces an iterator of `(usize, T)` tuples. The destructuring pattern `(index, label)` extracts both at once. This is the idiomatic way to write a "for each item, with index" loop in Rust.

### Task 4.4 -- The non-idiomatic alternative

Replace `main` with:

```rust
fn main() {
    let labels = ["morning", "afternoon", "evening", "night"];

    for i in 0..labels.len() {
        println!("Period {}: {}", i, labels[i]);
    }
}
```

The program compiles and runs and produces the same output as Task 4.3. But run `cargo clippy`:

```
warning: the loop variable `i` is only used to index `labels`
  |
  = help: consider using an iterator: `for (i, item) in labels.iter().enumerate()`
```

Clippy detects the index-loop anti-pattern and suggests `enumerate`. The index-loop form has two problems:

- It requires the compiler to insert a bounds check at every access (which the optimizer usually removes, but should not have been needed).
- It is more verbose and easier to make off-by-one mistakes.

Restore the `enumerate` version before continuing.

### Task 4.5 -- A nontrivial iteration

Replace `main` with:

```rust
fn main() {
    let readings = [21.5, 22.0, 22.8, 23.1, 22.5, 21.8, 22.3];

    let mut max_so_far = readings[0];
    let mut max_index = 0;

    for (index, &reading) in readings.iter().enumerate() {
        if reading > max_so_far {
            max_so_far = reading;
            max_index = index;
        }
    }

    println!("Maximum: {max_so_far} at index {max_index}");
}
```

Run the program. Expected output:

```
Maximum: 23.1 at index 3
```

Note the pattern `&reading` in the destructuring: `iter()` produces references (`&f64`), and the `&` in the pattern dereferences them so `reading` is `f64` rather than `&f64`. This is a small example of pattern matching, which we will cover in detail later.

### Checkpoints

1. The half-open range `0..5` produces `0 1 2 3 4` (five values). The closed range `0..=5` produces `0 1 2 3 4 5` (six values). Why is the half-open form more common in real code?
2. Task 4.4 showed clippy flagging the index-loop pattern. The optimizer removes the bounds check in most cases, so the runtime cost is the same as `enumerate`. Why does clippy flag it anyway?
3. The Task 4.5 program uses two mutable variables (`max_so_far` and `max_index`) to track state. Could this computation be expressed without `mut`? Sketch the approach. (Hint: `iter().enumerate().max_by(...)` is one option.)

---

## Exercise 5 -- `loop` as an Expression with `break` Values

**Estimated time:** 10-15 minutes
**Topics covered:** `break value`, retry patterns, search patterns

### Context

A `loop` block can produce a value when it exits, supplied by `break`. This pattern replaces a common imperative idiom where a mutable variable is set inside the loop and read after the loop exits. Once you internalize it, you will reach for it whenever a loop is naturally a search or a retry.

### Task 5.1 -- The basic pattern

Replace `main` with:

```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;
        }
    };

    println!("Counter: {counter}, result: {result}");
}
```

Run the program. Expected output:

```
Counter: 10, result: 20
```

The `loop` runs until `break counter * 2` is reached. The expression `counter * 2` evaluates to `20`, and that becomes the value of the entire `loop`. The value is bound to `result`.

Note that the `let result = loop { ... };` line ends with a semicolon. The semicolon belongs to the `let` statement, not the loop. The loop itself has no semicolon at the end of its body.

### Task 5.2 -- A retry pattern

Replace the contents of `src/main.rs` with:

```rust
use std::time::Duration;

fn try_get_reading(attempt: u32) -> Result<f64, &'static str> {
    if attempt < 3 {
        Err("sensor not responding")
    } else {
        Ok(22.5)
    }
}

fn main() {
    let mut attempt = 0;

    let reading = loop {
        attempt += 1;

        match try_get_reading(attempt) {
            Ok(value) => break value,
            Err(message) => {
                println!("Attempt {attempt} failed: {message}");
                if attempt >= 5 {
                    panic!("giving up");
                }
                std::thread::sleep(Duration::from_millis(100));
            }
        }
    };

    println!("Got reading: {reading}");
}
```

Run the program. Expected output:

```
Attempt 1 failed: sensor not responding
Attempt 2 failed: sensor not responding
Got reading: 22.5
```

The loop runs until either `Ok(value)` arrives (return the value via `break value`) or the retry limit is hit (panic). This is a textbook use of `break value`: the value of the loop is the result of the operation it was retrying.

> **Note:** The `match` and `Result` types used here are covered formally in later modules. For now, treat the `match` as an `if/else` that handles the two cases of "success" and "failure," and treat `Result` as a value that holds either an `Ok` or an `Err` variant.

### Task 5.3 -- A search pattern

Replace `main` with:

```rust
fn main() {
    let readings = [21.5, 22.0, 22.8, 23.1, 22.5, 21.8, 22.3];
    let threshold = 23.0;

    let mut index = 0;

    let position = loop {
        if index >= readings.len() {
            break None;
        }
        if readings[index] > threshold {
            break Some(index);
        }
        index += 1;
    };

    match position {
        Some(i) => println!("First reading above {threshold} is at index {i}"),
        None => println!("No reading exceeded {threshold}"),
    }
}
```

Run the program. Expected output:

```
First reading above 23 is at index 3
```

The loop has two `break` paths: one returns `Some(index)` when a match is found, and the other returns `None` when the array is exhausted. Both produce values of the same type (`Option<usize>`), so the loop's value is well-typed.

In real Rust, this exact computation would be expressed with the iterator method `.position()`:

```rust
let position = readings.iter().position(|&r| r > threshold);
```

The iterator form is shorter and more idiomatic. The manual loop is shown to demonstrate that `break value` is structurally equivalent. You write loops by hand only when no iterator method fits.

### Task 5.4 -- Type rules for `break`

Try adding a third break path with a different type:

```rust
    let position = loop {
        if index >= readings.len() {
            break None;
        }
        if readings[index] > threshold {
            break Some(index);
        }
        if readings[index] < 0.0 {
            break "negative reading found";    // wrong type
        }
        index += 1;
    };
```

Run `cargo build`. The compiler rejects this:

```
error[E0308]: mismatched types
   |
   |             break "negative reading found";
   |                   ^^^^^^^^^^^^^^^^^^^^^^^^ expected `Option<usize>`, found `&str`
```

Every `break` in a `loop` must produce the same type. The compiler unifies all the break values and refuses to mix incompatible ones.

Remove the bad branch and confirm the program builds.

### Task 5.5 -- A subtle distinction

Try using `break value` in a `for` loop:

```rust
fn main() {
    let readings = [21.5, 22.0, 22.8];

    let result = for r in readings {
        if r > 22.5 {
            break r;
        }
    };
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0571]: `break` with value from a `for` loop
```

Only `loop` can produce values via `break`. `while` and `for` cannot. The reason is that `while` and `for` can exit without ever executing a `break`: the `while` condition becomes false, or the iterator is exhausted. In those cases, what value would the loop produce? There is no good answer, so Rust does not allow it.

If you want a value from a `for`-style iteration, use iterator methods (covered in detail in a later module). For now, the choice is: use `loop` if you need a value, or compute the value separately if you must use `for`.

### Checkpoints

1. The Task 5.2 retry pattern uses `break value` instead of accumulating into a mutable variable. Identify two specific advantages of this approach.
2. Task 5.5 showed that `break` cannot return a value from `for`. Explain why this restriction exists, in terms of what value would be produced when the iterator is exhausted normally.
3. The iterator method `position` does the same job as the manual loop in Task 5.3. When would you nevertheless choose the manual `loop` form? Give a concrete example.

---

## Exercise 6 -- Labeled Loops and Nested Control

**Estimated time:** 10-15 minutes
**Topics covered:** loop labels, breaking outer loops, alternatives to labels

### Context

`break` and `continue` apply to the innermost loop by default. When loops are nested, you sometimes need to break out of an outer loop directly. Labels solve this. They are not common in everyday Rust, but they are the right tool when the alternative is awkward.

### Task 6.1 -- The problem without labels

Replace `main` with:

```rust
fn main() {
    let grid = [
        [21.5, 22.0, 22.8],
        [22.5, 24.5, 23.1],
        [22.5, 21.8, 22.3],
    ];
    let alarm_threshold = 24.0;

    let mut alarm_position: Option<(usize, usize)> = None;

    for (row_index, row) in grid.iter().enumerate() {
        for (col_index, &value) in row.iter().enumerate() {
            if value > alarm_threshold {
                alarm_position = Some((row_index, col_index));
                break;       // only breaks the inner loop
            }
        }
        if alarm_position.is_some() {
            break;           // additional check to exit the outer loop
        }
    }

    println!("Alarm at: {alarm_position:?}");
}
```

Run the program. Expected output:

```
Alarm at: Some((1, 1))
```

The `break` in the inner loop only exits that loop. The outer loop continues until the next iteration, where the `if alarm_position.is_some()` check finally breaks it. This works but it is awkward: there are two break sites, and the outer one duplicates the condition.

### Task 6.2 -- Apply a label

Replace `main` with:

```rust
fn main() {
    let grid = [
        [21.5, 22.0, 22.8],
        [22.5, 24.5, 23.1],
        [22.5, 21.8, 22.3],
    ];
    let alarm_threshold = 24.0;

    let mut alarm_position: Option<(usize, usize)> = None;

    'outer: for (row_index, row) in grid.iter().enumerate() {
        for (col_index, &value) in row.iter().enumerate() {
            if value > alarm_threshold {
                alarm_position = Some((row_index, col_index));
                break 'outer;
            }
        }
    }

    println!("Alarm at: {alarm_position:?}");
}
```

Run the program. The output is the same:

```
Alarm at: Some((1, 1))
```

The label `'outer:` is attached to the outer `for` loop. The statement `break 'outer` exits that loop directly, regardless of how deeply nested it is. The second `if alarm_position.is_some()` check is gone.

The single quote in `'outer` is the same syntax used for lifetime annotations (covered in the borrowing module). For now, treat loop labels as a self-contained syntactic feature.

### Task 6.3 -- Combine label with `break value`

Replace `main` with:

```rust
fn main() {
    let grid = [
        [21.5, 22.0, 22.8],
        [22.5, 24.5, 23.1],
        [22.5, 21.8, 22.3],
    ];
    let alarm_threshold = 24.0;

    let alarm_position = 'search: loop {
        for (row_index, row) in grid.iter().enumerate() {
            for (col_index, &value) in row.iter().enumerate() {
                if value > alarm_threshold {
                    break 'search Some((row_index, col_index));
                }
            }
        }
        break 'search None;
    };

    println!("Alarm at: {alarm_position:?}");
}
```

Run the program. The output is again:

```
Alarm at: Some((1, 1))
```

Now the outer construct is a `loop` (which can produce a value), and the label `'search` is on it. The `break 'search Some(...)` and `break 'search None` paths both produce `Option<(usize, usize)>`. The result is bound to `alarm_position` directly, with no mutable variable.

This is the most idiomatic form for this kind of search inside a function body.

### Task 6.4 -- The function-extraction alternative

Replace `main.rs` with:

```rust
fn find_alarm(grid: &[[f64; 3]], threshold: f64) -> Option<(usize, usize)> {
    for (row_index, row) in grid.iter().enumerate() {
        for (col_index, &value) in row.iter().enumerate() {
            if value > threshold {
                return Some((row_index, col_index));
            }
        }
    }
    None
}

fn main() {
    let grid = [
        [21.5, 22.0, 22.8],
        [22.5, 24.5, 23.1],
        [22.5, 21.8, 22.3],
    ];

    let alarm_position = find_alarm(&grid, 24.0);
    println!("Alarm at: {alarm_position:?}");
}
```

Run the program. The output is the same. The function uses `return` instead of `break 'outer`. `return` exits the function, which automatically exits all loops inside it. No labels needed.

This is usually the cleanest form when the search is a self-contained operation. Labels become useful when the loop must share state with the surrounding code or when extraction would force you to pass many parameters.

### Task 6.5 -- `continue` with a label

Replace `main` with:

```rust
fn main() {
    'outer: for i in 1..=3 {
        for j in 1..=3 {
            if j == 2 {
                continue 'outer;
            }
            println!("({i}, {j})");
        }
    }
}
```

Run the program. Expected output:

```
(1, 1)
(2, 1)
(3, 1)
```

For each outer `i`, the inner loop runs once with `j == 1`, then `continue 'outer` skips both the rest of the inner loop and the rest of the current outer iteration. The next outer iteration starts immediately.

### Checkpoints

1. Task 6.1 used a flag variable (`alarm_position`) and an extra check after the inner loop to break the outer one. Task 6.2 used a label. Identify a specific way the label form is harder to misread.
2. Task 6.4 replaced labels with a function and `return`. Why is function extraction usually preferred when it is feasible? When is it NOT feasible?
3. The `continue 'outer` form in Task 6.5 was clear because the example was small. Imagine a 50-line outer loop with several continue points inside an inner loop. Would labels still be the right tool? What might you do instead?

---

## Exercise 7 -- Putting It Together

**Estimated time:** 10 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from the module into one program: an `if` chain as an expression, all three loop constructs, range iteration, mutable references, `break value`, and a labeled loop. The program processes a stream of weather readings and produces a summary report.

### Task 7.1 -- The full program

Replace `main.rs` with:

```rust
const ALARM_THRESHOLD: f64 = 30.0;
const COMFORT_LOW: f64 = 18.0;
const COMFORT_HIGH: f64 = 25.0;

fn classify(temperature: f64) -> &'static str {
    if temperature > ALARM_THRESHOLD {
        "alarm"
    } else if temperature > COMFORT_HIGH {
        "warm"
    } else if temperature > COMFORT_LOW {
        "comfortable"
    } else {
        "cool"
    }
}

fn main() {
    let daily_readings = [
        [18.5, 19.0, 22.5, 26.5, 28.0, 27.5, 24.0],
        [19.0, 20.5, 23.0, 27.0, 31.5, 28.5, 25.0],
        [17.5, 18.5, 21.0, 24.5, 26.0, 25.5, 22.5],
    ];

    println!("Daily classifications:");
    for (day, hourly) in daily_readings.iter().enumerate() {
        for (hour, &reading) in hourly.iter().enumerate() {
            let category = classify(reading);
            println!("  Day {day}, hour {hour}: {reading}C ({category})");
        }
    }

    let first_alarm = 'search: loop {
        for (day, hourly) in daily_readings.iter().enumerate() {
            for (hour, &reading) in hourly.iter().enumerate() {
                if reading > ALARM_THRESHOLD {
                    break 'search Some((day, hour, reading));
                }
            }
        }
        break 'search None;
    };

    match first_alarm {
        Some((day, hour, reading)) => {
            println!("\nFirst alarm: day {day}, hour {hour}, reading {reading}C");
        }
        None => {
            println!("\nNo alarms");
        }
    }

    let mut total: f64 = 0.0;
    let mut count = 0;
    for hourly in daily_readings.iter() {
        for &reading in hourly.iter() {
            total += reading;
            count += 1;
        }
    }
    let average = total / count as f64;
    println!("\nOverall average: {average:.2}C ({})", classify(average));

    println!("\nCountdown to next reading:");
    for second in (1..=3).rev() {
        println!("  {second}");
    }
    println!("  Reading now");
}
```

Run the program. Expected output:

```
Daily classifications:
  Day 0, hour 0: 18.5C (comfortable)
  Day 0, hour 1: 19C (comfortable)
  Day 0, hour 2: 22.5C (comfortable)
  Day 0, hour 3: 26.5C (warm)
  Day 0, hour 4: 28C (warm)
  Day 0, hour 5: 27.5C (warm)
  Day 0, hour 6: 24C (comfortable)
  Day 1, hour 0: 19C (comfortable)
  Day 1, hour 1: 20.5C (comfortable)
  Day 1, hour 2: 23C (comfortable)
  Day 1, hour 3: 27C (warm)
  Day 1, hour 4: 31.5C (alarm)
  Day 1, hour 5: 28.5C (warm)
  Day 1, hour 6: 25C (comfortable)
  Day 2, hour 0: 17.5C (cool)
  Day 2, hour 1: 18.5C (comfortable)
  Day 2, hour 2: 21C (comfortable)
  Day 2, hour 3: 24.5C (comfortable)
  Day 2, hour 4: 26C (warm)
  Day 2, hour 5: 25.5C (warm)
  Day 2, hour 6: 22.5C (comfortable)

First alarm: day 1, hour 4, reading 31.5C

Overall average: 23.55C (comfortable)

Countdown to next reading:
  3
  2
  1
  Reading now
```

### Task 7.2 -- Identify the patterns

In `lab3-notes.md`, identify each instance of the following patterns in the program above:

1. An `if` chain used as an expression to produce a value.
2. A `for` loop iterating over a range.
3. A `for` loop iterating with `enumerate` to get index and value.
4. A `loop` block that produces a value via `break`.
5. A labeled loop.
6. A reversed range using `.rev()`.

This kind of close reading is how you internalize idiomatic Rust. Each pattern in the program is a deliberate choice, not an accident. The program could have been written with mutable variables and explicit early returns, but the form shown here is what experienced Rust developers write.

### Checkpoints

1. The function `classify` returns a `&'static str` rather than a `String`. The values are string literals like `"alarm"`. Why is `&'static str` the right return type for this kind of function?
2. The `first_alarm` block uses a labeled `loop` with `break 'search`. The same effect could be achieved by extracting `find_first_alarm` into a separate function with `return`. What practical considerations would push you toward the function form?
3. The accumulation of `total` and `count` in the average calculation uses two mutable variables. Could this be expressed with iterator methods (`flat_map`, `sum`, etc.) and no `mut` keywords? Sketch the approach without writing full code.

---

## Summary and Reflection

You have now used every control flow construct from Module 4 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- `if` as Statement | basic conditionals, `bool` requirement | Conditions must be `bool`. Braces are mandatory. |
| 2 -- `if` as Expression | `if` produces values | Branch type unification and mandatory `else` make the construct safe. |
| 3 -- Choosing a Loop | `loop`, `while`, `for` | Each loop expresses a different intent. `loop` for value-producing or middle-exit loops, `while` for conditional repetition, `for` for sequence iteration. |
| 4 -- Ranges and Iteration | range operators, three reference forms, `enumerate` | The `for` loop walks iterators, not indices. C-style index loops are non-idiomatic. |
| 5 -- `loop` with `break value` | value-producing loops | Retries and searches are the natural use cases. Iterator methods are usually preferred for searches. |
| 6 -- Labels | nested loop control | Labels are a real tool for nested early-exit. Function extraction with `return` is usually the better answer when it is feasible. |
| 7 -- Integration | all of the above | Idiomatic Rust uses these constructs together; reading code closely is how you learn the patterns. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab3-notes.md` before your next session.

1. Of the control flow features in this module, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. Compare Rust's three loop constructs to the loop facilities of another language you know well. What does Rust gain by separating `loop`, `while`, and `for`? What would you lose if Rust collapsed them into one keyword?

3. The lab exercises asked you to apply judgment several times: which loop fits which problem, when to use a label versus extract a function, when `break value` is clearer than a flag variable. Identify one case from this lab where you initially wrote one form and then refactored to another after considering the alternative. What changed your decision?

---

*End of Lab 3*
