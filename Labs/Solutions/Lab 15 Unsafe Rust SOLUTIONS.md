# Module 19: Unsafe Rust
## Lab 15 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Raw Pointers Basics

### Checkpoints

**1. Multiple raw pointers to the same data: why does the compiler not complain?**

The borrow checker tracks references (`&T` and `&mut T`) and enforces the aliasing rules:

- You can have multiple immutable references (`&T`) to the same data simultaneously.
- You can have one mutable reference (`&mut T`), but no other references (mutable or immutable) at the same time.

This rule prevents data races: if you have an `&mut T` and an `&T` to the same data, code reading through the `&T` could see partially-updated state from writes through the `&mut T`.

Raw pointers (`*const T` and `*mut T`) are not tracked by the borrow checker. The compiler does not enforce aliasing rules for them. You can have:

- Multiple `*const T` to the same data.
- Multiple `*mut T` to the same data.
- A mix of both.

The relaxation is what makes raw pointers useful for cases where the borrow checker's rules are too restrictive: linked data structures, FFI, custom allocators. But it comes with a price: you take responsibility for ensuring that accesses through raw pointers respect the aliasing invariants the compiler is no longer checking.

If you violate the invariants (concurrent writes through two `*mut T`, or reads through `*const T` while writing through `*mut T`), you get data races and undefined behavior. The compiler will not save you; you took the relaxation explicitly.

This is the central trade-off of raw pointers. The borrow checker's rules are strict because they enable safety guarantees; relaxing them gives you flexibility at the cost of those guarantees.

In practice, most uses of raw pointers fall into specific patterns where the developer can manually verify the aliasing rules:

- **Single thread, single pointer at a time.** Cleanly avoids data races.
- **Exclusive access tracked through types.** A struct with a `*mut T` that takes `&mut self` for any operation cannot have concurrent access through the same struct.
- **FFI to libraries with documented thread safety.** The C library's documentation specifies the rules; the Rust wrapper translates them.

Reflexively using raw pointers for things references could handle is wrong. Reach for raw pointers only when the flexibility is genuinely needed.

**2. Why keep unsafe blocks small?**

A block of code inside `unsafe { ... }` is your assertion that the unsafe operations within it are correct. The larger the block, the more operations you are vouching for.

Consider:

```rust
unsafe {
    let value = compute_something();
    let processed = transform(value);
    let pointer = get_pointer();
    let result = *pointer;
    // 20 more lines...
}
```

What part of this block is actually unsafe? Only the `*pointer` dereference. The other code (compute_something, transform, get_pointer) is regular safe code that does not need to be inside an unsafe block.

By putting it all inside `unsafe`, you obscure which operations are actually critical. A reviewer reading the block has to scan all 20+ lines looking for the unsafe operation. With a smaller scope:

```rust
let value = compute_something();
let processed = transform(value);
let pointer = get_pointer();

// SAFETY: ...justification for this specific dereference...
let result = unsafe { *pointer };

// 20 more lines of safe code
```

The unsafe block is one line. The reviewer immediately sees what is being asserted. The 20+ surrounding lines are verified by the compiler.

This also matters for refactoring. If someone modifies the surrounding code, they cannot accidentally introduce a new unsafe operation that the original `unsafe` block silently covers. With small unsafe blocks, adding a new unsafe operation requires explicitly opening a new `unsafe { ... }`.

The pattern is called "minimal unsafe scope" and it is the standard practice in idiomatic Rust. Every unsafe block should contain only the operations that actually require unsafe, nothing more.

**3. What does undefined behavior tell you about testing unsafe code?**

Undefined behavior is genuinely undefined: anything can happen, and the actual behavior depends on factors outside your control (compiler version, optimization level, memory layout, surrounding code, even timing).

This means:

- **Bugs may be invisible during testing.** The program appears to work; tests pass. The bug only manifests under specific conditions (production load, different OS, different compiler).
- **Bugs may move.** A small change elsewhere in the code (an unrelated refactoring, a different optimization) can suddenly make a latent bug visible.
- **Bugs are not local.** Undefined behavior can corrupt memory in ways that cause crashes far from the buggy code.

Implications for testing:

- **Standard unit tests are not enough.** Testing that "the function returns the right value" does not verify that the unsafe is correct; the test might be one of the cases where the bug happens to be invisible.
- **Fuzz testing helps.** Tools like `cargo-fuzz` generate random inputs and detect crashes. This finds bugs that targeted tests miss.
- **Miri is essential.** For code that can run under Miri (no FFI), Miri detects undefined behavior at runtime. It catches many bugs that pass normal testing.
- **Code review matters more.** A bug that escapes testing must be caught by careful review of the unsafe blocks and their SAFETY comments.
- **Multiple platforms.** Test on different operating systems, architectures, and optimization levels. UB that hides on one platform may surface on another.

The standard library invests heavily in all of these for its unsafe code. Standard library code with unsafe blocks is reviewed multiple times, tested under Miri, fuzzed, and used by millions of users who would surface latent bugs.

For application code with significant unsafe, you would need to invest similarly. Unsafe code without this level of testing is risky regardless of how well it appears to work.

This is part of why the Rust community recommends minimizing unsafe and using existing safe wrappers. Reusing well-tested unsafe (in the standard library, in popular crates) is much safer than writing your own.

---

## Exercise 2 -- The Five Superpowers

### Checkpoints

**1. Why provide atomic types when `static mut` exists?**

`static mut` is "global mutable state without synchronization." It is unsafe because nothing prevents concurrent access. To use it safely, you must somehow ensure single-threaded access, which the compiler cannot verify.

For a single-threaded program, `static mut` works (with unsafe blocks marking each access). But for any multi-threaded program, `static mut` is unsafe to use, and there is no way for the compiler to help you make it safe.

Atomic types provide synchronization built in:

- `AtomicU32::fetch_add(1, Ordering::SeqCst)` is an atomic increment. No data race even with concurrent access from multiple threads.
- `AtomicU32::load(Ordering::SeqCst)` is an atomic read. No data race.
- `AtomicU32::store(value, Ordering::SeqCst)` is an atomic write. No data race.

The methods are safe (no `unsafe` block required). The thread safety is built into the type. You can use atomic types from any number of threads without special handling.

The trade-offs:

- **Atomics are slightly slower than non-atomic operations.** The atomic versions use specific CPU instructions that synchronize across cores. The cost is small (a few cycles per operation) but real.
- **Atomic operations have ordering choices.** `SeqCst` is the strongest guarantee; `Relaxed` is the weakest. The choice affects performance and correctness in subtle ways.
- **Atomics are limited in scope.** They work for simple values (integers, booleans, pointers) but not for complex data structures. For complex data, you need `Mutex<T>` or similar.

For nearly all use cases where you want shared mutable state, atomic types or `Mutex<T>` are the right answer. `static mut` is appropriate only in very specific situations:

- Embedded code with a single thread of execution by design.
- Initialization-only state that is set once and then read.
- Cases where you genuinely cannot use the safe alternatives (extremely rare).

Even in these cases, the modern Rust convention is to use `OnceCell` or `LazyLock` (safe alternatives for "initialized once, then read") rather than `static mut`.

**2. Why is narrow unsafe safer than "all checks off"?**

Two main reasons.

**Reason 1: Most of Rust's safety guarantees are preserved.** Inside an unsafe block, the borrow checker still runs, lifetime checking still tracks references, type checking is still strict. You can still have compile errors for things that have nothing to do with unsafe operations.

If unsafe disabled all checks, every line inside an unsafe block could be wrong in countless ways. With narrow unsafe, only the five specific abilities are unlocked; everything else continues to be verified.

This dramatically reduces the surface area for bugs. A 100-line unsafe block in narrow-unsafe Rust might contain only 2-3 actually-unsafe operations; the other 97 lines are checked by the compiler. In all-checks-off unsafe, every line is potential for memory corruption.

**Reason 2: Narrow unsafe makes auditing tractable.** To verify safety of a Rust program:

- Identify all unsafe blocks (mechanical).
- For each, verify the specific unsafe operations are correct (focused review).
- Trust everything else (verified by compiler).

If unsafe were broad, auditing would mean reviewing every line of unsafe code for any kind of issue. With narrow unsafe, auditing means reviewing specific operations: this dereference, this FFI call, this `static mut` access. The reviewer has a specific question to answer, not a general one.

This is why Rust can have a meaningful safety story despite the existence of `unsafe`. The unsafe escape hatch is narrow enough that most code is auto-verified, and the unsafe regions are small enough to audit carefully.

C, by contrast, has no equivalent of "this region is safe to ignore." Every pointer access in C might be wrong. Auditing C for memory safety means reading every line, which scales poorly.

**3. Which superpower is most frequent in application code? Which is rare?**

Most frequent: **calling unsafe functions** (Power 2). Every FFI call to a C library is an `unsafe fn` call. Crates that wrap C libraries (`openssl`, `sqlx`, `reqwest` via its TLS backends) all have unsafe inside, and applications that use them transitively include this unsafe.

Even pure-Rust applications often call unsafe functions indirectly. Standard library methods like `slice::get_unchecked` (called by performance-tuned code) are unsafe fn. Generic library code uses them when it can verify the preconditions.

Most rare: **accessing union fields** (Power 5). Unions appear mainly in FFI with C code that uses C unions. In pure Rust, you would use enums instead, which are safer (the variant is tracked) and just as efficient for most purposes.

The other three powers fall in between:

- **Dereferencing raw pointers** (Power 1): common in FFI bindings and data structure implementations; rare in application code.
- **Accessing mutable statics** (Power 3): rare in modern code; atomic types and `OnceCell` cover most uses.
- **Implementing unsafe traits** (Power 4): rare unless you write low-level libraries. Most applications and even most libraries do not need this.

For typical application code, you should expect:

- Most of your code: zero unsafe.
- Library code you depend on: substantial unsafe internally, but their public APIs are safe.
- Your own code: zero or one unsafe block per project, most likely.

This is the intended pattern. Unsafe exists to enable systems programming; it does not need to be ubiquitous in application code.

---

## Exercise 3 -- Calling C Functions via FFI

### Checkpoints

**1. Why is `abs` unsafe even though it is harmless?**

The unsafe requirement at FFI call sites is a matter of language design, not a comment on the specific function's safety.

When Rust calls a C function:

- The C compiler may have compiled it with different assumptions.
- The C function may have undocumented preconditions.
- The C function may behave incorrectly in ways the Rust compiler cannot detect.
- Even if the function is currently safe, future versions of the C library might have different behavior.

The Rust compiler cannot reason across the language boundary. From its perspective, the C function is a black box that might do anything. To be safe, every FFI call is treated as potentially dangerous.

The unsafe block at the call site is your assertion: "I have checked that this specific call is correct given everything I know about the function and the surrounding context." For `abs`, the assertion is trivial (it is a pure function with no preconditions). For other functions (those that take pointers, allocate memory, or have undocumented thread-safety requirements), the assertion is substantial.

The cost of this design is friction: even simple FFI calls require unsafe. The benefit is that the language boundary is explicit; you cannot accidentally call into C without being aware of it.

Alternative designs would have to either:

- Trust C functions to be safe (which they often are not).
- Have the compiler statically analyze C code (which is impractical).
- Provide some other mechanism for verifying foreign code safety (which does not exist for C).

Rust's approach (require unsafe at every FFI call) is conservative but consistent. It puts the responsibility for verification in the right place: the developer writing the call, who can read documentation and consider context.

In practice, the convention is to write safe Rust wrappers around FFI calls (as done in this exercise). The wrappers contain the unsafe; downstream consumers of the wrapper write only safe code. The unsafe at FFI call sites is concentrated in the wrapper, not scattered throughout the application.

**2. The rule for safely extracting pointers from owned types?**

The rule: the owned type must be alive (not dropped) for the entire duration that the pointer is used.

Concretely:

```rust
// WRONG: temporary CString dropped at end of statement
let ptr = CString::new("...").unwrap().as_ptr();
// ptr is now dangling

// Right: name the CString to extend its lifetime
let s = CString::new("...").unwrap();
let ptr = s.as_ptr();
// s lives until the end of the enclosing scope; ptr is valid
```

The principle: pointers do not own the data they point to. The owning value is responsible for managing the memory. If the owner is destroyed, the pointer becomes invalid.

Common patterns where this rule matters:

- **CString to C functions.** The CString owns its buffer; the pointer to the buffer is valid only while the CString is alive.
- **Vec/slice to C functions.** Pass `vec.as_ptr()` and `vec.len()`; the Vec must outlive the function call.
- **String to FFI.** Convert to CString first, then take a pointer.

The temporary-lifetime pitfall is one of the most common FFI bugs. Modern Rust sometimes extends temporary lifetimes in ways that hide the bug, but this is not guaranteed; relying on it is undefined behavior.

The safe pattern: always bind owned values to variables before taking pointers. The variable's scope makes the lifetime explicit, and the borrow checker (and your eyes) can verify that the pointer is not used after the variable goes out of scope.

Some FFI helpers (like `slice::as_ptr()`) tie the pointer's validity to the borrow checker's reference rules. These are safer because the compiler enforces the lifetime. But you cannot pass a `&[u8]` directly to a C function; you must extract a raw pointer, and at that moment, the compiler's tracking stops. You take over responsibility for the lifetime.

**3. Why is "safe abstraction over unsafe" the right structure for FFI?**

Three main reasons.

**Reason 1: The audit surface is concentrated.** Without the wrapper, every place in your application that calls FFI is a place with unsafe code. To audit safety, you would have to review every FFI call site, each with its own context and potential pitfalls.

With the wrapper, the FFI calls happen in one place. Auditing means reviewing one function. Application code that uses the wrapper is automatically safe (from the FFI perspective).

For a library that wraps a substantial C dependency, this matters enormously. The library author writes carefully-reviewed FFI code once; thousands of downstream users get safe APIs without thinking about unsafe.

**Reason 2: Error handling can be idiomatic Rust.** C error handling is varied (return codes, errno, null pointers, output parameters). Rust error handling is uniform (`Result<T, E>` with the `?` operator).

A wrapper translates between the two. Inside, the wrapper handles the C-style error reporting; outside, callers see `Result`. The translation happens once, in the wrapper, instead of at every call site.

This dramatically improves API ergonomics. Callers do not have to remember "this function returns -1 on error" or "this function sets errno." They just call the wrapped function and pattern-match on the `Result`.

**Reason 3: Type safety can be applied.** C APIs typically pass raw pointers; Rust APIs can use references and lifetime parameters. A wrapper can convert raw pointers to references with proper lifetime tracking, ensuring that callers cannot misuse pointers.

For example, a C function returning a pointer to internal state might return null, might require a specific lifetime, and might be invalidated by certain operations. The wrapper can express these constraints with Rust's type system: returning `Option<&T>` for nullability, using lifetime parameters for the validity period, requiring `&mut self` for operations that invalidate.

The result is that the wrapper exposes the C library's contracts in a way that Rust's compiler can check. Misuse becomes a compile error rather than a runtime crash.

For all these reasons, FFI wrappers are the canonical structure for cross-language code in Rust. The Rust ecosystem has hundreds of these wrappers, providing safe access to thousands of C libraries.

---

## Exercise 4 -- Implementing `Send` and `Sync` Manually

### Checkpoints

**1. Why use the conservative auto-derivation rules?**

The auto-derivation rule for Send: "a struct is Send only if all its fields are Send." Similar for Sync.

This is conservative: it sometimes refuses Send/Sync for types that are actually safe (because their fields happen to include a raw pointer, even if the pointer is managed safely). But the conservatism is on the safe side.

The alternative would be to auto-derive optimistically, marking types as Send/Sync unless something obviously prevents it. This would catch some cases but miss others. For example, a struct with a raw pointer might or might not be Send depending on what the pointer references; the compiler cannot know without analyzing how the pointer is used.

The conservative rule says: "if I cannot prove this is safe, assume it is not." Manual `unsafe impl Send` is the escape hatch for cases where the developer knows it is safe even though the auto-derivation does not.

This puts responsibility in the right place:

- For types where auto-derivation works (most types), the developer does nothing; the compiler figures it out.
- For types where auto-derivation is too conservative, the developer manually asserts (with `unsafe impl`), explaining the reasoning in a SAFETY comment.
- For types where the developer is not sure, no manual impl is needed; the type is just not Send/Sync, and the developer designs around it.

The opposite design (optimistic auto-derivation, manual `!Send` to opt out) would be problematic. Many types are accidentally Send-incompatible without the developer realizing it. With optimistic derivation, these would be marked Send by default, and the developer would have to actively block them.

The current design is safer: types are Send/Sync only when proven so. Manual override is for the rare cases where the developer has additional knowledge.

**2. Why is the "wrong reason" pattern especially dangerous?**

Adding `unsafe impl Send` without verification has several specific dangers.

**The consequence is data races.** A type incorrectly marked Send can be sent across threads. If the type contains shared mutable state without synchronization, threads accessing it can race. Data races are undefined behavior; the consequences range from subtle wrong results to memory corruption to security vulnerabilities.

**The bugs are invisible in testing.** Unit tests typically run on a single thread or with controlled concurrency. The race conditions only manifest under specific timing (high load, specific scheduling). Tests that pass during development can fail in production.

**The bugs are hard to debug.** When a race condition causes corruption, the symptom (crash, wrong output) often appears far from the cause. Debugging requires understanding the entire program's concurrent behavior, not just the buggy region.

**The compiler does not help.** Once you write `unsafe impl Send`, the compiler trusts you. It allows the type to cross thread boundaries. The check that would normally catch the issue is exactly the check you bypassed.

Other unsafe misuses are bad, but they often have more local consequences:

- Wrong pointer dereferences cause crashes (visible).
- Wrong FFI calls produce wrong output (often visible).
- Wrong union access produces wrong types (caught by tests with diverse inputs).

Send/Sync bugs are special because they manifest only under specific timing. You can write a buggy type, test it extensively, deploy to production, and have it work correctly for months. Then, one day under high load, you start seeing rare crashes that cannot be reproduced. These are the most expensive bugs to track down.

For this reason, `unsafe impl Send` deserves extra scrutiny in code review. A reviewer should ask: "Does the SAFETY comment correctly justify this? Is the reasoning sound?" If the answer is unclear, the impl should not exist.

**3. Send vs Sync: practical difference and when to allow one but not the other?**

The two traits answer different questions:

- **Send:** can the type be transferred (moved) from one thread to another?
- **Sync:** can multiple threads access the same value through references (`&T`)?

Practical implications:

- A type that is Send but not Sync: can be moved between threads, but cannot be shared through references. You can send the only copy from one thread to another; you cannot have two threads holding references to it.
- A type that is Sync but not Send: rare; possible for types that allow concurrent access through references but cannot be moved (typically types tied to a specific thread or context).
- A type that is both Send and Sync: fully thread-portable.
- A type that is neither: must stay on one thread, no transfers, no sharing.

When to allow Send but not Sync:

- **Types with thread-affine resources.** A type that owns a file handle might be Send (you can transfer the handle) but not Sync (you cannot have two threads writing to the same file handle without synchronization).
- **Types with internal mutation through `&self`.** If the type has `Cell` or `RefCell` internally, it allows mutation through shared references; this is incompatible with Sync (which requires safe concurrent access).
- **Wrappers around mutex-protected resources.** The mutex provides exclusion within a single thread but the resource itself might not be reentrant.

The example in Task 4.5 (`DatabaseHandle`) followed this pattern: Send because the handle can be transferred (one thread at a time uses it), but not Sync because concurrent access through references is not supported by the underlying C library.

The choice between Send-only and Send+Sync is part of the type's API design. Be conservative: only declare Sync if the type actually supports concurrent reference access. Declaring Sync incorrectly is a common bug source; declaring Send-only when both could be supported just means callers must use mutexes for sharing.

For most types, the automatic derivation handles this correctly. Manual implementations should be deliberate and well-justified.

---

## Exercise 5 -- Building a Safe Abstraction

### Checkpoints

**1. Why is bounds-check-outside, deref-inside the right structure?**

Two reasons.

**Reason 1: The unsafe block contains only the operation that requires unsafe.** Bounds checking (the `if index < slice.len()`) is regular safe Rust; it does not need to be inside the unsafe block. The deref (`*slice.as_ptr().add(index)`) is what requires unsafe.

By putting only the deref inside `unsafe`, you make explicit which operation is the unsafe one. A reviewer reading the code sees clearly: "the bounds check is here; if it passes, the unsafe dereference is correct."

If the entire body were inside `unsafe`, the structure would be the same, but the visual cue (where the unsafe is) would be lost. A reviewer might miss the bounds check entirely and worry about the wrong things.

**Reason 2: It forces the safety reasoning to be visible.** The order is: bounds check first, then unsafe. The check IS the justification for the unsafe operation. Reading the code top to bottom:

- "If the index is in bounds..."
- "...then we can safely dereference."

This is exactly what the SAFETY comment will say. The structure of the code mirrors the structure of the reasoning. Putting them in the wrong order (unsafe first, then check) would not even compile correctly, but it would be conceptually backwards: you would be doing the unsafe operation before establishing why it is safe.

This pattern is used throughout the standard library. Every safe wrapper around an unsafe primitive follows it: validate the preconditions, then call the primitive.

**2. Checked vs unchecked: when to choose each?**

The `checked_get` function is the safe API:

- Always correct (handles invalid input gracefully).
- Returns `Option<i32>`, requiring the caller to handle the None case.
- Costs a bounds check on every call.

The `unchecked_get` function is the performance-oriented API:

- Faster (skips the bounds check).
- Requires the caller to have already verified bounds.
- Returns `i32` directly (no Option).
- Wrong usage is undefined behavior.

When to use the safe version (the default):

- Whenever you do not have a specific reason to use the unsafe one.
- When the cost of the bounds check is negligible (most code).
- When the input might be invalid (user input, parsed data, etc.).

When to reach for the unchecked version:

- Performance-critical loops where the bounds check is measurable overhead.
- The caller has already verified the bounds (e.g., they generated the index from a range that is known to be valid).
- The unchecked version is preferable for code that is otherwise carefully constructed.

For most application code, you should ALWAYS use the safe version. The unchecked version saves maybe a few CPU cycles per call; in a typical application, this is invisible.

For library code in hot paths, the unchecked version might matter. The standard library has both: `Vec::index` is safe (with bounds checking and panic on out-of-bounds), `slice::get_unchecked` is unsafe (no bounds check). Library authors choose based on what their users need.

A specific pattern: a safe iteration that internally uses the unchecked version. The iterator yields valid indices; the body of the loop uses the unchecked indexing because the validity is already established. This combines safety (the iterator interface) with performance (no redundant checks).

This is exactly how `Vec::iter()` is implemented. The iterator generates indices; the elements are accessed via unchecked indexing because the iterator has already verified bounds.

**3. What changes about review without SAFETY comments?**

Without SAFETY comments, reviewing unsafe code becomes substantially harder:

- **The reviewer must reconstruct the reasoning.** Why is this dereference safe? Looking at the code, they have to figure out the preconditions, find where they are established, and verify the logic. This takes much longer than reading a comment.
- **The reviewer might miss subtle invariants.** Some safety arguments depend on conditions established elsewhere in the code (a constructor's invariant, a check that happens in a different method). Without a comment pointing to them, the reviewer might verify a different (incorrect) reasoning and approve unsafe code that is actually wrong.
- **The reviewer cannot easily ask questions.** A SAFETY comment is something to discuss: "Is this argument complete? What if X happens?" Without the comment, the conversation has no anchor.
- **Future modifications become risky.** Someone changing the surrounding code might unintentionally break an invariant the unsafe block relied on. With a SAFETY comment, the next reviewer can check the comment against the new code; without one, the broken invariant might not be noticed.

Real-world Rust code reviews look at SAFETY comments first. An unsafe block without a comment is flagged for revision. The convention is so strong that unsafe code without comments is considered low-quality almost universally.

For the `MyBox` example specifically, the four SAFETY comments cover:

1. **The first comment** explains why allocating works (valid layout, alloc behavior).
2. **The second comment** explains why writing to the pointer is OK (non-null, aligned, freshly allocated).
3. **The third comment** explains why reading is OK (the value was initialized in new, still alive).
4. **The fourth comment** explains why freeing is correct (allocated by alloc, no remaining references).

Each comment is short, but each captures a specific verification. Removing them would force the reviewer to re-derive these arguments themselves, every time. The comments make the safety case explicit and auditable.

---

## Exercise 6 -- Recognizing and Auditing Unsafe

### Checkpoints

**1. Why is unsafe count only one factor in evaluating a crate?**

The amount of unsafe is a noisy signal. Many factors affect what it means:

- **Quality of the unsafe.** A small amount of carefully-reviewed, well-documented unsafe is often safer than no unsafe at all (if the alternative is a complicated work-around that has its own bugs).
- **Type of unsafe.** FFI bindings to a well-known C library (like libcurl) are common and well-understood; raw pointer manipulation in a custom data structure is less common and more error-prone.
- **Necessity.** A crate doing systems-level work (an allocator, a database driver) needs more unsafe than a pure-Rust crate. Comparing them by raw unsafe count would mislead.
- **Maintenance and audit history.** A crate with substantial unsafe that has been audited externally and is actively maintained is much safer than the same amount of unsafe in an unmaintained crate.

Other factors that matter as much or more:

- **Project age and activity.** Active projects get bug fixes; abandoned ones do not.
- **User base.** Popular crates have more eyes on them; obscure ones may have undiscovered bugs.
- **Author reputation.** Crates from established Rust ecosystem maintainers tend to have higher quality.
- **Test coverage.** Crates with extensive tests, fuzz tests, and CI are more reliable.
- **Documentation.** Well-documented crates are easier to use correctly.
- **Dependencies.** A crate depending on many other crates inherits risk from all of them.

For evaluating a dependency:

- Use `cargo geiger` to get a baseline understanding of unsafe usage.
- Read the crate's README and documentation.
- Check the maintainer's other crates and the crate's commit history.
- Look at issues and PRs for signs of active maintenance.
- Read the source if you are concerned (this is what makes Rust nice: you can actually inspect).

For critical dependencies (security-sensitive code, foundational libraries), external audits matter. Some organizations sponsor formal audits of important crates (the Rust Foundation, Google's OSS-Fuzz, etc.). For most other dependencies, community usage and maintainer reputation are the main signals.

**2. When is Miri worth the cost?**

Miri is slow (10-100x normal execution). Running tests under Miri can take minutes for a test suite that runs in seconds normally. This is a real cost.

The benefit is detecting undefined behavior at runtime, which is invaluable for code that:

- Contains unsafe blocks.
- Has been changed recently (where new bugs might be introduced).
- Will be deployed in production (where UB causes mysterious failures).
- Has had bugs in the past (which suggests areas that need more verification).

Miri is worth the cost when:

- **You are writing a library with substantial unsafe.** The library will be used by many consumers; bugs are expensive. Miri catches bugs your tests miss.
- **You are reviewing unsafe code changes.** Running Miri on the modified tests provides additional confidence beyond reading the changes.
- **You are debugging a strange bug.** UB can manifest as inexplicable crashes or wrong output. Miri can pinpoint where the UB occurs.
- **You suspect a regression.** Comparing Miri output before and after a change tells you whether new UB has been introduced.

Miri is overkill when:

- **Your code has no unsafe.** Miri's main value is detecting unsafe-related UB; for code that is entirely safe, Miri rarely finds anything that the regular compiler misses.
- **You are in fast-iteration development.** During quick development cycles, running Miri on every test adds friction. Use it for verification before commits or in CI.
- **The unsafe is in trusted dependencies.** You do not need to Miri-test the standard library; that has been done extensively already.

Common workflows:

- Development: normal `cargo test`. Fast feedback.
- Before commit: maybe `cargo +nightly miri test` on tests that exercise unsafe code.
- In CI: dedicated Miri job that runs on every PR, separate from the main test job.
- After major changes: full Miri run to catch any new issues.

For our `unsafe_explorer` project, Miri would run the Exercise 1 and 2 tests but skip the FFI tests in Exercises 3 and 7 (the actual C functions cannot run under Miri). This is a limitation; not all unsafe code is Miri-testable.

**3. When is exposing `pub unsafe fn` appropriate?**

The default should be: do not expose `pub unsafe fn`. Encapsulate unsafe inside safe public APIs.

But there are legitimate exceptions:

- **Performance escape hatch.** The standard library has `slice::get_unchecked` because some performance-critical code can verify bounds itself and would rather skip the check. The function is unsafe because the caller must verify; the function exists because the verification is sometimes done elsewhere.
- **Low-level building blocks.** A library providing primitives for building other abstractions might expose unsafe functions intentionally. Crates like `bytemuck` provide unsafe functions for type conversions that callers must verify are safe for their types.
- **FFI sys crates.** Crates named `*-sys` typically expose raw FFI bindings (all unsafe). Higher-level crates wrap these in safe APIs. The sys crate is intentionally low-level; safe wrappers are separate.
- **Construction with invariants the caller establishes.** A type might have a `new_unchecked` constructor that skips validation because the caller has verified the inputs through some other mechanism.

Common to all these: there is a clear reason for exposing unsafe directly. The reason is documented. The function's documentation includes a `# Safety` section listing what the caller must verify.

What is inappropriate:

- **Exposing unsafe to bypass library design issues.** If the safe API is awkward, fix the safe API rather than adding an unsafe alternative.
- **Exposing unsafe to "let users decide."** Most users do not want this responsibility; they want a safe API.
- **Exposing unsafe in libraries that should be pure-Rust.** A general-purpose library should not require users to write unsafe; that defeats the purpose of using Rust.

A useful test: would I want this function to be called from typical application code? If yes, it should be safe. If only from other library code with specific knowledge, unsafe might be appropriate.

For `unsafe_explorer`, the `unchecked_get` function in Exercise 5 was a borderline case. In a teaching example, having both safe and unsafe versions demonstrates the pattern. In real library design, the choice depends on whether downstream users would have legitimate reason to use the unchecked version.

---

## Exercise 7 -- Putting It Together

### Task 7.2 -- Pattern identification

1. **`extern "C"` block:** Lines declaring `strlen`, `strdup`, `free` from the C standard library.

2. **Safe public function with internal unsafe:** `c_strlen(s: &str) -> usize` has a fully safe signature; its body contains an unsafe block for the FFI call.

3. **Safe struct with safe public methods:** `OwnedCString` is the struct; `new`, `len`, `to_string_lossy` are all safe (no unsafe in their signatures).

4. **`CString` for Rust-to-C conversion:** Used in both `c_strlen` and `OwnedCString::new` to convert the input `&str` to a null-terminated C string.

5. **`CStr::from_ptr` for C-to-Rust conversion:** Used in `OwnedCString::to_string_lossy` to read the heap-allocated C string into a Rust `String`.

6. **`Drop` for cleanup:** The `impl Drop for OwnedCString` block frees the memory allocated by `strdup`.

7. **`unsafe impl Send`:** The line `unsafe impl Send for OwnedCString {}` declares the type as Send despite containing a raw pointer.

8. **Deliberate omission of `unsafe impl Sync`:** The comment explains that Sync is intentionally not implemented; the code does not need concurrent reference access.

9. **SAFETY comments:** Every unsafe block in the program has a comment justifying its correctness. The comments cover preconditions, what is being verified, and what the unsafe operation requires.

10. **CString bound to a local variable:** In `c_strlen`, `let c_string = ...` binds the CString to a named variable, ensuring it outlives the `strlen` call.

### Task 7.3 -- Predict and verify

**Change A:** Remove `unsafe impl Send for OwnedCString {}`.

**Prediction:** The code in `main` that spawns a thread with `move || ...` taking ownership of `to_send` will fail to compile. The error will say that `OwnedCString` cannot be sent between threads because of its raw pointer field.

**Result:** As predicted:

```
error[E0277]: `*mut i8` cannot be sent between threads safely
  --> src/main.rs:NN:NN
   |
NN |     let handle = std::thread::spawn(move || {
   |                  ------------------ ^^^^^^^^^^ `*mut i8` cannot be sent between threads safely
```

The auto-derivation rules see the `*mut c_char` field and conservatively refuse to mark the type as Send. Without the manual override, the type cannot cross thread boundaries.

The lesson: Send/Sync auto-derivation is conservative. Types containing raw pointers are not Send by default. Manual implementations are the escape hatch when the developer knows the type is safe.

**Change B:** Remove the null check in `OwnedCString::new`.

**Prediction:** Under normal conditions, the program will work exactly as before. `strdup` rarely returns null in practice (only when memory allocation fails). The Drop implementation will be called eventually with the (non-null) pointer and free it correctly.

Under unusual conditions (memory pressure, deliberately limited memory), `strdup` could return null. Without the check, the function would return an `OwnedCString` with a null pointer. Subsequent operations on it (`len()` calls `strlen` on null, `to_string_lossy()` calls `CStr::from_ptr` on null) would invoke undefined behavior. The Drop implementation calling `free(null)` is actually well-defined (free of null is a no-op in C), but the earlier operations are not.

**Result:** Under normal conditions, the program runs and produces the same output. The bug is invisible in testing. Under low-memory conditions (which you cannot easily reproduce in this lab), the program would crash or have undefined behavior.

The lesson: defensive checks have real value even when the failure case is rare. The cost (one branch per call) is negligible; the benefit (catching the rare failure) is potentially the difference between a clean error and undefined behavior.

This is also an example of why testing unsafe code is hard. The test inputs that would trigger the bug do not occur in normal test environments. The code might work for years before the bug manifests, possibly in production.

**Change C:** Remove the `free` call in Drop.

**Prediction:** The program still compiles. It still runs and produces the same output. The bug is a memory leak: every `OwnedCString` allocation is never freed.

For a small program like this one, the leak is invisible. The program runs, leaks a few bytes, exits. The OS reclaims everything.

For a long-running program (a server, a daemon), the leak would accumulate. Each `OwnedCString::new` call leaks the duplicated string. Over hours or days, the program's memory usage grows. Eventually it would run out of memory and crash.

**Result:** As predicted, the program runs to completion without issues. To see the leak, you would need:

- A memory profiling tool (like Valgrind or heaptrack).
- A test that creates many `OwnedCString` instances and watches memory usage.
- Running the program long enough for the leak to be noticeable.

The lesson: missing cleanup is a category of bug that is hard to detect in normal testing. Drop implementations matter; getting them right is essential for any type that owns external resources (heap memory, file handles, network connections, etc.).

This is also why Rust's automatic Drop call is so valuable. The cleanup happens automatically; you cannot forget to call it. The risk is forgetting to implement Drop in the first place, or implementing it incorrectly.

### Checkpoints

**1. Why is `c_strlen` safe rather than unsafe?**

The function `c_strlen` takes a `&str` (a safe Rust reference) and returns a `usize` (a plain value). It does not expose any unsafe operations to the caller; the caller cannot misuse the function in a way that violates memory safety.

Internally, the function does have unsafe operations (the FFI call to `strlen`). But the function manages these internally:

- It validates the input (the CString::new call rejects strings with null bytes).
- It establishes the precondition for the FFI call (the CString provides a valid null-terminated buffer).
- It handles the result safely (the returned `usize` is a plain value).

From the caller's perspective, `c_strlen` is just a function that takes a string and returns its length. No special invariants are required of the caller; no SAFETY comments are needed at call sites.

Making `c_strlen` an `unsafe fn` would push responsibility to the caller without good reason. Every call site would need an `unsafe { c_strlen(...) }` block, and the developers would have to reason about why each call is safe. This serves no purpose; the function is just as safe as `str::len()` from the caller's perspective.

The pattern is general: encapsulate unsafe inside safe functions whenever possible. Only expose `unsafe fn` when the function has genuine preconditions the caller must verify (like `slice::get_unchecked` requiring the index to be in bounds).

**2. Send but not Sync: practical implications and decision rationale?**

The `OwnedCString` implements Send but not Sync:

- **Send means:** the type can be moved from one thread to another. After the move, the new owning thread has exclusive use.
- **Not-Sync means:** multiple threads cannot hold references to the same `OwnedCString` simultaneously.

In practice:

- You can `thread::spawn(move || use(to_send))` where `to_send` is an OwnedCString. This moves ownership; only the new thread uses it.
- You cannot easily share an `&OwnedCString` between threads, because the type is not Sync. To share, you would need an `Arc<OwnedCString>` (and Arc is Sync only when its inner type is Sync, so this would not work directly).

The decision was based on what the code actually needs:

- The example uses the OwnedCString from one thread at a time (sending to a worker, with the original thread no longer using it). This is Send.
- The example does not need multiple threads holding references to the same OwnedCString. So Sync is not needed.

By implementing only what is needed, we keep the analysis simple. Sync would require additional reasoning about thread-safe access to the methods. The methods `len()` and `to_string_lossy()` both read through `self.ptr`; concurrent reads through `&self` would be safe in principle, but verifying it requires careful analysis.

For a real production library, you would consider whether Sync should be implemented. The cost is more careful analysis; the benefit is more flexible usage. For a teaching example, keeping Sync off avoids the extra complexity.

**3. The Drop implementation: what would happen without it?**

Without the Drop implementation, the heap memory allocated by `strdup` would never be freed. Each `OwnedCString::new` call leaks the duplicated buffer.

The consequences depend on the program:

- **Short programs:** invisible. The OS reclaims memory at process exit.
- **Long-running programs:** memory usage grows over time. Eventually, the process runs out of memory and crashes.
- **Programs with many allocations:** the leak compounds. A server allocating CStrings per request leaks memory per request; under load, the leak becomes significant quickly.

The bug is invisible in normal testing because:

- Tests typically allocate a small number of CStrings.
- Test programs are short; the leak does not accumulate.
- The leak does not affect correctness, just memory usage.

To detect the bug, you would need:

- A memory profiling tool (Valgrind, heaptrack, the OS-level memory monitor).
- Stress testing that allocates many CStrings and watches memory usage.
- Long-running testing that runs the program for an extended period.

Real-world impact: memory leaks are a common bug in C and C++ code. Rust's Drop mechanism makes them much less common, because forgetting to call a destructor is impossible: Drop runs automatically. The only way to leak is to NOT IMPLEMENT Drop in the first place (or to implement it incorrectly).

For types owning external resources, implementing Drop is essential. Heap memory, file handles, network connections, mutex locks, child processes—all need cleanup in Drop. Skipping it does not just affect performance; it can affect correctness (file handles might be exhausted, network connections might be limited per process, etc.).

The lesson: when implementing a type that owns external resources, always implement Drop. The compiler does not force you to (Drop is opt-in); the responsibility is yours. Doing it right is part of the contract you take on when writing unsafe code that manages resources.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

Common patterns:

- Developers from C find raw pointers familiar (they look exactly like C pointers). The unfamiliar parts are the safety constraints: the requirement for SAFETY comments, the convention of minimal unsafe scope, the strict enforcement against using unsafe to bypass the borrow checker. The C habit of "everything is a pointer" does not transfer well.
- Developers from Java find unsafe surprising at first; Java has no equivalent. The idea that some Rust code disables safety checks feels strange to someone used to JVM-guaranteed memory safety. After working with FFI, the necessity becomes clearer.
- Developers from Python find FFI familiar (Python's ctypes module has similar concepts). The Rust difference is the upfront cost (CString construction, lifetime management) versus Python's runtime errors when things go wrong.
- Developers from systems programming backgrounds find unsafe natural; it is essentially "the parts of C++ you would use anyway, but with Rust's safety story for everything else."

The most common reflection: "The SAFETY comment convention took the longest to internalize. I am used to writing code and trusting it to work; the discipline of justifying every unsafe operation in a comment felt like overhead. After running into a bug where I had not actually checked the invariant I thought I was checking, I came to appreciate the convention. Writing the SAFETY comment forces you to actually think through the reasoning."

Another common reflection: "Send/Sync took time to understand at the concept level. In other languages, thread safety is something you discuss in code review; in Rust, it is checked by the compiler via these markers. Learning when to manually implement them was the trickiest part. The pattern of 'auto-derivation handles most cases, manual for the rare exceptions' became clearer with practice."

**2. A safer alternative I was forced to find in Rust.**

Sample answer:

"In Python projects, I have written code that mutates a shared dictionary across threads, relying on the GIL to prevent corruption. The GIL serializes Python bytecode execution, so most operations are atomic by accident. This is not guaranteed for all operations (compound updates like `d[k] += 1` can race), but it usually works.

In Rust, this pattern is impossible. A `HashMap<K, V>` is not Sync; you cannot share it across threads without explicit synchronization. The compiler forces you to pick:

- An `Arc<Mutex<HashMap<K, V>>>` for synchronized access.
- A channel for message-passing.
- A thread-local HashMap if shared state is not actually needed.

I initially found this annoying; I just wanted the GIL-like behavior. But after working with it, the explicit synchronization made my code clearer. With the Python pattern, I had no idea which operations were safe and which were races waiting to happen. With Rust's explicit synchronization, I know exactly which operations require locking.

In one project, this saved me from a real bug. I was about to share a counter across threads using `Arc<AtomicU32>`, and the compiler made me think about the ordering parameter. Reading the documentation, I realized that I needed `SeqCst` for what I wanted, not the `Relaxed` I had initially used. In Python, I would have used the default semantics and possibly had subtle wrong results under high contention.

The trade-off: Rust takes more upfront work to design concurrent code, but the runtime behavior is much more predictable. For systems where reliability matters (servers, anything with a long uptime), the trade-off is worth it. For quick scripts or single-threaded code, the GIL approach is simpler. Rust suits the former better."

**3. The trust model and how the community manages it.**

Sample answer:

"The 'safe abstractions over unsafe code' pattern creates a chain of trust:

- Application developers trust library authors to write correct unsafe code.
- Library authors trust the standard library's safe wrappers (which themselves contain unsafe).
- Everyone trusts the Rust compiler to catch issues outside the unsafe blocks.

When the trust is misplaced, bugs can be severe: memory corruption, security vulnerabilities, undefined behavior.

The community manages this risk through several mechanisms:

**Tools for verification:**

- `cargo geiger` for counting unsafe usage.
- Miri for runtime UB detection.
- `cargo-audit` for known vulnerabilities.
- `cargo-fuzz` for fuzz testing.

**Conventions for quality:**

- SAFETY comments on every unsafe block.
- Minimal unsafe scope.
- Documented `# Safety` sections on unsafe functions.
- Code review focused on unsafe regions.

**Community processes:**

- Crates.io is centralized; popular crates accumulate community usage and scrutiny.
- The Rust Foundation funds audits of critical infrastructure.
- The Rust Security Response Working Group handles disclosed vulnerabilities.
- The unsafe-code-guidelines effort works to clarify what unsafe means precisely.

**Cultural norms:**

- The Rust community treats unsafe-related bugs as serious; bug reports get attention.
- Maintainers of crates with significant unsafe usage tend to be careful and responsive.
- The community pushes back against gratuitous unsafe; PRs that add unsafe without justification get challenged.

The combination is what makes Rust's safety story credible despite the existence of `unsafe`. Most application code is safe by construction. Library code that contains unsafe is concentrated in a relatively small number of well-maintained crates. Bugs do happen (memory safety issues have been found in popular crates), but they are rare and get addressed quickly.

For my own projects, I follow the discipline:

- I do not write unsafe in application code.
- I use widely-adopted crates for FFI and low-level work.
- I read SAFETY comments when reviewing dependencies.
- I run `cargo audit` in CI.
- I report security issues through proper channels when I find them.

This is what the trust looks like in practice: not blind trust, but trust based on observable signals (maintainer activity, community scrutiny, available verification tools). It is the same approach you would take with any infrastructure dependency; Rust's tooling makes the signals more visible than in many other ecosystems."

---

*End of Lab 15 Solutions*
