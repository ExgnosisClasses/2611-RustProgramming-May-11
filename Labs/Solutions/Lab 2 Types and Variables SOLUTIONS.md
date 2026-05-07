# Module 3: Rust Types and Variables
## Lab 2 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Scalar Types and Numeric Literals

### Task 1.2 -- Numeric literal answers

**1. The byte literal `b'A'` printed as `65`.**

A byte literal is a `u8` whose value is the ASCII code of the character. The capital letter `A` has ASCII code 65. The literal syntax is shorthand for the integer value at that codepoint. Byte literals only work for ASCII characters because their values must fit in a single byte; `b'中'` would be a compile error.

**2. The literal `1_000_000` and `1000000` produce identical output.**

The underscores are purely visual separators that the lexer discards before the number is interpreted. They exist to make long numbers readable. The same applies to `0xff_ff_ff` (hex) and `0b1111_0000` (binary), where the underscores group nibbles or bits naturally.

### Checkpoints

**1. Why does Rust default to `i32` for integers and `f64` for floats?**

The defaults reflect what each type is good at. `i32` is large enough for typical counts, indices, and small computations, while being efficient on every modern architecture. `f64` is the standard floating-point type for nearly all applications: scientific computation, financial calculations, graphics, and most general numeric work. On modern CPUs, `f64` is roughly the same speed as `f32` but has dramatically more precision.

The asymmetry is a recognition that integer use cases more often involve small values where 32 bits is plenty, while floating-point use cases more often need the precision that `f64` provides. Choosing 4 bytes for the integer default keeps memory usage reasonable in arrays of integers; choosing 8 bytes for the float default avoids subtle precision bugs that would come from defaulting to `f32`.

**2. What happens if a sensor reports humidity 250 to a `u8`?**

The value 250 fits in a `u8` (range 0 to 255), so storage is fine. The application-level problem is that 250 is not a valid humidity reading; humidity is a percentage and cannot exceed 100. The type prevents memory safety violations but cannot enforce domain validity.

This is acceptable for the lab because the goal is to model the storage, not the validation. In production code, you would either validate values at the boundary (rejecting bad sensor readings) or use a richer type that encodes the constraint, such as a custom `Humidity` type with a constructor that panics or returns `Result<Humidity, Error>` on invalid input. The type system can carry domain rules if you build types to enforce them.

**3. Why is `char` four bytes wide?**

A four-byte `char` can hold any Unicode scalar value. This includes characters in the supplementary planes (emoji, rare scripts, mathematical symbols) that require surrogate pairs in UTF-16 systems like Java. Rust's choice means there is no concept of "high surrogate" or "low surrogate" characters; every `char` represents one complete Unicode scalar value.

The cost is memory: a four-byte `char` uses four times the space of a one-byte ASCII character. The benefit is correctness and simplicity: there is one kind of `char`, the standard library never exposes half-characters, and indexing operations have predictable semantics.

This priority (correctness over memory efficiency) is consistent across Rust's design. Strings store UTF-8 (variable width) for memory efficiency, but the `char` type itself is always full width.

---

## Exercise 2 -- Type Inference and Annotations

### Checkpoints

**1. Why does inference work for `let temperature = 22.5` but fail for `let parsed = "42".parse()`?**

The right side of `let temperature = 22.5` has exactly one possible type: `f64`. There is no ambiguity, so the compiler can infer.

The right side of `let parsed = "42".parse()` calls a generic function. `parse` is declared as `fn parse<F: FromStr>(...) -> Result<F, F::Err>`. The return type depends on which `F` you want, and many types implement `FromStr`: `i32`, `i64`, `f32`, `f64`, `bool`, `IpAddr`, `SocketAddr`, and many more. The compiler refuses to guess which one you meant.

This is a deliberate choice. If the compiler picked `i32` by default, programs that meant to parse `f64` would silently lose precision and the bug would only surface when the values mattered. By forcing the developer to disambiguate, Rust catches the mistake at the point of declaration.

**2. Why does the empty Vec error suggest annotating the variable?**

There are two valid places to put the annotation: on the variable (`let history: Vec<f64> = Vec::new()`) or on the call (`Vec::<f64>::new()`). The compiler suggests the variable form because:

- Variable annotations are more visually prominent and easier to read.
- Variable annotations are familiar to developers from other typed languages.
- The turbofish syntax is unfamiliar to newcomers and looks unusual.

In real code, both forms are common. The turbofish is more concise when the function name itself is short, and is preferred when the variable name is long or when chaining methods that already produce a typed result.

**3. Why annotate function parameters always but local variables only sometimes?**

Function parameters are part of the public contract. Even if a function is private to a module, its signature is the documentation for everyone who reads or calls it. A reader should understand a function's interface without studying its body. Type annotations on parameters and return values are how that contract is expressed.

Local variables are an implementation detail. The reader is already inside the function, has access to the surrounding context, and can usually infer types from the right-hand side of assignments. Annotating every local variable adds visual noise without adding information. Annotations on locals are valuable only when the inferred type is non-obvious or when the developer wants to lock in a specific type for documentation or correctness.

The principle is "explicit at boundaries, inferred within." It minimizes ceremony in implementation code while keeping interfaces clear.

---

## Exercise 3 -- Immutability and `mut`

### Task 3.3 -- Analysis of bindings

**1. Which bindings are mutable, and why?**

`total` and `count` are mutable. They start at zero and accumulate during the loop. Without `mut`, the `+=` operations would not compile.

**2. Which bindings are immutable, and why?**

- `sensor_id`, `max_readings`: these are configuration values that should not change.
- `readings`: the data being processed. The loop reads from it but does not write to it.
- `average`: computed once after the loop. There is no reason to change it.

**3. Why leave `average` immutable rather than declaring it `mut`?**

Declaring `let average = ...` rather than `let mut average = ...` communicates intent. A reader sees `let average` and knows the value is computed once and never modified. If the binding were `let mut average`, the reader would expect to find later modifications and would have to scan the rest of the function looking for them. Using `let` correctly says "this is the final value; do not look for more assignments."

This is the everyday payoff of immutability-by-default: the absence of `mut` is meaningful information.

### Checkpoints

**1. What does the compiler error message reveal about Rust's design priorities?**

The message names the variable, points to the original binding, points to the offending reassignment, and offers an exact fix. Compare this to error messages in some other languages, which might just say "syntax error" or "cannot assign to this expression."

The level of detail reflects a design choice: the Rust team treats compiler errors as a primary teaching tool. New developers learn the language partly by making mistakes and reading the diagnostics. Investing engineering effort in good error messages pays off in faster onboarding and fewer abandoned learning attempts. This is also why `rustc` includes `--explain` for error codes and why many error messages include links to relevant chapters of the Rust Book.

**2. What patterns become more error-prone if mutability is the default?**

Two clear examples:

- Accidentally mutating a value passed to a function. If a function receives a binding and mutability is the default, the function might modify it without the caller realizing. With Rust's defaults, the caller can see at the call site whether mutation is permitted (via `&mut`, covered later) and the function signature has to declare it explicitly.
- Treating a configuration value as state. If `max_users = 100` is mutable by default, code anywhere in the program might overwrite it. With Rust's defaults, that overwrite would not compile, forcing the developer to either declare `mut` (acknowledging the intent) or rethink the design.

The general principle is that defaults should encourage the safer choice. Most variables do not need to change. Making mutation opt-in keeps the unintentional cases from compiling.

**3. Could the program be rewritten with no `mut` keywords?**

Yes, by replacing the imperative loop with iterator combinators. The total and count would be computed in a single expression using `iter().sum()` and `iter().count()` (or by using the array's `len()`):

```rust
let total: f64 = readings.iter().sum();
let count = readings.len();
let average = total / count as f64;
```

There are no mutable bindings because each value is computed in one expression rather than accumulated step by step. This is the more idiomatic Rust style for this kind of computation.

---

## Exercise 4 -- Constants and Statics

### Task 4.4 -- Const vs static answers

**1. Which values should be `const`, which should be `static`?**

- `FREEZING_POINT_CELSIUS`, `BOILING_POINT_CELSIUS`, `ABSOLUTE_ZERO_CELSIUS`, `MAX_SENSOR_READINGS`, `TEMPERATURE_UNIT`: all `const`. None of these need a stable memory address. They are values that should be inlined at every use.
- `APPLICATION_NAME`, `VERSION`: in this lab, either could be `const`. They could become `static` if you needed to expose them to FFI code that takes a `*const c_char`, but for pure-Rust use, `const` is fine.

The default rule: use `const` unless you have a specific reason to need a stable address. The reasons for needing a stable address are rare in pure-Rust code (FFI, certain lock-free data structures, embedded register definitions).

**2. Could `MAX_SENSOR_READINGS` be a `static`?**

Yes, in a sense. `static MAX_SENSOR_READINGS: usize = 1000;` would compile. But you cannot use a `static` in a context that requires a compile-time constant value, such as the size of an array type:

```rust
static SIZE: usize = 5;
let arr: [i32; SIZE] = [0; SIZE];   // does not compile
```

Array sizes must be `const`. The compiler needs the value at the moment it is choosing the array's type, before any code runs. A `static` is a runtime address; even though its value is determined at compile time, it is not treated as a constant expression.

This is one of the practical reasons to default to `const`: it works in more contexts, including type-level uses like array sizes and generic constant parameters.

### Checkpoints

**1. Why is `usize` the right type for `MAX_SENSOR_READINGS`?**

`usize` is the type used for collection indices, array sizes, and any quantity that represents a memory size or address. Using `usize` for a quantity like "maximum number of readings" matches the conventions of the standard library: when you index into the array, you will use a `usize` index, and the maximum size is naturally the same type.

If `MAX_SENSOR_READINGS` were `u32`, you would have to cast it to `usize` every time you used it for indexing. The cast would compile but it would add noise. Picking `usize` from the start aligns with how the value is used.

**2. Why can the compiler call some functions during compilation but not others?**

Functions marked `const fn` are compile-time-evaluable. The compiler executes them during compilation using a restricted interpreter. The restriction is significant: a `const fn` can do basic arithmetic, control flow, and call other `const fn`s, but it cannot allocate, perform I/O, access the system clock, generate random numbers, or take any action that requires the runtime.

`std::time::SystemTime::now()` requires reading the system clock, which is a runtime operation. The compiler refuses because the value would not be the same on every run, which contradicts what "constant" means.

This `const fn` restriction is the same reason you cannot use `Vec::new()` (which allocates) in a `const`. The `const` system gradually grows over time as more standard library functions become `const fn`-eligible, but allocation and I/O are unlikely to ever be allowed.

**3. Why is `PI` a `const` rather than a `static`?**

Three reasons:

- The value is genuinely constant: a mathematical constant computed at compile time. There is no reason to allocate a memory location for it.
- Inlining at use sites lets the optimizer fold it into surrounding expressions. If your code computes `2.0 * PI`, the compiler can replace that with the literal `6.283185307179586` directly.
- `const` works in more contexts than `static`. Math constants are commonly used in generic code, array sizes (less so for `f64`), and other compile-time positions where `static` would not work.

The standard library uses `const` for nearly every value of this kind: PI, E, the square root of 2, the maximum of a numeric type. The pattern is a model worth following.

---

## Exercise 5 -- Shadowing

### Checkpoints

**1. How does Task 5.1 not violate static type discipline?**

Each `let raw_reading = ...` creates a new variable that happens to share a name with the previous one. The compiler tracks each binding separately. From the compiler's perspective, there are four different variables, each with its own type, that are visible in different parts of the function:

```rust
let raw_reading_1: &str = "  22.5  ";
let raw_reading_2: &str = raw_reading_1.trim();
let raw_reading_3: f64 = raw_reading_2.parse().expect("not a number");
let raw_reading_4: f64 = raw_reading_3 * 1.8 + 32.0;
```

The shared name is a programmer convenience; under the hood, each binding is a distinct entity with a fixed type. Static type discipline is not violated because no variable ever changes its type; the appearance of a "type change" is just one variable going out of scope as another takes its name.

**2. Why does `mut` not allow type changes?**

A `mut` binding represents a single memory location that holds a value of a specific type. The size of the storage was determined when the binding was created. If `temperature: f64` is reassigned to a `&str`, the storage layout would have to change at runtime: floats and string slices have different sizes, alignments, and layouts.

Rust's type system is built on the assumption that a binding's type is fixed for its entire lifetime. This enables many things:

- The compiler knows how much stack space each function needs.
- The borrow checker can reason about lifetimes without considering type changes.
- Code generated for the binding does not need to handle type-changes at runtime.

Allowing `mut` to change types would either make the language much harder to compile efficiently or would require boxing every mutable binding (paying for heap allocation and dynamic dispatch). Shadowing solves the same problem without the cost: the new binding gets its own storage.

**3. When to shadow, when to use distinct names?**

Shadow when the new value is a transformation of the old one and the same conceptual variable persists:

```rust
let user_input = read_line();
let user_input = user_input.trim();
let user_input: u32 = user_input.parse().expect("invalid");
```

Use distinct names when the values represent different concepts:

```rust
let raw_input = read_line();
let username = raw_input.trim();
let user_id: u32 = lookup_user_id(username);
```

The distinction is whether the reader's mental model carries forward (same concept, different form) or branches (new concept). When in doubt, prefer distinct names; clarity to a reader is usually worth the extra word.

---

## Exercise 6 -- Tuples

### Checkpoints

**1. Why can tuples not be indexed with a runtime value?**

Tuple fields can have different types. `(i32, &str, f64)` has three fields with three different types. The size and alignment of each field is different, and the offset of each field within the tuple is determined at compile time.

If you wrote `tuple.i` where `i` is a runtime integer, the compiler would not know which field's type to produce. Should the result be `i32`, `&str`, or `f64`? There is no answer that works at runtime, because the type system requires an answer at compile time.

By contrast, arrays can be indexed with a runtime value because every element has the same type. The compiler always knows what type the result will be: the element type.

**2. When return a tuple, when return a struct?**

Return a tuple when:

- The number of values is small (two or three).
- The meaning of each position is obvious from the function name and type.
- The return is destructured immediately by the caller.

Return a struct when:

- The number of values is more than three.
- The values are easy to mix up because they share types.
- The return is passed around or stored, rather than destructured immediately.
- You want to add fields later without breaking calling code.

For `analyze_readings`, the three `f64` values are easy to mix up. A reasonable refactor would be:

```rust
struct Stats {
    min: f64,
    max: f64,
    avg: f64,
}

fn analyze_readings(values: &[f64]) -> Stats { ... }
```

Then callers use `stats.avg` instead of `result.2`, and a typo like `stats.mim` produces a compile error rather than the wrong value.

**3. What does Task 6.4 reveal about functions that do not return anything?**

Every Rust expression has a type. A function call is an expression. A function that "does not return anything" actually returns the unit type `()`, which has exactly one value (also written `()`). The implicit return is consistent with how every other expression works.

This consistency simplifies the language. There is no special category of "void function." Every function is an expression that produces a value. Some of them produce the unit value, which carries no information, but the type system treats them the same as any other function. This is why you can write `let x = some_function_that_prints();` and get a binding of type `()`. The binding is rarely useful, but the language does not need a special rule to forbid it.

---

## Exercise 7 -- Arrays and Slices

### Checkpoints

**1. Why is the size part of the array type?**

Two main reasons:

- **Stack allocation.** An array's size determines its stack footprint. The compiler must know the size to lay out the function's stack frame. If the size were a runtime quantity, arrays could not be stored on the stack, defeating their purpose.
- **Static guarantees.** When the size is part of the type, the compiler can verify many properties at compile time. A function that takes a `[f64; 3]` is guaranteed by the type system to receive exactly three values.

The cost is that you cannot have a function that "accepts arrays of any size" without using slices. For most purposes, this is the right design: when you need a runtime-sized sequence, use `Vec`; when you need a compile-time-sized sequence, use an array. The two roles stay separate.

**2. What does `&[f64]` gain over `[f64; 5]` or `Vec<f64>`?**

`&[f64]` accepts any contiguous sequence of `f64` values:

- A reference to an array of any size: `&[1.0, 2.0, 3.0]`.
- A reference to a `Vec<f64>`.
- A subrange of either: `&array[1..4]`.
- A reference to a slice already in scope.

`[f64; 5]` accepts only arrays of exactly 5 elements. `Vec<f64>` requires the caller to allocate or convert. Neither alternative is as flexible.

The convention in idiomatic Rust is: take the most general accepted type, return the most specific produced type. For sequence parameters, `&[T]` is almost always the right choice. For sequence return values, choose `Vec<T>` (if you produced a new collection) or `&[T]` (if you are returning a view into existing data).

**3. When can the compiler skip bounds checks?**

The optimizer can eliminate a bounds check when it can prove the index is in range at compile time. Common cases:

- Constant indices into known-sized arrays: `arr[3]` where `arr` is `[i32; 5]`. The index 3 is definitely less than 5.
- Loop variables with proven bounds: a `for i in 0..arr.len()` loop where `arr.len()` is known.
- Arithmetic on a value that has just been compared: `if i < arr.len() { arr[i] }` can elide the second check.

The compiler cannot eliminate the check when the index is a runtime value with no proof of safety:

- Indices read from input or untrusted data: `arr[user_input]`.
- Indices computed in ways the optimizer cannot follow: `arr[some_complex_function(x)]`.
- Indices passed across function boundaries without inlining.

Released optimized code on tight loops typically has zero bounds-check overhead because the optimizer can prove the loop bounds. Code on randomly-accessed data may pay one comparison per access, which is still much cheaper than the alternative cost of memory unsafety.

---

## Exercise 8 -- Putting It Together

### Task 8.2 -- Pattern identification

1. **Scalar type chosen deliberately:** `sensor_id: u32` for sensor identifiers. Could have been `i64` or `usize`, but `u32` is enough for billions of sensors and signed is unnecessary.

2. **Inference success and required annotation:**
   - Success: `let sensor_id: u32 = 1024;` would have been inferred without the annotation if the type-fixing was not needed.
   - Required: `let readings: Vec<f64> = ...collect();`. Without the annotation, the compiler cannot tell which collection type to produce from `collect`.

3. **Immutable binding that could have been mutable:** `readings`, `min`, `max`, `avg`, `avg_f`. Each is computed once and used. The absence of `mut` tells the reader these values are final.

4. **Constant that could not be `let`:** `FREEZING_POINT_C`, `BOILING_POINT_C`. These are at module scope, where `let` is not allowed. They could be `let` only if defined inside a function, where they would not be visible from other functions.

5. **Shadowing for type change:** technically not present in this version. The `readings` chain uses iterator combinators that produce the final type in one expression. A possible variant with shadowing:
   ```rust
   let readings = raw_data.split(',');
   let readings: Vec<f64> = readings.map(...).collect();
   ```

6. **Tuple for multi-value return:** `analyze` returns `(f64, f64, f64)`, and the caller destructures with `let (min, max, avg) = analyze(...)`.

7. **Slice as parameter:** `analyze(values: &[f64])` accepts any sequence of `f64`. The caller passes `&readings`, which is a `Vec<f64>`, and Rust automatically coerces to `&[f64]`.

### Checkpoints

**1. Hard-to-find bug from same-typed return values?**

Yes. The `(f64, f64, f64)` returns three values that look identical. A caller could write `let (avg, max, min) = analyze(&readings)` (with the names in the wrong order) and the program would compile and run, producing wrong output.

A struct prevents this:

```rust
struct Stats { min: f64, max: f64, avg: f64 }
```

The caller now writes `stats.avg` and a typo like `stats.mim` is a compile error. A misnamed binding (`let summary = analyze(...)`) produces no confusion because every field is named.

This is why named fields are valuable and why Rust developers reach for `struct` even for short groupings when the type repetition is risky.

**2. What changes if `readings` is `&[f64]` instead of `Vec<f64>`?**

The program would no longer own the `readings` data; it would borrow it. The data would have to come from somewhere else (a function parameter, a static, a literal). The current code constructs `readings` inside `main` from `raw_data.split(',').collect()`, which produces a `Vec`; you cannot collect into a `&[f64]` because there is no underlying storage for the slice to point to.

The change matters more for function design than for `main`. If `main` were a function called by other code, taking `&[f64]` would mean accepting borrowed data without taking ownership. That is more flexible for callers but limits what `main` can do (it cannot return the slice, for example, because the borrow tied to its caller's storage).

This is the first hint at the ownership system, which is covered in detail in the next several modules. The choice between owning (`Vec<f64>`) and borrowing (`&[f64]`) is one of the central design decisions in every Rust function.

**3. Why is the function signature considered the contract?**

A signature is what callers see. Callers do not read function bodies. They read signatures and trust them. If a function's signature says `fn celsius_to_fahrenheit(c: f64) -> f64`, the caller can rely on:

- The function takes one `f64`.
- The function returns one `f64`.
- The function does not modify the caller's data (it takes `f64` by value, not `&mut f64`).
- The function does not borrow data (no lifetimes in the signature).

The body's local types are an implementation detail. A reader who cares about correctness reads the signature; a reader who cares about implementation reads the body. Tools that work on code (rust-analyzer, the compiler's type checker, documentation generators) treat the signature as authoritative and infer body types only after the signature is fixed.

This split between interface and implementation is one reason Rust requires signature annotations but allows body inference. The boundary is where the contract lives; inside the boundary, brevity is acceptable.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept**

A common pattern: developers from C-family languages find type widths natural (they have seen `int32_t` and `uint8_t`) but find immutability-by-default counterintuitive. Developers from functional language backgrounds (Haskell, OCaml, Scala, Elm) find immutability natural but struggle with the explicit width choices.

The shadowing concept is often the most surprising. Most languages either disallow it or treat it as a mistake. Rust treats it as an idiom, which takes time to internalize.

**2. Rust's integer types vs another language**

Compared to Python, which has one arbitrary-precision integer type:

- Rust gains: predictable memory layout, no allocation for small numbers, faster arithmetic, explicit overflow behavior, ability to interoperate with C ABIs without conversion.
- Rust costs: more decisions to make at every variable declaration, easier to choose the wrong type, more code to write for things that "just work" in Python.

Compared to Java, which has fixed-width primitives but no unsigned types:

- Rust gains: unsigned types for cases where they are correct (sizes, indices, byte values), explicit overflow behavior at compile time.
- Rust costs: more variants to choose from (signed vs unsigned multiplied by widths).

The right tradeoff depends on the domain. For application programming, Python's "everything is a big integer" is convenient. For systems programming, Rust's explicit widths are essential. The fact that Rust forces the choice is part of why it is suitable for systems work.

**3. A decision the reader changed**

This is necessarily personal. Common patterns:

- "I initially declared a counter as `mut` and then realized I could compute it with `iter().count()` and avoid the `mut` entirely."
- "I initially used a tuple to return three results, then realized after re-reading that the values were easy to mix up and switched to a struct in my own code."
- "I initially annotated every local variable, then noticed the program was full of redundant noise and removed annotations from cases where the type was obvious from the right-hand side."

The lab is designed to surface several of these moments. The reflection is most valuable when the reader can identify a specific case where their first instinct was reasonable but the second instinct was better.

---

*End of Lab 2 Solutions*
