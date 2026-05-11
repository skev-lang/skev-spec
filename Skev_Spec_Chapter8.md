<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 8: Interoperability
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026)
**Status:** In Progress
**Depends On:** Chapter 4 (memory/unsafe), Chapter 5 (async/tasks), Chapter 6 (errors)
**Use Cases Tested:** All 7
**Rule H Verified:** skev.ffi, skev.scripting.lua, skev.ipc, skev.wasm — all checked ✅
**Rule I Applied:** Both directions considered for every mechanism ✅

---

## 8.1 — Overview

### 👨‍💻 Developer Version

No language exists in isolation. Every real game uses libraries from other languages — physics engines written in C++, audio middleware written in C, scripting systems in Lua, AI tools in Python. And every Skev library you write might need to be called from Unity (C#), Godot (GDScript), or a Python tool.

Skev handles all of this through a four-tier interoperability system:

```
Tier 1 — C ABI       Zero overhead.   C, C++, Rust, Swift, C# Native AOT
Tier 2 — Scripting   Minimal cost.    Lua, Python, JavaScript
Tier 3 — IPC         Milliseconds.    Java, C# (without Native AOT)
Tier 4 — WebAssembly Low overhead.    WASM plugins and browser games
```

**The most important rule in this chapter:**

> **Interop is always bidirectional.**
>
> For every mechanism: Skev can call the other language, AND the other language can call Skev. Both directions are designed, documented, and safe.

### ⚙️ Technical Version

Skev's interop model is built on the **C ABI** as the universal bridge language. All Tier 1 languages expose or consume a C-compatible binary interface — Skev does the same. Tier 2 languages embed their runtimes as C libraries — Skev calls their **C APIs**. Tier 3 uses OS-level IPC mechanisms. Tier 4 uses the WebAssembly binary instruction format with a defined host interface.

**C ABI vs C API — the distinction matters:**

```
C API (Application Programming Interface):
→ The SOURCE-LEVEL contract
→ Function names, parameter types, return types
→ What is written in a C header file (.h)
→ What Skev declares in an extern "C" block
→ Example: lua.h declares 200+ Lua C API functions

C ABI (Application Binary Interface):
→ The BINARY-LEVEL contract
→ Calling conventions — how arguments pass through registers
→ Stack layout and alignment rules
→ Symbol naming (no C++ name mangling)
→ What makes a .dll/.so from one compiler work
  with code compiled by a completely different compiler

The C ABI is WHY interop works.
The C API is WHAT you are calling.

When Rust writes #[no_mangle] pub extern "C" fn,
when Swift writes @_cdecl,
when C++ writes extern "C" —
they are all saying: "follow the C ABI for this function."

That shared binary contract is what lets Skev call Rust,
Rust call Swift, Swift call C, and every Tier 1 language
call every other. The C ABI is the universal handshake.
```

The `unsafe` block is Skev's quarantine mechanism — all foreign function calls must happen inside `unsafe` boundaries, making the dangerous zone visible, auditable, and compiler-enforced. Outside `unsafe`, all of Skev's safety guarantees apply.

---

## 8.2 — Tier 1: C ABI — The Universal Bridge

### 👨‍💻 Developer Version

C is the language every other language knows how to talk to. Rust exports C-compatible functions. Swift exports C-compatible functions. Even Java can expose a C interface via JNI.

Two terms appear constantly in this chapter. Here is the difference:

```
C API — the WHAT:
  The specific functions a library exposes.
  Names, parameters, return types.
  What you declare in Skev's extern block.
  "Call this function with these arguments."

C ABI — the WHY:
  The binary rules that make it all work.
  Which CPU register holds argument 1, argument 2...
  How the call stack is laid out.
  Why a Rust-compiled .dll works with a
  C++-compiled caller without any glue code.
  "Both of us agreed to follow these binary conventions."
```

The **C ABI** is the handshake that makes all Tier 1 languages interoperable. When you use `extern "C"` in any language — you are saying "I follow the C ABI for this function." That shared contract is why Skev can call Rust, Swift, or C++ and they can call back — with zero overhead, zero marshaling, zero wrappers at the binary level.

The **C API** is what you actually declare in Skev's `extern "C"` block — the names and types of the functions you want to call.

Skev both speaks and understands the C ABI natively.

---

### 8.2.1 — `extern "C"` — Skev Calling C/C++/Rust/Swift

**Skev → C: Declaring the interface**

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game uses PhysX for rigid body simulation, FMOD for audio, and a custom Rust pathfinder for AI navigation. All three are compiled C/C++/Rust libraries. Skev must call all of them from game entities.

**💥 Why This Is Hard**

Calling into foreign code means leaving the safety of your type system. You must match function signatures exactly — wrong types mean crashes, not compile errors. You must manage memory at the boundary — who allocates, who frees. You must handle the case where the foreign code returns null, negative error codes, or writes into output parameters.

---

**In C++23:**
```cpp
// C++ has no FFI concept — it IS the native language
// But calling a Rust library requires extern "C" declarations
extern "C" {
    int rust_find_path(float startX, float startY,
                       float endX, float endY,
                       float* outputX, float* outputY,
                       int maxWaypoints);
}

// Call it
float wx[100], wy[100];
int count = rust_find_path(0.0f, 0.0f, 10.0f, 10.0f,
                           wx, wy, 100);
if (count < 0) { /* error */ }
// No type safety on output params — easy to get buffer size wrong
// Return value error codes must be documented and checked manually
// 9 lines for one call with no safety
// // 9 lines total
```

**In C# 13:**
```csharp
// C# uses P/Invoke for C libraries
[DllImport("rust_pathfinder.dll", CallingConvention = CallingConvention.Cdecl)]
static extern int RustFindPath(
    float startX, float startY,
    float endX, float endY,
    [Out] float[] outputX, [Out] float[] outputY,
    int maxWaypoints);

// Call it
var wx = new float[100];
var wy = new float[100];
int count = RustFindPath(0f, 0f, 10f, 10f, wx, wy, 100);
if (count < 0) throw new Exception("Pathfinding failed");
// DllImport attribute is verbose and string-based — typos not caught at compile time
// No consistent error handling — some libraries throw, some return codes
// 11 lines total
```

**In Python 3.13:**
```python
import ctypes

lib = ctypes.CDLL("./librust_pathfinder.so")
lib.rust_find_path.argtypes = [
    ctypes.c_float, ctypes.c_float,
    ctypes.c_float, ctypes.c_float,
    ctypes.POINTER(ctypes.c_float),
    ctypes.POINTER(ctypes.c_float),
    ctypes.c_int
]
lib.rust_find_path.restype = ctypes.c_int

wx = (ctypes.c_float * 100)()
wy = (ctypes.c_float * 100)()
count = lib.rust_find_path(0, 0, 10, 10, wx, wy, 100)
# Type annotations are runtime — wrong type = crash, not compile error
# 14 lines of setup before any actual call
# 14 lines total
```

**In Skev — declaring the interface:**
```swift
# Declare the interface once — type-checked at compile time
extern "C" RustPathfinder >>
    rust_find_path(
        start_x:       float32,
        start_y:       float32,
        end_x:         float32,
        end_y:         float32,
        output_x:      *float32,
        output_y:      *float32,
        max_waypoints: int
    ) -> int    # returns count, negative = error
<< RustPathfinder
// 12 lines total — interface declaration (written once)
```

**In Skev — calling it through a safe wrapper:**
```swift
entity PathfindingSystem >>

    find_path(start: Vector3!, end: Vector3!) -> maybe list[Vector3!]

        output_x :: array[float32, 100]
        output_y :: array[float32, 100]
        count    :: int = 0

        unsafe >>
            count = RustPathfinder.rust_find_path(
                start.x, start.y,
                end.x, end.y,
                ffi.ptr(output_x),
                ffi.ptr(output_y),
                100
            )
        << unsafe

        if count <= 0 >>
            result nothing
        << count <= 0

        waypoints :: list[Vector3!]
        loop i from 0 to count - 1 >>
            waypoints.add(Vector3!(output_x[i], 0.0, output_y[i]))
        << i

        result waypoints

    << find_path

<< PathfindingSystem
// 28 lines total — safe wrapper (written once, used everywhere)
```

### 🧭 Walk-Through — extern "C"

**The `extern "C"` block:** Declares the interface. Type-checked at compile time — if you write `float32` where the C function expects `double`, it is a compile error, not a runtime crash. Written once per library.

**The `unsafe` block:** The quarantine zone. Inside here, the C function is called directly. ARC does not apply. Null is not checked. Buffer bounds are the developer's responsibility. This is visible — you cannot accidentally call C code outside unsafe.

**The wrapper entity:** Hides all unsafe code from the rest of the game. Game entities call `pathfinding.find_path()` — they never see `unsafe`, never see `extern`, never deal with raw pointers. This is the wrapper pattern every Skev FFI should follow.

### ✅ Honest Advantage Statement

Skev's extern block provides compile-time type checking that C# P/Invoke (string-based, runtime) and Python ctypes (runtime annotations) do not. C++ has no FFI concept — it is the native language, but calling Rust requires the same `extern "C"` pattern. Where Skev adds unique value: the `unsafe` block makes the dangerous zone structurally visible and bounded, and the wrapper pattern is idiomatic rather than optional. Skev is 12 lines for the interface declaration vs 14 lines in Python with runtime-only checking.

---

### 8.2.2 — `export "C"` — Other Languages Calling Skev

**Other languages → Skev: Exposing Skev as a library**

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A Unity C# game wants to use Skev's AI decision system as a plugin. A Python data science tool wants to call Skev's simulation engine. A Godot GDScript game wants Skev's pathfinding. All three need to call into compiled Skev code from their own language.

**In C++23:**
```cpp
// C++ exports are natural — just mark with extern "C"
extern "C" {
    int cpp_ai_decide(int agentId, float x, float y, float z) {
        return g_aiSystem->Decide(agentId, {x, y, z});
    }
}
// Simple — C++ has no extra overhead here
// But no integration with any safety system
// Global state (g_aiSystem) must be managed manually
// 6 lines total
```

**In C# 13:**
```csharp
// C# can export via Native AOT
// Without Native AOT — C# cannot be called from C
[UnmanagedCallersOnly(EntryPoint = "csharp_ai_decide")]
public static int AiDecide(int agentId, float x, float y, float z) {
    return AiSystem.Instance.Decide(agentId, new Vector3(x, y, z));
}
// Requires .NET Native AOT compilation — not standard Unity workflow
// Attribute-based — easy to forget [UnmanagedCallersOnly]
// 5 lines total (requires build configuration)
```

**In Python 3.13:**
```python
# Python cannot export native C functions at all
# Must use Cython, ctypes.CFUNCTYPE, or CFFI to create callable
# from ctypes import CFUNCTYPE, c_int, c_float
# Extremely verbose and rarely done
# Python is not designed to be called from C — only to call C
# Not practical for game use — Python as native library is rare
```

**In Skev:**
```swift
# Simple export — works with C, C#, Python ctypes,
# Rust FFI, GDScript, Java JNA — any language that can call C
export "C" skev_ai_decide(
    agent_id: int,
    x: float32, y: float32, z: float32
) -> int
    agent :: AIAgent = ai_system.get(agent_id)
    if not agent exists >>
        result -1    # error code
    << not agent exists
    action :: AIAction = agent.decide(Vector3!(x, y, z))
    result action.id
<< skev_ai_decide
// 10 lines total
```

**Calling from C# (Unity):**
```csharp
[DllImport("skev_ai.dll")]
static extern int skev_ai_decide(
    int agentId, float x, float y, float z);

int actionId = skev_ai_decide(agent.id, pos.x, pos.y, pos.z);
// 4 lines total — Unity side
```

**Calling from Python:**
```python
import ctypes
skev = ctypes.CDLL("./libskev_ai.so")
skev.skev_ai_decide.restype = ctypes.c_int
action_id = skev.skev_ai_decide(agent_id, x, y, z)
// 3 lines total — Python side
```

**Calling from GDScript (Godot):**
```gdscript
var lib = NativeLibrary.new()
lib.load("skev_ai.dll")
var action_id = lib.skev_ai_decide(agent_id, x, y, z)
# 3 lines total — GDScript side
```

### ✅ Honest Advantage Statement

`export "C"` makes Skev code callable from any language that can call C — which is essentially every major language. Python cannot do this natively (must use Cython or CFFI). C# requires Native AOT compilation (not standard Unity workflow). C++ can do this natively — similar to Skev. Skev's advantage: the exported function lives alongside the entity system, can use all Skev types internally, and the C-compatible interface is automatically generated at the boundary.

---

### 8.2.3 — C Type Mapping Reference

```swift
extern "C" ExampleLib >>
    # Primitives — exact mapping
    int_fn(a: int) -> int                    # int32_t → int32_t
    float_fn(a: float32) -> float32          # float → float
    double_fn(a: float64) -> float64         # double → double
    bool_fn(a: bool) -> bool                 # uint8_t

    # Strings — automatic marshaling
    string_fn(name: string) -> string        # const char* (auto null-term)

    # Pointers
    nullable_fn() -> *CStruct                # may be null
    nonnull_fn(s: &CStruct) -> nothing       # guaranteed non-null

    # Arrays
    array_fn(data: *float32, count: int) -> nothing   # T* + length

    # Return nothing
    void_fn() -> nothing                     # void return
<< ExampleLib
// 15 lines total
```

---

### 8.2.4 — `cdata` — C-Compatible Structs

### Rule F

**🌍 Real-World Scenario**

PhysX uses a `PxTransform` struct with specific memory layout. Skev must pass this struct to PhysX functions — the memory layout must match exactly what PhysX expects, or the physics simulation produces wrong results or crashes.

**In C++23:**
```cpp
// C++ struct layout matches C by default
struct PxTransform {
    PxQuat q;   // rotation (x, y, z, w floats)
    PxVec3 p;   // position (x, y, z floats)
};
// Natural in C++ — but any C++ specific feature
// (virtual functions, non-trivial constructors) breaks C layout
// 4 lines total
```

**In C# 13:**
```csharp
// C# requires [StructLayout] for guaranteed C-compatible layout
[StructLayout(LayoutKind.Sequential)]
struct PxTransform {
    public PxQuat Q;    // rotation
    public PxVec3 P;    // position
}
// Attribute required — easy to forget — silent wrong layout if missing
// 5 lines total
```

**In Python 3.13:**
```python
import ctypes

class PxTransform(ctypes.Structure):
    _fields_ = [
        ("q", PxQuat),    # rotation
        ("p", PxVec3),    # position
    ]
# Verbose ctypes.Structure inheritance
# 4 lines total
```

**In Skev:**
```swift
cdata PxQuat >>
    x :: float32
    y :: float32
    z :: float32
    w :: float32
<< PxQuat

cdata PxVec3 >>
    x :: float32
    y :: float32
    z :: float32
<< PxVec3

cdata PxTransform >>
    q :: PxQuat    # rotation
    p :: PxVec3    # position
<< PxTransform
// 14 lines total — but matches C layout automatically
```

**`cdata` rules:**
```
Fields: exact declaration order — no compiler reordering
Padding: C-standard alignment padding added automatically
Layout: identical to equivalent C struct
Safety: can be passed directly to/from C functions
vs data: data = optimised packing, cdata = C-exact layout
```

### ✅ Honest Advantage Statement

Skev's `cdata` is equivalent to C# `[StructLayout(Sequential)]` — both guarantee C-compatible layout. C++ structs match C layout by default but lose it when C++ features are added. Python ctypes.Structure is most verbose. Skev's `cdata` uses familiar `>> <<` syntax and makes the C-layout intent explicit through the keyword name rather than an attribute.

---

## 8.3 — Callbacks: C Calling Back Into Skev

### 👨‍💻 Developer Version

Sometimes the direction reverses mid-call. You call a C physics library, and the library calls back into your code when a collision occurs. This is called a callback — and it creates a thread safety challenge because the callback fires on the C library's thread, not on Skev's main thread.

Skev handles this automatically.

### Rule F

**🌍 Real-World Scenario**

PhysX fires collision callbacks from its simulation thread. The callback must reach Skev entity code — which runs on the main thread. Getting this wrong produces data races or crashes.

**In C++23:**
```cpp
// C++ developer manually marshals to game thread
struct CollisionCallback : public PxSimulationEventCallback {
    void onContact(const PxContactPairHeader& header,
                   const PxContactPair* pairs,
                   PxU32 nbPairs) override {
        // This fires on PhysX thread — must post to game thread
        GameThread::Post([=]() {
            OnCollision(header.actors[0], header.actors[1]);
        });
    }
};
// Manual thread marshaling — easy to forget — silent bugs
// 9 lines total
```

**In C# 13:**
```csharp
// C# Unity handles this via job system or main thread dispatcher
physicsWorld.OnCollisionEnter += (a, b) => {
    UnityMainThreadDispatcher.Instance().Enqueue(() => {
        HandleCollision(a, b);
    });
};
// Unity-specific dispatcher — verbose boilerplate
// 5 lines total
```

**In Python 3.13:**
```python
# Python callbacks with ctypes
def collision_callback(body_a_id, body_b_id):
    # GIL acquired — but on wrong thread for game logic
    # Must queue to main thread manually
    game_thread_queue.put((body_a_id, body_b_id))

CALLBACK_TYPE = ctypes.CFUNCTYPE(None, ctypes.c_int, ctypes.c_int)
callback = CALLBACK_TYPE(collision_callback)
# 5 lines total — but thread marshaling is manual
```

**In Skev:**
```swift
# Skev automatically marshals to main thread
callback on_collision(body_a: int, body_b: int) -> nothing
    # This code always runs on main thread — safe to access entities
    entity_a :: maybe Entity = physics.get_entity(body_a)
    entity_b :: maybe Entity = physics.get_entity(body_b)

    if entity_a exists and entity_b exists >>
        entity_a.on_physics_collision(entity_b)
        entity_b.on_physics_collision(entity_a)
    << entity_a exists and entity_b exists
<< on_collision

# Register with physics library
unsafe >>
    PhysXLib.set_collision_callback(
        ffi.as_ptr(on_collision)
    )
<< unsafe
// 16 lines total — automatic thread safety
```

### ✅ Honest Advantage Statement

All three comparison languages require manual thread marshaling for C callbacks. Skev's `callback` type automatically queues the callback to the main thread — the developer writes normal Skev entity code without thinking about threading. This eliminates an entire category of data race bugs that affect C++, C#, and Python game code.

---

## 8.4 — Tier 2: Scripting Languages

### 👨‍💻 Developer Version

Scripting languages (Lua, Python, JavaScript) cannot be called directly via the C ABI — they need their own runtime embedded in Skev. Skev provides built-in bridge libraries for each.

**The mental model:**
```
Skev is the HOST.
The scripting language is the GUEST.

Skev creates the guest's runtime.
Skev registers functions the guest can call.
Skev loads and executes guest scripts.
The guest calls Skev's registered functions.
```

---

### 8.4.1 — Lua Bridge (`skev.scripting.lua`)

Lua is the most important scripting language for games. Roblox, World of Warcraft, LÖVE, and thousands of indie games use Lua for modding and game logic.

### Rule F

**🌍 Real-World Scenario**

An open-world RPG supports player mods. Mods are Lua scripts that can spawn entities, modify stats, add quests, and respond to game events. Skev must host Lua, expose safe game APIs, and execute mod scripts without allowing mods to crash or corrupt the game.

**In C++23:**
```cpp
extern "C" {
    #include "lua.h"
    #include "lualib.h"
    #include "lauxlib.h"
}

// Register a C++ function that Lua can call
int lua_spawn_entity(lua_State* L) {
    const char* name = luaL_checkstring(L, 1);
    float x = luaL_checknumber(L, 2);
    float y = luaL_checknumber(L, 3);
    g_scene->SpawnByName(name, {x, y, 0.0f});
    return 0;  // number of return values
}

lua_State* L = luaL_newstate();
luaL_openlibs(L);
lua_register(L, "spawn_entity", lua_spawn_entity);
luaL_dofile(L, "mods/combat.lua");
// Manual Lua C API — verbose, error-prone, no type safety
// Must match Lua stack conventions exactly
// 15 lines total (just for one function registration)
```

**In C# 13:**
```csharp
// C# uses NLua or MoonSharp (third-party libraries)
var lua = new NLua.Lua();
lua.RegisterFunction("spawn_entity", this,
    typeof(GameBridge).GetMethod("SpawnEntity"));

// SpawnEntity must have specific signature
public void SpawnEntity(string name, float x, float y) {
    scene.SpawnByName(name, new Vector3(x, y, 0));
}
lua.DoFile("mods/combat.lua");
// Requires third-party library
// Reflection-based function registration — runtime errors
// 8 lines total
```

**In Python 3.13:**
```python
# Python would use lupa (LuaJIT binding) — third-party
import lupa
lua = lupa.LuaRuntime()
lua.globals().spawn_entity = lambda name, x, y: (
    scene.spawn_by_name(name, Vector3(x, y, 0))
)
lua.execute(open("mods/combat.lua").read())
# Requires third-party lupa library
# Lambda syntax limits complex function bodies
# 5 lines total
```

**In Skev:**
```swift
import skev.scripting.lua

entity ModSystem >>

    lua :: LuaState

    when scene_load
        lua = lua.create()

        # Register Skev functions — Lua can call these
        lua.register("spawn_entity") >> args
            name :: string   = args.string(0)
            x    :: float    = args.float(1)
            y    :: float    = args.float(2)
            scene.spawn_by_name(name, Vector3!(x, 0.0, y))
        << register

        lua.register("get_player_health") >> args
            result args.return_int(player.health)
        << register

        lua.register("set_time_of_day") >> args
            hour :: float = args.float(0)
            world.set_time(hour)
        << register

        # Load mod scripts
        load_mods()
    << scene_load

    async load_mods() -> result[nothing]
        mod_files :: list[string] = file.list("mods/")
        loop mod_file in mod_files >>
            if mod_file.ends_with(".lua") >>
                script :: string = -> await file.read_text(mod_file)
                match lua.execute(script) >>
                    succeed _ ->
                        debug.log("Loaded mod: {mod_file}")
                    fail error ->
                        debug.log("Mod error in {mod_file}: {error.message}")
                << lua.execute(script)
            << mod_file.ends_with(".lua")
        << mod_file
        succeed nothing
    << load_mods

    # Allow mods to trigger game events
    when lua_event(event: string, data: string)
        lua.call("on_game_event", event, data)
    << lua_event

<< ModSystem
// 44 lines total — zero third-party libraries
```

### 🧭 Walk-Through — Lua Bridge

**`lua.register("name") >> args ... << register`:** Registers a Skev function that Lua scripts can call. The `args` object extracts typed values from Lua's argument stack — `args.string(0)` gets the first argument as a Skev string, `args.float(1)` gets the second as a float. Type mismatches produce clear Lua errors, not crashes.

**`lua.execute(script)`:** Runs a Lua script string inside the sandboxed Lua runtime. Returns `result[T]` — syntax errors, runtime errors, and missing function calls all surface as proper Skev errors through the Chapter 6 error system.

**`lua.call("function", args...)`:** Calls a function defined in the Lua script from Skev. This is the other direction — Skev driving the script. Used for event callbacks: "the game just started — tell all mods."

**Error handling:** Every Lua execution error is a `result[T]` failure — the mod's error does not crash the game. The `match` block handles failure gracefully: log the error, continue loading other mods.

### ✅ Honest Advantage Statement

Skev's Lua bridge requires zero third-party libraries. C++ needs the Lua C API directly (verbose, error-prone). C# needs NLua or MoonSharp (third-party, reflection-based). Python needs lupa (third-party). Skev's `lua.register()` syntax with the `>> args` block is the cleanest registration pattern of all four — the body is normal Skev code, not a specially-typed callback function.

---

### 8.4.2 — Python Bridge (`skev.scripting.python`)

### Rule F

**🌍 Real-World Scenario**

A simulation game uses a Python AI model to make high-level strategic decisions. The simulation runs in Skev. Every 10 seconds, Skev sends the current world state to Python, runs the AI, and applies the resulting strategy.

**In C++23:**
```cpp
// Embedding Python in C++ requires the CPython C API
#include <Python.h>

Py_Initialize();
PyObject* pModule = PyImport_ImportModule("ai_strategy");
PyObject* pFunc = PyObject_GetAttrString(pModule, "decide");
PyObject* pArgs = PyTuple_New(1);
PyTuple_SetItem(pArgs, 0, PyUnicode_FromString(world_state_json));
PyObject* pResult = PyObject_CallObject(pFunc, pArgs);
const char* result = PyUnicode_AsUTF8(pResult);
// Extremely verbose — reference counting manual — 10 lines per call
// 10 lines total (per call — setup is additional 20 lines)
```

**In C# 13:**
```csharp
// C# uses IronPython or Python.NET (third-party)
using Python.Runtime;
PythonEngine.Initialize();
using (Py.GIL()) {
    dynamic ai = Py.Import("ai_strategy");
    string result = ai.decide(worldStateJson);
}
// Requires Python.NET third-party library
// GIL must be managed manually — easy to deadlock
// 6 lines total
```

**In Python 3.13:**
```python
# Python calling itself — trivially simple
import ai_strategy
result = ai_strategy.decide(world_state_json)
# 2 lines total — but this is calling Python FROM Python
# Cross-language aspect doesn't apply here
```

**In Skev:**
```swift
import skev.scripting.python

entity AIStrategyBridge >>

    python :: PythonState

    when scene_load
        python = python.create()
        python.add_path("ai/")    # where Python scripts live
    << scene_load

    async get_ai_decision(world_state: WorldState) -> result[AIDecision]
        # Serialise world state to pass to Python
        state_json :: string = json.serialise(world_state)
        python.set("world_state", state_json)

        # Run Python AI model
        script :: string = -> await file.read_text("ai/strategy.py")
        -> python.execute(script)

        # Get result back as JSON
        result_json :: string = -> python.get_string("decision")
        decision :: AIDecision = -> json.parse(result_json)
        succeed decision
    << get_ai_decision

    # Called every 10 seconds by timer
    when ai_timer
        state :: WorldState = world.capture_state()
        match await get_ai_decision(state) >>
            succeed decision ->
                world.apply_strategy(decision)
            fail error ->
                debug.log("AI decision failed: {error.message}")
                # Use fallback strategy
                world.apply_default_strategy()
        << await get_ai_decision(state)
    << ai_timer

<< AIStrategyBridge
// 34 lines total
```

**Python GIL constraint — honesty required:**
```
Python's Global Interpreter Lock (GIL) means only one thread
can execute Python bytecode at a time.

In Skev:
→ Only ONE task can run Python simultaneously
→ If two tasks call python.execute() at the same time:
  → Second task waits for first to complete
→ Compiler warns if Python is called from concurrent tasks

This is a genuine limitation.
Python is not suitable for per-frame parallel computation.
Use Python for: AI models, data analysis, tool integrations.
Do NOT use Python for: high-frequency game logic, parallel AI.
Use tasks with Skev code or Lua for those cases.
```

### ✅ Honest Advantage Statement

Skev's Python bridge is significantly cleaner than the CPython C API (10+ lines per call in C++) and comparable to Python.NET (6 lines). The GIL limitation is honestly documented — Python is appropriate for background AI and tools, not per-frame game logic. Skev's bridge integrates Python errors into the `result[T]` system from Chapter 6 — Python exceptions surface as Skev failures.

---

### 8.4.3 — JavaScript Bridge (`skev.scripting.javascript`)

For browser games and Electron-based game tools:

```swift
import skev.scripting.javascript

entity UIScriptBridge >>

    js :: JSState

    when scene_load
        js = javascript.create()    # uses QuickJS — embedded, fast

        js.register("update_hud") >> args
            health :: int = args.int(0)
            score  :: int = args.int(1)
            ui.update_hud(health, score)
        << register
    << scene_load

    async run_ui_script() -> result[nothing]
        script :: string = -> await file.read_text("ui/hud.js")
        -> js.execute(script)
        succeed nothing
    << run_ui_script

<< UIScriptBridge
// 18 lines total
```

---

## 8.5 — Tier 3: IPC-Based Interoperability

### 👨‍💻 Developer Version

Some languages cannot be embedded — Java and non-Native-AOT C# run on managed runtimes that cannot be called from C. For these, Skev uses inter-process communication: spawn the other language as a separate process and communicate via pipes or sockets.

**Important: Tier 3 has millisecond latency. Not for per-frame use.**

### Rule F

**🌍 Real-World Scenario**

A game studio has a Java tool that processes level data and generates optimised nav meshes. The Skev game engine needs to call this tool as part of the build pipeline — not per-frame, but when a level is first loaded.

**In C++23:**
```cpp
// C++ spawns a process and reads its output
#include <cstdio>

FILE* pipe = popen("java -jar navmesh_tool.jar level_data.json", "r");
char buffer[128];
std::string result;
while (fgets(buffer, sizeof(buffer), pipe) != NULL) {
    result += buffer;
}
pclose(pipe);
// Simple but: no async — blocks until Java finishes
// No error handling for Java crashes
// No bidirectional communication
// 8 lines total
```

**In C# 13:**
```csharp
var process = new Process {
    StartInfo = new ProcessStartInfo {
        FileName = "java",
        Arguments = "-jar navmesh_tool.jar",
        UseShellExecute = false,
        RedirectStandardInput = true,
        RedirectStandardOutput = true
    }
};
process.Start();
await process.StandardInput.WriteLineAsync(levelDataJson);
string result = await process.StandardOutput.ReadLineAsync();
await process.WaitForExitAsync();
// Verbose ProcessStartInfo setup — 10 lines of boilerplate
// 11 lines total
```

**In Python 3.13:**
```python
import asyncio

async def call_java_tool(level_data: str) -> str:
    proc = await asyncio.create_subprocess_exec(
        "java", "-jar", "navmesh_tool.jar",
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE
    )
    stdout, _ = await proc.communicate(level_data.encode())
    return stdout.decode()
# Clean Python async — 7 lines total
# 7 lines total
```

**In Skev:**
```swift
import skev.ipc

entity NavMeshBuilder >>

    async build_navmesh(level_data: string) -> result[NavMesh]
        proc :: Process = -> await ipc.spawn(
            command: "java",
            args: ["-jar", "navmesh_tool.jar"],
            stdin:  ipc.pipe,
            stdout: ipc.pipe,
            stderr: ipc.pipe
        )

        -> await proc.write_line(level_data)
        result_json :: string = -> await proc.read_line()
        exit_code   :: int    = -> await proc.wait()

        if exit_code != 0 >>
            error_msg :: string = -> await proc.read_error()
            fail NavMeshError.tool_failed with
                exit_code: exit_code,
                message:   error_msg
        << exit_code != 0

        navmesh :: NavMesh = -> json.parse(result_json)
        succeed navmesh
    << build_navmesh

<< NavMeshBuilder
// 21 lines total
```

### ✅ Honest Advantage Statement

Skev's IPC implementation is comparable to Python's asyncio approach in clarity (21 vs 7 lines) but adds structured error handling via `result[T]` and explicit error reading. C++ blocking `popen` is simplest but synchronous. C# ProcessStartInfo is most verbose. The honest trade-off: Skev uses more lines than Python here — the extra lines are explicit error handling that Python's version omits. Use Tier 3 only for background operations: build pipelines, data processing, tool integration.

---

## 8.6 — Tier 4: WebAssembly

### 👨‍💻 Developer Version

WebAssembly lets code compiled from any language (Rust, C, C++, Go) run inside Skev as a sandboxed plugin. Skev hosts the WASM module, provides it with specific capabilities, and the module cannot access anything outside what Skev explicitly grants.

This is the foundation of a **safe mod system** — mods run in WASM, cannot crash the game, cannot access the file system, cannot reach the network unless Skev allows it.

### Rule F

**🌍 Real-World Scenario**

A game has a mod marketplace. Mods are submitted by strangers — they cannot be trusted. They must run inside the game but cannot be allowed to crash it, read save files, or access the internet. WASM provides this sandbox automatically.

**In C++23:**
```cpp
// C++ has no built-in WASM hosting
// Must use wasmtime, wasmer, or WAVM (third-party)
#include <wasmtime.h>

// 100+ lines of WASM engine setup
// Manual host function registration
// Manual memory management for WASM linear memory
// No standard approach — library choice varies
// ~100 lines total just for setup
```

**In C# 13:**
```csharp
// C# uses Wasmtime.NET (third-party)
using var engine = new Engine();
using var store = new Store(engine);
using var module = Module.FromFile(engine, "mod.wasm");

// Register host functions
var linker = new Linker(engine);
linker.Define("env", "log_message",
    Function.FromCallback(store,
        (Caller caller, string msg) => Console.WriteLine(msg)));
// Third-party library, verbose setup
// 8 lines total just for module loading
```

**In Python 3.13:**
```python
# Python uses wasmtime-py (third-party)
from wasmtime import Store, Module, Linker, Engine

engine = Engine()
store = Store(engine)
module = Module.from_file(engine, "mod.wasm")
linker = Linker(engine)

@linker.define("env", "log_message")
def log_message(msg):
    print(f"[Mod] {msg}")
# Third-party — 8 lines total
```

**In Skev:**
```swift
import skev.wasm

entity ModRunner >>

    loaded_mods :: map[string, WasmModule]

    async load_mod(path: string, mod_id: string) -> result[nothing]
        wasm_bytes :: list[uint8] = -> await file.read_bytes(path)
        mod :: WasmModule = -> wasm.load(wasm_bytes)

        # Grant ONLY the capabilities this mod needs
        # Everything else is denied by default (sandbox)
        mod.import("env", "log_message") >> args: WasmArgs
            msg :: string = args.string(0)
            debug.log("[Mod:{mod_id}] {msg}")
        << import

        mod.import("env", "spawn_item") >> args: WasmArgs
            item_id :: int   = args.int32(0)
            x       :: float = args.float32(1)
            z       :: float = args.float32(2)
            # Validate — mod cannot spawn boss entities
            if item_id < 1000 >>
                scene.spawn_item(item_id, Vector3!(x, 0.0, z))
            << item_id < 1000
        << import

        # No file system access
        # No network access
        # No entity access outside spawn_item
        # Mod CANNOT do anything Skev did not explicitly allow

        loaded_mods.add(mod_id -> mod)
        succeed nothing
    << load_mod

    call_mod_event(mod_id: string, event: string, data: float)
        if loaded_mods contains mod_id >>
            mod :: WasmModule = loaded_mods[mod_id]
            mod.call(event, [data])
        << loaded_mods contains mod_id
    << call_mod_event

<< ModRunner
// 40 lines total — zero third-party libraries
```

### ✅ Honest Advantage Statement

Skev's WASM hosting requires zero third-party libraries. C++, C#, and Python all require third-party WASM runtimes. Skev's `mod.import()` registration with the `>> args` block is the cleanest registration syntax of all four comparisons. The security model is structurally enforced — mods can only call functions Skev explicitly registers. File system access, network access, and entity manipulation are denied by default. No other pattern in the comparison languages provides this structural safety without additional sandboxing frameworks.

---

## 8.7 — The Wrapper Pattern — Safe Bridge Architecture

### 👨‍💻 Developer Version

Every interop scenario should follow the same architecture. This is the most important pattern in Chapter 8:

```
Layer 1: Raw interface (extern/export/bridge) — unsafe or complex
          ↓
Layer 2: Skev wrapper entity — safe interface
          ↓
Layer 3: Game code — never sees unsafe
```

### Rule F

**🌍 Real-World Scenario**

A game needs to integrate Steam for achievements, leaderboards, and multiplayer matchmaking. The Steamworks SDK is a C++ library. Hundreds of game entities need to report achievements and query leaderboard data. None of them should ever see unsafe code or raw C pointers.

**In C++23:**
```cpp
// C++ — no wrapper pattern enforced — unsafe can spread everywhere
#include "steam_api.h"

// Any function anywhere can call Steam directly
// No guaranteed safety boundary
bool SubmitScore(int score) {
    if (!SteamUserStats()) return false;
    SteamUserStats()->SetStat("HighScore", score);
    SteamUserStats()->StoreStats();
    return true;
}
// Steamworks code scattered across codebase — hard to maintain
// ~8 lines total but multiplied across every file that needs Steam
```

**In C# 13:**
```csharp
// C# Unity uses Steamworks.NET (third-party wrapper)
// The wrapper exists but is third-party — not enforced by language
using Steamworks;

public class SteamManager : MonoBehaviour {
    public void SubmitScore(int score) {
        if (!SteamManager.Initialized) return;
        SteamUserStats.SetStat("HighScore", score);
        SteamUserStats.StoreStats();
    }
}
// Third-party wrapper already exists for Unity
// But pattern not enforced — any script can call Steamworks directly
// 8 lines total
```

**In Python 3.13:**
```python
# Python uses steamio or steam (third-party)
import steam

class SteamBridge:
    def submit_score(self, score: int) -> bool:
        try:
            client = steam.client.SteamClient()
            client.user_stats.set_stat("HighScore", score)
            return True
        except Exception:
            return False
# Third-party — 7 lines total
```

**In Skev — full wrapper pattern:**
```swift
# Layer 1 — Raw C interface (internal — never used directly by game)
extern "C" SteamworksLib >>
    SteamAPI_Init() -> bool
    SteamAPI_Shutdown() -> nothing
    SteamAPI_RunCallbacks() -> nothing
    SteamUserStats_SetStatInt(name: string, value: int) -> bool
    SteamUserStats_GetStatInt(name: string, out: *int) -> bool
    SteamUserStats_StoreStats() -> bool
    SteamUserStats_SetAchievement(name: string) -> bool
    SteamFriends_GetPersonaName() -> string
<< SteamworksLib

# Layer 2 — Safe Skev wrapper (this is the public interface)
entity SteamManager >>

    initialised :: bool = false
    player_name :: string = "Unknown"

    when scene_load
        start_init()
    << scene_load

    async start_init() -> result[nothing]
        success :: bool = false
        unsafe >>
            success = SteamworksLib.SteamAPI_Init()
        << unsafe

        if not success >>
            fail SteamError.init_failed
        << not success

        initialised = true

        unsafe >>
            player_name = SteamworksLib.SteamFriends_GetPersonaName()
        << unsafe

        debug.log("Steam initialised for: {player_name}")
        succeed nothing
    << start_init

    submit_score(stat_name: string, value: int) -> result[nothing]
        if not initialised >>
            fail SteamError.not_initialised
        << not initialised

        success :: bool = false
        unsafe >>
            success = SteamworksLib.SteamUserStats_SetStatInt(stat_name, value)
            if success >>
                SteamworksLib.SteamUserStats_StoreStats()
            << success
        << unsafe

        if not success >>
            fail SteamError.stat_update_failed with stat: stat_name
        << not success

        succeed nothing
    << submit_score

    unlock_achievement(achievement_id: string) -> result[nothing]
        if not initialised >>
            fail SteamError.not_initialised
        << not initialised

        success :: bool = false
        unsafe >>
            success = SteamworksLib.SteamUserStats_SetAchievement(achievement_id)
        << unsafe

        if not success >>
            fail SteamError.achievement_failed with id: achievement_id
        << not success

        succeed nothing
    << unlock_achievement

    when update(delta)
        if initialised >>
            unsafe >>
                SteamworksLib.SteamAPI_RunCallbacks()
            << unsafe
        << initialised
    << update

    when destroyed
        if initialised >>
            unsafe >>
                SteamworksLib.SteamAPI_Shutdown()
            << unsafe
        << initialised
    << destroyed

<< SteamManager

# Layer 3 — Game code (clean — no unsafe visible)
entity ScoreSystem >>

    has SteamManager    # attached as component

    when game_over(final_score: int)
        match steam_manager.submit_score("HighScore", final_score) >>
            succeed _ ->
                debug.log("Score submitted: {final_score}")
            fail SteamError.not_initialised ->
                debug.log("Steam not active — score saved locally only")
            fail error ->
                debug.log("Score submission failed: {error.message}")
        << steam_manager.submit_score("HighScore", final_score)
    << game_over

<< ScoreSystem
// 80 lines total — complete wrapper + usage
```

### 🧭 Walk-Through — Full Wrapper Pattern

**Layer 1 — `extern "C" SteamworksLib`:** Eight function declarations. Type-checked at compile time. Written once. Never touched by game code.

**Layer 2 — `SteamManager` entity:** All `unsafe` blocks are inside this entity only. Game entities that need Steam attach `SteamManager` as a component and call its safe methods. `submit_score()` takes a `string` and `int` — no raw pointers, no manual error codes. Returns `result[nothing]` — consistent with Chapter 6.

**Error handling:** Three distinct failure modes (`not_initialised`, `stat_update_failed`, `achievement_failed`) — each with context. The caller knows exactly what went wrong and can respond appropriately.

**Layer 3 — `ScoreSystem`:** Zero `unsafe`. Zero raw pointers. Zero `extern`. Calls `steam_manager.submit_score()` like any other Skev function. If Steam is not available — the `match` handles it gracefully.

### ✅ Honest Advantage Statement

The wrapper pattern is 80 lines in Skev vs ~8 lines in C++ (no wrapper enforced) or C# (third-party wrapper, not enforced). Skev is longer — but Skev's length is justified: the unsafe boundary is enforced by the compiler, error handling is structured and exhaustive, and game code never sees unsafe. C++ and C# allow Steamworks calls to spread across the entire codebase — Skev's architecture prevents this. The trade-off of 80 lines vs 8 lines is: 72 lines of safety that the compiler will enforce forever.

---

## 8.8 — Platform Library Discovery

```swift
import skev.ffi

# Platform-aware — resolves extension automatically
lib :: Library = -> ffi.load_platform("physics")
# Windows:  physics.dll
# macOS:    libphysics.dylib
# Linux:    libphysics.so
# iOS:      libphysics.a (static — embedded)
# Android:  libphysics.so

# Explicit per-platform when filenames differ
lib = -> match platform.current >>
    Platform.windows  -> ffi.load("PhysX_64.dll")
    Platform.macos    -> ffi.load("libPhysX.dylib")
    Platform.linux    -> ffi.load("libPhysX.so")
    Platform.ios      -> ffi.load("libPhysX_static.a")
    _                 -> fail FFIError.unsupported_platform
<< platform.current
// 8 lines total
```

---

## 8.9 — Console SDK Integration

Console SDKs (PlayStation, Xbox, Nintendo) are covered by NDAs and cannot be documented publicly.

**What the public spec states:**
- The `extern "C"` mechanism works for all console SDK C APIs
- Console-specific Skev libraries are distributed under console developer license agreements
- Skev console support follows the same pattern as Unity and Unreal console support
- Specific function names and types are in the console-specific documentation under NDA

---

## 8.10 — Bidirectional Interop Quick Reference

```
Skev CALLING OTHER LANGUAGES:

C, C++, Rust, Swift (Tier 1 — zero overhead):
  extern "C" LibName >>
      function_name(args...) -> ReturnType
  << LibName
  unsafe >>
      LibName.function_name(args)
  << unsafe

Lua, Python, JavaScript (Tier 2 — scripting):
  import skev.scripting.lua     # or .python / .javascript
  state = lua.create()
  state.register("fn") >> args ... << register
  await state.execute(script)
  state.call("lua_fn", args...)

Java, C# no-AOT (Tier 3 — IPC):
  import skev.ipc
  proc = -> await ipc.spawn(command, args, stdin: ipc.pipe, stdout: ipc.pipe)
  await proc.write_line(data)
  result = -> await proc.read_line()

WebAssembly (Tier 4):
  import skev.wasm
  mod = -> wasm.load(bytes)
  mod.import("env", "fn") >> args ... << import
  mod.call("wasm_fn", [args])

OTHER LANGUAGES CALLING Skev:

export "C" function_name(args: C_types) -> C_type
    # Normal Skev code here
    # Automatically accessible from any language
<< function_name

C# Unity:   [DllImport("skev.dll")] static extern Type skev_fn(args);
Python:     lib = ctypes.CDLL("libskev.so"); lib.skev_fn(args)
Rust FFI:   extern "C" { fn skev_fn(args) -> Type; }
GDScript:   lib.skev_fn(args)  # via GDExtension

C-COMPATIBLE STRUCTS:
cdata StructName >>
    field :: CType    # exact declaration order — no reordering
<< StructName

CALLBACKS (C calling back into Skev):
callback fn_name(args: Types) -> ReturnType
    # Automatically marshaled to main thread
    # Write normal Skev entity code here
<< fn_name
unsafe >>
    CLib.register_callback(ffi.as_ptr(fn_name))
<< unsafe

WRAPPER PATTERN (always use this):
# Layer 1 — extern block (internal only)
# Layer 2 — safe Skev wrapper entity
# Layer 3 — game code (never sees unsafe)
```

---

*End of Chapter 8 v0.1 — Next: Chapter 9: Build & Distribution*
