# Preface — How to read this book

This is the official *Rust Programming Language* book, taken apart and put back together for **one specific reader**: a third-year Cambridge Computer Science student who already writes C, C++, Swift, Python, JavaScript/TypeScript, and Java, and who has sat Foundations of Computer Science (OCaml), Programming in C and C++, Semantics of Programming Languages, Concurrent and Distributed Systems, and Compiler Construction.

That reader does not need to be told what a `for` loop is. They need the *delta* — the handful of genuinely new ideas in Rust — delivered at full altitude, in the order those ideas actually depend on one another, with every concept anchored to something they already understand.

## What changed, and why

The official book is excellent but it is written for *everyone*, including people for whom this is a first systems language. Three structural consequences follow that are wrong **for you**:

1. **The ownership system is scattered.** Ownership (ch. 4), lifetimes (ch. 10.3), and smart pointers (ch. 15) are three faces of one idea — the affine, region-checked memory model — yet the official book separates them by hundreds of pages. Here they form a single contiguous **Part II**, taught as one coherent theory.
2. **One topic, two locations.** Pattern matching is introduced in ch. 6 and completed in ch. 19; traits in ch. 10 and extended in ch. 20. We merge each into one place.
3. **The on-ramp is too long.** Variables, integer types, and control flow consume the first three chapters. We compress all of it into a single chapter — *The Language Delta* — written as a diff against C and OCaml.

The reordering principle is **conceptual dependency and difficulty-for-an-expert**, not gentle beginner ramp. We front-load the one thing that is actually hard and new (ownership/borrowing/lifetimes), then build the type system on top of it, then idioms, then concurrency, then the systems-level and metaprogramming machinery, then engineering and capstones.

## The mental model to hold throughout

> **Rust = an ML-family type system (sum types, exhaustive matching, traits-as-type-classes, no null) bolted onto a C++ memory model (RAII, moves, no GC) — with the borrow checker statically proving the thing C++ leaves to your discipline: that you never alias mutable state or use memory after it dies.**

Everything in this book is a corollary of that sentence. When a rule seems arbitrary, trace it back to "the compiler is proving memory- and data-race-safety at compile time, and it will only accept programs it can prove correct."

## House style — the recurring callouts

Throughout, you will see four kinds of margin-notes. Learn to read them:

> **🦀 From your toolbox →** maps a Rust feature to its closest analogue in a language you already know (C++ `unique_ptr`, OCaml `match`, Java `interface`, …) — *and names where the analogy breaks*.

> **⚙️ Under the hood →** what the compiler/runtime actually does: memory layout, monomorphisation, vtables, state machines. For when "it just works" isn't enough for you.

> **⚠️ Pitfall →** the specific error the borrow checker (or type checker) will throw, why, and the idiomatic fix. Learning Rust *is* learning to predict these.

> **🎓 Tripos link →** where a concept connects to a course you have already sat, with the formal framing made explicit.

Each module ends with a **Mental-model recap** (the three or four sentences worth memorising) and a few **Exercises** that target the failure modes, not the happy path.

## How to actually use it

- Read Parts I–II in order. They are load-bearing; nothing later makes sense without ownership.
- After that, Parts III–VII are loosely independent — but the given order is the recommended one.
- Keep a scratch crate open (`cargo new scratch`). Every time a callout claims the compiler will reject something, *make it reject it*, then fix it. The borrow checker is the best teacher in the book.
- The capstones in Part VII (`21`) integrate everything; treat them as the exam.

The appendix maps every module back to the official book's chapter numbers and to your specific Tripos courses, so you can cross-reference the canonical source whenever you want the "everyone" version of an explanation.
