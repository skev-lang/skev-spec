<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 1: Philosophy & Goals
**Version:** 0.1 — DRAFT  
**Authors:** AJ (Copyright © 2026)
**Status:** In Progress

---

## 1.1 — Mission Statement

### 👨‍💻 Developer Version

> **Skev exists because game developers deserve better.**

For decades, game developers have chosen between two painful options: write fast code in difficult languages like C++, or write clean code in slower languages like C#. Skev eliminates this choice entirely.

Skev is the first language designed from the ground up where performance and clarity are not opposites — they are the same thing.

### ⚙️ Technical Version

Skev is a statically typed, natively compiled, general-purpose language with primary optimization for real-time interactive systems. Skev compiles via LLVM to native machine code achieving performance within 5–10% of equivalent C++ implementations, while maintaining syntax complexity closer to high-level interpreted languages. Memory is managed via Automatic Reference Counting with zero garbage collection overhead.

---

## 1.2 — Core Design Principles

> **These principles are NON-NEGOTIABLE.** Every future decision made about Skev must be measured against these principles. If a feature violates even one of them, it does not enter the language.

---

### Principle 1 — Performance Is Never Optional

#### 👨‍💻 Developer Version

Skev will never introduce a feature that compromises runtime performance without explicit developer consent.

**What this means for you:**
- No hidden garbage collection ever
- No invisible runtime overhead
- No performance surprises
- What you write is what runs

#### ⚙️ Technical Version

Skev runtime introduces zero implicit heap allocations. All memory operations are deterministic and visible at the syntax level. ARC operations are compiler-optimized and elided where static analysis proves safety. LLVM optimization passes run at O2 minimum for all release builds.

---

### Principle 2 — Clarity Is Not Negotiable

#### 👨‍💻 Developer Version

Skev code should read like a description of what the game does — not a puzzle for the developer to solve.

**What this means for you:**
- No cryptic symbols without meaning
- Every keyword earns its place
- A developer unfamiliar with Skev should understand your code's intent within seconds of reading it

#### ⚙️ Technical Version

Skev enforces mandatory whitespace rules around all binary operators. Token design prioritizes unambiguous lexical parsing. No operator overloading permitted outside of game-native type primitives. Syntax grammar is designed for single-pass top-down parsing with zero ambiguity.

---

### Principle 3 — Games Are First Class

#### 👨‍💻 Developer Version

Skev is not a general language that supports games. Skev is a game language that supports general use.

**What this means for you:**
- `Vector3`, `Color`, `Transform` are primitives like `int` and `float` — not imported types
- Entity and component patterns are native language concepts
- Event systems are built into the language itself
- Physics, audio, and input are standard library citizens — not afterthoughts

#### ⚙️ Technical Version

Skev type system includes a dedicated game-native primitive category denoted by the `!` suffix. These types receive specialized LLVM IR generation with SIMD optimization where hardware supports it. Vector math operations are auto-vectorized. Transform operations compile to optimal matrix math without developer intervention.

---

### Principle 4 — Errors Should Be Impossible To Ignore

#### 👨‍💻 Developer Version

Skev treats silence as dangerous. If something could go wrong — Skev tells you at compile time, not at 3am when your game crashes in production.

**What this means for you:**
- Most bugs caught before you run anything
- Compiler messages written in plain English
- Compiler tells you what's wrong AND suggests how to fix it
- Runtime crashes are rare by design

#### ⚙️ Technical Version

Skev implements exhaustive pattern matching requirements, null safety via Option types, mandatory error handling for fallible operations, and strict type checking with zero implicit coercion between numeric types. Compiler diagnostic messages include error code, plain language description, affected source location, and suggested remediation where statically determinable.

---

### Principle 5 — Simplicity Scales

#### 👨‍💻 Developer Version

Skev should feel simple to a beginner on day one and powerful to an expert on year five. The same language. The same syntax. Different depths.

**What this means for you:**
- Beginners write working games quickly
- Experts unlock advanced features without switching languages
- No "beginner Skev" vs "expert Skev"
- Your skills compound over time

#### ⚙️ Technical Version

Skev implements progressive complexity through optional type annotations, optional manual memory hints, optional low-level unsafe blocks, and optional compiler optimization directives. All advanced features are strictly additive — removing them produces valid, performant Skev code. Beginner and expert code are parsed by identical grammar rules.

---

### Principle 6 — Skev Belongs To Its Community

#### 👨‍💻 Developer Version

Skev is open source. The compiler, specification, and standard library are all publicly visible. You can read every line of the compiler that runs your game. You can verify every safety claim. You can contribute bug fixes and improvements.

At the same time — Skev has clear direction. AJ owns the language vision. This is not a contradiction. It is the model that works: Swift (Apple), Kotlin (JetBrains), Go (Google), TypeScript (Microsoft). Open code. Clear leadership. Fast decisions.

**What this means for you:**
- Compiler is open source — every line visible and auditable
- Specification is public — all design decisions documented
- Standard library is open source — free forever
- Skev Studio Community Edition is free — full language access
- AJ controls direction — no design-by-committee slowdowns
- If Skev is ever abandoned — community inherits it fully

**What is commercial:**
- Skev Studio Pro — advanced profiler, team tools, cloud build
- Skev Package Hub private tier — private packages for studios
- Skev Console Toolchain — console platform development

**The guarantee:**
A developer can use Skev fully — for any of the 7 target domains — without ever paying for anything. The commercial tier adds convenience and power. It never gates the language itself.

#### ⚙️ Technical Version

Skev compiler, specification, and standard library are published under Apache 2.0 license. AJ controls the language specification — contributions are welcome but final decisions rest with the project lead. The edition system (see Chapter 9) governs backward compatibility — no breaking change ships without a migration path. A governance backstop clause in the license provides: if the Skev Language Foundation ceases maintenance for 24 consecutive months, all assets automatically relicense to Apache 2.0 with full community governance. This ensures the community can never be stranded regardless of what happens to the controlling entity.

---

## 1.3 — What Skev Is NOT

This is equally important as what Skev is.

### 👨‍💻 Developer Version

**Skev is NOT:**
- A web development language
- A data science language
- A replacement for SQL
- A shell scripting language
- An attempt to do everything

Skev does fewer things than most languages. But the things it does — it does better than anything else that has ever existed.

### ⚙️ Technical Version

**Skev explicitly excludes:**
- DOM manipulation primitives
- Tensor/matrix computation libraries in standard library
- Interpreted execution mode
- Dynamic typing or gradual typing
- Reflection or runtime type modification
- Eval or runtime code generation

These exclusions are permanent specification decisions — not implementation deferrals.

---

## 1.4 — The Skev Promise

> We promise your code will run fast.

> We promise your code will be readable.

> We promise the language will never be taken away or locked behind a paywall.

> We promise your feedback shapes what Skev becomes.

---

**Skev is not a tool you use.**  
**Skev is a language you belong to.**

---

*End of Chapter 1 — Next: Chapter 2: Syntax & Structure*
