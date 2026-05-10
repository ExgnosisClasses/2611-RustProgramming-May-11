# Modules 8 and 9: Structs, Enums, and Pattern Matching
## Lab 6 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Defining and Instantiating Structs

### Checkpoints

**1. Why does Rust not allow per-field mutability?**

Per-field mutability would complicate the type system significantly. The compiler would need to track which fields of each binding can be mutated, propagate this through method calls (does `&mut self` allow mutating any field, or only "mutable" ones?), and handle the interaction with references (can you take a `&mut` reference to a field of an "immutable" struct?).

Rust's choice is simpler: mutability is a property of the binding, not of individual fields. A `let mut user` allows mutation of any field; a `let user` allows none.

The pattern for "some fields are conceptually mutable but others are not" is encapsulation: make the never-changing fields private, expose them through getters, and provide methods that mutate the changeable ones. The compiler enforces "no direct mutation from outside the module" and the methods enforce the invariants.

This separation (binding mutability + module-level encapsulation) is more flexible than per-field mutability and has fewer corner cases. The cost is that you sometimes write a constructor and a method instead of marking a field `mut`.

**2. Why does field init shorthand require exact name matching?**

Exact matching keeps the rule unambiguous and easy to understand. The compiler does not have to guess; the rule is just "if the name matches, you can omit the value."

If the language allowed fuzzy matching (case-insensitive, snake-case-vs-camel-case conversion, etc.), there would be subtle ambiguities: which field should `Email` shorthand for? Email or email_address? The rule would be context-sensitive and surprising.

Exact matching is also a small encouragement to use consistent names throughout the codebase. If a function takes a `username` parameter and the struct has a `username` field, the shorthand works. If the function used a different name (`name` for the struct's `username`), the shorthand would not apply, and you would write the longer form. This is mild pressure toward consistency, which is generally desirable.

**3. Can `user1.sign_in_count` be used after `..user1`?**

No. After a partial move, the *whole struct* cannot be used, even though some individual fields (the `Copy` ones) were not moved.

The rule the compiler enforces: once any non-`Copy` field of a struct has been moved out, the struct as a whole is "partially moved" and cannot be accessed. Individual `Copy` fields cannot be accessed either, even though their values were technically copied rather than moved.

This rule exists because allowing access to the `Copy` fields would create surprising asymmetries. The struct exists as a unit; if you have moved its non-`Copy` parts, the unit is broken. The compiler treats the whole thing as moved-from rather than tracking which fields are still valid.

If you genuinely need a copy of a single `Copy` field before performing a move, copy it first:

```rust
let count = user1.sign_in_count;     // copy out before the move
let user2 = User { ..user1 };         // now user1 is partially moved
println!("{count}");                   // still works
// println!("{}", user1.sign_in_count); // would fail
```

This is an edge case. Most of the time, the rule prevents bugs rather than creating friction.

---

## Exercise 2 -- Methods and Associated Functions

### Checkpoints

**1. What does the `into_` prefix signal? What other naming conventions exist?**

The `into_` prefix signals that the method consumes `self` and produces a new value, typically of a related but different type. The pattern parallels the `Into` trait from Module 13: `into_string`, `into_iter`, `into_bytes`, etc.

Three common conventions for `self`-related methods:

- **`into_X`**: consumes self, produces an X. The original is gone after the call. Example: `String::into_bytes()` consumes the String and produces `Vec<u8>`.
- **`to_X`**: borrows self, produces an X. Often involves cloning or copying. Example: `str::to_string()` borrows the str and produces a new String.
- **`as_X`**: borrows self, produces a reference to a different view. No allocation. Example: `String::as_str()` returns a `&str` view of the String's data.

The conventions are not enforced by the language; they are community standards. Following them makes your code feel familiar to other Rust developers; deviating from them creates friction.

**2. When is `try_new` (returning Result) appropriate vs `new` (returning the value directly)?**

`new` is appropriate when the function can always succeed:

- The inputs are unconstrained.
- Default values are always valid.
- The construction does not involve any check that could fail.

`try_new` (or another `try_*` name) is appropriate when:

- The inputs need validation.
- Construction involves I/O or parsing that can fail.
- The caller might want to handle the failure gracefully.

The convention is: if construction can fail meaningfully, return `Result`. If it cannot, return the value directly.

A related variant: some types provide `new_unchecked` for cases where the caller is certain the inputs are valid. This is often `unsafe` (skipping safety checks for performance), but it can also just bypass validation. The `unchecked` suffix signals "you are responsible for the invariants."

**3. Why does Rust use different syntax for methods vs associated functions?**

Two reasons.

The first is conceptual clarity. A method operates on an instance; the dot syntax `instance.method()` reads as "do this thing to this instance." An associated function does not have an instance; the `::` syntax reads as "this function lives in this type's namespace." The two cases are conceptually different and the syntax distinguishes them.

The second is method resolution. When you write `x.foo()`, the compiler needs to know what `x` is to find the right method. With the `::` syntax, the type is on the left, so there is no ambiguity. With dot syntax, the compiler does method resolution based on the receiver's type, including auto-deref through references.

If both forms used the same syntax, method calls would need to handle both "operates on instance" and "namespaced function" cases, with rules about which interpretation applies. The current design keeps them separate, which is simpler.

A small note: you can call methods using the `::` syntax explicitly (`Type::method(&instance)`), which is useful when type inference fails or when you need to disambiguate. The dot syntax is just sugar for this, with auto-borrow inserted automatically.

---

## Exercise 3 -- Tuple Structs, Unit-Like Structs, and the Newtype Pattern

### Checkpoints

**1. What is the runtime cost of `struct Meters(f64)` vs a bare `f64`?**

Zero. The compiler treats a single-field tuple struct as identical to its inner type at the machine code level. There is no extra indirection, no extra memory, no extra instructions to access the inner value.

The "wrapping" exists only at the source level. The compiler sees `Meters(f64)` as a name that distinguishes one usage from another, but the generated assembly is the same as if you had used `f64` directly. This is what makes the newtype pattern so attractive: type safety without any cost.

This zero-cost guarantee is one of Rust's central design principles. Abstractions that look like they should be expensive (generics, traits, newtypes, iterators) compile to specialized code that has the same performance as hand-optimized equivalents.

**2. What does the validated newtype give you that runtime checks do not?**

A validated newtype gives you compile-time guarantees instead of runtime checks. Once you have a `PositiveInteger`, the type system guarantees the value is positive. Code that takes a `PositiveInteger` as a parameter does not need to re-check; the constructor already validated.

Without the newtype, you would need to either:

- Validate at every function entry point that requires a positive number (verbose, easy to forget, runtime overhead).
- Document the invariant in comments and trust that callers respect it (no enforcement, easy to violate).
- Use assertions, which run at runtime and might miss some paths during testing.

The newtype encodes the invariant in the type system. The compiler enforces it. There is no way to construct a `PositiveInteger` without going through the validating constructor; therefore, there is no way to have an invalid one. Code that requires a positive integer can simply require `PositiveInteger`, and the requirement is enforced everywhere automatically.

This is the same idea as Rust's broader philosophy: use the type system to enforce invariants. The borrow checker enforces memory safety; the option type enforces null handling; newtypes enforce domain invariants.

**3. When do tuple struct positional fields become unwieldy?**

Positional fields are fine when the meaning of each position is universally understood (a `Point(x, y)` is unambiguous). They become unwieldy when:

- **The struct has more than two or three fields.** Calling `p.0`, `p.1`, `p.2`, `p.3` is hard to read. Reviewers cannot easily tell what each field represents.
- **The fields have similar types.** A `User(String, String, String, String)` is a bug waiting to happen. Which String is the username? The email? Named fields prevent the question.
- **The fields are not in a canonical order.** A `Point(x, y)` is fine because `(x, y)` is the standard convention. A `Range(start, end, step)` is debatable because some libraries put step first.
- **You need to omit some fields when accessing.** A struct with named fields can be partially destructured (`Point { x, .. }`); a tuple struct must use all positions or `..`.

The rule of thumb: tuple structs for two-field types where the order is universally understood, or for the newtype pattern (one field). Regular structs with named fields for everything else.

---

## Exercise 4 -- Deriving Traits

### Checkpoints

**1. Why must all fields be `Copy` for a struct to derive `Copy`?**

The `Copy` semantics is "duplicate this value by copying its bits." For a struct, this means copying each field's bits. If any field is not safe to bitwise-copy (because it owns heap data, file handles, locks, etc.), bitwise-copying it would create two owners of the same resource.

The rule is structural and exact: every field, recursively, must be `Copy`. A struct with a `String` field cannot be `Copy` because `String` owns heap memory. A struct containing such a struct also cannot be `Copy`. The chain propagates.

The alternative (deep copying when needed) would mean `Copy` could not be a "free" operation; it would sometimes involve heap allocation. Rust separates these into two traits:

- `Copy` is for cheap, bitwise duplication.
- `Clone` is for explicit duplication, which may involve allocation.

By keeping `Copy` strict, the compiler guarantees that any code working with a `Copy` value pays no hidden cost for duplication. Code that needs to duplicate non-trivial data uses `Clone` and the cost is visible at the call site.

**2. Why is `Display` not in the list of derivable traits?**

`Display` is meant for user-facing output, and there is no obvious automatic implementation. A `Point` could display as "(3, 4)", "x=3,y=4", "Point at (3, 4)", or any number of other formats. Each application has different conventions; the language cannot pick a default.

`Debug`, by contrast, has a canonical format (the type name and field values, structured for developers). The format is consistent across all types and is meant for diagnosis rather than user-facing presentation. The canonical format makes it derivable.

The lesson is broader: deriving works when there is a single obvious implementation. For traits where the implementation involves design choices (Display, custom comparison, custom iteration), the developer should write the implementation explicitly.

The `thiserror` crate (Module 10) provides a way to derive `Display` for error enums, but it is not part of the standard library. The crate makes the design choice ("Display says X for variant A, Y for variant B") explicit in attributes, which is more flexible than a built-in derive could be.

**3. Why are `Hash` and `Eq` typically derived together?**

Using a type as a `HashMap` key requires both:

- `Hash` to compute the hash of a key (used to find the bucket).
- `Eq` (and therefore `PartialEq`) to compare keys for equality (used to confirm a key match within a bucket).

The hash and equality must be consistent: if `a == b`, then `hash(a) == hash(b)`. If you derive both, the compiler-generated implementations satisfy this constraint automatically. If you implement them manually, you must ensure consistency yourself.

The reason hash maps require both is that hashing alone is not a guarantee of equality (different keys can hash to the same bucket due to collisions). The implementation uses the hash for fast lookup and equality for verification.

The `Eq` trait is a marker trait that says "every value equals itself" (no NaN-like exceptions). This is required because hash maps need a real equivalence relation. Floating-point types do not implement `Eq` (because `NaN != NaN`), which is why you cannot use `f64` as a `HashMap` key directly.

The `PartialEq + Eq + Hash` combination is so common that you will see it on nearly every type that might be used as a key. The standard derive for ID-like wrapper types is exactly this set plus `Debug` and `Clone`:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct UserId(u64);
```

This single line makes `UserId` work as a HashMap key, comparable, cloneable, and printable. It is the canonical "value object" derive set.

---

## Exercise 5 -- Defining and Using Enums

### Checkpoints

**1. Why does Rust support all four variant shapes (struct-like, tuple-like, multi-field, unit)?**

The four shapes correspond to different ways data is naturally structured. The language could have required one canonical form (always struct-like, or always tuple-like), but each shape is awkward for some cases.

- **Struct-like variants** (`Click { x: i32, y: i32 }`) are clearest when fields have meaningful names and there are several of them.
- **Tuple-like variants** (`KeyPress(char)`, `Resize(u32, u32)`) are concise when the meaning is universally understood from context (a single character, a width and height).
- **Unit variants** (`Quit`) carry no data; they are pure markers.

A rule that forced everything to one shape would create awkwardness. Forcing all variants to be tuple-like would mean writing `Click(i32, i32)` and losing the field names. Forcing all variants to be struct-like would mean writing `Quit { }` (empty braces) and `KeyPress { key: char }` for cases where the bare form is clearer.

The flexibility costs nothing at the language level (the compiler handles all shapes uniformly) and lets each variant choose the shape that fits its data. This is consistent with Rust's broader philosophy of giving developers options when no single answer is obviously correct.

**2. Why do `Event::Click { .. }` and `Event::KeyPress(_)` use different syntax?**

The syntax mirrors the variant's shape. The pattern uses the same syntax as the construction.

- `Event::Click { x: 100, y: 200 }` constructs a struct-like variant. `Event::Click { x, y }` destructures it. `Event::Click { .. }` matches without binding.
- `Event::KeyPress('a')` constructs a tuple-like variant. `Event::KeyPress(key)` destructures it. `Event::KeyPress(_)` matches without binding.

The patterns are not arbitrary; they parallel the construction syntax. Once you know how to construct a variant, you know how to destructure it. The `..` and `_` placeholders fit naturally into each form.

This is the same parallelism between construction and destructuring that runs through patterns in general. Tuples, structs, and enums all use the same shapes for both purposes. Learning one direction teaches you the other.

**3. Why are `Option<T>` and `Result<T, E>` separate types rather than one general enum?**

They convey different intent. `Option` says "a value or nothing"; `Result` says "success or failure with an error value." Both are two-variant enums, but the variants mean different things.

A general enum like `Either<A, B>` exists in some libraries (and in some languages), but it does not convey the same intent. An `Option<i32>` clearly says "an integer or no integer." An `Either<i32, ()>` would convey the same information but require readers to interpret what `()` means.

Beyond intent, the two types have different APIs:

- `Option<T>` has methods like `unwrap_or`, `is_some`, `or_else` that assume the "no value" interpretation.
- `Result<T, E>` has methods like `map_err`, `ok` (converts to Option), and integration with the `?` operator for error propagation.

The methods on each type are tailored to the intent. A unified type would have a more generic API that does not match either case as well.

The consistency across the standard library (and across user code that follows the convention) is a real benefit. When you see `Option<T>`, you immediately know what it means. When you see `Result<T, E>`, you know it is about success/failure. Separate types make the meaning unambiguous in a way a single more general type could not.

---

## Exercise 6 -- Exhaustive Matching and `if let`

### Checkpoints

**1. What kind of bug does the exhaustiveness check catch?**

It catches the "forgot to handle a case" bug, especially when an enum is extended after its first uses are written.

Consider a real-world scenario: a codebase has a `Status` enum with five variants. There are 50 places in the code that match on `Status`. A new variant `Pending` is added. The compiler immediately flags every one of those 50 places as non-exhaustive. The developer cannot accidentally ship code that ignores the new variant.

In a language without exhaustiveness checking, the equivalent change would compile cleanly. The new variant would be silently treated as "fall through to the default case," which might do nothing, return a wrong value, or crash. The bug would only appear at runtime, possibly only on rare inputs.

Exhaustiveness checking moves this entire category of bugs to compile time. The cost is occasional explicit `_` arms (when you genuinely want to handle remaining cases the same way). The benefit is that "what about the new case?" is asked and answered before deployment.

This kind of compile-time enforcement is one of the strongest arguments for using enums in Rust over flag fields, sentinel values, or class hierarchies. The compiler is a relentless reviewer who never forgets to check that every case is handled.

**2. What problem does the catch-all (`_`) cause when adding new variants?**

The catch-all defeats exhaustiveness checking for that match. With `_`, any new variant is silently caught by the existing default arm, which might handle it incorrectly.

Suppose you have:

```rust
match status {
    Status::Active => "running",
    Status::Failed(_) => "error",
    _ => "other",                // catches Inactive and Pending
}
```

When `Status::Cancelled` is added, the `_` arm silently catches it as "other," which might not be correct. The compiler does not flag this; the code compiles fine. The semantic bug ("Cancelled should produce a different message") goes undetected.

The fix is to be explicit. Either list every variant individually (verbose but safe), or use the `_` only when you genuinely mean "anything else, handled the same way." The trade-off is verbosity vs safety; for important state machines and core logic, verbosity is usually worth it.

A useful pattern: name the catch-all variable rather than using `_`, and use it in the body. This forces you to at least think about what "the rest" means:

```rust
other_status => format!("unhandled status: {other_status:?}"),
```

This still catches all unmatched variants, but the body does something with them, which makes it harder to silently miss a new case.

**3. Could `if let` be made exhaustive? Why was it not?**

The language could have required `if let` to handle all variants, but doing so would defeat the point of `if let`. The whole purpose of `if let` is to be concise when you only care about one variant.

A hypothetical exhaustive `if let` would look something like:

```rust
if let Some(value) = result {
    // ...
} else if let None = result {
    // ...
}
```

This is just `match` with extra steps. The conciseness of `if let` comes from saying "if the value matches this pattern, do something; otherwise, do nothing or do the else block."

The trade-off the language makes: `if let` is not exhaustive, so it can be silently incomplete when an enum is extended. `match` is exhaustive, so it is safer but more verbose for one-variant cases. The developer chooses based on whether exhaustiveness matters for the situation.

For `Option` (always two variants, one of which is "no value"), `if let` is essentially safe; no future variant can be added. For larger enums, `match` is usually the better choice.

A new feature, `let else` (Module 9), splits the difference: `let Some(x) = result else { return; };` extracts the value or bails out, keeping the structure linear without requiring a full match. It is exhaustive in the sense that the else branch must handle the failure, but it does not require listing every variant.

---

## Exercise 7 -- Destructuring and Pattern Guards

### Checkpoints

**1. Could you achieve the `@` binding's effect with a guard? What is the difference?**

Yes, in many cases. The `@` binding:

```rust
n @ 0..=9 => format!("single digit: {n}"),
```

Could be rewritten as:

```rust
n if n >= 0 && n <= 9 => format!("single digit: {n}"),
```

Both work. The differences:

- **Pattern matching power.** The `@` binding uses a pattern (the range), which the compiler can analyze structurally. The compiler knows ranges are mutually exclusive in match arms; for guards, it cannot infer this. Pattern-based exhaustiveness checks work better with patterns than with guards.
- **Readability.** The `@` form puts the test in the pattern position; the guard puts it in a conditional. For purely structural tests (a range, a literal, a variant), the pattern form is more declarative.
- **Compiler optimization.** Patterns are often optimized into jump tables or efficient comparisons. Guards are arbitrary boolean expressions that may not optimize as well.

The choice is mostly stylistic, but the `@` form is preferred when the test is structural. Reserve guards for tests that pure patterns cannot express (e.g., testing one field against another, checking external state).

**2. Why does the compiler not automatically dereference in pattern guards?**

The compiler does perform some auto-dereferencing in patterns, but pattern guards are arbitrary boolean expressions, and inside an expression, the compiler cannot guess what you mean.

In a pattern like `Event::Failed { user_id, attempt }` matching on `&Event`, the bindings `user_id` and `attempt` are references (`&u32`). This is consistent: when you match on a reference, the bindings are references.

In the guard `if *attempt >= 3`, the comparison is between `&u32` and `u32`. The compiler does not auto-deref in this position because:

- The guard could be doing things other than dereferencing (calling methods, passing to functions).
- Auto-dereferencing in expressions could be ambiguous in some cases.
- Explicitness is consistent with Rust's broader design: the language prefers visible operations to implicit ones.

The fix is to add `*` explicitly. Some patterns also let you write `attempt @ ..` to explicitly bind by value (which would only work for `Copy` types). For most cases, the `*` in the guard is the natural fix.

A related point: the experimental `match ergonomics` feature (already in the language) does some auto-dereferencing for patterns, but not for guards. The language's choice to keep guards explicit reflects the trade-off between convenience and clarity.

**3. Why must the `let else` block diverge?**

The `let else` form binds a name from a refutable pattern. If the pattern matches, the name is bound and the code continues. If the pattern does not match, the else block runs. After the `let else`, the compiler must guarantee that the binding is valid; otherwise, code referring to the binding would be unsound.

If the else block could "fall through" (just complete normally), the binding would be undefined after a failed match, which is exactly the situation Rust's type system prevents. The compiler would have to track which paths lead to "binding valid" and which do not, complicating both the language and the analysis.

By requiring the else block to diverge (return, break, continue, panic, or call a `!`-returning function), the compiler can assume the binding is always valid after the `let else`. Code after that line uses the binding without further checks.

This is the same principle that makes early returns in functions sound: the compiler knows execution does not continue after `return`, so it does not need to worry about the state of variables that would have been initialized later.

`let else` was added to Rust in 2022 specifically to handle the "extract or bail out" pattern cleanly. Before it, the equivalent code required deeply-nested `if let`s or explicit `match` arms with returns.

---

## Exercise 8 -- Putting It Together

### Task 8.2 -- Pattern identification

1. **Struct with owned data and a collection:** `Machine` has `name: String` and `history: Vec<State>`.

2. **Enum with variants of different shapes:** `State` has unit variants (`Idle`, `Stopped`), struct-like variants (`Running`, `Paused`).

3. **Tuple-style variant with a single field:** `Event::Pause(String)`.

4. **Variant with named fields:** `State::Running { iteration: u32 }` and `State::Paused { reason: String }`.

5. **Unit-like variant:** `State::Idle`, `State::Stopped`, `Event::Start`, `Event::Resume`, `Event::Stop`, `Event::Tick`.

6. **Method with `&self`:** `next_state(&self, ...)`, `summary(&self)`.

7. **Method with `&mut self`:** `handle(&mut self, ...)`.

8. **Associated function as constructor:** `Machine::new(name: String)`.

9. **Match destructuring named fields:** `State::Running { iteration }` in `next_state` and `State::Paused { reason }`.

10. **Match on tuple of two values:** `match (&self.state, event)` in `next_state`.

11. **`if let Err(e) = ...`:** The check `if let Err(e) = machine.handle(&invalid_event)`.

12. **`while let` walking iterator:** `while let Some((i, state)) = iter.next()` at the end of `main`.

13. **`#[derive]` on multiple traits:** `#[derive(Debug, Clone, PartialEq)]` on `State`.

14. **`|` for multiple patterns:** `(State::Running { .. }, Event::Stop) | (State::Paused { .. }, Event::Stop)` in `next_state`.

### Task 8.3 -- Predict and verify

**Change A:** Remove the catch-all arm `(state, event) => Err(...)` from `next_state`.

**Prediction:** This will fail to compile with a non-exhaustive match error. The match arms above explicitly list valid transitions (Idle+Start, Running+Tick, Running+Pause, Paused+Resume, Running+Stop, Paused+Stop). Any other combination (Idle+Pause, Stopped+Start, etc.) would be unmatched.

**Result:** As predicted, this fails to compile. The error is similar to:

```
error[E0004]: non-exhaustive patterns: `(&Idle, &Pause(_))`, `(&Idle, &Resume)`, ...
```

The compiler lists every uncovered combination. This is exactly the value of exhaustiveness checking: it forces you to either handle every case explicitly or use a catch-all. Forgetting a case is impossible.

The lesson: when matching on tuples of enums, the number of combinations grows as the product of the variant counts. The compiler tracks all of them. Use a catch-all when most combinations should be handled the same way, or list each combination explicitly when each one needs its own handling.

**Change B:** Use nested matches instead of a tuple match.

**Prediction:** This will compile, but it will produce duplicated code paths. Each outer arm needs to handle every relevant Event variant, and the catch-all logic must be repeated.

**Result:** As predicted, this works but is more verbose. The nested form looks like:

```rust
match &self.state {
    State::Idle => match event {
        Event::Start => Ok(State::Running { iteration: 0 }),
        _ => Err(format!("invalid: {event:?} from Idle")),
    },
    State::Running { iteration } => match event {
        Event::Tick => Ok(State::Running { iteration: iteration + 1 }),
        Event::Pause(reason) => Ok(State::Paused { reason: reason.clone() }),
        Event::Stop => Ok(State::Stopped),
        _ => Err(format!("invalid: {event:?} from Running")),
    },
    // ...
}
```

This compiles but has duplicated structure: each state has its own catch-all error arm. The tuple match handles the same logic with one shared catch-all arm.

The lesson: tuple matches are the right tool for "transition tables" where you need to consider combinations of two pieces of state. Nested matches work but are typically more verbose and harder to maintain.

**Change C:** Add `State::Failed(String)` to the enum but do not update other code.

**Prediction:** This will fail to compile. The `next_state` function's match would now be non-exhaustive (the new variant is not handled in any of the patterns); the `summary` method's match would also be non-exhaustive.

**Result:** As predicted, multiple errors. The compiler points to every match expression that needs updating:

```
error[E0004]: non-exhaustive patterns: `&Failed(_)` not covered
  --> src/main.rs:42:20
```

The error appears in both `next_state` (where it would need to handle transitions involving `Failed`) and `summary` (where it would need to format `Failed`).

The lesson: this is the value of exhaustiveness checking. When you add a variant, the compiler tells you exactly which places in the codebase need to be updated. There is no chance of forgetting a place. Refactoring with confidence becomes possible.

In a language without exhaustiveness, you would need to manually search for every match (or switch, or if-chain) on the type and update each one. Some would inevitably be missed, with the bug appearing only in production.

### Checkpoints

**1. What does the tuple match enable that nested matches do not?**

The tuple match `match (&self.state, event)` lets you express transitions that depend on combinations of two values cleanly. Each arm can:

- Match specific combinations of state and event.
- Use the `|` operator to handle multiple combinations the same way (e.g., `(Running { .. }, Stop) | (Paused { .. }, Stop)`).
- Use a single catch-all arm for all invalid combinations.

Nested matches require you to choose an outer dimension (state or event), and the inner match must handle the other dimension separately. This duplicates the structure: each outer arm has its own catch-all, its own error formatting, its own boilerplate.

Tuple matches are also more compact. The number of patterns equals the number of valid transitions plus one catch-all. Nested matches multiply: outer arms times inner arms.

For state machines and similar transition logic, tuple matches are usually clearer. The match arm reads like a row in a transition table.

**2. What changes if `handle` returned `()` instead of `Result`?**

The `?` operator would not work. `?` requires the enclosing function to return a `Result` (or `Option`, or another type implementing `Try`). If `handle` returned `()`, the `?` would be a compile error.

The function would need to handle the error explicitly:

```rust
fn handle(&mut self, event: &Event) {
    let next_state = match self.next_state(event) {
        Ok(s) => s,
        Err(e) => {
            eprintln!("ignoring invalid transition: {e}");
            return;
        }
    };
    self.history.push(self.state.clone());
    self.state = next_state;
}
```

The structure is more verbose. The error must be handled inside `handle`, either by logging and returning, panicking, or some other action. The caller cannot decide what to do with the error.

This is one of the reasons functions return `Result`: it pushes the error-handling decision to the caller. The caller might want to log and continue, abort, retry, or do something else. A function that handles the error internally makes the choice for the caller.

For `Machine::handle`, returning `Result` is the right choice because invalid transitions are something the caller might reasonably want to handle (or ignore, depending on context).

**3. Could `summary` return `&str` instead of `String`?**

For some implementations, yes. For the current implementation, no.

The current implementation builds a fresh `String` using `format!`. This string is owned by the function; returning `&str` to it would require the string to outlive the function, which it does not.

To return `&str`, the function would need one of:

- A statically known set of strings (`&'static str`), where the return value is a literal or a reference to a static. This works for fixed text but cannot incorporate dynamic data like `iteration` or `reason`.
- Borrowing from the struct's data. If the struct stored a pre-computed summary string, the method could return `&str` borrowing from that field. This is sometimes appropriate but adds storage cost.
- Borrowing from a parameter. If `summary` took a buffer parameter, it could write into the buffer and return a slice of it.

In practice, returning `String` is usually the right choice when the function constructs a new string. The cost (one allocation) is small, and the API is straightforward. Returning `&str` is appropriate when the result is genuinely a view into existing data, not a new construction.

This is an instance of the broader principle from Module 6: take `&str` parameters, return `String` for new strings.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

Common patterns:

- Developers from C/C++ find structs natural (they look like familiar struct definitions) but are surprised by enums-with-data, which have no direct C equivalent. The unification of "tagged union" with pattern matching takes time.
- Developers from Java/C# find methods and constructors familiar, but the absence of inheritance and the trait-based approach to polymorphism is unfamiliar. Pattern matching is much more powerful than `switch`, and the exhaustiveness check feels strict at first.
- Developers from Python/JavaScript find struct definition more verbose than dictionaries but are usually quick to appreciate the clarity. Enums are revelatory: "I never realized how often I was using strings or dicts to fake this."
- The `Option` and `Result` patterns initially feel verbose but become natural quickly. Most students report "I cannot imagine going back" within a few weeks of practice.

The most common reflection: "Pattern matching is more powerful than I expected, and I now see how to model many problems with enums that I would have used class hierarchies for in other languages."

**2. Cross-language comparison.**

Sample answer focusing on Java's sealed classes:

"Java added sealed classes in 16 to model 'one of these specific subclasses,' which is similar to Rust's enums. The differences:

- **Memory.** A Java sealed class hierarchy uses an object reference (8 bytes) plus the heap allocation of each instance. Rust's enums are inline, with size equal to the largest variant plus a small discriminant.
- **Pattern matching.** Java's pattern matching (added in Java 21) is improving but does not destructure variants as fluidly as Rust's. The exhaustiveness check is also less mature.
- **Methods.** Java's hierarchy can use inheritance for shared methods. Rust's enums use a single `impl` block; shared logic is expressed by matching, not by inheriting.

What Rust gains: efficiency (no heap), compile-time exhaustiveness, integrated pattern matching. What Rust loses: the OO hierarchy for shared behavior is more awkward to express; if you have many variants with shared methods, the match expressions can grow long.

The trade-off favors Rust for value-like types (events, states, results) where efficient memory layout and exhaustive matching matter. It favors Java's hierarchy for object-like types where shared behavior is the dominant concern."

**3. Struct vs enum decision.**

Use a struct when:

- All instances of the type have the same shape (the same fields with the same types).
- The type represents a single concept with multiple aspects (a User has a name, email, and ID; these always exist together).
- Optional features can be modeled with `Option<T>` fields rather than separate variants.

Use an enum when:

- Instances come in distinct kinds, each with potentially different data (events, errors, parser tokens).
- Pattern matching on the kind would be awkward as a series of optional fields.
- The kinds have relationships that should be expressed as a union (e.g., either success or failure, never both).

Sometimes a hybrid: a struct that contains an enum field. This works when the type has both common data (always present) and kind-specific data (varies):

```rust
struct Event {
    timestamp: u64,                  // always present
    user_id: u32,                    // always present
    payload: EventPayload,            // varies by kind
}

enum EventPayload {
    Login,
    Logout,
    Purchase { item_id: u32, amount: f64 },
}
```

The decision is often clearer when you write out the alternatives. A struct with five `Option` fields where exactly one is `Some` is almost always better as an enum. A struct with one optional field is fine; introducing an enum would be over-engineering.

The pattern recognition develops with practice. After a few projects, the right choice usually feels obvious from the problem description.

---

*End of Lab 6 Solutions*
