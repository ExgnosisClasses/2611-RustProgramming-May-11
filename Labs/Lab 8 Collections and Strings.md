# Modules 11 and 12: Collections and Text Processing
## Lab 8 -- Working with Vectors, Maps, Sets, Iterators, and Strings

> **Course:** Mastering Rust
> **Modules:** 11 - Collections, 12 - Strings and Text Processing
> **Estimated time:** 90-105 minutes

---

## Overview

This lab gives you hands-on experience with the most-used parts of the standard library: collections (`Vec`, `HashMap`, `HashSet`, `BTreeMap`), the iterator system that ties them together, and the string handling that appears in nearly every program. You will work through every concept from Modules 11 and 12: collection creation and access, the entry API, iterator adaptors and consumers, lazy evaluation, the distinction between `String` and `&str`, the UTF-8 byte-vs-character issue, formatting with `format!`, and parsing with `FromStr`. The lab finishes with the `regex` crate for pattern-based text processing.

You will build a single Cargo project called `log_analyzer` that processes log lines from a hypothetical web server. Log analysis is a natural fit for these modules: it involves parsing strings, splitting them into structured data, accumulating counts and groupings in maps, filtering and transforming with iterators, and producing summary reports. Every concept from Modules 11 and 12 has a place in the program.

The focus is on idiomatic Rust: using iterators instead of explicit loops, using the entry API instead of contains-then-insert, using `&str` parameters for flexibility, using `format!` for clean string construction. Many exercises ask you to write the explicit verbose form first, then refactor to the idiomatic form so you see the difference.

By the end of this lab you will be able to:

- Use `Vec<T>` for ordered sequences, including the access methods, mutation methods, and iteration forms.
- Use `HashMap<K, V>` for key-value lookup and apply the entry API for the count-or-update pattern.
- Use `HashSet<T>` and `BTreeMap<K, V>` and choose them when their properties fit.
- Apply iterator adaptors (`map`, `filter`, `flat_map`, `take`, `skip`) to transform sequences.
- Apply consuming adaptors (`collect`, `sum`, `fold`, `any`, `all`, `find`) to produce final values.
- Distinguish `String` from `&str` and choose the right one for parameters.
- Create, concatenate, slice, and format strings idiomatically.
- Parse strings into typed values with `parse()` and the `FromStr` trait.
- Use the `regex` crate for pattern matching beyond what string methods provide.

---

## Before You Start

This lab requires the environment from Lab 1. Confirm:

```bash
rustc --version
cargo --version
```

VS Code with `rust-analyzer` and format-on-save should still be configured.

> **Note:** The final exercise uses the `regex` crate, which is not part of the standard library. It will be added through `Cargo.toml`, and Cargo will download it on the first build. Make sure you have an internet connection when you reach Exercise 8.

---

## Lab Project Setup

Create a new Cargo project for this lab:

```bash
cd ~/rust-course/labs
cargo new log_analyzer
cd log_analyzer
code .
```

Create a `lab8-notes.md` file in the project root for observations and checkpoint answers.

---

## Exercise 1 -- Vec<T>: The Workhorse Collection

**Estimated time:** 10-15 minutes
**Topics covered:** creating, accessing, mutating, and iterating over vectors

### Context

`Vec<T>` is the most-used collection in Rust. It is a heap-allocated, growable sequence, similar to Python's `list` or Java's `ArrayList`. This exercise establishes the basic mechanics through a small program that builds and processes a list of values.

### Task 1.1 -- Creating vectors

Replace the contents of `src/main.rs` with:

```rust
fn main() {
    // Empty vector with explicit type:
    let mut empty: Vec<i32> = Vec::new();
    empty.push(1);
    empty.push(2);
    println!("after push: {empty:?}");

    // The vec! macro:
    let from_literal = vec![10, 20, 30, 40, 50];
    println!("from literal: {from_literal:?}");

    // Repeated value:
    let zeros = vec![0; 5];
    println!("zeros: {zeros:?}");

    // Pre-allocated capacity:
    let mut preallocated = Vec::with_capacity(100);
    for i in 0..5 {
        preallocated.push(i);
    }
    println!("preallocated len: {}, capacity: {}", preallocated.len(), preallocated.capacity());
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
after push: [1, 2]
from literal: [10, 20, 30, 40, 50]
zeros: [0, 0, 0, 0, 0]
preallocated len: 5, capacity: 100
```

The `with_capacity` form is useful when you know roughly how many elements you will add. Without it, the vector grows by reallocating, which is fine for occasional pushes but wasteful in tight loops.

### Task 1.2 -- Accessing elements

Replace `main` with:

```rust
fn main() {
    let v = vec![10, 20, 30, 40, 50];

    // Direct indexing (panics on out-of-bounds):
    let first = v[0];
    let third = v[2];
    println!("first: {first}, third: {third}");

    // Safe access with get (returns Option):
    if let Some(value) = v.get(2) {
        println!("got: {value}");
    }

    if let Some(value) = v.get(100) {
        println!("got: {value}");
    } else {
        println!("index 100 is out of bounds");
    }

    // First and last:
    println!("first: {:?}", v.first());
    println!("last: {:?}", v.last());
}
```

Run the program. Expected output:

```
first: 10, third: 30
got: 30
index 100 is out of bounds
first: Some(10)
last: Some(50)
```

The two access patterns serve different needs: indexing when you know the index is valid (and want a panic if it is not, indicating a bug), and `get` when out-of-bounds is a legitimate possibility.

### Task 1.3 -- Mutating

Replace `main` with:

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    v.push(4);
    v.push(5);
    println!("after push: {v:?}");

    let last = v.pop();
    println!("popped {last:?}, vec: {v:?}");

    v.insert(0, 100);
    println!("after insert at 0: {v:?}");

    v.remove(2);
    println!("after remove at 2: {v:?}");

    // Mutating an element by index:
    v[0] = 999;
    println!("after assign: {v:?}");

    // Sorting:
    v.sort();
    println!("sorted: {v:?}");
}
```

Run the program. Expected output:

```
after push: [1, 2, 3, 4, 5]
popped Some(5), vec: [1, 2, 3, 4]
after insert at 0: [100, 1, 2, 3, 4]
after remove at 2: [100, 1, 3, 4]
after assign: [999, 1, 3, 4]
sorted: [1, 3, 4, 999]
```

A few notes:

- `push` and `pop` are O(1). `insert` and `remove` at arbitrary positions are O(n) because they shift elements.
- `pop` returns `Option<T>` because the vector might be empty.
- `sort` requires the elements to implement `Ord`. For custom comparison, use `sort_by` with a closure.

### Task 1.4 -- The three iteration forms

Replace `main` with:

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    // Iterate by reference: yields &i32
    print!("by reference: ");
    for x in &v {
        print!("{x} ");
    }
    println!();

    // After the loop, v is still valid:
    println!("v after by-ref: {v:?}");

    // Iterate by value (consumes the vector):
    let v_copy = v.clone();
    print!("by value: ");
    for x in v_copy {
        print!("{x} ");
    }
    println!();
    // v_copy is no longer valid here

    // Iterate by mutable reference:
    let mut v_mut = vec![1, 2, 3];
    for x in &mut v_mut {
        *x *= 10;
    }
    println!("after mutating: {v_mut:?}");
}
```

Run the program. Expected output:

```
by reference: 1 2 3 4 5
v after by-ref: [1, 2, 3, 4, 5]
by value: 1 2 3 4 5
after mutating: [10, 20, 30]
```

The three forms parallel the three reference types from Module 6:

- `&v` for reading (most common).
- `v` for consuming (consumes the vector, useful when you no longer need it).
- `&mut v` for in-place modification.

For `Copy` types like `i32`, the by-value form is fine because copying is cheap. For non-`Copy` types like `String`, you usually want `&v` to avoid losing the original.

### Task 1.5 -- Slicing

Replace `main` with:

```rust
fn process(values: &[i32]) -> i32 {
    values.iter().sum()
}

fn main() {
    let v = vec![10, 20, 30, 40, 50];

    let total = process(&v);                         // pass the whole vec
    let middle = process(&v[1..4]);                  // pass a slice
    let array = [100, 200, 300];
    let array_total = process(&array);               // pass an array

    println!("total: {total}");
    println!("middle sum: {middle}");
    println!("array sum: {array_total}");
}
```

Run the program. Expected output:

```
total: 150
middle sum: 90
array sum: 600
```

The function takes `&[i32]`, which accepts a `Vec`, an array, or a subrange of either. This is the same pattern from Lab 5: take slices for read-only access, get flexibility for free.

### Checkpoints

1. The `with_capacity(100)` example pre-allocated space for 100 elements. What practical situation would benefit from pre-allocation? What would happen if you pushed 200 elements?
2. The `get(index)` method returns `Option<&T>` while direct indexing `v[index]` panics on out-of-bounds. Both are legitimate; identify a specific situation where each is the right choice.
3. The function `process` takes `&[i32]` rather than `&Vec<i32>`. Beyond accepting more types, what does this signal to the reader of the function's signature?

---

## Exercise 2 -- HashMap<K, V> and the Entry API

**Estimated time:** 10-15 minutes
**Topics covered:** map creation, insertion, lookup, iteration, the entry API for count-or-update

### Context

`HashMap<K, V>` is the standard key-value collection. Lookups are O(1) on average. The entry API provides a clean idiom for "if the key exists, modify the value; otherwise, insert a new value," which is one of the most common patterns in real code.

### Task 2.1 -- Creating and populating

Replace `src/main.rs` with:

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<String, i32> = HashMap::new();

    scores.insert(String::from("Alice"), 95);
    scores.insert(String::from("Bob"), 87);
    scores.insert(String::from("Carol"), 92);

    println!("scores: {scores:?}");
    println!("count: {}", scores.len());
}
```

Run the program. Expected output (the order may vary because `HashMap` does not guarantee order):

```
scores: {"Alice": 95, "Bob": 87, "Carol": 92}
count: 3
```

Note that `HashMap` is in `std::collections`, so you need to import it. There is no built-in macro for literal map construction in the standard library.

### Task 2.2 -- Lookup

Replace `main` with:

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<String, i32> = HashMap::new();
    scores.insert(String::from("Alice"), 95);
    scores.insert(String::from("Bob"), 87);

    // get returns Option<&V>:
    if let Some(score) = scores.get("Alice") {
        println!("Alice's score: {score}");
    }

    if scores.get("Dave").is_none() {
        println!("Dave is not in the map");
    }

    // Direct indexing panics on missing key:
    let alice_score = scores["Alice"];
    println!("alice direct: {alice_score}");

    // contains_key:
    if scores.contains_key("Bob") {
        println!("Bob is in the map");
    }
}
```

Run the program. Expected output:

```
Alice's score: 95
Dave is not in the map
alice direct: 95
Bob is in the map
```

The pattern parallels `Vec`: use `get` when the key might be missing (returns `Option`), use direct indexing when you know it is there (panics on missing).

### Task 2.3 -- Iteration

Replace `main` with:

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<String, i32> = HashMap::new();
    scores.insert(String::from("Alice"), 95);
    scores.insert(String::from("Bob"), 87);
    scores.insert(String::from("Carol"), 92);

    // Iterate over (key, value) pairs:
    println!("entries:");
    for (name, score) in &scores {
        println!("  {name}: {score}");
    }

    // Iterate over keys only:
    print!("keys: ");
    for name in scores.keys() {
        print!("{name} ");
    }
    println!();

    // Iterate over values only:
    println!("sum of values: {}", scores.values().sum::<i32>());
}
```

Run the program. The output's order may vary:

```
entries:
  Alice: 95
  Bob: 87
  Carol: 92
keys: Alice Bob Carol 
sum of values: 274
```

The order is not guaranteed because `HashMap` uses a hash-based layout. If you need ordered iteration, use `BTreeMap` (Exercise 3).

### Task 2.4 -- The naive count: contains-then-insert

A common pattern is counting occurrences. Here is the naive way:

```rust
use std::collections::HashMap;

fn main() {
    let words = vec!["the", "quick", "brown", "fox", "jumps", "over", "the", "lazy", "dog", "the"];
    let mut counts: HashMap<String, i32> = HashMap::new();

    for word in &words {
        if counts.contains_key(*word) {
            let current = counts.get(*word).unwrap();
            counts.insert(word.to_string(), current + 1);
        } else {
            counts.insert(word.to_string(), 1);
        }
    }

    println!("word counts: {counts:?}");
}
```

Run the program. Expected output (order may vary):

```
word counts: {"the": 3, "quick": 1, "brown": 1, ...}
```

This works, but it does up to three operations per word (contains_key, get, insert) and looks awkward. The hash is computed multiple times. The next task shows the idiomatic alternative.

### Task 2.5 -- The entry API

Replace the loop with the entry API:

```rust
use std::collections::HashMap;

fn main() {
    let words = vec!["the", "quick", "brown", "fox", "jumps", "over", "the", "lazy", "dog", "the"];
    let mut counts: HashMap<String, i32> = HashMap::new();

    for word in &words {
        *counts.entry(word.to_string()).or_insert(0) += 1;
    }

    println!("word counts: {counts:?}");
}
```

Run the program. The output is identical to Task 2.4, but the code is one line.

The entry API does this:

- `entry(key)` returns an `Entry` enum (occupied or vacant).
- `or_insert(default)` inserts the default if vacant, then returns a mutable reference to the value (whether just inserted or already there).
- `*` dereferences and `+= 1` increments.

The hash is computed once. The map is touched once. The code is cleaner.

This pattern is so useful that it appears constantly in real Rust code. Whenever you have "for each item, update some count or accumulator in a map," reach for the entry API.

### Task 2.6 -- The entry API for grouping

A related pattern is grouping items by a key:

```rust
use std::collections::HashMap;

fn main() {
    let words = vec!["apple", "banana", "cherry", "avocado", "blueberry", "coconut"];

    let mut by_letter: HashMap<char, Vec<String>> = HashMap::new();

    for word in &words {
        let first = word.chars().next().unwrap();
        by_letter.entry(first).or_insert_with(Vec::new).push(word.to_string());
    }

    for (letter, words) in &by_letter {
        println!("{letter}: {words:?}");
    }
}
```

Run the program. Expected output:

```
a: ["apple", "avocado"]
b: ["banana", "blueberry"]
c: ["cherry", "coconut"]
```

The pattern: `or_insert_with(Vec::new)` inserts an empty vec if absent, returns a mutable reference to the value, then `push` appends to it.

`or_insert_with` takes a closure rather than a value. This is preferable when constructing the default has any cost (allocating an empty `Vec` is cheap but real). For trivial defaults, `or_insert(0)` works; for collections, `or_insert_with(Vec::new)` is the convention.

### Checkpoints

1. The naive count in Task 2.4 did up to three map operations per word. The entry API in Task 2.5 does one. What is the practical difference for a large input (say, counting words in a million-line file)?
2. The HashMap iteration order in Task 2.3 was unspecified. What downsides does this have? When is unordered iteration acceptable, and when is it a problem?
3. `or_insert_with(Vec::new)` takes a closure instead of `or_insert(Vec::new())`. What is the difference between these two forms? When does the difference matter?

---

## Exercise 3 -- HashSet and BTreeMap

**Estimated time:** 10 minutes
**Topics covered:** sets for unique values, BTreeMap for ordered iteration

### Context

`HashSet<T>` is for collections of unique values. `BTreeMap<K, V>` is a sorted map that gives you ordered iteration and range queries. Each has specific use cases.

### Task 3.1 -- HashSet for uniqueness

Replace `src/main.rs` with:

```rust
use std::collections::HashSet;

fn main() {
    let words = vec!["the", "quick", "the", "fox", "the", "brown", "fox"];

    let mut unique_words: HashSet<String> = HashSet::new();
    for word in &words {
        unique_words.insert(word.to_string());
    }

    println!("unique words: {unique_words:?}");
    println!("count: {}", unique_words.len());

    // contains check:
    println!("contains 'fox': {}", unique_words.contains("fox"));
    println!("contains 'cat': {}", unique_words.contains("cat"));
}
```

Run the program. Expected output (order may vary):

```
unique words: {"the", "quick", "fox", "brown"}
count: 4
contains 'fox': true
contains 'cat': false
```

`insert` returns a `bool`: `true` if the value was new, `false` if it was already present. This is useful for deduplication while iterating:

```rust
let mut seen = HashSet::new();
let unique: Vec<&str> = words.iter().filter(|w| seen.insert(w.to_string())).copied().collect();
```

The `seen.insert(w)` returns `true` only the first time `w` is encountered, which is exactly when you want to keep it. Adapt this if you want to try.

### Task 3.2 -- Set operations

Replace `main` with:

```rust
use std::collections::HashSet;

fn main() {
    let a: HashSet<i32> = [1, 2, 3, 4].iter().copied().collect();
    let b: HashSet<i32> = [3, 4, 5, 6].iter().copied().collect();

    let intersection: HashSet<&i32> = a.intersection(&b).collect();
    let union: HashSet<&i32> = a.union(&b).collect();
    let difference: HashSet<&i32> = a.difference(&b).collect();

    println!("a: {a:?}");
    println!("b: {b:?}");
    println!("intersection (in both): {intersection:?}");
    println!("union (in either): {union:?}");
    println!("difference (in a but not b): {difference:?}");
}
```

Run the program. Expected output:

```
a: {1, 2, 3, 4}
b: {3, 4, 5, 6}
intersection (in both): {3, 4}
union (in either): {1, 2, 3, 4, 5, 6}
difference (in a but not b): {1, 2}
```

These methods return iterators of references; `.collect()` turns them into concrete sets. They are useful for any code that does set-theoretic operations: comparing tag sets, filtering by membership, finding what changed between two states.

### Task 3.3 -- BTreeMap for sorted iteration

`BTreeMap<K, V>` keeps entries sorted by key. Replace `main` with:

```rust
use std::collections::BTreeMap;

fn main() {
    let mut log_counts: BTreeMap<String, u32> = BTreeMap::new();

    log_counts.insert(String::from("ERROR"), 5);
    log_counts.insert(String::from("WARN"), 12);
    log_counts.insert(String::from("INFO"), 250);
    log_counts.insert(String::from("DEBUG"), 1024);
    log_counts.insert(String::from("TRACE"), 5000);

    println!("log counts (always sorted by key):");
    for (level, count) in &log_counts {
        println!("  {level}: {count}");
    }
}
```

Run the program. Expected output (always in this order):

```
log counts (always sorted by key):
  DEBUG: 1024
  ERROR: 5
  INFO: 250
  TRACE: 5000
  WARN: 12
```

Unlike `HashMap`, the iteration is always sorted. This is the same input running multiple times will produce the same output, which matters for tests, deterministic reports, and reproducible builds.

### Task 3.4 -- Range queries on BTreeMap

`BTreeMap`'s killer feature is range queries:

```rust
use std::collections::BTreeMap;

fn main() {
    let mut events: BTreeMap<u64, String> = BTreeMap::new();
    events.insert(100, String::from("start"));
    events.insert(150, String::from("checkpoint A"));
    events.insert(200, String::from("milestone"));
    events.insert(250, String::from("checkpoint B"));
    events.insert(300, String::from("end"));

    println!("events between time 150 and 250 inclusive:");
    for (time, event) in events.range(150..=250) {
        println!("  {time}: {event}");
    }
}
```

Run the program. Expected output:

```
events between time 150 and 250 inclusive:
  150: checkpoint A
  200: milestone
  250: checkpoint B
```

The `range(150..=250)` walks only the entries with keys in that range. This is O(log n) to find the start, then O(k) to walk through the k matching entries. With a `HashMap`, you would have to iterate the entire map and filter; for large maps with small ranges, `BTreeMap` is dramatically faster.

### Task 3.5 -- Choosing between HashMap and BTreeMap

```rust
use std::collections::{HashMap, BTreeMap};

fn main() {
    // Use HashMap when:
    // - You only need point lookups.
    // - Iteration order does not matter.
    // - You want maximum throughput.
    let cache: HashMap<String, i32> = HashMap::new();
    let _ = cache;

    // Use BTreeMap when:
    // - You need sorted iteration.
    // - You need range queries.
    // - Determinism matters (e.g., reproducible test output).
    let timeline: BTreeMap<u64, String> = BTreeMap::new();
    let _ = timeline;

    println!("placeholders for the two map types");
}
```

For most code, `HashMap` is the default. Reach for `BTreeMap` when ordering or range queries are genuinely useful.

### Checkpoints

1. The deduplication trick `words.iter().filter(|w| seen.insert(w.to_string())).copied().collect()` uses a side effect inside `filter`. This works because `HashSet::insert` returns `bool`. Why does Rust allow this even though it has side effects in a "filter" call?
2. `BTreeMap` operations are O(log n) while `HashMap` operations are O(1) amortized. For a map with 10,000 entries, what is the practical difference in performance? When does it matter?
3. The range query in Task 3.4 was O(log n + k). Without `BTreeMap`, you would have to iterate every entry and filter. For a 1-million-entry map and a query that returns 10 results, what is the performance difference?

---

## Exercise 4 -- Iterators and Adaptors

**Estimated time:** 15 minutes
**Topics covered:** iterator basics, lazy evaluation, `map`, `filter`, `take`, `skip`

### Context

The iterator system unifies traversal across all collection types. Many things are iterators: vectors, ranges, characters in a string, lines in a file, even custom types. The same methods (`map`, `filter`, `collect`) work uniformly. This exercise covers the most-used adaptors.

### Task 4.1 -- The basics

Replace `src/main.rs` with:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // Map: transform each element
    let squared: Vec<i32> = numbers.iter().map(|x| x * x).collect();
    println!("squared: {squared:?}");

    // Filter: keep elements matching a predicate
    let even: Vec<&i32> = numbers.iter().filter(|x| **x % 2 == 0).collect();
    println!("even: {even:?}");

    // Combination: filter then map
    let even_squared: Vec<i32> = numbers
        .iter()
        .filter(|x| **x % 2 == 0)
        .map(|x| x * x)
        .collect();
    println!("even squared: {even_squared:?}");
}
```

Run the program. Expected output:

```
squared: [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
even: [2, 4, 6, 8, 10]
even squared: [4, 16, 36, 64, 100]
```

The `iter()` produces an iterator of `&i32` (references). The closures receive references; for `filter`, the closure receives a reference to the reference (the type is `&&i32`), which is why you see `**x` (two dereferences) in the predicate.

The `**x` looks awkward but is consistent with Rust's general principle of explicit dereferencing.

### Task 4.2 -- Lazy evaluation

The iterator chain does no work until something consumes it. Demonstrate this:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    println!("creating the iterator...");
    let iter = numbers.iter().map(|x| {
        println!("doubling {x}");
        x * 2
    });
    println!("iterator created");

    println!("now iterating:");
    for value in iter {
        println!("got {value}");
    }
}
```

Run the program. Expected output:

```
creating the iterator...
iterator created
now iterating:
doubling 1
got 2
doubling 2
got 4
doubling 3
got 6
doubling 4
got 8
doubling 5
got 10
```

The "doubling" prints appear during iteration, not when the iterator is created. The `map` registered a transformation but did not actually apply it. The work happened lazily, one element at a time, as the loop pulled values.

This laziness is what makes iterator chains efficient. A long chain compiles to a single tight loop with no intermediate allocations.

### Task 4.3 -- take and skip

`take(n)` produces the first n elements; `skip(n)` discards the first n. Replace `main` with:

```rust
fn main() {
    let numbers: Vec<i32> = (1..=20).collect();

    let first_five: Vec<&i32> = numbers.iter().take(5).collect();
    println!("first five: {first_five:?}");

    let after_ten: Vec<&i32> = numbers.iter().skip(10).collect();
    println!("after ten: {after_ten:?}");

    let middle: Vec<&i32> = numbers.iter().skip(5).take(5).collect();
    println!("middle: {middle:?}");

    let infinite_demo: Vec<i32> = (1..).map(|x| x * x).take(5).collect();
    println!("first five squares of an infinite range: {infinite_demo:?}");
}
```

Run the program. Expected output:

```
first five: [1, 2, 3, 4, 5]
after ten: [11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
middle: [6, 7, 8, 9, 10]
first five squares of an infinite range: [1, 4, 9, 16, 25]
```

The last example is interesting: `(1..)` is an infinite range. Without `take(5)`, the program would loop forever. With `take(5)`, it works because lazy evaluation only generates values until `take` has its 5.

This pattern (apply transformations to an unbounded source, then take a finite prefix) only works because of laziness. Eager evaluation would try to produce all infinitely-many squares before taking the first five.

### Task 4.4 -- flat_map for nested sequences

`flat_map` is `map` followed by flattening. It is useful when each input produces a sequence of outputs:

```rust
fn main() {
    let words = vec!["hello", "world", "rust"];

    // Each word produces an iterator of characters; flat_map flattens them:
    let all_chars: Vec<char> = words.iter().flat_map(|w| w.chars()).collect();
    println!("all chars: {all_chars:?}");

    // Each number produces 1..=n; flat_map flattens:
    let nested: Vec<i32> = (1..=4).flat_map(|n| 1..=n).collect();
    println!("nested: {nested:?}");
}
```

Run the program. Expected output:

```
all chars: ['h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd', 'r', 'u', 's', 't']
nested: [1, 1, 2, 1, 2, 3, 1, 2, 3, 4]
```

`flat_map` is powerful for processing collections of collections (or strings of characters, or nested ranges). The single call replaces a nested loop with explicit accumulation.

### Task 4.5 -- enumerate and zip

Two more useful adaptors:

```rust
fn main() {
    let names = vec!["Alice", "Bob", "Carol"];

    // enumerate produces (index, value) pairs:
    for (i, name) in names.iter().enumerate() {
        println!("{i}: {name}");
    }

    // zip combines two iterators into pairs:
    let scores = vec![95, 87, 92];
    for (name, score) in names.iter().zip(scores.iter()) {
        println!("{name}: {score}");
    }

    // zip stops at the shorter iterator:
    let extra = vec![1, 2, 3, 4, 5];
    let truncated: Vec<(&i32, &&str)> = extra.iter().zip(names.iter()).collect();
    println!("truncated: {truncated:?}");
}
```

Run the program. Expected output:

```
0: Alice
1: Bob
2: Carol
Alice: 95
Bob: 87
Carol: 92
truncated: [(1, "Alice"), (2, "Bob"), (3, "Carol")]
```

`enumerate` is the standard way to get an index in iteration; you saw it in Module 4 with for loops. `zip` is the canonical way to walk two parallel sequences together.

### Checkpoints

1. The lazy evaluation in Task 4.2 means transformations only happen when their results are needed. Identify a specific situation where this saves significant work compared to eager evaluation.
2. In Task 4.3, the line `(1..).map(|x| x * x).take(5).collect()` works on an infinite range. Could you write the same logic with eager evaluation? Why or why not?
3. The `flat_map` in Task 4.4 took an iterator of strings and produced an iterator of characters. The same effect could be achieved with `.map(...).collect::<Vec<...>>()` followed by another flatten. What does `flat_map` do that the manual version does not?

---

## Exercise 5 -- Consuming Adaptors

**Estimated time:** 10-15 minutes
**Topics covered:** `collect`, `sum`, `fold`, `count`, `any`, `all`, `find`, `min`, `max`

### Context

A consuming adaptor pulls all the values out of an iterator and produces a final result. They are how iterator chains terminate.

### Task 5.1 -- collect

`collect` gathers an iterator into a collection. Replace `src/main.rs` with:

```rust
use std::collections::HashSet;
use std::collections::HashMap;

fn main() {
    let numbers: Vec<i32> = (1..=10).collect();
    println!("vec: {numbers:?}");

    let unique: HashSet<i32> = vec![1, 2, 2, 3, 3, 3, 4].into_iter().collect();
    println!("set: {unique:?}");

    let pairs: HashMap<&str, i32> = vec![
        ("alice", 95),
        ("bob", 87),
    ].into_iter().collect();
    println!("map: {pairs:?}");

    let chars: String = vec!['h', 'i', '!'].into_iter().collect();
    println!("string: {chars}");
}
```

Run the program. Expected output:

```
vec: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
set: {1, 2, 3, 4}
map: {"alice": 95, "bob": 87}
string: hi!
```

`collect` is generic. The target type tells it what to produce. The same iterator can be collected into a `Vec`, `HashSet`, `HashMap`, `String`, or other collection that implements `FromIterator`.

### Task 5.2 -- collect with Result

A particularly useful pattern is collecting an iterator of `Result`s:

```rust
fn main() {
    let inputs = vec!["1", "2", "3", "4", "5"];

    let parsed: Result<Vec<i32>, _> = inputs.iter().map(|s| s.parse::<i32>()).collect();
    println!("all parsed: {parsed:?}");

    let with_bad = vec!["1", "2", "abc", "4"];
    let parsed: Result<Vec<i32>, _> = with_bad.iter().map(|s| s.parse::<i32>()).collect();
    println!("with bad: {parsed:?}");
}
```

Run the program. Expected output:

```
all parsed: Ok([1, 2, 3, 4, 5])
with bad: Err(ParseIntError { kind: InvalidDigit })
```

When you collect an iterator of `Result<T, E>`, you get a `Result<Vec<T>, E>`. If all elements are `Ok`, you get the vector. If any is `Err`, you get the first error and the rest are not processed.

This is the canonical pattern for parsing a list of values, building on the error handling from Lab 7.

### Task 5.3 -- sum, product, count

Numeric reductions:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let total: i32 = numbers.iter().sum();
    let product: i32 = numbers.iter().product();
    let count = numbers.iter().count();
    let count_even = numbers.iter().filter(|x| **x % 2 == 0).count();

    println!("total: {total}");
    println!("product: {product}");
    println!("count: {count}");
    println!("count of even: {count_even}");
}
```

Run the program. Expected output:

```
total: 15
product: 120
count: 5
count of even: 2
```

`sum` and `product` work on numeric types. `count` works on any iterator. The pattern `filter(...).count()` is the idiomatic way to count matching elements.

For known-size collections, prefer `vec.len()` over `vec.iter().count()`; it is O(1) instead of O(n).

### Task 5.4 -- fold

`fold` is the general accumulator. It takes a starting value and a closure that combines the accumulator with each element:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let sum = numbers.iter().fold(0, |acc, x| acc + x);
    println!("sum via fold: {sum}");

    let product = numbers.iter().fold(1, |acc, x| acc * x);
    println!("product via fold: {product}");

    let words = vec!["hello", "world", "rust"];
    let concatenated = words.iter().fold(String::new(), |mut acc, s| {
        if !acc.is_empty() {
            acc.push(' ');
        }
        acc.push_str(s);
        acc
    });
    println!("concatenated: {concatenated}");

    // Counting with fold:
    let count = numbers.iter().fold(0, |acc, _| acc + 1);
    println!("count via fold: {count}");
}
```

Run the program. Expected output:

```
sum via fold: 15
product via fold: 120
concatenated: hello world rust
count via fold: 5
```

`fold` is more general than `sum`, `product`, or `count`. Anything you can express as "start with X; for each element, update X" can be a `fold`. Use the specialized methods when they fit; reach for `fold` for anything else.

### Task 5.5 -- any and all

These return `bool`:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let has_even = numbers.iter().any(|x| x % 2 == 0);
    let all_positive = numbers.iter().all(|x| *x > 0);
    let all_even = numbers.iter().all(|x| x % 2 == 0);

    println!("has even: {has_even}");
    println!("all positive: {all_positive}");
    println!("all even: {all_even}");
}
```

Run the program. Expected output:

```
has even: true
all positive: true
all even: false
```

`any` short-circuits at the first match; `all` short-circuits at the first non-match. Both are O(n) worst case but often much faster on real data.

### Task 5.6 -- find, position, min, max

```rust
fn main() {
    let numbers = vec![3, 1, 4, 1, 5, 9, 2, 6];

    let first_big = numbers.iter().find(|x| **x > 4);
    let position = numbers.iter().position(|x| *x > 4);

    let smallest = numbers.iter().min();
    let largest = numbers.iter().max();

    println!("first > 4: {first_big:?}");
    println!("position: {position:?}");
    println!("min: {smallest:?}");
    println!("max: {largest:?}");
}
```

Run the program. Expected output:

```
first > 4: Some(5)
position: Some(4)
min: Some(1)
max: Some(9)
```

`find` returns the first matching element; `position` returns the index of the first match. `min` and `max` find extremes. All return `Option` because the iterator might be empty (or, for `find`/`position`, no element might match).

### Task 5.7 -- A practical pipeline

Combining adaptors and consumers:

```rust
fn main() {
    let log_lines = vec![
        "INFO: started",
        "INFO: handling request",
        "ERROR: connection refused",
        "INFO: handling request",
        "WARN: timeout",
        "ERROR: invalid input",
        "INFO: shutting down",
    ];

    let error_count = log_lines.iter()
        .filter(|line| line.starts_with("ERROR"))
        .count();

    let warning_lines: Vec<&&str> = log_lines.iter()
        .filter(|line| line.starts_with("WARN"))
        .collect();

    let first_error = log_lines.iter().find(|line| line.starts_with("ERROR"));

    let any_errors = log_lines.iter().any(|line| line.starts_with("ERROR"));

    println!("error count: {error_count}");
    println!("warnings: {warning_lines:?}");
    println!("first error: {first_error:?}");
    println!("any errors? {any_errors}");
}
```

Run the program. Expected output:

```
error count: 2
warnings: ["WARN: timeout"]
first error: Some("ERROR: connection refused")
any errors? true
```

Each computation reads as a small declarative pipeline: filter the data and apply a consumer. Compare this to the equivalent imperative loop with explicit counters and conditions; the iterator version is clearer.

### Checkpoints

1. The pattern `iter.collect::<Result<Vec<T>, E>>()` short-circuits on the first error. Why is this the natural behavior? When would you want to collect all errors instead?
2. The `fold` example for concatenating strings rebuilt a string by appending. Could the same effect be achieved with `vec.join(" ")`? When is each approach appropriate?
3. The `any` and `all` methods short-circuit. The `count` method does not (it walks all elements regardless). Why does `count` not short-circuit? When does this matter for performance?

---

## Exercise 6 -- String vs &str

**Estimated time:** 10-15 minutes
**Topics covered:** the two string types, when to use each, conversion, common methods

### Context

Rust has two string types: `String` (owned, growable) and `&str` (borrowed, immutable view). Choosing the right one for parameters and fields is one of the most-asked questions in Rust. This exercise establishes the patterns through working examples.

### Task 6.1 -- The two types

Replace `src/main.rs` with:

```rust
fn main() {
    let owned: String = String::from("hello");
    let literal: &str = "world";
    let borrowed: &str = &owned;

    println!("owned: {owned}");
    println!("literal: {literal}");
    println!("borrowed: {borrowed}");

    println!("owned len: {}", owned.len());
    println!("literal len: {}", literal.len());
    println!("borrowed len: {}", borrowed.len());
}
```

Run the program. Expected output:

```
owned: hello
literal: world
borrowed: hello
owned len: 5
literal len: 5
borrowed len: 5
```

Three things are interchangeable in many contexts:

- `String`: owned, heap-allocated, growable.
- `&str`: a borrowed slice into someone else's string data. Could be a `String`'s data or a string literal in the binary.
- String literals (`"hello"`): have type `&'static str`, valid for the entire program.

The methods (`len`, etc.) work on all three.

### Task 6.2 -- Functions that take &str

Replace `main` with:

```rust
fn print_length(s: &str) {
    println!("'{s}' has length {}", s.len());
}

fn main() {
    print_length("literal");                     // works
    let owned = String::from("owned");
    print_length(&owned);                         // works (deref coercion)
    print_length(&owned[0..3]);                   // works (substring slice)

    let another = String::from("another");
    print_length(&another);
}
```

Run the program. Expected output:

```
'literal' has length 7
'owned' has length 5
'own' has length 3
'another' has length 7
```

The function takes `&str`. It accepts:

- String literals directly (`"literal"`).
- `&String` (`&owned`), via deref coercion: the compiler converts `&String` to `&str` automatically.
- Substring slices (`&owned[0..3]`).

This flexibility is why `&str` is the preferred parameter type for read-only access. A function taking `String` would require all callers to either give up ownership or clone.

### Task 6.3 -- Functions that return String

```rust
fn shout(s: &str) -> String {
    s.to_uppercase()
}

fn build_greeting(name: &str) -> String {
    format!("Hello, {name}!")
}

fn main() {
    let result = shout("hello");
    println!("{result}");

    let greeting = build_greeting("Alice");
    println!("{greeting}");

    let name_input = String::from("Bob");
    let greeting2 = build_greeting(&name_input);    // pass &String, function takes &str
    println!("{greeting2}");
}
```

Run the program. Expected output:

```
HELLO
Hello, Alice!
Hello, Bob!
```

Functions that produce new string data (constructed via concatenation, formatting, or transformation) typically return `String`. The convention parallels `&[T]` for slices: take borrowed parameters, return owned values when constructing new ones.

### Task 6.4 -- Concatenation: + operator vs format!

Two ways to combine strings:

```rust
fn main() {
    let part1 = String::from("Hello, ");
    let part2 = "world";

    // The + operator (consumes left, borrows right):
    let combined1 = part1 + part2;
    println!("via +: {combined1}");
    // part1 is no longer valid here; it was moved into +

    // format! macro (does not consume):
    let part1 = String::from("Hello, ");
    let combined2 = format!("{part1}{part2}");
    println!("via format!: {combined2}");
    println!("part1 still: {part1}");

    // For multi-piece concatenation, format! is much cleaner:
    let name = "Alice";
    let age = 30;
    let role = "engineer";

    let bio = format!("{name}, age {age}, works as a {role}");
    println!("bio: {bio}");
}
```

Run the program. Expected output:

```
via +: Hello, world
via format!: Hello, world
part1 still: Hello, 
bio: Alice, age 30, works as a engineer
```

The `+` operator has a quirk: it takes the left operand by value (consuming it) and the right by reference. This is more efficient than naive concatenation but less convenient than `format!`.

The convention:

- For two-string concatenation, either works.
- For more than two pieces, `format!` is much clearer.
- For building a string in a loop, use `String::push_str` or `String::push`.

### Task 6.5 -- format! features

`format!` has rich formatting:

```rust
fn main() {
    let name = "Alice";
    let age = 30;
    let pi = 3.14159265;
    let count = 42;

    // Basic substitution:
    let s1 = format!("name: {name}, age: {age}");
    println!("{s1}");

    // Decimal places:
    let s2 = format!("pi to 2 places: {pi:.2}");
    let s3 = format!("pi to 5 places: {pi:.5}");
    println!("{s2}");
    println!("{s3}");

    // Width and alignment:
    let s4 = format!("[{name:<10}]");                // left-align in width 10
    let s5 = format!("[{name:>10}]");                // right-align
    let s6 = format!("[{name:^10}]");                // center
    println!("{s4}");
    println!("{s5}");
    println!("{s6}");

    // Zero-padding:
    let s7 = format!("count: {count:05}");
    println!("{s7}");

    // Hex, binary, octal:
    let s8 = format!("{count} in hex: {count:#x}, binary: {count:#b}");
    println!("{s8}");

    // Debug formatting:
    let v = vec![1, 2, 3];
    let s9 = format!("{v:?}");
    let s10 = format!("{v:#?}");
    println!("{s9}");
    println!("{s10}");
}
```

Run the program. Expected output:

```
name: Alice, age: 30
pi to 2 places: 3.14
pi to 5 places: 3.14159
[Alice     ]
[     Alice]
[  Alice   ]
count: 00042
42 in hex: 0x2a, binary: 0b101010
[1, 2, 3]
[
    1,
    2,
    3,
]
```

The format directives are versatile. The most useful for everyday work:

- `{:.N}` for N decimal places.
- `{:>N}`, `{:<N}`, `{:^N}` for alignment in width N.
- `{:0N}` for zero-padding to width N.
- `{:#x}`, `{:#b}` for hex/binary with prefix.
- `{:?}` and `{:#?}` for debug formatting.

### Task 6.6 -- String methods

Common string methods:

```rust
fn main() {
    let s = "  Hello, World! Welcome to Rust.  ";

    // Predicates:
    println!("starts with 'Hello': {}", s.trim().starts_with("Hello"));
    println!("contains 'Rust': {}", s.contains("Rust"));

    // Trimming:
    println!("trimmed: '{}'", s.trim());

    // Case conversion:
    let upper: String = s.trim().to_uppercase();
    let lower: String = s.trim().to_lowercase();
    println!("upper: '{upper}'");
    println!("lower: '{lower}'");

    // Splitting:
    let words: Vec<&str> = s.split_whitespace().collect();
    println!("words: {words:?}");

    // Splitting on a delimiter:
    let csv = "alice,30,engineer";
    let parts: Vec<&str> = csv.split(',').collect();
    println!("csv parts: {parts:?}");

    // Replacing:
    let replaced = s.replace("World", "Earth");
    println!("replaced: '{replaced}'");

    // Reversing characters:
    let reversed: String = "hello".chars().rev().collect();
    println!("reversed: '{reversed}'");

    // Joining:
    let joined = words.join(" | ");
    println!("joined: '{joined}'");
}
```

Run the program. Expected output:

```
starts with 'Hello': true
contains 'Rust': true
trimmed: 'Hello, World! Welcome to Rust.'
upper: 'HELLO, WORLD! WELCOME TO RUST.'
lower: 'hello, world! welcome to rust.'
words: ["Hello,", "World!", "Welcome", "to", "Rust."]
csv parts: ["alice", "30", "engineer"]
replaced: '  Hello, Earth! Welcome to Rust.  '
reversed: 'olleh'
joined: 'Hello, | World! | Welcome | to | Rust.'
```

These are the most-used string methods. The standard library has many more; consult the `std::str::str` documentation when you need something specific.

### Task 6.7 -- The UTF-8 byte vs character distinction

A key difference from many languages:

```rust
fn main() {
    let ascii = "hello";
    let mixed = "héllo";
    let emoji = "👋";

    println!("ascii: '{ascii}', len {} bytes, {} chars", ascii.len(), ascii.chars().count());
    println!("mixed: '{mixed}', len {} bytes, {} chars", mixed.len(), mixed.chars().count());
    println!("emoji: '{emoji}', len {} bytes, {} chars", emoji.len(), emoji.chars().count());
}
```

Run the program. Expected output:

```
ascii: 'hello', len 5 bytes, 5 chars
mixed: 'héllo', len 6 bytes, 5 chars
emoji: '👋', len 4 bytes, 1 chars
```

`s.len()` returns bytes, not characters. For ASCII, the two are equal. For non-ASCII, they differ. Use `chars().count()` if you need character count.

Indexing by byte position works only at character boundaries. A bad slice panics:

```rust
fn main() {
    let s = "héllo";
    let good = &s[0..1];        // 'h' (1 byte)
    println!("good: '{good}'");

    // This would panic:
    // let bad = &s[0..2];      // byte 1 is in the middle of é
}
```

For text processing, use methods like `chars()`, `split_whitespace()`, or `lines()` rather than byte indexing.

### Checkpoints

1. The function `print_length(s: &str)` accepts string literals, `&String`, and substring slices. Why is this flexibility valuable? What pattern in Rust achieves the same kind of flexibility for non-string types?
2. The `+` operator consumed `part1` in Task 6.4. Why does `+` work this way rather than borrowing both operands like `format!` does?
3. Task 6.7 showed that `s.len()` returns bytes, not characters. Some languages return character count by default, panicking less but performing worse on long strings. Why did Rust choose byte count, and what is the trade-off?

---

## Exercise 7 -- Parsing with FromStr

**Estimated time:** 10 minutes
**Topics covered:** the `parse` method, `FromStr` trait, parsing pipelines

### Context

The inverse of formatting is parsing: turning a string into a typed value. Rust uses the `FromStr` trait, exposed through the `parse` method on strings.

### Task 7.1 -- Parsing primitives

Replace `src/main.rs` with:

```rust
fn main() {
    let s1 = "42";
    let n1: i32 = s1.parse().unwrap();
    println!("{s1} -> {n1}");

    let s2 = "3.14";
    let n2: f64 = s2.parse().unwrap();
    println!("{s2} -> {n2}");

    let s3 = "true";
    let b: bool = s3.parse().unwrap();
    println!("{s3} -> {b}");

    // Failure case:
    let s4 = "abc";
    let result: Result<i32, _> = s4.parse();
    match result {
        Ok(n) => println!("got {n}"),
        Err(e) => println!("'{s4}' failed to parse: {e}"),
    }
}
```

Run the program. Expected output:

```
42 -> 42
3.14 -> 3.14
true -> true
'abc' failed to parse: invalid digit found in string
```

`parse` is generic. The target type tells it what to produce. You specify the type via a let annotation or with the turbofish syntax: `s.parse::<i32>()`.

### Task 7.2 -- Parsing pipelines

A common pattern is parsing a list of values:

```rust
fn main() {
    let csv = "10,20,30,40,50";

    let parsed: Result<Vec<i32>, _> = csv.split(',').map(|s| s.parse::<i32>()).collect();

    match parsed {
        Ok(values) => {
            let total: i32 = values.iter().sum();
            println!("values: {values:?}");
            println!("sum: {total}");
        }
        Err(e) => println!("parse error: {e}"),
    }

    let bad = "10,20,abc,40";
    let parsed: Result<Vec<i32>, _> = bad.split(',').map(|s| s.parse::<i32>()).collect();
    println!("bad input result: {parsed:?}");
}
```

Run the program. Expected output:

```
values: [10, 20, 30, 40, 50]
sum: 150
bad input result: Err(ParseIntError { kind: InvalidDigit })
```

The pipeline `split → map(parse) → collect::<Result<Vec, _>>` is the canonical way to parse a list. The collect short-circuits on the first error, returning `Err` for the whole result.

### Task 7.3 -- Implementing FromStr for a custom type

You can implement `FromStr` for your own types. Add this:

```rust
use std::str::FromStr;

#[derive(Debug)]
struct Point {
    x: f64,
    y: f64,
}

impl FromStr for Point {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let trimmed = s.trim_matches(|c: char| c == '(' || c == ')');
        let parts: Vec<&str> = trimmed.split(',').collect();

        if parts.len() != 2 {
            return Err(format!("expected '(x, y)', got '{s}'"));
        }

        let x: f64 = parts[0].trim().parse().map_err(|_| format!("invalid x: '{}'", parts[0]))?;
        let y: f64 = parts[1].trim().parse().map_err(|_| format!("invalid y: '{}'", parts[1]))?;

        Ok(Point { x, y })
    }
}

fn main() {
    let inputs = vec!["(3.0, 4.0)", "(0,0)", "(1.5, 2.5)", "not a point", "(1, abc)"];

    for input in inputs {
        match input.parse::<Point>() {
            Ok(p) => println!("'{input}' -> {p:?}"),
            Err(e) => println!("'{input}' -> ERROR: {e}"),
        }
    }
}
```

Run the program. Expected output:

```
'(3.0, 4.0)' -> Point { x: 3.0, y: 4.0 }
'(0,0)' -> Point { x: 0.0, y: 0.0 }
'(1.5, 2.5)' -> Point { x: 1.5, y: 2.5 }
'not a point' -> ERROR: expected '(x, y)', got 'not a point'
'(1, abc)' -> ERROR: invalid y: ' abc'
```

Implementing `FromStr` makes the type usable with `parse`. The `parse` method on a string sees that the target type implements `FromStr` and calls `from_str`.

For real code, you would use a custom error type (from Lab 7) instead of `String`. The structure is the same.

### Task 7.4 -- A practical use: parsing log lines

Real log data:

```rust
use std::str::FromStr;

#[derive(Debug)]
struct LogEntry {
    level: String,
    message: String,
}

impl FromStr for LogEntry {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let parts: Vec<&str> = s.splitn(2, ": ").collect();
        if parts.len() != 2 {
            return Err(format!("missing ': ' separator in '{s}'"));
        }

        let level = parts[0].to_string();
        let message = parts[1].to_string();

        if !["ERROR", "WARN", "INFO", "DEBUG"].contains(&level.as_str()) {
            return Err(format!("unknown level '{level}'"));
        }

        Ok(LogEntry { level, message })
    }
}

fn main() {
    let lines = vec![
        "INFO: server started",
        "WARN: slow query",
        "ERROR: connection failed",
        "FATAL: unrecoverable",
        "no level here",
    ];

    let entries: Vec<Result<LogEntry, _>> = lines.iter().map(|line| line.parse::<LogEntry>()).collect();

    for (i, entry) in entries.iter().enumerate() {
        match entry {
            Ok(e) => println!("line {i}: {e:?}"),
            Err(err) => println!("line {i}: ERROR: {err}"),
        }
    }
}
```

Run the program. Expected output:

```
line 0: LogEntry { level: "INFO", message: "server started" }
line 1: LogEntry { level: "WARN", message: "slow query" }
line 2: LogEntry { level: "ERROR", message: "connection failed" }
line 3: ERROR: unknown level 'FATAL'
line 4: ERROR: missing ': ' separator in 'no level here'
```

The `LogEntry::from_str` function combines string operations (splitting, validation, conversion) into a parsing pipeline. The result is a clean way to turn raw log lines into typed `LogEntry` values.

### Checkpoints

1. The `parse` method requires a type annotation (or turbofish). Why can the compiler not infer the target type from context?
2. Implementing `FromStr` lets a type be the target of `parse`. Why is this trait separate from `From<&str>` (which is also for converting from a string reference)?
3. The parsing pipeline `split → map(parse) → collect::<Result<Vec, _>>` short-circuits on the first error. When would you prefer to collect all errors? What pattern would you use?

---

## Exercise 8 -- Putting It Together with regex

**Estimated time:** 15 minutes
**Topics covered:** integration of Modules 11 and 12 with the `regex` crate

### Context

This final exercise combines everything from the lab into a realistic log-analysis program. The program reads simulated log data, parses it with the `regex` crate, accumulates statistics in maps, and produces a summary report. It demonstrates how iterator chains, collections, strings, and parsing work together in a real workflow.

### Task 8.1 -- Adding regex as a dependency

Open `Cargo.toml` and add:

```toml
[dependencies]
regex = "1"
```

Save the file. The next build will download the crate.

### Task 8.2 -- The full program

Replace your `src/main.rs` with this complete program:

```rust
use std::collections::{HashMap, BTreeMap, HashSet};
use regex::Regex;

// ============================================================
// Data types
// ============================================================

#[derive(Debug, Clone)]
struct LogEntry {
    timestamp: String,
    level: String,
    method: String,
    path: String,
    status: u16,
    response_ms: u32,
}

// ============================================================
// Parsing with regex
// ============================================================

fn parse_log_line(line: &str, re: &Regex) -> Option<LogEntry> {
    let caps = re.captures(line)?;

    Some(LogEntry {
        timestamp: caps.name("timestamp")?.as_str().to_string(),
        level: caps.name("level")?.as_str().to_string(),
        method: caps.name("method")?.as_str().to_string(),
        path: caps.name("path")?.as_str().to_string(),
        status: caps.name("status")?.as_str().parse().ok()?,
        response_ms: caps.name("ms")?.as_str().parse().ok()?,
    })
}

// ============================================================
// Analysis functions
// ============================================================

fn count_by_level(entries: &[LogEntry]) -> BTreeMap<String, u32> {
    let mut counts = BTreeMap::new();
    for entry in entries {
        *counts.entry(entry.level.clone()).or_insert(0) += 1;
    }
    counts
}

fn count_by_status(entries: &[LogEntry]) -> BTreeMap<u16, u32> {
    let mut counts = BTreeMap::new();
    for entry in entries {
        *counts.entry(entry.status).or_insert(0) += 1;
    }
    counts
}

fn count_by_path(entries: &[LogEntry]) -> HashMap<String, u32> {
    let mut counts = HashMap::new();
    for entry in entries {
        *counts.entry(entry.path.clone()).or_insert(0) += 1;
    }
    counts
}

fn average_response_time(entries: &[LogEntry]) -> f64 {
    if entries.is_empty() {
        return 0.0;
    }

    let total: u32 = entries.iter().map(|e| e.response_ms).sum();
    total as f64 / entries.len() as f64
}

fn slow_requests(entries: &[LogEntry], threshold_ms: u32) -> Vec<&LogEntry> {
    entries
        .iter()
        .filter(|e| e.response_ms >= threshold_ms)
        .collect()
}

fn unique_paths(entries: &[LogEntry]) -> HashSet<String> {
    entries.iter().map(|e| e.path.clone()).collect()
}

fn error_paths(entries: &[LogEntry]) -> Vec<&str> {
    entries
        .iter()
        .filter(|e| e.status >= 500)
        .map(|e| e.path.as_str())
        .collect()
}

// ============================================================
// Reporting
// ============================================================

fn format_report(entries: &[LogEntry]) -> String {
    let mut report = String::new();

    report.push_str("=== Log Analysis Report ===\n\n");

    report.push_str(&format!("Total entries: {}\n\n", entries.len()));

    report.push_str("Counts by level:\n");
    for (level, count) in count_by_level(entries) {
        report.push_str(&format!("  {level:>5}: {count}\n"));
    }

    report.push_str("\nCounts by status code:\n");
    for (status, count) in count_by_status(entries) {
        report.push_str(&format!("  {status}: {count}\n"));
    }

    report.push_str(&format!("\nUnique paths: {}\n", unique_paths(entries).len()));
    report.push_str(&format!("Average response time: {:.2}ms\n", average_response_time(entries)));

    let slow = slow_requests(entries, 500);
    report.push_str(&format!("\nSlow requests (>=500ms): {}\n", slow.len()));
    for entry in slow.iter().take(5) {
        report.push_str(&format!(
            "  [{}] {} {} -> {} ({}ms)\n",
            entry.timestamp, entry.method, entry.path, entry.status, entry.response_ms
        ));
    }

    let errors = error_paths(entries);
    if !errors.is_empty() {
        report.push_str(&format!("\nError paths (status >= 500): {}\n", errors.len()));
        let mut by_path: HashMap<&str, u32> = HashMap::new();
        for path in &errors {
            *by_path.entry(*path).or_insert(0) += 1;
        }
        let mut sorted_errors: Vec<(&&str, &u32)> = by_path.iter().collect();
        sorted_errors.sort_by(|a, b| b.1.cmp(a.1));
        for (path, count) in sorted_errors.iter().take(5) {
            report.push_str(&format!("  {path}: {count} errors\n"));
        }
    }

    report
}

// ============================================================
// Main
// ============================================================

fn main() {
    // Sample log data:
    let log_lines = vec![
        "[2024-01-15T10:00:00Z] INFO  GET /api/users 200 45ms",
        "[2024-01-15T10:00:01Z] INFO  GET /api/products 200 32ms",
        "[2024-01-15T10:00:02Z] WARN  POST /api/users 422 120ms",
        "[2024-01-15T10:00:03Z] INFO  GET /api/users 200 38ms",
        "[2024-01-15T10:00:04Z] ERROR GET /api/orders 500 850ms",
        "[2024-01-15T10:00:05Z] INFO  GET /api/products 200 28ms",
        "[2024-01-15T10:00:06Z] INFO  POST /api/login 200 95ms",
        "[2024-01-15T10:00:07Z] ERROR GET /api/orders 500 920ms",
        "[2024-01-15T10:00:08Z] WARN  GET /api/users 404 12ms",
        "[2024-01-15T10:00:09Z] INFO  GET /api/users 200 41ms",
        "[2024-01-15T10:00:10Z] ERROR GET /api/admin 500 750ms",
        "[2024-01-15T10:00:11Z] INFO  GET /api/health 200 5ms",
        "[2024-01-15T10:00:12Z] WARN  POST /api/users 422 110ms",
        "[2024-01-15T10:00:13Z] INFO  GET /api/products 200 29ms",
        "[2024-01-15T10:00:14Z] ERROR GET /api/orders 500 1200ms",
        "this line does not match the format",
        "[2024-01-15T10:00:15Z] INFO  GET /api/health 200 4ms",
    ];

    // Compile the regex once:
    let re = Regex::new(
        r"^\[(?P<timestamp>[^\]]+)\] +(?P<level>\w+) +(?P<method>\w+) +(?P<path>\S+) +(?P<status>\d+) +(?P<ms>\d+)ms$"
    ).expect("invalid regex");

    // Parse all lines, collecting only successful matches:
    let entries: Vec<LogEntry> = log_lines
        .iter()
        .filter_map(|line| parse_log_line(line, &re))
        .collect();

    let report = format_report(&entries);
    println!("{report}");

    let unparseable_count = log_lines.len() - entries.len();
    if unparseable_count > 0 {
        println!("\nNote: {unparseable_count} line(s) could not be parsed");
    }
}
```

Run the program:

```bash
cargo run
```

Expected output:

```
=== Log Analysis Report ===

Total entries: 16

Counts by level:
  ERROR: 4
  INFO: 9
  WARN: 3

Counts by status code:
  200: 9
  404: 1
  422: 2
  500: 4

Unique paths: 6
Average response time: 274.94ms

Slow requests (>=500ms): 4
  [2024-01-15T10:00:04Z] GET /api/orders -> 500 (850ms)
  [2024-01-15T10:00:07Z] GET /api/orders -> 500 (920ms)
  [2024-01-15T10:00:10Z] GET /api/admin -> 500 (750ms)
  [2024-01-15T10:00:14Z] GET /api/orders -> 500 (1200ms)

Error paths (status >= 500): 4
  /api/orders: 3 errors
  /api/admin: 1 errors

Note: 1 line(s) could not be parsed
```

### Task 8.3 -- Identify the patterns

In `lab8-notes.md`, identify each instance of the following patterns in the program above:

1. A `Vec<LogEntry>` collected from an iterator chain.
2. A `HashMap<String, u32>` used for unordered counting.
3. A `BTreeMap<u16, u32>` used for sorted iteration.
4. A `HashSet<String>` used for uniqueness.
5. The entry API used to count occurrences.
6. An iterator chain using `filter` and `map`.
7. A use of `filter_map` to combine filter and map into one pass.
8. A `format!` call that builds a string with multiple substitutions.
9. A use of `format!` with a width specifier.
10. A use of `format!` with a precision specifier.
11. A use of `parse()` with `.ok()` to convert Result to Option.
12. A use of regex named capture groups (`(?P<name>...)`).
13. A function that takes `&[LogEntry]` for flexibility.
14. A function that returns `String` rather than `&str`.
15. A use of `take(5)` to limit output.

### Task 8.4 -- Predict and verify

For each of the following code changes, predict whether the program will compile and what behavior changes you expect. Write your predictions in `lab8-notes.md` BEFORE running.

**Change A:** In `count_by_level`, replace `BTreeMap` with `HashMap`. What changes in the output?

**Change B:** In the iterator chain `log_lines.iter().filter_map(...).collect()`, replace `filter_map` with `map` followed by collecting into `Vec<Option<LogEntry>>`. What changes? How would the rest of the program have to adapt?

**Change C:** Change the regex to include a typo: change `\w+` to `\w++` (which is invalid regex syntax). What happens? When does the error appear: at compile time or at runtime?

For each change, write your prediction first, then run and verify.

### Checkpoints

1. The program uses both `HashMap` (for `count_by_path` and the error count) and `BTreeMap` (for `count_by_level` and `count_by_status`). Why are different map types used? What would be the consequence of using only one?
2. The regex compilation `Regex::new(...).expect(...)` uses `expect` rather than handling the error properly. Is this acceptable? When would it not be?
3. The function `format_report` builds a `String` by repeated `push_str` calls. Could it use `format!` directly to build the entire string? What is the trade-off?

---

## Summary and Reflection

You have now used every concept from Modules 11 and 12 in a working program.

| Exercise | Topic | Key Insight |
|---|---|---|
| 1 -- Vec | creation, access, mutation, slicing | `Vec<T>` is the workhorse. Take `&[T]` for parameters when you only read. |
| 2 -- HashMap and entry API | counts and groupings | The entry API replaces three operations with one and is the idiomatic count pattern. |
| 3 -- HashSet and BTreeMap | uniqueness and ordered iteration | Different collections optimize for different needs. Choose based on what you need. |
| 4 -- Iterator adaptors | map, filter, take, skip, flat_map | Iterators are lazy. Long chains compile to single tight loops. |
| 5 -- Consuming adaptors | collect, sum, fold, any, all, find | Adaptors terminate iterator chains. Many cases have specialized methods over fold. |
| 6 -- String vs &str | the two types and conversion | Take `&str`, return `String`. Use `format!` for multi-piece construction. |
| 7 -- Parsing with FromStr | parse method, custom FromStr | The parse pipeline is the canonical way to turn strings into typed values. |
| 8 -- Integration with regex | combining everything | Real text-processing code combines collections, iterators, strings, and pattern matching. |

### Final Reflection Questions

Take 10 minutes to write answers in `lab8-notes.md` before your next session.

1. Of the concepts in Modules 11 and 12, which one had the steepest learning curve for you, and which one felt most natural? Speculate about why each felt the way it did.

2. Iterator chains in Rust look like Java streams or functional pipelines from other languages, but they have specific differences (laziness, ownership integration, zero-cost abstraction). Pick one feature of Rust iterators that does not exist in another language you know. Explain in your own words why Rust has this feature and what it gains the language.

3. The lab covered both `String` and `&str`, the entry API for HashMaps, and iterator chains. These features are all about choosing the right tool for the job. Identify a specific situation from the lab where you initially picked one approach and then changed your mind after considering the alternative. What changed your decision?

---

*End of Lab 8*
