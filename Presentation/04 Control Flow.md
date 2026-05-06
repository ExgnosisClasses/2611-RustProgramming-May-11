# Module 4: Control Flow

## Module Overview

Control flow constructs decide which code runs and how often it runs. Most of what this module covers will look familiar: `if`, `while`, `for`, and `break` exist in nearly every language. The interesting part is what Rust does differently. Three design decisions matter:

1. **Most control flow constructs are expressions, not statements.** They produce values that you can bind to variables, return from functions, or pass to other expressions.
2. **Conditions must be `bool`, not "truthy" values.** No implicit conversion from integers, pointers, or anything else.
3. **The `for` loop is built on iterators**, not on integer indices. The construct you reach for to walk a sequence is the same construct that powers complex data pipelines.

By the end of this module, students will:

- Use `if` and `if let` as expressions, including in `let` bindings and function returns.
- Choose between `loop`, `while`, and `for` based on what the code actually expresses.
- Iterate over ranges, collections, and arbitrary iterators with `for`.
- Return values from `loop` using `break value` and apply this pattern in retry logic.
- Use loop labels to control nested loops cleanly.

The patterns in this module are the foundation for almost every nontrivial Rust program. Future modules on iterators, pattern matching, and error handling all build on the constructs introduced here.

---

## A. `if`, `else if`, `else` Expressions

### The Basic Form

The syntax for conditional logic in Rust mirrors most C-family languages, with one important difference: the parentheses around the condition are not required, and the braces around the body are required.

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

The braces are not optional. The following does not compile:

```rust
// Compile error
if temperature > 30.0 println!("Hot");
```

This eliminates an entire category of bugs that plagues C codebases, where adding a second statement to a single-line `if` body silently changes the program's behavior.

### Conditions Must Be `bool`

Rust does not have "truthy" values. A condition must evaluate to a `bool`. The following common C idioms are compile errors in Rust:

```rust
let count = 5;

// All compile errors:
// if count { ... }                  // count is i32, not bool
// if !count { ... }                  // ! requires bool
// if some_pointer { ... }            // pointers are not bool
```

The correct forms are explicit:

```rust
if count > 0 { /* ... */ }
if count != 0 { /* ... */ }
if some_option.is_some() { /* ... */ }
```

The reasoning is the same as the integer overflow design: implicit conversions are a frequent source of bugs, and Rust prefers to be verbose at the cost of being explicit. A condition like `if user_count` reads ambiguously: is the test "is `user_count` greater than zero" or "is `user_count` non-null"? Forcing a `bool` makes the intent clear.

### `if` as an Expression

This is where Rust diverges from many languages. An `if` block is an expression that produces a value, the same way `2 + 2` is an expression that produces `4`.

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

The entire `if` chain produces a single string slice that is bound to `category`. Notice the semicolon at the end of the `let` statement, but no semicolon at the end of each branch's value. Inside the braces, the last expression without a semicolon is the value of that branch.

### Rules for `if` Expressions

When using `if` as an expression, two rules are enforced by the compiler.

**Rule 1: All branches must produce the same type.**

```rust
// Compile error: mismatched types
let value = if condition {
    42         // i32
} else {
    "hello"    // &str
};
```

The compiler must be able to assign one type to the entire expression. Branches that produce different types cannot be unified.

**Rule 2: An `else` branch is required if the value is used.**

```rust
// Compile error: missing else
let value = if condition { 42 };
```

Without an `else`, the value would be undefined when the condition is false. Rust does not have implicit "null" or "default" values to fill in. If you want a value, every branch must produce one.

If the `if` is used as a statement (its value discarded), the `else` is optional:

```rust
if condition {
    println!("Yes");
}
```

### Why This Design?

Treating `if` as an expression has practical benefits.

**It eliminates a class of uninitialized-variable patterns.** Code in C and Java often follows this shape:

```java
String category;
if (temperature > 30.0) {
    category = "Hot";
} else if (temperature > 20.0) {
    category = "Warm";
} else {
    category = "Cold";
}
```

Each branch must remember to assign. If you add a fourth branch and forget to assign, the variable is uninitialized. Rust's expression form makes this impossible: the value flows out of the expression, and the compiler verifies that every branch produces a value.

**It makes simple cases concise.** Many short conditional assignments fit on one line:

```rust
let label = if value > 0 { "positive" } else { "non-positive" };
```

This replaces what would be a five-line `if/else` with assignment in each branch.

### `if let` for Pattern Matching

Rust has a related construct, `if let`, that combines a pattern match with a conditional. It is covered in detail in the pattern matching module, but a brief example shows the idea:

```rust
let maybe_value: Option<i32> = Some(42);

if let Some(n) = maybe_value {
    println!("Got a value: {n}");
} else {
    println!("No value");
}
```

This is shorter than a full `match` when you only care about one case. We will return to this construct later.

---

## B. Loops: `loop`, `while`, `for`

Rust has three loop constructs. They are not interchangeable; each one expresses a different intent.

### `loop`: Unconditional Repetition

The `loop` keyword starts a block that repeats forever, until something inside it explicitly stops the iteration.

```rust
fn main() {
    let mut count = 0;

    loop {
        count += 1;
        if count == 5 {
            break;
        }
        println!("Iteration {count}");
    }

    println!("Done");
}
```

Output:

```
Iteration 1
Iteration 2
Iteration 3
Iteration 4
Done
```

The `break` statement exits the innermost loop. The `continue` statement skips to the next iteration.

#### When to Use `loop`

Use `loop` when:

- The exit condition is not a simple expression at the top of the loop.
- The exit logic is naturally expressed in the middle or end of the body.
- You want to use the loop as an expression that produces a value (covered in Section D).

A common pattern is a retry loop where the exit condition depends on a result computed inside the body:

```rust
loop {
    let result = try_connect();
    if result.is_ok() {
        break;
    }
    sleep(retry_delay);
}
```

A `while` loop here would require either a mutable result variable initialized to a sentinel value before the loop or a duplicated call. `loop` expresses the intent more directly.

### `while`: Conditional Repetition

The `while` loop runs as long as a condition is true. The condition is checked at the top of each iteration.

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

Output:

```
5
4
3
2
1
Liftoff
```

#### When to Use `while`

Use `while` when:

- You have a single condition that determines whether to continue.
- The condition is naturally checked before each iteration.
- The number of iterations is not known in advance.

#### What `while` Replaces

In other languages, `while (true) { ... }` is a common idiom for an infinite loop. In Rust, prefer `loop`:

```rust
// Less idiomatic:
while true {
    /* ... */
    if done { break; }
}

// More idiomatic:
loop {
    /* ... */
    if done { break; }
}
```

`loop` is more honest about the intent (an infinite loop). It also enables the optimizer to reason about the construct more precisely. clippy will flag `while true` and suggest `loop`.

### `for`: Iteration Over a Sequence

The `for` loop walks every element of an iterator.

```rust
fn main() {
    let temperatures = [21.5, 22.0, 22.8, 23.1, 22.5];

    for t in temperatures {
        println!("Reading: {t}");
    }
}
```

Output:

```
Reading: 21.5
Reading: 22.0
Reading: 22.8
Reading: 23.1
Reading: 22.5
```

#### When to Use `for`

Use `for` when:

- You are walking every element of a collection.
- You are walking a range of integers.
- You are processing the output of an iterator chain.

This is by far the most common loop in idiomatic Rust. If you are reaching for `loop` or `while` to walk a collection, you are probably writing C-in-Rust.

### Choosing the Right Loop

A summary:

| Loop      | Use When                                                    |
|-----------|-------------------------------------------------------------|
| `for`     | You are processing each element of a sequence.              |
| `while`   | A single condition determines whether to continue.          |
| `loop`    | The exit logic is in the middle of the body, or you need a value from the loop. |

When a `while` condition is just `true`, replace it with `loop`. When a `for` could be expressed as `for i in 0..vec.len()`, restructure it to `for item in &vec` instead.

---

## C. Iterating with `for` and Ranges

The `for` loop in Rust does not work directly with integer indices. It works with iterators. To loop over a range of numbers, you create a range iterator with the `..` or `..=` operator.

### Range Operators

Rust has two range operators:

- `start..end` is a half-open range. It includes `start` but excludes `end`.
- `start..=end` is a closed range. It includes both `start` and `end`.

```rust
fn main() {
    println!("Half-open 0..5:");
    for i in 0..5 {
        print!("{i} ");
    }
    println!();

    println!("Closed 0..=5:");
    for i in 0..=5 {
        print!("{i} ");
    }
    println!();
}
```

Output:

```
Half-open 0..5:
0 1 2 3 4 
Closed 0..=5:
0 1 2 3 4 5 
```

The half-open form is far more common. It matches the natural way collections are indexed (positions 0 through length, exclusive of length) and avoids the off-by-one errors that come from inclusive ranges.

### Iterating in Reverse

Ranges have a `.rev()` method that produces a reversed iterator:

```rust
fn main() {
    for i in (1..=5).rev() {
        println!("{i}");
    }
    println!("Liftoff");
}
```

Output:

```
5
4
3
2
1
Liftoff
```

The parentheses around `1..=5` are required because `.rev()` would otherwise apply to just the `5`.

### Iterating Over Collections

For arrays, vectors, and other collections, the `for` loop walks every element directly. There are three common forms:

#### Iterate by Value

```rust
let temperatures = [21.5, 22.0, 22.8];
for t in temperatures {
    println!("{t}");
}
```

Each `t` is a fresh `f64`. For types that copy cheaply (numeric types, booleans, characters), this is fine. For types that own heap memory (`String`, `Vec`), this would move the values out of the collection, which is usually not what you want. We will return to this distinction in the ownership module.

#### Iterate by Reference

```rust
let temperatures = [21.5, 22.0, 22.8];
for t in &temperatures {
    println!("{t}");
}
```

The `&` prefix produces an iterator that yields references. Each `t` is now `&f64` instead of `f64`. The collection is not consumed; you can use it again after the loop.

For non-`Copy` types, this is the default form. Iterating over a `Vec<String>` typically uses `&vec` to walk references rather than moving each string out.

#### Iterate by Mutable Reference

```rust
let mut temperatures = [21.5, 22.0, 22.8];
for t in &mut temperatures {
    *t += 1.0;        // dereference to assign
}
println!("{temperatures:?}");      // [22.5, 23.0, 23.8]
```

The `&mut` form lets you modify each element in place. The `*` is the dereference operator: `t` is `&mut f64`, and `*t` accesses the `f64` it points to.

### Iterating with Indices

When you need both the index and the value, use `.enumerate()`:

```rust
let labels = ["morning", "afternoon", "evening"];
for (index, label) in labels.iter().enumerate() {
    println!("{index}: {label}");
}
```

Output:

```
0: morning
1: afternoon
2: evening
```

`.enumerate()` produces an iterator of `(usize, T)` tuples. The destructuring pattern `(index, label)` extracts both at once.

This pattern replaces the C-style `for (int i = 0; i < length; i++)` loop in nearly all cases.

### What NOT To Do

The following pattern is non-idiomatic Rust:

```rust
let temperatures = [21.5, 22.0, 22.8];
for i in 0..temperatures.len() {
    println!("{}: {}", i, temperatures[i]);
}
```

This is an index loop in disguise. It works but it is verbose and adds an unnecessary bounds check at every iteration (which the optimizer usually removes, but should never have been necessary). The `.enumerate()` form is faster, shorter, and more idiomatic. clippy will flag the index form, as you saw in earlier labs.

### Iterators Are Lazy

A range like `0..1_000_000_000` does not allocate memory or compute anything until something starts pulling values out of it. Iterators in Rust are lazy: they produce values on demand. This is why ranges of any size are cheap to write:

```rust
for i in 0..1_000_000_000 {
    if i % 100_000_000 == 0 {
        println!("Reached {i}");
    }
}
```

The range itself is just two integers (`start` and `end`) with a method that returns the next value. Nothing is allocated. Iterators are covered in detail in a later module, where the laziness becomes the foundation for building data pipelines.

---

## D. `loop` as an Expression with `break` Values

A `loop` block can produce a value when it exits. The value is supplied by the `break` statement.

### The Basic Pattern

```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;
        }
    };

    println!("Result: {result}");
}
```

Output:

```
Result: 20
```

The `loop` expression runs until `break counter * 2` is reached, then evaluates to `20`. That value is bound to `result`.

This is one of the more unusual features of Rust to programmers from other languages. C, Java, and Python do not have this. Most languages use `loop` purely for side effects; Rust treats it as a value-producing expression.

### Why Allow This?

The motivation is the same as for `if` expressions: many computations are naturally expressed as a search or a retry that produces a value. Without `break value`, you would need a mutable variable to carry the result out of the loop:

```rust
// Without break value
let mut found = None;
let mut i = 0;
loop {
    if some_condition(i) {
        found = Some(i * 2);
        break;
    }
    i += 1;
}
let result = found.unwrap();

// With break value
let result = loop {
    if some_condition(i) {
        break i * 2;
    }
    i += 1;
};
```

The second form is shorter, has no mutable state to manage, and the value cannot accidentally be observed before the loop completes.

### Practical Use: Retry Logic

Retry loops are a natural fit for `break value`. The classic pattern is "try something repeatedly, return the result of the first successful attempt":

```rust
fn fetch_with_retry() -> String {
    let mut attempts = 0;

    loop {
        attempts += 1;
        match try_fetch() {
            Ok(data) => break data,
            Err(_) if attempts < 5 => {
                // wait and try again
                std::thread::sleep(std::time::Duration::from_millis(100));
            }
            Err(e) => panic!("Failed after {attempts} attempts: {e}"),
        }
    }
}

fn try_fetch() -> Result<String, &'static str> {
    Ok(String::from("success"))   // simplified for the example
}
```

The function returns the first successful fetch. The `loop` is the natural shape: there is no fixed number of iterations, and the exit logic depends on the outcome of work done inside the body.

### Practical Use: Searching

A search that needs the matching index can return it directly from `break`:

```rust
fn find_first_negative(values: &[i32]) -> Option<usize> {
    let mut index = 0;
    let result = loop {
        if index >= values.len() {
            break None;
        }
        if values[index] < 0 {
            break Some(index);
        }
        index += 1;
    };
    result
}
```

In real Rust, you would use the iterator method `.position()` for this:

```rust
fn find_first_negative(values: &[i32]) -> Option<usize> {
    values.iter().position(|&x| x < 0)
}
```

The iterator form is the idiomatic choice. The `loop` form is shown to illustrate that `break value` is structurally equivalent: you write loops manually only when no iterator method fits.

### Type Rules

The same type discipline applies as with `if` expressions: every `break` in a `loop` must produce the same type, and the loop's value is that type.

```rust
// Compile error: incompatible types in break expressions
let x = loop {
    if condition_a { break 42; }
    if condition_b { break "hello"; }   // type mismatch
};
```

The compiler must assign one type to the loop. Mixing types is rejected.

### What About `while` and `for`?

Only `loop` can produce a value. `while` and `for` cannot:

```rust
// Compile error
let result = while condition { break 42; };
let result = for i in 0..10 { break i; };
```

The reason is that `while` and `for` can exit without ever reaching a `break`. A `while` loop ends when its condition becomes false; a `for` loop ends when the iterator is exhausted. In those cases, what value would the loop produce? There is no good answer.

`loop`, by contrast, only exits via `break`, so every exit path explicitly supplies a value.

If you want a value from a `for` loop, the answer is to use iterator methods like `.find()`, `.position()`, `.fold()`, or `.collect()`. We will cover these in detail in the iterator module.

---

## E. Labeled Loops and Nested Control Flow

`break` and `continue` apply to the innermost loop by default. When loops are nested, you sometimes need to break out of an outer loop directly. Labels solve this.

### The Problem

Consider searching a two-dimensional grid for a value:

```rust
fn main() {
    let grid = [
        [1, 2, 3],
        [4, 5, 6],
        [7, 8, 9],
    ];
    let target = 5;

    let mut found = None;
    for (row_index, row) in grid.iter().enumerate() {
        for (col_index, &value) in row.iter().enumerate() {
            if value == target {
                found = Some((row_index, col_index));
                break;     // only breaks the inner loop
            }
        }
        if found.is_some() {
            break;         // additional check to exit the outer loop
        }
    }

    println!("Found at: {found:?}");
}
```

The `break` in the inner loop exits that loop only. To exit the outer loop, we have to add another `break` after the inner loop, conditional on the result. This is awkward and easy to get wrong.

### Loop Labels

A label is an identifier prefixed with a single quote, attached to a loop. `break` and `continue` can target a labeled loop directly:

```rust
fn main() {
    let grid = [
        [1, 2, 3],
        [4, 5, 6],
        [7, 8, 9],
    ];
    let target = 5;

    let mut found = None;
    'outer: for (row_index, row) in grid.iter().enumerate() {
        for (col_index, &value) in row.iter().enumerate() {
            if value == target {
                found = Some((row_index, col_index));
                break 'outer;
            }
        }
    }

    println!("Found at: {found:?}");
}
```

The label `'outer:` names the outer loop. The statement `break 'outer` exits that loop directly, regardless of how deeply nested the `break` site is.

### Continue with Labels

`continue` works the same way. To skip to the next iteration of an outer loop:

```rust
'outer: for i in 0..3 {
    for j in 0..3 {
        if j == 1 {
            continue 'outer;       // skip the rest of the inner loop AND the rest of this outer iteration
        }
        println!("({i}, {j})");
    }
}
```

Output:

```
(0, 0)
(1, 0)
(2, 0)
```

For each outer `i`, the inner loop runs once with `j = 0`, then `continue 'outer` skips both the rest of the inner loop and goes to the next outer iteration.

### Combining Labels with Loop Values

Labels work with `break value` as well:

```rust
fn main() {
    let grid = [[1, 2, 3], [4, 5, 6], [7, 8, 9]];
    let target = 5;

    let position = 'search: loop {
        for (row_index, row) in grid.iter().enumerate() {
            for (col_index, &value) in row.iter().enumerate() {
                if value == target {
                    break 'search Some((row_index, col_index));
                }
            }
        }
        break 'search None;
    };

    println!("Position: {position:?}");
}
```

Here `'search` is on the outer `loop`. The inner `for` loops are nested inside it. When the target is found, `break 'search Some((row, col))` exits the outermost loop with a value. If no value is found, `break 'search None` outside the inner loops handles that case.

### When to Use Labels

Use labels when:

- You have nested loops and need to break or continue an outer loop based on something that happened in an inner loop.
- The alternative would require flag variables or duplicated condition checks.

Avoid labels when:

- The logic can be cleanly extracted into a function that uses `return` to exit early.
- An iterator method (`find`, `any`, `all`) expresses the same thing.

The function-extraction alternative is often the cleanest:

```rust
fn find_in_grid(grid: &[[i32; 3]], target: i32) -> Option<(usize, usize)> {
    for (row_index, row) in grid.iter().enumerate() {
        for (col_index, &value) in row.iter().enumerate() {
            if value == target {
                return Some((row_index, col_index));
            }
        }
    }
    None
}
```

`return` exits the function, which automatically exits all loops inside it. No labels needed. This is usually the right answer when the search is a self-contained operation.

Labels are most useful when the surrounding context cannot be easily extracted, such as when the loops share state with the code before or after them, or when the early exit is one of several outcomes that all need to update local variables.

### Label Naming Conventions

The convention is to use short, descriptive names: `'outer`, `'search`, `'retry`. The single-quote syntax is reused from Rust's lifetime annotations (covered in the borrowing module). For now, treat loop labels as a self-contained syntactic feature.

---

## Module Summary

- `if`, `else if`, and `else` are expressions that produce values. All branches must produce the same type, and an `else` branch is required if the value is used.
- Conditions must be `bool`. Rust does not have implicit truthiness.
- Three loop constructs serve different intents: `loop` for unconditional repetition, `while` for conditional repetition, `for` for iteration over a sequence.
- `for` loops walk iterators. To iterate integer indices, create a range with `..` (exclusive) or `..=` (inclusive). Use `.enumerate()` to get index and value together.
- `loop` blocks can produce values via `break value`. `while` and `for` cannot.
- Labels (`'name:`) allow `break` and `continue` to target outer loops in nested situations.

## Discussion Questions

1. Rust treats `if` as an expression, while C and Java treat it as a statement. Identify a specific code pattern that is shorter or safer in the Rust form. Identify a pattern that is unaffected by the difference.
2. The compiler rejects conditions that are not `bool`. Some languages allow integer truthiness as a convenience. What kinds of bugs does Rust's stricter rule prevent? What does it cost in cases where the conversion would have been correct?
3. `loop`, `while`, and `for` could in principle be reduced to a single construct, since `loop` can express the others with explicit `break` conditions. Why does Rust have all three?

---

