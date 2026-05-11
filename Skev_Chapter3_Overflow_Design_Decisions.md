<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 3 Addendum: Integer Overflow — Design Decisions

**Version:** 1.0
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked
**Process:** Steps 1–7 completed per Design Practices
**Adds to:** Skev_Spec_Chapter3.md as Section 3.12

---

## The Problem

Integer overflow is the silent killer of systems programs. When an integer
exceeds its type's maximum value, most languages either:
- Produce undefined behavior (C/C++) — silent data corruption
- Wrap silently (Go, Java, C#) — wrong answers with no warning
- Always panic (very safe but kills performance in hot loops)

Skev's previous spec left overflow semantics undefined — rated 🟡 partial
in the safety matrix as a result. This document locks them completely.

**Target:** Overflow rated ✅ fully defined — matching Zig's standard.

---

## Decision 1 — Default Arithmetic: Debug Panic, Release Wrap

**Syntax:** `+`, `-`, `*` (unchanged — no new syntax)

**Semantics:**
```
Debug build:   overflow → PANIC immediately
               Catches bugs. Always visible. Cannot be ignored.

Release build: overflow → wraps silently (2's complement)
               Matches CPU native behaviour. Zero overhead.
               Same as: int8 200 + 100 = 44 (wraps around)
```

**Why this choice:**
```
→ Debug: catches every overflow bug during development
→ Release: zero overhead for game loops where overflow is impossible
           (health is always clamped before this point, etc.)
→ Same model as Zig — proven correct for systems/game code
→ Developers who need different behaviour use explicit operators (below)
```

**What this is NOT:**
```
→ NOT undefined behaviour (C/C++) — behaviour is defined in both builds
→ NOT always panic — would break release performance
→ NOT always wrap — would hide bugs in development
```

**Status:** Locked ✅

---

## Decision 2 — Explicit Wrapping: `+%` `-%` `*%`

**Syntax:** `name +% value`, `name -% value`, `name *% value`

**Semantics:** Always wraps. Panics NEVER. Any build. Intentional overflow.

**When to use:**
```
→ Ring buffer indices:         index = (index +% 1) % capacity
→ Hash functions:              hash = hash *% 0x9E3779B9
→ Packet sequence numbers:     seq = seq +% 1     (wrap is correct)
→ Animation frame counters:    frame = frame +% max_frame
→ Audio sample positions:      pos = pos +% buffer_size
→ Any case where wrap is CORRECT behaviour, not a bug
```

**Status:** Locked ✅

---

## Decision 3 — Explicit Saturating: `+|` `-|` `*|`

**Syntax:** `name +| value`, `name -| value`, `name *| value`

**Semantics:** Clamps to type min/max. Never wraps. Never panics. Any build.

**Clamp values by type:**
```
int8:   clamps to -128 .. 127
int16:  clamps to -32768 .. 32767
int32:  clamps to -2147483648 .. 2147483647
int64:  clamps to INT64_MIN .. INT64_MAX
uint8:  clamps to 0 .. 255
uint16: clamps to 0 .. 65535
uint32: clamps to 0 .. 4294967295
int:    clamps to platform int64 range
```

**When to use:**
```
→ Health bars:     health -| damage       (never goes below 0)
→ Volume control:  volume +| delta        (never exceeds max)
→ Sensor values:   reading +| spike       (hardware physical limits)
→ Resource counts: gold +| reward         (never overflows budget)
→ Anywhere math.clamp was previously needed for overflow safety
```

**Saturating replaces this pattern:**
```swift
# Old — verbose, two operations:
health = math.clamp(health - damage, 0, max_health)

# New — one operation, expresses intent directly:
health -| damage
# (Saturates at 0 automatically because health :: uint declared positive)
```

**Status:** Locked ✅

---

## Decision 4 — Explicit Checked: `+!` `-!` `*!`

**Syntax:** `name +! value`, `name -! value`, `name *! value`

**Semantics:** Always panics on overflow. ANY build, including release.

**When to use:**
```
→ Safety-critical paths where overflow is ALWAYS a bug
→ Financial calculations where wrong answers are unacceptable
→ Medical device logic where overflow must never happen in production
→ Game scores/records where incorrect values are unacceptable
→ Any place where "I want to know about this in production too"
```

**Honest note:** `+!` in release adds one branch per operation. Acceptable
for safety-critical code. Not for inner loops. Use default `+` there.

**Status:** Locked ✅

---

## Decision 5 — Division and Modulo

**Division by zero:** Always panics. No special operator needed.
This was already implied. Now explicitly locked.

```swift
result :: int = a / 0     # PANIC: division by zero — any build
result :: int = a % 0     # PANIC: modulo by zero — any build
```

**Integer division truncation:** Truncates toward zero. Same as C/C++/Rust.

```swift
result :: int = 7 / 2     # 3 (not 3.5, not 4)
result :: int = -7 / 2    # -3 (truncates toward zero, not -4)
```

**Status:** Locked ✅

---

## Decision 6 — Float Overflow

**Decision: Float overflow follows IEEE 754. No special operators.**

```swift
big :: float = 1.0e38 * 1.0e38    # result: +infinity (IEEE 754)
neg :: float = -1.0e38 * 1.0e38   # result: -infinity
bad :: float = 0.0 / 0.0          # result: NaN

# Check in code:
if math.is_inf(big) >>
    handle_overflow()
<< math.is_inf(big)
```

Float overflow is DEFINED BEHAVIOUR by IEEE 754. No panic. No wrapping.
The developer checks with `math.is_inf()` and `math.is_nan()`.

**Status:** Locked ✅

---

## New Operators Summary

| Operator | Name | Behaviour | Panic? |
|----------|------|-----------|--------|
| `+` `-` `*` | Default | Debug: panic / Release: wrap | Debug only |
| `+%` `-%` `*%` | Wrapping | Always wrap | Never |
| `+|` `-|` `*|` | Saturating | Always clamp | Never |
| `+!` `-!` `*!` | Checked | Always panic | Always |
| `/` `%` | Division | Truncate / panic on zero | Zero only |

---

## Grammar Impact

Three new operator groups added to the grammar:

```
Wrapping:    [+\-*]%   matches: +%  -%  *%
Saturating:  [+\-*]\|  matches: +|  -|  *|
Checked:     [+\-*]!   matches: +!  -!  *!
```

These must be matched BEFORE single-character `+`, `-`, `*` in the
grammar to prevent partial matches.

Files to update:
```
skev.tmLanguage.json  → add overflow operator patterns
prism-skev.js         → add overflow operator patterns
sample.skev           → add overflow examples
snippets/skev.json    → no update needed (operators, not patterns)
```

---

## Concern Resolutions

| Concern | Resolution |
|---------|------------|
| Python transpiler | Python has no integer overflow. Semantics documented as limitation. |
| Shift operators | Not in current Skev spec. Separate decision — not added here. |
| Float overflow | IEEE 754 handles it — infinity/NaN. No new operators. |
| Division by zero | Already panics. Now explicitly locked. |
| `+!` performance | One branch per op in release. Acceptable for safety paths only. |
| uint types and `-|` | Saturates at 0 — correct for unsigned types. |
| Rule H | `+%` `+\|` `+!` — no trademark conflicts. |

---

## Consistency Bible Additions

| Decision | Rule |
|----------|------|
| Default overflow | Debug: panic. Release: wrap (2's complement). |
| Wrapping | `a +% b` — always wraps, never panics |
| Saturating | `a +| b` — always clamps to type min/max, never panics |
| Checked | `a +! b` — always panics on overflow, any build |
| Division by zero | Always panic. Any build. |
| Float overflow | IEEE 754. Infinity/NaN. No panic. |
| Grammar order | Overflow ops BEFORE single-char arithmetic ops |

---

*Copyright © 2026 AJ. All Rights Reserved. https://skev.dev | https://skev.org*
