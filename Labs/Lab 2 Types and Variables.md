# Module 3: Rust Types and Variables
## Lab 2 -- Working with Scalar and Compound Types

> **Course:** Mastering Rust
> **Module:** 3 - Rust Types and Variables
> **Estimated time:** 75-90 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 3: scalar types, type inference, immutability, constants, shadowing, and compound types. You will build a single Cargo project called `temperature_tracker`, which models temperature readings throughout a day. The project starts simple and grows with each exercise. By the end, the program demonstrates idiomatic use of all the type system features covered in the module.

The focus is on the type system itself, not on application logic. Most exercises take only a few lines of code; the value comes from observing how the compiler responds to each change and understanding why the language behaves the way it does.

By the end of this lab you will be able to:

- Choose appropriate integer and floating-point types for different kinds of numeric data.
- Recognize the situations where type inference fails and apply explicit annotations or turbofish syntax.
- Use immutability and `mut` to express intent in variable declarations.
- Distinguish `const` from `static` and choose the right one for a given case.
- Apply shadowing to express type transformations and intermediate values.
- Build, destructure, and iterate over tuples and arrays.
- Read compiler error messages well enough to fix type-related mistakes quickly.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm that the following are working:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` should be installed and the `editor.formatOnSave` setting should be enabled. If any of this is not in place, return to Lab 1 before continuing.

> **Note:** This lab assumes you have completed Module 3's reading. If you have not, several exercises will feel arbitrary. The lab applies concepts; it does not introduce them.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs        # or wherever you put Lab 1
cargo new temperature_tracker
cd temperature_tracker
code .
```

Create a `lab2-notes.md` file in the project root. You will use it for observations and checkpoint answers, just as in Lab 1.

---

## Exercise 1 -- Scalar Types and Numeric Literals

**Estimated time:** 10 minutes
**Topics covered:** integer types, floating-point types, numeric literals, type sizes

### Context

Rust integers are explicit about their width and signedness. The default `i32` is convenient but is not always correct. Picking the right type for a value is part of writing readable Rust. A counter that will never exceed 100 should not be a `u64`, and a temperature reading should not be an integer.

### Task 1.1 -- Inspect type sizes

Replace the contents of `src/main.rs` with the following:

```rust
fn main() {
    println!("Integer types:");
    println!("  i8:    {} bytes, range {} to {}", std::mem::size_of::<i8>(), i8::MIN, i8::MAX);
    println!("  i32:   {} bytes, range {} to {}", std::mem::size_of::<i32>(), i32::MIN, i32::MAX);
    println!("  i64:   {} bytes, range {} to {}", std::mem::size_of::<i64>(), i64::MIN, i64::MAX);
    println!("  u8:    {} bytes, range {} to {}", std::mem::size_of::<u8>(), u8::MIN, u8::MAX);
    println!("  usize: {} bytes (architecture-dependent)", std::mem::size_of::<usize>());

    println!("\nFloat types:");
    println!("  f32: {} bytes", std::mem::size_of::<f32>());
    println!("  f64: {} bytes", std::mem::size_of::<f64>());

    println!("\nOther scalar types:");
    println!("  bool: {} bytes", std::mem::size_of::<bool>());
    println!("  char: {} bytes", std::mem::size_of::<char>());
}
```

Run the program:

```bash
cargo run
```

Record the output in `lab2-notes.md`.

### Task 1.2 -- Numeric literals

Add the following to `src/main.rs`, inside `main`, after the existing code:

```rust
    println!("\nNumeric literals:");
    let decimal = 98_222;
    let hex = 0xff;
    let octal = 0o77;
    let binary = 0b1111_0000;
    let byte = b'A';

    println!("  decimal: {decimal}");
    println!("  hex:     {hex}");
    println!("  octal:   {octal}");
    println!("  binary:  {binary}");
    println!("  byte:    {byte}");
```

Run and confirm the output. Note that all four numeric literals print as their decimal value. The literal form is a writing convenience; the underlying value is the same.

In `lab2-notes.md`, answer:

1. The byte literal `b'A'` printed as `65`. Why?
2. The literal `1_000_000` and `1000000` produce identical output. Why does Rust accept the underscores?

### Task 1.3 -- Choosing types deliberately

Now begin building the temperature tracker. Replace the contents of `src/main.rs` with:

```rust
fn main() {
    let temperature_celsius: f64 = 22.5;
    let humidity_percent: u8 = 65;
    let sensor_id: u32 = 1024;
    let sensor_active: bool = true;
    let unit_symbol: char = 'C';

    println!("Sensor {sensor_id} is active: {sensor_active}");
    println!("Reading: {temperature_celsius}{unit_symbol}, humidity {humidity_percent}%");
}
```

Each variable uses an explicit type annotation. The choices are deliberate:

- `f64` for temperature: floating-point because temperatures are not integers.
- `u8` for humidity: humidity is between 0 and 100, so a single byte is more than enough.
- `u32` for sensor ID: large enough to identify millions of sensors, signed not needed because IDs are not negative.
- `bool` for status: the value is binary, true or false.
- `char` for unit: a single Unicode character.

Run the program. Confirm the output:

```
Sensor 1024 is active: true
Reading: 22.5C, humidity 65%
```

### Checkpoints

1. The default integer type is `i32` and the default float is `f64`. Both occupy 4 bytes for the integer and 8 for the float. Explain why the language chose different defaults rather than picking the same width for both.
2. You declared `humidity_percent: u8`. What happens at runtime if a sensor reports a humidity reading of 250? Why might that still be acceptable for this application?
3. The `char` type is 4 bytes wide, even though most characters used in a typical program fit in 1 byte. What does this tell you about Rust's design priorities?

---

## Exercise 2 -- Type Inference and Annotations

**Estimated time:** 10-15 minutes
**Topics covered:** inference, explicit annotations, the turbofish, parsing failures

### Context

Type inference is convenient but not magic. The compiler can only infer a type when it has enough information. This exercise walks through the common cases where inference works, where it fails, and how to fix the failures.

### Task 2.1 -- Inference that works

Replace the contents of `main` with:

```rust
fn main() {
    let temperature = 22.5;
    let count = 5;
    let label = "Sensor A";
    let active = true;
    let readings = vec![18.5, 20.1, 22.5, 21.3];

    println!("Temp:     {temperature} ({})", type_of(&temperature));
    println!("Count:    {count} ({})", type_of(&count));
    println!("Label:    {label} ({})", type_of(&label));
    println!("Active:   {active} ({})", type_of(&active));
    println!("Readings: {readings:?} ({})", type_of(&readings));
}

fn type_of<T>(_: &T) -> &'static str {
    std::any::type_name::<T>()
}
```

Run the program and observe the inferred types. Each variable's type is determined entirely from the right-hand side of the assignment.

> **Note:** The `type_of` helper uses a feature you have not seen yet (generics). For now, treat it as a black box that prints the type name of whatever you pass to it.

### Task 2.2 -- Inference that fails

Now add the following lines just before the `println!` calls and try to compile:

```rust
    let parsed = "42".parse();
    println!("Parsed: {parsed:?}");
```

Run `cargo build`. The compiler produces an error similar to:

```
error[E0282]: type annotations needed
  |
6 |     let parsed = "42".parse();
  |         ^^^^^^
  |
help: consider giving `parsed` an explicit type
  |
6 |     let parsed: /* Type */ = "42".parse();
  |               ++++++++++++
```

The `str::parse` method is generic. Without a target type, the compiler does not know whether you want to parse the string into an `i32`, an `f64`, a `bool`, or any of dozens of other parseable types.

In `lab2-notes.md`, copy the error message. Notice how rust-analyzer surfaces this error in the editor before you even try to build.

### Task 2.3 -- Three ways to fix it

There are three idiomatic fixes for this error. Try all three.

**Fix 1: Annotate the variable**

```rust
    let parsed: Result<i32, _> = "42".parse();
    println!("Parsed: {parsed:?}");
```

**Fix 2: Turbofish on the method**

Remove the previous fix and try:

```rust
    let parsed = "42".parse::<i32>();
    println!("Parsed: {parsed:?}");
```

**Fix 3: Use the result in a typed context**

Remove the previous fix and try:

```rust
    let parsed = "42".parse();
    let value: i32 = parsed.unwrap();
    println!("Parsed value: {value}");
```

This third form is interesting: the annotation appears on the next line, but the compiler propagates the type information backward to the parse call.

### Task 2.4 -- Empty collections

Add the following inside `main`:

```rust
    let history = Vec::new();
    println!("History: {history:?}");
```

Run `cargo build`. You will see another inference failure. The compiler cannot guess what kind of `Vec` you want.

Fix it by annotating the type:

```rust
    let history: Vec<f64> = Vec::new();
    println!("History: {history:?}");
```

Then try the turbofish form:

```rust
    let history = Vec::<f64>::new();
    println!("History: {history:?}");
```

Both compile. Both produce the same `Vec<f64>`. Choose whichever reads better in context.

### Checkpoints

1. The compiler accepted `let temperature = 22.5;` with no annotation but rejected `let parsed = "42".parse();`. Both lines produce a single binding from a single expression. What is different about the two cases that makes inference work for one and fail for the other?
2. The error message for the empty `Vec::new()` case suggests "consider giving the variable an explicit type." Why does the compiler suggest annotating the variable rather than annotating the call to `Vec::new()`?
3. Some Rust style guides recommend annotating local variables only when inference fails. Other guides recommend annotating every public function parameter. Why does the recommendation differ between the two cases?

---

## Exercise 3 -- Immutability and `mut`

**Estimated time:** 10 minutes
**Topics covered:** immutable bindings by default, the `mut` keyword, when to use each

### Context

Variables in Rust are immutable by default. The compiler refuses to let you reassign them. To allow reassignment, you must explicitly opt in with `mut`. This exercise demonstrates the rule and the error messages, then asks you to apply judgment about which bindings should be mutable.

### Task 3.1 -- The default behavior

Replace `main` with the following:

```rust
fn main() {
    let temperature = 22.5;
    println!("Initial: {temperature}");

    temperature = 23.7;
    println!("Updated: {temperature}");
}
```

Run `cargo build`. The compiler produces an error similar to:

```
error[E0384]: cannot assign twice to immutable variable `temperature`
  |
2 |     let temperature = 22.5;
  |         ----------- first assignment to `temperature`
  |     ...
5 |     temperature = 23.7;
  |     ^^^^^^^^^^^^^^^^^^^ cannot assign twice to immutable variable
  |
help: consider making this binding mutable
  |
2 |     let mut temperature = 22.5;
  |         +++
```

Note that the error message:

- Identifies the original binding.
- Identifies the offending reassignment.
- Suggests the exact fix.

This is the typical level of detail in Rust compiler errors. Read every diagnostic carefully; it almost always tells you what to do.

### Task 3.2 -- Apply the fix

Make the binding mutable:

```rust
fn main() {
    let mut temperature = 22.5;
    println!("Initial: {temperature}");

    temperature = 23.7;
    println!("Updated: {temperature}");
}
```

Run the program. It now produces:

```
Initial: 22.5
Updated: 23.7
```

### Task 3.3 -- Apply judgment

Replace `main` with this version, which simulates collecting a series of temperature readings and computing statistics:

```rust
fn main() {
    let sensor_id = 42;
    let max_readings = 5;
    let mut total = 0.0_f64;
    let mut count = 0;

    let readings = [21.5, 22.0, 22.8, 23.1, 22.5];

    for reading in readings {
        total += reading;
        count += 1;
    }

    let average = total / count as f64;

    println!("Sensor {sensor_id}: {count} of {max_readings} readings");
    println!("Total:   {total}");
    println!("Average: {average:.2}");
}
```

Read the code carefully. In `lab2-notes.md`, answer:

1. Which bindings are mutable? Why is each one mutable?
2. Which bindings are immutable? Why does each one not need to be mutable?
3. The `average` binding could have been declared as `mut`. Why is it more idiomatic to leave it immutable?

### Checkpoints

1. The compiler error message for an immutable reassignment offered an exact fix. What does this design choice reveal about how the Rust team thinks about developer experience?
2. Some languages make all variables mutable by default and require an explicit `final` or `const` keyword for immutability. Identify two specific code patterns that become more error-prone under that design.
3. Could the program in Task 3.3 be rewritten with no `mut` keywords at all? Sketch how you might do it. (You do not need to write working code; describe the approach.)

---

## Exercise 4 -- Constants and Statics

**Estimated time:** 10-15 minutes
**Topics covered:** `const`, `static`, naming conventions, the difference between them

### Context

`const` and `static` both create program-wide values. They differ in subtle ways that matter occasionally. This exercise teaches the rules and shows when each is the right choice.

### Task 4.1 -- Declare constants

Replace `main.rs` with:

```rust
const FREEZING_POINT_CELSIUS: f64 = 0.0;
const BOILING_POINT_CELSIUS: f64 = 100.0;
const ABSOLUTE_ZERO_CELSIUS: f64 = -273.15;
const MAX_SENSOR_READINGS: usize = 1000;
const TEMPERATURE_UNIT: char = 'C';

fn main() {
    println!("Freezing point:  {FREEZING_POINT_CELSIUS}{TEMPERATURE_UNIT}");
    println!("Boiling point:   {BOILING_POINT_CELSIUS}{TEMPERATURE_UNIT}");
    println!("Absolute zero:   {ABSOLUTE_ZERO_CELSIUS}{TEMPERATURE_UNIT}");
    println!("Max readings:    {MAX_SENSOR_READINGS}");
}
```

Run the program. Note several things:

- Constants are declared at module scope, outside any function.
- The naming convention is `SCREAMING_SNAKE_CASE`.
- Type annotations are required, even though the values look obvious.
- The values are inlined at every use site. There is no `FREEZING_POINT_CELSIUS` variable in the compiled binary, just the literal `0.0` everywhere it appears.

### Task 4.2 -- Try to break the rules

Add the following constant to your file:

```rust
const CURRENT_TIME: u64 = std::time::SystemTime::now()
    .duration_since(std::time::UNIX_EPOCH)
    .unwrap()
    .as_secs();
```

Run `cargo build`. The compiler rejects this with an error indicating that the value cannot be computed at compile time. `const` values must be constant expressions: literals, simple arithmetic, and a small set of functions explicitly marked as compile-time-evaluable.

Remove the broken constant before continuing.

### Task 4.3 -- Function-local constants

`const` does not have to be at module scope. Add this function below `main`:

```rust
fn celsius_to_fahrenheit(celsius: f64) -> f64 {
    const SCALE: f64 = 9.0 / 5.0;
    const OFFSET: f64 = 32.0;
    celsius * SCALE + OFFSET
}
```

Then call it from `main`:

```rust
    let body_temp_c = 37.0;
    let body_temp_f = celsius_to_fahrenheit(body_temp_c);
    println!("Body temperature: {body_temp_c}{TEMPERATURE_UNIT} = {body_temp_f}F");
```

Run the program. The function-scoped constants `SCALE` and `OFFSET` are visible only within `celsius_to_fahrenheit`. Outside the function, the names do not exist. This is useful for documenting magic numbers without exposing them to the rest of the codebase.

### Task 4.4 -- A static variable

Add the following at module scope, near the constants:

```rust
static APPLICATION_NAME: &str = "TemperatureTracker";
static VERSION: &str = "0.1.0";
```

In `main`, add:

```rust
    println!("\n{APPLICATION_NAME} v{VERSION}");
```

Run the program. Both `static` and `const` look identical at the use site, but they differ in how the compiler handles them.

In `lab2-notes.md`, write down:

1. Which of these values are best as `const`? Which are best as `static`? (Hint: think about whether anything would care about a memory address.)
2. The `MAX_SENSOR_READINGS` constant is used to allocate a fixed-size array later in this lab. Would `static` work in that role? Why or why not?

### Task 4.5 -- The forbidden form

Try adding this at module scope:

```rust
static mut CURRENT_READING: f64 = 0.0;
```

This compiles. Now try to use it from `main`:

```rust
    CURRENT_READING = 21.5;
    println!("Current: {CURRENT_READING}");
```

The compiler rejects this with:

```
error[E0133]: use of mutable static is unsafe and requires unsafe function or block
```

`static mut` exists but is `unsafe` to access, because the compiler cannot guarantee that two threads will not write to it simultaneously. In modern idiomatic Rust, almost no code uses `static mut`. The replacements (`OnceLock`, `Mutex`, `AtomicU32`) are covered in the concurrency module.

Remove the `static mut` and the assignment before continuing.

### Checkpoints

1. The constant `MAX_SENSOR_READINGS` is `usize`, not `u32` or `i32`. Why is `usize` the right choice?
2. The error in Task 4.2 said the value "cannot be computed at compile time." But `std::time::SystemTime::now()` is a perfectly valid Rust function. Why can the compiler call some functions during compilation and not others?
3. The Rust standard library has `std::f64::consts::PI` defined as a `const`, not a `static`. Why is `const` the right choice for a mathematical constant?

---

## Exercise 5 -- Shadowing

**Estimated time:** 10-15 minutes
**Topics covered:** shadowing, type-changing pipelines, block-scoped shadows

### Context

Rust allows you to declare a new variable with the same name as an existing one. This shadows the previous binding without modifying it. Shadowing is the idiomatic way to express type transformations and short value pipelines.

### Task 5.1 -- Shadowing across types

A common pattern is reading a string from input and converting it to a number. Replace `main` with:

```rust
fn main() {
    let raw_reading = "  22.5  ";
    println!("Raw input:        {raw_reading:?} (type: &str)");

    let raw_reading = raw_reading.trim();
    println!("After trim:       {raw_reading:?} (type: &str, but trimmed)");

    let raw_reading: f64 = raw_reading.parse().expect("not a number");
    println!("After parse:      {raw_reading} (type: f64)");

    let raw_reading = raw_reading * 1.8 + 32.0;
    println!("After conversion: {raw_reading} (type: f64, in Fahrenheit)");
}
```

Run the program. Each `let raw_reading = ...` creates a new binding. The type can change between bindings: from `&str` to `&str` to `f64` to `f64`.

This is shadowing, not mutation. The earlier `&str` binding is not modified. It is hidden by the new `f64` binding, but the underlying string still exists in memory. The compiler cleans it up when it is no longer reachable.

### Task 5.2 -- Why mutation does not work here

Try rewriting Task 5.1 with mutation. Replace `main` with:

```rust
fn main() {
    let mut raw_reading = "  22.5  ";
    println!("Raw input:  {raw_reading:?}");

    raw_reading = raw_reading.trim();
    println!("After trim: {raw_reading:?}");

    raw_reading = raw_reading.parse().expect("not a number");
    println!("After parse: {raw_reading}");
}
```

Run `cargo build`. The compiler rejects the third assignment:

```
error[E0308]: mismatched types
  |
8 |     raw_reading = raw_reading.parse().expect("not a number");
  |                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `&str`, found `f64`
```

A `mut` binding can change its value but not its type. Once `raw_reading` is declared as `&str`, every subsequent assignment must produce a `&str`. The shadowing version succeeded precisely because each `let` was a new binding free to choose its own type.

Restore the working version from Task 5.1 before continuing.

### Task 5.3 -- Block-scoped shadowing

Shadowing inside a block is limited to that block. Replace `main` with:

```rust
fn main() {
    let temperature = 22.5;
    println!("Outer (before): {temperature}");

    {
        let temperature = "twenty-two point five";
        println!("Inner: {temperature}");
    }

    println!("Outer (after):  {temperature}");
}
```

Run the program:

```
Outer (before): 22.5
Inner: twenty-two point five
Outer (after):  22.5
```

The inner `temperature` (a `&str`) shadowed the outer `temperature` (an `f64`) only within the block. After the block ends, the inner binding goes out of scope and the outer binding becomes visible again.

### Task 5.4 -- A practical pipeline

Now apply shadowing to a realistic transformation. Replace `main` with:

```rust
fn main() {
    let raw_data = "21.5,22.0,22.8,23.1,22.5";

    let readings: Vec<&str> = raw_data.split(',').collect();
    let readings: Vec<f64> = readings
        .iter()
        .map(|s| s.parse().expect("invalid number"))
        .collect();

    let total: f64 = readings.iter().sum();
    let average = total / readings.len() as f64;

    println!("Readings: {readings:?}");
    println!("Average:  {average:.2}");
}
```

The `readings` name is used twice: first as `Vec<&str>` (the split parts), then as `Vec<f64>` (the parsed numbers). Each `let` is a new binding. The first `readings` becomes unreachable after the second `let`, which is exactly what we want: we no longer have any reason to access the unparsed string slices.

Run the program and confirm the output.

### Checkpoints

1. In Task 5.1, the binding `raw_reading` was used four times with three different types. Why does this not violate Rust's static type discipline?
2. Task 5.2 showed that `mut` cannot change a binding's type. Why was this a deliberate language design choice rather than a limitation?
3. Some style guides discourage shadowing because it can confuse readers. The Rust community generally accepts it. Identify one situation where you would use shadowing, and one where you would deliberately use distinct names instead.

---

## Exercise 6 -- Tuples

**Estimated time:** 10 minutes
**Topics covered:** tuple construction, destructuring, indexing, returning multiple values

### Context

Tuples group a fixed number of values, possibly of different types, into one compound value. They are useful for short groupings, particularly when returning multiple values from a function. When the grouping has more than three or four fields, prefer a `struct` (covered in a later module).

### Task 6.1 -- Build and access tuples

Replace `main` with:

```rust
fn main() {
    let reading: (u32, f64, char, bool) = (1024, 22.5, 'C', true);

    let sensor_id = reading.0;
    let temperature = reading.1;
    let unit = reading.2;
    let is_active = reading.3;

    println!("Sensor {sensor_id}: {temperature}{unit}, active: {is_active}");
}
```

Run the program. Note the field-access syntax: `reading.0`, `reading.1`, and so on. The numbers are field positions, not indices into an array; you cannot compute them at runtime.

### Task 6.2 -- Destructure with patterns

The same data is more idiomatically extracted with destructuring. Replace `main` with:

```rust
fn main() {
    let reading: (u32, f64, char, bool) = (1024, 22.5, 'C', true);

    let (sensor_id, temperature, unit, is_active) = reading;

    println!("Sensor {sensor_id}: {temperature}{unit}, active: {is_active}");
}
```

The result is the same. The destructuring pattern says "this tuple has four fields; bind them to these four names." This is the form you should use whenever you want all the fields.

When you only want some fields, use `_` to ignore the rest:

```rust
    let (_, temperature, unit, _) = reading;
    println!("{temperature}{unit}");
```

### Task 6.3 -- Return multiple values

Tuples are commonly used to return more than one value from a function. Add this function to your file:

```rust
fn analyze_readings(values: &[f64]) -> (f64, f64, f64) {
    let min = values.iter().copied().fold(f64::INFINITY, f64::min);
    let max = values.iter().copied().fold(f64::NEG_INFINITY, f64::max);
    let sum: f64 = values.iter().sum();
    let avg = sum / values.len() as f64;
    (min, max, avg)
}
```

Use it from `main`:

```rust
fn main() {
    let readings = [21.5, 22.0, 22.8, 23.1, 22.5];
    let (min, max, avg) = analyze_readings(&readings);

    println!("Readings: {readings:?}");
    println!("Min: {min}, Max: {max}, Avg: {avg:.2}");
}
```

Run the program and confirm the output.

### Task 6.4 -- The unit type

Replace `main` with:

```rust
fn main() {
    let nothing: () = ();
    println!("Unit value: {nothing:?}");

    let result = print_and_return("Hello");
    println!("Return value: {result:?}");
}

fn print_and_return(message: &str) {
    println!("{message}");
}
```

Run the program. The output is:

```
Unit value: ()
Hello
Return value: ()
```

The `()` type, called the unit type, is the implicit return type of any function that does not return anything else. It has exactly one value, also written `()`. This is why you can write `let result = print_and_return(...)` and the binding has a meaningful (if empty) value.

### Checkpoints

1. Tuples can be indexed with `tuple.0`, `tuple.1`, and so on. They cannot be indexed with a variable: `tuple.i` is a compile error. Explain why this restriction exists.
2. The `analyze_readings` function returned a `(f64, f64, f64)`. A reader who sees this signature does not immediately know which value is which. When would you choose to return a tuple here, and when would you switch to a `struct`?
3. In Task 6.4, `print_and_return` had no explicit return type, but the binding `result` still received a value. What does this tell you about how Rust treats functions that do not return anything?

---

## Exercise 7 -- Arrays and Slices

**Estimated time:** 15-20 minutes
**Topics covered:** arrays, fixed sizes, indexing, bounds checks, slices

### Context

An array is a fixed-size, same-type sequence of values. The size is part of the type and is known at compile time. Arrays are stored on the stack with no heap allocation. Slices are references to contiguous ranges of arrays (or `Vec`s) and are the type used in most function signatures that work with sequences.

### Task 7.1 -- Build and access arrays

Replace `main` with:

```rust
fn main() {
    let readings: [f64; 5] = [21.5, 22.0, 22.8, 23.1, 22.5];
    let zeros = [0.0_f64; 24];
    let labels = ["morning", "afternoon", "evening"];

    println!("First reading:  {}", readings[0]);
    println!("Third reading:  {}", readings[2]);
    println!("Number of zeros: {}", zeros.len());
    println!("First label:    {}", labels[0]);
}
```

Run the program. Three things to note:

- `[f64; 5]` is the type. Five 64-bit floats.
- `[0.0_f64; 24]` is the "fill" syntax. It produces an array of 24 zeros.
- `len()` returns the array's length, which is always known at compile time.

### Task 7.2 -- Bounds are checked

Append this to `main`:

```rust
    let bad = readings[10];
    println!("Bad: {bad}");
```

Run the program. It panics with:

```
thread 'main' panicked at src/main.rs:..., index out of bounds: the len is 5 but the index is 10
```

This is one of Rust's central safety guarantees. An out-of-bounds array access never reads invalid memory; it panics. In C, the same access would produce a memory safety violation, often silently. Bounds checks are inserted by the compiler. The optimizer eliminates checks it can prove unnecessary, so the runtime cost is usually zero.

Remove the panicking lines before continuing.

### Task 7.3 -- Iterate without indexing

Replace `main` with:

```rust
fn main() {
    let readings: [f64; 5] = [21.5, 22.0, 22.8, 23.1, 22.5];

    println!("Iterate by value:");
    for r in readings {
        println!("  {r}");
    }

    println!("Iterate with index:");
    for (i, r) in readings.iter().enumerate() {
        println!("  [{i}] = {r}");
    }

    let sum: f64 = readings.iter().sum();
    let max = readings.iter().copied().fold(f64::NEG_INFINITY, f64::max);

    println!("Sum: {sum}, Max: {max}");
}
```

Run the program. Note that:

- `for r in readings` iterates by value. Each `r` is a fresh `f64`.
- `enumerate()` produces tuples of `(index, value)`.
- `sum()` and the various fold-based operations work on iterators.

You should almost never write `for i in 0..readings.len()`. It is more verbose, harder to read, and easier to misuse. clippy will flag it, as you saw in Lab 1.

### Task 7.4 -- Slices in function signatures

Add this function:

```rust
fn average(values: &[f64]) -> f64 {
    let sum: f64 = values.iter().sum();
    sum / values.len() as f64
}
```

Now use it from `main` in three different ways:

```rust
fn main() {
    let array_readings: [f64; 5] = [21.5, 22.0, 22.8, 23.1, 22.5];
    let vec_readings: Vec<f64> = vec![18.0, 19.5, 20.0, 21.5, 22.0, 22.5];

    let avg_array = average(&array_readings);
    let avg_vec = average(&vec_readings);
    let avg_partial = average(&array_readings[1..4]);

    println!("Array average:    {avg_array:.2}");
    println!("Vec average:      {avg_vec:.2}");
    println!("Partial average:  {avg_partial:.2}");
}
```

Run the program. The same `average` function works on:

- A fixed-size array.
- A heap-allocated `Vec`.
- A subrange of either.

This is why `&[T]` is the idiomatic parameter type for sequence-accepting functions. It is the most flexible signature: it accepts every sequence-shaped thing.

### Task 7.5 -- An array does not grow

Add this to `main`:

```rust
    let mut more_readings: [f64; 3] = [10.0, 20.0, 30.0];
    more_readings[0] = 11.0;
    println!("Modified: {more_readings:?}");
```

This works: you can mutate elements of a `mut` array. But try to add a new element:

```rust
    more_readings.push(40.0);
```

The compiler rejects this:

```
error[E0599]: no method named `push` found for array `[f64; 3]`
```

Arrays have a fixed size that is part of the type. To grow a sequence at runtime, use `Vec`. Remove the `push` line before continuing.

### Checkpoints

1. The type `[f64; 5]` and `[f64; 6]` are different types. A function declared as `fn process(arr: [f64; 5])` cannot accept a 6-element array. Why does Rust make the size part of the type rather than treating arrays as variable-length?
2. The function `average` takes `&[f64]` rather than `[f64; 5]` or `Vec<f64>`. What does the choice of slice signature gain over the alternatives?
3. The runtime panic on out-of-bounds access is checked at runtime, not at compile time. Identify one case where the compiler can statically prove that an index is in bounds and skip the runtime check, and one case where it cannot.

---

## Exercise 8 -- Putting It Together

**Estimated time:** 10 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from the module into one small program. There are no new ideas; the goal is to read working code that uses scalar types deliberately, applies inference appropriately, picks the right kind of constant, uses shadowing for transformations, and handles compound types idiomatically.

### Task 8.1 -- The full program

Replace `main.rs` with:

```rust
const FREEZING_POINT_C: f64 = 0.0;
const BOILING_POINT_C: f64 = 100.0;

fn celsius_to_fahrenheit(c: f64) -> f64 {
    c * 9.0 / 5.0 + 32.0
}

fn analyze(values: &[f64]) -> (f64, f64, f64) {
    let min = values.iter().copied().fold(f64::INFINITY, f64::min);
    let max = values.iter().copied().fold(f64::NEG_INFINITY, f64::max);
    let avg: f64 = values.iter().sum::<f64>() / values.len() as f64;
    (min, max, avg)
}

fn main() {
    let sensor_id: u32 = 1024;
    let raw_data = "21.5,22.0,22.8,23.1,22.5,21.8,22.3";

    let readings: Vec<f64> = raw_data
        .split(',')
        .map(|s| s.parse().expect("invalid number"))
        .collect();

    let (min, max, avg) = analyze(&readings);

    let avg_f = celsius_to_fahrenheit(avg);

    println!("Sensor {sensor_id}");
    println!("  Readings: {readings:?}");
    println!("  Range:    {min} to {max} C");
    println!("  Average:  {avg:.2} C ({avg_f:.2} F)");
    println!("  Within boiling/freezing range: {}",
        avg > FREEZING_POINT_C && avg < BOILING_POINT_C);
}
```

Run the program. Expected output:

```
Sensor 1024
  Readings: [21.5, 22.0, 22.8, 23.1, 22.5, 21.8, 22.3]
  Range:    21.5 to 23.1 C
  Average:  22.29 C (72.11 F)
  Within boiling/freezing range: true
```

### Task 8.2 -- Identify the patterns

In `lab2-notes.md`, identify each instance of the following patterns in the program above:

1. A scalar type chosen deliberately for the data it represents.
2. A type inference success and a place where annotation was required anyway.
3. An immutable binding that could have been mutable but was kept immutable for a reason.
4. A constant that could not be a `let` binding because of where it is declared.
5. A shadowing operation that changes the value's type (look in the `readings` chain).
6. A tuple used to return multiple values from a function.
7. An array or slice used as a function parameter.

This kind of close reading is how you internalize idiomatic Rust. The reflexes that experienced developers apply unconsciously are explicit choices in this program. Once you see them named here, you will see them in all the code you read after.

### Checkpoints

1. The `analyze` function returns `(f64, f64, f64)`. Read its body. Could two of those three values be the same type but reasonably mean very different things, leading to a hard-to-find bug if the caller mixed them up? How would you redesign the return type to prevent this?
2. The `readings` binding is created with type `Vec<f64>` and is immutable. The function `analyze` reads from it but does not modify it. What would change in the program if `readings` were declared as `&[f64]` instead of `Vec<f64>`?
3. The function `celsius_to_fahrenheit` does not declare its types explicitly inside the body. The signature says `f64 -> f64`. Why is the signature considered the documented contract, while the body's types are considered an implementation detail?

---

## Summary and Reflection

You have now used every type system feature from Module 3 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Scalar Types | Integers, floats, booleans, characters | Type widths are explicit. Choose the type that matches the data, not the type that is easiest to type. |
| 2 -- Type Inference | Annotations, turbofish, parsing | Inference works when the right side determines the type. It fails on generic results and empty collections. |
| 3 -- Immutability | `let` vs `let mut` | The default of immutability is a tool for reasoning. `mut` is reserved for state that genuinely changes. |
| 4 -- Constants | `const`, `static`, scope | Use `const` for almost all program-wide values. `static` is reserved for cases that need a stable address. |
| 5 -- Shadowing | New bindings with the same name | Shadowing is the idiomatic way to express type-changing pipelines. It is not mutation. |
| 6 -- Tuples | Heterogeneous compound values | Tuples are good for short groupings and multi-value returns. Beyond three fields, use `struct`. |
| 7 -- Arrays and Slices | Fixed-size sequences | Arrays are stack-allocated and have their size in the type. Slices are the flexible parameter type for sequence functions. |
| 8 -- Integration | All of the above | Idiomatic Rust applies several type system features together. Reading code closely is how you learn the patterns. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab2-notes.md` before your next session.

1. Of the type system features in this module, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. Compare Rust's approach to integer types (`i8`, `i16`, `i32`, `i64`, `i128`, `isize`, plus the unsigned variants) with another language you know well. What does Rust gain by being explicit about width, and what does it cost? Would you rather work in a language with one `int` type, or in Rust's model?

3. The lab exercises asked you to apply judgment several times: which bindings should be mutable, when shadowing is clearer than mutation, when to use `const` versus `static`. Identify one case from this lab where you initially picked one approach and then changed your mind after considering the alternative. What changed your decision?

---

*End of Lab 2*
