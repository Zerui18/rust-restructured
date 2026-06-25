# Preface — How to read this book

This is the official *Rust Programming Language* book, taken apart and put back together for **one specific reader**: a third-year Cambridge Computer Science student who is most fluent in Java, Swift, and Python (with working OCaml, TypeScript, and C), and who has sat Foundations of Computer Science (OCaml), Programming in C and C++, Concurrent and Distributed Systems, and Compiler Construction.

That reader does not need to be told what a `for` loop is. They need the *delta* — the handful of genuinely new ideas in Rust — delivered at full altitude, in the order those ideas actually depend on one another, with every concept anchored to something they already understand.

## What changed, and why

The official book is excellent but it is written for *everyone*, including people for whom this is a first systems language. Three structural consequences follow that are wrong **for you**:

1. **The ownership system is scattered.** Ownership (ch. 4), lifetimes (ch. 10.3), and smart pointers (ch. 15) are three faces of one idea — Rust's single model of who owns each value and for how long — yet the official book separates them by hundreds of pages. Here they form a single contiguous **Part II**, taught as one coherent topic.
2. **One topic, two locations.** Pattern matching is introduced in ch. 6 and completed in ch. 19; traits in ch. 10 and extended in ch. 20. We merge each into one place.
3. **The on-ramp is too long.** Variables, integer types, and control flow consume the first three chapters. We compress all of it into a single chapter — *The Language Delta* — written as a diff against the languages you already know.

The reordering principle is **conceptual dependency and difficulty-for-an-expert**, not gentle beginner ramp. We front-load the one thing that is actually hard and new (ownership/borrowing/lifetimes), then build the type system on top of it, then idioms, then concurrency, then the systems-level and metaprogramming machinery, then engineering and capstones.

## The mental model to hold throughout

> **Rust pairs a type system in the spirit of Swift and OCaml — enums that carry data, exhaustive `match`, no null, traits/protocols for shared behaviour — with hands-on control over memory and no garbage collector. The compiler's borrow checker then automatically proves the thing other languages leave to your discipline: that you never mutate data through one name while another name is still looking at it, and never touch memory after it has been freed.**

Everything in this book is a corollary of that sentence. When a rule seems arbitrary, trace it back to "the compiler is proving memory- and data-race-safety at compile time, and it will only accept programs it can prove correct."

## House style — the recurring callouts

Throughout, you will see four kinds of margin-notes. Learn to read them:

> **🦀 From your toolbox →** maps a Rust feature to its closest analogue in a language you already know (a Swift `enum` or `protocol`, a Java `interface`, Python's reference counting, …) — *and names where the analogy breaks down*.

> **⚙️ Under the hood →** what the compiler/runtime actually does: memory layout, monomorphisation, vtables, state machines. For when "it just works" isn't enough for you.

> **⚠️ Pitfall →** the specific error the borrow checker (or type checker) will throw, why, and the idiomatic fix. Learning Rust *is* learning to predict these.

> **🔧 In practice →** a concrete, real-world situation where the feature earns its keep — when and why you'd actually reach for it in real code, with a small realistic sketch.

Each module ends with a **Mental-model recap** (the three or four sentences worth memorising) and a few **Exercises** that push past the chapter — applying the idea to new situations rather than rehearsing it.

## How to actually use it

- Read Parts I–II in order. They are load-bearing; nothing later makes sense without ownership.
- After that, Parts III–VII are loosely independent — but the given order is the recommended one.
- Keep a scratch crate open (`cargo new scratch`). Every time a callout claims the compiler will reject something, *make it reject it*, then fix it. The borrow checker is the best teacher in the book.
- The capstones in Part VII (`21`) integrate everything; treat them as the exam.

The appendix maps every module back to the official book's chapter numbers and to the Tripos courses it draws on, so you can cross-reference the canonical source whenever you want the "everyone" version of an explanation.
