# Modules 11 and 12: Collections and Text Processing
## Lab 8 -- Solutions and Checkpoint Answers

> **Course:** Mastering Rust
> **Purpose:** This file contains completed code for every task and written answers
> for every Checkpoint and Reflection question. Use it to verify your work after
> attempting each task yourself. Reading the solution before attempting the task
> defeats the purpose of the exercise.

---

## Exercise 1 -- Vec<T>: The Workhorse Collection

### Checkpoints

**1. When does pre-allocation help? What happens with overflow?**

Pre-allocation helps when you know roughly how many elements you will add. The `Vec::with_capacity(n)` form allocates room for n elements up front; subsequent pushes do not need to reallocate until capacity is exceeded. Without pre-allocation, the vector grows by doubling its capacity each time it fills, which involves allocating a new buffer, copying all existing elements, and freeing the old buffer.

For most code, the default growth strategy is fine; the occasional reallocation cost is amortized away. Pre-allocation matters for tight loops where you push many elements and want to avoid the overhead, or for performance-critical paths where the unpredictability of reallocation is undesirable.

If you push 200 elements into a `Vec::with_capacity(100)`, the vector reallocates when it hits 100. It doubles to 200, doubles again to 400 if needed, and so on. The pre-allocation does not limit the size; it only sets the initial capacity. You always have room for more elements; pushing past the capacity just triggers reallocation, same as if you had started with `Vec::new()`.

In practice, the standard convention is: use `Vec::new()` when the size is unknown or small; use `Vec::with_capacity(n)` when you know n approximately. The performance difference matters for hot paths and is invisible elsewhere.

**2. When is `get()` right vs direct indexing?**

`get(index)` is right when the index might be out of bounds for legitimate reasons. Examples:

- Walking a vector with an index that comes from user input or external data.
- Looking up an item that may or may not exist (parallel arrays where some entries are missing).
- Defensive code that handles edge cases without panicking.

Direct indexing (`v[index]`) is right when you know the index is valid. Examples:

- Indexing into a vector by a loop counter that you just constructed from `0..v.len()`.
- Accessing an element immediately after `push`ing or `insert`ing it.
- Working in a function that has a precondition that the caller respects.

The choice is about intent: `get` says "the access might fail and I want to handle that"; direct indexing says "this access should always succeed and a failure is a bug." A panic from `v[10]` indicates a programming error; `get(10)` returning `None` is normal operation.

A common pattern in real code is to use direct indexing inside tight loops where the bounds are known and to use `get` at module boundaries where data comes from outside. The choice communicates the assumption to anyone reading the code.

**3. What does `&[i32]` signal beyond accepting more types?**

It signals that the function does not need to mutate the data or take ownership. The reader sees `&[i32]` and immediately understands: "this function reads but does not modify, and the caller retains ownership."

By contrast:

- `&mut [i32]` would signal "I might modify, but I do not take ownership."
- `Vec<i32>` would signal "I take ownership of this vector and might consume it."
- `&Vec<i32>` would signal "I read from this specific vector type" (overly restrictive).

The slice type is also the most general read-only interface. Functions taking `&[i32]` work with vectors, arrays, sub-slices, and even iterator outputs that produce slices. This generality is itself a signal: "I do not care where this data comes from; just give me a contiguous sequence to read."

This pattern is universal in idiomatic Rust. Whenever you write a function that reads sequence data, prefer `&[T]` over `&Vec<T>`. The signature is more honest about what the function needs and more flexible for callers.

---

## Exercise 2 -- HashMap<K, V> and the Entry API

### Checkpoints

**1. What is the practical difference between the naive count and the entry API?**

For a million-line input, the naive count does roughly three million hash-and-lookup operations: one `contains_key`, one `get`, one `insert` per word. The entry API does one million.

The hash function is not free; for a `String` key, it walks the entire string's bytes to produce the hash. Tripling the number of hash computations triples the dominant cost of the operation. On real-world data, the entry API can be measurably faster: typically 2-3x for hot paths, sometimes more.

Beyond performance, the entry API is also clearer. The intent ("count occurrences") is expressed in one line. The reader does not have to mentally trace through a contains/get/insert sequence to confirm that the logic is correct. The clarity matters as much as the performance for everyday code.

The pattern generalizes: any time you have "check if present, then either update or insert," reach for the entry API. The naive form is almost always longer, slower, and harder to read.

**2. What are the downsides of unordered HashMap iteration?**

Several concerns:

- **Tests become flaky.** A test that asserts on the iteration order of a `HashMap` will pass or fail depending on the hash function's behavior, which can vary across versions or even runs (Rust uses randomized hash seeds to defeat denial-of-service attacks). Tests should use sorted output (or `BTreeMap`).
- **Reports and logs become non-deterministic.** A report generated from `HashMap` iteration may have entries in different orders across runs, making it harder to diff or compare.
- **Debug output is harder to read.** Looking for a specific entry in unordered output is slower than in sorted output.

Unordered iteration is fine when:

- The consumer does not care about order (e.g., computing a sum, filtering, transforming).
- The output is processed by something that sorts later (a UI, a serializer).
- Performance matters and order does not.

For user-facing reports, debug logs, configuration listings, and tests, prefer `BTreeMap` (or sort the entries explicitly) so the output is deterministic. For internal computation, `HashMap` is faster and the order does not matter.

**3. `or_insert_with(Vec::new)` vs `or_insert(Vec::new())`.**

The closure form `or_insert_with(Vec::new)` only calls `Vec::new()` if the entry is vacant. The value form `or_insert(Vec::new())` constructs the empty `Vec` every time, regardless of whether the entry exists. For an empty vec, the cost difference is negligible.

The difference matters when:

- The default is expensive to construct (a large default config, a precomputed lookup table).
- The default has side effects (logging, registering with a service).
- You want to express clearly that the construction is conditional.

For trivial defaults like 0 or "", `or_insert` is fine. For collection types like `Vec::new()` and `HashMap::new()`, the convention is `or_insert_with` even though the cost is small; it expresses intent ("construct on demand") and matches the style used in the standard library documentation.

There is also `or_default()` for types implementing `Default`:

```rust
counts.entry(key).or_default().push(value);   // Vec implements Default
```

This is the most concise form for default-able types. The three forms (`or_insert`, `or_insert_with`, `or_default`) are equivalent in capability but differ in conciseness and intent.

---

## Exercise 3 -- HashSet and BTreeMap

### Checkpoints

**1. Why does Rust allow side effects in `filter`?**

Rust does not enforce purity of closures. A closure can do anything: print, modify captured state, return early, etc. The borrow checker enforces ownership and borrowing rules, but not functional purity.

The `seen.insert(w)` pattern uses this freedom. The closure does have a side effect (modifying `seen`), but the side effect is the very mechanism that filters: an item is kept only if it has not been seen before, and seeing it is what makes the filter discard duplicates.

Many functional languages would discourage this pattern in favor of an explicit accumulator (fold over the iterator with a tuple of (seen, kept)). Rust accepts both styles. The side-effect-in-filter form is more concise and idiomatic in Rust; the fold form is more "pure" but more verbose.

The general convention is that closures can have side effects, but readers should be able to predict what those effects are. A `filter` closure that does I/O or modifies unrelated state would be surprising; one that maintains a `seen` set is a recognized pattern.

The Rust documentation does not formally call out this idiom, but it appears regularly in production code and idiomatic examples. As you read more Rust, you will see it become natural.

**2. HashMap vs BTreeMap performance for 10,000 entries.**

For point lookups, HashMap is O(1) (typically a few nanoseconds) and BTreeMap is O(log n) (around 14 levels deep for 10,000 entries, so maybe 50-100ns). For 10,000 entries, the difference is real but small.

Where the difference matters:

- **Hot paths.** If you are doing a million lookups per second, the cumulative difference is significant. BTreeMap would add 30-50ms; HashMap would add 5-10ms.
- **Latency-sensitive code.** A web server that hits a map per request might prefer HashMap to keep p99 latency low.

Where the difference does not matter:

- **Occasional lookups.** Looking up something once per request out of thousands of operations: the few nanoseconds difference is invisible.
- **Cache-cold workloads.** If the map is rarely accessed, the per-operation cost is dominated by cache effects, not algorithmic complexity.

The general rule: use HashMap unless you need BTreeMap's ordering. The performance difference favors HashMap; the cases where BTreeMap is preferred are about correctness (need sorted iteration, range queries, deterministic output), not performance.

**3. Range queries: BTreeMap vs HashMap for 1M entries, 10 results.**

BTreeMap's `range(start..end)` is O(log n + k) where n is the size of the map and k is the number of matches. For 1 million entries:

- Finding the start: O(log 1,000,000) ≈ 20 levels.
- Walking k=10 matching entries: O(10).
- Total: a few hundred nanoseconds.

For HashMap, you would iterate every entry and filter:

- Walking 1,000,000 entries: O(1,000,000).
- Checking each against the predicate: O(1) per entry.
- Total: tens of milliseconds, even with simple comparisons.

The difference is roughly 10,000x or more. For range queries on large maps, BTreeMap is not just slightly better; it is in a different complexity class.

This is the central reason BTreeMap exists. For pure key-value lookups, HashMap is faster. For range queries, BTreeMap is dramatically faster. The choice depends entirely on whether you do range queries; if you do, BTreeMap is non-negotiable.

---

## Exercise 4 -- Iterators and Adaptors

### Checkpoints

**1. When does lazy evaluation save significant work?**

Several common cases:

- **Searching for an element.** `iter.find(|x| condition(x))` stops as soon as it finds a match. With eager evaluation, you would have to compute all transformations on all elements first, then search.
- **Limiting output size.** `iter.take(10)` stops after 10 elements. With eager evaluation, you would compute everything first, then discard most.
- **Pipeline-then-test patterns.** `iter.map(|x| expensive(x)).any(|y| condition(y))` stops at the first match. The `map` only runs as far as needed.
- **Infinite sequences.** Generators like `(1..).map(|n| compute(n)).take_while(|x| x < threshold)` only work with lazy evaluation; eager evaluation would loop forever.

The savings depend on the data. For 1000 elements where the first matches, the savings are 999 transformations. For 1 million elements where the 10th from the end matches, the savings are smaller. The point is that lazy evaluation lets you express the computation cleanly without manually optimizing for early exits.

The compiler also benefits. A lazy chain compiles to a single loop with the transformations and tests inlined. There are no intermediate allocations, no virtual dispatch, no unnecessary computation. The generated code is often identical to a hand-written loop.

**2. Could you do `(1..).map(square).take(5)` with eager evaluation?**

Not directly. Eager evaluation would try to enumerate all of `(1..)` (every positive integer), apply `square` to each, and only then `take(5)`. Since the range is infinite, the program would loop forever.

You could simulate the laziness manually:

```rust
let mut result = Vec::new();
let mut n = 1;
while result.len() < 5 {
    result.push(n * n);
    n += 1;
}
```

This works but is more verbose. The iterator version (`(1..).map(|n| n*n).take(5).collect()`) expresses the same logic declaratively.

The benefit of lazy iterators is composability. You can write small, focused transformations and combine them without worrying about whether each step computes too much. The chain itself optimizes for what is actually needed at the end.

This kind of compositional efficiency is hard to achieve with eager evaluation. Either you write fused functions for each combination ahead of time (combinatorial explosion), or you accept the cost of intermediate work. Lazy evaluation gives you composability without the cost.

**3. `flat_map` vs `.map(...).collect::<Vec<...>>()` then flatten.**

The `flat_map` version processes everything in a single lazy chain:

```rust
words.iter().flat_map(|w| w.chars())     // lazy: one tight loop
```

The alternative does two passes:

```rust
words.iter().map(|w| w.chars().collect::<Vec<char>>()).flatten()
```

The differences:

- **Allocations.** The `flat_map` version does no intermediate allocation. The alternative allocates a `Vec<char>` for each word.
- **Performance.** The `flat_map` version is faster because of the lack of allocation and the better cache behavior.
- **Memory.** For large inputs, the intermediate `Vec<Vec<char>>` could be substantial.

The `flat_map` is the idiomatic form. The alternative is acceptable for occasional use but should not be the default.

A related point: `flat_map` is more general than just flattening. It maps each input to an iterator of outputs (which can have varying lengths) and concatenates them. The output count is not fixed by the input count, unlike `map` (which is one-to-one).

For practical pattern recognition: any time you find yourself writing `iter.map(f).flatten()` or `iter.map(f).collect::<Vec<Vec<_>>>().into_iter().flatten()`, use `iter.flat_map(f)` instead. The result is cleaner and more efficient.

---

## Exercise 5 -- Consuming Adaptors

### Checkpoints

**1. Why does `Result`-collecting short-circuit?**

The natural pattern is: "I want to compute a transformation on every element, but if any of them fails, the whole computation fails." Short-circuiting matches this intent.

In particular, the typical case is parsing or validating a list. If one element is invalid, the rest are usually irrelevant (you cannot proceed with partial data). Reporting the first error is enough; processing the rest would waste effort.

When you want all errors instead, the natural collection is `Vec<Result<T, E>>`. You can then iterate and inspect each element separately. The standard library provides this directly:

```rust
let results: Vec<Result<i32, _>> = inputs.iter().map(|s| s.parse::<i32>()).collect();

let oks: Vec<i32> = results.iter().filter_map(|r| r.as_ref().ok().copied()).collect();
let errs: Vec<_> = results.iter().filter_map(|r| r.as_ref().err()).collect();
```

For partition-based collection (split into successes and failures simultaneously), the `partition` method is useful:

```rust
let (oks, errs): (Vec<_>, Vec<_>) = results.into_iter().partition(Result::is_ok);
```

The trade-off between short-circuit and collect-all is about what the caller wants to do. For "parse this list, error out if anything fails," short-circuit. For "validate this form, show the user every problem," collect-all. The standard pattern handles the common case; the alternatives are available when needed.

**2. Fold for string concatenation vs `join`.**

The `vec.join(" ")` method is much shorter and more idiomatic:

```rust
let concatenated = words.join(" ");
```

This is the right choice when you have a `Vec` (or slice) of strings and want them concatenated with a separator. It handles the "no separator before the first element" logic internally.

Fold is appropriate when:

- The combining logic is non-trivial (more than just concatenate with separator).
- You are working with an iterator that does not have a `join` method.
- You want to accumulate state of a different type than the elements.
- The separator depends on the elements (e.g., different separators between certain kinds of elements).

The fold version is more general but more verbose. For the common case of "join with a separator," use `join`. Save fold for cases where the accumulation is genuinely complex.

A common related pattern: if you have an iterator (not a Vec), you can collect into a Vec first, then join:

```rust
let s: String = iter.collect::<Vec<_>>().join(", ");
```

Or use `intersperse` (in newer Rust versions) directly on the iterator. The right approach depends on what is available; the `join`-on-Vec form is the most widely compatible.

**3. Why doesn't `count` short-circuit?**

The semantics of `count` is "how many elements are in this iterator?" The only way to know the count is to walk all of them. There is no shortcut.

`any` and `all` can short-circuit because their question has a definite answer the moment they find a counterexample. `count` does not have a counterexample; you do not know the answer until you have walked everything.

The practical implication: `iter.count()` is O(n) and processes everything. For known-size collections, prefer `vec.len()` which is O(1).

For combinations like `iter.filter(p).count()`, the count still requires walking everything (because the predicate must be evaluated on each element to determine if it counts). There is no shortcut.

If you only need to know whether the count exceeds a threshold, use `iter.filter(p).take(threshold).count() == threshold`. This short-circuits at the threshold, which can be faster than counting all matches when the threshold is small and the data is large.

The general rule: prefer methods that match the operation's natural complexity. `count` is O(n); `any` and `all` are O(n) worst case but often faster; `len` on collections is O(1). Choose the one that fits.

---

## Exercise 6 -- String vs &str

### Checkpoints

**1. What pattern in Rust gives non-string types the same flexibility?**

The general pattern is "function takes a reference to a borrowed type that the owned type derefs to." For strings, this is `&str` (since `String` derefs to `str`). For sequences, this is `&[T]` (since `Vec<T>` derefs to `[T]`). For paths, this is `&Path` (since `PathBuf` derefs to `Path`). For C-compatible strings, this is `&CStr` (since `CString` derefs to `CStr`).

The convention: when you have an "owning" type and a "view into a buffer" type that the owning type holds, write functions that take the view. Deref coercion makes the call sites work for both. This is the same pattern that makes `&str` parameters accept `&String`; it generalizes to many other types.

The advantage is uniform: any function written this way is maximally flexible for callers. Anyone who has a String, a string literal, a slice, or anything else that derefs to `str` can pass it directly.

The same idea can extend to your own types. If you define a `MyString` that holds heap data and you also have a `MyStr` reference type, define `MyString` to deref to `MyStr` and write functions taking `&MyStr`. Callers get the flexibility automatically.

**2. Why does `+` consume the left operand?**

The `+` operator on strings is implemented as `add(self, rhs: &str) -> String` (the receiver takes ownership). This was chosen to allow the implementation to reuse the receiver's buffer.

Internally, `+` typically:
1. Takes the receiver's heap buffer.
2. Appends the right operand's bytes to it (possibly reallocating).
3. Returns the resulting `String` (which owns the buffer).

If both operands were borrowed, the implementation would need to allocate a new buffer and copy both into it. By consuming the left operand, the implementation can sometimes avoid one allocation.

The trade-off: the API is less convenient (you cannot use the left operand afterward). For two-string concatenation, `+` works but is not the friendliest. For more than two, the consumption pattern compounds (you would write `s1 + &s2 + &s3 + ...`, and each step consumes the previous result).

The `format!` macro is more user-friendly: it takes everything by reference and constructs a fresh string. The trade-off is one extra allocation (for the intermediate format string), which is negligible for most code.

In practice: use `+` only for two-operand concatenation and only when you do not need the left operand afterward. Use `format!` for everything else. This is the community convention.

**3. Why does `s.len()` return bytes instead of characters?**

Returning bytes is O(1). Returning character count would be O(n) because UTF-8 characters can be 1-4 bytes, and the only way to know the character count is to walk the string and count.

Rust's design philosophy is to make costs visible. If a method is O(n), you should see it; you should not call something that looks like a property accessor (like `len`) and unknowingly trigger a walk.

The trade-off: callers who want character count must explicitly call `chars().count()`, which is O(n). This is honest about the cost but inconvenient for cases that genuinely want characters (like checking if a name is at most 20 characters long).

Several languages take the opposite approach: their `length` returns characters (or "code points," or "grapheme clusters"), which is more semantically meaningful but more expensive. Some try to cache the count, which makes mutating strings more complex.

Rust's choice: byte count by default; character count when you explicitly ask. The reasoning is:

- Most code uses byte counts (memory allocation, file sizes, network buffers).
- Character count is genuinely ambiguous: does an emoji with skin tone count as one or two? What about combining characters? The choice of "what is a character" varies by use case.
- Making the cost explicit is honest.

For text processing that needs character semantics, use `chars()`, `char_indices()`, or the `unicode-segmentation` crate (which provides grapheme cluster iteration for the visually correct count).

---

## Exercise 7 -- Parsing with FromStr

### Checkpoints

**1. Why does `parse` need a type annotation?**

The `parse` method is generic: `fn parse<F: FromStr>(&self) -> Result<F, F::Err>`. The compiler needs to know what `F` is to call the right implementation.

In some contexts, the type can be inferred:

```rust
let n: i32 = "42".parse().unwrap();    // type annotation on let
let n = "42".parse::<i32>().unwrap();   // turbofish
let n = use_int("42".parse().unwrap()); // inferred from function signature
```

When there is no context to infer from, an annotation is required:

```rust
let n = "42".parse().unwrap();          // ERROR: cannot infer type
```

This is consistent with how Rust handles generics in general. The compiler refuses to guess; you must provide enough information to disambiguate.

The turbofish syntax (`::<i32>`) is specifically designed for this case: you want to constrain the generic without binding the result to a typed variable. It is occasionally awkward to write but unambiguous.

The alternative would be to require parse-style methods to be specific (separate `parse_i32`, `parse_f64`, etc.). This would be more familiar to developers from other languages but less flexible. Rust's choice favors uniformity at the cost of occasional type annotations.

**2. Why is `FromStr` separate from `From<&str>`?**

Two reasons.

**Different semantics.** `From<T>` is for infallible conversions; if `T -> U` cannot fail, it can be a `From` implementation. `FromStr` is for parsing, which is inherently fallible (the string might not represent a valid value). The two cases need different signatures: `From` returns `U` directly, `FromStr` returns `Result<U, Err>`.

**Different uses.** `From` integrates with the `?` operator (via the auto-conversion in error handling), with `Into`, and with generic code. `FromStr` integrates specifically with the `parse` method on strings, which is the canonical way to invoke parsing.

Some types implement both. For example, a `Color` type might implement `FromStr` to parse "#RRGGBB" and `From<(u8, u8, u8)>` to construct from explicit components. The two are not redundant; they serve different conversion paths.

The separation also encodes intent. When you see `T: FromStr`, you know the type can be parsed from text. When you see `T: From<&str>`, you know there is a one-way infallible conversion. The two traits document different capabilities.

A related pattern: `TryFrom` for fallible non-string conversions. The full hierarchy is:

- `From<T>` for infallible (panic on failure is not an option).
- `TryFrom<T>` for fallible from non-string types.
- `FromStr` for fallible from strings.

Each fills a specific niche. The standard library uses them consistently; following the convention makes your APIs feel familiar to other Rust developers.

**3. When would you collect all parse errors?**

The same situations as with general `Result`-collecting (Exercise 5 checkpoint 1):

- **Validating user input.** A form with multiple fields; you want to show all errors at once rather than make the user fix one at a time.
- **Reporting issues in a file.** Parsing a config file with multiple errors; you want to list all of them so the user can fix them in one pass.
- **Quality reporting.** Processing a batch and reporting on overall data quality.

The pattern is:

```rust
let (oks, errs): (Vec<_>, Vec<_>) = inputs
    .iter()
    .map(|s| s.parse::<i32>())
    .partition(Result::is_ok);

let values: Vec<i32> = oks.into_iter().map(Result::unwrap).collect();
let errors: Vec<_> = errs.into_iter().map(Result::unwrap_err).collect();

if !errors.is_empty() {
    eprintln!("Found {} errors:", errors.len());
    for (i, e) in errors.iter().enumerate() {
        eprintln!("  {i}: {e}");
    }
}
```

This is more verbose than `collect::<Result<Vec, _>>`, but it gives you all the information at once.

For real applications, libraries like `validator` provide structured "collect all errors" patterns with better ergonomics. The manual form shown here is the standard-library approach.

---

## Exercise 8 -- Putting It Together with regex

### Task 8.3 -- Pattern identification

1. **`Vec<LogEntry>` collected from iterator chain:** `entries: Vec<LogEntry> = log_lines.iter().filter_map(...).collect()`.

2. **`HashMap<String, u32>` for unordered counting:** `count_by_path` returns `HashMap<String, u32>`.

3. **`BTreeMap<u16, u32>` for sorted iteration:** `count_by_status` returns `BTreeMap<u16, u32>`; the iteration over it in `format_report` is sorted by key.

4. **`HashSet<String>` for uniqueness:** `unique_paths` returns `HashSet<String>`.

5. **Entry API for counting:** `*counts.entry(entry.level.clone()).or_insert(0) += 1` in `count_by_level`. Similar pattern in `count_by_status` and `count_by_path`.

6. **Iterator chain with filter and map:** `error_paths` uses `entries.iter().filter(...).map(...).collect()`.

7. **`filter_map` combining filter and map:** The main parsing chain `log_lines.iter().filter_map(|line| parse_log_line(line, &re)).collect()`. `filter_map` discards `None` results and unwraps `Some`, equivalent to `filter(is_some) + map(unwrap)`.

8. **`format!` with multiple substitutions:** Many places, e.g., `format!("[{}] {} {} -> {} ({}ms)", entry.timestamp, entry.method, entry.path, entry.status, entry.response_ms)` in the slow requests section.

9. **`format!` with width specifier:** `format!("  {level:>5}: {count}\n")` uses `{level:>5}` to right-align in width 5.

10. **`format!` with precision specifier:** `format!("Average response time: {:.2}ms\n", ...)` uses `{:.2}` for 2 decimal places.

11. **`parse()` with `.ok()`:** Inside `parse_log_line`, the chain `caps.name("status")?.as_str().parse().ok()?` parses and converts the `Result` to `Option` via `.ok()`, then uses `?` to propagate.

12. **Regex named capture groups:** `(?P<timestamp>[^\]]+)`, `(?P<level>\w+)`, etc. The `(?P<name>...)` syntax names the group; `caps.name("name")` retrieves it.

13. **Function taking `&[LogEntry]`:** All analysis functions (`count_by_level`, `count_by_status`, `count_by_path`, `average_response_time`, `slow_requests`, `unique_paths`, `error_paths`, `format_report`).

14. **Function returning `String`:** `format_report` returns `String` because it constructs a fresh string.

15. **`take(5)` to limit output:** In `format_report`, `slow.iter().take(5)` limits the slow requests printed to 5, and `sorted_errors.iter().take(5)` limits the error paths to 5.

### Task 8.4 -- Predict and verify

**Change A:** Replace `BTreeMap` with `HashMap` in `count_by_level`.

**Prediction:** The program will compile. The output will change: the iteration order over the level counts will no longer be alphabetical (DEBUG, ERROR, INFO, etc.); instead, it will be in some hash-table-determined order, which may vary across runs.

**Result:** As predicted, the program compiles. The output of "Counts by level" appears in a different order. This is a small example of why determinism matters for reports.

The lesson: choose the map type based on what kind of iteration you need. HashMap for unordered (when order does not matter) and BTreeMap for ordered (when it does). For user-facing reports and test assertions, BTreeMap eliminates a class of flakiness.

**Change B:** Replace `filter_map` with `map` and collect into `Vec<Option<LogEntry>>`.

**Prediction:** The collection type changes; subsequent code would need to handle `Option<LogEntry>` instead of `LogEntry`. Every iteration would have to check `if let Some(...)` or filter the Nones. The `unparseable_count` calculation would also change because `entries.len()` would be the count of all lines (including unparseable ones), and the count of successful parses would need to be computed.

**Result:** The change requires several modifications:

- `entries` becomes `Vec<Option<LogEntry>>`.
- Functions like `count_by_level(entries: &[LogEntry])` need a different signature, or you need to filter the Some values before passing.
- The cleanest fix is to filter at the call site: `count_by_level(&entries.iter().filter_map(|x| x.as_ref()).collect::<Vec<_>>())`, but this is verbose.

The lesson: `filter_map` is exactly the right tool for "parse and keep only the successes." Doing the equivalent manually requires either an extra collection step or carrying the `Option` through the rest of the program, both of which are worse.

**Change C:** Change `\w+` to `\w++` (invalid regex).

**Prediction:** The change makes the regex invalid. The `Regex::new(...)` call will return `Err`. Because the code uses `.expect("invalid regex")`, the program will panic at runtime with that message.

The error appears at runtime, not compile time, because regex patterns are runtime strings to Rust; the regex crate parses them when `Regex::new` is called. The compiler does not validate regex syntax.

**Result:** As predicted, the program panics at startup:

```
thread 'main' panicked at 'invalid regex: regex parse error: ...
```

The lesson: regex patterns are runtime data from the compiler's perspective. If you want compile-time regex validation, the `regex_macros` crate provides procedural macros that check patterns at compile time. For most code, runtime validation is acceptable; the regex is hard-coded in the source and you find errors quickly during development.

This is a real difference from languages where regex literals are part of the syntax (e.g., JavaScript's `/pattern/`). Rust treats regex as a library, which has trade-offs: more flexibility (the regex crate is replaceable, alternatives like `fancy-regex` exist) but no compile-time validation.

### Checkpoints

**1. Why use different map types for different counts?**

The choice is about what kind of iteration the downstream code does:

- **Counts by level and status:** The report iterates these and prints in order. Ordered output makes the report deterministic and readable. BTreeMap provides this for free.
- **Counts by path:** The error reporting sorts by count (descending), not by path. There is no need for BTreeMap's ordering; HashMap is faster.
- **Unique paths:** Just a count, no iteration needed. HashSet is fastest.

Using only HashMap throughout would make the report non-deterministic (level and status would appear in arbitrary order). Using only BTreeMap would slow down operations that do not need ordering.

The choice is not "one is always better" but "pick the one whose properties match the use." This is a common theme in collection types.

**2. Is `.expect("invalid regex")` acceptable on a hard-coded regex?**

For a hard-coded regex, yes. The regex string is a constant in the source code; if it does not parse, that is a programming error. Panicking at startup is appropriate: the developer sees the error immediately, and fixing it is just editing the regex string.

For dynamic regexes (loaded from config files, user input, etc.), `expect` is inappropriate. The pattern might be wrong for legitimate reasons (the user typed it badly), and panicking would crash the program. In those cases, return a `Result` and let the caller handle it.

The same principle applies to all `unwrap`/`expect` calls: panic when the failure indicates a bug; return `Result` when the failure is expected. Hard-coded values are bugs if they are wrong; user input is not.

A real-world refinement: even for hard-coded regexes, some projects prefer to wrap them in a `lazy_static!` or `OnceCell::new()` to validate at first use rather than every time. This is a micro-optimization (regex compilation is fast) but it is the standard pattern for production code.

**3. Could `format_report` use one `format!` instead of repeated `push_str`?**

In principle, yes. A single `format!` could include all the lines:

```rust
let report = format!(
    "=== Log Analysis Report ===\n\n\
     Total entries: {}\n\n\
     Counts by level:\n{}\n...",
    entries.len(),
    /* formatted level counts */,
    ...
);
```

But this becomes unmanageable for non-trivial reports. The format string itself becomes a many-line literal with embedded placeholders for everything; the arguments become a long list that must match exactly in order. Adding a section requires editing both the format string and the argument list. Removing a section requires the same.

The repeated `push_str` form has a different structure: each section is built independently and appended. Adding a section means adding `report.push_str(&format!("..."))`. Removing means deleting one block. The structure is easier to maintain.

For very short fixed reports, single `format!` works. For real reports with multiple sections, repeated `push_str` (or building up a `Vec<String>` and joining at the end) is more maintainable.

A middle ground: use `writeln!` to a string buffer, which is essentially `push_str` of a `format!` result but with the format-and-append in one call:

```rust
use std::fmt::Write;
let mut report = String::new();
writeln!(report, "=== Log Analysis Report ===").unwrap();
writeln!(report, "Total entries: {}", entries.len()).unwrap();
```

The `writeln!` macro is the standard tool for building strings incrementally. It is more concise than `push_str(&format!(...))` and the same in efficiency.

---

## Final Reflection Questions

These are open-ended; sample answers are provided as a guide.

**1. Steepest learning curve and most natural concept.**

Common patterns:

- Developers from C find Vec, HashMap, and HashSet familiar (similar to STL or other library types) but are surprised by the absence of certain operations and the strictness of the borrow checker on collection access. The iterator system can feel verbose initially.
- Developers from Java find the collection types familiar but the iterator system unfamiliar; Java Streams (which are conceptually similar) are usually a recent addition and not as universally used. Once they recognize the parallel, iterators become natural quickly.
- Developers from Python find the explicit type annotations on collections (`Vec<i32>`, `HashMap<String, i32>`) initially verbose but appreciate the type safety. The String/&str distinction is often the steepest part of the curve; Python has only one string type.
- Developers from functional languages find iterators very natural (similar to Haskell, Scala, etc.). The collection types are familiar by analogy.

The most common reflection: "The String/&str distinction was confusing at first, but once I learned the pattern (take &str, return String), it became automatic. The iterator system felt like a revelation once I stopped trying to write explicit for loops."

**2. Cross-language comparison: a feature of Rust iterators not in other languages.**

Sample answer focusing on zero-cost abstraction:

"Java Streams look syntactically similar to Rust iterators: `.stream().filter(...).map(...).collect(...)` in Java versus `.iter().filter(...).map(...).collect()` in Rust. But the performance characteristics are different.

Java Streams are objects on the heap. Each `.filter(...)` creates a new Stream object that holds a reference to the previous one and a reference to the closure. Method calls go through virtual dispatch (which the JIT may optimize, but the abstraction is heap-based). For very hot loops, the abstraction has measurable cost.

Rust iterators are types, not objects. Each call to `.filter(...)` returns a struct whose type encodes the entire pipeline. `vec.iter().filter(...).map(...).take(...)` has a type like `Take<Map<Filter<Iter<i32>, ...>, ...>>`. The type is unique and known at compile time. The compiler generates specialized code for it, with all the closures inlined and all the dispatch resolved.

The result is that Rust iterators have zero runtime overhead. The compiled code for a long iterator chain is identical to a hand-written for loop with the same logic. There is no allocation, no virtual dispatch, no overhead.

Java Streams are fast (the JIT does a good job for typical cases), but they are not free. For most code, the difference is invisible. For very hot paths (image processing, parsers, simulation inner loops), Rust iterators can outperform by 2-5x.

Rust gains: composability without cost. You can write declarative iterator chains and trust them to be as fast as imperative loops. This is one of the language's most distinctive features."

**3. Situation from the lab where you changed your mind about an approach.**

Sample answer:

"In Exercise 8, when implementing `format_report`, I initially wrote everything inside a single large `format!` call. The format string had many embedded `{}` placeholders, and the argument list was long. I thought 'this is concise; one call produces the whole report.'

But editing it was awful. Adding a new section required adding to the format string and to the argument list, in matching positions. Changing the order required moving things in two places. Reading the code required scanning back and forth to figure out which argument went with which placeholder.

After a few revisions, I switched to repeated `push_str(&format!(...))` calls, one per section. Each section is independent: I can edit its format string and its arguments together, without touching anything else. Adding a section is appending; removing is deleting.

The single `format!` was 'cleaner' by line count, but the repeated form was easier to maintain. I learned that conciseness alone is not always the right measure; structure that matches how the code will evolve is often more valuable."

---

*End of Lab 8 Solutions*
