# Module 2: Setting Up the Rust Environment
## Lab 1 -- Your First Rust Program and the Rust Toolchain

> **Course:** Mastering Rust
> **Module:** 2 - Setting Up the Rust Environment
> **Estimated time:** 60-90 minutes

---

## Overview

This lab takes you from a freshly installed Rust toolchain to a working "Hello, World" program, then expands the program enough to exercise every part of the toolchain you met in Module 2. The focus is the developer workflow, not the language. You will write very little Rust code. What matters is that by the end of the lab, the editor, compiler, formatter, linter, and debugger all behave the way you expect them to on your machine.

By the end of this lab you will be able to:

- Verify your `rustup`, `cargo`, and `rustc` installations and inspect installed components.
- Create, build, and run a Cargo project from the command line and from VS Code.
- Use `cargo fmt` and `cargo clippy` to enforce style and catch lint violations.
- Configure VS Code with `rust-analyzer` for inline diagnostics, hover information, and inlay hints.
- Set breakpoints and step through Rust code with the CodeLLDB debugger.
- Pin a Rust toolchain to a project and switch between channels.
- Read assembly output produced by the Rust compiler in the Rust Playground.

---

## Before You Start

Confirm that the following are installed and working before beginning the exercises. Each command should print a version string. If any command fails, return to Module 2 and complete the installation steps.

```bash
rustup --version
rustc --version
cargo --version
```

The expected output is similar to:

```
rustup 1.27.1 (or newer)
rustc 1.84.0 (or newer)
cargo 1.84.0 (or newer)
```

You should also have:

- VS Code installed (https://code.visualstudio.com).
- The **rust-analyzer** extension installed (publisher: `rust-lang`).
- The **CodeLLDB** extension installed (publisher: `vadimcn`).
- Network access for `cargo` to fetch dependencies from `crates.io`.

> **Note:** Throughout this lab, command-line examples use `$` as the shell prompt on Linux and macOS. On Windows, run the same commands in PowerShell or the integrated VS Code terminal.

---

## Lab Project Layout

You will work in a single Cargo project named `hello_rustacean` for this lab. All exercises build on the same project. Choose a directory for your coursework and remember the path. A typical choice:

```
~/rust-course/labs/
```

Open a terminal in that directory before continuing.

---

## Exercise 1 -- Inspect Your Rust Installation

**Estimated time:** 5-10 minutes
**Topics covered:** `rustup show`, components, toolchain layout

### Context

Before writing any code, you need to know exactly what is installed. `rustup show` is the single most useful diagnostic command. Whenever a teammate asks for help with a broken Rust setup, the first request should be the output of `rustup show`. The command reveals which toolchains are installed, which is the default, and which is active in the current directory.

### Task 1.1 -- Survey installed toolchains

Run the following commands and copy the output into a file called `lab1-notes.md` (create it in your lab directory). You will add to this file throughout the lab.

```bash
rustup show
rustup component list --installed
rustup toolchain list
```

Expected output for `rustup component list --installed` should include at minimum:

```
cargo-x86_64-...
clippy-x86_64-...
rust-docs-x86_64-...
rust-std-x86_64-...
rustc-x86_64-...
rustfmt-x86_64-...
```

> **Common mistake:** If `clippy` or `rustfmt` is missing, your installation is incomplete. Run `rustup component add clippy rustfmt` to install them. They are bundled with stable by default but a custom installation may have skipped them.

### Task 1.2 -- Locate the toolchain on disk

Run the following command. The output reveals where Cargo's home directory is and where binaries on your PATH actually live.

```bash
# Linux and macOS
ls ~/.cargo/bin

# Windows PowerShell
Get-ChildItem $env:USERPROFILE\.cargo\bin
```

You should see executables for `rustc`, `cargo`, `rustup`, `rustfmt`, `cargo-clippy`, and `cargo-fmt`.

In `lab1-notes.md`, answer:

1. Where on your machine does Cargo install user binaries? (Hint: the directory you just listed.)
2. What is the full path to the `rustc` binary your system actually runs when you type `rustc`?

### Checkpoints

1. Why does `rustup` install to a per-user directory (`~/.cargo`, `~/.rustup`) instead of a system-wide location like `/usr/local/bin`? What advantages does this give a development team?
2. Open `~/.rustup/toolchains/`. How many toolchain directories do you see? What does each one contain?

---

## Exercise 2 -- Create and Run Your First Cargo Project

**Estimated time:** 10 minutes
**Topics covered:** `cargo new`, project layout, `cargo build`, `cargo run`, debug vs release

### Context

Cargo is the Rust build system and package manager. It manages dependencies, compilation, testing, and many other tasks. Almost no Rust code is built by invoking `rustc` directly. The standard workflow is `cargo new` followed by edits inside the generated project structure.

A Cargo project always has the same layout:

```
hello_rustacean/
├── Cargo.toml      <- project metadata and dependencies
├── Cargo.lock      <- exact versions of every dependency (auto-generated)
├── .gitignore      <- generated by cargo new
├── src/
│   └── main.rs     <- entry point for binary crates
└── target/         <- build artifacts (created on first build, gitignored)
```

### Task 2.1 -- Create the project

In your lab directory, run:

```bash
cargo new hello_rustacean
cd hello_rustacean
```

`cargo new` generates the directory structure shown above and initializes a Git repository. List the contents to confirm:

```bash
# Linux and macOS
ls -la

# Windows PowerShell
Get-ChildItem -Force
```

Open `Cargo.toml` and inspect it:

```toml
[package]
name = "hello_rustacean"
version = "0.1.0"
edition = "2024"

[dependencies]
```

The `edition` field selects the language edition (covered in a later module). The `[dependencies]` section is empty; you will add to it later.

### Task 2.2 -- Read the generated source

Open `src/main.rs`. Cargo has placed a minimal program there:

```rust
fn main() {
    println!("Hello, world!");
}
```

A few observations to note in `lab1-notes.md`:

1. There is no `import` or `include` statement. The `println!` macro lives in the standard library prelude, which is imported into every Rust program automatically.
2. `println!` is followed by an exclamation mark. This is how Rust spells macro invocations. It is not a regular function call.
3. `fn main() { ... }` is the program entry point. The signature returns the implicit unit type `()`.

### Task 2.3 -- Build and run

Run the program:

```bash
cargo run
```

Expected output:

```
   Compiling hello_rustacean v0.1.0 (/path/to/hello_rustacean)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.42s
     Running `target/debug/hello_rustacean`
Hello, world!
```

Run it a second time:

```bash
cargo run
```

Expected output:

```
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/hello_rustacean`
Hello, world!
```

The second run does not recompile because nothing changed. Cargo's incremental compilation tracks file timestamps and source hashes.

### Task 2.4 -- Compare debug and release builds

Cargo has two default profiles. The `dev` profile prioritizes fast compilation and good debugging information. The `release` profile prioritizes runtime performance.

Build with the release profile:

```bash
cargo build --release
```

Expected output ends with:

```
    Finished `release` profile [optimized] target(s) in 1.28s
```

Now compare the binary sizes:

```bash
# Linux and macOS
ls -lh target/debug/hello_rustacean target/release/hello_rustacean

# Windows PowerShell
Get-ChildItem target\debug\hello_rustacean.exe, target\release\hello_rustacean.exe |
    Select-Object Name, Length
```

The release binary is significantly smaller and faster. Record both sizes in `lab1-notes.md`.

### Checkpoints

1. The first `cargo run` showed `Compiling hello_rustacean v0.1.0`. The second run showed nothing about compilation. What did Cargo determine had not changed?
2. Inspect `target/`. Identify which subdirectory holds debug artifacts and which holds release artifacts. Why does Cargo separate them rather than rebuilding in place?
3. The release build took roughly three times longer than the debug build. What does that compile-time cost buy you at runtime?

---

## Exercise 3 -- Open the Project in VS Code

**Estimated time:** 10-15 minutes
**Topics covered:** `rust-analyzer` integration, inlay hints, hover diagnostics, code lens

### Context

VS Code with `rust-analyzer` provides far more than syntax highlighting. The language server runs incrementally in the background, providing the same type information the compiler would. Most of the productivity benefit of using Rust at scale comes from this tooling.

### Task 3.1 -- Open the project

From the project directory:

```bash
code .
```

VS Code should open with the project root visible in the Explorer. The first time you open a Rust file, `rust-analyzer` will display a notification reading "Indexing" or similar in the status bar. Wait for indexing to finish before continuing. On a small project like this one, indexing takes only a few seconds.

> **Common mistake:** If you see only basic syntax highlighting and no inline annotations, `rust-analyzer` is not active. Open the Output panel (View > Output) and select `rust-analyzer` from the dropdown. Look for error messages. The most common cause is that the VS Code extension is installed but disabled, or the workspace has not been trusted.

### Task 3.2 -- Configure recommended settings

Open the Command Palette (Ctrl+Shift+P or Cmd+Shift+P) and run "Preferences: Open User Settings (JSON)". Add the following entries. If the file already has settings, merge these in.

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

The most important entry is `rust-analyzer.check.command: clippy`. This makes save-time diagnostics use clippy's expanded ruleset instead of just the compiler's checks.

Save the settings file. Changes take effect immediately.

### Task 3.3 -- Observe rust-analyzer features

Open `src/main.rs`. Make the following observations and record your findings in `lab1-notes.md`.

**Observation 1: Inlay hints**

Modify `main.rs` to declare a variable:

```rust
fn main() {
    let greeting = "Hello, world!";
    println!("{greeting}");
}
```

After saving, look at the line `let greeting`. You should see a small annotation showing `: &str` next to the variable name. This is an inlay hint: rust-analyzer is showing you the inferred type even though you did not write it. The annotation is not part of the source file.

**Observation 2: Hover information**

Hover the cursor over `println`. A popup appears showing the macro signature, documentation, and the source location.

Hover over `greeting`. The popup shows the inferred type and a quick way to navigate to the declaration.

**Observation 3: Code lens**

Above `fn main()`, you should see the words `Run | Debug` in faded text. This is a code lens: a clickable annotation provided by rust-analyzer.

Click `Run`. A new terminal panel opens and runs `cargo run`. Click `Debug` and you will get an error because no debug configuration exists yet. You will fix that in Exercise 6.

### Task 3.4 -- Trigger a deliberate error

Replace `main.rs` with this incorrect code:

```rust
fn main() {
    let count: i32 = "five";
    println!("Count: {count}");
}
```

Save the file. Within a second or two, rust-analyzer will:

- Underline `"five"` with a red squiggle.
- Add a red dot in the gutter next to that line.
- Display an error in the Problems panel (Ctrl+Shift+M or Cmd+Shift+M).

Hover over the underline. The error message reads:

```
mismatched types
expected `i32`, found `&str`
```

This diagnostic appeared without running `cargo build`. rust-analyzer reports compilation errors as you type. Restore the original `main.rs` (the version with `let greeting = "Hello, world!"`) before continuing.

### Checkpoints

1. The inlay hint showing `: &str` was visible without running the compiler. What component is performing the type inference, and how is it different from `cargo build`?
2. You saw an error in the editor before saving. Some IDE configurations only show errors after a save or build. How might the rust-analyzer behaviour change your debugging habits compared to those workflows?
3. The code lens above `fn main()` says `Run | Debug`. Why does this only appear above `main` and test functions, and not above arbitrary functions?

---

## Exercise 4 -- Format Your Code with rustfmt

**Estimated time:** 5-10 minutes
**Topics covered:** `cargo fmt`, format-on-save, configuration

### Context

`rustfmt` enforces a single canonical Rust style. The Rust community deliberately avoided the formatting wars common in other ecosystems by establishing one official formatter and discouraging configuration. Almost all open-source Rust uses the defaults.

### Task 4.1 -- Write deliberately ugly code

Replace `src/main.rs` with this version. The whitespace is intentionally bad:

```rust
fn main(){
let     greeting    =      "Hello, world!"   ;
        let language="Rust";
   println!("{greeting} from {language}!"  );
                let numbers=vec![1,2,3,    4,5];
let total:i32=numbers.iter().sum();
        println!("Total: {total}");
}
```

Save the file.

### Task 4.2 -- Format with cargo fmt

In the terminal, run:

```bash
cargo fmt
```

The command produces no output on success. Reopen `main.rs`. The file has been rewritten in canonical style:

```rust
fn main() {
    let greeting = "Hello, world!";
    let language = "Rust";
    println!("{greeting} from {language}!");
    let numbers = vec![1, 2, 3, 4, 5];
    let total: i32 = numbers.iter().sum();
    println!("Total: {total}");
}
```

### Task 4.3 -- Verify format-on-save

Make the file ugly again:

```rust
fn   main()    {
    let greeting="Hello, world!";println!("{greeting}");
}
```

Save the file. If you configured `editor.formatOnSave` correctly in Exercise 3, the file will be reformatted instantly on save. The formatter produces:

```rust
fn main() {
    let greeting = "Hello, world!";
    println!("{greeting}");
}
```

> **Common mistake:** If format-on-save is not working, check three things: the `editor.formatOnSave` setting is `true`, the `[rust]` block specifies `rust-analyzer` as the default formatter, and you do not have another formatter extension that has registered itself for Rust files.

### Task 4.4 -- Use cargo fmt in check mode

In CI pipelines, you do not want `cargo fmt` to modify files. Instead you want to fail the build if any file is not formatted correctly. The `--check` flag does exactly this:

```bash
cargo fmt -- --check
```

If everything is formatted correctly, the command exits silently with code 0. To see what failure looks like, deliberately misformat the file again, save it without format-on-save (use Ctrl+K S in VS Code to save without formatting), and re-run:

```bash
cargo fmt -- --check
```

The output is a diff showing what would change. The exit code is non-zero. This is the form used in CI.

Restore the file, save normally, and run `cargo fmt -- --check` once more to confirm a clean exit.

### Checkpoints

1. The default `rustfmt` style uses 4-space indentation, no tabs, and a 100-column width. None of these were your choice. What benefit does the project gain by giving up that choice?
2. You ran `cargo fmt -- --check` instead of `cargo fmt`. Identify two contexts where the check-only mode is preferable to applying fixes.
3. Open `Cargo.toml` and add a `rustfmt.toml` file at the project root containing `max_width = 60`. Run `cargo fmt` again. What happens to long lines? Restore the file by deleting `rustfmt.toml`.

---

## Exercise 5 -- Lint Your Code with clippy

**Estimated time:** 10-15 minutes
**Topics covered:** `cargo clippy`, common lints, allow and deny attributes, idiomatic Rust

### Context

clippy is a separate Rust compiler that runs the same parsing and type-checking machinery but evaluates over 700 additional lints. Lints fall into categories such as `correctness` (likely bugs), `suspicious` (probably wrong), `complexity` (over-engineered code), `performance` (slow patterns), and `style` (non-idiomatic but harmless).

clippy's main value to a new Rust developer is teaching idioms. When clippy says "use `iter().sum()` instead of a manual loop," it is correcting code that compiles correctly but does not look like Rust.

### Task 5.1 -- Write deliberately non-idiomatic code

Replace `src/main.rs` with this version:

```rust
fn main() {
    let numbers = vec![10, 20, 30, 40, 50];

    let mut total = 0;
    for i in 0..numbers.len() {
        total = total + numbers[i];
    }

    let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
    let first = doubled.iter().nth(0);

    println!("Total: {}", total);
    println!("First doubled: {:?}", first);
}
```

This code compiles and runs correctly. But it is full of lint targets.

### Task 5.2 -- Run clippy

```bash
cargo clippy
```

clippy will report several warnings. The output looks similar to this (your exact wording may differ between clippy versions):

```
warning: the loop variable `i` is only used to index `numbers`
 --> src/main.rs:5:14
  |
5 |     for i in 0..numbers.len() {
  |              ^^^^^^^^^^^^^^^^
  |
  = help: consider using an iterator: `for n in &numbers`

warning: manual implementation of an assign operation
 --> src/main.rs:6:9
  |
6 |         total = total + numbers[i];
  |         ^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = help: replace with: `total += numbers[i]`

warning: called `.iter().nth(0)` on a slice
 --> src/main.rs:11:17
  |
11|     let first = doubled.iter().nth(0);
  |                 ^^^^^^^^^^^^^^^^^^^^^
  |
  = help: try: `doubled.first()`
```

Read every warning. clippy explains the issue and proposes a fix.

### Task 5.3 -- Apply the fixes manually

Rewrite `main.rs` based on clippy's suggestions:

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

Run `cargo clippy` again. The output should be clean:

```
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.18s
```

Run `cargo run` to confirm the output matches the previous version:

```
Total: 150
First doubled: Some(20)
```

The behaviour is identical. The code is shorter, more idiomatic, and matches what an experienced Rust developer would have written.

### Task 5.4 -- Use cargo clippy --fix

clippy can apply many fixes automatically. To see this in action, restore the non-idiomatic version of `main.rs` from Task 5.1.

Run:

```bash
cargo clippy --fix --allow-dirty
```

The `--allow-dirty` flag tells clippy to apply fixes even though the working directory is not committed to Git (clippy normally refuses, to protect your work). After the command finishes, reopen `main.rs`. clippy has applied the safe fixes automatically.

> **Note:** clippy does not auto-fix every warning. It only auto-fixes the ones it can apply with high confidence that the meaning will not change. You will still see manual suggestions for the others.

### Task 5.5 -- Treat warnings as errors

Production CI pipelines should fail when new warnings are introduced. The convention is to use the `-D warnings` flag, which promotes every warning to a compile-stopping error:

```bash
cargo clippy --all-targets -- -D warnings
```

If your code is clean from Task 5.3, this command exits with status 0. To see the failure mode, introduce one warning back into `main.rs` (for example, change `total: i32 = numbers.iter().sum()` to a manual loop) and re-run. The command now fails with a non-zero exit code.

Restore the clean version before continuing.

### Checkpoints

1. clippy and `rustc` are separate programs but share a code base. Why is the project structured this way instead of simply adding all the lints to `rustc`?
2. The fix for `total = total + numbers[i]` is `total += numbers[i]`. Both produce identical machine code. Why does clippy bother to flag the longer form?
3. Identify one lint category from `cargo clippy --help` (for example, `pedantic` or `restriction`) that you would NOT enable on a normal codebase. Explain why those lints exist if they are not meant to be enabled by default.

---

## Exercise 6 -- Debug a Program with CodeLLDB

**Estimated time:** 15-20 minutes
**Topics covered:** breakpoints, stepping, variable inspection, debugging tests

### Context

A debugger is rarely the first tool you reach for in Rust. The compiler eliminates most runtime errors, and `println!` debugging covers most logic issues. But the debugger remains useful for stepping through unfamiliar code, inspecting complex data structures, and pausing execution exactly when an interesting condition occurs.

CodeLLDB integrates LLDB into VS Code with proper Rust type formatters built in. Strings appear as strings, vectors as bracketed lists, and `Option` as `Some(...)` or `None`.

### Task 6.1 -- Write a program with a logic bug

Replace `src/main.rs` with this version. There is an intentional bug in `find_max`:

```rust
fn find_max(values: &[i32]) -> i32 {
    let mut max = 0;
    for value in values {
        if *value > max {
            max = *value;
        }
    }
    max
}

fn main() {
    let numbers = vec![5, 12, 7, 3, 9];
    let result = find_max(&numbers);
    println!("Max of {numbers:?} is {result}");

    let negatives = vec![-5, -12, -7, -3, -9];
    let result = find_max(&negatives);
    println!("Max of {negatives:?} is {result}");
}
```

Run the program:

```bash
cargo run
```

Expected (buggy) output:

```
Max of [5, 12, 7, 3, 9] is 12
Max of [-5, -12, -7, -3, -9] is 0
```

The second line is wrong. The maximum of `[-5, -12, -7, -3, -9]` is `-3`, not `0`. The bug is that `max` starts at `0`, which is greater than every negative number, so it is never replaced. Without inspecting the code carefully it is not obvious where the wrong answer comes from. We will use the debugger to find it.

### Task 6.2 -- Set a breakpoint

In VS Code, open `src/main.rs`. Click in the gutter to the left of the line number for `if *value > max {`. A red dot appears, indicating a breakpoint.

### Task 6.3 -- Launch the debugger

Above `fn main()`, click the `Debug` code lens.

VS Code switches to the Debug perspective. The program runs to your breakpoint and pauses. The line is highlighted, and the debug toolbar appears at the top of the editor with controls: Continue, Step Over, Step Into, Step Out, Restart, Stop.

The left sidebar now shows:

- **Variables**: Local variables in scope. You should see `value`, `values`, and `max`.
- **Watch**: Empty. You can add expressions here.
- **Call Stack**: Shows `find_max` called from `main`.

### Task 6.4 -- Step through the first call

Hover over `value`. The popup shows `&5`. The first element of the first vector is 5.

Hover over `max`. The popup shows `0`. The accumulator is at its initial value.

Click **Continue** (F5). Execution proceeds until the breakpoint is hit again, this time with `value` referencing 12. Continue stepping through the loop. After processing all five values, the function returns 12. Correct.

### Task 6.5 -- Step through the second call

Continue past the first `println!`. The breakpoint is hit again, now inside the second call to `find_max`. The first `value` is `-5`.

Hover over `max`. Its value is `0`.

Note the comparison: `*value > max` becomes `-5 > 0`, which is false. The body of the `if` does not execute. `max` stays at 0.

Click Continue and watch the next iterations. Every value is negative, every comparison is false, `max` is never updated, and the function returns `0`.

You have just used the debugger to confirm what the bug is and why it manifests.

### Task 6.6 -- Fix the bug

Stop the debugger. Modify `find_max` to handle negative numbers correctly. The fix is to start `max` at the first element of the slice rather than at zero. One correct version:

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

This version returns `Option<i32>` because the maximum of an empty slice is undefined. The `?` operator returns `None` early if the slice is empty. We will return to `Option`, the `?` operator, and similar constructs in detail in later modules. For now, just verify that the program now produces:

```
Max of [5, 12, 7, 3, 9] is Some(12)
Max of [-5, -12, -7, -3, -9] is Some(-3)
Max of [] is None
```

### Task 6.7 -- Inspect a complex data structure

Replace `main` with the following code, which builds a more interesting data structure:

```rust
fn main() {
    let people = vec![
        ("Alice", 30),
        ("Bob", 25),
        ("Carol", 35),
    ];

    let names: Vec<&str> = people.iter().map(|(name, _)| *name).collect();
    let total_age: i32 = people.iter().map(|(_, age)| age).sum();

    println!("Names: {names:?}");
    println!("Total age: {total_age}");
}
```

Set a breakpoint on the `println!` line. Click Debug.

In the Variables panel, expand `people`. You should see three entries, each a tuple of a string slice and an integer. Expand `names` to see the three string entries. Hover over `total_age`. The value is `90`.

This visualization is provided by CodeLLDB's built-in Rust formatters. Without them, you would see raw memory layouts.

### Checkpoints

1. The original `find_max` returned `0` for an all-negative vector. The compiler did not catch this. clippy did not catch it. What category of bug is this, and what tool in the toolchain (other than the debugger) is best suited to catching it?
2. The fix changed the return type from `i32` to `Option<i32>`. Why is this a more honest signature, even though it makes the calling code slightly more complex?
3. CodeLLDB shows `vec![1, 2, 3]` as `[1, 2, 3]` in the Variables panel rather than as a raw struct with `ptr`, `len`, and `capacity` fields. Where do those formatting rules come from, and what would the display look like without them?

---

## Exercise 7 -- Pin a Toolchain to the Project

**Estimated time:** 5-10 minutes
**Topics covered:** `rust-toolchain.toml`, channel selection, reproducible builds

### Context

Toolchain pinning is the most important feature for team Rust workflows. By committing a `rust-toolchain.toml` file to the project, every developer (and every CI run) automatically uses the exact same compiler version, regardless of what their default toolchain is. New hires can clone the repository and have the correct toolchain installed automatically on their first build.

### Task 7.1 -- Create rust-toolchain.toml

In the project root, create a file named `rust-toolchain.toml` with these contents:

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy", "rust-analyzer"]
profile = "default"
```

Save the file.

### Task 7.2 -- Verify the override is active

Run:

```bash
rustup show active-toolchain
```

The output should be similar to:

```
stable-x86_64-unknown-linux-gnu (overridden by '/path/to/hello_rustacean/rust-toolchain.toml')
```

The "overridden by" annotation confirms that the project's pinned toolchain is in effect.

### Task 7.3 -- Pin to a specific version

Modify `rust-toolchain.toml` to pin to a specific version instead of the rolling `stable` channel:

```toml
[toolchain]
channel = "1.84.0"
components = ["rustfmt", "clippy", "rust-analyzer"]
profile = "default"
```

Substitute the current stable version number for `1.84.0` if it is older than your installed version. Run:

```bash
rustc --version
```

The output should match the pinned version. If the version is not installed, rustup will download it on first use.

### Task 7.4 -- Test a one-off override

Without changing the file, run:

```bash
cargo +nightly --version
```

This invokes Cargo from the nightly toolchain just for this one command. If you do not have nightly installed, rustup will report an error. Install it with `rustup toolchain install nightly` if you want to test this further. The point is that the `+toolchain` syntax always wins over `rust-toolchain.toml`.

### Checkpoints

1. A new hire clones a Rust project and runs `cargo build`. The first thing they see is rustup downloading a toolchain. Why is this preferable to the build simply using whatever Rust they had installed?
2. The project pins to `1.84.0` exactly. A teammate proposes changing it to `1` (just the major version). What is the trade-off?
3. The `components` list includes `rust-analyzer`. Why does it matter to ship rust-analyzer in the pinned toolchain, given that your editor likely manages its own copy?

---

## Exercise 8 -- Read Compiler Output in the Rust Playground

**Estimated time:** 5-10 minutes
**Topics covered:** Rust Playground, ASM output, debug vs release optimization

### Context

The Rust Playground (https://play.rust-lang.org) is a browser-based environment for Rust experimentation. It is useful for sharing snippets, trying ideas without setting up a project, and inspecting compiler output. Showing students that high-level Rust code compiles to tight machine code is one of the most concrete demonstrations of the zero-cost abstractions principle.

### Task 8.1 -- Run a program in the Playground

Open https://play.rust-lang.org in a browser. Paste the following code into the editor:

```rust
use std::hint::black_box;

fn count_alpha(s: &str) -> usize {
    s.chars().filter(|c| c.is_alphabetic()).count()
}

fn main() {
    let input = black_box("Hello, Rustaceans!");
    let count = count_alpha(input);
    println!("{count}");
}
```

The `black_box` call prevents the compiler from optimizing away the function call by computing the result at compile time. Without it, the optimizer would replace the entire function with the literal `17`, which is the correct answer but defeats the purpose of inspecting the assembly.

Click **Run**. The output is `17`.

### Task 8.2 -- View the assembly in Debug mode

In the toolbar at the top of the page:

1. Confirm the build mode dropdown is set to **Debug**.
2. Click the dropdown arrow next to the **Run** button. A menu appears with options like Run, Build, ASM, LLVM IR, MIR, HIR, WASM.
3. Click **ASM**.

The right pane now shows the x86_64 assembly produced by `rustc`. Scroll until you find a label resembling `playground::count_alpha:`. The function occupies many lines of assembly. You will see a loop that dispatches through `core::str::Chars::next` and calls into `is_alphabetic`. There is no inlining and no specialization.

### Task 8.3 -- View the assembly in Release mode

Change the build mode dropdown from **Debug** to **Release**. Click ASM again.

The output is dramatically shorter. The `count_alpha` function now consists of a tight loop that walks the bytes of the string, decoding UTF-8 inline, checking each character against the alphabetic ranges, and incrementing a counter. There is no separate filter object, no separate counter object, no virtual dispatch. The iterator chain has been fused into a single loop.

This is the zero-cost abstraction principle made visible: the high-level code expressed with iterators compiles to assembly indistinguishable from a hand-rolled byte loop.

### Task 8.4 -- Share the snippet

Click the **Share** button in the Playground toolbar. The dialog produces a permanent URL that captures the exact code, build mode, and viewing options. Copy the URL into `lab1-notes.md`. URLs of this form are commonly used in Rust forums and bug reports to share reproducible examples.

### Checkpoints

1. The Debug build's assembly was much longer than the Release build's. Identify two specific differences you observed in the output and explain what each one means about how the compiler treated the iterator chain.
2. `black_box` is from `std::hint`. The standard library documentation describes it as "an identity function that hints to the compiler to be maximally pessimistic about what black_box could do." Why was it necessary in this exercise but would not be necessary in production code?
3. The Playground is convenient for quick experiments but has no persistent file system, no `Cargo.toml`, and no support for projects with multiple files. Identify two situations where the Playground is the right tool, and two where you should switch to a local Cargo project.

---

## Summary and Reflection

After completing this lab you have set up a working Rust development environment, used every part of the toolchain on a real project, and produced working code that has been formatted, linted, debugged, and inspected at the assembly level.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Inspect Installation | rustup, components, toolchain layout | The toolchain is per-user and self-contained. `rustup show` is your primary diagnostic. |
| 2 -- Cargo Project | `cargo new`, build, run, profiles | Cargo enforces a single project layout. Debug and release profiles are completely separate builds. |
| 3 -- VS Code | rust-analyzer, inlay hints, code lens | The language server provides type information and diagnostics without running the compiler. |
| 4 -- Formatting | `cargo fmt`, format-on-save | One canonical style eliminates a class of code review discussions. |
| 5 -- Linting | `cargo clippy`, idiomatic Rust | clippy teaches the conventions that make code look like Rust rather than translated C. |
| 6 -- Debugging | CodeLLDB, breakpoints, variable inspection | Most Rust bugs are caught at compile time, but the debugger remains essential for logic errors. |
| 7 -- Toolchain Pinning | `rust-toolchain.toml` | Reproducible builds across a team start with one pinned toolchain file. |
| 8 -- Playground | ASM output, zero-cost abstractions | High-level Rust compiles to the same code a careful C programmer would write by hand. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab1-notes.md` before your next session.

1. Of the seven tools you used in this lab (rustup, cargo, rustc, rustfmt, clippy, rust-analyzer, CodeLLDB), three are bundled with the official toolchain by default and four are not bundled or are optional. Identify which is which, and explain why some tools are "official" and others are recommended but separate.

2. The lab walked you through fixing a bug in `find_max` that the compiler did not catch. Identify three other categories of bug that the compiler does NOT catch, and for each one, name the tool in the Rust ecosystem that addresses it.

3. You changed `rust-analyzer.check.command` from its default of `"check"` to `"clippy"`. This trades faster save-time feedback for stricter warnings. In what circumstances might you reverse this decision? What setting would you choose for a very large codebase where clippy takes 30+ seconds to run?

---

*End of Lab 1*
