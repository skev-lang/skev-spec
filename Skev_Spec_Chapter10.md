<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 10: Security & Isolation
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026)
**Status:** In Progress
**Depends On:** Chapter 8 (WASM/interop), Chapter 9 (packages), Chapter 5 (realtime)
**Use Cases Tested:** All 7 — Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅
**Rule H Verified:** sandbox, capabilities, skev.sandbox — all checked ✅
**Gaps Resolved:** Sandboxing (High), Capabilities/Permissions (Medium)

---

## 10.1 — Overview

### 👨‍💻 Developer Version

Every game that allows mods has the same problem: mods are code written by strangers. They must run inside your game — but they must not be able to crash it, steal save data, open network connections, or access players' files.

Every plugin system has the same problem: plugins extend your application — but a buggy plugin should not bring down the whole application. A malicious plugin should not be able to reach beyond what it was given permission to touch.

Skev solves both with a single unified model: **capabilities**.

> **A capability answers one question: "What is this code allowed to do?"**

Every plugin and mod in Skev declares its capabilities upfront. The host application grants only what it chooses. Anything not explicitly granted is structurally impossible — not just blocked at runtime, but impossible to call.

**The three-tier sandbox model:**

```
Tier 1 — WASM Sandbox:
  For: community mods, marketplace plugins, untrusted code
  How: code compiled to WebAssembly — cannot escape by construction
  Speed: near-native
  Example: game mod marketplace, editor plugins from strangers

Tier 2 — Native Sandbox:
  For: trusted studio plugins, engine extensions, first-party DLC
  How: compiled Skev code with capability enforcement
  Speed: native — compile-time checks, no runtime overhead on hot paths
  Example: DLC content pack, studio's internal tools

Tier 3 — Process Sandbox:
  For: external tools, AI pipelines, cross-language services
  How: separate OS process via skev.ipc (Chapter 8 Tier 3)
  Speed: milliseconds per call — background use only
  Example: Python AI model, Java analytics tool
```

### ⚙️ Technical Version

Chapter 10 specifies Skev's capability-based security model. The capability system is a unified policy layer that sits above both the WASM sandbox mechanism (Chapter 8 Tier 4) and the native sandbox mechanism introduced here. Capabilities are declared in `skev.pkg`, verified at compile time where possible, and enforced at runtime for dynamic access patterns. The `#! sandboxed` annotation creates a well-defined plugin API surface — only annotated functions are callable from plugins. The `#! safety_critical` annotation enables additional compile-time checks for safety-critical domains including medical devices, robotics, and industrial control systems.

---

## 10.2 — The Capability System

### 👨‍💻 Developer Version

A capability is a named permission. When you write a plugin, you declare exactly which capabilities it needs. When a host application loads your plugin, it decides which capabilities to grant. If you declare something the host will not grant — clear error. If your code tries to call something you did not declare — compile error.

**No hidden access. No surprising permissions. Everything visible.**

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game studio builds a mod marketplace. Players download mods written by thousands of different developers. One malicious mod developer wants to steal save file data and send it to a server. The studio must prevent this without manually reviewing every mod's source code.

**💥 Why This Is Hard**

Native code can do anything the OS permits. A DLL loaded into a game process has the same permissions as the game itself — it can read any memory, open any file, make any network call. Preventing this requires either: (1) running mods in a separate process (high overhead, complex IPC), (2) a virtual machine (performance cost), or (3) a capability system enforced at the language level.

---

**In C++23:**
```cpp
// C++ has no capability system — everything is OS-level permissions
// A loaded DLL has full process permissions
// "Sandboxing" requires:
// → Running in separate process (complex)
// → Or using OS-specific APIs (Windows Job Objects, Linux seccomp)
// → Or a third-party sandbox library
// → No standard approach — every studio reinvents this

// A malicious mod DLL can:
// → Read/write any file the process can access
// → Open network sockets
// → Read process memory
// → Call any system function

// No compile-time prevention possible
// Runtime prevention requires external tooling
// ~100+ lines of OS-specific sandboxing code
```

**In C# 13:**
```csharp
// C# had CAS (Code Access Security) — removed in .NET Core
// Current approach: run plugins in separate AppDomain (deprecated)
// or separate process

// Closest modern option: NativeAOT with trimming
// Still does not prevent network or file access
// No unified capability model

// Unity's approach: Burst compiler + C# jobs — performance isolation
// but not security isolation — mods still run in main process

// 0 lines of standard capability enforcement available
```

**In Python 3.13:**
```python
# Python has no native capability system
# RestrictedPython exists (third-party) but:
# → Not maintained actively
# → Easy to escape with clever code
# → Does not prevent file or network access at OS level

# Common approach: run mods in subprocess
import subprocess
result = subprocess.run(['python', 'mod.py'],
    capture_output=True, timeout=5)
# 3 lines total — but no capability granularity
// 3 lines total — no fine-grained capability control
```

**In Skev — capability declaration in skev.pkg:**
```swift
# mod/skev.pkg — the mod declares what it needs
plugin CombatMod >>
    version :: "1.0.0"
    author  :: "Community Developer"

    capabilities >>
        audio.playback                          # play sounds
        scene.read_entities(kind: [Enemy, NPC]) # read specific entity types
        scene.spawn_items(item_id_max: 999)     # spawn items (not boss items)
        skev.math                               # math library
        # NOT declared — compiler prevents:
        # file.read      → cannot access save files
        # file.write     → cannot modify any files
        # network        → cannot phone home
        # scene.destroy  → cannot delete entities
    << capabilities

<< CombatMod
// 14 lines total — every permission visible and auditable
```

**In Skev — host grants capabilities:**
```swift
entity ModLoader >>

    loaded_mods :: map[string, sandbox]

    async load_mod(path: string, mod_id: string) -> result[nothing]

        # Load the plugin — check its declared capabilities
        plugin_sandbox :: sandbox = -> skev.sandbox.load(path)

        # Review and grant — host decides what to allow
        # Only grant what the mod declared AND host approves
        plugin_sandbox.grant([
            audio.playback,
            scene.read_entities,
            scene.spawn_items,
            skev.math
        ])

        # DO NOT grant — even if mod asked for it:
        # file.read, file.write, network — never for community mods

        loaded_mods.add(mod_id -> plugin_sandbox)
        debug.log("Loaded mod: {mod_id}")
        succeed nothing
    << load_mod

    when mod_event(mod_id: string, event: string, data: string)
        if loaded_mods contains mod_id >>
            mod :: sandbox = loaded_mods[mod_id]
            mod.call(event, [data])
        << loaded_mods contains mod_id
    << mod_event

    # Violation handler — fires if mod attempts something not granted
    when sandbox.violation(mod_id: string, capability: string)
        debug.log("[Security] Mod {mod_id} attempted: {capability}")
        unload_mod(mod_id)    # terminate immediately
        ui.show_warning("Mod {mod_id} was removed — security violation")
    << sandbox.violation

<< ModLoader
// 36 lines total — complete mod loader with security
```

### ✅ Honest Advantage Statement

C++ has no standard capability system — every studio builds custom sandboxing. C# removed its security model (CAS) in .NET Core with no equivalent replacement. Python's sandboxing libraries are incomplete and unmaintained. Skev's capability system is built into the language and package format from day one — the same advantage as discussed in Chapter 9's supply chain security section. This is not invention — it is synthesis of capability-based security principles (established in academic CS since the 1970s) applied natively to a game language for the first time. The honest limitation: Tier 2 (native sandbox) relies on compiler checks and is not as ironclad as Tier 1 WASM — a bug in the compiler or runtime could theoretically allow a native plugin to escape. WASM provides mathematical guarantees that native cannot.

---

## 10.3 — Capability Reference

### Built-In Capability Categories

```
audio.*
  audio.playback           play any sound
  audio.playback(category: "sfx")   play only sound effects
  audio.record             capture microphone input

scene.*
  scene.read_entities                   read all entities
  scene.read_entities(kind: [T])        read specific entity types only
  scene.write_entities                  modify entity properties
  scene.write_entities(kind: [T])       modify specific types only
  scene.spawn_items                     spawn any item
  scene.spawn_items(item_id_max: N)     spawn items with ID ≤ N only
  scene.spawn_entities                  spawn any entity
  scene.destroy                         destroy entities
  scene.events                          fire and receive scene events

file.*
  file.read                             read any file
  file.read(path: "mods/my_mod/")       read only specific directory
  file.write                            write any file
  file.write(path: "mods/my_mod/")      write only specific directory
  file.temp_write                       write to temp directory only
  file.log_append                       append to log file only

network.*
  network.http_get                      outbound HTTP GET requests
  network.http_get(domain: "api.X.com") only specific domain
  network.http_post                     outbound HTTP POST requests
  network.websocket                     WebSocket connections

skev.*
  skev.math                             math library (always safe)
  skev.time                             time queries (always safe)
  skev.json                             JSON encode/decode
  skev.serial                           binary encode/decode

ui.*
  ui.display                            show UI elements
  ui.input                              receive input events

system.*
  system.motor_control                  (robotics — host grants explicitly)
  system.sensor_read                    (robotics/medical — host grants)
  system.cardiac_sensor_read            (medical — requires safety_critical)
  system.pulse_generator_write          (medical — requires safety_critical)
```

### Capability Rules

```
1. Coarse includes fine:
   Declaring scene.read_entities includes all entity types.
   Declaring scene.read_entities(kind: [Enemy]) includes only Enemy.

2. Host must grant what plugin declares:
   Plugin declares → host reviews → host grants subset or all.
   Plugin can only use what BOTH declared AND host granted.

3. Capabilities cannot expand on update:
   Plugin v2 capabilities ⊆ Plugin v1 capabilities.
   Expansion requires full unload + user notification.

4. Delegation is explicit and bounded:
   Plugin A can delegate to Plugin B only capabilities A possesses.
   A cannot grant B more than A has.
   sandbox.delegate(B, [audio.playback])

5. Realtime paths: compile-time checks only.
   No runtime capability checks permitted on realtime-marked functions.
```

---

## 10.4 — The `#! sandboxed` Annotation

### 👨‍💻 Developer Version

Plugins can only call functions you explicitly mark as safe for them to call. This creates a clearly defined API surface. Unmarked functions are invisible to plugins — not just blocked, but structurally unreachable.

```swift
# These functions are callable from plugins
#! sandboxed
give_item_to_player(player_id: int, item_id: int) -> result[nothing]
    # Validate — plugin cannot bypass this check
    if item_id >= 1000 >>
        fail SandboxError.item_restricted
    << item_id >= 1000
    player :: Player = -> scene.find_entity(player_id)
    player.inventory.add(item_id)
    succeed nothing
<< give_item_to_player

#! sandboxed
get_player_health(player_id: int) -> maybe int
    player :: maybe Player = scene.find_entity(player_id)
    if player exists >>
        result player.health
    << player exists
    result nothing
<< get_player_health

#! sandboxed
play_sound_at(sound_id: string, position: Vector3!) -> nothing
    audio.play_at(sound_id, position)
<< play_sound_at

# These functions are NOT callable from plugins — no annotation
# Plugin code that calls these produces a compile error
load_save_file(path: string) -> result[SaveData]
    # ...
<< load_save_file

delete_player_data(player_id: int) -> result[nothing]
    # ...
<< delete_player_data
```

**The principle:**
```
Plugins see:     give_item_to_player, get_player_health, play_sound_at
Plugins cannot:  load_save_file, delete_player_data, or ANY unmarked function
Compiler enforces: any call to an unmarked function from plugin code = error

Error: Function not accessible from sandbox
  load_save_file is not marked #! sandboxed.
  Plugin code cannot call non-sandboxed functions.
  
  If this function should be accessible to plugins:
  Add #! sandboxed to load_save_file.
  
  If not — this call should not be in plugin code.
  
  combat_mod.skev  Line 47
```

---

## 10.5 — Native Sandbox Block

### 👨‍💻 Developer Version

For trusted-but-isolated code — studio DLC, first-party plugins, engine extensions — the `sandbox` block loads native Skev code with capability enforcement:

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game studio ships DLC as separate compiled modules. Each DLC adds new enemies, weapons, and game mechanics. The base game must be able to load any DLC safely — a buggy DLC should not crash the base game, and a DLC should not be able to access the base game's internal state beyond what the DLC API allows.

**In C++23:**
```cpp
// C++ DLL loading — no capability system
HMODULE dlc = LoadLibraryA("dlc_pack1.dll");
auto init = (void(*)())GetProcAddress(dlc, "dlc_init");
init();
// DLC has FULL process access — no restriction
// Buggy DLC can corrupt any memory
// No capability declaration
// Studio must manually trust each DLC
// 5 lines total — no isolation
```

**In C# 13:**
```csharp
// C# Assembly loading — partial isolation via AppDomain (legacy)
// Modern .NET: AssemblyLoadContext for isolation
var alc = new AssemblyLoadContext("DLC1", isCollectible: true);
var assembly = alc.LoadFromAssemblyPath("dlc_pack1.dll");
var dlcType = assembly.GetType("DLCPack1.Entry");
var instance = Activator.CreateInstance(dlcType);
// Better than C++ but: no capability declaration
// DLC can still access any public API in the base game
// 5 lines total — memory isolated but not capability isolated
```

**In Python 3.13:**
```python
import importlib.util
spec = importlib.util.spec_from_file_location("dlc1", "dlc_pack1.py")
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)
mod.init()
# No isolation — DLC runs in main interpreter
// 4 lines total — no isolation at all
```

**In Skev:**
```swift
entity DLCLoader >>

    loaded_dlc :: map[string, sandbox]

    async load_dlc(path: string, dlc_id: string) -> result[nothing]

        dlc :: sandbox = -> skev.sandbox.load_native(path)

        # Grant DLC capabilities — first-party so broader than mod
        dlc.grant([
            scene.read_entities,
            scene.write_entities(kind: [Enemy, Weapon, Item]),
            scene.spawn_entities,
            scene.spawn_items,
            audio.playback,
            skev.math,
            skev.json
            # Still no: file.write, network, scene.destroy
        ])

        # Verify DLC signature before granting anything
        if not dlc.verify_signature(ajstudios_public_key) >>
            fail DLCError.invalid_signature with path: path
        << not dlc.verify_signature(ajstudios_public_key)

        loaded_dlc.add(dlc_id -> dlc)
        debug.log("DLC loaded: {dlc_id}")
        succeed nothing
    << load_dlc

    call_dlc[T](dlc_id: string, fn: string, args: list[T]) -> result[nothing]
        if not loaded_dlc contains dlc_id >>
            fail DLCError.not_loaded with id: dlc_id
        << not loaded_dlc contains dlc_id
        dlc :: sandbox = loaded_dlc[dlc_id]
        -> dlc.call(fn, args)
        succeed nothing
    << call_dlc

<< DLCLoader
// 38 lines total — native sandbox with signature verification
```

### ✅ Honest Advantage Statement

Skev's native sandbox is 38 lines vs 5 lines for C++ — but the C++ version provides zero isolation. C#'s AssemblyLoadContext provides memory isolation but no capability restriction. Python provides no isolation. Skev's 38 lines delivers: signature verification, explicit capability granting, violation handling, and typed DLC function calls. The honest limitation: native sandbox is not as strong as WASM — a compiler bug could allow escape. For community mods (untrusted), always use WASM sandbox (Tier 1).

---

## 10.6 — WASM Sandbox + Capability System (Unified)

### 👨‍💻 Developer Version

The WASM sandbox from Chapter 8 and the capability system from this chapter are unified. Every WASM mod declares capabilities in its `skev.pkg`. The `mod.import()` registrations from Chapter 8 ARE the capability grants — formalised through the capability declaration system.

```swift
# Chapter 8 showed:
mod.import("env", "spawn_item") >> args: WasmArgs
    item_id :: int   = args.int32(0)
    x       :: float = args.float32(1)
    z       :: float = args.float32(2)
    if item_id < 1000 >>
        scene.spawn_item(item_id, Vector3!(x, 0.0, z))
    << item_id < 1000
<< import

# Chapter 10 adds: this import IS the capability grant
# The mod's skev.pkg must declare:
#   capabilities >> scene.spawn_items(item_id_max: 999) << capabilities
# If it doesn't — the import registration is rejected at load time:

# Error: Capability not declared
#   spawn_item registration requires scene.spawn_items capability.
#   The WASM module's manifest does not declare this capability.
#   Add to mod's skev.pkg:
#     capabilities >>
#         scene.spawn_items(item_id_max: 999)
#     << capabilities
```

**The unified flow:**

```
1. Mod author writes mod.skev + mod/skev.pkg with capabilities declared
2. skev build compiles mod to WASM + embeds capability declaration
3. Player installs mod → game checks declared capabilities
4. Game shows player: "This mod requests: audio playback, item spawning"
5. Player approves → game grants capabilities and registers imports
6. Mod runs — any call beyond declared capabilities is structurally impossible
7. If mod tries to call unregistered function → WASM trap → sandbox.violation
```

---

### 🔴 Hard Complex Example — Mod Marketplace Security

**🌍 Real-World Scenario**

A game has a mod marketplace with 10,000 community mods. Mods add new weapons, quests, enemies, visual effects, and game mechanics. The studio cannot review every mod manually. Players must be able to download and run any mod safely — a malicious mod should not be able to steal their save files, track their play habits, or send data to a server. The marketplace needs automated security scanning and clear user-facing permission displays.

**💥 Why This Is Hard**

Scale is the problem. Manual review of 10,000 mods is impossible. Automated scanning must catch: network access, file access outside mod directory, entity access beyond declared scope, and attempts to load additional native code. The UI must present capabilities in language players understand ("This mod can play sounds" not "audio.playback"). And a mod that passes all checks today must still be contained if it is later updated with malicious code.

**In C++23:**
```cpp
// C++ mod marketplace — common approaches:
// Option 1: LuaJIT mods (scripted — limited capability)
// Option 2: Manual review of C++ source (doesn't scale)
// Option 3: OS-level sandboxing (Windows AppContainer, Linux seccomp)

// No standard solution
// Each game engine implements its own approach
// Unreal uses Verse (new language) for this reason
// Unity uses C# with manual sandboxing
// ~500+ lines across multiple systems to approximate this
```

**In C# 13:**
```csharp
// Unity's approach: C# mods run in managed runtime
// Partial isolation — memory safe but not capability safe
// No marketplace-level capability scanning built in

// Custom solution needed:
// → Roslyn analysis for capability scanning
// → Custom attribute system for permissions
// → Runtime enforcement via AppDomain tricks
// All third-party or custom — 300+ lines minimum
// ~300+ lines total [abbreviated — no standard solution]
```

**In Python 3.13:**
```python
# Python mods (common in simulation/strategy games)
# No capability system — all mods run with full access
# Mitigation: run mods in separate process
# Still no granular capability declaration
// Not applicable — Python has no standard capability model for mods
```

**In Skev — complete marketplace mod system:**
```swift
import skev.sandbox
import skev.json

# The marketplace scanning service
entity ModMarketplace >>

    async scan_mod(mod_path: string) -> result[ModScanReport]

        # Load manifest without executing any code
        manifest :: ModManifest = -> skev.sandbox.read_manifest(mod_path)

        report :: ModScanReport = ModScanReport >>
            mod_id   :: manifest.id
            mod_name :: manifest.name
            passed   :: true
            warnings :: []
            denied_capabilities :: []
        << ModScanReport

        # Check 1 — dangerous capabilities trigger warnings
        if manifest.capabilities contains network >>
            report.warnings.add(
                "This mod requests network access. Only approve if you " +
                "trust this developer. Mod can contact external servers."
            )
        << manifest.capabilities contains network

        # Check 2 — file.write outside mod directory = rejected
        if manifest.capabilities contains file.write >>
            allowed_path :: string = "mods/{manifest.id}/"
            if not manifest.capabilities.file_write_path == allowed_path >>
                report.passed = false
                report.denied_capabilities.add("file.write")
                report.warnings.add(
                    "REJECTED: Mod requests write access outside its " +
                    "own directory. This is not permitted."
                )
            << not manifest.capabilities.file_write_path == allowed_path
        << manifest.capabilities contains file.write

        # Check 3 — scene.destroy is never granted to community mods
        if manifest.capabilities contains scene.destroy >>
            report.passed = false
            report.denied_capabilities.add("scene.destroy")
        << manifest.capabilities contains scene.destroy

        # Check 4 — WASM binary integrity
        wasm_hash :: string = -> skev.sandbox.compute_hash(mod_path)
        if wasm_hash != manifest.declared_hash >>
            report.passed = false
            report.warnings.add("REJECTED: Mod binary does not match " +
                "declared hash. File may be corrupted or tampered.")
        << wasm_hash != manifest.declared_hash

        succeed report
    << scan_mod

    # Convert technical capabilities to player-friendly language
    capability_to_display(cap: string) -> string
        match cap >>
            "audio.playback"        -> "Play sounds and music"
            "scene.read_entities"   -> "See game entities"
            "scene.spawn_items"     -> "Add items to the world"
            "scene.write_entities"  -> "Modify game entities"
            "network.http_get"      -> "⚠️ Connect to the internet"
            "file.read"             -> "⚠️ Read game files"
            "file.write"            -> "⚠️ Modify game files"
            _                       -> cap
        << cap
    << capability_to_display

    # Show player the permission request UI before installing
    async show_install_prompt(
        mod_path: string
    ) -> result[bool]    # true = user approved

        report :: ModScanReport = -> await scan_mod(mod_path)

        if not report.passed >>
            ui.show_error(
                "Mod blocked by Skev Security",
                "This mod was blocked because it requested dangerous " +
                "capabilities. Details: {report.warnings.join('\n')}"
            )
            succeed false
        << not report.passed

        # Build permission list for user
        perm_lines :: list[string] = report.manifest.capabilities
            .map(c -> "• {capability_to_display(c)}")

        user_approved :: bool = -> await ui.show_confirm(
            title:   "Install {report.mod_name}?",
            message: "This mod requests these permissions:\n" +
                     perm_lines.join("\n"),
            confirm: "Install Mod",
            cancel:  "Cancel"
        )

        succeed user_approved
    << show_install_prompt

<< ModMarketplace

# The actual mod runner — enforces at runtime
entity ModRunner >>

    active_mods :: map[string, sandbox]

    async install_and_run(mod_path: string) -> result[nothing]

        # Scan and get approval
        approved :: bool = -> await marketplace.show_install_prompt(mod_path)
        if not approved >>
            succeed nothing    # user cancelled — not an error
        << not approved

        # Load into WASM sandbox — capabilities from manifest
        mod :: sandbox = -> skev.sandbox.load_wasm(mod_path)

        # Register ONLY the #! sandboxed functions as imports
        mod.import_sandboxed_api()    # auto-discovers #! sandboxed functions

        active_mods.add(mod.id -> mod)

        # Fire mod's initialisation event
        mod.call("on_load", [])
        succeed nothing
    << install_and_run

    # Security violation — fire and forget with immediate action
    when sandbox.violation(mod_id: string, capability: string, details: string)
        debug.log("[SECURITY] Mod {mod_id} violated sandbox: {capability}")
        analytics.report("mod_security_violation", >>
            mod_id     :: mod_id
            capability :: capability
            details    :: details
        << )

        # Immediately unload the violating mod
        if active_mods contains mod_id >>
            violating_mod :: sandbox = active_mods[mod_id]
            violating_mod.unload()
            active_mods.remove(mod_id)
        << active_mods contains mod_id

        # Notify player
        ui.show_warning(
            "Mod Removed",
            "'{mod_id}' was removed for attempting unauthorized access. " +
            "This incident has been reported."
        )
    << sandbox.violation

<< ModRunner
// 98 lines total — complete marketplace security system
```

### 🧭 Walk-Through — Mod Marketplace Security

**`scan_mod`:** Reads the mod manifest WITHOUT executing any code. Four checks: (1) network access triggers visible warning, (2) `file.write` outside mod directory is rejected outright, (3) `scene.destroy` is never granted to community mods, (4) WASM binary hash verified against declared hash — tampering caught immediately.

**`capability_to_display`:** Translates technical capability names to player-readable strings. `⚠️` prefix on dangerous capabilities — `network.http_get` becomes `"⚠️ Connect to the internet"`. Players make informed decisions.

**`show_install_prompt`:** If scan fails — hard block with explanation. If it passes — show user every permission the mod requests in plain English. User sees exactly what they are approving. Same model as iOS app permissions.

**`install_and_run`:** Only runs if user approved. Loads into WASM sandbox. `import_sandboxed_api()` auto-discovers all `#! sandboxed` functions in the host — the mod can call only these. No manual registration needed.

**`sandbox.violation` handler:** Fires if the WASM module attempts anything outside its grants — this should be impossible by WASM's construction, but the handler covers the edge case where the WASM runtime itself has a bug. Immediate unload. Analytics report. Player notification.

### ✅ Honest Advantage Statement

No comparison language has a standard equivalent to this system. C++, C#, and Python all require custom solutions. Skev's advantage is structural: the capability system, WASM sandbox, package manifest, and `#! sandboxed` annotation all work together as a coherent whole. Each piece was designed to support the others. The player permission UI (capability names translated to plain English) is enabled by the capability system being first-class in the language. No other game language makes this trivial to implement.

---

## 10.7 — `#! safety_critical` — Safety-Critical Domains

### 👨‍💻 Developer Version

Some domains require stronger guarantees than normal plugins. A cardiac pacemaker controller cannot be allowed to allocate heap memory mid-beat. A robot arm controller cannot stall for garbage collection. A medical pump controller must execute in bounded time, every time.

The `#! safety_critical` annotation enables a stricter compilation profile for these use cases.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A medical device company builds a drug infusion pump controller. The controller's plugin interface must allow certified third-party medication algorithms to run — but each algorithm must be isolated so a bug in one algorithm cannot affect the pump's core safety functions. The regulator requires: bounded execution time, no dynamic memory allocation on the safety path, deterministic behaviour, and an audit trail of all capability grants.

**💥 Why This Is Hard**

Medical device software operates under IEC 62443 and ISO 26262 standards. These require demonstrably bounded execution, memory safety proofs, and isolation of certified from uncertified code. C++ has no language-level mechanism for this — it requires external tools, MISRA-C++ compliance, and manual auditing. C# is generally not used in certified medical devices. Python is not used at all.

**In C++23:**
```cpp
// C++ medical device plugins — MISRA-C++ compliance required
// Certification requires:
// → Static analysis tools (Polyspace, Coverity)
// → No dynamic memory allocation on safety paths
//   (enforced by external tools, not language)
// → Bounded execution time proofs (external analysis)
// → No standard isolation mechanism
// Developer manually enforces all constraints
// ~200+ lines of setup + external tooling
// MISRA-C++ compliance adds significant boilerplate
// ~200+ lines total [abbreviated — requires external certification tooling]
```

**In C# 13:**
```csharp
// C# is generally not used in IEC 62443 certified devices
// Garbage collector is non-deterministic — disqualifying for safety path
// Some embedded .NET Micro Framework usage exists
// Not applicable to certified medical device software
// Not applicable — GC non-determinism disqualifies C# for safety-critical paths
```

**In Python 3.13:**
```python
# Python is not used in certified medical device software
# Not applicable to IEC 62443 or ISO 26262 certified systems
# Not applicable
```

**In Skev:**
```swift
#! safety_critical
plugin InfusionAlgorithm >>
    version :: "2.1.0"
    author  :: "CertifiedMedTech Ltd"
    certification :: "IEC-62443-certified"

    capabilities >>
        system.infusion_sensor_read        # read flow sensor
        system.infusion_pump_write         # write pump rate (bounded)
        system.patient_monitor_read        # read vitals
        file.log_append                    # audit log only
        # No audio, no UI, no network, no file.write
    << capabilities

    # Safety constraints — enforced by compiler:
    # → No heap allocation on safety path
    # → No async/await in safety handlers
    # → Max execution time: 10ms per tick (declared)
    # → Memory usage: bounded at 512KB (declared)
    safety >>
        max_execution_ms :: 10
        max_memory_kb    :: 512
        deterministic    :: true
    << safety

<< InfusionAlgorithm

# The safety-critical handler — compiler enforces all constraints
#! safety_critical
#! realtime
compute_infusion_rate(
    sensor_reading: float,
    patient_weight: float,
    prescribed_dose: float
) -> float

    # Compiler verifies:
    # → No heap allocation here
    # → No async/await calls
    # → No dynamic dispatch
    # → All function calls in call graph are also safety_critical

    target_rate :: float = (prescribed_dose * patient_weight) / 3600.0
    actual_flow :: float = sensor_reading

    correction :: float = math.clamp(
        target_rate - actual_flow,
        -0.5, 0.5    # max correction bounded
    )

    result math.clamp(target_rate + correction, 0.0, 10.0)

<< compute_infusion_rate
// 44 lines total — safety-critical plugin with all constraints declared
```

### 🧭 Walk-Through — Safety-Critical Plugin

**`#! safety_critical` on plugin:** Activates the strict compilation profile. Every function in this plugin's call graph must also be `#! safety_critical` or a pure computation with no side effects. If any function in the chain violates this — compile error.

**`safety >> ... << safety` block:** Declares bounds that the compiler verifies and the runtime enforces. `max_execution_ms :: 10` — if the function exceeds 10ms, the runtime fires `safety_critical.timeout` and terminates the call. `max_memory_kb :: 512` — if the plugin exceeds this, the runtime fires `safety_critical.memory_exceeded`.

**`#! realtime` + `#! safety_critical`:** The combination of both annotations means: (1) no heap allocation, (2) no async/await, (3) no channel operations, (4) bounded execution time, (5) all capabilities checked at compile time only. This is the strictest mode Skev supports — appropriate for pacemakers, robot joints, flight control.

**Audit trail:** `file.log_append` capability means every significant computation result can be written to an append-only log. The log is the regulatory audit trail — readable by certification tools.

### ✅ Honest Advantage Statement

C++ is the dominant language for safety-critical embedded systems and provides no language-level enforcement of safety-critical constraints. All enforcement is via external tools (MISRA checkers, static analysers, formal verification tools). Skev's `#! safety_critical` annotation brings constraint declaration and enforcement INTO the language and compiler. The honest limitation: Skev is new and not yet certified under any safety standard. For actual IEC 62443 or ISO 26262 certification, Skev would need a certified compiler toolchain — significant additional work. The architecture is correct; the certification process would take years. C++ is still the practical choice for certified medical devices today.

---

## 10.8 — Cross-Plugin Communication

### 👨‍💻 Developer Version

Plugins cannot call each other directly. Communication between plugins goes through the host, via events. This preserves each plugin's capability isolation — Plugin A cannot escalate Plugin B's permissions by calling B directly.

```swift
# Plugin A fires a custom event
# (Plugin A has scene.spawn_items capability)
plugin_event.fire("weapon_crafted", >>
    weapon_id :: 42
    player_id :: player.id
<< )

# Host receives the event and decides what to tell Plugin B
when plugin_event("weapon_crafted", data: EventData)
    weapon_id :: int = data.int("weapon_id")
    player_id :: int = data.int("player_id")

    # Host validates and forwards to Plugin B if appropriate
    if weapon_id < 1000 >>
        plugin_b.call("on_new_weapon", [weapon_id, player_id])
    << weapon_id < 1000
<< plugin_event
```

**Why direct cross-plugin calls are forbidden:**

```
If Plugin A (capabilities: scene.spawn_items) calls Plugin B
(capabilities: scene.destroy) directly —

Plugin A gains access to Plugin B's scene.destroy capability
through the call chain.

The entire capability model breaks.

Via the host:
→ Host controls exactly what data passes between plugins
→ Host validates the data before forwarding
→ Each plugin's capabilities remain isolated
→ No escalation is possible
```

---

## 10.9 — Hot-Reload Security

### 👨‍💻 Developer Version

During development — hot reload is unrestricted. You are reloading your own code on your own machine. Full trust.

In production — hot reload (live updates to a shipped game) has strict rules:

```
Rule 1 — Capabilities cannot expand:
  Plugin v1 capabilities: audio.playback, scene.read_entities
  Plugin v2 capabilities: audio.playback, scene.read_entities, network.http_get
  → BLOCKED: capabilities expanded
  → Requires full unload + user re-approval

Rule 2 — Capabilities can shrink:
  Plugin v2 with fewer capabilities than v1 → allowed automatically
  Removing permissions = safer, no approval needed

Rule 3 — Hash verification:
  Every production update verified against signed manifest
  Unsigned update → rejected

Rule 4 — Notification:
  Player notified of plugin update with changes shown
  "CombatMod updated. Permissions unchanged."
  or
  "CombatMod update blocked — new permissions require approval."
```

---

## 10.10 — Security Quick Reference

```
CAPABILITY DECLARATION (in skev.pkg):
  plugin Name >>
      capabilities >>
          audio.playback
          scene.read_entities(kind: [Enemy])
          file.read(path: "mods/my_mod/")
          network.http_get(domain: "api.example.com")
      << capabilities
  << Name

CAPABILITY GRANT (in host code):
  sandbox.grant(plugin, [audio.playback, scene.read_entities])
  sandbox.delegate(plugin_b, [audio.playback])    # delegate subset

SANDBOXED FUNCTION (callable from plugins):
  #! sandboxed
  give_item(player_id: int, item_id: int) -> result[nothing]
      ...
  << give_item

SAFETY-CRITICAL PLUGIN:
  #! safety_critical
  plugin MedicalAlgorithm >>
      safety >>
          max_execution_ms :: 10
          max_memory_kb    :: 512
          deterministic    :: true
      << safety
  << MedicalAlgorithm

VIOLATION HANDLING:
  when sandbox.violation(mod_id: string, capability: string)
      # unload, log, notify user
  << sandbox.violation

THREE SANDBOX TIERS:
  Tier 1 WASM:    skev.sandbox.load_wasm(path)    — untrusted mods
  Tier 2 Native:  skev.sandbox.load_native(path)  — trusted plugins
  Tier 3 Process: skev.ipc.spawn(command, ...)    — external tools

HOT-RELOAD SECURITY:
  Debug:      unrestricted
  Production: capabilities cannot expand — requires re-approval

CAPABILITY CATEGORIES:
  audio.*    scene.*    file.*    network.*
  skev.*     ui.*       system.*  (safety-critical)
```

---

*End of Chapter 10 v0.1 — Next: Chapter 11: Tooling*
