<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 6: Error Handling — Design Decisions
**Version:** 0.1
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked — Chapter 6 written
**Process:** Steps 1–7 completed per Design Practices v1.7
**Use Cases Tested:** All 7
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅

---

## Core Architecture Decision

**Three paths evaluated:**

| Path | Model | Fast? | Readable? | Verdict |
|---|---|---|---|---|
| A | Exceptions (C#, Python) | ⚠️ Stack unwind cost | ⚠️ Hidden control flow | ❌ |
| B | Error codes (C) | ✅ | ❌ Verbose, ignorable | ❌ |
| C | Result types (Rust, Swift) | ✅ Zero-cost success | ✅ If designed well | ✅ |

**Decision: Result types — game-specific design to maintain readability.**

---

## Decision 1 — `result[T]` Type

```
result[LevelData]    success: LevelData, failure: error
result[int]          success: int,       failure: error
result[nothing]      success: no value,  failure: error
```

Uses `[]` notation — consistent with `list[T]`, `map[K,V]`. **Status: Locked ✅**

---

## Decision 2 — Error Types Use `kind` and `data`

No new `error` keyword. Errors are values — they use existing type system.

```
kind  → error categories (variants)
data  → error context (structured details)
```

Reuses existing type system — Practice 8 (Minimal Surface Area) maintained. **Status: Locked ✅**

---

## Decision 3 — `->` Propagation Operator

Reuses existing `->` arrow from match arms. New meaning in assignment context.

```swift
value :: T = -> operation()
```

If fail: stop function, return failure, ARC cleans up.
If succeed: unwrap value, continue.

**NOT valid in `when` blocks** — must use `async` function or `match` inline. **Status: Locked ✅**

---

## Decision 4 — Three Handling Patterns

```
match   → full control — all cases explicitly handled
or_else → fallback value or alternative operation
if succeed / if fail → single outcome checks
```

All three compile to exhaustive handling — compiler enforces. **Status: Locked ✅**

---

## Decision 5 — Panic for Programming Errors

```
Panics:  unrecoverable — cannot be caught — fix the code
assert:  developer invariant — panics if violated
```

Development: full diagnostic, halt, debugger.
Release: graceful exit via `engine.on_panic` hook.

**Status: Locked ✅**

---

## Decision 6 — Error Propagation Through Async and Tasks

`async` functions return `result[T]` — `->` works inside them.
Tasks use `task.result` — checked with `if compute_task succeed/fail`.
ARC cleanup on `->` propagation is automatic — identical to normal scope exit.

**Status: Locked ✅**

---

## Concern Resolutions

| Concern | Resolution |
|---|---|
| Multiple error kinds from different systems | Error unions inferred automatically by compiler from `->` propagation |
| `->` in `when` blocks | Not permitted — must call async function or use inline match |
| Panic release hook | `engine.on_panic >> info` — called before graceful exit |
| Error messages | `kind` for simple categories, `data` for rich context |
| Context for kind errors | `fail ErrorKind.variant with key: value` inline context |

---

## New Keywords Added to Consistency Bible

| Keyword | Purpose |
|---|---|
| `succeed` | Creates successful result value |
| `fail` | Creates failed result value |
| `or_else` | Fallback for failed result |
| `assert` | Developer invariant — panic if violated |
| `result[T]` | Fallible operation return type |
| `->` (assignment) | Error propagation — extends existing `->` |

---

## Use Case Verification

| Use Case | result[T] | Panics | Async errors |
|---|---|---|---|
| Games | ✅ Save/load, asset loading | ✅ Array bounds, invariants | ✅ Level loading |
| XR/VR | ✅ Content loading | ✅ Hardware invariants | ✅ Asset streaming |
| Simulation | ✅ Data parsing | ✅ Numerical invariants | ✅ Data import |
| Robotics | ✅ Command validation | ✅ Safety invariants | ✅ Hardware comms |
| Animation/VFX | ✅ File export | ✅ Render invariants | ✅ Asset pipeline |
| Interactive Apps | ✅ API calls, file I/O | ✅ UI invariants | ✅ Network requests |
| Real-time Net | ✅ Packet handling | ✅ Protocol invariants | ✅ Connection management |

---

## Rule C Gap Update

Error Handling gap is now **RESOLVED**. Removing from critical gaps.

---

*All decisions locked — Chapter 6 written*
*Version 0.1*
