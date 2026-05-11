<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 7: Standard Library — Design Decisions
**Version:** 0.1
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked — Chapter 7 written
**Process:** Steps 1–8 completed per Design Practices v2.1
**Rule H Applied:** skev.network (not skev.net) ✅

---

## Decision 1 — Three-Layer Architecture

```
Layer 1: Language primitives — always available, zero import
Layer 2: Core library — import skev.core
Layer 3: Domain libraries — import skev.game, skev.robotics, etc.
```

Rationale: C++ too explicit, Python too implicit. Skev balances both. **Status: Locked ✅**

---

## Decision 2 — String Operations

- Interpolation: `{expression}` inside string literals
- Literal brace: `{{` and `}}`
- Methods: snake_case — `.upper()`, `.lower()`, `.trim()`, `.contains()`, etc.
- Expressions in `{}` are type-checked at compile time

**Status: Locked ✅**

---

## Decision 3 — Math Library (`skev.math`)

Key game-specific additions missing from other standard libraries:
- `math.clamp()` — missing from Python standard library
- `math.lerp()` — missing from Python, C++ pre-C++20
- `math.smoothstep()` — missing from all three comparison languages
- `math.map()` — missing from all three comparison languages
- `math.noise()` — missing from all three comparison languages

**Status: Locked ✅**

---

## Decision 4 — Time Library (`skev.time`)

- `timer(seconds) >> ... << timer` — one-shot timer as first-class block
- `every(seconds) >> ... << every` — repeating timer as first-class block
- Both main-thread only — compiler enforces (no timer inside task blocks)
- `Stopwatch` type for performance measurement

**Status: Locked ✅**

---

## Decision 5 — File I/O (`skev.file`)

- All operations async by default — game loop never blocks
- All fallible operations return `result[T]` — consistent with Ch6
- `file.path()` normalises separators automatically
- Forward slash always works regardless of platform

**Status: Locked ✅**

---

## Decision 6 — Serialisation

- `skev.json` — human-readable (config, debug)
- `skev.serial` — binary efficient (production saves, network)
- `#! serialisable` annotation on `data` types — automatic
- Entity references NOT serialisable — use stable IDs instead
- Both return `result[T]` — consistent with Ch6

**Status: Locked ✅**

---

## Decision 7 — `set[T]` Type

Fills the gap identified in the architecture audit.
- Unordered unique values
- O(1) add, remove, contains
- Set mathematics: union, intersect, minus
- Element must be Hashable — compiler enforces

**Status: Locked ✅**

---

## Decision 8 — Networking (`skev.network`)

**Rule H applied:** `skev.net` renamed to `skev.network`
- Reason: `net` evokes Microsoft's .NET platform
- `skev.network` is unambiguous

Covers: HTTP GET/POST, WebSocket, result-based errors

**Status: Locked ✅**

---

## Concern Resolutions

| Concern | Resolution |
|---|---|
| Platform file path separators | Normalised automatically — always use `/` |
| `{}` conflict with existing syntax | No conflict — `{}` not used elsewhere in Skev |
| Serialising entity references | Compiler error — use stable IDs |
| `timer`/`every` inside tasks | Compiler error — main thread only |
| Standard library versioning | Chapter 9 scope — flagged as medium priority |

---

## New to Consistency Bible

| Decision | Rule |
|---|---|
| String interpolation | `{expression}` in string literals |
| Literal brace | `{{` and `}}` |
| String methods | snake_case — `.upper()`, `.lower()`, etc. |
| timer block | `timer(seconds) >> ... << timer` — main thread only |
| every block | `every(seconds) >> ... << every` — main thread only |
| file operations | Always async — return `result[T]` |
| File path separators | Normalised automatically |
| #! serialisable | Marks data for serialisation — no entity refs |
| set[T] | Unordered unique — element must be Hashable |
| skev.network | Not skev.net — Rule H applied |

---

## Rule C Gap Update

Standard Library gap is now **RESOLVED**. Removing from high-priority gaps.

Remaining critical gaps:
- Generics — Chapter 3.5
- Interoperability/FFI — Chapter 8

---

*All decisions locked — Chapter 7 written*
*Version 0.1*
