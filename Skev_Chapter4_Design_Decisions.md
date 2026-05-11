<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 4: Memory Model & Object Layout — Design Decisions
**Version:** 0.1
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked — Chapter 4 written
**Process:** Steps 1–7 completed per Design Practices v1.5
**Use Cases Tested:** All 7 — Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅

---

> This document records all design decisions made before Chapter 4 was written. Every decision here passed the Seven Question Test, was cross-checked against the Consistency Bible, tested across all use cases, and formally locked.

---

## Decision 1 — Entity Memory Layout

**Problem:** How should a Skev entity be laid out in RAM?

**Solution: 24-byte fixed header + properties + component data**

```
Offset  0: uint64_t arc_count       — atomic reference count
Offset  8: void*    dispatch_table  — pointer to type's event dispatch table
Offset 16: uint64_t component_mask  — bitmask of attached component IDs
Offset 24: [properties]             — compiler-reordered for alignment
Offset ?: [component data]          — inline (≤256b) or pointer (>256b)
```

**Key insight — shared dispatch table:**
Event dispatch table is ONE per entity TYPE — shared by all instances.
10,000 Dragon instances share ONE dispatch table. Massive memory saving.

**Five Question Test:** ✅ All five passed
**Use Cases:** ✅ Works for all 7 — verified pre-chapter
**Status:** Locked ✅

---

## Decision 2 — Stack vs Heap Rules

**Stack (automatic):**
```
→ int, float, bool — all primitives
→ All precision types (int8 through float64)
→ All game-native types (Vector3!, Color! etc)
→ kind types — uint32 discriminant only
→ data types ≤ 64 bytes — fits one cache line
→ array[T, N] — fixed size compile-time known
```

**Heap (ARC managed):**
```
→ entity instances
→ component instances
→ string values
→ list[T], map[K, V] contents
→ data types > 64 bytes — scoped heap (NOT ARC)
```

**64-byte threshold reasoning:** Modern CPU cache line = 64 bytes. Data fitting in one cache line stays fast on stack. Larger data is more efficient on heap.

**Status:** Locked ✅

---

## Decision 3 — ARC Implementation Detail

**Count increases:** new binding, function parameter, add to collection, assign to property
**Count decreases:** scope end, remove from collection, property overwrite, scene.destroy()
**Count zero:** freed immediately on calling thread — no pause

**Circular reference solution — `weak` keyword:**
```swift
weak T → non-owning reference
       → does NOT increment ARC count
       → always implies maybe — same access rules
       → auto-clears to nothing when referenced entity freed
       → compiler detects potential cycles and warns
```

**ARC thread safety:** All operations use C11 atomic instructions — `atomic_fetch_add` and `atomic_fetch_sub`. Thread-safe without mutex per entity. ~3-5ns cost per operation on modern hardware — acceptable.

**Status:** Locked ✅

---

## Decision 4 — Memory Alignment — Auto-Reordering

**Rule:** Compiler automatically reorders properties for optimal packing. Developer's declared order is always shown in Skev Studio debugger — memory order is invisible.

**Alignment requirements:**
```
bool, int8, uint8       → 1 byte
int16, uint16           → 2 bytes
int, uint32, float      → 4 bytes
int64, uint64, float64  → 8 bytes
string, entity pointers → 8 bytes
Vector3!, Vector4!      → 16 bytes (SIMD)
Quat!, Color!           → 16 bytes (SIMD)
Transform!              → 16 bytes (SIMD)
```

**Skev Studio:** Always shows properties in developer's declared order via a display table mapping declaration order to memory offset. Display table stripped from release builds.

**Status:** Locked ✅

---

## Decision 5 — No Runtime Reflection

**Decision:** Skev has NO runtime reflection.

**Reasons:**
- Reflection requires per-object metadata overhead
- Violates closed type system principle
- Games rarely need it
- ARIA AI assistant makes it unnecessary for dev tooling

**Replacement — Compile-time reflection (opt-in):**
```swift
#! introspectable
data Settings >>
    volume :: float = 0.5
<< Settings

info :: TypeInfo! = TypeInfo!.of(Settings)
loop prop in info.properties >>
    print(prop.name, prop.type_name, prop.default)
<< prop
```

**Rules:**
- `#! introspectable` works on `data` and `kind` only
- NEVER on `entity` — runtime state makes it meaningless
- Zero cost when not used
- Metadata stored in read-only `.rodata` segment

**Use case that required this decision:**
Animation/VFX tools and Interactive Apps need to inspect types for property panels and dynamic UI. This design covers those needs without burdening every entity with metadata overhead.

**Status:** Locked ✅

---

## Decision 6 — Component Memory Layout

**Rule:**
```
Component size ≤ 256 bytes → inline in entity body
Component size > 256 bytes → pointer to heap allocation
```

**256-byte threshold reasoning:** 4 CPU cache lines. Components fitting in 4 cache lines benefit from being adjacent to entity data in memory. Larger components exceed this benefit — pointer indirection is more efficient.

**Developer experience:** Completely invisible. `has Physics3D` works identically regardless of Physics3D's size. Compiler decides.

**Status:** Locked ✅

---

## Concern Resolutions

### Concern 1 — weak and maybe Interaction
**Resolution:** `weak` ALWAYS implies `maybe`. Writing `weak maybe T` is a compile error — redundant. Access requires `if value exists` check exactly like `maybe`.

### Concern 2 — ARC Thread Safety
**Resolution:** All ARC operations are atomic. Slight performance cost (~3-5ns) is acceptable. Creates formal dependency on Chapter 5 threading model.

### Concern 3 — Skev Studio Debugger Order
**Resolution:** Studio shows developer's declared order always. Memory reorder is invisible. Display table maps declaration order to memory offsets at compile time.

### Concern 4 — scene.destroy() Semantics
**Resolution:** destroy() marks entity as "destroying" (bit 63 of arc_count). Treated as nothing for all access. ARC still tracks. Freed when count reaches zero. Fires `destroyed` event before marking.

### Concern 5 — Large data on Heap
**Resolution:** data > 64 bytes uses scoped heap allocation — NOT ARC. Freed at block end (>> block close). Cannot escape scope — compiler prevents this. Faster than ARC for temporaries.

---

## Use Case Verification

| Decision | Games | XR/VR | Sim | Robotics | Anim/VFX | Interactive | Net |
|---|---|---|---|---|---|---|---|
| Entity layout | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Stack/heap rules | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ARC model | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Alignment | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| No runtime reflection | ✅ | ✅ | ✅ | ✅ | ⚠️ → fixed | ⚠️ → fixed | ✅ |
| Component layout | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

Animation/VFX and Interactive Apps required compile-time reflection addition — gap caught and resolved before chapter was written.

---

## Full Decision Summary

| Decision | Rule | Status |
|---|---|---|
| Entity header | 24 bytes: ARC + dispatch ptr + component mask | Locked ✅ |
| Dispatch table | Shared per entity TYPE — not instance | Locked ✅ |
| Stack threshold | data ≤ 64 bytes on stack | Locked ✅ |
| Heap threshold | data > 64 bytes on scoped heap | Locked ✅ |
| ARC atomicity | All operations atomic — thread safe | Locked ✅ |
| weak keyword | Non-owning — implies maybe — auto-clears | Locked ✅ |
| weak + maybe | weak always implies maybe — never combine | Locked ✅ |
| No runtime reflection | Closed type system — zero overhead | Locked ✅ |
| Compile-time reflection | #! introspectable — data and kind only | Locked ✅ |
| Auto-alignment | Compiler reorders — Studio shows dev order | Locked ✅ |
| Component inline | ≤ 256 bytes inline — > 256 bytes pointer | Locked ✅ |
| scene.destroy() | Marks destroying — fires event — ARC tracks | Locked ✅ |
| Scoped heap | Large data freed at block end — not ARC | Locked ✅ |

---

*All decisions locked — Chapter 4 written*
*Version 0.1 — Chapter 4 Pre-Design Decisions*
