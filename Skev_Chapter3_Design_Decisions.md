<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 3: Data Types — Pre-Design Decisions
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026) & Claude (Anthropic)
**Status:** All decisions locked — ready for chapter writing
**Process:** Steps 1–7 completed per Design Practices

---

> This document records all design decisions made before Chapter 3 was written. Every decision here passed the Five Question Test, was cross-checked against the Consistency Bible, real-world tested, and formally locked.

---

## Step 1 — What Chapter 3 Must Cover

```
→ Primitive types (int, float, bool, string)
→ Numeric type sizing
→ Game-native types (full spec — introduced in Chapter 2)
→ Collections (list, array, map)
→ Custom types — data containers
→ Enumerated types — kind
→ Null safety — maybe
→ Type conversion
→ Type aliases
→ Type inference rules
```

---

## Decision 1 — Numeric Type Sizing

### The Problem

Every language handles numeric sizing differently:

| Language | Approach | Problem |
|---|---|---|
| Python | Arbitrary precision | Too slow for games |
| C++ | `int32_t`, `uint64_t` | Too complex and verbose |
| C# | `int`, `long`, `short` | Confusing size assumptions |
| Rust | `i32`, `i64`, `f32`, `f64` | Precise but verbose |
| Swift | `Int`, `Double` | Platform-dependent size |

### The Skev Solution — Two-Tier Numeric System

**Tier 1 — Simple types for 99% of game code:**

```swift
int     →  32-bit signed integer
float   →  32-bit floating point
bool    →  true or false
string  →  UTF-8 encoded text
```

**Tier 2 — Precision types for experts who need control:**

```swift
int8,   int16,  int32,  int64
uint8,  uint16, uint32, uint64
float32, float64
```

**Why `float` defaults to 32-bit:**
- Every game engine uses 32-bit floats
- GPU shaders use 32-bit floats
- 64-bit floats double memory usage for no game benefit
- SIMD operations are optimised for 32-bit
- Games simply do not need 64-bit float precision

**Five Question Test:** ✅ All five passed
**Status:** Locked ✅

---

## Decision 2 — Collections

### The Problem

| Language | Approach | Problem |
|---|---|---|
| C++ | `vector<T>`, `map<K,V>` | Verbose generics, `<>` confusion |
| C# | `List<T>`, `Dictionary<K,V>` | Class-based, GC overhead |
| Python | `list`, `dict` | Dynamic, no type safety |
| Rust | `Vec<T>`, `HashMap<K,V>` | Safe but syntax heavy |

### The Skev Solution — `[]` Collection Syntax

```swift
list[int]              →  dynamic list of integers
list[Enemy]            →  dynamic list of Enemy entities
array[float, 16]       →  fixed-size array of 16 floats
map[string, int]       →  key-value store
```swift

**Why `[]` not `<>` like C++/C#:**
- `<>` is visually confused with comparison operators `<` and `>`
- `[]` is universally understood as "collection of"
- Consistent with array access syntax already using `[]`

**Collection literals:**

```swift
# List literal
scores :: list[int] = [10, 20, 30, 40]

# Map literal — uses existing -> from match arms
loot_table :: map[string, float] = [
    "gold_coin"     -> 0.8,
    "health_potion" -> 0.4,
    "rare_sword"    -> 0.05,
]

# Multi-line — trailing comma allowed
waypoints :: list[Vector3!] = [
    Vector3!(0.0, 0.0, 0.0),
    Vector3!(10.0, 0.0, 0.0),
    Vector3!(10.0, 0.0, 10.0),
]

# Empty collection
inventory :: list[string] = []
```

**Why `->` for map literals:**
- Already exists in match arms
- "key -> value" reads naturally as "key maps to value"
- No new syntax introduced — fully consistent

**Five Question Test:** ✅ All five passed
**Status:** Locked ✅

---

## Decision 3 — Pure Data Containers — `data` keyword

### The Problem

Entities carry full engine overhead — physics, rendering, audio systems. Using entity for pure data transfer objects is wasteful and semantically wrong. But adding `struct` would copy C/C++ which violates originality principles.

### The Skev Solution — `data` keyword

```swift
data DamageEvent >>
    amount  :: float
    type    :: DamageType
    source  :: maybe Entity
    critical :: bool = false
<< DamageEvent
```swift

**Why `data`:**
- Describes exactly what it is — pure data, nothing more
- No engine overhead whatsoever
- Value type — copies like game-native types
- Cannot have `when` event handlers
- Cannot `has` components
- Cannot access engine systems
- No existing language uses `data` as a type declaration keyword

**Usage:**

```swift
event :: DamageEvent >>
    amount   :: 50.0
    type     :: DamageType.fire
    source   :: maybe Entity = attacker
    critical :: true
<< DamageEvent

damage_log :: list[DamageEvent]
damage_log.add(event)    # event is COPIED into list
```

**Five Question Test:** ✅ All five passed
**Status:** Locked ✅

---

## Decision 4 — Enumerated Types — `kind` keyword

### The Problem

| Language | Approach | Problem |
|---|---|---|
| C++ | `enum class Color { Red }` | Verbose |
| C# | `enum Color { Red }` | No methods, limited |
| Python | `class Color(Enum)` | Class-based, ugly |
| Rust | `enum` with data | Powerful but complex |

### The Skev Solution — `kind` keyword

```swift
kind DamageType >>
    physical
    fire
    ice
    poison
    magic
<< DamageType

kind GameState >>
    loading
    menu
    playing
    paused
    game_over
<< GameState
```swift

**Why `kind`:**
- An enum defines what **kinds** of values exist
- Reads naturally: "a DamageType is a kind of thing"
- No existing language uses `kind` as an enum keyword
- Short, clean, memorable
- Consistent with `>>` and `<<` block system

**Usage:**

```swift
damage_type :: DamageType = DamageType.fire
state       :: GameState  = GameState.playing

match damage_type >>
    DamageType.fire    -> apply_burn(target)
    DamageType.ice     -> apply_freeze(target)
    DamageType.poison  -> apply_poison(target)
    _                  -> apply_damage(target)
<< damage_type
```

**Five Question Test:** ✅ All five passed
**Status:** Locked ✅

---

## Decision 5 — Null Safety — `maybe` keyword

### The Problem

Every null pointer crash in gaming history exists because languages allow null by default:

| Language | Approach | Problem |
|---|---|---|
| C++ | Pointers null by default | Crashes everywhere |
| C# | Nullable reference types added late | Messy, optional |
| Swift | `Optional<T>` / `T?` syntax | Good but borrowed |
| Kotlin | `T?` syntax | Clean but borrowed |

### The Skev Solution — `maybe` keyword

In Skev — **nothing is null by default. Ever.**

If a value might not exist — declare it with `maybe`:

```swift
target  :: maybe Entity       # might or might not have a target
weapon  :: maybe Sword        # player might not hold a weapon
boss    :: maybe DragonBoss   # boss might be defeated already
```swift

**Accessing a `maybe` value — two safe patterns:**

```swift
# Pattern 1 — exists check
if target exists >>
    target.take_damage(10)
    move_toward(target.position, speed)
<< target exists

# Pattern 2 — or fallback
current_target = target or scene.find(Enemy)
```

**Why `maybe`:**
- Reads like natural English — "maybe Entity"
- "if target exists" — plain English null check
- No crashes — compiler forces handling before access
- No `?` syntax borrowed from Swift or Kotlin
- Completely original approach
- Nothing in Skev is ever null without explicit `maybe`

**Compiler enforcement:**

```swift
Error: Unsafe access to maybe type
  "target" is declared as maybe Entity
  and may not contain a value.

  Use:
  if target exists >>
      target.take_damage(10)
  << target exists

  Or provide a fallback:
  safe_target = target or scene.find(Enemy)

  Line 12 — player.skev
```

**Five Question Test:** ✅ All five passed
**Status:** Locked ✅

---

## Decision 6 — Type Conversion — `.as()` method

### The Skev Solution — Explicit Only

Skev never converts types silently. Every conversion is visible in the code:

```swift
# INVALID — compiler rejects silent conversion ❌
health :: int   = 100
ratio  :: float = health      # Error: implicit conversion

# VALID — explicit conversion ✅
ratio  :: float = health.as(float)

# Precision loss — compiler warns ⚠️
damage_int :: int = damage_float.as(int)    # Warning: truncation
```swift

**Why `.as()`:**
- Reads naturally: "health as float"
- Makes every conversion visible — no hidden behaviour
- Consistent with Skev's "no hidden performance costs" principle
- Short enough not to be painful to write
- Compiler warns when conversion loses precision

**Status:** Locked ✅

---

## Concern 1 Resolved — Collection Membership

### Problem
The `in` keyword was accidentally used for membership checking (`if x in list`) but `in` is already reserved for loops (`loop enemy in active_enemies`). Same keyword, two meanings — violates unambiguity principle.

### Solution — `contains` keyword

```swift
# INVALID — "in" is reserved for loops ❌
if damage.type in weaknesses >>

# VALID ✅
if weaknesses contains damage.type >>
    multiplier = 2.0
<< weaknesses contains damage.type
```

**Why `contains`:**
- Zero conflict with loop `in` keyword
- Reads naturally — "weaknesses contains this damage type"
- Subject-first pattern — object contains value
- Works for all collection types
- Never used as membership keyword in major languages

**Status:** Locked ✅

---

## Concern 2 Resolved — List Initialisation Syntax

### Solution

```swift
# Empty
scores    :: list[int]
inventory :: list[string] = []

# With values
scores    :: list[int]    = [10, 20, 30]
tags      :: list[string] = ["hero", "player"]

# Multi-line — trailing comma allowed
waypoints :: list[Vector3!] = [
    Vector3!(0.0, 0.0, 0.0),
    Vector3!(10.0, 0.0, 0.0),
    Vector3!(10.0, 0.0, 10.0),
]

# Map with values — uses existing -> syntax
loot_table :: map[string, float] = [
    "gold_coin"     -> 0.8,
    "health_potion" -> 0.4,
    "rare_sword"    -> 0.05,
]
```swift

**Status:** Locked ✅

---

## Concern 3 Resolved — data vs entity Boundary

### Type Boundary Table

| Feature | `entity` | `component` | `data` | `kind` |
|---|---|---|---|---|
| Properties `::` | ✅ | ✅ | ✅ | ❌ |
| Events `when` | ✅ | ✅ | ❌ | ❌ |
| Components `has` | ✅ | ❌ | ❌ | ❌ |
| Engine access | ✅ | ✅ | ❌ | ❌ |
| ARC managed | ✅ | ✅ | ❌ | ❌ |
| Value type | ❌ | ❌ | ✅ | ✅ |
| Variants | ❌ | ❌ | ❌ | ✅ |

Violations produce immediate compiler errors specifying exactly what was attempted and which type to use instead.

**Status:** Locked ✅

---

## Concern 4 Resolved — ARC and Collections

### Rules

**Entity references in collections — ARC tracked:**
```swift
active_enemies :: list[Enemy]

active_enemies.add(boss)      # boss ARC count increases
active_enemies.remove(boss)   # boss ARC count decreases
                              # memory freed when count hits 0
```

**Same entity in multiple collections — safe:**
```swift
active_enemies.add(boss)      # ARC count: 2
visible_enemies.add(boss)     # ARC count: 3
active_enemies.remove(boss)   # ARC count: 2 — still alive
```swift

**`data` types in collections — always copied:**
```swift
damage_log :: list[DamageEvent]
damage_log.add(event)    # event COPIED into list
                         # modifying event afterwards
                         # does NOT affect the copy
                         # ARC does NOT apply
```

**Status:** Locked ✅

---

## Concern 5 Resolved — kind Exhaustive Match

### Rules

**Rule 1 — Every match on a `kind` must be exhaustive:**

```swift
kind GameState >>
    loading
    menu
    playing
    paused
<< GameState

match state >>
    loading -> show_spinner()
    menu    -> show_menu()
    playing -> run_game()
    # Error — paused not handled
<< state
```

```
Error: Non-exhaustive match
  kind "GameState" has 4 variants.
  Your match handles 3.

  Missing variants:
  → paused

  Fix options:
  1. Handle missing variant explicitly
  2. Add a default case with _ ->
```

**Rule 2 — Adding a variant triggers errors across the ENTIRE project:**

```
Error: Non-exhaustive match
  kind "GameState" has 6 variants.
  Your match handles 4.

  Missing variants:
  → game_over
  → credits

  Affected files:
  → game_loop.skev      Line 24
  → menu_system.skev    Line 67
  → ui_manager.skev     Line 103
```swift

**Rule 3 — `_` default arm opts out of exhaustive checking:**

```swift
match state >>
    playing -> run_game()
    paused  -> show_pause_screen()
    _       -> handle_other_states()
<< state
```

**Status:** Locked ✅

---

## Full Decision Summary

| Decision | Keyword / Syntax | Status |
|---|---|---|
| Default integer | `int` = 32-bit signed | Locked ✅ |
| Default float | `float` = 32-bit | Locked ✅ |
| Precision types | `int8` through `uint64`, `float64` | Locked ✅ |
| Dynamic list | `list[Type]` | Locked ✅ |
| Fixed array | `array[Type, size]` | Locked ✅ |
| Key-value store | `map[KeyType, ValueType]` | Locked ✅ |
| Collection literal | `[value1, value2]` | Locked ✅ |
| Map literal | `["key" -> value]` | Locked ✅ |
| Membership check | `contains` keyword | Locked ✅ |
| Data container | `data` keyword | Locked ✅ |
| Enumerated type | `kind` keyword | Locked ✅ |
| Null safety | `maybe TypeName` | Locked ✅ |
| Null check | `if value exists >>` | Locked ✅ |
| Null fallback | `value or fallback` | Locked ✅ |
| Type conversion | `.as(Type)` method | Locked ✅ |
| Type boundaries | Boundary table | Locked ✅ |
| ARC + collections | Entity refs tracked, data copies | Locked ✅ |
| kind matching | Exhaustive — cross-file reporting | Locked ✅ |

---

*All decisions locked — Chapter 3 writing begins next*
*Version 0.1 — Pre-Chapter 3 Design Decisions*
