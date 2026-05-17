# Module 19: Unsafe Rust

## Module Overview

Most of Rust enforces strict safety guarantees: no data races, no use-after-free, no buffer overflows, no null-pointer dereferences. The borrow checker, lifetime system, and type system work together to prevent entire categories of bugs that plague C and C++ programs. This safety is what makes Rust distinctive.

But some things genuinely cannot be expressed under those rules: calling into C libraries, manipulating memory layouts directly, implementing data structures whose invariants the compiler cannot verify, hardware register access in embedded systems. For these cases, Rust provides `unsafe`: a way to opt out of certain checks, with the understanding that you take responsibility for upholding the invariants the compiler can no longer verify.

Three ideas are central:

1. **Unsafe does not turn off safety; it turns off five specific checks.** The borrow checker still runs. Type checking still runs. Pattern matching is still exhaustive. Memory is still freed when values go out of scope. `unsafe` unlocks exactly five additional abilities; everything else continues to work normally. This is a much narrower exception than "anything goes."

2. **Most unsafe code lives at boundaries.** Real Rust programs do not contain unsafe blocks scattered throughout. They concentrate unsafe in specific places: FFI bindings, fundamental data structures (`Vec`, `HashMap`, `Rc`), and system-level interfaces. These unsafe regions are encapsulated in safe abstractions; the rest of the code never sees `unsafe`. This pattern is what makes Rust simultaneously safe (most code is checked) and capable (some code can do anything).

3. **Unsafe is a contract with the compiler.** When you write `unsafe`, you are claiming "I have checked that the invariants hold." If you are wrong, the program has undefined behavior: anything can happen, including crashes, data corruption, security vulnerabilities. The compiler trusts you; it does not double-check. This shifts responsibility from compiler to programmer in exactly the regions where you have explicitly accepted it.

By the end of this module, students will:

- Recognize when `unsafe` is genuinely needed versus when it is an attempt to bypass legitimate compiler complaints.
- Understand the five things `unsafe` permits, and what each implies about the invariants you must maintain.
- Use raw pointers `*const T` and `*mut T` correctly, including the differences from references.
- Call C functions from Rust through FFI (foreign function interface).
- Recognize what makes a trait "unsafe to implement" and why some traits in the standard library carry this designation.
- Design safe abstractions that encapsulate unsafe code so callers never see it.
- Apply the practical guidelines for keeping unsafe blocks minimal, well-documented, and auditable.

This module is more cautionary than aspirational. Most production Rust code contains little or no unsafe. The most important skill is knowing when to reach for unsafe (rarely) and when to find a different design (usually).

> **A note on scope.** Writing genuinely safe unsafe code is a substantial skill, deeper than this module can fully teach. The Rust community has resources for this (notably the Rustonomicon at https://doc.rust-lang.org/nomicon/) for developers who need to write low-level libraries. This module is an introduction: enough to read unsafe code, recognize it in libraries you depend on, and write small amounts when necessary. For substantial unsafe code, expect to study further.

---

## A. When and Why to Use Unsafe

Most Rust developers write no unsafe code, ever. The standard library and ecosystem provide safe abstractions over the cases that need it; application code consumes those abstractions without writing unsafe itself.

Unsafe is genuinely needed in five categories:

### Calling Foreign Code (FFI)

C libraries do not follow Rust's safety rules. When Rust calls into a C function, the compiler cannot verify that the C function behaves correctly (no buffer overflows, no use-after-free, no race conditions). Rust marks foreign function calls as `unsafe` so that the call site explicitly accepts responsibility for the foreign code's behavior.

This is the most common reason to use unsafe in real applications. Almost every nontrivial Rust program depends on at least one library that wraps a C dependency: cryptography libraries (OpenSSL, ring), system libraries (libc, libsystem), graphics (Vulkan, OpenGL bindings), databases (SQLite, PostgreSQL clients with C bindings), and many others.

You typically use these through safe Rust wrappers: a crate like `openssl` provides Rust-style APIs that internally call OpenSSL's C functions through unsafe blocks. Your code is safe; the unsafe is concentrated in the crate's bindings.

### Hardware Access and Embedded Systems

Embedded Rust programs need to read and write memory-mapped hardware registers, manipulate interrupt vectors, and control DMA controllers. These operations have no concept of references or lifetimes; they are direct memory accesses with hardware-specific timing requirements.

The `cortex-m` family of crates (for ARM Cortex-M microcontrollers) and similar libraries encapsulate this complexity. Application code uses safe APIs like `gpio.set_high()` or `usart.send(byte)`. The implementations contain unsafe blocks that perform the actual register writes.

For developers not writing embedded code, this category is invisible. For embedded developers, it is constant.

### Implementing Fundamental Data Structures

`Vec<T>`, `HashMap<K, V>`, `String`, `Box<T>`, `Rc<T>`, and the standard library's core types contain unsafe internally. Their public APIs are entirely safe, but their implementations manipulate raw memory in ways the borrow checker cannot verify.

For example, `Vec::push` must:

- Allocate a new backing buffer when capacity is exceeded.
- Copy bytes from the old buffer to the new one.
- Manage uninitialized memory in the unused capacity.
- Free the old buffer.

These operations cannot be expressed using only safe Rust. The unsafe code is hidden inside `Vec`; users of `Vec` write only safe code (`v.push(x)`). The smart pointers covered in Module 15 (Rc, RefCell, Box) similarly contain internal unsafe that their safe APIs encapsulate.

If you are writing a new fundamental data structure (a lock-free queue, a memory allocator, a custom smart pointer), expect to use unsafe. If you are writing application code, you almost never need this.

### Implementing Unsafe Traits

A few traits in Rust are marked `unsafe` to implement: `Send`, `Sync`, `GlobalAlloc`, and a few others. Implementing them requires asserting properties the compiler cannot verify.

For example, marking your type `Sync` claims "this type is safe to share across threads via references." The compiler cannot verify this for types with custom thread-safety mechanisms; you must assert it manually with `unsafe impl Sync for MyType {}`.

This category is rare. The standard library does most of the work; auto-derived implementations cover the common cases. Manual `unsafe impl` appears in specialized libraries.

### Working with Memory Layout

Some operations need precise control over memory layout: parsing binary protocols, interfacing with shared memory, writing serialization formats that match a specific byte layout. The `repr` attributes (`#[repr(C)]`, `#[repr(packed)]`) let you control layout, but the actual byte-level access often requires unsafe.

This is rare in application code but common in protocol implementations and systems programming.

### What `unsafe` Is NOT For

A common misconception: unsafe is used to "bypass annoying compiler errors." This is wrong.

If the borrow checker rejects your code, the answer is almost always:

- Restructure the code to satisfy the borrow checker.
- Use a smart pointer pattern like `Rc<RefCell<T>>` (from Module 15).
- Use a channel for cross-thread communication (from Module 14).
- Use a different algorithm.

Reaching for unsafe to silence the borrow checker is a bug waiting to happen. The borrow checker's complaints are usually correct: they reflect real safety issues. Silencing them with unsafe does not fix the issues; it just hides them from the compiler.

The right test for "should I use unsafe?" is: "Does this operation genuinely need to do something the language cannot express safely?" If the answer is yes (calling foreign code, manipulating memory layout, etc.), unsafe is appropriate. If the answer is no (just trying to make the compiler happy), find a different approach.

---

## B. The Five Unsafe Superpowers

The `unsafe` keyword does not turn off all safety checks. It enables exactly five additional abilities. Knowing what these are makes the contract clear: when you write `unsafe`, you are accepting responsibility for these specific things and nothing else.

### Power 1: Dereference Raw Pointers

Raw pointers (`*const T` and `*mut T`) can be created in safe code. They can hold any address (including null, dangling, or unaligned). Reading or writing through a raw pointer requires unsafe.

```rust
fn main() {
    let x = 42;
    let ptr: *const i32 = &x;

    // Creating the pointer is safe:
    println!("pointer: {ptr:?}");

    // Dereferencing requires unsafe:
    unsafe {
        println!("value: {}", *ptr);
    }
}
```

The compiler cannot verify that the pointer is valid (points to allocated memory of the right type, properly aligned, not null). Dereferencing might cause a crash or undefined behavior. The unsafe block is your assertion that you have checked these conditions.

### Power 2: Call an `unsafe fn`

Functions can be declared `unsafe fn`. They typically have preconditions the compiler cannot check: alignment requirements, validity of input pointers, exclusive access to data.

```rust
unsafe fn dangerous(ptr: *const u8) -> u8 {
    *ptr
}

fn main() {
    let value = 42u8;
    let ptr = &value as *const u8;

    // Calling unsafe fn requires unsafe block:
    let read = unsafe { dangerous(ptr) };
    println!("read: {read}");
}
```

The function's `unsafe` marker declares "I have preconditions you must verify." Callers must do that verification before calling, and the unsafe block at the call site acknowledges the verification.

This is how the standard library exposes operations like `slice::get_unchecked` (skips bounds checking; caller must ensure the index is in range) or `String::from_utf8_unchecked` (skips UTF-8 validation; caller must ensure the bytes are valid UTF-8).

### Power 3: Access or Modify Mutable Static Variables

Mutable statics (`static mut`) are global mutable state. Reading or writing them is unsafe because nothing prevents concurrent access from multiple threads.

```rust
static mut COUNTER: u32 = 0;

fn increment() {
    unsafe {
        COUNTER += 1;
    }
}

fn main() {
    increment();
    increment();
    increment();

    unsafe {
        println!("counter: {COUNTER}");
    }
}
```

The unsafe is required because two threads accessing `COUNTER` concurrently would race. For single-threaded programs (like this example), the access is fine, but the compiler cannot verify single-threaded-ness, so it requires the unsafe block.

For multi-threaded code, the right tool is usually `AtomicU32` or `Mutex<u32>` (covered in Module 14), both of which have safe APIs. Direct `static mut` should be very rare.

### Power 4: Implement an `unsafe` Trait

Some traits are declared `unsafe trait`. Implementing them requires `unsafe impl`. The traits' documentation typically lists invariants the implementor must uphold.

```rust
unsafe trait Aligned {
    fn alignment(&self) -> usize;
}

struct MyType;

unsafe impl Aligned for MyType {
    fn alignment(&self) -> usize {
        16
    }
}
```

The `unsafe impl` is your assertion that you have read the trait's invariants and your implementation upholds them. The compiler does not verify this; you are responsible.

This category is covered more thoroughly in Section E.

### Power 5: Access Fields of `union`s

Rust supports `union` types (similar to C unions): a type where multiple fields share the same memory. Reading any field is unsafe because the compiler cannot verify which field is currently valid.

```rust
union IntOrFloat {
    int_val: u32,
    float_val: f32,
}

fn main() {
    let u = IntOrFloat { int_val: 1065353216 };

    // Reading any field requires unsafe:
    unsafe {
        println!("as int: {}", u.int_val);
        println!("as float: {}", u.float_val);
    }
}
```

Unions are rare in Rust. Most code uses enums (which track which variant is active) instead. Unions appear mainly in FFI with C code that uses unions, or in specialized memory-layout-sensitive code.

### What `unsafe` Does NOT Disable

The five powers above are the ENTIRE list. Everything else still works:

- **The borrow checker** still verifies that references are valid and respect aliasing rules.
- **Lifetime checking** still tracks reference lifetimes.
- **Type checking** still verifies that values have the expected types.
- **Exhaustive pattern matching** still requires you to handle all cases.
- **Move semantics** still apply.
- **Drop runs** still happens at end of scope.

In particular: writing `unsafe` does NOT let you ignore the borrow checker. You can still get borrow-check errors inside unsafe blocks. The borrow checker is not optional; it always runs.

This is a subtle but important point. Unsafe is much narrower than "no rules apply." It is "these five specific things are unlocked; everything else is still enforced."

---

## C. Raw Pointers: `*const T` and `*mut T`

Raw pointers are the most common ingredient in unsafe code. They are similar to references but with weaker guarantees.

### Differences From References

References (`&T` and `&mut T`) in Rust come with strong guarantees:

- They are never null.
- They are always aligned.
- They point to valid initialized data.
- They respect aliasing rules (multiple `&T` OR one `&mut T`, never mixed).
- They are bounded by lifetimes.

Raw pointers (`*const T` and `*mut T`) have none of these guarantees:

- They can be null.
- They can be unaligned.
- They can point to freed memory, uninitialized memory, or arbitrary addresses.
- They can alias freely (two `*mut T` to the same location is allowed).
- They have no lifetime tracking.

The relaxation is what makes raw pointers useful for FFI and low-level code, where these constraints cannot be maintained or are externally enforced.

### Creating Raw Pointers

Raw pointers can be created safely. Only dereferencing them requires unsafe:

```rust
let x = 42;
let r = &x;                          // a reference
let const_ptr: *const i32 = &x;      // a const raw pointer
let mut_ptr: *mut i32 = &mut 0;       // a mut raw pointer (from a temporary)

// You can also create from a reference:
let from_ref: *const i32 = r;

// All of the above is safe code.
```

You can have multiple raw pointers to the same data, including a mix of const and mut, without the compiler complaining. The compiler simply does not track raw pointers the way it tracks references.

### Reading and Writing Through Raw Pointers

To use a raw pointer, you dereference it inside an unsafe block:

```rust
let x = 42;
let ptr: *const i32 = &x;

let value = unsafe { *ptr };
println!("value: {value}");
```

For a `*mut T`, you can also write:

```rust
let mut x = 42;
let ptr: *mut i32 = &mut x;

unsafe {
    *ptr = 100;
}

println!("x: {x}");
```

The unsafe block is your assertion: "I have verified that this pointer is valid: non-null, aligned, pointing to a properly-typed value, not aliasing any other access that would cause a data race." The compiler trusts the assertion.

### Pointer Arithmetic

You can do arithmetic on raw pointers (offsets in elements, not bytes):

```rust
let array = [10, 20, 30, 40, 50];
let ptr: *const i32 = array.as_ptr();

unsafe {
    for i in 0..5 {
        let elem_ptr = ptr.add(i);     // offset by i elements
        println!("element {i}: {}", *elem_ptr);
    }
}
```

The `add(i)` method advances the pointer by `i` elements. For `*const i32`, that is `i * 4` bytes (assuming 4-byte alignment). The method handles the size calculation correctly.

Pointer arithmetic is how you traverse arrays manually, but it is also how you can step off the end and read invalid memory. The unsafe block is again your assertion that you have not done this.

### Null Pointers

Raw pointers can be null. Dereferencing a null pointer is undefined behavior.

```rust
let null: *const i32 = std::ptr::null();

// This is undefined behavior:
// let _ = unsafe { *null };

if !null.is_null() {
    unsafe {
        println!("{}", *null);
    }
}
```

The `is_null()` method checks for null. You should always check before dereferencing if there is any chance the pointer might be null.

The `std::ptr::null()` and `std::ptr::null_mut()` functions create null pointers explicitly. They are useful when calling FFI functions that expect "no value" as a null parameter.

### Converting Between References and Raw Pointers

You can convert between references and raw pointers:

```rust
let value = 42i32;
let reference: &i32 = &value;
let raw: *const i32 = reference;

// Convert back (unsafe):
let back: &i32 = unsafe { &*raw };
```

Going from reference to raw pointer is safe. Going back (creating a reference from a raw pointer) is unsafe, because the compiler cannot verify the pointer's validity.

When you do this conversion, you typically need to take care of lifetimes manually. The reference created from the raw pointer has whatever lifetime you assign; the compiler will not warn if you outlive the actual data.

### Pointers as Function Parameters

In FFI and low-level APIs, functions often take raw pointers:

```rust
fn process(data: *const u8, len: usize) {
    if data.is_null() {
        return;
    }

    unsafe {
        for i in 0..len {
            let byte = *data.add(i);
            print!("{byte} ");
        }
    }
    println!();
}

fn main() {
    let bytes = [72u8, 101, 108, 108, 111];
    process(bytes.as_ptr(), bytes.len());
}
```

Output: `72 101 108 108 111`

The function accepts a raw pointer and a length. This is the standard C-style way to pass arrays. In pure Rust, you would use a slice (`&[u8]`), but for FFI compatibility, raw pointers are necessary.

### When You Need Raw Pointers

In application code, almost never. The cases where they are essential:

- **FFI:** C functions that take pointers must be called with raw pointers.
- **Custom data structures:** Implementing a linked list, tree, or other structure where the internal representation uses raw pointers.
- **Memory manipulation:** Implementing memory allocators, doing low-level I/O with buffers.
- **Interfacing with hardware:** Reading or writing to memory-mapped registers.

For everything else, references and Rust's safe abstractions (Box, Rc, Arc, Vec, etc.) are the right tools.

---

## D. Calling Unsafe Functions and External Code (FFI)

The most common use of unsafe in real Rust applications is FFI: calling functions from C libraries (or libraries written in other languages that expose a C-compatible interface).

### The `extern "C"` Block

To call C functions, you declare them in an `extern "C"` block:

```rust
extern "C" {
    fn abs(input: i32) -> i32;
    fn strlen(s: *const u8) -> usize;
}

fn main() {
    // Calling extern functions is unsafe:
    let absolute = unsafe { abs(-42) };
    println!("absolute: {absolute}");
}
```

The `extern "C"` specifies the calling convention (how arguments are passed). C is the most common; other conventions exist for specific platforms (`extern "stdcall"` on Windows, etc.).

Functions declared in `extern` blocks are automatically `unsafe`. Calling them requires an unsafe block, because the foreign code does not follow Rust's safety rules.

### Linking Against C Libraries

To use functions from a C library, the Rust build must link against it. For widely-available libraries (libc, the C standard library), this happens automatically. For other libraries, you add a `build.rs` script or use a "-sys" crate (a community convention: `<library>-sys` crates provide raw bindings to a C library).

For example, to use libc's `abs` function (which has more functions than the standard math operators), you would typically use the `libc` crate:

```toml
[dependencies]
libc = "0.2"
```

```rust
use libc::abs;

fn main() {
    let absolute = unsafe { abs(-42) };
    println!("absolute: {absolute}");
}
```

The `libc` crate provides Rust declarations for thousands of C functions and types. It is widely used for FFI projects.

### Passing Strings to C

C strings are null-terminated byte arrays (`const char*`). Rust's `String` is not null-terminated. To pass a string to C, you use `CString` (an owned null-terminated string):

```rust
use std::ffi::CString;

extern "C" {
    fn puts(s: *const u8) -> i32;
}

fn main() {
    let rust_string = "Hello from Rust!";
    let c_string = CString::new(rust_string).expect("contains null byte");

    unsafe {
        puts(c_string.as_ptr() as *const u8);
    }
}
```

Output: `Hello from Rust!`

`CString::new(s)` allocates a new null-terminated copy of the string. If the input contains a null byte, this fails (because the resulting C string would be truncated unexpectedly).

`c_string.as_ptr()` gives you a raw pointer to the null-terminated data. The pointer is valid as long as the `CString` is alive. When `c_string` is dropped, the memory is freed; using the pointer afterward is undefined behavior.

This is a common pitfall in FFI: handing out a pointer to data that gets dropped. The fix is to ensure the data outlives all uses of the pointer. Sometimes this means holding the `CString` in a variable instead of using a temporary:

```rust
// WRONG: temporary dropped before the call ends
unsafe {
    puts(CString::new("hello").unwrap().as_ptr() as *const u8);
}

// Right: name the CString to extend its lifetime:
let s = CString::new("hello").unwrap();
unsafe {
    puts(s.as_ptr() as *const u8);
}
```

(In the WRONG example, modern Rust may actually keep the temporary alive long enough, but this is undefined; do not rely on it.)

### Receiving Strings From C

Going the other direction: a C function returns a `*const c_char`. To convert to Rust:

```rust
use std::ffi::CStr;

extern "C" {
    fn getenv(name: *const u8) -> *const u8;
}

fn get_env(name: &str) -> Option<String> {
    let c_name = std::ffi::CString::new(name).ok()?;

    unsafe {
        let result = getenv(c_name.as_ptr() as *const u8);
        if result.is_null() {
            None
        } else {
            let c_str = CStr::from_ptr(result as *const i8);
            Some(c_str.to_string_lossy().into_owned())
        }
    }
}

fn main() {
    if let Some(home) = get_env("HOME") {
        println!("HOME = {home}");
    }
}
```

`CStr::from_ptr(p)` creates a `&CStr` from a pointer to a null-terminated C string. The `&CStr` is valid as long as the underlying memory is. The function `to_string_lossy()` converts to a Rust string, replacing invalid UTF-8 with replacement characters.

The unsafe block is required because the compiler cannot verify:

- The pointer is non-null (we check explicitly).
- The pointer points to a null-terminated string.
- The memory pointed to is valid for the duration of the read.

We have to ensure all three conditions hold; the compiler trusts our assertion.

### Defining Functions for C to Call

Going the other direction: exposing Rust functions to C. Use `extern "C"` on the function definition, with `#[no_mangle]` to prevent name mangling:

```rust
#[no_mangle]
pub extern "C" fn add_one(x: i32) -> i32 {
    x + 1
}
```

This function can be called from C code as:

```c
extern int add_one(int x);
```

The function does not need to be `unsafe`. It is callable from unsafe contexts (the C side), but the Rust side does not need to write `unsafe`. The function's body is just regular Rust code.

This is how Rust libraries can be consumed by non-Rust programs: write the function in Rust, expose it with `extern "C"` and `#[no_mangle]`, build as a dynamic or static library, link from C/C++/Python/etc.

### FFI Errors and Common Pitfalls

FFI is one of the most error-prone areas in Rust. Common pitfalls:

- **Dangling pointers.** Passing a pointer to data that gets dropped before the C call completes (the temporary issue from earlier).
- **Mismatched calling conventions.** Declaring a function as `extern "C"` when the actual library uses a different convention. Crashes or wrong behavior result.
- **Wrong types.** C's `int` is not always `i32`; `size_t` is not always `usize` on all platforms. The `libc` crate provides correct type aliases.
- **Null pointers.** Many C APIs return null on error or accept null as "no value." Forgetting to check causes crashes.
- **Lifetime confusion.** A C function returning a pointer might return a pointer to memory you must free (caller responsibility) or pointer to memory owned by the C library (do not free). The documentation tells you which; misreading causes leaks or double-frees.
- **Thread safety assumptions.** C libraries often have undocumented threading rules. Calling a non-thread-safe library from multiple threads is undefined behavior.

For most projects, you should use existing "safe bindings" crates rather than writing FFI yourself. The `openssl` crate, for example, provides a safe Rust API over the OpenSSL C library, hiding all the FFI details. Writing your own bindings is appropriate only when:

- No suitable safe bindings exist.
- You are writing the bindings themselves (perhaps to publish as a crate).
- You have a specialized need that the existing bindings do not address.

Even then, prefer to encapsulate the unsafe code so the rest of your application can be entirely safe.

---

## E. Implementing Unsafe Traits

A few traits in Rust are declared `unsafe trait`. Implementing them requires `unsafe impl`. The unsafe marker says "the implementation must uphold invariants the compiler cannot verify."

### `Send` and `Sync`

The most common unsafe traits are `Send` and `Sync`. From Module 14, you know what they do:

- `Send` means the type can be transferred between threads.
- `Sync` means `&T` can be shared between threads.

Most types implement these automatically based on their fields (if all fields are `Send`, the type is `Send`; same for `Sync`). The compiler handles auto-derivation.

But some types need manual implementations. For example, suppose you write a type that contains a raw pointer:

```rust
struct MyBuffer {
    data: *mut u8,
    len: usize,
}
```

By default, this type is `!Send` (raw pointers are not `Send`). If you know that the data is safely transferable between threads (because of how you allocate and manage it), you can declare it manually:

```rust
unsafe impl Send for MyBuffer {}
unsafe impl Sync for MyBuffer {}
```

The `unsafe impl` says "I have verified that this type can safely cross thread boundaries; trust me."

If the assertion is wrong (the buffer is not actually thread-safe), code using your type can have data races. The compiler will not catch this; the unsafe is exactly the place where the compiler stops checking.

For most types, do NOT manually implement `Send` or `Sync`. The auto-derivation is correct. Only override when you have specific reason to know that your type's thread-safety is different from what its fields suggest.

### `GlobalAlloc`

The `GlobalAlloc` trait defines a global memory allocator. Implementing it lets you replace the default allocator (typically the system malloc) with a custom one.

```rust
use std::alloc::{GlobalAlloc, Layout};

struct MyAllocator;

unsafe impl GlobalAlloc for MyAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // ... allocate memory according to layout ...
        std::ptr::null_mut()
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // ... free memory ...
    }
}

#[global_allocator]
static ALLOCATOR: MyAllocator = MyAllocator;
```

The trait is unsafe because the implementor must:

- Return correctly-aligned memory of the requested size from `alloc`.
- Accept the same `Layout` in `dealloc` that was returned from `alloc`.
- Handle errors appropriately (return null on failure).

A bug in the allocator can cause heap corruption affecting the entire program. The `unsafe impl` is your assertion that you have done this correctly.

Custom allocators are rare. They appear in specialized systems: embedded code without an OS, games with custom memory management, security-critical code with hardened allocators. Most projects use the default.

### When to Use Unsafe Traits

The rule of thumb: a trait should be `unsafe` to implement when:

- Implementations have invariants the compiler cannot verify.
- Violating those invariants causes undefined behavior or memory corruption.
- The compiler relies on the trait's contract being upheld (e.g., depending on `Send` to allow code to run on multiple threads).

If a trait is just "implement this method correctly," it should be a regular trait. The `unsafe` keyword should be reserved for traits whose misuse breaks memory safety.

In your own code, you will rarely declare `unsafe trait`. The standard library has the main examples. Specialized libraries (especially low-level ones) sometimes add their own.

If you write a library that has invariants needing manual verification, consider whether the right model is:

- An `unsafe trait` (caller asserts they uphold invariants).
- A regular trait with safe defaults (callers cannot break things).
- A different design entirely (avoiding the issue).

Each has trade-offs. The `unsafe trait` model gives users power at the cost of complexity; sometimes that is the right trade-off.

### Reading Unsafe Trait Implementations

When reading code that contains `unsafe impl SomeTrait for MyType`, recognize it as a contract:

- The trait has documented invariants.
- The implementor has manually verified those invariants.
- The compiler does not verify them; bugs in this code can corrupt memory.

This is one of the places where you should pay attention to surrounding documentation. The `unsafe` keyword is signaling "this is a place where verification matters." Skimming past it without understanding what is being asserted is risky.

---

## F. Writing Safe Abstractions Over Unsafe Code

The pattern that makes Rust simultaneously safe and capable: encapsulate unsafe code behind safe APIs. The Rust standard library does this extensively; idiomatic Rust libraries do too.

### The General Pattern

You have a low-level operation that requires unsafe (raw pointer manipulation, FFI call, custom memory management). You wrap it in a function with a safe signature. The function's body contains unsafe blocks, but its API is safe.

```rust
fn process_byte_at(slice: &[u8], index: usize) -> Option<u8> {
    if index >= slice.len() {
        return None;
    }

    // Safe to access; we just checked bounds:
    unsafe {
        Some(*slice.as_ptr().add(index))
    }
}

fn main() {
    let data = [10, 20, 30, 40, 50];

    println!("{:?}", process_byte_at(&data, 2));      // Some(30)
    println!("{:?}", process_byte_at(&data, 99));     // None
}
```

The function's signature `fn process_byte_at(slice: &[u8], index: usize) -> Option<u8>` is entirely safe. The caller does not see unsafe; they just call the function. Inside, the function uses unsafe to do its work, but it has manually verified the precondition (index in bounds) that the unsafe operation requires.

This is exactly what `slice::get` does in the standard library. It uses unchecked indexing internally but exposes a safe API.

### Why This Pattern Works

The unsafe code is concentrated in one place. To audit your program for safety:

1. Identify all `unsafe` blocks. (Tools like `cargo geiger` help.)
2. For each, verify the invariants are upheld.
3. Trust everything else.

The smaller the unsafe regions, the easier the audit. The larger the safe regions around them, the more code is automatically verified by the compiler.

This is the opposite of C, where every piece of code might be unsafe. There is no equivalent of "this region is safe to ignore" in C; you have to audit everything for memory errors. In Rust, the borrow checker covers the safe code, and you audit only the unsafe.

### Example: A Safe Wrapper Around a C Function

Suppose you are wrapping a hypothetical C function:

```c
int divide(int dividend, int divisor, int* result);
// Returns 0 on success, -1 on error (e.g., division by zero).
// On success, *result contains the quotient.
```

The Rust wrapper:

```rust
extern "C" {
    fn divide(dividend: i32, divisor: i32, result: *mut i32) -> i32;
}

pub fn safe_divide(dividend: i32, divisor: i32) -> Result<i32, &'static str> {
    if divisor == 0 {
        return Err("division by zero");
    }

    let mut result = 0;
    let return_code = unsafe {
        divide(dividend, divisor, &mut result as *mut i32)
    };

    if return_code == 0 {
        Ok(result)
    } else {
        Err("division failed")
    }
}

fn main() {
    match safe_divide(10, 2) {
        Ok(q) => println!("quotient: {q}"),
        Err(e) => println!("error: {e}"),
    }

    match safe_divide(10, 0) {
        Ok(q) => println!("quotient: {q}"),
        Err(e) => println!("error: {e}"),
    }
}
```

The `safe_divide` function has a fully safe signature (`fn safe_divide(dividend: i32, divisor: i32) -> Result<i32, &'static str>`). It:

- Validates the divisor isn't zero (preventing one class of error).
- Allocates a local variable for the result.
- Calls the unsafe FFI function.
- Returns a `Result`.

Callers never see unsafe. They get a Rust-idiomatic API with proper error handling. The unsafe is sealed off in one tiny region.

This is the model for FFI wrappers throughout the Rust ecosystem. Every C-binding crate looks roughly like this: small unsafe blocks calling foreign code, big safe APIs that the rest of the program consumes.

### Example: A Safe Wrapper Around Raw Pointers

For a more substantive example, here is a sketch of how `Vec` exposes safe `push`:

```rust
// Simplified pseudocode for Vec::push:
pub fn push(&mut self, value: T) {
    if self.len == self.capacity {
        self.grow();    // allocates a new buffer (uses unsafe internally)
    }

    // Write the value into the buffer at index `len`:
    unsafe {
        let dest = self.ptr.add(self.len);
        std::ptr::write(dest, value);
    }

    self.len += 1;
}
```

The `push` method takes `&mut self` and a value. From outside, it is a perfectly normal safe method. Inside, it uses raw pointer manipulation to write the value at the correct location.

The safety argument:

- `self.ptr.add(self.len)` is in bounds because we ensured `len < capacity` (growing if needed).
- The destination memory is uninitialized; `ptr::write` correctly handles this (it does not run Drop on the previous value).
- `self.len` is updated after the write, so any future operation sees the correct length.

If you mess up any of these, you have undefined behavior. The standard library implementers have manually verified the argument; you can use `Vec::push` safely because of their work.

### Documenting Unsafe Code

When writing unsafe code, document the invariants you are relying on. A common pattern:

```rust
pub fn safe_function(input: &[u8], index: usize) -> u8 {
    // SAFETY: We check that `index` is in bounds before
    // calling `get_unchecked`. The slice's pointer is valid
    // by the type system; `index < input.len()` ensures we
    // do not read past the end.
    if index < input.len() {
        unsafe { *input.as_ptr().add(index) }
    } else {
        0
    }
}
```

The `// SAFETY:` comment explains why the unsafe block is safe. This is the conventional way to document unsafe code in Rust. Anyone reviewing the code can read the SAFETY comment to understand the invariants.

Conversely, if you call an unsafe function, the SAFETY comment explains why the call is safe:

```rust
unsafe fn dangerous(ptr: *const u8) -> u8 {
    *ptr
}

fn safe_caller(slice: &[u8]) -> u8 {
    if slice.is_empty() {
        return 0;
    }
    // SAFETY: We checked that the slice is non-empty,
    // so `slice.as_ptr()` is a valid pointer to an
    // initialized u8.
    unsafe { dangerous(slice.as_ptr()) }
}
```

Real Rust code is full of SAFETY comments wherever unsafe appears. They are essential for code review and for future readers. An unsafe block without an explanation is a red flag in code review.

---

## G. Guidelines: Minimizing and Auditing Unsafe Blocks

The practical advice for working with unsafe code.

### Keep Unsafe Blocks Small

The smaller an unsafe block, the easier it is to audit. A block that contains 100 lines of code has many places where invariants could be violated; a block with 1 line has one place.

Bad:

```rust
unsafe {
    let p = get_pointer();
    if validate_pointer(p) {
        let value = *p;
        process(value);
        let q = transform(p);
        *q = 42;
        // ... 50 more lines ...
    }
}
```

Better:

```rust
let p = get_pointer();
if validate_pointer(p) {
    let value = unsafe { *p };
    process(value);
    let q = transform(p);
    unsafe { *q = 42; }
    // ... 50 more lines outside unsafe ...
}
```

The minimal unsafe blocks make clear exactly which operations require the unsafe. The 50 lines of regular code can be checked by the compiler; only the 2 unsafe operations need manual review.

### Write SAFETY Comments

Every unsafe block should have a comment explaining why the operation is safe. The comment should reference:

- What invariants the operation requires.
- Why those invariants hold at this call site.
- Any assumptions about the caller or the environment.

Without these comments, future readers (including future you) cannot tell whether the unsafe is correct or just left over from earlier development. A code review of any new unsafe block should ask: "Where is the SAFETY comment?"

### Use Existing Safe Wrappers

For almost every operation that requires unsafe, someone has written a safe wrapper. The standard library covers many cases (raw pointer manipulation, FFI types). Crates exist for almost every C library you might want to use. Before writing your own unsafe code, search for an existing safe abstraction.

Even if the existing wrapper does not do exactly what you want, it might be close enough. Using a well-tested safe abstraction is much better than writing your own unsafe code.

### Use `cargo geiger` to Audit

The `cargo-geiger` tool counts unsafe code usage across your project and its dependencies. Running it shows which crates contain unsafe and how much.

This is useful for:

- **Auditing your own code.** Knowing how much unsafe you have.
- **Evaluating dependencies.** A crate with heavy unsafe usage needs more scrutiny.
- **Tracking trends.** Watching how unsafe usage evolves over time.

A reasonable goal: zero unsafe in your application code, with unsafe usage concentrated in well-known, well-audited crates.

### Run Miri Where Possible

Miri is an interpreter for Rust's MIR (intermediate representation) that detects undefined behavior at runtime. It catches bugs that escape the compiler:

- Use-after-free.
- Out-of-bounds memory access.
- Data races (with some configurations).
- Type confusion.

You run Miri as part of your test suite:

```bash
cargo +nightly miri test
```

Miri is slow (typically 10-100x slower than native execution) and not all code can run under it (some FFI cannot). But for code with significant unsafe, running tests under Miri catches a lot of bugs.

For application code with no unsafe, Miri provides little benefit. For library code with unsafe, Miri is one of the strongest tools available.

### Prefer Atomic Types and Sync Primitives

When you need shared mutable state across threads, reach for safe abstractions first:

- `AtomicI32`, `AtomicBool`, `AtomicUsize` for atomic operations.
- `Mutex<T>`, `RwLock<T>` for guarded access.
- Channels for message-passing.

Manual `static mut` or unsafe pointer manipulation should be very rare. The standard library provides safe alternatives for nearly every common case.

### Test Thoroughly

Code that contains unsafe needs more thorough testing than code that does not. Reasons:

- **The compiler is not helping.** In unsafe code, the compiler does not catch bugs that it normally would.
- **Bugs are subtle.** Undefined behavior can be invisible in testing (program appears to work) and devastating in production (rare crashes, security vulnerabilities).
- **Bugs are not local.** Undefined behavior can manifest far from its source, making debugging hard.

For libraries with unsafe code: write tests (covered in Module 16), run Miri, fuzz test (using `cargo-fuzz`), and have multiple people review. The investment is worth it; the alternative is users hitting bugs in production.

For application code: avoid unsafe entirely if possible. If you must use it, isolate it in small, well-tested helpers.

### Final Word of Caution

Unsafe is a powerful tool with sharp edges. The Rust community has invested enormous effort in keeping unsafe usage minimal and well-audited. Standard library types like `Vec`, `Box`, `HashMap`, and `String` contain unsafe code, but it has been reviewed by many people over many years.

When you write your own unsafe, you are taking on that same responsibility, but without the same resources. A single bug can corrupt memory, leak data, or crash production systems. The Rust safety guarantees that make most code reliable are the things you have explicitly opted out of.

The pragmatic approach:

- **Default to safe code.** Use safe abstractions for everything you can.
- **Reach for unsafe rarely.** Only when no safe alternative exists.
- **Encapsulate.** Build safe APIs over unsafe internals.
- **Document.** SAFETY comments for every unsafe block.
- **Test.** More thoroughly than safe code.
- **Audit.** Periodically review unsafe blocks.

This is the practice that has kept Rust's safety reputation intact despite the existence of `unsafe`. Most code is safe; the parts that are not are concentrated, documented, and audited.

---

## Module Summary

- `unsafe` does not turn off all safety; it enables exactly five additional operations (deref raw pointers, call unsafe functions, access mutable statics, implement unsafe traits, access union fields).
- The borrow checker, lifetime checker, and type system still run in unsafe blocks.
- Raw pointers (`*const T`, `*mut T`) are similar to references but without lifetime tracking, alignment guarantees, or aliasing rules. They can be created safely but dereferenced only in unsafe code.
- FFI (calling C from Rust) is the most common use of unsafe in real applications. `extern "C"` blocks declare foreign functions; calling them is unsafe.
- A few traits are declared `unsafe trait` (`Send`, `Sync`, `GlobalAlloc`). Implementing them requires `unsafe impl` because the compiler cannot verify the invariants.
- Safe abstractions over unsafe code is the pattern that makes Rust both safe and capable. Unsafe code is concentrated in small regions; the rest of the program is safe.
- Best practices: minimize unsafe blocks, write SAFETY comments, use existing safe wrappers, run Miri, test thoroughly.
- The five powers do not include "bypassing the borrow checker." If the borrow checker rejects your code, the answer is almost always a different design, not `unsafe`.

## Discussion Questions

1. The module argues that `unsafe` is "narrower than 'anything goes'": it unlocks five specific abilities and nothing more. From a language design perspective, what does this scoping accomplish that a more permissive unsafe (where the compiler stopped checking everything) would not?

2. Many Rust projects depend on at least one C library through FFI, which inevitably means some unsafe code in the dependency graph. How should developers evaluate the safety implications of using FFI-heavy crates? What does "audited unsafe" mean in practice?

3. The pattern of "safe abstractions over unsafe code" lets a library expose a safe API while internally using unsafe. This puts trust in the library author's verification of safety. What kinds of bugs does this trust enable, and how does the Rust community manage this risk in practice (with tools, conventions, and community processes)?

