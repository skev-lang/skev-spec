<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 4: Memory Model & Object Layout
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026)
**Status:** In Progress
**Depends On:** Chapter 3 v0.1 — All type decisions locked
**Use Cases Tested:** Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅

---

## 4.1 — Overview

### 👨‍💻 Developer Version

Chapter 4 is the engine room of Skev. Everything in this chapter happens automatically — you never directly interact with any of it. But understanding it explains why Skev is fast, safe, and predictable in ways that C++ and C# cannot match simultaneously.

Three principles govern everything in this chapter:

> **Principle 1: What you cannot see cannot surprise you.**
> Memory management is invisible. It just works.

> **Principle 2: Deterministic is better than automatic.**
> Memory is freed exactly when it should be — not whenever the runtime feels like it.

> **Principle 3: The compiler works harder so you don't have to.**
> Every layout decision, every alignment, every reference count — the compiler handles it all.

### ⚙️ Technical Version

Chapter 4 specifies the complete memory model for Skev — covering object layout in RAM, stack and heap allocation rules, Automatic Reference Counting implementation, memory alignment strategy, component storage, reflection capabilities, and the semantics of entity destruction. All decisions are verified against Skev's core identity: fast like C++ and C#, easy to read like Python. All decisions are verified against all seven of Skev's target use cases.

---

## 4.2 — Entity Memory Layout

### 👨‍💻 Developer Version

Every entity in Skev has a precise, predictable layout in memory. You never see this layout directly — but it determines how fast your game runs and how safely memory is managed.

A Skev entity in memory has four sections:

```
Skev Entity in RAM:
┌─────────────────────────────────────────┐
│  ARC Count            8 bytes           │  How many things reference this entity
│  Event Table Pointer  8 bytes           │  Points to this entity type's events
│  Component Mask       8 bytes           │  Which components are attached
│  Properties           variable          │  The entity's data — health, speed etc
│  Component Data       variable          │  Data from attached components
└─────────────────────────────────────────┘
```

**What each section does:**

```swift
entity Dragon >>
    health     :: 1000        # ← stored in Properties section
    move_speed :: 3.5         # ← stored in Properties section
    position   :: Vector3!    # ← stored in Properties section

    has Physics3D             # ← Physics3D data stored in Component Data
    has Animator              # ← Animator data stored in Component Data
    has AudioSource           # ← AudioSource data stored in Component Data

<< Dragon
```

**The Event Table is shared — this is important:**

```
If your game has 10,000 Dragon enemies:

WITHOUT shared event table:
→ 10,000 Dragons × event table = enormous memory waste

WITH shared event table (Skev's approach):
→ ONE event table per Dragon TYPE
→ 10,000 Dragons all point to the SAME table
→ Massive memory saving
→ Better CPU cache usage
→ Faster dispatch

Event Dispatch Table (one per entity TYPE — not per instance):
┌─────────────────────────────────────────┐
│  update pointer                         │
│  collision pointer                      │
│  destroyed pointer                      │
│  health_changed pointer                 │
│  custom event pointers...               │
└─────────────────────────────────────────┘
```

### ⚙️ Technical Version

The Skev entity header is a fixed 24-byte structure:

```
Offset  0: uint64_t arc_count       — atomic reference count
Offset  8: void*    dispatch_table  — pointer to type's event dispatch table
Offset 16: uint64_t component_mask  — bitmask of attached component IDs
```swift

Properties follow the header, reordered by the compiler for optimal alignment (see section 4.5). Component data follows properties — inline for components ≤ 256 bytes, pointer for components > 256 bytes (see section 4.7).

The event dispatch table is allocated once per entity type at program initialisation and stored in read-only memory. It is shared by all instances of that type — a pointer indirection of one level. This matches C++ vtable semantics in cost but differs in that Skev's dispatch table is keyed by event name, not by virtual function slot index, enabling O(1) named event lookup.

The component mask is a 64-bit bitmask supporting up to 64 distinct component types per entity. The compiler assigns each component type a unique bit position at compile time. Component presence check is a single bitwise AND operation — O(1).

---

## 4.3 — Stack vs Heap Allocation

### 👨‍💻 Developer Version

Skev decides automatically where each value lives in memory. You never write `new` or `malloc` or `alloc`. The compiler figures it out. Here are the rules it follows:

**Stack — fast, automatic, size known at compile time:**

```swift
entity Player >>
    health      :: int          # stack — primitive
    speed       :: float        # stack — primitive
    alive       :: bool         # stack — primitive
    position    :: Vector3!     # stack — game-native type
    damage_type :: DamageType   # stack — kind (just a number)
    config      :: WeaponConfig # stack — data type ≤ 64 bytes
    matrix      :: array[float, 16]  # stack — fixed array

<< Player
```

**Heap (ARC managed) — flexible, tracked, reference counted:**

```swift
entity Player >>
    name        :: string          # heap — ARC managed
    inventory   :: list[string]    # heap — ARC managed
    target      :: maybe Enemy     # heap — ARC managed reference
    stats_map   :: map[string, int] # heap — ARC managed

<< Player
```

**The complete rules — memorise these once, forget them forever:**

```
ALWAYS on stack:
→ int, float, bool, string literals
→ All precision types (int8, float64 etc)
→ All game-native types (Vector3!, Color! etc)
→ kind types — they are just a uint32
→ data types where total size ≤ 64 bytes
→ array[T, N] — fixed size known at compile time
→ Function parameters and local variables

ALWAYS on heap (ARC managed):
→ entity instances
→ component instances
→ string values (the content, not the reference)
→ list[T] contents
→ map[K, V] contents
→ data types where total size > 64 bytes
   (uses scoped heap — see section 4.6)

Skev decides. You never think about this.
```

**Why 64 bytes for the data type threshold:**

```
Modern CPU cache line = 64 bytes
A data type that fits in one cache line = fast stack access
A data type larger than one cache line = heap is more efficient

Examples:
data PlayerInput >>              # 20 bytes — stack ✅
    direction  :: Vector3!       # 12 bytes
    jump       :: bool           # 1 byte
    sprint     :: bool           # 1 byte
    fire       :: bool           # 1 byte
    padding    :: 5 bytes (auto) # compiler adds alignment
<< PlayerInput

data FullCharacterSheet >>       # ~200 bytes — heap (scoped) ✅
    name          :: string      # 8 bytes (reference)
    stats         :: array[int, 20]   # 80 bytes
    abilities     :: array[string, 8] # 64 bytes
    backstory     :: string      # 8 bytes (reference)
    ... many more fields
<< FullCharacterSheet
```

### ⚙️ Technical Version

Stack allocation uses LLVM `alloca` instructions within the function frame. The compiler performs escape analysis to determine whether a value can safely remain on the stack — if a value escapes its creating scope (passed to another thread, stored in a heap collection, or assigned to a longer-lived binding), it is promoted to heap allocation automatically.

Heap allocation uses Skev's runtime allocator — a thread-local slab allocator for small objects (≤ 512 bytes) and a general-purpose allocator for larger objects. The slab allocator eliminates lock contention for the most common allocation sizes. All heap-allocated Skev objects are 16-byte aligned to support SIMD operations on any contained game-native type fields.

The 64-byte threshold for `data` types is based on cache line size of modern x86-64 and ARM64 processors. Stack-allocated `data` types benefit from being fully contained in a single cache line during access. Larger `data` types exceed this benefit and are more efficiently managed on heap with scoped lifetime (section 4.6).

---

## 4.4 — Automatic Reference Counting (ARC)

### 👨‍💻 Developer Version

ARC is how Skev manages memory for entities and components. It tracks how many things are pointing to each entity. When nothing points to an entity — it is freed. Automatically. Immediately. With no pause.

This is different from garbage collection in a critical way:

```
Garbage Collector (C#, Python):
→ Runs periodically in background
→ Pauses your game to clean up memory
→ You cannot predict WHEN it runs
→ Mid-battle pause = frame rate spike = bad experience

ARC (Skev):
→ Frees memory the INSTANT nothing references it
→ No background process
→ No pauses
→ Completely predictable timing
→ Frame rate stays smooth always
```swift

**How ARC works — the reference count:**

```swift
entity GameManager >>

    when update(delta)
        # Step 1 — spawn creates entity, count starts at 1
        # (the scene holds one reference)
        enemy :: Enemy = scene.spawn(Enemy)    # count: 1

        # Step 2 — assigning to a variable adds a reference
        active_enemies.add(enemy)              # count: 2

        # Step 3 — passing to a function adds a reference
        #          while inside the function
        process_enemy(enemy)                   # count: 3 inside function
                                               # count: 2 after function returns

        # Step 4 — removing from collection drops a reference
        active_enemies.remove(enemy)           # count: 1

        # Step 5 — destroying removes scene's reference
        scene.destroy(enemy)                   # count: 0 → freed immediately
    << update

<< GameManager
```

**You never call free. You never call delete. ARC does it.**

### The `weak` Reference — Breaking Cycles

There is one situation where ARC needs your help: circular references.

```swift
# This creates a problem:
entity Player >>
    target :: maybe Enemy       # Player holds Enemy — count goes up
<< Player

entity Enemy >>
    attacker :: maybe Player    # Enemy holds Player — count goes up
<< Enemy

# When game ends:
# Player holds Enemy → Enemy count: 1 → not freed
# Enemy holds Player → Player count: 1 → not freed
# Both stay in memory forever — MEMORY LEAK
```swift

**The fix — `weak` references:**

```swift
entity Enemy >>
    # weak means: I know about this Player
    #             but I do NOT own it
    #             ARC count does NOT increase
    attacker :: weak Player
<< Enemy

# Now when game ends:
# Player is destroyed → Player count hits 0 → freed ✅
# Enemy's weak reference automatically becomes nothing ✅
# Enemy count hits 0 → freed ✅
# No memory leak ✅
```

**`weak` always implies `maybe` — same access rules:**

```swift
entity Enemy >>
    attacker :: weak Player

    when update(delta)
        # Must check — attacker might have been freed
        if attacker exists >>
            move_toward(attacker.position, speed * delta)
        << attacker exists

        # Accessing without check — compiler error
        move_toward(attacker.position, speed)   # Error: unsafe access
    << update

<< Enemy
```swift

**When to use `weak`:**

```swift
# Use weak for back-references
entity Child >>
    parent :: weak Entity       # child knows parent — doesn't own it
<< Child

# Use weak for observers
entity UIHealthBar >>
    observed_player :: weak Player   # watches player — doesn't own it
<< UIHealthBar

# Use weak for caches
entity EnemyManager >>
    last_target_cache :: weak Enemy  # cached reference — doesn't own it
<< EnemyManager

# Use weak whenever you would otherwise create a cycle
```

**Compiler cycle detection:**

```swift
Warning: Potential circular reference detected
  Player.target  → Enemy    (maybe Enemy)
  Enemy.attacker → Player   (maybe Player)

  This may cause a memory leak.
  Consider making one reference weak:

  In enemy.skev:
  attacker :: weak Player

  player.skev Line 8 — enemy.skev Line 6
```

### ⚙️ Technical Version

ARC reference count operations are implemented using C11 atomic operations — `atomic_fetch_add` for increment and `atomic_fetch_sub` with a subsequent zero check for decrement. Atomic operations ensure thread safety when multiple threads hold references to the same entity without requiring a mutex on every access. The small performance cost of atomic operations over non-atomic is acceptable for game use cases — measured at approximately 3-5ns per operation on modern hardware.

ARC increment occurs at the following LLVM IR generation points:
1. Assignment to a new binding (`let`-equivalent)
2. Passing as a function argument with reference semantics
3. Storing into a heap collection
4. Storing into an entity property

ARC decrement occurs at:
1. End of binding scope
2. Function return (for parameter decrements)
3. Removal from a heap collection
4. Property overwrite (old value decremented before new value stored)
5. `scene.destroy()` call (scene's own reference decremented)

When the count reaches zero, the destructor sequence fires: `destroyed` event dispatched, component destructors called in reverse declaration order, memory returned to the allocator.

`weak` references are implemented as a pointer to a weak reference table entry rather than directly to the entity. When an entity is freed, its weak reference table entries are zeroed atomically. Subsequent `if value exists` checks test the table entry for null — O(1) and branch-predictor friendly.

---

## 4.5 — Memory Alignment & Property Reordering

### 👨‍💻 Developer Version

CPUs access memory fastest when values sit at addresses that match their size. An integer at address 4 is fast. An integer at address 5 is slow — the CPU has to do extra work to read across a boundary.

Skev's compiler automatically reorders your properties to use memory as efficiently as possible. You write them in the order that reads best. The compiler stores them in the order that performs best.

**What the compiler does:**

```swift
# Developer writes — in whatever order makes logical sense:
entity Enemy >>
    name        :: string   # 8 bytes (reference pointer)
    active      :: bool     # 1 byte
    health      :: int      # 4 bytes
    dangerous   :: bool     # 1 byte
    position    :: Vector3! # 12 bytes (SIMD aligned to 16)
    level       :: int      # 4 bytes
<< Enemy

# Compiler stores in memory — optimised order:
# Offset  0: position   Vector3!  16 bytes (SIMD — must be 16-byte aligned)
# Offset 16: name       string     8 bytes (pointer — 8-byte aligned)
# Offset 24: health     int        4 bytes (4-byte aligned)
# Offset 28: level      int        4 bytes (4-byte aligned)
# Offset 32: active     bool       1 byte
# Offset 33: dangerous  bool       1 byte
# Offset 34: padding               2 bytes (compiler adds for next alignment)
# Total:                          36 bytes

# Without reordering (naive order):
# name       8 bytes
# active     1 byte  + 3 padding = 4 bytes wasted
# health     4 bytes
# dangerous  1 byte  + 11 padding = 12 bytes wasted (SIMD alignment for Vector3!)
# position  16 bytes
# level      4 bytes + 4 padding = 4 bytes wasted
# Total:    54 bytes — 18 bytes wasted
#
# Skev saves 18 bytes per instance — ×10,000 enemies = 180KB saved
```

**Skev Studio always shows your order — not the memory order:**

```
In Skev Studio debugger you see:
  name:      "Goblin"
  active:    true
  health:    85
  dangerous: false
  position:  (10.0, 0.0, 5.0)
  level:     3

Exactly as you wrote it.
The memory optimisation is invisible.
```

**Alignment rules the compiler follows:**

```
Type              Alignment
────────────────────────────
bool              1 byte
int8, uint8       1 byte
int16, uint16     2 bytes
int, uint32       4 bytes
float, float32    4 bytes
int64, uint64     8 bytes
float64           8 bytes
string (pointer)  8 bytes
entity (pointer)  8 bytes
Vector2!          8 bytes
Vector3!         16 bytes  ← SIMD requirement
Vector4!         16 bytes  ← SIMD requirement
Quat!            16 bytes  ← SIMD requirement
Color!           16 bytes  ← SIMD requirement
Transform!       16 bytes  ← SIMD requirement
```swift

### ⚙️ Technical Version

Property reordering is performed during the type layout pass of the Skev compiler, after type checking but before LLVM IR emission. The algorithm is a variant of the first-fit decreasing bin packing heuristic:

1. Sort properties by alignment requirement descending
2. Place properties greedily into the smallest gap that satisfies alignment
3. Add padding at the end to ensure the total struct size is a multiple of the maximum alignment requirement

SIMD-aligned game-native types (`Vector3!`, `Vector4!`, `Quat!`, `Color!`, `Transform!`) require 16-byte alignment to enable SSE/AVX/NEON load and store instructions without alignment fault. The compiler guarantees this alignment for all stack and heap allocations of entity types containing these fields.

Skev Studio maintains a separate property display table mapping from property declaration order to memory offset — the debugger renders properties in declaration order by translating through this table. This table is stripped from release builds.

---

## 4.6 — Scoped Heap Allocation for Large Data Types

### 👨‍💻 Developer Version

`data` types larger than 64 bytes go to the heap. But unlike entities — they do not use ARC. Instead they use something called **scoped heap allocation** — memory is automatically freed when the scope (`>>` block) they were created in closes.

```swift
entity QuestSystem >>

    when collision(other: QuestNPC)
        # This data type is large — goes to scoped heap automatically
        quest_sheet :: CharacterSheet >>
            name         :: other.character_name
            background   :: other.generate_background()
            dialogue     :: other.generate_dialogue()
            # ... many more fields — total > 64 bytes
        << CharacterSheet

        # Use it freely within this scope
        display_quest_intro(quest_sheet)
        save_quest_data(quest_sheet)

    << collision: QuestNPC     # ← quest_sheet freed HERE automatically
                               # Scope ended — memory released immediately

<< QuestSystem
```

**What happens if you try to keep it beyond its scope:**

```swift
stored_sheet :: maybe CharacterSheet   # stored at entity level

when collision(other: QuestNPC)
    quest_sheet :: CharacterSheet >>
        name :: other.character_name
    << CharacterSheet

    stored_sheet = quest_sheet    # Compiler error ❌
<< collision: QuestNPC
```

```
Error: Scoped heap value cannot escape its scope
  "quest_sheet" is a large data type allocated on scoped heap.
  It cannot be assigned to a binding that outlives its scope.

  Options:
  1. Process quest_sheet within this scope
  2. Copy only the fields you need to a smaller data type
  3. If you need to store it — use an entity instead

  Line 12 — quest_system.skev
```

**Why scoped heap instead of ARC for large data:**

```
ARC requires a reference count — 8 extra bytes per allocation.
For large data types used as temporaries — this overhead is waste.
Scoped allocation is faster and uses less memory.
Lifetime is guaranteed by scope — no counting needed.
```swift

### ⚙️ Technical Version

Scoped heap allocations use LLVM's `llvm.stacksave` and `llvm.stackrestore` intrinsics where the size is statically known, or a thread-local bump allocator with scope-level restore points where the size involves runtime components. The bump allocator maintains a stack of restore pointers — one per open `>>` block that contains scoped heap allocations. When a `<<` block-close is encountered at the scope level, all bump allocations since the corresponding restore point are freed in a single pointer restoration — O(1) regardless of allocation count within the scope.

Escape analysis prevents scoped values from being stored to longer-lived locations. The compiler performs this check during the ARC reference graph construction pass — any assignment where the source is a scoped allocation and the destination outlives the source scope is a compile error.

---

## 4.7 — Component Memory Layout

### 👨‍💻 Developer Version

Components are attached to entities with `has`. In memory, component data lives inside the entity itself — it is not a separate allocation. This makes accessing component data fast because everything is close together in memory.

```swift
entity Player >>
    health :: int = 100

    has Animator      # small component — inline
    has AudioSource   # small component — inline
    has Physics3D     # large component — pointer

<< Player
```

**In memory:**

```
Player entity in RAM:
┌────────────────────────────────────────────┐
│ ARC Count           8 bytes               │
│ Event Table Ptr     8 bytes               │
│ Component Mask      8 bytes               │
│ health (int)        4 bytes               │
│ padding             4 bytes               │
├────────────────────────────────────────────┤
│ Animator data       inline (small ≤256b)  │  ← fast access
│ AudioSource data    inline (small ≤256b)  │  ← fast access
├────────────────────────────────────────────┤
│ Physics3D pointer   8 bytes               │  ← one pointer hop
└────────────────────────────────────────────┘
                          ↓
                    [Physics3D data on heap]
```

**Why the 256-byte threshold:**

```swift
CPU cache line = 64 bytes
4 cache lines = 256 bytes

A component that fits in 4 cache lines:
→ Likely to be in CPU cache when accessed
→ Inline placement keeps it with entity data
→ No extra pointer dereference

A component larger than 256 bytes:
→ Would push entity out of cache
→ Better placed separately on heap
→ One pointer hop is cheaper than cache miss

Most game components are under 256 bytes.
Physics components tend to be large — pointer is correct for them.
```

**You never think about this. Skev decides automatically.**

### ⚙️ Technical Version

Component size is measured at compile time as the sum of all component property sizes plus alignment padding. The compiler applies the inline/pointer decision at entity type layout time. Inline components are placed after entity properties in declaration order, each aligned to their maximum property alignment. Pointer components are stored as a single pointer in the entity body, with the component data allocated on the heap alongside the entity — same allocation slab, adjacent address, maximising cache locality even for the pointer-based case.

Component initialisation fires in declaration order — the component mask bit for each component is set after its data is fully initialised, ensuring no component is visible via `has` check before it is ready.

---

## 4.8 — Entity Destruction Semantics

### 👨‍💻 Developer Version

When you call `scene.destroy(entity)` — the entity is not immediately gone. Skev follows a safe, predictable destruction sequence.

**The destruction sequence:**

```swift
# Developer calls:
scene.destroy(enemy)

# What Skev does, in order:
# Step 1 — Mark the entity as "destroying"
#           Any code that checks "if enemy exists"
#           will now get false — immediately

# Step 2 — Fire the "destroyed" event
#           The entity gets one last chance to clean up
when destroyed
    effects.spawn(particles!.explosion, position)
    audio.play(sounds.death)
    scene.trigger(events.enemy_defeated)
<< destroyed

# Step 3 — Scene releases its own reference
#           scene.destroy decrements ARC count

# Step 4 — If ARC count is now 0 — freed immediately
#           If ARC count is still > 0 — entity stays
#           in memory until all references release it
#           But it is treated as "nothing" for all access
```swift

**Safe access during destruction:**

```swift
entity Player >>
    target :: maybe Enemy

    when update(delta)
        # Safe — Skev checks "destroying" flag automatically
        if target exists >>
            move_toward(target.position, speed * delta)
            # Even if scene.destroy(target) was called elsewhere
            # this check correctly returns false
        << target exists
    << update

<< Player
```

**What "treating as nothing" means:**

```swift
If another entity holds a reference to a "destroying" entity:

Regular maybe check:
    if target exists >>        ← returns false during destruction
    << target exists

weak reference:
    attacker :: weak Player    ← automatically becomes nothing
                                  when Player is destroyed

ARC still tracks:
    The "destroying" entity stays in RAM until
    all references release it.
    But no code can ACCESS it as a live entity.
    It is invisible — just not freed yet.
```

### ⚙️ Technical Version

The "destroying" state is implemented as a reserved bit in the `arc_count` field — specifically bit 63. When `scene.destroy()` is called, this bit is set atomically before the `destroyed` event fires. All `if value exists` checks include a test of this bit in addition to the null check — a single bitwise AND and branch.

The `destroyed` event fires synchronously on the calling thread. Component destructors fire after the `destroyed` event completes, in reverse declaration order — the last-declared component is destroyed first, matching typical RAII destruction ordering. After component destruction, the scene's own ARC reference is decremented. If the count (excluding the destroying bit) reaches zero, the allocator free is called immediately on the same thread.

If the count does not reach zero, the entity remains in memory in a zombie state — the destroying bit prevents all access, and a finalisation callback is registered that will free the memory when the last reference releases. This finalisation path is O(1) — the callback is stored in the weak reference table and fires during the last ARC decrement.

---

## 4.9 — Compile-Time Reflection

### 👨‍💻 Developer Version

Skev does not support runtime reflection — the ability to inspect objects while the game is running. This keeps every entity lean with no metadata overhead.

But some use cases genuinely need to inspect types — animation tools, visualisation apps, property editors, modding systems. For these, Skev provides **compile-time reflection** — an opt-in feature that adds metadata to specific types at compile time only.

**Default — zero overhead:**

```swift
entity Dragon >>
    health :: int = 1000
    # No metadata. No overhead. Pure performance. ✅
<< Dragon
```swift

**Opt-in — compile-time reflection:**

```swift
#! introspectable
data Settings >>
    volume       :: float = 0.5
    brightness   :: float = 0.8
    fullscreen   :: bool  = false
    language     :: string = "English"
<< Settings
```

**Using introspectable types:**

```swift
# Access type metadata via TypeInfo!
info :: TypeInfo! = TypeInfo!.of(Settings)

# Read property names and types
loop prop in info.properties >>
    print(prop.name)        # "volume", "brightness", "fullscreen", "language"
    print(prop.type_name)   # "float", "float", "bool", "string"
    print(prop.default)     # "0.5", "0.8", "false", "English"
<< prop

# Useful for: property editors, serialisation, UI generation
# Example — auto-generate a settings UI:
loop prop in info.properties >>
    ui.create_field(prop.name, prop.type_name, prop.default)
<< prop
```swift

**What `#! introspectable` adds:**

```
Without #! introspectable:
data Settings → pure LLVM struct, zero metadata

With #! introspectable:
data Settings → LLVM struct + static metadata table
                metadata table contains:
                → property names (string literals)
                → property type names (string literals)
                → property byte offsets
                → property default values
                → stored in read-only data segment
                → zero runtime cost unless accessed
```swift

**Rules for introspectable:**

```swift
# Works on data types ✅
#! introspectable
data WeaponConfig >>
    damage :: float
<< WeaponConfig

# Works on kind types ✅
#! introspectable
kind GameState >>
    playing
    paused
    menu
<< GameState

# Does NOT work on entity — entities are never introspectable
#! introspectable
entity Dragon >>    # Error: entities cannot be introspectable
<< Dragon
```

```
Error: introspectable not supported on entity types
  Entities have runtime state and ARC management.
  Compile-time reflection on entities is not supported.

  If you need to inspect entity properties —
  use Skev Studio's live property inspector.

  Line 1 — dragon.skev
```

**Why entities are excluded:**

```
Entities have:
→ Runtime ARC count — changes every frame
→ maybe references — might be null
→ Component data — varies per instance
→ Event dispatch — runtime behaviour

Compile-time reflection on live runtime state
is inherently misleading and dangerous.
Use Skev Studio for entity inspection.
```

### ⚙️ Technical Version

`#! introspectable` is a compiler annotation processed during the type layout pass. For annotated types, the compiler emits a parallel `TypeMetadata` structure in the read-only data segment (`.rodata`). This structure contains:

- Property count (uint32)
- Per-property records containing: name pointer (const char*), type name pointer (const char*), byte offset (uint32), size (uint32), default value blob pointer

`TypeInfo!` is a game-native type — a thin wrapper around a pointer to the `TypeMetadata` structure. `TypeInfo!.of(T)` is resolved at compile time to the address of T's metadata — zero runtime cost.

The `loop prop in info.properties` pattern is syntactic sugar for iteration over the properties array in the metadata structure — a standard sequential scan of the fixed-size array. No heap allocation occurs during iteration.

`#! introspectable` is not permitted on `entity` types because entity property values are live runtime state — their offsets are stable but their values and ARC states are not. Skev Studio provides live entity inspection through a separate debug protocol, not through language-level reflection.

---

## 4.10 — Memory Model Across Use Cases

### 👨‍💻 Developer Version

Chapter 4's decisions were designed to work for every domain Skev targets — not just games:

**XR / VR / AR:**
```
Zero GC pauses → critical for 90fps VR (any pause = nausea)
ARC → deterministic memory → smooth frame timing
Scoped heap → perfect for per-frame pose data
TypeInfo! → useful for AR overlay property inspection
```

**Robotics:**
```
Atomic ARC → thread-safe references across control threads
Deterministic destruction → predictable real-time timing
Scoped heap → perfect for sensor reading temporaries
Stack allocation for data → minimal latency for control loops
```

**Simulation:**
```
Shared dispatch tables → thousands of simulation agents efficiently
Auto-alignment → SIMD operations on physics data
Scoped heap → per-timestep calculation data
No GC pauses → simulation timestep never interrupted
```

**Animation/VFX Tools:**
```swift
#! introspectable → property panels for all adjustable parameters
TypeInfo! → auto-generate UI for data types
Scoped heap → per-frame render data
Inline components → fast access to keyframe data
```

**Interactive Applications:**
```swift
#! introspectable → dynamic form generation
TypeInfo! → data-driven UI construction
ARC → clean memory management for UI hierarchies
weak references → parent-child UI relationships without cycles
```

**Real-time Networking:**
```
Atomic ARC → safe reference sharing across network threads
Scoped heap → per-packet data processing
Stack allocation → minimal allocation latency for hot paths
Shared dispatch tables → thousands of network entities efficiently
```

---

## 4.11 — Memory Model Quick Reference

```swift
ENTITY LAYOUT:
    [ARC count 8b][dispatch ptr 8b][component mask 8b][properties][component data]
    Event dispatch table: ONE per entity type — shared by all instances

STACK (automatic — no overhead):
    int, float, bool          all primitive types
    int8 through float64      all precision types
    Vector3!, Color! etc      all game-native types
    kind variants             just a uint32
    data ≤ 64 bytes           fits in one cache line
    array[T, N]               fixed size known at compile time

HEAP — ARC managed (automatic — reference counted):
    entity instances          reference types
    component instances       reference types
    string values             ARC managed
    list[T], map[K,V]        ARC managed collections
    data > 64 bytes           scoped heap (NOT ARC — scope managed)

ARC RULES:
    Count up:     new reference created (assign, pass, add to collection)
    Count down:   reference released (scope end, remove, overwrite, destroy)
    Count zero:   freed immediately — no pause
    Atomic:       all count operations are thread-safe

WEAK REFERENCES:
    weak T          non-owning reference — does not increment ARC
    weak implies maybe — must use "if value exists" to access
    auto-clears to nothing when referenced entity is freed
    use for: back-references, observers, caches, any cycle-breaking

ALIGNMENT:
    Compiler reorders properties for optimal packing
    Skev Studio always shows developer's declared order
    SIMD types (Vector3!, Color! etc) aligned to 16 bytes

COMPONENTS:
    ≤ 256 bytes → inline in entity body (fast — same cache lines)
    > 256 bytes → pointer to heap allocation (one hop)

DESTRUCTION (scene.destroy):
    1. Mark "destroying" — treated as nothing immediately
    2. Fire "destroyed" event
    3. Decrement scene's ARC reference
    4. Free when count reaches zero

COMPILE-TIME REFLECTION:
    Default:          zero metadata — zero overhead
    #! introspectable add metadata to data and kind types
    TypeInfo!.of(T)   access metadata — zero runtime cost
    NOT for entities  use Skev Studio for live entity inspection
```

---

## 4.12 — Complete Memory Model Example

```swift
#! combat_system.skev
#! Demonstrates all Chapter 4 memory concepts in one system.

#! introspectable          ← compile-time reflection enabled
data CombatConfig >>
    base_damage      :: float  = 10.0    # stack — 4 bytes
    crit_multiplier  :: float  = 2.0     # stack — 4 bytes
    crit_chance      :: float  = 0.15    # stack — 4 bytes
    friendly_fire    :: bool   = false   # stack — 1 byte
<< CombatConfig                          # total: 16 bytes → stack ✅

kind HitResult >>                        # uint32 → stack ✅
    miss
    hit
    critical
    blocked
<< HitResult

data HitEvent >>                         # calculated by compiler:
    result    :: HitResult               # 4 bytes
    damage    :: float                   # 4 bytes
    position  :: Vector3!                # 12 bytes
    timestamp :: float                   # 4 bytes
<< HitEvent                             # total: 24 bytes → stack ✅

entity CombatSystem >>

    # Stack allocated — fast
    config      :: CombatConfig          # 16 bytes — stack
    hit_count   :: int = 0               # 4 bytes — stack

    # Heap allocated — ARC managed
    hit_log     :: list[HitEvent]        # heap — ARC managed
    target_map  :: map[string, float]    # heap — ARC managed

    # weak reference — no ARC increment — no cycle
    game_manager :: weak GameManager

    has NetworkSync                      # component — inline or pointer

    when update(delta)
        # Large temporary data — scoped heap
        # (assuming AnalysisReport > 64 bytes)
        report :: AnalysisReport >>
            hits        :: hit_count
            total_dmg   :: calculate_total(hit_log)
            config_used :: config
        << AnalysisReport

        # Use within scope — freed automatically at << update
        if game_manager exists >>
            game_manager.update_hud(report)
        << game_manager exists

        # TypeInfo! — compile-time reflection on CombatConfig
        info :: TypeInfo! = TypeInfo!.of(CombatConfig)
        loop prop in info.properties >>
            debug.log(prop.name + ": " + prop.default)
        << prop

    << update                            # report freed here ✅

    when collision(other: Enemy)
        # HitEvent is value type — stack allocated
        hit :: HitEvent >>
            result    :: calculate_hit(other)
            damage    :: config.base_damage
            position  :: other.position
            timestamp :: time.elapsed
        << HitEvent

        match hit.result >>
            HitResult.miss     -> audio.play(sounds.miss)
            HitResult.hit >>
                hit_count += 1
                hit_log.add(hit)       # copied to heap list
                other.take_damage(hit.damage.as(int))
            << HitResult.hit
            HitResult.critical >>
                hit_count += 1
                final_damage :: float = hit.damage * config.crit_multiplier
                hit_log.add(hit)
                other.take_damage(final_damage.as(int))
                effects.spawn(particles!.critical, hit.position)
            << HitResult.critical
            HitResult.blocked  -> audio.play(sounds.block)
        << hit.result

    << collision: Enemy

    when destroyed
        hit_log.clear()                # list freed — ARC count drops
        target_map.clear()             # map freed — ARC count drops
        # game_manager weak ref — nothing to clean up
    << destroyed

<< CombatSystem
