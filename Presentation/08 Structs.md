# Module 8: Structs

## Module Overview

Structs are how you define your own data types in Rust. They group related values into a named, typed bundle. If you have used `class` in Java or C#, `struct` in C, or `dict` with fixed keys in Python, the basic concept is familiar. Rust's structs add the type system and ownership integration you have already seen, plus a clean separation between data definition and the methods that operate on the data.

The interesting parts of this module are not the syntax (which is straightforward) but the design choices behind it. Three ideas are central:

1. **Structs separate data from behavior.** A struct definition contains only fields. Methods are defined separately in `impl` blocks. This is different from object-oriented languages where data and methods live together.
2. **Constructors are not special.** Rust does not have a constructor keyword. Instead, the convention is to write a regular function (often called `new`) that returns an instance.
3. **Shared functionality is opted into via traits.** Capabilities like printing, copying, and comparing are implemented through the trait system, often via a single `derive` line. This is more flexible than inheritance and avoids the fragile-base-class problem.

By the end of this module, students will:

- Define structs with named fields and create instances of them.
- Use the field init shorthand and the struct update syntax to write concise initialization code.
- Recognize tuple structs and unit-like structs and know when to use each.
- Define methods and associated functions on structs using `impl` blocks.
- Apply the constructor pattern to validate and construct struct instances.
- Use `#[derive]` to add common functionality with one line.

This module is the first in a sequence (structs, then enums, then traits) that gives you the tools to model complex domains. Most of the data structures you will design in Rust are built from these pieces.

> **A note on terminology.** Rust's `struct` keyword corresponds to what some languages call a "class" or "record" or "data class." It defines a new type. Rust does not have inheritance; the relationships between types are expressed through traits, covered in Module 9. For students from object-oriented backgrounds, the mental shift is to think of structs as data definitions, traits as capability definitions, and the combination of the two as Rust's answer to classes.

---

## A. Defining and Instantiating Structs

### Defining a Struct

A struct definition introduces a new type. Each field has a name and a type, just like function parameters:

```rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
```

The struct is named `User`. It has four fields. The naming convention is `PascalCase` for the struct name and `snake_case` for the field names.

A struct definition does not allocate memory or create an instance. It is a description of a shape. Instances are created by writing the struct name followed by values for each field.

### Creating an Instance

```rust
fn main() {
    let user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        sign_in_count: 1,
        active: true,
    };

    println!("Active: {}", user.active);
    println!("Email: {}", user.email);
}
```

Three things to note:

- The fields can be listed in any order. The names match them up.
- Every field must be given a value at creation. There is no notion of an uninitialized struct.
- Field access uses dot notation: `user.email`, `user.active`.

### Mutability

Like all bindings, struct instances are immutable by default. To modify a field, the entire instance must be `mut`:

```rust
fn main() {
    let mut user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        sign_in_count: 1,
        active: true,
    };

    user.email = String::from("alice@new.com");
    user.sign_in_count += 1;
}
```

Rust does not allow per-field mutability. You cannot say "this field is mutable but that one is not." If you want partial immutability, the typical pattern is to make some fields private (covered later in the module on modules) and expose them only through methods that enforce the invariants.

### Struct Layout in Memory

A struct's fields are stored contiguously in memory. The compiler may reorder fields for efficient layout (smaller padding, better alignment), but each instance occupies a single contiguous region.

For the `User` struct above, an instance occupies roughly:

- 24 bytes for the `username` (a `String` is a pointer, length, and capacity).
- 24 bytes for the `email` (same).
- 8 bytes for the `sign_in_count`.
- 1 byte for `active`, plus padding for alignment.

The actual layout is implementation-defined; if you need a specific layout (for FFI or wire formats), you can use `#[repr(C)]` to lock it in.

The point for now: structs do not have hidden fields, vtables, or runtime metadata. They are exactly what you wrote, packed efficiently in memory.

---

## B. Field Init Shorthand and Struct Update Syntax

Two small syntactic features make struct construction cleaner in common cases.

### Field Init Shorthand

When the variable name matches the field name, you can omit the explicit assignment:

```rust
fn build_user(username: String, email: String) -> User {
    User {
        username,
        email,
        sign_in_count: 1,
        active: true,
    }
}
```

The fields `username` and `email` are written without `username: username`. The compiler matches them by name. This is a small convenience that reduces noise in functions that construct structs from their parameters.

The longer form still works:

```rust
User {
    username: username,           // verbose but legal
    email,                        // shorthand
    // ...
}
```

The shorthand is the convention. Use it whenever the names match.

### Struct Update Syntax

When you want to create a new struct that is mostly the same as an existing one but with a few fields changed, the struct update syntax (`..`) lets you say "fill in the rest from this other struct":

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
}
```

The `..user1` says "for any fields I did not list above, copy them from `user1`." `user2` ends up with the same `username`, `sign_in_count`, and `active` as `user1`, but its own `email`.

### Move Semantics with Update Syntax

The update syntax follows the ownership rules. If the source struct contains non-`Copy` fields that are pulled in, the source struct is moved (or partially moved). After the update, you cannot use the source if any of its non-`Copy` fields were used.

In the example above, `user2`'s `username` was copied (well, moved) from `user1`. Because `String` is not `Copy`, `user1.username` is no longer valid. After the update, `user1` is partially moved, and you cannot use it as a whole.

If you want a deep copy, use `.clone()`:

```rust
let user2 = User {
    email: String::from("alice@new.com"),
    ..user1.clone()
};
```

This requires `User` to implement `Clone`, covered in Section F.

### Where the Update Syntax Helps

The update syntax is useful when:

- A struct has many fields and you want to change one or two.
- You are writing functional-style code that produces new struct values rather than mutating in place.
- You want to clearly signal "this is mostly the same as that" in the source.

It is not useful (and probably confusing) when you would change most fields anyway. In that case, write the full constructor.

---

## C. Tuple Structs and Unit-Like Structs

Rust has three kinds of structs. Most of the time you use the regular kind with named fields. The other two have specific use cases.

### Tuple Structs

A tuple struct has fields without names; the fields are accessed by position:

```rust
struct Point(i32, i32);
struct Color(u8, u8, u8);
struct Inches(f64);
```

You construct them like a function call and access fields by index:

```rust
fn main() {
    let origin = Point(0, 0);
    let pixel = Color(255, 128, 0);
    let length = Inches(12.5);

    println!("x: {}", origin.0);
    println!("g: {}", pixel.1);
    println!("inches: {}", length.0);
}
```

A tuple struct is technically a tuple with a name on it. The compiler treats `Point(i32, i32)` as a distinct type from `(i32, i32)` and from any other tuple struct, even one with identical fields.

### Why Tuple Structs?

Tuple structs are useful in two specific situations:

**The newtype pattern.** When you want a distinct type that wraps an existing type, tuple structs are concise. The classic example is using the type system to prevent unit confusion:

```rust
struct Meters(f64);
struct Feet(f64);

fn distance_in_meters(d: Meters) -> Meters {
    Meters(d.0 * 1.5)
}

fn main() {
    let height = Meters(100.0);
    let result = distance_in_meters(height);

    // distance_in_meters(Feet(5.0));      // ERROR: type mismatch
}
```

Both `Meters` and `Feet` wrap an `f64`, but they are different types. The function only accepts `Meters`. This prevents the entire class of bugs where someone passes feet to a function expecting meters.

**Short groupings where named fields would be redundant.** A `Point(x, y)` is well-known enough that field names like `x` and `y` would not add much information. The tuple struct is concise.

In general, prefer regular structs when there are more than two fields or when the meaning of each position would not be obvious. Tuple structs work best for two-field types or for the newtype pattern.

### Unit-Like Structs

A unit-like struct has no fields at all:

```rust
struct AlwaysEqual;
```

You construct it without parentheses or braces:

```rust
let instance = AlwaysEqual;
```

A unit-like struct contains no data; it is a name with no fields. This sounds useless, but it has a specific purpose: implementing traits that need a type but no data.

For example, you might define a marker type that implements a trait to signal something:

```rust
struct ReadOnly;
struct ReadWrite;

// These are markers used with type parameters elsewhere in the program.
// They have no data; they exist purely to differentiate at the type level.
```

This pattern (called the type-state pattern) is more advanced and shows up in libraries that use the type system heavily. For everyday code, you will write unit-like structs occasionally.

### Summary

| Kind         | When to Use                                          |
|--------------|------------------------------------------------------|
| Named fields | The default. Most structs.                            |
| Tuple struct | Newtype pattern, two-field types, distinct types over the same primitive. |
| Unit-like    | Markers and type-level signaling.                     |

---

## D. `impl` Blocks: Methods and Associated Functions

A struct definition contains only fields. To attach behavior to a struct, you write an `impl` block.

### Methods

A method is a function that takes `self` (the instance) as its first parameter:

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
}

fn main() {
    let rect = Rectangle { width: 3.0, height: 4.0 };

    println!("area: {}", rect.area());
    println!("perimeter: {}", rect.perimeter());
}
```

The methods are defined inside `impl Rectangle { ... }`. They are called with dot notation: `rect.area()`. The compiler automatically passes the instance as `self` when you use dot notation.

### The Three Forms of `self`

The first parameter of a method determines how the method receives the instance:

- **`&self`**: borrow the instance immutably. Most common; used when the method only reads.
- **`&mut self`**: borrow the instance mutably. Used when the method modifies fields.
- **`self`**: take ownership of the instance. Used when the method consumes or transforms the instance.

```rust
impl Rectangle {
    // &self: read-only access
    fn area(&self) -> f64 {
        self.width * self.height
    }

    // &mut self: can modify
    fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }

    // self: takes ownership; the original instance cannot be used afterward
    fn into_square(self) -> Rectangle {
        let side = (self.width + self.height) / 2.0;
        Rectangle { width: side, height: side }
    }
}
```

The three forms parallel the three reference types from Module 6. Choose based on what the method needs:

- Reads only: `&self`.
- Modifies the instance in place: `&mut self`.
- Consumes the instance to produce something else: `self`.

The convention is: most methods take `&self`. A few take `&mut self`. Very few take `self`. When you see `self` (without a `&`), the method consumes the instance, and the caller cannot use the original after the call.

### Associated Functions

A function defined in an `impl` block that does not take `self` is called an **associated function**. It is not a method (no instance is involved); it is a function associated with the type.

```rust
impl Rectangle {
    fn square(side: f64) -> Rectangle {
        Rectangle { width: side, height: side }
    }
}

fn main() {
    let sq = Rectangle::square(5.0);
    println!("area: {}", sq.area());
}
```

Associated functions are called with the `TypeName::function_name` syntax (`Rectangle::square(5.0)`), not with dot notation. They are most commonly used as constructors, covered in the next section.

### Multiple `impl` Blocks

A struct can have multiple `impl` blocks. They are equivalent to one combined block:

```rust
impl Rectangle {
    fn area(&self) -> f64 { self.width * self.height }
}

impl Rectangle {
    fn perimeter(&self) -> f64 { 2.0 * (self.width + self.height) }
}
```

Both methods are available on `Rectangle`. Multiple blocks are useful for organizing large impl groups, separating implementations of different concerns, or implementing traits in their own blocks (covered in Module 9).

### Method vs Function

A common question: when should logic be a method, and when should it be a free function?

A method is appropriate when:

- The behavior is naturally associated with the type.
- It conceptually operates on an instance of the type.
- Most of the work uses fields of the instance.

A free function is appropriate when:

- The behavior is general and could apply to multiple types.
- No single type "owns" the operation.
- The function is a utility (formatting, printing, comparison) that does not need privileged access to fields.

There is no hard rule. Most things that act on a single struct are best as methods. Things that combine multiple types or are part of a general framework often work better as free functions.

---

## E. The Constructor Pattern (`new()`)

Rust does not have a built-in constructor mechanism. There is no `constructor` keyword and no special syntax that runs at construction time. Instead, the convention is to write an associated function (often named `new`) that returns an instance.

### The Basic Pattern

```rust
struct Point {
    x: f64,
    y: f64,
}

impl Point {
    fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }
}

fn main() {
    let p = Point::new(3.0, 4.0);
    println!("({}, {})", p.x, p.y);
}
```

The `new` function takes the parameters needed to construct an instance and returns a `Point`. The body uses field init shorthand (`x, y` instead of `x: x, y: y`) because the parameter names match the field names.

This is the convention. Any function name is allowed, but `new` is recognized and expected by Rust developers.

### Why This Pattern?

A constructor function has several practical advantages:

**Validation.** The constructor can reject invalid input by returning a `Result` or panicking:

```rust
impl Point {
    fn new(x: f64, y: f64) -> Result<Point, String> {
        if x.is_nan() || y.is_nan() {
            Err("coordinates cannot be NaN".to_string())
        } else {
            Ok(Point { x, y })
        }
    }
}
```

**Encapsulation.** If some fields are computed from others, the constructor handles the computation:

```rust
struct Circle {
    radius: f64,
    area: f64,
    circumference: f64,
}

impl Circle {
    fn new(radius: f64) -> Circle {
        Circle {
            radius,
            area: std::f64::consts::PI * radius * radius,
            circumference: 2.0 * std::f64::consts::PI * radius,
        }
    }
}
```

**Default values.** A constructor can supply defaults for fields the caller does not need to specify:

```rust
struct Config {
    timeout_ms: u32,
    retries: u32,
    verbose: bool,
}

impl Config {
    fn new() -> Config {
        Config {
            timeout_ms: 5000,
            retries: 3,
            verbose: false,
        }
    }
}
```

### Multiple Constructors

A struct can have any number of associated functions. Common patterns:

```rust
impl Rectangle {
    fn new(width: f64, height: f64) -> Rectangle {
        Rectangle { width, height }
    }

    fn square(side: f64) -> Rectangle {
        Rectangle { width: side, height: side }
    }

    fn from_diagonal(d: f64) -> Rectangle {
        let side = d / 2.0_f64.sqrt();
        Rectangle { width: side, height: side }
    }
}
```

The naming convention: `new` for the most general constructor, `from_X` for constructors that take specific kinds of input, and descriptive names like `square` for special cases. There is no language enforcement; the convention is just for clarity.

### When to Use a Constructor vs. Direct Construction

Direct construction (`Point { x: 3.0, y: 4.0 }`) is fine for simple structs with no validation or computation. A constructor function is preferred when:

- Validation is needed.
- Some fields are computed from others.
- The construction logic is non-trivial.
- The struct's fields might become private later (a constructor is API-stable; direct construction is not).

For library code, constructors are the norm because they leave room to add validation or change the implementation later without breaking callers.

### The `Default` Trait

When the constructor with no arguments is the canonical "default" instance, Rust provides a trait called `Default` that formalizes the pattern:

```rust
impl Default for Config {
    fn default() -> Config {
        Config {
            timeout_ms: 5000,
            retries: 3,
            verbose: false,
        }
    }
}

fn main() {
    let cfg: Config = Default::default();
    let cfg2 = Config::default();           // equivalent
}
```

Implementing `Default` lets `Config::default()` work and also lets the struct participate in generic code that asks for default values. We will return to traits and `Default` formally in Module 9; for this module, the pattern is worth knowing.

---

## F. Deriving Common Traits: `Debug`, `Clone`, `PartialEq`

A trait, in Rust, is a named set of capabilities a type can implement. You will see traits in detail in Module 9. For this module, the relevant fact is that some traits are so common that the compiler can implement them for you automatically. The `#[derive]` attribute is the syntax for asking the compiler to do this.

### `Debug`: Formatted Printing

The `println!` macro has two formatting placeholders for arbitrary values:

- `{}` requires the type to implement `Display` (user-facing output).
- `{:?}` requires the type to implement `Debug` (developer-facing, debugging output).

By default, a struct implements neither. Trying to print one fails:

```rust
struct Point { x: f64, y: f64 }

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    println!("{p:?}");                    // ERROR: Point doesn't implement Debug
}
```

The fix is one line:

```rust
#[derive(Debug)]
struct Point { x: f64, y: f64 }

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    println!("{p:?}");                    // Point { x: 3.0, y: 4.0 }
    println!("{p:#?}");                   // pretty-printed across lines
}
```

The `#[derive(Debug)]` attribute tells the compiler to generate a `Debug` implementation automatically. The output uses the struct name and all the field names and values.

`Debug` is appropriate for almost all structs. The convention is to derive it on every struct unless there is a specific reason not to. It is invaluable when debugging.

### `Clone`: Explicit Duplication

`Clone` lets you call `.clone()` on a value to get an independent copy:

```rust
#[derive(Debug, Clone)]
struct Point { x: f64, y: f64 }

fn main() {
    let p1 = Point { x: 3.0, y: 4.0 };
    let p2 = p1.clone();
    println!("{p1:?}, {p2:?}");           // both work
}
```

For `Clone` to be derivable, every field of the struct must implement `Clone`. The compiler generates an implementation that calls `.clone()` on each field.

Most user-defined structs should derive `Clone` unless cloning would be expensive or unsafe (for example, if the struct holds a unique resource like a file handle that should not be duplicated).

### `Copy`: Implicit Duplication

`Copy` is a more restrictive version of `Clone`. Types that derive `Copy` can be duplicated by simply copying their bits, with no destructor logic. As Module 6 covered, integers, floats, and other simple types are `Copy`.

A struct can derive `Copy` only if every field is `Copy`. Adding `Copy` to a struct lets it behave like a primitive: assigning it makes a copy, and passing it to a function does not move it.

```rust
#[derive(Debug, Clone, Copy)]
struct Point { x: f64, y: f64 }

fn main() {
    let p1 = Point { x: 3.0, y: 4.0 };
    let p2 = p1;                          // copy, not move
    println!("{p1:?}, {p2:?}");           // both work
}
```

The convention: derive `Copy` for small structs whose fields are all `Copy`. Skip `Copy` for structs that own heap data (like a `String`) or that should not be silently duplicated.

When you derive `Copy`, you must also derive `Clone`. The two go together.

### `PartialEq` and `Eq`: Equality Comparison

`PartialEq` lets you use `==` and `!=`:

```rust
#[derive(Debug, PartialEq)]
struct Point { x: f64, y: f64 }

fn main() {
    let p1 = Point { x: 3.0, y: 4.0 };
    let p2 = Point { x: 3.0, y: 4.0 };
    let p3 = Point { x: 1.0, y: 2.0 };

    println!("{}", p1 == p2);             // true
    println!("{}", p1 == p3);             // false
}
```

The derived implementation compares all fields. Two instances are equal if and only if every field is equal.

`Eq` is a stronger version of `PartialEq` for types that support full equality (every value equals itself, no `NaN`-like exceptions). Floating-point types do not implement `Eq` because `NaN != NaN`. For integer-valued structs, deriving `Eq` is appropriate; for floating-point structs, `PartialEq` alone is correct.

### Other Common Derivable Traits

- **`PartialOrd` / `Ord`**: comparison with `<`, `<=`, `>`, `>=`.
- **`Hash`**: lets the type be a key in a `HashMap` or `HashSet`.
- **`Default`**: provides a "default" value (zero, empty, etc.).

The convention for many structs is to derive a standard set:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct UserId(u64);
```

This gives the type printability, copying via `.clone()`, equality, and hashability. It can be used in a `HashMap`, printed with `{:?}`, and compared with `==`. All from one line.

### When `derive` Is Not Enough

If the struct contains a field that does not implement the trait you want to derive, the compiler refuses. For example, a function pointer does not implement `PartialEq`:

```rust
#[derive(PartialEq)]            // ERROR: function pointers don't implement PartialEq
struct Handler {
    callback: fn(i32) -> i32,
}
```

In these cases, you can implement the trait by hand (write the comparison logic yourself) or restructure the type so the derivation works.

You can also implement traits for types that have fields supporting them, but where you want different behavior. For example, two structs might be considered equal even if some metadata field differs. In that case, write the implementation manually instead of deriving.

Manual trait implementations are covered in Module 9. For this module, `derive` handles the vast majority of cases.

### A Practical Example

A typical struct in a real codebase:

```rust
#[derive(Debug, Clone, PartialEq)]
struct User {
    id: u64,
    username: String,
    email: String,
    active: bool,
}

impl User {
    fn new(id: u64, username: String, email: String) -> User {
        User { id, username, email, active: true }
    }

    fn deactivate(&mut self) {
        self.active = false;
    }
}

fn main() {
    let mut user = User::new(1, "alice".to_string(), "alice@example.com".to_string());
    println!("{user:?}");

    user.deactivate();
    println!("{user:?}");

    let user2 = user.clone();
    println!("equal: {}", user == user2);
}
```

This pattern (struct definition, `derive` line, `impl` block with `new` and methods) is what you will write most often when defining custom types.

---

## Module Summary

- A struct is a named type with a fixed set of typed fields. The struct itself describes a shape; instances hold the actual data.
- The field init shorthand (`name` for `name: name`) and struct update syntax (`..source`) make construction concise.
- Tuple structs have positional fields without names. They are useful for the newtype pattern. Unit-like structs have no fields and are used for type-level markers.
- Methods and associated functions are defined in `impl` blocks. Methods take `&self`, `&mut self`, or `self` as their first parameter, paralleling the three reference types.
- The constructor pattern uses a regular associated function, conventionally named `new`. This handles validation, computed fields, and defaults.
- The `#[derive]` attribute generates implementations of common traits: `Debug` for printing, `Clone` for explicit duplication, `Copy` for implicit duplication, `PartialEq` for equality. Most structs derive a standard set of these.

## Discussion Questions

1. Rust separates struct definitions from their methods (in `impl` blocks), unlike languages that combine them in a single class definition. What does this separation enable, and what does it cost compared to the integrated approach?
2. Constructors are not a special concept in Rust; they are just associated functions. Identify a benefit of this approach and a downside compared to languages with dedicated constructor syntax.
3. The `#[derive]` attribute generates trait implementations automatically. What kinds of types should NOT derive `Copy` even though all their fields are `Copy`? What kinds should NOT derive `PartialEq` even though equality could be defined?
