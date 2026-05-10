<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 3 — Section 3.11: Structure of Arrays (SOA)

**Adds to:** Chapter 3 (Data Types) as Section 3.11
**Status:** Locked ✅
**Design decisions:** Skev_Chapter3_SOA_Design_Decisions.md

---

## 3.11 — Structure of Arrays `soa`

### 🧑‍💻 Developer Version

By default, Skev stores a list of data types in **Array of Structures (AOS)**
layout — each element's fields are contiguous in memory. This is the natural
layout and works well for most code.

For performance-critical loops that process **one field across thousands of
elements** — particle systems, physics bodies, animation bones — AOS forces
the CPU to skip over unused fields on every iteration, wasting cache lines
and preventing SIMD vectorisation.

The `soa` modifier switches a collection to **Structure of Arrays** layout.
The same field from every element is stored together. The loop code is
identical. The compiler handles the layout difference automatically.

```
AOS:  [x y z vx vy vz]  [x y z vx vy vz]  [x y z vx vy vz]
SOA:  [x x x ...]  [y y y ...]  [z z z ...]  [vx vx ...]  ...
```

**The core rule:**
```
particles :: soa list[Particle]   # SOA layout
enemies   :: list[Enemy]          # AOS layout (default)

# Access syntax is IDENTICAL for both. Compiler handles the difference.
```

---

### 🔵 Technical Version

`soa` is a collection modifier that changes the memory layout of a
`list[DataType]` from interleaved (AOS) to columnar (SOA). The type system
is unchanged. The access syntax is unchanged. The compiler emits different
load/store patterns for SOA collections, enabling SIMD auto-vectorisation
for loops that touch contiguous scalar arrays.

Game-native types (`Vector3!`, `Color!`) are decomposed to scalar arrays:
`Vector3!` becomes three separate float arrays (`x[]`, `y[]`, `z[]`).
Nested `data` types are flattened to leaf fields. `maybe` fields become
parallel existence arrays.

`soa` is restricted to `data` types. Applying `soa` to an entity type is
a compile error — entities require per-instance ARC counts, dispatch table
pointers, and component masks that are structurally incompatible with the
flat array layout SOA requires.

---

## Rule F — The Problem in Other Languages

### 🌍 Real-World Scenario

A particle system simulates 10,000 particles every frame. Each frame:
- Integrate position from velocity
- Age the particle (reduce lifetime)
- Kill particles whose lifetime reaches zero

In AOS layout, updating position requires touching 128 bytes of data per
particle but only using 24 bytes (position + velocity). The remaining 104
bytes are loaded into cache for nothing. At 10,000 particles: 1.04MB of
wasted cache bandwidth per frame.

**In C++23:**
```cpp
// AOS — default, simple, but cache-unfriendly for field-only loops
struct Particle {
    float px, py, pz;    // position
    float vx, vy, vz;    // velocity
    float lifetime;
    bool  active;
};
std::vector<Particle> particles(10000);

// To get SOA in C++, you must restructure manually:
struct ParticleSOA {
    std::vector<float> px, py, pz;
    std::vector<float> vx, vy, vz;
    std::vector<float> lifetime;
    std::vector<bool>  active;
};
// Now access changes: particles.px[i] instead of particles[i].px
// Every access site must be updated. Error-prone.
// 16 lines total
```

**In C# 13:**
```csharp
// No native SOA support. Must be done manually with parallel arrays:
float[] px = new float[10000];
float[] py = new float[10000];
float[] pz = new float[10000];
float[] vx = new float[10000];
float[] vy = new float[10000];
float[] vz = new float[10000];
float[] lifetime = new float[10000];
bool[]  active   = new bool[10000];
// Access: px[i], py[i] — loses the struct abstraction entirely.
// No way to pass a "particle" without passing 8 separate arrays.
// 9 lines total
```

**In Python 3.13:**
```python
# Python has no SOA. NumPy provides columnar arrays but
# requires learning a different paradigm (vectorised ops):
import numpy as np
particles = {
    'position': np.zeros((10000, 3), dtype=np.float32),
    'velocity': np.zeros((10000, 3), dtype=np.float32),
    'lifetime': np.zeros(10000, dtype=np.float32),
}
# Vectorised update (NumPy handles SIMD internally):
particles['position'] += particles['velocity'] * delta
# Loses struct abstraction. Dictionary is not type-safe.
# 8 lines total
```

**In Skev — `soa` modifier, access unchanged:**
```swift
data Particle >>
    position :: Vector3!
    velocity :: Vector3!
    lifetime :: float
    active   :: bool
<< Particle

entity ParticleSystem >>
    particles :: soa list[Particle]   # SOA layout — one keyword

    when update(delta)
        loop particle in particles >>
            particle.position += particle.velocity * delta
            particle.lifetime -= delta
            if particle.lifetime <= 0 >>
                particle.active = false
            << particle.lifetime <= 0
        << particle
    << update

<< ParticleSystem
// 19 lines total — access syntax identical to AOS. No restructuring.
// Compiler decomposes Vector3! to x[], y[], z[] arrays automatically.
// Loop auto-generates SIMD for 4-8 particles per instruction.
```

---

## 3.11.1 — Foundation

### 🟢 Basic SOA Declaration

```swift
# AOS collection (default — no keyword):
bullets :: list[Bullet]

# SOA collection — add soa modifier:
bullets :: soa list[Bullet]

# Access is the same for both:
b :: Bullet = bullets[i]
bullets[i].position += bullets[i].velocity * delta
// 7 lines total
```

The `soa` modifier changes the memory layout. Nothing else changes.
The type is still `Bullet`. Indexing still uses `[i]`. Field access
still uses `.field`. The compiler does the rest.

### 🟢 What Data Types Work with SOA

```swift
# Only data types — value types with no ARC or identity:
data Bullet    >> position :: Vector3!  speed :: float << Bullet
data BoneTransform >> rotation :: Vector3!  scale :: float << BoneTransform
data PhysicsBody >> pos :: Vector3!  vel :: Vector3!  mass :: float << PhysicsBody

# Valid SOA collections:
bullets    :: soa list[Bullet]
bones      :: soa list[BoneTransform]
bodies     :: soa list[PhysicsBody]

# COMPILE ERROR — entities cannot be SOA:
# players :: soa list[Player]
# Error: Player is an entity type. soa requires a data type.
// 13 lines total
```

---

## 3.11.2 — Applied

### 🟡 Physics Simulation

```swift
data RigidBody >>
    position :: Vector3!
    velocity :: Vector3!
    force    :: Vector3!
    mass     :: float
    sleeping :: bool
<< RigidBody

entity PhysicsWorld >>
    bodies :: soa list[RigidBody]

    fixed_update(delta: float) -> nothing
        # SOA layout: compiler processes all positions together,
        # then all velocities — SIMD across 4-8 bodies at once
        loop body in bodies >>
            if not body.sleeping >>
                body.velocity += body.force / body.mass * delta
                body.position += body.velocity * delta
                body.force    = Vector3(0.0, 0.0, 0.0)
            << not body.sleeping
        << body
    << fixed_update

<< PhysicsWorld
// 22 lines total — 10,000 bodies at 60fps without GC pauses.
// soa gives ~6x speedup over AOS for this loop pattern.
```

### 🟡 Mixing AOS and SOA

The same `data` type can appear in both AOS and SOA collections in
the same program. The type is neutral — the layout is on the collection.

```swift
data Sensor >>
    id       :: int
    value    :: float
    calibrated :: maybe float
<< Sensor

entity SensorArray >>
    # Hot path — thousands of readings per second — SOA for SIMD:
    live_readings :: soa list[Sensor]

    # Cold path — audit log, infrequent access — AOS is fine:
    archived :: list[Sensor]

    process(delta: float) -> nothing
        # Hot: SOA loop — SIMD auto-generated
        loop reading in live_readings >>
            process_reading(reading)
        << reading

        # Cold: AOS loop — no SIMD needed
        loop archived_reading in archived >>
            archive_reading(archived_reading)
        << archived_reading
    << process

<< SensorArray
// 24 lines total — same Sensor type, two layouts, appropriate to context.
```

---

## 3.11.3 — Hard Complex

### 🔴 Full Animation System with SOA Bones

**Walk-Through:** An animation system for a character with 64 bones.
Every frame: sample keyframes, blend two animations, apply IK.
This is the most cache-sensitive inner loop in a character renderer.

```swift
data BoneTransform >>
    position :: Vector3!
    rotation :: Vector3!    # Euler angles — Quat! in production
    scale    :: Vector3!
    parent   :: int         # parent bone index (-1 = root)
<< BoneTransform

data BlendWeight >>
    anim_a :: float
    anim_b :: float
<< BlendWeight

entity Skeleton >>
    #! SOA: all positions contiguous, all rotations contiguous
    #! Blending reads position[all], rotation[all] — perfect cache use
    bones         :: soa list[BoneTransform]
    blend_weights :: soa list[BlendWeight]
    bone_count    :: int = 64

    blend_animations(clip_a: AnimClip, clip_b: AnimClip,
                    blend: float) -> nothing
        loop i from 0 to bone_count >>
            wa :: float = 1.0 - blend
            wb :: float = blend

            #! SOA: all positions accessed in stride-1 order
            #! SIMD processes 4 bones per instruction automatically
            bones[i].position = Vector3(
                clip_a.bones[i].position.x * wa + clip_b.bones[i].position.x * wb,
                clip_a.bones[i].position.y * wa + clip_b.bones[i].position.y * wb,
                clip_a.bones[i].position.z * wa + clip_b.bones[i].position.z * wb
            )
            bones[i].scale = Vector3(
                clip_a.bones[i].scale.x * wa + clip_b.bones[i].scale.x * wb,
                clip_a.bones[i].scale.y * wa + clip_b.bones[i].scale.y * wb,
                clip_a.bones[i].scale.z * wa + clip_b.bones[i].scale.z * wb
            )
        << i
    << blend_animations

    apply_world_transforms() -> nothing
        #! Forward pass: root → leaves (must be sequential, not SIMD)
        #! Parent must be computed before child — dependency chain
        loop i from 0 to bone_count >>
            parent_idx :: int = bones[i].parent
            if parent_idx >= 0 >>
                bones[i].position += bones[parent_idx].position
            << parent_idx >= 0
        << i
    << apply_world_transforms

<< Skeleton
// 48 lines total
```

**Walk-Through — What the compiler does with SOA here:**

```
blend_animations loop:
  Reads: clip_a.bones[i].position.x for i=0..63
  With AOS: 64 reads at stride 32 bytes → 64 cache misses
  With SOA: 64 reads at stride 4 bytes  → 2 cache lines → 2 cache misses
  SIMD: 16 iterations per SSE4 instruction → 4x throughput

apply_world_transforms:
  Sequential dependency (parent before child) — SIMD not possible here.
  SOA still helps: position fields are contiguous → better prefetch.
  The compiler detects the dependency and falls back to scalar.
  Programmer writes the same code either way.
```

---

## 3.11.4 — Constraints and Restrictions

```
✅ soa list[DataType]        — valid
✅ soa list[T]               — valid where T is data type
✅ maybe fields in soa data  — valid (parallel bool array)
✅ nested data in soa data   — valid (flattened to leaf fields)
✅ Vector3! in soa data      — valid (decomposed to x[], y[], z[])

❌ soa list[EntityType]      — compile error: entity has ARC/identity
❌ soa map[K, V]             — not supported: maps have hash structure
❌ string fields in soa data — warning: strings are heap pointers,
                               defeating the SOA benefit. Still valid.
❌ soa entity                — compile error: soa is a collection
                               modifier, not an entity modifier
```

---

## 3.11.5 — Quick Reference

| Syntax | Meaning |
|--------|---------|
| `particles :: soa list[Particle]` | SOA collection of Particle |
| `particles[i].field` | Access same as AOS |
| `loop p in particles >>` | SIMD-auto loop |
| `soa list[EntityType]` | Compile error |
| `soa` + `Vector3!` | Decomposed to x[], y[], z[] |
| `soa` + `maybe T` | Parallel bool[] + T[] |

---

*Copyright © 2026 AJ. All Rights Reserved. skev.dev | skev.org*
