<!--
Copyright © 2026 AJ. All Rights Reserved.
https://skev.dev | https://skev.org
-->
# About Skev

Skev is a compiled, statically typed programming language for game development and systems programming. Fast like C++. Readable like Python.

-----

## Who Is Building This

I started a CS diploma in 2012 and dropped out in 2014-2015. I did not write a single line of code from that point until 2025 — a gap of more than ten years.

In 2025, I started building a programming language. Not a tutorial project. Not a toy. A compiled, statically typed language with its own type system, memory model, concurrency model, and standard library.

I am building it with Claude as my primary technical partner. Every chapter of the specification, every architectural decision, and the compiler itself is being built in deep collaboration with Claude and Claude Code. This is not casual AI-assisted coding — it is a structured, multi-year co-design process involving language theory, compiler engineering, and game development.

I am 34 years old. I am a solo developer. I have no VC funding and no team.

I am building Skev because the language I wanted to use for game development did not exist.

-----

## The Role of Claude

Without Claude, Skev would not exist.

That is not false modesty. It is the accurate description of how this project works. Claude is the technical partner that bridges the gap between what I understand and what the project requires. The language specification was co-designed through structured conversations. The compiler is being built using Claude Code. Every gap, every flag, every design decision has been discussed, challenged, and resolved in collaboration.

Skev is, among other things, a demonstration of what AI-assisted language development looks like in 2025-2026.

-----

## What Exists Today

Everything listed here is real and verifiable in this repository.

**Specification** — 11 chapters plus supplementary chapters. The full language design: type system, memory model (ARC), concurrency model (structured tasks), error handling (result[T]), generics, interoperability, tooling, and standard library. Complete.

**Python Transpiler** — A transpiler that converts Skev source to Python. 417 tests passing. This is the compliance suite that the compiler is validated against.

**LLVM Compiler** — Being built in Rust using LLVM as the backend. Phases A (lexer), B (parser), C (type checker), and D (code emitter) are complete. Phase E (ARC runtime) is ready to build.

**VS Code Extension** — Syntax highlighting and basic language support. Shipped.

The compiler is not finished. Skev cannot compile a full program yet. That is the honest state of the project. The specification is complete. The implementation is in progress.

-----

## What Skev Is Designed For

Skev targets two domains:

**Game development** — The primary domain. Skev has game-native types (Vector3!, Color!, Quat!), an entity/component model, when-handlers for event-driven game logic, and ARC memory management that eliminates GC pauses. A game written in Skev should be as fast as C++ and as readable as Python.

**Systems programming** — Compiled to native code via LLVM. ARC handles memory automatically with no garbage collector. Unsafe blocks are available for hardware-level work. The FFI layer supports direct C ABI interop at zero overhead.

-----

## How Skev Compares

There are other languages in this space. A developer considering Skev deserves an honest comparison.

**Odin** is a pragmatic, manually-managed language with C-like simplicity and no official package manager by design. It is stable, has a growing community, and is used commercially (JangaFX builds EmberGen in Odin). Skev differs in: ARC memory management instead of manual, enforced error handling via result[T], and a package system with content-addressed lock files.

**Jai** is Jonathan Blow’s language for game development, in closed beta as of May 2026. It is expected to open-source after Order of the Sinking Star ships in 2026. Jai’s signature feature is compile-time code execution (#run). Skev defers compile-time execution to v1.1, after the compiler is stable. Jai has no package manager.

**Zig** is a systems language designed as a better C, with manual memory management, compile-time execution, and strong C interop. TIOBE #39 as of April 2026. Approaching 1.0. Growing rapidly in indie game development. Zig is the most direct competition for Skev in the systems space.

**Where Skev is different from all three:**

- ARC memory management — not manual, not garbage collected. Deterministic, compiler-enforced.
- result[T] error handling — enforced by the compiler. Cannot be silently ignored.
- A package system with content-addressed lock files, provenance attestation, and supply chain security built in from day one.
- Designed for AI-assisted development — formal grammar, conformance test suite, Cursor rules, all planned for launch.

None of these is a claim that Skev is better than Odin, Jai, or Zig. They are different tradeoffs. Skev’s tradeoffs are made for developers who want memory safety without a borrow checker, error handling that the compiler enforces, and a package ecosystem with security guarantees from day one.

-----

## What Is Coming

The near-term roadmap in order:

1. **Phase E — ARC Runtime** — The last phase before the compiler can build real programs. Implementing the ARC runtime as a static C library.
1. **Standard Library** — 33 namespaces in pure Skev. No C underneath.
1. **Phase G — Testing** — 417 transpiler tests validated against the real compiler. LSP, DWARF5 debugger, error messages.
1. **v1.0** — A stable, cross-platform Skev compiler targeting Windows, macOS, and Linux.
1. **First game** — A SKEV mobile game built entirely in Skev. The design documentation is complete.

Longer-horizon items exist. They are not listed here because they do not have code yet.

-----

## Following the Project

The project is public on GitHub at [github.com/skev-lang](https://github.com/orgs/skev-lang/repositories).

Website: [skev.dev](https://skev.dev)

The compiler is being built openly. The specification is published openly. The design decisions are documented openly.

Skev is Apache 2.0. The specification, compiler, standard library, and tooling are all open source.

-----

*Skev. Fast like C++. Readable like Python.*
*Copyright © 2026 AJ. All Rights Reserved.*
