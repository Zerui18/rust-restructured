# 16. Unsafe Rust, FFI & the Memory Model

Everything you have read so far has been *safe Rust*: a language whose type checker proves, statically, the absence of use-after-free, double-free, data races, and out-of-bounds access. That proof is the whole product. But the proof rests on a checker that is — and must be — *conservative*. A safe checker rejects every unsafe program and, as a price, also rejects some perfectly correct programs it simply cannot reason about. `Vec`'s growth, `split_at_mut`'s non-overlapping borrows, a hand-rolled doubly-linked list, calling `malloc`, talking to an MMIO register at a fixed address — all correct, all unprovable by the borrow checker.

`unsafe` is the escape hatch for exactly these cases, and it is far narrower than its name suggests. The single most important sentence in this chapter:

> **`unsafe` does not turn off the borrow checker.** It unlocks exactly five operations the checker cannot verify, and leaves *every other* check fully on.

The right mental model is one of obligations. Safe Rust is a system in which the compiler *proves*, on your behalf, that the program can never cause memory corruption. `unsafe` is you writing, in effect, `// trust me: I've checked this by hand`. You take on a guarantee the checker handed back because it could not establish it. Get the reasoning right and your code is exactly as safe as any safe code — `Vec` and `Box` are nothing but unsafe code wearing a correct hand-checked guarantee. Get it wrong and you have a bug of exactly the C/C++ flavour you came to Rust to escape.

## The five superpowers (and nothing else)

Inside an `unsafe` block (or an `unsafe fn` body, via an inner `unsafe` block) you gain the ability to:

1. **Dereference a raw pointer** (`*const T`, `*mut T`).
2. **Call an `unsafe fn`** (including any function declared in an `extern` block).
3. **Implement an `unsafe trait`** (e.g. `Send`, `Sync`).
4. **Access or modify a mutable `static`**.
5. **Read a field of a `union`**.

That is the complete list. There is no sixth. Crucially, the following are *still checked* inside `unsafe`:

```rust
unsafe {
    let x = 5;
    let r = &mut x;   // error[E0596]: cannot borrow `x` as mutable — still checked
}
```

Ownership, borrowing, lifetimes, `Sized`, exhaustiveness, type checking — all unaffected. If you write `&mut` to immutable data inside `unsafe`, you still get the same error you would outside it. `unsafe` is not a mode; it is a five-item capability grant.

> **🦀 From your toolbox →** Think of how Swift fences off `UnsafeMutablePointer` and friends, or how Python lets you reach into `ctypes` — a syntactically marked island where one specific guarantee becomes your job. This is unlike C, where there is *no* boundary at all: every pointer deref is "unsafe" and the language never tells you which line is the dangerous one. The light C++ touch — `reinterpret_cast` — breaks down as an analogy because C++ never hands you a list of exactly which operations just became dangerous; in Rust the grep-able `unsafe` keyword *is* the audit trail. The honest gap: Swift's `Unsafe...Pointer` and Python's `ctypes` are mostly about reaching out to C, whereas Rust's `unsafe` is also how the standard library builds its *own* safe abstractions from the inside.

> **🔧 In practice →** You're writing an image filter that walks a `&mut [u8]` pixel buffer and, for blur, needs to read the row above while writing the current row. The borrow checker won't let you hold two `&mut` slices of the same buffer at once. You reach for `split_at_mut` (or its raw-pointer guts) to carve the buffer into provably non-overlapping rows, do the per-row work in safe code, and confine the entire `unsafe` to the one split. The payoff is concrete: no per-pixel bounds check in the hot loop, and the rest of the filter stays ordinary safe Rust.

## Undefined behaviour: the same beast as in C

What exactly are you promising not to do? You are promising not to cause **undefined behaviour (UB)**. This is the identical concept from your C and C++ course: a precondition the language assumes the program never violates, and on whose violation the compiler is licensed to do *anything* — including miscompile code far away from the violation, because the optimiser reasoned under an assumption you broke.

Rust's UB list (the [Reference's full list](https://doc.rust-lang.org/reference/behavior-considered-undefined.html) is authoritative) includes:

- **Dereferencing a dangling, null, or unaligned pointer** — same as C.
- **Reading uninitialised memory**, or producing an *invalid value* for a type: a `bool` that is not `0`/`1`, a `char` outside the Unicode scalar range, a `&T`/`&mut T` that is null or dangling, an enum with an out-of-range discriminant. Producing such a value is instant UB even if you never "use" it.
- **Out-of-bounds access** via raw pointer arithmetic (`.add`, `.offset`).
- **Data races** — concurrent unsynchronised access where at least one side writes.
- **Breaking the aliasing rules** — having a live `&mut T` while any other pointer reads or writes the same memory (see the memory model below). This is the one with no C equivalent and the one that bites hardest.

The asymmetry with C: in safe Rust *none* of these are reachable; inside `unsafe` all of them are. The optimiser is *more* aggressive than a typical C compiler because `&mut T` carries a no-aliasing guarantee (below), so UB here can produce stranger results than the equivalent C bug.

> **⚠️ Pitfall →** A common cargo-cult error is `let x: bool = unsafe { std::mem::transmute(2u8) };`. This compiles. It is instant UB — `2` is not a valid `bool` — and the optimiser may now assume `x` is simultaneously `true` and `false` in different branches. The fix is never to construct invalid values; use `2u8 != 0` for an explicit conversion. `transmute` is the single most dangerous function in the language and almost always the wrong tool.

## Raw pointers: C pointers, re-typed

Raw pointers are the heart of the first superpower. `*const T` and `*mut T` are *exactly* C's `const T*` and `T*`: a machine word holding an address, with no guarantees. Specifically they may be null, may dangle, may alias each other and references freely, are not bound by lifetimes, and run no destructor when dropped.

You can *create* them in completely safe code — only *dereferencing* requires `unsafe`:

```rust
let mut count = 7i32;
let p_const: *const i32 = &raw const count; // raw borrow operators (1.82+)
let p_mut:   *mut i32   = &raw mut count;

unsafe {
    println!("{}", *p_const);  // deref: superpower #1
    *p_mut = 9;
}
assert_eq!(count, 9);
```

The `&raw const` / `&raw mut` operators are the modern, correct way to take a raw pointer: they produce a pointer *without* first creating an intermediate reference, which matters because forming even a transient `&mut` to a place you are about to alias is itself UB under the aliasing model. (The old `&v as *const _` form silently creates that reference.) You can also conjure a pointer from an integer — `let p = 0xdead_beef_usize as *const i32;` — legal to write, UB to dereference, because nothing guarantees anything lives there.

The asterisk in `*const T` is part of the *type name*, not a dereference; `*p_const` is the deref. You can hold a `*const` and a `*mut` to one location and write through one while reading the other — the compiler will not stop you, and you will have hand-built a data race. That freedom is the point: it lets you express patterns the borrow checker rejects.

> **⚙️ Under the hood →** A `*const T`/`*mut T` is one pointer-sized word for `Sized` `T`, byte-identical to a C pointer — this is why FFI works with zero marshalling. A *fat* raw pointer `*const [T]` or `*const dyn Trait` is two words (pointer + length, or pointer + vtable), exactly like the `&[T]`/`&dyn` fat references from [slices](05-slices-and-duality.md) and [trait objects](09-trait-objects-and-oop.md). `&T as *const T` is a no-op at the machine level — references and raw pointers have identical representation; the only difference is the static guarantees the type carries.

## The safe/unsafe boundary: soundness as the contract

The professional use of `unsafe` is never to sprinkle it at call sites. It is to **encapsulate** a minimal unsafe core behind a *sound safe API*. The canonical example is `slice::split_at_mut`, which hands back two mutable sub-slices of one slice — provably non-overlapping, but the borrow checker only sees "two `&mut` from the same `values`" and refuses ([references](03-references-and-borrowing.md)):

```rust
use std::slice;

fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = values.len();
    let ptr = values.as_mut_ptr();
    assert!(mid <= len);                       // the precondition we check
    // SAFETY: `mid <= len`, so both ranges lie within the original
    // allocation and `[0, mid)` is disjoint from `[mid, len)`. The two
    // resulting &mut slices therefore never alias.
    unsafe {
        (
            slice::from_raw_parts_mut(ptr, mid),
            slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```

The function is *not* marked `unsafe`: its signature is callable from safe code, and **no input can make it cause UB**. That property is the definition of **soundness** for a safe abstraction. The `assert!` is load-bearing — it converts the precondition into a checked panic, so the only path into the `unsafe` block is one where the reasoning holds. `from_raw_parts_mut` and `.add` are themselves `unsafe fn`s whose contracts ("valid for `len` elements", "offset stays in-bounds") this function discharges.

The inverse — a safe-looking function that *can* be driven to UB by some safe caller — is **unsound**, and an unsound abstraction is a *bug*, full stop, even if it never misbehaves in your tests. `Vec`, `Box`, `Rc`, `Mutex`, `String` are all this pattern: a tiny audited `unsafe` core, wrapped so no safe caller can ever trigger UB. You have run on unsafe code the entire book; it was sound, so you never noticed.

> **🦀 From your toolbox →** This is the same discipline as a class with an invariant that every method preserves — a Java class with `private` fields and methods that keep them consistent, or a Swift `struct`/`class` whose stored properties are only mutated through methods that maintain the invariant. (The lightest C++ touch: a class that owns a raw pointer and frees it in its destructor.) The difference Rust adds is the *enforced boundary*: the `unsafe` keyword marks precisely where the invariant is hand-maintained, so an auditor greps for `unsafe` and reads only those sites. In Java or Swift the invariant-maintaining code and the merely-correct code look identical; Rust makes the hand-checked spots visible.

> **⚙️ Under the hood →** `unsafe fn` and a safe fn containing an `unsafe` block are *different contracts*. An `unsafe fn` pushes the proof obligation onto its **caller** (the caller's `unsafe` block is the promise). A safe fn that internally uses `unsafe` keeps the obligation **inside itself** and discharges it for all callers. `split_at_mut` is the latter; `from_raw_parts_mut` is the former. Choosing which to write is choosing where the proof obligation lives.

## Mutable statics and unsafe traits

A `static` is a global with a fixed address (unlike a `const`, which the compiler may inline and duplicate at each use). An immutable `static` is fine and safe to read. A `static mut` is global *mutable* state, and reading or writing it is superpower #4 — because nothing stops two threads touching it concurrently, i.e. a data race:

```rust
static mut TICKS: u64 = 0;

/// SAFETY: caller must guarantee no concurrent access from another thread.
unsafe fn tick() {
    unsafe { TICKS += 1; }
}
```

Modern Rust additionally *denies by default* taking a reference to a `static mut` (the `static_mut_refs` lint) — even the invisible `&` inside `println!("{TICKS}")` — because a `&mut` to a global is an aliasing landmine; you must go through a raw pointer, `*(&raw const TICKS)`. The idiom is to avoid `static mut` entirely and reach for the [thread-safe primitives](13-fearless-concurrency.md) (`AtomicU64`, `Mutex`, `OnceLock`) which encode the synchronisation in the type and stay in safe code.

Superpower #3, the **`unsafe trait`**, you already met. A trait is `unsafe` when implementing it asserts an invariant the compiler cannot check. The marquee examples are `Send`/`Sync` from [concurrency](13-fearless-concurrency.md): auto-derived structurally, but if you build a type around a raw pointer (`!Send`/`!Sync`) and *know* your synchronisation makes it thread-safe, you assert it with `unsafe impl Send for MyType {}` — promising the thread-safety the auto-derivation refused to assume. A wrong `unsafe impl Send` manufactures UB from entirely safe-looking downstream code.

## FFI: the C ABI boundary

The original reason `unsafe` exists is that the hardware and the rest of the software universe are written in C. **FFI** (Foreign Function Interface) is how Rust calls C and C calls Rust, through the `extern` keyword.

### Calling C from Rust

```rust
unsafe extern "C" {
    // libc's abs; the "C" selects the C ABI (calling convention).
    fn abs(input: i32) -> i32;

    // No memory-safety preconditions, so we can mark it `safe`:
    safe fn strlen(s: *const std::ffi::c_char) -> usize;
}

fn main() {
    let n = unsafe { abs(-42) }; // declared-unsafe by default
    println!("{n}");
}
```

The `"C"` string is the **ABI**: it tells rustc to use C's calling convention (argument registers, stack layout) at the assembly level. The whole `extern` block is `unsafe` because the foreign side does not obey Rust's rules and rustc cannot check it — so by default every declared function is `unsafe` to call. The 2024-edition refinement is the `safe` keyword: a function with genuinely no memory-safety preconditions may be written `safe fn`, and callers skip the `unsafe` block. `safe` does not *make* it safe; it is *your promise* that it is, and you own the consequences if wrong.

> **🔧 In practice →** You're wrapping a mature C library — say a system audio codec or a hardware vendor's SDK — that Rust will never get a native port of. You write a thin `unsafe extern "C"` block declaring the handful of functions you need, then build a small safe Rust module on top: it owns the C handle, runs the `unsafe` calls inside `// SAFETY:`-commented blocks, and exposes ordinary safe methods (`codec.decode(&bytes)?`) to the rest of your app. The whole crate is safe Rust except for that one audited file — and that file is what you point a reviewer at.

### Calling Rust from C

The other direction exports a Rust function under the C ABI with an un-mangled name:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}
```

`extern "C"` gives it the C calling convention. `#[unsafe(no_mangle)]` (the `unsafe(...)` wrapper is 2024-edition syntax) suppresses name mangling so the symbol is literally `rust_add` and C's linker can find it — mangling is the compiler rewriting names to encode types/modules so overloaded and generic functions get distinct symbols, a detail from Compiler Construction; C cannot parse the mangled form. The `unsafe(...)` wrapper is there because un-mangled symbols can *collide* across libraries, and avoiding collisions is now your responsibility. Compile as a `cdylib`/`staticlib`, declare `int rust_add(int, int);` on the C side, link, done.

### Layout, structs, and strings across the boundary

C only understands C layout. Rust's default `struct` layout is **unspecified** — the compiler may reorder fields to minimise padding — so a struct shared with C must be annotated `#[repr(C)]` to force C's declaration-order, C-padding layout:

```rust
#[repr(C)]
pub struct Point { pub x: f64, pub y: f64 } // matches `struct Point { double x, y; }`
```

Strings are the classic trap: Rust's `&str`/`String` are UTF-8 with a length and *no terminating NUL*; C strings are NUL-terminated `char*` with no length. They are not interchangeable. Cross the boundary with `std::ffi::CString` (owns a NUL-terminated buffer, to send *to* C) and `CStr` (borrows one, to receive *from* C):

```rust
use std::ffi::{CString, c_char};

unsafe extern "C" { safe fn puts(s: *const c_char) -> i32; }

let owned = CString::new("hello from rust").unwrap(); // appends the NUL
puts(owned.as_ptr());                                 // &owned keeps it alive
// `owned` must outlive the call; if it drops, `as_ptr()` dangles — UB.
```

**Ownership does not cross FFI automatically.** The `Drop`-based RAII ownership model of [chapter 2](02-ownership-and-moves.md) is a Rust-only fiction; C has no idea your `Box` should be freed by Rust's allocator. The boundary rules, all *your* responsibility to uphold:

- Memory allocated by Rust is freed by Rust (round-trip the pointer back through a Rust `extern "C"` free function via `Box::from_raw`); C memory is freed by C. Mixing allocators is UB.
- A pointer handed to C must stay valid as long as C may use it — the `CString` trap above is the most common dangling-pointer bug in Rust FFI.
- `Box::into_raw` *releases* ownership across the boundary (leaking it from Rust's view); `Box::from_raw` *reclaims* it. These are the explicit ownership-transfer verbs FFI forces you to write by hand, because the type system stops at the boundary.

> **🦀 From your toolbox →** `#[repr(C)]` is about pinning down byte layout the way you would for a network wire format or an `mmap`'d file header. Here Java actually sharpens the contrast: the JVM never lets you observe object layout at all, and Swift likewise hides it (which is exactly why both reach for explicit (de)serialisation to talk to the outside world). The analogy to C breaks down on *default* layout: in C, declaration order *is* the layout, full stop, whereas Rust's default is deliberately unspecified so the compiler can pack fields — you must opt in to the C guarantee. Treat any `struct` that touches FFI, hardware, or a file format without `#[repr(C)]` as a bug.

## The aliasing model: why `&mut` is `noalias`

This is the part with no C analogue, and the reason Rust's optimiser can be more aggressive than C's. Rust guarantees a live `&mut T` is the *unique* path to its referent — no other live reference or pointer may read or write that memory while the `&mut` is usable. rustc tells LLVM this by tagging `&mut` parameters `noalias` (and `&T` `readonly`), the hint C's `restrict` gives but applied *automatically and pervasively*. The optimiser then keeps values in registers and reorders loads/stores across calls, trusting nothing else can observe or clobber that location.

The catch for `unsafe` authors: this guarantee holds for *raw-pointer code too*. If you create a `&mut` and a raw pointer to the same place and write through both while the `&mut` is live, you have violated `noalias` — UB — even though no `&mut`/`&mut` pair ever existed. This is why `&raw mut x` exists: it takes a pointer without ever forming the conflicting reference.

There is an ongoing research effort to write down precisely *the rules raw pointers must obey to coexist with references*. **Stacked Borrows** (Jung et al.) modelled borrows as a per-location stack of permission tags, each access checking and popping the stack; its successor **Tree Borrows** generalises this to a tree to admit more valid patterns. Neither is yet *the* official, normative spec — but **Miri implements them**, so they are the de-facto rulebook your unsafe code is tested against today.

> **🎓 Tripos link →** The mental picture from Semantics of Programming Languages is useful here, stated plainly: Stacked/Tree Borrows is a step-by-step machine that tracks, for each memory location, which pointers are currently allowed to touch it, and *gets stuck* the moment an access breaks the rules — that "stuck" state is exactly UB. It is the runtime side of the same aliasing discipline the borrow checker enforces at compile time. `&mut`-as-`noalias` — "while this mutable reference is live, nothing else may read or write here" — is that discipline pushed all the way down to the machine.

## Tooling: how to actually trust your unsafe code

The borrow checker is *static* and now stopped helping, so you need dynamic tools that run your program and watch for violations:

- **Miri** (`cargo +nightly miri test`) interprets your program against Rust's abstract machine *and* the Tree-Borrows model. It catches OOB access, use-after-free, invalid values, unaligned access, data races, and aliasing violations no other tool sees, because it knows about `noalias`. On `slice::from_raw_parts_mut(0x1234 as *mut i32, 10000)` it reports a dangling pointer with no provenance.
- **AddressSanitizer / ThreadSanitizer** (`RUSTFLAGS="-Zsanitizer=address"`, nightly) — the ASan/TSan from your C work, instrumenting the real binary. They catch heap/stack overflows and data races *in machine code*, including across the FFI boundary into C where Miri cannot reach.
- **`cargo-careful`** runs your tests with extra debug assertions baked into a specially built std.

The decisive limitation: all three are *dynamic*. **If Miri flags your code it is a bug; if Miri is silent your code is not proven correct** — only that the tested paths were clean. Coverage of your unsafe paths is a first-class testing concern, exactly as in C.

> **⚠️ Pitfall →** The cardinal sin is reaching for `unsafe` to silence the borrow checker on safe code — e.g. transmuting `&T` to `&mut T` to "just mutate it". This is *always* UB (it fabricates a second `&mut`/aliasing `&`, breaking `noalias`), even when it appears to work, and the optimiser will eventually miscompile it. The fix is one of: `RefCell`/`Cell`/`Mutex` for interior mutability ([smart pointers](15-smart-pointers.md)), a redesign of ownership, or — if you genuinely need raw pointers — `&raw mut` plus a *written-down* `// SAFETY:` comment and a Miri run. If you cannot articulate why it's correct in the comment, you do not have a reason.

## When `unsafe` is justified

`unsafe` is justified, roughly, only when: (1) you are at an FFI/hardware/OS boundary where Rust's guarantees provably do not apply; (2) you implement a data structure whose soundness the borrow checker cannot see (intrusive lists, arenas, lock-free structures) and wrap it in a sound safe API; or (3) you have a *measured* performance need that a checked bound demonstrably costs. The deliverable is always the same: a minimal `unsafe` block, a `// SAFETY:` comment stating why it's correct, and Miri/sanitizer coverage. Everything else is cargo-culting; the community norm — encoded in lints like `clippy::undocumented_unsafe_blocks` — treats an unexplained `unsafe` as a code smell. The motivation is not pedantry: the entire memory-safety-vulnerability class (CWE-119 buffer overflow, CWE-416 use-after-free) that drives roughly 70% of critical CVEs at Microsoft and Google lives *only* inside unsafe code in a Rust program. Minimising and auditing `unsafe` is the security argument for the language.

## Mental-model recap

- `unsafe` unlocks exactly five superpowers — deref raw pointers, call `unsafe fn`, `unsafe impl`, touch `static mut`, read `union` fields — and **turns off no other check**. Ownership, borrowing, lifetimes, and types are all still enforced inside `unsafe`. It is a capability grant, not a mode switch.
- The model is one of obligations: `unsafe` is you taking on, by hand, a memory-safety guarantee the conservative checker handed back. That guarantee is *not local* — one wrong `unsafe` block can corrupt the whole program, and the optimiser (which trusts `&mut` is `noalias`) can miscompile far from the violation.
- The professional pattern is *encapsulation*: a tiny audited `unsafe` core behind a **sound** safe API — one no safe caller can drive to UB. That is precisely what `Vec`, `Box`, `Rc`, and `split_at_mut` are. An unsound "safe" API is a bug even if it never misbehaves in testing.
- Raw pointers (`*const T`/`*mut T`) *are* C pointers — no lifetimes, may alias, may dangle, no destructor; create with `&raw const`/`&raw mut`, deref only in `unsafe`. FFI rides the C ABI via `extern "C"`, needs `#[repr(C)]` for shared structs and `CString`/`CStr` for strings, and **ownership does not cross the boundary** — you transfer it by hand with `into_raw`/`from_raw`.
- The compiler stopped checking, so *you* must: Miri (aliasing + the abstract machine, dynamic), ASan/TSan (the real binary, across FFI), `cargo-careful`. Miri flagging means a bug; Miri silent means *only* "the tested paths were clean", never "proven sound".

## Exercises

1. Here is a safe-looking function with no bounds check: `fn get(v: &[i32], i: usize) -> i32 { unsafe { *v.as_ptr().add(i) } }`. Without changing the body's strategy, decide where the fix belongs: would you add `assert!(i < v.len())` *before* the `unsafe`, or mark `get` itself as `unsafe fn` and push the obligation to the caller? Argue which choice makes `get` a *sound safe API* versus which just relocates the danger, and write the one-line `// SAFETY:` comment your chosen version needs.

2. Predict, *before running*, whether `cargo +nightly miri run` complains about this, and say which line is the violation: `let r = &mut 0i32; let p = r as *mut i32; *r = 1; unsafe { *p = 2; } println!("{}", *r);`. Explain it in terms of `noalias` — which write happens while `r` is still the unique live path. Then rewrite it with `&raw mut` so Miri stays silent, and say in one sentence why that version is fine.

3. You have three ways to share mutable global state: `static mut COUNTER: u64`, `static COUNTER: AtomicU64`, and `static COUNTER: Mutex<u64>`. For a counter incremented from several threads, which would you reach for and why? Then name one situation where you'd pick the `Mutex` over the `Atomic` even though both are safe. (You do not need to write the increment code — just the design reasoning.)

4. Predict the output of `std::mem::size_of` and `std::mem::offset_of!` for `struct Header { tag: u8, len: u32 }` under the default `repr` versus `#[repr(C)]`, *then* verify by running it. Explain why the default layout would corrupt data if you passed such a struct to a C function expecting `struct { uint8_t tag; uint32_t len; }`, and what specifically `#[repr(C)]` pins down.

5. Which of these two `unsafe impl Send for Wrapper {}` impls is sound, and which is a latent data race? (a) `Wrapper` holds a `*mut T` but every method takes `&self` through an owned `Mutex<()>` lock before touching the pointer; (b) `Wrapper` holds a `*mut T` and methods read/write it directly with no synchronisation. State the invariant the sound one relies on, and describe (no full program needed) the two-thread scenario that makes the unsound one race — and what Miri would report.

6. (★) The signature `fn alias(x: &mut i32) -> (&i32, &mut i32)` cannot be written in safe Rust. Explain in one sentence why, using aliasing-XOR-mutability. Now suppose someone implements it with raw pointers and `transmute` so it compiles. Argue whether *any* implementation could be sound, and if not, name the rule a caller could break to get UB. (The lesson: `unsafe` lets you write things that are true-but-unprovable, not things that are false — here there is nothing correct to stand behind.)
