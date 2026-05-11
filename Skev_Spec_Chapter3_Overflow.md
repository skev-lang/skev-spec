<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 3 — Section 3.12: Integer Overflow

**Adds to:** Chapter 3 (Data Types) as Section 3.12
**Status:** Locked ✅
**Design decisions:** Skev_Chapter3_Overflow_Design_Decisions.md

---

## 3.12 — Integer Overflow

### 🧑‍💻 Developer Version

Integer overflow is when arithmetic produces a result larger than the type
can hold. In C++, this is undefined behaviour — the compiler can do anything.
In Go and Java, it silently wraps — wrong answers with no warning.
In Skev, overflow behaviour is fully defined and always visible.

**Four explicit choices:**

```
+   -   *     Default:   debug panics, release wraps
+%  -%  *%    Wrapping:  always wraps, never panics
+|  -|  *|    Saturating: always clamps to min/max, never panics
+!  -!  *!    Checked:   always panics — even in release builds
```

The operator you choose expresses your INTENT. The compiler enforces it.

---

### 🔵 Technical Version

Default integer arithmetic (`+`, `-`, `*`) generates checked instructions
in debug builds — any overflow fires a `SkevPanic` before the result is
stored. In release builds, the compiler emits native 2's complement wrap
instructions with no overhead.

Wrapping operators (`+%`, `-%`, `*%`) always emit native wrap instructions
regardless of build type. Saturating operators (`+|`, `-|`, `*|`) emit LLVM
`llvm.sadd.sat` / `llvm.uadd.sat` intrinsics — single CPU instructions on
x86 (VPADDSSB etc.) and ARM (QADD). Checked operators (`+!`, `-!`, `*!`)
emit LLVM `llvm.sadd.with.overflow` with a branch — one extra instruction
in release builds.

---

## Rule F — The Problem in Other Languages

### 🌍 Real-World Scenario

A game tracks player score (accumulates), health (should not go below 0),
and a ring buffer index (must wrap). These are three different overflow
requirements in one system. Every language handles them differently.

**In C++23:**
```cpp
// C++: overflow on signed integers = undefined behaviour.
// The compiler is allowed to assume it never happens.
// In practice: wraps on most platforms, crashes on others.
int score = INT_MAX;
score += 100;   // UNDEFINED BEHAVIOUR — compiler may optimise away
                // the entire branch, delete loop guards, anything.

// To be safe you must check manually:
if (score > INT_MAX - 100) { /* handle */ } else { score += 100; }

// Saturating in C++23 — finally added:
#include <numeric>
score = std::add_sat(score, 100);   // C++26 actually

// Ring buffer index — must wrap: manual masking:
index = (index + 1) & (CAPACITY - 1);
// 9 lines total — three different patterns for three overflow needs
```

**In C# 13:**
```csharp
// C#: default wraps silently — dangerous
int score = int.MaxValue;
score += 100;   // silently wraps to int.MinValue — no exception!

// Must opt-in to checked arithmetic:
checked {
    score += 100;  // now throws OverflowException
}

// Saturating: no built-in, must write manually:
score = (int)Math.Min((long)score + 100, int.MaxValue);

// Ring buffer: no built-in wrapping operator:
index = (index + 1) % capacity;
// 10 lines total — checked block, manual saturate, manual modulo
```

**In Python 3.13:**
```python
# Python: integers never overflow (arbitrary precision).
# This is safe but: game engines need fixed-width integers for SIMD.
# A "score" in Python can grow to millions of digits — no overflow.
score = 2**63 - 1
score += 100   # fine — Python just makes it bigger

# For fixed-width (game/systems code), must use ctypes or numpy:
import ctypes
score = ctypes.c_int32(2**31 - 1)
score.value += 100   # wraps — but loss of type safety
# 5 lines total — no fixed-width integer overflow semantics by default
```

**In Skev — four operators, one for each intent:**
```swift
entity ScoreSystem >>
    score       :: int = 0          # accumulates — may overflow
    health      :: int = 100        # clamps — must not go below 0
    ring_index  :: uint32 = 0       # ring buffer — wrapping is correct
    safe_tally  :: int = 0          # financial — never wrong

    when enemy_killed(points: int)
        score +!= points            # checked: wrong score = bug always
    << enemy_killed

    when damage_taken(amount: int)
        health -|= amount           # saturating: health clamps at 0
    << damage_taken

    when advance_ring()
        ring_index +%= 1            # wrapping: correct for ring buffer
    << advance_ring

    when add_currency(amount: int)
        safe_tally +!= amount       # checked: financial — always verify
    << add_currency

<< ScoreSystem
// 23 lines total — four overflow needs, one operator each.
// Intent is visible at the call site. No manual checks. No clamping calls.
```

---

## 3.12.1 — Foundation

### 🟢 Default Behaviour

```swift
# Default: + - * panic in debug, wrap in release
value :: int = 100
value += 50        # fine — no overflow
value += 999999999999   # debug: PANIC   release: wraps

# Seeing a panic in debug = good. Fix the code.
# Common fix: use the right operator for your intent (see below).
// 5 lines total
```

### 🟢 Choosing the Right Operator

```swift
# health should never go below 0 → saturating
health   :: int = 100
health -|= 30    # 70   (subtracted)
health -|= 200   # 0    (clamped at 0, not -100)
health -|= 999   # 0    (still 0, not negative)

# score uses a ring buffer position → wrapping
ring_pos :: uint8 = 250
ring_pos +%= 10    # 4   (wrapped: 250 + 10 = 260, 260 % 256 = 4)
ring_pos +%= 10    # 14  (continued from 4)

# critical tally must never be wrong → checked
tally :: int = 0
tally +!= 100      # 100 (fine)
tally +!= 99999    # 99199 (fine)
# if overflow: PANIC even in release build
// 14 lines total
```

---

## 3.12.2 — Applied

### 🟡 Health and Resource System

```swift
entity ResourceManager >>
    health     :: int = 100
    max_health :: int = 100
    gold       :: uint32 = 0
    stamina    :: uint8 = 255

    take_damage(amount: int) -> nothing
        health -|= amount    # clamps at 0 — never goes negative
        # No math.clamp needed. Saturating handles it.
    << take_damage

    heal(amount: int) -> nothing
        # Would overshoot max_health? Clamp at max_health manually:
        remaining :: int = max_health - health
        health +|= math.min(amount, remaining)
    << heal

    earn_gold(amount: uint32) -> nothing
        gold +|= amount      # clamps at uint32 max — no overflow
    << earn_gold

    use_stamina(amount: uint8) -> nothing
        stamina -|= amount   # clamps at 0 — never wraps to 255
    << use_stamina

<< ResourceManager
// 24 lines total — no explicit clamping calls.
// -| expresses the intent directly.
```

### 🟡 Ring Buffer for Audio

```swift
data AudioBuffer >>
    samples   :: list[float]
    capacity  :: uint32
    write_pos :: uint32 = 0
    read_pos  :: uint32 = 0
<< AudioBuffer

entity AudioMixer >>
    buffer :: AudioBuffer

    write_sample(sample: float) -> nothing
        buffer.samples[buffer.write_pos] = sample
        # Wrapping: when write_pos reaches capacity, it rolls back to 0
        # This is CORRECT for a ring buffer — wrapping is the feature
        buffer.write_pos +%= 1
        if buffer.write_pos >= buffer.capacity >>
            buffer.write_pos = 0
        << buffer.write_pos >= buffer.capacity
    << write_sample

<< AudioMixer
// 16 lines total — +%= makes the wrapping intent explicit.
```

---

## 3.12.3 — Hard Complex

### 🔴 Safety-Critical Robot Motor Controller

**Walk-Through:** A robot motor receives position commands. A position
error (overflow) must NEVER produce a wrong motor command — even in a
release build. `+!` ensures any overflow triggers a panic, which the
robot's safety system catches and executes an emergency stop.

```swift
kind MotorError >>
    position_overflow
    velocity_overflow
    command_invalid
    hardware_fault
<< MotorError

data MotorCommand >>
    target_position :: int32    # millimetres from home
    max_velocity    :: int32    # mm/s
    acceleration    :: int32    # mm/s²
<< MotorCommand

entity MotorController >>
    #! safety_critical: any panic triggers emergency stop
    current_position :: int32 = 0
    current_velocity :: int32 = 0
    error_count      :: uint16 = 0

    execute_command(cmd: MotorCommand) -> result[int32]
        if cmd.target_position < -500000 or cmd.target_position > 500000 >>
            fail MotorError.command_invalid
        << cmd.target_position < -500000 or cmd.target_position > 500000

        # +! in ALL builds — a wrong position command is always a bug.
        # If this panics in production, the safety system does an
        # emergency stop — that is BETTER than a wrong position.
        delta_pos :: int32 = current_position +! cmd.target_position
        new_vel   :: int32 = current_velocity +! cmd.acceleration

        # Velocity IS allowed to clamp — saturation is safe here:
        clamped_vel :: int32 = new_vel +| 0    # saturate at signed range
        if clamped_vel > cmd.max_velocity >>
            clamped_vel = cmd.max_velocity
        << clamped_vel > cmd.max_velocity

        current_velocity = clamped_vel
        current_position +!= delta_pos        # checked: position drift = always wrong

        succeed current_position
    << execute_command

    log_error() -> nothing
        error_count +%= 1     # wrapping: log counter rolls over — that is fine
    << log_error

<< MotorController
// 38 lines total
```

**Walk-Through — Why each operator:**
```
current_position +!= delta_pos   → checked: position drift in production
                                    is always a bug. Panic and stop safely.

new_vel +| 0                      → saturating: velocity clamping is safe.
                                    A motor going slower than commanded is OK.
                                    A motor at wrong position is not OK.

error_count +%= 1                 → wrapping: log counter. Rolling over
                                    at 65535 back to 0 is fine — it is
                                    a counter, not a position.
```

---

## 3.12.4 — Augmented Assignment Forms

All overflow operators have augmented assignment forms:

```swift
# Standard form:          Augmented form:
a = a +% b               a +%= b
a = a -% b               a -%= b
a = a *% b               a *%= b
a = a +| b               a +|= b
a = a -| b               a -|= b
a = a *| b               a *|= b
a = a +! b               a +!= b
a = a -! b               a -!= b
a = a *! b               a *!= b
// 9 lines total
```

---

## 3.12.5 — Quick Reference

| Operator | Name | Debug | Release | Use for |
|----------|------|-------|---------|---------|
| `+` `-` `*` | Default | panic | wrap | General arithmetic |
| `+%` `-%` `*%` | Wrapping | wrap | wrap | Ring buffers, hash, sequence numbers |
| `+\|` `-\|` `*\|` | Saturating | clamp | clamp | Health, volume, resource counts |
| `+!` `-!` `*!` | Checked | panic | panic | Safety-critical, financial |
| `/` `%` | Division | zero→panic | zero→panic | Division and modulo |

---

*Copyright © 2026 AJ. All Rights Reserved. https://skev.dev | https://skev.org*
