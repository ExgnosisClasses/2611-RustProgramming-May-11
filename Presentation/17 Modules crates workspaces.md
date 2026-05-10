# Module 17: Modules, Crates, and Workspaces

## Module Overview

This module covers how Rust code is organized at the source level. The previous modules covered language features. This one covers everything around them: how files are arranged, how visibility works, how dependencies are managed, and how multi-crate projects are structured.

The interesting parts of this module are not the syntax (which is straightforward) but the design choices behind it. Three ideas are central:

1. **Visibility is opt-in.** Everything is private by default. To make something accessible from outside its module, you mark it `pub`. This is the opposite of many languages where you have to explicitly hide things.
2. **Crates are the unit of compilation and distribution.** A crate is what you upload to crates.io, what shows up in `Cargo.toml`, and what the compiler builds as one piece. Modules organize code within a crate; crates organize code into distributable units.
3. **Cargo is part of the language.** The build system, package manager, and dependency resolver are not external tools bolted onto Rust. They are part of the standard development experience and inform language-level features like `cfg` attributes.

By the end of this module, students will:

- Organize code into modules with `mod`, control visibility with `pub`, and bring names into scope with `use`.
- Use the visibility modifiers `pub(crate)` and `pub(super)` to expose items at the right level.
- Distinguish binary crates (programs) from library crates (reusable code).
- Read and write `Cargo.toml` files including dependencies, features, and profiles.
- Apply semantic versioning to library design and dependency selection.
- Structure multi-crate projects as Cargo workspaces.
- Publish a crate to crates.io.
- Use feature flags to provide optional functionality.

This module is heavy on practical patterns. Most of what makes a Rust project pleasant to work in (or unpleasant) comes down to the organization decisions covered here.

> **A note on scope.** This module covers the most important parts of Cargo and the module system. Cargo has many features beyond what is covered here: custom build scripts, build profiles for specific targets, registry configuration, and so on. The official Cargo book is the comprehensive reference. This module covers what 95% of projects need.

---

## A. Modules: `mod`, `pub`, `use`, and Paths

### What Is a Module?

A module is a namespace for items: functions, types, constants, other modules. Modules let you organize code into logical groups and control which parts are visible to which other parts.

In Rust, every file is implicitly a module. You can also create modules within files with the `mod` keyword.

### Modules Within a File

```rust
mod math {
    pub fn square(n: i32) -> i32 {
        n * n
    }

    pub fn cube(n: i32) -> i32 {
        n * n * n
    }
}

fn main() {
    let s = math::square(5);
    let c = math::cube(3);
    println!("{s}, {c}");
}
```

The `math` block creates a module named `math`. Inside, the functions are accessible as `math::square` and `math::cube`. Without `pub`, the functions would be private to the module and inaccessible from `main`.

### Modules in Separate Files

For larger codebases, modules go in separate files. Two patterns work:

**Pattern 1: A single file named after the module.**

```
src/
├── main.rs
└── math.rs
```

In `main.rs`:

```rust
mod math;        // declares the math module, loaded from math.rs

fn main() {
    let s = math::square(5);
}
```

In `math.rs`:

```rust
pub fn square(n: i32) -> i32 {
    n * n
}
```

The `mod math;` declaration in `main.rs` tells the compiler "look for the `math` module in `math.rs` (or `math/mod.rs`)." The contents of `math.rs` become the contents of the `math` module.

**Pattern 2: A directory with submodules.**

```
src/
├── main.rs
└── math/
    ├── mod.rs       (or math.rs in modern projects)
    ├── basic.rs
    └── advanced.rs
```

In `main.rs`:

```rust
mod math;
```

In `math/mod.rs` (or the modern `src/math.rs` with files in `src/math/`):

```rust
pub mod basic;
pub mod advanced;
```

In `math/basic.rs`:

```rust
pub fn square(n: i32) -> i32 { n * n }
```

This produces a hierarchical structure: `math::basic::square`, `math::advanced::something`. The directory structure mirrors the module structure.

The modern preferred convention (since the 2018 edition) uses `math.rs` next to a `math/` directory rather than `math/mod.rs`. Both work; the modern style is slightly cleaner.

### `pub`: Making Things Public

Items in a module are private by default. To expose them, mark them `pub`:

```rust
mod math {
    pub fn square(n: i32) -> i32 { n * n }    // public
    fn helper() -> i32 { 42 }                  // private; not callable from outside
}
```

The same rule applies to types, constants, and submodules. `pub struct`, `pub const`, `pub mod` all make their items accessible from outside the module.

For struct fields, `pub` is per-field. A struct can be public while having private fields:

```rust
mod data {
    pub struct Container {
        pub size: usize,
        contents: Vec<i32>,        // private field
    }

    impl Container {
        pub fn new() -> Self {
            Container { size: 0, contents: Vec::new() }
        }
    }
}
```

`Container` is public, so external code can use the type. The `contents` field is private, so external code cannot access or modify it. The struct can only be constructed through the public `new` method (which acts as a controlled entry point).

### `use`: Bringing Names Into Scope

The `use` keyword imports names into the current scope so you do not have to write the full path every time:

```rust
use std::collections::HashMap;
use std::io::{self, Read, Write};

fn main() {
    let mut map: HashMap<String, i32> = HashMap::new();
    // ...
}
```

`use` brings names into scope in three ways:

```rust
use std::collections::HashMap;            // bring HashMap into scope
use std::io;                              // bring the io module into scope
use std::io::{Read, Write};               // bring multiple items in one statement
use std::collections::*;                  // bring everything (rarely a good idea)
```

The `as` keyword renames an imported item:

```rust
use std::io::Result as IoResult;          // avoid name conflicts
```

Naming conflicts are common when working with multiple libraries that have similar names. `as` is the typical fix.

### Paths: Absolute and Relative

When referring to items in modules, you use paths. Two kinds:

- **Absolute paths** start from the crate root: `crate::math::square`.
- **Relative paths** start from the current module: `math::square` or `super::math::square`.

```rust
mod math {
    pub fn square(n: i32) -> i32 { n * n }
    
    pub fn helper(n: i32) -> i32 {
        // From within math, refer to square directly:
        square(n) + 1
    }
}

mod other {
    pub fn use_math() -> i32 {
        // From a different module, use a path:
        crate::math::square(5)
    }
}
```

The `crate::` prefix means "starting from the crate root." `self::` means "starting from this module." `super::` means "starting from the parent module."

### A Practical File Layout

A typical small project might look like this:

```
src/
├── main.rs              (binary entry point)
├── lib.rs               (library entry point, if applicable)
├── config.rs            (config module)
├── server/
│   ├── handlers.rs
│   └── routes.rs
└── server.rs            (server module, declares submodules)
```

In `main.rs`:

```rust
mod config;
mod server;

fn main() {
    let cfg = config::load();
    server::run(cfg);
}
```

In `server.rs`:

```rust
pub mod handlers;
pub mod routes;

use crate::config::Config;

pub fn run(config: Config) {
    // ...
}
```

The directory structure makes the module structure visible at a glance. Files map to modules; subdirectories map to submodules. This pattern scales from small projects to large ones.

---

## B. Visibility Rules: `pub`, `pub(crate)`, `pub(super)`

`pub` is the most common visibility modifier, but Rust has finer-grained options.

### The Problem

Sometimes you want an item to be visible to some code but not others. For example:

- A library has multiple modules that need to share helper functions, but those helpers should not be part of the library's public API.
- A submodule needs to expose items to its parent module but not to the rest of the crate.

`pub` (visible everywhere) and the absence of `pub` (visible only within the same module) are too coarse for these cases.

### `pub(crate)`: Visible Within the Crate

`pub(crate)` makes an item visible to any code in the same crate but invisible to external users:

```rust
pub(crate) fn internal_helper() -> i32 {
    42
}
```

This is the right choice for items that need to be shared across modules within a library but should not be part of the library's public API. The compiler will reject any use of `internal_helper` from outside the crate.

This is one of the most useful visibility modifiers for library development. It lets you organize code into modules without leaking implementation details.

### `pub(super)`: Visible to the Parent Module

`pub(super)` makes an item visible only to the parent module:

```rust
mod outer {
    mod inner {
        pub(super) fn helper() -> i32 { 42 }
    }

    fn use_helper() {
        inner::helper();              // OK: outer is the parent of inner
    }
}

fn main() {
    // outer::inner::helper();        // ERROR: not visible at the crate root
}
```

This is useful when an inner module wants to expose items to its immediate parent without making them broadly visible.

### `pub(in path)`: Visible Within a Specific Module

For more precise control, you can name a module:

```rust
mod outer {
    mod middle {
        pub(in crate::outer) fn helper() -> i32 { 42 }
    }

    fn use_it() {
        middle::helper();              // OK
    }
}
```

`pub(in crate::outer)` says "visible to anything inside `crate::outer` (and its submodules), nowhere else." This is rarely needed; `pub(crate)` and `pub(super)` cover most cases.

### Choosing the Right Visibility

The default rule is "everything as private as possible." Start with no modifier (private), and add visibility only when needed:

| Visibility            | Used for                                                      |
|-----------------------|---------------------------------------------------------------|
| (no modifier)         | Implementation details internal to one module.                |
| `pub(super)`          | Helpers shared with the immediate parent module.              |
| `pub(crate)`          | Helpers shared across the crate but not part of public API.   |
| `pub`                 | Items that are part of the public API.                        |

Library authors should be careful with `pub`. Anything marked `pub` becomes part of the contract with users and cannot be changed without breaking them. Items marked `pub(crate)` can be reorganized freely.

### Why Default-Private?

Rust's default is opposite to many languages (Java's package-private, Python's everything-public-by-convention). The reasoning:

- **Encapsulation by default.** Implementation details should not leak unless intended.
- **Easier refactoring.** Code that is private can be reorganized without breaking anyone.
- **Smaller public APIs.** Library authors are forced to think about what they expose.

The cost is a few extra `pub` keywords. The benefit is that what you mean to expose is what you actually expose.

---

## C. Crates: Binary vs. Library Crates

A crate is a unit of compilation. When you run `cargo build`, you are building a crate. Crates come in two kinds.

### Binary Crates

A binary crate produces an executable. It has a `main` function as its entry point:

```rust
// src/main.rs
fn main() {
    println!("hello, world");
}
```

`cargo new my_app` creates a binary crate. The default file is `src/main.rs`. Running `cargo run` compiles the crate and runs the resulting executable.

A package can have multiple binaries. Additional binaries go in `src/bin/`:

```
my_app/
├── Cargo.toml
└── src/
    ├── main.rs           (the default binary)
    └── bin/
        ├── tool1.rs      (a second binary)
        └── tool2.rs      (a third binary)
```

You can run a specific binary with `cargo run --bin tool1`.

### Library Crates

A library crate produces a reusable library. It has no `main` function. The entry point is `src/lib.rs`:

```rust
// src/lib.rs
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

`cargo new my_lib --lib` creates a library crate. Running `cargo build` compiles the library; there is nothing to run with `cargo run`.

Library crates are what you publish to crates.io for others to use. The public API of a library is everything marked `pub` in `lib.rs` and its submodules.

### Combined Packages

A package can have both a binary and a library:

```
my_project/
├── Cargo.toml
└── src/
    ├── main.rs            (binary)
    └── lib.rs             (library)
```

The binary can use the library:

```rust
// src/main.rs
use my_project::process;        // imports from src/lib.rs

fn main() {
    process();
}
```

```rust
// src/lib.rs
pub fn process() {
    println!("processing");
}
```

This is the typical structure for projects that have both reusable logic (in the library) and a command-line interface (in the binary). The library can be used by other Rust projects; the binary is what users run.

### `Cargo.toml` for Each Type

The `Cargo.toml` is similar for both, with small differences:

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2024"

[dependencies]
# ...
```

For an explicit binary or library configuration, you can add sections:

```toml
[lib]
name = "my_lib"
path = "src/lib.rs"

[[bin]]
name = "tool"
path = "src/bin/tool.rs"
```

The defaults handle most projects without explicit configuration. You add these sections only when you need non-default behavior (different file paths, multiple binaries with custom names, etc.).

---

## D. `Cargo.toml`: Dependencies, Features, and Profiles

`Cargo.toml` is the manifest file for a Rust package. It describes the package's metadata, dependencies, features, and build configuration.

### The Basic Structure

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2024"
authors = ["Alice <alice@example.com>"]
description = "A short description"
license = "MIT"

[dependencies]
serde = "1.0"
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
proptest = "1"

[build-dependencies]
cc = "1"
```

The `[package]` table contains metadata. The `[dependencies]` table lists runtime dependencies. `[dev-dependencies]` are used only for tests and examples. `[build-dependencies]` are used by build scripts.

### Specifying Versions

Several version specifications:

```toml
[dependencies]
serde = "1.0"                              # any 1.x compatible (1.0.0 to <2.0.0)
serde = "=1.0.130"                          # exactly 1.0.130
serde = "^1.0"                              # same as "1.0" (caret is implicit)
serde = "~1.0.30"                           # 1.0.x (>=1.0.30, <1.1.0)
serde = "*"                                 # any version (almost never used)
```

The default specification (`"1.0"`) is "compatible with 1.0." This means any 1.x version that does not break compatibility. Cargo picks the latest compatible version when you run `cargo build`.

For most dependencies, the default form is what you want. The dependency author has committed to not breaking compatibility within a major version, so you get bug fixes and features without breaking changes.

### Dependency Sources

Dependencies can come from several places:

```toml
[dependencies]
# crates.io (the default)
serde = "1.0"

# git repository
my_lib = { git = "https://github.com/me/my_lib", branch = "main" }

# local path (for workspaces or development)
my_lib = { path = "../my_lib" }

# registry other than crates.io
my_lib = { version = "1.0", registry = "internal" }
```

The crates.io dependencies are most common. Git and path dependencies are useful for development before publishing.

### Features

Features are optional capabilities that a crate exposes. The crate author defines them; the user enables the ones they need:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

The `tokio` crate has many features. `"full"` enables all of them. For production code, you might enable only what you need:

```toml
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net"] }
```

This produces a smaller binary because unused features are not compiled.

If you do not want default features:

```toml
serde = { version = "1.0", default-features = false, features = ["derive"] }
```

This is useful when the default features pull in dependencies you do not want.

### Profiles

Cargo has different build profiles for different purposes. The most important are `dev` and `release`:

```toml
[profile.dev]
opt-level = 0           # no optimization (fast compile, slow runtime)
debug = true            # include debug info

[profile.release]
opt-level = 3           # full optimization (slow compile, fast runtime)
debug = false           # no debug info
lto = false             # link-time optimization disabled by default
```

`cargo build` uses the `dev` profile. `cargo build --release` uses the `release` profile. You can override settings:

```toml
[profile.release]
opt-level = 3
lto = true              # enable link-time optimization for smaller, faster binaries
codegen-units = 1       # one codegen unit (slower compile, faster runtime)
strip = true            # strip debug symbols from the final binary
panic = "abort"         # smaller binary; no unwinding on panic
```

These settings are common for production binaries where binary size and runtime speed matter more than compile time.

For libraries used by others, you typically do not customize profiles. The end binary's profile applies to your library code too.

### `Cargo.lock`

When Cargo builds, it produces a `Cargo.lock` file that records the exact versions of every dependency:

```toml
# Cargo.lock (auto-generated, simplified)
[[package]]
name = "serde"
version = "1.0.193"
checksum = "..."
```

The lock file ensures that everyone building the project gets the same dependency versions. This is critical for reproducibility.

The convention:

- For binaries (applications), commit `Cargo.lock`. This pins the dependency versions for reproducible builds.
- For libraries, do not commit `Cargo.lock`. The library does not control the dependency tree of its users; the lock file is meaningless for them.

---

## E. Semantic Versioning and Dependency Management

Rust's dependency system uses semantic versioning (semver) by convention. Understanding semver is essential for both consuming and publishing libraries.

### The Semver Format

A semver version has three numbers: `MAJOR.MINOR.PATCH`.

- **MAJOR**: incremented for breaking changes.
- **MINOR**: incremented for backward-compatible additions.
- **PATCH**: incremented for backward-compatible bug fixes.

For example:

- `1.0.0`: initial stable release.
- `1.0.1`: bug fix; safe to upgrade.
- `1.1.0`: new feature added; safe to upgrade.
- `2.0.0`: breaking change; not safe to upgrade automatically.

### Cargo's Compatibility Rule

The default version specification in Cargo (`"1.0"` or `"^1.0"`) means "any version that is compatible with 1.0":

- `1.0.5`: compatible (PATCH bump).
- `1.5.0`: compatible (MINOR bump, adds features).
- `2.0.0`: NOT compatible (MAJOR bump, breaking changes).

Cargo picks the latest compatible version when resolving dependencies. The lock file then pins that exact version.

For pre-1.0 versions (`0.x.y`), Cargo treats MINOR as the breaking version: `0.5.0` and `0.6.0` are considered incompatible. This reflects the convention that pre-1.0 software is allowed to break compatibility freely.

### Implications for Library Authors

If you publish a library, semver is a contract with your users:

- Bug fixes go in PATCH releases.
- New features go in MINOR releases.
- Breaking changes require a MAJOR release.

What counts as a breaking change in Rust:

- Removing or renaming any public item.
- Changing the signature of any public function or method.
- Adding a non-default trait bound.
- Removing a trait implementation that users might depend on.
- Many other subtler things.

Maintaining strict semver is hard, especially for ergonomic improvements that might technically break some user. The Rust community generally accepts that minor breakage during MINOR bumps is sometimes unavoidable, especially in pre-1.0 crates.

### Implications for Library Users

When you depend on a library at version `"1.0"`:

- Cargo automatically uses the latest 1.x version when you run `cargo update`.
- You should be able to upgrade within the 1.x range without changing your code.
- Upgrading to 2.x might require code changes; you opt in by changing the dependency line.

If a library breaks compatibility within a MINOR or PATCH release (which is rare but happens), the fix is to pin to the working version: `"=1.0.5"`.

### Updating Dependencies

`cargo update` updates dependencies to the latest compatible versions:

```bash
cargo update                    # update all dependencies
cargo update -p serde           # update only serde
cargo update -p serde --precise 1.0.130    # update to a specific version
```

The first form updates everything. The second updates only one dependency. The third pins to an exact version.

For production code, regular updates are good practice; patches often include security fixes and performance improvements. Major version updates require more care.

### Tools for Dependency Hygiene

Several tools help manage dependencies:

- **`cargo outdated`** (a third-party plugin): lists dependencies with newer versions available.
- **`cargo audit`** (a third-party plugin): checks dependencies against a database of known vulnerabilities.
- **`cargo tree`**: prints the dependency tree.

These are not part of `cargo` itself but are widely used in production projects.

---

## F. Cargo Workspaces for Multi-Crate Projects

Real-world projects often have multiple related crates: a library, a binary, several utilities, all sharing dependencies. Cargo workspaces are the structure for managing these.

### A Simple Workspace

A workspace is a directory with a `Cargo.toml` that declares its members:

```
my_project/
├── Cargo.toml
├── core/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
├── cli/
│   ├── Cargo.toml
│   └── src/
│       └── main.rs
└── server/
    ├── Cargo.toml
    └── src/
        └── main.rs
```

The top-level `Cargo.toml`:

```toml
[workspace]
members = ["core", "cli", "server"]
resolver = "2"
```

Each member crate has its own `Cargo.toml` and `src/` directory. The workspace coordinates them.

### Cross-Crate Dependencies

Members can depend on each other through path dependencies:

In `cli/Cargo.toml`:

```toml
[package]
name = "my_project_cli"
version = "0.1.0"
edition = "2024"

[dependencies]
my_project_core = { path = "../core" }
clap = "4"
```

The `cli` crate depends on `my_project_core` from the workspace. Changes to `core` are immediately visible to `cli` without publishing or updating versions.

### Shared Dependencies

The workspace can specify shared dependency versions:

```toml
[workspace]
members = ["core", "cli", "server"]
resolver = "2"

[workspace.dependencies]
serde = "1"
tokio = { version = "1", features = ["full"] }
```

Members reference these:

```toml
[dependencies]
serde = { workspace = true }
tokio = { workspace = true }
```

This ensures all crates use the same version of each dependency, avoiding the headache of mismatched versions across the project.

### Building and Testing the Workspace

From the workspace root:

```bash
cargo build                     # build all crates
cargo build -p cli              # build only the cli crate
cargo test                      # test all crates
cargo run -p cli                # run the cli binary
```

The workspace's target directory (`target/`) is shared across all crates. This avoids redundant compilation of common dependencies and significantly speeds up builds in multi-crate projects.

### When to Use a Workspace

Workspaces are the right choice when:

- A project has multiple binaries that share library code.
- A library is large and benefits from being split into focused sub-libraries.
- You want to develop multiple related crates together before publishing.

For very small projects, a single crate is simpler. The break-even point is roughly when you have one library plus more than one binary, or when a single library is approaching 10,000+ lines.

### A Realistic Example

A typical web service workspace might look like:

```
my_service/
├── Cargo.toml
├── domain/                     (data types and business logic)
├── api/                        (HTTP server)
├── worker/                     (background job processor)
├── cli/                        (admin and operations tools)
└── tests/                      (integration tests)
```

The `domain` crate is shared by all three executables. Each executable is a separate crate, allowing independent build configuration. Integration tests in the workspace `tests/` directory exercise the system end-to-end.

This structure scales well to large projects. Many Rust services are organized this way.

---

## G. Publishing Crates to crates.io

Crates.io is the central package registry for the Rust ecosystem. Publishing a library makes it available to all Rust developers.

### Preparing to Publish

Before publishing, ensure your `Cargo.toml` has:

```toml
[package]
name = "my_useful_lib"
version = "0.1.0"
edition = "2024"
authors = ["Your Name <you@example.com>"]
description = "A short, clear description of what this does"
license = "MIT OR Apache-2.0"           # standard for Rust projects
repository = "https://github.com/you/my_useful_lib"
documentation = "https://docs.rs/my_useful_lib"
readme = "README.md"
keywords = ["parsing", "config"]
categories = ["parser-implementations"]
```

The `description`, `license`, and either `repository` or `documentation` are required for publishing. The other fields help users find your crate.

The `name` must be unique across crates.io. The `version` follows semver. After publishing, you cannot republish the same version; bumping the version is the only way to publish updates.

### Publishing Workflow

The basic workflow:

1. **Create an account at crates.io.** Sign in with GitHub.
2. **Generate an API token.** crates.io provides one in your account settings.
3. **Login locally:** `cargo login <token>`.
4. **Publish:** `cargo publish`.

The `cargo publish` command:

- Verifies the manifest is complete.
- Ensures the working directory is clean (no uncommitted changes).
- Packages the source into a tarball.
- Uploads it to crates.io.
- Makes the crate available to other developers.

After publishing, the crate is discoverable at `https://crates.io/crates/my_useful_lib`. Documentation is auto-generated and hosted at `https://docs.rs/my_useful_lib`.

### Versioning Updates

To publish a new version:

1. Make your changes.
2. Bump the version in `Cargo.toml` according to semver.
3. Update the `CHANGELOG.md` (a common convention) describing what changed.
4. Run `cargo publish` again.

You cannot un-publish a version. You can yank a version (`cargo yank --version 1.0.5`), which prevents new projects from depending on it but does not break existing dependents.

### Best Practices for Published Crates

Some conventions:

- **Document everything public.** Every `pub` item should have a documentation comment (`///` above it). Documentation is shown on docs.rs.
- **Provide examples.** Either inline in doc comments or in `examples/` directory.
- **Write integration tests.** In `tests/`, separate from unit tests in `src/`.
- **Use semantic versioning seriously.** Breaking changes are MAJOR bumps.
- **Maintain a CHANGELOG.** Users want to know what changed between versions.
- **Provide both MIT and Apache-2.0 licenses.** This is the Rust community standard, allowing maximum compatibility.

The Rust community has high standards for crate quality. Well-documented, well-tested, well-maintained crates get adopted widely; poorly-documented ones do not.

### Private Registries

For internal company crates, you can run your own registry. Cargo supports custom registries through `Cargo.toml` configuration. The standard solutions are:

- **Cloud registries** like JFrog Artifactory, AWS CodeArtifact, or Cloudsmith.
- **Self-hosted** options like `kellnr` or the open-source `panamax` for crates.io mirroring.

Configuring private registries is beyond this module; the Cargo book has the details. For most learners and small-to-medium projects, crates.io is sufficient.

---

## H. Feature Flags with `cfg` and Cargo Features

Feature flags allow code to be conditionally compiled based on configuration. They serve two main purposes: optional functionality (Cargo features) and platform-specific code (`cfg` attributes).

### Cargo Features

A Cargo feature is a named flag that enables some functionality. Features are declared in `Cargo.toml`:

```toml
[features]
default = ["network"]
network = ["dep:reqwest"]
encryption = ["dep:rsa"]
serde = ["dep:serde"]

[dependencies]
reqwest = { version = "0.12", optional = true }
rsa = { version = "0.9", optional = true }
serde = { version = "1", optional = true }
```

This declares three features. Each feature can enable optional dependencies. The `default` feature is enabled unless the user opts out.

Users select features when depending on the crate:

```toml
[dependencies]
my_lib = { version = "1.0", features = ["encryption"] }

# Or to disable defaults:
my_lib = { version = "1.0", default-features = false, features = ["serde"] }
```

### Conditional Compilation with `#[cfg]`

Inside the crate, code can be conditionally included based on features:

```rust
#[cfg(feature = "network")]
pub mod network {
    pub fn fetch(url: &str) -> String {
        // implementation that uses reqwest
    }
}

#[cfg(feature = "encryption")]
pub mod encryption {
    pub fn encrypt(data: &[u8]) -> Vec<u8> {
        // implementation that uses rsa
    }
}
```

The `#[cfg(feature = "network")]` attribute means "only compile this when the `network` feature is enabled." If the user does not enable that feature, the entire `network` module is omitted from the build.

### Why Features?

Features let library users opt into functionality they need without paying for what they do not:

- **Smaller binaries.** Unused code is not compiled.
- **Fewer dependencies.** Optional dependencies are not pulled in unless the corresponding feature is enabled.
- **Faster compilation.** Less code to compile.
- **Modular APIs.** Users can build a custom subset of the library.

For example, the `serde` feature in many libraries adds `Serialize` and `Deserialize` implementations for the library's types. Users who do not need serialization save the compilation cost of serde.

### Platform-Specific Code

The `#[cfg]` attribute also handles platform differences:

```rust
#[cfg(target_os = "windows")]
fn platform_specific() {
    println!("running on Windows");
}

#[cfg(target_os = "linux")]
fn platform_specific() {
    println!("running on Linux");
}

#[cfg(any(target_os = "macos", target_os = "ios"))]
fn platform_specific() {
    println!("running on Apple OS");
}
```

The compiler picks the right implementation based on the target. Common `cfg` predicates:

- `target_os`: "windows", "linux", "macos", "android", etc.
- `target_arch`: "x86_64", "aarch64", "wasm32", etc.
- `target_pointer_width`: "32", "64".
- `debug_assertions`: enabled in debug builds.
- `test`: enabled when running `cargo test`.

You can combine them with `all`, `any`, and `not`:

```rust
#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
fn linux_x64_specific() { /* ... */ }

#[cfg(not(target_os = "windows"))]
fn unix_only() { /* ... */ }
```

### `cfg!` Macro for Runtime Checks

The `#[cfg]` attribute conditionally includes code at compile time. The `cfg!` macro returns a boolean at runtime:

```rust
fn main() {
    if cfg!(target_os = "windows") {
        println!("running on Windows");
    } else {
        println!("not on Windows");
    }
}
```

The compiler still resolves this at compile time (the `if` branch is constant-folded), but the syntax allows runtime-style code.

### Recommended Patterns

A few rules of thumb for features:

- **Keep the default feature small.** Users who want a quick start should get something useful with no configuration; users who want full control should opt in.
- **Document features clearly.** The crates.io page shows features; users need to understand what each one does.
- **Test feature combinations.** Common ones, at least. A library with three features has eight possible combinations; not all need testing, but the common ones do.
- **Avoid feature creep.** Too many features make the crate hard to understand. If features become numerous, consider splitting the crate.

For application code (not libraries), features are less common but useful for things like development-only code or platform-specific implementations. Most application code uses `cfg` directly without defining new features.

---

## Module Summary

- Modules organize code within a crate. `mod` declares them; `pub` controls visibility; `use` brings names into scope.
- Visibility is private by default. `pub`, `pub(crate)`, `pub(super)`, and `pub(in path)` give finer control over what is exposed at each level.
- Crates come in two kinds: binary (with `main`) and library (with public API). A package can have both.
- `Cargo.toml` describes the package: metadata, dependencies, features, profiles. Cargo automatically resolves dependencies and produces `Cargo.lock` for reproducibility.
- Semantic versioning is the convention for dependency versions. MAJOR for breaking, MINOR for additions, PATCH for fixes.
- Workspaces structure multi-crate projects. Crates within a workspace can depend on each other and share build artifacts.
- Crates are published to crates.io with `cargo publish`. Documentation is auto-generated at docs.rs.
- Features and `#[cfg]` enable conditional compilation, allowing optional functionality and platform-specific code.

## Discussion Questions

1. Rust defaults to private visibility, requiring `pub` to expose items. Many languages default to public. Identify two specific kinds of bug or maintenance problem that the default-private rule prevents. Are there cases where the default-public rule would be more convenient?
2. Cargo workspaces share a single `target/` directory across all member crates. This means a build of one crate can use cached artifacts from a previous build of a sibling crate. What does this design optimize for? What does it cost in terms of build correctness or isolation?
3. Cargo features and `#[cfg]` allow optional functionality. Identify a specific case where features make sense for a library, and a case where they would make the library too complicated. What signals tell you which side of the line you are on?

