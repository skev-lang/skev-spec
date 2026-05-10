<!--
Copyright © 2026 AJ. All Rights Reserved.
skev.dev | skev.org
-->

# Skev Language Specification

> "Fast like C++ and C#. Easy to read like Python."

Skev is a compiled, statically-typed programming language built for:
- Games & Interactive Entertainment
- XR / VR / AR
- Simulations
- Robotics & Real-time Systems

**skev.dev** | **skev.org**

---

## The Specification

The complete Skev language specification — all chapters are open and free to read.

| Chapter | Topic | Status |
|---------|-------|--------|
| Chapter 1  | Introduction & Philosophy | ✅ Complete |
| Chapter 2  | Syntax & Structure | ✅ Complete |
| Chapter 3  | Data Types | ✅ Complete |
| Chapter 3.2 | String Types | ✅ Complete |
| Chapter 3.5 | Generics | ✅ Complete |
| Chapter 3 (SOA) | Structure of Arrays | ✅ Complete |
| Chapter 3 (Overflow) | Integer Overflow | ✅ Complete |
| Chapter 4  | Memory Model (ARC) | ✅ Complete |
| Chapter 5  | Concurrency | ✅ Complete |
| Chapter 5 (DataRace) | Data Race Prevention | ✅ Complete |
| Chapter 6  | Error Handling | ✅ Complete |
| Chapter 7  | Standard Library | ✅ Complete |
| Chapter 8  | Interoperability (FFI) | ✅ Complete |
| Chapter 9  | Build System | ✅ Complete |
| Chapter 10 | Safety & Sandboxing | ✅ Complete |
| Chapter 11 | Tooling | ✅ Complete |

---

## Key Design Decisions

Every design decision is documented alongside the spec:

```
Skev_Chapter3_Design_Decisions.md   — why types work as they do
Skev_Chapter5_Design_Decisions.md   — concurrency model decisions
Skev_Chapter3_SOA_Design_Decisions.md   — SOA syntax decisions
...
```

---

## Quick Example

```swift
entity Player >>
    health   :: int   = 100
    position :: Vector3!
    alive    :: bool  = true

    has Physics

    when collision(other: Enemy)
        health -|= other.damage      # saturating: never below 0
        if health <= 0 >>
            alive = false
        << health <= 0
    << collision: Enemy

    heal(amount: int) -> nothing
        if alive >>
            health = math.clamp(health + amount, 0, 100)
        << alive
    << heal

<< Player
```

---

## Language Features

```
✅ ARC memory         No GC pauses. No manual free. No borrow checker.
✅ Game-native types  Vector3! Color! Transform! as language primitives.
✅ Entity system      entity and when are keywords, not patterns.
✅ result[T]          Compiler-enforced error handling.
✅ maybe T            Null safety — null pointer crashes impossible.
✅ SOA support        soa list[Particle] for cache-friendly game loops.
✅ Overflow control   +% +| +! — choose wrapping, saturating, or checked.
```

---

## Roadmap

```
✅ Milestone 1   Specification complete
✅ Milestone 2   Python transpiler (417 tests passing)
✅ Milestone 2.5 Pre-compiler work
⬜ Milestone 3   Real compiler (Rust + LLVM)
⬜ Milestone 4   Standard library + first game
⬜ Milestone 5   Skev Studio IDE
⬜ Milestone 6   Self-hosting
```

---

## Compiler

The Python transpiler is at:
[github.com/skev-lang/skev-transpiler](https://github.com/skev-lang/skev-transpiler)

The real compiler (Milestone 3) will be at:
[github.com/skev-lang/skev](https://github.com/skev-lang/skev)

---

## Copyright

Copyright © 2026 AJ. All Rights Reserved.

The Skev specification is open to read. The compiler and standard library
are licensed under Apache 2.0.

https://skev.dev | https://skev.org
