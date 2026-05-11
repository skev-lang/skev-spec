<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 5: Concurrency Model
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026)
**Status:** In Progress
**Depends On:** Chapter 4 v0.1 — Memory model and ARC atomic operations locked
**Use Cases Tested:** All 7 — Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅

---

## 5.1 — Overview

### 👨‍💻 Developer Version

Modern hardware has 8, 16, even 32 CPU cores. A language that only uses one of them is wasting your hardware — and your players' hardware.

But concurrency is also the source of the most difficult bugs in programming: data races, deadlocks, and timing errors that only appear in production under specific conditions that you cannot reproduce.

Skev solves both problems simultaneously:

> **Use all available CPU cores automatically.**
> **Make data races structurally impossible.**
> **Without making you think about threads.**

This is Skev's concurrency philosophy:

```
Tier 1 — Game Loop:        You write code here. Always safe.
Tier 2 — Engine Systems:   Runs in parallel automatically.
Tier 3 — Developer Tasks:  Opt-in parallel work. Always safe.
```

Most developers never leave Tier 1. Their game still runs on all CPU cores because the engine handles Tier 2 automatically. When you need more — Tier 3 is there, and the compiler guarantees it is safe.

### ⚙️ Technical Version

Skev implements structured concurrency with three distinct execution tiers. Tier 1 (game loop) provides single-threaded semantics for developer code — all `when` event handlers execute on a single main thread with sequential memory consistency. Tier 2 (engine systems) runs physics, rendering, audio, and networking on separate OS threads managed by the Skev engine runtime — developer code does not interact with these threads directly. Tier 3 (developer tasks) uses a work-stealing thread pool with snapshot-based data isolation — tasks receive immutable snapshots of entity data at creation time, making data races structurally impossible at the language level. ARC reference count operations, established as atomic in Chapter 4, are the foundation upon which the concurrency safety model is built.

---

## 5.2 — The Three-Tier Threading Model

### 👨‍💻 Developer Version

**Tier 1 — The Game Loop (Main Thread)**

This is where your code lives. Every `when` event handler runs here. You write normal, readable Skev code with no concurrency concerns whatsoever.

```swift
entity Player >>
    health   :: int   = 100
    position :: Vector3!

    # This runs on the main thread — always safe
    when update(delta)
        position += input.direction * speed * delta
        health   -= environment.damage_per_second * delta
    << update

    # This runs on the main thread — always safe
    when collision(other: Enemy)
        health -= other.attack_power
        audio.play(sounds.hurt)
    << collision: Enemy

<< Player
```

**What runs on the main thread:**
```
→ All "when" event handlers
→ All async function completions
→ All task result processing (after await)
→ Input processing
→ Scene management
→ UI updates
```

**What you never worry about:**
```
→ Thread safety of entity properties
→ Synchronisation with physics
→ Synchronisation with rendering
→ Synchronisation with audio
All handled automatically by the engine.
```

---

**Tier 2 — Engine Systems (Automatic Parallel)**

The Skev engine runs these systems in parallel with your game loop — automatically, invisibly, correctly:

```
Physics Thread:
  → Runs at fixed timestep (default 60Hz)
  → Reads entity positions and velocities
  → Writes back physics results
  → Synchronised with main thread via double-buffer

Rendering Thread:
  → Reads entity transform snapshots
  → Never reads live entity data — always a snapshot
  → Runs at display refresh rate (60/120/144Hz)
  → Completely decoupled from game logic speed

Audio Thread:
  → Reads AudioSource component state
  → Streams audio data independently
  → Zero impact on game loop timing

Networking Thread:
  → Handles socket I/O independently
  → Communicates with main thread via channel[T]
  → Never blocks game loop waiting for network
```swift

**You never interact with these threads directly. They are part of the engine — not the language.** Your entities just work correctly across all of them.

---

**Tier 3 — Developer Tasks (Explicit Parallel)**

When you need to run your own heavy computation in parallel — this is where it goes.

```swift
entity AIDirector >>

    when update(delta)
        # Without tasks — runs on main thread — blocks game loop
        loop agent in active_agents >>      # 200 agents
            agent.path = pathfinder.find(   # expensive!
                agent.position,
                agent.target_position
            )
        << agent

        # With tasks — runs in parallel — game loop continues
        task compute_paths >>
            loop agent in active_agents >>
                agent.path = pathfinder.find(
                    agent.position,
                    agent.target_position
                )
            << agent
        << compute_paths

        await compute_paths    # wait for results

        # Results available — apply on main thread
        loop agent in active_agents >>
            agent.begin_following_path(delta)
        << agent
    << update

<< AIDirector
```

**The three-tier model across use cases:**

| Use Case | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|
| Games | Entity logic | Physics, render, audio | AI, pathfinding |
| XR/VR | Interaction logic | Render (90fps), tracking | Scene generation |
| Simulation | Agent logic | Physics, rendering | Batch computation |
| Robotics | Decision logic | Sensor reading | Path planning |
| Animation/VFX | Tool logic | Render preview | Batch processing |
| Interactive Apps | UI logic | Render | Data processing |
| Real-time Net | Game logic | Network I/O | Data analysis |

### ⚙️ Technical Version

Tier 1 executes on a dedicated main thread. The main thread's event queue is processed sequentially — one `when` handler completes before the next begins. This provides sequential consistency for all entity property reads and writes on the main thread without locks.

Tier 2 threads are created by the Skev engine runtime at startup — one thread per system (physics, renderer, audio, networking). These threads run at their respective rates independently. Synchronisation with Tier 1 uses the double-buffer transform system (section 5.7) for spatial data, and `shared` properties with atomic access for other cross-tier data.

Tier 3 uses a work-stealing thread pool sized to `hardware_concurrency - 2` threads (reserving threads for main and render). Tasks submitted via `task` blocks are queued to the pool. Work-stealing ensures CPU utilisation stays high when tasks have uneven execution times. The snapshot isolation model (section 5.4) ensures Tier 3 tasks cannot create data races with Tier 1.

---

## 5.3 — Data Race Prevention

### 👨‍💻 Developer Version

A data race happens when two threads read and write the same data at the same time. In C++ and C# this produces silent corruption, crashes, and bugs that only appear on certain machines. In Skev — data races are caught at compile time before your game ever runs.

**The ownership rule is simple:**
```
Entity properties belong to the main thread.
Only the main thread can write them.
Other threads can only read copies.
```

**Trying to write from the wrong place — compile error:**

```swift
entity Player >>
    health :: int = 100
<< Player

# On main thread — fine ✅
when update(delta)
    health -= 10
<< update

# Inside a task — compile error ❌
task dangerous >>
    health -= 10    # Error: writing entity property from task
<< dangerous
```

```
Error: Entity property write from task context
  "health" is a main-thread property and cannot
  be written inside a task block.

  Tasks receive read-only snapshots of entity data.

  Options:
  1. Return the new value and apply it after await
  2. Send a message via channel[T] to main thread
  3. Move this logic to a when block

  Line 3 — player.skev
```swift

**The `shared` modifier — intentional cross-thread access:**

For properties that genuinely need to be read from multiple threads, use `shared`. This marks the property for atomic access automatically — no manual lock needed:

```swift
entity NetworkPlayer >>
    health        :: int    = 100    # main thread only
    position      :: Vector3!        # main thread only
    shared network_id   :: uint32 = 0   # readable from network thread
    shared session_key  :: string = ""  # readable from network thread
<< NetworkPlayer
```

**`shared` rules:**
```
→ Reads from any thread — atomic
→ Writes from main thread only — atomic
→ No manual mutex
→ No manual atomic operations
→ Compiler generates atomic access automatically
→ Performance: ~3-5ns per access — same as ARC count
```

### Rule F — The Problem in Other Languages

**C++23:**
```cpp
// C++ has no compile-time data race detection
// This silently corrupts health under concurrency:
class Player {
    int health = 100;  // no protection

    void TakeDamage(int amount) {
        health -= amount;  // not thread-safe — silent bug
    }
};

// Developer must manually add synchronisation:
class Player {
    std::atomic<int> health{100};  // explicit atomic
    // OR
    std::mutex health_mutex;
    int health = 100;
    void TakeDamage(int amount) {
        std::lock_guard<std::mutex> lock(health_mutex);
        health -= amount;
    }
};
// Manual. Error-prone. Easy to forget.
// No compile-time verification that you got it right.
```

**C# 13:**
```csharp
// C# has lock() but no compile-time enforcement
class Player {
    private int health = 100;
    private readonly object healthLock = new object();

    public void TakeDamage(int amount) {
        lock (healthLock) {
            health -= amount;
        }
    }
}
// Better than C++ but still manual.
// Compiler does not verify you locked correctly.
// Easy to access health without the lock accidentally.
```

**Python 3.13:**
```python
# Python GIL prevents true parallelism for CPU-bound tasks
# Threading works but GIL means only one thread executes at once
# For game use: threading gives no performance benefit for game logic
import threading

class Player:
    def __init__(self):
        self.health = 100
        self._lock = threading.Lock()

    def take_damage(self, amount):
        with self._lock:
            self.health -= amount
# Slow (GIL), manual, no compile-time verification
```swift

**Skev:**
```swift
entity Player >>
    health :: int = 100    # compiler enforces main-thread only

    when collision(other: Enemy)
        health -= other.attack_power    # safe — main thread
    << collision: Enemy

<< Player
# Zero boilerplate. Compile-time enforced.
# No mutex. No lock. No atomic keyword.
# Data race is structurally impossible.
```

**Honest advantage statement:**
Skev eliminates data races on entity properties by making cross-thread writes a compile error. No other language in this comparison does this at compile time. The developer writes zero synchronisation code for the common case — and the `shared` modifier handles the uncommon case with automatic atomic access. This is not just convenience — it is a structural guarantee.

### ⚙️ Technical Version

Thread ownership is tracked in the entity's type descriptor, which records the thread affinity of each property. During Tier 3 task compilation, the compiler performs an ownership check on every property access — properties not marked `shared` and not owned by the current task's snapshot produce a `CROSS_THREAD_WRITE` error.

`shared` properties are stored with an alignment guarantee matching the platform's native atomic word size (8 bytes on x86-64 and ARM64). Read access compiles to `atomic_load` with relaxed memory ordering — sufficient for independent reads. Write access compiles to `atomic_store` with release memory ordering — ensuring writes are visible to all threads before the next main-thread event processes.

---

## 5.4 — The `task` Keyword

### 👨‍💻 Developer Version

`task` is Skev's way to run work in parallel. It is opt-in, safe, and reads almost exactly like regular Skev code.

**Basic task:**
```swift
entity WorldGenerator >>

    when scene_load
        # Generate the world in parallel — game loop continues
        task generate_world >>
            terrain.generate(seed, biome_map)
            vegetation.populate(terrain, density_map)
            enemy_spawns.calculate(terrain, difficulty)
        << generate_world

        # Meanwhile main thread handles loading screen
        ui.show_loading_screen()

        # Wait until generation complete
        await generate_world

        # Apply results on main thread
        scene.build_from(terrain, vegetation, enemy_spawns)
        ui.hide_loading_screen()
    << scene_load

<< WorldGenerator
```swift

**Multiple tasks in parallel:**
```swift
entity FrameProcessor >>

    when update(delta)
        # Fire three tasks simultaneously
        task update_ai >>
            process_all_ai_decisions(active_enemies, delta)
        << update_ai

        task simulate_particles >>
            particle_system.simulate(active_particles, delta)
        << simulate_particles

        task stream_next_chunk >>
            world_stream.preload_adjacent_chunks(player.position)
        << stream_next_chunk

        # Wait for ALL three simultaneously — most efficient
        await update_ai, simulate_particles, stream_next_chunk

        # All results available and safe to use
        apply_ai_results(active_enemies)
        particle_system.apply_results(active_particles)
    << update

<< FrameProcessor
```

**What tasks can and cannot do:**

```swift
entity ExampleSystem >>
    data_list :: list[float]
    result    :: float = 0.0

    when update(delta)
        task compute >>
            # ✅ Can: read entity properties (snapshot)
            total :: float = 0.0

            # ✅ Can: use local variables freely
            loop value in data_list >>   # data_list snapshot
                total += value
            << value

            # ✅ Can: call pure functions
            processed :: float = math.sqrt(total)

            # ✅ Can: write to LOCAL variables
            local_result :: float = processed * 2.0

            # ❌ Cannot: write to entity properties directly
            result = local_result    # Error: cross-thread write

        << compute

        await compute

        # ✅ Can: apply results after await — back on main thread
        result = compute.output    # see section 5.4.1 below
    << update

<< ExampleSystem
```swift

**Returning values from tasks:**

```swift
when update(delta)
    task find_nearest_enemy >>
        nearest :: maybe Enemy = nothing
        min_dist :: float = 999999.0

        loop enemy in active_enemies >>
            dist :: float = Vector3!.distance(
                player.position,
                enemy.position
            )
            if dist < min_dist >>
                min_dist = dist
                nearest  = enemy
            << dist < min_dist
        << enemy

        # Return value from task using "result"
        task.result = nearest
    << find_nearest_enemy

    await find_nearest_enemy

    # Safely access result on main thread
    if find_nearest_enemy.result exists >>
        player.target = find_nearest_enemy.result
    << find_nearest_enemy.result exists
<< update
```

### Rule F — Real-World Example: Game Development

**200 AI agents pathfinding — the classic game bottleneck:**

```swift
entity AICoordinator >>
    active_agents :: list[AIAgent]
    nav_mesh      :: NavigationMesh

    when update(delta)
        # Batch pathfinding for all agents in parallel
        # Split into groups of 50 for optimal thread utilisation
        task paths_group_a >>
            loop agent in active_agents[0..49] >>
                agent.current_path = nav_mesh.find_path(
                    agent.position,
                    agent.destination
                )
            << agent
        << paths_group_a

        task paths_group_b >>
            loop agent in active_agents[50..99] >>
                agent.current_path = nav_mesh.find_path(
                    agent.position,
                    agent.destination
                )
            << agent
        << paths_group_b

        task paths_group_c >>
            loop agent in active_agents[100..149] >>
                agent.current_path = nav_mesh.find_path(
                    agent.position,
                    agent.destination
                )
            << agent
        << paths_group_c

        task paths_group_d >>
            loop agent in active_agents[150..199] >>
                agent.current_path = nav_mesh.find_path(
                    agent.position,
                    agent.destination
                )
            << agent
        << paths_group_d

        # All four run in parallel
        await paths_group_a, paths_group_b, paths_group_c, paths_group_d

        # All paths computed — apply movement on main thread
        loop agent in active_agents >>
            agent.follow_path(delta)
        << agent
    << update

<< AICoordinator
```swift

### Rule F — Real-World Example: Simulation

**Climate simulation — batch atmospheric calculation:**

```swift
entity AtmosphericSimulator >>
    grid_cells :: array[ClimateCell, 10000]    # 100×100 grid

    when fixed_update(delta)
        # Divide grid into quadrants for parallel processing
        task simulate_quadrant_a >>
            loop i from 0 to 2499 >>
                grid_cells[i].simulate_pressure(delta)
                grid_cells[i].simulate_temperature(delta)
                grid_cells[i].simulate_humidity(delta)
            << i
        << simulate_quadrant_a

        task simulate_quadrant_b >>
            loop i from 2500 to 4999 >>
                grid_cells[i].simulate_pressure(delta)
                grid_cells[i].simulate_temperature(delta)
                grid_cells[i].simulate_humidity(delta)
            << i
        << simulate_quadrant_b

        task simulate_quadrant_c >>
            loop i from 5000 to 7499 >>
                grid_cells[i].simulate_pressure(delta)
                grid_cells[i].simulate_temperature(delta)
                grid_cells[i].simulate_humidity(delta)
            << i
        << simulate_quadrant_c

        task simulate_quadrant_d >>
            loop i from 7500 to 9999 >>
                grid_cells[i].simulate_pressure(delta)
                grid_cells[i].simulate_temperature(delta)
                grid_cells[i].simulate_humidity(delta)
            << i
        << simulate_quadrant_d

        await simulate_quadrant_a, simulate_quadrant_b,
              simulate_quadrant_c, simulate_quadrant_d

        apply_boundary_conditions(grid_cells)
    << fixed_update

<< AtmosphericSimulator
```

### ⚙️ Technical Version

`task` blocks compile to work items submitted to the engine's thread pool via a lock-free work queue. Each task captures a snapshot of referenced entity data at submission time — the snapshot is a shallow copy of the entity property region, performed atomically using the ARC mechanism. Entity references within the snapshot are weak — they do not increment ARC count, preventing the task from keeping entities alive beyond their intended lifetime.

The `await` expression suspends the calling async function (or suspends processing of the current event's continuation) and registers a callback fired when all awaited tasks complete. The callback is enqueued to the main thread's event queue — ensuring the continuation always runs on the main thread with correct thread affinity.

Multiple tasks passed to a single `await` are awaited concurrently — the runtime does not sequence them. The completion callback fires when ALL listed tasks have signalled completion, implemented via an atomic counter decremented by each completing task.

---

## 5.5 — Async / Await

### 👨‍💻 Developer Version

`async` and `await` let you write code that waits for something — a file to load, an asset to download, a database query to complete — without freezing your game.

**Without async — the wrong way:**
```swift
# ❌ This blocks the main thread — game freezes for 500ms
when scene_load
    level_data :: LevelData = file.load("level_2.skev_level")
    # Game completely unresponsive during this load
    scene.build(level_data)
<< scene_load
```swift

**With async — the right way:**
```swift
# ✅ Game continues running while loading happens
when scene_load
    start_level_load()    # starts loading — returns immediately
    ui.show_loading_bar()
<< scene_load

async start_level_load()
    # These lines yield — game loop continues between them
    level_data :: LevelData = await file.load("level_2.skev_level")

    # Game ran normally during the above await.
    # Now we have level_data — continue:
    textures :: list[Texture] = await assets.load_batch(level_data.textures)
    audio    :: list[AudioClip] = await assets.load_batch(level_data.audio)

    # Back on main thread — apply results
    scene.build_from(level_data, textures, audio)
    ui.hide_loading_bar()
    ui.show_level_ready()
<< start_level_load
```

**How `await` works — what actually happens:**

```
Developer writes:
    level_data = await file.load("level_2.skev_level")

What Skev does:
1. Starts the file load on the I/O thread
2. Suspends "start_level_load" function at this point
3. Returns control to the game loop immediately
4. Game loop continues: input, physics, rendering all work
5. When file finishes loading (500ms later):
   → Completion callback queued to main thread
   → Next frame: start_level_load resumes from line 2
   → level_data now contains the loaded file
6. Execution continues to the next line
```

**Async functions — the rules:**

```swift
# async marks a function that can use await
async load_game_data()
    # Inside here — await is allowed
    data :: SaveData = await file.load("save.skev_save")
    apply_save_data(data)
<< load_game_data

# Regular functions cannot use await
load_game_data_sync()
    data :: SaveData = await file.load("save.skev_save")  # Error ❌
<< load_game_data_sync
```

```
Error: await used outside async function
  "await" can only be used inside an "async" function.
  Mark this function as async:
  async load_game_data_sync()
  Line 2 — game_system.skev
```swift

**Multiple async operations:**

```swift
async initialise_game()
    # Sequential — each waits for the previous
    config    :: Config   = await file.load("config.json")
    save_data :: SaveData = await file.load("save.skev_save")
    scene.apply_config(config)
    scene.apply_save(save_data)
<< initialise_game

async initialise_game_fast()
    # Parallel — both load simultaneously
    task load_config >>
        task.result = await file.load("config.json")
    << load_config

    task load_save >>
        task.result = await file.load("save.skev_save")
    << load_save

    await load_config, load_save    # both load in parallel

    scene.apply_config(load_config.result)
    scene.apply_save(load_save.result)
<< initialise_game_fast
```

### Rule F — The Problem in Other Languages

**C++23:**
```cpp
// C++ futures — complex, verbose, error-prone
#include <future>
#include <fstream>

std::future<LevelData> futureLevel = std::async(
    std::launch::async,
    [](const std::string& path) {
        // Load file...
        return LevelData{};
    },
    "level_2.dat"
);

// Check if ready — have to poll manually
if (futureLevel.wait_for(std::chrono::seconds(0))
    == std::future_status::ready) {
    LevelData data = futureLevel.get();
    // Apply...
}
// Chaining multiple futures: complex template hell
// No language-level integration — library solution only
```

**C# 13:**
```csharp
// C# async/await — genuinely good, Skev draws inspiration
async Task LoadNextLevel() {
    var levelData = await File.ReadAllBytesAsync("level_2.dat");
    var textures = await LoadTexturesAsync(levelData);
    ApplyLevel(levelData, textures);
}
// C# async/await is well-designed — honest acknowledgement
// Skev difference: no Task<T> wrapper type needed
// Skev difference: async integrates with entity system directly
// Skev difference: no async void antipattern possible
```

**Python 3.13:**
```python
# Python asyncio — requires event loop understanding
import asyncio

async def load_next_level():
    async with aiofiles.open("level_2.dat", "rb") as f:
        level_data = await f.read()
    textures = await load_textures(level_data)
    apply_level(level_data, textures)

# Requires understanding event loop, aiofiles library,
# asyncio.run(), and the difference between coroutines
# and regular functions — steep learning curve
asyncio.run(load_next_level())
```swift

**Skev:**
```swift
async load_next_level()
    level_data :: LevelData  = await file.load("level_2.skev_level")
    textures   :: list[Texture] = await assets.load_batch(level_data.textures)
    scene.apply(level_data, textures)
<< load_next_level
```

**Honest advantage statement:**
Skev's async/await is simpler than Python's (no event loop concept) and cleaner than C++'s (no future/promise API). C# 13's async/await is the closest comparison — genuinely well-designed. Skev's advantage over C# here is integration with the entity system, no `Task<T>` wrapper type, and natural fit with Skev's `<<` block syntax. This is a case where Skev refines rather than revolutionises.

### Rule F — Real-World Example: Interactive Application

**Data visualisation dashboard — loading live data:**

```swift
entity DataDashboard >>
    chart_data  :: list[DataPoint]
    last_update :: float = 0.0
    updating    :: bool  = false

    when update(delta)
        last_update += delta

        # Refresh data every 5 seconds
        if last_update > 5.0 and not updating >>
            updating = true
            refresh_data()
        << last_update > 5.0 and not updating
    << update

    async refresh_data()
        # Fetch from API — does not block UI
        raw_data :: string = await http.get("https://api.example.com/data")

        # Parse on a task — does not block UI
        task parse_data >>
            task.result = json.parse(raw_data)
        << parse_data
        await parse_data

        # Apply to chart on main thread
        chart_data  = parse_data.result
        last_update = 0.0
        updating    = false
        ui.refresh_chart(chart_data)
    << refresh_data

<< DataDashboard
```

### ⚙️ Technical Version

`async` functions compile to state machines using LLVM's coroutine intrinsics (`llvm.coro.begin`, `llvm.coro.suspend`, `llvm.coro.end`). Each `await` point becomes a suspension point in the state machine. The coroutine frame — containing local variables that must survive across suspension points — is heap-allocated at async function entry and freed at completion.

`await expr` suspends the coroutine and registers a continuation with the awaited operation. For file and asset I/O, the operation is dispatched to an OS-level async I/O queue (io_uring on Linux, IOCP on Windows, kqueue on macOS). Completion triggers the continuation to be enqueued to the main thread's event queue, ensuring the async function resumes on the correct thread.

The restriction against `await` in `when` blocks directly is enforced at parse time — the parser tracks whether the current scope is inside an `async` function. This prevents accidental suspension of the main event loop at unexpected points.

---

## 5.6 — channel[T] — Cross-Thread Communication

### 👨‍💻 Developer Version

`channel[T]` is how different threads communicate safely. One thread sends data. Another thread receives it. No shared state. No race conditions. No locks.

Think of a channel like a mailbox:
```
Sender:   puts a letter in the mailbox (any thread, any time)
Receiver: checks the mailbox and reads letters (main thread)
```swift

**Declaring and using channels:**

```swift
entity NetworkManager >>
    # Channel carries DamageEvent messages from network to main thread
    incoming_damage     :: channel[DamageEvent]
    incoming_positions  :: channel[PositionUpdate]
    outgoing_events     :: channel[GameEvent]

    when update(delta)
        # Process all incoming damage events
        loop while incoming_damage.has_messages >>
            event :: DamageEvent = incoming_damage.receive()
            apply_network_damage(event)
        << while incoming_damage.has_messages

        # Process all position updates
        loop while incoming_positions.has_messages >>
            update :: PositionUpdate = incoming_positions.receive()
            sync_remote_player_position(update)
        << while incoming_positions.has_messages
    << update

    # Called by networking thread when packet arrives
    when network_packet_received(packet)
        match packet.type >>
            PacketType.damage >>
                event :: DamageEvent = packet.parse_damage()
                incoming_damage.send(event)     # safe from any thread
            << PacketType.damage

            PacketType.position >>
                pos :: PositionUpdate = packet.parse_position()
                incoming_positions.send(pos)    # safe from any thread
            << PacketType.position
        << packet.type
    << network_packet_received

<< NetworkManager
```

**Channel configuration:**

```swift
entity AudioSystem >>
    # Default: capacity 256, overflow: drop_oldest
    audio_commands :: channel[AudioCommand]

    # Custom capacity for high-frequency events
    frame_events :: channel[FrameEvent, capacity: 1024]

    # Block sender when full (careful — can stall sender thread)
    critical_events :: channel[CriticalEvent, capacity: 64, overflow: block]

    # Drop newest instead of oldest when full
    sensor_data :: channel[SensorReading, capacity: 512, overflow: drop_newest]

<< AudioSystem
```swift

**Draining all messages at once:**

```swift
when update(delta)
    # Process all messages as a batch
    damage_batch :: list[DamageEvent] = incoming_damage.drain()

    loop event in damage_batch >>
        apply_damage(event.target, event.amount)
    << event
<< update
```

**Channel overflow behaviour:**

```
When channel is full:
→ drop_oldest (default): oldest message removed, new message added
  Best for: game events, position updates, sensor readings
  Old data is less relevant than new data

→ drop_newest: new message discarded, channel unchanged
  Best for: critical state that must not be overwritten
  Example: player save data

→ block: sender waits until space is available
  Use with extreme caution — can stall the sending thread
  Only for: truly critical data where loss is unacceptable
```

### Rule F — Real-World Example: Game Development

**Game server broadcasting to all connected clients:**

```swift
entity GameServer >>
    connected_clients :: list[ClientConnection]
    broadcast_channel :: channel[ServerEvent, capacity: 2048]

    when update(delta)
        # Drain all pending broadcasts
        loop while broadcast_channel.has_messages >>
            event :: ServerEvent = broadcast_channel.receive()

            # Send to all connected clients
            loop client in connected_clients >>
                if client.is_connected >>
                    client.send(event)
                << client.is_connected
            << client
        << while broadcast_channel.has_messages
    << update

    # Called from networking thread when game event occurs
    async broadcast(event: ServerEvent)
        broadcast_channel.send(event)    # thread-safe — any thread
    << broadcast

<< GameServer
```swift

### Rule F — Real-World Example: Robotics

**Sensor fusion — combining data from multiple sensor threads:**

```swift
entity SensorFusion >>
    lidar_channel    :: channel[LidarReading, capacity: 100]
    camera_channel   :: channel[CameraFrame, capacity: 10]
    imu_channel      :: channel[IMUReading, capacity: 500]

    fused_state      :: RobotState

    when fixed_update(delta)
        # Collect latest readings from all sensors
        latest_lidar  :: maybe LidarReading  = nothing
        latest_camera :: maybe CameraFrame   = nothing
        latest_imu    :: maybe IMUReading    = nothing

        if lidar_channel.has_messages >>
            latest_lidar  = lidar_channel.drain().last
        << lidar_channel.has_messages

        if camera_channel.has_messages >>
            latest_camera = camera_channel.drain().last
        << camera_channel.has_messages

        if imu_channel.has_messages >>
            latest_imu    = imu_channel.drain().last
        << imu_channel.has_messages

        # Fuse available sensor data
        fused_state = sensor_fusion.update(
            latest_lidar,
            latest_camera,
            latest_imu,
            delta
        )

        # Apply fused state to robot control
        robot_controller.update(fused_state, delta)
    << fixed_update

<< SensorFusion
```

### ⚙️ Technical Version

`channel[T]` is implemented as a lock-free single-producer single-consumer (SPSC) ring buffer when the channel is accessed by exactly one sender and one receiver thread — detected at compile time. For multi-producer scenarios (multiple threads calling `.send()`), a lock-free multi-producer single-consumer (MPSC) queue is used. Both implementations use atomic operations exclusively — no mutex, no condition variable.

The ring buffer is allocated with a capacity rounded up to the next power of two for efficient modulo-free index arithmetic using bitwise AND. Elements are copied on send — `send()` performs a `memcpy` of the value into the ring buffer slot. For ARC-managed types (entity references), ARC increment fires at copy time and ARC decrement fires when the slot is overwritten by the overflow policy or consumed by `.receive()`.

`.has_messages` is a single atomic load of the producer index — O(1) and branch-predictor friendly. `.drain()` performs a bulk copy of all available slots to a stack-allocated `list[T]` in a single pass — more efficient than repeated `.receive()` calls for high-throughput scenarios.

---

## 5.7 — Double-Buffered Transform System

### 👨‍💻 Developer Version

The rendering thread and the physics thread both need access to entity positions. But the renderer might be reading a position at the exact moment physics is writing a new one. This would cause visual glitches — entities appearing to teleport for one frame.

Skev prevents this automatically. You never configure it, see it, or think about it.

**What happens under the hood:**

```
Frame structure:

Physics thread:    [simulate step]──[write Buffer A]──[simulate step]──...
Main thread:       [update events]──────────────────[update events]───...
Render thread:            [read Buffer B]───────────────[read Buffer B]──...
                                    ↑
                             [atomic swap A↔B]
                           (happens once per frame)
```

**Why this matters:**
```
Without double-buffering:
→ Physics writes position = (10.2, 0, 5.1)
→ Renderer reads halfway through the write
→ Renderer gets position = (10.0, 0, 5.1)  ← half-updated
→ Visual glitch — entity jumps

With double-buffering (Skev):
→ Physics writes to Buffer A
→ Renderer reads from Buffer B (previous frame's data)
→ One atomic swap per frame at defined sync point
→ No lock during rendering or physics
→ Perfect. Zero glitches.
```

**Impact on XR/VR:**
```
VR requires 90+ fps.
Any visual glitch = motion sickness.
Double-buffered transforms = visually perfect at any frame rate.
Critical for the XR use case.
```

**Developer experience:** Zero. You write `position += velocity * delta` and it works perfectly across all threads. The engine handles everything.

### ⚙️ Technical Version

Each entity maintains two transform buffers — current (index 0) and previous (index 1). An entity-level atomic byte tracks the current write index. The physics thread always writes to the current buffer. The rendering thread always reads from `1 - current_index` — the previous buffer.

Once per frame, at the defined synchronisation point between the update phase and the render phase, the engine performs an atomic swap of the current index byte across all entities simultaneously using a barrier operation. This swap is the only synchronisation point — O(1) regardless of entity count.

The Skev engine runtime guarantees this swap fires between the completion of all Tier 1 `when update()` handlers and the beginning of the next render pass. Developers cannot influence this timing — it is an engine invariant.

---

## 5.8 — Task Cancellation

### 👨‍💻 Developer Version

Sometimes you need to stop a task before it finishes. A player skips the loading screen. A procedural generation is restarted. A search query is superseded.

Skev uses cooperative cancellation — you signal that you want cancellation, and the task stops cleanly when it is safe to do so:

```swift
entity LevelLoader >>
    load_task_active :: bool = false

    when update(delta)
        if input.key_pressed(KEY_ESCAPE) and load_task_active >>
            cancel load_next_level    # signal cancellation
            ui.hide_loading_screen()
            load_task_active = false
        << input.key_pressed(KEY_ESCAPE) and load_task_active
    << update

    when collision(other: LevelPortal)
        load_task_active = true
        ui.show_loading_screen()
        load_next_level_task()
    << collision: LevelPortal

    async load_next_level_task()
        task load_next_level >>
            assets :: list[Asset] = level.get_asset_list()

            loop asset in assets >>
                # Check for cancellation before each asset
                if task.cancelled >>
                    stop    # exit loop cleanly
                << task.cancelled

                await assets.load_single(asset)
            << asset

            if not task.cancelled >>
                task.result = level.build_from(assets)
            << not task.cancelled
        << load_next_level

        await load_next_level

        if not load_next_level.cancelled >>
            scene.transition_to(load_next_level.result)
            load_task_active = false
        << not load_next_level.cancelled
    << load_next_level_task

<< LevelLoader
```

**Cancellation rules:**
```
cancel task_name:
→ Sets task.cancelled flag atomically
→ Does NOT forcefully stop the task
→ Task must check task.cancelled and exit cleanly
→ ARC handles all memory cleanup automatically on exit
→ After cancellation: task.cancelled is true
  task.result contains whatever was set before cancellation

Why cooperative (not forced)?
→ Forced termination leaves resources in unknown state
→ Cooperative cancellation allows clean resource release
→ Same approach as Swift, Kotlin, and .NET — proven safe
```swift

---

## 5.9 — Real-Time Tasks — `realtime` Modifier

### 👨‍💻 Developer Version

For robotics, industrial control, and precision simulation — some code must run at a fixed interval with guaranteed timing. Not approximately 1ms. Exactly 1ms, every time, with no jitter.

The `realtime` modifier tells Skev and the OS scheduler that this task has hard real-time requirements:

```swift
entity RobotController >>
    arm_position    :: Vector3!
    target_position :: Vector3!
    motor_commands  :: array[float, 6]    # pre-allocated

    when scene_load
        # Start real-time control loop
        # Runs at exactly 1000Hz (1ms interval)
        # HIGH priority — OS scheduler gives it dedicated time
        task control_loop realtime(priority: high, interval: 1ms) >>
            # Read sensors — must be pre-allocated data
            joint_angles :: array[float, 6] = sensor_bus.read_joints()
            end_effector :: Vector3! = kinematics.forward(joint_angles)

            # Compute control — pure stack computation
            error    :: Vector3! = target_position - end_effector
            commands :: array[float, 6] = pid_controller.compute(
                error,
                joint_angles
            )

            # Write to actuators
            motor_bus.write_commands(commands)
        << control_loop
    << scene_load

<< RobotController
```

**What `realtime` tasks CAN use:**
```
✅ Stack-allocated primitives (int, float, Vector3! etc)
✅ Pre-allocated arrays and buffers
✅ Pure mathematical functions
✅ Direct hardware I/O (sensor_bus, motor_bus)
✅ Fixed-size data types
```

**What `realtime` tasks CANNOT use (compiler enforced):**
```
❌ Heap allocation — non-deterministic timing
❌ await — yields are incompatible with real-time
❌ channel[T] — queuing has latency
❌ list[T] or map[K, V] — dynamic allocation
❌ string operations — allocation possible
❌ scene access — main thread only
```

```swift
Error: Heap allocation in realtime task
  "active_enemies" is a list[Enemy] — dynamic heap allocation.
  realtime tasks cannot use heap-allocated types.

  Pre-allocate a fixed array before the task:
  enemy_buffer :: array[Enemy, 64]

  Line 8 — robot_controller.skev
```

**Why these restrictions exist:**
```
Heap allocation: allocator has non-deterministic latency
                 could take 10μs on a bad frame — unacceptable

await:           suspends execution — timing guarantee broken

channel:         message queue has variable latency

All of these could cause the 1ms control loop to
take 1.5ms, then 0.8ms, then 1.2ms — unpredictable.
A robot arm with jittery control is dangerous.
These restrictions are safety requirements.
```

### Rule F — Real-World Example: Robotics

**6-DOF robot arm precision control:**

```swift
entity IndustrialRobotArm >>
    # Pre-allocated buffers — no heap allocation in control loop
    joint_buffer  :: array[float32, 6]
    torque_buffer :: array[float32, 6]
    error_buffer  :: array[float32, 6]

    target_pose   :: Transform!
    current_pose  :: Transform!

    shared emergency_stop :: bool = false    # readable from safety thread

    when scene_load
        # Safety monitor — medium priority, 100Hz
        task safety_monitor realtime(priority: medium, interval: 10ms) >>
            if sensor_bus.read_emergency_stop() >>
                emergency_stop = true
            << sensor_bus.read_emergency_stop()
        << safety_monitor

        # Control loop — high priority, 1000Hz
        task arm_control realtime(priority: high, interval: 1ms) >>
            # Emergency stop check
            if emergency_stop >>
                motor_bus.halt_all()
                stop
            << emergency_stop

            # Read current joint positions
            joint_buffer = sensor_bus.read_joints_raw()

            # Compute inverse kinematics
            current_pose = kinematics.forward_raw(joint_buffer)

            # PID control for each joint
            loop i from 0 to 5 >>
                error_buffer[i] = target_pose.position[i]
                                  - current_pose.position[i]
                torque_buffer[i] = pid_bank[i].compute_raw(
                    error_buffer[i]
                )
            << i

            # Write to motors
            motor_bus.write_torques_raw(torque_buffer)
        << arm_control
    << scene_load

<< IndustrialRobotArm
```

**This code is impossible to write safely in C++, C#, or Python:**
```
C++:   No language-level real-time constraints
       Developer must manually use POSIX SCHED_FIFO
       Heap allocation in control loop = silent timing bug
       Nothing prevents writing "vector<float>" in RT code

C#:    GC pauses make real-time impossible
       Even with Native AOT — GC still runs
       .NET has no real-time thread model

Python: Interpreted — cannot meet 1ms timing
        GIL prevents true real-time threading

Skev:  Compiler prevents heap allocation in realtime task
       Emergency stop uses shared atomic — safe
       Pre-allocated buffers enforced by type system
       This is the ONLY language that makes this safe by default
```

### ⚙️ Technical Version

`realtime` tasks are mapped to OS-level real-time threads using platform-specific APIs: `SCHED_FIFO` on Linux, `SetThreadPriority(THREAD_PRIORITY_TIME_CRITICAL)` on Windows, and the appropriate QoS class on macOS/iOS. The `interval` parameter configures a periodic timer that fires the task at the specified frequency.

The compiler performs a purity check on `realtime` task bodies — scanning the entire call graph for operations that could perform heap allocation, block on a mutex, or call into the Skev runtime's async machinery. Any such call is a compile error. This check is conservative — if the compiler cannot statically prove an operation is allocation-free, it rejects it.

All data accessed within a `realtime` task must have a statically known size — verified by the type system. This ensures the compiler can guarantee the task's stack frame size at compile time, preventing stack allocation surprises at runtime.

---

## 5.10 — Concurrency Across All Use Cases

### 👨‍💻 Developer Version

Every concurrency decision was verified to work for all of Skev's target domains. Here is how each domain uses Skev's concurrency model:

**Game Development:**
```
Tier 1:  Entity logic, input, scene management
Tier 2:  Physics, rendering, audio — automatic
Tier 3:  AI pathfinding, level generation, asset streaming
channel: Network events → game events
async:   Level loading, save game, asset downloads
```

**XR / VR / AR:**
```
Tier 1:  Interaction logic, UI, content management
Tier 2:  Render (90fps critical), head tracking
Tier 3:  Scene generation, asset streaming
channel: Tracking data → interaction system
async:   Content loading, cloud assets
realtime: Haptic feedback timing (if needed)
```

**Simulation:**
```
Tier 1:  Agent decisions, simulation control
Tier 2:  Physics, visualisation
Tier 3:  Batch agent computation, data analysis
channel: External data feeds → simulation state
async:   Result export, data import
realtime: Fixed-timestep integration (physics)
```

**Robotics:**
```
Tier 1:  High-level behaviour, mission planning
Tier 2:  Sensor preprocessing
Tier 3:  Path planning, ML inference
channel: Sensor threads → fusion → control
realtime: CRITICAL — motor control, sensor sampling
```

**Animation / VFX Tools:**
```
Tier 1:  Tool UI, viewport interaction
Tier 2:  Preview rendering
Tier 3:  Batch render computation, simulation
channel: Render thread → progress UI
async:   File I/O, asset export
```

**Interactive Applications:**
```
Tier 1:  UI logic, user interaction
Tier 2:  Rendering
Tier 3:  Data processing, analysis
channel: Background updates → UI refresh
async:   API calls, database queries, file I/O
```

**Real-time Networking:**
```
Tier 1:  Game/application logic
Tier 2:  Network I/O thread
Tier 3:  Data processing, analytics
channel: Network thread → main thread (primary pattern)
async:   Connection management, request/response
```swift

---

## 5.11 — Complete Concurrency Example

All five concurrency mechanisms in one realistic system:

```swift
#! multiplayer_game_manager.skev
#! Manages all concurrent aspects of a multiplayer game session.

kind NetworkEventType >>
    player_joined
    player_left
    damage_dealt
    position_sync
    game_state_change
<< NetworkEventType

data NetworkEvent >>
    type      :: NetworkEventType
    player_id :: uint32
    payload   :: string
    timestamp :: float64
<< NetworkEvent

entity MultiplayerManager >>

    #{ Channels for cross-thread communication }#
    incoming_events :: channel[NetworkEvent, capacity: 512]
    outgoing_events :: channel[NetworkEvent, capacity: 512]

    #{ Shared state — accessible from network thread }#
    shared session_id      :: uint32 = 0
    shared players_online  :: int    = 0
    shared game_running    :: bool   = false

    #{ Main thread state }#
    active_players  :: list[Player]
    game_state      :: GameState = GameState.waiting
    tick_counter    :: int = 0

    has NetworkSync

    when scene_load
        # Start network I/O task
        start_network()
        game_running = true
    << scene_load

    when update(delta)
        tick_counter += 1

        # Process all incoming network events on main thread
        loop while incoming_events.has_messages >>
            event :: NetworkEvent = incoming_events.receive()
            handle_network_event(event)
        << while incoming_events.has_messages

        # Every 10 ticks — batch update AI for all players' bots
        if tick_counter >= 10 >>
            tick_counter = 0
            batch_update_bots()
        << tick_counter >= 10
    << update

    # Async: start networking without blocking game loop
    async start_network()
        connection :: NetworkConnection = await network.connect(
            server_address,
            port: 7777
        )

        session_id = await network.join_session(connection)
        players_online = await network.get_player_count(session_id)
        game_running = true

        ui.show_message("Connected — " + players_online.as(string)
                        + " players online")
    << start_network

    # Handle events received from network thread via channel
    handle_network_event(event: NetworkEvent)
        match event.type >>
            NetworkEventType.player_joined >>
                new_player :: Player = Player.create(event.player_id)
                active_players.add(new_player)
                ui.show_join_notification(new_player.name)
            << NetworkEventType.player_joined

            NetworkEventType.player_left >>
                leaving :: maybe Player = active_players.find(
                    event.player_id
                )
                if leaving exists >>
                    active_players.remove(leaving)
                    ui.show_leave_notification(leaving.name)
                << leaving exists
            << NetworkEventType.player_left

            NetworkEventType.damage_dealt >>
                target :: maybe Player = active_players.find(
                    event.player_id
                )
                if target exists >>
                    damage :: int = event.payload.as(maybe int) or 0
                    target.take_damage(damage)
                << target exists
            << NetworkEventType.damage_dealt

            NetworkEventType.position_sync >>
                remote :: maybe Player = active_players.find(
                    event.player_id
                )
                if remote exists >>
                    pos :: Vector3! = Vector3!.parse(event.payload)
                    remote.sync_position(pos)
                << remote exists
            << NetworkEventType.position_sync

            _ -> debug.log("Unhandled event: " + event.type.as(string))
        << event.type
    << handle_network_event

    # Batch AI update using parallel tasks
    batch_update_bots()
        bot_players :: list[Player] = active_players.filter(is_bot)

        if bot_players.count > 0 >>
            task compute_bot_decisions >>
                loop bot in bot_players >>
                    bot.ai.compute_decision(game_state)
                << bot
            << compute_bot_decisions

            await compute_bot_decisions

            loop bot in bot_players >>
                bot.apply_ai_decision()
            << bot
        << bot_players.count > 0
    << batch_update_bots

    when destroyed
        game_running = false
        # Cancel any in-progress network operations
        cancel start_network
        network.disconnect()
    << destroyed

<< MultiplayerManager
```

---

## 5.12 — Concurrency Quick Reference

```swift
THREE TIERS:
  Tier 1 — Game loop:     Your when blocks run here — always safe
  Tier 2 — Engine:        Physics, render, audio — automatic
  Tier 3 — Tasks:         Parallel work — opt-in

TASK:
  task name >>            open parallel task block
      ...                 runs on worker thread
  << name                 close task block
  await name              wait for result (main thread)
  await a, b, c           wait for multiple simultaneously
  task.result = value     return value from task
  task.cancelled          check if cancelled
  cancel name             signal cancellation (cooperative)

ASYNC:
  async func_name()       mark function as awaitable
      ...
  << func_name
  await operation         yield until operation complete
                          game loop continues during yield

CHANNEL:
  channel[T]                    default: capacity 256, drop_oldest
  channel[T, capacity: N]       custom capacity
  channel[T, overflow: block]   block sender when full
  .send(value)                  send copy — any thread
  .receive()                    receive next — main thread only
  .has_messages                 bool — thread safe
  .drain()                      all pending → list[T]

SHARED:
  shared prop :: Type           atomic cross-thread access
                                writes: main thread only
                                reads: any thread

REALTIME:
  task name realtime(priority: high, interval: 1ms) >>
  ❌ No heap, no await, no channel, no dynamic collections
  ✅ Stack only, pre-allocated buffers, pure math, hardware I/O

DATA RACE PREVENTION:
  Entity properties → main thread only (compiler enforced)
  shared properties → atomic access (automatic)
  task data         → snapshot — read only (compiler enforced)
  channel data      → copy on send (always safe)
```

---

*End of Chapter 5 v0.1 — Next: Chapter 6: Error Handling*
