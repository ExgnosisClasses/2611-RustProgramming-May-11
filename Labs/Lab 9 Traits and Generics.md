# Module 13: Traits and Generics
## Lab 9 -- Defining Traits, Writing Generic Code, and Using the Standard Trait Vocabulary

> **Course:** Mastering Rust
> **Module:** 13 - Traits and Generics
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 13: defining traits and implementing them, default methods, trait bounds in their various syntactic forms (`impl Trait`, named generics, `where` clauses), returning trait types statically and dynamically, defining generic functions and structs, recognizing the standard-library traits, operator overloading, and the newtype pattern.

You will build a single Cargo project called `shape_library` that models geometric shapes through a hierarchy of traits, then exercises every trait and generic concept on top of that foundation. Geometric shapes are a natural fit for traits because they share behavior (computing area, perimeter) while differing in shape-specific data. The lab progresses from single traits on single types to generic functions with multiple bounds to operator overloading with custom types.

The focus is on judgment: when a trait is the right tool versus a simple function, when to use static dispatch versus dynamic dispatch, when to add a bound versus refactor the design. Many decisions have multiple defensible answers; the lab guides you toward idiomatic Rust and asks you to compare alternatives.

By the end of this lab you will be able to:

- Define traits with required and default methods and implement them on multiple types.
- Use trait bounds in three syntactic forms (`impl Trait`, generics, `where`) and choose between them.
- Return traits using `impl Trait` for static dispatch and `Box<dyn Trait>` for dynamic dispatch.
- Define and use generic functions, structs, and enums.
- Combine multiple trait bounds.
- Implement common standard-library traits (`Display`, `PartialEq`, `From`, `Default`).
- Overload operators using the `std::ops` traits.
- Apply the newtype pattern to add behavior to existing types.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** This lab assumes familiarity with structs and enums from Lab 6, and with iterators from Lab 8. Several exercises pass closures to generic functions and return them from functions; if closures still feel unfamiliar, review Module 5 before starting. Trait bounds on generic functions can produce verbose compiler errors when types do not satisfy them; reading these errors carefully is part of the learning.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new shape_library
cd shape_library
code .
```

Create a `lab9-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Defining and Implementing a Trait

**Estimated time:** 10-15 minutes
**Topics covered:** trait declaration, implementing a trait on multiple types, calling trait methods

### Context

A trait is a named contract: "any type that implements this trait can do these things." This exercise defines a `Shape` trait and implements it on three different types, demonstrating the basic mechanics.

### Task 1.1 -- Define the trait

Replace the contents of `src/main.rs` with:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;
}
```

The trait declares two methods. Any type implementing `Shape` must provide both. The signatures alone do not contain bodies; the implementor writes the bodies.

### Task 1.2 -- Implement on a Rectangle

Add a struct and an implementation:

```rust
struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }
}

fn main() {
    let r = Rectangle { width: 3.0, height: 4.0 };
    println!("rectangle area: {}", r.area());
    println!("rectangle perimeter: {}", r.perimeter());
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
rectangle area: 12
rectangle perimeter: 14
```

The `impl Shape for Rectangle` block provides the methods the trait requires. The `Rectangle` now "is a" `Shape`; code that needs a shape can use a `Rectangle`.

### Task 1.3 -- Implement on a Circle

Add a second type:

```rust
struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }

    fn perimeter(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }
}

fn main() {
    let r = Rectangle { width: 3.0, height: 4.0 };
    let c = Circle { radius: 5.0 };

    println!("rectangle area: {}", r.area());
    println!("circle area: {:.2}", c.area());
    println!("circle perimeter: {:.2}", c.perimeter());
}
```

Run the program. Expected output:

```
rectangle area: 12
circle area: 78.54
circle perimeter: 31.42
```

Two different types now implement the same trait. The trait does not care how each computes area; it only requires that they can.

### Task 1.4 -- Implement on a Triangle

Add a third type:

```rust
struct Triangle {
    a: f64,
    b: f64,
    c: f64,
}

impl Shape for Triangle {
    fn area(&self) -> f64 {
        // Heron's formula:
        let s = (self.a + self.b + self.c) / 2.0;
        (s * (s - self.a) * (s - self.b) * (s - self.c)).sqrt()
    }

    fn perimeter(&self) -> f64 {
        self.a + self.b + self.c
    }
}

fn main() {
    let r = Rectangle { width: 3.0, height: 4.0 };
    let c = Circle { radius: 5.0 };
    let t = Triangle { a: 3.0, b: 4.0, c: 5.0 };

    println!("rectangle: area={}, perimeter={}", r.area(), r.perimeter());
    println!("circle: area={:.2}, perimeter={:.2}", c.area(), c.perimeter());
    println!("triangle: area={}, perimeter={}", t.area(), t.perimeter());
}
```

Run the program. Expected output:

```
rectangle: area=12, perimeter=14
circle: area=78.54, perimeter=31.42
triangle: area=6, perimeter=12
```

Three different types, one shared interface. The traits abstraction lets the rest of the program treat them uniformly when that is useful and treat them specifically when it is not.

### Task 1.5 -- A function that takes any Shape

Add a generic function that uses the trait:

```rust
fn print_shape_info(shape: &impl Shape) {
    println!("area: {:.2}, perimeter: {:.2}", shape.area(), shape.perimeter());
}

fn main() {
    let r = Rectangle { width: 3.0, height: 4.0 };
    let c = Circle { radius: 5.0 };
    let t = Triangle { a: 3.0, b: 4.0, c: 5.0 };

    print_shape_info(&r);
    print_shape_info(&c);
    print_shape_info(&t);
}
```

Run the program. Expected output:

```
area: 12.00, perimeter: 14.00
area: 78.54, perimeter: 31.42
area: 6.00, perimeter: 12.00
```

The `&impl Shape` parameter means "a reference to any type implementing Shape." One function, three uses. This is the simplest form of trait-based generic code.

The function does not know what shape it received. It only knows the shape has `area` and `perimeter` methods. That is exactly what the trait promises.

### Checkpoints

1. The trait `Shape` declares two methods that any implementor must provide. What would happen if `impl Shape for Triangle` provided `area` but not `perimeter`?
2. The function `print_shape_info` takes `&impl Shape`, a reference to any type implementing `Shape`. Could it take `impl Shape` (no reference)? What would change about how callers use it?
3. The trait does not specify how each type implements `area`. The rectangle uses `width * height`; the circle uses `pi * r^2`. Why is it valuable for the trait not to mandate an implementation strategy?

---

## Exercise 2 -- Default Methods

**Estimated time:** 10 minutes
**Topics covered:** providing default implementations, overriding them

### Context

A trait can provide default implementations for some or all of its methods. Implementors get the default for free; they can override it if needed. This is a powerful way to add behavior to many types from a single trait definition.

### Task 2.1 -- Add a default method

Add a default-implemented method to the trait:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;

    // Default implementation:
    fn describe(&self) -> String {
        format!("a shape with area {:.2} and perimeter {:.2}", self.area(), self.perimeter())
    }
}
```

The `describe` method has a body. Implementors do not need to provide one unless they want to override the default.

Use the default from `main`:

```rust
fn main() {
    let r = Rectangle { width: 3.0, height: 4.0 };
    let c = Circle { radius: 5.0 };

    println!("{}", r.describe());
    println!("{}", c.describe());
}
```

Run the program. Expected output:

```
a shape with area 12.00 and perimeter 14.00
a shape with area 78.54 and perimeter 31.42
```

Neither `Rectangle` nor `Circle` provides a `describe` method, yet both have one. The trait's default implementation applies to all implementors.

The default method calls `area` and `perimeter`, which each implementor provides. The default itself does not know which shape is involved; it relies on the required methods to do the work.

### Task 2.2 -- Override the default

Add a specialized version on `Circle`:

```rust
impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }

    fn perimeter(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }

    fn describe(&self) -> String {
        format!("a circle with radius {:.2}", self.radius)
    }
}
```

Run the program. Now `Circle` prints its own description while `Rectangle` still uses the default:

```
a shape with area 12.00 and perimeter 14.00
a circle with radius 5.00
```

The override applies only to `Circle`. Other implementors still get the default.

This is the "inherit, but customize when needed" pattern. Default methods provide a shared baseline; overrides handle specific cases that benefit from custom behavior.

### Task 2.3 -- A more complex default

Add another default method:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;

    fn describe(&self) -> String {
        format!("a shape with area {:.2} and perimeter {:.2}", self.area(), self.perimeter())
    }

    fn is_large(&self) -> bool {
        self.area() > 100.0
    }

    fn classify(&self) -> &'static str {
        if self.is_large() {
            "large"
        } else {
            "small"
        }
    }
}
```

The `classify` method is a default that calls `is_large`, which is itself a default. Implementors can override either or neither.

Test it:

```rust
fn main() {
    let r1 = Rectangle { width: 3.0, height: 4.0 };
    let r2 = Rectangle { width: 20.0, height: 20.0 };
    let c = Circle { radius: 5.0 };

    for shape in [&r1, &r2].iter() {
        println!("{}: area={:.2}, classification={}", shape.describe(), shape.area(), shape.classify());
    }
    println!("{}: area={:.2}, classification={}", c.describe(), c.area(), c.classify());
}
```

Run the program. Expected output:

```
a shape with area 12.00 and perimeter 14.00: area=12.00, classification=small
a shape with area 400.00 and perimeter 80.00: area=400.00, classification=large
a circle with radius 5.00: area=78.54, classification=small
```

Defaults can build on other defaults. This lets you add layers of behavior on top of a small set of required methods.

### Checkpoints

1. The default `describe` method calls `self.area()` and `self.perimeter()`. These are not overridden methods (they have no defaults), so each implementor must provide them. Why does the default work even though `Shape` does not know what each implementor will compute?
2. `Circle` overrode `describe` to print its radius rather than the default summary. What happens if `Rectangle` overrides `describe` but the override calls `self.area()`? Does it use the trait's `area` or the implementor's?
3. The chain `classify -> is_large -> area` showed defaults building on defaults. What is the practical benefit of this layering? Why not write a single `classify` default that contains all the logic?

---

## Exercise 3 -- Trait Bounds and Generic Functions

**Estimated time:** 15 minutes
**Topics covered:** `impl Trait`, named generics, `where` clauses, multiple bounds

### Context

A function that works for any type implementing a trait is a generic function with a trait bound. There are three syntactic forms for the bound; this exercise covers each and shows when to use them.

### Task 3.1 -- Three syntactic forms

Replace `main` with a function written three ways:

```rust
// Form 1: impl Trait syntax
fn print_info_v1(shape: &impl Shape) {
    println!("area: {:.2}", shape.area());
}

// Form 2: named generic parameter
fn print_info_v2<S: Shape>(shape: &S) {
    println!("area: {:.2}", shape.area());
}

// Form 3: where clause
fn print_info_v3<S>(shape: &S)
where
    S: Shape,
{
    println!("area: {:.2}", shape.area());
}

fn main() {
    let r = Rectangle { width: 3.0, height: 4.0 };
    print_info_v1(&r);
    print_info_v2(&r);
    print_info_v3(&r);
}
```

Run the program. All three print the same output:

```
area: 12.00
area: 12.00
area: 12.00
```

The three forms are equivalent for this case. The compiler treats them identically. The choice is stylistic:

- **Form 1 (`&impl Shape`)** is most concise for simple bounds. It reads as "a thing that is a Shape."
- **Form 2 (`<S: Shape>(s: &S)`)** introduces a name for the type. Use this when you need to refer to the type elsewhere in the signature (e.g., taking two parameters of the same generic type).
- **Form 3 (`where S: Shape`)** moves the bounds out of the parameter list. Use this when the bounds are complex or numerous; the function signature stays readable.

### Task 3.2 -- When the named form is necessary

Some signatures need a name because the same type appears in multiple positions:

```rust
fn total_area<S: Shape>(s1: &S, s2: &S) -> f64 {
    s1.area() + s2.area()
}

fn main() {
    let r1 = Rectangle { width: 3.0, height: 4.0 };
    let r2 = Rectangle { width: 5.0, height: 6.0 };

    let total = total_area(&r1, &r2);
    println!("total: {total}");
}
```

Run the program. Expected output:

```
total: 42
```

Both parameters have type `&S`, where `S` is the same type. The function requires both arguments to be the same kind of shape.

Try passing different types:

```rust
let c = Circle { radius: 5.0 };
let mixed = total_area(&r1, &c);        // ERROR: expected &Rectangle, found &Circle
```

The compiler rejects this because `S` is bound to one specific type per call. With `r1`, `S` becomes `Rectangle`; the second argument must also be `&Rectangle`.

If you want to accept two shapes of different types, declare two type parameters:

```rust
fn total_area<S1: Shape, S2: Shape>(s1: &S1, s2: &S2) -> f64 {
    s1.area() + s2.area()
}
```

Now `s1` and `s2` can be different types, each implementing `Shape`. Try this and verify it works with `&r1, &c`.

### Task 3.3 -- Multiple bounds

A type parameter can require multiple traits:

```rust
use std::fmt::Debug;

fn print_and_describe<S: Shape + Debug>(shape: &S) {
    println!("debug: {shape:?}");
    println!("area: {:.2}", shape.area());
}

#[derive(Debug)]
struct Square {
    side: f64,
}

impl Shape for Square {
    fn area(&self) -> f64 {
        self.side * self.side
    }

    fn perimeter(&self) -> f64 {
        4.0 * self.side
    }
}

fn main() {
    let s = Square { side: 5.0 };
    print_and_describe(&s);
}
```

Run the program. Expected output:

```
debug: Square { side: 5.0 }
area: 25.00
```

The bound `S: Shape + Debug` requires both. `Square` derives `Debug`, so it satisfies both.

Try passing a type that has Shape but not Debug:

```rust
let r = Rectangle { width: 3.0, height: 4.0 };
print_and_describe(&r);                  // ERROR if Rectangle does not derive Debug
```

Add `#[derive(Debug)]` to `Rectangle`, `Circle`, and `Triangle` so they all satisfy `Shape + Debug`. This pattern (require all the traits you need) is common.

### Task 3.4 -- The where clause for complex bounds

When you have many bounds or long types, `where` is more readable:

```rust
use std::fmt::{Debug, Display};

fn complex<S, T>(shape: &S, label: T) -> String
where
    S: Shape + Debug,
    T: Display + Clone,
{
    format!("{} (debug: {shape:?}): area = {:.2}", label, shape.area())
}

fn main() {
    let r = Rectangle { width: 3.0, height: 4.0 };
    let result = complex(&r, "rectangle");
    println!("{result}");
}
```

Run the program. Expected output:

```
rectangle (debug: Rectangle { width: 3.0, height: 4.0 }): area = 12.00
```

The `where` clause keeps the function signature `fn complex<S, T>(shape: &S, label: T) -> String` clean. The bounds are listed below. For functions with one or two simple bounds, the inline form is fine; for many bounds, `where` is much more readable.

### Task 3.5 -- A practical use of multiple bounds

A function that sorts and prints shapes:

```rust
fn describe_largest<S: Shape + Debug>(shapes: &[S]) {
    if shapes.is_empty() {
        println!("(no shapes)");
        return;
    }

    let mut sorted: Vec<&S> = shapes.iter().collect();
    sorted.sort_by(|a, b| a.area().partial_cmp(&b.area()).unwrap());

    let largest = sorted.last().unwrap();
    println!("largest: {largest:?} with area {:.2}", largest.area());
}

fn main() {
    let rects = vec![
        Rectangle { width: 1.0, height: 1.0 },
        Rectangle { width: 3.0, height: 4.0 },
        Rectangle { width: 2.0, height: 8.0 },
    ];

    describe_largest(&rects);
}
```

Run the program. Expected output:

```
largest: Rectangle { width: 2.0, height: 8.0 } with area 16.00
```

The function uses both `Shape` (to call `area`) and `Debug` (to print). The bounds are minimal; only what the function actually needs.

### Checkpoints

1. The three forms (`impl Trait`, generics, `where`) are equivalent for simple cases. Identify a specific situation where one form is the only sensible choice.
2. In Task 3.2, `total_area<S: Shape>(s1: &S, s2: &S)` forced both parameters to be the same type. Why might a function want this constraint? When is the two-type-parameter version better?
3. The function `describe_largest` required both `Shape` and `Debug`. Why is each bound necessary? What error would the compiler produce if you removed each one?

---

## Exercise 4 -- Returning Trait Types

**Estimated time:** 10-15 minutes
**Topics covered:** `impl Trait` in return position, `Box<dyn Trait>` for dynamic dispatch

### Context

Sometimes you want to return a value of a trait type without exposing the concrete type. Rust has two ways: `impl Trait` (static, one type per function) and `Box<dyn Trait>` (dynamic, can be any type).

### Task 4.1 -- impl Trait in return position

A function that returns an iterator-like type:

```rust
fn make_unit_square() -> impl Shape {
    Rectangle { width: 1.0, height: 1.0 }
}

fn main() {
    let s = make_unit_square();
    println!("unit square area: {}", s.area());
}
```

Run the program. Expected output:

```
unit square area: 1
```

The function returns "some type that implements Shape." The caller does not need to know the concrete type. The compiler does; it knows the return type is exactly `Rectangle`. This is static dispatch.

The function compiles to specialized code returning a `Rectangle`. There is no allocation, no runtime cost. The `impl Shape` is just a way to hide the concrete type from the caller.

### Task 4.2 -- The single-type restriction

`impl Trait` in return position requires all return paths to produce the same concrete type:

```rust
fn shape_or_other(use_circle: bool) -> impl Shape {
    if use_circle {
        Circle { radius: 5.0 }                    // ERROR: not the same type
    } else {
        Rectangle { width: 3.0, height: 4.0 }
    }
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0308]: `if` and `else` have incompatible types
```

Even though both `Circle` and `Rectangle` implement `Shape`, the compiler insists on one concrete type. `impl Trait` is type-checked at the function boundary, not at the call site.

When you genuinely need to return different concrete types depending on a runtime condition, you need `Box<dyn Trait>` (next task).

### Task 4.3 -- Box<dyn Trait> for dynamic dispatch

```rust
fn make_shape(kind: &str) -> Box<dyn Shape> {
    match kind {
        "circle" => Box::new(Circle { radius: 5.0 }),
        "rectangle" => Box::new(Rectangle { width: 3.0, height: 4.0 }),
        "triangle" => Box::new(Triangle { a: 3.0, b: 4.0, c: 5.0 }),
        _ => Box::new(Rectangle { width: 1.0, height: 1.0 }),
    }
}

fn main() {
    for kind in &["circle", "rectangle", "triangle"] {
        let shape = make_shape(kind);
        println!("{}: area={:.2}", kind, shape.area());
    }
}
```

Run the program. Expected output:

```
circle: area=78.54
rectangle: area=12.00
triangle: area=6.00
```

`Box<dyn Shape>` is a heap-allocated trait object. Three things happen:

- The concrete type is allocated on the heap.
- A pointer to it goes into the `Box`.
- A vtable (a table of function pointers) tracks which implementation of `area`, `perimeter`, etc., applies to this value.

When you call `shape.area()`, the compiler dispatches through the vtable. This is dynamic dispatch: the function called is determined at runtime, not compile time.

The trade-offs:

- **Allocation.** Each `Box<dyn Shape>` requires a heap allocation.
- **Indirection.** Method calls go through a pointer (the vtable).
- **Flexibility.** A function can return different concrete types, or a `Vec<Box<dyn Shape>>` can hold a mix of shape types.

### Task 4.4 -- A vector of trait objects

The pattern of heterogeneous collections:

```rust
fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Rectangle { width: 3.0, height: 4.0 }),
        Box::new(Circle { radius: 5.0 }),
        Box::new(Triangle { a: 3.0, b: 4.0, c: 5.0 }),
    ];

    let mut total = 0.0;
    for shape in &shapes {
        total += shape.area();
    }
    println!("total area: {:.2}", total);
}
```

Run the program. Expected output:

```
total area: 96.54
```

A `Vec<Box<dyn Shape>>` can hold any mix of shape types. This is the standard pattern for "collection of things that share an interface but have different concrete types."

A `Vec<Rectangle>` would only hold rectangles. A `Vec<Box<dyn Shape>>` holds rectangles, circles, triangles, and any future shape that implements the trait.

### Task 4.5 -- When to choose which

A summary:

- **`impl Shape`** when the function returns exactly one type. Zero cost. Most common case.
- **`Box<dyn Shape>`** when the function might return different types, or when storing a collection of mixed types.

The default should be `impl Trait`. Reach for `Box<dyn Trait>` when you specifically need the flexibility.

A third option, references to trait objects (`&dyn Shape`), exists but is less common. It avoids the allocation of `Box` but requires the caller to manage the lifetime of the underlying data. For most cases, `impl Trait` or `Box<dyn Trait>` covers the need.

### Checkpoints

1. The compiler rejected `shape_or_other` in Task 4.2 because the two branches return different types. From the caller's perspective, both types implement `Shape`. Why does the compiler care about the concrete type?
2. `Box<dyn Shape>` requires a heap allocation. What does the program lose by needing this allocation? When does the cost matter, and when is it negligible?
3. The vector in Task 4.4 holds `Box<dyn Shape>` to hold mixed concrete types. Could you do the same with an enum (a `Shape` enum with variants for `Rectangle`, `Circle`, `Triangle`)? What trade-offs would each approach involve?

---

## Exercise 5 -- Generic Structs and Enums

**Estimated time:** 10-15 minutes
**Topics covered:** generic types, methods on generic structs, multiple type parameters

### Context

So far, generics have been applied to functions. They also apply to structs, enums, and method implementations. This exercise builds generic data structures and methods.

### Task 5.1 -- A generic Pair struct

Replace `main` with:

```rust
#[derive(Debug)]
struct Pair<T> {
    first: T,
    second: T,
}

fn main() {
    let int_pair = Pair { first: 1, second: 2 };
    let float_pair = Pair { first: 1.5, second: 2.5 };
    let string_pair = Pair { first: String::from("a"), second: String::from("b") };

    println!("int_pair: {int_pair:?}");
    println!("float_pair: {float_pair:?}");
    println!("string_pair: {string_pair:?}");
}
```

Run the program. Expected output:

```
int_pair: Pair { first: 1, second: 2 }
float_pair: Pair { first: 1.5, second: 2.5 }
string_pair: Pair { first: "a", second: "b" }
```

The type parameter `T` is filled in at use. Each different `T` produces a different concrete type. The compiler generates separate code for each (called monomorphization).

`Pair<i32>` and `Pair<f64>` are different types; you cannot pass one where the other is expected.

### Task 5.2 -- Methods on a generic struct

Add methods:

```rust
impl<T> Pair<T> {
    fn new(first: T, second: T) -> Pair<T> {
        Pair { first, second }
    }

    fn first(&self) -> &T {
        &self.first
    }
}

fn main() {
    let p = Pair::new(10, 20);
    println!("first: {}", p.first());
}
```

Run the program. Expected output:

```
first: 10
```

The `impl<T> Pair<T>` block introduces the type parameter `T` again. The methods can use `T` anywhere they would normally use a concrete type. The compiler generates a separate method implementation for each concrete `Pair<T>` that gets used.

### Task 5.3 -- Methods that require trait bounds

Some methods only make sense for certain `T`. For example, comparing the two elements of a `Pair` requires `T: PartialOrd`:

```rust
impl<T: PartialOrd> Pair<T> {
    fn largest(&self) -> &T {
        if self.first > self.second {
            &self.first
        } else {
            &self.second
        }
    }
}

fn main() {
    let int_pair = Pair::new(10, 20);
    let string_pair = Pair::new(String::from("apple"), String::from("banana"));

    println!("largest int: {}", int_pair.largest());
    println!("largest string: {}", string_pair.largest());
}
```

Run the program. Expected output:

```
largest int: 20
largest string: banana
```

The `impl<T: PartialOrd>` block adds methods only to those `Pair<T>` where `T` is comparable. For `Pair<NonComparableType>`, the `largest` method would not exist.

This is called a conditional impl: methods that are available based on the type parameters' traits. Standard library types use this extensively. For example, `Vec<T>` has `sort` only when `T: Ord`.

### Task 5.4 -- Multiple type parameters

A struct can have several type parameters:

```rust
#[derive(Debug)]
struct KeyValue<K, V> {
    key: K,
    value: V,
}

impl<K, V> KeyValue<K, V> {
    fn new(key: K, value: V) -> KeyValue<K, V> {
        KeyValue { key, value }
    }
}

fn main() {
    let entry1 = KeyValue::new("name", String::from("Alice"));
    let entry2 = KeyValue::new(42, 3.14);

    println!("{entry1:?}");
    println!("{entry2:?}");
}
```

Run the program. Expected output:

```
KeyValue { key: "name", value: "Alice" }
KeyValue { key: 42, value: 3.14 }
```

`K` and `V` are independent. They can be any types. The pattern parallels `HashMap<K, V>` from the standard library.

### Task 5.5 -- A generic enum

Enums can also be generic. The standard library uses this for `Option<T>` and `Result<T, E>`. Define your own:

```rust
#[derive(Debug)]
enum Either<L, R> {
    Left(L),
    Right(R),
}

impl<L, R> Either<L, R> {
    fn is_left(&self) -> bool {
        matches!(self, Either::Left(_))
    }
}

fn main() {
    let l: Either<i32, String> = Either::Left(42);
    let r: Either<i32, String> = Either::Right(String::from("hello"));

    println!("l: {l:?}, is_left: {}", l.is_left());
    println!("r: {r:?}, is_left: {}", r.is_left());
}
```

Run the program. Expected output:

```
l: Left(42), is_left: true
r: Right("hello"), is_left: false
```

`Either<L, R>` represents "either an L or an R." It is more general than `Result` (which carries the connotation of success/failure); some functional programming libraries use it as a building block.

### Task 5.6 -- Generic functions on generic data

Combining generic functions with generic structs:

```rust
fn max_of_pair<T: PartialOrd + Copy>(p: &Pair<T>) -> T {
    if p.first > p.second {
        p.first
    } else {
        p.second
    }
}

fn main() {
    let int_pair = Pair::new(10, 20);
    let float_pair = Pair::new(1.5, 2.5);

    println!("max int: {}", max_of_pair(&int_pair));
    println!("max float: {}", max_of_pair(&float_pair));
}
```

Run the program. Expected output:

```
max int: 20
max float: 2.5
```

The function works on `Pair<T>` for any `T` that is both `PartialOrd` and `Copy`. The `Copy` bound is needed because the function returns `T` by value; without `Copy`, you would need to return `&T` instead.

This composition (generic function on generic struct) is the basis of most code in the Rust standard library. `Vec::iter().map(...)` is exactly this: a generic method on a generic struct returning a generic value.

### Checkpoints

1. The compiler generates separate code for each concrete instantiation (`Pair<i32>`, `Pair<f64>`, `Pair<String>`). What is the implication for binary size in a project that uses `Pair` with 10 different types?
2. In Task 5.3, the `largest` method was on `impl<T: PartialOrd> Pair<T>` rather than `impl<T> Pair<T>`. Why is this the right place for the bound rather than on `Pair` itself?
3. The `Either<L, R>` enum in Task 5.5 is more general than `Result<T, E>`. Why does the standard library use `Result<T, E>` (which is just a renamed `Either`) for error handling rather than the general `Either`?

---

## Exercise 6 -- Common Standard Library Traits

**Estimated time:** 15 minutes
**Topics covered:** `Display`, `PartialEq`, `From` and `Into`, `Default`, `Iterator`

### Context

The standard library defines a vocabulary of traits that nearly every type uses. Knowing this vocabulary is essential for reading and writing Rust code. This exercise implements several of these traits for a custom type.

### Task 6.1 -- Implementing Display

Replace `main` with:

```rust
use std::fmt;

#[derive(Debug)]
struct Color {
    red: u8,
    green: u8,
    blue: u8,
}

impl fmt::Display for Color {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "#{:02X}{:02X}{:02X}", self.red, self.green, self.blue)
    }
}

fn main() {
    let c = Color { red: 255, green: 100, blue: 50 };

    println!("debug: {c:?}");
    println!("display: {c}");
}
```

Run the program. Expected output:

```
debug: Color { red: 255, green: 100, blue: 50 }
display: #FF6432
```

The `Display` trait controls the `{}` format placeholder. `Debug` controls `{:?}`. The two are different: `Debug` is for developers (showing all fields); `Display` is for users (a clean, readable form).

Unlike `Debug`, `Display` cannot be derived because there is no obvious canonical format. Each type decides what its user-facing representation is.

### Task 6.2 -- Implementing PartialEq

Add equality:

```rust
impl PartialEq for Color {
    fn eq(&self, other: &Self) -> bool {
        self.red == other.red && self.green == other.green && self.blue == other.blue
    }
}

fn main() {
    let c1 = Color { red: 255, green: 100, blue: 50 };
    let c2 = Color { red: 255, green: 100, blue: 50 };
    let c3 = Color { red: 0, green: 0, blue: 0 };

    println!("c1 == c2: {}", c1 == c2);
    println!("c1 == c3: {}", c1 == c3);
}
```

Run the program. Expected output:

```
c1 == c2: true
c1 == c3: false
```

The `PartialEq` trait enables the `==` and `!=` operators. The required method `eq` returns whether two values are equal.

For most types, `PartialEq` can be derived: `#[derive(PartialEq)]`. The derive compares all fields. The manual implementation here is for illustration; in real code, you would usually just derive.

### Task 6.3 -- Implementing From and Into

`From` provides conversions:

```rust
impl From<(u8, u8, u8)> for Color {
    fn from(tuple: (u8, u8, u8)) -> Self {
        Color { red: tuple.0, green: tuple.1, blue: tuple.2 }
    }
}

fn main() {
    let c1 = Color::from((255, 128, 0));
    let c2: Color = (100, 200, 50).into();

    println!("c1: {c1}");
    println!("c2: {c2}");
}
```

Run the program. Expected output:

```
c1: #FF8000
c2: #64C832
```

The pattern: implement `From<T> for U`, and the standard library automatically provides `Into<U> for T`. Now you can construct a `Color` from a tuple in two ways: `Color::from((r, g, b))` or `(r, g, b).into()`.

The `.into()` form is sometimes more readable, especially when the target type is clear from context (annotated `let`, function parameter, etc.).

`From` and `Into` are also what makes the `?` operator work across error types: a `Result<T, E>` is propagated through `?` to a function returning `Result<T, F>` whenever `F: From<E>` exists.

### Task 6.4 -- Implementing Default

```rust
impl Default for Color {
    fn default() -> Self {
        Color { red: 0, green: 0, blue: 0 }
    }
}

fn main() {
    let black: Color = Color::default();
    let also_black: Color = Default::default();

    println!("black: {black}");
    println!("also_black: {also_black}");
}
```

Run the program. Expected output:

```
black: #000000
also_black: #000000
```

`Default::default()` produces a default value. The trait can be derived for structs whose fields all have defaults; the derive uses each field's default.

`Default` integrates with other parts of the language: `Vec<T>` has `T::default()` available in some methods, `HashMap<K, V>::or_default()` uses it, struct update syntax can use it with `..Default::default()`.

For numeric types, `default()` is 0. For booleans, `false`. For collections, empty. For options, `None`. The convention is "the most natural empty/zero/initial value."

### Task 6.5 -- Implementing Iterator

The Iterator trait is the foundation of all iteration. Implement it for a counter:

```rust
struct Counter {
    current: u32,
    max: u32,
}

impl Counter {
    fn new(max: u32) -> Counter {
        Counter { current: 0, max }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.current < self.max {
            self.current += 1;
            Some(self.current)
        } else {
            None
        }
    }
}

fn main() {
    let mut counter = Counter::new(5);

    for n in counter {
        print!("{n} ");
    }
    println!();

    // Use iterator methods:
    let sum: u32 = Counter::new(10).sum();
    let squares: Vec<u32> = Counter::new(5).map(|n| n * n).collect();
    let filtered: Vec<u32> = Counter::new(10).filter(|n| n % 2 == 0).collect();

    println!("sum 1..=10: {sum}");
    println!("squares 1..=5: {squares:?}");
    println!("even 1..=10: {filtered:?}");
}
```

Run the program. Expected output:

```
1 2 3 4 5 
sum 1..=10: 55
squares 1..=5: [1, 4, 9, 16, 25]
even 1..=10: [2, 4, 6, 8, 10]
```

Two things to notice:

- The `type Item = u32;` is an associated type. The trait `Iterator` has one method (`next`) and one associated type (`Item`). The implementation provides both.
- Once you implement `next`, all the other iterator methods (`map`, `filter`, `sum`, etc.) work automatically. They are default methods on the trait. You implement one method; the rest come for free.

This is one of the most powerful patterns in the standard library: a single small required method unlocks dozens of useful default methods. The `Iterator` trait alone has over 70 methods, all built on top of `next`.

### Task 6.6 -- Combining the traits

A complete type with the standard set:

```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

impl Default for Point {
    fn default() -> Self {
        Point { x: 0, y: 0 }
    }
}

impl From<(i32, i32)> for Point {
    fn from(t: (i32, i32)) -> Self {
        Point { x: t.0, y: t.1 }
    }
}

fn main() {
    let origin = Point::default();
    let p1 = Point::from((3, 4));
    let p2: Point = (5, 6).into();

    println!("origin: {origin}");
    println!("p1: {p1}");
    println!("p2: {p2}");
    println!("p1 == p2: {}", p1 == p2);

    let mut points = std::collections::HashMap::new();
    points.insert(p1.clone(), "first");
    points.insert(p2.clone(), "second");
    println!("p1's label: {:?}", points.get(&p1));
}
```

Run the program. Expected output:

```
origin: (0, 0)
p1: (3, 4)
p2: (5, 6)
p1 == p2: false
p1's label: Some("first")
```

The combination (`Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `Display`, `Default`, `From`) is typical for any value-like type in Rust. The first five are derived; `Display` and `From` are usually implemented manually; `Default` can be derived or implemented depending on the desired default.

### Checkpoints

1. The `Display` trait cannot be derived because there is no canonical format. `Debug` can be derived because it has one. What does this difference tell you about the design intent of each trait?
2. Once you implement `From<T> for U`, the standard library provides `Into<U> for T` automatically. What is the practical benefit of the asymmetric implementation? Why not require both?
3. The `Iterator` trait has one required method (`next`) and many default methods. What is the design principle behind this pattern? When would you choose to implement a trait this way?

---

## Exercise 7 -- Operator Overloading and the Newtype Pattern

**Estimated time:** 10-15 minutes
**Topics covered:** `std::ops::Add` and related traits, the newtype pattern

### Context

Operators in Rust are syntactic sugar for trait methods. `a + b` is `a.add(b)` from `std::ops::Add`. By implementing these traits, you can use operators on your own types. The newtype pattern leverages this for type safety.

### Task 7.1 -- Implementing Add

Replace `main` with:

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Vector2D {
    x: f64,
    y: f64,
}

impl Add for Vector2D {
    type Output = Vector2D;

    fn add(self, other: Vector2D) -> Vector2D {
        Vector2D {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    let v1 = Vector2D { x: 1.0, y: 2.0 };
    let v2 = Vector2D { x: 3.0, y: 4.0 };
    let sum = v1 + v2;

    println!("v1 + v2 = ({}, {})", sum.x, sum.y);
}
```

Run the program. Expected output:

```
v1 + v2 = (4, 6)
```

The `+` operator on `Vector2D` calls the `add` method from the `Add` trait. The associated type `Output = Vector2D` says what type the addition produces (usually the same as the operands, but you can return a different type for asymmetric operations).

### Task 7.2 -- Implementing multiple operators

```rust
use std::ops::{Add, Sub, Mul};

#[derive(Debug, Clone, Copy)]
struct Vector2D {
    x: f64,
    y: f64,
}

impl Add for Vector2D {
    type Output = Vector2D;
    fn add(self, other: Vector2D) -> Vector2D {
        Vector2D { x: self.x + other.x, y: self.y + other.y }
    }
}

impl Sub for Vector2D {
    type Output = Vector2D;
    fn sub(self, other: Vector2D) -> Vector2D {
        Vector2D { x: self.x - other.x, y: self.y - other.y }
    }
}

impl Mul<f64> for Vector2D {
    type Output = Vector2D;
    fn mul(self, scalar: f64) -> Vector2D {
        Vector2D { x: self.x * scalar, y: self.y * scalar }
    }
}

fn main() {
    let v1 = Vector2D { x: 1.0, y: 2.0 };
    let v2 = Vector2D { x: 3.0, y: 4.0 };

    let sum = v1 + v2;
    let diff = v1 - v2;
    let scaled = v1 * 3.0;

    println!("sum: {sum:?}");
    println!("diff: {diff:?}");
    println!("scaled: {scaled:?}");
}
```

Run the program. Expected output:

```
sum: Vector2D { x: 4.0, y: 6.0 }
diff: Vector2D { x: -2.0, y: -2.0 }
scaled: Vector2D { x: 3.0, y: 6.0 }
```

The `Mul<f64>` is an asymmetric operator: it multiplies a `Vector2D` by an `f64` (producing a `Vector2D`). The trait `Mul<T>` is generic over the right-hand operand's type, allowing each pair of types to have its own multiplication semantics.

Note: `v1 * 3.0` works, but `3.0 * v1` does not (out of the box). To support both orders, you would implement `Mul<Vector2D> for f64` as well. The standard library does this for built-in numeric types.

### Task 7.3 -- The newtype pattern with operators

The newtype pattern (introduced in Lab 6) becomes more powerful with operator overloading:

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy, PartialEq)]
struct Meters(f64);

#[derive(Debug, Clone, Copy, PartialEq)]
struct Feet(f64);

impl Add for Meters {
    type Output = Meters;
    fn add(self, other: Meters) -> Meters {
        Meters(self.0 + other.0)
    }
}

impl Meters {
    fn to_feet(self) -> Feet {
        Feet(self.0 * 3.28084)
    }
}

fn main() {
    let distance1 = Meters(100.0);
    let distance2 = Meters(50.0);

    let total: Meters = distance1 + distance2;
    println!("total: {total:?}");
    println!("in feet: {:?}", total.to_feet());

    // This would NOT compile, preventing unit confusion:
    // let confused = Meters(100.0) + Feet(50.0);
}
```

Run the program. Expected output:

```
total: Meters(150.0)
in feet: Feet(492.126)
```

The newtype `Meters` is a distinct type from `f64`. You can add two `Meters` together, but you cannot accidentally add `Meters` to `Feet` (the type system rejects it). The conversion `to_feet` is explicit.

This pattern eliminates the entire class of "unit confusion" bugs. The Mars Climate Orbiter famously crashed because of metric/imperial unit confusion; the newtype pattern makes such errors compile-time impossible.

### Task 7.4 -- A wrapper that adds behavior

Another use of the newtype pattern: adding methods to a type you do not own:

```rust
struct Wrapper(Vec<String>);

impl Wrapper {
    fn join_with(&self, sep: &str) -> String {
        self.0.join(sep)
    }
}

fn main() {
    let w = Wrapper(vec![
        String::from("hello"),
        String::from("world"),
        String::from("rust"),
    ]);

    let joined = w.join_with(" | ");
    println!("{joined}");
}
```

Run the program. Expected output:

```
hello | world | rust
```

The `Wrapper` type holds a `Vec<String>` but adds its own methods. You cannot add methods directly to `Vec<String>` (you do not own the type), but you can wrap it.

The trade-off: the wrapper hides the underlying type. To access the inner data, you go through the wrapper's methods. For some cases, this encapsulation is exactly what you want; for others, you would want to delegate methods to the inner type (which involves writing wrapper methods for everything you want to expose).

### Task 7.5 -- Combining newtype with traits

A complete example: a strong-typed identifier:

```rust
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct UserId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct OrderId(u64);

impl fmt::Display for UserId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "U{:06}", self.0)
    }
}

impl fmt::Display for OrderId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "O{:06}", self.0)
    }
}

fn process_user(id: UserId) {
    println!("processing user {id}");
}

fn process_order(id: OrderId) {
    println!("processing order {id}");
}

fn main() {
    let user = UserId(42);
    let order = OrderId(1000);

    process_user(user);
    process_order(order);

    // This would NOT compile:
    // process_user(order);             // ERROR: expected UserId, got OrderId

    // Use as HashMap keys:
    let mut users = std::collections::HashMap::new();
    users.insert(user, "Alice");
    println!("user 42 is {:?}", users.get(&user));
}
```

Run the program. Expected output:

```
processing user U000042
processing order O001000
user 42 is Some("Alice")
```

`UserId` and `OrderId` are distinct types despite both wrapping `u64`. The compiler refuses to silently convert; the type system prevents the entire class of "passed an order ID where a user ID was expected" bugs.

The standard derive set (`Debug, Clone, Copy, PartialEq, Eq, Hash`) plus a `Display` impl gives you a value type that:

- Can be printed in user-facing output.
- Can be cloned and compared.
- Can be used as a HashMap key.
- Cannot be confused with other newtypes.

This is the canonical pattern for typed IDs in Rust applications.

### Checkpoints

1. Operator overloading in Rust uses traits like `Add`. Why is this design preferable to having `+` be a built-in language feature that only works on numbers?
2. The newtype pattern in Task 7.3 created `Meters` as distinct from `Feet`. Both wrap `f64`. What does this give you beyond the runtime guarantees of just using `f64` with naming conventions?
3. The wrapper in Task 7.4 hid the underlying `Vec<String>`. Users have to go through wrapper methods. When is this encapsulation a benefit? When is it a problem?

---

## Exercise 8 -- Putting It Together

**Estimated time:** 15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from Module 13 into a single program: a trait hierarchy with default methods, generic functions with various bounds, both static and dynamic dispatch, generic data structures, several standard-library traits, and operator overloading on a newtype. The program is a small shape-rendering pipeline that demonstrates how the pieces fit together in realistic code.

### Task 8.1 -- The full program

Replace your `src/main.rs` with this complete program:

```rust
use std::fmt;
use std::ops::Add;

// ============================================================
// The Shape trait hierarchy
// ============================================================

trait Shape: fmt::Debug {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;

    fn density(&self) -> f64 {
        if self.perimeter() == 0.0 {
            0.0
        } else {
            self.area() / self.perimeter()
        }
    }

    fn summary(&self) -> String {
        format!("{:?}: area={:.2}, perimeter={:.2}, density={:.3}",
            self, self.area(), self.perimeter(), self.density())
    }
}

// ============================================================
// Concrete shapes
// ============================================================

#[derive(Debug, Clone, Copy)]
struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }
}

#[derive(Debug, Clone, Copy)]
struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }

    fn perimeter(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }

    fn summary(&self) -> String {
        format!("Circle(r={:.2}): area={:.2}, perimeter={:.2}",
            self.radius, self.area(), self.perimeter())
    }
}

#[derive(Debug, Clone, Copy)]
struct Triangle {
    a: f64,
    b: f64,
    c: f64,
}

impl Shape for Triangle {
    fn area(&self) -> f64 {
        let s = (self.a + self.b + self.c) / 2.0;
        (s * (s - self.a) * (s - self.b) * (s - self.c)).sqrt()
    }

    fn perimeter(&self) -> f64 {
        self.a + self.b + self.c
    }
}

// ============================================================
// A typed scale factor (newtype with operator overloading)
// ============================================================

#[derive(Debug, Clone, Copy, PartialEq)]
struct Scale(f64);

impl Add for Scale {
    type Output = Scale;
    fn add(self, other: Scale) -> Scale {
        Scale(self.0 + other.0)
    }
}

impl fmt::Display for Scale {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}x", self.0)
    }
}

// ============================================================
// A generic container
// ============================================================

#[derive(Debug)]
struct Group<S: Shape> {
    name: String,
    shapes: Vec<S>,
}

impl<S: Shape> Group<S> {
    fn new(name: &str) -> Group<S> {
        Group {
            name: name.to_string(),
            shapes: Vec::new(),
        }
    }

    fn add(&mut self, shape: S) {
        self.shapes.push(shape);
    }

    fn total_area(&self) -> f64 {
        self.shapes.iter().map(|s| s.area()).sum()
    }

    fn largest(&self) -> Option<&S> {
        self.shapes.iter().max_by(|a, b| a.area().partial_cmp(&b.area()).unwrap())
    }
}

// ============================================================
// Generic function with bounds
// ============================================================

fn describe_shapes<S: Shape>(shapes: &[S]) {
    println!("describe_shapes (static dispatch):");
    for shape in shapes {
        println!("  {}", shape.summary());
    }
}

// ============================================================
// Function using dynamic dispatch
// ============================================================

fn describe_mixed(shapes: &[Box<dyn Shape>]) {
    println!("describe_mixed (dynamic dispatch):");
    for shape in shapes {
        println!("  {}", shape.summary());
    }
}

// ============================================================
// Generic function returning impl Trait
// ============================================================

fn make_square(side: f64) -> impl Shape {
    Rectangle { width: side, height: side }
}

// ============================================================
// Function with multiple trait bounds
// ============================================================

fn largest_area_index<S>(shapes: &[S]) -> Option<usize>
where
    S: Shape + fmt::Debug,
{
    if shapes.is_empty() {
        return None;
    }

    let (max_index, _) = shapes
        .iter()
        .enumerate()
        .max_by(|(_, a), (_, b)| a.area().partial_cmp(&b.area()).unwrap())?;

    Some(max_index)
}

// ============================================================
// Main: tie everything together
// ============================================================

fn main() {
    // Static dispatch through generics:
    let rectangles = vec![
        Rectangle { width: 3.0, height: 4.0 },
        Rectangle { width: 5.0, height: 6.0 },
        Rectangle { width: 2.0, height: 8.0 },
    ];
    describe_shapes(&rectangles);
    println!();

    // Generic container:
    let mut rect_group: Group<Rectangle> = Group::new("rectangles");
    for r in &rectangles {
        rect_group.add(*r);
    }
    println!("group '{}': total area = {:.2}", rect_group.name, rect_group.total_area());
    if let Some(largest) = rect_group.largest() {
        println!("largest: {:?}", largest);
    }
    println!();

    // Dynamic dispatch (mixed types):
    let mixed: Vec<Box<dyn Shape>> = vec![
        Box::new(Rectangle { width: 3.0, height: 4.0 }),
        Box::new(Circle { radius: 5.0 }),
        Box::new(Triangle { a: 3.0, b: 4.0, c: 5.0 }),
    ];
    describe_mixed(&mixed);
    println!();

    // impl Trait return:
    let square = make_square(5.0);
    println!("square via impl Trait: {}", square.summary());
    println!();

    // Generic function with multiple bounds:
    let triangles = vec![
        Triangle { a: 3.0, b: 4.0, c: 5.0 },
        Triangle { a: 5.0, b: 12.0, c: 13.0 },
        Triangle { a: 6.0, b: 8.0, c: 10.0 },
    ];
    if let Some(idx) = largest_area_index(&triangles) {
        println!("largest triangle is index {idx}: {:?}", triangles[idx]);
    }
    println!();

    // Newtype with operator overloading:
    let s1 = Scale(2.0);
    let s2 = Scale(3.0);
    let combined = s1 + s2;
    println!("scales: {s1} + {s2} = {combined}");
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
describe_shapes (static dispatch):
  Rectangle { width: 3.0, height: 4.0 }: area=12.00, perimeter=14.00, density=0.857
  Rectangle { width: 5.0, height: 6.0 }: area=30.00, perimeter=22.00, density=1.364
  Rectangle { width: 2.0, height: 8.0 }: area=16.00, perimeter=20.00, density=0.800

group 'rectangles': total area = 58.00
largest: Rectangle { width: 5.0, height: 6.0 }

describe_mixed (dynamic dispatch):
  Rectangle { width: 3.0, height: 4.0 }: area=12.00, perimeter=14.00, density=0.857
  Circle(r=5.00): area=78.54, perimeter=31.42
  Triangle { a: 3.0, b: 4.0, c: 5.0 }: area=6.00, perimeter=12.00, density=0.500

square via impl Trait: Rectangle { width: 5.0, height: 5.0 }: area=25.00, perimeter=20.00, density=1.250

largest triangle is index 1: Triangle { a: 5.0, b: 12.0, c: 13.0 }

scales: 2x + 3x = 5x
```

### Task 8.2 -- Identify the patterns

In `lab9-notes.md`, identify each instance of the following patterns in the program above:

1. A trait definition with both required and default methods.
2. A trait with a supertrait bound (`Shape: fmt::Debug`).
3. A type that overrides a default method (`Circle::summary`).
4. A generic function using `impl Trait` syntax for parameters.
5. A generic function using named generics syntax.
6. A function using a `where` clause for its bounds.
7. A function returning `impl Trait`.
8. A function taking `&[Box<dyn Trait>]` for dynamic dispatch.
9. A generic struct with a type parameter and trait bound.
10. A method that uses the type parameter's bound to do something specific.
11. An implementation of `Add` on a newtype.
12. An implementation of `Display` on a newtype.

### Task 8.3 -- Predict and verify

For each of the following code changes, predict whether the program will compile, and run it to verify. Write your predictions in `lab9-notes.md` BEFORE running.

**Change A:** Remove the `Shape: fmt::Debug` supertrait bound from the trait definition. What changes? What error appears, and where?

**Change B:** Change `make_square` to return `impl Shape` but have its body conditionally return either a `Rectangle` or a `Circle` depending on a parameter. What error do you expect?

**Change C:** Change `describe_shapes<S: Shape>(shapes: &[S])` to `describe_shapes(shapes: &[Box<dyn Shape>])`. What changes at the call site? What is the performance implication?

For each change, write your prediction first, then run and verify.

### Checkpoints

1. The trait `Shape` has a supertrait bound `Shape: fmt::Debug`. What does this give you in the trait's default methods? What does it require of implementors?
2. The function `describe_shapes` uses static dispatch (generic over `S: Shape`), while `describe_mixed` uses dynamic dispatch (`&[Box<dyn Shape>]`). Both work; what tells you when to choose each?
3. The newtype `Scale` implements `Add` and `Display`. The bare `f64` already has both. What does wrapping it as `Scale` actually accomplish?

---

## Summary and Reflection

You have now used every concept from Module 13 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Defining Traits | trait declaration, multiple implementations | Traits are contracts; types provide the implementations. Multiple types can implement one trait. |
| 2 -- Default Methods | trait with default, override | Defaults let traits provide shared behavior. Implementors override only what they need. |
| 3 -- Trait Bounds | impl Trait, generics, where | Three syntactic forms; choose based on complexity and need to name the type. |
| 4 -- Returning Traits | impl Trait, Box<dyn Trait> | Static dispatch is zero-cost; dynamic dispatch is flexible. Default to static unless you need the flexibility. |
| 5 -- Generic Data | generic structs and enums, conditional impl | Type parameters can be filled with any type; bounds enable methods only when appropriate. |
| 6 -- Standard Traits | Display, PartialEq, From, Default, Iterator | The standard vocabulary of traits is what makes types feel "Rust-like" in the ecosystem. |
| 7 -- Operators and Newtype | std::ops, the newtype pattern | Operators are traits; the newtype pattern adds type safety without runtime cost. |
| 8 -- Integration | all of the above | Real Rust uses traits and generics together pervasively. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab9-notes.md` before your next session.

1. Of the concepts in Module 13, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. Rust's traits replace what would be inheritance hierarchies in many languages. Pick a specific design pattern from inheritance-based OO (e.g., template method, strategy, decorator) and describe how you would express it in Rust using traits and generics. What does Rust gain by not having inheritance, and what does it lose?

3. The lab covered both static dispatch (`impl Trait`, generics) and dynamic dispatch (`Box<dyn Trait>`). Identify a specific situation from this lab where you initially chose one approach and then changed your mind after considering the alternative. What changed your decision?

---

*End of Lab 9*
