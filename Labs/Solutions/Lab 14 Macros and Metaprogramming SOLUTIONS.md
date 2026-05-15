# Module 18: Macros and Metaprogramming
## Lab 14 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Your First Declarative Macros

### Checkpoints

**1. The purpose of the parentheses in `say_hello!()`?**

The parentheses contain the arguments to the macro. For `say_hello!()`, the arguments are empty (no input). The pattern `() => { ... }` in the macro definition specifically matches "no input."

The parentheses are required syntactically. `say_hello!` (without parentheses) is not a valid macro call. Macros must be invoked with one of three delimiters:

- `say_hello!()`  (parentheses)
- `say_hello![]`  (square brackets)
- `say_hello!{}`  (curly braces)

All three delimiters work for the same macro; the choice is stylistic. By convention:

- `()` for function-like invocations: `println!("...")`, `assert_eq!(a, b)`.
- `[]` for collection-like macros: `vec![1, 2, 3]`.
- `{}` for block-like macros that contain statements: `lazy_static! { ... }`.

The delimiter does not affect parsing of the macro's arguments (the macro itself defines what to match inside). It is purely a syntactic choice that affects how the call site reads.

So `say_hello!()` and `say_hello![]` and `say_hello!{}` are all equivalent. The community convention has settled on `()` for simple invocations.

**2. What happens if two rules could match the same input?**

The compiler tries rules in declaration order. The first rule that matches is used; later rules are not even attempted.

This means rule order matters when rules overlap. Consider:

```rust
macro_rules! example {
    ($x:expr) => { println!("expression: {}", $x) };
    ($x:expr, $y:expr) => { println!("two expressions: {}, {}", $x, $y) };
}
```

Calling `example!(1, 2)` matches the second rule (two expressions). The first rule could not match (it expects exactly one expression, no comma).

But for overlapping rules where both could match:

```rust
macro_rules! example {
    ($x:expr) => { println!("any expression: {}", $x) };
    ($x:literal) => { println!("a literal: {}", $x) };  // never reached
}
```

The first rule (`expr`) matches anything the second rule could match (`literal` is a subset of `expr`). The second rule is never used.

The compiler does NOT warn about unreachable rules; it just silently uses the first matching rule. This is a real footgun: you can have rules that look like they handle specific cases but never run.

The fix is to order rules from most specific to most general. The more restrictive pattern should come first. For the example above, swapping the order would make the `literal` rule fire for literals, with `expr` as the fallback for everything else.

The compiler's behavior is deterministic (always picks the first match), but the responsibility for good ordering falls on the macro author.

**3. When is `+` preferable to `*` for repetition?**

`*` allows zero or more matches; `+` requires at least one. The choice depends on whether "zero matches" is a valid input.

`*` (zero or more) is appropriate when:

- An empty input is meaningful and valid.
- The macro produces meaningful output even with no matches.

Example: `vec![]` with `$(...)*` accepts zero items, producing an empty vector. This is intentional.

`+` (one or more) is appropriate when:

- An empty input would be invalid or meaningless.
- The macro requires at least one match to do its job.

Example: a `max_of!(...)` macro that takes a series of values to compare. With zero values, what would it return? `+` forces the user to provide at least one value, catching the error at compile time.

The choice has practical consequences:

- With `*`, the user gets the macro's "empty" expansion (which might be confusing or wrong).
- With `+`, the user gets a "no rules matched" compile error for empty input.

For most macros, `*` is the default. For macros that genuinely require at least one input, `+` provides a small but real safety improvement: invalid empty calls fail at compile time with a clear error.

---

## Exercise 2 -- Macro Hygiene

### Checkpoints

**1. Why is variable collision a problem in C-style macros?**

In C-style preprocessors, macros are textual substitution. The macro's body is literally inserted at the call site. This causes name collisions because both the macro's variables and the caller's variables exist in the same scope after substitution.

Consider a C-style macro:

```c
#define DOUBLE(x) do { int temp = x; temp * 2; } while(0)
```

If the caller has:

```c
int temp = 100;
int result = DOUBLE(temp);
```

After preprocessing, this becomes:

```c
int temp = 100;
int result = do { int temp = temp; temp * 2; } while(0);
//                ^^^^^^^^^^^^^^^ ambiguous
```

The macro's `int temp` shadows the caller's `temp`, but the right-hand side `= x` was substituted with `= temp` (the caller's name). The result depends on language scoping rules and is usually wrong or undefined.

The bug is invisible at the call site. The user sees `DOUBLE(temp)` and expects it to work; the macro's internal naming is hidden. The bug manifests as subtle wrong results.

The traditional fix is naming conventions: every macro variable starts with an unusual prefix (`__macro_temp` instead of `temp`). This relies on discipline; nobody enforces it. Bugs slip through.

Rust's hygiene rule eliminates the entire class. The macro's `temp` and the caller's `temp` are syntactically the same name but semantically distinct. The compiler tracks which "syntax context" each name originated in. Two `temp` names from different contexts are treated as if they had different names.

This is a real advantage over C-style macros. You can write macros without worrying about variable name collisions; the compiler handles it. The cost is some implementation complexity (the compiler must track contexts), but users see none of it.

**2. Why does the `$name:ident` escape hatch work?**

The hygiene rule is: identifiers introduced by the macro have one syntax context; identifiers passed in by the caller have another. The two are distinct.

When the macro writes `let temp = ...`, the `temp` identifier is introduced by the macro itself. It belongs to the macro's syntax context. The caller's `temp` (in a different context) is unaffected.

When the macro accepts `$name:ident` and uses `let $name = ...`, the `$name` comes from the caller's input. The identifier belongs to the caller's syntax context, not the macro's. So the binding is visible to the caller.

In short:

- `let temp = ...` (literal identifier in macro body): macro's context, isolated from caller.
- `let $name = ...` (identifier from caller's input): caller's context, visible to caller.

The macro author chooses which to use. The default (literal identifier) gives hygiene; the escape hatch (input identifier) breaks hygiene deliberately.

The mechanism is subtle but well-defined. It is what makes `vec!` work: `vec![1, 2, 3]` introduces a new identifier `v` internally (or whatever the macro uses), but `let mut v = vec![...]` works because the outer `v` is in the caller's context.

For most macros, the default hygiene is what you want. The escape hatch is rare; it appears in macros that genuinely need to introduce caller-visible names (debugging tools, scoped variable creators).

**3. What is gained that justifies the restriction?**

The restriction is: macros cannot accidentally collide with caller names.

This is genuinely valuable:

- **Maintainability.** You can refactor a macro's internals without worrying about every caller.
- **Composability.** Two macros from different sources can be used together without their internal names interfering.
- **Predictability.** Reading a macro call, you do not have to know the macro's implementation to use it safely.

The cost is real but small:

- Macros cannot "share" variables with callers without explicit input identifiers.
- Some patterns that work in C-style macros (introducing a "temp" variable that the caller can use) require explicit support.

For most macros, the default hygiene is exactly what you want. The cases where you DO want shared variables (rare) are still possible with the explicit identifier mechanism.

The trade-off favors safety. Subtle name-collision bugs are nearly impossible in Rust macros, removing a class of bugs that plagued C-style macros for decades. The cost (explicit identifier parameters for the rare cases that need them) is small and well-documented.

---

## Exercise 3 -- The Common Built-In Macros

### Checkpoints

**1. Why is `vec![value; count]` a macro and not a function?**

The semicolon syntax (`value; count`) is not a valid function call. Functions accept comma-separated arguments. A function-based equivalent would have to use commas:

```rust
Vec::repeat(value, count)            // hypothetical function
```

But the standard library prefers the macro for several reasons:

- **Unified syntax.** Three forms (`vec![]`, `vec![1, 2, 3]`, `vec![0; 10]`) share one syntactic pattern. With functions, you would need three separate functions.
- **Compile-time type inference.** The macro can determine the element type from the values. A function returning `Vec<T>` would need explicit type parameters or coercion.
- **Convention.** The macro form (`vec![...]`) is universally recognized; users do not need to remember different function names.

The alternative would be `Vec::new()` + `push()` + a loop, but that is verbose. The macro provides cleaner syntax for a common operation.

Note that `Vec` itself does provide some related methods:

- `vec.resize(count, value)` mutates an existing vector to have `count` copies of `value`.
- The `iter::repeat(value).take(count).collect()` pattern with iterators.

But for "construct a vector with a repeated value," `vec![value; count]` is the idiomatic choice. The macro form is recognized at a glance.

**2. What bug does compile-time format checking catch?**

The bug is mismatched format strings: too few or too many `{}` placeholders compared to arguments, or mismatched types.

Without compile-time checking (in languages like C with `printf`):

```c
printf("Hello, %s! You are %d.\n", name, age);
```

The format string says "expect a string, then an integer." But the compiler does not verify this. If `age` is a string instead of an integer, the result is undefined behavior at runtime: a crash, garbage output, or worse.

In C with strict warnings (`-Wformat`), the compiler will sometimes catch these. But the check is optional, the warnings are not always enabled, and the check does not work for runtime-built format strings.

In Rust:

```rust
println!("Hello, {}! You are {}.", name, age);
```

The compiler checks the format string at compile time. It verifies:

- Number of placeholders matches number of arguments.
- Each argument's type implements the required formatting trait (`Display`, `Debug`, etc.).
- Format specifiers (precision, padding, etc.) are valid.

If any check fails, the code does not compile. The error message tells you exactly what is wrong.

This eliminates a category of runtime bugs entirely. With C-style printf, you can deploy code with mismatched format strings; bugs appear when the offending code runs. With Rust's macros, the bug is impossible: the code does not compile if the format string is wrong.

In other languages without this check, the bug appears at runtime, typically in production after deployment. Users see crashes or wrong output. With Rust's macros, the same bug is caught immediately, during development, with a clear error message.

**3. Why can macros capture source text but functions cannot?**

This is about WHEN the work happens. Functions execute at runtime, with access only to the values passed in. Macros expand at compile time, with access to the source code (the actual tokens).

For `dbg!(x * 2)`:

- A function with signature `fn dbg<T>(value: T) -> T` would receive only the result of `x * 2` (say, the number 10). It cannot reconstruct the expression "x * 2" because that information was lost when the expression was evaluated.
- A macro `dbg!($expr)` sees the actual tokens `x * 2`. It can use them as a label in the output.

The macro's expansion is something like:

```rust
let result = x * 2;
eprintln!("[{}:{}] x * 2 = {:?}", file!(), line!(), result);
result
```

The string "x * 2" comes from the macro's `stringify!($expr)` (or the equivalent). The macro substitutes the original tokens as a string literal.

Functions cannot do this because the value-passing convention loses the original syntax. By the time the function receives the value, the syntax tree has been compiled away.

This is the fundamental difference between macros and functions:

- **Macros are syntax transformers.** They take tokens and produce tokens.
- **Functions are value transformers.** They take values and produce values.

For "I need to know what the user wrote, not just what it evaluates to," macros are necessary. `dbg!`, `assert_eq!` (which shows both the expressions and values in failure messages), `format_args!` (the foundation of `println!`), and others rely on this.

---

## Exercise 4 -- Derive Macros for Your Own Types

### Checkpoints

**1. Why is the standard derive set so common? When to NOT include Copy?**

The standard derive set (`Debug, Clone, Copy, PartialEq, Eq, Hash`) gives a type the abilities of a "value type":

- **Debug** for `{:?}` printing during development.
- **Clone** for explicit duplication.
- **Copy** for implicit duplication on assignment (zero-cost for trivial types).
- **PartialEq** for `==` and `!=`.
- **Eq** for "equivalence-relation" semantics (required for HashMap keys).
- **Hash** for hashing (required for HashMap keys).

Together, these let a type behave like a number or a string for most purposes. You can print it, copy it, compare it, and put it in a HashMap.

When to omit `Copy`:

- **The type contains non-`Copy` fields.** If any field is `String`, `Vec`, `Box`, `Rc`, etc., the type cannot be `Copy` (the compiler enforces this). Use only `Clone`.
- **The type is conceptually expensive to copy.** Even if technically `Copy`-able, very large types might benefit from `Clone` (forcing explicit duplication).
- **The type has semantic invariants that should not be silently duplicated.** A unique identifier, a connection handle, a sensitive value. Making it `Copy` would mean it can be duplicated accidentally; `Clone` requires explicit `clone()` calls.
- **You want to enforce uniqueness.** Some "ID" types are deliberately `!Copy` so that handing one to a function explicitly consumes it.

For "small value types" (integers, simple structs), `Copy` is appropriate. For "owned data types" (vectors, strings, complex objects), `Clone` is better.

The decision is part of the type's API design. Making something `Copy` says "this is cheap to duplicate; do not worry about it." Making something `Clone`-only says "duplication is possible but worth thinking about."

**2. Why is the `Copy` requirement strict?**

`Copy` semantics is "bitwise copy with no special handling." When you write `let b = a`, the bits of `a` are copied to `b`. Both `a` and `b` are valid afterward (no move semantics).

For types with heap-allocated data (like `String`), bitwise copy is wrong:

- `String` contains a pointer to heap data, plus length and capacity.
- Bitwise copying the `String` makes both `a` and `b` point to the same heap allocation.
- When both `a` and `b` go out of scope, both try to free the same heap memory: double-free, undefined behavior.

The compiler refuses `#[derive(Copy)]` for types with non-`Copy` fields to prevent this category of bug. It is not arbitrary; it is preventing a real safety issue.

If you bypass the check (which you cannot, but hypothetically):

```rust
let s1 = String::from("hello");
let s2 = s1;       // hypothetical Copy semantics
// s1 and s2 both point to the same buffer
drop(s1);          // frees the buffer
println!("{s2}");  // use-after-free: undefined behavior
```

This is exactly the kind of bug Rust's safety guarantees prevent. The `Copy` requirement (all fields must be `Copy`) is part of the enforcement.

For types where bitwise copy IS correct (integers, plain structs of integers, pointers without ownership semantics), `Copy` is fine. The compiler verifies this and lets you derive it.

The rule is: `Copy` is for types whose values are bit-exact independent copies. `Clone` is for types that require some logic (allocation, reference counting, etc.) to duplicate.

**3. Display vs Debug: design intent?**

`Debug` is for developers, debugging, internal tooling. Its purpose:

- Show the structure of values for diagnosis.
- Be uniform across all types (so any type can be Debug-printed).
- Include all fields, leaving little to interpretation.

This means there is a canonical format: "type name { field: value, ... }" for structs, "Variant(values...)" for enums, etc. The derive can generate this format automatically; there is one right answer.

`Display` is for users, user-facing output. Its purpose:

- Present a value in a human-readable form.
- Vary by type's semantics (a color might display as "red", "#FF0000", "rgb(255,0,0)", etc.).
- Reflect the type's intent.

There is no canonical format for arbitrary types. `Greeting { name: "Alice" }` could display as "Hello, Alice!", "Greeting to Alice", "alice", or anything else. The author must choose.

The derive system can handle `Debug` because there is a canonical format. It cannot handle `Display` because there is not.

For some specific patterns, you could derive Display. The `derive_more` crate provides this for specific cases:

```rust
#[derive(derive_more::Display)]
#[display(fmt = "Hello, {}!")]
struct Greeting(String);
```

But this requires specifying the format, which is exactly the choice that cannot be defaulted. `derive_more` is third-party; the standard library deliberately does not provide `Display` derivation.

The lesson generalizes: derives are appropriate when there is one obvious correct implementation. They are not appropriate when implementation choices need to be made.

`Clone` can be derived (clone every field). `Default` can be derived (use each field's default). `Hash` can be derived (hash each field). All have canonical implementations.

`Display`, `From` (target type unclear), `Iterator` (semantic-specific iteration), and similar cannot be derived. They require choices the compiler cannot make.

---

## Exercise 5 -- Third-Party Derive Macros

### Checkpoints

**1. What requires a procedural macro for serde?**

The derive needs to inspect the struct's fields and generate code that handles each one according to its type. This is beyond what `macro_rules!` can do.

Specifically:

- **Field iteration.** The macro must enumerate the struct's fields. `macro_rules!` cannot inspect AST structure beyond simple syntactic matching.
- **Type-specific code.** For each field, the macro generates serialization code based on the field's type. For `String`, it serializes as a string. For `Vec<T>`, it serializes as an array. For `Option<T>`, it serializes as `T` or null. The macro must understand types, which `macro_rules!` cannot.
- **Recursive generation.** For nested types, the macro generates code that calls the serializer recursively. This works because the inner types also derive `Serialize`.
- **Helper attribute processing.** `#[serde(rename = "...")]`, `#[serde(default)]`, etc. The macro reads these attributes and adjusts its output.

A declarative macro can do none of this. `macro_rules!` is limited to pattern matching on tokens. It cannot:

- Determine what fields a struct has.
- Inspect a field's type to decide what code to generate.
- Iterate over multiple fields with different handling per field.
- Parse and act on attributes.

The procedural macro can do all of these because it is a full Rust program running at compile time. It has access to the parsed syntax tree (via the `syn` crate) and can generate any code (via the `quote` crate).

The result is real:

- `serde` for one struct generates dozens to hundreds of lines.
- `tokio::main` rewrites the entire `main` function.
- `sqlx::query!` parses SQL and validates against a database schema.

None of this is possible with declarative macros. Procedural macros enable a class of code generation that fundamentally requires compile-time computation.

**2. What does helper attribute support say about procedural macros?**

The attributes `#[serde(rename = "...")]`, `#[serde(default)]`, etc., look like generic Rust attributes but are specific to serde. The compiler does not understand them; they have meaning only inside serde's derive macro.

This tells you several things:

- **Procedural macros define their own DSL.** Each derive can have its own attribute vocabulary. `#[serde(...)]` is serde-specific; `#[derive(StructOpt)]` had its own; `#[derive(Parser)]` (clap) has its own again.
- **The compiler is a tool, not the final authority.** The compiler delegates handling of `#[derive(MyTrait)]` to the macro for `MyTrait`. The macro decides what attributes mean.
- **Errors come from the macro, not the compiler.** If you write `#[serde(rename = bad)]` (rename should be a string), the error comes from serde's macro at compile time. The compiler does not know what `rename` means; serde does.

This is what makes procedural macros so powerful and also so opaque. Users need to learn each crate's attribute vocabulary. The compiler cannot help directly; you must read the crate's documentation.

The flip side: this lets crates provide rich, expressive APIs that feel "native" to Rust. `#[serde(rename = "userId")]` reads like a Rust attribute; it acts like one syntactically. But the actual meaning is serde-specific.

This pattern is everywhere in modern Rust:

- `#[tokio::main]` is tokio-specific.
- `#[derive(Error)]` and `#[error("...")]` are thiserror-specific.
- `#[derive(Parser)]` and `#[arg(...)]` are clap-specific.

Each crate provides its own DSL via attributes. Users must learn each one. The flexibility makes Rust's tooling expressive at the cost of some learning curve.

**3. thiserror versus manual implementation: trade-offs?**

**thiserror (derive-based):**

```rust
#[derive(Debug, Error)]
enum MyError {
    #[error("file not found")]
    NotFound,
    #[error("invalid input: {0}")]
    InvalidInput(String),
}
```

Roughly 6 lines. The macro generates `Display` and `Error` impls.

**Manual implementation:**

```rust
#[derive(Debug)]
enum MyError {
    NotFound,
    InvalidInput(String),
}

impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            MyError::NotFound => write!(f, "file not found"),
            MyError::InvalidInput(s) => write!(f, "invalid input: {s}"),
        }
    }
}

impl std::error::Error for MyError {}
```

Roughly 15 lines for the same result.

**Trade-offs:**

For thiserror:

- **Less code to write.** Saves typing for every new error variant.
- **Less to maintain.** Adding a variant only adds one line (variant + `#[error]`).
- **Consistent style.** All thiserror-using crates look similar.
- **Helper attributes for advanced cases.** `#[from]`, `#[source]`, `#[transparent]` cover common needs.

For manual:

- **No dependency on thiserror.** One less crate to compile.
- **More explicit.** A reader can see exactly what each error formats to without learning thiserror's vocabulary.
- **More flexible.** Manual impls can handle cases that thiserror cannot express.
- **Better error messages.** Macro errors are sometimes confusing; manual code has clearer errors.

The trade-off is similar to `derive` versus manual `impl` for any trait:

- For "simple, repetitive boilerplate," the derive wins.
- For "complex, customized behavior," manual wins.

In practice, most projects use thiserror. The savings compound for projects with many error types. The dependency cost is small (thiserror is a focused, well-maintained crate).

For libraries that minimize dependencies (because every dependency adds compile time and complexity for users), manual implementations are sometimes preferred. The standard library, for example, does not use thiserror; it implements all its errors manually.

For applications, thiserror is usually the right choice. The savings are real and the cost is negligible.

---

## Exercise 6 -- Attribute and Function-Like Macros

### Checkpoints

**1. How does `#[test]` make functions discoverable?**

`#[test]` is built into the compiler (technically not a procedural macro, but it behaves similarly). When you run `cargo test`, the compiler:

1. **Scans the source for `#[test]` attributes.** Each marked function is collected into a list.
2. **Generates a test harness.** The compiler creates a `main` function (different from your normal `main`) that contains code to call each test function, capture their results, and print summaries.
3. **Builds a test binary.** This binary is what runs when you invoke `cargo test`. It contains your library code, your test functions, and the generated harness.

So `#[test]` does not modify the function itself; it adds metadata that the compiler uses elsewhere. The function looks normal in source; the behavior comes from compilation context.

Other attribute macros work similarly:

- `#[allow(...)]` and `#[warn(...)]` control compiler lints. They do not change the code; they change how the compiler reports issues.
- `#[cfg(...)]` controls conditional compilation. The function is included or excluded entirely based on the configuration.
- `#[derive(...)]` triggers code generation for the item. The item itself is unchanged; new impls are added alongside.

The pattern: attributes are metadata that other parts of the toolchain (compiler, test harness, code generators) read and act on.

This is what makes attributes flexible. The same syntactic mechanism (`#[name]`) handles many different concerns. Each attribute has its own meaning, defined by its handler.

**2. Why is `#[tokio::main]`'s rewrite acceptable?**

The transparent rewriting is acceptable for `#[tokio::main]` because:

- **The transformation is well-documented.** Tokio's documentation explains that `#[tokio::main]` creates a runtime and runs the body on it.
- **The transformation is predictable.** Every use produces the same kind of transformation; users learn it once.
- **The transformation is sensible.** The body is wrapped in a runtime, which is exactly what you would write manually. The macro saves boilerplate without adding surprises.
- **The user opts in explicitly.** Adding `#[tokio::main]` is a deliberate choice; the user accepts the transformation.

Attribute macros that DO concern me:

- **Macros that hide significant behavior.** If `#[some_attr]` rewrites the function to add caching, logging, retries, and error handling all at once, the user might not understand what their code actually does.
- **Macros without clear documentation.** Reading the source, you cannot infer what the macro does. Users must consult external docs.
- **Macros that perform side effects.** A macro that registers a function with a global registry might lead to surprising behavior if registrations are not tracked.

The general principle: attribute macros should do one obvious thing, well-documented, that the user can predict. The more the macro hides, the harder it becomes to reason about the code.

Tokio's macro hits the sweet spot: significant transformation, but well-understood and well-scoped. The user knows what to expect.

A counterexample: web frameworks that use heavy attribute macros (`#[get("/path")]`, `#[post("/path")]`, etc.) can make controllers feel magical. The transformation is opaque; you cannot tell from the source what the route registration looks like. This is a real trade-off between convenience and clarity.

For libraries you author, lean toward transparency. For libraries you use, accept that some hide complexity in exchange for convenience.

**3. Why have both declarative and function-like procedural macros?**

The two have different implementation models, even though their call sites look identical.

**Declarative macros (`macro_rules!`):**

- Pattern-matching templates.
- Defined in the same crate (or imported normally).
- Limited expressiveness (cannot do arbitrary computation).
- Fast compile (the macro itself is simple).
- Standard library, simple application macros.

**Function-like procedural macros:**

- Full Rust programs running at compile time.
- Defined in a separate crate with `proc-macro = true`.
- Unlimited expressiveness (can do anything Rust can do).
- Slower compile (the macro is itself compiled).
- Used by `sqlx::query!`, `lazy_static!`, complex DSLs.

The reason both exist:

- For simple cases (variadic args, pattern-based expansion), declarative macros are sufficient. They are easier to write, easier to debug, and faster to compile.
- For complex cases (parsing arbitrary syntax, validating against external data, generating substantial code), procedural macros are necessary. Declarative macros cannot do the work.

If only one existed:

- Only declarative: many useful macros (sqlx::query, serde derives, etc.) would be impossible.
- Only procedural: simple macros (like `vec!`) would require unreasonable infrastructure to write.

Having both lets each tool serve its appropriate use case. Users do not always know the difference (and usually do not need to); they just use the macros that exist.

For macro authors, the choice is:

- "Can I do this with pattern matching?" → declarative.
- "Do I need to inspect AST, generate from types, parse arbitrary syntax?" → procedural.

Most application code is fine with declarative macros. Procedural macros are mostly authored by library maintainers; users consume them.

---

## Exercise 7 -- Macros vs Functions vs Generics

### Checkpoints

**1. What did the macro version of max_all cost in code quality?**

The macro version was convenient at the call site, but it gave up several things:

- **Type errors point inside the macro expansion.** If you pass incompatible types, the error message shows the macro's generated code, which can be confusing. Function errors point at the call site, which is more readable.
- **IDE support is reduced.** Autocomplete, "go to definition," "find references," and other IDE features may not work as well with macros. The function's signature is clear; the macro's is not.
- **Refactoring is harder.** Renaming a parameter in a function is mechanical (the IDE handles it). Renaming a macro's metavariable might break code that uses the macro.
- **Documentation is harder.** Function signatures self-document (parameters, return types, trait bounds). Macro signatures (the matchers) are harder to read.
- **The behavior is opaque.** A function's body is visible and walkable. A macro's expansion happens at the call site; you have to mentally unroll the expansion to understand what runs.

For one-time uses or quick prototyping, the convenience might be worth the cost. For production code that others will maintain, the function version is usually better.

Some IDEs and tools (like `rust-analyzer`) have improving support for macros, but they will probably never match the support for functions. Functions are first-class language constructs; macros are a meta-feature.

**2. Why reach for macros reluctantly?**

The list of "what macros cost" answers part of this. But the bigger picture:

Macros are an escape hatch. They let you do things the language otherwise cannot do. With that power comes responsibility:

- **Maintenance burden.** Macros are harder to read, harder to modify, harder to test than functions. The code becomes more complex.
- **Onboarding cost.** New developers must learn each macro's syntax. Functions have universal conventions; macros have their own DSLs.
- **Error message degradation.** Compile errors in macro-using code often point at confusing locations. Debugging takes longer.
- **Tooling friction.** IDEs, code review tools, static analyzers all work better with functions than with macros.

For these costs to be worth it, the macro must provide value that functions cannot. The Module 18 reading lists the legitimate uses: variadic arguments, source-code capture, compile-time evaluation, DSLs, type-based code generation.

For everything else, functions and generics are better. The function's signature is documentation; the body is testable; the behavior is predictable.

A practical heuristic: write your code first as a function. If you find yourself wanting variadic arguments or compile-time behavior, consider a macro. If the function works, keep it.

This is especially true for application code. Library code, where macros enable expressive APIs, sometimes benefits from more macros. Application code usually does not need them.

**3. Why have `vec!` alongside `Vec::from`?**

Both can construct a vector. But they have different use cases:

- **`vec![1, 2, 3]`**: variadic, comma-separated values. The "list" syntax is familiar from other languages.
- **`Vec::from(array)`**: constructs from an array. Useful when you already have an array; less elegant for inline values.
- **`Vec::from(&[1, 2, 3])`**: constructs from a slice. Same as above.

The macro `vec!` is more convenient for the inline-values case. You write your values directly; the macro builds the vector.

Why not have just one? The macro is more flexible (handles `vec![]`, `vec![x; n]`, `vec![1, 2, 3]` all with one mechanism). The function form is more explicit (a function call with a known signature). Both exist for the cases where each is preferable.

The standard library is full of examples like this:

- `format!` macro for inline format strings; `String::from` function for direct construction.
- `assert!` macro for compile-time format-string checking; manual `if !x { panic!(...) }` for cases the macro does not handle.
- `dbg!` macro for source-aware debugging; `println!("{x:?}")` for fully explicit prints.

Each pair serves overlapping but distinct needs. Users learn both and pick the appropriate one. The cost (more API to learn) is small; the benefit (right tool for the job) is real.

A general pattern in Rust APIs: provide multiple ways to do common things, with each variant tuned to a specific case. Users get options without having to write workarounds.

---

## Exercise 8 -- Putting It Together

### Task 8.2 -- Pattern identification

1. **Custom declarative macro:** `macro_rules! event!` defines the event-construction macro.

2. **Multiple rules in one macro:** The `event!` macro has two rules: the basic three-argument form and the form with metadata.

3. **Repetition syntax:** Inside the metadata form, `$( $key:expr => $value:expr ),*` and `$( event.metadata.insert(...); )*` use the `$( ... ),*` and `$( ... )*` repetition patterns.

4. **Third-party derive on struct:** `#[derive(Debug, Clone, Serialize, Deserialize)]` on `Event`.

5. **Third-party derive on enum:** `#[derive(Debug, Error)]` on `EventError`.

6. **Helper attributes:** `#[error("...")]` on each variant of `EventError`; `#[from]` on the `SerializeError` variant for automatic conversion.

7. **`vec!` macro:** `let events = vec![login, upload, unauthorized];` builds the events vector.

8. **`assert_eq!` macro:** `assert_eq!(test_event.kind, "test");` verifies the event's kind.

9. **`dbg!` macro:** `let result = dbg!(process_event(event));` inspects the result.

10. **`todo!` macro:** `todo!("event aggregation across multiple events")` marks the unfinished function.

### Task 8.3 -- Predict and verify

**Change A:** Remove `#[derive(Serialize, Deserialize)]` from `Event`.

**Prediction:** The `serde_json::to_string_pretty(event)` call will fail because `Event` no longer implements `Serialize`. The error will point at the call site (in `process_event` or `main`), saying "the trait `Serialize` is not implemented for `Event`."

**Result:** As predicted:

```
error[E0277]: the trait bound `Event: Serialize` is not satisfied
   --> src/main.rs:NN:NN
    |
NN  |     let json = serde_json::to_string_pretty(event)?;
    |                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Serialize` is not implemented for `Event`
```

The error points at the call site. The compiler is helpful here: it tells you exactly which trait is missing.

If you remove only `Serialize`, you would also get a similar error for `Deserialize` later. Removing both produces an error at the first use.

The lesson: derives create implementations. Code that depends on the implementations fails to compile if the derive is removed. This is exactly what you want; the failure is local to the use site.

**Change B:** Change `$kind:expr` to `$kind:ident`.

**Prediction:** The macro expects an identifier, but the call site passes a string literal `"user.action"`. String literals are not identifiers. The macro fails to match the pattern, producing a "no rules matched" error.

**Result:** As predicted:

```
error: no rules expected the token `"user.action"`
```

The compiler tells you exactly what went wrong: the macro could not parse the input.

The lesson: fragment specifiers constrain what the macro accepts. `ident` is strict; only valid Rust identifiers match. `expr` is permissive; almost anything that can be a value matches.

For most macros, `expr` is the right specifier for "a value." Use `ident` only when you specifically need an identifier (like for a variable name to declare).

**Change C:** Add a new variant `#[error("missing field")] MissingField` without field references.

**Prediction:** The code should compile fine. The format string `"missing field"` has no placeholders, so no fields are referenced. `thiserror` generates the appropriate `Display` impl for a variant with no data.

**Result:** As predicted, the code compiles. You can use the new variant as `EventError::MissingField`, and it displays as "missing field" with no problems.

Where it gets interesting: if you wrote `#[error("missing field {0}")]`, you would need a positional field. If you wrote `#[error("missing field {name}")]`, you would need a `name` field. The format string's references determine what fields are required.

The lesson: `thiserror`'s derive is forgiving about static format strings. It only enforces field references when they exist. A simple no-placeholder format works for any variant shape.

### Checkpoints

**1. What does the custom event! macro provide?**

Two things:

- **A DSL for event construction.** The form `event!(kind, user, action, metadata: { key => value, ... })` is more readable than the equivalent function calls. Especially with multiple metadata entries, the macro's syntax is cleaner.
- **Variadic metadata.** A function would need to take a `HashMap` parameter (requiring the caller to construct it first) or be limited to a fixed number of entries. The macro accepts any number of key-value pairs.

Without the macro, the call site would look like:

```rust
let mut upload = Event::new("user.action", "bob", "upload");
upload.metadata.insert("file".to_string(), "document.pdf".to_string());
upload.metadata.insert("size_bytes".to_string(), "1024".to_string());
upload.metadata.insert("checksum".to_string(), "abc123".to_string());
```

Four lines per event. The macro condenses this to one logical block:

```rust
let upload = event!("user.action", "bob", "upload", metadata: {
    "file" => "document.pdf",
    "size_bytes" => "1024",
    "checksum" => "abc123",
});
```

The macro is genuinely useful here because the variadic metadata pattern fits a macro well. For two or three events, the savings are noticeable; for many events, they compound.

This is a sensible use of a custom macro: the syntactic improvement is real, the macro is small and focused, and the alternative (verbose construction code) is genuinely tedious.

**2. Could the derives all be implemented manually?**

Yes, every derive can be replaced with a manual implementation.

- `Debug`: write `impl fmt::Debug for Event { ... }` showing each field.
- `Clone`: write `impl Clone for Event { fn clone(&self) -> Event { ... } }` cloning each field.
- `Serialize`: write `impl Serialize for Event { fn serialize<S: Serializer>(&self, s: S) -> ... }`. This is dozens of lines of `s.serialize_struct(...).serialize_field("kind", &self.kind)...`.
- `Deserialize`: write `impl<'de> Deserialize<'de> for Event { fn deserialize<D: Deserializer<'de>>(d: D) -> ... }`. This is even more substantial.

The manual implementations would be tedious and error-prone. `Serialize` and `Deserialize` for a struct with 4 fields might be 50-100 lines each. With derives, it is one line.

Why use the derives:

- **Less code.** Instantly. Lines of code reduce by 10x or more.
- **Less maintenance.** Adding a field to the struct does not require updating the impls; the derive picks it up automatically.
- **Less risk of bugs.** Manual implementations can have subtle bugs (forgetting a field, wrong serialization format). Derives are tested by the library author.
- **Consistency with the ecosystem.** Every serde user writes the same derives; the code looks similar across projects.

When NOT to use derives:

- **You need custom serialization logic.** A field that is computed on serialization, a unique format requirement, etc. Manual impls give you full control.
- **You want to minimize dependencies.** Each derive's macro is a procedural macro, which adds compile cost. For libraries that minimize dependencies, manual impls avoid this.
- **You need behavior the derive cannot provide.** Some patterns (interning, lazy initialization on deserialize) require manual impls.

For most application code, the derives are the right choice. The standard library uses manual impls for its types (because it cannot depend on third-party derives), but applications and libraries typically use derives.

**3. What does thiserror gain over a manual impl?**

For an enum like `EventError` with multiple variants, manual `Display` is:

```rust
impl std::fmt::Display for EventError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            EventError::NoKind => write!(f, "event has no kind specified"),
            EventError::Unauthorized(user, action) => {
                write!(f, "user '{}' is not authorized for action '{}'", user, action)
            }
            EventError::SerializeError(e) => write!(f, "serialization error: {}", e),
        }
    }
}

impl std::error::Error for EventError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            EventError::SerializeError(e) => Some(e),
            _ => None,
        }
    }
}

impl From<serde_json::Error> for EventError {
    fn from(err: serde_json::Error) -> Self {
        EventError::SerializeError(err)
    }
}
```

That is 25+ lines for three variants. With thiserror, the same is 6 lines (the enum definition plus attributes).

What thiserror provides:

- **Concise.** The format strings are right there with each variant. Adding a variant adds one line.
- **No bugs.** The Display impl, From impl, and Error::source method are all generated correctly.
- **Discoverable.** The attributes (`#[error]`, `#[from]`) document what the variant means.
- **Idiomatic.** Rust projects widely use thiserror; the patterns are familiar.

The trade-off (an additional dependency, slightly slower compile) is small. For most projects, thiserror is the right choice for any non-trivial error enum.

For libraries that minimize dependencies (because their consumers pay the cost), manual impls might be preferred. The standard library does not use thiserror for the same reason. But for applications and high-level libraries, thiserror is the convention.

The general lesson: derive macros from popular crates (serde, thiserror, clap, etc.) provide substantial value at low cost. Embracing them is part of being a productive Rust developer.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

Common patterns:

- Developers from C find declarative macros familiar (pattern-based substitution) but the strictness (hygiene, fragment specifiers) is new. The C-style macros they are used to have none of these guardrails.
- Developers from Java find macros generally unfamiliar; Java has no equivalent. Both declarative and procedural macros feel like new mechanisms. The closest analog is annotation processing (AspectJ-style), but the connection is loose.
- Developers from Python find some patterns familiar (decorators are similar to attribute macros) but the compile-time nature is new. Python's metaprogramming happens at runtime; Rust's happens at compile time.
- Developers from Lisp find Rust macros restrictive. Lisp macros operate on the same s-expressions as the rest of the language; Rust macros operate on tokens with type-checked output. The Lisp approach is more powerful but less safe.

The most common reflection: "Procedural macros felt magical at first. I understand declarative macros are pattern-matching templates. But procedural macros generate dozens of lines from one attribute, and I cannot see what the generated code looks like. I learned about `cargo expand` halfway through and that helped a lot."

Another common reflection: "Hygiene was the most surprising concept. I thought variable name collisions were a fact of life for macros; learning that Rust handles them automatically was a relief. It is one of those quiet improvements over C-style preprocessors that you appreciate once you understand it."

**2. A specific situation where a macro was the right tool.**

Sample answer:

"The `event!` macro from Exercise 8 was a clear case. The structure `event!(kind, user, action, metadata: { key => value, ... })` is a small DSL that would not work as a function.

A function signature `fn event(kind: &str, user: &str, action: &str, metadata: HashMap<String, String>) -> Event` would require the caller to construct the HashMap explicitly:

```rust
let mut meta = HashMap::new();
meta.insert("file".to_string(), "document.pdf".to_string());
meta.insert("size_bytes".to_string(), "1024".to_string());
let upload = event("user.action", "bob", "upload", meta);
```

That is significantly more verbose than the macro form. For events with many metadata entries, the macro saves real typing and improves readability.

What made the macro the right choice:

- **Variadic metadata.** The number of key-value pairs varies; a function cannot have variable arity in Rust.
- **Clean syntax.** The `key => value` form reads as data, not as code. It is closer to JSON or YAML than to function calls.
- **Limited scope.** The macro is small (10 lines), used in one place, and does one thing. The complexity is contained.

If the macro grew more elaborate, I might reconsider. But for variadic key-value construction, this is exactly what macros are for."

**3. A risk of relying heavily on procedural macros.**

Sample answer:

"The biggest risk is opacity. Procedural macros run at compile time; their output is generated code that you typically do not see. A codebase using `#[derive(...)]` heavily might have implementations for traits that nobody on the team has ever read.

This becomes a problem when:

- **The macro produces something unexpected.** I had a case where `#[derive(Serialize)]` produced JSON output that did not match the API contract because of default field handling. The fix required `#[serde(rename_all = ...)]` attributes, but it took time to figure out what was happening.
- **A macro has bugs.** A buggy procedural macro produces buggy generated code. Tracing the bug to its source is hard because the generated code is hidden.
- **Macros conflict.** Two crates' derives might overlap (both providing `Display`, for example), with no clear way to resolve the conflict.
- **The macro disappears.** If you depend on a macro from a less-maintained crate, you are stuck if that crate becomes incompatible with future Rust versions.

Mitigation strategies I have used:

- **Stick to popular, well-maintained crates.** `serde`, `thiserror`, `tokio` are battle-tested. Their macros work reliably. Lesser-known crates' macros are riskier.
- **Read the documentation thoroughly.** Each macro has its own conventions, attributes, and gotchas. Skimming the docs leads to surprises.
- **Use `cargo expand` when debugging.** Seeing the generated code makes mysterious errors comprehensible. I rely on this whenever a derive does not behave as expected.
- **Limit custom procedural macros.** Writing my own would multiply the opacity. I have not done so yet; I rely on the well-known third-party crates.
- **Prefer simpler tools when possible.** A manual `impl` is sometimes the right choice over a derive macro, especially when the impl is short and the macro's behavior is hard to predict.

The flip side: avoiding procedural macros entirely is unrealistic for modern Rust. `serde`, `tokio::main`, `clap`'s derive parser, and similar tools are widely used and substantial improvements over their manual alternatives. The right approach is to use them selectively, understand what they do, and have the tools to debug when something goes wrong."

---

*End of Lab 14 Solutions*
