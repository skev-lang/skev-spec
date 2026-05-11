<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Syntax Highlighting & Color Scheme
## Official Visual Design Reference
**Version:** 1.0
**Authors:** AJ (Copyright © 2026)
**Status:** Locked
**Applies To:** Skev Studio, VS Code extension, documentation site, all tooling

---

> Color is not decoration in Skev — it is information. Every token category has a specific color with a specific reason. All tools must use these colors consistently.

---

## Dark Theme — Primary

Used in Skev Studio, VS Code extension, and developer documentation.

| Token | Color | Hex | Reason |
|---|---|---|---|
| Background | ■ Deep Dark | `#0D1117` | Game developer standard — easy on eyes |
| Default text | ■ Soft White | `#E6EDF3` | Maximum readability on dark background |
| **Keywords** | ■ Warm Red | `#FF7B72` | Commands that DO things — attention-grabbing |
| **Game-native `!`** | ■ Bright Blue | `#79C0FF` | The feature unique to Skev — visually distinct |
| **Operators** | ■ Warm Orange | `#FF9E64` | Symbols that CONNECT things |
| **Strings** | ■ Light Blue | `#A5D6FF` | Stored data — cool and calm |
| **Numbers** | ■ Bright Blue | `#79C0FF` | Same family as game-native types |
| **Comments** | ■ Muted Grey | `#8B949E` | Important but secondary — recedes |
| **Doc comments `#!`** | ■ Bright Green | `#3FB950` | Documentation — stands out positively |
| **Property names** | ■ Amber | `#FFA657` | State of entities — warm data |
| **Type names** | ■ Lavender | `#D2A8FF` | Classifications — distinct from properties |
| **Block labels `<<`** | ■ Light Green | `#7EE787` | Navigation markers — always findable |
| **Punctuation** | ■ Muted Grey | `#8B949E` | Structural but not primary |

---

## Light Theme — Documentation Site

Used on the Skev documentation website and printed materials.

| Token | Color | Hex |
|---|---|---|
| Background | White | `#FFFFFF` |
| Default text | Near Black | `#24292F` |
| **Keywords** | Dark Red | `#CF222E` |
| **Game-native `!`** | Dark Blue | `#0550AE` |
| **Operators** | Dark Orange | `#953800` |
| **Strings** | Deep Blue | `#0A3069` |
| **Numbers** | Dark Blue | `#0550AE` |
| **Comments** | Medium Grey | `#6E7781` |
| **Doc comments `#!`** | Dark Green | `#116329` |
| **Property names** | Dark Amber | `#953800` |
| **Type names** | Purple | `#8250DF` |
| **Block labels `<<`** | Dark Green | `#116329` |
| **Punctuation** | Medium Grey | `#6E7781` |

---

## Token Categories — Complete Reference

### Keywords (`#FF7B72` dark / `#CF222E` light)

All reserved words that define structure and behaviour:

```swift
Declaration keywords:
entity    component    data    kind    alias

Property modifiers:
fixed    shared    hidden    weak    maybe

Event and control:
when    has    if    else    match    loop

Concurrency:
task    async    await    cancel    shared    realtime

Values:
true    false    nothing

Logic:
and    or    not    is    exists    in    contains

Control flow:
stop    skip    result

Type conversion:
as
```

### Game-Native Types (`#79C0FF` dark / `#0550AE` light)

All types ending with `!` — always bright blue to stand out:

```
Vector2!    Vector3!    Vector4!
Quat!       Color!      Transform!
Rect!       Ray!        TypeInfo!
particles!  
```

### Operators (`#FF9E64` dark / `#953800` light)

```swift
Block:       >>    <<
Property:    ::
Arrow:       ->
Assignment:  =    +=    -=    *=    /=
Comparison:  <    >    <=    >=    ==    !=
```

### Block Labels (`#7EE787` dark / `#116329` light)

The text following `<<` is a block label — highlighted separately from operators:

```swift
<< Dragon              # "Dragon" is highlighted as block label
<< collision: Enemy    # "collision: Enemy" is highlighted as block label
<< health < 300        # "health < 300" is highlighted as block label
<< update              # "update" is highlighted as block label
```swift

Block labels are bright green — they are Skev's most important navigation feature. A developer scrolling through 500 lines can immediately spot where every block ends.

### Type Names (`#D2A8FF` dark / `#8250DF` light)

Any PascalCase identifier that is a type:

```swift
entity Player >>           # "Player" is type name color
data DamageEvent >>        # "DamageEvent" is type name color
kind GameState >>          # "GameState" is type name color
target :: maybe Enemy      # "Enemy" is type name color
has Physics3D              # "Physics3D" is type name color
```

### Property Names (`#FFA657` dark / `#953800` light)

snake_case identifiers that are property declarations or accesses:

```swift
health      :: int = 100   # "health" is property color
move_speed  :: float = 5.0 # "move_speed" is property color
player.health              # "health" after . is property color
other.damage_power         # "damage_power" is property color
```

---

## Visual Example — Dark Theme

Here is how a Skev code block looks with full syntax highlighting applied:

```
entity Dragon >>                    # entity=red, Dragon=purple, >>=orange
    health      :: 1000             # health=amber, ::=orange, 1000=blue
    position    :: Vector3!         # Vector3!=bright blue
    tint        :: Color! = Color!.red  # Color!=blue

    has Physics3D                   # has=red, Physics3D=purple
    has Animator                    # Animator=purple

    when update(delta)              # when=red, update=amber
        position += velocity * delta  # +=orange, *=orange

        if health < 300 >>          # if=red, <<=orange
            tint = Color!.red       # Color!=blue
        << health < 300             # << block label=green

    << update                       # << block label=green

    when collision(other: Enemy)    # when=red, Enemy=purple
        health -= other.attack_power  # -==orange

        # This dragon is under attack!    # #=grey
        #! @param other — the attacker    # #!=green

    << collision: Enemy             # << block label=green

<< Dragon                           # << block label=green
```swift

---

## Color Philosophy

### Why Red for Keywords?

Keywords are the skeleton of your code. They define structure. They must be immediately visible. Red draws the eye — it says "this is load-bearing." When you scan unfamiliar Skev code, red keywords instantly show you the shape of the program.

### Why Blue for Game-Native Types?

Game-native types — `Vector3!`, `Color!`, `Transform!` — are Skev's most unique feature. They deserve to stand out immediately. Bright blue against the dark background is the most eye-catching color combination in dark themes. When you see blue, you know you are looking at something no other language has.

### Why Green for Block Labels?

Block labels (`<< Dragon`, `<< update`, `<< collision: Enemy`) are Skev's navigation system. In a 500-line file, green labels let you scan vertically and immediately understand structure. Green is universally associated with "go" and "complete" — a block label says "this scope is complete." Bright green makes this unmissable.

### Why Amber for Properties?

Properties hold the state of your entities. They are the data you think about most during development — health, position, speed, damage. Amber is warm, slightly urgent, data-focused. It sits between red (keywords) and orange (operators) without competing with either.

### Why Purple for Types?

Type names — `Player`, `DamageEvent`, `GameState` — are classifications. They categorise. Purple has historically been associated with distinction and category. It is different enough from red, blue, orange, and amber to be immediately distinguishable, while being softer than all of them — types are important but not as urgent as keywords.

---

## MD File Approximation — Temporary

Until Chapter 11 delivers proper Skev tooling, MD files use `swift` as a language hint:

````
```swift
entity Dragon >>
    health :: 1000
    ...
<< Dragon
```
````

**What `swift` approximates correctly:**
- String literals → highlighted
- Numbers → highlighted
- PascalCase type names → highlighted as types
- `if`, `else` → highlighted as keywords

**What `swift` does NOT approximate:**
- `entity`, `component`, `data`, `kind` → not highlighted
- `>>` and `<<` → not highlighted correctly
- `::` → not highlighted correctly
- `when`, `has` → not highlighted
- `#` comments → not highlighted (Swift uses `//`)

This is a known limitation. Proper Skev syntax highlighting is a Chapter 11 deliverable requiring a TextMate grammar file and LSP implementation.

---

## Tooling Deliverables (Chapter 11)

| Tool | Deliverable | Color Source |
|---|---|---|
| Skev Studio | Native syntax highlighting | This document |
| VS Code extension | `.tmGrammar.json` | This document |
| Documentation site | Prism.js plugin | This document |
| GitHub rendering | `.tmGrammar.json` (same as VS Code) | This document |
| Terminal output | ANSI color approximation | This document |

All tools use the same color values from this document. No tool may define its own Skev color scheme independently. Consistency across all surfaces is mandatory.

---

## Version History

| Version | Change |
|---|---|
| 1.0 | Initial color scheme locked — dark and light themes defined |

---

*This document is locked. Color values require explicit agreement to change.*
*Consistency across all Skev tools is mandatory.*
