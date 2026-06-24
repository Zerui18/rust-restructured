# Rust, Restructured

*A systems-and-semantics path to mastering Rust — the official book, reordered for an experienced programmer.*

📖 **Read it online:** https://zerui18.github.io/rust-restructured/

## What this is

The 21 chapters of *The Rust Programming Language* reorganised by **conceptual dependency and difficulty-for-an-expert** rather than a gentle beginner ramp:

- the ownership / borrowing / lifetime system is unified into one contiguous part (the official book scatters it across chapters 4, 10, and 15);
- pattern matching is taught in one place (official ch. 6 + 19 merged);
- trivial syntax is compressed into a single *“language delta”* against C and OCaml;
- every concept is anchored to languages a working programmer already knows — C/C++ (RAII, moves, `unique_ptr`/`shared_ptr`), OCaml (ADTs, pattern matching), Java (interfaces, generics), Swift, TypeScript.

22 modules across 7 parts — see the [table of contents](src/SUMMARY.md).

## Build locally

```bash
cargo install mdbook
mdbook serve --open      # live preview
mdbook build             # static HTML → docs/  (what GitHub Pages serves)
```

## Attribution & disclaimer

This is an **original, independent re-teaching** written for personal study. It is **not** affiliated with, endorsed by, or copied from the Rust project. It is *inspired by* the structure and topic coverage of [*The Rust Programming Language*](https://doc.rust-lang.org/book/) (© the Rust project contributors, dual-licensed [MIT](https://github.com/rust-lang/book/blob/main/LICENSE-MIT) / [Apache-2.0](https://github.com/rust-lang/book/blob/main/LICENSE-APACHE)); all prose and code examples here were written fresh for this restructuring.

The text was produced with AI assistance and, while technically spot-checked, may contain errors — verify against the [official book](https://doc.rust-lang.org/book/) and the [`std` docs](https://doc.rust-lang.org/std/) before relying on it.
