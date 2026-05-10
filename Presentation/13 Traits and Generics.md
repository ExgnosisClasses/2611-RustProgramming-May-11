# Module 13: Traits and Generics

## Module Overview

Traits are how Rust expresses shared behavior across types. Generics are how Rust expresses code that works for many types. The two are inseparable: most generic code uses traits as bounds, and traits are most useful when applied to generic code.

You have been using traits and generics throughout the course, mostly without naming them. `Vec<T>` is generic. `Option<T>` is generic. `Iterator` is a trait. `Display`, `Debug`, `Clone`, `PartialEq` are all traits. The `derive` attribute generates trait implementations. The `?` operator works through the `From` trait. This module makes all of this explicit.

Three ideas are central:

1. **Traits define behavior; types implement it.** A trait is a contract: "anything implementing this trait can do these things." Multiple types can implement the same trait, and the same type can implement many traits. This replaces inheritance for code reuse.
2. **Generics give one definition for many types.** Instead of writing one function per type, you write one generic function that works for any type satisfying certain trait bounds. The compiler generates specialized code for each type that calls it.
3. **Together, traits and generics enable zero-cost abstractions.** A generic function with trait bounds compiles to specialized code for each concrete type. There is no runtime dispatch and no boxing. The abstraction has the same cost as hand-written specialized code.

By the end of this module, students will:

- Define traits and implement them on types.
- Use default implementations to provide behavior that implementations can override.
- Use traits as parameter bounds, with `impl Trait`, named generics, and `where` clauses.
- Return traits using `impl Trait` (static) and `Box<dyn Trait>` (dynamic).
- Define generic functions, structs, and enums.
- Combine multiple trait bounds.
- Recognize and use the most common standard library traits.
- Use traits to overload operators.
- Apply the newtype pattern to add behavior to existing types.

This module is the last major theoretical module. After it, you will be able to read any Rust function signature and understand what it requires.

> **A note on terminology.** Traits in Rust correspond to "interfaces" in Java, "protocols" in Swift, and "type classes" in Haskell. They are not classes. Generics correspond to "templates" in C++ and "generics" in Java or C#. The mental model that fits Rust best is: traits are sets of capabilities; generics are placeholders for "any type with these capabilities."

---

## A. Defining and Implementing Traits

### Defining a Trait

A trait is a named set of method signatures:

```rust
trait Summary {
    fn summarize(&self) -> String;
}
```

The trait `Summary` has one method: `summarize`. Any type that implements this trait must provide a `summarize` method that takes `&self` and returns a `String`.

A trait definition does not contain method bodies (yet; default implementations come in Section B). It only declares what methods exist.

### Implementing a Trait

To implement a trait for a type, write an `impl Trait for Type` block:

```rust
struct Article {
    title: String,
    body: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}", self.title, &self.body[..50.min(self.body.len())])
    }
}

struct Tweet {
    username: String,
    content: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("@{}: {}", self.username, self.content)
    }
}

fn main() {
    let article = Article {
        title: String::from("Rust 2024"),
        body: String::from("This year saw the release of..."),
    };

    let tweet = Tweet {
        username: String::from("rustlang"),
        content: String::from("Just shipped 1.84"),
    };

    println!("{}", article.summarize());
    println!("{}", tweet.summarize());
}
```

Both `Article` and `Tweet` implement `Summary` with their own logic. They share the trait but not the code.

### The Orphan Rule

You can implement a trait for a type only if at least one of them is defined in your crate. This is called the **orphan rule** or **coherence rule**.

```rust
// In your own crate:
struct MyType;

impl Summary for MyType { /* OK: MyType is yours */ }
impl std::fmt::Display for MyType { /* OK: MyType is yours */ }

// In your own crate:
trait MyTrait { }

impl MyTrait for String { /* OK: MyTrait is yours */ }
impl MyTrait for Vec<i32> { /* OK: MyTrait is yours */ }

// In your own crate:
// impl Display for Vec<i32> { /* ERROR: neither is yours */ }
```

The orphan rule prevents two crates from independently implementing the same trait for the same type, which would create a conflict. It is occasionally limiting (you sometimes need the newtype pattern, covered in Section I, to work around it) but is essential for the ecosystem to compose cleanly.

### Calling Trait Methods

Once implemented, trait methods are called like any other method:

```rust
let article = Article { /* ... */ };
let summary = article.summarize();
```

The compiler resolves the call based on the type of the receiver. If multiple traits provide methods with the same name, you can disambiguate:

```rust
trait A {
    fn name(&self) -> &str;
}

trait B {
    fn name(&self) -> &str;
}

struct Thing;

impl A for Thing {
    fn name(&self) -> &str { "thing-a" }
}

impl B for Thing {
    fn name(&self) -> &str { "thing-b" }
}

fn main() {
    let t = Thing;
    let a_name = A::name(&t);            // "thing-a"
    let b_name = B::name(&t);            // "thing-b"
    
    // t.name() would be ambiguous and the compiler would ask
    // you to disambiguate.
}
```

This kind of conflict is rare. When it arises, the explicit syntax handles it.

---

## B. Default Implementations

A trait method can have a default implementation that types use unless they override it.

### A Default Body

```rust
trait Summary {
    fn summarize(&self) -> String {
        String::from("(no summary available)")
    }
}

struct Article {
    title: String,
}

impl Summary for Article {
    // No summarize override; uses the default
}

struct Tweet {
    content: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("Tweet: {}", self.content)
    }
}

fn main() {
    let article = Article { title: String::from("Rust 2024") };
    let tweet = Tweet { content: String::from("hello rust") };

    println!("{}", article.summarize());     // "(no summary available)"
    println!("{}", tweet.summarize());        // "Tweet: hello rust"
}
```

`Article` uses the default implementation because it does not provide its own. `Tweet` overrides it.

### Defaults That Use Other Trait Methods

Default implementations can call other methods of the same trait, including ones without defaults. This lets a trait define a small set of required methods and provide derived methods automatically:

```rust
trait Summary {
    fn summarize_author(&self) -> String;       // required, no default

    fn summarize(&self) -> String {              // default uses summarize_author
        format!("(read more from {}...)", self.summarize_author())
    }
}

struct Article {
    author: String,
    title: String,
}

impl Summary for Article {
    fn summarize_author(&self) -> String {
        format!("@{}", self.author)
    }
    // summarize uses the default
}

fn main() {
    let article = Article {
        author: String::from("alice"),
        title: String::from("Rust 2024"),
    };
    println!("{}", article.summarize());        // "(read more from @alice...)"
}
```

`Article` only had to provide `summarize_author`. The default `summarize` calls it to produce the full summary.

This pattern is common in well-designed traits: a small set of required methods that the implementor must define, plus many derived methods with default implementations that the implementor gets for free.

### The Iterator Trait

You saw this pattern in Module 11 with `Iterator`. The trait requires only `next`:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;

    // dozens of methods with default implementations:
    fn map<B, F>(self, f: F) -> Map<Self, F> where /* ... */ { /* default */ }
    fn filter<P>(self, pred: P) -> Filter<Self, P> where /* ... */ { /* default */ }
    fn collect<B>(self) -> B where /* ... */ { /* default */ }
    // ... many more
}
```

Anyone implementing `Iterator` only writes `next`. They get `map`, `filter`, `collect`, `sum`, `count`, and many others for free, all built on top of `next`. This is the power of default implementations: one required method gives you a rich API.

---

## C. Traits as Parameters: `impl Trait` and Trait Bounds

A function can accept any type that implements a particular trait. This is one of the most common uses of generics in Rust.

### The Simple Form: `impl Trait`

The shortest syntax for "any type implementing this trait":

```rust
fn print_summary(item: &impl Summary) {
    println!("{}", item.summarize());
}

fn main() {
    let article = Article { /* ... */ };
    let tweet = Tweet { /* ... */ };

    print_summary(&article);
    print_summary(&tweet);
}
```

The parameter type `&impl Summary` reads as "a reference to any type that implements `Summary`." The compiler will accept any concrete type that satisfies the bound.

### The Equivalent: Generic with Trait Bound

The `impl Trait` syntax is shorthand for the generic form:

```rust
fn print_summary<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}
```

These two forms are equivalent. The generic form is more verbose but lets you reference the type parameter `T` in the rest of the signature, which `impl Trait` does not.

### When Each Form Is Appropriate

Use `impl Trait` when:

- You only need to mention the type once (in the parameter).
- The function takes a single argument with a trait bound.
- You do not need to constrain multiple parameters to the same type.

Use the named generic form when:

- You need to mention the type elsewhere (in another parameter, return type, or where clause).
- Multiple parameters must be the same type:

```rust
// Different forms for different intent:

// Two parameters, can be different types as long as both implement Summary:
fn pair_summary(a: &impl Summary, b: &impl Summary) -> String {
    format!("{}, {}", a.summarize(), b.summarize())
}

// Two parameters, must be the SAME type:
fn pair_summary_same<T: Summary>(a: &T, b: &T) -> String {
    format!("{}, {}", a.summarize(), b.summarize())
}
```

The first version accepts an `Article` and a `Tweet` together. The second requires both to be the same type.

### Multiple Trait Bounds with `+`

To require a parameter to implement multiple traits, use `+`:

```rust
fn process(item: &(impl Summary + std::fmt::Display)) {
    println!("processing: {}", item);             // uses Display
    println!("summary: {}", item.summarize());    // uses Summary
}
```

Or with the named generic form:

```rust
fn process<T: Summary + std::fmt::Display>(item: &T) {
    // ...
}
```

Both forms produce the same code. The choice is stylistic.

---

## D. Returning Traits: `impl Trait` and Dynamic Dispatch

Returning a value of a trait-bounded type is more involved than accepting one. There are two main forms, with different trade-offs.

### `impl Trait` in Return Position

The simpler form returns "some specific type implementing this trait":

```rust
fn make_summary(verbose: bool) -> impl Summary {
    if verbose {
        Article { /* ... */ }
    } else {
        Article { /* ... */ }   // must be the same concrete type
    }
}
```

`impl Trait` in return position means: "I am returning some specific type that implements this trait. The caller does not need to know exactly what type, but it is one specific type."

This works well when the function returns one type. The compiler picks that concrete type and uses it consistently.

### Limitation: Single Concrete Return Type

`impl Trait` does not work when the function might return different types from different code paths:

```rust
// Does NOT compile:
fn make_summary(verbose: bool) -> impl Summary {
    if verbose {
        Article { /* ... */ }       // returns Article
    } else {
        Tweet { /* ... */ }         // returns Tweet
    }
}
```

Even though both `Article` and `Tweet` implement `Summary`, the function cannot return both as `impl Summary`. The compiler needs one specific concrete type at compile time.

### Dynamic Dispatch with `Box<dyn Trait>`

When you genuinely need to return different concrete types, use `Box<dyn Trait>`:

```rust
fn make_summary(verbose: bool) -> Box<dyn Summary> {
    if verbose {
        Box::new(Article { /* ... */ })
    } else {
        Box::new(Tweet { /* ... */ })
    }
}
```

`Box<dyn Summary>` is a heap-allocated value that implements `Summary`. Each call to `make_summary` allocates a box and stores either an `Article` or a `Tweet` inside.

Calls to methods through `Box<dyn Summary>` go through dynamic dispatch: at runtime, the program looks up which method to call (since the concrete type is not known at compile time). This is one extra memory load per call.

### Static vs Dynamic Dispatch

The two forms have different costs and capabilities:

| Property                    | `impl Trait`            | `Box<dyn Trait>`         |
|-----------------------------|-------------------------|--------------------------|
| Compile-time type known     | Yes                     | No                       |
| Runtime overhead            | None                    | Heap alloc, vtable       |
| Multiple concrete types     | No                      | Yes                      |
| Inlinable                   | Yes                     | Limited                  |
| Storable in collections     | Only with same type     | Yes (uniform type)       |

For most code, prefer `impl Trait` when you can. Reach for `Box<dyn Trait>` when you genuinely need the flexibility of multiple types.

### Object Safety

Not all traits can be used as `dyn Trait`. A trait is **object-safe** if it can be used through dynamic dispatch. The rules are technical, but the most common limitations are:

- The trait cannot have generic methods (methods with their own type parameters).
- The trait cannot have methods that return `Self`.
- The trait cannot have associated constants in some cases.

Most practical traits are object-safe. When a trait is not, the compiler will tell you, and you usually need to redesign the trait or use static dispatch instead.

---

## E. Generics in Functions, Structs, and Enums

Generics let one definition work for many types. You have already seen them in `Vec<T>`, `Option<T>`, `Result<T, E>`, and `HashMap<K, V>`. This section shows how to write your own.

### Generic Functions

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let numbers = vec![10, 25, 3, 47, 12];
    let words = vec!["apple", "banana", "cherry"];

    println!("largest number: {}", largest(&numbers));
    println!("largest word: {}", largest(&words));
}
```

The function `largest` works for any type `T` that implements `PartialOrd` (the trait for `<` and `>`). The same compiled code, when called with different concrete types, produces specialized versions.

### Generic Structs

A struct can have type parameters:

```rust
struct Pair<T> {
    first: T,
    second: T,
}

impl<T: std::fmt::Display> Pair<T> {
    fn print(&self) {
        println!("({}, {})", self.first, self.second);
    }
}

fn main() {
    let pair = Pair { first: 1, second: 2 };
    pair.print();

    let pair = Pair { first: "hello", second: "world" };
    pair.print();
}
```

The struct definition has `<T>` after the name. The fields use `T` instead of a concrete type. The `impl` block also has `<T: ...>` to declare the type parameter.

The bound on the `impl` block (`T: Display`) restricts which types get the `print` method. A `Pair<i32>` and a `Pair<&str>` both have `print` (because `i32` and `&str` both implement `Display`), but a `Pair<Vec<i32>>` would not.

You can have multiple type parameters:

```rust
struct KeyValue<K, V> {
    key: K,
    value: V,
}

impl<K: std::fmt::Display, V: std::fmt::Display> KeyValue<K, V> {
    fn print(&self) {
        println!("{} = {}", self.key, self.value);
    }
}
```

### Generic Enums

`Option<T>` and `Result<T, E>` are generic enums. You can define your own:

```rust
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
    let value: Either<i32, String> = Either::Left(42);
    println!("{}", value.is_left());     // true
}
```

`Either<L, R>` has two type parameters because the variants hold different types. This is the same shape as `Result<T, E>` but without the success/failure semantic.

### Generic Methods

Methods on a generic type can have their own type parameters:

```rust
struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { items: Vec::new() }
    }

    fn push(&mut self, item: T) {
        self.items.push(item);
    }

    // A method with its own type parameter
    fn convert_to<U>(self, f: impl Fn(T) -> U) -> Stack<U> {
        let new_items = self.items.into_iter().map(f).collect();
        Stack { items: new_items }
    }
}
```

The `convert_to` method has its own type parameter `U`. It transforms a `Stack<T>` into a `Stack<U>` by applying a closure to each element. The struct's `T` and the method's `U` are independent.

### Monomorphization

The compiler implements generics through **monomorphization**: for each concrete type the generic is used with, the compiler generates a specialized version with the type substituted in.

```rust
fn identity<T>(x: T) -> T { x }

let a = identity(5);             // generates identity::<i32>
let b = identity("hello");       // generates identity::<&str>
let c = identity(3.14);          // generates identity::<f64>
```

After compilation, there is no "generic identity function" in the binary. There are three specialized versions, one for each type. They are completely independent of each other.

The benefit: each specialization is fully optimized for its specific type. There is no virtual dispatch, no type erasure, no boxing. The cost is binary size; each unique generic instantiation takes space.

For most code, the trade-off is heavily worth it. Rust generics are zero-cost abstractions: the generic version compiles to the same code you would write by hand for each type.

---

## F. Multiple Bounds and Where Clauses

For complex generic signatures, multiple bounds and where clauses keep the syntax readable.

### Inline Bounds

For a small number of bounds, inline syntax is fine:

```rust
fn process<T: Display + Clone>(item: T) {
    let copy = item.clone();
    println!("{copy}");
}
```

This requires `T` to implement both `Display` and `Clone`.

### Where Clauses

For larger signatures, where clauses are easier to read:

```rust
fn complex_function<T, U, V>(a: T, b: U, c: V) -> Vec<T>
where
    T: Clone + Display,
    U: Iterator<Item = T> + Sized,
    V: Fn(T) -> bool,
{
    // ...
}
```

The bounds appear after the parameter list, in a `where` clause. This separates "what the function does" from "what its parameters require."

The two forms are equivalent:

```rust
// Inline:
fn foo<T: Display + Clone>(x: T) { /* ... */ }

// Where clause:
fn foo<T>(x: T) where T: Display + Clone { /* ... */ }
```

The convention in idiomatic Rust:

- Inline bounds for one or two simple constraints.
- Where clauses when there are three or more bounds, when one parameter has multiple bounds, or when the constraint involves an associated type.

### Where Clauses on `impl` Blocks

Where clauses also appear on `impl` blocks:

```rust
struct Container<T> {
    items: Vec<T>,
}

impl<T> Container<T>
where
    T: Clone + std::fmt::Display,
{
    fn print_all(&self) {
        for item in &self.items {
            println!("{item}");
        }
    }
}
```

The methods in this `impl` block are only available when `T` implements both `Clone` and `Display`. A `Container<i32>` has `print_all` (i32 satisfies both), but a `Container<MyType>` would only have it if `MyType` implemented both.

This kind of conditional implementation is one of the most useful generic patterns. It lets you provide methods that only make sense for some types.

---

## G. Commonly Used Standard Library Traits

Several traits in the standard library appear constantly. Knowing them well is essential for reading and writing idiomatic Rust.

### `Display` and `Debug`

Both control how values are formatted (Module 12 covered this).

- `Display` is for user-facing output. Used by `{}`. Must be implemented manually.
- `Debug` is for developer-facing output. Used by `{:?}`. Can be `#[derive(Debug)]`.

```rust
#[derive(Debug)]
struct Point { x: f64, y: f64 }

impl std::fmt::Display for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

### `Clone` and `Copy`

Both control duplication (Module 6 covered this).

- `Clone` is explicit duplication via `.clone()`. Heap data is duplicated.
- `Copy` is implicit, byte-wise duplication. For types that are entirely on the stack.

```rust
#[derive(Clone)]
struct OwnedData { name: String, values: Vec<i32> }

#[derive(Copy, Clone)]
struct Point { x: f64, y: f64 }
```

You must derive `Clone` to derive `Copy`. The two go together.

### `From` and `Into`

Both control type conversion. `From` is the primary trait; `Into` is automatically derived from it.

```rust
struct Celsius(f64);
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

fn main() {
    let c = Celsius(100.0);
    let f: Fahrenheit = c.into();         // uses From<Celsius>

    // Or explicitly:
    let f = Fahrenheit::from(Celsius(0.0));
}
```

Implementing `From<A> for B` automatically gives you `Into<B> for A`. The two are equivalent in different syntactic positions: `B::from(a)` and `a.into()`.

These traits are how the `?` operator does error type conversion. When `?` unwraps a `Result<T, E1>` in a function returning `Result<T, E2>`, it converts via `E2::from(e1)`. As long as `From<E1> for E2` is implemented, the conversion is automatic.

### `Iterator`

The trait that powers iterator chains (Module 11 covered this).

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // many more methods with default implementations
}
```

Implementing `Iterator` for a custom type gives you the entire iterator API for free.

### `Default`

Provides a "default" value for a type:

```rust
trait Default {
    fn default() -> Self;
}
```

Many standard library types implement `Default`: integers (default 0), strings (empty), vectors (empty), `Option` (None).

You can derive it for structs whose fields all implement `Default`:

```rust
#[derive(Default)]
struct Config {
    timeout_ms: u32,
    retries: u32,
    verbose: bool,
}

let cfg = Config::default();         // all fields zero/false
```

`Default` is useful in generic code that needs to produce "empty" or "starting" values.

### `PartialEq` and `Eq`

`PartialEq` allows `==`. `Eq` is a marker for full equality (no NaN-like exceptions). Both can be derived.

### `PartialOrd` and `Ord`

`PartialOrd` allows `<`, `>`, `<=`, `>=`. `Ord` requires a total ordering. Both can be derived.

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Version {
    major: u32,
    minor: u32,
    patch: u32,
}

let v1 = Version { major: 1, minor: 2, patch: 3 };
let v2 = Version { major: 1, minor: 3, patch: 0 };

assert!(v1 < v2);                    // Lexicographic order on the fields
```

The derived implementations compare fields in declaration order. For `Version`, this means: first compare `major`; if equal, compare `minor`; if equal, compare `patch`.

### `Hash`

Required for using a type as a `HashMap` key or `HashSet` element. Can be derived if all fields implement `Hash`.

```rust
#[derive(Hash, PartialEq, Eq)]
struct UserId(u64);

let mut users: HashMap<UserId, String> = HashMap::new();
users.insert(UserId(1), String::from("alice"));
```

### A Common Derive Set

Many structs in idiomatic Rust derive a standard set:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct UserId(u64);
```

This gives the type formatted printing, explicit copying, equality testing, and hashability. From one line, the type can be used in `HashMap`, `HashSet`, `Vec`, comparisons, and debug output. Most of the work is done; you only write methods specific to your type.

---

## H. Operator Overloading with Traits

Most operators in Rust are implemented as traits. Implementing the trait for your type lets you use the operator on values of that type.

### The Operator Traits

A few of the most common:

- `Add` for `+`
- `Sub` for `-`
- `Mul` for `*`
- `Div` for `/`
- `Neg` for unary `-`
- `Not` for `!`
- `Index` for `[]`

These are in `std::ops`. Each one has a specific shape.

### Implementing `Add`

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Vector2 {
    x: f64,
    y: f64,
}

impl Add for Vector2 {
    type Output = Vector2;

    fn add(self, other: Vector2) -> Vector2 {
        Vector2 {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    let a = Vector2 { x: 1.0, y: 2.0 };
    let b = Vector2 { x: 3.0, y: 4.0 };

    let sum = a + b;
    println!("{sum:?}");                      // Vector2 { x: 4.0, y: 6.0 }
}
```

The `Add` trait has an associated type `Output` for the result type and a method `add` for the operation. Implementing the trait makes `+` work for `Vector2 + Vector2`.

### Adding Different Types

You can implement `Add` between different types by using the generic parameter:

```rust
impl Add<f64> for Vector2 {
    type Output = Vector2;

    fn add(self, scalar: f64) -> Vector2 {
        Vector2 {
            x: self.x + scalar,
            y: self.y + scalar,
        }
    }
}

// Now this works:
let a = Vector2 { x: 1.0, y: 2.0 };
let scaled = a + 5.0;                  // Vector2 { x: 6.0, y: 7.0 }
```

Note that addition is not commutative for the compiler. `a + 5.0` works because `Add<f64> for Vector2` is implemented; `5.0 + a` would require `Add<Vector2> for f64`, which is separate.

### When NOT to Overload Operators

Just because you can does not mean you should:

- Operators should mean what they normally mean. `+` should be addition-like, `*` should be multiplication-like, etc. Overloading them with arbitrary semantics is confusing.
- For non-mathematical types, named methods are usually clearer than overloaded operators.
- Operator overloading is most useful for types with genuine mathematical meaning: vectors, matrices, complex numbers, durations, money.

For example, `String` does NOT implement `*` for repeating, even though Python uses `*` for that. `"abc" * 3` is a compile error in Rust. The standard library uses the named `repeat` method instead, which is more discoverable.

### `PartialOrd` and Custom Ordering

For comparison operators (`<`, `>`, etc.), you implement `PartialOrd`:

```rust
use std::cmp::Ordering;

struct Length {
    millimeters: u64,
}

impl PartialEq for Length {
    fn eq(&self, other: &Length) -> bool {
        self.millimeters == other.millimeters
    }
}

impl PartialOrd for Length {
    fn partial_cmp(&self, other: &Length) -> Option<Ordering> {
        Some(self.millimeters.cmp(&other.millimeters))
    }
}
```

For most types, deriving `PartialOrd` (along with `PartialEq`, `Eq`, `Ord`) is sufficient. The derived implementations compare fields in declaration order.

---

## I. The Newtype Pattern

The newtype pattern wraps an existing type in a tuple struct to give it a new identity. This is one of the most useful patterns in Rust.

### The Pattern

```rust
struct UserId(u64);
struct ProductId(u64);
```

Both wrap a `u64`, but they are distinct types. The compiler will not confuse them.

### Why Use It

Three main reasons.

**Type safety.** Two wrapper types over the same primitive are different types. The compiler prevents you from passing one where the other is expected:

```rust
fn lookup_user(id: UserId) -> Option<User> { /* ... */ }

fn main() {
    let user_id = UserId(42);
    let product_id = ProductId(42);

    let user = lookup_user(user_id);            // OK
    // let user = lookup_user(product_id);      // ERROR: types differ
}
```

In a system with many ID-like values, this prevents the entire class of "I passed the wrong ID" bugs.

**Adding traits to external types.** The orphan rule (Section A) prevents implementing a trait you do not own on a type you do not own. The newtype pattern works around this:

```rust
// Can't do this:
// impl Display for Vec<i32> { /* ... */ }    // ERROR: both are external

// Can do this:
struct Wrapper(Vec<i32>);

impl std::fmt::Display for Wrapper {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "[")?;
        for (i, n) in self.0.iter().enumerate() {
            if i > 0 { write!(f, ", ")?; }
            write!(f, "{n}")?;
        }
        write!(f, "]")
    }
}
```

The wrapper is a type you own, so you can implement any trait on it.

**Restricting an interface.** A wrapper type can expose only some of the underlying type's operations:

```rust
struct PositiveInteger(u64);

impl PositiveInteger {
    fn new(value: u64) -> Option<Self> {
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
```

This type guarantees that the inner value is always positive (because the constructor rejects zero). Code that needs a positive integer can require `PositiveInteger`, and the compiler enforces the invariant.

### The Cost

Newtype wrappers have zero runtime cost. The compiler treats them as identical to the underlying type at the machine code level. The "cost" is at the source code level: you have to deconstruct them with `.0` (for tuple structs) or methods you provide.

For frequently-used wrappers, you can implement `Deref` to make the inner methods accessible directly, but this partially defeats the encapsulation. The right balance depends on the use case.

### When to Use the Newtype Pattern

Common situations:

- **IDs and identifiers.** `UserId(u64)`, `OrderNumber(u32)`, `SessionToken(String)`. Prevents mixing them up.
- **Units of measurement.** `Meters(f64)`, `Kilograms(f64)`, `Seconds(u32)`. Catches unit-conversion bugs.
- **Validated values.** `Email(String)`, `PositiveNumber(i32)`. Encapsulates validation.
- **Implementing external traits on external types.** When the orphan rule blocks a direct implementation.
- **Adding domain semantics.** A `Username(String)` is more meaningful than a bare `String`.

The newtype pattern is one of the most idiomatic Rust techniques. It costs nothing at runtime and adds significant safety at the source level.

---

## Module Summary

- Traits define behavior. Types implement traits to gain that behavior. Multiple types can implement the same trait, and the same type can implement many traits.
- Default implementations let traits provide methods that all implementations share unless they override them. The `Iterator` trait uses this heavily.
- `impl Trait` and trait bounds let functions accept any type implementing a trait. Generic functions with bounds are the most common form.
- Returning traits uses `impl Trait` for static dispatch (one concrete type) or `Box<dyn Trait>` for dynamic dispatch (multiple types possible at runtime).
- Generics in functions, structs, and enums allow one definition to work for many types. Monomorphization makes this zero-cost.
- Complex generic signatures are organized with multiple bounds and `where` clauses.
- Common standard library traits (`Display`, `Debug`, `Clone`, `From`, `Into`, `Iterator`) appear constantly. Knowing them well is essential.
- Operators are implemented as traits. Implementing `Add`, `Sub`, etc. lets your types use operators naturally.
- The newtype pattern wraps existing types in tuple structs. It enforces type distinctions, allows trait implementations on external types, and encapsulates invariants.

## Discussion Questions

1. Rust uses traits where many languages use inheritance. Identify a specific kind of design that is easier in Rust's trait system, and one that is harder. What does this trade-off optimize for?
2. Generics in Rust are monomorphized: the compiler generates specialized code for each concrete type. Java generics use type erasure. What does each approach optimize for, and what does it cost?
3. The newtype pattern wraps a type just to give it a new identity. This kind of "weight without runtime cost" pattern is rare in many languages. What does Rust's type system gain by making this pattern available, and what does it cost in terms of code volume?

