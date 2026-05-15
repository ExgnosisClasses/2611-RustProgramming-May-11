# Module 17: Modules, Crates, and Workspaces
## Lab 13 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Modules in a Single File

### Checkpoints

**1. Why offer granular visibility levels rather than just public/private?**

The two-level model (public/private) is sufficient for simple programs, but it forces tough choices in larger codebases:

- An item must be either exposed to the whole world (public) or visible only within its module (private).
- "I want this visible to my library but not to library consumers" cannot be expressed.
- "I want this visible to the parent module only" cannot be expressed.

The result, in two-level languages, is workarounds:

- **Java's package-private** (the default) is similar to `pub(crate)`. But Java has no equivalent of `pub(super)`.
- **C++'s `friend`** declarations let one class access another's privates, but the mechanism is awkward and requires modifying the target.
- **Python's `_underscore` convention** is a community convention, not enforced. Tools "respect" it; the language does not.

Rust's granularity matches how programmers actually think about code visibility:

- **`pub`**: "I am providing this as part of my external API; commit to backward compatibility."
- **`pub(crate)`**: "Internal API; modules within my crate can use this, but I can change it without warning external users."
- **`pub(super)`**: "Just for the parent module; not part of the broader crate API."
- **Private**: "Implementation detail; do not depend on this."

Each level communicates intent and provides actual enforcement. A library maintainer can refactor freely behind a `pub(crate)` boundary; consumers of the library only see `pub`.

The cost is a slightly more complex mental model. The benefit is more honest API boundaries. For library code in particular, this enforcement is invaluable.

**2. What is the difference between a `use` declaration and inline path use?**

The two are functionally equivalent for finding the item; the difference is in how the code reads and where the items are documented.

```rust
// With use:
use std::collections::HashMap;
let m = HashMap::new();
let m2 = HashMap::new();
let m3 = HashMap::new();

// Without use:
let m = std::collections::HashMap::new();
let m2 = std::collections::HashMap::new();
let m3 = std::collections::HashMap::new();
```

The `use` declaration:

- **Reduces noise** at call sites. The full path is written once; the short name is used many times.
- **Documents dependencies** at the top of the file. A reader can see at a glance what items this file uses.
- **Makes refactoring easier.** Moving an item is a change to one `use` declaration; the call sites do not change.

The inline path:

- **Is explicit at each call site.** A reader does not need to scroll up to find where `HashMap` came from.
- **Avoids namespace pollution.** Many `use` declarations at the top of a file can hide name collisions or make it unclear what is in scope.
- **Is sometimes necessary** for disambiguation (when two items have the same name).

The general pattern: `use` for items used many times in the file. Inline path for items used once or twice, or where disambiguation matters.

For frequently-used items, `use` is almost universal. For one-time uses, inline is fine. Rust style guides do not strictly mandate either; project conventions vary.

**3. When to use `crate::` versus relative paths?**

The two cover different situations.

**`crate::`** (absolute, from the crate root):

- **When the relative path would be deep.** From `mod a::b::c`, getting to `mod x::y::z` via relative paths is awkward. The absolute path `crate::x::y::z` is clearer.
- **For consistency in large modules.** Some codebases use `crate::` everywhere for predictability: every path starts the same way.
- **When refactoring is expected.** If a module might move, absolute paths from `crate::` are less likely to break than relative paths.

**`super::`** (relative, parent module):

- **For one level up.** From `mod tasks::status`, getting to `mod tasks::` is just `super::`.
- **Less invasive.** If the parent module renames, code using `super::` continues to work.
- **More natural for tightly-coupled siblings.** When `status` and `priority` are both submodules of `tasks` and need to reference each other, `super::priority` is more readable than `crate::tasks::priority`.

The practical guidance:

- Use `super::` for one or two levels up. It is concise and resilient to module renames.
- Use `crate::` for cross-cutting references. When you are far from the destination, the absolute path is clearer.
- Use `self::` rarely. It is occasionally needed for disambiguation but usually adds nothing.

For most code, the difference is style rather than correctness. The compiler resolves both forms identically. Pick what reads best.

---

## Exercise 2 -- Splitting Modules Across Files

### Checkpoints

**1. Why was the modern convention adopted?**

The `mod.rs` convention had practical problems:

- **Many `mod.rs` files.** A project with 20 modules has 20 files named `mod.rs`. Opening one in an editor shows just `mod.rs`; you need the directory name to know what you are editing. This was confusing.
- **Hard to find files by name.** Searching for "tasks.rs" used to find the module's body; with the `mod.rs` convention, you searched for "mod.rs" and disambiguated by directory.
- **Editor tabs.** With multiple `mod.rs` files open, the tab labels were all "mod.rs"; the only differentiator was the title bar or tooltip.

The new convention solves these:

- **Each file has a meaningful name.** `tasks.rs` clearly contains the `tasks` module's body. No ambiguity.
- **Searches work intuitively.** Looking for the `tasks` module? Search for `tasks.rs`.
- **Editor tabs are clear.** Each tab shows the actual module name, not a generic `mod.rs`.

The cost of the new convention is one extra directory creation when a module gets submodules. Worth it for the readability gains.

The older convention is still allowed for backward compatibility. New projects should use the new convention; older projects you contribute to may use either.

**2. Why are mod declarations relative to the containing file?**

Several reasons:

- **Path is location-aware.** From `src/lib.rs`, `mod foo;` looks for `src/foo.rs` or `src/foo/mod.rs`. From `src/lib/utils.rs`, `mod helpers;` looks for `src/lib/utils/helpers.rs` (or the directory variant). The compiler knows where the `mod` declaration lives and searches relative to that.
- **Refactoring is local.** Moving a module to a new location does not require updating its inner module declarations. The relative search adapts.
- **Composable.** You can copy-paste a module subtree into a new location, and it works (as long as the relative structure is preserved).

If `mod` searches were always from the crate root, you would need full paths everywhere. From `src/lib/utils.rs`, you would write something like `mod lib::utils::helpers;` to declare a submodule. This is verbose and conflates the file path with the module path.

The relative-from-declaration rule keeps each file's view local. A module file declares its submodules without needing to know its position in the larger structure.

**3. Heavy `use` versus minimal `use`: trade-offs?**

**Heavy use (many `use` declarations at the top, short names at call sites):**

- **Pros:** Call sites are concise and read like business logic. Easier to scan the actual algorithm without paths cluttering it.
- **Cons:** Reader has to scroll to the top to learn where each name comes from. With many `use` declarations, the source can become non-obvious. Can introduce name collisions.

**Minimal use (full paths at call sites):**

- **Pros:** Every reference is explicit and self-documenting. No need to scroll. Reduces chance of name collisions or unclear imports.
- **Cons:** Code is more verbose. Long paths can dominate the source visually, making the actual logic harder to see.

In practice, the convention is "use for common items; inline path for rare or ambiguous items":

- Standard items (`HashMap`, `Vec`, common types) are usually used.
- Local items (from the same module) need no path; they are already in scope.
- Items from many places might be combined: `use std::{collections::HashMap, sync::Arc};`.
- Items that might be confusing (same name in two places) might be left as full paths for clarity.

A specific antipattern is `use module::*;` (wildcard imports). These bring everything from a module into scope, often unintentionally. They can hide what is actually in use, and can cause name collisions when items are added to the imported module. Most style guides discourage wildcards except for specific cases (test imports, prelude modules).

The right balance varies by codebase. Rust's official style is moderately heavy on `use`; community style guides agree. The key principle: be consistent within a codebase. Mixing styles randomly is harder to read than committing to one approach.

---

## Exercise 3 -- Binary vs. Library Crates

### Checkpoints

**1. What does the package name refer to versus the use statement?**

The `name` field in `Cargo.toml` names the **package** (a unit of distribution). A package can contain one or more crates.

When the package contains a library, that library's crate name defaults to the package name. When the package contains binaries, each binary's crate name defaults to either the package name (for the main binary in `src/main.rs`) or the file name (for binaries in `src/bin/`).

So `name = "taskforge"` in `Cargo.toml` produces:

- **One package** named `taskforge`.
- **One library crate** named `taskforge` (because `src/lib.rs` exists).
- **One binary crate** named `taskforge` (because `src/main.rs` exists).

When the binary uses `use taskforge::...`, it refers to the library crate (not itself). The binary cannot reference itself by name in `use` declarations; it just calls its own functions directly.

To customize crate names independently of package names, you can use the `[lib]` or `[[bin]]` sections:

```toml
[lib]
name = "tasklib"            # library's crate name (overrides package name)

[[bin]]
name = "task"               # binary's name
path = "src/main.rs"
```

For most projects, the defaults are fine. The library crate uses the package name (`use taskforge::...`), and the binary is invoked by its file-derived name (`cargo run --bin taskforge`).

The dash-to-underscore translation also applies: package name `task-forge` becomes crate name `task_forge` (Rust identifiers cannot contain dashes).

**2. What did extracting the library gain? What did it cost?**

**Gains:**

- **Reusability.** The library can be used by other binaries (within the package or in other packages).
- **Testability.** Integration tests in `tests/` can use the library; they could not access internals of a binary-only project.
- **Multiple binaries.** The same package can have many executables sharing the library.
- **Better documentation.** `cargo doc` generates docs for the library's public API; binaries do not generate user-facing docs.
- **Cleaner separation.** Application code (in the binary) is distinct from reusable code (in the library). The binary becomes a thin entry point.

**Costs:**

- **Slightly more setup.** Two files (`main.rs` and `lib.rs`) instead of one. Two `pub` levels (public to crate, public to consumers) to think about.
- **Slightly longer compile time.** Building both a library and a binary takes marginally more time than building one.
- **More to learn.** A new developer encountering the project must understand the library/binary distinction.

For any project that might grow beyond a quick script, the library+binary structure is worth it. The flexibility for testing alone justifies the small overhead.

For one-shot programs (a single script that you will run a few times and discard), keeping everything in `main.rs` is fine. The library structure is overkill for genuinely throwaway code.

The transition from "binary only" to "library + binary" is mechanical: create `lib.rs`, move the modules from `main.rs`, change `mod` declarations to `use` declarations, fix any references. It can be done at any time. Many real projects start as binary-only and move to library-style after a few weeks.

**3. What belongs in the binary, not the library?**

The binary should be thin. Generally:

- **Command-line argument parsing.** The binary handles `argv`; the library does not know about it.
- **Logging configuration.** The binary sets up loggers; the library uses them (or accepts them as a parameter).
- **The runtime entry point.** Async runtimes, thread pool initialization, signal handlers.
- **Top-level error handling.** The binary's `main` decides what to do when an error escapes the library (exit with a code, print to stderr, etc.).
- **Configuration loading.** The binary reads config files; the library accepts the configuration as parameters.
- **Process-level concerns.** PID files, daemonization, OS signal handling.

The library should not know:

- Whether it is being used by a CLI, a web server, or a test.
- How its output will be presented (logs, files, network).
- The user's environment (working directory, env vars).

The library's API takes input as parameters and returns output as values or errors. The binary translates between this API and the surrounding world (filesystem, network, terminal).

This separation is what allows the library to be reused. If the library accepted command-line arguments directly, it could be used only as a CLI. With the separation, the same library can power a CLI, a web service, a desktop app, and the test suite, each providing the runtime-specific glue.

The "thin binary" pattern also makes testing easier. The library is unit-testable; the binary is so small that integration testing covers it. Test logic in the library, not in the binary.

---

## Exercise 4 -- Cargo.toml: Dependencies, Features, and Profiles

### Checkpoints

**1. Why not pin to exact versions?**

The flexibility of version requirements (`"1"` matching any 1.x) gives you several benefits:

- **Automatic security patches.** A patch release (1.0.5 → 1.0.6) usually includes bug fixes. Pinning to 1.0.5 means you do not get those fixes; the flexible requirement lets `cargo update` pick them up.
- **Diamond dependency resolution.** If your project depends on A (which wants serde 1.0) and B (which wants serde 1.2), Cargo can pick a single serde version compatible with both. With exact pins, the two requirements might conflict.
- **Smaller `Cargo.lock`.** With flexible requirements, multiple crates can share a single dependency version. With exact pins, you might get multiple copies of the same dependency at slightly different versions.

The pin is in `Cargo.lock`, not `Cargo.toml`. `Cargo.lock` records what was actually selected; the next `cargo update` is what changes it. This gives you reproducibility (the lock is committed) without rigidity (you can update when you want).

When IS exact pinning appropriate?

- **Build reproducibility in a specific moment.** `Cargo.lock` does this for you; pinning in `Cargo.toml` is overkill.
- **Avoiding a specific known-bad version.** You can use `>= 1.2.3, < 1.2.5` to skip 1.2.4 if it has bugs.
- **Compliance requirements.** Some environments require exact pinning for auditing. In these cases, `=1.2.3` in `Cargo.toml` does what is needed.

For most projects, flexible requirements in `Cargo.toml` plus a committed `Cargo.lock` is the right combination.

**2. Why not always use release optimization?**

The dev profile is fast to compile; the release profile produces fast binaries. They optimize for different things.

**Dev profile (`opt-level = 0`):**

- Compile time: fast (incremental compilation rebuilds quickly).
- Runtime: slower (no optimization).
- Debug info: included (debuggers work).
- Binary size: larger (no stripping).

**Release profile (`opt-level = 3`):**

- Compile time: slow (every change triggers re-optimization).
- Runtime: fast (full optimization, possibly LTO).
- Debug info: stripped (smaller binary, less debug capability).
- Binary size: smaller (with `strip = true`).

During development, you want fast iteration. You change code, rebuild, run tests, change again. Optimizing each rebuild would make the cycle painful. The dev profile sacrifices runtime speed for fast feedback.

For deployment, you want fast binaries. The slower compile is paid once (during release); the runtime gains are paid back every time the program runs.

Real numbers vary, but typical patterns:

- Dev compile: 5 seconds (warm cache).
- Release compile: 60 seconds (warm cache); first build can be much longer.
- Dev runtime: maybe 3-5x slower than release for compute-heavy code.

The trade-off makes sense once you internalize it. Most development happens in the dev profile; release is for the final artifact.

**3. Why commit `Cargo.lock` for binaries but not libraries?**

For binaries, `Cargo.lock` is what makes your build reproducible. Every developer building the project gets exactly the same dependency versions. Every CI run uses the same versions. Production binaries are built from the same lock file as your tests.

For libraries, your `Cargo.lock` is irrelevant to consumers. When someone uses your library as a dependency, their `Cargo.toml` is what matters, and Cargo resolves dependencies based on their requirements (including your library's requirements). Your library's `Cargo.lock` is used only when building your library standalone (typically for your tests). Committing it would just create merge conflicts; consumers would not benefit.

In effect:

- **Binary `Cargo.lock`:** "These are the exact versions for this binary; everyone should use these."
- **Library `Cargo.lock`:** "These are the versions I happen to use for testing; not relevant to consumers."

For libraries that also produce binaries (like the `taskforge` package in this lab), there is a question. The convention is: if the binary is meaningful (used in production), commit the lock; if the binary is just a demonstration, do not. For libraries with very minor binaries, opinions vary.

A practical rule: when `cargo install <yourcrate>` is a common workflow, commit `Cargo.lock`. Users installing your crate as a binary get the locked versions, ensuring the installation behaves identically across machines. When the binaries are just internal tools, the lock is optional.

---

## Exercise 5 -- Cargo Workspaces

### Checkpoints

**1. The workspace `Cargo.toml` has no `[package]`. What does this say about workspaces?**

A workspace is metadata, not a crate. It lists members and provides shared configuration, but it does not produce build output of its own.

You cannot:

- Have source code in the workspace root (no `src/` at the top level).
- Have a binary or library named after the workspace.
- `use` items from the workspace itself.

You can:

- Build any member crate from the workspace root.
- Run any member binary from the workspace root.
- Define dependencies once at the workspace level (for sharing across members).
- Configure the workspace's `target/` directory and `Cargo.lock`.

This matches how multi-crate projects work in practice. The workspace is an organizational structure; the actual code lives in member crates. The top-level directory is just a container.

Some workspaces do have a top-level binary or library crate (with both a `[workspace]` and a `[package]` section). This is fine and common; it just means the workspace root is also a member crate. The main rule is "workspaces themselves do not contain code; members do."

**2. Path dependencies versus version dependencies?**

**Path dependencies:**

- Point to a crate in a relative directory.
- Used for crates that are not (or not yet) published.
- Common within a workspace where members depend on each other.
- Cannot be published to crates.io as path-only (the path is local to the developer).

**Version dependencies:**

- Point to a crate on crates.io.
- Used for published crates from the ecosystem.
- Required for publication: every dependency of a published crate must have a version specifier (a `path` is not enough).

You can combine them: `taskforge-core = { path = "../taskforge-core", version = "0.1" }`. When building locally (in the workspace), the path is used. When the crate is published, the path is irrelevant; the version dependency is what crates.io respects.

For workspaces with crates that will be published independently, the combined approach is standard:

```toml
[dependencies]
my-shared-lib = { path = "../my-shared-lib", version = "0.5" }
```

This lets you develop and test locally (path) while preserving the ability to publish (version).

For crates that will never be published (internal tools, applications), path-only is fine and simpler.

**3. What does workspace-level dependency declaration give you?**

For a workspace with many members using the same dependencies:

- **Single source of truth for version.** Updating `serde` from 1.0 to 1.2 changes one line in the workspace root. Without this, you would update every member crate's `Cargo.toml`.
- **Guaranteed version consistency.** All members use the same `serde` version. There is no risk of accidentally upgrading one crate's dependency and getting a version mismatch.
- **Easier maintenance.** New members can just say `serde = { workspace = true }`; they get the same version as everyone else.

For a workspace with 20 crates all using `serde`, the workspace dependency mechanism is a big win. Without it, you would need to coordinate updates across all 20 manually.

The cost is minor: each member crate has a slightly different syntax for declaring dependencies (`{ workspace = true }` instead of a version requirement). After a brief learning curve, it is natural.

For workspaces with only 2-3 members, the workspace dependency mechanism is convenient but not essential. For larger workspaces, it becomes essential. Real Rust projects (the compiler itself, large web frameworks) use it routinely.

---

## Exercise 6 -- Feature Flags

### Checkpoints

**1. Optional dependencies versus always-included?**

Without features, every dependency you list is downloaded, compiled, and linked into the binary. For libraries that support multiple optional backends (JSON, YAML, TOML; or HTTP, gRPC, WebSocket), this would mean:

- Every consumer downloads every backend's dependencies.
- Every consumer compiles every backend.
- Every consumer's binary contains every backend, even if they use only one.

This is wasteful. A user who only needs JSON does not benefit from YAML and TOML support; they just pay for it.

With feature flags:

- The default features are minimal (or whatever the common case is).
- Users opt into the backends they need.
- The compile cost and binary size match what is actually used.

For libraries with many optional integrations, this is essential. The `tokio` ecosystem is a good example: `tokio` itself has dozens of features (`net`, `fs`, `signal`, `sync`, etc.) and users enable only what they use. A simple program using `tokio` to run async tasks might enable just `rt` and `macros`; a full-featured server enables `full`.

The principle: pay only for what you use. Features make this explicit.

**2. Compile-time feature flags versus runtime configuration?**

Both can control behavior; they differ in when the decision is made.

**Compile-time (features):**

- Decided when the binary is built.
- Excluded code is not compiled (zero runtime overhead).
- Cannot change without rebuilding.
- Reduces binary size when features are off.

**Runtime configuration:**

- Decided when the program runs.
- All code is compiled; the runtime checks decide what executes.
- Can change between runs (or even during a run).
- All code is in the binary regardless.

Use compile-time when:

- The decision will not change across runs (e.g., "this build is for embedded systems; no debug logging").
- The feature has its own dependencies (no point compiling them if not used).
- Binary size matters (mobile, embedded, distribution).

Use runtime when:

- The decision should be user-configurable.
- The feature might be turned on or off based on operational concerns.
- The cost of always-on code is acceptable.

For the `verbose-logs` example in the lab, runtime configuration would also work: a `--verbose` CLI flag, or a `LOG_LEVEL` env var. The compile-time approach is cleaner if the logging is meant only for developers (and never enabled in production); the runtime approach is better if users might want to enable it themselves.

In practice, many production systems use both. Compile-time features control "what the binary can do"; runtime configuration controls "what the binary actually does in this run."

**3. Default features: trade-offs?**

**Heavy default features:**

- **Pros:** New users get a working setup immediately. The crate "just works" out of the box.
- **Cons:** Users who do not need the features still pay for them. Binary size and compile time are larger by default. Opting out is extra work.

**Minimal default features:**

- **Pros:** Smallest baseline. Users opt into what they need.
- **Cons:** New users may be confused by initial limitations. "Why does this not work? Oh, I need to enable a feature."

The right balance:

- **Default features should cover the most common use case.** If 90% of users want JSON support, enable it by default.
- **Default features should NOT include heavy or controversial dependencies.** A library that uses tokio internally should NOT enable tokio by default (or it forces every user into tokio).
- **Default features should be additive only.** Enabling a default feature should not break code; disabling it should leave a working (if limited) library.

Real examples:

- **serde's default features** include `std` (which makes serde usable; without it, you get `no_std` for embedded). This default is right because almost everyone wants `std`.
- **tokio's default features** are minimal; users enable what they need. This is right because the feature set is large and varied.
- **reqwest's default features** include `default-tls` (HTTPS support). Right because most users want HTTPS.

For your own libraries, default to the common case. Document carefully what is included so users can opt out if needed.

---

## Exercise 7 -- Preparing for Publication

### Checkpoints

**1. Why is metadata important even though it does not affect compilation?**

Metadata is what makes the crate discoverable and usable:

- **`description`** appears in crates.io search results. Without it, users see the name but not what the crate does.
- **`keywords`** and **`categories`** make the crate findable by users browsing by topic.
- **`repository`** lets users see the source, file issues, contribute. Without it, the crate feels unmaintained.
- **`license`** is legally required. Crates without explicit licenses cannot legally be used by others (default copyright applies).
- **`readme`** is what crates.io shows as the front page. Without it, the crate's page is bare.

For publication, the metadata is the difference between a crate that gets used and one that does not. Users browsing crates.io evaluate quickly based on the metadata; a crate without a description or repository link is usually skipped.

Cargo requires several fields for publication specifically because the crate is going to be public. The requirements force authors to provide enough information for users to evaluate the crate.

For unpublished crates (internal libraries, workspace members), the metadata still helps:

- Future maintainers know what the crate does without reading the code.
- IDE tools can show the description in popups.
- Generated docs include the metadata.

Time spent on good metadata pays off every time someone encounters the crate.

**2. Why `publish = false` even for crates with no publication plans?**

It is a safety net. The cost of an accidental publication is high:

- **Cannot un-publish.** Once a version is on crates.io, it stays forever (you can only "yank" it, which signals "do not use" but does not remove it).
- **Names are reserved.** Publishing a crate reserves the name. If you accidentally publish `my-internal-tool`, that name is taken; nobody else can use it.
- **Secrets might leak.** Internal crates sometimes include things you do not want public: hardcoded credentials in tests, internal URLs, references to private infrastructure.

`publish = false` prevents these accidents. `cargo publish` refuses to publish the crate. The intent is documented in the manifest.

The pattern is cheap (one line of TOML) and significantly reduces risk. The convention in many Rust workspaces: every crate that is not explicitly meant for publication has `publish = false`.

For workspace members, applying `publish = false` is especially useful: workspaces typically have a mix of crates (some intended for publication, some internal). Marking each explicitly prevents confusion.

A related field is `[workspace.package]` which can set defaults for all workspace members. Setting `publish = false` at the workspace level (and overriding to true on the published members) is one possible pattern.

**3. What information is in the README that should not also be in doc comments?**

Some overlap is expected, but the README and doc comments serve different purposes:

**README (high-level, front-page):**

- What the crate is and what problem it solves.
- A short usage example showing the most common case.
- Major feature flags and what they do.
- License and contact information.
- Installation/quick start.
- Links to deeper documentation.

**Doc comments (per-item, reference):**

- Specifics about each function, type, or module.
- API contracts (what does this do, what does it return, when does it panic).
- Examples for specific items.
- Internal links between items.

The README is for someone deciding whether to use the crate. The doc comments are for someone already using it.

A common mistake is to make the README a tour of every API item, duplicating the doc comments. This is wasteful (the docs are already comprehensive) and quickly goes out of date. The README should orient and motivate, not exhaustively document.

A useful pattern: the README has 2-3 representative examples (the "hello world" and the most common scenarios); the docs cover everything systematically.

For libraries on docs.rs, the README appears on crates.io and the docs appear on docs.rs. Users navigate from one to the other as needed.

---

## Exercise 8 -- Putting It Together

### Task 8.5 -- Pattern identification

1. **Package with both binary and library:** `taskforge-core` has both `src/main.rs` (binary) and `src/lib.rs` (library).

2. **Secondary binary in `src/bin/`:** `taskforge-core/src/bin/admin.rs` produces a binary named `admin`.

3. **`pub(crate)` for crate-only visibility:** (Not actually used in the final workspace, but demonstrated in Exercise 1, Task 1.3.)

4. **Module split across files:** `taskforge-core/src/tasks.rs` declares submodules; the submodules are in `taskforge-core/src/tasks/status.rs` and `taskforge-core/src/tasks/priority.rs`.

5. **Path dependency between workspace members:** `taskforge-cli/Cargo.toml` has `taskforge-core = { path = "../taskforge-core", ... }`.

6. **Workspace-level dependency:** The root `Cargo.toml` has `[workspace.dependencies] serde = ...`. Members reference it with `serde = { workspace = true }`.

7. **Optional dependency behind a feature:** `taskforge-core/Cargo.toml` has `serde_json = { version = "1", optional = true }` paired with `json = ["dep:serde_json"]`.

8. **Boolean feature flag:** `verbose-logs = []` in `taskforge-core/Cargo.toml`. Toggles code via `#[cfg(feature = "verbose-logs")]` without adding any dependencies.

9. **`publish = false`:** `taskforge-cli/Cargo.toml` and `taskforge-report/Cargo.toml` both have this.

10. **Dash-to-underscore translation:** Cargo name `taskforge-core` becomes Rust crate name `taskforge_core`. Used in `use taskforge_core::tasks::...` statements throughout the binaries.

### Task 8.6 -- Predict and verify

**Change A:** Remove `features = ["json"]` from `taskforge-cli`'s dependency on `taskforge-core`.

**Prediction:** The `to_json` function is gated behind the `json` feature. Without the feature, the function does not exist (it is not compiled). The use statement `use taskforge_core::tasks::to_json;` will fail because the imported item does not exist. The build fails with an error like "no `to_json` in `tasks`."

**Result:** As predicted:

```
error[E0432]: unresolved import `taskforge_core::tasks::to_json`
  --> taskforge-cli/src/main.rs:2:50
   |
 2 | use taskforge_core::tasks::{new_task, status, to_json};
   |                                                ^^^^^^^ no `to_json` in `tasks`
```

The error is at compile time. The compiler does not know about the `to_json` function because the feature that enables it is not active. This is the value of conditional compilation: code that is not in scope is not just missing at runtime; it does not exist at all.

The fix is to either re-enable the feature, or to remove the use of `to_json`.

The lesson: feature flags partition the API. Different feature combinations produce different available APIs. Documentation should make clear which features are needed for which APIs.

**Change B:** Change `taskforge-core = { path = "../taskforge-core" }` to `taskforge-core = "0.1"`.

**Prediction:** Cargo will try to find `taskforge-core` version 0.1 on crates.io. Since you have not published it, Cargo will fail to find it. The error will mention something like "no matching package found for `taskforge-core`."

**Result:** As predicted:

```
error: failed to load source for dependency `taskforge-core`

Caused by:
  Unable to update registry `crates-io`

Caused by:
  no matching package named `taskforge-core` found
  location searched: registry `crates-io`
  required by package `taskforge-report v0.1.0 (...)`
```

Cargo searches crates.io for any package named `taskforge-core` at version 0.1, finds none, and fails.

The lesson: workspace path dependencies are local. To use a version dependency, the crate must actually be published to a registry. The combined form (`{ path = "...", version = "..." }`) lets you have both: local development with paths, publishability with versions.

**Change C:** Add `fn internal_helper()` (private) to `lib.rs`, then change to `pub(crate)`.

**Prediction:** With private visibility, the CLI cannot see `internal_helper`. The build fails with a privacy error.

Changing to `pub(crate)` makes it visible within `taskforge-core` but NOT to consumers (like `taskforge-cli`). The CLI is in a different crate, so the build still fails.

**Result:** As predicted in both cases.

With private:
```
error[E0603]: function `internal_helper` is private
```

With `pub(crate)`:
```
error[E0603]: function `internal_helper` is crate-private
```

Both errors are at compile time. The CLI cannot access the function in either case.

To make it visible to the CLI, the function must be `pub` (truly public).

The lesson: `pub(crate)` is genuinely restricted to within a single crate. A workspace's separate crates each have their own crate scope; `pub(crate)` does not leak between them. This is what enables clean library design: you can have helpers that are visible internally without exposing them to external consumers (including other crates in the same workspace).

### Checkpoints

**1. What does the shared `Cargo.lock` prevent?**

Without a shared lock file, each member crate could resolve dependencies independently. This means:

- **Version drift.** If `taskforge-core` resolves `serde` to 1.0.150 and `taskforge-report` resolves it to 1.0.196, the workspace has two different serde versions in use. The compiler can sometimes handle this, but it produces larger binaries and longer compile times.
- **Type incompatibility.** Types from different versions of the same crate are NOT compatible. A `serde::Serialize` from 1.0.150 is a different trait than from 1.0.196. Code that needs to pass values between crates fails to compile.
- **Inconsistent updates.** Updating dependencies in one crate but not another can produce subtle bugs.

With a shared lock file:

- Every member crate uses the exact same version of every dependency.
- Types from one member can flow into another member without version issues.
- `cargo update` updates everything together.

For workspaces with many shared dependencies (the common case), this is essential. The shared lock is a feature, not just a convenience.

The same applies to non-workspace setups. When you depend on multiple crates that depend on each other, Cargo prefers to resolve them at a single compatible version rather than multiple incompatible versions. The workspace's shared lock makes this explicit.

**2. Why does feature-set granularity per dependency matter?**

Two reasons:

- **Different consumers need different things.** `taskforge-cli` needs JSON output; another consumer might need CSV but not JSON. The features each consumer enables should not affect the others.
- **Build cost depends on enabled features.** Enabling more features compiles more code. Consumers who do not need a feature should not pay its compile cost.

If a dependency offered "all or nothing" (no features), every consumer would have to take all the bells and whistles. The unused parts of the dependency would still be compiled in.

With granular features, each consumer enables only what it needs. The library is built once per unique feature combination; consumers with the same features share the build. Consumers with different features can still coexist.

For library design, this means: think about what is core (always available) versus what is optional (gated behind features). Make every reasonable subset of features valid (do not have features that depend on each other implicitly). Document carefully which features are needed for what.

Real-world example: `reqwest` has features for `blocking` (sync API), `json` (JSON support), `cookies` (cookie handling), TLS backends (`rustls-tls`, `native-tls`), and so on. A web scraping script might enable `blocking + json`; a server might enable async + `rustls-tls`. The same library serves both with appropriate compile-time customization.

**3. Why are crate names underscored in Rust paths but dashed in Cargo names?**

The reason is straightforward: Rust identifiers cannot contain dashes (`-`). Identifiers are restricted to letters, digits, and underscores. A name like `taskforge-core` is not a valid Rust identifier.

So Cargo translates names to Rust-compatible form:

- Cargo name `taskforge-core` → Rust crate name `taskforge_core`.
- Cargo name `tokio-tungstenite` → Rust crate name `tokio_tungstenite`.
- Cargo name `serde_json` (already has underscore) → Rust crate name `serde_json`.

The translation is automatic; you do not need to configure it.

Why use dashes in Cargo names? Convention. Cargo names are user-facing (URLs, command-line, package indexes). Dashes are the convention for hyphenated names in URL-like contexts (CSS classes, npm packages, Linux package names). Underscores are the convention for identifiers in code.

The translation lets you have both: hyphens where they read naturally (`taskforge-core` as a package name) and underscores where the syntax requires (`taskforge_core` as an identifier).

A few notes:

- The translation is one-way: dashes become underscores. Underscores in the Cargo name stay as underscores. You cannot have a Cargo name with both dashes and underscores that would create ambiguity.
- The translation applies to library and binary names, but binaries are usually invoked via their Cargo names (`cargo run --bin taskforge-cli`). The dash version is what you type at the command line.
- For one-word names (`tokio`, `serde`), the question does not arise.

When choosing names for your own crates: use dashes if the name is multi-word. The Rust convention is `kebab-case` for Cargo names; the translation to `snake_case` for identifiers is automatic.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

Common patterns:

- Developers from Java find the module system familiar in some ways (packages map roughly to modules) but unusual in others. The file-to-module correspondence rules are stricter than Java's free-form package layouts.
- Developers from Python find the explicit `mod` declarations unusual. Python's "import does the searching" feels lighter; Rust's "declare and the compiler finds" feels heavier. The trade-off becomes clearer with time: explicit declarations make module structure obvious, while Python's discovery can hide where things come from.
- Developers from C++ find Cargo's package and dependency management refreshingly simple. CMake or Make-based dependency management requires significantly more effort. Workspaces feel natural; they resemble multi-project CMake setups but with much less ceremony.
- Developers from JavaScript find npm-like dependency resolution familiar. The semantic versioning rules are similar. The big difference is the workspace concept; npm has it as `workspaces` but it is less central to the experience.

The most common reflection: "The visibility levels took the longest to internalize. I am used to public/private only; `pub(crate)` and `pub(super)` felt like over-engineering at first. After working on a multi-crate project, I see why they exist: they let you have honest API boundaries that the compiler enforces, instead of relying on conventions or documentation."

Another common reflection: "Workspaces felt natural once I understood them. Coming from monorepo experience in other languages, the structure made sense. The single `Cargo.lock` and `target/` is exactly what I wanted; not having to configure it (no `lerna` equivalent needed) is a relief."

**2. Signs to split a crate or extract a workspace.**

Sample answer:

"Signs to split a single crate into multiple crates within a workspace:

- **The crate exceeds 10,000 lines or 50 files.** At this size, incremental compilation gets slow because Cargo rebuilds the entire crate on any change. Splitting allows parallel and incremental builds.
- **Multiple binaries need shared code.** If you have one binary now but plan a second (and they share substantial logic), extracting a library is the next step. Doing it before the second binary exists makes the transition easier.
- **You want to test the public API as a consumer would.** Integration tests in `tests/` work, but for genuinely external testing, a separate consumer crate verifies that your library is usable from outside.
- **You see natural boundaries forming.** A 'core' module that does not depend on anything specific; a 'CLI' module that handles command-line concerns; a 'web' module that adds HTTP. If these layers exist in your code, formal separation makes them explicit.
- **You want to release parts independently.** A library that others might want without the CLI. The library can be published while the CLI stays internal.

Signs to extract a workspace from what was a single project:

- **You have multiple binaries that share substantial code.** This is the trigger. Once you have several binaries (CLI, server, admin tool, migration tool), a workspace organizes them naturally.
- **You have multiple libraries.** When the codebase has more than one library (perhaps with different release schedules or audiences), separate crates are the right structure.
- **You want different feature/dependency profiles for different parts.** A workspace lets each member have its own feature flags and dependencies.

Signs to NOT split:

- **The codebase is small and unlikely to grow.** A single crate is simpler than a workspace.
- **All code is tightly coupled.** Forcing modularity can produce arbitrary boundaries that create more friction than they solve.
- **Your team is small and the crate is short-lived.** Workspaces are an investment; they pay off over time, but a project that will be done in two weeks does not benefit.

The general principle: start simple, refactor toward more structure when the signs appear. Premature workspace creation creates complexity for hypothetical benefits. Reactive workspace creation matches structure to actual needs."

**3. Strictness of Rust's module system: gains and costs.**

Sample answer:

"What Rust's strict module system gives you:

- **No surprises in what is exported.** Every public item is explicitly marked with `pub`. There is no auto-export, no 'all symbols are public by default,' no implicit re-exports. Reading a module's surface tells you exactly what is public.
- **Refactoring confidence.** When you move an item or rename it, the compiler catches every consequence. With weaker module systems (Python's, JavaScript's), you often discover broken references at runtime.
- **Enforceable encapsulation.** The visibility levels let you say 'this is internal to my crate' or 'this is internal to my module' and have the compiler enforce it. With convention-based systems (Python's underscore prefix), the rules are aspirational; the compiler does not help.
- **Clear file-to-module mapping.** The file system structure tells you the module hierarchy. You can navigate by directory and find what you need; no need for a separate index of which file contains which symbols.

What it costs:

- **More boilerplate.** Every module needs `mod` declarations in its parent. Every public item needs `pub`. Every cross-module reference needs explicit paths or `use` statements.
- **Less flexibility.** You cannot have circular dependencies between modules (which is sometimes inconvenient). You cannot dynamically add items to modules. You cannot construct module structures programmatically.
- **A steeper learning curve.** New Rust programmers spend real time understanding visibility levels, paths, and file conventions. In a language like Python, you write a function in a file, import it, and move on.

For a personal script or a small experimental project, the cost might outweigh the benefit. For a library that other people will use, or a long-lived application that will evolve, the strictness pays off. The compiler-enforced module boundaries prevent the gradual decay that affects projects with weaker module systems.

I have seen Python projects where, over years, what was originally a clean module hierarchy became a tangled web of cross-imports and unclear dependencies. The lack of enforcement let the structure degrade. Rust's strictness prevents this: if your module system would degrade, the compiler stops compilation. You either accept the structure (and refactor to make it clean) or live without the change.

The strictness is a forcing function for good design. That is its real value, beyond any specific feature it enables."

---

*End of Lab 13 Solutions*
