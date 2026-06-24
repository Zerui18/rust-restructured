# 18. Macros & Metaprogramming

You have already written code that writes code. In Compiler Construction you built a code generator that walked a typed AST and emitted target instructions; in Further Java you relied on `javac` synthesising bridge methods. Rust's macros expose that same machinery — *programs that transform programs* — but they run as a phase of `rustc`, before type checking, over a Rust-specific representation of source. This chapter covers what that representation is, how the two macro families consume it, and the one property (hygiene) that makes Rust macros categorically safer than anything `#define` ever gave you.

The single most important mental shift: a macro is not a function. A function is a value-level abstraction invoked at runtime over fully-typed arguments. A macro is a *syntactic* abstraction invoked at compile time over **token trees**, expanded into source before the compiler knows the types of anything. That difference is exactly why macros can do three things functions cannot: take a variadic, heterogeneously-typed argument list (`println!("{} {}", a, b)`), construct items like `impl` blocks (which must exist at compile time), and build domain-specific syntax the type system cannot otherwise express.

## What a macro operates on: token trees

Before parsing into an AST, `rustc` lexes source into a flat token stream, then groups it into **token trees**: a token tree is either a single token (`identifier`, literal, punctuation) or a *delimited group* — anything balanced inside `()`, `[]`, or `{}`, recursively. So `vec![1, 2, 3]` is the macro name `vec`, the bang `!`, and one bracket-delimited group containing the trees `1` `,` `2` `,` `3`.

Macros run on this layer. A declarative macro pattern-matches over token trees; a procedural macro receives a `proc_macro::TokenStream` (a sequence of token trees) and returns one. Crucially, **the input need not be valid Rust** — `sql!(SELECT * FROM posts)` lexes into legal token trees even though `SELECT * FROM posts` is not a Rust expression. The only hard constraint is that delimiters balance. This is the source of macros' expressive power and their cost: the tokens carry no types yet.

> **🎓 Tripos link →** This is precisely the lexer/parser boundary from Compiler Construction. Lexing produces tokens; macro expansion is a *tree-rewriting pass* inserted between tokenisation and full parsing/name-resolution. Declarative macros are a small term-rewriting system over the token-tree grammar; procedural macros are arbitrary `TokenStream -> TokenStream` functions — i.e. a hook for you to write an extra compiler pass in safe Rust and have `rustc` dynamically load and run it.

## Family one: `macro_rules!` (declarative)

A declarative macro is defined with `macro_rules!` and reads like a `match` whose scrutinee is *source structure* rather than a runtime value. Each arm has a **matcher** (the pattern over token trees) and a **transcriber** (the replacement tokens). Here is a small but real one — a `HashMap` literal, the kind of thing the standard library pointedly does *not* give you:

```rust
macro_rules! hmap {
    // matcher: zero or more `key => value` pairs, comma-separated,
    // with an optional trailing comma.
    ($($k:expr => $v:expr),* $(,)?) => {{
        let mut m = ::std::collections::HashMap::new();
        $( m.insert($k, $v); )*
        m
    }};
}

let scores = hmap! {
    "alice" => 10,
    "bob"   => 7,
};
```

Dissect the matcher `($($k:expr => $v:expr),* $(,)?)`:

- `$k` and `$v` are **metavariables** — bindings in the macro's own namespace, sigil'd with `$` so they never collide with Rust identifiers.
- `:expr` is a **fragment specifier**. It tells the matcher *which grammar production* to parse `$k` as. `expr` means "one full Rust expression."
- `$( ... ),*` is a **repetition**: match the inner fragment zero or more times, with a literal `,` separator. The separator is optional and sits between the `)` and the `*`; the repetition operator is one of `*` (zero+), `+` (one+), or `?` (zero or one).
- `$(,)?` is a second, separate repetition allowing a single optional trailing comma — the standard idiom for accepting `a, b,`.

The transcriber `$( m.insert($k, $v); )*` re-uses the *same* repetition shape: for each `(k, v)` pair the matcher captured, it emits one `insert` call, substituting the captured fragments. The number of repetitions in the transcriber is inferred from the metavariables mentioned inside it — `$k` and `$v` were each matched the same number of times, so one `insert` is generated per pair.

> **🦀 From your toolbox →** The matcher/transcriber pair is OCaml `match` over an algebraic structure, and `$(...)* ` is a fold over the matched list. But the analogy breaks in two ways. First, you are matching *syntax*, not a value: `$x:expr` binds an unevaluated expression tree, not a result. Second, there is no exhaustiveness check and no types — an arm either parses or it doesn't, and a malformed call is a parse error, not a non-exhaustive-match warning.

### Fragment specifiers are a contract, and `expr` fragments are opaque

The fragment specifier you choose is a promise about what `rustc` will parse and — critically — what it will let you do with the result afterwards. The common ones:

| Specifier | Matches | Typical use |
|-----------|---------|-------------|
| `expr` | an expression | values, arguments |
| `ty` | a type | generic bounds, field types |
| `ident` | an identifier | names of generated items/vars |
| `pat` | a pattern | match arms, `let` bindings |
| `path` | a path like `a::b::C` | trait/type references |
| `tt` | a single token tree | "raw, parse-it-myself" escape hatch |
| `block`, `stmt`, `item`, `literal`, `lifetime`, `meta`, `vis` | as named | structural pieces |

Once the matcher binds a fragment as `expr`, `ty`, etc., that capture becomes a **single opaque AST node** — you cannot subsequently re-inspect or pattern-match *inside* it with another matcher. Concretely:

```rust
macro_rules! twice {
    ($e:expr) => { { let v = $e; v + v } };
}

twice!(3 + 4) // expands to `{ let v = 3 + 4; v + v }` == 14
```

`3 + 4` is captured as one expression and substituted as a unit; it is **not** re-tokenised and pasted twice (which would have given `3 + 4 + 3 + 4`). This opacity is a feature: it guarantees that a captured expression binds with the precedence it had at the call site, so `twice!(a, b)` won't accidentally smear across the `,`. The escape hatch is `tt`, which captures a *raw* token tree with no parsing commitment — that is how recursive "tt-munching" macros and DSLs pull arguments apart one token at a time:

```rust
macro_rules! count {
    () => (0usize);
    ($head:tt $($tail:tt)*) => (1usize + count!($($tail)*));
}
count!(a b "x" 9) // == 4
```

> **⚠️ Pitfall →** Specifier ordering matters because the macro matcher commits as it parses. After certain fragments, only a restricted set of "follow" tokens is permitted; e.g. an `:expr` or `:stmt` fragment may only be followed by `=>`, `,`, or `;`. Writing `($e:expr $rest:tt)` fails with *"`$e:expr` is followed by `$rest`, which is not allowed for `expr` fragments"*. The fix is to insert one of the permitted separators (`($e:expr ; $rest:tt)`) or downgrade `$e` to `:tt` and parse it yourself. This restriction exists so the matcher stays unambiguous without unbounded lookahead — the same LL-parsing concern you met in Compiler Construction.

### Hygiene — the entire reason this is not the C preprocessor

This is the headline feature, and it is the one place where your `cpp` instincts will actively mislead you. Consider the canonical C disaster:

```c
#define SWAP(a, b) { int tmp = a; a = b; b = tmp; }
// SWAP(x, tmp) expands to { int tmp = x; x = tmp; tmp = tmp; } -- broken.
```

`#define` is **text substitution**. The macro's `tmp` and the caller's `tmp` are the same lexical name, so they capture each other. C macro authors paper over this with `__` prefixes and prayer. Rust's declarative macros are **hygienic**: identifiers introduced *by the macro body* are tagged with a hidden "syntax context" distinct from identifiers that arrive *through metavariables from the call site*. They live in different scopes even when spelled identically. Watch:

```rust
macro_rules! make_temp {
    ($e:expr) => {{
        let tmp = $e;   // `tmp` introduced by the macro
        tmp * 2
    }};
}

let tmp = 10;                 // caller's `tmp`
let r = make_temp!(tmp + 1);  // `$e` = caller's `tmp + 1`
// prints: 10 22
println!("{} {}", tmp, r);
```

The macro-introduced `tmp` and the caller's `tmp` *do not collide*: `$e` resolves to the caller's `tmp` (because it came in through a metavariable), while the `let tmp` inside the body is invisible to the outside world. The C version of this is silently wrong; the Rust version is correct by construction, and the compiler proved it.

> **⚙️ Under the hood →** Hygiene is implemented by attaching a `SyntaxContext` to every identifier's span. Name resolution compares the (symbol, context) pair, not just the symbol. Tokens written literally in the macro definition carry the *definition site* context; tokens substituted from a metavariable retain their *call site* context. Two `tmp`s with different contexts are different names, full stop. Note the boundaries of hygiene: it is *mixed-site* for `macro_rules!` — local variables and labels are hygienic (definition-site), but for things like type and `fn` name lookup the resolution can reach call-site scope. When a macro needs to name a standard-library item reliably regardless of what the caller has shadowed, use the special `$crate` metavariable (e.g. `$crate::collections::HashMap`) so the path resolves relative to the *defining* crate.

> **🦀 From your toolbox →** Closest analogue is hygienic `let` in a lambda you pass to a higher-order function — the closure's locals can't be clobbered by the caller's bindings, and lexical scoping handles capture. The analogy breaks because a macro isn't a value and creates *new bindings in the caller's code*; hygiene is what re-imposes lexical-scoping discipline on something that would otherwise be raw text splicing.

### Scope and export

Unlike functions, a `macro_rules!` macro must be *textually defined before it is used* within a file (it is brought into scope by source order, an artefact of expansion happening early). To use one from another crate, annotate it with `#[macro_export]`, which hoists it to the crate root; within a crate, `pub use` and the `use crate::path::macro_name` form also work in the 2018+ module system.

## Family two: procedural macros

When pattern-matching over token trees is not enough — you need to actually *parse* the input, walk its structure, and compute new code — you reach for a **procedural macro**. A proc-macro is, mechanically, a public function `fn(TokenStream) -> TokenStream` (or two-argument, for attributes) that `rustc` compiles into a plugin and *executes during compilation of the downstream crate*. There are three kinds, distinguished only by the attribute on the function:

1. **Derive macros** — `#[proc_macro_derive(Name)]` — invoked by `#[derive(Name)]` on a struct/enum; they *add* items (typically trait impls).
2. **Attribute macros** — `#[proc_macro_attribute]` — define a new attribute `#[name(...)]` usable on most items; they *replace* the annotated item with transformed output.
3. **Function-like macros** — `#[proc_macro]` — invoked as `name!(...)`; like `macro_rules!` but with arbitrary parsing logic.

A structural constraint: proc-macros **must live in their own crate** with `proc-macro = true` set in `Cargo.toml`, because the crate is compiled for the *host* (the compiler) rather than the target, and loaded as a dynamic library by `rustc`. The convention is that a library crate `foo` ships its derive macros in a sibling crate `foo_derive`.

> **🎓 Tripos link →** A proc-macro is a user-supplied compiler pass, dynamically linked into `rustc`. The `TokenStream -> TokenStream` signature is exactly the interface of a syntax-directed translation stage between lexing and elaboration. Because it runs *inside* the compiler with full Turing-completeness, it can do arbitrary computation (read files, hit the network — and it will run on every build), which is why the community treats proc-macros as a sharp tool and audits them.

### The `syn` / `quote` toolchain

The `proc_macro` crate gives you `TokenStream`, but parsing raw tokens into a usable Rust AST by hand is the full parser problem you'd rather not redo. Two ecosystem crates make proc-macros tractable:

- **`syn`** parses a `TokenStream` into typed AST structs (e.g. `DeriveInput`, `ItemFn`, `Expr`).
- **`quote`** does the inverse: the `quote! { ... }` macro is a *quasi-quoter* that lets you write output Rust with `#var` interpolation holes, producing a `TokenStream`.

Here is the complete shape of the most common case — a derive macro. The user wants `#[derive(Describe)]` to synthesise an `impl` of a `Describe` trait whose method returns the type's name:

```rust
// crate: describe_derive  (Cargo.toml: [lib] proc-macro = true; deps: syn, quote)
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    // 1. Parse the annotated item into a typed AST node.
    let ast = parse_macro_input!(input as DeriveInput);
    let name = &ast.ident;                       // the type's identifier
    let (ig, tg, wc) = ast.generics.split_for_impl(); // generic plumbing

    // 2. Build the output as a quasi-quote, splicing `name` in with `#`.
    let expanded = quote! {
        impl #ig Describe for #name #tg #wc {
            fn describe() -> &'static str {
                stringify!(#name)
            }
        }
    };

    // 3. Hand the generated tokens back to the compiler.
    expanded.into()
}
```

Three things to internalise. First, `parse_macro_input!` is the idiomatic front door — on a parse error it emits a proper compiler diagnostic and returns early instead of panicking opaquely. Second, `split_for_impl` threads the input type's generics and `where`-clause through to the impl, so the derive works for `struct Pair<T>` as well as `struct Unit`; forgetting it is the classic "my derive doesn't handle generics" bug. Third, `stringify!(#name)` turns the spliced identifier into a `&'static str` *at compile time* with zero allocation — Rust has no runtime reflection, so the name must be baked in during expansion; that is the whole reason this is a derive macro rather than a blanket trait impl.

A derive macro returns *additional* items appended to the source. An attribute macro's signature is `fn(attr: TokenStream, item: TokenStream) -> TokenStream`: `attr` is the tokens inside `#[name(...)]`, `item` is the entire annotated item, and the returned stream *replaces* it (so `#[route(GET, "/")] fn index() {}` lets the macro rewrite `index` wholesale). A function-like proc-macro is `fn(TokenStream) -> TokenStream` invoked as `name!(...)` and is what powers things like a compile-time-checked `sql!(...)`.

> **⚙️ Under the hood →** `#[derive(Describe)]` does not mutate your struct — the original definition is left intact and the macro's output `TokenStream` is *concatenated* into the module. This is why a derive can only *add* impls and items, never edit fields, whereas an attribute macro receives the item itself and can rebuild it. The generated tokens are then fed back through the normal parse → resolve → typecheck pipeline, so a buggy macro surfaces as a type error in *generated* code — see spans below for how that error gets a sane location.

### Spans, hygiene, and error reporting in proc-macros

Every token carries a **`Span`**: a record of where it came from in the source, used for diagnostics and for hygiene. When you build output with `quote!`, the synthesised tokens get `Span::call_site()` by default — meaning errors in generated code point at the macro *invocation*, which is what users want. Proc-macros are **not** automatically hygienic the way `macro_rules!` is: identifiers you fabricate (via `Span::call_site()`) resolve at the call site and *can* capture or collide. If you need a genuinely fresh, uncapturable identifier, mint it with `Span::mixed_site()` or `Span::def_site()` semantics (`quote::format_ident!` plus the right span), or simply generate obscure names. For surfacing your own errors, the right move is `syn::Error::new_spanned(&offending_node, "message").to_compile_error()` — this produces a `compile_error!` invocation tied to the *exact span* of the offending input, so the user sees a red squiggle under their own code, not inside your macro.

> **⚠️ Pitfall →** A derive function *must* return `TokenStream`, not `Result`, so it has no `?`. The lazy path is `syn::parse(input).unwrap()`, but a parse failure then panics with a useless `"called Result::unwrap on an Err"` and no location. Always prefer `parse_macro_input!` (handles the early-return for you) and `syn::Error::to_compile_error()` for semantic errors, so failures become real diagnostics with spans instead of a compiler-internal panic.

## When to reach for a macro (mostly: don't)

Macros are the highest-leverage and highest-cost abstraction in the language. The decision tree:

- **Function** — the default. If the inputs have known, fixed types and you only need value-level abstraction, write a function. It is type-checked at definition, plays perfectly with tooling, and costs nothing in readability.
- **Generics + traits** — when you need to abstract over *types* with a known shape of operations (see [generics and traits](08-generics-and-traits.md)). Monomorphisation gives you the performance of code generation with full type checking and clean errors. This handles the vast majority of "I want this to work for many types" cases.
- **Macro** — only when the type system genuinely cannot express it:
  - **Variadics / heterogeneous arg lists** — `println!`, `vec!`, your `hmap!`. Functions can't vary arity or argument types.
  - **Item generation** — synthesising `impl` blocks (`#[derive]`), which must exist at compile time and which no function call can produce.
  - **DSLs and compile-time validation** — embedding foreign syntax (`sql!`, HTML templating, `quote!` itself) or validating input before runtime.
  - **Boilerplate elimination** the type system can't fold away — e.g. generating one impl per type in a list.

The cost is real and not just stylistic. A macro's contract is its grammar, which IDEs and `rust-analyzer` understand only partially; "go to definition" through a macro is lossy, error messages can point into expanded code, and a proc-macro runs arbitrary code on every build and slows compilation. The community idiom is unambiguous: **prefer functions, then generics, and treat macros as a last resort justified by one of the bullet points above.**

> **🦀 From your toolbox →** Closest cousins: C++ templates and Java annotation processors. `#[derive]` is conceptually a Java annotation processor (`javax.annotation.processing`) — both generate code from annotations at compile time. The difference is that Rust derives produce real, type-checked Rust integrated into the same compilation, with no reflection and no runtime cost; Java processors generate `.java`/bytecode and frequently lean on reflection at runtime. C++ variadic templates overlap with variadic macros but are type-driven and far more constrained than `TokenStream` rewriting.

## Aside: build scripts (`build.rs`)

There is a third compile-time code-generation mechanism that is *not* a macro but solves adjacent problems. A file named `build.rs` at a crate root is compiled and run by Cargo *before* the crate itself. It can generate `.rs` files (which the crate then `include!`s), compile and link C code, probe the environment, and emit instructions to Cargo via specially-formatted stdout lines like `cargo:rustc-link-lib=foo`. Use it for codegen driven by *external* inputs — protobuf/`bindgen` output, generated tables, FFI link configuration (relevant to [unsafe and FFI](16-unsafe-and-ffi.md)). It runs once per build on the host, has full filesystem/network access, and — unlike a macro — operates on whole files rather than token streams.

## Mental-model recap

- A macro transforms **token trees** at compile time, *before* type checking; a function transforms typed **values** at run time. That timing difference is why macros alone can be variadic, build `impl`s, and host DSLs.
- `macro_rules!` is a term-rewriting system: matchers pattern-match token trees via **fragment specifiers** (`expr`/`ty`/`ident`/`tt`/…) and **repetitions** (`$(...)sep*`); a captured `expr`/`ty` fragment is an **opaque** node, not re-tokenisable text.
- **Hygiene** is the decisive advantage over `cpp`: macro-introduced identifiers carry a distinct syntax context and cannot capture or be captured by call-site names. `#define` has none of this; Rust's collision is impossible by construction.
- Procedural macros are user-written compiler passes (`TokenStream -> TokenStream`) in a dedicated `proc-macro = true` crate, built on **`syn`** (parse) and **`quote`** (emit); three flavours — `derive`, `attribute`, function-like. They are *not* auto-hygienic, so manage **spans** for both hygiene and error reporting.
- Decision order: **function → generics/traits → macro**, last only. Macros tax readability, tooling, and build time; spend that cost only for what the type system cannot express.

## Exercises

1. Write a `vec_of_strings!` macro that accepts a comma-separated list of string-literal-or-expression arguments and produces a `Vec<String>`, calling `.to_string()` on each. Support a trailing comma. Then call it with **zero** arguments — what type is `Vec::new()` inferred to be, and why does it still compile here but would fail if you used the result ambiguously?

2. Take the hygiene example's `make_temp!` and *deliberately* try to break it: write a macro `leak!` that introduces `let leaked = ...;` and a caller that reads `leaked` afterwards. Confirm it fails to compile, read the `cannot find value leaked in this scope` error, and explain in one sentence why this is hygiene working as designed rather than a bug.

3. (★) Implement a `tt`-munching macro `min!` that takes one or more comma-separated expressions and expands to nested `std::cmp::min` calls (`min!(a, b, c)` → `min(a, min(b, c))`). You will need a base case (single expression) and a recursive case. Why must the elements be `:expr` and the recursion be driven by re-invoking `min!`, and where would a naive `$($x:expr),+` single-arm definition get stuck?

4. (★) Sketch a derive macro `#[derive(FieldNames)]` that, for a named-field struct, generates `fn field_names() -> &'static [&'static str]`. Outline the `syn` types you'd match on to get at the fields (start from `DeriveInput.data`), how you'd iterate them in `quote!`, and how you'd emit a proper `compile_error!` with a correct **span** if the macro is applied to an enum or a tuple struct instead.

5. Explain why `#[proc_macro_derive]` functions return `TokenStream` rather than `Result<TokenStream, _>`, and rewrite a panicking `syn::parse(input).unwrap()` into the idiomatic non-panicking pattern. What does the *user* of your derive see in each case when their annotated code is malformed?

6. For each of these, decide function / generic / declarative macro / proc-macro and justify in one line: (a) a routine that swaps two `&mut T`; (b) `assert_all_eq!(a, b, c, d)` that checks every argument is equal and prints the first mismatch; (c) deriving JSON (de)serialisation for arbitrary structs; (d) a `#[timed]` attribute that wraps a function body in start/stop timing.
