<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 2: Syntax & Structure
**Version:** 0.2 — DRAFT  
**Authors:** AJ (Copyright © 2026) & Claude (Anthropic)  
**Status:** In Progress  
**Updated:** Block system, closing rules, event qualifiers, match arms

---

## 2.1 — Syntax Overview

### 👨‍💻 Developer Version

Skev uses a completely original syntax system designed to be readable, unambiguous, and visually clean. Unlike any existing language, Skev replaces traditional brace-based or pure indentation-based structures with a purpose-built set of symbols and keywords — each chosen for a specific reason.

Every Skev file reads almost like a story describing what your game does. This is intentional. If you cannot read your own Skev code aloud and have it make sense — something is wrong.

### ⚙️ Technical Version

Skev source files are UTF-8 encoded plain text with the `.skev` extension. The lexer performs a single-pass tokenisation producing a flat token stream consumed by a recursive descent parser. Grammar is LL(1) compatible — no backtracking required. Mandatory whitespace rules are enforced at the lexer level before parsing begins, eliminating ambiguity at source.

---

## 2.2 — File Structure & Naming

### 👨‍💻 Developer Version

Every Skev source file follows a clear structure. One entity or component per file is the recommended convention — though not enforced. Files use the `.skev` extension and are named to match their primary entity or component.

```swift
# player.skev
# ─────────────────────────────────────
# File header comment (optional)
# ─────────────────────────────────────

import Physics
import AudioSource
import weapons.Sword

entity Player >>
    speed  :: 5.0
    health :: 100

    has Physics
    has AudioSource

    when update(delta)
        move(input.direction * speed * delta)
    << update

    when collision(other: Enemy)
        health -= other.damage
    << collision: Enemy

<< Player
```swift

### ⚙️ Technical Version

A Skev source file is parsed in three sequential passes:
1. Import resolution and symbol table construction
2. Type checking and ARC reference graph construction
3. LLVM IR emission

File-level scope contains only import declarations and top-level entity, component, and type definitions. No executable statements are permitted at file scope. Circular imports are detected in pass one and produce a compile error with full import chain displayed.

### Naming Conventions

| Construct | Convention | Example |
|---|---|---|
| Entity | PascalCase | `Player`, `DragonBoss`, `FireballProjectile` |
| Component | PascalCase | `Physics`, `AudioSource`, `NetworkSync` |
| Property | snake_case | `health`, `move_speed`, `jump_force` |
| Event name | snake_case | `collision`, `health_changed`, `game_over` |
| File name | snake_case.skev | `player.skev`, `dragon_boss.skev` |
| Import path | snake_case / PascalCase | `weapons.Sword`, `physics.Rigidbody` |

---

## 2.3 — The Block System `>>` and `<<`

### 👨‍💻 Developer Version

Skev uses `>>` to open any block and `<<` with a matching label to close it. Think of `>>` as a door opening into a new scope and `<<` as that same door closing — labelled clearly so you always know exactly what just ended.

> `>>` means: open a new scope here.
> `<< label` means: this named scope is now complete.

**The closing order rule:**
```
Innermost block closes FIRST.
Outermost block closes LAST.
```

Like stacking plates:
```
Open:   Dragon → collision → hero check → health check
Close:  health check → hero check → collision → Dragon

Always the reverse. Always.
```swift

### ⚙️ Technical Version

The `>>` token is a dedicated block-open terminal in the Skev grammar. Block scope is determined by indentation — each level is exactly 4 spaces. Tabs are illegal and produce a lexer error. The `<<` token followed by a label is a block-close terminal. The compiler maintains a scope stack and verifies each `<<` label matches the corresponding `>>` declaration exactly. Label matching is case-sensitive and byte-for-byte. A mismatch is a fatal compile error.

---

## 2.4 — Auto-Close vs Manual Close

### 👨‍💻 Developer Version

Not all blocks behave the same way. This is intentional.

**Main blocks — Auto-close**

`entity` and `component` blocks are auto-closed by Skev Studio the moment you open them:

```swift
# Developer types:
entity Player >>

# Skev Studio instantly generates:
entity Player >>
    # cursor lands here


<< Player       ← auto-generated immediately
```

The closing tag is always there. Always correct. Always matching. You never forget it.

**Inner blocks — Manual close required**

Every block inside an entity or component must be manually closed with an exact matching label:

```swift
entity Player >>

    when update(delta)
        move(input.direction * speed * delta)
    << update                       # manual — required

    when collision(other: Enemy)
        if other.damage > 50 >>
            health -= other.damage
        << other.damage > 50        # manual — required
    << collision: Enemy             # manual — required

<< Player                           # auto-generated
```

If a manual close is missing the compiler halts immediately:

```
Error: Unclosed block
  Block opened:   "if other.damage > 50"    Line 9
  Expected close: << other.damage > 50
  Before:         << collision: Enemy       Line 13
  player.skev

  Every inner block requires an explicit
  closing label matching its opening exactly.
```swift

### ⚙️ Technical Version

Auto-close applies exclusively to top-level `entity` and `component` declarations. Skev Studio inserts the closing `<<` token at the correct indentation upon `>>` being typed at file scope. The compiler enforces presence regardless of IDE — a missing top-level close is still a compile error. Inner block closes are verified by the scope stack at parse time. Each `<<` label is compared byte-for-byte against the corresponding `>>` opener. No fuzzy matching. No case folding. Exact match only.

---

## 2.5 — Exact Label Matching Rules

Labels are not optional decoration. They are compiler-enforced. Here is the exact rule for every block type:

```swift
# entity — label is the entity name exactly
entity Dragon >>
<< Dragon

# component — label is the component name exactly
component HealthBar >>
<< HealthBar

# when — single handler, no qualifier needed
when update(delta)
<< update

# when — multiple handlers, type qualifier required
when collision(other: Hero)
<< collision: Hero

when collision(other: Fireball)
<< collision: Fireball

# if — label is the exact condition as written
if other is Hero >>
<< other is Hero

if health < 300 >>
<< health < 300

if alive and ready >>
<< alive and ready

# match — label is the variable being matched
match state >>
<< state

# loop — label matches the loop declaration
loop i from 0 to 10 >>
<< i

loop enemy in active_enemies >>
<< enemy

loop while health > 0 >>
<< while health > 0
```

**Mismatch error:**
```swift
Error: Closing label mismatch
  Expected:  << health < 300
  Found:     << health<300
  Line 24 — dragon_boss.skev

  Labels are case-sensitive and must match
  their opening block exactly.
  Spaces are part of the label.
```

---

## 2.6 — Duplicate Event Handlers

### 👨‍💻 Developer Version

When one entity reacts differently to different entity types, use type-qualified labels:

```swift
entity DragonBoss >>

    when collision(other: Hero)
        health -= other.weapon.damage
        audio.play(sounds.dragon_roar)
    << collision: Hero

    when collision(other: Fireball)
        health -= other.damage * 2.0
        effects.spawn(particles!.burn, position)
    << collision: Fireball

    when collision(other: PowerUp)
        speed += other.boost
        audio.play(sounds.powerup)
    << collision: PowerUp

<< DragonBoss
```

**The Rule:**
```
One handler:      when collision(other)
                  << collision           ← no qualifier needed

Multiple handlers: when collision(other: Hero)
                   << collision: Hero    ← qualifier required

                   when collision(other: Fireball)
                   << collision: Fireball
```

**Missing qualifier error:**
```
Error: Ambiguous event handler label
  Multiple "collision" handlers defined.
  Closing label must specify type:

  << collision: Hero
  << collision: Fireball
  << collision: PowerUp

  Line 31 — dragon_boss.skev
```

### ⚙️ Technical Version

Event handler labels are stored in the entity's dispatch table keyed by `event_name:TypeName`. When a single handler exists, the type qualifier is optional and the key defaults to `event_name:any`. When multiple handlers exist for the same event name the compiler requires explicit type qualifiers on all handlers for that event. Registering two handlers for the same `event_name:TypeName` compound key is a compile error.

---

## 2.7 — Nested Identical Conditions

### 👨‍💻 Developer Version

Writing the exact same condition nested inside itself is almost always a logic bug. Skev catches it:

```
Warning: Identical nested condition detected
  Outer: if health < 300    Line 12
  Inner: if health < 300    Line 13

  This is likely a logic error.
  Consider restructuring your conditions.
  dragon_boss.skev
```swift

When genuinely needed — add a disambiguating suffix:

```swift
if health < 300 >>
    # outer logic

    if health < 300 :: critical_check >>
        state :: dying
        audio.play(sounds.final_warning)
    << health < 300 :: critical_check

<< health < 300
```

The `:: suffix` tells both the compiler and the reader this is intentional, distinct, and not a mistake.

### ⚙️ Technical Version

During parsing, the scope stack tracks the condition string of every open `if` block. When a new `if` opens, its condition string is compared against all currently open `if` blocks. An exact string match produces a `IDENTICAL_NESTED_CONDITION` warning. A `:: suffix` creates a unique compound key suppressing the warning and requiring the same compound key on the closing `<<`. The suffix follows standard snake_case convention.

---

## 2.8 — Property Definitions `::`

### 👨‍💻 Developer Version

> `::` means: this name is defined as this value.

```swift
entity Player >>
    # Explicit types
    speed    :: float  = 5.0
    health   :: int    = 100
    name     :: string = "Hero"

    # Type inference
    jump_force :: 12.0        # inferred: float
    lives      :: 3           # inferred: int
    alive      :: true        # inferred: bool

    # Game-native types
    position :: Vector3! = Vector3!(0.0, 0.0, 0.0)
    tint     :: Color!   = Color!.white

    # Property modifiers
    fixed  MAX_HEALTH :: 100    # immutable
    shared score      :: 0      # shared across all instances
    hidden temp_flag  :: false  # not visible externally

<< Player
```swift

### ⚙️ Technical Version

The `::` token introduces a property binding in the current entity or component scope. Properties participate fully in ARC. Inferred types are resolved via Hindley-Milner unification constrained to Skev's closed type set. Game-native `!` types receive stack allocation when possible. Mutable properties are default; `fixed` marks immutable; `shared` creates a class-level binding; `hidden` restricts external symbol visibility.

---

## 2.9 — The Event System `when`

### 👨‍💻 Developer Version

> `when` means: whenever this happens, do this.

```swift
entity Player >>
    health :: 100

    when update(delta)
        move(input.direction * speed * delta)
    << update

    when collision(other: Enemy)
        health -= other.damage
        audio.play(sounds.hurt)
    << collision: Enemy

    when destroyed
        audio.play(sounds.death)
        effects.spawn(particles!.explosion, position)
    << destroyed

    when health_changed(current, max)
        ui.update_health_bar(current, max)
        if current < 30 >>
            audio.play(sounds.heartbeat_warning)
        << current < 30
    << health_changed

<< Player
```

### Built-in Engine Events

| Event | Parameters | When It Fires |
|---|---|---|
| `update(delta)` | float | Every frame |
| `fixed_update(delta)` | float | Fixed timestep — use for physics |
| `collision(other: Type)` | Entity | This entity touches another |
| `collision_end(other: Type)` | Entity | This entity stops touching another |
| `destroyed` | none | Before entity is removed from scene |
| `scene_load` | none | Scene containing this entity loads |
| `scene_unload` | none | Scene containing this entity unloads |
| `enabled` | none | Entity is activated |
| `disabled` | none | Entity is deactivated |

### ⚙️ Technical Version

Event handlers compile to function pointers in the entity's dispatch table keyed by `event_name:TypeName`. Engine events fire at defined game loop points. Custom events use pub-sub with O(1) dispatch. Primitives passed by value. Entity references passed by ARC-managed reference. Returning from a `when` block is a compile error — events are fire-and-forget.

---

## 2.10 — The Component System `has`

### 👨‍💻 Developer Version

> `has` means: this entity is enhanced by this capability.

```swift
entity Dragon >>
    health :: 1000

    has Physics3D
    has Animator
    has AudioSource
    has NetworkSync
    has HealthBar

    when update(delta)
        animator.play(anims.fly)
    << update

<< Dragon
```swift

**Defining your own component:**

```swift
component HealthBar >>
    bar_color :: Color!   = Color!.green
    offset    :: Vector3! = Vector3!(0.0, 2.0, 0.0)

    when health_changed(current, max)
        ui.update_bar(current, max, bar_color)
        if current < max * 0.25 >>
            bar_color :: Color!.red
        << current < max * 0.25
    << health_changed

<< HealthBar
```

### ⚙️ Technical Version

`has` declarations are resolved at compile time. Component interfaces are merged into the owning entity's symbol table. Implemented as struct mixins at IR level — no vtable, no per-component heap allocation. Must appear before any `when` blocks. Duplicates produce a compile error. Initialisation follows declaration order.

---

## 2.11 — Game-Native Types `!`

### 👨‍💻 Developer Version

> `!` means: this type speaks the language of games natively.

Always available. Never imported. Hardware-optimised automatically.

```swift
entity Spaceship >>
    position :: Vector3!  = Vector3!(0.0, 0.0, 0.0)
    rotation :: Quat!     = Quat!.identity
    tint     :: Color!    = Color!.white

    when update(delta)
        position += input.direction * speed * delta
        rotation  = Quat!.look_at(position, target.position)
    << update

<< Spaceship
```

### Complete Game-Native Type Reference

| Type | Size | Description |
|---|---|---|
| `Vector2!` | 8 bytes | 2D position, direction, or UV coordinate |
| `Vector3!` | 12 bytes | 3D position, direction, or scale |
| `Vector4!` | 16 bytes | 4D vector for shader data |
| `Quat!` | 16 bytes | Quaternion rotation — avoids gimbal lock |
| `Color!` | 16 bytes | RGBA colour with HDR support |
| `Transform!` | 64 bytes | Combined position, rotation, scale |
| `Rect!` | 16 bytes | 2D rectangle for UI and collision |
| `Ray!` | 24 bytes | Origin and direction for raycasting |

### ⚙️ Technical Version

Game-native types are defined in the compiler itself — not the standard library. LLVM IR generation uses SIMD intrinsics automatically (SSE4.2 on x86-64, NEON on ARM64). ARC does not apply — they are value types with copy semantics. Large assignments optimise to `memcpy` where beneficial.

---

## 2.12 — Operators & Mandatory Spacing

### 👨‍💻 Developer Version

Mandatory spacing around all operators is a hard compiler rule — not a style guideline.

```swift
# VALID ✅
health -= other.damage
speed  += boost.value

# INVALID — compiler error ❌
health-=other.damage
```

### Complete Operator Reference

| Op | Name | Example | Meaning |
|---|---|---|---|
| `=` | Assign | `speed = 10.0` | Set value |
| `+=` | Add assign | `health += potion.value` | Increase |
| `-=` | Subtract assign | `health -= damage` | Decrease |
| `*=` | Multiply assign | `score *= combo` | Multiply |
| `/=` | Divide assign | `speed /= friction` | Divide |
| `+` | Add | `pos = pos + dir` | Add |
| `-` | Subtract | `diff = a - b` | Subtract |
| `*` | Multiply | `vel = dir * speed` | Multiply |
| `/` | Divide | `ratio = health / max` | Divide |
| `==` | Equals | `if health == 0` | Equality check |
| `!=` | Not equals | `if state != idle` | Inequality check |
| `<` | Less than | `if health < 30` | Compare |
| `>` | Greater than | `if score > best` | Compare |
| `<=` | Less or equal | `if lives <= 0` | Compare |
| `>=` | Greater or equal | `if power >= max` | Compare |
| `is` | Type check | `if other is Enemy` | Entity type check |
| `not` | Negate | `if not alive` | Logical negation |
| `and` | Logical and | `if alive and ready` | Both true |
| `or` | Logical or | `if hurt or stunned` | Either true |

---

## 2.13 — Control Flow

### Conditionals — `if`

```swift
if health < 30 >>
    state :: wounded
    audio.play(sounds.warning)
<< health < 30
else if health < 60 >>
    state :: hurt
<< health < 60
else >>
    state :: normal
<< else
```swift

### Pattern Matching — `match`

Single-line arms use `->`. Multi-line arms use `>>` and `<<`:

```swift
match state >>

    idle -> patrol(delta)

    combat >>
        attack(target, delta)
        audio.play(sounds.combat_music)
        animator.trigger(anims.attack)
    << combat

    wounded >>
        retreat(delta)
        health += 0.5 * delta
        if health < 100 >>
            state :: dying
        << health < 100
    << wounded

    _ -> idle(delta)

<< state
```

**The Rule:**
```swift
Single action:    case -> action()           no >> needed
Multiple actions: case >>
                      action_one()
                      action_two()
                  << case
```

### Loops

```swift
loop i from 0 to 10 >>
    spawn_enemy(i)
<< i

loop while health > 0 >>
    regenerate(delta)
<< while health > 0

loop enemy in active_enemies >>
    enemy.update(delta)
<< enemy
```swift

---

## 2.14 — Comments

```swift
# Single-line comment

entity Player >>
    speed :: 5.0    # inline comment

#{
    Multi-line comment.
    Nesting supported safely.
}#

#! Doc comment — generates API documentation automatically
entity Dragon >>

<< Dragon
<< Player
```

---

## 2.15 — Complete Syntax Example

```swift
#! dragon_boss.skev
#! Final boss entity — patrols, hunts, attacks, retreats.

import weapons.FireBreath
import vfx.DragonEffects

entity DragonBoss >>

    health       :: 2000
    max_health   :: 2000
    move_speed   :: 3.5
    attack_power :: 80
    rage_timer   :: 0.0
    position     :: Vector3! = Vector3!(0.0, 5.0, 0.0)
    tint         :: Color!   = Color!.white

    fixed MAX_RAGE_TIME :: 10.0

    has Physics3D
    has Animator
    has AudioSource
    has HealthBar
    has DragonEffects

    when update(delta)
        rage_timer += delta
        match state >>

            idle -> patrol(delta)

            hunting >>
                chase(target, delta)
                animator.trigger(anims.run)
            << hunting

            combat >>
                attack(target, delta)
                if rage_timer > MAX_RAGE_TIME >>
                    move_speed += 0.5
                    rage_timer  = 0.0
                << rage_timer > MAX_RAGE_TIME
            << combat

            wounded >>
                retreat(delta)
                tint = Color!.red
            << wounded

            _ -> idle(delta)

        << state
    << update

    when collision(other: Hero)
        health -= other.weapon.damage
        audio.play(sounds.dragon_roar)

        if health < 600 >>
            state     :: hunting
            move_speed += 0.5
        << health < 600

        if health < 300 >>
            state     :: wounded
            move_speed += 1.5
        << health < 300

    << collision: Hero

    when collision(other: Fireball)
        health -= other.damage * 2.0
        effects.spawn(particles!.burn, position)
    << collision: Fireball

    when collision(other: PowerUp)
        speed += other.boost
        audio.play(sounds.powerup)
    << collision: PowerUp

    when health_changed(current, max)
        ui.update_health_bar(current, max)
        if current < max * 0.3 >>
            tint       = Color!.red
            state     :: wounded
            move_speed += 1.5
        << current < max * 0.3
    << health_changed

    when destroyed
        effects.spawn(particles!.explosion, position)
        scene.trigger(events.boss_defeated)
        audio.play(sounds.victory)
    << destroyed

<< DragonBoss
```

---

## 2.16 — Syntax Quick Reference Card

```
OPEN BLOCK:            Name >>
CLOSE MAIN BLOCK:      << Name              auto-generated
CLOSE INNER BLOCK:     << exact condition   manual, exact match required

PROPERTY:              name :: value
TYPED PROPERTY:        name :: Type = value
IMMUTABLE:             fixed name :: value
SHARED:                shared name :: value
HIDDEN:                hidden name :: value

COMPONENT:             has ComponentName

SINGLE EVENT:          when event(param)
                       << event
TYPED EVENT:           when collision(other: TypeName)
                       << collision: TypeName

MATCH SINGLE ARM:      case -> action()
MATCH MULTI ARM:       case >>
                           action()
                       << case

LOOP COUNT:            loop i from 0 to N >>  << i
LOOP WHILE:            loop while condition >>  << while condition
LOOP COLLECTION:       loop item in list >>  << item

COMMENT:               # single line
BLOCK COMMENT:         #{ multi line }#
DOC COMMENT:           #! generates documentation