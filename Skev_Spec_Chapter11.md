<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 11: Tooling
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026)
**Status:** In Progress
**Depends On:** Chapter 9 (build/CLI), Chapter 10 (sandbox/plugins), Chapter 6 (errors)
**Use Cases Tested:** All 7 — Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅
**Rule H Verified:** skev.log, skev.vscode-skev, test, mock, bench — all checked ✅
**Gaps Resolved:** Testing Framework (High), REPL/Playground (Medium), Observability (Medium)

---

## 11.1 — Overview

### 👨‍💻 Developer Version

A programming language is only as good as the tools that support it. The most expressive syntax in the world means nothing if your editor shows no completions, your tests take 5 minutes to write, and your crashes produce useless stack traces.

Chapter 11 specifies everything developers interact with outside the language itself:

```
Testing:       Write and run tests in the same syntax as your game code
Benchmarks:    Measure performance with built-in tooling
Observability: Structured logging, tracing, and crash reporting
LSP:           Editor completions, diagnostics, go-to-definition
TextMate:      Syntax highlighting for every editor
REPL:          Interactive exploration of Skev
Skev Studio:   The dedicated IDE
Documentation: Generate API docs from code comments
```

> **The goal: every Skev developer has world-class tools from day one.**

Not "world-class tools available if you configure five plugins." World-class tools that work immediately when you install Skev.

### ⚙️ Technical Version

Chapter 11 specifies: the `test`/`bench`/`mock` testing DSL built into the Skev compiler, the `skev.log` structured logging library, the Language Server Protocol implementation (skev-lsp), the TextMate grammar (source.skev), the `skev repl` interactive interpreter, the `skev doc` documentation generator, and the Skev Studio IDE architecture. All tools are open source (Apache 2.0, consistent with Rule J). Skev Studio Pro tier is commercial. All tools share the locked color scheme from the Skev Color Scheme specification.

---

## 11.2 — Testing Framework

### 👨‍💻 Developer Version

Testing in Skev uses the same syntax as the rest of the language. No new format to learn. No separate test runner to install. `test` blocks are part of Skev. `skev test` runs them.

> **You already know how to write Skev tests. The syntax is Skev.**

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game developer needs to test: a damage calculation function, an entity that applies damage and checks for death, an async asset loader that must complete within 200ms, and a pathfinder that must find routes in under 1ms at p99. These represent the four fundamental testing needs in any game codebase.

**💥 Why This Is Hard**

Testing game code has three specific challenges other domains do not have: (1) entities have ARC-managed state that must be cleanly isolated between tests, (2) real-time constraints mean performance is a first-class test requirement, (3) systems interact through events and components — mocking these is verbose in languages without first-class event systems.

---

**In C++23:**
```cpp
// C++ — must choose a framework: Google Test, Catch2, doctest
// Using Google Test:
#include <gtest/gtest.h>

TEST(PlayerTest, TakeDamageFloorsAtZero) {
    Player player(100);
    player.TakeDamage(150);
    EXPECT_EQ(player.GetHealth(), 0);
    EXPECT_FALSE(player.IsAlive());
}

TEST(PathfinderTest, BlockedPathReturnsEmpty) {
    NavMesh nav = NavMesh::CreateBlocked();
    auto path = nav.FindPath({0,0,0}, {10,0,10});
    EXPECT_TRUE(path.empty());
}
// 12 lines total — framework-specific syntax, separate install required
// Benchmark: Google Benchmark is a SEPARATE third-party library
// Mock: Google Mock is ANOTHER separate library
```

**In C# 13:**
```csharp
// C# — must choose: xUnit, NUnit, or MSTest
// Using xUnit:
using Xunit;

public class PlayerTests {
    [Fact]
    public void TakeDamage_FloorsAtZero() {
        var player = new Player(health: 100);
        player.TakeDamage(150);
        Assert.Equal(0, player.Health);
        Assert.False(player.IsAlive());
    }

    [Fact]
    public void BlockedPath_ReturnsEmpty() {
        var nav = NavMesh.CreateBlocked();
        var path = nav.FindPath(Vector3.zero, new Vector3(10,0,10));
        Assert.Empty(path);
    }
}
// 16 lines total — [Fact] attribute, Assert class
// Benchmark: BenchmarkDotNet — third-party
// Mock: Moq — third-party
```

**In Python 3.13:**
```python
# Python — pytest is de facto standard
import pytest
from game import Player, NavMesh

def test_take_damage_floors_at_zero():
    player = Player(health=100)
    player.take_damage(150)
    assert player.health == 0
    assert not player.is_alive()

def test_blocked_path_returns_none():
    nav = NavMesh.create_blocked()
    path = nav.find_path((0,0,0), (10,0,10))
    assert path is None
# 9 lines total — clean, but benchmark/mock are third-party
```

**In Skev:**
```swift
test "player take_damage floors at zero" >>
    player :: Player = Player >> health :: 100 << Player
    player.take_damage(150)
    assert player.health == 0        "Health at floor"
    assert player.is_alive() == false "Player is dead"
<< test

test "blocked path returns nothing" >>
    nav  :: NavMesh = NavMesh.create_blocked()
    path :: maybe list[Vector3!] = nav.find_path(
        Vector3!(0.0, 0.0, 0.0),
        Vector3!(10.0, 0.0, 10.0)
    )
    assert path == nothing "Blocked path is nothing"
<< test
// 11 lines total — same Skev syntax, no framework needed
```

### ✅ Honest Advantage Statement

Skev's test syntax is marginally more verbose than Python's (11 vs 9 lines) but significantly cleaner than C# (11 vs 16). The core advantage is not line count — it is consistency: test blocks use `>>` `<<`, `assert` from Chapter 6, entity construction from Chapter 3. A developer who knows Skev already knows how to write Skev tests. No separate framework, no separate documentation, no `npm install` or `pip install` needed. Python's pytest is excellent — Skev's honest position is that built-in testing eliminates a decision (which framework?) and a dependency.

---

### 🟢 Foundation Examples

```swift
# Test a pure function
test "math.clamp constrains values" >>
    assert math.clamp(5, 0, 10)  == 5   "Value within range"
    assert math.clamp(-5, 0, 10) == 0   "Value below min"
    assert math.clamp(15, 0, 10) == 10  "Value above max"
<< test

# Test a data type
test "Pair stores both values" >>
    p :: Pair[string, int] = Pair >> first :: "Gold"  second :: 500 << Pair
    assert p.first  == "Gold"  "First value stored"
    assert p.second == 500     "Second value stored"
<< test

# Test error handling
test "load_level fails on missing file" >>
    outcome :: result[LevelData] = file.load_level("nonexistent.skev")
    match outcome >>
        succeed _ -> assert false "Should not succeed"
        fail FileError.not_found -> # correct — test passes
        fail _ -> assert false "Wrong error type"
    << outcome
<< test
// 18 lines total
```

---

### 🟡 Applied Example — Entity Testing With Mocks

```swift
# Mock audio system — records calls without playing sounds
test "combat plays collision sound on hit" >>
    recorded_sounds :: list[string]

    audio :: AudioSystem = mock AudioSystem >>
        when play_at(sound: string, position: Vector3!)
            recorded_sounds.add(sound)
        << play_at
        when play(sound: string)
            recorded_sounds.add(sound)
        << play
    << mock AudioSystem

    combat :: CombatSystem = CombatSystem >>
        has audio
    << CombatSystem

    combat.process_collision(
        enemy:  Enemy >> health :: 100 << Enemy,
        player: Player >> health :: 100 << Player
    )

    assert recorded_sounds.contains("collision_impact") "Collision sound played"
<< test

# Test with shared setup — expensive NavMesh created once
test_setup >>
    arena :: NavMesh = -> await NavMesh.load("test_assets/arena.skev")
<< test_setup

test "pathfinder finds direct path" >>
    path :: maybe list[Vector3!] = arena.find_path(
        Vector3!(0.0, 0.0, 0.0),
        Vector3!(5.0, 0.0, 5.0)
    )
    assert path exists "Path found"
    assert (path or []).count > 0 "Path has waypoints"
<< test

test "pathfinder returns nothing for blocked route" >>
    path :: maybe list[Vector3!] = arena.find_path(
        Vector3!(0.0, 0.0, 0.0),
        Vector3!(-100.0, 0.0, -100.0)    # outside arena
    )
    assert path == nothing "Blocked route returns nothing"
<< test
// 40 lines total — shared setup + mock + multiple tests
```

---

### 🔴 Hard Complex Example — Multiplayer Sync Testing

**🌍 Real-World Scenario**

A real-time multiplayer game must synchronise entity positions across all clients with < 100ms latency and < 2% desynchronisation rate, even with 10% packet loss and 80ms network latency. Testing this in production requires real servers — but real servers cannot be automated. The test must simulate realistic network conditions locally and verify the synchronisation algorithm mathematically.

**💥 Why This Is Hard**

Network conditions are non-deterministic. Real network tests are flaky by nature. The solution is deterministic mock networking: a simulated network layer with controlled packet loss, latency, and jitter that produces the same results every run — making tests reliable without requiring a real network.

**In C++23:**
```cpp
// C++ — requires: Google Test + custom network mock + thread coordination
class MockNetwork : public INetworkLayer {
    std::default_random_engine rng{42};
    std::uniform_real_distribution<float> dist{0.0f, 1.0f};
public:
    float packet_loss_rate;
    int latency_ms;

    bool Send(const Packet& p) override {
        if (dist(rng) < packet_loss_rate) return false;
        std::this_thread::sleep_for(
            std::chrono::milliseconds(latency_ms));
        return true;
    }
};

TEST(MultiplayerTest, PositionSyncsWithPacketLoss) {
    MockNetwork net;
    net.packet_loss_rate = 0.1f;
    net.latency_ms = 80;
    GameServer server;
    GameClient client;
    server.Connect(&client, &net);

    auto player = server.SpawnPlayer({0,0,0});
    server.MovePlayer(player->id, {5,0,0});
    for (int i = 0; i < 60; ++i) {
        server.Tick(16);
        client.Tick(16);
    }
    auto pos = client.GetPlayerPosition(player->id);
    EXPECT_GT(pos.x, 4.0f);
}
// 30 lines total — verbose mock class, manual threading
```

**In C# 13:**
```csharp
[Fact]
public async Task PositionSyncs_WithPacketLoss() {
    var net = new MockNetwork(packetLoss: 0.1f, latencyMs: 80, seed: 42);
    using var server = new GameServer();
    using var client = new GameClient();
    server.Connect(client, net);

    var playerId = server.SpawnPlayer(Vector3.zero);
    server.MovePlayer(playerId, new Vector3(5, 0, 0));

    for (int i = 0; i < 60; i++) {
        await server.TickAsync(16);
        await client.TickAsync(16);
    }
    var pos = client.GetPosition(playerId);
    Assert.True(pos.x > 4.0f);
}
// 15 lines total — cleaner, but MockNetwork must be written separately
```

**In Python 3.13:**
```python
def test_position_syncs_with_packet_loss():
    net = MockNetwork(packet_loss=0.1, latency_ms=80, seed=42)
    server = GameServer()
    client = GameClient()
    server.connect(client, via=net)

    player_id = server.spawn_player((0, 0, 0))
    server.move_player(player_id, (5, 0, 0))

    for _ in range(60):
        server.tick(16)
        client.tick(16)

    pos = client.get_position(player_id)
    assert pos[0] > 4.0
# 11 lines total — but MockNetwork still written separately
// 11 lines total — no built-in mock networking
```

**In Skev:**
```swift
test async "player position syncs with 10% packet loss" >>

    # Built-in mock network — deterministic, seeded
    mock_net :: NetworkLayer = mock NetworkLayer.simulated(
        packet_loss: 0.1,
        latency_ms:  80,
        jitter_ms:   10,
        seed:        42      # deterministic — same result every run
    )

    server :: GameServer = GameServer.create_test()
    client :: GameClient = GameClient.create_test()
    server.connect(client, via: mock_net)

    # Spawn and move player on server
    player_id :: int = server.spawn_player(Vector3!(0.0, 0.0, 0.0))
    server.move_player(player_id, Vector3!(5.0, 0.0, 0.0))

    # Simulate 60 frames (1 second at 60fps)
    loop i from 0 to 59 >>
        -> await server.tick(16)
        -> await client.tick(16)
    << i

    # Verify position synchronised within tolerance
    client_pos :: Vector3! = client.get_player_position(player_id)
    assert client_pos.x > 4.0 "Position synced within 1m tolerance"
    assert client_pos.x < 6.0 "Position not overshot"

    # Verify sync statistics
    stats :: SyncStats = client.get_sync_stats()
    assert stats.desync_rate < 0.02 "Desync rate under 2%"
    assert stats.avg_latency_ms < 100.0 "Average latency under 100ms"

<< test
// 30 lines total — built-in mock network, full sync verification
```

### 🧭 Walk-Through — Multiplayer Sync Test

**`mock NetworkLayer.simulated(...)`:** Built-in simulated network mock. `seed: 42` makes packet loss deterministic — test produces identical results every run. No flakiness. The mock records all send/receive operations and provides `get_sync_stats()` for post-run analysis.

**`test async`:** The test is async because `server.tick()` and `client.tick()` are async (they involve timer callbacks from Chapter 7). The `-> await` propagates any tick failures as test failures automatically.

**60-frame loop:** Simulates 1 second at 60fps. Server moves the player, packets arrive with simulated 80ms delay and 10% loss. After 60 frames, the client should have received enough packets to converge on the server position.

**`stats.desync_rate < 0.02`:** The mathematical assertion. Not just "did position arrive" but "is the synchronisation algorithm performing within spec." This is the assertion that catches algorithm regressions — a future change to the sync algorithm that degrades to 5% desync fails this test immediately.

### ✅ Honest Advantage Statement

Skev's test is 30 lines — longer than Python (11 lines). The extra lines are the desync rate and latency assertions — which Python's version omits. At equivalent assertion depth, all languages are comparable. The genuine Skev advantage: `mock NetworkLayer.simulated(...)` is built-in — no third-party library needed. C++ requires a custom `MockNetwork` class. C# and Python both require custom mock implementations. The `seed:` parameter for deterministic randomness is a first-class feature, not an afterthought.

---

### 11.2.1 — Benchmark Tests

Performance is a first-class concern in Skev. `bench` blocks measure execution time with statistical rigour.

### Rule F — Non-Game Use Case: Robotics Arm Calibration

**🌍 Real-World Scenario**

A robotic arm's inverse kinematics solver must compute joint angles in under 5ms (p99) to maintain real-time control at 200Hz. The benchmark must verify this across 1000 iterations with a realistic workspace distribution of target positions — not just a single easy case.

**In C++23:**
```cpp
// C++ benchmark — Google Benchmark (third-party)
#include <benchmark/benchmark.h>

static void BM_IKSolver(benchmark::State& state) {
    RobotArm arm = RobotArm::CreateTestArm();
    std::mt19937 rng(42);
    std::uniform_real_distribution<float> dist(-0.5f, 0.5f);

    for (auto _ : state) {
        Eigen::Vector3f target(
            dist(rng), dist(rng), 0.5f + dist(rng));
        auto result = arm.SolveIK(target);
        benchmark::DoNotOptimize(result);
    }
}
BENCHMARK(BM_IKSolver)->Iterations(1000);
BENCHMARK_MAIN();
// 15 lines total — Google Benchmark required — separate install + CMake setup
```

**In C# 13:**
```csharp
// C# benchmark — BenchmarkDotNet (third-party)
[MemoryDiagnoser]
public class IKBenchmarks {
    private RobotArm _arm = RobotArm.CreateTestArm();
    private Random _rng = new(42);

    [Benchmark]
    public JointAngles SolveIK() {
        var target = new Vector3(
            (float)_rng.NextDouble() - 0.5f,
            (float)_rng.NextDouble() - 0.5f,
            0.5f + (float)_rng.NextDouble() - 0.5f);
        return _arm.SolveIK(target);
    }
}
// 14 lines total — BenchmarkDotNet required
// Class per benchmark suite — not integrated with test syntax
```

**In Python 3.13:**
```python
# Python benchmark — timeit or pytest-benchmark (third-party)
import timeit
import random

rng = random.Random(42)
arm = RobotArm.create_test_arm()

def benchmark_ik_solver():
    target = (rng.uniform(-0.5, 0.5),
              rng.uniform(-0.5, 0.5),
              0.5 + rng.uniform(-0.5, 0.5))
    return arm.solve_ik(target)

result = timeit.timeit(benchmark_ik_solver, number=1000)
print(f"Average: {result/1000*1000:.2f}ms")
# 10 lines total — no assertion support, manual stat calculation
// 10 lines total — no p99 assertion, no built-in stats
```

**In Skev:**
```swift
bench "robot arm IK solver — real-time constraint" >>

    arm :: RobotArm = RobotArm.create_test_arm()
    rng :: math.Random = math.Random.seeded(42)

    # bench_run executes 1000 times — collects timing statistics
    bench_run >>
        target :: Vector3! = Vector3!(
            rng.float(-0.5, 0.5),
            rng.float(-0.5, 0.5),
            0.5 + rng.float(-0.5, 0.5)
        )
        angles :: JointAngles = arm.solve_ik(target)
    << bench_run

    # Assert statistical performance requirements
    assert bench.median_ms < 2.0   "Median IK solve under 2ms"
    assert bench.p95_ms    < 4.0   "p95 IK solve under 4ms"
    assert bench.p99_ms    < 5.0   "p99 IK solve under 5ms ← real-time requirement"
    assert bench.max_ms    < 10.0  "Max IK solve under 10ms ← safety margin"

    # Report for CI dashboard
    log.info("ik_bench", >>
        median_ms :: bench.median_ms
        p99_ms    :: bench.p99_ms
        iterations :: bench.iteration_count
    << )

<< bench
// 26 lines total — built-in stats, p99 assertions, CI reporting
```

### ✅ Honest Advantage Statement

Skev's bench block (26 lines) is longer than Python (10 lines) but Python provides no statistical assertions — just timing output. At equivalent functionality (p99 assertions + reporting), all languages are comparable in length. The built-in advantage: `bench.median_ms`, `bench.p95_ms`, `bench.p99_ms` are first-class — no manual calculation. The `log.info` call at the end feeds CI dashboards automatically. Google Benchmark and BenchmarkDotNet are excellent tools — Skev's bench block is not better, it is integrated.

---

## 11.3 — Observability

### 👨‍💻 Developer Version

Observability is the ability to understand what your program is doing — in development and in production. Skev provides three tools: structured logging, performance tracing, and crash reporting. All three are in `skev.log`.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A live-service game has 500,000 daily players. A bug causes random crashes for 0.1% of players — about 500 people per day. The crash reports must: (1) identify the crash location, (2) capture game state at crash time, (3) be searchable across thousands of reports, and (4) not collect personally identifiable information. Additionally, the game team needs performance metrics: which scenes are slow, what is the median frame time, and which systems exceed their time budgets.

**In C++23:**
```cpp
// C++ logging — spdlog (third-party) for structured logs
#include <spdlog/spdlog.h>
#include <spdlog/sinks/rotating_file_sink.h>

// Setup (50+ lines of configuration)
auto logger = spdlog::rotating_logger_mt(
    "game", "logs/game.log", 1024*1024*5, 3);

// Structured log (unformatted — just strings in practice)
spdlog::info("player_spawned player_id={} x={} y={} z={}",
    player.id, player.pos.x, player.pos.y, player.pos.z);

// Crash handling — platform specific (Windows SEH, Unix signals)
// No standard cross-platform crash reporting
// ~80 lines total just for crash handling setup
// 7 lines shown — 80+ total for full setup
```

**In C# 13:**
```csharp
// C# logging — Microsoft.Extensions.Logging (built-in but verbose)
// Or Serilog (third-party — better structured logging)
using var log = LoggerFactory.Create(b =>
    b.AddConsole().AddFile("logs/game.log"))
    .CreateLogger<GameSystem>();

log.LogInformation("player_spawned {@Player}",
    new { player.Id, player.Position });

// Crash reporting — crash services (App Center, Sentry, etc.)
// Third-party integration required for production crash reporting
// 7 lines total — but setup and crash reporting are third-party
```

**In Python 3.13:**
```python
import logging
import json

logging.basicConfig(
    format='%(asctime)s %(name)s %(levelname)s %(message)s',
    filename='game.log'
)
logger = logging.getLogger('game')

# Structured log (manual JSON)
logger.info(json.dumps({
    "event": "player_spawned",
    "player_id": player.id,
    "position": list(player.position)
}))
# 11 lines total — manual JSON, no crash integration
// 11 lines total — verbose setup, manual JSON structure
```

**In Skev:**
```swift
import skev.log

# Structured logging — fields are typed, not strings
log.info("player_spawned", >>
    player_id :: player.id
    position  :: player.position
    level     :: current_level.name
<< )

# Performance trace — measures execution time of a block
trace "enemy_pathfinding" >>
    path :: maybe list[Vector3!] = nav.find_path(
        enemy.position,
        player.position
    )
<< trace
# Automatically records: start time, end time, duration
# Feeds into Skev Studio profiler timeline
# Feeds into skev.log metrics pipeline

# Warning with context
if frame_delta > 0.033 >>
    log.warn("frame_time_exceeded", >>
        delta_ms   :: frame_delta * 1000.0
        threshold  :: 33.3
        scene_name :: current_scene.name
    << )
<< frame_delta > 0.033

# Crash handler — captures full game state
engine.on_panic >> crash_info
    log.crash("unhandled_panic", >>
        message    :: crash_info.message
        stack      :: crash_info.stack_trace
        scene      :: current_scene.name
        game_time  :: time.elapsed()
        # No player PII — deliberately omitted
    << )
<< on_panic
// 31 lines total — typed fields, trace integration, crash handling
```

### ✅ Honest Advantage Statement

Skev's structured logging (31 lines shown) is comparable to Serilog for C# in concept but requires zero third-party installation. The `>>` field block for log entries integrates with the language's existing block syntax. The `trace` block's automatic connection to Skev Studio's profiler timeline is the most unique feature — no other language in this comparison provides a built-in connection between source-level trace blocks and an IDE profiler. C++ `spdlog` and Python `logging` both require manual performance measurement code.

---

### Log Levels Reference

```swift
# Debug — development only, stripped from release builds
log.debug("entity_update", >>
    entity_id :: entity.id
    delta     :: delta
<< )

# Info — normal operational events
log.info("level_loaded", >>
    level_name    :: level.name
    load_time_ms  :: load_time * 1000.0
    entity_count  :: scene.entity_count
<< )

# Warn — unexpected but handled
log.warn("save_retry", >>
    attempt   :: retry_count
    path      :: save_path
    reason    :: last_error.message
<< )

# Error — something failed but game continues
log.error("network_packet_dropped", >>
    packet_type :: packet.type.as(string)
    sequence    :: packet.sequence
    reason      :: "timeout"
<< )

# Crash — unrecoverable, always captured
log.crash("physics_invariant_violated", >>
    entity_id :: entity.id
    details   :: "ARC count went negative"
<< )
```

---

### 🔴 Hard Complex Example — Production Observability Pipeline

**🌍 Real-World Scenario**

A live-service RPG collects performance metrics, error rates, and player behaviour data. The studio needs: per-scene frame time distribution, system-level time budgets with breach detection, error rates per build version, and an analytics dashboard showing which game systems are causing lag spikes. All data must be: anonymised, structured for querying, low-overhead in production, and GDPR-compliant (no PII).

```swift
import skev.log

# System-level performance monitoring
entity PerformanceMonitor >>

    physics_budget_ms  :: float = 4.0   # budget per frame
    ai_budget_ms       :: float = 3.0
    render_budget_ms   :: float = 8.0
    network_budget_ms  :: float = 2.0

    when update(delta)
        monitor_systems(delta)
    << update

    monitor_systems(delta)

        # Trace each system — feeds profiler timeline
        physics_time :: float64 = time.measure >>
            trace "physics_update" >>
                physics.update(delta)
            << trace
        << time.measure

        ai_time :: float64 = time.measure >>
            trace "ai_update" >>
                ai_director.update(delta)
            << trace
        << time.measure

        render_time :: float64 = time.measure >>
            trace "render_frame" >>
                renderer.render(delta)
            << trace
        << time.measure

        # Record frame metrics — structured, queryable
        frame_ms :: float = delta * 1000.0
        log.info("frame_metrics", >>
            frame_ms       :: frame_ms
            physics_ms     :: physics_time * 1000.0
            ai_ms          :: ai_time * 1000.0
            render_ms      :: render_time * 1000.0
            scene          :: current_scene.name
            build_version  :: app.version
            # No user ID — GDPR compliant
        << )

        # Budget breach detection — warns when systems are slow
        if physics_time * 1000.0 > physics_budget_ms >>
            log.warn("budget_breach", >>
                system     :: "physics"
                actual_ms  :: physics_time * 1000.0
                budget_ms  :: physics_budget_ms
                scene      :: current_scene.name
            << )
        << physics_time * 1000.0 > physics_budget_ms

        if ai_time * 1000.0 > ai_budget_ms >>
            log.warn("budget_breach", >>
                system     :: "ai"
                actual_ms  :: ai_time * 1000.0
                budget_ms  :: ai_budget_ms
                scene      :: current_scene.name
            << )
        << ai_time * 1000.0 > ai_budget_ms

        # Spike detection — frames over 33ms are spikes
        if frame_ms > 33.0 >>
            log.warn("frame_spike", >>
                frame_ms      :: frame_ms
                physics_ms    :: physics_time * 1000.0
                ai_ms         :: ai_time * 1000.0
                render_ms     :: render_time * 1000.0
                likely_cause  :: identify_spike_cause(
                    physics_time, ai_time, render_time
                )
            << )
        << frame_ms > 33.0

    << monitor_systems

    identify_spike_cause(
        physics_ms: float64,
        ai_ms:      float64,
        render_ms:  float64
    ) -> string
        max_time :: float64 = math.max(physics_ms,
                              math.max(ai_ms, render_ms))
        if max_time == physics_ms >> result "physics"  << max_time == physics_ms
        if max_time == ai_ms      >> result "ai"       << max_time == ai_ms
        result "render"
    << identify_spike_cause

<< PerformanceMonitor
// 72 lines total — complete production observability system
```

### 🧭 Walk-Through — Production Observability

**`time.measure >> ... << time.measure`:** Returns execution time of a block in seconds. Combined with `trace` blocks — feeds Skev Studio's profiler timeline AND generates structured log entries. One block does both: human-readable profiling AND machine-queryable metrics.

**`log.info("frame_metrics", >> fields << )`:** The field block uses `::` — same as property declarations. Fields are typed at compile time. Passing `physics_ms :: "slow"` (string instead of float) is a compile error. This is fundamentally different from `printf`-style logging where format mismatches are runtime bugs.

**Budget breach detection:** Each system has a declared budget. If `physics_time > physics_budget_ms` — `log.warn` fires with the system name, actual time, and budget. Studio query: `SELECT scene, AVG(actual_ms) WHERE system='physics' AND actual_ms > budget_ms GROUP BY scene` — immediately shows which scenes are physics-heavy.

**No user ID:** GDPR compliance is explicit. The comment `# No user ID — GDPR compliant` documents the deliberate omission. Analytics pipelines built on structured logs can verify compliance by checking every `log.info` call for PII fields.

---

## 11.4 — Language Server Protocol (LSP)

### 👨‍💻 Developer Version

The Language Server Protocol means one implementation — every editor. `skev-lsp` is the Skev LSP server. VS Code, JetBrains, Neovim, Zed, Helix, and any LSP-compatible editor gets the same Skev support automatically.

### Standard LSP Features

```
Completions:
  → entity Player >> [Tab] → shows all valid event names
  → player.[Tab] → shows all entity properties and methods
  → skev.math.[Tab] → shows all math library functions
  → match state >> [Tab] → shows all kind variants

Diagnostics (real-time):
  → Wrong << label → underlined immediately
  → Missing error handling → warning inline
  → Capability violation → error inline
  → Type mismatch → error inline

Hover:
  → Hover entity name → shows all properties and events
  → Hover function → shows signature and doc comment
  → Hover type → shows full type definition

Go to definition:
  → Cmd+Click any entity, function, or type
  → Works across files in the project

Find all references:
  → Right-click any name → all usages in project

Rename:
  → Rename entity → updates all references
  → Rename property → updates all accesses

Format:
  → skev format or editor format command
  → 4-space indentation, consistent spacing
  → Consistent >> << alignment
```

### Skev-Specific LSP Features

```swift
# Feature: Block pair highlighting
# Hover << Dragon → highlights matching >> Dragon
entity Dragon >>        # ← highlighted when hovering <<
    health :: int = 200
    when update(delta)
        patrol(delta)
    << update           # ← hover this to see matching when
<< Dragon               # ← hover this to see matching entity Dragon

# Feature: Incomplete << label detection
entity Player >>
    health :: int = 100
    when update(delta)
        move(input.direction * speed * delta)
    << updat             # ← LSP: "Did you mean << update?"
<< Player               # ← error: << label 'updat' doesn't match 'update'

# Feature: kind exhaustion hint
kind GameState >>
    menu
    playing
    paused
    game_over
<< GameState

match state >>
    menu    -> show_menu()
    playing -> run_game()
    # LSP: "Match is non-exhaustive. Missing: paused, game_over"
<< state
```

### Entity Graph View (Skev Studio)

```
Skev Studio shows a live entity graph:

Player ──── has ──── Physics
        └── has ──── AudioSource
        └── has ──── NetworkSync

Enemy  ──── has ──── Physics
        └── has ──── AIBrain
        └── fires ─→ DamageEvent ─→ Player.on_damage

This view updates as you edit code.
Clicking any node jumps to that entity's file.
```

---

## 11.5 — TextMate Grammar

### 👨‍💻 Developer Version

The TextMate grammar formally defines Skev syntax highlighting for every editor. This is the chapter that makes `>>`, `<<`, `::`, `when`, `has`, and all other Skev-specific syntax highlight correctly.

**After Chapter 11:** All spec documents switch from ` ```swift ` to ` ```skev ` for code blocks. GitHub renders Skev correctly via the linguist entry. VS Code highlights Skev via the extension.

### Token Categories

```
Source scope:  source.skev

KEYWORDS (red #FF7B72):
  entity component data kind alias
  when has fixed shared hidden weak maybe
  if else match loop stop skip
  result succeed fail or_else assert
  and or not is in contains exists
  async await task cancel realtime
  extern unsafe export cdata callback
  sandbox test bench mock
  import

GAME-NATIVE TYPES (bright blue #79C0FF):
  Vector2! Vector3! Vector4!
  Quat! Color! Transform!
  Rect! Ray!
  Any identifier ending in !

OPERATORS (warm orange #FF9E64):
  >> << :: -> = += -= *= /=
  < > <= >= == != not

BLOCK LABELS (bright green #7EE787):
  Text following << to end of line
  Example: << Dragon   << update   << health < 300

STRINGS (light blue #A5D6FF):
  "..." literals
  {expression} inside strings — highlighted differently

NUMBERS (bright blue #79C0FF):
  Integer literals: 42, 1_000_000
  Float literals:   3.14, 2.0
  Precision:        uint8, int32, float64

COMMENTS (muted grey #8B949E):
  # single line comment
  #{ multi-line comment }#

DOC COMMENTS (green #3FB950):
  #! documentation comment
  #! sandboxed
  #! serialisable(version: N)
  #! safety_critical
  #! introspectable

TYPE NAMES (lavender #D2A8FF):
  PascalCase identifiers: Player, DragonBoss, Vector3

PROPERTIES (amber #FFA657):
  snake_case identifiers after ::
  snake_case identifiers after .
```

### Publication Path

```
Step 1 — skev.tmLanguage.json published to GitHub
  → github.com/skev-lang/skev-vscode
  → MIT license — any editor can use it

Step 2 — VS Code extension published
  → marketplace.visualstudio.com
  → Name: "Skev Language Support"
  → Publisher: skev-lang
  → Features: syntax highlighting + LSP client

Step 3 — GitHub linguist registration
  → PR to github/linguist
  → lib/linguist/languages.yml entry for Skev
  → Extension: .skev
  → GitHub renders Skev code with correct highlighting

Step 4 — JetBrains plugin
  → Uses same skev-lsp server
  → Plugin: plugins.jetbrains.com/skev
  → Works in: Rider, IntelliJ, CLion

Step 5 — Documentation site (skev.dev)
  → Prism.js plugin: prism-skev.js
  → All code examples on skev.dev highlighted correctly
```

---

## 11.6 — REPL & Playground

### 👨‍💻 Developer Version

```
skev repl              # interactive REPL in terminal
skev play              # opens browser at skev.dev/play
```

**Terminal REPL:**
```
$ skev repl
Skev 1.0.0 REPL — type .help for commands

>> health :: int = 100
→ int: 100

>> damage :: int = 30

>> health - damage
→ int: 70

>> entity Player >>
..     health :: int = 100
.. << Player
→ defined entity Player

>> p :: Player = Player >> health :: 80 << Player
→ Player { health: 80 }

>> p.health
→ int: 80

>> .type p
→ Player (entity, ARC count: 1)

>> .clear
Cleared all definitions.

>> .load examples/combat.skev
Loaded: CombatSystem, DamageEvent, AttackResult

>> .exit
$
```

**Online Playground (skev.dev/play):**
```
→ Full Skev editor in browser (Monaco-based)
→ Syntax highlighting via TextMate grammar
→ Runs Skev via WebAssembly (Skev compiler compiled to WASM)
→ Share button: generates permalink to code
→ Examples library: one-click load of spec examples
→ Output panel: shows print/log output
→ Error panel: shows compiler errors with source locations
→ No installation required — immediate Skev experience
```

---

## 11.7 — Documentation Generator (`skev doc`)

### 👨‍💻 Developer Version

`#!` doc comments generate API documentation. `skev doc` builds the docs.

```swift
#! Represents a player character in the game world.
#! Players have health, can take damage, and respond to input.
entity Player >>

    #! Current health points. Range: [0, max_health].
    #! When health reaches 0, fires the player_died event.
    health :: int = 100

    #! Maximum health. Cannot be exceeded by healing.
    max_health :: int = 100

    #! Apply damage to this player.
    #! @param amount Damage to apply. Must be positive.
    #! @example
    #!   player.take_damage(50)    # player.health = 50
    #!   player.take_damage(1000)  # player.health = 0 (floored)
    take_damage(amount: int)
        health = math.max(0, health - amount)
        if health == 0 >>
            event player_died
        << health == 0
    << take_damage

<< Player
```

**`skev doc` output:**

```
skev doc              # generate docs to docs/ directory
skev doc --serve      # serve docs at localhost:8080
skev doc --format md  # generate Markdown (for GitHub wiki)
skev doc --format html # generate HTML (for website)
```

---

## 11.8 — Skev Studio

### 👨‍💻 Developer Version

Skev Studio is the dedicated IDE for Skev. It goes beyond what an LSP extension provides:

**Scene Editor:**
```
Visual drag-and-drop entity placement
Component inspector (click entity → see all properties)
Live edit: change health :: 100 in inspector → updates in running game
Entity hierarchy tree with has relationships visible
```

**Profiler:**
```
Frame timeline showing all trace blocks
Per-system time breakdown (physics / AI / render / network)
Frame spike detection with cause identification
Memory usage per entity type
ARC reference graph (see which entities hold references)
```

**Hot Reload:**
```
Save .skev file → game updates within ~200ms
No restart, no re-entering the level
Works for: logic changes, property value changes
Does NOT work for: entity struct changes (full reload needed)
Compiler tells you which changes need full reload
```

**Debugger:**
```
Breakpoints in .skev files
Step through when event handlers
Inspect entity state at any breakpoint
Conditional breakpoints (break when health < 10)
Watch expressions
```

**Skev Studio Tiers (Rule J):**
```
Community (free):
  All above features
  Unlimited projects
  Local development only

Pro (commercial subscription):
  Cloud build (skev build on remote servers)
  Team collaboration (shared scene editing)
  Console dev tools (PS5/Xbox/Switch devkits)
  Advanced profiler (allocation tracker, ARC graph)
  Priority support
```

---

## 11.9 — Tooling Quick Reference

```
TESTING:
  test "description" >>    ...    << test
  test async "description" >> ... << test
  test_setup >> ... << test_setup
  bench "description" >>
      bench_run >> ... << bench_run
      assert bench.median_ms < N
  << bench
  mock EntityName >> ... << mock EntityName
  skev test           run all tests
  skev test --bench   run benchmarks too
  skev test --watch   re-run on file change

OBSERVABILITY:
  import skev.log
  log.debug/info/warn/error/crash("event", >> fields << )
  trace "name" >> ... << trace
  time.measure >> ... << time.measure  → float64 (seconds)

REPL:
  skev repl        terminal REPL
  skev play        open browser playground
  .load file.skev  load a file in REPL
  .type expr       show type of expression
  .clear           reset all definitions
  .exit            quit REPL

DOCUMENTATION:
  #! Doc comment line
  skev doc              generate docs
  skev doc --serve      serve locally
  skev doc --format md  Markdown output

LSP:
  skev-lsp              LSP server binary
  Works with: VS Code, JetBrains, Neovim, Zed, Helix

TEXTMATE:
  scope: source.skev
  extension: .skev
  grammar: skev.tmLanguage.json
  VS Code extension: skev.vscode-skev

SKEV STUDIO:
  Scene editor, entity inspector, profiler
  Hot reload, debugger, breakpoints
  Community: free   Pro: commercial
```

---

*End of Chapter 11 v0.1 — The Skev specification is now complete (Chapters 1–11 + 3.5)*
*Copyright © 2026 AJ. All Rights Reserved. https://skev.dev | https://skev.org*
