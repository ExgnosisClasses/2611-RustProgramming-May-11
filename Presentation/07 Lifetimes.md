# Module 7: Lifetimes

## Module Overview

Lifetimes are the syntax that makes the borrow checker's reasoning visible. Every reference in Rust has a lifetime: a region of code during which the reference is guaranteed to be valid. In most cases, the compiler infers lifetimes automatically and you never see them. In other cases, the compiler needs you to spell out the relationships between references, and that is what lifetime annotations are for.

The intimidating-looking syntax (`'a`, `'static`, `&'a T`) is actually the simplest part of the topic. The harder part is understanding what the syntax describes: the connection between a reference and the data it refers to. Once you have that mental model, the syntax becomes a way of expressing it.

By the end of this module, students will:

- Understand what a lifetime is and why every reference has one.
- Read function signatures with lifetime annotations and understand what they require of callers.
- Add lifetime annotations to structs that hold references.
- Recognize the `'static` lifetime and know when it applies.
- Apply the three lifetime elision rules to predict when annotations are required.
- Understand the basic idea of lifetime subtyping and why it makes the system flexible.

This module builds directly on Module 6. The borrow checker rules you saw there are enforced through lifetimes; this module shows how. After this module, the rest of the language (generics, traits, error handling, concurrency) will be much smoother because you will be fully equipped to read any function signature.

> **A note on difficulty.** Lifetimes have a reputation for being hard. They are easier than they look, but they require holding several ideas in mind at once: the existence of a reference, the data it refers to, the scope where each is valid, and the relationship between them. Read this module twice if you need to. The lab gives you concrete examples to work through, which is where the concepts solidify.

---

## A. What Lifetimes Are and Why They Exist

### Recap from Module 6

In Module 6 you saw that every reference in Rust must point to valid data. The borrow checker enforces this by tracking how long each reference must remain valid and rejecting code where a reference might outlive what it points to.

```rust
fn dangle() -> &String {
    let s = String::from("hello");
    &s                                  // ERROR: s is dropped when the function returns
}
```

The compiler rejects this because the returned reference would point to memory that has been freed. The mechanism it uses to detect this is called the **lifetime system**.

### The Definition

A **lifetime** is a compile-time region of code during which a reference is guaranteed to be valid. Every reference has a lifetime, even when you do not write one explicitly.

Lifetimes are not runtime values. They do not exist in the compiled binary. They exist only during compilation, as labels the compiler attaches to references to track their validity. After the compiler has verified the program, the lifetimes are erased; the binary contains only ordinary pointers.

### A Simple Example

```rust
fn main() {
    let s = String::from("hello");      // s is created here; its lifetime begins
    let r = &s;                          // r borrows from s; r's lifetime begins

    println!("{r}");                     // r is valid because s is still valid
}                                        // r and s both end of scope here
```

The variable `s` is alive from its declaration to the end of `main`. The reference `r` is alive from its declaration to its last use. Because `r`'s lifetime is contained within `s`'s lifetime, the reference is always valid.

Now consider a case where this is violated:

```rust
fn main() {
    let r;                               // r will be a reference
    {
        let s = String::from("hello");   // s lives only inside this inner block
        r = &s;                          // r borrows from s
    }                                    // s is dropped here
    println!("{r}");                     // ERROR: r references freed memory
}
```

The compiler rejects this. The reference `r` would outlive the data it points to. In the language of lifetimes: `s`'s lifetime is shorter than `r`'s, so `r` cannot validly borrow from `s`.

### Why Lifetimes Are Visible Sometimes

Most of the time you do not write lifetimes. The compiler infers them. You see them only in places where the compiler needs help understanding the relationships between references.

Specifically, you write lifetime annotations when:

- A function takes more than one reference and the return type is also a reference, and the compiler cannot determine which input the output is borrowed from.
- A struct holds a reference (covered in Section C).
- You explicitly want to require that a reference live for the entire program (`'static`).

In all other cases, the compiler figures it out. The next sections introduce the syntax and the rules.

### A Mental Model

For the rest of this module, hold this picture in mind:

- Every value has a region of code where it is alive.
- Every reference has a lifetime: the region during which it is guaranteed to be valid.
- A reference's lifetime must be contained within the lifetime of the data it points to.
- The compiler tracks all of this and rejects programs that would have dangling references.
- Lifetime annotations let you express relationships between lifetimes when the compiler needs help.

The system is more strict than it sounds when stated abstractly. In practice, most code falls into patterns the compiler recognizes, and you rarely need to write annotations.

---

## B. Lifetime Annotations in Functions

### When You Need Annotations

Lifetime annotations are required on function signatures in specific cases. The clearest example is a function that takes two references and returns a reference:

```rust
fn longer(s1: &str, s2: &str) -> &str {
    if s1.len() >= s2.len() { s1 } else { s2 }
}
```

This does not compile. The compiler says:

```
error[E0106]: missing lifetime specifier
  |
  | fn longer(s1: &str, s2: &str) -> &str {
  |                                  ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but
          the signature does not say whether it is borrowed from `s1` or `s2`
```

The function returns a reference, but the compiler cannot determine whose lifetime the returned reference shares. Is the return value borrowed from `s1`? From `s2`? From either, depending on input?

The caller needs to know, because the caller has to make sure the value the reference points to remains alive for as long as the reference is used. Without that information, the compiler cannot verify the caller's code is safe.

### The Syntax

The fix is to declare a lifetime parameter and use it to express the relationship:

```rust
fn longer<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() >= s2.len() { s1 } else { s2 }
}
```

Three changes were made:

- `<'a>` after the function name declares a lifetime parameter named `'a`. This is a generic parameter, like a type parameter, but for lifetimes.
- `&'a str` says the parameter is a reference with lifetime `'a`.
- `-> &'a str` says the return value is a reference with lifetime `'a`.

The single quote `'` before the name is the lifetime syntax. Lifetime names conventionally start with a lowercase letter; `'a`, `'b`, `'input`, `'data` are typical.

### What This Says

The annotation expresses a relationship: "the lifetime `'a` is some region of code; both inputs and the output are valid for that region."

In effect, the function tells the caller: "give me two references that are both valid for some region; I will return a reference that is valid for the same region." The caller is responsible for ensuring both inputs are valid wherever the return value is used.

When the compiler analyzes a call site, it computes a specific concrete lifetime for `'a` based on the actual references passed in. If the inputs have different lifetimes, the compiler picks the shorter one (the intersection), since that is the longest the return value can validly remain valid.

### A Working Use

```rust
fn main() {
    let s1 = String::from("longer string");
    let s2 = String::from("short");

    let result = longer(&s1, &s2);

    println!("longer: {result}");
}
```

This compiles and runs. Both `s1` and `s2` are valid for the same scope, so the lifetime `'a` resolves to that scope, and `result` is valid for the same scope.

### A Failing Use

```rust
fn main() {
    let s1 = String::from("longer string");
    let result;
    {
        let s2 = String::from("short");
        result = longer(&s1, &s2);
    }                                    // s2 is dropped here
    println!("longer: {result}");        // ERROR: result might reference dropped s2
}
```

The compiler rejects this. The lifetime `'a` is the intersection of `s1`'s and `s2`'s lifetimes, which is the inner block. Using `result` outside the inner block means using it after `'a` ends, which is exactly what the lifetime system forbids.

This is the value of the annotation: the caller's code is checked against the function's contract. If the caller cannot satisfy the contract, the call is rejected.

### Multiple Lifetimes

Sometimes a function's references have unrelated lifetimes. For example, a function that takes two references and returns one of them, but only ever returns the first:

```rust
fn first<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    x
}
```

Two lifetime parameters, `'a` and `'b`, are independent. The return value is tied to `'a` only. The function's caller does not need `y` to outlive the return value, because the return value never references `y`.

You declare as many lifetime parameters as you need. They are independent unless you write a relationship between them (covered briefly in Section F).

### Lifetimes Do Not Change Behavior

A common confusion: lifetime annotations look like they might affect what the function does, but they do not. They are descriptive, not prescriptive. The function `longer` runs identically with or without the annotations; the annotations only let the compiler verify that the function and its callers are correctly using references.

In other words, you do not write lifetime annotations to make a function work a certain way. You write them to tell the compiler what relationships already hold, so the compiler can verify your callers' code.

---

## C. Lifetime Annotations in Structs

### Structs with References

A struct can hold a reference as one of its fields. When it does, the struct itself has a lifetime: the struct cannot outlive the data its references point to.

```rust
struct Excerpt {
    content: &str,                       // ERROR: missing lifetime
}
```

The compiler rejects this. A struct that holds a reference must declare the lifetime of that reference, so the compiler knows the struct cannot outlive what it borrows from.

The fix is the same shape as for functions:

```rust
struct Excerpt<'a> {
    content: &'a str,
}
```

The declaration `<'a>` introduces a lifetime parameter for the struct. The field `content: &'a str` says "this field is a reference that is valid for the same lifetime as the struct itself."

### What This Means

An `Excerpt<'a>` is valid only as long as the data its `content` field points to is valid. The compiler enforces this whenever you create or use an `Excerpt`.

```rust
fn main() {
    let novel = String::from("It was the best of times, it was the worst of times.");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");

    let excerpt = Excerpt { content: first_sentence };

    println!("excerpt: {}", excerpt.content);
}
```

The `excerpt` borrows from `novel` (indirectly, via the `first_sentence` slice). The compiler tracks this. As long as `novel` is alive, `excerpt` is valid.

### A Failing Use

```rust
fn make_excerpt() -> Excerpt {           // ERROR: missing lifetime
    let novel = String::from("It was...");
    Excerpt { content: &novel }          // would be returning a struct that borrows from local data
}
```

The function tries to return an `Excerpt` that borrows from a local `String`. The local `String` is dropped when the function returns. The `Excerpt` would have a dangling reference. The compiler rejects this.

This is the same kind of error as the dangling-reference example in Module 6, just inside a struct.

### Methods on Structs with Lifetimes

When you implement methods on a struct that has a lifetime parameter, you have to mention the lifetime in the `impl` block:

```rust
impl<'a> Excerpt<'a> {
    fn length(&self) -> usize {
        self.content.len()
    }
}
```

The `impl<'a>` introduces the lifetime, and the `Excerpt<'a>` part says the methods apply to `Excerpt`s with that lifetime. The methods themselves rarely need additional lifetime annotations because of the elision rules covered in Section E.

### The Pattern

The general pattern for structs holding references:

```rust
struct Holder<'a> {
    data: &'a SomeType,
}

impl<'a> Holder<'a> {
    fn new(data: &'a SomeType) -> Self { Holder { data } }
    fn get(&self) -> &SomeType { self.data }
}
```

The lifetime parameter says "this struct borrows from data that lives for `'a`; the struct cannot outlive that data." Most struct definitions you write will not include lifetimes, because most structs own their data rather than borrowing it. The pattern is reserved for cases where you specifically want a struct that does not own.

### Owned vs Borrowed Structs

A common decision when designing a struct is whether to own its data or borrow it:

```rust
// Owned: the struct owns its data, which can come from anywhere
struct OwnedExcerpt {
    content: String,
}

// Borrowed: the struct borrows from data that must outlive it
struct BorrowedExcerpt<'a> {
    content: &'a str,
}
```

Owned structs are simpler. They have no lifetime parameter. They can be returned from functions without restrictions. They can be stored anywhere. Their cost is that creating them often involves cloning or allocating.

Borrowed structs avoid the allocation but tie the struct's lifetime to its source. They are useful when the struct is short-lived and you do not want to pay for a copy.

The general advice: prefer owned structs unless you have a specific reason to borrow. Lifetimes-in-structs are an advanced pattern; most code uses owned data.

---

## D. The `'static` Lifetime

### What `'static` Means

The lifetime `'static` is a special name for "the entire duration of the program." A reference with `'static` lifetime is valid from the start of the program to the end.

```rust
let literal: &'static str = "hello";
```

String literals have type `&'static str`. They are stored in the program's read-only data segment, alongside the compiled code. They exist as long as the program runs.

### Where `'static` Comes From

Two main sources of `'static` references:

**String literals.** Anything you write in source code as a quoted string is a `'static` reference:

```rust
let greeting = "Hello, world!";              // type is &'static str
```

**Static items.** Variables declared with the `static` keyword have addresses that exist for the entire program:

```rust
static MAX_USERS: u32 = 100_000;

fn show_max() {
    let max_ref: &'static u32 = &MAX_USERS;
    println!("{max_ref}");
}
```

Since `MAX_USERS` exists for the entire program, references to it have lifetime `'static`.

### `'static` as a Bound

Sometimes you see `'static` as a bound rather than as a literal lifetime:

```rust
fn store_message(msg: &'static str) {
    // msg can be stored in a global, sent to another thread, etc.
    // because it will be valid forever.
}
```

This function requires its argument to be a reference with `'static` lifetime. The caller must pass a reference that is valid for the entire program. String literals satisfy this; references into local variables do not.

This bound is common in code that needs to store references long-term, especially in threaded code. A thread might outlive the function that spawned it; if the thread holds a reference, the reference must be valid forever, hence `'static`.

### What `'static` Does NOT Mean

A common confusion: `'static` does not mean "the value never changes" or "the value is in static memory." It means "this reference is valid for the entire program."

A `String` you create at runtime is not `'static` (it lives only as long as its owner). But a reference to a leaked `String` could be `'static` (the leak makes it live forever). The distinction is between the reference's lifetime and any other property of the data.

### When You Need It

`'static` appears most often in:

- Function signatures where data must outlive any usage.
- Trait bounds for cross-thread code (covered in the concurrency module).
- Definitions of static variables.

For everyday function writing, you rarely need to reach for `'static`. When you do, it usually signals that you want to send data somewhere that has no lifetime relationship to the current scope: another thread, a long-lived data structure, or a static collection.

---

## E. Lifetime Elision Rules

You have already written hundreds of functions that take references without ever writing lifetime annotations. This is not because the references have no lifetimes; it is because the compiler has rules that infer the lifetimes for you in common cases.

These are called the **lifetime elision rules**. They handle the patterns that account for the vast majority of function signatures.

### The Three Rules

The compiler applies these rules in order. If they produce a complete answer, you do not need annotations. If they leave any lifetime undetermined, the compiler asks you to write the annotation.

**Rule 1: Each input reference gets its own lifetime.**

Every parameter that is a reference gets its own lifetime parameter. So `fn foo(x: &str, y: &str)` is treated as if it were written `fn foo<'a, 'b>(x: &'a str, y: &'b str)`.

**Rule 2: If there is exactly one input lifetime, it is used for all output lifetimes.**

If a function takes one reference and returns a reference, the output is assumed to have the same lifetime as the input. So `fn foo(x: &str) -> &str` is treated as `fn foo<'a>(x: &'a str) -> &'a str`.

**Rule 3: If there is a `&self` or `&mut self` parameter, its lifetime is used for all output lifetimes.**

In methods, the lifetime of `self` is used for any output references. This is why methods on structs with lifetime parameters do not usually need additional annotations on each method.

### Examples Where Elision Works

```rust
// Rule 1 + Rule 2: one input, one output. The input lifetime is used for the output.
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}

// Rule 1 only: two inputs, no output reference. No elision needed for the return.
fn print_both(a: &str, b: &str) {
    println!("{a}, {b}");
}

// Rule 1 + Rule 3: method with &self and a returned reference. Self's lifetime is used.
impl Excerpt<'_> {
    fn first_part(&self) -> &str {
        self.content.split('.').next().unwrap_or("")
    }
}
```

In each case, you do not need to write any lifetime annotations. The rules figure out what the compiler should infer.

### Examples Where Elision Fails

```rust
// Two input references, return is a reference. Rule 2 cannot apply.
// Rule 3 does not apply (no self). The compiler asks for annotations.
fn longer(s1: &str, s2: &str) -> &str {     // ERROR: needs lifetime
    if s1.len() >= s2.len() { s1 } else { s2 }
}
```

The function has two input lifetimes (one per reference), and a return type that is a reference. There is no rule that picks one input over the other, so the compiler asks the developer to specify.

The fix, as you saw in Section B:

```rust
fn longer<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() >= s2.len() { s1 } else { s2 }
}
```

### Why Have Elision Rules?

The rules cover the common cases, which means most function signatures need no annotations. Writing every lifetime explicitly would be unnecessarily verbose:

```rust
// Without elision, this trivial function would look like:
fn first_char<'a>(s: &'a str) -> &'a str {
    &s[0..1]
}

// With elision, it is just:
fn first_char(s: &str) -> &str {
    &s[0..1]
}
```

The elision rules are not magic. They are a small set of patterns that always work in straightforward cases. When the patterns are not enough, the compiler asks for help. This balance keeps simple code clean and forces explicitness only where needed.

### The `'_` Anonymous Lifetime

Sometimes you need to mention a lifetime in a place where the compiler could infer it. The `'_` syntax means "a lifetime here, please figure out which one."

```rust
impl Excerpt<'_> {                       // 'a is the lifetime, but elision handles it
    // ...
}
```

This is shorthand for "the struct has a lifetime parameter, but I do not need to name it here." It is not a different kind of lifetime; it is a placeholder for an inferred one.

---

## F. Lifetime Subtyping and Variance (Introduction)

This section is brief and introductory. The full machinery of lifetime subtyping and variance comes up in advanced library work and rarely affects everyday programming. The basic idea is worth knowing because it explains why some lifetime-related code "just works" when you might expect it to fail.

### The Basic Idea

Lifetimes form a partial order: some lifetimes are longer than others. A lifetime that is longer can be used wherever a shorter one is expected. This is called **lifetime subtyping**.

```rust
fn main() {
    let outer = String::from("outer string");      // outer's lifetime: long

    {
        let inner = String::from("inner string");  // inner's lifetime: short

        // longer expects two references with the same lifetime.
        // outer's lifetime is longer than inner's.
        // The compiler "shortens" outer's lifetime to match inner's, allowing the call.
        let result = longer(&outer, &inner);

        println!("{result}");
    }
}

fn longer<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() >= s2.len() { s1 } else { s2 }
}
```

The two references have different lifetimes (`outer` lives longer than `inner`). The function expects them to share a lifetime `'a`. The compiler resolves this by treating `&outer` as having a shorter lifetime, equal to `inner`'s. The call works.

In subtyping terminology: a longer lifetime is a **subtype** of a shorter one. Anywhere a shorter lifetime is acceptable, a longer one works too.

### Why This Matters

Subtyping is what makes lifetime-annotated code flexible. Without it, every call to a function with lifetime parameters would require all input references to have exactly the same lifetime, which would force unnecessary restructuring of code.

With subtyping, the compiler accepts a wider range of valid programs. Most of the time, you do not have to think about subtyping; it just works.

### Variance

Variance is how lifetime subtyping interacts with other types. The full theory is intricate; the practical takeaway is that variance determines whether changing a type's lifetime to a longer or shorter one preserves type compatibility.

The standard library types are mostly **covariant** in their lifetime parameters. This means a `&'long T` can be used where a `&'short T` is expected (the longer reference can substitute for a shorter one). This matches the subtyping rule above.

Some types are **invariant** in their lifetimes, which means lifetimes cannot be substituted at all. Mutable references (`&mut T`) and `Cell<T>` are invariant. This is for soundness reasons that involve mutable shared data; the details are beyond this module.

For everyday code, variance is invisible. You see it only when you write generic code involving lifetimes and run into a mysterious-looking error. When that happens, the term "variance" is your search keyword.

### When You Will Care

Subtyping and variance become relevant when:

- You write structs or functions that expose lifetime parameters as part of a public API.
- You implement traits with lifetime constraints.
- You write generic code that has to work for multiple lifetime configurations.

For learning the language, the takeaway is: lifetimes have a "longer than" relationship, and the compiler uses it automatically to make calls work when references have different but compatible lifetimes. Most code does not need to think about it.

---

## Module Summary

- A lifetime is a compile-time region of code during which a reference is guaranteed to be valid. Every reference has one.
- The compiler infers lifetimes in most cases. Annotations are required when the compiler cannot determine the relationships, typically in functions returning references derived from multiple inputs and in structs holding references.
- Lifetime annotations are descriptive, not prescriptive. They tell the compiler about relationships that already hold.
- The `'static` lifetime means "the entire duration of the program." String literals and static items satisfy it.
- The lifetime elision rules cover the common patterns automatically: each input reference gets its own lifetime, a single input lifetime is used for outputs, and `&self`'s lifetime is used for outputs in methods.
- Lifetime subtyping (longer-than relationships) is what makes the system flexible. Most code uses it without thinking about it.

## Discussion Questions

1. Lifetime annotations are described as descriptive rather than prescriptive. What does this mean in practice? How is this different from how type annotations work?
2. The lifetime elision rules cover the common cases. Identify a function signature you have written (or seen in earlier labs) that benefited from elision. What would the explicit annotations look like?
3. Compare Rust's lifetime system to how reference validity is handled in a language you know well (Java, Python, C, C++). What does each language's approach optimize for?

