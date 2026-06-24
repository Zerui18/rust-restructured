# 10. Common Collections

Arrays and tuples (from [the language delta](01-language-delta.md)) are fixed-size and live on the stack: their layout is known at compile time. The collections in `std::collections` are the opposite — heap-backed, growable, and generic over their element types. You already know what a dynamic array, a hash table, and a balanced tree *are* (Algorithms). What is new in Rust is not the data structures; it is that every one of them is threaded through the [ownership](02-ownership-and-moves.md) and [borrowing](03-references-and-borrowing.md) rules. A `Vec<T>` owns its elements. Indexing returns a *borrow*, not a copy. Reallocation can invalidate outstanding references — and the borrow checker statically forbids the use-after-free that would result. This chapter is the std collections at working speed, with the ownership consequences foregrounded.

## `Vec<T>`: the growable array

A `Vec<T>` is exactly the data structure behind Java's `ArrayList<T>`, Swift's `Array`, or Python's `list`: a heap allocation plus bookkeeping. Concretely, a `Vec<T>` value is three words on the stack — a pointer to the heap buffer, a `len`, and a `cap` (capacity):

```text
stack:  Vec { ptr ──┐  len: 3  cap: 4 }
                     ↓
heap:            [ e0 | e1 | e2 | ___ ]
```

The elements live contiguously on the heap; `len` is how many slots are initialised; `cap` is how many are allocated. When you `push` into a full buffer, `Vec` allocates a new buffer (the std implementation grows geometrically, typically doubling), bitwise-copies the old elements over, and frees the old buffer. This is the standard amortised-O(1) push: any single push may be O(n), but a sequence of n pushes is O(n) total.

```rust
let mut v: Vec<i32> = Vec::new();   // empty: ptr dangling, len=0, cap=0
v.push(5);
v.push(6);

let w = vec![1, 2, 3];              // vec! macro: type inferred as Vec<i32>
```

`Vec::new()` does not allocate; a fresh empty vector has capacity 0 and a dangling (non-null, well-aligned) pointer. The first `push` triggers the first allocation.

> **🦀 From your toolbox →** This is Java's `ArrayList`, Swift's `Array`, Python's `list` — same heap buffer, same geometric growth, same amortised push. Two differences worth holding onto. (1) In Java and Python a variable is a *reference* to the list; assigning `w = v` makes a second handle to the *same* list. In Rust `let w = v;` *moves* the three-word header and leaves `v` unusable — there is exactly one owner (see [moves](02-ownership-and-moves.md)). Swift sits in between: its `Array` is a value type with copy-on-write, so `let w = v` looks like a copy but shares the buffer until one side mutates; Rust makes the single-owner story explicit instead of hiding it behind reference counting. (2) On growth the elements are *moved* by a plain `memcpy` — Rust has no copy/move hooks that run user code on a relocation, so a moved-from slot is simply abandoned. *Where it breaks down:* don't picture Java's resizing as identical — Java copies object *references* on growth, while Rust copies the elements themselves (for `Vec<String>`, the three-word `String` headers), since the elements live inline in the buffer.

### Indexing vs `get`: panic vs `Option`

There are two ways to read an element, and the choice is about *how out-of-bounds is handled*, not ergonomics:

```rust
let v = vec![10, 20, 30];

let a: &i32 = &v[1];           // indexing: borrows; panics if out of bounds
let b: Option<&i32> = v.get(5); // get: returns None if out of bounds
```

`v[i]` desugars through the `Index` trait and panics on an out-of-range index — use it when an out-of-bounds access is a *bug* and crashing is the correct response. `v.get(i)` returns `Option<&T>` ([enums](07-enums-and-pattern-matching.md)) — use it when the index might legitimately miss (e.g. derived from user input). There is no third option that returns garbage: Rust will not hand you an unchecked element. For mutation, `get_mut` returns `Option<&mut T>`:

```rust
let mut v = vec![10, 20, 30];
if let Some(x) = v.get_mut(1) {
    *x += 5;                   // dereference the &mut, then write
}
```

> **🔧 In practice →** you're paginating: a request comes in for "page 7" of a result set, and you compute `let start = page * page_size;` then want `results.get(start)`. If you wrote `&results[start]` and the user asked for a page past the end, your service panics and the request 500s. With `results.get(start)` you get `None`, which you map to an empty page or a 404 — the out-of-bounds case becomes ordinary control flow:
> ```rust
> match results.get(start) {
>     Some(_) => render_page(&results[start..end]),
>     None    => render_empty_page(),   // no data this far in
> }
> ```
> Rule of thumb: index with `[]` when an out-of-range value means *your own code has a bug*; reach for `get` the moment the index is derived from anything outside your control.

### Reading while mutating: the borrow rule that surprises everyone

This is the canonical `Vec` borrow-check failure, and it is worth dwelling on because it is *correct*, not pedantic:

```rust
let mut v = vec![1, 2, 3, 4, 5];
let first = &v[0];   // immutable borrow of v begins
v.push(6);           // wants &mut v — ERROR
println!("{first}");
```

```text
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
```

The reason is the layout above. `first` points *into the heap buffer*. `push(6)` might reallocate, moving the buffer and leaving `first` dangling. If you have ever held a reference to an element of a Python `list` (or a Java collection) and then mutated the collection through another path, you have met this hazard's cousins — `ConcurrentModificationException` in Java is the runtime version of the same problem. Rust's rule ("one `&mut` xor any number of `&`") makes the hazard a *compile* error instead of a runtime crash: holding `&v[0]` is an outstanding `&` borrow of `v`, so the `&mut v` that `push` needs is rejected. The fix is to dereference/copy before mutating, or to scope the borrow so it ends before the `push`.

The same protection covers iteration. A `for` loop over `&v` holds a borrow of the whole vector for the loop's duration, so you cannot `push` or `remove` mid-iteration:

```rust
let mut v = vec![1, 2, 3];
for x in &v {        // borrows v
    v.push(*x);      // ERROR: cannot mutate v while iterating
}
```

> **⚙️ Under the hood →** Reallocation invalidating references is *iterator invalidation* enforced at compile time. Note Rust does not track *which* index `first` points at — the borrow is of the whole `Vec`. So even `let first = &v[0]; v.push(6);` where logically `first` might survive is rejected: the borrow checker reasons at the granularity of the whole value, not individual slots. This is conservative but sound.

### Three ways to iterate, three ownership stories

The single most important `Vec` distinction for ownership is the trio `iter` / `iter_mut` / `into_iter`. They differ only in what they yield and what happens to the vector ([more on iterators in 12](12-closures-and-iterators.md)):

```rust
let v = vec![String::from("a"), String::from("b")];

for s in v.iter()      { /* s: &String     — v still usable after */ }
// v alive here

let mut w = vec![1, 2, 3];
for n in w.iter_mut()  { *n *= 10; }   // n: &mut i32 — mutate in place
// w == [10, 20, 30]

let owned = vec![String::from("x"), String::from("y")];
for s in owned.into_iter() { /* s: String — owned, moved out of the vec */ }
// owned is CONSUMED — using it here is a compile error
```

`&v` in a `for` loop is sugar for `v.iter()`; `&mut v` for `v.iter_mut()`; a bare `v` for `v.into_iter()`. The third *consumes* the vector and hands you each element *by value* — this is how you move owned elements out without cloning. For `Copy` element types (`i32`) the distinction is mostly academic; for owning types (`String`, `Vec<_>`) it is the difference between borrowing the contents and dismantling the container.

> **🦀 From your toolbox →** `iter()`/`iter_mut()` are the read-only vs in-place-mutating loops you already write in Swift (`for x in arr` vs mutating through indices) or Java (`for (T x : list)` with the `Iterator.remove`/`set` story). `into_iter()` is the one with no everyday equivalent in those languages: it hands you each element *by value and empties the container as it goes*, so afterward the vector is gone and there is no risk of touching a half-moved collection. In Python you might fake "drain everything out" with a `while lst: x = lst.pop()`, but Python keeps the list object alive and never gives you a moved-out value. *Where it breaks down:* in Java/Python the loop variable is always a reference into a still-living collection; only `into_iter` actually transfers ownership and leaves nothing behind.

### Heterogeneity via enums

`Vec<T>` is homogeneous — every element is the same type, because the layout demands a fixed stride. To store a "row of mixed cells", wrap the alternatives in an enum so the *static* type is uniform:

```rust
enum Cell {
    Int(i64),
    Real(f64),
    Label(String),
}
let row = vec![Cell::Int(7), Cell::Label("hi".into()), Cell::Real(2.5)];
```

This is the same trick as a Swift `enum` with associated values (`enum Cell { case int(Int64); case real(Double); case label(String) }`) collected into an `[Cell]`: the array stays homogeneous because every element is a `Cell`, and the variant carries the real payload. Each slot is `size_of::<Cell>()` wide (the tag plus the largest variant). When the variant set is open-ended and unknown at compile time, you instead reach for trait objects, `Vec<Box<dyn Trait>>` — see [trait objects](09-trait-objects-and-oop.md).

### Capacity tuning

If you know the size ahead of time, `Vec::with_capacity(n)` allocates once and eliminates the intermediate reallocations of repeated `push`. This is a pure performance optimisation — it changes nothing about semantics:

```rust
let mut buf = Vec::with_capacity(1024);   // one allocation, cap >= 1024
for i in 0..1024 { buf.push(i); }         // no further reallocation
```

Also worth knowing: `swap_remove(i)` removes element `i` in O(1) by swapping it with the last element (order not preserved), whereas `remove(i)` is O(n) because it shifts the tail down.

## `String`: a `Vec<u8>` with a UTF-8 invariant

A `String` *is* a `Vec<u8>` — same three-word header, same growth strategy — with one added invariant: the bytes are always valid UTF-8. That invariant is exactly why you cannot index a `String` by integer; the full treatment of UTF-8, `char`, byte/scalar/grapheme views, and `&str` slicing lives in [slices and the owned/borrowed duality](05-slices-and-duality.md). Here we focus only on *building and mutating* strings.

```rust
let mut s = String::new();
s.push_str("foo");        // append a &str
s.push('!');              // append a single char
let owned = "abc".to_string();    // &str -> String
let also  = String::from("abc");  // identical effect
```

`push_str` takes `&str`, not `String`, so it borrows rather than consumes its argument — you keep ownership of what you append. Concatenation with `+` is the one quietly surprising operation:

```rust
let a = String::from("Hello, ");
let b = String::from("world!");
let c = a + &b;   // a is MOVED; b is borrowed; c is the result
```

The `+` operator calls `add(self, &str) -> String`. `self` is taken *by value*, so the left operand is moved into the result and reused as its buffer — no copy of the left side. The right operand is `&str`, so `&b` (a `&String`) is deref-coerced to `&str` ([coercion in 15](15-smart-pointers.md)) and only *its* bytes are copied. For anything beyond two operands, reach for `format!`, which borrows all arguments and is far more readable:

```rust
let (x, y, z) = ("tic", "tac", "toe");
let s = format!("{x}-{y}-{z}");   // borrows everything, allocates once
```

> **⚠️ Pitfall →** Writing `let c = a + &b;` and then using `a` gives `error[E0382]: borrow of moved value: a`. The `+` operator consumed `a`. If you need `a` afterward, either clone it (`a.clone() + &b`) or use `format!("{a}{b}")`, which moves nothing. New Rustaceans expect `+` to behave like `String` concatenation in Java/Python (which produce a fresh string and leave both operands intact); Rust trades that for a guaranteed buffer reuse on the left.

## `HashMap<K, V>`: keys to values

`HashMap<K, V>` is a hash table, heap-allocated like `Vec` — Java's `HashMap`, Swift's `Dictionary`, Python's `dict`. It is not in the prelude, so you must bring it in:

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
```

### Ownership on insert

`insert(k, v)` *takes ownership* of both key and value. For `Copy` types the value is copied in; for owning types like `String` it is moved, and the map becomes the owner:

```rust
let name = String::from("Blue");
let val  = String::from("ten");
let mut m = HashMap::new();
m.insert(name, val);
// name and val are MOVED — using either is a compile error now
```

This is where the Java/Python intuition needs adjusting. In Java `map.put(name, val)` stores *references*; `name` still points at the same `String` object afterward, and so does the map. In Rust there is one owner, and `insert` transferred it to the map — the local bindings are now used up. Reading is by reference. `get` returns `Option<&V>` (the key might be absent), and you commonly `.copied()` to turn `Option<&i32>` into `Option<i32>` for a `Copy` value, then `.unwrap_or(default)`:

```rust
let blue = scores.get("Blue").copied().unwrap_or(0);
```

Note you can call `get("Blue")` with a `&str` even though the keys are `String` — `HashMap` keys can be looked up by any type the key *borrows as* (`String: Borrow<str>`), so no temporary `String` is allocated for the lookup. Iteration yields `(&K, &V)` pairs **in arbitrary, run-to-run-varying order** — never depend on it.

> **⚠️ Pitfall →** Iterating a `HashMap` and expecting insertion or sorted order is a portability/reproducibility bug. The order is randomised per execution (a consequence of the DoS-resistant hashing below). If you need order, use a `BTreeMap` (sorted by key) or collect into a `Vec` and sort.

### The `entry` API: why it beats `get`-then-`insert`

The natural "insert if absent, otherwise update" pattern is hostile to the borrow checker if you write it the obvious way: a `get` borrows the map, and the follow-up `insert` needs a fresh mutable borrow, and threading the two together is awkward. The `entry` API does the lookup *once* and hands back a handle to the slot:

```rust
let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);

scores.entry(String::from("Yellow")).or_insert(50); // absent -> inserts 50
scores.entry(String::from("Blue")).or_insert(50);   // present -> leaves 10
```

`entry(k)` returns an `Entry` enum — `Occupied` or `Vacant` — and `or_insert(d)` returns a `&mut V` to the existing value (if present) or to the freshly inserted default. Because it returns `&mut V`, you can read-modify-write through it. The classic word-count is one line of real work:

```rust
let text = "the cat the dog the bird";
let mut counts: HashMap<&str, i32> = HashMap::new();
for word in text.split_whitespace() {
    *counts.entry(word).or_insert(0) += 1;
}
// counts == {"the": 3, "cat": 1, "dog": 1, "bird": 1}
```

`or_insert(0)` returns `&mut i32`; the leading `*` dereferences it to apply `+= 1`. The borrow lasts only for that statement, so the next loop iteration is free to mutate the map again. `or_default()` does the same with `V::default()`, and `or_insert_with(f)` defers constructing the default until it is actually needed (use it when the default is expensive):

```rust
let mut groups: HashMap<i32, Vec<i32>> = HashMap::new();
for n in 0..6 {
    groups.entry(n % 2).or_default().push(n);  // bucket by parity
}
```

> **🔧 In practice →** you're aggregating server logs and want, for each user ID, the list of endpoints they hit. In Python you would reach for `collections.defaultdict(list)` and write `groups[user].append(endpoint)`; `entry(...).or_default()` is exactly that, with the borrow checker keeping the lookup-and-mutate to a single map probe:
> ```rust
> let mut by_user: HashMap<UserId, Vec<&str>> = HashMap::new();
> for entry in log_lines {
>     by_user.entry(entry.user).or_default().push(entry.endpoint);
> }
> ```
> Reach for `entry` whenever the shape is "look it up; if it's there update it, if not seed a default" — counters, multimaps, memoisation caches, dedup-with-tally. The naive `if map.contains_key(k) { ... } else { ... }` does two lookups and fights the borrow checker; `entry` does one of each.

> **🎓 Tripos link →** *Concurrent and Distributed Systems.* The reason `entry` "plays nicely with the borrow checker" is the same reason you prefer a single atomic update over a separate read-then-write: it collapses two steps into one. `get`-then-`insert` reads the map, then comes back later to write it — two separate touches the borrow checker has to reconcile across a gap, just as a lock-read-then-lock-write leaves a window where another thread could interleave. `entry` holds one mutable handle for the whole read-modify-write, which is both easier to prove sound and a single hash probe instead of two.

### `Eq`, `Hash`, and the hashing function

A type may be a `HashMap` key only if it implements both `Hash` and `Eq` ([traits](08-generics-and-traits.md)), and the two must agree: `a == b` must imply `hash(a) == hash(b)`. For your own structs, `#[derive(Hash, Eq, PartialEq)]` gives the field-wise implementations, which is almost always what you want. Floating-point types are *not* `Eq` (because of `NaN`), so `f64` cannot be a key without a wrapper.

By default `HashMap` uses **SipHash 1-3**, a keyed hash seeded with per-process randomness. This is a deliberate choice: it is not the fastest hash, but it resists *hash-flooding* DoS attacks, where an adversary who can predict your hash function feeds you keys that all collide into one bucket, degrading the table to O(n) per operation. The randomised seed is also why iteration order varies between runs. If you have profiled and the hash is your bottleneck — and your keys are not attacker-controlled — you can swap the hasher by parameterising the map (`HashMap<K, V, S>` where `S: BuildHasher`); crates like `fxhash` or `ahash` provide faster, non-DoS-resistant hashers as drop-in replacements.

> **⚙️ Under the hood →** Today's `std` `HashMap` is a port of Google's SwissTable (the `hashbrown` crate): open-addressing with SIMD-accelerated probing over a control byte array, not the separate-chaining buckets you may picture from Algorithms. The `BuildHasher` trait exists precisely so the table can construct a *fresh, independently-seeded* `Hasher` per map instance — that per-instance seed is what randomises iteration order and frustrates collision attacks.

### Constructing and collecting

You can build a map from an array of pairs, or `collect` one from any iterator of `(K, V)` ([iterators in 12](12-closures-and-iterators.md)) — the latter is the idiomatic functional construction:

```rust
let m = HashMap::from([("a", 1), ("b", 2)]);

let squares: HashMap<i32, i32> = (1..=5).map(|n| (n, n * n)).collect();
```

`with_capacity(n)` works here too, pre-sizing the table to avoid rehashing during a known bulk insert.

## The rest of `std::collections`, and when to reach for each

`Vec`, `String`, and `HashMap` cover the large majority of programs. The other four exist for specific access patterns. They share the same ownership discipline — owned elements, borrows on read, mutation gated by `&mut`.

| Type | Backing structure | Ordered? | Use when |
| --- | --- | --- | --- |
| `Vec<T>` | growable array | insertion | default sequence; index/iterate; push/pop at the **end** |
| `VecDeque<T>` | ring buffer | insertion | a queue or deque: O(1) push/pop at **both** ends |
| `HashMap<K,V>` | SwissTable | no (random) | key→value lookup, fastest average case |
| `BTreeMap<K,V>` | B-tree | **sorted by key** | need keys in order, range queries (`.range(a..b)`), `.first_key_value()` |
| `HashSet<T>` | `HashMap<T,()>` | no | membership / dedup, fast average case |
| `BTreeSet<T>` | B-tree | **sorted** | membership *and* ordered iteration / ranges |
| `BinaryHeap<T>` | binary max-heap | partial (peek = max) | priority queue: O(1) `peek` of the max, O(log n) `push`/`pop` |

Two points worth pinning down. First, the `Hash`-vs-`Btree` axis is the classic hash-table-vs-balanced-tree tradeoff from Algorithms: `HashMap` gives O(1) average lookup but no order and worst-case O(n); `BTreeMap` gives O(log n) lookup *and* sorted traversal plus range queries, which the hash map fundamentally cannot offer. (If you have used Java's `HashMap` vs `TreeMap`, this is precisely that pair.) Choose `BTreeMap`/`BTreeSet` whenever ordering or ranges matter, `HashMap`/`HashSet` otherwise. Second, `BinaryHeap` is a **max-heap** — `pop` returns the *largest* element. For a min-heap, store `std::cmp::Reverse<T>`:

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

let mut max = BinaryHeap::from([3, 1, 4, 1, 5]);
assert_eq!(max.pop(), Some(5));          // largest first

let mut min = BinaryHeap::new();
min.push(Reverse(3));
min.push(Reverse(1));
assert_eq!(min.pop(), Some(Reverse(1))); // Reverse flips the ordering
```

Every owning collection here frees its buffer *and* recursively drops every element when it goes out of scope — dropping a `Vec<String>` drops each `String`, which frees each string's heap buffer. This is the cleanup story you may know from Swift's ARC or C++'s destructors-at-scope-end, but applied *transitively* and decided at compile time: unlike Java's or Python's garbage collector, there is no background reclaimer and no nondeterministic timing — the frees happen exactly when the owner's scope ends, with the borrow checker guaranteeing no reference outlives the collection that owns the data.

## Mental-model recap

- A `Vec<T>`/`String` is `(ptr, len, cap)` on the stack over a heap buffer; growth is amortised-O(1) but *reallocates*, which is why holding `&v[i]` blocks `v.push(...)` — that rule prevents a use-after-free, statically.
- Indexing (`v[i]`, `s + ...`, panicking) vs the checked form (`v.get(i)`, `.entry`, returning `Option`/handles) is a deliberate choice about how absence is handled — Rust never returns garbage.
- `iter` borrows, `iter_mut` mutably borrows, `into_iter` *consumes and moves out* — for owning element types this is the difference between keeping and dismantling the container.
- `HashMap::insert` takes ownership of key and value; the `entry` API collapses lookup-then-update into a single `&mut` borrow and one hash probe, which is why it beats `get`-then-`insert`.
- `HashMap` order is randomised (SipHash + per-instance seed, for DoS resistance); reach for `BTreeMap`/`BTreeSet` whenever you need ordering or ranges, and remember `BinaryHeap` is a *max*-heap.

## Exercises

1. You're writing a function that returns the *last* element of a slice. Sketch the signature `fn last<T>(xs: &[T]) -> Option<&T>` and decide: should the body use `xs[xs.len() - 1]` or `xs.last()`? Call your reasoning out explicitly — what happens for each choice when `xs` is empty, and which one makes the empty case impossible to forget at the call site?

2. Predict whether each of these compiles, and for the failures name the rule being enforced:
   ```rust
   // (a)
   let mut v = vec![1, 2, 3];
   let x = v[0];          // note: no &
   v.push(4);
   println!("{x}");

   // (b)
   let mut v = vec![String::from("a"), String::from("b")];
   let r = &v[0];
   v.push(String::from("c"));
   println!("{r}");
   ```
   Why does swapping `&v[0]` for `v[0]` change the answer? Tie it back to the `(ptr, len, cap)` layout.

3. You have `let words: Vec<String> = ...;` and want a single `String` joining them with `", "` between. Decide between `words.iter()` and `words.into_iter()` for the job, and say what each choice costs: which one forces a clone of every element, which one leaves `words` usable afterward, and which property actually matters for *this* task.

4. A teammate stores request timestamps as `HashMap<u64, LogLine>` and now needs "all log lines from the last 5 minutes" as a sorted range. Without writing the query, decide what single change to the *type* unlocks this, explain why the `HashMap` fundamentally cannot answer the range question, and state the rough cost of the range scan in terms of the tree's height and the number of results returned.

5. (★) You're tallying votes where each ballot is an `f64` score, and you reach for `HashMap<f64, u32>` — it won't compile. Name the missing trait bound and the one IEEE value that is to blame. Then design a key type that lets you bucket by score *while preserving the numeric value* (hint: scores arrive rounded to two decimals — what integer could stand in for the key?), and say one thing your wrapper gives up compared to keying on the raw `f64`.
