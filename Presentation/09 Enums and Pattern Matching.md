# Module 9: Enums and Pattern Matching

## Module Overview

Enums in Rust are not the simple integer-labelled constants of C or Java. They are a way of saying "a value of this type is one of these variants, and each variant can carry its own data." Combined with pattern matching, they become one of the most powerful features in the language. The standard library uses them everywhere: `Option<T>` to represent "a value or nothing," `Result<T, E>` to represent "success or failure," and dozens of others.

This module covers both halves of the picture: how to define and use enums, and how to inspect them with pattern matching. The two are inseparable. Almost every enum you encounter will be unpacked using `match`, `if let`, or one of the other pattern-matching constructs.

Three ideas are central:

1. **Enums make illegal states unrepresentable.** Instead of using flags, sentinels, or null values to indicate "no value here," you put that case directly in the type. The compiler then forces you to handle every variant.
2. **Pattern matching is exhaustive.** The compiler checks that every possible variant is handled. If you add a new variant later, the compiler will flag every place that needs to be updated.
3. **Patterns destructure values.** Pattern matching is not just "which variant is this" but "which variant is this AND what is inside it." The same syntax pulls apart structs, tuples, and enums.

By the end of this module, students will:

- Define enums with and without associated data.
- Use `Option<T>` instead of null and `Result<T, E>` instead of exceptions.
- Write `match` expressions that handle every case the compiler requires.
- Use `if let` and `while let` for the common case of "I only care about one variant."
- Destructure structs, enums, and tuples to extract their parts.
- Distinguish refutable from irrefutable patterns and know where each is valid.
- Use pattern guards and the `@` binding for advanced pattern matching.

This module is foundational for everything that follows. Error handling (Module 10), iterators (Module 11), and most of the standard library all rely on these constructs.

> **A note on the standard library.** Many of the examples in this module use `Option` and `Result` from the standard library. These are themselves enums, defined exactly the way you would define your own. After this module, looking at the standard library source code will feel familiar; the constructs are the same ones you write yourself.

---

## A. Defining Enums with and without Data

### Simple Enums

The simplest form of enum is a list of named variants:

```rust
enum Direction {
    North,
    South,
    East,
    West,
}
```

This defines a new type `Direction`. A value of this type is exactly one of the four variants. You construct values with the variant name:

```rust
fn main() {
    let heading = Direction::North;

    match heading {
        Direction::North => println!("going up"),
        Direction::South => println!("going down"),
        Direction::East => println!("going right"),
        Direction::West => println!("going left"),
    }
}
```

The naming convention: `PascalCase` for both the enum name and the variant names. Variants are accessed with the `EnumName::VariantName` syntax.

This kind of simple enum corresponds roughly to enums in C or Java. The interesting features come when variants carry data.

### Enums with Data

Each variant of an enum can carry its own data, with its own type:

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

This defines four variants:

- `Quit` carries no data (like a simple enum).
- `Move` carries named fields, like a struct.
- `Write` carries a single `String`, like a tuple struct.
- `ChangeColor` carries three `i32`s, like a tuple struct.

Each variant is essentially its own type, but they all share the enum type. A value of type `Message` is exactly one of these four shapes.

You construct values by writing the variant name with the appropriate data:

```rust
fn main() {
    let m1 = Message::Quit;
    let m2 = Message::Move { x: 10, y: 20 };
    let m3 = Message::Write(String::from("hello"));
    let m4 = Message::ChangeColor(255, 128, 0);
}
```

### What Enums Replace

Without enums, this kind of "value of one of several shapes" is awkward to express. In C or Java, you would use:

- A class hierarchy with a base class and subclasses (heavy machinery, runtime dispatch).
- A struct with a tag field and a union of possibilities (manual and error-prone).
- Multiple flag fields with conventions about which combinations are valid (subtle bugs).

Rust enums make this single concept ("one of these shapes, no other shapes") a first-class language feature. The compiler tracks which shapes are possible and forces you to handle them.

### Methods on Enums

Like structs, enums can have methods. The `impl` block syntax is identical:

```rust
impl Message {
    fn describe(&self) -> &str {
        match self {
            Message::Quit => "quit",
            Message::Move { .. } => "move",
            Message::Write(_) => "write",
            Message::ChangeColor(..) => "change color",
        }
    }
}

fn main() {
    let m = Message::Write(String::from("hello"));
    println!("{}", m.describe());
}
```

The `match` expression in `describe` examines `self` and returns a different string for each variant. The `..` and `_` patterns are placeholders for "anything here, I don't care about the values" (covered in Section F).

### Memory Layout

An enum value occupies space equal to the largest variant, plus a small tag (one to a few bytes) that identifies which variant is active. So the `Message` enum above occupies enough space to hold a `String` (24 bytes) plus a discriminant.

This is unlike a class hierarchy, which typically uses pointers and heap allocation for each instance. An enum value lives directly on the stack (or wherever its container lives), with no indirection. This is one of the things that makes enums efficient in Rust.

---

## B. `Option<T>`: Replacing Null

The most important enum in the standard library is `Option<T>`. It represents a value that may or may not be present.

### The Definition

`Option<T>` is defined essentially like this:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

The `<T>` is a generic type parameter (covered in detail in Module 10). It means `Option` can hold any type: `Option<i32>`, `Option<String>`, `Option<User>`, etc.

A value of type `Option<T>` is either `Some(value)` for some value of type `T`, or `None` indicating no value.

### Why It Exists

Most languages have `null`, `nil`, or `None` as a special value that any pointer can take. This is convenient but unsafe: any time you dereference a pointer, the value might be null, and forgetting to check for null is the source of an enormous number of crashes. Tony Hoare, the inventor of null references, called it his "billion-dollar mistake."

Rust does not have null. Instead, the type system distinguishes between "definitely a value" and "maybe a value":

```rust
let definitely_a_number: i32 = 5;            // always has a value
let maybe_a_number: Option<i32> = Some(5);    // currently Some, might be None
let nothing: Option<i32> = None;              // explicitly nothing
```

A function that might fail to return a value uses `Option`:

```rust
fn find_user(id: u64) -> Option<User> {
    // returns Some(user) if found, None if not
}
```

The caller cannot ignore the possibility of `None`. To use the value, they must explicitly handle both cases. The compiler enforces this.

### Using Option

The most common way to handle an `Option` is with `match`:

```rust
fn main() {
    let result: Option<i32> = Some(42);

    match result {
        Some(value) => println!("got {value}"),
        None => println!("nothing"),
    }
}
```

Two things to note:

- The `match` checks each variant. The compiler verifies that every variant is handled.
- The `Some(value)` pattern destructures the variant: `value` is bound to the inner `i32`.

For the common case of "do something only if Some," `if let` is more concise:

```rust
let result: Option<i32> = Some(42);

if let Some(value) = result {
    println!("got {value}");
}
```

This is covered in Section E.

### Common Methods

`Option<T>` has many methods that handle common patterns without writing out a `match`:

```rust
let x: Option<i32> = Some(5);
let y: Option<i32> = None;

x.is_some();              // true
y.is_none();              // true

x.unwrap();               // 5; PANICS if None
y.unwrap_or(0);           // 0 (returns the default if None)
x.unwrap_or_else(|| compute_default());     // calls the closure if None

x.map(|n| n * 2);         // Some(10)
y.map(|n| n * 2);         // None

x.and_then(|n| if n > 0 { Some(n) } else { None });    // chains operations that might fail
```

`unwrap` is convenient but dangerous: if the `Option` is `None`, the program panics. Use it only when you are absolutely certain the value is `Some` (and consider whether you should restructure the code instead).

`unwrap_or` and `unwrap_or_else` provide safe defaults. `map` transforms the inner value if present. `and_then` chains operations. These methods are covered in detail in Module 11.

### When to Use Option

Use `Option<T>` whenever:

- A value might be missing.
- A search might find nothing.
- An optional field on a struct may or may not be filled in.
- A function that usually returns a value might in some cases produce nothing meaningful.

The signal for using `Option` is "the absence of a value is a normal, expected outcome." If absence indicates an error, `Result` is usually better (covered next).

---

## C. `Result<T, E>`: Recoverable Error Handling

`Result<T, E>` is the second most important enum in the standard library. It represents the outcome of an operation that can succeed or fail.

### The Definition

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Two type parameters: `T` for the success value, `E` for the error. A `Result<T, E>` is either `Ok(value)` if the operation succeeded, or `Err(error)` if it failed.

### A Simple Example

```rust
fn divide(numerator: f64, denominator: f64) -> Result<f64, String> {
    if denominator == 0.0 {
        Err(String::from("division by zero"))
    } else {
        Ok(numerator / denominator)
    }
}

fn main() {
    match divide(10.0, 2.0) {
        Ok(value) => println!("result: {value}"),
        Err(message) => println!("error: {message}"),
    }

    match divide(10.0, 0.0) {
        Ok(value) => println!("result: {value}"),
        Err(message) => println!("error: {message}"),
    }
}
```

The function returns either `Ok(value)` for success or `Err(message)` for failure. The caller handles both cases with `match`.

### Why Not Exceptions?

Most languages handle errors with exceptions: a function can throw an exception, which propagates up the call stack until something catches it. This is convenient but has costs:

- Exceptions are invisible in the type system. A function's signature does not tell you what exceptions it might throw.
- Catch-all handlers can hide bugs by silencing exceptions you did not expect.
- The performance overhead of exception machinery is real, especially in hot paths.

Rust's `Result` makes errors visible in the type system. A function that can fail returns `Result<T, E>`. The caller knows from the signature that errors are possible and is forced by the type system to handle them.

### The `?` Operator

Handling `Result` with `match` everywhere is verbose. The `?` operator is a shortcut for "if this is Ok, give me the value; if this is Err, return the error from the current function."

```rust
fn process(input: &str) -> Result<i32, String> {
    let parsed: i32 = input.parse().map_err(|e| e.to_string())?;
    let doubled = parsed * 2;
    Ok(doubled)
}
```

The `?` after `parse().map_err(...)` says: "if this returned Ok, unwrap it and continue; if it returned Err, return that error from this function immediately."

This lets you write code that looks linear and natural even though it has multiple potential failure points:

```rust
fn fetch_and_process(url: &str) -> Result<Data, MyError> {
    let response = http_get(url)?;          // returns early on error
    let body = response.body()?;            // returns early on error
    let parsed = parse(body)?;              // returns early on error
    Ok(transform(parsed))
}
```

Each `?` is a potential exit point, but the happy path reads as a sequence of steps. Module 10 covers `?` and error handling in depth.

### When to Use Result

Use `Result<T, E>` whenever:

- An operation can fail in ways the caller might want to handle.
- The failure is meaningful information rather than an unrecoverable bug.
- You are reading from files, networks, or other I/O.
- You are parsing data from external sources.
- You are validating user input.

For unrecoverable bugs (failed assertions, programming errors), Rust uses panics. The distinction is: `Result` is for situations the program might recover from; `panic!` is for situations the program cannot continue past.

### Option vs Result

A useful rule of thumb:

- If absence is a **normal outcome**, use `Option`.
- If absence is an **error**, use `Result`.

`HashMap::get` returns `Option<&V>` because "key not found" is a normal occurrence. `File::open` returns `Result<File, io::Error>` because failing to open a file is an error condition with a reason worth conveying.

---

## D. The `match` Expression

`match` is the most powerful pattern-matching construct in Rust. It compares a value against a series of patterns and runs the code for the first match.

### The Basic Form

```rust
fn describe(n: i32) -> &'static str {
    match n {
        0 => "zero",
        1 => "one",
        2 => "two",
        _ => "many",
    }
}
```

Three things to note:

- Each arm has a pattern, an arrow `=>`, and a body. Arms are separated by commas.
- The `_` pattern matches anything. It is the catch-all, conventionally placed last.
- The match is an expression. It produces a value (the body of the matching arm) that can be assigned, returned, or used in any expression context.

### Exhaustiveness

The compiler checks that every possible value is covered. Forgetting a case is a compile error:

```rust
fn describe_direction(d: Direction) -> &'static str {
    match d {
        Direction::North => "up",
        Direction::South => "down",
        Direction::East => "right",
        // forgot West
    }
}
```

The compiler rejects this:

```
error[E0004]: non-exhaustive patterns: `Direction::West` not covered
```

This is one of the most useful properties of `match`. If you add a new variant to an enum later, every `match` that handled the old variants will fail to compile until you add the new case. There is no possibility of "forgot to update this place" bugs.

When you genuinely want to handle all remaining cases the same way, use `_`:

```rust
match d {
    Direction::North => "up",
    Direction::South => "down",
    _ => "horizontal",          // covers East and West
}
```

### Match with Enums Carrying Data

When the variants carry data, the patterns destructure them:

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn process(msg: Message) {
    match msg {
        Message::Quit => println!("quitting"),
        Message::Move { x, y } => println!("moving to ({x}, {y})"),
        Message::Write(text) => println!("writing: {text}"),
        Message::ChangeColor(r, g, b) => println!("color: rgb({r}, {g}, {b})"),
    }
}
```

Each pattern not only checks which variant `msg` is but also extracts the inner data. By the time the body of the arm runs, `x`, `y`, `text`, or `r`/`g`/`b` are bound to the actual values.

### Match with Multiple Patterns

You can match multiple patterns in one arm using `|`:

```rust
match n {
    1 | 2 | 3 => "small",
    4 | 5 | 6 => "medium",
    _ => "other",
}
```

Each pattern is tried in order; the arm matches if any of them does.

### Match with Ranges

Range patterns match a range of values:

```rust
match n {
    0..=9 => "single digit",
    10..=99 => "two digits",
    _ => "many digits",
}
```

The `..=` is the inclusive range from Module 4. Ranges work with integers and characters.

### A Practical Example

Pattern matching with enums replaces what would be a verbose `if`/`else` chain in many languages:

```rust
enum Status {
    Active,
    Inactive,
    Pending(u32),       // pending with a wait time
    Failed(String),     // failed with a reason
}

fn describe(status: &Status) -> String {
    match status {
        Status::Active => String::from("currently active"),
        Status::Inactive => String::from("not active"),
        Status::Pending(seconds) => format!("pending for {seconds} seconds"),
        Status::Failed(reason) => format!("failed: {reason}"),
    }
}
```

The match is exhaustive (the compiler verifies every variant is handled), each arm extracts the data it needs, and the result is a single expression. Compare this to the same logic written with chained conditions and explicit field access; the match is shorter, safer, and more readable.

---

## E. `if let` and `while let`

`match` is powerful but verbose for the common case of "I only care about one variant." `if let` and `while let` are concise forms for that pattern.

### `if let`

The `if let` construct runs a block if a pattern matches, and optionally an `else` block if it does not:

```rust
let result: Option<i32> = Some(42);

if let Some(value) = result {
    println!("got {value}");
} else {
    println!("nothing");
}
```

This is equivalent to:

```rust
match result {
    Some(value) => println!("got {value}"),
    None => println!("nothing"),
}
```

For two-variant cases like `Option`, `if let` is shorter and reads more naturally. The pattern `Some(value)` matches the `Some` variant, and `value` is bound to the inner value if it does.

When you only care about one case and want to ignore the rest, drop the `else`:

```rust
if let Some(value) = result {
    println!("got {value}");
}
// no else: silently does nothing for None
```

### `while let`

`while let` runs a block as long as a pattern continues to match. It is most often used with iterators or state that drains:

```rust
let mut stack = vec![1, 2, 3];

while let Some(top) = stack.pop() {
    println!("popped {top}");
}
```

`Vec::pop` returns `Option<T>`: `Some(value)` while there are values, `None` when the vector is empty. The loop continues as long as `pop` returns `Some`, and exits when it returns `None`.

This is much cleaner than the equivalent with `match` or with a manual `while !stack.is_empty()` check.

### When to Use Each

The choice between `match`, `if let`, and `while let`:

- **`match`** when you need to handle every variant (or several variants, with a catch-all).
- **`if let`** when you only care about one variant and want to ignore the rest.
- **`while let`** when you want to loop as long as a value matches a pattern.

`if let` does not check exhaustiveness. If you only handle one variant of a five-variant enum and add a sixth variant later, `if let` will not flag any places that need updating. For that reason, `if let` is best for cases where ignoring the other variants is genuinely correct (such as `Option`, where ignoring `None` means "do nothing if no value").

For larger enums, `match` is usually safer.

---

## F. Destructuring Structs, Enums, and Tuples

Pattern matching is not just for enums. The same syntax works for any structured data. This is one of the most useful features of the language and shows up in many places.

### Destructuring Tuples

You have already seen tuple destructuring in `let` bindings:

```rust
let point = (3, 4);
let (x, y) = point;

println!("x = {x}, y = {y}");
```

The pattern `(x, y)` matches a tuple of two elements and binds the elements to `x` and `y`.

The same works in function parameters:

```rust
fn print_point((x, y): (i32, i32)) {
    println!("({x}, {y})");
}

fn main() {
    let p = (3, 4);
    print_point(p);
}
```

The parameter pattern destructures the tuple as soon as the function is called.

### Destructuring Structs

Structs can be destructured the same way:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 3, y: 4 };

    let Point { x, y } = p;
    println!("x = {x}, y = {y}");
}
```

The pattern `Point { x, y }` matches any `Point` and binds `x` and `y` to its fields. You can also rename the bindings:

```rust
let Point { x: a, y: b } = p;
println!("a = {a}, b = {b}");
```

The pattern `{ x: a }` says "match the `x` field and bind it to `a`."

### Ignoring Fields

To ignore some fields, use `..` to mean "any other fields":

```rust
let p = Point { x: 3, y: 4 };

let Point { x, .. } = p;        // bind x, ignore the rest
println!("x = {x}");
```

Or use `_` to ignore a specific field by name:

```rust
let Point { x, y: _ } = p;
```

### Destructuring in Match

All of these patterns work inside `match` arms:

```rust
fn describe_point(p: Point) -> String {
    match p {
        Point { x: 0, y: 0 } => String::from("origin"),
        Point { x: 0, y } => format!("on the y-axis at {y}"),
        Point { x, y: 0 } => format!("on the x-axis at {x}"),
        Point { x, y } => format!("at ({x}, {y})"),
    }
}
```

This combines literal matching (`x: 0` matches only when `x` equals 0) with binding (`y` matches anything and binds it). The arms are tried in order; the most specific patterns come first.

### Destructuring Enum Variants

You have already seen this in Section D. Each variant can be destructured to extract its data:

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle(f64, f64, f64),
}

fn area(s: Shape) -> f64 {
    match s {
        Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        Shape::Rectangle { width, height } => width * height,
        Shape::Triangle(a, b, c) => {
            let s = (a + b + c) / 2.0;
            (s * (s - a) * (s - b) * (s - c)).sqrt()
        }
    }
}
```

Each pattern matches its variant and extracts the relevant data. The body of the arm uses the bound values directly.

### Nested Patterns

Patterns can be nested. For example, matching an `Option<Point>`:

```rust
fn describe_optional(p: Option<Point>) -> String {
    match p {
        Some(Point { x: 0, y: 0 }) => String::from("the origin"),
        Some(Point { x, y }) => format!("({x}, {y})"),
        None => String::from("no point"),
    }
}
```

The pattern `Some(Point { x: 0, y: 0 })` matches only when the option is `Some` AND the inner point has `x` and `y` both zero. This kind of nested matching is one of the things that makes patterns expressive.

---

## G. Refutability: Irrefutable vs. Refutable Patterns

Patterns come in two kinds.

**Irrefutable patterns** match every possible value of the type. They cannot fail. Examples:

- `let x = 5;` (the pattern `x` matches any `i32`)
- `let (a, b) = (1, 2);` (the tuple pattern matches any 2-tuple)
- `let Point { x, y } = p;` (the struct pattern matches any `Point`)
- `let _ = compute();` (the wildcard matches anything)

**Refutable patterns** can fail to match. Examples:

- `Some(value)` (does not match `None`)
- `Ok(value)` (does not match `Err`)
- `Direction::North` (does not match other variants)
- `0` (matches only the value zero)

### Where Each Is Allowed

The two forms appear in different syntactic positions:

- `let` bindings require **irrefutable** patterns. Otherwise the binding might fail to match, and the variable would be uninitialized.
- `if let` and `while let` require **refutable** patterns. Otherwise there would be no point in the conditional; the pattern would always match.
- `match` arms can use either. A refutable pattern might not match (and another arm will), and an irrefutable pattern always matches (and no later arms can be reached).

### Examples

```rust
let x = 5;                      // irrefutable: always matches
let Some(y) = result;           // ERROR: refutable in let

if let Some(y) = result { }     // OK: refutable in if let
if let x = 5 { }                // works but warns: irrefutable in if let

match result {                  // both kinds OK in match
    Some(y) => { /* ... */ },
    _ => { /* ... */ },
}
```

The compiler's diagnostics are clear about which kind of pattern is expected:

```
error[E0005]: refutable pattern in local binding: `None` not covered
  | let Some(y) = result;
  |     ^^^^^^^ pattern `None` not covered
```

The error tells you exactly what is missing: the pattern fails to match `None`, so it cannot be used in `let`.

### `let else`

A useful construct for "try to match this pattern; if it fails, do something":

```rust
fn process(input: Option<i32>) {
    let Some(value) = input else {
        println!("no input");
        return;
    };

    // value is bound to the i32; only reachable if the pattern matched
    println!("got {value}");
}
```

The `let ... else { ... }` syntax says: "match the pattern; if it does not match, run the else block." The else block must diverge (return, panic, or continue out of the function), so the compiler can verify that the binding is always valid afterward.

This avoids deeply-nested `if let` blocks for the common pattern of "extract a value or bail out."

### Why This Distinction?

The distinction between refutable and irrefutable patterns enforces correctness:

- A `let` that could fail would leave the variable uninitialized. Forbidding refutable patterns in `let` makes that impossible.
- An `if let` with an irrefutable pattern is meaningless (the body always runs). The warning helps catch programming mistakes.

For most code, you do not have to think about this consciously. The compiler tells you when you are using the wrong kind of pattern, and the fix is usually obvious.

---

## H. Pattern Guards and Binding with `@`

Two more pattern features round out the topic. Both are advanced but useful in specific situations.

### Pattern Guards

A pattern guard is an `if` condition added to a `match` arm. The arm matches only if the pattern matches AND the guard evaluates to `true`:

```rust
fn classify(n: i32) -> &'static str {
    match n {
        0 => "zero",
        x if x < 0 => "negative",
        x if x % 2 == 0 => "positive even",
        _ => "positive odd",
    }
}
```

The first arm matches only the literal 0. The second arm matches any negative integer (the pattern `x` matches anything, then the guard `x < 0` further restricts). The third arm matches any positive even integer. The fourth arm catches the rest.

Guards can use any expression that returns `bool`. They have access to the bindings introduced by the pattern.

### When Guards Are Useful

Guards are useful when:

- The condition cannot be expressed as a pattern alone. Patterns can match literal values and ranges, but cannot evaluate arbitrary expressions.
- Two cases have similar structure but different conditions. Guards let you keep the structure similar and put the difference in the guard.

```rust
match (status, attempt_count) {
    (Status::Failed(_), 0) => "first attempt failed",
    (Status::Failed(_), n) if n < 3 => "still retrying",
    (Status::Failed(_), _) => "gave up",
    (Status::Active, _) => "running",
    _ => "other",
}
```

The first three arms all match `Status::Failed`, distinguished by the attempt count. The first uses a literal pattern; the second uses a guard.

### Binding with `@`

The `@` operator lets you both match a pattern and bind the matched value to a name. It is useful when you want to test a condition on a value but also want the value itself:

```rust
fn classify(n: i32) -> String {
    match n {
        x @ 0..=9 => format!("single digit: {x}"),
        x @ 10..=99 => format!("two digits: {x}"),
        x => format!("many digits: {x}"),
    }
}
```

The pattern `x @ 0..=9` says: "match values in the range 0 to 9, AND bind the matched value to `x`." Without the `@`, you could write `0..=9` to match the range, but you would not have the value bound to a name.

The same construct with structs:

```rust
struct Person { age: u32, name: String }

fn greet(p: Person) -> String {
    match p {
        Person { age: 0..=12, name } => format!("hello young {name}"),
        Person { age: age @ 13..=19, name } => format!("hello {name}, age {age}"),
        Person { age, name } => format!("hello {name}, age {age}"),
    }
}
```

The pattern `age: age @ 13..=19` matches a teenager and binds the actual age to `age`. The first arm does not bind the age (it just matches the range), so the body cannot reference it.

### When to Use `@`

Use `@` when you need both:

- The structural test (a range, a specific variant, a literal pattern).
- The actual value (so you can use it in the body).

Without `@`, you would either need to test the value separately (using a guard) or extract it after matching (less ergonomic). The `@` syntax combines both into one pattern.

It is not used often. When you need it, it is the right tool.

---

## Module Summary

- Enums in Rust let you define types whose values are one of several variants. Variants can carry data of different shapes.
- `Option<T>` represents "a value or nothing" and replaces null. `Result<T, E>` represents "success or failure" and replaces exceptions.
- `match` is the primary pattern-matching construct. It is exhaustive: the compiler verifies every variant is handled.
- `if let` and `while let` are concise forms for the common case of caring about only one variant.
- Pattern matching also destructures structs, enums, tuples, and nested combinations of these.
- Patterns are either refutable (can fail to match) or irrefutable (always match). Each kind is allowed in different syntactic positions.
- Pattern guards (`if condition`) and `@` bindings extend the basic pattern syntax for more complex matches.

## Discussion Questions

1. Tony Hoare called null references his "billion-dollar mistake." Rust replaces null with `Option<T>`. Identify a specific kind of bug that this design choice prevents and a specific kind of code that becomes more verbose because of it. On balance, is the trade-off worth it?
2. The `match` expression is required to be exhaustive. The compiler rejects programs that miss a variant. How does this help when you add a new variant to an existing enum used throughout a large codebase? What would the equivalent process look like in a language without exhaustiveness checking?
3. Pattern matching unifies several features that are separate in other languages: destructuring assignment, conditional checks, and type discrimination. What does Rust gain from this unification? What does it cost in terms of learning curve?


