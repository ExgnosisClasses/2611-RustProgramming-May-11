# Module 19: Unsafe Rust
## Lab 15 -- Raw Pointers, FFI, Unsafe Traits, and Safe Abstractions

> **Course:** Mastering Rust
> **Module:** 19 - Unsafe Rust
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with every concept from Module 19: the five things `unsafe` enables, working with raw pointers `*const T` and `*mut T`, calling C functions through FFI, implementing unsafe traits, building safe abstractions over unsafe code, and applying the practical guidelines for writing and auditing unsafe blocks.

You will build a single Cargo project called `unsafe_explorer` that progressively explores each unsafe capability. The project starts with simple raw pointer manipulation, adds FFI calls to the C standard library, demonstrates the `Send`/`Sync` unsafe traits, and ends with an integration exercise that builds a safe wrapper around unsafe code. The lab is deliberately more cautionary than aspirational: most exercises are short, focused on understanding rather than extensive practice.

The focus is on judgment: when unsafe is genuinely needed versus when it is an attempt to bypass legitimate compiler complaints, how to recognize unsafe in dependencies, how to write SAFETY comments that justify each unsafe block. Many decisions have multiple defensible answers; the lab guides you toward idiomatic Rust and asks you to compare alternatives.

By the end of this lab you will be able to:

- Create and dereference raw pointers (`*const T`, `*mut T`).
- Recognize the five superpowers that `unsafe` unlocks.
- Call C functions from Rust using `extern "C"` blocks.
- Use `CString` and `CStr` to bridge Rust strings and C strings.
- Write `unsafe impl Send for T` and `unsafe impl Sync for T` when needed.
- Build safe abstractions that encapsulate unsafe code.
- Write proper SAFETY comments justifying each unsafe block.
- Recognize when reaching for unsafe is wrong, and find a safer alternative.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** This lab uses the `libc` crate for FFI examples. It will be added through `Cargo.toml`, and Cargo will download it on the first build. Make sure you have an internet connection when you reach Exercise 3.

> **A note about undefined behavior.** Some tasks in this lab deliberately demonstrate what NOT to do. These appear as commented-out code or as explanations of what would go wrong if you violated an invariant. Do not uncomment these blocks. Undefined behavior is genuinely undefined: the program might appear to work, then fail catastrophically later, often in completely unrelated parts of the code. The lab points these out as teaching moments, not as exercises to perform.

> **A note about the lab's scope.** Most production Rust code contains little or no unsafe. The skills in this lab are for recognizing and reading unsafe code in libraries you depend on, and for the rare cases where you need to write a small amount yourself. The lab does NOT teach you to write substantial unsafe code; that requires deeper study (see the Rustonomicon).

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new unsafe_explorer
cd unsafe_explorer
code .
```

Create a `lab15-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Raw Pointers Basics

**Estimated time:** 15 minutes
**Topics covered:** creating `*const T` and `*mut T`, dereferencing, the safety boundary

### Context

Raw pointers are the most common ingredient in unsafe code. They are similar to references but with weaker guarantees. This exercise establishes the basic mechanics: creating raw pointers (safe), dereferencing them (unsafe), and understanding the contract.

### Task 1.1 -- Creating raw pointers

Replace the contents of `src/main.rs` with:

```rust
fn main() {
    let x = 42;

    // Creating raw pointers is safe:
    let const_ptr: *const i32 = &x;
    let mut_ptr: *mut i32 = &x as *const i32 as *mut i32;

    println!("const_ptr: {const_ptr:?}");
    println!("mut_ptr: {mut_ptr:?}");

    // You can have multiple pointers to the same data:
    let another_const: *const i32 = &x;
    let yet_another: *const i32 = const_ptr;

    println!("another_const: {another_const:?}");
    println!("yet_another: {yet_another:?}");
}
```

Run the program:

```bash
cargo run
```

Expected output (the actual addresses will vary):

```
const_ptr: 0x7ffe1234abcd
mut_ptr: 0x7ffe1234abcd
another_const: 0x7ffe1234abcd
yet_another: 0x7ffe1234abcd
```

Three things to notice:

- Creating raw pointers does not require `unsafe`. The pointer values themselves are just addresses; computing them is harmless.
- You can have multiple pointers to the same data, including a mix of `*const` and `*mut`, without the compiler complaining. The compiler does not track raw pointers the way it tracks references.
- The cast `&x as *const i32 as *mut i32` converts an immutable reference into a mutable raw pointer. This is allowed but should make you pause: it claims write access to data that was originally borrowed immutably.

### Task 1.2 -- Dereferencing raw pointers

To read or write through a raw pointer, you need an unsafe block:

```rust
fn main() {
    let x = 42;
    let ptr: *const i32 = &x;

    // Dereferencing requires unsafe:
    let value = unsafe { *ptr };
    println!("value: {value}");

    // Same for mutable raw pointers:
    let mut y = 100;
    let mut_ptr: *mut i32 = &mut y;

    unsafe {
        *mut_ptr = 200;
    }

    println!("y: {y}");
}
```

Run the program. Expected output:

```
value: 42
y: 200
```

The unsafe block is your assertion: "I have verified that this pointer is valid (non-null, aligned, pointing to a properly-typed value, not aliasing in a way that causes a data race)." The compiler trusts the assertion; it does NOT check.

If you violate the invariants (dereference a null pointer, read uninitialized memory, etc.), you have undefined behavior. The program might crash, produce wrong results, or appear to work and fail later in unrelated code.

### Task 1.3 -- Pointer arithmetic

You can traverse arrays manually using pointer arithmetic:

```rust
fn main() {
    let array = [10, 20, 30, 40, 50];
    let ptr: *const i32 = array.as_ptr();

    unsafe {
        for i in 0..5 {
            let elem_ptr = ptr.add(i);
            println!("element {i}: {}", *elem_ptr);
        }
    }
}
```

Run the program. Expected output:

```
element 0: 10
element 1: 20
element 2: 30
element 3: 40
element 4: 50
```

The `add(i)` method advances the pointer by `i` elements. For `*const i32`, that is `i * 4` bytes (assuming 4-byte alignment of i32). The method handles size calculation; you specify offsets in elements, not bytes.

This is what indexing a slice (`array[i]`) compiles to internally, but with bounds checking. The pointer arithmetic version skips the bounds check, which is faster but unsafe if you might go out of bounds.

### Task 1.4 -- What goes wrong without checks

Consider what happens if you go past the end:

```rust
fn main() {
    let array = [10, 20, 30];
    let ptr: *const i32 = array.as_ptr();

    unsafe {
        for i in 0..10 {        // BAD: reads past the end
            let elem_ptr = ptr.add(i);
            println!("element {i}: {}", *elem_ptr);
        }
    }
}
```

DO NOT run this. The behavior is undefined. In practice, you might see:

- Random integer values from adjacent memory.
- A segmentation fault if you happen to cross a memory page boundary.
- Apparently-correct output that crashes the program at some later, unrelated point.
- Different behavior every time you run it.

The compiler cannot save you here. You wrote `unsafe`; you asserted that the access was valid; the compiler trusted you. The mistake is yours.

This is the central trade-off of unsafe. With safety, you get clear errors and predictable behavior. Without safety, you get bugs that can be invisible during testing and catastrophic in production.

### Task 1.5 -- Null pointers

Raw pointers can be null:

```rust
fn main() {
    let null: *const i32 = std::ptr::null();
    let null_mut: *mut i32 = std::ptr::null_mut();

    println!("null is null: {}", null.is_null());
    println!("null_mut is null: {}", null_mut.is_null());

    // Dereferencing a null pointer is undefined behavior.
    // DO NOT do this:
    // let _ = unsafe { *null };

    // Always check before dereferencing:
    if !null.is_null() {
        let _value = unsafe { *null };
    } else {
        println!("(would not dereference null)");
    }
}
```

Run the program. Expected output:

```
null is null: true
null_mut is null: true
(would not dereference null)
```

The `is_null()` method checks for null. You should always check before dereferencing if there is any possibility the pointer might be null.

In FFI code (the next exercise), many C functions return null on error. Forgetting to check is a common source of crashes.

### Checkpoints

1. Task 1.1 created multiple raw pointers (some const, some mut) all pointing at the same data. The compiler did not complain. Compare this to references: how does the borrow checker handle multiple references to the same data, and why are raw pointers different?
2. The unsafe block in Task 1.2 was small (just `*ptr` or `*mut_ptr = 200`). Why is keeping unsafe blocks small important? What is the difference between `unsafe { *ptr; some_other_code; another_thing; }` and minimizing the scope of each unsafe block?
3. The bad code in Task 1.4 reads past the end of an array. The compiler does not catch this; the program might appear to work in some runs and crash in others. What does this tell you about how unsafe code should be tested?

---

## Exercise 2 -- The Five Superpowers

**Estimated time:** 15 minutes
**Topics covered:** the five operations `unsafe` enables, what `unsafe` does NOT disable

### Context

The `unsafe` keyword does not turn off all safety checks. It enables exactly five additional abilities. This exercise demonstrates each one and verifies that everything else still works normally inside unsafe blocks.

### Task 2.1 -- Power 1: Dereferencing raw pointers

You have already seen this in Exercise 1. The first superpower is dereferencing `*const T` and `*mut T`:

```rust
fn main() {
    let value = 42;
    let ptr: *const i32 = &value;

    // SAFETY: ptr was just created from a reference to `value`,
    // which is alive for the duration of this block.
    let read = unsafe { *ptr };

    println!("read: {read}");
}
```

Run the program. Expected output:

```
read: 42
```

Notice the `// SAFETY:` comment. This is the conventional way to document why an unsafe block is safe. Every unsafe block in your code should have one. A code review of unsafe code looks first at the SAFETY comments; an unsafe block without justification is a red flag.

### Task 2.2 -- Power 2: Calling unsafe functions

Functions can be declared `unsafe fn`. Calling them requires an unsafe block:

```rust
unsafe fn dangerous(ptr: *const u8) -> u8 {
    // Inside an unsafe fn, the body is implicitly unsafe.
    *ptr
}

fn main() {
    let value: u8 = 42;
    let ptr = &value as *const u8;

    // SAFETY: ptr is created from a valid reference; the byte
    // it points to is valid for the duration of this block.
    let result = unsafe { dangerous(ptr) };

    println!("result: {result}");
}
```

Run the program. Expected output:

```
result: 42
```

The function `dangerous` has `unsafe fn` in its signature. This signals "I have preconditions you must verify before calling." The caller must verify them and acknowledge with an unsafe block.

This is how the standard library exposes operations like `slice::get_unchecked` (skips bounds checking) or `String::from_utf8_unchecked` (skips UTF-8 validation). Both are useful when you have already done the verification yourself.

### Task 2.3 -- Power 3: Accessing mutable statics

Mutable static variables are global mutable state:

```rust
static mut COUNTER: u32 = 0;

fn increment() {
    // SAFETY: this function is called only from main, which is
    // single-threaded. No race conditions are possible.
    unsafe {
        COUNTER += 1;
    }
}

fn current() -> u32 {
    // SAFETY: same as increment.
    unsafe { COUNTER }
}

fn main() {
    increment();
    increment();
    increment();

    println!("counter: {}", current());
}
```

Run the program. Expected output:

```
counter: 3
```

`static mut` is genuinely global mutable state. The compiler cannot verify single-threaded access, so it requires unsafe.

For multi-threaded programs, you almost always want `AtomicU32` instead (from Module 14):

```rust
use std::sync::atomic::{AtomicU32, Ordering};

static COUNTER: AtomicU32 = AtomicU32::new(0);

fn main() {
    COUNTER.fetch_add(1, Ordering::SeqCst);
    COUNTER.fetch_add(1, Ordering::SeqCst);
    COUNTER.fetch_add(1, Ordering::SeqCst);

    println!("counter: {}", COUNTER.load(Ordering::SeqCst));
}
```

Same result, but the atomic type provides safe concurrent access. No unsafe is needed at the call sites. Use atomic types whenever possible; reserve `static mut` for the rare single-threaded cases.

### Task 2.4 -- Power 4: Implementing unsafe traits

Some traits are declared `unsafe trait`. Implementing them requires `unsafe impl`. This is covered more thoroughly in Exercise 5; here, observe that the pattern exists:

```rust
unsafe trait MyUnsafeMarker {
    fn marker_value(&self) -> u32;
}

struct MyType;

// The unsafe impl asserts that the implementation upholds
// the trait's invariants (whatever they are).
unsafe impl MyUnsafeMarker for MyType {
    fn marker_value(&self) -> u32 {
        42
    }
}

fn main() {
    let t = MyType;
    println!("marker: {}", t.marker_value());
}
```

Run the program. Expected output:

```
marker: 42
```

The `unsafe impl` is your assertion that you have read the trait's documented invariants and your implementation upholds them. The compiler does not check; you take responsibility.

### Task 2.5 -- Power 5: Accessing union fields

Unions are types where multiple fields share the same memory:

```rust
union IntOrFloat {
    int_val: u32,
    float_val: f32,
}

fn main() {
    let u = IntOrFloat { int_val: 1065353216 };

    // SAFETY: we constructed u as an int_val. Reading int_val
    // is reading what we wrote. Reading float_val reinterprets
    // the same bytes as f32, which is valid for this union.
    unsafe {
        println!("as int: {}", u.int_val);
        println!("as float: {}", u.float_val);
    }
}
```

Run the program. Expected output:

```
as int: 1065353216
as float: 1
```

The integer 1065353216 and the float 1.0 share the same bit pattern in IEEE 754. The union lets you reinterpret the same memory as different types.

Unions are rare in Rust. Most code uses enums (which track which variant is active and prevent reading the wrong variant). Unions appear mainly in FFI with C code that uses unions, or in low-level memory-layout code.

### Task 2.6 -- What unsafe does NOT disable

The five powers above are the entire list. Everything else still applies. Try this:

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    // Inside an unsafe block, the borrow checker still runs:
    unsafe {
        let r1 = &v;
        let r2 = &mut v;    // ERROR: cannot borrow as mutable while immutable borrow exists
        println!("{r1:?} {r2:?}");
    }
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
```

The borrow checker complains exactly as it would in safe code. `unsafe` does not turn it off.

Remove the bad line:

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    unsafe {
        let r1 = &v;
        println!("{r1:?}");
    }

    v.push(4);
    println!("{v:?}");
}
```

This compiles and runs fine. Expected output:

```
[1, 2, 3]
[1, 2, 3, 4]
```

The point: unsafe is narrow. It unlocks the five specific abilities listed above and nothing more. The borrow checker, lifetime checker, type checker, and exhaustiveness checking all still run. Most of Rust's safety guarantees are preserved even inside unsafe blocks.

### Checkpoints

1. Task 2.3 showed `static mut` for a counter and then showed `AtomicU32` as a safe alternative. The atomic version achieves the same result without unsafe. Why does the standard library provide atomic types instead of just relying on `static mut`?
2. Task 2.6 demonstrated that the borrow checker runs inside unsafe blocks. Some students assume `unsafe` disables all safety; this exercise shows it does not. Why does this design (narrow unsafe) make Rust safer than a hypothetical "all checks off" unsafe?
3. The five powers are: dereferencing raw pointers, calling unsafe functions, accessing mutable statics, implementing unsafe traits, accessing union fields. Of these five, which would you expect to see most frequently in application code? Which would you expect to almost never see?

---

## Exercise 3 -- Calling C Functions via FFI

**Estimated time:** 15-20 minutes
**Topics covered:** `extern "C"` blocks, the `libc` crate, `CString`/`CStr`, common pitfalls

### Context

FFI (foreign function interface) is the most common use of unsafe in real applications. This exercise calls functions from the C standard library through Rust's FFI machinery.

### Task 3.1 -- Add libc as a dependency

Open `Cargo.toml` and add:

```toml
[dependencies]
libc = "0.2"
```

Save the file. The next build will download `libc`. The `libc` crate provides Rust declarations for thousands of C standard library functions and types.

### Task 3.2 -- Call a simple C function

Replace your `src/main.rs` with:

```rust
use libc::abs;

fn main() {
    let negative = -42i32;

    // SAFETY: `abs` is a pure function with no preconditions
    // beyond accepting an i32. Any i32 value is valid input.
    let absolute = unsafe { abs(negative) };

    println!("abs({negative}) = {absolute}");
}
```

Run:

```bash
cargo run
```

Expected output:

```
abs(-42) = 42
```

The `abs` function comes from the C standard library. Its signature in C is `int abs(int);`. The `libc` crate provides a Rust binding that matches: `pub unsafe extern "C" fn abs(arg1: c_int) -> c_int;`.

Calling it requires unsafe because it crosses a language boundary; the Rust compiler cannot verify what happens inside the C function. For `abs`, the function is pure and harmless, but the unsafe is required by the type system regardless.

### Task 3.3 -- A more interesting C function

The `getpid` function returns the current process ID:

```rust
use libc::getpid;

fn main() {
    // SAFETY: `getpid` has no preconditions; it always succeeds.
    let pid = unsafe { getpid() };

    println!("process id: {pid}");
}
```

Run. Expected output (the actual PID will vary):

```
process id: 12345
```

Again, the unsafe is required because of the language boundary, not because anything specifically dangerous is happening. Many FFI calls follow this pattern: short, safe at runtime, but requiring unsafe at the call site as a matter of language design.

### Task 3.4 -- Passing strings to C

Most C functions that take strings expect null-terminated byte arrays. Rust's `String` is NOT null-terminated. To bridge, use `CString`:

```rust
use std::ffi::CString;
use libc::{c_int, c_char};

extern "C" {
    fn puts(s: *const c_char) -> c_int;
}

fn main() {
    let message = "Hello from Rust via puts!";
    let c_string = CString::new(message).expect("contains null byte");

    // SAFETY:
    // - c_string.as_ptr() is non-null and points to a valid,
    //   null-terminated C string.
    // - The CString outlives this call (it's not dropped yet).
    // - puts does not write through the pointer; it only reads.
    let result = unsafe { puts(c_string.as_ptr()) };

    println!("puts returned: {result}");
}
```

Run. Expected output:

```
Hello from Rust via puts!
puts returned: 26
```

`puts` prints the string followed by a newline and returns the number of bytes written (including the newline, so 26 for our 25-char message).

The pattern:

- `CString::new(rust_str)` allocates a null-terminated copy of the string.
- `c_string.as_ptr()` produces a raw pointer to it.
- The CString must outlive the call. As long as `c_string` is alive in the function, the pointer remains valid.
- When the function returns, `c_string` is dropped and the memory is freed.

### Task 3.5 -- The temporary-CString pitfall

A subtle bug:

```rust
extern "C" {
    fn puts(s: *const c_char) -> c_int;
}

fn main() {
    // BAD: temporary CString may be dropped before the call
    let ptr = CString::new("temporary").unwrap().as_ptr();

    // SAFETY: ... actually, this is NOT safe!
    // The CString from the previous line is dropped at the end
    // of that statement. ptr is now dangling.
    unsafe {
        puts(ptr);  // undefined behavior
    }
}
```

DO NOT run this. The bug:

- `CString::new("temporary").unwrap()` creates a temporary CString.
- `.as_ptr()` extracts a pointer to its buffer.
- At the end of the statement, the temporary CString is dropped.
- The buffer is freed.
- The pointer `ptr` now points to freed memory.
- Calling `puts(ptr)` reads freed memory: undefined behavior.

The fix is to name the CString to extend its lifetime:

```rust
let c_string = CString::new("permanent").unwrap();
// c_string lives until the end of this function

// SAFETY: c_string is named and lives for the rest of the
// function; its buffer remains valid for the puts call.
unsafe {
    puts(c_string.as_ptr());
}
```

This is a real pitfall. Modern Rust sometimes extends temporary lifetimes in ways that hide the bug, but you cannot rely on this; the original pattern is undefined behavior.

The rule: when extracting a pointer from an owned type, ensure the owned type lives for the entire duration of the pointer's use. This usually means naming a binding rather than chaining method calls.

### Task 3.6 -- Receiving strings from C

Going the other direction: a C function returns a `*const c_char`. To convert it to a Rust string:

```rust
use std::ffi::{CStr, CString};
use libc::{c_char, c_int};

extern "C" {
    fn getenv(name: *const c_char) -> *const c_char;
}

fn get_env(name: &str) -> Option<String> {
    let c_name = CString::new(name).ok()?;

    // SAFETY:
    // - c_name.as_ptr() is valid (c_name is alive for the call).
    // - getenv returns either a valid C string pointer or null.
    let result_ptr = unsafe { getenv(c_name.as_ptr()) };

    if result_ptr.is_null() {
        return None;
    }

    // SAFETY:
    // - result_ptr is non-null (checked above).
    // - The string is null-terminated (getenv's contract).
    // - The memory remains valid until we call to_string_lossy
    //   (which copies the bytes).
    let c_str = unsafe { CStr::from_ptr(result_ptr) };
    Some(c_str.to_string_lossy().into_owned())
}

fn main() {
    if let Some(home) = get_env("HOME") {
        println!("HOME = {home}");
    } else {
        println!("HOME is not set");
    }

    if let Some(path) = get_env("PATH") {
        println!("PATH = {} chars", path.len());
    }

    println!("NONEXISTENT_VAR = {:?}", get_env("NONEXISTENT_VAR"));
}
```

Run. Expected output (your values will vary):

```
HOME = /home/yourname
PATH = 156 chars
NONEXISTENT_VAR = None
```

Several patterns are demonstrated:

- The function signature is fully safe (`fn get_env(name: &str) -> Option<String>`). Callers see no unsafe.
- Internally, two unsafe blocks: one for the `getenv` call, one for `CStr::from_ptr`.
- Each unsafe block has a SAFETY comment explaining why it is correct.
- The null check between the two unsafe blocks is essential: `CStr::from_ptr` with a null pointer would be undefined behavior.
- The result is converted to an owned `String` so it does not depend on the C library's memory remaining valid.

This is the canonical FFI wrapper pattern: small unsafe blocks inside a fully safe public function, with explicit checks and conversions to bridge between C and Rust idioms.

### Task 3.7 -- Common FFI pitfalls

A quick reference for the most common FFI bugs:

- **Dangling pointers** (Task 3.5): pointer outlives its owner.
- **Mismatched types**: declaring `int` as `i32` when the C int is actually 64-bit on some platforms. Use `libc::c_int` and similar aliases for portability.
- **Mismatched calling conventions**: declaring `extern "C"` for a library that uses `extern "stdcall"`. Crashes on Windows.
- **Null pointers**: forgetting to check before dereferencing.
- **Lifetime confusion**: a C function returning a pointer might require you to free the memory, or might require you NOT to. Check the documentation.
- **Thread safety**: C libraries often have undocumented threading constraints. Calling a non-thread-safe library from multiple threads is undefined behavior.

For most projects, prefer existing "safe bindings" crates over writing FFI yourself. The `openssl` crate, the `sqlx` crate (for databases), and many others provide safe Rust APIs over their C dependencies.

### Checkpoints

1. The `abs` call in Task 3.2 was unsafe even though abs is harmless. Why does Rust require unsafe at every FFI call site, even for safe-seeming C functions?
2. Task 3.5 demonstrated the dangling-CString pitfall. The expression `CString::new("...").unwrap().as_ptr()` looks innocent but is wrong. What is the rule for safely extracting pointers from owned types?
3. The `get_env` function in Task 3.6 had a fully safe public signature but contained two unsafe blocks internally. This is the "safe abstraction over unsafe" pattern. Why is this the right structure for FFI bindings?

---

## Exercise 4 -- Implementing `Send` and `Sync` Manually

**Estimated time:** 10-15 minutes
**Topics covered:** unsafe traits, when to manually implement `Send`/`Sync`, marker types

### Context

The `Send` and `Sync` traits (from Module 14) determine whether a type can cross thread boundaries. Most types implement them automatically; some require manual `unsafe impl`. This exercise shows when and how.

### Task 4.1 -- A type that isn't Send

Replace `src/main.rs` with:

```rust
use std::thread;

struct MyBuffer {
    data: *mut u8,
    len: usize,
}

impl MyBuffer {
    fn new(len: usize) -> MyBuffer {
        let data = vec![0u8; len].leak().as_mut_ptr();
        MyBuffer { data, len }
    }
}

fn main() {
    let buffer = MyBuffer::new(100);

    // Try to send the buffer to another thread:
    let handle = thread::spawn(move || {
        println!("buffer has {} bytes", buffer.len);
    });

    handle.join().unwrap();
}
```

Run `cargo build`. The compiler rejects this:

```
error[E0277]: `*mut u8` cannot be sent between threads safely
  |
  | let handle = thread::spawn(move || {
  |              ------------- ^^^^^^^^^^ `*mut u8` cannot be sent between threads safely
```

The struct `MyBuffer` contains a `*mut u8`, which is not `Send` by default. The auto-deriving rules say: a struct is `Send` only if all its fields are `Send`. Raw pointers are not `Send`. So `MyBuffer` is not `Send`.

The compiler refuses to move it to another thread. This is the `Send` machinery working correctly: it prevents potential data races.

### Task 4.2 -- The unsafe impl Send

If you know that your type is actually safe to send across threads (because of how you allocate and manage the memory), you can declare it manually:

```rust
use std::thread;

struct MyBuffer {
    data: *mut u8,
    len: usize,
}

// SAFETY: MyBuffer's data is allocated via Vec::leak, which produces
// memory that is owned exclusively by the MyBuffer. No other code
// holds a reference to the same allocation. Moving the buffer between
// threads transfers ownership of the allocation; no aliasing occurs.
unsafe impl Send for MyBuffer {}

impl MyBuffer {
    fn new(len: usize) -> MyBuffer {
        let data = vec![0u8; len].leak().as_mut_ptr();
        MyBuffer { data, len }
    }
}

fn main() {
    let buffer = MyBuffer::new(100);

    let handle = thread::spawn(move || {
        println!("buffer has {} bytes", buffer.len);
    });

    handle.join().unwrap();
}
```

Run the program. Expected output:

```
buffer has 100 bytes
```

The `unsafe impl Send for MyBuffer {}` line is your assertion: "I have verified that this type can safely cross thread boundaries." The compiler now accepts the type as `Send` and lets it be moved into a thread.

The SAFETY comment is essential. It explains why the assertion is true. Without it, a future reader cannot tell whether this impl is correct or just a bypass of the compiler's complaint.

If your reasoning is wrong (the data is actually shared across threads in some way you did not notice), you have created the conditions for data races. The compiler will not catch it; you took responsibility when you wrote `unsafe impl`.

### Task 4.3 -- A wrong reason to implement Send

The wrong reason to use `unsafe impl Send`:

```rust
// BAD: implementing Send just to bypass the compiler
unsafe impl Send for MyBuffer {}    // no SAFETY comment, no justification
```

This pattern is a red flag. The developer hit a compile error, added `unsafe impl`, and moved on. There is no analysis of whether the type is actually safe to send.

This is exactly the kind of bug that unsafe traits enable. The code compiles; tests pass (because tests do not stress the threading). In production, under high load, you get rare crashes or data corruption that are impossible to debug.

The right approach:

- If you are confident the type is safe: write the `unsafe impl` with a SAFETY comment explaining why.
- If you are not sure: do not write the `unsafe impl`. Find a different design (use `Arc<Mutex<T>>`, use a different type, use a channel).

The choice is yours, but it must be deliberate. Reflexively reaching for `unsafe impl Send` to silence the compiler is one of the most dangerous patterns in Rust.

### Task 4.4 -- A type that should NOT be Send

For comparison, here is a type where `unsafe impl Send` would be genuinely wrong:

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct BadIdea {
    counter: Rc<RefCell<u32>>,
}

// DO NOT do this. Rc uses non-atomic reference counting.
// Sending an Rc to another thread can cause data races on
// the reference count, leading to use-after-free or memory leaks.
// unsafe impl Send for BadIdea {}
```

The comment-out is essential. Implementing `Send` here would be unsound: `Rc` is deliberately `!Send` because its reference counting is not thread-safe. Manually overriding the auto-derivation does not change the underlying issue.

If you need shared state across threads, use `Arc<Mutex<T>>` (which IS `Send` and `Sync`) instead of `Rc<RefCell<T>>`. The compiler will accept it without manual `unsafe impl`.

### Task 4.5 -- A marker type pattern

A common use of manual `Send`/`Sync` impls: marker types in libraries that bind to C code with documented thread-safety guarantees:

```rust
struct DatabaseHandle {
    _ptr: *mut std::ffi::c_void,
}

// SAFETY: The underlying C library documents that DatabaseHandle
// can be safely transferred between threads when no thread is
// currently performing operations on it. Our API enforces this
// by taking &mut self for any operation, which prevents concurrent
// use through references.
unsafe impl Send for DatabaseHandle {}

// Note: we do NOT impl Sync because concurrent access from multiple
// threads (via &DatabaseHandle) IS unsafe per the library's docs.

impl DatabaseHandle {
    fn open(_path: &str) -> DatabaseHandle {
        // Would call C library to open database
        DatabaseHandle { _ptr: std::ptr::null_mut() }
    }
}

fn main() {
    let _handle = DatabaseHandle::open("data.db");
    println!("handle created");
}
```

Run the program. Expected output:

```
handle created
```

This pattern is common in real FFI bindings. The C library has specific threading rules (read its documentation); the Rust wrapper translates those rules into `Send`/`Sync` impls.

A few important details:

- The `Send` impl is justified by reading the C library's threading documentation.
- The decision to NOT impl `Sync` is also deliberate: concurrent access through references would be unsafe.
- The struct's API uses `&mut self`, which enforces exclusive access from Rust's side.

These decisions cannot be made by the compiler; they require understanding the C library's contract. The `unsafe impl` is your assertion that you have understood and translated that contract correctly.

### Checkpoints

1. The `Send` auto-derivation rules say "a struct is Send only if all its fields are Send." Why does the compiler use this conservative rule rather than allowing manual override at every type?
2. Task 4.3 described the "wrong reason" pattern: adding `unsafe impl Send` to silence a compile error. Why is this pattern especially dangerous compared to other misuses of unsafe? What makes the consequences hard to detect?
3. The pattern in Task 4.5 implemented `Send` but deliberately NOT `Sync`. What is the practical difference, and when would you want to allow one but not the other?

---

## Exercise 5 -- Building a Safe Abstraction

**Estimated time:** 15 minutes
**Topics covered:** encapsulating unsafe in safe APIs, the SAFETY comment convention

### Context

The most important pattern in unsafe Rust: small unsafe regions hidden inside safe APIs. This exercise builds one such abstraction.

### Task 5.1 -- The starting point

Suppose you want a function that, given a slice and an index, returns the element at that index. The safe version is `slice.get(index)`, which returns `Option<&T>`. Here we build a similar function ourselves to see the pattern.

Replace `src/main.rs` with:

```rust
fn checked_get(slice: &[i32], index: usize) -> Option<i32> {
    if index < slice.len() {
        // SAFETY: we just checked that index < slice.len(),
        // so the offset is in bounds. slice.as_ptr() is valid
        // for `slice.len()` elements by the type system.
        unsafe {
            Some(*slice.as_ptr().add(index))
        }
    } else {
        None
    }
}

fn main() {
    let data = [10, 20, 30, 40, 50];

    println!("{:?}", checked_get(&data, 2));     // Some(30)
    println!("{:?}", checked_get(&data, 4));     // Some(50)
    println!("{:?}", checked_get(&data, 5));     // None
    println!("{:?}", checked_get(&data, 100));   // None
}
```

Run the program. Expected output:

```
Some(30)
Some(50)
None
None
```

What the function does:

- The signature `fn checked_get(slice: &[i32], index: usize) -> Option<i32>` is entirely safe. Callers see no unsafe.
- Inside, the bounds check (`if index < slice.len()`) verifies the precondition for the unsafe operation.
- The unsafe block performs raw pointer indexing, with a SAFETY comment explaining why it is correct.
- Out-of-bounds indices return `None` instead of accessing invalid memory.

This is the pattern in miniature. The unsafe is concentrated in one small block; the bounds check is right next to it; the safe abstraction wraps it all.

### Task 5.2 -- Adding unchecked variants

The standard library exposes both safe and unchecked versions of many operations. `slice::get` is safe and returns `Option`; `slice::get_unchecked` is unsafe and skips the bounds check (faster but caller must verify bounds).

You can do the same:

```rust
fn checked_get(slice: &[i32], index: usize) -> Option<i32> {
    if index < slice.len() {
        // SAFETY: bounds checked above.
        Some(unsafe { unchecked_get(slice, index) })
    } else {
        None
    }
}

/// Returns the element at `index` without bounds checking.
///
/// # Safety
///
/// The caller must ensure that `index < slice.len()`.
/// Calling this with an out-of-bounds index is undefined behavior.
unsafe fn unchecked_get(slice: &[i32], index: usize) -> i32 {
    *slice.as_ptr().add(index)
}

fn main() {
    let data = [10, 20, 30, 40, 50];

    println!("{:?}", checked_get(&data, 2));

    // Using the unchecked version directly:
    // SAFETY: we know data has at least 3 elements.
    let value = unsafe { unchecked_get(&data, 2) };
    println!("{value}");
}
```

Run the program. Expected output:

```
Some(30)
30
```

The function `unchecked_get` is `unsafe fn`. Its documentation explicitly lists what the caller must guarantee (a `# Safety` section, which is the standard convention for unsafe functions).

Callers in performance-critical code might use `unchecked_get` directly when they have already verified bounds. The safe wrapper `checked_get` is the safe API for everyone else.

This dual pattern (safe wrapper + unsafe primitive) is common in the standard library: `String::from_utf8` (safe, validates) and `String::from_utf8_unchecked` (unsafe, skips validation); `slice::get` (safe) and `slice::get_unchecked` (unsafe); and many more.

### Task 5.3 -- A more complex example: a simple Box

Building a slightly more complex safe abstraction. A miniature version of `Box`:

```rust
use std::alloc::{alloc, dealloc, Layout};

struct MyBox {
    ptr: *mut i32,
}

impl MyBox {
    pub fn new(value: i32) -> MyBox {
        let layout = Layout::new::<i32>();

        // SAFETY: Layout::new::<i32>() returns a valid layout.
        // alloc returns either a valid pointer or null.
        let ptr = unsafe { alloc(layout) as *mut i32 };

        if ptr.is_null() {
            std::alloc::handle_alloc_error(layout);
        }

        // SAFETY: ptr is non-null (checked above) and properly aligned
        // for i32 (Layout::new::<i32>() ensures alignment). Writing
        // through it initializes the memory.
        unsafe {
            *ptr = value;
        }

        MyBox { ptr }
    }

    pub fn get(&self) -> i32 {
        // SAFETY: self.ptr was allocated and initialized in new();
        // it remains valid until self is dropped.
        unsafe { *self.ptr }
    }

    pub fn set(&mut self, value: i32) {
        // SAFETY: same as get; we have &mut self so exclusive access.
        unsafe { *self.ptr = value; }
    }
}

impl Drop for MyBox {
    fn drop(&mut self) {
        let layout = Layout::new::<i32>();
        // SAFETY: self.ptr was allocated via alloc with this layout.
        // After drop, no one else has a reference to this memory.
        unsafe {
            dealloc(self.ptr as *mut u8, layout);
        }
    }
}

fn main() {
    let mut b = MyBox::new(42);
    println!("initial: {}", b.get());

    b.set(100);
    println!("after set: {}", b.get());

    let b2 = MyBox::new(999);
    println!("another box: {}", b2.get());

    // Both boxes are automatically dropped at end of scope,
    // each freeing its own memory.
}
```

Run the program. Expected output:

```
initial: 42
after set: 100
another box: 999
```

This is a real (if minimal) smart pointer:

- `MyBox::new(value)` allocates heap memory and stores the value.
- `get()` and `set()` access the value through safe APIs.
- `Drop` frees the memory automatically when the box goes out of scope.

The user sees only safe code: `MyBox::new`, `b.get()`, `b.set(100)`. The unsafe blocks (four of them) are concentrated inside the implementation, each with a SAFETY comment.

This is exactly how the real `Box<T>` works, just simplified. The standard library's `Box`, `Vec`, `String`, `Rc`, and similar types use this pattern: a safe public API over carefully-managed unsafe internals.

### Task 5.4 -- Reading the SAFETY comments

Look back at the four SAFETY comments in `MyBox`:

```rust
// SAFETY: Layout::new::<i32>() returns a valid layout.
// alloc returns either a valid pointer or null.

// SAFETY: ptr is non-null (checked above) and properly aligned
// for i32 (Layout::new::<i32>() ensures alignment). Writing
// through it initializes the memory.

// SAFETY: self.ptr was allocated and initialized in new();
// it remains valid until self is dropped.

// SAFETY: self.ptr was allocated via alloc with this layout.
// After drop, no one else has a reference to this memory.
```

Each comment justifies why the unsafe operation is correct:

- The first explains the precondition of `alloc`.
- The second explains why writing to the pointer is valid (non-null, aligned).
- The third explains why reading from the pointer is valid (still allocated, still owned).
- The fourth explains why freeing is correct (allocated this way, no other references).

This is the standard for unsafe code in Rust. Every unsafe block has a comment explaining its correctness. Code reviews of unsafe code start with the SAFETY comments.

If you remove the comments, the code still compiles. But it becomes much harder to review. A future developer reading the code (including future you) cannot tell whether each unsafe is correct.

### Task 5.5 -- The encapsulation principle

The structure of `MyBox` demonstrates the encapsulation principle:

- **From outside:** completely safe. Users call `new()`, `get()`, `set()`. The Drop is automatic. No unsafe needed.
- **From inside:** small, well-documented unsafe blocks.
- **Audit boundary:** to verify safety, an auditor reads only the unsafe blocks and their SAFETY comments.

This is why Rust can be both safe and capable. Most code is automatically checked by the compiler; the small unsafe regions are concentrated, documented, and auditable.

Compare to C, where every pointer access might be wrong. Auditing a C program for memory safety means reading every line. Auditing a Rust program means reading only the unsafe blocks.

The leverage is enormous. A library author writes a small amount of carefully-reviewed unsafe code; thousands of downstream users get a safe API.

### Checkpoints

1. The `checked_get` function in Task 5.1 had four lines of safety-critical code, but only one of them (the dereference) was inside an unsafe block. Why is this organization (bounds check outside, deref inside) better than putting the entire body inside an unsafe block?
2. Task 5.2 introduced both `checked_get` (safe) and `unchecked_get` (unsafe). Both serve real purposes. When would a caller use the safe version? When would they reach for the unsafe one?
3. The `MyBox` implementation in Task 5.3 had four unsafe blocks, each with a SAFETY comment. Imagine these comments were missing. What would change about the code's review process?

---

## Exercise 6 -- Recognizing and Auditing Unsafe

**Estimated time:** 10-15 minutes
**Topics covered:** `cargo-geiger`, `Miri`, finding unsafe in dependencies

### Context

When using Rust in production, you depend on many crates that contain unsafe code internally. Understanding what to look for and how to audit is a practical skill. This exercise covers the tools.

### Task 6.1 -- cargo-geiger overview

The `cargo-geiger` tool counts unsafe usage across your project and its dependencies. Install it:

```bash
cargo install cargo-geiger
```

(This may take several minutes; the tool itself is a substantial Rust crate.)

If installation fails (the tool sometimes has compatibility issues), you can skip this task and continue with Task 6.2.

If installation succeeds, run it from your project root:

```bash
cargo geiger
```

You see a report listing each crate in your dependency tree, with counters for unsafe expressions, unsafe `impl` blocks, and unsafe items. Crates with no unsafe show "0"; crates with substantial unsafe show their counts.

For our `unsafe_explorer` project, you will see:

- The `unsafe_explorer` crate itself (varying unsafe counts depending on which exercises you have completed).
- The `libc` crate (substantial unsafe usage, expected).
- Various transitive dependencies.

This gives you a sense of how much unsafe is in your project's dependencies. For most Rust projects, the answer is "some, mostly in well-known crates like `libc`, `tokio`, `serde`."

### Task 6.2 -- Evaluating dependency safety

When choosing a dependency, you can consider its unsafe usage as one factor. Reasonable questions:

- **How much unsafe is in the crate?** A crate with extensive unsafe needs more scrutiny.
- **How well-maintained is the crate?** Active maintenance increases the chance that bugs are caught and fixed.
- **How widely used is the crate?** Popular crates have more eyes on them, including for security audits.
- **What kind of unsafe is used?** FFI to a well-known C library (OpenSSL) is different from raw pointer manipulation in a young, niche crate.

The Rust community has done a lot of work on auditing key infrastructure. The `cargo-audit` tool checks for known security advisories in your dependencies:

```bash
cargo install cargo-audit
cargo audit
```

This scans your dependencies against a database of known vulnerabilities. For your `unsafe_explorer` project, you should see no advisories (the dependencies are mainstream and current).

For production code, running `cargo audit` in CI is standard practice. It catches "your dependency has a known security bug; please update."

### Task 6.3 -- Miri overview

Miri is an interpreter for Rust that detects undefined behavior at runtime. It catches bugs that escape the compiler:

- Use-after-free.
- Out-of-bounds memory access.
- Type confusion.
- Some data races.

To use Miri, you need a nightly Rust toolchain:

```bash
rustup install nightly
rustup component add miri --toolchain nightly
```

Then run your tests under Miri:

```bash
cargo +nightly miri test
```

Miri is slow (10-100x slower than native execution) and not all code can run under it (FFI to actual C libraries cannot, for example). But for code with significant unsafe and good test coverage, Miri is one of the strongest verification tools available.

For our `unsafe_explorer` project, the FFI code in Exercise 3 will NOT run under Miri (the actual `libc::abs`, `puts`, `getenv` functions are not available). The raw pointer code from earlier exercises would run under Miri and could catch bugs like out-of-bounds access if present.

For real libraries with substantial unsafe, the workflow is:

1. Write tests that exercise the unsafe code.
2. Run tests normally (`cargo test`).
3. Run tests under Miri (`cargo +nightly miri test`).
4. Fix any UB that Miri finds.

This is how the standard library and major crates verify their unsafe code. The investment is real but worthwhile for any code with substantial unsafe.

### Task 6.4 -- A practical example: searching for unsafe

In a real codebase, you might want to find all unsafe usage. A simple way:

```bash
grep -rn "unsafe" src/ | grep -v "^.*://" 
```

This lists every line containing `unsafe`, excluding lines that look like comments (`//`). Try it in your project:

The output should include all the unsafe blocks and `unsafe fn` declarations from your code.

For larger projects, dedicated tools like `cargo-geiger` are better, but `grep` works for quick checks.

In a code review, you would look at each unsafe occurrence and verify:

- Is there a SAFETY comment?
- Does the comment correctly justify the safety?
- Is the unsafe block as small as possible?
- Could a safe alternative have been used?

These are the same questions an external auditor would ask. Writing well-justified unsafe code makes the audit easy; writing unsafe without justification makes it impossible.

### Task 6.5 -- The encapsulation audit

A more useful audit, for a library: verify that all unsafe code is encapsulated within safe APIs. The check is:

1. Identify each `pub fn` (or `pub struct` with `pub` methods).
2. For each, ask: does the public signature contain `unsafe`?
3. If yes, the function exposes unsafe to callers.
4. If no, the function should not be able to violate safety regardless of how it is called.

For a well-designed library, most public APIs should NOT be `unsafe fn`. The unsafe should be hidden inside the implementation. Public unsafe functions are appropriate only when the function genuinely has preconditions the caller must verify (like `slice::get_unchecked`).

Look at your `MyBox` implementation from Exercise 5. The public methods (`new`, `get`, `set`) are all safe. The unsafe is encapsulated inside.

This is the model. A library with many `pub unsafe fn` has not been encapsulated; users of the library must understand the unsafe contracts. A library with no `pub unsafe fn` has been encapsulated; users can use it without ever writing `unsafe` themselves.

### Checkpoints

1. The `cargo-geiger` tool counts unsafe usage in your dependencies. Why is "amount of unsafe" only one factor in evaluating a crate, not a deciding one? What other factors matter?
2. Miri catches undefined behavior at runtime, but it is much slower than native execution. When is this trade-off worth it? When is it overkill?
3. The encapsulation audit asks: "do public APIs contain `unsafe fn`?" For most crates, the answer should be no. What is special about the cases where exposing `unsafe fn` is appropriate?

---

## Exercise 7 -- Putting It Together

**Estimated time:** 15 minutes
**Topics covered:** integration of all module concepts

### Context

This final exercise combines every concept from Module 19 into a single program: a safe wrapper around a small piece of unsafe code that calls a C function and manages a raw pointer. The program demonstrates the full pattern: safe public API, FFI in the implementation, careful documentation, proper error handling.

### Task 7.1 -- The full program

Replace your `src/main.rs` with this complete program:

```rust
//! A safe wrapper around C's strlen function with related operations.

use std::ffi::{CStr, CString};
use libc::{c_char, size_t};

// ============================================================
// FFI declarations
// ============================================================

extern "C" {
    fn strlen(s: *const c_char) -> size_t;
    fn strdup(s: *const c_char) -> *mut c_char;
    fn free(p: *mut libc::c_void);
}

// ============================================================
// A safe wrapper around strlen
// ============================================================

/// Computes the byte length of a Rust string, using C's strlen.
///
/// This is purely for demonstration; in real Rust you would use
/// `str::len()` directly, which is faster and entirely safe.
pub fn c_strlen(s: &str) -> usize {
    // CString::new fails if the input contains a null byte.
    // For demonstration, we just return 0 in that case.
    let c_string = match CString::new(s) {
        Ok(cs) => cs,
        Err(_) => return 0,
    };

    // SAFETY: c_string.as_ptr() is non-null and points to a valid,
    // null-terminated C string. c_string is alive for the duration
    // of the call (it's bound to a named local variable). strlen
    // only reads through the pointer; it does not modify the data.
    unsafe { strlen(c_string.as_ptr()) }
}

// ============================================================
// A safe wrapper that owns a heap-allocated C string
// ============================================================

/// An owned, heap-allocated null-terminated C string,
/// constructed by calling C's strdup.
pub struct OwnedCString {
    ptr: *mut c_char,
}

impl OwnedCString {
    /// Allocates a new owned C string by duplicating the input.
    pub fn new(s: &str) -> Result<OwnedCString, &'static str> {
        let c_string = CString::new(s).map_err(|_| "input contains null byte")?;

        // SAFETY: c_string.as_ptr() is valid (c_string is alive).
        // strdup returns a pointer to newly allocated memory containing
        // a copy of the C string, or null on allocation failure.
        let ptr = unsafe { strdup(c_string.as_ptr()) };

        if ptr.is_null() {
            return Err("allocation failed");
        }

        Ok(OwnedCString { ptr })
    }

    /// Returns the length of the string in bytes (not counting the null terminator).
    pub fn len(&self) -> usize {
        // SAFETY: self.ptr is non-null (verified in `new`) and points
        // to a null-terminated C string allocated by strdup. strlen
        // reads up to but not including the null terminator.
        unsafe { strlen(self.ptr) }
    }

    /// Returns the contents as a Rust String, replacing invalid UTF-8.
    pub fn to_string_lossy(&self) -> String {
        // SAFETY: self.ptr is non-null (verified in `new`) and points
        // to a null-terminated C string that remains valid as long as
        // self is alive.
        let c_str = unsafe { CStr::from_ptr(self.ptr) };
        c_str.to_string_lossy().into_owned()
    }
}

impl Drop for OwnedCString {
    fn drop(&mut self) {
        // SAFETY: self.ptr was allocated by strdup, which returns
        // memory that must be freed with free(). No other code holds
        // a reference to the same allocation; after drop, the pointer
        // is not used again.
        unsafe {
            free(self.ptr as *mut libc::c_void);
        }
    }
}

// SAFETY: OwnedCString owns its memory exclusively. No reference to
// the buffer is shared. Moving the OwnedCString between threads
// transfers ownership of the underlying allocation. The C standard
// library's strlen and free do not have thread-affinity requirements.
unsafe impl Send for OwnedCString {}

// We do NOT implement Sync. Concurrent &OwnedCString access through
// to_string_lossy would not race (reading is fine), but we leave
// Sync off for simplicity; concurrent access is not needed.

// ============================================================
// Main: exercise everything
// ============================================================

fn main() {
    // Use c_strlen with a few inputs:
    println!("=== c_strlen demo ===");
    let inputs = ["", "hello", "Hello, world!", "Rust 🦀"];
    for s in &inputs {
        println!("c_strlen({s:?}) = {}", c_strlen(s));
    }

    // The OwnedCString type:
    println!("\n=== OwnedCString demo ===");
    let owned = OwnedCString::new("Hello from a strdup'd buffer").unwrap();
    println!("length: {}", owned.len());
    println!("contents: {:?}", owned.to_string_lossy());

    // Error cases:
    println!("\n=== Error cases ===");
    match OwnedCString::new("contains\0null") {
        Ok(_) => println!("unexpected success"),
        Err(e) => println!("expected error: {e}"),
    }

    println!("c_strlen with null byte: {}", c_strlen("has\0null"));

    // Demonstrate that OwnedCString is Send:
    println!("\n=== Send demo ===");
    let to_send = OwnedCString::new("transferred between threads").unwrap();
    let handle = std::thread::spawn(move || {
        println!("in thread: {:?}", to_send.to_string_lossy());
        to_send.len()
    });
    let result = handle.join().unwrap();
    println!("thread returned length: {result}");

    println!("\n=== Done ===");
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
=== c_strlen demo ===
c_strlen("") = 0
c_strlen("hello") = 5
c_strlen("Hello, world!") = 13
c_strlen("Rust 🦀") = 9

=== OwnedCString demo ===
length: 27
contents: "Hello from a strdup'd buffer"

=== Error cases ===
expected error: input contains null byte
c_strlen with null byte: 0

=== Send demo ===
in thread: "transferred between threads"
thread returned length: 27

=== Done ===
```

The program exercises every concept from the module:

- **Raw pointers:** `*mut c_char` stored inside `OwnedCString`.
- **The five powers:** dereferencing raw pointers (via `CStr::from_ptr`), calling unsafe functions (`strlen`, `strdup`, `free`), implementing an unsafe trait (`unsafe impl Send`).
- **FFI:** `extern "C"` block declaring `strlen`, `strdup`, `free`.
- **CString/CStr:** bridging Rust strings and C strings.
- **Safe abstractions:** every public function and method has a safe signature; unsafe is concentrated inside the implementation.
- **SAFETY comments:** every unsafe block has one explaining why it is correct.
- **Drop:** automatic cleanup of the heap allocation when `OwnedCString` goes out of scope.

### Task 7.2 -- Identify the patterns

In `lab15-notes.md`, identify each instance of the following patterns in the program above:

1. An `extern "C"` block declaring C functions.
2. A safe public function (`c_strlen`) that contains an unsafe block internally.
3. A safe struct (`OwnedCString`) whose public methods are all safe.
4. The use of `CString` to convert a Rust string to a null-terminated C string.
5. The use of `CStr::from_ptr` to read a null-terminated C string into Rust.
6. A `Drop` implementation that frees heap memory allocated by C.
7. An `unsafe impl Send` for a type containing a raw pointer.
8. The deliberate omission of `unsafe impl Sync` (with explanation).
9. SAFETY comments on every unsafe block.
10. The pattern of binding a `CString` to a local variable to extend its lifetime.

### Task 7.3 -- Predict and verify

For each of the following changes, predict what will happen and verify. Write your predictions in `lab15-notes.md` BEFORE running.

**Change A:** Remove the `unsafe impl Send for OwnedCString {}` line. What happens to the thread-spawning code in `main`?

**Change B:** In the `OwnedCString::new` method, remove the `if ptr.is_null()` check. What changes about the program's behavior under normal conditions? What could change in unusual conditions (low memory)?

**Change C:** In the `Drop` implementation, remove the `free(self.ptr as *mut libc::c_void)` call entirely. Does the program still compile? Does it still run correctly? What goes wrong, and when?

For each change, write your prediction first, then run and verify. Remember to revert each change before doing the next.

### Checkpoints

1. The `c_strlen` function has a safe signature (`fn c_strlen(s: &str) -> usize`) but contains an unsafe FFI call internally. Why is this the right structure compared to making `c_strlen` itself an `unsafe fn`?
2. The `OwnedCString` type uses `unsafe impl Send` but deliberately not `Sync`. What is the practical difference, and what guided the decision?
3. The `Drop` implementation frees the heap memory. Without it, what would happen, and when would the problem become visible?

---

## Summary and Reflection

You have now used every concept from Module 19 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Raw Pointers | `*const T`, `*mut T`, dereferencing | Creating raw pointers is safe; dereferencing requires unsafe. Pointer arithmetic is powerful but dangerous. |
| 2 -- The Five Superpowers | what `unsafe` enables | Unsafe unlocks five specific operations; everything else (borrow checker, lifetimes, types) still applies. |
| 3 -- FFI | `extern "C"`, CString/CStr | FFI is the most common use of unsafe; the temporary-CString pitfall is a real-world bug. |
| 4 -- Unsafe Traits | `unsafe impl Send/Sync` | Manual Send/Sync impls require careful justification; reflexive impls are dangerous. |
| 5 -- Safe Abstractions | encapsulation | The canonical pattern: safe public API, small unsafe blocks inside, SAFETY comments throughout. |
| 6 -- Auditing | cargo-geiger, Miri | Tools exist for tracking and verifying unsafe; running them is part of unsafe-conscious development. |
| 7 -- Integration | all of the above | Real safe wrappers around unsafe code combine all these patterns systematically. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab15-notes.md` before your next session.

1. Of the unsafe concepts in Module 19, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. The module argued that "unsafe is narrower than 'anything goes'": it unlocks five specific abilities, not unlimited license. After working through this lab, identify one specific situation where you might have reached for unsafe in the past (in another language) but Rust's design forced you to find a safer alternative. What does this say about the cost-benefit trade-off of strict-by-default safety?

3. The pattern of "safe abstractions over unsafe code" is what makes Rust simultaneously safe and capable. This puts trust in the library author's verification of safety. What kinds of bugs does this trust enable in practice? How does the Rust community manage this risk (with tools, conventions, code review, and community processes)?

---

*End of Lab 15*
