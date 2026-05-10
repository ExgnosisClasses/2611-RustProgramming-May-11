# Module 13: Traits and Generics
## Lab 9 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Defining and Implementing a Trait

### Checkpoints

**1. What happens if `impl Shape for Triangle` provides `area` but not `perimeter`?**

The compiler rejects the implementation:

```
error[E0046]: not all trait items implemented, missing: `perimeter`
```

The trait is a contract. By writing `impl Shape for Triangle`, you are promising that `Triangle` satisfies the contract. Providing only `area` is incomplete; the compiler refuses to let you claim a trait implementation that does not actually implement the trait.

This strict enforcement is the source of the trait system's power. Once a type implements a trait, callers can rely on every trait method being available. There is no "this implementation forgot to override one method, watch out for it" footgun.

The fix is either to provide the missing method or, if the method has a sensible default, to use the default. The compiler error tells you exactly which methods are missing.

**2. Could `print_shape_info` take `impl Shape` instead of `&impl Shape`?**

Yes, but the semantics would differ. Compare:

```rust
fn print_shape_info_ref(shape: &impl Shape) {
    println!("area: {:.2}", shape.area());
}

fn print_shape_info_owned(shape: impl Shape) {
    println!("area: {:.2}", shape.area());
}

// Calling them:
let r = Rectangle { width: 3.0, height: 4.0 };
print_shape_info_ref(&r);     // passes a reference; r still valid
print_shape_info_owned(r);     // moves r; cannot use after
```

The owned version consumes the shape. The caller cannot use the shape afterward. For functions that only read, the reference version is universally preferable.

The owned version is appropriate when the function needs to modify the shape and return it, store it, or otherwise take ownership. For a simple read-only operation like printing, references are the right choice.

This parallels the broader convention: take borrowed parameters (`&T`) unless you need ownership.

**3. Why is it valuable for the trait not to mandate an implementation strategy?**

The trait is about *what* a shape can do, not *how* it does it. Different shapes compute area differently:

- Rectangle: `width * height`.
- Circle: `π * r²`.
- Triangle: Heron's formula or `(base * height) / 2`.
- Polygon: shoelace formula.
- Spline-bounded region: numerical integration.

Each implementation knows its own formula. The trait is the abstraction layer: "every shape has an area; how you compute it depends on what kind of shape you are."

If the trait mandated an implementation strategy (say, "all area methods must use Heron's formula"), the abstraction would be broken. Rectangles do not use Heron's formula; the constraint would either force inefficient implementations or rule out valid shape types entirely.

The trait's job is to give callers a uniform way to ask "what is your area?" without caring about how the answer is computed. This separation is the central value of trait-based polymorphism: callers depend on the contract; implementors choose their strategy.

---

## Exercise 2 -- Default Methods

### Checkpoints

**1. Why does the default `describe` method work without knowing the concrete implementor?**

The default method calls `self.area()` and `self.perimeter()`, which are required methods of the trait. The compiler does not need to know *what* concrete type `self` is at the point of writing the default; it only needs to know that whatever type `self` is, it will provide `area` and `perimeter`.

When the default is invoked on a `Rectangle`, the compiler resolves `self.area()` to `Rectangle::area`. When invoked on a `Circle`, it resolves to `Circle::area`. The default is parameterized by the implementor without explicit generics; the trait system handles the dispatch.

This is exactly the same mechanism as virtual method calls in OO languages. The terminology differs (traits vs interfaces, default methods vs default implementations in Java 8+), but the runtime behavior is similar. The key difference in Rust is that for generic functions taking `impl Shape`, the dispatch happens at compile time (monomorphization); for `dyn Shape`, it happens at runtime (vtable lookup).

The default method itself is a kind of generic: it works for any implementor, with the implementor providing the building blocks.

**2. If `Rectangle` overrides `describe` and the override calls `self.area()`, which `area` is used?**

The implementor's `area`. Method resolution goes through the same path regardless of whether you are in a default or override.

The rule: when you call a trait method on `self`, Rust dispatches to the most specific implementation available. For `Rectangle`, that is `Rectangle::area` (which it must provide because the trait requires it).

This means default methods can be customized by overriding either themselves or any of the methods they call. If you override `area`, every default method that uses `area` automatically picks up the override. This makes traits very composable: you can change behavior at any level of the hierarchy without rewriting unrelated code.

A practical implication: when designing a trait, the choice of which methods are required vs. defaulted shapes the API. Required methods are the "primitive operations" that implementors must define. Default methods are derived operations that follow from the primitives. Picking the right division matters: if you require too many methods, implementors have more boilerplate; if you require too few, defaults become awkward or impossible to write.

**3. Why have `classify -> is_large -> area` instead of one big `classify` default?**

Three reasons:

- **Reusability.** `is_large` is potentially useful by itself. Code that needs to ask "is this shape large?" can call `is_large` directly rather than calling `classify` and parsing its result. Each method is an entry point.
- **Customizability.** A specific implementor might want to change just the threshold for "large" without rewriting the classification logic. By making `is_large` its own method, the implementor can override only it; `classify` automatically uses the new threshold.
- **Readability.** The layered structure documents the logic. A reader sees `classify` calls `is_large` calls `area`, and understands the dependency chain. A single `classify` method with all the logic mixed in is harder to follow and reason about.

This layering is a general pattern in trait design: break operations into the smallest useful primitives, then build defaults on top. Each layer is independently meaningful, overrideable, and reusable. The cost is a few extra method declarations; the benefit is a more flexible and maintainable trait.

Compare to OO inheritance: the same pattern (small primitive methods, default behavior built on top) is the Template Method design pattern. Rust's traits with default methods give you this pattern as a language feature.

---

## Exercise 3 -- Trait Bounds and Generic Functions

### Checkpoints

**1. When is one form the only sensible choice?**

Each form has cases where it is the only good option:

- **`impl Trait`:** only when you want to constrain a single parameter to implement a trait and you have no need to refer to the type elsewhere. It is the most concise and is preferred for simple cases. It cannot be used when you need to refer to the parameter's type elsewhere (e.g., in the return type or in another parameter).

- **Named generics (`<S: Shape>`):** required when the same type parameter appears in multiple positions. For example, `fn pair<S: Shape>(a: &S, b: &S) -> (S, S)` forces both inputs and outputs to be the same type. You cannot express this with `impl Trait`.

- **`where` clauses:** required when bounds involve generic types of the parameters themselves, or when there are many bounds that would make the inline form unreadable. For example: `fn process<T>(items: T) where T: IntoIterator, T::Item: Display`. The where clause can refer to `T::Item`, which would be awkward inline.

For simple single-parameter, single-bound functions, all three forms work and the choice is purely stylistic. As complexity grows, the constraints push toward specific forms.

The community convention: use `impl Trait` for simple parameters, named generics when the type appears multiple times, and where clauses for complex bounds. Mixing forms in one function is fine; use whichever fits each constraint.

**2. Why might a function want both parameters to be the same type? When is the two-type version better?**

The same-type constraint is appropriate when the function's behavior assumes the parameters are the same kind of thing:

- A function that returns one of the two parameters (the result type must be unambiguous).
- A function that compares them (e.g., `min(a, b) -> T`).
- A function that combines them in a way that requires them to be interchangeable.

The two-type version is appropriate when the function genuinely accepts any two shapes:

- A function that computes a total area (just needs them to have `area()`).
- A function that prints both (just needs them to be `Debug`).
- A function that returns a single result type unrelated to the inputs.

The single-type constraint is more restrictive; use it when the restriction is meaningful. The two-type version is more flexible; use it when there is no reason to require the same type.

Consider also: even if a function logically takes two shapes, the named-single-type form `fn f<S: Shape>(a: &S, b: &S)` is often used when callers will naturally pass the same type. The constraint is "documentation through types"; it signals that the function is symmetric in its inputs.

**3. Why does `describe_largest` need each bound?**

The function uses both `Shape` (to call `.area()`) and `Debug` (to print the largest shape with `{:?}`).

If you remove `Shape`:

```
error[E0599]: no method named `area` found for type parameter `S`
```

Without the `Shape` bound, the compiler does not know that `S` has an `area` method. The bound is what makes the method available.

If you remove `Debug`:

```
error[E0277]: `S` doesn't implement `Debug`
```

The `{:?}` placeholder requires `Debug`. Without the bound, the compiler does not know that `S` can be debug-formatted.

The principle: bound only what you need. The function actually uses `Shape::area` and `Debug` formatting, so it needs both bounds. If you added more bounds (e.g., `Clone`) without using them, the compiler would not complain, but callers would face an unnecessary constraint.

Adding bounds also documents what callers must provide. A function with `<S: Shape + Debug>` tells callers exactly what they need: a shape that can be debug-printed. The signature is the contract; clear contracts make code reusable.

---

## Exercise 4 -- Returning Trait Types

### Checkpoints

**1. Why does `impl Trait` require all return paths to be the same concrete type?**

`impl Trait` is static dispatch: the compiler picks one concrete type for the function's return value and generates specialized code. The concrete type is part of the function's signature from the compiler's perspective, even though it is not visible to callers.

If `shape_or_other` could return either `Circle` or `Rectangle`, the function would need to dispatch at runtime: "which type am I returning right now?" That is exactly what `Box<dyn Trait>` does. The runtime dispatch requires a vtable, allocation, and indirection.

`impl Trait` was deliberately designed to avoid these costs. By requiring a single concrete type, the compiler can compile the function as if it returned that type. The `impl Trait` syntax is purely about hiding the concrete type from the caller (so the function can change its return type internally without breaking the API).

The naming is somewhat misleading: `impl Trait` looks like it might be a trait object, but it is not. It is "some specific type that implements this trait, chosen by the function." When you need a true "different types at runtime," use `Box<dyn Trait>`.

**2. What does `Box<dyn Shape>` cost? When does it matter?**

`Box<dyn Shape>` costs:

- **One heap allocation per shape.** The concrete shape data is stored on the heap.
- **One vtable pointer per `Box`.** The Box stores both a pointer to the data and a pointer to the vtable (so it is two pointers wide, not one).
- **Indirection on method calls.** Each method call goes through the vtable: load the function pointer, then call it. This is one extra memory access compared to a direct call.
- **No inlining.** The compiler cannot inline dynamic-dispatch calls because it does not know which implementation to inline. Some optimizations are blocked.

When the cost matters:

- **Hot loops.** A tight loop calling `shape.area()` a million times will be noticeably slower with dynamic dispatch (maybe 2-3x for trivial methods). For non-trivial methods, the indirection cost is dominated by the work the method does, and the difference is smaller.
- **Memory-constrained environments.** Each `Box` is an extra allocation; large collections of shapes mean many allocations.
- **Cache-sensitive code.** Heap allocations scatter data across memory, hurting cache performance. Contiguous arrays of one type fit better in cache.

When the cost is negligible:

- **Occasional method calls.** Dispatching once or twice per request: invisible.
- **I/O-bound code.** If you are waiting on disk or network, dispatch overhead is rounding error.
- **Methods with substantial work.** The dispatch is one memory access; the method itself does much more.

The general rule: prefer static dispatch by default. Reach for `Box<dyn Trait>` when you need the runtime flexibility. The cost is real but usually small; the flexibility is sometimes essential.

**3. Heterogeneous collection: trait objects vs enum?**

Both approaches work; the trade-offs are different.

`Vec<Box<dyn Shape>>`:

- **Pros:** Extensible. New shape types can be added without changing the collection or its consumers. Each variant can be independently defined and added.
- **Cons:** Heap allocation per element. Indirect dispatch on every method call.

`Vec<ShapeEnum>` where `ShapeEnum` is an enum:

- **Pros:** No heap allocation; all variants are inline. Static dispatch (the enum can be matched directly). Compact memory layout.
- **Cons:** Closed set. Adding a new shape type means modifying the enum and every match expression that uses it.

When to choose each:

- **Trait objects** when the set of types is open or extensible (plugins, user-defined types, libraries that users extend).
- **Enums** when the set of types is closed and known at compile time (parser tokens, state machine states, shapes in a graphics library you control).

For "shapes in a graphics library," an enum is often the better choice because the set of shapes is usually fixed. For "plugins that extend my program," trait objects are necessary because the implementations come from outside your code.

The enum approach is often called "sum type polymorphism" while trait objects are called "subtype polymorphism." Rust supports both; choose based on the problem.

A practical hybrid: an enum where one variant holds a `Box<dyn Trait>` for the "everything else" case. This gives you fast dispatch for known types and flexibility for unknown ones.

---

## Exercise 5 -- Generic Structs and Enums

### Checkpoints

**1. What does monomorphization mean for binary size?**

Each different concrete instantiation of a generic produces a separate copy of the compiled code. `Pair<i32>` and `Pair<String>` have completely different generated code. Their methods are separate (`Pair<i32>::new`, `Pair<String>::new`, etc.).

For a project using `Pair` with 10 different types, the binary contains 10 copies of the `Pair` implementation. If `Pair` has 20 methods, that is 200 method bodies compiled in.

The trade-off:

- **Pros:** Each instantiation is fully specialized. The compiler optimizes for each concrete type. No runtime dispatch.
- **Cons:** Binary size grows. Compilation time grows.

For most projects, the binary-size cost is small (kilobytes, perhaps). For libraries with very generic abstractions used many ways, it can add up.

A few techniques mitigate this:

- **Mark code as non-generic where possible.** A method that does not use type parameters can sometimes be factored out of the generic impl into a non-generic helper.
- **Use trait objects (`dyn Trait`)** for the parts of the code that handle many types. This collapses many instantiations into one.
- **Profile-guided builds and link-time optimization** can dedupe some redundancy across instantiations.

In practice, monomorphization is rarely a problem. The standard library uses generics extensively without bloating binaries unreasonably. For most code, the performance benefit outweighs the size cost.

**2. Why put `PartialOrd` on `impl<T: PartialOrd>` rather than on `Pair<T>` itself?**

The bound on `Pair<T>` itself would say "you can only create a `Pair<T>` if `T: PartialOrd`." This would prevent `Pair<String>` from existing if `String` did not implement `PartialOrd`, even if you never wanted to call `largest`.

The conditional impl is more flexible: any `T` can be in a `Pair<T>`, and you can call `largest` only on those where it makes sense. The set of available methods grows with the type's capabilities.

This is a pervasive pattern in the standard library:

- `Vec<T>` always exists; `Vec<T>::sort` exists only when `T: Ord`.
- `HashMap<K, V>` always exists; methods on it have specific bounds (`K: Hash + Eq` for inserting, etc.).
- `Option<T>` always exists; `Option<T>::unwrap_or_default` exists only when `T: Default`.

The pattern: the data structure should be as flexible as possible. Methods should have just the bounds they need. This maximizes the structure's usefulness.

If you put bounds on the structure itself, you restrict its applicability. Sometimes this is correct (a `SortedVec<T>` would naturally require `T: Ord`), but more often it is overconstraining.

**3. Why `Result<T, E>` instead of `Either<L, R>`?**

Several reasons:

- **Semantic clarity.** `Result` conveys "success or failure"; the variants are `Ok` and `Err`. `Either` is symmetric; nothing privileges one side. For error handling, the asymmetry is the point.

- **API design.** `Result` has methods that assume the "left is success" interpretation: `is_ok`, `unwrap`, `map`, `map_err`. `Either` would need a different API: `is_left`, `unwrap_left`, etc. The Result API matches the error-handling use case better.

- **`?` integration.** The `?` operator is built around `Result`; it expects `Ok` (success) and `Err` (failure). A generic `Either` would need explicit conventions to integrate.

- **Standard library uniformity.** Every error-returning function in the standard library uses `Result`. If error handling used `Either`, the convention would be `Left = success`, but readers would have to remember which side is which. `Result` makes it obvious.

The trade-off: `Result` is less general but more communicative. For the specific case of error handling, the specificity is worth it.

If you genuinely need a sum type that does not imply success/failure (e.g., "this returns either a parsed number or a parsed string, both valid"), you can define your own `Either` or use one from the `either` crate. Most code does not need this; `Result` covers the common case, and `Option` covers "maybe a value."

---

## Exercise 6 -- Common Standard Library Traits

### Checkpoints

**1. Display cannot be derived; Debug can. What does this design tell you?**

`Debug` is for developers. It has a canonical format: type name and field values, with consistent quoting and structure. The format is uniform across all types and is meant for diagnosis. Because there is one obvious right answer, the compiler can derive it.

`Display` is for users. The format depends entirely on the application: should a `Color` display as "#FF0000," "rgb(255, 0, 0)," "red," or something else? Should a `Date` display in ISO 8601, locale-specific format, or human-readable text? Each application makes its own choice. There is no canonical format, so the compiler refuses to guess.

The lesson: derive what has a canonical implementation; write manual implementations for what does not. Other examples:

- `Clone` can be derived (clone every field). Cannot always be derived if a field's clone is not the natural choice.
- `Default` can be derived (use each field's default). Cannot be derived if no obvious default exists.
- `Hash` can be derived (hash every field). Cannot be derived if the type needs special equivalence semantics.

The pattern is consistent: when there is one obvious answer, the compiler provides it. When there are choices, the developer makes them.

**2. Why provide `From` and not require both `From` and `Into`?**

The standard library provides a blanket implementation: for every `T: From<U>`, there is automatically `U: Into<T>`. This means implementing one gives you both.

The asymmetry is intentional. `From` is the canonical direction: "construct a T from a U." `Into` is the "consume a U to produce a T" framing. They are the same operation, viewed differently.

The choice to write `From` rather than `Into`:

- `From` reads naturally as constructor: `Color::from(...)` parallels `Color::new(...)`.
- `Into` reads naturally as conversion: `value.into()` says "convert this value."

Both are useful at different call sites. `From::from` is explicit about the target type; `into()` is shorter and lets the target be inferred.

By implementing `From`, you provide both. The standard library would have to be more verbose if implementors had to write both. The blanket impl is a small but pervasive convenience.

There is one case where the asymmetry breaks down: you cannot implement `Into<T>` for `U` without owning `T` (the orphan rule prevents implementing foreign traits for foreign types). For your own type `T`, this is fine; for foreign types, the `From` direction may be unusable, leaving only manual implementations.

**3. Why one required method (`next`) plus many defaults? When is this pattern appropriate?**

The pattern is: identify the smallest set of operations from which everything else can be derived. `next` is the absolute minimum for iteration: "give me the next element or signal that there are no more." Every iteration method can be expressed in terms of `next`:

- `map(f)`: call next, apply f, return.
- `filter(p)`: call next repeatedly until p returns true.
- `count()`: call next until None, counting.
- `sum()`: accumulate from next calls.

By providing default implementations for all of these, the trait gives implementors a lot of functionality "for free." A custom iterator needs only `next`; it automatically has 70+ methods.

When is this pattern appropriate?

- **When there is a clear smallest set of primitives.** Iteration has one: `next`. Comparison has one: `cmp` (with `PartialOrd::partial_cmp` as a fallback for partial orderings).
- **When defaults are correct for most implementors.** If most implementors would override most defaults, the defaults are not pulling their weight; the pattern is verbose without benefit.
- **When the operations form a coherent algebra.** Iterator methods compose: you can chain them, knowing they all work uniformly. If the methods were unrelated, the unified trait would be artificial.

The Iterator trait is a textbook example. Other traits in the standard library also use this pattern but less extensively. The principle generalizes: when designing a trait, find the small core, write good defaults, and let implementors focus on the essentials.

---

## Exercise 7 -- Operator Overloading and the Newtype Pattern

### Checkpoints

**1. Why are operators traits in Rust instead of built-in?**

Several reasons:

- **Consistency with the type system.** Operators are just methods with special syntax. Treating them as traits means they go through the same mechanism (trait resolution, generics, dispatch) as other methods. There is no special "operator dispatch" rule to learn.

- **Extensibility.** Any type can define how operators work on it. `Vector + Vector`, `Matrix * Matrix`, `Date + Duration` are all natural. Built-in operators would only work for built-in types; extending them would require special language support.

- **Generic code.** A function `fn sum<T: Add<Output = T>>(items: &[T]) -> T` works for any type that implements `Add`. With built-in operators, this kind of generic numeric code would be hard or impossible.

- **No magic.** When you see `a + b` in Rust, you know what is happening: a method call to `Add::add`. You can find the implementation, override it, see what it does. With built-in operators, the behavior is opaque.

The cost is verbosity. Implementing `+` requires writing an `Add` impl. In a language where `+` is built-in for numbers, you do not have to write anything. The trade-off favors flexibility for languages that prioritize type safety and generic code.

C++ has operator overloading via member functions and free functions. Java does not allow it. Python uses `__add__` magic methods. Rust's traits-based approach is closer to Python's idea but with stricter typing.

**2. What does the newtype `Meters` give beyond runtime guarantees?**

Compile-time prevention of unit confusion. Without the newtype:

```rust
fn calc_orbit(distance: f64, velocity: f64) -> f64 { /* ... */ }

let distance_m = 1000.0;     // we think this is meters
let velocity_fps = 50.0;     // this is feet per second
let result = calc_orbit(distance_m, velocity_fps);    // disaster waiting
```

The code compiles. The runtime might silently produce wrong results. The Mars Climate Orbiter crash was exactly this kind of bug: one team used meters, another used feet, the conversion was forgotten, and the spacecraft was destroyed.

With newtypes:

```rust
fn calc_orbit(distance: Meters, velocity: MetersPerSecond) -> Meters { /* ... */ }

let distance = Meters(1000.0);
let velocity = FeetPerSecond(50.0);
let result = calc_orbit(distance, velocity);    // ERROR: expected MetersPerSecond
```

The compiler refuses to silently mix units. Conversion must be explicit (and the developer thinks about whether they have the right conversion direction).

The runtime cost is zero: `Meters(f64)` compiles to the same machine code as a bare `f64`. The wrapping exists only at the source level. This is the magic of "zero-cost abstraction": type safety without performance cost.

The pattern generalizes beyond physical units:

- `UserId(u64)` vs `OrderId(u64)`: prevent passing one where the other is expected.
- `Validated<T>` vs `Unvalidated<T>`: prevent using unvalidated input as validated.
- `Encrypted<T>` vs `Plaintext<T>`: prevent leaking plaintext as encrypted.

Each pattern uses the type system to enforce a invariant that would otherwise depend on developer discipline. The discipline is unreliable; the type system is not.

**3. When is encapsulation in a wrapper a benefit vs problem?**

Benefits of encapsulation:

- **Invariant enforcement.** If the wrapper has invariants (e.g., the inner value is always non-negative, or always sorted), hiding the inner value prevents code from violating them.
- **API control.** The wrapper exposes only the methods you want. Internal details can change without breaking callers.
- **Documentation.** The wrapper's name documents what the data represents. `Wrapper<Vec<String>>` named as `EmailAddresses` is more meaningful than a bare `Vec<String>`.

Problems with encapsulation:

- **Loss of methods.** The inner type's methods are not accessible. To expose them, you have to write delegation methods (one per inner method you want to expose). This is tedious for types with many methods.
- **Loss of trait implementations.** Trait implementations on the inner type do not automatically apply to the wrapper. You have to reimplement them (or use `Deref`, which has its own issues).
- **Boilerplate.** A wrapper that does nothing but expose the inner type adds friction without benefit.

When to use a wrapper:

- The wrapper has invariants the inner type does not enforce.
- The wrapper's API differs meaningfully from the inner type's.
- You want to prevent confusion between the wrapper and other uses of the inner type.

When not to:

- You just want a different name for the same operations. Use a type alias (`type EmailList = Vec<String>;`).
- You want to expose all the inner methods. Use a transparent wrapper with `Deref`, or accept the cost of delegation.

The newtype is a tool; like any tool, it has uses and misuses. The judgment is "what invariant or distinction am I adding?" If the answer is meaningful, the wrapper is worth it. If not, the wrapper is just noise.

---

## Exercise 8 -- Putting It Together

### Task 8.2 -- Pattern identification

1. **Trait with required and default methods:** `Shape` has required `area` and `perimeter`, with default `density` and `summary`.

2. **Supertrait bound:** `trait Shape: fmt::Debug` says any `Shape` implementor must also implement `Debug`. This lets the default `summary` use `{:?}` formatting.

3. **Type overriding a default:** `impl Shape for Circle` provides its own `summary` method, overriding the trait default.

4. **`impl Trait` syntax for parameters:** `describe_mixed(shapes: &[Box<dyn Shape>])` uses `dyn Trait` rather than `impl Trait`, but `make_square(side: f64) -> impl Shape` uses `impl Trait` in return position. (None of the functions in this program use `&impl Trait` for parameters, but it would look like `fn print(shape: &impl Shape)`.)

5. **Named generics:** `describe_shapes<S: Shape>(shapes: &[S])` uses the `<S: Shape>` form.

6. **`where` clause:** `largest_area_index<S>(...) where S: Shape + fmt::Debug` uses the `where` form for clarity given the multiple bounds.

7. **Function returning `impl Trait`:** `make_square(side: f64) -> impl Shape`.

8. **Function taking `&[Box<dyn Trait>]`:** `describe_mixed(shapes: &[Box<dyn Shape>])`.

9. **Generic struct with bound:** `struct Group<S: Shape>`.

10. **Method using the bound:** `Group::total_area` and `Group::largest` use `S::area()` (which comes from the `Shape` bound).

11. **Add on a newtype:** `impl Add for Scale` provides the `+` operator.

12. **Display on a newtype:** `impl fmt::Display for Scale` provides `{}` formatting.

### Task 8.3 -- Predict and verify

**Change A:** Remove `Shape: fmt::Debug` supertrait.

**Prediction:** The default `summary` method uses `{:?}` formatting on `self`, which requires `Debug`. Without the supertrait bound, the compiler does not know that implementors will have `Debug`. The trait definition itself will fail to compile.

**Result:** As predicted, the compiler rejects the trait definition:

```
error[E0277]: `Self` doesn't implement `Debug`
```

The error points to the use of `{:?}` in `summary`. The fix is either to restore the supertrait bound or to remove the use of `{:?}` from the default method.

The lesson: supertrait bounds let default methods rely on capabilities from other traits. They are a way to compose trait requirements: "any `Shape` is also `Debug`."

**Change B:** Make `make_square` conditionally return either `Rectangle` or `Circle`.

**Prediction:** The compiler rejects this because `impl Trait` requires a single concrete type for all return paths.

**Result:** As predicted:

```
error[E0308]: `if` and `else` have incompatible types
```

The compiler tells you the branches return different types. The fix is to use `Box<dyn Shape>`:

```rust
fn make_square(side: f64, as_circle: bool) -> Box<dyn Shape> {
    if as_circle {
        Box::new(Circle { radius: side / 2.0 })
    } else {
        Box::new(Rectangle { width: side, height: side })
    }
}
```

The change forces a heap allocation per call, but it allows the conditional return. This is the right trade-off when the function genuinely needs to return different concrete types.

**Change C:** Change `describe_shapes` from generic to taking `&[Box<dyn Shape>]`.

**Prediction:** The function now uses dynamic dispatch. Callers must convert their typed slices to vectors of boxed trait objects (each shape needs to be boxed). The signature loses information about the concrete type. Performance becomes slightly worse due to dynamic dispatch (each method call goes through a vtable).

**Result:** As predicted, the call site requires conversion:

```rust
let boxed: Vec<Box<dyn Shape>> = rectangles.iter()
    .map(|r| Box::new(*r) as Box<dyn Shape>)
    .collect();
describe_shapes(&boxed);
```

This is significantly more verbose. The original generic version accepted `&rectangles` directly.

The performance implications:

- The generic version compiles to a specialized function for `Rectangle` slices, with `area()` calls fully inlined.
- The dynamic version compiles to a single function that goes through the vtable for every `area()` call.
- For 3 rectangles, the difference is unmeasurable. For 3 million, the difference is real but small (perhaps 2-3x for a tight loop).

The lesson: static dispatch is the default for good reason. It is more efficient and more ergonomic for callers. Dynamic dispatch is appropriate when you need the flexibility (heterogeneous collections, plugin systems). Forcing dynamic dispatch where it is not needed makes the code slower and more verbose without benefit.

### Checkpoints

**1. What does the supertrait `Shape: fmt::Debug` provide?**

It requires every implementor of `Shape` to also implement `Debug`. This in turn allows the default `summary` method to use `{:?}` formatting on `self` without any additional bound.

From an implementor's perspective: when you write `impl Shape for Rectangle`, you must also have `impl Debug for Rectangle` (or `#[derive(Debug)]` on `Rectangle`). The trait system enforces this; without `Debug`, the impl will not compile.

From a consumer's perspective: any code that has `&dyn Shape` or `S: Shape` can call `Debug` methods on it. The supertrait essentially adds `Debug` to the contract.

Supertraits are how you build hierarchies of trait requirements. Without them, you would have to add `Debug` to every function that uses `Shape` if the function needs debug formatting. With them, the requirement is part of the `Shape` trait itself; consumers do not need to redeclare it.

The trade-off: supertraits make implementing the trait more demanding. Adding a supertrait can break existing implementors who do not satisfy it. So they are a design decision: do you want to require this capability of all implementors, or leave it to consumers to require where needed?

**2. When to choose static vs dynamic dispatch?**

**Static dispatch (`impl Trait`, `<S: Trait>`) when:**

- The set of types is known at compile time.
- The collection is homogeneous (all the same type).
- Performance matters (hot loops, large collections).
- You want the compiler to inline and optimize.

**Dynamic dispatch (`Box<dyn Trait>`) when:**

- The set of types is open (plugins, user-defined types).
- The collection is heterogeneous (mixed types).
- Method dispatch is rare (called infrequently per use).
- You need to store a single name to refer to different types (e.g., a configuration that loads different processor types).

The default choice in Rust is static dispatch. Idiomatic Rust uses generics and `impl Trait` freely; `Box<dyn Trait>` appears when there is a specific reason.

In other languages, dynamic dispatch is often the default (Java interface references, C++ virtual methods). Rust's design encourages the opposite: prefer static, opt into dynamic when needed. The result is generally faster code at the cost of slightly more compile time and larger binaries.

**3. What does wrapping `f64` as `Scale` accomplish?**

Type-level distinction. Without `Scale`, the code passing scale factors around would use bare `f64`, indistinguishable from any other floating-point quantity. A function `fn apply(rotation: f64, scale: f64)` could be called with arguments in the wrong order, and the compiler would not notice.

With `Scale`, the type system catches the error. A function `fn apply(rotation: Rotation, scale: Scale)` rejects `apply(scale_value, rotation_value)` because the types do not match. The newtype wrapping makes the meaning of the value part of its type.

Beyond preventing argument-order bugs:

- **Documentation.** Reading `fn f(s: Scale)` tells you what kind of value goes in. Reading `fn f(s: f64)` does not.
- **Method namespace.** `Scale` can have methods specific to scale factors (`combine`, `clamp_to_range`, etc.) that do not pollute the `f64` namespace.
- **Display.** `Scale::Display` formats as `2x` rather than `2.0`, communicating intent.

The pattern adds no runtime cost. `Scale(f64)` compiles to the same code as `f64`. The wrapping is purely at the source level: a hint to the compiler about how to type-check, and a hint to readers about what the value means.

For values that have no semantic distinction (a generic floating-point computation that does not care what the numbers represent), bare `f64` is fine. For values with meaning (units, identifiers, validated data), newtypes are the safer choice.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

Common patterns:

- Developers from C find traits unfamiliar (no equivalent), but generics map onto C++ templates with familiarity. The trait bounds syntax can feel verbose.
- Developers from Java find traits similar to interfaces (with default methods), and generics map onto Java generics. The biggest adjustment is the lack of inheritance: composition through trait composition replaces inheritance hierarchies.
- Developers from Python find the explicit type system intimidating at first; generics with bounds (especially with multiple bounds) can feel like overhead. Once they internalize that bounds are documentation enforced by the compiler, it becomes natural.
- Developers from Haskell or Scala find traits very natural (similar to type classes or traits in those languages). The pattern of "trait + generic function with bound" is familiar.

The most common reflection: "Trait objects (`dyn Trait`) versus generics (`impl Trait`, `<T: Trait>`) was confusing at first. I kept reaching for trait objects because they felt simpler, but I gradually learned that generics are usually the right choice. The performance and ergonomic difference is real."

**2. Inheritance pattern in Rust.**

Sample answer focusing on the Template Method pattern:

"The Template Method pattern in Java uses inheritance: a base class defines an algorithm with steps as abstract methods; subclasses implement the steps. The base class's method calls them in order.

In Rust, the same pattern uses a trait with default methods. The trait defines the algorithm (the default method); the abstract steps are required methods that implementors must provide. Implementors override only the steps; the algorithm is shared.

```rust
trait Renderer {
    fn setup(&mut self);                // required step
    fn draw_content(&mut self);          // required step
    fn cleanup(&mut self);               // required step

    fn render(&mut self) {               // template method
        self.setup();
        self.draw_content();
        self.cleanup();
    }
}
```

Each implementor defines `setup`, `draw_content`, and `cleanup`; the `render` method is shared. This is exactly the Template Method pattern, just expressed differently.

What Rust gains: composition. A type can implement multiple traits, each providing a different template method. With inheritance, you usually have one base class; with traits, you have orthogonal capabilities.

What Rust loses: shared state. A base class can have fields that subclasses use. Traits cannot have fields. To share state, you have to combine traits with structs, often passing data through method parameters or storing it in struct fields with accessor methods. This is slightly more verbose but encourages thinking about what data is shared and what is not.

For simple template methods, Rust traits are equivalent in expressiveness. For complex inheritance hierarchies with deep shared state, Rust forces a different design. Usually the different design is better (composition over inheritance is a long-standing OO principle), but the migration can be uncomfortable."

**3. Static vs dynamic dispatch decision change.**

Sample answer:

"In Exercise 4, I initially wrote `make_shape` to return `impl Shape`. The signature was clean and avoided the heap allocation that `Box<dyn Shape>` requires. I thought 'this is just one shape; static dispatch is appropriate.'

But the function needed to return different types based on the input string. The compiler rejected it because `impl Trait` requires a single concrete type. I tried various ways to work around this (one function per shape type, generic over a marker type) before realizing that `Box<dyn Shape>` was the natural fit. The function genuinely returns different concrete types; dynamic dispatch is what 'different concrete types from one function' means.

What changed my mind was realizing that I was fighting the type system to avoid the heap allocation, when the heap allocation was actually telling me something true about the design: this is dynamic behavior, and Rust models that with `Box<dyn Trait>`. The cost is small (one allocation per shape) and the flexibility is essential.

The lesson generalizes: when the type system rejects your code, try to understand what it is telling you about your design. Sometimes the right move is to satisfy the type system on its own terms; sometimes the rejection points to a deeper issue with the approach. In this case, the rejection was pointing me toward the correct tool. Static dispatch is the right default; dynamic dispatch is the right tool for genuine runtime polymorphism."

---

*End of Lab 9 Solutions*
