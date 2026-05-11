<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 3.5: Generics — Design Decisions
**Version:** 0.1
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked — Chapter 3.5 written
**Process:** Steps 1–7 completed per Design Practices v2.6

---

## Decision 1 — Generic Syntax: `[T]` notation

`[T]` after function/type name — consistent with `list[T]`, `result[T]`, `map[K,V]`, `channel[T]`.

```swift
identity[T](value: T) -> T          # generic function
data Pair[A, B] >>                   # generic data type
alias Grid[T] = list[list[T]]        # generic alias
```

**Status: Locked ✅**

---

## Decision 2 — Constraints: `where T: Constraint`

```swift
find_max[T where T: Comparable](items: list[T]) -> maybe T
```

Built-in constraints: `Comparable`, `Hashable`, `Displayable`, `Numeric`, `Copyable`, `Serialisable`

**Status: Locked ✅**

---

## Decision 3 — What Is Forbidden

```
❌ entity Dragon[T]       — fixed ARC layout
❌ component Health[T]    — fixed ARC layout
❌ kind Option[T]         — no value storage in variants
❌ when event[T]()        — runtime dispatch incompatible
```

**Status: Locked ✅**

---

## Decision 4 — Type Inference

Inferred from argument types and variable declarations. Explicit `[T]` always valid.

**Status: Locked ✅**

---

## Decision 5 — ARC + Generics

Monomorphisation generates ARC-correct code per concrete type automatically. Developer writes nothing special.

**Status: Locked ✅**

---

## Decision 6 — Recursive Generic Types

Allowed. `data Tree[T] >> value :: T  left :: maybe Tree[T] << Tree`  ARC handles cleanup recursively.

**Status: Locked ✅**

---

## Decision 7 — Standard Library Unification

All existing `list[T]`, `map[K,V]`, `result[T]`, `channel[T]`, `array[T,N]` are retroactively unified under the same generic system. No special cases.

**Status: Locked ✅**

---

## Concern Resolutions

| Concern | Resolution |
|---|---|
| Recursive generic types | Allowed — ARC handles naturally |
| Generic functions returning generic types | Allowed — fully supported |
| Constraints on multiple parameters | Each gets own constraint. Type equality not in v1. |
| Generic type aliases | Allowed — `alias Grid[T] = list[list[T]]` |
| Compile time impact | No hard limit. Warning at >50 monomorphisations. |

---

## New Consistency Bible Entries

| Decision | Rule |
|---|---|
| Generic function | `name[T](params) -> ReturnType` |
| Generic data type | `data Name[T] >>` |
| Generic alias | `alias Name[T] = ExistingType[T]` |
| Constraint | `where T: Constraint` |
| Multiple params | `[A, B]` or with constraints |
| Entities/components | ❌ Cannot be generic |
| Kind types | ❌ Cannot be generic |
| Monomorphisation | One concrete version per type — zero runtime overhead |

---

## Rule C Gap Update

**Generics gap is now ✅ RESOLVED — the last critical gap.**

No critical gaps remain. All high and medium priority gaps are in Chapter 10 and 11.

---

*Version 0.1 — All decisions locked — Chapter 3.5 written*
