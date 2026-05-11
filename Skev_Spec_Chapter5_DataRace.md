<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 5 — Section 5.8: Data Race Prevention

**Adds to:** Chapter 5 (Concurrency) as Section 5.8
**Status:** Locked ✅
**Design decisions:** Skev_Chapter5_DataRace_Design_Decisions.md
**Grammar files updated:** None — these are compiler enforcement rules

---

## 5.8 — Data Race Prevention

### 🧑‍💻 Developer Version

A data race happens when two threads read and write the same memory
at the same time without coordination. The result is unpredictable —
the program may crash, produce wrong values, or appear to work correctly
most of the time and fail under load.

Skev prevents data races with six compile-time rules. You do not need
a lock, a mutex, or a memory barrier for the common game patterns.
The rules are enforced automatically.

```
The simple version of all six rules:

  property without shared modifier:
      → only your when handlers may write it
      → safe by construction — no rule to remember

  property with shared modifier:
      → any thread may read or write it
      → compiler generates atomic operations automatically

  channel[T]:
      → sends a COPY — sender and receiver are independent
      → T must be a data type (value type) or primitive

  task block:
      → captured entity references are always weak
      → check if entity exists before using it
```

---

### 🔵 Technical Version

Skev's concurrency model assigns thread ownership at the type level,
not the value level. Entity properties without the `shared` modifier
are statically owned by the main thread — the compiler tracks which
code paths execute on which thread and rejects writes to main-thread
properties from worker-thread contexts (`task`, `realtime`).

`shared` properties are lowered to LLVM `atomicrmw` and `load atomic`
/ `store atomic` instructions with `seq_cst` ordering, providing
sequential consistency for all atomic operations.

`channel[T]` uses a lock-free SPSC (Single Producer Single Consumer)
queue for the common case. The T constraint (data type or primitive)
ensures every send is a memcpy — no pointer aliasing, no shared memory.

`task` blocks capture entity references at `WeakRef<T>` — the ARC count
is not incremented, and the compiler inserts a `maybe T` check before
any entity member access inside the task. This makes use-after-free
impossible structurally.

---

## Rule F — The Problem in Other Languages

### 🌍 Real-World Scenario

A game has three concurrent systems: the physics thread updates positions,
the AI thread reads positions to make decisions, and the main thread
renders and handles input. All three need some form of position data.

**In C++23:**
```cpp
// C++: no protection. You must add synchronisation manually.
// Forgetting one mutex lock = undefined behaviour.
struct Player { glm::vec3 position; int health; };

Player player;

// Thread 1 (physics) — writing position:
std::mutex player_mutex;
{
    std::lock_guard<std::mutex> lock(player_mutex);
    player.position += velocity * delta;
}

// Thread 2 (AI) — reading position (must also lock!):
glm::vec3 pos_copy;
{
    std::lock_guard<std::mutex> lock(player_mutex);
    pos_copy = player.position;
}
// If ANY thread forgets the lock: data race, undefined behaviour.
// The compiler does NOT check this. Race conditions in C++ ship to production.
// 14 lines total — manual, error-prone, nothing checked
```

**In C# 13:**
```csharp
// C#: also not checked. Common approach: lock() statement.
class Player {
    public Vector3 Position { get; private set; }
    private readonly object _lock = new();

    public void UpdatePosition(Vector3 delta) {
        lock (_lock) { Position += delta; }
    }
    public Vector3 GetPosition() {
        lock (_lock) { return Position; }
    }
}
// Forgetting lock() = silent data race at runtime.
// The compiler does not check. Race detector exists (dotnet-trace).
// 11 lines total — better than C++ but still manual
```

**In Python 3.13:**
```python
import threading

class Player:
    def __init__(self):
        self.position = [0.0, 0.0, 0.0]
        self._lock = threading.Lock()

    def update_position(self, delta):
        with self._lock:
            self.position = [p + d for p, d in zip(self.position, delta)]

# Python's GIL provides some protection for simple ops,
# but compound operations still need explicit locks.
# No compile-time checking. Race conditions are runtime bugs.
# 9 lines total — GIL helps but doesn't eliminate all races
```

**In Skev — six rules, checked at compile time:**
```swift
entity Player >>
    # Non-shared: main thread owns these — when handlers only
    health      :: int    = 100
    move_input  :: Vector3!

    # Shared: any thread may access — atomic automatically
    shared position :: Vector3!
    shared score    :: int = 0

    when update(delta)
        # Main thread: non-shared writes are fine here
        health -= 0    # fine — when handler, main thread
        move_input = get_input()

        # Shared write — atomic, no lock needed:
        position += move_input * delta
    << update

<< Player

# Physics task: can read shared position, cannot write non-shared
task physics_step >>
    # player.health = 0    ← COMPILE ERROR: health is non-shared
    #                         task runs on worker thread
    pos :: Vector3! = player.position    # fine: position is shared
    do_collision_check(pos)
<< task
// 26 lines total — rules enforced at compile time. No mutex. No lock.
// Forgetting to make a property shared = compile error, not runtime bug.
```

---

## 5.8.1 — Foundation

### 🟢 Rule 1: Non-Shared Properties — Main Thread Only

```swift
entity Enemy >>
    health   :: int   = 100    # non-shared: main thread only
    position :: Vector3!       # non-shared: main thread only
    shared aggro_level :: int = 0    # shared: any thread

    when update(delta)
        health -= 0            # fine: when handler = main thread
        position += Vector3(1.0, 0.0, 0.0)    # fine: main thread
        aggro_level +%= 1      # fine: shared = atomic
    << update

<< Enemy

# From a task block:
task ai_decision >>
    # enemy.health = 0       ← COMPILE ERROR
    #   health is non-shared. task runs on worker thread.
    level :: int = enemy.aggro_level    # fine: shared is readable
<< task
// 16 lines total
```

### 🟢 Rule 3: `channel[T]` — Data Types Only

```swift
# Valid channel types:
data CollisionEvent >>
    entity_a_id :: int
    entity_b_id :: int
    force       :: float
<< CollisionEvent

collision_bus :: channel[CollisionEvent]   # valid: data type
score_bus     :: channel[int]              # valid: primitive
state_bus     :: channel[GameState]        # valid: kind (enum)

# COMPILE ERROR — entities cannot be sent through channels:
# player_bus :: channel[Player]
# Error: Player is an entity type. channel[T] requires a data type.
# Reason: entities have ARC and identity — cannot be safely copied.
# Fix: create a data type with the fields you need to communicate.
// 14 lines total
```

---

## 5.8.2 — Applied

### 🟡 The Pattern: Shared State + Channel Communication

```swift
data PhysicsResult >>
    entity_id :: int
    new_position :: Vector3!
    new_velocity :: Vector3!
<< PhysicsResult

entity PhysicsWorld >>
    shared step_count :: int = 0       # main thread reads, physics writes
    result_channel :: channel[PhysicsResult]

    when fixed_update(delta)
        # Main thread: process results from physics task
        loop while result_channel.has_message >>
            result :: PhysicsResult = result_channel.receive()
            apply_physics_result(result)
        << while result_channel.has_message

        # Dispatch next physics step as a task
        step_count +%= 1      # shared: atomic increment
        task run_physics >>
            # Cannot write non-shared properties of any entity
            # CAN write to shared properties
            # CAN send data through channels
            bodies_snapshot :: list[PhysicsResult] = compute_physics(delta)
            loop r in bodies_snapshot >>
                result_channel.send(r)    # send data back to main thread
            << r
        << task
    << fixed_update

<< PhysicsWorld
// 28 lines total — clean producer/consumer.
// No mutex. No lock. Compiler enforces the boundaries.
```

### 🟡 Rule 5: Task Entity Capture Is Always Weak

```swift
entity GameManager >>
    score :: int = 0

    when level_complete
        # player is captured as weak — not as strong reference
        # task may run after player is destroyed
        task award_bonus >>
            # player is maybe Player here — must check
            if player exists >>
                player.score +!= 100    # COMPILE ERROR
                # score is non-shared — cannot write from task
                # Fix: use a channel to send the award back
                award_channel.send(100)
            << player exists
        << task
    << level_complete

<< GameManager
// 16 lines total — weak capture = no use-after-free possible.
// The maybe check is enforced — compiler rejects direct access.
```

---

## 5.8.3 — Hard Complex

### 🔴 Robot Control System — Realtime + Main Thread

**Walk-Through:** A robot has a realtime thread for motor control and a
main thread for planning. The realtime thread writes motor positions at
1kHz. The planning thread reads those positions. Both need to share
state safely — one path is time-critical, one is not.

```swift
data MotorReading >>
    motor_id  :: int
    position  :: float
    velocity  :: float
    timestamp :: float
<< MotorReading

data MotorCommand >>
    motor_id   :: int
    target_pos :: float
    max_torque :: float
<< MotorCommand

entity RobotController >>
    #! Shared: realtime thread writes, main thread reads
    shared motor_0_pos :: float = 0.0
    shared motor_1_pos :: float = 0.0
    shared motor_2_pos :: float = 0.0
    shared emergency_stop :: bool = false

    #! Non-shared: main thread only
    planned_trajectory :: list[MotorCommand]
    current_step       :: int = 0

    #! Channels: typed safe communication
    command_channel :: channel[MotorCommand]
    reading_channel :: channel[MotorReading]

    #! Realtime block: runs at 1kHz on dedicated thread
    #! Declares what it writes — compiler checks for conflicts
    realtime motor_control >> writes: motor_0_pos, motor_1_pos, motor_2_pos
        if emergency_stop >>    # shared: safe to read from any thread
            set_all_motors_zero()
            result nothing
        << emergency_stop

        if command_channel.has_message >>
            cmd :: MotorCommand = command_channel.receive()
            execute_motor_command(cmd)
        << command_channel.has_message

        #! Atomic writes — realtime thread owns these:
        motor_0_pos = read_encoder(0)
        motor_1_pos = read_encoder(1)
        motor_2_pos = read_encoder(2)

        #! Send reading to main thread via channel (data type = safe)
        reading :: MotorReading = MotorReading(
            motor_id  = 0,
            position  = motor_0_pos,
            velocity  = compute_velocity(motor_0_pos),
            timestamp = time.now()
        )
        reading_channel.send(reading)
    << realtime motor_control

    #! Main thread: planning runs here
    when update(delta)
        #! Read shared motor positions (atomic — safe from main thread):
        pos_0 :: float = motor_0_pos
        pos_1 :: float = motor_1_pos
        pos_2 :: float = motor_2_pos

        #! Plan next move if trajectory has steps:
        if current_step < planned_trajectory.count >>
            cmd :: MotorCommand = planned_trajectory[current_step]
            command_channel.send(cmd)
            current_step += 1
        << current_step < planned_trajectory.count

        #! Process readings from realtime thread:
        loop while reading_channel.has_message >>
            reading :: MotorReading = reading_channel.receive()
            log_reading(reading)
        << while reading_channel.has_message
    << update

<< RobotController
// 60 lines total
```

**Walk-Through — What the compiler checks here:**

```
1. motor_0_pos, motor_1_pos, motor_2_pos are shared:
   → realtime thread writes: atomic store ✅
   → main thread reads: atomic load ✅
   → Compiler generates atomic ops for all accesses

2. planned_trajectory, current_step are non-shared:
   → Only written in when update() — main thread ✅
   → The realtime block cannot access these
   → Compile error if attempted

3. channel[MotorCommand] and channel[MotorReading]:
   → Both T types are data types ✅
   → Send = copy. No shared memory.

4. realtime block declares writes: motor_0_pos, motor_1_pos, motor_2_pos:
   → These are shared — both realtime and main can access
   → No conflict: main reads, realtime writes (atomic)
   → If any of these were non-shared → compile error

5. emergency_stop is shared:
   → Readable from realtime thread ✅
   → Settable from main thread (planning override) ✅
```

---

## 5.8.4 — The Six Rules Quick Reference

| Rule | What it means | Error type |
|------|---------------|------------|
| 1 | Non-shared properties → main thread only | Compile error on worker write |
| 2 | `shared` properties → atomic, any thread | No error — atomic generated |
| 3 | `channel[T]` → data type or primitive only | Compile error on entity T |
| 4 | `realtime` → declares write set, conflicts caught | Compile error on conflict |
| 5 | `task` entity capture → always weak (maybe T) | Compile error on direct access |
| 6 | `when` handlers → single non-shared write gate | Enforced by Rule 1 |

---

## 5.8.5 — Honest Limitations

```
What these rules prevent:
  ✅ Non-shared property written from worker thread
  ✅ Entity reference sent through channel
  ✅ Task block directly accessing entity after possible destruction
  ✅ realtime write conflicting with non-shared main-thread property

What these rules do NOT prevent:
  ❌ Two realtime blocks writing the same shared property
     (atomic individually — but combined logic may still be wrong)
  ❌ ABA problems in shared state
  ❌ Lock-free algorithm correctness (use unsafe for these)
  ❌ Deadlock (Skev has no locks, so deadlock is rare but possible
     via blocking channel reads on both ends)

For patterns beyond these rules:
  → Use unsafe block with explicit documentation
  → Use a third-party verified concurrency library
  → Consider restructuring to use channels instead of shared state
```

---

*Copyright © 2026 AJ. All Rights Reserved. https://skev.dev | https://skev.org*
