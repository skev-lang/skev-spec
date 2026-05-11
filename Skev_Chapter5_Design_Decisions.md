<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 5: Concurrency Model — Design Decisions
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked — Chapter 5 ready to write
**Process:** Steps 1–7 completed per Design Practices v1.5
**Use Cases Tested:** All 7 — Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅

---

> This document records all design decisions made before Chapter 5 is written. Every decision here passed the Seven Question Test, was cross-checked against the Consistency Bible, real-world tested, and formally locked.

---

## Step 1 — What Chapter 5 Must Cover

```
→ Core threading model
→ How the game loop runs
→ How engine systems run in parallel
→ How developer parallel work is expressed
→ Data race prevention
→ Async I/O and file/asset loading
→ Cross-thread communication
→ Real-time thread support (robotics use case)
→ How ARC atomic operations fit the model
→ Task cancellation
→ Interaction with Chapter 4 memory model
```

---

## Decision 1 — Core Threading Model — Three Tiers

**Three-tier structured concurrency:**

```swift
Tier 1 — Game Loop (main thread — automatic):
  → Entity update events
  → Input processing
  → Scene management
  → All developer when blocks run here
  → Single-threaded semantics for developer
  → Developer never thinks about threading here

Tier 2 — Engine Systems (parallel — engine managed):
  → Physics thread (fixed timestep)
  → Rendering thread (reads entity transform snapshots)
  → Audio thread (reads audio source state)
  → Networking thread (read/write network state)
  → Developer never manages these
  → Skev engine handles all synchronisation

Tier 3 — Developer Parallel Tasks (explicit — opt-in):
  → AI batch processing
  → Pathfinding for many agents
  → Procedural generation
  → Asset loading
  → Developer opts in using task keyword
  → Language guarantees safety
```

**Why not raw threads (C++ style):** Data races, mutex hell, deadlocks — violates "easy to read like Python" identity and "reliable" parameter.

**Why not single-threaded (JavaScript style):** Modern CPUs have 8-32 cores. Single-threaded wastes them. Violates "fast like C++" identity.

**Why structured concurrency:** Safe by default, parallel where beneficial, game-specific design avoids typical structured concurrency overhead.

**Five Question Test:** ✅ All five passed
**Status:** Locked ✅

---

## Decision 2 — Data Race Prevention — Ownership Per Tier

**Rule 1 — Tier 1 owns entity properties:**
All entity properties are written only on main thread. Engine system threads read only. Cross-thread write = compile error.

**Rule 2 — Engine systems own their data:**
```
Physics owns:    velocity, angular velocity, collision state
Renderer owns:   transform snapshots (copies of positions)
Audio owns:      volume, pitch, play state
Networking owns: sync state
```

**Rule 3 — shared modifier for cross-thread properties:**
```swift
entity Player >>
    health    :: int = 100           # main thread only
    position  :: Vector3!            # main thread only
    shared network_id :: uint32 = 0  # safe cross-thread access
<< Player
```
`shared` properties use automatic atomic access. No manual mutex. No manual lock.

**Rule 4 — Tasks get isolated snapshots:**
Tasks receive read-only copies of any data accessed. Cannot accidentally race with main thread.

**Compiler enforcement:**
```
Error: Cross-thread property access
  "health" is not marked shared and
  cannot be accessed from a task.
  Line 24 — player.skev
```swift

**Status:** Locked ✅

---

## Decision 3 — The `task` Keyword

**Syntax:**
```swift
task find_paths >>
    loop agent in active_agents >>
        agent.path = pathfinder.find(
            agent.position,
            agent.target.position
        )
    << agent
<< find_paths

await find_paths
```

**Rules:**
```swift
task name >>    opens parallel task block
    ...         runs on worker thread from pool
<< name         closes task block

await name      waits for task to complete

Multiple tasks in parallel:
await task_a, task_b, task_c
```

**Safety model:**
- Task receives snapshots of external entity data — not live references
- Cannot write to main thread entities directly
- Returns results merged after await
- No shared mutable state — structurally prevents races

**Task + ARC (Concern 1 resolution):**
Tasks capture entity references as WEAK snapshots. Does NOT increment ARC count. If entity destroyed during task — reference becomes nothing. Task must check `if entity exists` after any yield point. Compiler enforces this.

**Status:** Locked ✅

---

## Decision 4 — Double-Buffered Transform System

**Problem:** Physics writes position while renderer reads it = visual glitch.

**Solution — invisible to developer:**
```
Buffer A — "Current" — physics writes every step
Buffer B — "Previous" — renderer reads safely

Each frame:
→ Renderer finishes reading Buffer B
→ Engine atomically swaps A and B (one atomic op)
→ Physics writes to new Buffer A
→ Renderer reads from new Buffer B
→ No lock during frame. Zero visual glitch.
```

Developer writes `position += velocity * delta`. Engine handles everything else. Zero API surface. Zero configuration.

**Status:** Locked ✅

---

## Decision 5 — `async` / `await` for I/O

**Syntax:**
```swift
async load_next_level()
    level_data :: LevelData = await file.load("level_2.skev_level")
    textures   :: list[Texture] = await assets.load_batch(level_data.textures)
    current_level = Level.build(level_data, textures)
    loading = false
<< load_next_level
```

**Rules:**
```
async keyword:    marks a function that can yield
await expression: yields this async function until result ready
                  resumes automatically when done
                  game loop continues running — no freeze

Restrictions:
→ async functions only callable from when blocks
  or other async functions
→ Cannot write entity properties during a yield point
→ when blocks cannot await directly —
  must call an async function instead
  (preserves game loop timing guarantee)
```swift

**await in when blocks (Concern 2 resolution):**
The when block calls an async function. Async function hits await. The update event completes and returns. Async function resumes next frame when ready. Game never freezes. Main thread is never suspended.

**Status:** Locked ✅

---

## Decision 6 — `channel[T]` for Cross-Thread Communication

**Syntax:**
```swift
# Declaration
incoming_damage :: channel[DamageEvent]

# Send from any thread — copies value
incoming_damage.send(event)

# Receive on main thread
loop while incoming_damage.has_messages >>
    event :: DamageEvent = incoming_damage.receive()
    apply_damage(event)
<< while incoming_damage.has_messages
```

**Rules:**
```swift
channel[T]                    thread-safe message queue
.send(value)                  sends a COPY — safe from any thread
.receive()                    main thread only
.has_messages                 bool — thread safe read
.drain()                      returns list[T] of all pending

channel[T, capacity: N]       custom capacity
channel[T, overflow: block]   block sender when full
channel[T, overflow: drop_newest]  drop new when full
Default: capacity 256, overflow: drop_oldest
```

**Backpressure (Concern 3 resolution):**
Default capacity 256. When full: drop oldest, fire channel_overflow warning event. Developer can configure. Default is appropriate for games — new events matter more than old ones.

**Status:** Locked ✅

---

## Decision 7 — Task Cancellation

**Syntax:**
```swift
task load_level >>
    ...
<< load_level

cancel load_level    # signals cancellation

# Inside task — check for cancellation:
async load_assets()
    loop asset in asset_list >>
        if task.cancelled >>
            stop
        << task.cancelled
        await assets.load(asset)
    << asset
<< load_assets
```swift

**Model:** Cooperative cancellation. cancel sets a flag. Task must check `if task.cancelled` and stop cleanly. Resources released via ARC automatically on exit. Same approach as Swift, Kotlin, .NET — proven and safe.

**Status:** Locked ✅

---

## Decision 8 — `realtime` Modifier for Robotics

**Syntax:**
```swift
task control_loop realtime(priority: high, interval: 1ms) >>
    read_sensors()
    compute_control()
    apply_actuators()
<< control_loop
```

**Compiler-enforced constraints for realtime tasks:**
```
❌ No heap allocation inside realtime task
❌ No await (breaks timing)
❌ No channel access (latency risk)
✅ Stack-allocated data only
✅ Pre-allocated buffers only
✅ Compiler enforces ALL of the above
✅ Violation = compile error with clear message
```

**Why this matters for robotics:** Hard real-time control loops at fixed interval require deterministic timing. Standard task scheduling has jitter. realtime modifier tells the OS scheduler to give this task high priority and fixed timing.

**Status:** Locked ✅

---

## New Keywords Added to Consistency Bible

| Keyword | Purpose |
|---|---|
| `task` | Parallel work block — uses >> << syntax |
| `async` | Marks function that can use await |
| `await` | Yields async function until result ready |
| `cancel` | Signals cooperative cancellation of a task |
| `shared` | Marks property safe for cross-thread atomic access |
| `realtime` | Expert modifier for hard real-time tasks |
| `channel[T]` | Thread-safe message queue type |

---

## Use Case Verification

| Decision | Games | XR/VR | Sim | Robotics | Anim/VFX | Interactive | Net |
|---|---|---|---|---|---|---|---|
| Three-tier model | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Data race prevention | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| task keyword | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Double-buffer | ✅ | ✅ | ✅ | ⚠️ N/A | ✅ | ⚠️ N/A | ⚠️ N/A |
| async/await | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| channel[T] | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| realtime modifier | ⚠️ Rare | ✅ | ✅ | ✅ Critical | ⚠️ Rare | ⚠️ Rare | ✅ |

---

## Full Decision Summary

| Decision | Rule | Status |
|---|---|---|
| Threading model | Three tiers — game loop / engine / developer tasks | Locked ✅ |
| Main thread ownership | All entity writes — single-threaded for developer | Locked ✅ |
| Engine system parallel | Physics, renderer, audio, net — engine managed | Locked ✅ |
| task keyword | Parallel work block — >> << syntax | Locked ✅ |
| task + ARC | Weak snapshots — no ARC increment | Locked ✅ |
| async keyword | Marks awaitable functions | Locked ✅ |
| await keyword | Yields function — never blocks game loop | Locked ✅ |
| async in when | Via async function call only — never direct | Locked ✅ |
| shared modifier | Atomic cross-thread property access | Locked ✅ |
| channel[T] | Thread-safe queue — default capacity 256 | Locked ✅ |
| channel overflow | Default drop_oldest — configurable | Locked ✅ |
| Double-buffer | Engine managed — invisible to developer | Locked ✅ |
| cancel keyword | Cooperative cancellation — flag-based | Locked ✅ |
| realtime modifier | Hard real-time — heap/await/channel forbidden | Locked ✅ |
| Data race prevention | Compiler enforces thread ownership | Locked ✅ |

---

*All decisions locked — Chapter 5 writing begins next*
*Version 0.1 — Pre-Chapter 5 Design Decisions*
