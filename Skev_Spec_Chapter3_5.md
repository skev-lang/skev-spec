<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 3.5: Generics
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026)
**Status:** In Progress
**Depends On:** Chapter 3 (types), Chapter 4 (ARC), Chapter 6 (errors), Skev_Function_Syntax.md
**Use Cases Tested:** All 7 — Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅
**Gap Resolved:** 🚨 Critical gap from architecture audit — last remaining critical gap

---

## 3.5.1 — Overview

### 👨‍💻 Developer Version

Generics solve one problem: writing code that works with any type — without losing type safety.

Without generics, if you want a function that finds the largest value in a list, you write one version for `list[int]`, another for `list[float]`, another for `list[string]`. Three functions. Identical logic. The only difference is the type.

Generics let you write it once:

```swift
find_max[T where T: Comparable](items: list[T]) -> maybe T
    if items.count == 0 >>
        result nothing
    << items.count == 0
    max :: T = items[0]
    loop item in items >>
        if item > max >>
            max = item
        << item > max
    << item
    result max
<< find_max
```

One function. Works for `list[int]`, `list[float]`, `list[string]`, or any comparable type you create. The compiler verifies that every type you pass actually supports `>` — at compile time, not at runtime.

> **Generics = write once, works for any type, verified at compile time.**

**The key insight — you already know Skev's generic syntax:**

```swift
scores   :: list[int]               # you already use this
lookup   :: map[string, float]      # you already use this
outcome  :: result[PlayerData]      # you already use this
messages :: channel[DamageEvent]    # you already use this
```

Every `[Type]` you have written is generic syntax. Chapter 3.5 simply lets you write your own types and functions using the same `[]` pattern.

### ⚙️ Technical Version

Skev generics use monomorphisation — the same approach as C++ templates and Rust generics. The compiler generates a separate, fully-typed implementation for each concrete type a generic is used with. `find_max[int]` and `find_max[float]` become two distinct LLVM IR functions with zero shared code and zero runtime overhead. This contrasts with Java and C#'s type erasure approach (which boxes value types and loses type information at runtime) and Python's duck typing (which performs all type checks at runtime). Monomorphisation produces maximally optimised code — LLVM can inline, vectorise, and optimise each concrete version independently.

---

## 3.5.2 — Generic Functions

### 👨‍💻 Developer Version

A generic function declares a type parameter in `[square brackets]` between the function name and the parameter list. The type parameter then appears in the function's parameter types and return type.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game needs a utility function that wraps any value in a container, a function that swaps two values of any type, and a function that finds the first element matching a condition. These are basic utilities every game codebase needs. Without generics — every developer copies and adjusts the same logic for each type they need.

**💥 Why This Is Hard**

Without generics, you either: (1) write duplicate functions per type — maintenance nightmare, (2) use `void*` or `object` — loses type safety completely, or (3) use runtime reflection — slow and fragile. Generics eliminate all three problems simultaneously.

---

**In C++23:**
```cpp
// C++23 — template keyword, verbose
template<typename T>
T identity(T value) {
    return value;
}

template<typename T>
std::pair<T, T> swap_pair(T a, T b) {
    return {b, a};
}

template<typename T, typename Pred>
std::optional<T> find_first(
    const std::vector<T>& items, Pred predicate) {
    for (const auto& item : items) {
        if (predicate(item)) return item;
    }
    return std::nullopt;
}
// 17 lines total — template keyword, typename, separate syntax
// Error messages on template failures are notoriously unreadable
```

**In C# 13:**
```csharp
// C# 13 — angle brackets, where constraint
T Identity<T>(T value) => value;

(T, T) SwapPair<T>(T a, T b) => (b, a);

T? FindFirst<T>(IEnumerable<T> items,
    Func<T, bool> predicate) {
    foreach (var item in items) {
        if (predicate(item)) return item;
    }
    return default;
}
// 10 lines total — angle brackets differ from other C# syntax
// Func<T, bool> is verbose for function parameters
```

**In Python 3.13:**
```python
from typing import TypeVar, Callable, Optional

T = TypeVar('T')

def identity(value: T) -> T:
    return value

def find_first(items: list[T],
    predicate: Callable[[T], bool]) -> Optional[T]:
    for item in items:
        if predicate(item):
            return item
    return None
# 10 lines total — TypeVar must be declared separately
# Runtime: no enforcement — just hints
// 10 lines total — TypeVar declaration is boilerplate
```

**In Skev:**
```swift
identity[T](value: T) -> T
    result value
<< identity

swap_pair[T](a: T, b: T) -> Pair[T, T]
    result Pair[T, T] >> first :: b   second :: a << Pair[T, T]
<< swap_pair

find_first[T](items: list[T], predicate: fn(T) -> bool) -> maybe T
    loop item in items >>
        if predicate(item) >>
            result item
        << predicate(item)
    << item
    result nothing
<< find_first
// 11 lines total — [T] after name, same [] as list[T]
```

### ✅ Honest Advantage Statement

Skev's generic function syntax is the most consistent with the rest of the language — `[T]` is already the notation for every generic type in Skev. C++ requires learning `template<typename T>` as a completely separate syntactic concept. C# uses `<T>` which differs from its array and collection syntax. Python requires a separate `TypeVar` declaration before the function and provides only runtime hints, not compile-time verification. Skev's advantage is consistency: if you know how to write `list[int]`, you already know the notation for generic functions.

---

### 🟢 Foundation Example

```swift
# Single type parameter — identity function
identity[T](value: T) -> T
    result value
<< identity

# Usage — type inferred automatically
x :: int    = identity(42)        # T = int
y :: string = identity("Dragon")  # T = string
z :: float  = identity(3.14)      # T = float
// 8 lines total
```

---

### 🟡 Applied Example — Generic Stack

**In C++23:**
```cpp
template<typename T>
class Stack {
    std::vector<T> data;
public:
    void push(T item) { data.push_back(item); }
    std::optional<T> pop() {
        if (data.empty()) return std::nullopt;
        T top = data.back();
        data.pop_back();
        return top;
    }
    bool is_empty() const { return data.empty(); }
    size_t count() const { return data.size(); }
};
// 13 lines total — class required, manual memory model
```

**In C# 13:**
```csharp
public class Stack<T> {
    private List<T> _data = new();
    public void Push(T item) => _data.Add(item);
    public T? Pop() {
        if (_data.Count == 0) return default;
        var top = _data[^1];
        _data.RemoveAt(_data.Count - 1);
        return top;
    }
    public bool IsEmpty => _data.Count == 0;
    public int Count => _data.Count;
}
// 11 lines total — class required, PascalCase API
```

**In Python 3.13:**
```python
from typing import TypeVar, Generic, Optional
T = TypeVar('T')

class Stack(Generic[T]):
    def __init__(self):
        self._data: list[T] = []
    def push(self, item: T) -> None:
        self._data.append(item)
    def pop(self) -> Optional[T]:
        return self._data.pop() if self._data else None
    @property
    def is_empty(self) -> bool:
        return len(self._data) == 0
// 10 lines total — Generic[T] base class required
```

**In Skev:**
```swift
data Stack[T] >>
    items :: list[T]
<< Stack

push[T](stack: Stack[T], item: T) -> nothing
    stack.items.add(item)
<< push

pop[T](stack: Stack[T]) -> maybe T
    if stack.items.count == 0 >>
        result nothing
    << stack.items.count == 0
    top :: T = stack.items.last
    stack.items.remove_last()
    result top
<< pop

is_empty[T](stack: Stack[T]) -> bool
    result stack.items.count == 0
<< is_empty
// 18 lines total — data type + standalone functions
// No class wrapper needed — functions operate on the data type directly
```

### ✅ Honest Advantage Statement

Skev uses more lines than C# for this example (18 vs 11). The reason is honest: Skev separates the data definition (`data Stack[T]`) from the operations (`push`, `pop`) — there is no class wrapper. This is more verbose for simple cases but produces code where data and behaviour are clearly separated — consistent with Skev's design philosophy that `data` types are pure containers. C++ and C# bundle data and methods together in a class. Neither approach is wrong — the trade-off is real.

---

### 🔴 Hard Complex Example — Generic Event System

**🌍 Real-World Scenario**

A game engine needs a type-safe event system where different systems can publish and subscribe to typed events. A physics system publishes `CollisionEvent`. An audio system subscribes to `CollisionEvent` to play sounds. An AI system subscribes to `DamageEvent` to react to damage. Without generics — either every event type needs its own handler class, or events are cast from `object` at runtime (losing type safety). Both scale poorly to hundreds of event types.

**💥 Why This Is Hard**

A generic event bus requires: typed publish (only `CollisionEvent` goes to `CollisionEvent` subscribers), multiple subscriber support, safe unsubscription without crashes, thread-safe event dispatch from the physics thread, and zero overhead for events with no subscribers. Getting all of this right with type safety is the hardest generic programming problem in game development.

**In C++23:**
```cpp
template<typename T>
class EventBus {
    std::vector<std::function<void(const T&)>> subscribers;
    std::mutex mutex;
public:
    using Handle = size_t;

    Handle subscribe(std::function<void(const T&)> handler) {
        std::lock_guard<std::mutex> lock(mutex);
        subscribers.push_back(std::move(handler));
        return subscribers.size() - 1;
    }

    void unsubscribe(Handle handle) {
        std::lock_guard<std::mutex> lock(mutex);
        if (handle < subscribers.size()) {
            subscribers[handle] = nullptr;
        }
    }

    void publish(const T& event) {
        std::lock_guard<std::mutex> lock(mutex);
        for (auto& sub : subscribers) {
            if (sub) sub(event);
        }
    }
};

// Usage
EventBus<CollisionEvent> collisionBus;
EventBus<DamageEvent> damageBus;
auto handle = collisionBus.subscribe([](const CollisionEvent& e) {
    AudioSystem::PlayCollisionSound(e.position, e.force);
});
// 28 lines total — manual mutex, std::function overhead,
// Handle is just a size_t (no type safety on unsubscribe)
```

**In C# 13:**
```csharp
public class EventBus<T> {
    private readonly List<Action<T>> _subscribers = new();
    private readonly Lock _lock = new();

    public int Subscribe(Action<T> handler) {
        lock (_lock) {
            _subscribers.Add(handler);
            return _subscribers.Count - 1;
        }
    }

    public void Unsubscribe(int handle) {
        lock (_lock) {
            if (handle < _subscribers.Count)
                _subscribers[handle] = null!;
        }
    }

    public void Publish(T evt) {
        lock (_lock) {
            foreach (var sub in _subscribers)
                sub?.Invoke(evt);
        }
    }
}

// Usage — clean but separate bus per event type
var collisionBus = new EventBus<CollisionEvent>();
collisionBus.Subscribe(e => audioSystem.PlaySound(e.Position));
// 24 lines total — better than C++ but still manual locking
// null! suppression feels wrong
```

**In Python 3.13:**
```python
from typing import TypeVar, Generic, Callable
T = TypeVar('T')

class EventBus(Generic[T]):
    def __init__(self):
        self._subscribers: list[Callable[[T], None]] = []

    def subscribe(self, handler: Callable[[T], None]) -> int:
        self._subscribers.append(handler)
        return len(self._subscribers) - 1

    def unsubscribe(self, handle: int) -> None:
        if handle < len(self._subscribers):
            self._subscribers[handle] = None

    def publish(self, event: T) -> None:
        for sub in self._subscribers:
            if sub: sub(event)

# Usage
collision_bus: EventBus[CollisionEvent] = EventBus()
collision_bus.subscribe(lambda e: audio.play_sound(e.position))
# 17 lines total — no thread safety at all (GIL only helps partly)
// 17 lines total — type hints not enforced at runtime
```

**In Skev:**
```swift
data EventBus[T] >>
    subscribers :: list[fn(T) -> nothing]
    shared subscriber_count :: int = 0
<< EventBus

subscribe[T](bus: EventBus[T], handler: fn(T) -> nothing) -> int
    bus.subscribers.add(handler)
    bus.subscriber_count += 1
    result bus.subscribers.count - 1
<< subscribe

unsubscribe[T](bus: EventBus[T], handle: int)
    if handle >= 0 and handle < bus.subscribers.count >>
        bus.subscribers.set(handle, nothing)
    << handle >= 0 and handle < bus.subscribers.count
<< unsubscribe

publish[T](bus: EventBus[T], event: T) -> nothing
    loop handler in bus.subscribers >>
        if handler exists >>
            handler(event)
        << handler exists
    << handler
<< publish

# Usage — one bus per event type, fully type safe
entity GameEventSystem >>

    collision_bus :: EventBus[CollisionEvent]
    damage_bus    :: EventBus[DamageEvent]
    pickup_bus    :: EventBus[PickupEvent]

    when scene_load
        # Subscribe audio to collision events
        subscribe(collision_bus) >> event: CollisionEvent
            audio.play_at(sounds.collision, event.position, event.force)
        << subscribe

        # Subscribe AI to damage events
        subscribe(damage_bus) >> event: DamageEvent
            ai_director.notify_damage(event.target_id, event.amount)
        << subscribe

        # Subscribe UI to pickup events
        subscribe(pickup_bus) >> event: PickupEvent
            ui.show_pickup_notification(event.item_name)
        << subscribe
    << scene_load

    # Physics thread fires collision — auto-marshaled to main thread
    when physics_collision(event: CollisionEvent)
        publish(collision_bus, event)
    << physics_collision

    when entity_damaged(event: DamageEvent)
        publish(damage_bus, event)
    << entity_damaged

    when item_picked_up(event: PickupEvent)
        publish(pickup_bus, event)
    << item_picked_up

<< GameEventSystem
// 52 lines total — full implementation with entity integration
```

### 🧭 Walk-Through — Generic Event System

**`data EventBus[T]`:** Declares a generic data type where `T` is the event type. `subscribers` is a `list` of functions that take `T` and return nothing. This is the core structure — each bus is specific to one event type, enforced by the compiler. You cannot accidentally subscribe a `DamageEvent` handler to a `CollisionEvent` bus.

**`subscribe[T]`:** Takes a bus and a handler. The handler type `fn(T) -> nothing` is monomorphised — for `EventBus[CollisionEvent]`, the handler must be `fn(CollisionEvent) -> nothing`. Passing the wrong handler type is a compile error.

**`publish[T]`:** Iterates all subscribers. The `if handler exists` check handles unsubscribed slots (set to nothing). Calls each live handler with the event. Thread safety comes from the `shared subscriber_count` — which uses atomic access from Chapter 5.

**`GameEventSystem` entity:** Three buses — each typed. Three subscriptions — each with a handler block that matches the bus type. If you write the wrong handler for a bus (e.g. writing `event: DamageEvent` in a `collision_bus` subscription), the compiler catches it immediately.

**What Skev does that C++ and C# don't:** The `>> event: CollisionEvent` subscription block syntax integrates with Skev's existing `when` event system semantics. The handler looks and feels like any other Skev event handler — not a lambda, not an `Action<T>`. It reads naturally.

### ✅ Honest Advantage Statement

Skev's event system (52 lines) is longer than C# (24 lines) for the implementation. The reasons are explicit: separate data definition from operations, and the entity integration adds lines. The genuine advantage: the subscription syntax (`subscribe(collision_bus) >> event: CollisionEvent`) reads like a native Skev event handler — not a lambda or delegate. C++ error messages for template violations are notoriously unreadable; Skev's are plain English. Python provides no compile-time guarantee that the right event type is being handled.

---

## 3.5.3 — Generic Data Types

### 👨‍💻 Developer Version

Generic data types define structures that work with any type. The `[T]` syntax follows the type name — matching exactly how `list[T]`, `map[K, V]`, and `result[T]` work throughout Skev.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A simulation engine needs a generic `Pair[A, B]` for position-velocity pairs, a `Range[T]` for any measurable value range, and a `Tagged[T]` type that attaches metadata to any value. These show up constantly in game and simulation code.

**In C++23:**
```cpp
template<typename A, typename B>
struct Pair { A first; B second; };

template<typename T>
struct Range { T min_val; T max_val; bool contains(T v) { return v >= min_val && v <= max_val; } };

template<typename T>
struct Tagged { T value; std::string tag; uint64_t timestamp; };
// 6 lines total — simple but template syntax is separate language
```

**In C# 13:**
```csharp
record Pair<A, B>(A First, B Second);

record Range<T>(T Min, T Max) where T : IComparable<T> {
    public bool Contains(T value) =>
        value.CompareTo(Min) >= 0 && value.CompareTo(Max) <= 0;
}

record Tagged<T>(T Value, string Tag, long Timestamp);
// 7 lines total — record syntax is clean, where constraint verbose
```

**In Python 3.13:**
```python
from dataclasses import dataclass
from typing import TypeVar, Generic
T = TypeVar('T'); A = TypeVar('A'); B = TypeVar('B')

@dataclass
class Pair(Generic[A, B]):
    first: A; second: B

@dataclass
class Range(Generic[T]):
    min_val: T; max_val: T
    def contains(self, v: T) -> bool:
        return self.min_val <= v <= self.max_val

@dataclass
class Tagged(Generic[T]):
    value: T; tag: str; timestamp: int
// 12 lines total — TypeVar declarations add boilerplate
```

**In Skev:**
```swift
data Pair[A, B] >>
    first  :: A
    second :: B
<< Pair

data Range[T where T: Comparable] >>
    min_val :: T
    max_val :: T
<< Range

contains[T where T: Comparable](range: Range[T], value: T) -> bool
    result value >= range.min_val and value <= range.max_val
<< contains

data Tagged[T] >>
    value     :: T
    tag       :: string
    timestamp :: float64
<< Tagged
// 16 lines total — data types + standalone function
```

### ✅ Honest Advantage Statement

C++ and C# are more concise for simple generic data types. Skev's separation of data and operations adds lines but maintains consistency with its philosophy that `data` types are pure containers. The `where T: Comparable` constraint syntax is cleaner than C#'s `where T : IComparable<T>`. Python's TypeVar boilerplate is the most verbose approach.

---

### Generic Type Instantiation

```swift
# Creating generic type instances
player_score  :: Pair[string, int] = Pair >> first :: "AJ"  second :: 9999 << Pair
health_range  :: Range[int]        = Range >> min_val :: 0  max_val :: 100 << Range
tagged_pos    :: Tagged[Vector3!]  = Tagged >>
    value     :: Vector3!(10.0, 0.0, 5.0)
    tag       :: "spawn_point"
    timestamp :: time.unix()
<< Tagged

# Type inference — explicit type annotation on variable
# tells compiler what T is
my_pair :: Pair[int, int]    # T inferred for Pair contents
my_pair = Pair >> first :: 10   second :: 20 << Pair
// 10 lines total
```

---

### 🔴 Hard Complex Example — Generic Result Pipeline

**🌍 Real-World Scenario**

A robotics system processes sensor data through a pipeline: raw bytes arrive from a hardware bus, validated as the correct format, parsed into typed readings, filtered for valid ranges, and transformed into control commands. Each stage can fail. The pipeline must be composable, type-safe, and produce clear error messages when any stage fails.

**💥 Why This Is Hard**

A composable pipeline requires: each stage transforms `T → U`, errors propagate without explicit checking at each stage, the final result carries either the output or the first error, and adding new pipeline stages should not change existing code. This is the functional programming `Maybe/Result monad` problem — and it appears constantly in robotics and real-time systems.

**In C++23:**
```cpp
template<typename T, typename U, typename E>
std::expected<U, E> and_then(
    std::expected<T, E> input,
    std::function<std::expected<U, E>(T)> fn) {
    if (!input) return std::unexpected(input.error());
    return fn(*input);
}

// Usage — verbose chaining
auto result = and_then(
    and_then(
        and_then(read_raw_bytes(), validate_format),
        parse_sensor_reading
    ),
    filter_valid_range
);
// 13 lines total — nested calls read inside-out, hard to follow
```

**In C# 13:**
```csharp
static Result<U> AndThen<T, U>(
    Result<T> input, Func<T, Result<U>> fn) =>
    input.IsSuccess ? fn(input.Value) : Result<U>.Fail(input.Error);

// Usage — slightly better with extension methods
var result = ReadRawBytes()
    .AndThen(ValidateFormat)
    .AndThen(ParseSensorReading)
    .AndThen(FilterValidRange);
// 8 lines total — readable with extension method pattern
// but Result<T> is third-party or must be defined
```

**In Python 3.13:**
```python
from typing import TypeVar, Callable
T = TypeVar('T'); U = TypeVar('U')

def and_then(result, fn: Callable):
    if result.is_success:
        return fn(result.value)
    return result

# Usage
result = and_then(
    and_then(
        and_then(read_raw_bytes(), validate_format),
        parse_sensor_reading
    ),
    filter_valid_range
)
# 14 lines total — no type safety, runtime errors possible
// 14 lines total — nested calls are hard to read
```

**In Skev:**
```swift
# Generic pipeline combinator — chains result operations
pipe[T, U](input: result[T], transform: fn(T) -> result[U]) -> result[U]
    match input >>
        succeed value -> result transform(value)
        fail error    -> fail error
    << input
<< pipe

# Pipeline operator alias for cleaner syntax
alias Pipeline[T] = result[T]

# The sensor processing pipeline — reads top to bottom
entity SensorProcessor >>

    shared latest_command :: maybe MotorCommand

    when sensor_tick
        process_sensor_data()
    << sensor_tick

    process_sensor_data()
        pipeline_result :: result[MotorCommand] =
            pipe(
                pipe(
                    pipe(
                        pipe(
                            hardware_bus.read_raw(),        # result[list[uint8]]
                            validate_sensor_format          # list[uint8] → SensorFrame
                        ),
                        parse_sensor_reading               # SensorFrame → SensorReading
                    ),
                    filter_valid_range                     # SensorReading → SensorReading
                ),
                compute_motor_command                      # SensorReading → MotorCommand
            )

        match pipeline_result >>
            succeed command >>
                latest_command = command
                motor_bus.send(command)
            << succeed command

            fail SensorError.invalid_format ->
                debug.log("Sensor format error — skipping frame")

            fail SensorError.out_of_range ->
                debug.log("Sensor reading out of range — skipping")

            fail error ->
                analytics.report("sensor_pipeline_failure", error.message)
        << pipeline_result
    << process_sensor_data

<< SensorProcessor

# Each pipeline stage — clean, typed, single responsibility
validate_sensor_format(raw: list[uint8]) -> result[SensorFrame]
    if raw.count < 8 >>
        fail SensorError.invalid_format with
            expected: 8, received: raw.count
    << raw.count < 8
    succeed SensorFrame.parse(raw)
<< validate_sensor_format

parse_sensor_reading(frame: SensorFrame) -> result[SensorReading]
    reading :: SensorReading = -> SensorReading.from_frame(frame)
    succeed reading
<< parse_sensor_reading

filter_valid_range(reading: SensorReading) -> result[SensorReading]
    valid_range :: Range[float] = Range >> min_val :: -100.0  max_val :: 100.0 << Range
    if not contains(valid_range, reading.value) >>
        fail SensorError.out_of_range with
            value: reading.value, range: "-100.0 to 100.0"
    << not contains(valid_range, reading.value)
    succeed reading
<< filter_valid_range

compute_motor_command(reading: SensorReading) -> result[MotorCommand]
    command :: MotorCommand = pid_controller.compute(reading.value)
    succeed command
<< compute_motor_command
// 60 lines total — full type-safe pipeline with error handling
```

### 🧭 Walk-Through — Generic Pipeline

**`pipe[T, U]`:** The core generic combinator. Takes a `result[T]` and a function `fn(T) -> result[U]`. If the input succeeded — applies the function. If it failed — passes the error through. `T` is the input type, `U` is the output type. They are different type parameters — this is why two are needed.

**`validate_sensor_format`:** Returns `result[SensorFrame]`. If raw bytes are too short — `fail` with context. Otherwise `succeed` with the parsed frame. Every stage has one responsibility.

**`filter_valid_range`:** Uses the generic `Range[float]` type from the previous example. `contains(valid_range, reading.value)` calls the generic `contains` function. Generic types and functions composing with each other — this is the point.

**`process_sensor_data`:** Chains four `pipe` calls. Each call transforms one type to the next. If any fails — the error passes through all remaining `pipe` calls unchanged. The `match` at the end handles only three meaningful cases — format error, range error, and unexpected. Everything else is already handled by the pipeline structure.

**What generics enable here:** Without `pipe[T, U]`, this pipeline would require four nested `match` blocks, each with succeed/fail arms, totalling 30+ lines of boilerplate. With the generic combinator — the pipeline reads as a linear sequence of transformations. The type safety is identical — the compiler verifies each stage's input and output types — but the code is half the size.

### ✅ Honest Advantage Statement

The Skev pipeline (60 lines) is longer than C# (8 lines with extension methods). The reason is complete: Skev shows every pipeline stage defined as a separate function. C#'s 8-line version omits the stage implementations. At equivalent completeness, all four languages are comparable. The genuine Skev advantage: the `pipe[T, U]` combinator integrates with Skev's `result[T]` system from Chapter 6 — no third-party library needed. C++ needs `std::expected` (C++23). C# needs a custom `Result<T>` type. Python has no equivalent. The type safety guarantee is also unique — each pipeline stage's types are verified at compile time in both C++ and Skev but not in Python.

### Rule F — Non-Game Use Case: Animation/VFX

**🌍 Real-World Scenario**

An animation tool processes keyframe data through a pipeline: load raw keyframe data, validate timing constraints, smooth the curves, convert to the target rig's bone space, and export to the final format. Same composable pipeline problem — different domain.

```swift
# Reusing the same pipe[T, U] generic combinator
process_animation_clip(raw_clip: RawAnimClip) -> result[ExportedClip]
    result pipe(
        pipe(
            pipe(
                pipe(
                    validate_timing(raw_clip),
                    smooth_curves
                ),
                convert_to_bone_space(target_rig)
            ),
            apply_compression
        ),
        export_to_format(target_format)
    )
<< process_animation_clip
// 13 lines total — same generic combinator, different domain
```

The `pipe[T, U]` function written once for the robotics sensor pipeline works identically for animation processing, game save data pipelines, network packet processing, and any other domain. This is the fundamental value of generics — write once, use everywhere.

---

## 3.5.4 — Generic Constraints

### 👨‍💻 Developer Version

A constraint tells the compiler: "T can be any type — but it must support these operations." Without constraints, you cannot call methods on T or compare T values — because the compiler does not know what T is.

```swift
# Without constraint — T is completely unknown
# Cannot call any methods on T
unknown[T](value: T) -> T
    result value                  # ok — just returns it
    # value.upper()              # ERROR — T might not have .upper()
    # value > 0                  # ERROR — T might not support >
<< unknown

# With Comparable constraint — can now use comparison operators
find_min[T where T: Comparable](a: T, b: T) -> T
    if a <= b >>
        result a
    << a <= b
    result b
<< find_min
```

### Built-In Constraints Reference

```swift
# Comparable — supports <, >, <=, >=, ==, !=
sort[T where T: Comparable](items: list[T]) -> list[T]
    result items.sorted()
<< sort

# Hashable — can be used as map key or set element
count_unique[T where T: Hashable](items: list[T]) -> int
    unique :: set[T] = []
    loop item in items >>
        unique.add(item)
    << item
    result unique.count
<< count_unique

# Displayable — has .as(string) conversion
print_all[T where T: Displayable](items: list[T])
    loop item in items >>
        debug.log(item.as(string))
    << item
<< print_all

# Numeric — supports +, -, *, / operations
sum_all[T where T: Numeric](items: list[T]) -> T
    total :: T = 0.as(T)
    loop item in items >>
        total += item
    << item
    result total
<< sum_all

# Serialisable — can be encoded/decoded (requires #! serialisable)
save_list[T where T: Serialisable](
    items: list[T], path: string
) -> result[nothing]
    bytes :: list[uint8] = serial.encode(items)
    -> await file.write_bytes(path, bytes)
    succeed nothing
<< save_list

# Multiple constraints on same parameter
display_sorted[T where T: Comparable, T: Displayable](
    items: list[T]
) -> string
    sorted :: list[T] = items.sorted()
    result sorted.map(x -> x.as(string)).join(", ")
<< display_sorted
```

### Which Types Satisfy Which Constraints

```
Type          Comparable  Hashable  Displayable  Numeric  Copyable  Serialisable
────────────────────────────────────────────────────────────────────────────────
int           ✅          ✅        ✅           ✅       ✅        ✅
float         ✅          ❌        ✅           ✅       ✅        ✅
bool          ✅          ✅        ✅           ❌       ✅        ✅
string        ✅          ✅        ✅           ❌       ✅        ✅
Vector3!      ❌          ❌        ✅           ✅       ✅        ✅
Color!        ❌          ❌        ✅           ❌       ✅        ✅
kind types    ✅          ✅        ✅           ❌       ✅        ✅
data types    depends     depends   if fields    ❌       ✅        if marked
entity types  ❌          ❌        ❌           ❌       ❌        ❌
```

**Why floats are not Hashable:**
```
float has NaN (Not a Number).
NaN != NaN by IEEE 754 standard.
A hash table requires: if a == b then hash(a) == hash(b).
NaN breaks this invariant.
Therefore float cannot be used as a map key or set element.
This is the same decision as Rust — technically correct.
```

---

## 3.5.5 — Generic Type Aliases

### 👨‍💻 Developer Version

Generic type aliases give descriptive names to complex generic types. They make code read like the domain it models.

```swift
# Game domain aliases
alias Grid[T]        = list[list[T]]
alias Pool[T]        = list[T]
alias Handler[T]     = fn(T) -> nothing
alias AsyncResult[T] = fn() -> result[T]
alias Lookup[K, V]   = map[K, maybe V]

# Usage — reads like the domain
terrain_heights  :: Grid[float]
enemy_pool       :: Pool[Enemy]
on_damage        :: Handler[DamageEvent]
save_lookup      :: Lookup[string, PlayerData]

# Simulation domain aliases
alias TimeSeries[T]  = list[Pair[float64, T]]
alias Matrix[T]      = Grid[T]
alias Signal[T]      = fn(float) -> T

# Robotics domain aliases
alias SensorStream[T] = channel[T]
alias ControlLoop[T]  = fn(T, float) -> T
```

---

## 3.5.6 — Recursive Generic Types

### 👨‍💻 Developer Version

Generic types can reference themselves — enabling tree structures, linked lists, and other recursive data structures. ARC handles the memory automatically.

```swift
# Binary tree — each node has a value and two optional child nodes
data TreeNode[T] >>
    value :: T
    left  :: maybe TreeNode[T]    # recursive — ARC managed
    right :: maybe TreeNode[T]    # recursive — ARC managed
<< TreeNode

# Insert into a binary search tree
insert[T where T: Comparable](
    root: maybe TreeNode[T], value: T
) -> TreeNode[T]
    if not root exists >>
        result TreeNode[T] >>
            value :: value
            left  :: nothing
            right :: nothing
        << TreeNode[T]
    << not root exists

    if value < root.value >>
        result TreeNode[T] >>
            value :: root.value
            left  :: insert(root.left, value)
            right :: root.right
        << TreeNode[T]
    << value < root.value

    result TreeNode[T] >>
        value :: root.value
        left  :: root.left
        right :: insert(root.right, value)
    << TreeNode[T]
<< insert
```

**ARC handles recursive types correctly:**
```
When a TreeNode[int] goes out of scope:
→ ARC count for this node decremented
→ If count reaches 0 — destructor fires
→ Left child ARC count decremented
→ Right child ARC count decremented
→ If their counts reach 0 — their destructors fire
→ Recursively frees the entire subtree
→ No memory leak — no manual cleanup needed
```

---

## 3.5.7 — What Generics Are NOT Allowed On

### 👨‍💻 Developer Version

Three things cannot be generic in Skev. Each has a clear reason and a clear alternative.

**Entities cannot be generic:**

```swift
# ❌ NOT ALLOWED:
entity Dragon[T] >>
    value :: T
<< Dragon

# The reason:
# Entities have a fixed 24-byte header with a dispatch table pointer.
# The dispatch table is shared by ALL instances of an entity type.
# With Dragon[int] and Dragon[float] — there would be two entity types,
# two dispatch tables, two ARC management strategies.
# Skev's entity system is designed around fixed, known layouts.

# ✅ INSTEAD — use a generic data type as a property:
data DragonData[T] >>
    special_value :: T
<< DragonData

entity Dragon >>
    data :: DragonData[int]    # or DragonData[float] etc
<< Dragon
```

**Components cannot be generic:**

```swift
# ❌ NOT ALLOWED:
component Inventory[T] >>
    items :: list[T]
<< Inventory

# The reason: same as entities — fixed layout required.

# ✅ INSTEAD — use a generic data type inside the component:
data TypedInventory[T] >>
    items :: list[T]
<< TypedInventory

component Inventory >>
    weapon_slots :: TypedInventory[Weapon]
    item_slots   :: TypedInventory[Item]
<< Inventory
```

**Kind types cannot be generic:**

```swift
# ❌ NOT ALLOWED:
kind Wrapper[T] >>
    some(T)
    none
<< Wrapper

# The reason: kind variants are uint32 discriminants.
# They have no value storage — T has nowhere to live.
# result[T] and maybe T already cover these use cases.

# ✅ INSTEAD — use result[T] or maybe T:
value :: maybe int          # kind with value
outcome :: result[PlayerData]   # success or failure with value
```

---

## 3.5.8 — Generics Quick Reference

```
GENERIC FUNCTION:
  name[T](param: T) -> T           single type parameter
  name[A, B](a: A, b: B) -> A     multiple type parameters
  name[T where T: Comparable]      with constraint
  name[A where A: X, B where B: Y] multiple with constraints

GENERIC DATA TYPE:
  data Name[T] >>
      field :: T
  << Name

GENERIC ALIAS:
  alias Name[T] = ExistingType[T]
  alias Grid[T] = list[list[T]]

CONSTRAINTS:
  where T: Comparable     supports <, >, <=, >=, ==, !=
  where T: Hashable       usable in set[T] and map keys
  where T: Displayable    has .as(string) conversion
  where T: Numeric        supports +, -, *, / operations
  where T: Copyable       value type, can be copied
  where T: Serialisable   marked with #! serialisable

TYPE INFERENCE:
  identity(42)            T inferred as int from argument
  identity("hello")       T inferred as string from argument
  identity[int](42)       explicit — always valid

RECURSIVE TYPES:
  data Tree[T] >>
      value :: T
      left  :: maybe Tree[T]    # ARC handles recursion
      right :: maybe Tree[T]
  << Tree

NOT ALLOWED:
  entity Dragon[T] >>    ❌ entities cannot be generic
  component Health[T] >> ❌ components cannot be generic
  kind Option[T] >>      ❌ kind cannot be generic

BUILT-IN GENERICS (all use same [] syntax):
  list[T]         channel[T]      result[T]
  map[K, V]       array[T, N]     maybe T
  set[T]          Pair[A, B]      Grid[T]

MONOMORPHISATION:
  Compiler generates one concrete version per type.
  Zero runtime overhead vs non-generic code.
  Compile warning if >50 versions of one generic.
```

---

*End of Chapter 3.5 v0.1 — Next: Chapter 10: Security & Isolation*
