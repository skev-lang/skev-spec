<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Function Syntax — Formal Definition
**Version:** 0.1
**Authors:** AJ (Copyright © 2026)
**Status:** Gap discovered in new chat — all decisions locked here
**Context:** Function syntax was used implicitly across Chapters 6, 7, 8
             but never formally defined. This document locks it formally.
**Depends On:** Chapter 2 (syntax), Chapter 3 (types), Chapter 6 (errors)

---

## Why This Document Exists

Function syntax was used in Chapters 6, 7, and 8 without ever being formally
designed as a language decision. The new Claude instance correctly identified
this gap when beginning Chapter 3.5 (Generics). This document formally locks
all function syntax decisions so Chapter 3.5 can proceed.

---

## The Complete Function Syntax

### Standalone Function

```swift
# Basic function
function_name(param1: Type1, param2: Type2) -> ReturnType
    # body — local variables use :: as always
    local_var :: Type = value
    result local_var
<< function_name
```

### Function That Returns Nothing

```swift
apply_damage(target: Entity, amount: int) -> nothing
    target.health -= amount
    audio.play(sounds.hit)
<< apply_damage

# Or omit -> nothing — same meaning
apply_damage(target: Entity, amount: int)
    target.health -= amount
<< apply_damage
```

### Fallible Function (returns result[T])

```swift
validate_health(value: int) -> result[int]
    if value < 0 >>
        fail ValidationError.negative
    << value < 0
    succeed value
<< validate_health
```

### Async Function

```swift
async load_level(path: string) -> result[LevelData]
    data :: LevelData = -> await file.load(path)
    succeed data
<< load_level
```

### Method on Entity

Functions declared inside an entity body are methods.
They have access to entity properties via `self` (implicit).

```swift
entity Player >>
    health :: int = 100
    speed  :: float = 5.0

    # Method — part of entity
    take_damage(amount: int)
        health -= amount
        if health <= 0 >>
            scene.destroy(self)
        << health <= 0
    << take_damage

    # Method with return type
    is_alive() -> bool
        result health > 0
    << is_alive

    # Fallible method
    equip_weapon(weapon_id: int) -> result[Weapon]
        weapon :: Weapon = -> inventory.find(weapon_id)
        equipped_weapon = weapon
        succeed weapon
    << equip_weapon

<< Player
```

---

## All Function Syntax Rules — Locked

### Rule 1 — Declaration Syntax

```
name(param1: Type, param2: Type) -> ReturnType
    body
<< name
```

- Parameter list in parentheses — always named
- `->` for return type — same `->` as match arms and error propagation
- Body starts immediately on next line — no `>>` needed
- Closed with `<< function_name` — same as all Skev blocks
- No `fun`, `def`, `fn`, `function` keyword — the name IS the declaration

### Rule 2 — Return Values

```swift
# result keyword — returns a value from non-result functions
result value

# succeed keyword — returns success from result[T] functions
succeed value

# fail keyword — returns failure from result[T] functions
fail ErrorType

# -> nothing — returns nothing explicitly
-> nothing is allowed as a synonym for falling off end of function
```

**Why `result` not `return`:**
```
"return" is a universal keyword in other languages.
Skev avoids it because:
→ result already exists as a type (result[T])
→ Using it as a return keyword is consistent
→ Reads naturally: "result: this value"
→ Avoids confusion with "return" from loops (Skev uses "stop")
```

### Rule 3 — Parameter Rules

```swift
# Parameters always named — no positional-only
do_thing(x: float, y: float)    # always named

# Default values allowed
greet(name: string, greeting: string = "Hello")
    debug.log("{greeting}, {name}!")
<< greet

# Calling with defaults
greet("Dragon")              # "Hello, Dragon!"
greet("Dragon", "Welcome")   # "Welcome, Dragon!"

# Multiple return values — use data type
get_min_max(values: list[float]) -> MinMax
    result MinMax >>
        min :: values.min()
        max :: values.max()
    << MinMax
<< get_min_max
```

### Rule 4 — Functions vs When Blocks vs Methods

```
when block (event handler):
→ Declared inside entity with "when" keyword
→ Fires in response to events
→ Cannot return a value — fire-and-forget
→ Lives in entity dispatch table

Method (entity function):
→ Declared inside entity WITHOUT "when" keyword
→ Called explicitly — not event-driven
→ CAN return a value
→ Has access to entity properties

Standalone function:
→ Declared OUTSIDE any entity
→ Pure computation — no entity state
→ CAN return a value
→ Useful for utilities, algorithms, helpers
```

**Example of all three:**
```swift
# Standalone function — outside any entity
clamp_health(value: int, max: int) -> int
    result math.clamp(value, 0, max)
<< clamp_health

entity Player >>
    health    :: int = 100
    max_health :: int = 100

    # Method — inside entity, no "when"
    heal(amount: int)
        health = clamp_health(health + amount, max_health)
    << heal

    # Event handler — inside entity, WITH "when"
    when collision(other: HealthPickup)
        heal(other.heal_amount)    # calls the method
    << collision: HealthPickup

<< Player
```

### Rule 5 — Function Modifiers

```swift
# async — function can yield (Chapter 5)
async load_data() -> result[Data]

# export "C" — expose to other languages (Chapter 8)
export "C" skev_ai_decide(agent_id: int) -> int

# callback — C-callable with auto main-thread marshaling (Chapter 8)
callback on_collision(body_a: int, body_b: int) -> nothing
```

### Rule 6 — Functions Are Values

```swift
# Functions can be stored and passed
processor :: fn(int, int) -> int = add_numbers

# Type syntax for function values
fn(ParamType, ParamType) -> ReturnType

# Common uses:
sort_fn   :: fn(Enemy, Enemy) -> bool
callback  :: fn(int, int) -> nothing
transform :: fn(float) -> float

# Passing a function
loop active_enemies.sort_by(sort_by_distance)
```

### Rule 7 — Scope and Visibility

```swift
# Standalone functions — file-scope by default
# Accessible from other files in same package

# Hidden function — not accessible outside file
hidden calculate_internal(x: float) -> float
    result x * 1.414
<< calculate_internal

# Entity methods — accessible via entity instance
player.heal(10)
player.is_alive()
```

### Rule 8 — Recursion

```swift
# Recursion is supported
fibonacci(n: int) -> int
    if n <= 1 >>
        result n
    << n <= 1
    result fibonacci(n - 1) + fibonacci(n - 2)
<< fibonacci

# Tail recursion — compiler optimises to loop
count_down(n: int) -> nothing
    if n <= 0 >>
        result nothing
    << n <= 0
    debug.log("{n}")
    count_down(n - 1)    # tail call — optimised
<< count_down
```

---

## Language Comparison — Function Syntax

### The Problem
Every language needs a way to define reusable named blocks of logic. The syntax
varies significantly — and the design choice reveals the language's philosophy.

### In C++23
```cpp
// Free function
int clamp_health(int value, int max_health) {
    return std::clamp(value, 0, max_health);
}

// Method
int Player::heal(int amount) {
    health = clamp_health(health + amount, maxHealth);
    return health;
}
// 7 lines total — requires class declaration separately
```

### In C# 13
```csharp
// Static method (closest to free function)
static int ClampHealth(int value, int maxHealth) {
    return Math.Clamp(value, 0, maxHealth);
}

// Instance method
public int Heal(int amount) {
    health = ClampHealth(health + amount, maxHealth);
    return health;
}
// 8 lines total — PascalCase methods, static keyword needed for free functions
```

### In Python 3.13
```python
# Free function
def clamp_health(value: int, max_health: int) -> int:
    return max(0, min(value, max_health))

# Method
def heal(self, amount: int) -> int:
    self.health = clamp_health(self.health + amount, self.max_health)
    return self.health
# 6 lines total — def keyword, self parameter, colon syntax
```

### In Skev
```swift
# Free function
clamp_health(value: int, max_health: int) -> int
    result math.clamp(value, 0, max_health)
<< clamp_health

# Method (inside entity)
heal(amount: int) -> int
    health = clamp_health(health + amount, max_health)
    result health
<< heal
// 8 lines total — no keyword, << closes, result returns
```

### ✅ Honest Advantage Statement

Skev's function syntax has no dedicated keyword (`def`, `fn`, `func`) — the
function name IS the declaration. This is more minimal than all comparison
languages. C++ and C# require separate class declarations for methods. Python
requires explicit `self`. Skev methods inside entities have implicit access to
entity state — no `self` needed.

Trade-off: the absence of a function keyword means a function declaration looks
similar to a function call at first glance. Skev compensates with: the `<< name`
closing label makes functions visually distinct in any file, and the `->` return
type annotation is a clear signal.

---

## Consistency Bible — New Entries From This Document

| Decision | Rule |
|---|---|
| Function declaration | `name(params) -> ReturnType` — no keyword |
| Function body | No `>>` — body starts immediately after signature |
| Function close | `<< function_name` — same as all Skev blocks |
| Return value | `result value` for non-result types |
| Return from result fn | `succeed value` / `fail ErrorType` |
| Default parameters | `param: Type = default_value` |
| Function type syntax | `fn(Type, Type) -> ReturnType` |
| Method vs event | Method: no `when`. Event: with `when`. |
| Standalone function | Outside entity — file scope by default |
| Hidden function | `hidden` modifier — file-private |
| Implicit self | Methods inside entity access properties directly |
| Tail call optimisation | Compiler optimises tail recursive calls |

---

## What This Unlocks for Chapter 3.5 (Generics)

With function syntax formally defined, Chapter 3.5 can now define:

```swift
# Generic function — syntax to be locked in Chapter 3.5
process[T](value: T) -> T
    ...
<< process

# Generic data type — syntax to be locked in Chapter 3.5
data Pair[A, B] >>
    first  :: A
    second :: B
<< Pair
```

The `[T]` generic parameter placement is consistent with:
- `list[T]` — already locked in Chapter 3
- `result[T]` — already locked in Chapter 6
- `channel[T]` — already locked in Chapter 5

This consistency means the generic syntax will feel immediately natural
to any Skev developer who has used the standard library.

---

## Answer to the New Chat's Questions

**Question 1: Does Chapter 1 define function syntax?**
No. Chapter 1 covers philosophy and goals only. Function syntax was never
formally defined in any chapter. It was used implicitly from Chapter 6 onward
but never locked as a design decision.

**Question 2: Design now or scope to generic data types only?**
Design now — this document locks all function syntax. Chapter 3.5 should
cover BOTH generic functions AND generic data types. The syntax is consistent
with existing patterns in the spec and requires no new concepts.

**The function syntax is now fully locked in this document.**
Chapter 3.5 can proceed immediately.

---

*Version 0.1 — Function Syntax Gap Resolution*
*Created to resolve gap discovered when beginning Chapter 3.5*
