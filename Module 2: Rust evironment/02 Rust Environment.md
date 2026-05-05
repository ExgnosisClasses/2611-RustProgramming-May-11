# Module 2: Setting Up the Rust Environment

## Module Overview

This module establishes a working Rust development environment. Students will install the toolchain, understand how it is structured, configure an editor with full language support, and set up debugging. By the end of the module, students will:

- Install Rust correctly on Windows, macOS, and Linux using `rustup`.
- Manage multiple toolchains and target platforms.
- Understand the role of stable, beta, and nightly release channels.
- Use the essential auxiliary tools (`rustfmt`, `clippy`, `rust-analyzer`).
- Configure VS Code as a productive Rust IDE.
- Step through Rust code with a debugger.

The goal is more than a working installation. Students should understand why the toolchain is structured the way it is, so they can diagnose problems and onboard teammates later.

---

## A. Installing via rustup (Cross-Platform)

### What is rustup?

`rustup` is the official installer and version manager for Rust. It is comparable to `nvm` for Node.js, `pyenv` for Python, or `rbenv` for Ruby. The principle is the same: rather than installing a single fixed compiler version system-wide, you install a tool that manages compiler installations on your behalf.

### Why a Version Manager?

A common question from developers used to system package managers (`apt`, `brew`, `yum`) is why Rust insists on `rustup` rather than the OS package. The reasons are practical:

1. **Multiple toolchains**: A single project may require stable for production builds, nightly for some experimental feature, and a specific older version for compatibility testing.
2. **Multiple targets**: Cross-compiling to embedded ARM, WebAssembly, or Windows from Linux requires per-target toolchain components. `rustup` handles this uniformly.
3. **Frequent releases**: Rust ships a stable release every six weeks. OS package managers lag behind, often significantly.
4. **Component management**: `rustfmt`, `clippy`, and `rust-analyzer` are versioned with the compiler. `rustup` keeps them in sync.
5. **No root required**: `rustup` installs into the user's home directory, not system paths. This matters for shared machines and CI.

### Installation by Platform

#### Linux and macOS

The official installation command:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

This downloads and runs an installer script. Run it as a regular user, not root. The defaults are correct for most users.

After installation, either restart your shell or source the environment file:

```bash
source "$HOME/.cargo/env"
```

#### Windows

Two options:

1. **rustup-init.exe**: Download from https://rustup.rs and run the installer.
2. **winget**: `winget install Rustlang.Rustup`

Windows additionally requires the **Microsoft C++ Build Tools** for the linker. The installer will prompt you to install Visual Studio Build Tools if they are missing. Select the "Desktop development with C++" workload.

#### Verification

After installation, verify everything is in place:

```bash
rustc --version
cargo --version
rustup --version
```

Expected output:

```
rustc 1.84.0 (or newer)
cargo 1.84.0 (or newer)
rustup 1.27.1 (or newer)
```

If any command is not found, the installer's PATH modifications have not taken effect. Open a new terminal or restart the shell.

### What rustup Installs

By default, `rustup` installs:

| Component       | Purpose                                          |
|-----------------|--------------------------------------------------|
| `rustc`         | The Rust compiler.                               |
| `cargo`         | Build system and package manager.                |
| `rust-std`      | The standard library, precompiled.               |
| `rust-docs`     | Local copy of the standard library documentation.|
| `rustup`        | The version manager itself (self-managing).      |

### Directory Layout

After installation:

```
~/.rustup/                 Toolchains, targets, components.
~/.cargo/                  Cargo's home: registry cache, installed binaries.
~/.cargo/bin/              Executables on your PATH (rustc, cargo, etc.).
~/.cargo/config.toml       User-wide Cargo configuration.
```

Knowing this layout helps with troubleshooting and CI configuration.

---

## B. Managing Toolchains with rustup

### What is a Toolchain?

A toolchain is a complete Rust installation: compiler, standard library, and components, all built together for a specific target. A typical developer machine has one or two toolchains. A library author working across editions or platforms may have many.

### Listing Installed Toolchains

```bash
rustup show
```

Sample output:

```
Default host: x86_64-unknown-linux-gnu
rustup home:  /home/dev/.rustup

installed toolchains
--------------------
stable-x86_64-unknown-linux-gnu (active, default)
nightly-x86_64-unknown-linux-gnu

active toolchain
----------------
stable-x86_64-unknown-linux-gnu (default)
rustc 1.84.0 (9fc6b4312 2025-01-07)
```

### Installing Additional Toolchains

```bash
rustup toolchain install stable
rustup toolchain install beta
rustup toolchain install nightly
rustup toolchain install 1.75.0      # specific version
```

### Switching the Default Toolchain

```bash
rustup default stable
rustup default nightly
```

This changes which toolchain runs when you type `rustc` or `cargo` without an override.

### Per-Project Toolchains

Most projects pin a toolchain by checking a `rust-toolchain.toml` file into the repository:

```toml
[toolchain]
channel = "1.84.0"
components = ["rustfmt", "clippy"]
targets = ["wasm32-unknown-unknown"]
profile = "minimal"
```

When `cargo` runs in this directory, `rustup` automatically uses the specified toolchain, downloading it on first use. This is a critical feature for team consistency: a new hire's first build will pull the exact toolchain everyone else uses.

### One-Off Overrides

To run a single command with a different toolchain:

```bash
cargo +nightly build
rustc +1.75.0 main.rs
```

The `+toolchain` syntax works with all `cargo` and `rustc` invocations.

### Targets and Cross-Compilation

A target is the platform you are compiling for. By default, you compile for the host platform. To cross-compile, install the corresponding standard library:

```bash
rustup target add wasm32-unknown-unknown        # WebAssembly
rustup target add x86_64-pc-windows-gnu         # Windows from Linux
rustup target add aarch64-unknown-linux-gnu     # ARM64 Linux
rustup target add thumbv7em-none-eabihf         # Cortex-M4 microcontroller
```

Then build with:

```bash
cargo build --target wasm32-unknown-unknown --release
```

Note that for many targets, you also need a cross-linker installed separately. `rustup` only manages the Rust standard library for the target.

### Updating

```bash
rustup update                # update all toolchains
rustup update stable         # update only stable
rustup self update           # update rustup itself
```

It is good practice to run `rustup update` weekly or when starting a new project.

### Uninstalling Components

```bash
rustup toolchain uninstall nightly
rustup target remove wasm32-unknown-unknown
rustup self uninstall        # complete uninstall
```

A complete uninstall removes `~/.rustup` and `~/.cargo` entirely. There is no other state on the system.

---

## C. Stable vs. Beta vs. Nightly Channels

### The Train Model

Rust uses a "release train" model adapted from Mozilla's Firefox process. Three channels move forward together:

```
nightly  -->  beta  -->  stable
   |            |          |
new commits   freeze     6-week release
```

Every six weeks:

1. The current `beta` is promoted to `stable`.
2. The current `nightly` is forked to become the new `beta`.
3. `nightly` continues with new commits.

This means a feature merged into `nightly` today reaches `stable` users in roughly 12 weeks, after going through `beta`.

### The Stability Promise

This is one of the most important properties of Rust as a production language:

> Code that compiles on stable Rust will continue to compile on future stable Rust versions, indefinitely.

This promise applies to the language itself and to the standard library. Breaking changes are absorbed through:

- **Editions** (covered later) for opt-in syntax changes.
- **Soundness fixes** which are documented and given long deprecation windows.

For industry, this means a Rust codebase from 2017 still builds today with minimal effort.

### When to Use Each Channel

#### Stable

The default. Use this for:

- Production deployments.
- Library crates intended for public release.
- Anything you expect another team to build.

Stable is what you get from `rustup default stable`.

#### Beta

A release candidate for the next stable. Use it for:

- Verifying your code works on the upcoming release.
- CI pipelines that test against future Rust to catch regressions early.

Beta is rarely used as a daily driver.

#### Nightly

Built every night from the main branch. Use it for:

- Experimenting with unstable features behind feature gates.
- Specialized tooling (`cargo-fuzz`, `miri`, `cargo-asm` for some output formats).
- Working on the compiler or core libraries themselves.

#### Feature Gates

Unstable features must be opted into in the source code. Example:

```rust
#![feature(let_chains)]

fn main() {
    let opt = Some(42);
    if let Some(x) = opt && x > 10 {
        println!("Large value: {x}");
    }
}
```

Compile this with:

```bash
cargo +nightly build
```

Once a feature stabilizes, the `#![feature(...)]` attribute becomes unnecessary and is rejected on stable.

### Channel Selection Strategy

For most teams, the right answer is:

- **Default to stable** for everything.
- **Pin a specific version** in `rust-toolchain.toml` for reproducible builds.
- **Run CI against beta** as an early warning system.
- **Use nightly only when necessary** and isolate the dependency.

Avoid building production systems on nightly unless you have a specific reason and can absorb the maintenance cost.

### Edition vs. Channel

These are often confused:

| Concept    | Frequency         | What it Affects                                    |
|------------|-------------------|----------------------------------------------------|
| Channel    | Every 6 weeks     | Which compiler version you use.                    |
| Edition    | Every 3 years     | The language dialect (syntax, defaults, idioms).   |

Editions are 2015, 2018, 2021, 2024. They are declared in `Cargo.toml`:

```toml
[package]
edition = "2024"
```

A modern compiler can build any edition. You choose your edition once per crate, independently of the compiler version. Editions are covered in detail in later modules.

---

## D. Essential Components: rustfmt, clippy, rust-analyzer

These three tools, plus the compiler, define the daily Rust development experience. All are official, all are versioned with the compiler, and all are installed through `rustup`.

### rustfmt

`rustfmt` is the standard code formatter. It is to Rust what `gofmt` is to Go.

#### Why a Standard Formatter?

The Rust community's position is that formatting debates are not worth having. A single canonical style means:

- No formatting discussions in code review.
- All Rust code looks the same, regardless of author.
- Editors can format on save without surprises.

#### Installation

`rustfmt` ships with stable toolchains by default. To verify:

```bash
rustup component list --installed
```

If missing:

```bash
rustup component add rustfmt
```

#### Usage

```bash
cargo fmt                    # format all files in the current crate
cargo fmt -- --check         # check without modifying (used in CI)
cargo fmt --all              # format all crates in a workspace
```

#### Configuration

Configuration goes in `rustfmt.toml` at the project root. Most projects need no configuration. Common overrides:

```toml
edition = "2024"
max_width = 100
hard_tabs = false
tab_spaces = 4
```

The community convention is to use defaults unless there is a strong reason not to.

### clippy

`clippy` is the official linter. It catches:

- Common mistakes and anti-patterns.
- Inefficient code.
- Non-idiomatic code that has a more conventional expression.
- Suspicious patterns that often indicate bugs.

It currently includes over 700 lints, organized into categories: correctness, suspicious, style, complexity, performance, pedantic, restriction, and nursery.

#### Installation

Bundled with stable toolchains. To verify or add:

```bash
rustup component add clippy
```

#### Usage

```bash
cargo clippy                              # run lints
cargo clippy --all-targets                # include tests, examples, benchmarks
cargo clippy -- -D warnings               # treat warnings as errors (CI)
cargo clippy --fix                        # auto-apply suggestions where safe
```

#### Example

Given this code:

```rust
fn main() {
    let v = vec![1, 2, 3];
    let mut sum = 0;
    for i in 0..v.len() {
        sum = sum + v[i];
    }
    println!("{sum}");
}
```

Clippy will produce several suggestions:

```
warning: the loop variable `i` is only used to index `v`
  = help: consider using `for x in &v { sum = sum + x; }`

warning: manual implementation of an assign operation
  = help: use `sum += v[i]` instead

warning: this loop could be replaced by a sum
  = help: use `v.iter().sum::<i32>()` instead
```

The idiomatic Rust version:

```rust
fn main() {
    let v = vec![1, 2, 3];
    let sum: i32 = v.iter().sum();
    println!("{sum}");
}
```

#### Allowing and Denying Lints

Lints can be controlled at the crate, module, or item level:

```rust
#![warn(clippy::all)]
#![deny(clippy::correctness)]
#![allow(clippy::module_name_repetitions)]

#[allow(clippy::needless_return)]
fn explicit() -> i32 {
    return 42;
}
```

In CI, the convention is to deny warnings to keep the codebase clean:

```bash
cargo clippy --all-targets --all-features -- -D warnings
```

### rust-analyzer

`rust-analyzer` is the official Language Server for Rust. It implements the Language Server Protocol (LSP), so it works with any editor that supports LSP: VS Code, Neovim, Helix, Emacs, JetBrains IDEs (with a plugin), and others.

#### What It Provides

- Inline error and warning diagnostics.
- Autocomplete with full type information.
- Go to definition, find references.
- Hover documentation showing types and signatures.
- Rename refactoring across the entire crate.
- Inlay hints (showing inferred types and parameter names).
- Code actions (quick fixes for diagnostics).
- Symbol search across the project.

#### Installation

Two options:

1. **Via rustup** (recommended for most users):
   ```bash
   rustup component add rust-analyzer
   ```

2. **Via your editor**: VS Code, for example, downloads its own copy through the rust-analyzer extension. This is the simplest path on a fresh install.

The two paths are independent. If both are present, your editor will pick one. The rustup-managed copy is updated with `rustup update`. The editor-managed copy updates with the extension.

#### Architecture Note

`rust-analyzer` does not invoke `rustc`. It maintains its own incremental analysis engine that re-uses results across keystrokes. This is why it can give responses in milliseconds as you type. It uses the same type inference algorithm as the compiler but operates on partial, in-progress code.

This matters in practice: if `rust-analyzer` reports an error that `cargo build` does not, or vice versa, the analyzer can occasionally be out of sync. The fix is usually to restart the language server (a single command in your editor).

### Tool Summary

| Tool             | Purpose            | Trigger                          |
|------------------|--------------------|----------------------------------|
| `rustc`          | Compile.           | `cargo build`, `cargo run`.      |
| `rustfmt`        | Format.            | `cargo fmt`, format on save.     |
| `clippy`         | Lint.              | `cargo clippy`, on-save in editor.|
| `rust-analyzer`  | Editor intelligence.| Runs as a background process.   |

---

## E. VS Code with rust-analyzer

Visual Studio Code is the most widely used editor in the Rust community. It is free, cross-platform, and the `rust-analyzer` extension provides a first-class experience.

### Installation

1. Install VS Code from https://code.visualstudio.com.
2. Open the Extensions panel (Ctrl+Shift+X or Cmd+Shift+X).
3. Search for "rust-analyzer" (publisher: rust-lang).
4. Click Install.

The extension will download a matching `rust-analyzer` server binary on first launch. No additional configuration is required to get started.

### First-Run Verification

Create a new Cargo project and open it in VS Code:

```bash
cargo new hello_rust
cd hello_rust
code .
```

Open `src/main.rs`. Within a few seconds you should see:

- Inlay hints showing inferred types.
- Hover popups with type information.
- A "Run" and "Debug" code lens above `fn main()`.

If none of this appears, open the Output panel (View > Output) and select "rust-analyzer" from the dropdown to see initialization logs.

### Recommended Settings

Open your User Settings (JSON) via the command palette: `Preferences: Open User Settings (JSON)`. Add:

```json
{
  "editor.formatOnSave": true,
  "[rust]": {
    "editor.defaultFormatter": "rust-lang.rust-analyzer",
    "editor.formatOnSave": true
  },
  "rust-analyzer.check.command": "clippy",
  "rust-analyzer.cargo.features": "all",
  "rust-analyzer.inlayHints.parameterHints.enable": true,
  "rust-analyzer.inlayHints.typeHints.enable": true,
  "rust-analyzer.inlayHints.chainingHints.enable": true,
  "rust-analyzer.completion.callable.snippets": "fill_arguments"
}
```

Notable choices:

- `rust-analyzer.check.command: clippy` runs `cargo clippy` instead of `cargo check` for save-time diagnostics. You get clippy lints in the editor as you write code.
- `editor.formatOnSave` runs `rustfmt` on every save.
- Inlay hints are valuable for new Rust developers learning the type system.

### Useful Commands

Open the Command Palette (Ctrl+Shift+P or Cmd+Shift+P) and type "Rust Analyzer" to see all commands. The most useful:

| Command                                | Use Case                                        |
|----------------------------------------|-------------------------------------------------|
| Rust Analyzer: Restart server          | Fix stale state or sync issues.                 |
| Rust Analyzer: Reload Workspace        | After editing `Cargo.toml`.                     |
| Rust Analyzer: Expand macro recursively| See what `println!` or `derive` produces.       |
| Rust Analyzer: View Hir                | Inspect the high-level IR of an expression.     |
| Rust Analyzer: Run                     | Run the test or binary under the cursor.        |

---

## F. Useful VS Code Extensions for Rust

The `rust-analyzer` extension is the only required one. The following are commonly used additions.

### Essential Companion Extensions

#### CodeLLDB

- Publisher: vadimcn
- Provides a debugger for Rust (and C, C++, Swift) using LLDB.
- Required for the debugging workflow in Section G.

#### Even Better TOML

- Publisher: tamasfe
- Syntax highlighting, validation, and schema-aware autocomplete for TOML files.
- Especially useful for `Cargo.toml`, where it knows the schema.

#### crates

- Publisher: serayuzgur
- Inline display of latest crate versions in `Cargo.toml`.
- Highlights outdated dependencies and offers one-click upgrades.

### Productivity Extensions

#### Error Lens

- Publisher: usernamehw
- Displays diagnostics inline next to the offending line, instead of only in the Problems panel.
- Significantly reduces the time between making a mistake and seeing it.

#### GitLens

- Publisher: GitKraken
- Shows git blame inline, history navigation, and PR integration.
- Not Rust-specific but heavily used.

#### Todo Tree

- Publisher: Gruntfuggly
- Aggregates `TODO`, `FIXME`, and similar markers in a sidebar.
- Useful in larger codebases.

### Specialized Extensions

#### Dependi

- Publisher: fill-labs
- Modern alternative to the `crates` extension.
- Supports multiple ecosystems including Cargo.

#### CodeSnap

- Publisher: adpyke
- Generates clean code screenshots for documentation and presentations.

#### Coverage Gutters

- Publisher: ryanluker
- Displays line coverage in the gutter.
- Pairs with `cargo-tarpaulin` (covered in the testing module).

### Avoid These Pitfalls

A few extensions are listed in older tutorials but should be avoided:

- **The original "Rust" extension** (publisher: rust-lang). This is the deprecated predecessor to `rust-analyzer`. It is no longer maintained. Uninstall it if present.
- **Multiple Rust language servers**. If you have both the legacy "Rust" extension and "rust-analyzer" enabled, they will conflict.

---

## G. Debugging with CodeLLDB in VS Code

A debugger is one of the practical reasons to use VS Code over a plain text editor for Rust work. CodeLLDB provides a graphical interface to LLDB with full Rust support.

### Why a Debugger?

Print debugging (`println!`, `dbg!`) covers most cases in Rust. The borrow checker eliminates most runtime errors, leaving fewer mysterious crashes than in C++. However, a debugger remains useful for:

- Stepping through complex iterator chains or async state machines.
- Inspecting the contents of large data structures.
- Diagnosing logic errors in unfamiliar code.
- Setting conditional breakpoints.

### Installation

1. Install the **CodeLLDB** extension in VS Code.
2. On Windows, no additional setup is required. CodeLLDB bundles its own LLDB.
3. On Linux and macOS, the bundled LLDB also works without a separate installation.

### Building with Debug Symbols

By default, `cargo build` produces debug builds with full symbol information. This is the right binary for debugging. Do not use `--release` for an interactive debug session unless you specifically need to observe optimized behavior, because the optimizer will reorder and inline code in ways that make stepping confusing.

```bash
cargo build               # debug build, with symbols
cargo build --release     # optimized, harder to debug
```

### The Easy Path: Code Lens

For binary projects, the simplest workflow uses the code lens that `rust-analyzer` adds above `fn main()`:

```
Run | Debug
fn main() {
    ...
}
```

Click **Debug**. CodeLLDB launches the binary, hits any breakpoints you have set, and opens the debug toolbar.

The same lens appears above test functions, allowing you to debug a single test in isolation.

### Setting Breakpoints

Click the gutter to the left of any line number, or place the cursor on a line and press F9.

### The Debug View

When debugging, the left sidebar shows:

- **Variables**: Local variables in the current frame, with expand-to-inspect support.
- **Watch**: Expressions you want continuously evaluated.
- **Call Stack**: The function call chain that led to the current location.
- **Breakpoints**: All breakpoints in the project.

The bottom panel includes the **Debug Console**, where you can evaluate expressions in the current frame:

```
> v.len()
3
> v.iter().map(|x| x * 2).collect::<Vec<i32>>()
[2, 4, 6]
```

### Configuring launch.json for More Control

For projects with command-line arguments, environment variables, or non-default binaries, configure `launch.json`:

1. Open the Run and Debug panel (Ctrl+Shift+D or Cmd+Shift+D).
2. Click "create a launch.json file".
3. Choose "LLDB" when prompted.

A typical configuration:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "Debug my_app",
      "cargo": {
        "args": ["build", "--bin=my_app", "--package=my_app"],
        "filter": { "name": "my_app", "kind": "bin" }
      },
      "args": ["--input", "data.txt", "--verbose"],
      "env": { "RUST_LOG": "debug" },
      "cwd": "${workspaceFolder}"
    },
    {
      "type": "lldb",
      "request": "launch",
      "name": "Debug unit tests",
      "cargo": {
        "args": ["test", "--no-run", "--lib", "--package=my_app"],
        "filter": { "name": "my_app", "kind": "lib" }
      },
      "args": [],
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

The `cargo` block tells CodeLLDB to invoke Cargo and find the resulting binary. The `args` block contains arguments passed to your program (not Cargo).

### Pretty Printing

Rust types like `Vec`, `String`, `HashMap`, and `Option` are not displayed in their natural form by default. CodeLLDB handles this automatically through built-in formatters, so you should see:

```
v: Vec<i32, alloc::alloc::Global> = [1, 2, 3]
s: String = "hello"
opt: Option<i32> = Some(42)
```

If you see raw memory layouts instead, ensure the `lldb.launch.expressions` setting is `"native"` (the default).

### Worked Example

Given this program, save as `src/main.rs`:

```rust
fn fibonacci(n: u32) -> u64 {
    if n <= 1 {
        return n as u64;
    }
    fibonacci(n - 1) + fibonacci(n - 2)
}

fn main() {
    let inputs = vec![5, 10, 15];
    for n in &inputs {
        let result = fibonacci(*n);
        println!("fib({n}) = {result}");
    }
}
```

Steps:

1. Set a breakpoint on the `let result = fibonacci(*n)` line.
2. Click **Debug** above `main`.
3. When execution stops, inspect `n` and `inputs` in the Variables panel.
4. Click "Step Into" (F11) to enter the `fibonacci` function.
5. Watch the call stack grow as the recursion descends.
6. Click "Step Out" (Shift+F11) to return to the loop.

### Debugging Tests

To debug a specific test:

```rust
#[test]
fn it_doubles() {
    let v = vec![1, 2, 3];
    let doubled: Vec<i32> = v.iter().map(|x| x * 2).collect();
    assert_eq!(doubled, vec![2, 4, 6]);
}
```

Click "Debug" in the code lens above the test function. The test runs in isolation, breakpoints are honored, and panics open in the debugger so you can inspect the state at the moment of failure.

### Common Issues

| Problem                                     | Likely Cause                                         |
|---------------------------------------------|------------------------------------------------------|
| Breakpoints show as hollow circles          | Code is optimized away (release build) or unreached. |
| Variables show as "<not available>"         | Optimized out. Switch to debug build.                |
| Pretty printing shows raw layout            | Older CodeLLDB. Update the extension.                |
| Cannot find binary                          | Run `cargo build` first, or fix `launch.json`.       |
| Linker errors only when debugging on macOS  | Allow LLDB in System Preferences > Security.         |

---

## Module Summary

- `rustup` is the version manager for the Rust toolchain. It handles installation, updates, multiple toolchains, and cross-compilation targets.
- Rust ships on three channels: stable (production), beta (release candidate), and nightly (experimental). Most production code uses stable.
- The toolchain includes four key components: the compiler `rustc`, the formatter `rustfmt`, the linter `clippy`, and the language server `rust-analyzer`.
- VS Code with the `rust-analyzer` extension is the most common Rust development environment. A small set of additional extensions improves the experience further.
- CodeLLDB provides graphical debugging with proper Rust type display, integrated with the `rust-analyzer` code lens for one-click debugging.

