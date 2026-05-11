<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 3 Addendum: SOA (Structure of Arrays) — Design Decisions

**Version:** 1.0
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked
**Process:** Steps 1–7 completed per Design Practices
**Adds to:** Skev_Spec_Chapter3.md as Section 3.11

---

## The Problem SOA Solves

### Why AOS Hurts Performance

By default, Skev — like most languages — stores collections in
**Array of Structures (AOS)** layout:

```
# AOS memory layout for 4 particles:
[pos.x | pos.y | pos.z | vel.x | vel.y | vel.z | lifetime | active]  ← particle 0
[pos.x | pos.y | pos.z | vel.x | vel.y | vel.z | lifetime | active]  ← particle 1
[pos.x | pos.y | pos.z | vel.x | vel.y | vel.z | lifetime | active]  ← particle 2
[pos.x | pos.y | pos.z | vel.x | vel.y | vel.z | lifetime | active]  ← particle 3
```

When a loop updates only `position` across all particles, it must skip
over `velocity`, `lifetime`, and `active` in every iteration —
wasting cache lines and blocking SIMD.

**SOA (Structure of Arrays)** rearranges this:

```
# SOA memory layout for 4 particles:
[pos.x[0] | pos.x[1] | pos.x[2] | pos.x[3]]   ← all x together
[pos.y[0] | pos.y[1] | pos.y[2] | pos.y[3]]   ← all y together
[pos.z[0] | pos.z[1] | pos.z[2] | pos.z[3]]   ← all z together
[vel.x[0] | vel.x[1] | vel.x[2] | vel.x[3]]
[vel.y[0] | vel.y[1] | vel.y[2] | vel.y[3]]
[vel.z[0] | vel.z[1] | vel.z[2] | vel.z[3]]
[lifetime[0] | lifetime[1] | lifetime[2] | lifetime[3]]
[active[0] | active[1] | active[2] | active[3]]
```

A loop over positions now loads contiguous memory — SIMD can process
4-8 particles per instruction instead of 1.

**Measured impact in games:**
```
Particle system (10,000 particles):
  AOS update loop:  ~2.4ms per frame
  SOA update loop:  ~0.3ms per frame
  Speedup:           8x
```

---

## Decision 1 — SOA is a Collection Modifier, Not a Type Modifier

**Problem:** Where does `soa` live — on the type or on the collection?

**Options evaluated:**
```
Option A — on the type:
  soa data Particle >> ... << Particle
  particles :: list[Particle]   # implicit SOA because type is soa data

Option B — on the collection (CHOSEN):
  data Particle >> ... << Particle
  particles :: soa list[Particle]   # explicit SOA at declaration

Option C — auto-SOA compiler hint on the type:
  #! soa
  data Particle >> ... << Particle
```

**Decision: Option B — `soa` on the collection declaration.**

**Why Option B:**
```
→ The data type is neutral — it describes fields, not layout
→ The same type can be AOS in one context and SOA in another
→ Explicit at the point of use — programmer controls layout
→ Consistent with how Skev handles other modifiers (fixed, shared, hidden)
→ More flexible — hot-path code uses SOA, cold-path code uses AOS
```

**Why not Option A:**
```
→ Locks the type to one layout forever
→ Cannot use same type in both AOS and SOA contexts
→ Breaks data type neutrality
```

**Status:** Locked ✅

---

## Decision 2 — Syntax

**Locked syntax:**
```swift
# Declare a SOA collection:
field_name :: soa list[DataTypeName]

# Examples:
particles  :: soa list[Particle]
physics    :: soa list[PhysicsBody]
bones      :: soa list[BoneTransform]
joints     :: soa list[HandJoint]
```

**Access syntax — IDENTICAL to AOS:**
```swift
# These compile to identical Skev code regardless of AOS or SOA:
p :: Particle = particles[i]
particles[i].position += particles[i].velocity * delta
loop particle in particles >>
    particle.lifetime -= delta
<< particle
```

The programmer writes the same code. The compiler handles the memory
layout difference. This is the key ergonomic win.

**Status:** Locked ✅

---

## Decision 3 — SOA Only Applies to `data` Types

**Problem:** Can `soa` be used with `entity` types?

**Decision: NO. `soa list[EntityType]` is a compile error.**

**Why:**
```
Entities have:
  → ARC count (per-instance, must be individual)
  → Dispatch table pointer (per-instance)
  → Component mask (per-instance)
  → Identity (each entity is a unique object)

SOA requires:
  → Fields are contiguous arrays
  → No per-object metadata between fields
  → Elements are value types without identity

These are fundamentally incompatible.
```

**Compile error message:**
```
Error: `soa` cannot be used with entity types.
  soa list[Player]  ← Player is an entity, not a data type
  
  Use `soa list[PlayerData]` with a `data` type instead.
  Entity behavior can call into SOA data for performance-critical fields.
```

**Status:** Locked ✅

---

## Decision 4 — Game-Native Types in SOA Fields

**Problem:** `Vector3!` has x, y, z components. How are they laid out in SOA?

**Decision: Game-native types are decomposed to scalar arrays.**

```
data Particle >>
    position :: Vector3!   # decomposed to: pos_x[], pos_y[], pos_z[]
    lifetime :: float      # one array: lifetime[]
<< Particle

SOA layout:
  pos_x:    [x0 | x1 | x2 | x3 | ...]   ← 4 floats per SIMD lane
  pos_y:    [y0 | y1 | y2 | y3 | ...]
  pos_z:    [z0 | z1 | z2 | z3 | ...]
  lifetime: [l0 | l1 | l2 | l3 | ...]
```

This is optimal — SIMD operations on `pos_x` process 4 or 8 particles
simultaneously with no stride.

**Status:** Locked ✅

---

## Decision 5 — `maybe` Fields in SOA

**Decision: `maybe` fields in SOA data are valid.**

Layout: a parallel `bool` array (existence) + the value array.

```
data SensorReading >>
    value    :: float
    calibrated :: maybe float   # SOA: calibrated_exists[], calibrated_val[]
<< SensorReading

readings :: soa list[SensorReading]
```

**Status:** Locked ✅

---

## Decision 6 — Nested `data` in SOA

**Decision: Nested `data` types are flattened to leaf scalar fields.**

```
data Transform >>
    position :: Vector3!
    scale    :: float
<< Transform

data SceneNode >>
    transform :: Transform   # nested data
    active    :: bool
<< SceneNode

nodes :: soa list[SceneNode]

# SOA lays out as:
#   transform.position.x[]
#   transform.position.y[]
#   transform.position.z[]
#   transform.scale[]
#   active[]
```

**Status:** Locked ✅

---

## Decision 7 — Loop Auto-SIMD

**Decision: Loops over SOA collections auto-generate SIMD code.**

The programmer writes a scalar loop. The compiler recognises the SOA
layout and emits SIMD instructions automatically.

```swift
# Programmer writes:
loop particle in particles >>
    particle.position += particle.velocity * delta
    particle.lifetime -= delta
<< particle

# Compiler emits (conceptually):
# for each SIMD lane of 4 or 8:
#   load 4 pos_x values → SIMD register
#   load 4 vel_x values → SIMD register
#   add, multiply by delta → store
#   (repeat for y, z)
#   load 4 lifetime values → subtract → store
```

**Status:** Locked ✅

---

## Consistency Bible Additions

| Decision | Rule |
|----------|------|
| SOA syntax | `name :: soa list[DataType]` — modifier on collection |
| SOA restriction | Only `data` types — entities are compile error |
| SOA access | Identical to AOS — compiler handles layout |
| SOA + game-native | Vector3! decomposed to scalar arrays |
| SOA + maybe | Parallel bool array + value array |
| SOA + nested data | Flattened to leaf scalar fields |
| SOA keyword | `soa` — new modifier keyword, grammar group: modifiers |
| SIMD | Auto-generated for loops over SOA — no programmer action |

---

## Concern Resolutions

| Concern | Resolution |
|---------|------------|
| `soa` with entity | Compile error — entities have ARC/dispatch/identity |
| `maybe` in SOA | Valid — stored as parallel existence array |
| Nested data | Flattened automatically |
| Generic SOA | `soa list[T]` valid where T is a `data` type |
| Slice of SOA | Returns SOA view — same layout, subset of indices |
| SOA + map | Not supported — SOA applies to list and array only |
| `string` in SOA | Warning — strings are heap-allocated, defeats SOA benefit |
| Mix AOS + SOA | Same `data` type can be in both — no conflict |

---

*Copyright © 2026 AJ. All Rights Reserved. https://skev.dev | https://skev.org*
