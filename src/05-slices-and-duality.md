# 5. Slices & the Owned/Borrowed Duality

The previous three chapters built a single machine: [ownership](02-ownership-and-moves.md) gives every value exactly one owner, [borrowing](03-references-and-borrowing.md) hands out checked references, and [lifetimes](04-lifetimes.md) are the regions over which those references stay valid. So far every reference has pointed at *one whole value*. This chapter introduces the reference that points at a *contiguous run of values inside* a larger one: the **slice**, written `&[T]` and `&str`.

Slices are where the ownership model stops being an abstract discipline and starts paying you back. They are the reason idiomatic Rust signatures look the way they do, and they expose a pattern that recurs across the entire standard library — the **owned/borrowed pair**: `String`/`&str`, `Vec<T>`/`&[T]`, `PathBuf`/`&Path`, `OsString`/`&OsStr`. Learn the pattern once and a dozen APIs become predictable.

## The `(ptr, len)` idiom, but checked

If you have ever passed an array to a function in a lower-level language, you have passed a pointer and a length side by side — `process(data, len)` — and trusted the caller and callee to agree about them. That pair `(ptr, len)` *is* a slice: a view into someone else's array. The low-level version refuses to enforce anything about it: nothing stops `len` outliving the allocation `data` points into, nothing bounds-checks `data[i]`, nothing stops a caller passing `len` of 50 for a buffer of 10. The convention lives entirely in your head and in the comments.

A Rust slice is exactly that pair — a pointer and a length — promoted to a first-class type whose two invariants the compiler enforces:

```rust
fn process(data: &[i32]) { /* ... */ }
```

```rust
let a = [1, 2, 3, 4, 5];
let view = &a[1..3];          // type &[i32], points at a[1], length 2
assert_eq!(view, &[2, 3]);
```

The range `1..3` is half-open: start inclusive, end exclusive, length `3 - 1 = 2`. Indexing `view[i]` is bounds-checked against that length; the lifetime of `view` is tied by the borrow checker to `a`, so the view can never outlive its backing storage. The two failure modes you would otherwise debug by hand — an index past the end, and a view that survives its data — are now a panic and a compile error respectively, instead of silent corruption.

> **🦀 From your toolbox →** A slice is the "pass an array plus its length" pattern made safe. In Java you would pass the `int[]` itself (the array object carries its own `.length` and every access is bounds-checked) or, since Java 9, a `List` view; in Swift you would pass an `ArraySlice<Int>` or just the `Array`; in Python a list (and `a[1:3]` *copies* a sub-list). Rust's `&[i32]` gives you Java-and-Swift-style bounds safety, but unlike all three it is a genuine *non-owning window* into the original buffer — no copy, no separate object, no garbage collector keeping the backing array alive. Where the analogy breaks down: a Python or Swift slice keeps its source alive by reference counting; Rust instead ties the view's *lifetime* to the source at compile time, so the view simply cannot outlive what it points into. (If you have met C++'s `string_view` / `span`: same idea, but those carry no lifetime, so a dangling one compiles and is a classic use-after-free — Rust rejects that statically.)

> **⚙️ Under the hood →** `&[T]` and `&str` are **fat pointers**: two machine words, `(*const T, usize)` for the data pointer and the element count. On 64-bit, `size_of::<&[i32]>() == 16` and `size_of::<&str>() == 16`, versus `size_of::<&i32>() == 8` for an ordinary thin reference. The length lives in the pointer, not in the pointee — which is the whole trick: a slice of a heap `Vec`, a stack array, or a `&str` baked into the binary all have the identical runtime representation, because the metadata travels with the reference rather than the data. We will see in [smart pointers](15-smart-pointers.md) that this is the general mechanism for *dynamically sized types* (DSTs): `[T]` and `str` have no statically known size, so you can only ever hold them behind a fat pointer (`&[T]`, `Box<[T]>`, `&str`, …) that supplies the missing length.

## Why a slice and not an index

Consider the textbook task: return the first whitespace-delimited word of a string. Without slices, your return type is impoverished — you can only hand back a `usize` index into the original:

```rust
fn first_word_end(text: &str) -> usize {
    for (i, b) in text.bytes().enumerate() {
        if b == b' ' {
            return i;
        }
    }
    text.len()
}
```

That index is a *detached* fact. It is only meaningful paired with the exact string it was computed from, in the exact state that string was in. Nothing in the type system ties them together, so this compiles and is a latent bug:

```rust
let mut line = String::from("alpha beta");
let end = first_word_end(&line);   // 5
line.clear();                      // line is now ""
// `end` is still 5, now pointing past the end of an empty string
```

This is the same class of error as a dangling pointer, or as an index into a Python list after you have `del`-ed elements out from under it, or Java's `ConcurrentModificationException` — a stored position that the container has silently moved out from under you (Java catches its cousin at runtime, on a good day). A slice fixes it by *fusing the index range into a borrow*:

```rust
fn first_word(text: &str) -> &str {
    for (i, b) in text.bytes().enumerate() {
        if b == b' ' {
            return &text[..i];
        }
    }
    text                  // the whole thing is one word
}
```

Now the return value is a `&str` borrowed *from* `text`. By the lifetime-elision rules from [lifetimes](04-lifetimes.md), the output's lifetime is tied to the input's, so the borrow checker treats the returned word as an outstanding immutable borrow of the source string for as long as you hold it. The clear-after-borrow bug stops compiling:

```rust
let mut line = String::from("alpha beta");
let word = first_word(&line);   // immutable borrow of `line` begins
line.clear();                   // wants &mut line — ERROR
println!("{word}");             // borrow still live here
```

> **🔧 In practice →** This is exactly how you write a tokenizer or a parser without copying. Say you are reading a log line `"2026-06-24 ERROR disk full"` and want the level field. You return `&line[11..16]` — a `&str` pointing straight into the original buffer — and pass it around, compare it, match on it, all with zero allocation. The borrow checker guarantees you cannot accidentally free or overwrite `line` while any token still points into it, which is the bug that makes hand-rolled C parsers leak and crash. A real sketch:
> ```rust
> fn parse_level(line: &str) -> Option<&str> {
>     line.split_whitespace().nth(1)   // borrows from `line`, no copy
> }
> ```
> The returned `Option<&str>` is alive only as long as `line` is — try to drop `line` while you still hold the level and the compiler stops you.

> **⚠️ Pitfall →** The above is `error[E0502]: cannot borrow `line` as mutable because it is also borrowed as immutable`. `String::clear` needs `&mut self`, but `word` holds a live `&` into the same string, and the [borrowing](03-references-and-borrowing.md) rules forbid a mutable and an immutable borrow coexisting. The fix is not a workaround — it is the point. The compiler has proved that mutating the string while a view into it is alive would invalidate the view, exactly the bug the index version hid. Finish using `word` first, or compute an owned `String` if you genuinely need to outlive the source.

## The owned/borrowed pair

`String` and `&str` are not two unrelated string types; they are the **owning** and **borrowing** halves of one design. This split is the central idea of the chapter, so make it explicit:

| Owned (heap, `Drop`s its buffer) | Borrowed view (fat pointer, owns nothing) |
| --- | --- |
| `String` | `&str` |
| `Vec<T>` | `&[T]` |
| `Box<T>` | `&T` |
| `PathBuf` | `&Path` |
| `OsString` | `&OsStr` |
| `Box<[T]>` | `&[T]` |

The owned type is responsible for the allocation: it holds `(ptr, len, capacity)`, can grow, and frees the buffer in its destructor. The borrowed type is a `(ptr, len)` window that allocates nothing and frees nothing — it is purely a *view* with a lifetime.

> **⚙️ Under the hood →** `String` is a `Vec<u8>` with a UTF-8 invariant bolted on; both are `(ptr, len, capacity)`, three words, `size_of == 24` on 64-bit. `&str` is the same `(ptr, len)` fat pointer as `&[u8]`, two words, `size_of == 16`, but with the static guarantee that the bytes are valid UTF-8 — which is why you can slice into a `String` and get a `&str` for free: it is a pointer-plus-length into the `String`'s existing buffer, no copy, no allocation.

The payoff is **write your function signature against the borrowed type**:

```rust
fn count_vowels(s: &str) -> usize { /* ... */ }   // not &String
fn sum(xs: &[i32]) -> i32 { /* ... */ }           // not &Vec<i32>
```

A `&str` parameter accepts *both* an owned `String` (via the deref coercion below) and a borrowed `&str`, including string literals and sub-slices. A `&[i32]` accepts a whole `Vec`, a stack array, or any sub-slice of either. Taking `&String` or `&Vec<T>` is a beginner tell: it needlessly narrows your callers to one of the two halves and buys you nothing — `&String` gives you no capability over a string that `&str` does not.

> **🦀 From your toolbox →** In Java you reach for the widest interface that still does the job — you type a parameter as `CharSequence` or `List<T>` rather than `String` or `ArrayList<T>`, so any conforming implementation can be passed. In Swift you would write against a protocol like `StringProtocol` or `Collection` for the same reason. The owned/borrowed pair achieves that same "accept anything string-like / array-like" generality, but with a *single concrete type and no genericity at all*, because the owned type is built to hand out a borrow of exactly the view type. Where it differs: Java's `CharSequence` and Swift's `StringProtocol` are abstractions over *many implementations*; `&str` is one concrete type that every owner narrows down to. (The OCaml angle is weak here — OCaml `string` is immutable and `bytes` is mutable, but neither is a borrowed window: there is no substring-without-copy because there are no interior references for a lifetime to bound.)

## Deref coercion: why `&String` becomes `&str`

When you pass a `&String` where a `&str` is expected, the compiler silently inserts a conversion called a **deref coercion**: `String` implements `Deref<Target = str>`, so `&String` coerces to `&str` (concretely, `&s[..]`). The same machinery turns `&Vec<T>` into `&[T]`, `&Box<T>` into `&T`, and so on — every owned type in the table above implements `Deref` to its borrowed view.

```rust
let owned: String = String::from("rust");
let lit: &str = "rust";
count_vowels(&owned);   // &String --(deref coercion)--> &str
count_vowels(lit);      // already &str, no coercion needed
count_vowels(&owned[..]);  // explicit slice, identical result
```

This is also the resolution to the `s1 + &s2` puzzle for `String` concatenation. The `+` operator calls `add(self, &str)`, yet you pass `&s2` whose type is `&String` — it compiles because `&String` coerces to `&str`. Note `+` consumes its left operand (`self` by value, a move) and borrows the right, so `s1` is moved out and only `s2` survives. For anything beyond two strings, reach for `format!`, which borrows everything and moves nothing:

```rust
let s = format!("{a}-{b}-{c}");   // no operand is consumed
```

> **⚙️ Under the hood →** Deref coercion is resolved at compile time during method/argument type-checking — it is not a runtime conversion and emits no code beyond the trivial "narrow a fat/owning pointer to a fat view" (often a no-op or a single field load). We treat it properly in [smart pointers](15-smart-pointers.md), where `Deref` is also what makes `Box<T>`, `Rc<T>`, and your own wrappers feel like the thing they wrap. The mental model: `Deref` lets the compiler chase a chain of `&Owned -> &Borrowed` rewrites until the types line up.

## `String` is UTF-8: no O(1) `char` indexing

Here is where prior-language intuition actively misleads. A Rust `String` is a buffer of UTF-8 bytes. It is **not** an array of characters, and you cannot index it by character position:

```rust
let s = String::from("hi");
let c = s[0];   // error[E0277]: `String` cannot be indexed by `{integer}`
```

This is a deliberate refusal, not a missing feature. UTF-8 is variable-width: a Unicode scalar value occupies 1–4 bytes. Ze in Cyrillic, "З", is two bytes (`208, 151`); the Devanagari "नमस्ते" is 18 bytes for 6 scalar values. So `s[0]` has no good answer. A byte (`208` — meaningless alone)? A scalar value (would require a scan)? Rust declines to pick a wrong default. Three distinct views coexist, and you must say which you mean:

- **Bytes** — `s.bytes()` / `s.as_bytes()` yields `u8`. `s.len()` is the **byte** length.
- **Scalar values** — `s.chars()` yields `char` (a 32-bit Unicode scalar value, *not* a C `char`/byte). This is what most languages misleadingly call a "character".
- **Grapheme clusters** — what a human calls a letter (e.g. "नमस्ते" is 4 of these). Not in `std`; use the `unicode-segmentation` crate.

```rust
let s = "Зд";
assert_eq!(s.len(), 4);              // bytes
assert_eq!(s.chars().count(), 2);    // scalar values
assert_eq!(s.bytes().count(), 4);    // bytes again
```

There is a second reason beyond ambiguity: indexing is expected to be O(1), but locating the *n*-th scalar value in UTF-8 requires scanning from the start. Rust will not let an operator silently hide a linear scan behind `[]` syntax. If you genuinely want the *n*-th scalar value, you write the cost explicitly: `s.chars().nth(n)`.

> **🦀 From your toolbox →** Every language you know lies about strings, just differently. Python 3: `str` is a sequence of Unicode scalar values, so `s[0]` and `len(s)` count *code points* — convenient, but the underlying storage and the cost model are hidden from you, and you silently pay for the abstraction. Java (and JavaScript): `String` is a sequence of UTF-16 code units, so `.length()` and `charAt` count *code units* — an emoji or other astral-plane character is two units and `charAt` can hand you a lone surrogate (the same "you got half a character" bug, just relocated). Swift gets closest to honest: `String` is a collection of *grapheme clusters* (`Character`), so `count` matches human intuition — but it deliberately drops integer subscripting (you index with `String.Index`, not `Int`) for the very reason Rust does: there is no O(1) answer. Rust's choice — UTF-8 bytes, no integer indexing, explicit `.chars()`/`.bytes()` — forces you to name the granularity, which is why it feels obstructive coming from Python and correct after a week.

You *can* slice a string by **byte** range, and it yields a `&str` — but the range bounds must fall on UTF-8 character boundaries or the program **panics at runtime**:

```rust
let s = "Здравствуйте";
let head = &s[0..4];   // OK: "Зд", boundary at byte 4
let bad  = &s[0..1];   // panics: byte index 1 is inside 'З' (bytes 0..2)
```

> **⚠️ Pitfall →** `byte index 1 is not a char boundary; it is inside 'З' (bytes 0..2)`. Byte-range slicing is the one string operation that trades a compile error for a runtime panic, because the validity depends on data the compiler cannot see. Treat raw byte-range slicing of arbitrary (possibly non-ASCII) text as a code smell. Prefer boundary-aware iterators: `s.char_indices()` gives you `(byte_offset, char)` pairs whose offsets are guaranteed valid, or `split`/`find`/`splitn` which return sub-`&str`s on correct boundaries by construction.

## Slicing syntax and slice patterns

Range syntax is uniform across `&str` and `&[T]`, and the endpoints are optional:

```rust
let v = [10, 20, 30, 40, 50];
&v[1..3];   // [20, 30]   — 1 inclusive, 3 exclusive
&v[..3];    // [10, 20, 30] — from the start
&v[2..];    // [30, 40, 50] — to the end
&v[..];     // whole slice — borrow everything as &[i32]
&v[1..=3];  // [20, 30, 40] — inclusive end with ..=
```

Slices also participate in [pattern matching](07-enums-and-pattern-matching.md), which is where they get genuinely expressive. The `..` *rest pattern* binds nothing or, with `name @ ..`, binds the middle as a sub-slice:

```rust
fn describe(xs: &[i32]) -> String {
    match xs {
        [] => "empty".into(),
        [only] => format!("one element: {only}"),
        [first, .., last] => format!("from {first} to {last}"),
    }
}
```

```rust
let nums = [3, 1, 4, 1, 5];
if let [head, tail @ ..] = &nums[..] {
    // head: &i32 = &3,  tail: &[i32] = &[1, 4, 1, 5]
    println!("head {head}, {} more", tail.len());
}
```

> **🦀 From your toolbox →** This is OCaml's `match xs with | [] -> ... | x :: rest -> ...` from *Foundations of Computer Science* — and the same head/tail destructuring you would write in Swift with `if case let` on an enum, or fake in Python with `head, *tail = xs`. The crucial win over the OCaml/Python list version: a slice is a contiguous random-access window, so `[first, .., last]` (front *and* back at once) is O(1), which a cons list or a Python list-with-star-unpack cannot do without walking or copying. And unlike OCaml lists, a fixed-length pattern like `[a, b, c]` is *refutable* against `&[T]` (the length is a runtime fact), so the compiler forces you to handle the other lengths — there is no silent fall-through.

> **🔧 In practice →** Slice patterns shine when you are dispatching on a parsed command. You split a line into arguments and match on the shape directly, instead of indexing-and-hoping:
> ```rust
> fn run(args: &[&str]) -> Result<(), String> {
>     match args {
>         []                 => Err("no command".into()),
>         ["help"]           => { print_help(); Ok(()) }
>         ["add", item]      => { add(item); Ok(()) }
>         ["mv", from, to]   => { rename(from, to); Ok(()) }
>         ["add", rest @ ..] => Err(format!("add takes 1 arg, got {}", rest.len())),
>         [cmd, ..]          => Err(format!("unknown command: {cmd}")),
>     }
> }
> ```
> The arity of each command is checked by the pattern itself — wrong number of arguments falls through to a real error arm instead of panicking on an out-of-bounds index. This is how you would route subcommands in a small CLI tool.

> **⚙️ Under the hood →** `tail @ ..` does not copy. It produces a new fat pointer `(ptr + 1, len - 1)` aliasing the same backing buffer — pure pointer arithmetic, no allocation. The match on `[]` / `[only]` / `[first, .., last]` compiles to a length comparison plus offset loads, the same code you would hand-write from a `(ptr, len)` pair, but with the bounds proven correct by the pattern's exhaustiveness rather than by you.

## String literals are `&'static str`

The type of `"hello"` is `&'static str`: a slice pointing into a read-only region of the compiled binary, with the `'static` lifetime because that data lives for the whole program. That single fact explains several things at once: literals are immutable (a `&str` is an immutable borrow), they need no allocation, and they can be returned from any function without a lifetime worry — they outlive everything. It also closes the loop on `first_word(s: &str)`: a literal *is already* a `&str`, so it passes with no coercion at all, while a `String` passes via deref coercion. One signature, every caller.

## Mental-model recap

- A slice is the `(ptr, len)` pair as a checked, lifetime-bearing **fat pointer** (two words). `&[T]` and `&str` own nothing; they are *views* whose lifetime the borrow checker ties to the backing storage, killing dangling-view and index-out-of-sync bugs at compile time.
- The **owned/borrowed pair** is a standard-library design law: `String`/`&str`, `Vec<T>`/`&[T]`, `PathBuf`/`&Path`. Write parameters against the *borrowed* half (`&str`, `&[T]`) — it accepts both callers and grants every capability `&Owned` would.
- **Deref coercion** (`String: Deref<Target=str>`) is what auto-narrows `&String -> &str` and `&Vec<T> -> &[T]` at compile time; it is also why `s1 + &s2` and `&v[..]` Just Work.
- A `String` is UTF-8 **bytes**, not a `char` array: no integer indexing (it would be ambiguous *and* not O(1)). Choose your granularity explicitly with `.bytes()`, `.chars()`, or a grapheme crate. `s.len()` counts bytes.
- Byte-range slicing of a `&str` is the one operation that can **panic at runtime** (non-char-boundary). Prefer `char_indices`, `split`, `find` over raw `&s[i..j]` on non-ASCII text.

## Exercises

1. **Predict the errors.** Here are three call sites against `fn first_word(text: &str) -> &str`. Decide which compile and which do not, and for each rejection name the borrow that conflicts and roughly where it ends:
   ```rust
   // (a)
   let s = String::from("alpha beta");
   let w = first_word(&s);
   println!("{w}");
   drop(s);

   // (b)
   let s = String::from("alpha beta");
   let w = first_word(&s);
   let n = s.len();          // another shared borrow
   println!("{w} {n}");

   // (c)
   let s = String::from("alpha beta");
   let w = first_word(&s);
   s.push_str(" gamma");     // wants &mut s
   println!("{w}");
   ```
   Why does (b) compile even though `w` is still live, while (c) does not?

2. **Choose the signature.** You are writing `fn count_vowels(?) -> usize`. A teammate proposes `&String`; you propose `&str`. Give one concrete caller that compiles against `&str` but *not* against `&String`, and explain why the reverse direction (a `&str` caller against a `&String` parameter) never even comes up. Then state the one situation in which taking `String` *by value* would actually be the right choice.

3. **Adapt the parser.** Starting from `parse_level` in the "In practice" callout (returns `Option<&str>`), write `fn fields(line: &str) -> Option<(&str, &str, &str)>` that returns the date, level, and the *rest of the message* of a log line like `"2026-06-24 ERROR disk full"`, all as sub-slices of `line` with no allocation. Then explain: what is the lifetime relationship between the three returned `&str`s and `line`, and why can none of them outlive it?

4. **Slice patterns vs. owning.** Implement `fn split_last<T>(xs: &[T]) -> Option<(&[T], &T)>` with a slice pattern (`None` for empty). Give the `ptr`/`len` of the returned `&[T]` relative to `xs`, and confirm it allocates nothing. Now suppose you instead needed to return the elements *reversed* — argue why that return type cannot stay `&[T]` and must become `Vec<T>`.

5. **The non-ASCII trap.** This function looks reasonable but can panic:
   ```rust
   fn first_three(s: &str) -> &str { &s[0..3] }
   ```
   Give one input on which it returns a correct `&str` and one (non-ASCII) input on which it panics, quoting the kind of message you would get. Then rewrite it to return the first three *scalar values* safely (hint: `char_indices` or `chars`), and say what your new version returns for a string with only two scalar values.

6. (★) **Why a borrow won't do.** Consider `fn dedup_view(xs: &[i32]) -> ???` that collapses consecutive duplicates (`[1,1,2,3,3,3]` → `[1,2,3]`). Explain precisely why the return type cannot be `&[i32]`: what would such a slice have to be a view *into*, and which ownership rule forbids returning it? Contrast with exercise 4's `split_last`, where returning a borrow *is* fine — what is structurally different about the two outputs? Then give the return type that does work, and say in one line what it costs that `split_last` did not.
