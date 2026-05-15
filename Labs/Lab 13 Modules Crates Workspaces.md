# Module 17: Modules, Crates, and Workspaces
## Lab 13 -- Organizing Code with Modules, Crates, Workspaces, and Features

> **Course:** Mastering Rust
> **Module:** 17 - Modules, Crates, and Workspaces
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 17: organizing code with `mod` and `pub`, controlling visibility with `pub(crate)` and `pub(super)`, splitting code across files, the difference between binary and library crates, the structure of `Cargo.toml`, semantic versioning, building multi-crate workspaces, preparing a crate for publication, and conditional compilation with feature flags.

You will build a single multi-crate workspace called `taskforge` that organizes a small task management system. The workspace starts with a single binary crate, grows to include a library crate, splits the library across modules and files, and adds feature flags to enable optional functionality. Each exercise builds on the previous one. The progression mirrors what happens in real Rust projects as they grow from prototypes to organized codebases.

The focus is on judgment: when to split code into a new module, when to extract a module into a separate file, when a library crate makes sense versus keeping everything in one binary, when to create a workspace versus a single crate, when feature flags are the right tool. Many decisions have multiple defensible answers; the lab guides you toward idiomatic Rust and asks you to compare alternatives.

By the end of this lab you will be able to:

- Organize code into modules with `mod`, `pub`, and `use`.
- Use the path syntax (`crate::`, `super::`, `self::`) to refer to items.
- Apply visibility modifiers (`pub`, `pub(crate)`, `pub(super)`).
- Split a module across multiple files.
- Distinguish between binary and library crates and choose between them.
- Read and write `Cargo.toml` files with dependencies, features, and profiles.
- Understand semantic versioning and how version requirements are specified.
- Set up a Cargo workspace with multiple member crates.
- Configure a crate for publication to crates.io.
- Use feature flags for conditional compilation.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** This lab uses only the standard library and a few common crates (`serde` for one exercise as an illustration). All dependencies will be added through `Cargo.toml` as needed. Make sure you have an internet connection when you reach exercises that add dependencies.

> **A note about the structure.** Unlike previous labs that build inside a single Cargo project, this lab reorganizes the project several times as it grows. You will run `cargo new` more than once, restructure files, and convert single crates into workspaces. The reorganization is part of the lesson; experienced Rust developers do this routinely as projects grow.

---

## Lab Project Setup

Create a working directory for the lab:

```bash
mkdir -p ~/rust-course/labs/taskforge
cd ~/rust-course/labs/taskforge
```

You will create the actual Cargo project inside this directory in Exercise 1.

Create a `lab13-notes.md` file in the lab directory for observations and checkpoint answers.

---

## Exercise 1 -- Modules in a Single File

**Estimated time:** 10-15 minutes
**Topics covered:** `mod`, `pub`, `use`, paths (`crate::`, `super::`, `self::`)

### Context

A module is a named scope that contains items: functions, structs, enums, constants, other modules. Modules organize code, control visibility, and create namespaces. This exercise establishes the basic mechanics inside a single file.

### Task 1.1 -- Create the initial project

From the `taskforge` directory:

```bash
cargo init --bin
```

This creates a binary crate in the current directory. The `--bin` flag is the default for `cargo init` (versus `--lib`), but specifying it is clearer.

Verify the structure:

```
taskforge/
├── Cargo.toml
└── src/
    └── main.rs
```

### Task 1.2 -- A single-file module

Replace the contents of `src/main.rs` with:

```rust
// A module defined inline:
mod tasks {
    pub struct Task {
        pub title: String,
        pub completed: bool,
    }

    pub fn new_task(title: &str) -> Task {
        Task {
            title: title.to_string(),
            completed: false,
        }
    }

    pub fn complete(task: &mut Task) {
        task.completed = true;
    }
}

fn main() {
    let mut task = tasks::new_task("Write the lab");
    println!("created: {}, completed: {}", task.title, task.completed);

    tasks::complete(&mut task);
    println!("after complete: {}, completed: {}", task.title, task.completed);
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
created: Write the lab, completed: false
after complete: Write the lab, completed: true
```

Three things to notice:

- The `mod tasks { ... }` block defines a module named `tasks`. Everything inside the braces is part of the module.
- Items inside the module are private by default. The `pub` keyword on `Task`, `new_task`, and `complete` makes them visible outside the module.
- From outside the module, items are accessed via the path: `tasks::new_task(...)`, `tasks::complete(...)`.

The struct's fields also need `pub` to be accessible from outside. The `pub` on `Task` makes the type itself public; without `pub` on each field, the fields would still be private. Try removing `pub` from `pub completed: bool` and run again:

```
error[E0616]: field `completed` of struct `tasks::Task` is private
```

The error tells you exactly what is wrong. Restore the `pub`.

### Task 1.3 -- Visibility levels

Rust has more visibility options than just public/private:

```rust
mod tasks {
    pub struct Task {
        pub title: String,
        pub completed: bool,
        pub(crate) internal_id: u32,
        priority: u8,
    }

    pub fn new_task(title: &str) -> Task {
        Task {
            title: title.to_string(),
            completed: false,
            internal_id: 42,
            priority: 1,
        }
    }

    pub fn complete(task: &mut Task) {
        task.completed = true;
    }

    pub(super) fn debug_print(task: &Task) {
        println!("priority={}", task.priority);
    }
}

fn main() {
    let task = tasks::new_task("Write the lab");

    println!("title: {}", task.title);                    // OK: pub
    println!("internal: {}", task.internal_id);           // OK: pub(crate)
    // println!("priority: {}", task.priority);           // ERROR: private
    
    tasks::debug_print(&task);                            // OK: pub(super)
}
```

Run the program. Expected output:

```
title: Write the lab
internal: 42
priority=1
```

The four visibility levels demonstrated:

- **`pub`**: visible everywhere. Used for items meant for external consumers.
- **`pub(crate)`**: visible anywhere in the same crate. Used for internal-but-cross-module APIs.
- **`pub(super)`**: visible to the parent module (one level up). Used for helpers shared with the parent.
- **No modifier (private)**: visible only within the module where defined.

Try uncommenting the `priority` line:

```
error[E0616]: field `priority` of struct `tasks::Task` is private
```

The compiler enforces the visibility rules at compile time.

The choice of visibility level encodes intent. `pub(crate)` says "this is for internal use; do not depend on it from outside." `pub` says "this is part of the public API; treat it as a commitment."

### Task 1.4 -- The `use` keyword

Long paths get tedious. The `use` keyword brings items into scope:

```rust
mod tasks {
    pub struct Task {
        pub title: String,
        pub completed: bool,
    }

    pub fn new_task(title: &str) -> Task {
        Task {
            title: title.to_string(),
            completed: false,
        }
    }

    pub fn complete(task: &mut Task) {
        task.completed = true;
    }
}

use tasks::{Task, new_task, complete};

fn main() {
    let mut task: Task = new_task("Use shortcut");
    println!("before: {}, {}", task.title, task.completed);

    complete(&mut task);
    println!("after: {}, {}", task.title, task.completed);
}
```

Run the program. Expected output:

```
before: Use shortcut, false
after: Use shortcut, true
```

The `use tasks::{Task, new_task, complete};` line imports three items into the current scope. Now `Task`, `new_task`, and `complete` can be used directly without the `tasks::` prefix.

`use` works the same way as `import` in other languages: it brings names into scope for convenience. The underlying paths still exist; `use` is just shorthand.

Common patterns:

- `use module::item;` for one item.
- `use module::{item1, item2};` for multiple items.
- `use module::*;` to import everything (rarely used; can pollute the namespace).
- `use module::item as alias;` for renaming on import.

### Task 1.5 -- Nested modules and paths

Modules can contain other modules:

```rust
mod tasks {
    pub mod status {
        pub enum Status {
            Pending,
            InProgress,
            Complete,
        }

        pub fn describe(s: &Status) -> &'static str {
            match s {
                Status::Pending => "not started",
                Status::InProgress => "in progress",
                Status::Complete => "done",
            }
        }
    }

    pub struct Task {
        pub title: String,
        pub status: status::Status,
    }

    pub fn new_task(title: &str) -> Task {
        Task {
            title: title.to_string(),
            status: status::Status::Pending,
        }
    }
}

fn main() {
    let task = tasks::new_task("Build a feature");
    let description = tasks::status::describe(&task.status);
    println!("{}: {}", task.title, description);
}
```

Run the program. Expected output:

```
Build a feature: not started
```

The path `tasks::status::describe` walks down the module tree. Each `::` separates one level.

The `pub mod status` declaration makes the nested module itself public; without `pub`, the `status` module would be private and inaccessible from outside `tasks`.

### Task 1.6 -- Path types: crate, super, self

Rust has three special path roots:

- **`crate::`** starts from the crate root.
- **`super::`** starts from the parent module.
- **`self::`** starts from the current module.

Demonstrate them:

```rust
mod tasks {
    pub fn count() -> u32 {
        42
    }

    pub mod stats {
        pub fn report() {
            // super:: refers to the parent (tasks):
            let total = super::count();
            println!("total: {total}");

            // crate:: starts from the crate root:
            crate::print_header();

            // self:: refers to this module (stats):
            self::helper();
        }

        fn helper() {
            println!("(helper called)");
        }
    }
}

fn print_header() {
    println!("--- Report ---");
}

fn main() {
    tasks::stats::report();
}
```

Run the program. Expected output:

```
--- Report ---
total: 42
(helper called)
```

The three path roots cover most needs:

- **`crate::`** is useful for absolute paths from the crate root, no matter how deeply nested the calling code is.
- **`super::`** is useful for accessing items in the parent module without writing the full path.
- **`self::`** is rarely necessary (you can usually just use the bare name), but is sometimes needed to disambiguate.

Most code uses bare names (for items in the current module), `super::` (for parent items), or paths from a `use` statement (for items elsewhere). Explicit `crate::` paths show up mainly in larger projects where the same name might exist in multiple places.

### Checkpoints

1. The struct in Task 1.3 had different visibility on each field: `pub`, `pub(crate)`, `pub(super)`, and private. Why does Rust offer this granularity rather than just public/private?
2. The `use` keyword brings names into scope. What is the difference between `use module::item;` at the top of a file and just writing `module::item` at each call site?
3. The three path types (`crate::`, `super::`, `self::`) cover different starting points. When would you specifically need `crate::` versus a relative path with `super::`?

---

## Exercise 2 -- Splitting Modules Across Files

**Estimated time:** 10-15 minutes
**Topics covered:** module files, `mod.rs` vs `module_name.rs`, the file system to module tree correspondence

### Context

As code grows, single-file modules become unwieldy. Rust lets you split a module's contents into a separate file. The file system structure mirrors the module hierarchy.

### Task 2.1 -- The single-file declaration

Currently your `main.rs` has modules defined inline. To move a module to its own file, change the inline `mod` definition to a bare declaration.

Replace your `src/main.rs` with:

```rust
mod tasks;          // declare the module; body is elsewhere

fn main() {
    let task = tasks::new_task("Try external module");
    println!("{}: {}", task.title, task.completed);
}
```

This will not compile yet because the `tasks` module's content is missing. Create `src/tasks.rs`:

```rust
pub struct Task {
    pub title: String,
    pub completed: bool,
}

pub fn new_task(title: &str) -> Task {
    Task {
        title: title.to_string(),
        completed: false,
    }
}

pub fn complete(task: &mut Task) {
    task.completed = true;
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
Try external module: false
```

What happened:

- The `mod tasks;` in `main.rs` declares the module. The semicolon (instead of a `{ ... }` block) tells the compiler "the body is in a file."
- The compiler looks for the module's content in `src/tasks.rs` (a file with the module's name).
- The content of `tasks.rs` becomes the body of the `tasks` module.

From the rest of the program's perspective, nothing changed. The module is still `tasks`; items are still accessed as `tasks::new_task`, `tasks::Task`, etc. Only the source file location differs.

### Task 2.2 -- Nested modules in subdirectories

To add a submodule to a file-based module, you have two options. The older convention uses `mod.rs`; the newer convention uses a file matching the parent module's name plus a directory.

Update `src/tasks.rs`:

```rust
pub mod status;     // submodule declaration

pub struct Task {
    pub title: String,
    pub completed: bool,
    pub status: status::Status,
}

pub fn new_task(title: &str) -> Task {
    Task {
        title: title.to_string(),
        completed: false,
        status: status::Status::Pending,
    }
}
```

This declares `pub mod status;` as a submodule of `tasks`. The compiler will look for the submodule in either:

- `src/tasks/status.rs` (the newer convention), OR
- `src/tasks/mod.rs` (the older convention).

The newer convention requires a directory matching the parent module's name. Create the directory:

```bash
mkdir src/tasks
```

Wait — there's a complication. If you have BOTH `src/tasks.rs` and `src/tasks/` as a directory, the compiler gets confused: where does the `tasks` module's body live?

The new convention requires you to move the parent module's content into the directory. The directory contains the submodule files, but the parent module's body must be elsewhere. The two paths are:

- **Files:** `src/tasks.rs` for the module, `src/tasks/status.rs` for the submodule.
- **Directories:** `src/tasks/mod.rs` for the module, `src/tasks/status.rs` for the submodule.

You cannot mix these. Pick one. The modern recommendation is the first (separate file plus directory); the older recommendation is the second (`mod.rs` plus same directory).

For this lab, use the modern convention. Create `src/tasks/status.rs`:

```rust
pub enum Status {
    Pending,
    InProgress,
    Complete,
}

pub fn describe(s: &Status) -> &'static str {
    match s {
        Status::Pending => "not started",
        Status::InProgress => "in progress",
        Status::Complete => "done",
    }
}
```

And update `src/main.rs` to use the submodule:

```rust
mod tasks;

fn main() {
    let task = tasks::new_task("Build a feature");
    let desc = tasks::status::describe(&task.status);
    println!("{}: {}", task.title, desc);
}
```

Run the program. Expected output:

```
Build a feature: not started
```

The file structure now mirrors the module structure:

```
src/
├── main.rs              // crate root
├── tasks.rs             // mod tasks (body here)
└── tasks/
    └── status.rs        // mod tasks::status
```

This is the canonical layout for modern Rust projects.

### Task 2.3 -- The mod.rs convention (alternative)

For comparison, briefly consider the older convention. You would not actually do this in modern code, but it appears in older codebases.

The `mod.rs` convention puts the parent module's body inside the directory:

```
src/
├── main.rs
└── tasks/
    ├── mod.rs           // mod tasks (body here)
    └── status.rs        // mod tasks::status
```

Both layouts produce the same module structure. The difference is purely organizational:

- **`tasks.rs` plus `tasks/` directory:** the parent and children are at the same level; the directory contains only children.
- **`tasks/mod.rs` plus `tasks/status.rs`:** everything is in the directory; the parent is `mod.rs`.

The modern (`tasks.rs`) convention is preferred because:

- File names match what you see in the source. When you write `mod tasks;`, you look for `tasks.rs`. With `mod.rs`, you look for a folder, then for the `mod.rs` inside.
- Multiple `mod.rs` files (one per module folder) make navigation in editors confusing.
- Newer editions of Rust (2018 and later) prefer the new convention.

You will see both in real codebases. Older code uses `mod.rs`; newer code uses the named-file convention. The choice does not affect compilation; pick one and be consistent within a project.

### Task 2.4 -- Multiple submodules

Add another submodule. Create `src/tasks/priority.rs`:

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Priority {
    Low,
    Medium,
    High,
    Critical,
}

impl Priority {
    pub fn weight(&self) -> u32 {
        match self {
            Priority::Low => 1,
            Priority::Medium => 5,
            Priority::High => 10,
            Priority::Critical => 100,
        }
    }
}
```

Declare it in `src/tasks.rs`:

```rust
pub mod status;
pub mod priority;

pub struct Task {
    pub title: String,
    pub completed: bool,
    pub status: status::Status,
    pub priority: priority::Priority,
}

pub fn new_task(title: &str, p: priority::Priority) -> Task {
    Task {
        title: title.to_string(),
        completed: false,
        status: status::Status::Pending,
        priority: p,
    }
}
```

Update `main.rs`:

```rust
mod tasks;

use tasks::priority::Priority;
use tasks::status;

fn main() {
    let task = tasks::new_task("Fix the bug", Priority::High);
    println!(
        "{} (priority {:?}, weight {}): {}",
        task.title,
        task.priority,
        task.priority.weight(),
        status::describe(&task.status)
    );
}
```

Run the program. Expected output:

```
Fix the bug (priority High, weight 10): not started
```

The project structure is:

```
src/
├── main.rs
├── tasks.rs
└── tasks/
    ├── status.rs
    └── priority.rs
```

Each submodule lives in its own file. The parent module (`tasks.rs`) declares them and uses their types. The pattern scales: you can keep adding submodules without bloating any single file.

### Task 2.5 -- The use statement for cleaner code

The path `tasks::status::describe(...)` is verbose. Use `use` to clean it up:

```rust
mod tasks;

use tasks::priority::Priority;
use tasks::status::{Status, describe};

fn main() {
    let task = tasks::new_task("Fix the bug", Priority::High);

    let status_desc = describe(&task.status);
    println!(
        "{}: {} (status: {})",
        task.title, task.priority.weight(), status_desc
    );

    // We can also reference the type without the full path:
    let _another: Status = Status::Complete;
}
```

Run the program. The output is similar:

```
Fix the bug: 10 (status: not started)
```

`use` declarations should generally go at the top of a file, after the `mod` declarations. The pattern: declare modules, import what you need, then write your code.

### Checkpoints

1. The modern convention puts the parent module in `tasks.rs` and submodules in `tasks/`. Older code puts everything in `tasks/`, with the parent in `mod.rs`. Why was the change made? What does the newer convention give you?
2. The compiler determines where to look for a module's content based on the `mod` declaration. If you write `mod tasks;` in `main.rs`, the compiler looks for `src/tasks.rs` or `src/tasks/mod.rs`. If you write `mod tasks;` inside `src/lib.rs`, the search starts from the same place. Why is the search relative to the file containing the declaration rather than from the crate root?
3. Long paths can be shortened with `use`. What is the trade-off between heavily using `use` (short call-site code, less obvious where items come from) versus minimal `use` (long call-site code, very obvious where items come from)?

---

## Exercise 3 -- Binary vs. Library Crates

**Estimated time:** 10-15 minutes
**Topics covered:** binary crates, library crates, having both in one package

### Context

A binary crate produces an executable; a library crate produces a reusable component. A single package can contain one or both. This exercise restructures the lab project to have both.

### Task 3.1 -- The current state

Your project is currently a binary crate. The `src/main.rs` is the entry point; `cargo run` builds and runs the executable.

If you want to expose the task logic to other code (other binaries, integration tests, future workspace members), it needs to be in a library crate.

### Task 3.2 -- Add a library crate to the same package

A package can contain both a binary and a library by adding `src/lib.rs` alongside `src/main.rs`. Move the task logic into the library.

Create `src/lib.rs`:

```rust
//! The taskforge library: core types and functions for task management.

pub mod tasks;
```

Move `src/tasks.rs` to the library by leaving it in place; it is still found by Cargo. But update it to remove dependencies on the binary:

```rust
pub mod status;
pub mod priority;

pub struct Task {
    pub title: String,
    pub completed: bool,
    pub status: status::Status,
    pub priority: priority::Priority,
}

pub fn new_task(title: &str, p: priority::Priority) -> Task {
    Task {
        title: title.to_string(),
        completed: false,
        status: status::Status::Pending,
        priority: p,
    }
}
```

Now update `src/main.rs` to use the library. The package name (from `Cargo.toml`) becomes the library's crate name:

```rust
use taskforge::tasks::priority::Priority;
use taskforge::tasks::{new_task, status};

fn main() {
    let task = new_task("Fix the bug", Priority::High);
    println!(
        "{}: {} (priority {})",
        task.title,
        status::describe(&task.status),
        task.priority.weight()
    );
}
```

Notice the path. Instead of `mod tasks;` (which was a module inside the binary), the binary now references `taskforge::tasks` (a module inside the library).

Run the program:

```bash
cargo run
```

Expected output:

```
Fix the bug: not started (priority 10)
```

The same output as before. The structure changed:

- Before: one binary crate with everything in `main.rs` and submodules.
- After: a package with both a library crate and a binary crate. The library contains the reusable logic; the binary uses the library.

Cargo automatically detected both:

- `src/lib.rs` makes this a library crate.
- `src/main.rs` makes this a binary crate.

The package name in `Cargo.toml` (`taskforge`) is used for both, but they are technically separate crates.

### Task 3.3 -- Why have both?

Splitting into library plus binary unlocks several benefits:

- **Integration tests** in the `tests/` directory can use the library, exercising the public API as a real consumer would. They could not do this with a pure-binary project (binaries have no public API).
- **Other crates** can depend on the library. If you publish the package or use it in a workspace, downstream consumers use the library, not the binary.
- **Multiple binaries** can share the library. A single project can have several executables (typically in `src/bin/`) all using the same library.
- **Documentation** generated by `cargo doc` includes the library's public API. The binary itself does not generate user-facing docs.

A common pattern: most of the logic lives in the library; the binary is a thin wrapper that handles command-line parsing, sets up the runtime, and calls into the library.

### Task 3.4 -- Add a second binary

For demonstration, add a second executable. Create `src/bin/admin.rs`:

```bash
mkdir -p src/bin
```

Then create `src/bin/admin.rs`:

```rust
use taskforge::tasks::priority::Priority;
use taskforge::tasks::new_task;

fn main() {
    println!("Admin Tool");
    println!("==========");

    let critical = new_task("Restart server", Priority::Critical);
    let high = new_task("Review logs", Priority::High);
    let low = new_task("Update README", Priority::Low);

    let tasks = vec![&critical, &high, &low];
    let total_weight: u32 = tasks.iter().map(|t| t.priority.weight()).sum();

    println!("total tasks: {}", tasks.len());
    println!("total priority weight: {total_weight}");
}
```

Run it:

```bash
cargo run --bin admin
```

Expected output:

```
Admin Tool
==========
total tasks: 3
total priority weight: 111
```

You now have two binaries (`taskforge` from `src/main.rs`, and `admin` from `src/bin/admin.rs`) that share the library. Cargo picks which binary to run based on the `--bin <name>` flag (defaulting to the main binary).

The structure:

```
taskforge/
├── Cargo.toml
└── src/
    ├── main.rs              // binary "taskforge"
    ├── lib.rs               // library "taskforge"
    ├── tasks.rs
    ├── tasks/
    │   ├── status.rs
    │   └── priority.rs
    └── bin/
        └── admin.rs         // binary "admin"
```

Both binaries link against the library. The library is built once; both binaries depend on it.

### Task 3.5 -- Where to put what

A practical question: should new code go in the library or in the binary?

**Library if:**

- Multiple parts of your project need it.
- You might want to test it as a unit (integration tests can access only the library).
- It is reusable logic (data types, algorithms, business rules).
- It might be shared with other projects later.

**Binary if:**

- It is specific to one executable (command-line parsing for that binary, that binary's main loop, etc.).
- It deals with the entry point or process setup.
- It is "glue code" that ties library logic to a specific deployment.

The general guidance: put most of the logic in the library, keep the binary thin. This makes the code more testable, more reusable, and more flexible to evolve.

For one-shot scripts that will never grow, a single binary is fine. For anything that might evolve into a real project, start with a library plus binary structure.

### Checkpoints

1. The library crate (`src/lib.rs`) and binary crate (`src/main.rs`) share a package name (`taskforge`). What does the package name refer to in `Cargo.toml` versus in `use taskforge::...` statements in the binary?
2. The library could have stayed as inline modules inside `main.rs`. What did extracting it into a library gain you? What did it cost?
3. The pattern "thin binary, fat library" is a common recommendation. What kinds of code genuinely belong in the binary, not the library?

---

## Exercise 4 -- Cargo.toml: Dependencies, Features, and Profiles

**Estimated time:** 10-15 minutes
**Topics covered:** `[dependencies]`, `[dev-dependencies]`, `[features]`, `[profile.*]`, semantic versioning

### Context

`Cargo.toml` is the manifest file that describes your package, its dependencies, and how it is built. This exercise covers the most common sections.

### Task 4.1 -- The default Cargo.toml

Open your `Cargo.toml`. It looks something like:

```toml
[package]
name = "taskforge"
version = "0.1.0"
edition = "2021"

[dependencies]
```

The three required sections of `[package]`:

- **`name`**: the package name. Used by Cargo and as the library/binary name (unless overridden).
- **`version`**: the package's version, following semantic versioning (covered in Task 4.5).
- **`edition`**: which "edition" of Rust to use. Editions are language revisions; `2021` is current at the time of writing.

The `[dependencies]` section is empty. You will add to it next.

### Task 4.2 -- Adding a dependency

Add `serde` (a widely-used serialization library) for an example. Modify `Cargo.toml`:

```toml
[package]
name = "taskforge"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
```

Save the file. Run:

```bash
cargo build
```

Cargo downloads `serde` and its dependencies. The first build takes longer; subsequent builds are fast.

Now you can use serde. Update `src/tasks/priority.rs`:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Copy, PartialEq, Serialize, Deserialize)]
pub enum Priority {
    Low,
    Medium,
    High,
    Critical,
}

impl Priority {
    pub fn weight(&self) -> u32 {
        match self {
            Priority::Low => 1,
            Priority::Medium => 5,
            Priority::High => 10,
            Priority::Critical => 100,
        }
    }
}
```

The `#[derive(Serialize, Deserialize)]` adds serialization support. To verify it works without making the lab much longer, just confirm the project still builds:

```bash
cargo build
```

The build succeeds; the new derive macros are processed.

### Task 4.3 -- Version requirements

The dependency `serde = { version = "1", features = ["derive"] }` accepts any version 1.x. Cargo selects the highest compatible version at build time and records it in `Cargo.lock`.

The version string is a *requirement*, not an exact pin:

- `"1"` means `>=1.0.0, <2.0.0` (any 1.x).
- `"1.0"` means `>=1.0.0, <2.0.0` (same as `"1"`).
- `"1.0.196"` means `>=1.0.196, <2.0.0` (1.x but at least 196).
- `"=1.0.196"` means exactly that version (rare; usually not what you want).
- `">=1.0.0, <1.5.0"` for ranges.

The default behavior is "the highest compatible version." Cargo uses semantic versioning rules: 1.0.196 is compatible with 1.5.0 (same major version), but 2.0.0 is not.

For most code, write the simplest version requirement that captures your intent:

- `version = "1"` is sufficient for most cases.
- `version = "1.2"` if you need a specific minor version's features.
- `version = "1.2.3"` is rarely needed; the patch version is usually irrelevant.

### Task 4.4 -- Dependency types

Cargo has three kinds of dependencies:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }

[dev-dependencies]
# Used only for tests, benchmarks, and examples:
# mockall = "0.12"

[build-dependencies]
# Used by build.rs scripts:
# cc = "1"
```

You have seen `[dev-dependencies]` already (Lab 12 used `mockall`). They are not included in production builds.

`[build-dependencies]` are used by build scripts (`build.rs` files), which run during the build process. Rare in most projects; common in projects that bind to C libraries or generate code.

The three sections separate concerns: regular dependencies for production code, dev for testing/benchmarking, build for the build process.

### Task 4.5 -- Semantic versioning

Cargo uses semantic versioning (semver) to determine compatibility. A version number has three parts:

```
MAJOR.MINOR.PATCH
```

The rules:

- **MAJOR**: incremented for breaking changes. `1.x.x` and `2.x.x` are incompatible.
- **MINOR**: incremented for new features that do not break existing code. `1.0.x` and `1.5.x` are compatible (any code using 1.0 should work with 1.5).
- **PATCH**: incremented for bug fixes that do not change behavior. `1.2.3` and `1.2.4` should behave identically except for the fixed bugs.

For pre-1.0 versions (`0.x.y`), the rules shift:

- `0.x` and `0.y` are considered incompatible if `x != y`. So `0.3.0` and `0.4.0` are incompatible.
- This is because pre-1.0 software is considered unstable; any minor version bump might be breaking.

This is why you sometimes see version requirements like `0.12.5`. For a 0.x crate, you need to be specific about the minor version. For 1.x+ crates, just the major version is usually enough.

The implication for library authors:

- Releasing a new major version requires extra work (users must update). Do it sparingly.
- Minor versions add features. Common in active libraries.
- Patch versions fix bugs. Frequent.

Cargo follows these rules automatically. `cargo update` upgrades dependencies within the compatibility range. `Cargo.lock` records the exact versions used; this ensures reproducible builds.

### Task 4.6 -- Profile configuration

Cargo has different build profiles. Add to `Cargo.toml`:

```toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
lto = true
strip = true
```

The two main profiles:

- **`dev`** is used by `cargo build` and `cargo run` (without `--release`). It optimizes for fast compile times, not runtime speed. Includes debug info.
- **`release`** is used by `cargo build --release` and `cargo run --release`. It optimizes for runtime speed, sacrificing compile time. Strips debug info.

Common profile options:

- **`opt-level`**: optimization level. 0 = no optimization, 1-3 = increasing optimization. Default is 0 for dev, 3 for release.
- **`debug`**: include debug information. Default true for dev, false for release.
- **`lto`**: link-time optimization. Slower compile, faster runtime. Default false.
- **`strip`**: remove debug symbols from the final binary. Smaller binary size.

You generally do not need to configure profiles unless you have specific needs (smaller binaries, faster debug builds, etc.).

Try:

```bash
cargo build --release
```

This builds with the release profile. The result is in `target/release/` instead of `target/debug/`. Release builds take longer but produce faster executables.

### Task 4.7 -- The Cargo.lock file

After running `cargo build`, you should see a `Cargo.lock` file in your project root.

`Cargo.lock` records the exact versions of every dependency (and every transitive dependency) used in the build. It is what makes builds reproducible: anyone who clones your project and runs `cargo build` gets exactly the same dependencies you had.

The rules:

- **For binaries (applications):** commit `Cargo.lock` to version control. You want everyone to use the same dependency versions.
- **For libraries:** typically do NOT commit `Cargo.lock`. Library consumers use their own Cargo.lock; yours is only used when you build the library standalone (for testing).

Cargo automatically respects this: when you build a library by itself, it uses the lock file. When you build a dependent project, that project's lock file is used.

You can update dependencies with `cargo update`. This re-resolves versions within the compatibility ranges and writes a new `Cargo.lock`. Useful when you want to pick up patch releases or specific updates.

### Checkpoints

1. The version requirement `"1"` means "any version 1.x." Why does Cargo not just pin to the exact version it first installed? What does the flexibility give you?
2. The release profile has `opt-level = 3` and `lto = true`. What is the trade-off? Why is the dev profile not just always set to the same?
3. For binaries, you commit `Cargo.lock`; for libraries, you typically do not. What does this difference protect against in each case?

---

## Exercise 5 -- Cargo Workspaces

**Estimated time:** 15-20 minutes
**Topics covered:** workspaces, multi-crate projects, shared dependencies

### Context

A workspace is a collection of crates that share a single `Cargo.lock` and `target/` directory. Workspaces are how you organize multi-crate projects (a typical pattern for non-trivial Rust applications).

### Task 5.1 -- Convert to a workspace

Currently your `taskforge` is a single package. Let me restructure it as a workspace with multiple member crates.

First, back out of the current project structure. The simplest approach is to move the contents to a member directory:

```bash
# From the taskforge directory:
mkdir taskforge-core
mv src taskforge-core/
mv Cargo.toml taskforge-core/

# Verify:
ls taskforge-core/
```

You should see `src/` and `Cargo.toml` inside `taskforge-core/`.

Now create a new top-level `Cargo.toml` for the workspace:

```toml
[workspace]
resolver = "2"
members = [
    "taskforge-core",
]
```

The `[workspace]` section identifies this as a workspace manifest. The `members` field lists the directories containing member crates. The `resolver = "2"` specifies which dependency resolver to use (version 2 is current).

Also update `taskforge-core/Cargo.toml`. Change the name from `taskforge` to `taskforge-core`:

```toml
[package]
name = "taskforge-core"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
```

Now the project structure is:

```
taskforge/
├── Cargo.toml                       # workspace manifest
└── taskforge-core/
    ├── Cargo.toml                   # member crate manifest
    └── src/
        ├── main.rs
        ├── lib.rs
        ├── tasks.rs
        ├── tasks/
        │   ├── status.rs
        │   └── priority.rs
        └── bin/
            └── admin.rs
```

Update `taskforge-core/src/main.rs` and `taskforge-core/src/bin/admin.rs` to use the new crate name. Change `taskforge::` to `taskforge_core::` (note the underscore; Cargo translates dashes in crate names to underscores in Rust code):

```rust
// taskforge-core/src/main.rs:
use taskforge_core::tasks::priority::Priority;
use taskforge_core::tasks::{new_task, status};

fn main() {
    let task = new_task("Fix the bug", Priority::High);
    println!(
        "{}: {} (priority {})",
        task.title,
        status::describe(&task.status),
        task.priority.weight()
    );
}
```

```rust
// taskforge-core/src/bin/admin.rs:
use taskforge_core::tasks::priority::Priority;
use taskforge_core::tasks::new_task;

fn main() {
    println!("Admin Tool");
    println!("==========");

    let critical = new_task("Restart server", Priority::Critical);
    let high = new_task("Review logs", Priority::High);
    let low = new_task("Update README", Priority::Low);

    let tasks = vec![&critical, &high, &low];
    let total_weight: u32 = tasks.iter().map(|t| t.priority.weight()).sum();

    println!("total tasks: {}", tasks.len());
    println!("total priority weight: {total_weight}");
}
```

From the workspace root, build:

```bash
cargo build
```

Expected output: a successful build. Run the main binary:

```bash
cargo run --bin taskforge-core
```

Expected output:

```
Fix the bug: not started (priority 10)
```

The workspace contains one member crate. The next task adds a second.

### Task 5.2 -- Add a second member crate

Add a new crate that depends on `taskforge-core`. This new crate will provide a different command-line interface to the same core library.

From the workspace root:

```bash
cargo new taskforge-cli --bin
```

This creates a new package in `taskforge-cli/`. Now add it to the workspace by updating the root `Cargo.toml`:

```toml
[workspace]
resolver = "2"
members = [
    "taskforge-core",
    "taskforge-cli",
]
```

Make `taskforge-cli` depend on `taskforge-core`. Edit `taskforge-cli/Cargo.toml`:

```toml
[package]
name = "taskforge-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
taskforge-core = { path = "../taskforge-core" }
```

The `path` dependency tells Cargo "this crate is at this relative directory." Workspaces use path dependencies extensively.

Replace `taskforge-cli/src/main.rs` with:

```rust
use taskforge_core::tasks::priority::Priority;
use taskforge_core::tasks::{new_task, status};

fn main() {
    println!("TaskForge CLI v1.0");
    println!();

    let tasks = vec![
        new_task("Deploy to production", Priority::Critical),
        new_task("Write release notes", Priority::High),
        new_task("Update documentation", Priority::Medium),
        new_task("Schedule team lunch", Priority::Low),
    ];

    println!("{} tasks:", tasks.len());
    for task in &tasks {
        println!(
            "  [{:?}] {} ({})",
            task.priority,
            task.title,
            status::describe(&task.status)
        );
    }

    let total_weight: u32 = tasks.iter().map(|t| t.priority.weight()).sum();
    println!("\nTotal priority weight: {total_weight}");
}
```

Build everything from the workspace root:

```bash
cargo build
```

Run the new binary:

```bash
cargo run --bin taskforge-cli
```

Expected output:

```
TaskForge CLI v1.0

4 tasks:
  [Critical] Deploy to production (not started)
  [High] Write release notes (not started)
  [Medium] Update documentation (not started)
  [Low] Schedule team lunch (not started)

Total priority weight: 116
```

The new binary in `taskforge-cli` uses the library from `taskforge-core`. The two are separate crates but share the workspace's `Cargo.lock` and `target/` directory.

### Task 5.3 -- Workspace structure

The current project:

```
taskforge/
├── Cargo.toml                       # workspace
├── taskforge-core/
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs                  # binary "taskforge-core"
│       ├── lib.rs
│       ├── tasks.rs
│       ├── tasks/
│       │   ├── status.rs
│       │   └── priority.rs
│       └── bin/
│           └── admin.rs             # binary "admin"
└── taskforge-cli/
    ├── Cargo.toml
    └── src/
        └── main.rs                  # binary "taskforge-cli"
```

What the workspace gives you:

- **Single `target/` directory.** Both crates build into the same place. Dependencies are compiled once and shared.
- **Single `Cargo.lock`.** All crates agree on dependency versions. No drift between the core library and consumers.
- **Build everything at once.** `cargo build` from the root builds all members.
- **Run any binary.** `cargo run --bin <name>` works from anywhere in the workspace.

The pattern scales. Large Rust projects (compilers, web servers, embedded systems) routinely have 10-50 crates in a single workspace.

### Task 5.4 -- Shared workspace dependencies

When multiple crates need the same dependency, the workspace can declare it once. Add a workspace-level dependencies section to the root `Cargo.toml`:

```toml
[workspace]
resolver = "2"
members = [
    "taskforge-core",
    "taskforge-cli",
]

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
```

Now each member crate can reference the workspace declaration. Update `taskforge-core/Cargo.toml`:

```toml
[package]
name = "taskforge-core"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { workspace = true }
```

The `serde = { workspace = true }` line says "use the workspace's version specification for serde." If you also need serde in `taskforge-cli`, do the same there.

The benefit: the version is specified once at the workspace level. Updating from serde 1.0 to 1.5 means changing one place, not every member crate. This is essential for large workspaces.

Run a build to verify nothing broke:

```bash
cargo build
```

The build succeeds. The `workspace = true` reference resolves to the workspace's version specification.

### Task 5.5 -- When to use a workspace

Workspaces are appropriate when:

- **Multiple related crates evolve together.** A library and its companion CLI. A web service and its shared types. Multiple modules that have grown enough to warrant separate compilation units.
- **You want separate publishing.** Workspace members can be published independently to crates.io with different version numbers.
- **You need separate features per binary.** Each binary can have its own dependencies and feature requirements.
- **The project is large enough to benefit from incremental compilation.** Changing one crate only recompiles that crate (plus dependents); single large crates recompile entirely.

A single crate is appropriate when:

- The project is small.
- All code logically belongs together (no clear boundaries to split on).
- You do not anticipate multiple consumers of the library.

The split into multiple crates is reversible (you can collapse them back into one). The decision is not permanent; choose what works for the current project size.

### Checkpoints

1. The workspace `Cargo.toml` lists members but does not have `[package]` itself. What does this say about workspaces? Can the workspace contain non-crate metadata or build outputs of its own?
2. The `taskforge-cli` crate depends on `taskforge-core` via `path = "../taskforge-core"`. This is a path dependency. How does this differ from a version dependency (`taskforge-core = "0.1"`)? When would each be appropriate?
3. The `[workspace.dependencies]` section lets you define common dependencies once. What is the practical benefit when the workspace has 20 crates that all use the same version of `serde`?

---

## Exercise 6 -- Feature Flags

**Estimated time:** 10-15 minutes
**Topics covered:** `[features]`, optional dependencies, `#[cfg(feature = "...")]`

### Context

Feature flags let you compile parts of your code conditionally. Users of your crate enable features they need; unused features are excluded from the build. This is useful for optional functionality that has its own dependencies.

### Task 6.1 -- Add a feature

Add a feature to `taskforge-core` that enables JSON serialization. Update `taskforge-core/Cargo.toml`:

```toml
[package]
name = "taskforge-core"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { workspace = true }
serde_json = { version = "1", optional = true }

[features]
json = ["dep:serde_json"]
```

Two new things:

- **`serde_json = { version = "1", optional = true }`** declares a dependency that is not used by default. It is only included when explicitly enabled.
- **`[features]`** section defines features. The `json` feature lists what it activates: in this case, the `serde_json` dependency.

The `"dep:serde_json"` syntax says "enabling this feature implies enabling the `serde_json` optional dependency." Without `dep:`, Cargo would try to find a feature named `serde_json`.

### Task 6.2 -- Conditional code

Code can be conditionally compiled based on which features are enabled. Add a function to `taskforge-core/src/tasks.rs`:

```rust
pub mod status;
pub mod priority;

pub struct Task {
    pub title: String,
    pub completed: bool,
    pub status: status::Status,
    pub priority: priority::Priority,
}

pub fn new_task(title: &str, p: priority::Priority) -> Task {
    Task {
        title: title.to_string(),
        completed: false,
        status: status::Status::Pending,
        priority: p,
    }
}

#[cfg(feature = "json")]
pub fn to_json(task: &Task) -> Result<String, serde_json::Error> {
    use serde::Serialize;

    #[derive(Serialize)]
    struct TaskOutput<'a> {
        title: &'a str,
        completed: bool,
        priority_weight: u32,
    }

    let output = TaskOutput {
        title: &task.title,
        completed: task.completed,
        priority_weight: task.priority.weight(),
    };

    serde_json::to_string(&output)
}
```

The `#[cfg(feature = "json")]` attribute on the `to_json` function says "only compile this when the `json` feature is enabled."

Build with the feature disabled (the default):

```bash
cargo build
```

The build succeeds. The `to_json` function is not compiled; `serde_json` is not downloaded; the code does not depend on it.

Now build with the feature enabled:

```bash
cargo build --features json
```

This time, Cargo downloads `serde_json`, and the `to_json` function is compiled.

### Task 6.3 -- Using the feature from the CLI

Update `taskforge-cli/Cargo.toml` to enable the feature:

```toml
[package]
name = "taskforge-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
taskforge-core = { path = "../taskforge-core", features = ["json"] }
```

The `features = ["json"]` tells Cargo "when building `taskforge-core`, enable the `json` feature."

Now use the `to_json` function. Update `taskforge-cli/src/main.rs`:

```rust
use taskforge_core::tasks::priority::Priority;
use taskforge_core::tasks::{new_task, status, to_json};

fn main() {
    println!("TaskForge CLI v1.0");
    println!();

    let tasks = vec![
        new_task("Deploy to production", Priority::Critical),
        new_task("Write release notes", Priority::High),
        new_task("Update documentation", Priority::Medium),
        new_task("Schedule team lunch", Priority::Low),
    ];

    println!("{} tasks:", tasks.len());
    for task in &tasks {
        println!(
            "  [{:?}] {} ({})",
            task.priority,
            task.title,
            status::describe(&task.status)
        );
    }

    println!("\nAs JSON:");
    for task in &tasks {
        match to_json(task) {
            Ok(json) => println!("  {json}"),
            Err(e) => eprintln!("  error: {e}"),
        }
    }
}
```

Run it:

```bash
cargo run --bin taskforge-cli
```

Expected output:

```
TaskForge CLI v1.0

4 tasks:
  [Critical] Deploy to production (not started)
  [High] Write release notes (not started)
  [Medium] Update documentation (not started)
  [Low] Schedule team lunch (not started)

As JSON:
  {"title":"Deploy to production","completed":false,"priority_weight":100}
  {"title":"Write release notes","completed":false,"priority_weight":10}
  {"title":"Update documentation","completed":false,"priority_weight":5}
  {"title":"Schedule team lunch","completed":false,"priority_weight":1}
```

The CLI gets JSON output because it enabled the `json` feature.

Other consumers of `taskforge-core` that do not need JSON would not enable the feature. They would not pay the cost (download time, compile time, binary size) of `serde_json`.

### Task 6.4 -- A second feature

Features can also enable code paths without adding new dependencies. Add another feature:

```toml
[features]
json = ["dep:serde_json"]
verbose-logs = []
```

The `verbose-logs` feature does not enable any dependencies (the list is empty). It is just a flag.

Use it to conditionally include code. In `taskforge-core/src/tasks.rs`:

```rust
pub fn new_task(title: &str, p: priority::Priority) -> Task {
    #[cfg(feature = "verbose-logs")]
    println!("[log] creating task: {title}");

    Task {
        title: title.to_string(),
        completed: false,
        status: status::Status::Pending,
        priority: p,
    }
}
```

With `verbose-logs` enabled, the function logs each creation. Without it, the log line is excluded from compilation entirely.

Try enabling it:

```bash
cargo run --bin taskforge-cli --features taskforge-core/verbose-logs
```

(The feature is specified from the workspace member that needs it.)

Expected output (with logging):

```
TaskForge CLI v1.0

[log] creating task: Deploy to production
[log] creating task: Write release notes
[log] creating task: Update documentation
[log] creating task: Schedule team lunch
4 tasks:
...
```

The log lines appear because the feature was enabled. Without the flag, they would not be compiled in.

### Task 6.5 -- Default features

You can specify which features are enabled by default. Add to `taskforge-core/Cargo.toml`:

```toml
[features]
default = ["json"]
json = ["dep:serde_json"]
verbose-logs = []
```

The `default` feature is automatic; anyone using your crate gets the default features unless they explicitly opt out. Now `cargo build` builds with `json` enabled without needing `--features json`.

Consumers who do not want the default features pass `default-features = false`:

```toml
[dependencies]
taskforge-core = { path = "../taskforge-core", default-features = false }
```

This is useful for minimal builds (embedded systems, very small binaries) where you want only the strict minimum.

The trade-off: defaults make the common case easy (most users get what they need). The cost is that opting out is slightly more work.

For libraries: default features should cover the "common case." For applications: usually use the defaults.

### Task 6.6 -- When to use feature flags

Feature flags are appropriate for:

- **Optional dependencies.** A library that supports multiple backends (JSON, YAML, TOML); each backend has its own dependency.
- **Optional functionality.** A library that has a "lite" mode (smaller binary) and a "full" mode (more features).
- **Compatibility.** A library that supports multiple Rust versions or platforms; feature flags enable platform-specific code.
- **Performance trade-offs.** A library where some features cost compile time or binary size; users opt in.

Features are NOT appropriate for:

- **Configuration that should be runtime.** If users would want to change it without recompiling, use runtime configuration, not features.
- **Hiding bugs.** A feature flag is not a substitute for handling errors properly.
- **Internal-only experimentation.** Use private branches or `cfg(test)` for that.

The general rule: features are for compile-time decisions that affect what code or dependencies are included. They are particularly valuable for libraries that want to be usable in different contexts.

### Checkpoints

1. The `json` feature enables an optional dependency. Why is this preferable to just including `serde_json` as a regular dependency that everyone gets?
2. The `verbose-logs` feature enables code without adding dependencies. The conditionally compiled `println!` is excluded when the feature is off. Why not just check at runtime?
3. Default features are convenient but can hide what users are actually getting. What is the trade-off between "many default features" and "minimal default, opt-in features"?

---

## Exercise 7 -- Preparing for Publication

**Estimated time:** 10 minutes
**Topics covered:** crate metadata, README, license, what is needed for crates.io

### Context

Publishing a crate to crates.io requires more than just code: metadata, documentation, a README, a license. This exercise covers what to add.

### Task 7.1 -- Crate metadata

Update `taskforge-core/Cargo.toml` with publication metadata:

```toml
[package]
name = "taskforge-core"
version = "0.1.0"
edition = "2021"
description = "Core task management types and operations."
authors = ["Your Name <you@example.com>"]
license = "MIT OR Apache-2.0"
repository = "https://github.com/yourname/taskforge"
readme = "README.md"
keywords = ["task", "tracker", "productivity"]
categories = ["data-structures"]

[dependencies]
serde = { workspace = true }
serde_json = { version = "1", optional = true }

[features]
default = ["json"]
json = ["dep:serde_json"]
verbose-logs = []
```

The new fields:

- **`description`**: a short summary (one or two sentences). Shows up in crate listings.
- **`authors`**: who maintains the crate. Used in some tools.
- **`license`**: the license identifier. Use SPDX names. `"MIT OR Apache-2.0"` is the standard for Rust crates (dual-licensed, most permissive).
- **`repository`**: link to the source code. Required for many tools to work.
- **`readme`**: relative path to a README file. The crates.io page shows this.
- **`keywords`**: up to 5 keywords for search. Choose specific ones.
- **`categories`**: standard categories. The complete list is at https://crates.io/category_slugs.

### Task 7.2 -- A README

Create `taskforge-core/README.md`:

```markdown
# taskforge-core

Core types and operations for the TaskForge task management system.

## Features

- Task model with priority and status.
- Optional JSON serialization (enable the `json` feature).
- Verbose logging for debugging (enable the `verbose-logs` feature).

## Usage

```rust
use taskforge_core::tasks::priority::Priority;
use taskforge_core::tasks::new_task;

let task = new_task("Write the docs", Priority::High);
println!("{}: priority {}", task.title, task.priority.weight());
```

## Feature Flags

- `json` (default): JSON serialization via `serde_json`.
- `verbose-logs`: Print log messages during task operations.

## License

MIT OR Apache-2.0
```

The README is the front page of your crate on crates.io. Make it clear and informative:

- What the crate does (one sentence).
- The main features.
- A simple usage example.
- Any feature flags.
- The license.

For published crates, the README is essential. People decide whether to use your crate based on it.

### Task 7.3 -- A license file

For dual-licensed crates (MIT OR Apache-2.0), include both license texts. Create `taskforge-core/LICENSE-MIT` with the MIT license text and `taskforge-core/LICENSE-APACHE` with the Apache 2.0 license text. (The actual text is available at https://opensource.org/licenses; you would copy the appropriate file in a real project.)

For this lab, you can skip creating the actual license files (it would be a lot of copy-pasting). But know that they are required for actual publication.

### Task 7.4 -- The dry run

Cargo can verify your crate is ready for publication without actually publishing:

```bash
cd taskforge-core
cargo publish --dry-run
```

This packages the crate as if for publication, runs all the validation checks, and reports any problems. Common errors:

- Missing `description`, `license`, or `repository` fields.
- A `path` dependency that has no `version` field (path-only deps are not valid for published crates).
- Files that would be included but should not be (build artifacts, secrets).

Address each error before doing a real publish.

For this lab, the dry run will likely fail because `taskforge-core` has path-only dependencies through the workspace. To publish, you would need to first publish the dependencies (or remove them).

### Task 7.5 -- The publish step

For an actual publication (not in this lab):

```bash
cargo login
# Paste your crates.io API token
cargo publish
```

The token comes from your crates.io account. Once you publish a version, it is permanent. You cannot delete it or replace it; you can only "yank" it (mark it as unsuitable for new projects). This is why the dry-run validation matters.

For libraries you intend to publish:

- Use semantic versioning carefully. Breaking changes mean a major version bump.
- Once published, treat the version as immutable.
- Yank versions only if they have severe problems (broken builds, security issues).

For internal crates that will never be published, this section is just context. The metadata and README are still useful even for unpublished crates (they help colleagues and your future self).

### Task 7.6 -- The publish field

If you have crates that should NEVER be published (internal libraries, application binaries), prevent accidental publishing:

```toml
[package]
name = "taskforge-cli"
publish = false
```

The `publish = false` line tells Cargo "this crate should not be published." `cargo publish` will refuse.

This is good practice for any crate that you do not intend to publish. It prevents accidents.

Add `publish = false` to `taskforge-cli/Cargo.toml`. The CLI is not a library; nobody else needs it. Marking it appropriately documents the intent.

### Checkpoints

1. The Cargo.toml has many metadata fields (`description`, `authors`, `license`, etc.). Most are not required for compilation. What is their purpose, and why does Cargo require them for publication?
2. The `publish = false` field prevents accidental publication. Why is this useful even for crates that you have no plans to publish?
3. The README appears on crates.io. What information do users need that would not naturally be in the doc comments or in the code itself?

---

## Exercise 8 -- Putting It Together

**Estimated time:** 15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise reviews the workspace structure you have built and exercises the patterns one more time. You will add a third workspace member that demonstrates the patterns together: it depends on the core library, uses path dependencies, has its own feature flags, and shows the full multi-crate organization.

### Task 8.1 -- Add a reporting crate

From the workspace root, create a new library crate:

```bash
cargo new taskforge-report --lib
```

Update the root `Cargo.toml` to include it:

```toml
[workspace]
resolver = "2"
members = [
    "taskforge-core",
    "taskforge-cli",
    "taskforge-report",
]

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
```

Update `taskforge-report/Cargo.toml`:

```toml
[package]
name = "taskforge-report"
version = "0.1.0"
edition = "2021"
description = "Reporting utilities for TaskForge."
license = "MIT OR Apache-2.0"
publish = false

[dependencies]
taskforge-core = { path = "../taskforge-core" }
serde = { workspace = true }

[features]
csv = []
```

The new crate:

- Depends on `taskforge-core` via path.
- Uses the workspace's `serde` declaration.
- Has its own `csv` feature for CSV output.
- Is marked `publish = false` (internal only).

### Task 8.2 -- Implement the report library

Replace `taskforge-report/src/lib.rs` with:

```rust
//! Reporting utilities for the TaskForge system.
//!
//! This crate provides functions for summarizing collections of tasks
//! in various output formats.

use taskforge_core::tasks::Task;
use taskforge_core::tasks::status::Status;

/// Computes summary statistics for a slice of tasks.
pub fn summary(tasks: &[Task]) -> Summary {
    let total = tasks.len();
    let completed = tasks.iter().filter(|t| t.completed).count();
    let pending = tasks.iter().filter(|t| matches!(t.status, Status::Pending)).count();
    let in_progress = tasks.iter().filter(|t| matches!(t.status, Status::InProgress)).count();
    let total_weight: u32 = tasks.iter().map(|t| t.priority.weight()).sum();

    Summary {
        total,
        completed,
        pending,
        in_progress,
        total_weight,
    }
}

/// A summary of a task collection.
#[derive(Debug)]
pub struct Summary {
    pub total: usize,
    pub completed: usize,
    pub pending: usize,
    pub in_progress: usize,
    pub total_weight: u32,
}

impl Summary {
    /// Format the summary as a human-readable string.
    pub fn to_text(&self) -> String {
        format!(
            "Total: {}, Completed: {}, Pending: {}, In Progress: {}, Weight: {}",
            self.total, self.completed, self.pending, self.in_progress, self.total_weight
        )
    }
}

/// Format tasks as CSV. Available only with the `csv` feature.
#[cfg(feature = "csv")]
pub fn to_csv(tasks: &[Task]) -> String {
    let mut output = String::from("title,priority_weight,completed\n");
    for task in tasks {
        output.push_str(&format!(
            "{},{},{}\n",
            task.title,
            task.priority.weight(),
            task.completed
        ));
    }
    output
}
```

### Task 8.3 -- Use the report library in the CLI

Update `taskforge-cli/Cargo.toml` to depend on the report library:

```toml
[package]
name = "taskforge-cli"
version = "0.1.0"
edition = "2021"
publish = false

[dependencies]
taskforge-core = { path = "../taskforge-core", features = ["json"] }
taskforge-report = { path = "../taskforge-report", features = ["csv"] }
```

Update `taskforge-cli/src/main.rs`:

```rust
use taskforge_core::tasks::priority::Priority;
use taskforge_core::tasks::{new_task, status, to_json};
use taskforge_report::{summary, to_csv};

fn main() {
    println!("TaskForge CLI v1.0");
    println!();

    let tasks = vec![
        new_task("Deploy to production", Priority::Critical),
        new_task("Write release notes", Priority::High),
        new_task("Update documentation", Priority::Medium),
        new_task("Schedule team lunch", Priority::Low),
    ];

    println!("Tasks:");
    for task in &tasks {
        println!(
            "  [{:?}] {} ({})",
            task.priority,
            task.title,
            status::describe(&task.status)
        );
    }

    println!();
    let s = summary(&tasks);
    println!("Summary: {}", s.to_text());

    println!();
    println!("As JSON:");
    for task in &tasks {
        if let Ok(json) = to_json(task) {
            println!("  {json}");
        }
    }

    println!();
    println!("As CSV:");
    print!("{}", to_csv(&tasks));
}
```

Build and run:

```bash
cargo run --bin taskforge-cli
```

Expected output:

```
TaskForge CLI v1.0

Tasks:
  [Critical] Deploy to production (not started)
  [High] Write release notes (not started)
  [Medium] Update documentation (not started)
  [Low] Schedule team lunch (not started)

Summary: Total: 4, Completed: 0, Pending: 4, In Progress: 0, Weight: 116

As JSON:
  {"title":"Deploy to production","completed":false,"priority_weight":100}
  {"title":"Write release notes","completed":false,"priority_weight":10}
  {"title":"Update documentation","completed":false,"priority_weight":5}
  {"title":"Schedule team lunch","completed":false,"priority_weight":1}

As CSV:
title,priority_weight,completed
Deploy to production,100,false
Write release notes,10,false
Update documentation,5,false
Schedule team lunch,1,false
```

The CLI now depends on two libraries (`taskforge-core` and `taskforge-report`), each with their own feature flag enabled. The workspace ties them together cleanly.

### Task 8.4 -- Inspect the workspace

The final project structure:

```
taskforge/
├── Cargo.toml                       # workspace
├── Cargo.lock
├── target/                          # shared build output
├── taskforge-core/
│   ├── Cargo.toml
│   ├── README.md
│   └── src/
│       ├── main.rs                  # binary "taskforge-core"
│       ├── lib.rs                   # library "taskforge_core"
│       ├── tasks.rs
│       ├── tasks/
│       │   ├── status.rs
│       │   └── priority.rs
│       └── bin/
│           └── admin.rs             # binary "admin"
├── taskforge-cli/
│   ├── Cargo.toml
│   └── src/
│       └── main.rs                  # binary "taskforge-cli"
└── taskforge-report/
    ├── Cargo.toml
    └── src/
        └── lib.rs                   # library "taskforge_report"
```

Three crates:

- `taskforge-core` (binary + library) provides the core types and optional JSON.
- `taskforge-cli` (binary) is the command-line interface, depending on both libraries.
- `taskforge-report` (library) provides reporting utilities with optional CSV.

Run `cargo build` from the workspace root. Cargo builds everything: 3 crates, 2 main binaries, and 1 secondary binary. All share the same `target/` directory and `Cargo.lock`.

### Task 8.5 -- Identify the patterns

In `lab13-notes.md`, identify each instance of the following patterns in the workspace above:

1. A package containing both a binary crate and a library crate.
2. A secondary binary in `src/bin/` (the `admin.rs` binary).
3. Use of `pub(crate)` to limit visibility to within a crate.
4. A module split into multiple files under `tasks/`.
5. A workspace member with a `path` dependency on another workspace member.
6. A workspace-level dependency definition shared across members.
7. An optional dependency that is gated behind a feature flag.
8. A feature flag that does not introduce any dependencies (boolean toggle).
9. A `publish = false` declaration on a crate that should not be published.
10. The dash-to-underscore translation between Cargo names (`taskforge-core`) and Rust paths (`taskforge_core`).

### Task 8.6 -- Predict and verify

For each of the following changes, predict what will happen and verify. Write your predictions in `lab13-notes.md` BEFORE running.

**Change A:** Remove the `features = ["json"]` from `taskforge-cli`'s dependency on `taskforge-core`. What happens to the `to_json` call in `main.rs`? What is the error message?

**Change B:** In `taskforge-report/Cargo.toml`, change `taskforge-core = { path = "../taskforge-core" }` to `taskforge-core = "0.1"` (a version-only dependency). What happens when you try to build?

**Change C:** Add a new function `internal_helper` (without `pub`) to `taskforge-core/src/lib.rs`. Try to use it from `taskforge-cli/src/main.rs`. What happens? Now change it to `pub(crate)` and try again. What changes?

For each change, write your prediction first, then run and verify. Remember to revert each change before doing the next.

### Checkpoints

1. The workspace has a single `Cargo.lock` shared by all members. What problem does this prevent that having separate `Cargo.lock` files per crate would cause?
2. The CLI depends on both `taskforge-core` (with the `json` feature) and `taskforge-report` (with the `csv` feature). Each dependency has its own feature set. Why does this granularity matter for library design?
3. The library crate names in source code use underscores (`taskforge_core`), but the Cargo names use dashes (`taskforge-core`). Why this difference? What is the rule?

---

## Summary and Reflection

You have now used every concept from Module 17 in a working workspace.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Modules in a Single File | `mod`, `pub`, `use`, paths | Modules organize code; visibility levels (`pub`, `pub(crate)`, etc.) control what is accessible from where. |
| 2 -- Splitting Modules Across Files | file-to-module correspondence | The file system mirrors the module tree. The `mod` declaration tells the compiler where to find the body. |
| 3 -- Binary vs Library Crates | dual-purpose packages | Most code belongs in a library; the binary is a thin wrapper. This enables testing, sharing, and multiple binaries. |
| 4 -- Cargo.toml | dependencies, profiles, versioning | Cargo.toml is the manifest. Semver controls compatibility; profiles control how the build is optimized. |
| 5 -- Workspaces | multi-crate organization | Workspaces share a single `Cargo.lock` and `target/`. Members depend on each other via `path`. |
| 6 -- Feature Flags | conditional compilation | Features let users opt into functionality with its own dependencies. Use `#[cfg(feature = "...")]` for code, optional dependencies for the rest. |
| 7 -- Preparing for Publication | metadata, README, license | Publishing requires metadata; even unpublished crates benefit from good metadata. |
| 8 -- Putting It Together | all of the above | Real Rust projects combine these patterns. The workspace grows as the project grows. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab13-notes.md` before your next session.

1. Of the organization concepts in Module 17, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. The lab restructured the project several times: first a single binary, then a binary plus library, then a multi-crate workspace. In a real project, you would not start as a workspace; you would refactor toward it as the code grew. What signs would tell you it is time to split a crate into multiple, or extract a workspace from a single project?

3. Rust's module system has stricter rules than many languages (explicit `mod` declarations, file system correspondence, multiple visibility levels). What does this strictness give you that languages with more flexible module systems (Python, JavaScript) do not provide? What does it cost?

---

*End of Lab 13*
