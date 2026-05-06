# Module 2: Setting Up the Rust Environment
## Lab 1 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## How to Use This File

Each section below corresponds to an exercise in the lab. Within each exercise, the completed code or commands appear first, followed by written answers to the Checkpoint questions. The structure mirrors the lab file exactly.

---

## Exercise 1 -- Inspect Your Rust Installation

### Task 1.1 -- Sample expected output

`rustup show` should produce output similar to:

```
Default host: x86_64-unknown-linux-gnu
rustup home:  /home/student/.rustup

installed toolchains
--------------------
stable-x86_64-unknown-linux-gnu (active, default)

active toolchain
----------------
stable-x86_64-unknown-linux-gnu (default)
rustc 1.84.0 (9fc6b4312 2025-01-07)
```

`rustup component list --installed` should include all of:

```
cargo-x86_64-unknown-linux-gnu
clippy-x86_64-unknown-linux-gnu
rust-docs-x86_64-unknown-linux-gnu
rust-std-x86_64-unknown-linux-gnu
rustc-x86_64-unknown-linux-gnu
rustfmt-x86_64-unknown-linux-gnu
```

### Task 1.2 -- Locating binaries

The Cargo binary directory on Linux and macOS is `~/.cargo/bin`. On Windows it is `%USERPROFILE%\.cargo\bin`.

The full path to `rustc` is the rustup proxy at `~/.cargo/bin/rustc` (or the Windows equivalent). This is not the actual compiler binary. It is a small shim that reads `rust-toolchain.toml` (if present), determines which toolchain should be active, and forwards to the real compiler at `~/.rustup/toolchains/<toolchain-name>/bin/rustc`. The shim is what allows different projects to use different toolchains without any reconfiguration.

### Checkpoints

**1. Why does `rustup` install to a per-user directory instead of system-wide?**

A per-user installation has several advantages for a development team:

- No root or administrator privileges are required to install or update Rust. This is critical on shared machines, build servers, and locked-down corporate environments.
- Multiple developers on the same machine can use different Rust versions without affecting each other.
- Updates do not require system package manager involvement and do not have to wait for OS package maintainers.
- An uninstall is clean. Removing `~/.rustup` and `~/.cargo` leaves no system-wide artifacts behind.
- CI systems can install rustup without privilege escalation, which is a security improvement over running package installers as root.

**2. Toolchain directories**

Right after installation, there is typically one toolchain directory: `stable-<host-triple>`. After completing Exercise 7 with a pinned `1.84.0`, you may see two directories. Each directory contains a complete Rust installation: `bin/`, `lib/`, `share/`, and so on. Toolchains are independent of each other; you can have a broken nightly without affecting stable.

---

## Exercise 2 -- Create and Run Your First Cargo Project

### Task 2.4 -- Sample binary sizes

Approximate binary sizes for the default `Hello, world!` program:

- Debug: 4 to 6 MB on Linux, slightly larger on Windows.
- Release: 400 KB to 600 KB on Linux.

The exact numbers vary by platform and compiler version. The release binary is roughly 10x smaller. Most of the size in the debug binary comes from debug symbols and the unoptimized standard library. The release binary still contains a fair amount of code because Rust statically links the standard library by default. To produce truly small binaries, additional steps (`strip`, `panic = "abort"`, `lto = true`) are required. These are covered in a later module on production builds.

### Checkpoints

**1. What did Cargo determine had not changed?**

Cargo computed a hash of every source file (and every dependency, build script, profile setting, and feature flag) and compared it against the hash from the previous build. Nothing had changed, so no recompilation was needed. The build artifacts in `target/debug/` were valid and Cargo simply re-ran the existing binary. This is incremental compilation. It is the reason `cargo run` feels almost free for small edits.

**2. Why does Cargo separate debug and release artifacts?**

The two profiles produce binaries with completely different optimization levels, debug symbols, and panic behavior. They cannot be intermixed. By keeping them in separate directories (`target/debug/` and `target/release/`), Cargo can have both a debug build and a release build available simultaneously, without recompiling when you switch between them. This matters during development, when you might run `cargo run` (debug) for a quick test and then `cargo run --release` (release) to verify performance, alternating between the two.

**3. What does the longer release compile time buy you?**

The release profile enables optimization passes that take time to run: inlining, dead code elimination, loop unrolling, vectorization, link-time optimization in some configurations, constant folding across function boundaries, and many others. The result is code that often runs 5-100x faster than the debug build, depending on the workload. The compile-time cost is paid once. The runtime benefit is paid every time the program runs in production.

---

## Exercise 3 -- Open the Project in VS Code

### Task 3.2 -- Recommended settings

The settings JSON entries are reproduced from the lab file:

```json
{
  "editor.formatOnSave": true,
  "[rust]": {
    "editor.defaultFormatter": "rust-lang.rust-analyzer",
    "editor.formatOnSave": true
  },
  "rust-analyzer.check.command": "clippy",
  "rust-analyzer.inlayHints.parameterHints.enable": true,
  "rust-analyzer.inlayHints.typeHints.enable": true,
  "rust-analyzer.inlayHints.chainingHints.enable": true
}
```

### Checkpoints

**1. What component is performing the type inference?**

`rust-analyzer` is performing the type inference. It is a separate program from `rustc`, although it shares some library code with the compiler. rust-analyzer maintains an in-memory model of the entire crate and updates it incrementally as you type. It performs the same Hindley-Milner type inference that the compiler does, but it is optimized for partial, often-broken code, where the compiler is optimized for complete, correct code.

The key difference is responsiveness. `cargo build` runs the full compilation pipeline and produces an executable. rust-analyzer runs only the analysis passes, in memory, and produces diagnostics. This is why rust-analyzer can give feedback in milliseconds while `cargo build` typically takes seconds even on small projects.

**2. How does inline error reporting change debugging habits?**

Workflows that wait for save-and-build feedback encourage longer edits between checks. Inline error reporting encourages smaller edits with immediate verification. This shortens the debug loop and makes it easier to learn the type system, because you see the consequences of every edit as you type. It also reduces the cost of experimentation; trying a refactor and seeing immediately whether it compiles is much cheaper than running a full build cycle.

In practice, developers who come from inline-error environments (TypeScript with TS Server, Java with IntelliJ) adapt to rust-analyzer easily. Developers coming from save-and-build workflows often need a few sessions before they trust the editor's diagnostics.

**3. Why does the code lens only appear above main and tests?**

The Run and Debug code lens is produced for entry points that rust-analyzer can directly invoke. `fn main()` is the entry point of a binary. Test functions, marked with `#[test]`, can be invoked individually by the test runner. Both have well-defined invocation semantics that the code lens can target.

Arbitrary functions cannot be invoked directly. There is no way to "run" a function called `compute_total(items: &[Item]) -> f64` without the rest of the program providing arguments. Adding a Run lens above arbitrary functions would either be misleading or would require a synthetic input dialog, neither of which is desirable.

---

## Exercise 4 -- Format Your Code with rustfmt

### Task 4.4 -- Sample failure output

`cargo fmt -- --check` on a misformatted file produces a diff like:

```
Diff in /path/to/src/main.rs at line 1:
-fn   main()    {
-    let greeting="Hello, world!";println!("{greeting}");
+fn main() {
+    let greeting = "Hello, world!";
+    println!("{greeting}");
 }
```

The exit code is 1.

### Checkpoints

**1. What does the project gain by giving up formatter choice?**

Three concrete benefits:

- Code review focuses on logic instead of style. A reviewer who sees a long line in clippy never has to ask "should this wrap?" because the formatter already decided.
- New hires onboard faster. They do not have to learn a project-specific style or configure their editor differently from public Rust code they have seen.
- Mechanical churn from style debates does not appear in version control history. `git blame` reliably shows who last meaningfully edited a line, not who reformatted it.

The community accepts that the default style is not perfect for every line of code in every situation. The collective benefit of consistency outweighs the occasional individual annoyance.

**2. When is `--check` mode preferable to applying fixes?**

Two clear contexts:

- Continuous Integration. CI must not modify source files; it must report whether the source is correct. `--check` returns a non-zero exit code on any formatting issue, which fails the build.
- Pre-commit hooks that report problems but want the developer to choose whether to fix them. Some teams prefer pre-commit hooks that warn rather than auto-fix, because auto-fixing can hide unexpected diffs from the developer.

**3. Setting `max_width = 60`**

Long lines will be wrapped more aggressively. For example, `println!("Hello, world!")` might stay on one line, but a longer expression like `let total: i32 = numbers.iter().map(|n| n * 2).sum();` will likely be split across multiple lines. The behaviour confirms that `rustfmt.toml` configuration is honored. The community convention is to leave the default 100 unless there is a specific style guide reason.

---

## Exercise 5 -- Lint Your Code with clippy

### Task 5.3 -- Idiomatic version

```rust
fn main() {
    let numbers = vec![10, 20, 30, 40, 50];

    let total: i32 = numbers.iter().sum();
    let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
    let first = doubled.first();

    println!("Total: {total}");
    println!("First doubled: {first:?}");
}
```

### Checkpoints

**1. Why are clippy and rustc separate?**

Several reasons:

- `rustc` lints must apply to all Rust code, everywhere. They are very conservative because false positives in `rustc` would be catastrophic. clippy lints are looser and more opinionated; some are correct in 90 percent of cases but wrong in the other 10 percent. Keeping them separate means clippy can be aggressive without compromising the compiler.
- Release cadence and stability requirements differ. clippy lints can be added, modified, and even removed between releases. `rustc` warnings, once stabilized, are very difficult to change.
- Compile time matters more for `rustc`. Every clippy lint adds a small amount of analysis cost. Bundling all of them into the compiler would slow down every build, even when the developer does not want lint feedback.

**2. Why flag `total = total + numbers[i]`?**

Both forms compile to identical machine code. clippy still flags the longer form because the lint goal is not performance; it is readability and idiomatic style. `total += value` is the conventional Rust expression. Code reviewers expect to see it. Flagging the longer form keeps the codebase uniformly idiomatic, which makes it easier to read code from any author.

This is also a good example of why clippy's `style` category exists separately from `correctness` and `performance`. A style lint is not catching a bug; it is catching code that does not look like Rust.

**3. A lint category you would NOT enable**

`pedantic` and `restriction` are common answers.

The `pedantic` category contains lints that enforce a strict, sometimes annoying style. Some of these lints are reasonable defaults for new projects, but they generate enough noise on existing codebases to be impractical.

The `restriction` category contains lints that are not "good practices" at all. They are restrictions on the language for projects with specific, unusual requirements. For example, `restriction::float_arithmetic` denies all floating-point math, useful for an embedded system that lacks an FPU but absurd for a graphics engine.

These lints exist because clippy is the right place to express any opinion about Rust code, even if that opinion is not universally applicable. Each lint is opt-in, so the project that does need to ban floating-point math can do so by enabling exactly that lint.

---

## Exercise 6 -- Debug a Program with CodeLLDB

### Task 6.6 -- Fixed implementation

```rust
fn find_max(values: &[i32]) -> Option<i32> {
    let mut max = *values.first()?;
    for value in values {
        if *value > max {
            max = *value;
        }
    }
    Some(max)
}

fn main() {
    let numbers = vec![5, 12, 7, 3, 9];
    let result = find_max(&numbers);
    println!("Max of {numbers:?} is {result:?}");

    let negatives = vec![-5, -12, -7, -3, -9];
    let result = find_max(&negatives);
    println!("Max of {negatives:?} is {result:?}");

    let empty: Vec<i32> = vec![];
    let result = find_max(&empty);
    println!("Max of {empty:?} is {result:?}");
}
```

A more idiomatic version that demonstrates how Rust developers typically express this with iterators:

```rust
fn find_max(values: &[i32]) -> Option<i32> {
    values.iter().copied().max()
}
```

The `Iterator::max` method already returns an `Option` for exactly the reason discussed in the lab: the maximum of an empty collection is undefined.

### Checkpoints

**1. What category of bug, and what tool addresses it?**

This is a logic bug, sometimes called an off-by-default or boundary bug. The function works correctly on the inputs it was designed for (positive numbers) and fails silently on inputs the author did not consider (negatives, empty slice).

The compiler cannot catch this kind of bug because the code has well-defined behaviour for every input. clippy cannot catch it because the suspicious pattern (initializing an accumulator with a constant) is also a perfectly valid pattern when the constant is correct.

The right tool is the test suite. Specifically, a test that includes negative numbers and empty input would have caught this immediately. Property-based testing tools like `proptest` and `quickcheck` are even more effective; they generate inputs you would not have thought of, including empty inputs and inputs with extreme values.

**2. Why is the `Option<i32>` signature more honest?**

The original `i32` return type promised the caller a maximum for every possible input. That promise is impossible to keep when the input is empty: no maximum exists. The function fulfilled the type signature only by lying (returning 0, which is plausibly a real value). The caller has no way to detect the lie.

`Option<i32>` makes the partiality explicit. The signature now matches reality: there is sometimes a maximum, and sometimes there is not. The caller is forced to handle both cases. This is one of the central design principles in Rust: encode preconditions and postconditions in the type system, so the compiler enforces them rather than relying on runtime checks or convention.

**3. Where do CodeLLDB's formatting rules come from?**

CodeLLDB ships with a set of LLDB type summary providers (called "data formatters" in LLDB terminology) for the standard library types. They are written as LLDB scripts that know how to interpret a `Vec<T>`'s `(ptr, len, capacity)` triple as a sequence of `T` values, a `String` as a UTF-8 byte sequence, and so on.

Without these formatters, a `Vec<i32>` containing `[1, 2, 3]` would display as a struct with three numeric fields: a pointer, a length, and a capacity. To see the actual elements you would have to manually dereference the pointer and read three consecutive integers, which is tedious and error-prone.

This is also the reason that the same program debugged with a different IDE (or with `lldb` directly without the CodeLLDB formatters) may show much less helpful output. The formatters are a CodeLLDB feature, not a Rust language feature.

---

## Exercise 7 -- Pin a Toolchain to the Project

### Task 7.1 -- Final rust-toolchain.toml

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy", "rust-analyzer"]
profile = "default"
```

### Task 7.3 -- Pinned to a specific version

```toml
[toolchain]
channel = "1.84.0"
components = ["rustfmt", "clippy", "rust-analyzer"]
profile = "default"
```

### Checkpoints

**1. Why is auto-downloading the toolchain preferable?**

Two reasons:

- Reproducibility. Every build of the project, on every machine, in CI, in production, uses the exact same compiler. There is no "works on my machine" scenario stemming from a version mismatch. Bug reports referring to the project can be reproduced exactly.
- Onboarding. A new hire can clone the repository and have a working build with no Rust knowledge. They do not have to know what version the project uses, where to install it, or how to switch toolchains. The act of running `cargo build` configures their environment correctly.

The cost is one slow first build while the toolchain downloads. After that the toolchain is cached on the developer's machine and subsequent projects pinned to the same version use the same cache.

**2. Trade-off: `1.84.0` versus `1`**

Pinning to `1.84.0` exactly gives complete reproducibility. Every build uses the same compiler, period. The cost is that you must explicitly bump the version number when you want to upgrade, and you stay on an older compiler until you do.

Pinning to `1` would mean any 1.x compiler is acceptable. Builds across machines and CI runs could use different patch versions or even minor versions. This is more permissive but breaks reproducibility. A bug found in production might not reproduce locally because the compiler version differs.

In practice, projects that care about reproducibility (most production services) pin exactly. Projects that care about following the latest stable (most libraries) often use `channel = "stable"` and update their CI matrix periodically.

**3. Why pin rust-analyzer?**

Two reasons:

- The `rustup`-managed copy of rust-analyzer is updated when the toolchain is updated. If the team pins the compiler version, the analyzer should match. Otherwise the in-editor diagnostics may use language features the compiler does not yet understand, or vice versa.
- New hires get rust-analyzer automatically. Even if their VS Code extension would download its own copy, having the rustup-managed copy guarantees that command-line invocations of `rust-analyzer` (used by some tooling and CI scripts) work without additional setup.

The editor's bundled copy and the rustup-managed copy can coexist. The editor will typically prefer its own. Pinning rust-analyzer is mostly insurance.

---

## Exercise 8 -- Read Compiler Output in the Rust Playground

### Checkpoints

**1. Two specific differences between Debug and Release assembly**

Concrete observations students should record:

- In the Debug output, `count_alpha` contains explicit calls to functions like `core::str::Chars::next` and `<char as core::primitive::Char>::is_alphabetic`. In the Release output, these calls disappear; the body of each is inlined directly into the loop.
- The Debug output contains more memory loads and stores, often using stack slots for intermediate values. The Release output keeps almost everything in registers, and the loop body is a small handful of instructions.

The implication: in Debug builds, the iterator chain executes as a series of small function calls with full state. In Release builds, the entire chain has been fused into a single loop with no abstraction overhead at all.

**2. Why `black_box` is necessary in this exercise but not in production**

The Rust optimizer is aggressive about constant propagation. The string `"Hello, Rustaceans!"` is a constant. The function `count_alpha` is pure (no side effects, depends only on its input). The optimizer can therefore evaluate `count_alpha("Hello, Rustaceans!")` at compile time, producing the literal `17`. At that point, all the assembly we wanted to inspect has been deleted.

`black_box` is an opaque function that the optimizer cannot see through. Wrapping a value in `black_box` forces the optimizer to assume the value is unknown and that any code consuming it must actually run.

In production code, the inputs come from user input, files, network responses, and other sources that are inherently runtime values. The optimizer cannot constant-fold across those boundaries, so the protection `black_box` provides is automatic. We only need it in the Playground because the example's input is a hard-coded constant chosen to make the demo readable.

**3. Right-tool situations for the Playground vs a local Cargo project**

Playground is the right tool when:

- Asking a question on a forum, where reviewers need to run the exact code to reproduce the issue.
- Testing whether a small language feature behaves as expected (a one-liner, a short example).

Local Cargo project is the right tool when:

- The code has more than one file, requires a build script, or needs to integrate with system libraries.
- The project must be checked into version control or shared with a team.
- The work involves dependencies the Playground does not include.
- The build needs custom profile settings, conditional compilation features, or a specific toolchain.

---

## Final Reflection Questions

**1. Bundled vs separate tools**

Bundled with the official toolchain by default:
- `rustup` (the toolchain installer)
- `cargo` (the build system, bundled in every toolchain)
- `rustc` (the compiler, the core of every toolchain)
- `rustfmt` (bundled with stable; can be added to other channels)
- `clippy` (bundled with stable; can be added to other channels)

Optional or separate:
- `rust-analyzer` (an installable component, but most users get it from their editor's extension)
- VS Code (a separate IDE, not part of the Rust project)
- CodeLLDB (a third-party VS Code extension built on LLDB)

The pattern is that anything required to compile correct Rust code lives in the official toolchain. Anything that improves the experience but is not strictly required is treated as optional. The IDE choice is left to the developer because the Rust team does not want to mandate a specific editor; rust-analyzer's commitment to LSP is what makes that choice work.

**2. Three categories of bugs the compiler does not catch**

- **Logic bugs.** The compiler verifies that the code is well-typed and memory-safe but cannot determine whether the code computes what the developer intended. The fix is the test suite (`cargo test`), property-based testing (`proptest`), and code review.
- **Performance regressions.** The compiler will accept O(n^2) code that should be O(n). The fix is benchmarking (`criterion`), profiling (`perf`, `flamegraph`), and load testing.
- **Concurrency bugs that are not data races.** Rust's borrow checker prevents data races, but it does not prevent deadlocks, livelocks, or higher-level concurrency bugs like incorrect ordering of operations. Tools that help include `loom` for testing concurrent code, `tokio-console` for async runtime inspection, and traditional testing under load.

**3. Reversing rust-analyzer.check.command from clippy back to check**

The default `cargo check` runs only the compiler's own diagnostics. It is significantly faster than `cargo clippy` because clippy adds hundreds of additional analyses. On a small project the difference is unnoticeable. On a large project (tens of thousands of lines, many crates), clippy might take 30+ seconds to run after every save, which interferes with the inline-error-feedback workflow that makes rust-analyzer useful in the first place.

Reasonable choices for large codebases:

- Use `check` for the editor's save-time analysis, and run `clippy` separately on a slower cadence (in CI, in a pre-commit hook, or on demand via a keybinding).
- Use `clippy` but configure it to skip the most expensive lint groups (`pedantic`, `nursery`).
- Split the codebase into smaller crates so that rust-analyzer only re-analyzes the affected crate after each save.

The right choice depends on team workflow and codebase shape. A team that values continuous lint feedback might accept the slower save behaviour. A team that values fast iteration might run clippy only in CI.

---

*End of Lab 1 Solutions*
