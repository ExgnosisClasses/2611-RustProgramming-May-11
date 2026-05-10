# Module 12: Strings and Text Processing

## Module Overview

Strings in Rust are more involved than in most languages. This is not because strings are hard, but because Rust exposes a distinction that other languages hide: the difference between an owned string and a borrowed string. The two types `String` and `&str` look almost identical at the call site but have very different ownership properties.

This module covers both types, the operations on them, and the mechanics of formatting and parsing. The unusual complexity comes from one specific source: Rust strings are stored as UTF-8, and Rust refuses to give you a misleadingly simple "character at index" API for them. The reasons are good, and once you understand them, the API makes sense.

Three ideas are central:

1. **Strings are owned values; string slices are references.** The same distinction you saw with `Vec<T>` and `&[T]` applies to text. `String` owns its data; `&str` borrows from someone else's data.
2. **Strings are UTF-8.** Indexing by byte position is allowed but rarely what you want. The methods you use to walk a string are by character or by delimiter, not by integer index.
3. **Formatting and parsing are trait-based.** `Display` controls how a value is formatted; `FromStr` controls how a string can be parsed into a value. These traits are part of the standard library and any type can implement them.

By the end of this module, students will:

- Distinguish `String` from `&str` and choose the right one for parameters, fields, and returns.
- Create strings, concatenate them, and format them.
- Understand why string indexing by integer is restricted and use the right alternatives.
- Apply common string methods for searching, splitting, and transforming.
- Use `format!` and the `Display` trait to control how values are formatted.
- Use `FromStr` and `parse()` to convert strings to other types.
- Use the `regex` crate for pattern matching that goes beyond simple string methods.

This module's lab will work with text data extensively, so the patterns introduced here will get immediate practice.

> **A note on third-party libraries.** Section G introduces the `regex` crate, which is not part of the standard library. It is, however, the universally-used regex implementation in the Rust ecosystem. The standard library deliberately omits regex because it would force a particular design choice into the language; the `regex` crate is a separate dependency that everyone uses but no one is locked into.

---

## A. `str` vs. `String`: Ownership and Borrowing

### The Two Types

Rust has two main string types:

- **`String`**: an owned, growable, heap-allocated string.
- **`&str`** (pronounced "string slice"): a borrowed view into string data.

You have already encountered both in Module 6. They are the string equivalent of `Vec<T>` and `&[T]`: one owns its data, the other references data owned elsewhere.

```rust
fn main() {
    let owned: String = String::from("hello");        // String owns the data
    let borrowed: &str = &owned;                       // &str borrows from owned

    println!("{owned}");
    println!("{borrowed}");
}
```

### How They Differ in Memory

A `String` is a small struct on the stack with three fields: a pointer to heap data, a length, and a capacity. The actual bytes live on the heap, owned by the `String`.

A `&str` is a fat pointer: a pointer plus a length. The bytes it points to are owned by something else (a `String`, a string literal in the binary, or some other source).

```
String:                              &str:
  [ptr | len | cap]                    [ptr | len]
        |                                    |
        v                                    v
  heap: [bytes]                        somewhere: [bytes]
```

The `String` can grow because it owns its allocation. The `&str` cannot grow because it does not own anything.

### When to Use Each

The convention is the same as for `Vec` and slices:

- **Function parameters**: take `&str` when the function only needs to read.
- **Function returns**: return `String` when the function constructs new string data.
- **Struct fields**: use `String` when the struct owns its data; use `&str` (with a lifetime) when it borrows.
- **Local variables**: use whichever is appropriate for the source.

A function that takes `&str` is more flexible than one taking `String`. It can be called with:

- A `&String` (which automatically coerces to `&str`).
- A string literal (which is `&'static str`).
- A reference to a `String` field of a struct.
- Any other `&str`.

A function taking `String` requires the caller to either give up ownership or clone, neither of which is usually what you want.

```rust
// Bad: forces caller to give up ownership
fn print_owned(s: String) {
    println!("{s}");
}

// Good: works with anything
fn print_borrowed(s: &str) {
    println!("{s}");
}

fn main() {
    let owned = String::from("hello");

    print_borrowed(&owned);             // works
    print_borrowed("world");            // works (literal is &'static str)

    print_owned(owned);                 // works but consumes owned
    // print_owned("world");            // works only because of compiler magic
}
```

### String Literals

String literals like `"hello"` are not `String`s. Their type is `&'static str`: a slice of bytes baked into the program's binary, valid for the entire program.

```rust
let literal: &'static str = "hello";        // explicit type
let inferred = "hello";                      // also &'static str
```

This is why functions taking `&str` accept literals directly. The literal is a `&str`, and the function accepts `&str`.

To turn a literal into a `String`, several methods work:

```rust
let s1: String = String::from("hello");
let s2: String = "hello".to_string();
let s3: String = "hello".to_owned();
let s4: String = String::new() + "hello";
```

These all produce equivalent `String`s. The convention varies by codebase; `to_string()` and `String::from()` are both common.

### Why Two Types?

The same reasons as `Vec` versus slice. Having both types lets the system optimize for the common case (read-only access by reference) while still supporting the case where the data needs to be owned and modified.

In garbage-collected languages, every string is essentially `String` (heap-allocated, owned by the GC). The cost is hidden but real: every string operation involves heap allocation. In Rust, you can choose to share read-only access (cheap, no allocation) or take ownership (more flexible, costs an allocation).

---

## B. String Creation, Concatenation, and Formatting

### Creating Strings

Several ways to create a `String`:

```rust
fn main() {
    // Empty
    let empty = String::new();

    // From a literal
    let from_literal = String::from("hello");
    let to_string = "hello".to_string();

    // With capacity (no contents yet)
    let preallocated = String::with_capacity(100);

    // From a vector of bytes (must be valid UTF-8)
    let from_bytes = String::from_utf8(vec![104, 101, 108, 108, 111]).unwrap();

    println!("{empty:?}");
    println!("{from_literal}");
    println!("{from_bytes}");
}
```

`String::with_capacity` is useful when you are about to append a known number of characters; pre-allocating avoids growing the buffer.

### Appending to a String

```rust
let mut s = String::from("hello");

s.push_str(" world");           // append a &str
s.push('!');                    // append a single char

println!("{s}");                // hello world!
```

`push_str` takes a `&str`, so you can append literals or other strings:

```rust
let suffix = String::from(" everyone");
s.push_str(&suffix);            // append from another String
```

The `&suffix` produces a `&str` from the `&String`, which `push_str` accepts.

### Concatenation with `+`

The `+` operator works on strings, with one quirk:

```rust
let s1 = String::from("hello");
let s2 = String::from(" world");
let s3 = s1 + &s2;              // s1 is moved; s3 is the new combined string

println!("{s3}");
// println!("{s1}");            // ERROR: s1 was moved
```

The `+` operator takes the left operand by value (consuming it) and the right operand by reference. It modifies the left operand's allocation in place if possible.

This is more efficient than naive concatenation but less convenient than the `+` operator in some other languages. For complex concatenation, `format!` (next subsection) is usually clearer.

### Concatenation with `format!`

The `format!` macro is the most flexible way to construct strings:

```rust
let name = "Alice";
let age = 30;

let greeting = format!("Hello, {name}! You are {age} years old.");
println!("{greeting}");
```

`format!` works exactly like `println!` but returns a `String` instead of printing to stdout. It supports all the same formatting directives:

```rust
let pi = 3.14159265;
let s1 = format!("{pi}");                    // "3.14159265"
let s2 = format!("{pi:.2}");                 // "3.14" (2 decimal places)
let s3 = format!("{pi:>10.2}");              // "      3.14" (right-aligned, width 10)

let n = 42;
let s4 = format!("{n:b}");                   // "101010" (binary)
let s5 = format!("{n:#x}");                  // "0x2a" (hex with prefix)
let s6 = format!("{n:08}");                  // "00000042" (zero-padded width 8)
```

The format syntax is rich. The most common directives:

- `{:.N}` for N decimal places (floats).
- `{:>N}`, `{:<N}`, `{:^N}` for right, left, or center alignment in width N.
- `{:0N}` for zero-padding to width N.
- `{:b}`, `{:o}`, `{:x}`, `{:X}` for binary, octal, lower-hex, upper-hex.
- `{:#x}`, `{:#b}` etc. for prefixed forms (`0x`, `0b`).
- `{:?}` for Debug formatting (for any type implementing `Debug`).

Use `format!` when you need to combine multiple values, especially when format specifiers help. Use `+` for simple two-string concatenations, or `push_str` for building a string in steps.

### Why Not Implement `+` for `&str + &str`?

The `+` operator requires one operand to own its data, because the result must be a `String` and somebody has to provide the allocation. Two `&str` borrowed values would both need to allocate, which is what `format!` does explicitly. Forcing the user to choose makes the cost visible.

---

## C. String Slicing and Indexing (and why it's complex)

This is the section that surprises most students from other languages. In Python, you can write `s[0]` to get the first character. In Rust, you cannot. Here is why.

### The Setup

A `String` is stored as UTF-8. Each character can take from one to four bytes. For ASCII characters, one byte equals one character. For other characters (accented letters, ideographs, emoji), the encoding uses multiple bytes.

```rust
let ascii = "hello";              // 5 bytes, 5 characters
let mixed = "héllo";              // 6 bytes, 5 characters (é is 2 bytes)
let emoji = "👋";                  // 4 bytes, 1 character
```

`s.len()` returns the number of bytes, not characters:

```rust
println!("{}", "hello".len());    // 5
println!("{}", "héllo".len());    // 6
println!("{}", "👋".len());        // 4
```

This is intentional. When the standard library needs to give a single number for "size of this string," bytes is the answer that is unambiguous and useful for memory calculations.

### Why Indexing by Integer Is Disallowed

In a UTF-8 string, the byte at position 1 might be the second byte of a multi-byte character, not the start of the second character. If `s[1]` were defined to return a byte, the result would be confusing. If it returned a character, the operation would be O(n) (because the system has to decode UTF-8 to find character boundaries), and the index would not correspond to anything meaningful for memory-related operations.

Rust's solution: do not allow `s[i]` for an integer index. The compiler rejects it:

```rust
let s = String::from("hello");
let c = s[0];                    // ERROR: cannot index a String by integer
```

Instead, you slice strings by byte ranges, but only at character boundaries:

```rust
let s = String::from("hello");
let h = &s[0..1];                // "h"
let ello = &s[1..];              // "ello"
```

If the byte range falls in the middle of a character, the program panics at runtime:

```rust
let s = String::from("héllo");
let bad = &s[0..2];              // PANIC: byte 1 is not a character boundary
```

The byte at position 1 is the second byte of `é`. Slicing across it would produce invalid UTF-8.

### Walking a String

To process a string character by character, use `chars()`:

```rust
let s = "héllo";

for c in s.chars() {
    println!("{c}");
}
// h
// é
// l
// l
// o

let count = s.chars().count();   // 5 (correct character count)
```

`chars()` produces an iterator of `char` values. Each `char` is a complete Unicode scalar value, regardless of how many bytes it took in the original string.

`chars().nth(i)` gets the i-th character:

```rust
let s = "héllo";
let third = s.chars().nth(2);    // Some('l')
```

But note: this is O(n). Walking the string is fast (one pass), but random access by character index requires walking from the start. Code that needs random access by character usually benefits from converting to a `Vec<char>` first.

### Walking by Bytes

If you really do want to walk by byte:

```rust
let s = "héllo";

for b in s.bytes() {
    println!("{b}");             // 104, 195, 169, 108, 108, 111
}
```

`bytes()` produces an iterator of `u8` values. This is meaningful for binary protocols, ASCII manipulation, or memory-level work, but it does not correspond to characters.

### Walking by Grapheme Clusters

Even `chars()` is not always the right granularity. A "character" as a user perceives it (a "grapheme cluster") may consist of multiple Unicode scalar values. The é in "héllo" might be one scalar (U+00E9) or two (U+0065 + U+0301). Both look identical when displayed.

For full Unicode-correct text processing, the `unicode-segmentation` crate provides grapheme cluster iteration. For most programming tasks, `chars()` is sufficient, but be aware that human-perceived characters do not always match `char` boundaries.

### Splitting Instead of Indexing

For most tasks, you want to split a string by a delimiter rather than slice by index:

```rust
let csv = "alice,30,engineer";

let parts: Vec<&str> = csv.split(',').collect();
println!("{parts:?}");           // ["alice", "30", "engineer"]
```

Splitting works at character boundaries (because the delimiter is a character), so there is no risk of slicing through a multi-byte character. This is the right tool for parsing CSV, splitting paths, or breaking up structured text.

### The Big Picture

Rust's string API is designed around the realization that "character" is ambiguous in Unicode. Rather than picking a convenient but incorrect default, Rust forces you to choose:

- Bytes (`bytes()`, `len()`, `&s[byte_range]`)?
- Code points (`chars()`)?
- Graphemes (third-party libraries)?

Each of these has a use. None of them is "indexing" in the sense some other languages use. Once you adapt to the explicit choice, the API is more predictable than the looks-simple-but-misbehaves-on-non-ASCII alternative.

---

## D. Common String Methods

A practical reference for the most-used string methods.

### Predicates

```rust
let s = "Hello, world!";

s.is_empty();                    // false
s.starts_with("Hello");          // true
s.ends_with("!");                // true
s.contains("world");             // true
```

These methods accept either a string or a closure-based pattern.

### Trimming

```rust
let s = "  hello  \n";

s.trim();                        // "hello"
s.trim_start();                  // "hello  \n"
s.trim_end();                    // "  hello"
```

By default, these remove ASCII whitespace and Unicode whitespace. There are variants (`trim_matches`, etc.) for trimming specific characters.

### Case Conversion

```rust
let s = "Hello, World!";

s.to_lowercase();                // "hello, world!"
s.to_uppercase();                // "HELLO, WORLD!"
```

These return a new `String`, since case conversion can change the byte length (some characters become multiple characters when uppercased).

### Searching

```rust
let s = "the quick brown fox";

s.find("brown");                 // Some(10) (byte index)
s.find("zebra");                 // None
s.rfind("the");                  // Some(0) (right-find: last occurrence)
```

`find` returns the byte index of the first match, or `None`. The byte index is useful for slicing:

```rust
if let Some(index) = s.find("brown") {
    let after = &s[index..];     // "brown fox"
    println!("{after}");
}
```

### Splitting

```rust
let s = "alice,bob,carol";

let parts: Vec<&str> = s.split(',').collect();          // ["alice", "bob", "carol"]
let lines: Vec<&str> = "a\nb\nc".lines().collect();     // ["a", "b", "c"]
let words: Vec<&str> = "  hi  there  ".split_whitespace().collect();   // ["hi", "there"]
```

The `split` method takes a delimiter (a character, a string, or a closure-based pattern). `split_whitespace` is a convenience for splitting on any whitespace. `lines` splits on newlines.

### Replacing

```rust
let s = "hello world hello";

s.replace("hello", "goodbye");   // "goodbye world goodbye"
s.replacen("hello", "hi", 1);    // "hi world hello" (only first n)
```

These return a new `String`. The original is unchanged.

### Building from Parts

When you have an iterator of strings and want to combine them:

```rust
let words = vec!["hello", "world", "rust"];
let joined = words.join(" ");                // "hello world rust"
let csv = words.join(",");                   // "hello,world,rust"
```

`join` takes a separator and combines all the elements into one string. This is the inverse of `split`.

### Other Useful Methods

A grab bag:

```rust
"hello".repeat(3);               // "hellohellohello"
"hello".chars().rev().collect::<String>();   // "olleh"
" hello ".trim().len();          // 5
```

The standard library has many more methods. The full list is in the `std::str` and `std::string::String` documentation. For everyday work, the methods covered here are enough.

---

## E. `format!` and the `Display` Trait

`format!` is the macro for building formatted strings. Its behavior depends on traits implemented by the values being formatted.

### The Two Main Formatting Traits

- **`Display`**: user-facing output (used by `{}` placeholders).
- **`Debug`**: developer-facing output (used by `{:?}` placeholders).

Most standard library types implement both. Custom types implement them as needed.

### Custom Display

To make your type printable with `{}`, implement `Display`:

```rust
use std::fmt;

struct Point {
    x: f64,
    y: f64,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    println!("{p}");                    // (3, 4)
    let s = format!("{p}");             // s = "(3, 4)"
}
```

The `fmt` method receives a formatter and writes the textual representation to it. The `write!` macro is the same as `format!` and `println!` but writes to the formatter instead of returning a `String` or printing to stdout.

### Why Not Derive Display?

Unlike `Debug`, you cannot `#[derive(Display)]`. The reason is that `Display` is meant to be user-facing output, and there is no obvious default for what that should look like. A `Point` could display as "(3, 4)", "x=3,y=4", "point at (3, 4)", or any number of other things. Forcing the developer to write the implementation makes the choice explicit.

`Debug`, by contrast, has a canonical format (the type name and field values), so it can be derived.

### When to Implement Each

- **Always derive `Debug`** unless there is a specific reason not to. It is invaluable for debugging.
- **Implement `Display`** when the type has a meaningful user-facing string representation: dates, errors, IDs, formatted numbers.
- **Skip `Display`** for types that are not naturally textual (collections, complex structures). The user will use `Debug` instead.

### Format Specifiers in Custom Display

The `Display` implementation can use format specifiers:

```rust
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.2}, {:.2})", self.x, self.y)        // 2 decimal places
    }
}
```

Or you can support format specifiers from the caller. The `Formatter` provides access to the user's format flags:

```rust
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match f.precision() {
            Some(p) => write!(f, "({:.p$}, {:.p$})", self.x, self.y),
            None => write!(f, "({}, {})", self.x, self.y),
        }
    }
}
```

Now `format!("{p:.3}")` would use 3 decimal places. This kind of custom format support is rare but available when needed.

### Display in Error Types

Module 10 covered error handling. Custom error types must implement `Display` for their user-facing message:

```rust
use std::fmt;

#[derive(Debug)]
enum ParseError {
    EmptyInput,
    InvalidNumber(String),
}

impl fmt::Display for ParseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ParseError::EmptyInput => write!(f, "input cannot be empty"),
            ParseError::InvalidNumber(s) => write!(f, "'{s}' is not a number"),
        }
    }
}
```

The `thiserror` crate (Module 10) generates this implementation from a format string in the attribute. The result is the same; the macro just removes boilerplate.

### `to_string`

Any type implementing `Display` automatically gets a `to_string` method:

```rust
let p = Point { x: 3.0, y: 4.0 };
let s: String = p.to_string();              // "(3, 4)"
```

This is equivalent to `format!("{p}")` but slightly more discoverable. It works because of a blanket implementation in the standard library: `impl<T: Display> ToString for T`.

---

## F. Parsing: `FromStr` and `parse()`

The inverse of formatting is parsing: turning a string into a typed value.

### The `parse` Method

The standard library provides a `parse` method on `str`:

```rust
let s = "42";

let n: i32 = s.parse().unwrap();             // 42
let n: f64 = "3.14".parse().unwrap();        // 3.14
let b: bool = "true".parse().unwrap();        // true
```

`parse` is generic. The target type tells it what to produce; you specify it via the type annotation on the variable or via the turbofish:

```rust
let n = "42".parse::<i32>().unwrap();
```

If parsing fails, `parse` returns `Err`:

```rust
let result: Result<i32, _> = "abc".parse();
match result {
    Ok(n) => println!("{n}"),
    Err(e) => println!("parse error: {e}"),
}
```

### The `FromStr` Trait

`parse` is defined for any type that implements `FromStr`:

```rust
trait FromStr {
    type Err;
    fn from_str(s: &str) -> Result<Self, Self::Err>;
}
```

Implementing `FromStr` for your own type lets `.parse()` work on strings to produce that type:

```rust
use std::str::FromStr;

struct Point {
    x: f64,
    y: f64,
}

impl FromStr for Point {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let trimmed = s.trim_start_matches('(').trim_end_matches(')');
        let parts: Vec<&str> = trimmed.split(',').collect();

        if parts.len() != 2 {
            return Err(format!("expected (x, y), got '{s}'"));
        }

        let x: f64 = parts[0].trim().parse().map_err(|_| "x is not a number".to_string())?;
        let y: f64 = parts[1].trim().parse().map_err(|_| "y is not a number".to_string())?;

        Ok(Point { x, y })
    }
}

fn main() {
    let p: Point = "(3.0, 4.0)".parse().unwrap();
    println!("({}, {})", p.x, p.y);

    let bad: Result<Point, _> = "not a point".parse();
    println!("{bad:?}");
}
```

The `from_str` method returns `Result<Point, String>` because parsing can fail. The associated type `Err` says what kind of error.

For a real implementation, you would use a custom error type rather than `String`, following the patterns from Module 10. But the structure is the same: parse, return errors as `Err`, return the value as `Ok`.

### Why Not Just Use `From`?

You might wonder why `FromStr` exists when `From` already provides "convert from one type to another." The answer is failure: `From` is for infallible conversions (always succeeds), while `FromStr` is for fallible ones. Parsing can fail; constructing one type from another (when both are valid) cannot.

The standard library has both: `From<&str> for String` (infallible) and `FromStr for i32` (fallible). They serve different purposes.

### The `Parse` Pipeline

Combining `parse` with iterator methods produces clean parsing pipelines:

```rust
let input = "1,2,3,4,5";

let numbers: Result<Vec<i32>, _> = input
    .split(',')
    .map(|s| s.parse::<i32>())
    .collect();

match numbers {
    Ok(nums) => println!("{nums:?}"),
    Err(e) => println!("parse error: {e}"),
}
```

The `collect` on an iterator of `Result`s gives a `Result<Vec<T>, E>` (Module 11). If any element fails, the whole parse fails.

This is the canonical way to parse a list of values: split, parse each, collect with error short-circuiting.

---

## G. Introduction to the `regex` Crate

For pattern matching beyond what `find`, `split`, and `replace` can express, use the `regex` crate.

### Adding the Dependency

In `Cargo.toml`:

```toml
[dependencies]
regex = "1"
```

The `regex` crate is the standard regex library in Rust. It is fast, well-tested, and follows the syntax used in most modern regex engines (with some specific differences noted in the documentation).

### Basic Matching

```rust
use regex::Regex;

fn main() {
    let re = Regex::new(r"\d+").unwrap();

    if re.is_match("abc123def") {
        println!("found a number");
    }

    if let Some(m) = re.find("abc123def") {
        println!("found '{}' at {}-{}", m.as_str(), m.start(), m.end());
    }
}
```

The `Regex::new` call compiles the pattern. It returns a `Result` because the pattern might be invalid. Once compiled, the regex can be used for many operations.

The raw string syntax `r"..."` is convenient for regexes because it avoids double-escaping backslashes. `\d+` in a raw string means what you would expect; `"\\d+"` would be the equivalent in a non-raw string.

### Capturing Groups

Parentheses in a regex create capture groups:

```rust
let re = Regex::new(r"(\w+) (\w+)").unwrap();

let text = "hello world";
if let Some(caps) = re.captures(text) {
    println!("first: {}", &caps[1]);     // "hello"
    println!("second: {}", &caps[2]);    // "world"
}
```

`captures` returns the matched text along with each capture group. Index 0 is the full match; indices 1 and beyond are the capture groups.

Named capture groups make complex patterns more readable:

```rust
let re = Regex::new(r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})").unwrap();

if let Some(caps) = re.captures("2024-03-15") {
    println!("year: {}", &caps["year"]);
    println!("month: {}", &caps["month"]);
    println!("day: {}", &caps["day"]);
}
```

### Iteration

To find all matches in a string:

```rust
let re = Regex::new(r"\d+").unwrap();
let text = "1 fish 2 fish 3 fish";

for m in re.find_iter(text) {
    println!("{}", m.as_str());          // 1, 2, 3
}
```

`find_iter` produces an iterator of matches. There is also `captures_iter` for matches with capture groups.

### Replacement

```rust
let re = Regex::new(r"\d+").unwrap();

let text = "I have 3 apples and 4 oranges";
let result = re.replace_all(text, "N");
println!("{result}");                   // "I have N apples and N oranges"
```

`replace_all` substitutes every match with the replacement string. There is also `replace` for replacing only the first match.

For replacements that depend on the matched text, pass a closure:

```rust
let re = Regex::new(r"\d+").unwrap();
let text = "I have 3 apples and 4 oranges";

let result = re.replace_all(text, |caps: &regex::Captures| {
    let n: i32 = caps[0].parse().unwrap();
    format!("{}", n * 2)
});

println!("{result}");                   // "I have 6 apples and 8 oranges"
```

### Compile Once, Use Many Times

A regex is compiled when `Regex::new` is called. Compilation has nontrivial cost. For regexes used repeatedly, compile once and reuse:

```rust
// Bad: compiles the regex on every call
fn extract_digits(text: &str) -> Vec<String> {
    let re = Regex::new(r"\d+").unwrap();
    re.find_iter(text).map(|m| m.as_str().to_string()).collect()
}

// Good: compile once, lazily
use std::sync::OnceLock;

fn extract_digits(text: &str) -> Vec<String> {
    static DIGITS: OnceLock<Regex> = OnceLock::new();
    let re = DIGITS.get_or_init(|| Regex::new(r"\d+").unwrap());
    re.find_iter(text).map(|m| m.as_str().to_string()).collect()
}
```

For module-level constants, the `lazy_static` or `once_cell` crates are common alternatives. The `OnceLock` shown above is from the standard library and is sufficient for most uses.

### When NOT to Use Regex

Regular expressions are powerful, but they are not always the right tool:

- For fixed string searches, `s.contains("...")` is faster and clearer.
- For simple splits, `s.split(',')` is faster than a regex.
- For structured data (JSON, CSV, etc.), use a real parser, not regex. Regex is famously bad at parsing nested structures.
- For Unicode-aware text processing (case-insensitive matching, normalized comparison), the `unicode-segmentation` and `unicode-normalization` crates are usually a better fit.

Use regex when the pattern genuinely needs the power of regular expressions: choices, repetition, capture groups, anchors. For fixed strings, simpler tools are better.

---

## Module Summary

- `String` is owned, growable, and heap-allocated. `&str` is a borrowed string slice, often pointing into a `String` or a literal in the binary.
- The convention is to take `&str` parameters and return `String`. This mirrors the slice/Vec convention from Module 6.
- `format!` builds strings with formatting directives. `println!` and friends use the same syntax.
- Strings are stored as UTF-8. Indexing by integer is disallowed because byte positions do not always correspond to character boundaries. Walk strings with `chars()`, `bytes()`, or split them by delimiters.
- `Display` controls user-facing formatting; `Debug` controls developer-facing formatting. Custom types implement `Display` to be printable with `{}`.
- `FromStr` is the parsing trait. `s.parse::<T>()` works for any `T` implementing `FromStr`.
- The `regex` crate provides regular expressions. Compile once for performance; use simpler tools when regex is overkill.

## Discussion Questions

1. Rust has two main string types: `String` and `&str`. Most languages have one. What does the two-type approach give you, and what does it cost in terms of code complexity?
2. Rust disallows integer indexing on strings (`s[0]` is a compile error). The official reason is UTF-8 correctness. Is this trade-off worth it, or would a slightly-incorrect convenient API be better in practice? What kinds of bugs would the convenient API hide?
3. The standard library has `Display` (for users) and `Debug` (for developers) as separate traits. Why have two? What goes wrong if a language collapses them into one?

#