<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 3: Data Types
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026) & Claude (Anthropic)
**Status:** In Progress
**Depends On:** Chapter 2 v0.2 — All syntax decisions locked

---

## 3.1 — Overview

### 👨‍💻 Developer Version

Every value in Skev has a type. Types tell Skev what a value is, how much memory it needs, and what operations are valid on it. Skev's type system is designed around one core idea:

> **What you see is what you get. No surprises. No hidden behaviour. No silent conversions.**

Skev has six categories of types:

| Category | Keywords | Purpose |
|---|---|---|
| Primitive | `int`, `float`, `bool`, `string` | Basic everyday values |
| Precision | `int32`, `float64` etc | Expert-level control |
| Game-Native | `Vector3!`, `Color!` etc | Built-in game math types |
| Collections | `list`, `array`, `map` | Groups of values |
| Data | `data` | Pure data containers |
| Enumerated | `kind` | Named sets of variants |

Plus two type modifiers:
- `maybe` — marks a value that might not exist
- `.as()` — explicit type conversion

### ⚙️ Technical Version

Skev implements a closed, static type system. All types are resolved at compile time — zero runtime type discovery. The type checker runs as the second of three compiler passes after import resolution. Skev's type system is sound — a program that passes type checking will never produce a type error at runtime. Generic types are implemented via monomorphisation at the IR level — no boxing, no type erasure. The type system is intentionally closed: developers cannot add new primitive categories or modify built-in type behaviour outside of the `data` and `kind` mechanisms.

---

## 3.2 — Primitive Types

### 👨‍💻 Developer Version

Primitive types are the foundation of everything in Skev. Four types cover the vast majority of game development needs:

```swift
entity Player >>
    # int — whole numbers
    health      :: int    = 100
    lives       :: int    = 3
    score       :: int    = 0

    # float — decimal numbers
    speed       :: float  = 5.0
    jump_force  :: float  = 12.5
    gravity     :: float  = -9.8

    # bool — true or false
    alive       :: bool   = true
    can_jump    :: bool   = true
    invincible  :: bool   = false

    # string — text
    name        :: string = "Hero"
    title       :: string = "The Last Guardian"
    empty       :: string = ""

<< Player
```swift

**Key rules for primitives:**

```swift
# int never accepts float — compiler error
health :: int = 5.0      # Error: cannot assign float to int

# float never accepts int silently — compiler error
speed  :: float = 5      # Error: cannot assign int to float
speed  :: float = 5.0    # Correct ✅

# bool is ONLY true or false — never 0 or 1
alive :: bool = 1        # Error: use true or false
alive :: bool = true     # Correct ✅

# string uses double quotes only
name :: string = 'Hero'  # Error: use double quotes
name :: string = "Hero"  # Correct ✅
```

### ⚙️ Technical Version

**`int`** maps to a 32-bit signed two's complement integer. Range: −2,147,483,648 to 2,147,483,647. Stored on stack. Zero-initialised if no value provided. Overflow is a compile warning and runtime panic in debug builds — wraps silently in release builds with explicit `unsafe` annotation required.

**`float`** maps to an IEEE 754 single-precision 32-bit floating point value. Chosen as default over 64-bit because game engines universally operate in 32-bit float space and GPU shaders require 32-bit alignment. NaN and Infinity are detectable via standard library functions. Arithmetic on NaN propagates silently — no exception.

**`bool`** is a 1-byte value in memory despite representing a single bit. Stored as 0x00 (false) or 0x01 (true). No implicit conversion from int or any other type.

**`string`** is a UTF-8 encoded, ARC-managed heap allocation. Strings are immutable — mutation creates a new allocation. The compiler interns string literals — identical literal strings share a single allocation. String length is O(1) via stored byte count. Character indexing is O(n) — strings are not random-access by design.

---

## 3.3 — Precision Types

### 👨‍💻 Developer Version

For the 1% of cases where `int` and `float` are not enough, Skev provides precision types. These are for expert use — networking, binary file formats, shader data, and memory-critical systems.

```swift
entity NetworkPacket >>
    # Signed integers
    tiny_value   :: int8    = 127          # -128 to 127
    small_value  :: int16   = 32767        # -32,768 to 32,767
    normal_value :: int32   = 2147483647   # same as int
    large_value  :: int64   = 9223372036854775807

    # Unsigned integers — no negative values, larger positive range
    byte_value   :: uint8   = 255          # 0 to 255
    port_number  :: uint16  = 8080         # 0 to 65,535
    player_id    :: uint32  = 4294967295   # 0 to 4 billion
    session_id   :: uint64  = 0            # 0 to 18 quintillion

    # Double precision float — when 32-bit is not enough
    latitude     :: float64 = 51.507351    # GPS coordinate precision
    longitude    :: float64 = -0.127758

<< NetworkPacket
```swift

**When to use precision types:**

```swift
# Use uint8 for colour channel values — 0 to 255
red_channel :: uint8 = 255

# Use int64 for large counters that exceed int range
total_damage_dealt :: int64 = 0

# Use float64 for GPS, astronomy, or financial calculations
world_position_x :: float64 = 51.507351

# Use int16 for audio sample data
audio_sample :: int16 = 0
```

**When NOT to use precision types:**

```swift
# Do not use int32 just because you know you want 32-bit
# Skev's int IS 32-bit — this is redundant
health :: int32 = 100    # Unnecessary — use int instead
health :: int   = 100    # Correct ✅

# Do not use float64 for game positions and physics
# 32-bit is correct and faster for all game use
position :: float64 = 0.0    # Wrong for game use
position :: float   = 0.0    # Correct ✅
```swift

### ⚙️ Technical Version

Precision types map directly to their LLVM IR integer and floating point equivalents. `uint` variants are unsigned — overflow wraps silently in all build modes. Precision types are fully interoperable with standard library serialisation and networking functions. Mixing precision types requires explicit `.as()` conversion — no implicit widening or narrowing. The compiler emits a `PRECISION_TRUNCATION` warning when converting from a wider to a narrower type.

---

## 3.4 — Game-Native Types `!`

### 👨‍💻 Developer Version

Game-native types are Skev's most powerful feature. They are always available — never imported. They have built-in math operations, hardware acceleration, and behave like primitives. The `!` suffix marks them so you always know at a glance which types the engine understands natively.

> `!` means: this type speaks the language of games natively.

```swift
entity Spaceship >>
    # Position in 3D world space
    position    :: Vector3!   = Vector3!(0.0, 0.0, 0.0)

    # Rotation — always use Quat not Euler angles
    rotation    :: Quat!      = Quat!.identity

    # Scale of the object in world space
    scale       :: Vector3!   = Vector3!(1.0, 1.0, 1.0)

    # Colour with full HDR support
    tint        :: Color!     = Color!.white

    # Combined position + rotation + scale
    transform   :: Transform! = Transform!.default

    # 2D rectangle — used for UI and 2D collision
    hit_area    :: Rect!      = Rect!(0.0, 0.0, 64.0, 64.0)

    # Ray for raycasting and line of sight
    sight_ray   :: Ray!       = Ray!(position, Vector3!.forward)

    when update(delta)
        # Math works naturally — no boilerplate
        position += input.direction * speed * delta
        rotation  = Quat!.look_at(position, target.position)

        # Game-native types support all expected operators
        distance  :: float = Vector3!.distance(position, target.position)
        direction :: Vector3! = (target.position - position).normalised
    << update

<< Spaceship
```

### Complete Game-Native Type Reference

| Type | Size | Components | Description |
|---|---|---|---|
| `Vector2!` | 8 bytes | `x`, `y` | 2D position, direction, or UV coordinate |
| `Vector3!` | 12 bytes | `x`, `y`, `z` | 3D position, direction, or scale |
| `Vector4!` | 16 bytes | `x`, `y`, `z`, `w` | 4D vector for shader data |
| `Quat!` | 16 bytes | `x`, `y`, `z`, `w` | Quaternion rotation — avoids gimbal lock |
| `Color!` | 16 bytes | `r`, `g`, `b`, `a` | RGBA colour with HDR support above 1.0 |
| `Transform!` | 64 bytes | position, rotation, scale | Complete object placement in the world |
| `Rect!` | 16 bytes | `x`, `y`, `width`, `height` | 2D rectangle for UI and collision |
| `Ray!` | 24 bytes | `origin`, `direction` | Origin point and direction for raycasting |

### Game-Native Type Constant Reference

```swift
# Vector constants
Vector2!.zero       # (0.0, 0.0)
Vector2!.one        # (1.0, 1.0)
Vector2!.up         # (0.0, 1.0)
Vector2!.right      # (1.0, 0.0)

Vector3!.zero       # (0.0, 0.0, 0.0)
Vector3!.one        # (1.0, 1.0, 1.0)
Vector3!.up         # (0.0, 1.0, 0.0)
Vector3!.forward    # (0.0, 0.0, 1.0)
Vector3!.right      # (1.0, 0.0, 0.0)

# Quaternion constants
Quat!.identity      # no rotation applied

# Colour constants
Color!.white        # (1.0, 1.0, 1.0, 1.0)
Color!.black        # (0.0, 0.0, 0.0, 1.0)
Color!.red          # (1.0, 0.0, 0.0, 1.0)
Color!.green        # (0.0, 1.0, 0.0, 1.0)
Color!.blue         # (0.0, 0.0, 1.0, 1.0)
Color!.transparent  # (0.0, 0.0, 0.0, 0.0)

# Transform constants
Transform!.default  # position zero, identity rotation, scale one
```swift

### Game-Native Math Operations

```swift
entity Combat >>
    position :: Vector3! = Vector3!(0.0, 0.0, 0.0)
    target   :: maybe Entity

    when update(delta)
        if target exists >>
            # Distance between two positions
            distance :: float = Vector3!.distance(
                position,
                target.position
            )

            # Direction from self to target (normalised)
            direction :: Vector3! = (target.position - position).normalised

            # Linearly interpolate toward target
            position = Vector3!.lerp(position, target.position, 0.1)

            # Dot product — useful for field of view checks
            dot :: float = Vector3!.dot(direction, Vector3!.forward)

            # Cross product — useful for perpendicular vectors
            perp :: Vector3! = Vector3!.cross(direction, Vector3!.up)
        << target exists
    << update

<< Combat
```

### ⚙️ Technical Version

Game-native types are defined in the Skev compiler core — not the standard library. They receive specialised LLVM IR generation. On x86-64 platforms, all `Vector3!`, `Vector4!`, `Quat!`, and `Color!` operations compile to SSE4.2 SIMD instructions automatically. On ARM64, NEON intrinsics are used. The compiler selects the optimal instruction set for the build target without developer intervention.

ARC does not apply to game-native types. They are value types with strict copy semantics — assignment always produces an independent copy. This matches GPU shader expectations and eliminates aliasing bugs. Large game-native type copies are optimised to aligned `memcpy` by the compiler when the size threshold is exceeded. `Transform!` at 64 bytes always uses `memcpy`. Smaller types use register moves where possible.

---

## 3.5 — Collections

### 👨‍💻 Developer Version

Skev has three collection types. Each serves a distinct purpose. All use `[]` notation — consistent with how arrays are accessed in all languages.

> `[]` always means: a collection of something.

---

> **Version note:** C# 13 introduced collection expressions with `[1, 2, 3]` syntax — similar to Skev's list literals. Skev's approach predates and is independent of this. The key difference is that Skev's type system is fully closed and statically inferred — C# still requires explicit type context. Skev's map literal syntax using `->` is also unique to Skev.

### `list[Type]` — Dynamic ordered collection

Size changes at runtime. Use for most game data.

```swift
entity GameManager >>
    active_enemies  :: list[Enemy]
    inventory       :: list[string]
    damage_history  :: list[float]
    waypoints       :: list[Vector3!]

    # With initial values
    allowed_states  :: list[string] = ["menu", "playing", "paused"]
    starting_items  :: list[string] = [
        "health_potion",
        "short_sword",
        "wooden_shield",
    ]

    when update(delta)
        # Add to list
        active_enemies.add(new_enemy)

        # Remove from list
        active_enemies.remove(dead_enemy)

        # Check membership
        if active_enemies contains boss >>
            music.play(sounds.boss_theme)
        << active_enemies contains boss

        # Iterate over list
        loop enemy in active_enemies >>
            enemy.update(delta)
        << enemy

        # List properties
        count  :: int  = active_enemies.count
        empty  :: bool = active_enemies.is_empty
        first  :: Enemy = active_enemies.first
        last   :: Enemy = active_enemies.last
    << update

<< GameManager
```swift

---

### `array[Type, size]` — Fixed-size collection

Size is set at compile time. Never changes. Use for performance-critical data.

```swift
entity Shader >>
    # Fixed arrays — size is part of the type
    matrix       :: array[float, 16]      # 4x4 transformation matrix
    bone_weights :: array[float, 4]       # max 4 bone influences
    colour_ramp  :: array[Color!, 8]      # 8-step colour gradient

    # Access by index
    when update(delta)
        first_weight :: float = bone_weights[0]
        last_colour  :: Color! = colour_ramp[7]

        # Out of bounds — compile error if index is a constant
        bad_access :: float = bone_weights[10]    # Error: index 10 out of bounds for array[float, 4]

        # Out of bounds — runtime panic if index is a variable
        i :: int = get_index()
        maybe_bad :: float = bone_weights[i]      # Runtime panic if i >= 4
    << update

<< Shader
```

---

### `map[KeyType, ValueType]` — Key-value store

Look up values by a unique key. Use for named lookups.

```swift
entity LootSystem >>
    # map — key maps to value
    drop_rates    :: map[string, float]
    ability_costs :: map[string, int]
    item_names    :: map[int, string]

    # With initial values — uses -> consistent with match arms
    drop_rates :: map[string, float] = [
        "gold_coin"      -> 0.80,
        "health_potion"  -> 0.40,
        "magic_sword"    -> 0.05,
        "legendary_item" -> 0.01,
    ]

    when collision(other: Enemy)
        # Check if key exists
        if drop_rates contains "gold_coin" >>
            rate :: float = drop_rates["gold_coin"]
            if math.random() < rate >>
                spawn_item("gold_coin", other.position)
            << math.random() < rate
        << drop_rates contains "gold_coin"

        # Add entry
        drop_rates.add("new_item" -> 0.1)

        # Remove entry
        drop_rates.remove("old_item")

        # Map properties
        size  :: int  = drop_rates.count
        empty :: bool = drop_rates.is_empty
    << collision: Enemy

<< LootSystem
```swift

### Collection Quick Reference

| Operation | `list[T]` | `array[T, N]` | `map[K, V]` |
|---|---|---|---|
| Add item | `.add(value)` | ❌ Fixed size | `.add(key -> value)` |
| Remove item | `.remove(value)` | ❌ Fixed size | `.remove(key)` |
| Access by index | `list[index]` | `array[index]` | `map[key]` |
| Check membership | `contains value` | `contains value` | `contains key` |
| Count items | `.count` | `.count` | `.count` |
| Check empty | `.is_empty` | `.is_empty` | `.is_empty` |
| Iterate | `loop x in list` | `loop x in array` | `loop k -> v in map` |
| Size fixed? | ❌ Dynamic | ✅ Compile-time | ❌ Dynamic |
| ARC managed? | ✅ Refs tracked | ✅ Refs tracked | ✅ Refs tracked |

### ⚙️ Technical Version

`list[T]` is implemented as a heap-allocated dynamic array with amortised O(1) append. Initial capacity is 8 elements. Capacity doubles on overflow. ARC applies to the list allocation itself and to each element if the element type is ARC-managed. Removal is O(n) — list does not guarantee order preservation unless `.add_sorted()` is used.

`array[T, N]` is a stack-allocated fixed-size buffer. Size `N` must be a compile-time constant integer. No heap allocation. No ARC overhead. Bounds checking is performed at compile time for constant indices and at runtime for variable indices. Runtime bounds violations produce a panic with full source location in debug builds; undefined behaviour in `unsafe` release builds.

`map[K, V]` is implemented as an open-addressing hash table with 70% load factor and Robin Hood hashing for collision resolution. Key type must implement the built-in `Hashable` protocol — all primitive types and `string` are `Hashable` by default. Custom `data` types can be made `Hashable` via a standard library annotation. Average O(1) lookup. ARC applies to both key and value if they are ARC-managed types.

---

## 3.6 — Data Containers — `data`

### 👨‍💻 Developer Version

`data` is Skev's answer to a simple question: what do you use when you need to group values together but you don't need a full entity with events and components?

The answer is `data`. Pure. Simple. Just values.

> `data` means: this is a container of values and nothing else.

```swift
data Vector >>
    x :: float = 0.0
    y :: float = 0.0
    z :: float = 0.0
<< Vector

data DamageEvent >>
    amount   :: float
    type     :: DamageType
    source   :: maybe Entity
    critical :: bool = false
    position :: Vector3!
<< DamageEvent

data PlayerStats >>
    kills          :: int   = 0
    deaths         :: int   = 0
    damage_dealt   :: float = 0.0
    damage_taken   :: float = 0.0
    playtime       :: float = 0.0
    highest_score  :: int   = 0
<< PlayerStats
```

**What `data` CAN do:**

```swift
data WeaponConfig >>
    damage       :: float = 10.0
    fire_rate    :: float = 0.5
    reload_time  :: float = 2.0
    magazine     :: int   = 30
    name         :: string = "Default Weapon"

    # Default values
    fixed MAX_DAMAGE :: float = 999.0

<< WeaponConfig

# Creating and using a data value
sword_config :: WeaponConfig >>
    damage      :: 45.0
    fire_rate   :: 0.0
    reload_time :: 0.0
    magazine    :: 1
    name        :: "Ancient Sword"
<< WeaponConfig

# Accessing properties
damage :: float = sword_config.damage
name   :: string = sword_config.name

# data types are always COPIED — not referenced
backup_config :: WeaponConfig = sword_config    # full copy made
backup_config.damage = 0.0                      # original unchanged
```swift

**What `data` CANNOT do:**

```swift
data BadExample >>
    health :: int = 100

    when update(delta)          # Error: data cannot have event handlers
        health -= 1
    << update

    has Physics                 # Error: data cannot have components

    audio.play(sounds.hit)      # Error: data cannot access engine systems

<< BadExample
```

**Compiler errors for boundary violations:**

```swift
Error: Event handler inside data type
  "data" types are pure data containers.
  They cannot have event handlers.

  Found:   when update(delta)
  Inside:  data BadExample
  Line 6 — bad_example.skev

  If you need event handling — use "entity" instead.
```

### ⚙️ Technical Version

`data` declarations compile to plain LLVM structs with no vtable, no dispatch table, and no ARC bookkeeping on the struct itself. Properties within a `data` type that are ARC-managed types (entity references, strings, lists) retain their individual ARC tracking. The `data` struct itself is always passed by value — memcpy on assignment. The compiler optimises small `data` types (≤ 16 bytes) to register passing in function calls. Larger `data` types are passed by pointer with copy-on-write semantics where the compiler can prove it is safe. `data` types cannot be subtyped or extended — no inheritance.

---

## 3.7 — Enumerated Types — `kind`

### 👨‍💻 Developer Version

`kind` defines a fixed set of named variants. Use it whenever a value can only be one of a known list of possibilities.

> `kind` means: this value can only ever be one of these specific things.

```swift
kind DamageType >>
    physical
    fire
    ice
    poison
    magic
    true_damage     # ignores all resistances
<< DamageType

kind GameState >>
    loading
    main_menu
    playing
    paused
    game_over
    credits
<< GameState

kind Direction >>
    north
    south
    east
    west
<< Direction
```swift

**Using `kind` values:**

```swift
entity Player >>
    state        :: GameState = GameState.playing
    facing       :: Direction = Direction.north

    when update(delta)
        match state >>
            GameState.playing >>
                handle_input(delta)
                update_physics(delta)
            << GameState.playing

            GameState.paused ->
                show_pause_menu()

            GameState.game_over ->
                show_game_over_screen()

            _ -> idle()
        << state
    << update

<< Player
```

### Exhaustive Match — The Safety Net

Every `match` on a `kind` must handle all variants. This is enforced by the compiler:

```swift
kind Weapon >>
    sword
    bow
    staff
    dagger
<< Weapon

# Non-exhaustive match — compiler error
match current_weapon >>
    Weapon.sword  -> attack_melee()
    Weapon.bow    -> attack_ranged()
    # staff and dagger not handled
<< current_weapon
```

```
Error: Non-exhaustive match
  kind "Weapon" has 4 variants.
  Your match handles 2.

  Missing variants:
  → Weapon.staff
  → Weapon.dagger

  Fix options:
  1. Handle each missing variant explicitly
  2. Add a default case with _ ->
```swift

**Adding a variant — compiler finds all affected code:**

```swift
# Developer adds new variant
kind Weapon >>
    sword
    bow
    staff
    dagger
    greatsword    # ← new variant added
<< Weapon
```

```
Error: Non-exhaustive match — kind updated
  kind "Weapon" now has 5 variants.
  The following matches are now incomplete:

  → combat_system.skev    Line 34
  → inventory_ui.skev     Line 89
  → tutorial.skev         Line 12

  Missing variant in all files:
  → Weapon.greatsword

  Update all affected matches before building.
```

**Opting out of exhaustive checking:**

```swift
# Use _ -> to handle all unspecified variants
match current_weapon >>
    Weapon.sword -> attack_melee()
    Weapon.bow   -> attack_ranged()
    _            -> attack_default()    # covers staff, dagger, greatsword
<< current_weapon
```swift

### ⚙️ Technical Version

`kind` variants compile to a 32-bit unsigned integer discriminant. The compiler assigns discriminants sequentially starting at 0. Discriminant values are stable across compilations — adding a variant appends to the end and does not change existing discriminant values, making `kind` types safe for serialisation. `match` on a `kind` compiles to a jump table — O(1) dispatch regardless of variant count. Exhaustive checking is performed during the type-checking pass using the variant set recorded in the symbol table. The `_` arm suppresses exhaustive checking and compiles to the jump table's default target.

---

## 3.8 — Null Safety — `maybe`

### 👨‍💻 Developer Version

In Skev — **nothing is null by default. Ever.**

Every type in Skev always contains a valid value unless you explicitly mark it with `maybe`. This eliminates an entire category of crashes that plague C++, C#, and Unity games.

> `maybe` means: this value might not exist right now.

```swift
entity Enemy >>
    # These always have a value — can never be null
    health   :: int    = 100
    position :: Vector3!

    # These might not have a value — must use "maybe"
    target        :: maybe Entity       # might not be targeting anything
    patrol_route  :: maybe list[Vector3!]  # might have no patrol route
    last_hit_by   :: maybe Entity       # might not have been hit yet
    equipped_item :: maybe Weapon       # might not hold a weapon

<< Enemy
```

**Accessing a `maybe` value — Pattern 1: exists check**

```swift
when update(delta)
    # Must check before accessing
    if target exists >>
        move_toward(target.position, speed * delta)
        if Vector3!.distance(position, target.position) < attack_range >>
            attack(target)
        << Vector3!.distance(position, target.position) < attack_range
    << target exists

    # Accessing without check — compiler error
    move_toward(target.position, speed)    # Error: unsafe access to maybe type
<< update
```swift

**Accessing a `maybe` value — Pattern 2: or fallback**

```swift
when update(delta)
    # Provide a fallback if value does not exist
    safe_target :: Entity = target or scene.find_nearest(Enemy)

    # Now safe_target is guaranteed to exist
    move_toward(safe_target.position, speed * delta)
<< update
```

**Assigning and clearing `maybe` values:**

```swift
when collision(other: Hero)
    # Assign a value to a maybe
    last_hit_by = other

    # Clear a maybe — set it to nothing
    target = nothing
<< collision: Hero
```swift

**`maybe` in data types:**

```swift
data QuestEntry >>
    title       :: string
    description :: string
    reward      :: int
    location    :: maybe Vector3!      # some quests have no map marker
    prerequisite :: maybe string       # some quests have no prerequisite
<< QuestEntry
```

**Compiler enforcement:**

```swift
Error: Unsafe access to maybe type
  "target" is declared as maybe Entity
  and may not contain a value at this point.

  Use one of these safe access patterns:

  Pattern 1 — existence check:
  if target exists >>
      target.take_damage(10)
  << target exists

  Pattern 2 — fallback value:
  safe = target or fallback_entity

  Line 18 — enemy.skev
```

### ⚙️ Technical Version

`maybe T` is implemented as a tagged union — a discriminant byte followed by the value storage. The discriminant is 0x00 for nothing and 0x01 for a present value. For ARC-managed types, the ARC count is only incremented when the `maybe` holds a present value. The compiler performs flow-sensitive type narrowing — inside an `if value exists >>` block, the type of `value` is narrowed from `maybe T` to `T`, allowing direct access without runtime overhead. The `or` fallback operator compiles to a branch on the discriminant with the fallback evaluated lazily — it is not evaluated if the `maybe` holds a present value. `nothing` is the zero value of the tagged union — discriminant 0x00, value storage zeroed.

---

## 3.9 — Type Conversion — `.as()`

### 👨‍💻 Developer Version

Skev never converts types silently. Every conversion must be written explicitly using `.as(Type)`. This makes your code honest — you can always see exactly where a type changes.

> `.as()` means: convert this value to this type right here, explicitly.

```swift
entity ScoreSystem >>
    score     :: int   = 1500
    max_score :: int   = 10000
    multiplier :: float = 1.5

    when update(delta)
        # Convert int to float for division
        ratio        :: float = score.as(float) / max_score.as(float)
        display_pct  :: float = ratio * 100.0

        # Convert float to int — compiler warns about truncation
        rounded_score :: int = (score.as(float) * multiplier).as(int)

        # Convert int to string for display
        score_text :: string = score.as(string)
        display_text :: string = "Score: " + score_text
    << update

<< ScoreSystem
```swift

**Conversion rules:**

```swift
# int → float — safe, no data loss
ratio :: float = score.as(float)     # ✅ safe

# float → int — truncates decimal part — compiler warns ⚠️
damage :: int = 45.7.as(int)         # result: 45 — decimal lost

# int → string — always safe
text :: string = score.as(string)    # "1500"

# float → string — always safe
text :: string = ratio.as(string)    # "0.75"

# string → int — may fail — returns maybe int
parsed :: maybe int = "42".as(maybe int)
if parsed exists >>
    use_value(parsed)
<< parsed exists

# int → precision type — safe if value fits
small :: int8 = 127.as(int8)        # ✅ fits in int8 range
large :: int8 = 500.as(int8)        # ⚠️ Warning: value 500 exceeds int8 range

# Chained conversions — allowed but each step is explicit
result :: string = score.as(float).as(string)
```

**Compiler truncation warning:**

```
Warning: Precision loss in type conversion
  Converting float 45.7 to int will truncate to 45.
  Decimal portion 0.7 will be lost.

  If this is intentional use math.floor() or math.ceil()
  to make your rounding intent explicit.

  Line 12 — score_system.skev
```

### ⚙️ Technical Version

`.as(Type)` is a compiler intrinsic — not a standard library function. It emits the appropriate LLVM IR conversion instruction for the source and target type pair. Integer widening uses `sext` (signed extend) or `zext` (zero extend) based on source signedness. Integer narrowing uses `trunc`. Float to integer uses `fptosi` (float to signed integer) which truncates toward zero. Integer to float uses `sitofp`. `string` conversions are dispatched to standard library formatting functions at the IR level. The `maybe` variant of string-to-numeric conversions (`"42".as(maybe int)`) uses the standard library parser which returns `nothing` on parse failure rather than throwing.

---

## 3.10 — Type Inference

### 👨‍💻 Developer Version

Skev can figure out the type of a value automatically when it is obvious from context. You do not have to write the type — but you always can if you want to be explicit.

```swift
entity InferenceExample >>
    # Explicit — you declare the type
    health     :: int   = 100
    speed      :: float = 5.0
    alive      :: bool  = true
    name       :: string = "Hero"

    # Inferred — Skev figures out the type
    health     :: 100       # Skev infers: int
    speed      :: 5.0       # Skev infers: float
    alive      :: true      # Skev infers: bool
    name       :: "Hero"    # Skev infers: string

    # Both mean exactly the same thing
    # Use explicit when clarity matters
    # Use inferred when the type is obvious

<< InferenceExample
```swift

**Inference rules — exactly what Skev infers:**

```swift
# Integer literals → int
count :: 10         # int

# Decimal literals → float (must have decimal point)
ratio :: 0.5        # float
ratio :: 0          # int — no decimal = int, not float

# true / false → bool
flag :: true        # bool

# Double-quoted text → string
label :: "Hello"    # string

# Game-native constructors → their type
pos :: Vector3!(1.0, 0.0, 0.0)    # Vector3!
col :: Color!.red                  # Color!

# Collection literals → list
items :: ["sword", "shield"]       # list[string]
nums  :: [1, 2, 3, 4, 5]          # list[int]

# Kind variant → that kind
state :: GameState.playing         # GameState

# Another entity's property → matches that property's type
speed :: other.speed               # same type as other.speed
```

**When to use explicit types:**

```swift
# Always use explicit types when:

# 1 — The type is not immediately obvious
damage_ratio :: float = 0    # Looks like int — be explicit
damage_ratio :: 0.0          # Still not obvious — use explicit
damage_ratio :: float = 0.0  # Clear ✅

# 2 — Documenting intent for other developers
max_enemies :: int = 50      # Clear this is intentional int ✅

# 3 — Precision types — always explicit
session_id :: uint64 = 0     # Explicit — inference would give int ✅
```swift

### ⚙️ Technical Version

Type inference is performed during the second compiler pass using Hindley-Milner unification constrained to Skev's closed type set. Inference is always local — Skev does not perform whole-program type inference. Each property binding is inferred independently from its initialiser expression. Integer literals default to `int` (32-bit). Decimal literals default to `float` (32-bit). Inference failure produces a `TYPE_AMBIGUITY` compile error requiring an explicit annotation. Collection literal inference requires all elements to share a common type — mixed-type literals are a compile error.

---

## 3.10.5 — Generics

### 👨‍💻 Developer Version

User-defined generic functions and data types are fully defined in **Chapter 3.5 — Generics**. The built-in collections `list[T]`, `map[K,V]`, `result[T]`, `channel[T]` use the same `[]` notation as all user-defined generics.

```swift
# Built-in generics — unchanged
scores  :: list[int]
lookup  :: map[string, float]

# User-defined generic function — see Chapter 3.5
identity[T](value: T) -> T
    result value
<< identity

# User-defined generic data type — see Chapter 3.5
data Pair[A, B] >>
    first  :: A
    second :: B
<< Pair
```

> **Status:** ✅ Resolved — see Chapter 3.5 for full specification
> **Gap was:** Critical — last critical gap in the specification

### ⚙️ Technical Version

Generics use monomorphisation — the compiler generates a separate concrete implementation for each type a generic function or data type is used with. This interacts with LLVM IR generation, ARC reference graph construction, and the closed type system. Full specification in Chapter 3.5.

---

## 3.11 — Type Aliases

### 👨‍💻 Developer Version

Type aliases let you give a new name to an existing type. Use them to make your code more readable and domain-specific.

```swift
# Define a type alias
alias EntityID  = uint32
alias Health    = int
alias Damage    = float
alias Timestamp = float64

# Use it like the original type
entity Player >>
    id        :: EntityID  = 0
    health    :: Health    = 100
    last_seen :: Timestamp = 0.0

<< Player

# Aliases are interchangeable with their base type
score :: int     = 100
value :: Health  = score    # valid — Health IS int
```

**Aliases do NOT create new types:**

```swift
alias Health = int
alias Armour = int

health :: Health = 100
armour :: Armour = 50

# These are fully interchangeable — same underlying type
health = armour    # valid — both are int underneath
```swift

### ⚙️ Technical Version

Type aliases are resolved during the first compiler pass — symbol table construction. An alias is a pure compile-time substitution with zero runtime representation. The compiler substitutes the base type wherever the alias appears before type checking begins. Aliases are therefore transparent to the type checker — `Health` and `int` are indistinguishable after substitution. Aliases cannot be recursive. Aliases cannot alias generic collection types without providing type parameters — `alias IntList = list[int]` is valid; `alias MyList = list` is not.

---

## 3.12 — Complete Data Types Example

All decisions from this chapter in one unified real-world system:

```swift
#! combat_system.skev
#! Handles all damage calculation, application, and logging.

# ── Enumerated Types ──────────────────────────────────────────
kind DamageType >>
    physical
    fire
    ice
    poison
    magic
    true_damage
<< DamageType

kind ResistanceLevel >>
    none
    weak
    normal
    resistant
    immune
<< ResistanceLevel

# ── Data Containers ───────────────────────────────────────────
data DamageEvent >>
    amount       :: float
    type         :: DamageType
    source       :: maybe Entity
    target       :: maybe Entity
    critical     :: bool    = false
    position     :: Vector3!
    timestamp    :: float   = 0.0
<< DamageEvent

data ResistanceProfile >>
    physical     :: ResistanceLevel = ResistanceLevel.normal
    fire         :: ResistanceLevel = ResistanceLevel.normal
    ice          :: ResistanceLevel = ResistanceLevel.normal
    poison       :: ResistanceLevel = ResistanceLevel.normal
    magic        :: ResistanceLevel = ResistanceLevel.normal
<< ResistanceProfile

# ── Type Aliases ──────────────────────────────────────────────
alias DamageLog   = list[DamageEvent]
alias ResistMap   = map[string, float]

# ── Main Entity ───────────────────────────────────────────────
entity CombatSystem >>

    # Primitive types
    total_damage_dealt  :: float  = 0.0
    total_hits          :: int    = 0
    critical_hit_chance :: float  = 0.15
    critical_multiplier :: float  = 2.0

    # Collections
    damage_log          :: DamageLog
    resistance_modifiers :: ResistMap = [
        "fire_weakness"   -> 1.5,
        "ice_resistance"  -> 0.5,
        "magic_immunity"  -> 0.0,
    ]

    # maybe types
    last_attacker       :: maybe Entity
    last_damage_event   :: maybe DamageEvent

    has NetworkSync

    when update(delta)
        # Clean up old damage log entries
        # keeping only last 100 events
        loop while damage_log.count > 100 >>
            damage_log.remove(damage_log.first)
        << while damage_log.count > 100
    << update

    when collision(other: Projectile)
        # Build the damage event
        event :: DamageEvent >>
            amount    :: calculate_damage(other)
            type      :: other.damage_type
            source    :: other.owner
            target    :: other.hit_target
            critical  :: math.random() < critical_hit_chance
            position  :: other.position
            timestamp :: time.elapsed
        << DamageEvent

        # Apply critical multiplier
        if event.critical >>
            final_damage :: float = event.amount * critical_multiplier
        << event.critical
        else >>
            final_damage :: float = event.amount
        << else

        # Apply resistance
        if resistance_modifiers contains event.type.as(string) >>
            modifier :: float = resistance_modifiers[event.type.as(string)]
            final_damage *= modifier
        << resistance_modifiers contains event.type.as(string)

        # Update tracking
        total_damage_dealt += final_damage
        total_hits += 1
        last_attacker    = other.owner
        last_damage_event = event

        # Store in log
        damage_log.add(event)

        # Apply the damage
        if event.target exists >>
            event.target.receive_damage(final_damage.as(int))
        << event.target exists

    << collision: Projectile

<< CombatSystem
```

---

## 3.13 — Data Types Quick Reference

```swift
PRIMITIVES:
    int           32-bit signed integer
    float         32-bit float
    bool          true or false
    string        UTF-8 text

PRECISION TYPES:
    int8  int16  int32  int64
    uint8 uint16 uint32 uint64
    float32  float64

GAME-NATIVE TYPES (! suffix — always available):
    Vector2!  Vector3!  Vector4!
    Quat!     Color!    Transform!
    Rect!     Ray!

COLLECTIONS:
    list[Type]              dynamic ordered
    array[Type, size]       fixed-size ordered
    map[KeyType, ValueType] key-value lookup
    [v1, v2, v3]            list literal
    ["k1" -> v1]            map literal

CUSTOM TYPES:
    data Name >>     pure data container — value type
        prop :: Type
    << Name

    kind Name >>     enumerated variants
        variant_one
        variant_two
    << Name

NULL SAFETY:
    maybe Type          value might not exist
    if x exists >>      safe access pattern
    x or fallback       fallback pattern
    x = nothing         clear a maybe value

TYPE CONVERSION (explicit only):
    value.as(float)     convert to float
    value.as(int)       convert to int (truncates)
    value.as(string)    convert to string
    "42".as(maybe int)  parse string — returns maybe

TYPE ALIASES:
    alias Name = ExistingType
```

---

*End of Chapter 3 v0.1 — Next: Chapter 4: Memory Model (ARC)*
