<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 8: Interoperability — Design Decisions
**Version:** 0.1
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked — Chapter 8 ready to write
**Process:** Steps 1–8 completed per Design Practices v2.4
**Rule H Applied:** skev.ffi (standard abbreviation — no conflicts) ✅
**Rule I Applied:** Both directions considered for all mechanisms ✅

---

## Core Architecture — Four-Tier Model

```
Tier 1 — C ABI (zero overhead):
  Languages: C, C++, Rust, Swift, Zig, Go (with export), C# Native AOT
  Mechanism: extern "C" (Skev calls them), export "C" (they call Skev)

Tier 2 — Embeddable scripting (minimal/moderate overhead):
  Languages: Lua, Python, JavaScript
  Mechanism: skev.scripting.X — Skev hosts the runtime

Tier 3 — IPC-based (high latency — not per-frame):
  Languages: Java, Kotlin, C# (no Native AOT), Ruby
  Mechanism: skev.ipc — process spawn and pipes

Tier 4 — WebAssembly (low overhead):
  Mechanism: skev.wasm — Skev hosts WASM modules
```

---

## Decision 1 — `extern "C"` Block (Skev → C/C++/Rust/Swift)

```swift
extern "C" PhysicsLib >>
    skev_physics_init(gravity: float) -> int
    skev_physics_step(delta: float32) -> nothing
<< PhysicsLib
```

- Uses `>>` `<<` — consistent with all Skev blocks
- Names the library for clarity
- Type signatures verified at compile time
- Must be called inside `unsafe` blocks

**Status: Locked ✅**

---

## Decision 2 — `unsafe` Block

```swift
unsafe >>
    result = PhysicsLib.skev_physics_init(-9.8)
<< unsafe
```

- C functions ONLY callable inside unsafe blocks
- ARC and null safety do NOT apply inside
- Compiler error if C function called outside unsafe
- Developer takes full responsibility inside the block

**Status: Locked ✅**

---

## Decision 3 — `export "C"` (Other languages → Skev)

```swift
export "C" skev_find_path(
    start_x: float32, start_y: float32,
    end_x:   float32, end_y:   float32
) -> *float32
    path :: NavPath = nav.find(
        Vector3!(start_x, start_y, 0.0),
        Vector3!(end_x, end_y, 0.0)
    )
    result ffi.to_c_array(path.waypoints)
<< skev_find_path
```

**Use cases:**
- C# Unity P/Invoke calling Skev plugin
- Python ctypes/cffi calling Skev library
- C++ calling Skev AI/physics systems
- GDScript (Godot) via GDExtension
- Unreal Engine calling Skev module

**Status: Locked ✅**

---

## Decision 4 — C Type Mapping

| Skev Type | C Type | Notes |
|---|---|---|
| `int` | `int32_t` | Exact match |
| `uint32` | `uint32_t` | Exact match |
| `float` | `float` | Exact match |
| `float64` | `double` | Exact match |
| `bool` | `uint8_t` | 0 or 1 |
| `string` | `const char*` | Automatic null-term conversion |
| `*TypeName` | `TypeName*` | Raw pointer — may be null |
| `&TypeName` | `TypeName*` | Non-null guaranteed |
| `nothing` | `void` | Return type only |
| `array[T, N]` | `T[N]` | Inline array |

**Status: Locked ✅**

---

## Decision 5 — `cdata` Keyword (C-compatible structs)

```swift
cdata PxVec3 >>
    x :: float32
    y :: float32
    z :: float32
<< PxVec3
```

- Fields in exact declaration order — no compiler reordering
- C-standard alignment padding added
- Layout matches what C compiler would produce
- Safe to pass directly to/from C functions

`cdata` vs `data`:
- `data`: compiler reorders for optimal packing
- `cdata`: exact C-compatible layout

**Status: Locked ✅**

---

## Decision 6 — `callback` Type (C calling back into Skev)

```swift
callback on_collision(body_a: int, body_b: int) -> nothing
    scene.handle_collision(body_a, body_b)
<< on_collision

unsafe >>
    PhysXLib.register_collision_callback(
        ffi.as_ptr(on_collision)
    )
<< unsafe
```

**Callback marshaling rules:**
- Normal callbacks → automatically queued to main thread
- `realtime` callbacks → fire immediately on calling thread (compiler enforces restrictions)
- Developer never thinks about thread safety for normal callbacks

**Status: Locked ✅**

---

## Decision 7 — Tier 2: Scripting Languages

### Lua (`skev.scripting.lua`)

```swift
lua :: LuaState = lua.create()

lua.register("spawn_entity") >> args
    name :: string = args.string(0)
    pos  :: Vector3! = args.vector3(1)
    scene.spawn_by_name(name, pos)
<< register

async load_mod() -> result[nothing]
    script :: string = -> await file.read_text("mods/combat.lua")
    -> lua.execute(script)
    succeed nothing
<< load_mod
```

### Python (`skev.scripting.python`)

```swift
python :: PythonState = python.create()
python.set("input_data", game_data)
-> python.execute(script)
result :: string = -> python.get_string("output")
```

**Python GIL constraint:** Only one Skev task can run Python at a time. Compiler warns if Python called from concurrent tasks.

### JavaScript (`skev.scripting.javascript`)

```swift
js :: JSState = javascript.create()
js.set("gameData", game_state_json)
-> js.execute(script)
```

**Status: Locked ✅**

---

## Decision 8 — Tier 3: IPC-Based

```swift
import skev.ipc

proc :: Process = -> await ipc.spawn(
    command: "java",
    args: ["-jar", "tool.jar"],
    stdin:  ipc.pipe,
    stdout: ipc.pipe
)

-> await proc.write_line(command)
response :: string = -> await proc.read_line()
```

**Warning — high latency:** Milliseconds per call. Not suitable for per-frame use. Background operations only.

**Status: Locked ✅**

---

## Decision 9 — Tier 4: WebAssembly

```swift
import skev.wasm

wasm_bytes :: list[uint8] = -> await file.read_bytes("plugin.wasm")
mod :: WasmModule = -> wasm.load(wasm_bytes)

# Skev exposes functions to WASM
mod.import("env", "log_message") >> msg: string
    debug.log("[Plugin] {msg}")
<< import

# Skev calls WASM functions
result :: float = module.call("compute", [x, y, z])
```

**WASM memory model:** Only primitives and byte arrays are zero-copy. Complex types require serialisation at the boundary.

**Status: Locked ✅**

---

## Decision 10 — Dynamic Library Loading

```swift
import skev.ffi

# Platform-aware — automatic extension resolution
lib :: Library = -> ffi.load_platform("physics")
# Windows: physics.dll | macOS: libphysics.dylib | Linux: libphysics.so

# Explicit per-platform
lib = -> match platform.current >>
    Platform.windows -> ffi.load("physics.dll")
    Platform.macos   -> ffi.load("libphysics.dylib")
    Platform.linux   -> ffi.load("libphysics.so")
    _                -> fail FFIError.unsupported_platform
<< platform.current

# Symbol lookup
unsafe >>
    fn_ptr = lib.symbol("physics_step")
<< unsafe
```

**Status: Locked ✅**

---

## Decision 11 — String Marshaling

```
Skev string → C const char*:
  → Automatic null-terminated copy on stack
  → Valid only for duration of the C call
  → ≤ 4095 bytes: stack buffer (zero heap allocation)
  → > 4095 bytes: temporary heap allocation, freed after call

C const char* → Skev string:
  → Must call ffi.string_from_ptr(ptr) explicitly
  → Developer responsible for pointer lifetime
```

**Status: Locked ✅**

---

## Decision 12 — Wrapper Pattern (The Safe Bridge)

**The architecture every interop scenario follows:**

```
Layer 1 — Raw extern block (unsafe — internal only)
         ↓
Layer 2 — Skev wrapper entity (safe — this is the public API)
         ↓
Layer 3 — Game code (never sees unsafe)
```

This pattern applies in both directions:
- Skev wrapping a C library (Skev → C)
- Skev exposing itself to Unity (C# → Skev export)

**Status: Locked ✅**

---

## Decision 13 — Skev Plugin ABI (Hot-Reload Pattern)

```swift
# Game shell (stable — compiled once)
lib :: Library = -> ffi.load_platform("game_logic")
update_fn = lib.symbol("game_update")
render_fn = lib.symbol("game_render")

# Game logic DLL (hot-reloadable)
export "C" game_update(delta: float32) -> nothing
    world.update(delta)
<< game_update

export "C" game_render() -> nothing
    renderer.draw(world)
<< game_render
```

Used for: live development, editor-game separation, mod systems, console certification (ship shell, update logic).

**Status: Locked ✅**

---

## Bidirectional Interop Table (Rule I)

| Language | Skev → Them | Them → Skev | Tier |
|---|---|---|---|
| C | `extern "C"` | `export "C"` | 1 |
| C++ | `extern "C"` | `export "C"` | 1 |
| Rust | `extern "C"` | `export "C"` | 1 |
| Swift | `extern "C"` | `export "C"` | 1 |
| C# Native AOT | `extern "C"` | `export "C"` | 1 |
| C# Unity | — | P/Invoke → `export "C"` | 1 |
| Python | `skev.scripting.python` | `ctypes` → `export "C"` | 2 |
| Lua | `skev.scripting.lua` | `lua.register()` | 2 |
| JavaScript | `skev.scripting.javascript` | `mod.import()` | 2/4 |
| Java | `skev.ipc` | `skev.ipc` (respond) | 3 |
| WebAssembly | `skev.wasm` | `mod.import()` | 4 |
| GDScript | `extern "C"` | `export "C"` (GDExtension) | 1 |

---

## Concern Resolutions

| Concern | Resolution |
|---|---|
| C calling back into Skev | `callback` type — marshaled to main thread |
| Struct layout matching | `cdata` keyword — exact C layout |
| FFI callback thread safety | Normal: auto main thread. Realtime: immediate |
| Platform library discovery | `ffi.load_platform()` — automatic extension |
| Console SDK (NDA) | `extern "C"` mechanism works — names not documented |
| Python GIL | Compiler warns on concurrent Python task calls |
| WASM memory model | Primitives zero-copy, complex types serialised |
| IPC latency | Documented — not for per-frame use |

---

## New Keywords (Rule H Verified)

| Keyword | Purpose | Rule H Check |
|---|---|---|
| `extern` | Declare C library interface | ✅ No conflict |
| `unsafe` | Safety boundary marker | ✅ Similar to Rust but different syntax |
| `export` | Export Skev function to C | ✅ Standard term |
| `cdata` | C-compatible struct | ✅ No conflict |
| `callback` | C-callable function pointer | ✅ No conflict |
| `skev.ffi` | FFI utilities library | ✅ Standard abbreviation |
| `skev.scripting.lua` | Lua bridge | ✅ No conflict |
| `skev.scripting.python` | Python bridge | ✅ No conflict |
| `skev.ipc` | IPC utilities | ✅ Standard abbreviation |
| `skev.wasm` | WebAssembly host | ✅ No conflict |
| `*TypeName` | Raw C pointer | ✅ C convention |
| `&TypeName` | Non-null C reference | ✅ C++ convention |

---

## Rule C Gap Update

Interoperability/FFI gap is now **RESOLVED**. Removing from critical gaps.

**Remaining critical gaps:**
- Generics — Chapter 3.5 (only critical gap remaining)

---

*All decisions locked — Chapter 8 ready to write*
*Version 0.1*
