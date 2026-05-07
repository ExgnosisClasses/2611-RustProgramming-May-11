# Module 5: Functions and Closures
## Lab 4 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Defining and Calling Functions

### Checkpoints

**1. What is the actual return type of `print_result`?**

The return type is `()`, the unit type. When you omit `->` and a return type, Rust treats the function as returning unit. This is equivalent to writing `-> ()` explicitly.

You can verify this in several ways. The simplest is to bind the call to a variable and ask `rust-analyzer` to show the inferred type:

```rust
let result = print_result("test", 42);     // result has type ()
```

Hovering over `result` in VS Code (with rust-analyzer active) will show the type. Alternatively, you can use the `type_of` helper from Lab 2's Exercise 2.1, or attempt to use the result as something else (for example, `result + 1`) and read the compiler error.

**2. Why does Rust accept forward references when C does not?**

C uses a single-pass compiler model where declarations must appear before use. The compiler does not buffer the file and resolve names later; it processes top to bottom and needs each name to be known when it is encountered. To call a function defined later, you provide a forward declaration (in the source) or a header file (across translation units).

Rust uses a different model. The compiler reads the entire module before resolving function calls. The first pass discovers all top-level items (functions, types, constants, etc.); the second pass resolves names and types using the complete picture. This makes forward declarations unnecessary at the cost of slightly more work in the compiler.

This design is not unique to Rust; most modern languages (Java, C#, Python, Go) work this way. C's design reflects the constraints of 1970s hardware, where multi-pass compilation was expensive. Modern compilers can afford the second pass, and the developer experience benefit (no header files, no ordering rules) is significant.

**3. Why warn rather than reject for non-conventional naming?**

Three reasons:

- **Source compatibility.** A function called `getX` might have come from a binding generator or from FFI code where the underlying name is fixed. Rejecting the name outright would break legitimate use cases.
- **Gradual adoption.** Code translated from another language might have hundreds of incorrectly-named identifiers. A hard error would prevent the code from compiling at all, which is hostile to people learning the language. Warnings let the code compile while still encouraging the conventional style.
- **Tooling.** clippy and rustfmt cannot rename identifiers automatically (renaming is a refactoring that may require updating call sites in unrelated files). The warning surface tells the developer about the issue without forcing them to fix it before testing the code.

If the convention were a hard error, refactoring tools and IDE renaming features would be required infrastructure rather than nice-to-haves. The choice to warn keeps the language usable without these tools.

---

## Exercise 2 -- Expressions vs. Statements

### Checkpoints

**1. Why is the semicolon distinction a deliberate design choice?**

The expression-versus-statement model is what makes everything else in Rust work the way it does. Specifically:

- **Implicit returns** depend on the last expression being the value. Without the rule, you would need `return` everywhere.
- **`if` as an expression** depends on each branch being an expression. The `let speed = if cond { ... } else { ... }` pattern would not work.
- **Block expressions** depend on blocks evaluating to their last expression. The `let value = { ... }` pattern would not work.

If the language allowed both "expression with no semicolon means return" and "statements never produce values," the compiler would need additional rules to determine which mode each line is in. Making the semicolon the explicit marker of "this is a statement, not the value of the block" eliminates that ambiguity.

The cost is real: a misplaced semicolon changes a function's behavior. But the cost is paid once, when learning the rule, and the compiler error messages catch the mistake when it happens. The benefit (a clean, composable expression system) is paid on every line of code that uses it.

**2. A pattern where block-as-expression is cleaner.**

When the intermediate variables in a computation are not needed after the binding, scoping them inside a block keeps them out of the surrounding namespace. Compare:

```rust
// Without block: intermediates visible afterward
let raw = read_input();
let trimmed = raw.trim();
let parsed: f64 = trimmed.parse().unwrap();
let coordinates = Coordinates { value: parsed };

// raw, trimmed, and parsed are all still in scope here, even though we are done with them.

// With block: intermediates scoped tightly
let coordinates = {
    let raw = read_input();
    let trimmed = raw.trim();
    let parsed: f64 = trimmed.parse().unwrap();
    Coordinates { value: parsed }
};

// Only `coordinates` is in scope here.
```

The block form is preferable when the intermediates are scaffolding rather than meaningful program state. A reader of the code below the block sees only the result, not the intermediate computation. This is one form of structuring a function body for readability.

**3. Could `loop` or `while` be used as the body of a function?**

`loop` can. A `loop` block is an expression that produces a value (via `break value`), so it can be the body of a function:

```rust
fn first_match(values: &[i32], target: i32) -> Option<usize> {
    let mut i = 0;
    loop {
        if i >= values.len() { break None; }
        if values[i] == target { break Some(i); }
        i += 1;
    }
}
```

The function body is the `loop` expression. Each `break` produces a value of type `Option<usize>`, which becomes the return value.

`while` and `for` cannot be used the same way because they are statements, not expressions. They produce `()` and exit when their condition is false or their iterator is exhausted, with no value. A function body that is just a `while` loop returns `()`, regardless of the function's declared return type. Module 4 covered this distinction; it is one of the reasons `loop` exists as a separate construct.

---

## Exercise 3 -- Closure Syntax and Type Inference

### Checkpoints

**1. Why is closure type-locking reasonable?**

A closure is meant to be a small specialized function for a specific use case. Unlike `fn` definitions, closures are typically defined and used in the same scope, often as arguments to higher-order functions. The compiler has enough context to determine a single signature.

If closures were generic by default, every closure would carry full generic machinery, which has performance and complexity costs. The compiler would need to monomorphize each closure for every type it is used with, and the type errors when something does not match would be more complicated.

The simpler model "first use determines the type, errors after that" matches how closures are actually written. If you genuinely need a generic closure-like thing, you can write a generic function or use trait objects (covered in later modules).

**2. Why can't an unused closure be left untyped?**

The compiler must give every variable a concrete type at compile time. An unused closure with no usage to infer from has no information to determine its type. The compiler refuses to compile rather than guess, because guessing would lead to confusing errors later if the closure was eventually used.

The fix is to provide enough information for inference. Annotating the parameter types is usually sufficient, because the return type can usually be inferred from the body.

This is the same principle as `let v = Vec::new()` not compiling without further information: the compiler refuses to guess types when it could be wrong.

**3. When is a closure preferable, and when is a function preferable?**

The closure form is preferable when:
- The function is small (one to three lines).
- It captures values from the surrounding scope.
- It is used right where it is defined (as a callback or iterator method argument).
- Naming it would be more clutter than benefit.

The function form is preferable when:
- The body is more than a few lines or has nontrivial logic.
- The function is reusable across multiple call sites.
- A descriptive name would help future readers understand the intent.
- The function takes no captured state (so closure is unnecessary).
- You want an explicit signature that documents the contract.

The general rule: closures for small inline behavior, functions for reusable logic. Iterator chains usually use closures; module-level helpers usually use functions. The choice is mostly stylistic; both work in most contexts.

---

## Exercise 4 -- Capturing Variables

### Checkpoints

**1. Why is automatic capture mode selection preferable to explicit specification?**

Three reasons:

- **It picks the right answer.** The compiler always picks the least restrictive mode that makes the closure work. A developer trying to specify the mode might make it too restrictive (preventing valid uses) or too permissive (capturing more than necessary).
- **It reduces ceremony.** In C++, the developer must specify capture lists explicitly: `[&x]`, `[x]`, `[&]`, `[=]`, etc. This adds visual noise to every closure. Rust closures are cleaner because the compiler does the work.
- **It tracks code changes automatically.** If you modify a closure's body to mutate a previously-read-only capture, the capture mode changes from borrow to mutable borrow without any source change. With explicit capture lists, the developer has to remember to update them.

The trade-off is that the developer cannot easily see the capture mode by reading the code. They have to know the rules and trace through the body. In practice, this is rarely a problem because the rules are simple ("read-only is borrow, mutating is mutable borrow, move forces value") and the compiler will catch any inconsistency.

**2. What kind of bug does the borrow restriction prevent?**

The rule "you cannot read a value while a closure has it borrowed mutably" prevents a category of concurrency-like bugs that show up even in single-threaded code.

Consider this hypothetical code:

```rust
let mut count = 0;
let mut increment = || count += 1;
let snapshot = count;        // suppose this were allowed
increment();
println!("snapshot: {snapshot}, count: {count}");
```

If `snapshot = count` were allowed while `increment` could later modify `count`, the developer would have to reason carefully about the order of operations. In a longer function, this kind of "what is the current state of count?" question becomes exhausting.

The Rust rule eliminates the question. If a closure has a mutable hold on `count`, no one else can interact with `count` until the closure is gone. The language enforces a single-writer-or-multiple-readers invariant that simplifies reasoning about state.

The same rule applies to ordinary mutable references and is the foundation of borrow checking. We will see the broader picture in Module 6.

**3. Why is borrowing data into a thread dangerous?**

A thread runs concurrently with the rest of the program and may live longer than the function that spawned it. If a closure passed to the thread borrows data from the spawning function, two things can go wrong:

- **The function returns and frees its local data.** The thread is now reading freed memory: a use-after-free bug. Without `move`, this is exactly what would happen.
- **The thread races with other code.** If the closure has a mutable borrow, but the spawning function continues using the variable, two threads are accessing the same data without synchronization. This is a data race.

The `move` keyword forces the closure to take ownership of its captures. The data lives inside the closure (and therefore inside the thread) for the duration of the thread's life, regardless of what happens to the spawning function. The thread is self-contained.

The borrow checker recognizes this danger and refuses to compile a closure that borrows in a context that could outlive the borrow. The error is stated in terms of lifetimes (covered in Module 7), but the underlying concern is the same: do not let a reference outlive what it points to.

---

## Exercise 5 -- The `Fn`, `FnMut`, and `FnOnce` Categories

### Checkpoints

**1. Why does `Fn` work where `FnMut` is expected, but not the reverse?**

This follows from how the categories relate. `Fn` is more restrictive than `FnMut`; an `Fn` closure does less. Specifically, an `Fn` closure can be called multiple times without mutation, which means it satisfies any requirement that allows mutation as well.

A function that asks for `FnMut` is saying "I might mutate the closure's state when I call it, but I do not have to." If you give it an `Fn` closure (which never mutates anyway), the function still works. The closure is "stronger" than required.

The reverse direction does not work. A function that asks for `Fn` is saying "I will not mutate the closure's state." If you give it an `FnMut` closure, the function would still not mutate it (because the function code does not), but the type system forbids the conversion because the closure could mutate when called, which the function's contract said it would not.

This is the same pattern as covariance and contravariance in other type systems: a more restrictive type can satisfy a more permissive requirement, but not the other way around.

**2. What changes if `retry` uses `FnOnce` or `Fn` instead of `FnMut`?**

**With `FnOnce`:** The function would only be able to call the closure once. Since the loop iterates `max_attempts` times, you would have to call the closure once outside the loop and then exit. The `retry` semantic would be impossible to implement, because retrying requires multiple calls. `FnOnce` is wrong for this use case.

**With `Fn`:** The function would only accept closures that do not mutate state. The closure in the example uses `attempt_count += 1`, which mutates. With `Fn`, that closure would be rejected. To use `Fn`, the closure body would have to either use no shared state or use interior mutability (covered in a later module). `Fn` is too restrictive for this use case.

`FnMut` is the right choice because:
- The function calls the closure multiple times (so `FnOnce` is too restrictive).
- The closure may need to track per-attempt state (so `Fn` is too restrictive).

**3. What does it tell you that the compiler decides the categories?**

It signals that Rust prefers verification over declaration. The categories describe what the closure actually does, not what the developer thinks it should do. This has two practical consequences:

- The developer cannot lie. If they think a closure is `Fn` but it actually mutates, the compiler corrects them (by attaching `FnMut` to the closure and then rejecting any code that requires `Fn`). There is no way for the closure's labels to drift out of sync with its behavior.

- The developer does not have to think about it. The compiler attaches the appropriate categories silently. The developer only sees the categories when they write a function that requires a specific one, and then the compiler tells them whether their closure satisfies the requirement.

This is the same pattern that runs through the rest of the language: types describe reality (what data is, what closures do), and the compiler enforces that reality. Developers state the intent in function signatures; the compiler verifies that everything else matches.

---

## Exercise 6 -- Higher-Order Functions

### Checkpoints

**1. Another higher-order iterator method.**

Many possible answers; common ones include:

- **`fold`**: Walks the iterator, accumulating a result. `iter.fold(0, |acc, x| acc + x)` is equivalent to `iter.sum()`. The closure receives the accumulator and the next element and returns the new accumulator.
- **`any`**: Returns `true` if any element matches the closure's predicate.
- **`all`**: Returns `true` if every element matches the closure's predicate.
- **`take_while`**: Yields elements while the closure's predicate is true, stopping at the first one that fails.
- **`skip_while`**: Skips elements while the closure's predicate is true, then yields the rest.
- **`flat_map`**: For each element, the closure produces an iterator; the results are flattened into one stream.
- **`for_each`**: Walks the iterator, calling the closure for each element. Used for side effects.

Each takes one or more closures and parameterizes the iteration's behavior.

**2. Why does `impl Fn(i32) -> i32` work for `make_adder`?**

`impl Trait` in return position means "the function returns some specific type that satisfies this trait. The caller does not need to know the exact type, but it is one specific type."

In `make_adder`, every call returns a closure with a different captured value but the same shape. Each closure has its own anonymous type, but the compiler chooses one specific type (whatever shape the body produces) and uses it consistently for all calls to `make_adder`.

The caller treats the result as "some `Fn(i32) -> i32`." The caller does not care whether the underlying type is named `Closure_at_line_3_v1` or whatever; only that it satisfies the trait.

This works because the compiler can determine the exact type at compile time (each call to `make_adder` returns a single specific closure type). This is different from `Box<dyn Fn(i32) -> i32>`, which can hold any closure that satisfies the trait at runtime; that form supports returning different concrete types from different branches but pays for the flexibility with heap allocation and dynamic dispatch.

**3. Why `&[i32]` and `Fn(i32) -> i32` rather than `Vec<i32>` and `&dyn Fn(i32) -> i32`?**

Taking `&[i32]` is more flexible than `Vec<i32>`:

- `&[i32]` accepts arrays, vectors, and subranges of either. The function does not care where the data lives.
- Taking `Vec<i32>` would require the caller to either own a vector or convert their data into one, which is wasteful.
- Taking `&[i32]` does not transfer ownership. The caller still owns their data after the call.

Taking a generic `F: Fn(i32) -> i32` is more flexible than `&dyn Fn(i32) -> i32`:

- A generic `F` is monomorphized: the function is specialized for each closure type, with full inlining and zero overhead.
- `&dyn Fn` would require runtime dispatch through a vtable, costing one indirect call per use.
- The generic form accepts both regular functions and closures with captures uniformly; the dyn form does too, but with extra runtime cost.

For functions called in tight loops (like `apply_all`, which calls the closure once per element), the generic form is much faster. The trade-off is binary size: each unique closure type produces a specialized version of the function. For most code, this is the right trade-off; only when binary size is a critical concern do developers reach for `dyn`.

---

## Exercise 7 -- Function Pointers

### Checkpoints

**1. Could the function table hold both functions and capturing closures?**

No, not directly. The array `[fn(i32, i32) -> i32; 3]` requires the `fn` type, which is a concrete function pointer. Capturing closures cannot fit into this type because each one has its own anonymous type with potentially different size and contents.

To hold both, you would need a more flexible type:

```rust
let operations: [Box<dyn Fn(i32, i32) -> i32>; 3] = [
    Box::new(add),
    Box::new(subtract),
    Box::new(|a, b| a * b * multiplier),    // capturing closure
];
```

`Box<dyn Fn(...)>` is a heap-allocated trait object that can hold any closure satisfying the trait. The cost is:

- Heap allocation for each entry.
- Dynamic dispatch (one indirect call per use).
- An extra level of indirection.

For three operations called a few times, this cost is negligible. For a hot path that calls the operation millions of times, the function-pointer form is faster.

**2. When can you convert one direction but not the other?**

You can convert from regular function or non-capturing closure to function pointer:

```rust
let f: fn(i32) -> i32 = some_regular_function;     // works
let g: fn(i32) -> i32 = |x| x * 2;                  // works (no captures)
```

You cannot convert from capturing closure to function pointer:

```rust
let n = 3;
let h: fn(i32) -> i32 = |x| x * n;       // ERROR
```

You can always convert from function pointer or any closure to `Fn`/`FnMut`/`FnOnce` (if the closure satisfies the trait):

```rust
fn takes_fn<F: Fn(i32) -> i32>(f: F) {}
takes_fn(some_regular_function);          // works
takes_fn(|x| x * 2);                      // works
takes_fn(move |x| x * n);                 // works
```

The pattern: `fn` is the most restrictive (concrete, no captures), then non-capturing closures, then capturing closures. `Fn`/`FnMut`/`FnOnce` accept everything.

**3. What does "zero-cost" mean for function pointers, and what does `Box<dyn Fn>` cost?**

A function pointer is just a memory address. Calling through one is a single indirect call instruction, which on modern CPUs is virtually free (one extra memory load and one branch, both well-predicted). There is no allocation, no indirection beyond the address itself, and no virtual table lookup. This is what "zero-cost" refers to: no overhead beyond what a regular function call would have.

`Box<dyn Fn>` has multiple costs:

- **Heap allocation.** Each `Box::new(closure)` allocates memory on the heap to hold the closure (the function plus its captures). For a long-lived program with many trait objects, this fragments memory.
- **Indirection through the box.** Calling the closure goes through the `Box`, which is a pointer to the heap.
- **Virtual table lookup.** The `dyn` part means dynamic dispatch: at runtime, the code looks up which function implementation to call through a vtable. This is one extra memory load per call.
- **Loss of inlining.** The compiler cannot inline through `dyn` calls because it does not know the concrete function at compile time. Inlining is one of the most important optimizations, and losing it can have outsized performance impact in tight loops.

For most code, these costs are invisible (microseconds out of milliseconds). For hot inner loops, they can be a 2x to 10x slowdown compared to the generic equivalent. The right choice depends on whether the flexibility is needed.

---

## Exercise 8 -- Putting It Together

### Task 8.2 -- Pattern identification

1. **Function whose body is a single expression with no `return`:** `classify` is a single `if` chain that returns its branch values. `double`, `halve`, and `square` are single expressions in their bodies.

2. **Early `return` and final implicit return:** None in this specific program. (The lab structure had this pattern in Task 2.3 of Exercise 2; here the program does not include an example. A conscientious student might identify this as a missing pattern and either add one to their version or note its absence.)

3. **Higher-order function accepting `Fn`:** `count_above` takes `predicate: F where F: Fn(i32) -> bool`.

4. **Function returning closure with `impl Fn`:** `make_threshold_predicate` returns `impl Fn(i32) -> bool`.

5. **Closure capturing by `move`:** Inside `make_threshold_predicate`, the closure `move |value| value > threshold` takes ownership of `threshold`.

6. **Closure satisfying `FnMut`:** `count_with_side_effect` mutates `total_counted`. The closure is `FnMut`, and the surrounding block was used to scope its borrow tightly.

7. **Function pointer for type-erasing names:** The `transforms` array uses `fn(i32) -> i32` so that `double`, `halve`, and `square` can share one type and be stored together.

### Checkpoints

**1. Why `Fn` rather than `FnMut` for `count_above`?**

Two reasons:

- **The function does not need mutation.** `count_above` walks the values, calls the predicate on each, and counts. The predicate is a question, not an action; it should not have side effects.
- **`Fn` is more permissive at the call site.** A function with the bound `Fn` accepts any closure that satisfies `Fn`, including closures that satisfy `Fn` because they only read from captures. A function with the bound `FnMut` rejects nothing (since `Fn` is also `FnMut`), but the signature documents that the function might mutate, which is misleading if it does not.

If `count_above` were changed to use `FnMut`, the call site code would still compile. The closures passed in `main` (`|v| v > 50`, `above_100`) are `Fn`, and `Fn` is also `FnMut`, so they are accepted. But the signature would falsely suggest the function might mutate the closure's state, which is harder for callers to reason about.

The general principle: declare the most restrictive bound that the function actually needs. This is a form of contract design.

**2. Could `make_threshold_predicate` capture `threshold` by borrow instead?**

Not easily, and not for this use case. The function is structured to outlive its caller's stack frame: it returns a closure, and the caller uses that closure later. If the closure borrowed `threshold` instead of taking ownership, the borrow would be tied to the function's stack frame, which goes away when the function returns.

To borrow safely, the function would need to accept `threshold` as a reference and the returned closure would have to carry a lifetime tied to the input:

```rust
fn make_threshold_predicate<'a>(threshold: &'a i32) -> impl Fn(i32) -> bool + 'a {
    |value| value > *threshold
}
```

This works but ties the closure to the lifetime of the input. The caller has to keep `threshold` alive while the closure is in use. In practice, taking `i32` by value with `move` is simpler: `i32` is `Copy`, so taking ownership is cheap, and the resulting closure has no lifetime constraints.

The general lesson: prefer `move` capture for closures that are returned or stored. Borrow capture is fine for closures that are used in the same scope as their captures.

**3. Why was the FnMut closure scoped in a block?**

The closure `count_with_side_effect` borrows `total_counted` mutably. While the closure exists and might be called, no other code can use `total_counted`. After the closure is consumed (by `filter().count()` walking the entire iterator), the borrow is technically released, but the binding still exists in scope.

Putting the closure in a block makes its scope explicit and ensures the borrow is released at the end of the block. After the block, `total_counted` is freely usable again, and the program can print it.

If the closure outlived the surrounding scope's use of `total_counted`, the borrow would still be active and the print would fail to compile. The block is a tactical use of scoping to control exactly when the borrow ends.

Module 6 covers borrow scopes in detail; for now, the pattern "wrap a mutating closure in a block to release its borrow" is a useful reflex.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

A common pattern: developers from C-family languages find function syntax natural but struggle with the expression-versus-statement distinction. The semicolon footgun is a one-time stumble that everyone makes. Closures themselves usually feel familiar (the lambda comparison helps), but the `Fn`/`FnMut`/`FnOnce` distinction is unusual and takes practice.

The reverse pattern: developers from functional languages find closures and expression-orientation natural but struggle with the explicit category system. Their languages typically have one closure type and one capture mode; Rust's three categories feel like overhead.

The most common reflection: "I expected closures to be straightforward, but the rules around capturing took me a while to internalize." This is consistent with the broader theme that Rust's surface syntax is familiar but its semantics is more strict than other languages.

**2. A Rust-specific feature and what it gains.**

Sample answers:

- **The unique anonymous type.** This is what makes closures zero-cost in generic contexts. Java and Python pay heap allocation and dynamic dispatch for every closure. Rust pays neither, and the cost (the unique type) is a compile-time-only complication for the developer. The gain is that closures can be used in performance-critical code where Java's overhead would be unacceptable.

- **The Fn/FnMut/FnOnce hierarchy.** This is what makes closures fit into the ownership model. A function that asks for `Fn` is making a guarantee: "this closure will not mutate state when I call it." Java provides no such guarantee. Rust's hierarchy turns implicit assumptions into explicit type constraints, which is what allows closures to be used safely in concurrent code without runtime checks.

- **The explicit `move` keyword.** This is what makes closures safe to send across threads. In Java, every closure already lives on the heap and is reference-counted by the GC, so there is no equivalent decision to make. In Rust, the developer explicitly chooses to transfer ownership when needed. The gain is that closures that do not need to outlive their context pay nothing; only `move` closures pay the cost of ownership transfer.

The general theme: each Rust feature adds explicit information that other languages handle implicitly through runtime mechanics. The cost is a steeper learning curve; the gain is performance and safety guarantees that other languages cannot match.

**3. A decision the reader changed.**

This is necessarily personal. Common patterns:

- "I initially declared a higher-order function with `FnMut`, then realized the closure only reads its captures and changed it to `Fn` to be more honest about the contract."
- "I initially captured a value by reference (no `move`), then realized the closure needed to outlive the capturing scope and added `move`."
- "I initially used a `Vec<Box<dyn Fn>>` for a small dispatch table, then realized all the entries were regular functions and switched to `[fn(...); N]`."

The reflection is most valuable when the reader can identify a specific case where they switched approaches and articulate why.

---

*End of Lab 4 Solutions*
