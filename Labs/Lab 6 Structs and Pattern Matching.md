# Modules 8 and 9: Structs, Enums, and Pattern Matching
## Lab 6 -- Modeling Data with Structs, Enums, and Pattern Matching

> **Course:** Mastering Rust
> **Modules:** 8 - Structs, 9 - Enums and Pattern Matching
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with the tools Rust provides for modeling your own data: structs, enums, and pattern matching. You will work through every concept from Modules 8 and 9: defining structs with named fields, tuple structs, and unit-like structs; methods and associated functions; the constructor pattern; deriving common traits; defining enums with and without data; using `Option` and `Result`; the `match` expression with its exhaustiveness checks; `if let` and `while let` for the common one-variant case; destructuring; refutability; and pattern guards with `@` bindings.

You will build a single Cargo project called `event_system` that models a small event-driven system: events of various kinds, handlers that process them, and a state machine that tracks transitions. The project deliberately uses a wide variety of struct and enum shapes so you encounter the patterns in context rather than in isolation.

The focus is on choosing the right shape for your data: when to use a struct, when to use an enum, when an enum's variant should carry data, and when pattern matching is clearer than if/else chains. Many decisions have multiple defensible answers; the lab guides you toward idiomatic Rust and asks you to compare alternatives.

By the end of this lab you will be able to:

- Define structs with named fields, tuple structs, and unit-like structs, and choose between them.
- Implement methods using `&self`, `&mut self`, and `self` appropriately.
- Apply the constructor pattern with `new` and other associated functions.
- Use `#[derive]` to add `Debug`, `Clone`, `PartialEq`, and other capabilities.
- Define enums whose variants carry different shapes of data.
- Use `Option<T>` and `Result<T, E>` to replace null and exceptions.
- Write exhaustive `match` expressions and recognize when the compiler insists on missing cases.
- Use `if let` and `while let` for the common case of caring about one variant.
- Destructure structs, enums, and tuples in patterns.
- Distinguish refutable from irrefutable patterns and understand where each is allowed.
- Use pattern guards and `@` bindings for advanced matching.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** This lab assumes familiarity with ownership and references from Lab 5. Several exercises pass references to methods, return references from match arms, and require thinking about whether you want to consume or borrow values. If borrows feel unfamiliar, review Module 6 before starting.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new event_system
cd event_system
code .
```

Create a `lab6-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Defining and Instantiating Structs

**Estimated time:** 10 minutes
**Topics covered:** struct syntax, field initialization, mutability, field access

### Context

Structs are the most common way to define data types in Rust. A struct definition introduces a new type with a fixed set of named, typed fields. Once defined, you create instances by listing the field values. This exercise establishes the basics.

### Task 1.1 -- A simple struct

Replace the contents of `src/main.rs` with:

```rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

fn main() {
    let user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        sign_in_count: 1,
        active: true,
    };

    println!("Username: {}", user.username);
    println!("Email: {}", user.email);
    println!("Sign-ins: {}", user.sign_in_count);
    println!("Active: {}", user.active);
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
Username: alice
Email: alice@example.com
Sign-ins: 1
Active: true
```

Three things to note:

- Struct names use `PascalCase`. Field names use `snake_case`.
- Every field must be initialized when creating an instance. There is no concept of an "uninitialized" struct.
- Field access uses dot notation: `user.email`.

### Task 1.2 -- Mutability

Try modifying a field:

```rust
fn main() {
    let user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        sign_in_count: 1,
        active: true,
    };

    user.email = String::from("alice@new.com");        // ERROR
    println!("{}", user.email);
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0594]: cannot assign to `user.email`, as `user` is not declared as mutable
```

The struct is immutable because `user` is declared with `let`, not `let mut`. Field mutability follows the binding's mutability; you cannot have one mutable field while the rest are immutable.

Fix it by making `user` mutable:

```rust
let mut user = User {
    // ... same as before
};

user.email = String::from("alice@new.com");
println!("{}", user.email);
```

Run the program. It now compiles and prints `alice@new.com`.

### Task 1.3 -- Field init shorthand

When variable names match field names, you can use the shorthand. Add this function:

```rust
fn build_user(username: String, email: String) -> User {
    User {
        username,                    // shorthand for username: username
        email,                        // shorthand for email: email
        sign_in_count: 1,
        active: true,
    }
}
```

Use it from `main`:

```rust
fn main() {
    let user = build_user(
        String::from("bob"),
        String::from("bob@example.com"),
    );

    println!("{}: {}", user.username, user.email);
}
```

Run the program. The shorthand is purely cosmetic; it produces exactly the same code as the long form. Use it whenever the names match.

### Task 1.4 -- Struct update syntax

When creating a new struct that is mostly the same as an existing one, the update syntax (`..`) fills in the rest from another struct. Replace `main` with:

```rust
fn main() {
    let user1 = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        sign_in_count: 1,
        active: true,
    };

    let user2 = User {
        email: String::from("alice@new.com"),
        ..user1
    };

    println!("user2.username: {}", user2.username);
    println!("user2.email: {}", user2.email);
}
```

Run the program. Expected output:

```
user2.username: alice
user2.email: alice@new.com
```

The `..user1` syntax says "copy the remaining fields from `user1`." `user2` ends up with `user1`'s `username`, `sign_in_count`, and `active`, plus its own `email`.

### Task 1.5 -- The update syntax can move

The update syntax follows the ownership rules. After the update, try to use `user1`:

```rust
println!("user1.username: {}", user1.username);        // ERROR
```

Run `cargo build`. The compiler rejects this:

```
error[E0382]: borrow of partially moved value: `user1`
  |
  | let user2 = User {
  |     ..user1
  |       ----- value partially moved here
```

The update syntax moved `user1`'s `String` fields (`username`) into `user2`. After the partial move, `user1` cannot be used as a whole. The non-`String` fields (`sign_in_count`, `active`) were `Copy`, so they were copied; the `String` was moved.

This is the same ownership behavior you saw in Lab 5. Update syntax is not magic; it just expands to ordinary field-by-field assignment.

Remove the bad print to continue.

### Checkpoints

1. The compiler error in Task 1.2 mentioned that the *binding* is not mutable, not the field. Why does Rust not let you mark individual fields as mutable while others remain immutable?
2. The field init shorthand requires the variable name to match the field name exactly. Why is exact matching the rule rather than allowing some kind of fuzzy matching?
3. After `..user1` in Task 1.5, `user1` cannot be used as a whole, but the comment notes that the `Copy` fields were copied rather than moved. Could `user1.sign_in_count` be used after the update? Why or why not?

---

## Exercise 2 -- Methods and Associated Functions

**Estimated time:** 10-15 minutes
**Topics covered:** `impl` blocks, the three forms of `self`, associated functions, the constructor pattern

### Context

Methods are functions attached to a type. The first parameter of a method indicates how it accesses the instance: `&self` for read-only, `&mut self` for mutating, or `self` for consuming. Associated functions do not take `self` and are used as constructors and utilities.

### Task 2.1 -- Methods with `&self`

Replace `src/main.rs` with:

```rust
struct Rectangle {
    width: f64,
    height: f64,
}

impl Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }

    fn is_square(&self) -> bool {
        self.width == self.height
    }
}

fn main() {
    let rect = Rectangle { width: 3.0, height: 4.0 };

    println!("Width: {}", rect.width);
    println!("Height: {}", rect.height);
    println!("Area: {}", rect.area());
    println!("Perimeter: {}", rect.perimeter());
    println!("Is square: {}", rect.is_square());
}
```

Run the program. Expected output:

```
Width: 3
Height: 4
Area: 12
Perimeter: 14
Is square: false
```

Three methods, all taking `&self`. They read the instance's fields without modifying it. After all the calls, `rect` is still valid and unchanged.

### Task 2.2 -- Methods with `&mut self`

Add this method:

```rust
impl Rectangle {
    fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }
}
```

Use it:

```rust
fn main() {
    let mut rect = Rectangle { width: 3.0, height: 4.0 };

    println!("Original area: {}", rect.area());

    rect.scale(2.0);

    println!("After scaling 2x: {} x {} = {}", rect.width, rect.height, rect.area());
}
```

Run the program. Expected output:

```
Original area: 12
After scaling 2x: 6 x 8 = 48
```

Two requirements for calling `&mut self` methods:

- The binding must be `mut` (here, `let mut rect`).
- No conflicting borrows can be active when the method is called (the borrow rules from Lab 5).

### Task 2.3 -- Methods with `self`

Add this method that consumes the rectangle:

```rust
impl Rectangle {
    fn into_square(self) -> Rectangle {
        let side = (self.width + self.height) / 2.0;
        Rectangle { width: side, height: side }
    }
}
```

Use it:

```rust
fn main() {
    let rect = Rectangle { width: 3.0, height: 4.0 };

    let square = rect.into_square();
    println!("Square: {} x {}", square.width, square.height);

    // println!("Original: {}", rect.width);     // would fail: rect was consumed
}
```

Run the program. Expected output:

```
Square: 3.5 x 3.5
```

The method takes `self` (no reference), consuming the rectangle. After the call, `rect` is no longer valid.

The naming convention `into_X` signals "this consumes the value and produces an X." It is parallel to `From`/`Into` from Module 13. When you see `into_*` on a type, expect ownership transfer.

### Task 2.4 -- Associated functions

A function in an `impl` block that does not take `self` is an associated function. The most common use is constructors. Add these:

```rust
impl Rectangle {
    fn new(width: f64, height: f64) -> Rectangle {
        Rectangle { width, height }
    }

    fn square(side: f64) -> Rectangle {
        Rectangle { width: side, height: side }
    }
}
```

Use them:

```rust
fn main() {
    let rect = Rectangle::new(3.0, 4.0);
    let sq = Rectangle::square(5.0);

    println!("Rectangle: {} x {} (area {})", rect.width, rect.height, rect.area());
    println!("Square: {} x {} (area {})", sq.width, sq.height, sq.area());
}
```

Run the program. Expected output:

```
Rectangle: 3 x 4 (area 12)
Square: 5 x 5 (area 25)
```

Note the call syntax: associated functions use `Type::function(args)`, not dot notation. There is no instance for the function to operate on, so dot notation does not apply.

### Task 2.5 -- A constructor with validation

Add a constructor that rejects invalid input:

```rust
impl Rectangle {
    fn try_new(width: f64, height: f64) -> Result<Rectangle, String> {
        if width <= 0.0 {
            return Err(format!("width must be positive, got {width}"));
        }
        if height <= 0.0 {
            return Err(format!("height must be positive, got {height}"));
        }
        Ok(Rectangle { width, height })
    }
}
```

Use it:

```rust
fn main() {
    match Rectangle::try_new(3.0, 4.0) {
        Ok(r) => println!("Created: {} x {}", r.width, r.height),
        Err(e) => println!("Error: {e}"),
    }

    match Rectangle::try_new(-1.0, 4.0) {
        Ok(r) => println!("Created: {} x {}", r.width, r.height),
        Err(e) => println!("Error: {e}"),
    }
}
```

Run the program. Expected output:

```
Created: 3 x 4
Error: width must be positive, got -1
```

The convention `try_new` (returning `Result`) parallels `new` (which would either accept any input or panic). When validation matters, the `try_*` form lets the caller handle invalid input gracefully.

### Checkpoints

1. The method `into_square` was named with the `into_` prefix. What does this prefix signal to readers of the code? What other naming conventions for `self`-consuming methods are common?
2. The `try_new` function returns `Result<Rectangle, String>`, while `new` could return `Rectangle` directly. When is each appropriate? What does the choice tell callers about the operation?
3. Associated functions are called with `Type::function(args)`. Methods are called with `instance.method(args)`. Why does Rust use different syntax for these? What would change if the language used the same syntax for both?

---

## Exercise 3 -- Tuple Structs, Unit-Like Structs, and the Newtype Pattern

**Estimated time:** 10 minutes
**Topics covered:** the three kinds of structs, when each is appropriate

### Context

Most structs have named fields, but Rust supports two other shapes. Tuple structs have positional fields without names; unit-like structs have no fields at all. Each has specific use cases.

### Task 3.1 -- Tuple structs

Replace `src/main.rs` with:

```rust
struct Point(f64, f64);
struct Color(u8, u8, u8);

impl Point {
    fn distance_from_origin(&self) -> f64 {
        (self.0 * self.0 + self.1 * self.1).sqrt()
    }
}

fn main() {
    let origin = Point(0.0, 0.0);
    let p = Point(3.0, 4.0);
    let red = Color(255, 0, 0);

    println!("origin: ({}, {})", origin.0, origin.1);
    println!("p: ({}, {})", p.0, p.1);
    println!("p distance from origin: {}", p.distance_from_origin());
    println!("red: ({}, {}, {})", red.0, red.1, red.2);
}
```

Run the program. Expected output:

```
origin: (0, 0)
p: (3, 4)
p distance from origin: 5
red: (255, 0, 0)
```

Tuple structs use positional access (`p.0`, `p.1`, etc.) instead of named fields. They are appropriate when the field names would be redundant (a `Point` is universally understood as `(x, y)`, so naming the fields adds little).

### Task 3.2 -- The newtype pattern

The most useful application of tuple structs is the newtype pattern: wrapping a primitive type to give it a distinct identity. Replace `main.rs` with:

```rust
struct Meters(f64);
struct Feet(f64);

fn convert_meters_to_feet(m: Meters) -> Feet {
    Feet(m.0 * 3.28084)
}

fn main() {
    let height = Meters(100.0);
    let height_feet = convert_meters_to_feet(height);

    println!("converted: {} feet", height_feet.0);

    // The type system prevents unit confusion:
    // let bad = convert_meters_to_feet(Feet(5.0));    // ERROR: expected Meters, got Feet
}
```

Run the program. Expected output:

```
converted: 328.084 feet
```

Both `Meters` and `Feet` wrap an `f64`, but they are different types. The function only accepts `Meters`. Try uncommenting the bad line and see the error:

```
error[E0308]: mismatched types
  |
  |     let bad = convert_meters_to_feet(Feet(5.0));
  |                                       ^^^^^^^^ expected `Meters`, found `Feet`
```

The compiler refuses to silently convert. This eliminates the entire class of "passed feet to a function expecting meters" bugs that has caused real-world problems (the Mars Climate Orbiter is a famous example).

### Task 3.3 -- Validated newtypes

The newtype pattern can also encapsulate validation. Add this to your file:

```rust
struct PositiveInteger(u64);

impl PositiveInteger {
    fn new(value: u64) -> Option<PositiveInteger> {
        if value > 0 {
            Some(PositiveInteger(value))
        } else {
            None
        }
    }

    fn value(&self) -> u64 {
        self.0
    }
}

fn main() {
    let valid = PositiveInteger::new(42);
    let invalid = PositiveInteger::new(0);

    match valid {
        Some(n) => println!("valid: {}", n.value()),
        None => println!("valid: none"),
    }

    match invalid {
        Some(n) => println!("invalid: {}", n.value()),
        None => println!("invalid: none"),
    }
}
```

Run the program. Expected output:

```
valid: 42
invalid: none
```

The constructor rejects zero by returning `None`. Once you have a `PositiveInteger`, you know it is positive; the type system guarantees it. Code that needs a positive integer can require `PositiveInteger`, and the compiler enforces the invariant.

This is a powerful pattern: invariants encoded in types are enforced everywhere automatically, without runtime checks.

### Task 3.4 -- Unit-like structs

A unit-like struct has no fields at all. Add this:

```rust
struct AlwaysEqual;

fn main() {
    let a = AlwaysEqual;
    let b = AlwaysEqual;

    // Both a and b are valid AlwaysEqual instances.
    // They contain no data; they are markers.
    println!("instances created");
}
```

Run the program. Expected output:

```
instances created
```

A unit-like struct is rare in everyday code. It is most commonly used as a marker type for trait implementations or for type-level signaling. You will see it occasionally; it is worth recognizing.

### Checkpoints

1. The newtype pattern in Task 3.2 uses `struct Meters(f64)` to wrap an `f64`. What is the runtime cost of this wrapping compared to using a bare `f64`?
2. Compare the validated newtype in Task 3.3 (`PositiveInteger`) to checking the value at every call site. What does the newtype pattern give you that the runtime check does not?
3. A tuple struct with named fields uses `self.0`, `self.1`, etc. When does this become unwieldy enough that you should switch to a regular struct with named fields? Identify a specific characteristic that signals the switch is needed.

---

## Exercise 4 -- Deriving Traits

**Estimated time:** 10 minutes
**Topics covered:** the `#[derive]` attribute, `Debug`, `Clone`, `Copy`, `PartialEq`

### Context

Rust types do not automatically support printing, copying, or comparing. The `#[derive]` attribute generates these capabilities for you when the language supports it. This exercise covers the most common derived traits.

### Task 4.1 -- Without `Debug`

Replace `src/main.rs` with:

```rust
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    println!("{p:?}");        // ERROR
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0277]: `Point` doesn't implement `Debug`
```

The `{:?}` format placeholder requires the type to implement `Debug`. By default, structs do not.

### Task 4.2 -- Adding `Debug`

Add `#[derive(Debug)]` above the struct:

```rust
#[derive(Debug)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    println!("{p:?}");
    println!("{p:#?}");        // pretty-printed (multi-line)
}
```

Run the program. Expected output:

```
Point { x: 3.0, y: 4.0 }
Point {
    x: 3.0,
    y: 4.0,
}
```

`{:?}` produces a compact one-line representation. `{:#?}` produces a pretty-printed multi-line version. Both are useful for debugging.

The convention is: derive `Debug` on every struct unless there is a specific reason not to. It is invaluable when something goes wrong and you need to print values to figure out what is happening.

### Task 4.3 -- Adding `Clone`

Add `Clone` to the derive:

```rust
#[derive(Debug, Clone)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p1 = Point { x: 3.0, y: 4.0 };
    let p2 = p1.clone();
    println!("p1: {p1:?}");
    println!("p2: {p2:?}");
}
```

Run the program. Both `p1` and `p2` are independent copies. The `.clone()` method works because `Clone` was derived.

### Task 4.4 -- Adding `Copy` for cheap types

For simple types, `Copy` makes assignment automatic (no explicit `.clone()`). Add it:

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p1 = Point { x: 3.0, y: 4.0 };
    let p2 = p1;        // copy, not move
    println!("p1: {p1:?}");        // p1 still valid
    println!("p2: {p2:?}");
}
```

Run the program. Both `p1` and `p2` work. Without `Copy`, the assignment would have moved `p1`, making the first `println!` fail.

A struct can derive `Copy` only if every field is `Copy`. `Point` has `f64` fields, both of which are `Copy`, so this works. If we had a `String` field, deriving `Copy` would fail.

When you derive `Copy`, you must also derive `Clone`. The two go together.

### Task 4.5 -- Adding `PartialEq`

Add `PartialEq` to support `==` and `!=`:

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p1 = Point { x: 3.0, y: 4.0 };
    let p2 = Point { x: 3.0, y: 4.0 };
    let p3 = Point { x: 1.0, y: 2.0 };

    println!("p1 == p2: {}", p1 == p2);
    println!("p1 == p3: {}", p1 == p3);
    println!("p1 != p3: {}", p1 != p3);
}
```

Run the program. Expected output:

```
p1 == p2: true
p1 == p3: false
p1 != p3: true
```

The derived `PartialEq` compares all fields. Two points are equal if and only if both fields are equal.

### Task 4.6 -- The standard derive set

Many structs in idiomatic Rust derive a standard set of traits:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct UserId(u64);

fn main() {
    let id1 = UserId(42);
    let id2 = UserId(42);

    println!("{id1:?} == {id2:?}: {}", id1 == id2);

    // Can be used as HashMap key:
    use std::collections::HashMap;
    let mut scores: HashMap<UserId, i32> = HashMap::new();
    scores.insert(UserId(1), 95);
    scores.insert(UserId(2), 87);

    println!("{scores:?}");
}
```

Run the program. The five derived traits give the type:

- `Debug`: printable with `{:?}`.
- `Clone`: explicit duplication via `.clone()`.
- `PartialEq` and `Eq`: comparable with `==`.
- `Hash`: usable as a `HashMap` key.

This combination is so common that you will see it on nearly every wrapper type.

### Task 4.7 -- When derive is not enough

Try deriving `Copy` on a struct with a non-`Copy` field:

```rust
#[derive(Debug, Clone, Copy)]
struct User {
    name: String,
    id: u64,
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0204]: the trait `Copy` may not be implemented for this type
  |
  | name: String,
  |       ------ this field does not implement `Copy`
```

`String` is not `Copy`. A struct can derive `Copy` only if all its fields are `Copy`. Remove the `Copy` derive (keep `Clone`):

```rust
#[derive(Debug, Clone)]
struct User {
    name: String,
    id: u64,
}
```

This compiles. The struct can be cloned explicitly with `.clone()` but is moved on assignment.

### Checkpoints

1. The derive `Copy` requires all fields to be `Copy`. Why is this rule absolute rather than allowing `Copy` on structs with non-`Copy` fields by deep-copying?
2. The list of derivable traits is small (Debug, Clone, Copy, PartialEq, Eq, Hash, Default, PartialOrd, Ord). Why does `Display` not appear in this list, given that it would be useful?
3. Task 4.6 derived `Hash` along with `Eq`. Why are these two traits typically derived together? What constraint does using a type as a HashMap key impose?

---

## Exercise 5 -- Defining and Using Enums

**Estimated time:** 10-15 minutes
**Topics covered:** simple enums, enums with data, methods on enums, the standard `Option` and `Result`

### Context

Enums in Rust are more powerful than in C or Java. Each variant can carry its own data of its own shape. Combined with pattern matching, enums become one of the most useful features in the language.

### Task 5.1 -- A simple enum

Replace `src/main.rs` with:

```rust
#[derive(Debug)]
enum Direction {
    North,
    South,
    East,
    West,
}

impl Direction {
    fn opposite(&self) -> Direction {
        match self {
            Direction::North => Direction::South,
            Direction::South => Direction::North,
            Direction::East => Direction::West,
            Direction::West => Direction::East,
        }
    }
}

fn main() {
    let heading = Direction::North;
    let reverse = heading.opposite();

    println!("Heading: {heading:?}");
    println!("Reverse: {reverse:?}");
}
```

Run the program. Expected output:

```
Heading: North
Reverse: South
```

The enum has four variants. The method examines `self` with a `match` expression and returns a different value for each variant. The compiler verifies the match is exhaustive (we will explore this in Exercise 6).

Note that `heading` was not consumed by the call to `opposite()` because the method takes `&self`. The variable is still valid afterward.

### Task 5.2 -- An enum with data

Variants can carry data. Replace the file with:

```rust
#[derive(Debug)]
enum Event {
    Click { x: i32, y: i32 },
    KeyPress(char),
    Resize(u32, u32),
    Quit,
}

fn describe(event: &Event) -> String {
    match event {
        Event::Click { x, y } => format!("clicked at ({x}, {y})"),
        Event::KeyPress(key) => format!("pressed '{key}'"),
        Event::Resize(width, height) => format!("resized to {width}x{height}"),
        Event::Quit => String::from("quitting"),
    }
}

fn main() {
    let events = vec![
        Event::Click { x: 100, y: 200 },
        Event::KeyPress('a'),
        Event::Resize(1024, 768),
        Event::Quit,
    ];

    for event in &events {
        println!("{}", describe(event));
    }
}
```

Run the program. Expected output:

```
clicked at (100, 200)
pressed 'a'
resized to 1024x768
quitting
```

Each variant has a different shape:

- `Click` has named fields (like a struct).
- `KeyPress` has a single positional field (like a tuple struct).
- `Resize` has two positional fields.
- `Quit` has no data (like a unit-like struct).

A single enum can mix all four shapes. Each variant represents one possible kind of event, with whatever data that kind needs.

### Task 5.3 -- Methods on enums

Add a method:

```rust
impl Event {
    fn category(&self) -> &str {
        match self {
            Event::Click { .. } => "input",
            Event::KeyPress(_) => "input",
            Event::Resize(_, _) => "window",
            Event::Quit => "lifecycle",
        }
    }
}
```

Use it in `main`:

```rust
fn main() {
    let events = vec![
        Event::Click { x: 100, y: 200 },
        Event::KeyPress('a'),
        Event::Resize(1024, 768),
        Event::Quit,
    ];

    for event in &events {
        println!("{} ({})", describe(event), event.category());
    }
}
```

Run the program. Expected output:

```
clicked at (100, 200) (input)
pressed 'a' (input)
resized to 1024x768 (window)
quitting (lifecycle)
```

Two new patterns appear in the match:

- `Event::Click { .. }` matches the variant without binding any of its fields. Useful when you only care which variant it is.
- `Event::KeyPress(_)` matches the variant but ignores its single field. Same idea for tuple-style variants.

### Task 5.4 -- The Option enum

`Option<T>` is the standard library's enum for "a value or nothing." Replace `main.rs` with:

```rust
fn find_user(id: u64) -> Option<String> {
    if id == 1 {
        Some(String::from("Alice"))
    } else if id == 2 {
        Some(String::from("Bob"))
    } else {
        None
    }
}

fn main() {
    let result1 = find_user(1);
    let result2 = find_user(99);

    match result1 {
        Some(name) => println!("Found: {name}"),
        None => println!("Not found"),
    }

    match result2 {
        Some(name) => println!("Found: {name}"),
        None => println!("Not found"),
    }
}
```

Run the program. Expected output:

```
Found: Alice
Not found
```

`Option<String>` says "this is either Some String or None." The function returns one or the other; the caller has to handle both cases. The compiler enforces the handling.

### Task 5.5 -- The Result enum

`Result<T, E>` is the standard library's enum for "success or failure." Add this:

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    let r1 = divide(10.0, 2.0);
    let r2 = divide(10.0, 0.0);

    match r1 {
        Ok(value) => println!("{value}"),
        Err(message) => println!("error: {message}"),
    }

    match r2 {
        Ok(value) => println!("{value}"),
        Err(message) => println!("error: {message}"),
    }
}
```

Run the program. Expected output:

```
5
error: division by zero
```

`Result<f64, String>` says "this is either Ok(f64) or Err(String)." Like `Option`, the caller must handle both cases.

### Checkpoints

1. The `Event` enum mixes four different variant shapes (struct-like, tuple-like, multi-field, unit-like). Why does Rust support all four shapes rather than requiring one canonical form?
2. The pattern `Event::Click { .. }` matches without binding any fields. The pattern `Event::KeyPress(_)` matches without binding the single field. Why are the syntaxes different?
3. Both `Option<T>` and `Result<T, E>` are enums with two variants. Why are they separate types in the standard library rather than a single more general enum?

---

## Exercise 6 -- Exhaustive Matching and `if let`

**Estimated time:** 10-15 minutes
**Topics covered:** the exhaustiveness check, the `_` catch-all, `if let`, `while let`

### Context

The `match` expression must handle every possible case. The compiler verifies this; missing a case is a compile error. For the common case of caring about only one variant, `if let` and `while let` are concise alternatives.

### Task 6.1 -- The exhaustiveness check

Replace `src/main.rs` with:

```rust
#[derive(Debug)]
enum Status {
    Active,
    Inactive,
    Pending(u32),
    Failed(String),
}

fn describe(status: &Status) -> String {
    match status {
        Status::Active => String::from("active"),
        Status::Inactive => String::from("inactive"),
        Status::Pending(seconds) => format!("pending for {seconds}s"),
        // Missing: Status::Failed
    }
}

fn main() {
    let s = Status::Failed(String::from("network error"));
    println!("{}", describe(&s));
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0004]: non-exhaustive patterns: `&Status::Failed(_)` not covered
```

The error tells you exactly which case is missing. This is one of the most useful properties of `match`: if you add a new variant to an enum, every match expression that handled the old variants will fail to compile until you add the new case. There is no "forgot to update this place" possibility.

Add the missing arm:

```rust
Status::Failed(reason) => format!("failed: {reason}"),
```

Now the program compiles and prints `failed: network error`.

### Task 6.2 -- The catch-all

Sometimes you genuinely want to handle several remaining cases the same way. Replace `describe` with:

```rust
fn describe(status: &Status) -> String {
    match status {
        Status::Pending(seconds) => format!("pending for {seconds}s"),
        Status::Failed(reason) => format!("failed: {reason}"),
        _ => String::from("not actively running"),     // catches Active and Inactive
    }
}

fn main() {
    let cases = vec![
        Status::Active,
        Status::Inactive,
        Status::Pending(30),
        Status::Failed(String::from("timeout")),
    ];

    for s in &cases {
        println!("{:?}: {}", s, describe(s));
    }
}
```

Run the program. Expected output:

```
Active: not actively running
Inactive: not actively running
Pending(30): pending for 30s
Failed("timeout"): failed: timeout
```

The `_` arm catches anything not matched by earlier arms. Convention: place it last (it would be unreachable if placed earlier, since match arms are tried in order).

The `_` is the catch-all. To bind the value to a name (so you can reference it in the body), use a name instead of `_`:

```rust
match status {
    Status::Failed(reason) => format!("failed: {reason}"),
    other => format!("other: {other:?}"),       // binds the value to `other`
}
```

### Task 6.3 -- Multiple patterns with `|`

You can match multiple patterns in one arm using `|`:

```rust
fn describe(status: &Status) -> String {
    match status {
        Status::Active | Status::Inactive => String::from("not running"),
        Status::Pending(_) | Status::Failed(_) => String::from("transitional"),
    }
}

fn main() {
    let cases = vec![
        Status::Active,
        Status::Inactive,
        Status::Pending(30),
        Status::Failed(String::from("err")),
    ];

    for s in &cases {
        println!("{:?}: {}", s, describe(s));
    }
}
```

Run the program. Expected output:

```
Active: not running
Inactive: not running
Pending(30): transitional
Failed("err"): transitional
```

The arm matches if any of its patterns matches. This is concise when several variants should be handled the same way.

### Task 6.4 -- `if let` for one-variant cases

Sometimes you only care about one variant. `if let` is more concise than `match`:

```rust
fn main() {
    let result: Option<i32> = Some(42);

    // The match version:
    match result {
        Some(value) => println!("got {value}"),
        None => {} // we do not care about None
    }

    // The if let version:
    if let Some(value) = result {
        println!("got {value}");
    }
}
```

Run the program. Both forms produce the same output. The `if let` form drops the boilerplate of an empty `None` arm.

### Task 6.5 -- `if let else`

`if let` can have an `else` branch:

```rust
fn main() {
    let result: Option<i32> = None;

    if let Some(value) = result {
        println!("got {value}");
    } else {
        println!("nothing");
    }
}
```

Run the program. Expected output:

```
nothing
```

This is sometimes the most natural form for two-variant enums like `Option`. For larger enums, `match` is usually clearer because of exhaustiveness checking.

### Task 6.6 -- `while let`

`while let` runs a block as long as a pattern continues to match:

```rust
fn main() {
    let mut stack = vec![1, 2, 3, 4, 5];

    while let Some(top) = stack.pop() {
        println!("popped: {top}");
    }

    println!("stack is now empty");
}
```

Run the program. Expected output:

```
popped: 5
popped: 4
popped: 3
popped: 2
popped: 1
stack is now empty
```

`Vec::pop` returns `Option<T>`: `Some(value)` while the vector has elements, `None` when empty. The loop continues as long as `pop` returns `Some` and exits when it returns `None`.

This is much cleaner than the alternatives (a manual `while !stack.is_empty()` with separate `pop` calls, or a `loop` with `match`).

### Task 6.7 -- The trade-off

`if let` is concise but does not check exhaustiveness. If you add a variant to the enum later, an `if let` that used to be sufficient might silently miss the new case. Demonstrate this by changing your `Status` enum:

```rust
#[derive(Debug)]
enum Status {
    Active,
    Inactive,
    Pending(u32),
    Failed(String),
    Cancelled,           // new variant
}

fn handle(status: &Status) {
    if let Status::Failed(reason) = status {
        println!("Failed: {reason}");
    }
    // Other variants are silently ignored. Cancelled is now ignored too.
}

fn main() {
    let cases = vec![
        Status::Active,
        Status::Failed(String::from("timeout")),
        Status::Cancelled,
    ];

    for s in &cases {
        handle(s);
    }
}
```

Run the program. Only the `Failed` case produces output. The new `Cancelled` variant was not handled, but the compiler did not warn.

If `handle` had used `match`, the compiler would have flagged it as non-exhaustive when `Cancelled` was added. The `if let` form is concise but lacks that safety net.

The rule of thumb: `if let` for genuinely two-variant enums (especially `Option`); `match` for everything else.

### Checkpoints

1. The exhaustiveness check (Task 6.1) catches missing cases at compile time. Identify a specific kind of bug this prevents that would be hard to find with runtime testing.
2. The `_` catch-all in Task 6.2 made the code shorter, but it has a downside that `match` without it does not. What happens if you add a new variant after writing the catch-all? Why might this be a problem?
3. Task 6.7 showed that `if let` does not check exhaustiveness. Could the language have made `if let` exhaustive? What would that mean and why did the language not go that direction?

---

## Exercise 7 -- Destructuring and Pattern Guards

**Estimated time:** 10-15 minutes
**Topics covered:** destructuring structs/enums/tuples, pattern guards, `@` bindings, refutability

### Context

Pattern matching is not just for enum variants. Patterns can destructure structs, tuples, and nested combinations. Pattern guards add boolean conditions to match arms. The `@` operator combines pattern matching with binding.

### Task 7.1 -- Destructuring tuples

Replace `src/main.rs` with:

```rust
fn classify_position(point: (i32, i32)) -> &'static str {
    match point {
        (0, 0) => "origin",
        (0, _) => "on y-axis",
        (_, 0) => "on x-axis",
        (x, y) if x == y => "on the diagonal y=x",
        (_, _) => "elsewhere",
    }
}

fn main() {
    let points = vec![(0, 0), (0, 5), (3, 0), (4, 4), (1, 2)];

    for p in points {
        println!("{p:?}: {}", classify_position(p));
    }
}
```

Run the program. Expected output:

```
(0, 0): origin
(0, 5): on y-axis
(3, 0): on x-axis
(4, 4): on the diagonal y=x
(1, 2): elsewhere
```

Two patterns of interest:

- `(0, _)`: matches a tuple where the first element is 0 and the second is anything.
- `(x, y) if x == y`: matches any tuple, binds elements to `x` and `y`, then the guard `if x == y` further restricts.

The pattern guard `if x == y` is what makes the diagonal check work. Patterns alone can match against literals and ranges; guards can express arbitrary conditions.

### Task 7.2 -- Destructuring structs

Add a struct and its destructuring patterns:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn classify_struct_point(p: &Point) -> &'static str {
    match p {
        Point { x: 0, y: 0 } => "origin",
        Point { x: 0, y: _ } => "on y-axis",
        Point { x: _, y: 0 } => "on x-axis",
        Point { x, y } if x == y => "on the diagonal y=x",
        Point { .. } => "elsewhere",
    }
}

fn main() {
    let points = vec![
        Point { x: 0, y: 0 },
        Point { x: 0, y: 5 },
        Point { x: 3, y: 0 },
        Point { x: 4, y: 4 },
        Point { x: 1, y: 2 },
    ];

    for p in &points {
        println!("({}, {}): {}", p.x, p.y, classify_struct_point(p));
    }
}
```

Run the program. Expected output (matches the tuple version):

```
(0, 0): origin
(0, 5): on y-axis
(3, 0): on x-axis
(4, 4): on the diagonal y=x
(1, 2): elsewhere
```

The struct destructuring uses `Field: pattern` syntax. `Point { x: 0, y: _ }` matches when `x` is 0 and `y` is anything. The shorthand `Point { x, y }` is equivalent to `Point { x: x, y: y }`, binding both fields.

The pattern `Point { .. }` matches any `Point` and ignores all fields. It is the catch-all for structs, parallel to `_` for tuples.

### Task 7.3 -- Pattern guards on enums

Combine pattern matching with guards on a more interesting type:

```rust
#[derive(Debug)]
enum Event {
    Login { user_id: u32, attempt: u32 },
    Logout { user_id: u32 },
    Failed { user_id: u32, attempt: u32 },
}

fn handle(event: &Event) -> String {
    match event {
        Event::Login { user_id, attempt: 1 } => format!("first login for user {user_id}"),
        Event::Login { user_id, attempt } => format!("login attempt {attempt} for user {user_id}"),
        Event::Failed { user_id, attempt } if *attempt >= 3 => {
            format!("user {user_id} locked out after {attempt} attempts")
        }
        Event::Failed { user_id, attempt } => {
            format!("user {user_id} failed (attempt {attempt})")
        }
        Event::Logout { user_id } => format!("user {user_id} logged out"),
    }
}

fn main() {
    let events = vec![
        Event::Login { user_id: 1, attempt: 1 },
        Event::Login { user_id: 2, attempt: 3 },
        Event::Failed { user_id: 3, attempt: 2 },
        Event::Failed { user_id: 3, attempt: 5 },
        Event::Logout { user_id: 1 },
    ];

    for e in &events {
        println!("{}", handle(e));
    }
}
```

Run the program. Expected output:

```
first login for user 1
login attempt 3 for user 2
user 3 failed (attempt 2)
user 3 locked out after 5 attempts
user 1 logged out
```

The match arms become specific:

- `attempt: 1` matches only when attempt is exactly 1 (literal pattern).
- `attempt` matches any value and binds it.
- `if *attempt >= 3` is a guard that further restricts the arm.

Note `*attempt` in the guard. The pattern bound `attempt` to a reference (because we matched on `&Event`), so the guard needs to dereference to compare with the integer.

### Task 7.4 -- The `@` binding

Sometimes you want to test a pattern AND have access to the matched value. Use `@`:

```rust
fn classify(n: i32) -> String {
    match n {
        n @ 0..=9 => format!("single digit: {n}"),
        n @ 10..=99 => format!("two digits: {n}"),
        n @ 100..=999 => format!("three digits: {n}"),
        n => format!("other: {n}"),
    }
}

fn main() {
    let numbers = vec![5, 42, 999, 1000];
    for n in numbers {
        println!("{}", classify(n));
    }
}
```

Run the program. Expected output:

```
single digit: 5
two digits: 42
three digits: 999
other: 1000
```

The pattern `n @ 0..=9` says: "match values in 0..=9 AND bind the matched value to `n`." Without the `@`, you could match the range, but you would not have the value bound to a name to use in the body.

Compare with the alternative using a guard:

```rust
n if n >= 0 && n <= 9 => format!("single digit: {n}"),
```

Both work, but the `@` form is more declarative: the test and the binding are part of the same pattern.

### Task 7.5 -- Refutable vs irrefutable patterns

Some patterns can fail to match (refutable); others always match (irrefutable). The two forms appear in different syntactic positions. Try this:

```rust
fn main() {
    let pair = (1, 2);
    let (a, b) = pair;          // OK: pattern always matches
    println!("{a}, {b}");

    let result: Option<i32> = Some(42);
    let Some(value) = result;    // ERROR: refutable in let
    println!("{value}");
}
```

Run `cargo build`. The compiler rejects the second `let`:

```
error[E0005]: refutable pattern in local binding
```

The pattern `Some(value)` could fail (if `result` were `None`). `let` requires patterns that always match.

The fix is to use `let else`, which provides a fallback for the failure case:

```rust
fn main() {
    let result: Option<i32> = Some(42);

    let Some(value) = result else {
        println!("got nothing");
        return;
    };

    println!("got {value}");
}
```

Run the program. Expected output:

```
got 42
```

The `let else` form says: "match the pattern; if it fails, run the else block, which must diverge (return, panic, break, or continue)." After the `let else`, the binding is guaranteed to be valid.

This is the cleanest pattern for "extract a value from an Option, or bail out otherwise." It avoids deeply-nested `if let` blocks for the common case.

### Checkpoints

1. The `@` binding in Task 7.4 combines a structural test (a range) with a name binding. Could you achieve the same effect using a pattern guard? What is the difference between the two approaches?
2. Pattern matching on a `&Event` (Task 7.3) bound the fields as references, requiring `*attempt` in the guard. Why does the compiler not automatically dereference in the guard?
3. The `let else` form (Task 7.5) requires the else block to diverge (return, panic, break, etc.). Why is this requirement necessary? What would go wrong if the else block were allowed to "fall through" to the code after?

---

## Exercise 8 -- Putting It Together

**Estimated time:** 15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from Modules 8 and 9 into one program: structs and enums, methods and constructors, derived traits, pattern matching with all its features, `Option` and `Result`, and `if let`/`while let`. The program models a small event-driven system: a state machine that processes events and tracks state transitions.

### Task 8.1 -- The full program

Replace `src/main.rs` with:

```rust
#[derive(Debug, Clone, PartialEq)]
enum State {
    Idle,
    Running { iteration: u32 },
    Paused { reason: String },
    Stopped,
}

#[derive(Debug, Clone)]
enum Event {
    Start,
    Pause(String),
    Resume,
    Stop,
    Tick,
}

#[derive(Debug)]
struct Machine {
    name: String,
    state: State,
    history: Vec<State>,
}

impl Machine {
    fn new(name: String) -> Machine {
        Machine {
            name,
            state: State::Idle,
            history: vec![State::Idle],
        }
    }

    fn handle(&mut self, event: &Event) -> Result<(), String> {
        let next_state = self.next_state(event)?;
        self.history.push(self.state.clone());
        self.state = next_state;
        Ok(())
    }

    fn next_state(&self, event: &Event) -> Result<State, String> {
        match (&self.state, event) {
            (State::Idle, Event::Start) => Ok(State::Running { iteration: 0 }),
            (State::Running { iteration }, Event::Tick) => {
                Ok(State::Running { iteration: iteration + 1 })
            }
            (State::Running { .. }, Event::Pause(reason)) => {
                Ok(State::Paused { reason: reason.clone() })
            }
            (State::Paused { .. }, Event::Resume) => Ok(State::Running { iteration: 0 }),
            (State::Running { .. }, Event::Stop) | (State::Paused { .. }, Event::Stop) => {
                Ok(State::Stopped)
            }
            (state, event) => Err(format!(
                "invalid transition: {state:?} cannot handle {event:?}"
            )),
        }
    }

    fn summary(&self) -> String {
        let state_desc = match &self.state {
            State::Idle => String::from("idle"),
            State::Running { iteration } => format!("running (iteration {iteration})"),
            State::Paused { reason } => format!("paused: {reason}"),
            State::Stopped => String::from("stopped"),
        };

        format!("{}: {} ({} transitions)", self.name, state_desc, self.history.len() - 1)
    }
}

fn main() {
    let mut machine = Machine::new(String::from("worker-1"));

    let events = vec![
        Event::Start,
        Event::Tick,
        Event::Tick,
        Event::Pause(String::from("manual hold")),
        Event::Resume,
        Event::Tick,
        Event::Stop,
    ];

    for event in &events {
        match machine.handle(event) {
            Ok(()) => println!("Applied {event:?}: {}", machine.summary()),
            Err(e) => println!("Rejected {event:?}: {e}"),
        }
    }

    // Try to apply an event that is invalid in the current state:
    let invalid_event = Event::Start;
    if let Err(e) = machine.handle(&invalid_event) {
        println!("As expected: {e}");
    }

    // Walk the history:
    println!("\nState history:");
    let mut iter = machine.history.iter().enumerate();
    while let Some((i, state)) = iter.next() {
        println!("  step {i}: {state:?}");
    }
}
```

Run the program. Expected output:

```
Applied Start: worker-1: running (iteration 0) (1 transitions)
Applied Tick: worker-1: running (iteration 1) (2 transitions)
Applied Tick: worker-1: running (iteration 2) (3 transitions)
Applied Pause("manual hold"): worker-1: paused: manual hold (4 transitions)
Applied Resume: worker-1: running (iteration 0) (5 transitions)
Applied Tick: worker-1: running (iteration 1) (6 transitions)
Applied Stop: worker-1: stopped (7 transitions)
As expected: invalid transition: Stopped cannot handle Start

State history:
  step 0: Idle
  step 1: Idle
  step 2: Running { iteration: 0 }
  step 3: Running { iteration: 1 }
  step 4: Running { iteration: 2 }
  step 5: Paused { reason: "manual hold" }
  step 6: Running { iteration: 0 }
  step 7: Running { iteration: 1 }
```

### Task 8.2 -- Identify the patterns

In `lab6-notes.md`, identify each instance of the following patterns in the program above:

1. A struct with both owned data (`String`) and a collection (`Vec<State>`).
2. An enum where different variants carry different shapes of data.
3. An enum variant with a single tuple-style field.
4. An enum variant with named fields.
5. A unit-like enum variant (no data).
6. A method that takes `&self` for read-only access.
7. A method that takes `&mut self` to modify the struct.
8. An associated function used as a constructor.
9. A `match` expression that destructures an enum variant with named fields.
10. A `match` expression on a tuple of two values (for the state machine transition).
11. A use of `if let Err(e) = ...` for the one-variant case.
12. A use of `while let` to walk an iterator.
13. A use of `#[derive]` to add `Debug`, `Clone`, and `PartialEq`.
14. The `|` operator to match multiple patterns in one arm.

### Task 8.3 -- Predict and verify

For each of the following code changes, predict whether the program will compile, and run it to verify. Write your prediction in `lab6-notes.md` BEFORE running.

**Change A:** Remove the catch-all arm `(state, event) => Err(...)` from `next_state`:

**Change B:** Change the `next_state` function's `match` to use `&self.state` and `event` directly (without the tuple):

```rust
fn next_state(&self, event: &Event) -> Result<State, String> {
    match &self.state {
        State::Idle => match event {
            Event::Start => Ok(State::Running { iteration: 0 }),
            _ => Err(format!("invalid: {event:?} from Idle")),
        },
        // ... other states
    }
}
```

**Change C:** Add a new variant `State::Failed(String)` to the `State` enum but do not update `next_state` or `summary`:

For each change, write your prediction first, then run and observe the actual result.

### Checkpoints

1. The `next_state` method matches on a tuple `(&self.state, event)`. What does this enable that two nested matches would not?
2. The `Machine::handle` method calls `self.next_state(event)?` and uses `?` to propagate the error. The `?` operator works because `next_state` and `handle` both return `Result`. If `handle` returned `()` instead, what would change?
3. The `summary` method takes `&self`, but it constructs and returns a new `String`. Could it return `&str` instead? What would change at the call site?

---

## Summary and Reflection

You have now used every concept from Modules 8 and 9 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Defining Structs | named fields, mutability, init shorthand, update syntax | Structs are immutable by default; mutability is per-binding, not per-field. |
| 2 -- Methods | `&self`, `&mut self`, `self`, associated functions | The first parameter determines how the method accesses the instance. |
| 3 -- Tuple and Unit Structs | the newtype pattern, type-level distinctions | Wrapping a primitive in a struct gives it a distinct identity at zero runtime cost. |
| 4 -- Deriving Traits | `Debug`, `Clone`, `Copy`, `PartialEq`, `Hash` | Common trait implementations come from one line. The standard set is widely used. |
| 5 -- Defining Enums | variants with data, methods on enums, `Option`, `Result` | Each variant can carry its own shape of data. The compiler tracks all variants. |
| 6 -- Match and if let | exhaustiveness, the catch-all, `if let`, `while let` | The compiler verifies all cases are handled. `if let` is a concise alternative for one variant. |
| 7 -- Destructuring and Guards | structs/enums/tuples, guards, `@`, refutability | Patterns destructure data while testing it. Guards add boolean conditions. |
| 8 -- Integration | all of the above | Real Rust programs combine these patterns; reading code closely is how you learn the idioms. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab6-notes.md` before your next session.

1. Of the concepts in Modules 8 and 9, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. Rust's enums (with data per variant) and pattern matching together replace several features that are separate in other languages: union types, sealed class hierarchies, switch statements, destructuring assignment, and type discrimination. Pick one of these features from a language you know and compare how that language solves the same problem. What does Rust gain by unifying them, and what does it lose?

3. The lab covered both structs (Module 8) and enums (Module 9). When designing a new type, you often have a choice: model it as a struct with optional fields, or as an enum with variants. Identify a specific situation where you would choose each approach. What criteria help you decide?

---

*End of Lab 6*
