# Module 11: Collections

## Module Overview

Collections are how programs hold groups of related values. Rust's standard library provides a small set of collections, each with specific performance characteristics and use cases. They are unified by the `Iterator` trait, which makes traversing and transforming them uniform across types.

The interesting parts of this module are the iterator system. Once you have used iterators in Rust, going back to imperative loops feels limiting. Three ideas are central:

1. **Collections own their data.** A `Vec<T>` owns its elements; dropping the vector drops them. A `HashMap<K, V>` owns its keys and values. The ownership rules from Module 6 apply uniformly.
2. **Iterators are lazy.** Calling `.iter().map(...)` does not actually compute anything. Work happens only when something consumes the iterator. This is what allows long chains of transformations to compile to single tight loops.
3. **The same iterator methods work everywhere.** A `Vec`, a `HashMap`, a `HashSet`, a string, a range, even a custom type can all be iterated with the same methods. The iterator API is one of the most learn-once-use-everywhere features in Rust.

By the end of this module, students will:

- Use `Vec<T>` for ordered, growable sequences and understand its access and mutation methods.
- Use `HashMap<K, V>` for key-value lookup and apply the entry API for insert-or-modify patterns.
- Recognize `HashSet<T>` and `BTreeMap<K, V>` and choose them when their ordering or uniqueness properties fit.
- Use the `Iterator` trait and recognize what makes a type iterable.
- Apply the common iterator adaptors (`map`, `filter`, `flat_map`, `take`, `skip`) to transform sequences.
- Use consuming adaptors (`collect`, `sum`, `fold`, `any`, `all`) to produce final values from iterator chains.
- Chain iterators idiomatically and understand why this style is fast.

This module is heavy on practical patterns. Most data manipulation in Rust flows through iterators; learning the common methods is the single highest-leverage thing students can do at this stage.

> **A note on what is not covered.** The standard library has more collections than this module covers: `VecDeque`, `LinkedList`, `BinaryHeap`. They have specific use cases and the standard library documentation is the right reference when you need them. The four covered here (`Vec`, `HashMap`, `HashSet`, `BTreeMap`) handle the vast majority of real-world data needs.

---

## A. `Vec<T>`: Creation, Access, Mutation, Iteration

`Vec<T>` is the workhorse collection. It is a growable, heap-allocated sequence of `T`s, similar to Python's `list`, Java's `ArrayList`, or C++'s `std::vector`.

### Creating a Vec

Several ways to create a vector:

```rust
fn main() {
    // Empty vec, type inferred later
    let mut empty: Vec<i32> = Vec::new();

    // Vec with initial capacity (no elements yet)
    let mut preallocated: Vec<i32> = Vec::with_capacity(100);

    // From a literal
    let numbers = vec![1, 2, 3, 4, 5];

    // Repeated value
    let zeros = vec![0; 10];

    println!("{numbers:?}");        // [1, 2, 3, 4, 5]
    println!("{zeros:?}");          // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
}
```

The `vec!` macro is the most common form for initializing with known values. `Vec::with_capacity` is useful when you know roughly how many elements you will add; pre-allocating avoids repeated reallocations as the vector grows.

### Adding and Removing Elements

```rust
let mut v = vec![1, 2, 3];

v.push(4);              // [1, 2, 3, 4]
v.push(5);              // [1, 2, 3, 4, 5]

let last = v.pop();     // Some(5), v is now [1, 2, 3, 4]
let _ = v.pop();        // Some(4), v is now [1, 2, 3]

v.insert(0, 100);       // [100, 1, 2, 3]
v.remove(1);            // returns 1, v is now [100, 2, 3]

v.clear();              // []
```

`push` and `pop` are O(1) amortized. `insert` and `remove` are O(n) because they shift elements. For frequent insert-at-front operations, `VecDeque` is more appropriate.

### Accessing Elements

Two ways to access elements:

```rust
let v = vec![10, 20, 30, 40, 50];

// Direct indexing: panics on out-of-bounds
let first = v[0];
let third = v[2];

// Get by index: returns Option, no panic
let third = v.get(2);           // Some(&30)
let nope = v.get(100);          // None

println!("{first}, {third:?}");
```

Direct indexing is appropriate when you have already verified the index is in bounds. The `get` method is appropriate when the index might be out of bounds, because the `Option` return forces you to handle that case explicitly.

### Iteration

The most common operation on a vector is iterating over its elements:

```rust
let v = vec![1, 2, 3, 4, 5];

// Iterate by reference (most common)
for x in &v {
    println!("{x}");
}

// Iterate by value (consumes the vector)
for x in v {                    // v is moved here
    println!("{x}");
}
// v is no longer valid here

// Iterate by mutable reference
let mut v = vec![1, 2, 3];
for x in &mut v {
    *x *= 2;
}
// v is now [2, 4, 6]
```

Three forms, three semantics. The choice mirrors the three reference types from Module 6:

- `&v`: borrow each element. The vector is unchanged.
- `v`: take ownership of each element. The vector is consumed.
- `&mut v`: borrow each element mutably. You can modify in place.

For `Copy` types like `i32`, the by-value form is fine because copying is cheap. For non-`Copy` types like `String`, you usually want the by-reference form.

### Common Methods

A few methods you will use constantly:

```rust
let v = vec![1, 2, 3, 4, 5];

v.len();                        // 5
v.is_empty();                   // false
v.contains(&3);                 // true (note the reference)
v.first();                      // Some(&1)
v.last();                       // Some(&5)

let mut sorted = vec![3, 1, 4, 1, 5];
sorted.sort();                  // [1, 1, 3, 4, 5]
sorted.reverse();               // [5, 4, 3, 1, 1]
sorted.dedup();                 // removes consecutive duplicates
```

Many of these have variants for custom comparison: `sort_by_key`, `sort_by`, `dedup_by`, etc. They all take closures, building on Module 5.

### Slicing

A vector can be sliced like an array (Module 6):

```rust
let v = vec![10, 20, 30, 40, 50];

let middle = &v[1..4];              // &[20, 30, 40]
let first_two = &v[..2];            // &[10, 20]
let last_three = &v[2..];           // &[30, 40, 50]

// Functions usually take slices, not Vecs
fn sum(values: &[i32]) -> i32 {
    values.iter().sum()
}

let total = sum(&v);                // works
let middle_total = sum(&v[1..4]);   // also works
```

This is the same pattern from Module 6: `&[T]` is the flexible parameter type. A function that accepts `&[T]` can be called with a `Vec<T>`, an array, or a subrange of either.

---

## B. `HashMap<K, V>`: Insertion, Lookup, Entry API

`HashMap<K, V>` stores key-value pairs. Lookups are O(1) on average. It corresponds to Python's `dict`, Java's `HashMap`, or C++'s `std::unordered_map`.

### Creating and Populating a HashMap

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<String, i32> = HashMap::new();

    scores.insert(String::from("Alice"), 95);
    scores.insert(String::from("Bob"), 87);
    scores.insert(String::from("Carol"), 92);

    println!("{scores:?}");
}
```

`HashMap` is in the `std::collections` module, so you need to import it. Unlike `Vec`, there is no built-in macro for literal construction in the standard library, though several alternatives exist (covered below).

The keys are `String` and the values are `i32`. The keys must implement `Hash` and `Eq` (covered in Module 13); most standard types satisfy this.

### Inserting and Updating

`insert` does both insertion and update:

```rust
let mut scores = HashMap::new();
scores.insert("Alice".to_string(), 95);
scores.insert("Alice".to_string(), 100);    // overwrites the previous value

println!("{scores:?}");                      // {"Alice": 100}
```

The previous value, if any, is returned:

```rust
let old = scores.insert("Alice".to_string(), 50);
println!("{old:?}");                         // Some(100)
```

### Looking Up Values

```rust
let mut scores = HashMap::new();
scores.insert("Alice".to_string(), 95);

// get returns Option<&V>
let alice_score = scores.get("Alice");      // Some(&95)
let bob_score = scores.get("Bob");          // None

// Direct indexing panics on missing key
let score = scores["Alice"];                // 95
// let score = scores["Bob"];               // panic: key not found
```

`get` is the safe form. Direct indexing is convenient but should only be used when you know the key exists. In most code, `get` is the right choice.

### Iteration

```rust
let mut scores = HashMap::new();
scores.insert("Alice".to_string(), 95);
scores.insert("Bob".to_string(), 87);

// Iterate over (key, value) pairs
for (name, score) in &scores {
    println!("{name}: {score}");
}

// Iterate over keys only
for name in scores.keys() {
    println!("{name}");
}

// Iterate over values only
for score in scores.values() {
    println!("{score}");
}
```

Note that `HashMap` iteration order is not guaranteed. The order can change between runs and even between iterations. If you need deterministic ordering, use `BTreeMap` (Section C) or sort the keys explicitly.

### The Entry API

A common pattern is "if the key exists, modify its value; otherwise, insert a new value." The naive way is verbose:

```rust
let mut counts = HashMap::new();

if counts.contains_key(&word) {
    *counts.get_mut(&word).unwrap() += 1;
} else {
    counts.insert(word.clone(), 1);
}
```

The entry API does this cleanly:

```rust
let mut counts = HashMap::new();
*counts.entry(word).or_insert(0) += 1;
```

`entry(word)` returns an `Entry` enum (occupied or vacant). `or_insert(0)` inserts the value 0 if the entry is vacant, then returns a mutable reference to the value (whether it was just inserted or was already there). The `*` dereferences to access the value, and `+= 1` increments it.

This pattern is so common (counting word frequencies, building up groups, accumulating sums) that the entry API is one of the most-used parts of `HashMap`.

### A Word Count Example

```rust
use std::collections::HashMap;

fn main() {
    let text = "the quick brown fox jumps over the lazy dog the cat";
    let mut counts: HashMap<&str, u32> = HashMap::new();

    for word in text.split_whitespace() {
        *counts.entry(word).or_insert(0) += 1;
    }

    println!("{counts:?}");
    // {"the": 3, "cat": 1, "fox": 1, ...}
}
```

The entry API turns three or four lines of explicit checking into one line.

### Common Methods

```rust
scores.contains_key("Alice");           // true
scores.len();                            // number of entries
scores.is_empty();                       // false
scores.remove("Alice");                  // returns Option<V>
scores.clear();                          // empties the map
```

Most of what you need is here or accessible via the entry API.

---

## C. `HashSet<T>` and `BTreeMap<K, V>`

Two related collections worth knowing.

### `HashSet<T>`

A `HashSet<T>` is a collection of unique values. It is essentially a `HashMap<T, ()>` with a more convenient API:

```rust
use std::collections::HashSet;

fn main() {
    let mut visited: HashSet<i32> = HashSet::new();

    visited.insert(1);
    visited.insert(2);
    visited.insert(2);              // already present, no effect
    visited.insert(3);

    println!("{}", visited.contains(&2));    // true
    println!("{}", visited.len());            // 3
}
```

`insert` returns `bool`: `true` if the value was new, `false` if it was already present. This is useful for deduplication-while-iterating patterns:

```rust
let mut seen = HashSet::new();
let unique: Vec<i32> = numbers
    .iter()
    .filter(|&&x| seen.insert(x))
    .copied()
    .collect();
```

The `seen.insert(x)` is `true` only the first time `x` is encountered, which is exactly when you want to keep it.

### Set Operations

`HashSet` supports the standard set-theoretic operations:

```rust
let a: HashSet<i32> = [1, 2, 3, 4].iter().copied().collect();
let b: HashSet<i32> = [3, 4, 5, 6].iter().copied().collect();

let intersection: HashSet<&i32> = a.intersection(&b).collect();        // {3, 4}
let union: HashSet<&i32> = a.union(&b).collect();                      // {1, 2, 3, 4, 5, 6}
let difference: HashSet<&i32> = a.difference(&b).collect();            // {1, 2}
let symmetric: HashSet<&i32> = a.symmetric_difference(&b).collect();   // {1, 2, 5, 6}
```

These return iterators of references; the `collect` calls turn them into concrete sets.

### When to Use HashSet

Use `HashSet<T>` when:

- You need to test for membership efficiently.
- You need to deduplicate values.
- You need set-theoretic operations.

If you only need to check whether you have seen a value before, a `HashSet` is much faster than scanning a `Vec` for each check.

### `BTreeMap<K, V>`

`BTreeMap<K, V>` is a sorted map. Like `HashMap`, it stores key-value pairs, but the entries are kept in sorted order by key:

```rust
use std::collections::BTreeMap;

fn main() {
    let mut scores = BTreeMap::new();
    scores.insert("Charlie", 90);
    scores.insert("Alice", 95);
    scores.insert("Bob", 87);

    for (name, score) in &scores {
        println!("{name}: {score}");
    }
    // Always prints in alphabetical order:
    // Alice: 95
    // Bob: 87
    // Charlie: 90
}
```

The sorting is by the natural ordering of the key type. Strings are sorted alphabetically, integers numerically, etc.

### Performance Differences

`BTreeMap` has different performance characteristics than `HashMap`:

| Operation | `HashMap`           | `BTreeMap`          |
|-----------|---------------------|---------------------|
| Insert    | O(1) average        | O(log n)            |
| Lookup    | O(1) average        | O(log n)            |
| Delete    | O(1) average        | O(log n)            |
| Iteration order | unspecified   | sorted by key       |

`HashMap` is faster for individual operations. `BTreeMap` is slower per operation but provides ordering "for free."

### When to Use BTreeMap

Use `BTreeMap<K, V>` when:

- You need to iterate in sorted order.
- You need range queries (find all entries with keys between X and Y).
- You need deterministic iteration order across program runs.

Use `HashMap<K, V>` when:

- You only need point lookups (no range queries).
- Iteration order does not matter.
- You want maximum throughput.

For most code, `HashMap` is the default. Reach for `BTreeMap` when ordering is genuinely useful.

### Range Queries on BTreeMap

One of `BTreeMap`'s killer features is range queries:

```rust
let mut events: BTreeMap<u64, &str> = BTreeMap::new();
events.insert(100, "start");
events.insert(150, "checkpoint");
events.insert(200, "milestone");
events.insert(250, "checkpoint");
events.insert(300, "end");

// Get all events between time 150 and 250
for (time, event) in events.range(150..=250) {
    println!("{time}: {event}");
}
```

This is O(log n) to find the start of the range, then O(k) to walk through the k matching entries. With a `HashMap`, you would have to iterate the entire map and filter.

### `BTreeSet<T>`

By analogy with `HashSet`, there is also `BTreeSet<T>`. It is a sorted set of unique values, with the same trade-offs as `BTreeMap`. The standard library uses it less often than `BTreeMap`, but it is available when needed.

---

## D. Iterators and the `Iterator` Trait

Everything you have done with `for` loops in Rust has used the iterator system implicitly. This section makes it explicit.

### The `Iterator` Trait

The `Iterator` trait is defined approximately like this:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    
    // Many other methods with default implementations
}
```

A type implements `Iterator` by providing a `next` method. Calling `next()` returns the next value as `Some(item)` or `None` when the sequence is exhausted. That single method is enough; everything else (`map`, `filter`, `collect`, `sum`, etc.) is built on top of it.

### Creating an Iterator

For a collection, three common iterator-creating methods:

```rust
let v = vec![1, 2, 3];

// Iterate by reference: yields &T
let r = v.iter();

// Iterate by mutable reference: yields &mut T (requires the collection to be mut)
let mut v = vec![1, 2, 3];
let m = v.iter_mut();

// Iterate by value: yields T (consumes the collection)
let v = vec![1, 2, 3];
let o = v.into_iter();
```

The three forms produce iterators with different element types:

- `iter()` produces an iterator of references (`&T`).
- `iter_mut()` produces an iterator of mutable references (`&mut T`).
- `into_iter()` produces an iterator of owned values (`T`).

Choose based on what you need to do with the elements. For reading, use `iter()`. For modifying in place, use `iter_mut()`. For consuming the collection, use `into_iter()`.

### What `for` Does Behind the Scenes

The `for` loop is sugar for calling `next()` until it returns `None`:

```rust
let v = vec![1, 2, 3];

// This:
for x in v.iter() {
    println!("{x}");
}

// Is equivalent to this:
let mut iter = v.iter();
while let Some(x) = iter.next() {
    println!("{x}");
}
```

The compiler does the conversion automatically. You will rarely call `next()` directly, but knowing what is happening helps when you read iterator code.

### Iterators Are Lazy

Calling an iterator method that returns another iterator does not do any work. The transformation is recorded but not executed. Work happens only when something consumes the iterator:

```rust
let v = vec![1, 2, 3, 4, 5];

let doubled = v.iter().map(|x| x * 2);   // nothing computed yet

// Now consume:
for d in doubled {
    println!("{d}");                      // prints 2, 4, 6, 8, 10
}
```

The `map` call returns a new iterator that "knows how to" double each element. It does not produce a new vector. The doubling happens as the `for` loop pulls values out.

This laziness is what makes long iterator chains efficient. If you write:

```rust
let result: Vec<i32> = v.iter()
    .map(|x| x * 2)
    .filter(|x| x > &5)
    .take(3)
    .collect();
```

There is one pass through the data. The compiler turns the entire chain into a single loop with no intermediate collections. We will return to this in Section G.

### Iterators That Are Not Collections

Many things are iterators without being collections:

```rust
// Ranges
for i in 0..10 { /* ... */ }

// String characters
for c in "hello".chars() { /* ... */ }

// Lines from a file
for line in std::io::stdin().lines() { /* ... */ }

// Network packets, sensor readings, generated values, etc.
```

The iterator abstraction is general. Anything that produces a sequence of values, lazy or not, can implement `Iterator`. The same `map`, `filter`, `collect`, etc. methods work for all of them.

This is one of the things that makes iterators so powerful: learning the iterator API once gives you tools that work everywhere in Rust.

---

## E. Iterator Adaptors: `map`, `filter`, `flat_map`, `take`, `skip`

Iterator adaptors are methods that return new iterators. They transform a sequence into another sequence. They are lazy: calling them only sets up the transformation; nothing happens until the iterator is consumed.

### `map`

`map` transforms each element by applying a closure:

```rust
let numbers = vec![1, 2, 3, 4, 5];
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
println!("{doubled:?}");        // [2, 4, 6, 8, 10]
```

The closure receives each element and returns a new value. The output iterator yields the transformed values.

`map` is one of the most-used iterator methods. Whenever you want "for each element, produce one new element," `map` is the answer.

### `filter`

`filter` keeps only the elements that match a predicate:

```rust
let numbers = vec![1, 2, 3, 4, 5, 6, 7];
let even: Vec<&i32> = numbers.iter().filter(|x| **x % 2 == 0).collect();
println!("{even:?}");           // [2, 4, 6]
```

The closure receives each element (as a reference, even with `iter()`) and returns `bool`. Elements where the closure returns `true` are kept; the rest are skipped.

The double dereference `**x` is because `iter()` produces `&i32`, and `filter` passes references to its closure, so the closure receives `&&i32`. Each `*` strips one layer of reference.

### Combining `map` and `filter`

```rust
let numbers = vec![1, 2, 3, 4, 5, 6, 7];

let result: Vec<i32> = numbers
    .iter()
    .filter(|x| **x % 2 == 0)
    .map(|x| x * x)
    .collect();

println!("{result:?}");         // [4, 16, 36]
```

Read this top to bottom: take the iterator, keep even numbers, square each one, collect into a vector. The chain reads like a description of what you want, not a recipe for how to compute it.

### `flat_map`

`flat_map` is `map` followed by flattening. It is useful when each element produces a sequence, and you want to combine all the sequences:

```rust
let words = vec!["hello", "world"];
let letters: Vec<char> = words.iter().flat_map(|w| w.chars()).collect();
println!("{letters:?}");        // ['h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd']
```

Each word is mapped to its iterator of characters. `flat_map` then flattens those iterators into a single sequence.

A common use case: processing nested collections without explicitly nesting loops:

```rust
let groups = vec![vec![1, 2, 3], vec![4, 5], vec![6]];
let all: Vec<i32> = groups.into_iter().flat_map(|g| g.into_iter()).collect();
println!("{all:?}");            // [1, 2, 3, 4, 5, 6]
```

`flat_map` here flattens a `Vec<Vec<i32>>` into a `Vec<i32>`.

### `take` and `skip`

`take(n)` produces the first `n` elements; `skip(n)` discards the first `n` elements and produces the rest:

```rust
let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

let first_three: Vec<&i32> = numbers.iter().take(3).collect();
println!("{first_three:?}");    // [1, 2, 3]

let after_three: Vec<&i32> = numbers.iter().skip(3).collect();
println!("{after_three:?}");    // [4, 5, 6, 7, 8, 9, 10]

let middle: Vec<&i32> = numbers.iter().skip(2).take(3).collect();
println!("{middle:?}");         // [3, 4, 5]
```

`take` and `skip` are useful for pagination, sampling, or trimming sequences.

### A Few Other Useful Adaptors

A short list of other adaptors you will encounter:

- **`enumerate()`**: produces `(index, value)` tuples (you saw this in Module 4).
- **`zip(other)`**: combines two iterators into one of pairs.
- **`chain(other)`**: produces the elements of one iterator followed by the elements of another.
- **`rev()`**: reverses the iterator (only works on iterators that know their length).
- **`step_by(n)`**: yields every nth element.
- **`take_while(pred)` / `skip_while(pred)`**: take/skip while a predicate is true.
- **`cloned()` / `copied()`**: turn an iterator of references into an iterator of owned values.

The standard library documentation has the full list. For most purposes, the half-dozen methods covered here are enough.

### Closure Captures in Iterator Methods

Closures passed to iterator methods can capture state from the surrounding scope:

```rust
let threshold = 50;
let big: Vec<i32> = numbers.into_iter()
    .filter(|&x| x > threshold)
    .collect();
```

The closure captures `threshold`. Iterator methods are happy to accept any of the closure categories (`Fn`, `FnMut`, `FnOnce`) covered in Module 5.

---

## F. Consuming Adaptors: `collect`, `sum`, `fold`, `any`, `all`

A consuming adaptor pulls all the values out of an iterator and produces a final result. They are how iterator chains terminate.

### `collect`

`collect` gathers an iterator's output into a collection:

```rust
let numbers = vec![1, 2, 3, 4, 5];

let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
let as_set: HashSet<i32> = numbers.iter().copied().collect();
let as_string: String = vec!['h', 'i'].into_iter().collect();
```

`collect` is generic: it can produce any collection that knows how to be built from an iterator. The target type tells it what to produce, either through a type annotation on the variable or with the turbofish:

```rust
let doubled = numbers.iter().map(|x| x * 2).collect::<Vec<i32>>();
```

`collect` works with `Vec<T>`, `HashMap<K, V>`, `HashSet<T>`, `String`, and many other types.

### `collect` with Result

A particularly useful pattern: an iterator of `Result`s collected into a single `Result<Vec<T>, E>`:

```rust
let inputs = vec!["1", "2", "3", "abc", "4"];

let parsed: Result<Vec<i32>, _> = inputs.iter().map(|s| s.parse::<i32>()).collect();

match parsed {
    Ok(values) => println!("{values:?}"),
    Err(e) => println!("parse error: {e}"),
}
```

`collect` on an iterator of `Result<T, E>` produces a `Result<Vec<T>, E>`. If all elements are `Ok`, you get the vector. If any is `Err`, you get the first error. This is the idiomatic way to handle a sequence of fallible operations.

### `sum` and `product`

Numeric reductions:

```rust
let numbers = vec![1, 2, 3, 4, 5];

let total: i32 = numbers.iter().sum();          // 15
let product: i32 = numbers.iter().product();    // 120
```

Both methods require the iterator to produce numeric types (or types implementing the appropriate trait).

### `fold`

`fold` is the general-purpose accumulator. It takes an initial value and a closure that combines the accumulator with each element:

```rust
let numbers = vec![1, 2, 3, 4, 5];

let sum = numbers.iter().fold(0, |acc, x| acc + x);             // 15
let product = numbers.iter().fold(1, |acc, x| acc * x);          // 120
let concatenated = vec!["a", "b", "c"].iter().fold(
    String::new(),
    |mut acc, s| { acc.push_str(s); acc }
);                                                                // "abc"
```

The closure receives the running accumulator and the next element, and returns the new accumulator. After the iterator is exhausted, `fold` returns the final accumulator.

`fold` is more general than `sum` or `product`. Anything you can express as "start with X; for each element, update X" can be a `fold`.

### `count`

```rust
let numbers = vec![1, 2, 3, 4, 5];
let count = numbers.iter().count();             // 5

let even_count = numbers.iter().filter(|x| **x % 2 == 0).count();    // 2
```

`count` walks the iterator and returns how many elements it produced. For a known-size collection, prefer `.len()` directly; `count()` is for iterators where the size is not known in advance.

### `any` and `all`

These return `bool` based on a predicate:

```rust
let numbers = vec![1, 2, 3, 4, 5];

let has_even = numbers.iter().any(|x| x % 2 == 0);          // true
let all_positive = numbers.iter().all(|x| *x > 0);           // true
let all_even = numbers.iter().all(|x| x % 2 == 0);           // false
```

`any` returns `true` if any element matches; `all` returns `true` if every element matches. Both short-circuit: `any` stops at the first match, `all` stops at the first non-match.

### `find` and `position`

`find` returns the first element matching a predicate; `position` returns its index:

```rust
let numbers = vec![1, 2, 3, 4, 5];

let first_even = numbers.iter().find(|x| **x % 2 == 0);         // Some(&2)
let position = numbers.iter().position(|x| *x > 3);              // Some(3)
```

Both return `Option`: `Some` if a match is found, `None` if the iterator is exhausted without a match.

### `min` and `max`

```rust
let numbers = vec![3, 1, 4, 1, 5, 9, 2, 6];

let smallest = numbers.iter().min();            // Some(&1)
let largest = numbers.iter().max();             // Some(&9)
```

For iterators that might be empty, the `Option` return makes sense. For custom comparison, use `min_by` / `max_by` (with a comparator closure) or `min_by_key` / `max_by_key` (with a key extractor).

### Choosing the Right Consumer

A summary:

| Goal                           | Method                |
|--------------------------------|-----------------------|
| Build a collection             | `collect()`           |
| Sum or multiply numbers        | `sum()`, `product()`  |
| General accumulation           | `fold()`              |
| Count matching elements        | `filter().count()`    |
| Test for any match             | `any()`               |
| Test that all match            | `all()`               |
| Find first match               | `find()`              |
| Find index of first match      | `position()`          |
| Extreme value                  | `min()`, `max()`      |

When in doubt, `fold` can express almost anything. The other methods are convenient specializations.

---

## G. Chaining Iterators and Performance Considerations

This section is about why iterator code is fast and what to do (and not do) to keep it that way.

### Chained Iterators Compile to Tight Loops

A long iterator chain compiles to a single tight loop, not to multiple passes through the data. Consider:

```rust
let result: i32 = numbers
    .iter()
    .filter(|x| **x > 0)
    .map(|x| x * x)
    .take(10)
    .sum();
```

Naively, you might expect this to:

1. Walk `numbers`, build a new iterator of references where the value is positive.
2. Walk that iterator, build a new iterator of squared values.
3. Walk that iterator, taking the first 10 elements.
4. Walk those 10 elements, summing them up.

That would be four separate passes with three intermediate iterators. In fact, the optimizer compiles all of these into a single pass:

```
for each x in numbers:
    if x > 0:
        let squared = x * x
        accumulate squared into the sum
        if we have 10 squared values:
            break
```

The intermediate iterators do not exist at runtime. They are compile-time abstractions that the optimizer fuses into a single loop.

This is the "zero-cost abstraction" principle in action. The high-level iterator code is just as fast as the hand-written loop. You pay nothing for the abstraction.

### Verifying with Assembly

If you want to confirm this for yourself, the Rust Playground can show the assembly output (you saw this in Lab 1 and the integer overflow demo). For Release builds, iterator chains compile to assembly indistinguishable from manually-written loops. The fused loop is what the compiler produces.

### When Chains Are Slow

The fast pattern depends on the optimizer's ability to inline closures and fuse operations. Two specific things prevent this:

**Boxed iterators (`Box<dyn Iterator>`).** When you erase the iterator type, the optimizer cannot see through the boxing. Each call to `next()` becomes a virtual dispatch. Use this only when you genuinely need to return different iterator types from different code paths.

**Calls into separately-compiled crates.** If your iterator chain crosses a crate boundary into a non-inlinable function, the optimizer cannot fuse across that boundary. Most of the time this is not an issue (Rust's generics force inlining at use sites), but for very hot code, it can matter.

For everyday code, neither of these is a concern. Write your iterator chains, and the compiler will produce excellent code.

### Imperative Loops vs Iterators

A common question: "should I write a `for` loop or an iterator chain?"

For most cases, the iterator chain is preferable:

- It is shorter.
- It is harder to write incorrectly.
- Clippy will recommend it.
- It composes with other iterator methods naturally.

For some cases, an explicit loop is clearer:

- When the body has many statements, complex control flow, or multiple side effects.
- When you need to break early from non-trivial logic.
- When the loop variable is used after the loop.

The two compile to similar code. The choice is mostly stylistic. In doubt, write the iterator chain first; if it becomes contorted, fall back to the loop.

### Avoiding Unnecessary Allocation

A common antipattern is collecting an intermediate iterator only to immediately iterate over the result:

```rust
// Wasteful: collects into a Vec, then iterates
let evens: Vec<i32> = numbers.iter().filter(|x| **x % 2 == 0).copied().collect();
for x in &evens {
    println!("{x}");
}

// Better: iterate the iterator directly
for x in numbers.iter().filter(|x| **x % 2 == 0) {
    println!("{x}");
}
```

The first version allocates a vector that is immediately discarded. The second version iterates lazily, with no allocation. The result is identical, but the second version is faster.

The rule of thumb: collect into a `Vec` when you actually need the vector for subsequent operations. If you only iterate it once, skip the collection.

### Iterator Methods Return Different Types

You may have noticed that the inferred type of an iterator chain is something like `std::iter::Map<std::iter::Filter<std::slice::Iter<i32>, ...>, ...>`. This is because each adaptor returns a different concrete type, and the type contains the closures.

This is a feature, not a bug. The unique types are what enable monomorphization and inlining: the compiler generates specialized code for each iterator chain. If all iterators were the same type, this would not be possible.

For most code, you never see these types because you do not name them. They appear in error messages occasionally; just read past them and look at the actual error.

### Summary of Performance Properties

A few rules of thumb for fast iterator code:

1. **Prefer iterator chains over manual loops.** They compile to the same code but are easier to write and read.
2. **Do not collect intermediate results unless necessary.** Lazy evaluation is faster.
3. **Avoid `Box<dyn Iterator>` unless you specifically need it.** It defeats inlining.
4. **Always benchmark in release mode.** Debug builds do not optimize iterators.
5. **Trust the compiler.** Idiomatic Rust iterator code is fast; if you are tempted to "optimize" it into a loop, benchmark first.

---

## Module Summary

- `Vec<T>` is the workhorse collection: ordered, growable, heap-allocated. Most code uses it as the default sequence type.
- `HashMap<K, V>` is the workhorse map. The entry API is the idiomatic way to handle insert-or-modify patterns.
- `HashSet<T>` is for unique values; `BTreeMap<K, V>` is for sorted key-value pairs with range query support.
- The `Iterator` trait unifies traversal across all collection types and many other sources of sequences. Iterators are lazy; transformations are recorded but not executed until consumed.
- Iterator adaptors (`map`, `filter`, `flat_map`, `take`, `skip`) transform sequences into new sequences. Consuming adaptors (`collect`, `sum`, `fold`, `any`, `all`, `find`) produce final values.
- Iterator chains compile to tight loops with no abstraction overhead. Idiomatic Rust uses chains heavily.

## Discussion Questions

1. The entry API for `HashMap` replaces a common three-line pattern with one line. What does this tell you about how Rust's standard library is designed compared to languages where this kind of API is rare?
2. Iterators are lazy: nothing happens until something consumes them. What practical benefits does this design have? What are the costs to a developer who is not expecting laziness?
3. The standard library has both `HashMap` and `BTreeMap`. They have similar APIs but very different performance characteristics. Identify a real-world data structure problem where each one is the right choice.

