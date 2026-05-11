<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 5 Addendum: Data Race Detection — Design Decisions

**Version:** 1.0
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked
**Process:** Steps 1–7 completed per Design Practices
**Adds to:** Skev_Spec_Chapter5.md as Section 5.8
**Grammar files updated:** NONE — semantic rules only

---

## The Problem

Skev's concurrency safety was rated 🟡 partial in the comparison document.
The mechanisms existed (`shared`, `channel[T]`, `task`, `realtime`) but the
formal compiler rules — what it must REJECT and why — were never written.

Without formal rules, a compiler author must guess. Guesses create
inconsistencies. This document eliminates that gap by locking six rules
that govern data race prevention at compile time.

**Before this document:** Skev had mechanisms. No formal guarantees.
**After this document:** Skev has six enforceable compile-time rules.

---

## What Skev's Concurrency Model Already Has

Before stating the new rules, the existing model must be understood:

```
Main thread:
  → All when handlers execute here
  → Game loop runs here
  → Entity properties are owned here

Worker threads:
  → task blocks run here (fire-and-forget, async work)
  → realtime blocks run here (time-critical control loops)

Safe communication:
  → channel[T] — typed message passing between any threads
  → shared properties — atomic read/write from any thread

Existing safety:
  → channel[T] sends a COPY of a data value — no shared memory
  → shared uses atomic operations — no torn reads/writes
```

The six rules below formalise what the compiler must enforce around
these mechanisms.

---

## Decision 1 — Entity Properties Are Main-Thread Owned

**Rule:**
```
Entity properties (non-shared) may only be written from:
  → when handlers (which always run on the main thread)
  → Entity methods called from when handlers (same thread)

Any write to a non-shared entity property from a task block
or realtime block is a COMPILE ERROR.
```

**Why:**
```
Entity properties have no atomic protection by default.
A concurrent write from a worker thread while the main thread
reads = data race = undefined behaviour.

The rule makes this structurally impossible: the compiler
knows which thread context each code block runs in.
```

**Compile error example:**
```
task slow_operation >>
    player.health -= 10    # COMPILE ERROR
    # health is a non-shared entity property
    # task blocks run on worker thread
    # Write from worker to non-shared property = data race
<< task

# Fix 1: use channel[T] to send result back to main thread
# Fix 2: make health shared (if truly needs cross-thread access)
```

**Status:** Locked ✅

---

## Decision 2 — `shared` Properties Are Thread-Safe by Contract

**Rule:**
```
Properties declared with the shared modifier may be read and
written from any thread — main thread, task block, or
realtime block.

The compiler generates atomic operations for all shared
property accesses. No explicit lock or mutex is needed.

A property is either shared (atomic) or not (main-thread only).
There is no "sometimes shared" — the modifier is permanent.
```

**Why:**
```
shared is the explicit opt-in to cross-thread access.
Atomic operations have a small cost — not all properties
should pay it. The modifier makes the cost visible.

shared score :: int = 0    # developer consciously chose atomic
health      :: int = 100   # developer consciously chose main-thread only
```

**Atomicity guarantee:**
```
Read:   atomic load  — always sees a complete value, never a torn read
Write:  atomic store — always writes a complete value, never a torn write
+=:     atomic fetch-and-add — read-modify-write is one atomic operation
```

**Status:** Locked ✅

---

## Decision 3 — `channel[T]` Requires T to be a `data` Type or Primitive

**Rule:**
```
The T in channel[T] must be:
  → A data type (value type — copied on send)
  → A primitive (int, float, bool, string)
  → A kind variant (enum — copied on send)

Attempting to send an entity reference through a channel is
a COMPILE ERROR.

channel[Enemy]    # COMPILE ERROR — Enemy is an entity
channel[int]      # valid — primitive
channel[DamageEvent]  # valid — DamageEvent is a data type
```

**Why:**
```
Sending an entity reference across a channel would:
  → Transfer ARC ownership across thread boundaries without locking
  → Allow the receiving thread to write entity properties (violates Rule 1)
  → Create undefined ARC count behaviour

data types are copied on send — the sender and receiver have
independent copies. No shared memory. No race condition possible.
```

**The transfer model:**
```
Send:    value is COPIED into the channel buffer
Receive: receiver gets their own COPY of the value
Result:  sender and receiver have completely independent values
         — modifying the received copy cannot affect the sender
```

**Status:** Locked ✅

---

## Decision 4 — `realtime` Blocks Declare Their Read/Write Set

**Rule:**
```
A realtime block must declare which entity properties it writes.
If a realtime block writes property X, and a when handler on the
main thread also writes property X without X being shared,
the compiler emits an error.

realtime motor_control >> writes: motor_position, motor_velocity
    ...
<< realtime

# If motor_position is also written in a when handler without
# shared modifier → COMPILE ERROR: write conflict
```

**Why:**
```
realtime blocks run on dedicated threads for timing guarantees.
If both the realtime thread and main thread write the same
non-shared property, one of them will see stale data or cause
a torn write.

The writes: declaration makes the conflict visible at
definition time — not at runtime.
```

**The writes: syntax:**
```swift
realtime motor_control >> writes: position, velocity
    # This block exclusively owns: position, velocity
    # Main thread may NOT write these without shared modifier
    position += velocity * delta
<< motor_control
```

**If the conflict is intentional:** make the property `shared`.
Then both threads can write it safely via atomic operations.

**Status:** Locked ✅

---

## Decision 5 — `task` Blocks Capture Entities as `weak`

**Rule:**
```
When a task block references an entity, the reference is
automatically captured as weak — it does NOT increment the
ARC count.

Inside the task block, the entity reference is a maybe T.
The task must check if the entity still exists before using it.

If the entity is destroyed before the task runs or during the
task: the weak reference becomes nothing automatically.
No use-after-free. No dangling pointer. No crash.
```

**Why:**
```
task blocks run asynchronously. Between when the task was
created and when it runs, the entity may have been destroyed
(ARC count reached zero).

Strong capture would hold the entity alive for the duration
of the task — preventing destruction even when nothing else
holds it. This is a memory leak pattern in async code.

Weak capture allows natural destruction. The task handles
the "entity gone" case explicitly via maybe T check.
```

**The pattern:**
```swift
when player_near
    task process_nearby >>
        # player is captured as weak — may be nothing by now
        if player exists >>
            do_work_with(player)
        << player exists
    << task
<< player_near
```

**Status:** Locked ✅

---

## Decision 6 — `when` Handlers Are the Single Non-Shared Write Gate

**Rule:**
```
For any non-shared entity property:
  → ONLY when handlers (and methods they call) may write it
  → This is enforced at compile time
  → No exceptions — not task, not realtime, not external calls

This makes the data flow for non-shared properties completely
predictable: only the main game loop thread touches them.
```

**Why:**
```
This rule, combined with Rules 1-5, gives a simple mental model:

  "If my property is not shared, only my when handlers touch it.
   I never have to worry about concurrent writes."

This is weaker than Rust's borrow checker (which tracks borrows
by value, not by thread ownership) but covers the common game
development case with far less complexity.
```

**Status:** Locked ✅

---

## What These Six Rules Achieve

**Before:** Skev had mechanisms, no formal guarantees.
**After:** Six enforceable rules, checked at compile time.

```
Rule 1: Non-shared properties → main thread only
Rule 2: shared → always atomic, any thread
Rule 3: channel[T] → data types only, copied on send
Rule 4: realtime → declares write set, conflicts caught early
Rule 5: task entity capture → always weak, always maybe T check
Rule 6: when handlers → single non-shared write gate
```

**The honest comparison:**
```
Rust:  borrow checker proves memory safety formally.
       Data races are compile-time impossible for ALL patterns.
       Cost: lifetimes, borrow annotations, steeper learning curve.

Skev:  six structural rules cover the common game patterns.
       Data races are compile-time impossible for the primary patterns.
       Complex patterns (lock-free algorithms) require unsafe.
       Cost: simpler than Rust for typical game code.

Verdict: Skev is significantly safer than C++/Odin/Zig.
         Skev does not match Rust's formal completeness.
         For game development: Skev's rules cover 95%+ of patterns.
```

---

## Concern Resolutions

| Concern | Resolution |
|---------|------------|
| Lock-free algorithms | Available via unsafe block — developer takes responsibility |
| `shared` atomic cost | Small but real. Only pay it for properties that need it. |
| Rule 4 syntax complexity | `writes:` declaration is optional if no write conflict exists |
| entity in channel | Compile error with clear message and fix suggestion |
| task captures strong ref | Not possible — compiler automatically captures as weak |
| Multiple realtime blocks writing same property | Compile error — must use shared |
| Grammar files | NONE updated — all rules are semantic |

---

## Consistency Bible Additions

| Rule | Behaviour |
|------|-----------|
| Non-shared entity property | Main thread only — when handlers and their callees |
| shared property | Atomic — readable/writable from any thread |
| channel[T] type constraint | T must be data type or primitive — no entities |
| task entity capture | Always weak — entity may be nothing when task runs |
| realtime write conflict | Compile error if property also written in when handler |
| when handlers | Single write gate for all non-shared properties |

---

*Copyright © 2026 AJ. All Rights Reserved. https://skev.dev | https://skev.org*
