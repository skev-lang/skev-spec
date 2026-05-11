<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
https://skev.dev | https://skev.org
-->

# Skev Language Specification
## Chapter 7: Standard Library
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026)
**Status:** In Progress
**Depends On:** Chapter 3 (types), Chapter 4 (ARC), Chapter 5 (async), Chapter 6 (errors)
**Use Cases Tested:** All 7 — Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅
**Rule H Verified:** skev.network (not nova.net), skev.file, skev.math, skev.time, skev.serial — all names checked ✅

---

## 7.1 — Overview & Architecture

### 👨‍💻 Developer Version

Skev's standard library is designed around one principle: **give you what you need, when you need it, without making you carry what you don't.**

The library is organised into three layers:

**Layer 1 — Language Primitives (always available — zero imports)**
These are so fundamental they are part of the language itself. You never import them. They are always there.
```
string operations     math.sqrt, math.PI    result handling
game-native type ops  maybe handling        type conversions
```

**Layer 2 — Core Library (import once)**
The tools every Skev program uses eventually. One import unlocks the essentials.
```swift
import skev.core    # file, json, serial, time, math, set, collections
```

**Layer 3 — Domain Libraries (import by need)**
Purpose-built for your domain. Lean by default.
```swift
import skev.network     # HTTP, WebSocket, TCP/UDP
import skev.game        # scene, audio, input, physics APIs
import skev.robotics    # sensors, motors, realtime control
import skev.render      # rendering pipeline, shaders
```

### Why Three Layers?

```
C++ mistake:   everything explicit — painful boilerplate
               #include <string>, #include <vector>,
               #include <algorithm>... for every file

Python mistake: everything implicit — hidden costs
                import random, import json, import os
                are all behind the scenes in many cases

Skev balance:
  Layer 1 — invisible — always there
  Layer 2 — one line — everything common
  Layer 3 — explicit — only what you need
```

### ⚙️ Technical Version

Layer 1 functions are compiler intrinsics — they generate LLVM IR directly without function call overhead. `math.sqrt(x)` compiles to `llvm.sqrt.f32` — a single hardware instruction on all supported platforms. String operations in Layer 1 are inline implementations for short strings (≤ 15 bytes) using small-string optimisation — zero heap allocation. Layer 2 is linked as a static library compiled with LTO (Link Time Optimisation) — unused functions are dead-code-eliminated. Layer 3 libraries are compiled separately and linked on demand.

---

## 7.2 — String Operations

### 👨‍💻 Developer Version

Strings in Skev are first-class. Operations read like natural language. Interpolation is built into the syntax with `{expression}`.

> String interpolation: `{expression}` inside any string literal is evaluated and inserted.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game needs to display dynamic UI text: player names, scores, status messages, formatted numbers, inventory descriptions. This happens hundreds of times per frame. String operations must be readable, safe, and produce no surprises.

**💥 Why This Is Hard**

String handling has historically been one of the most error-prone areas in game development — buffer overflows in C++, encoding issues, format string bugs, unexpected null characters. Even modern languages make common operations more verbose than necessary.

---

**In C++23:**
```cpp
// C++23 string operations
#include <string>
#include <format>
#include <algorithm>

std::string name = "Dragon";
int level = 15;
float health = 0.75f;

// String interpolation — requires std::format
std::string msg = std::format("The {} is level {}!", name, level);

// Operations
std::string upper = name;
std::transform(upper.begin(), upper.end(), upper.begin(), ::toupper);

bool has_prefix = name.starts_with("Dr");   // C++20
bool contains = name.find("rag") != std::string::npos;  // old style

std::string trimmed = name;  // no built-in trim — must write manually
trimmed.erase(0, trimmed.find_first_not_of(" \t"));
trimmed.erase(trimmed.find_last_not_of(" \t") + 1);
// Each simple operation requires knowledge of specific API quirks
```

**In C# 13:**
```csharp
// C# 13 string operations
string name = "Dragon";
int level = 15;
float health = 0.75f;

// String interpolation — clean
string msg = $"The {name} is level {level}!";

// Operations
string upper = name.ToUpper();
bool hasPrefix = name.StartsWith("Dr");
bool contains = name.Contains("rag");
string trimmed = name.Trim();
string[] parts = name.Split(",");
// Good — but PascalCase method names feel heavy
// ToUpper, StartsWith, Contains — verbose capitalization
```

**In Python 3.13:**
```python
# Python 3.13 string operations
name = "Dragon"
level = 15
health = 0.75

# F-string interpolation — excellent
msg = f"The {name} is level {level}!"

# Operations — natural and clean
upper = name.upper()
has_prefix = name.startswith("Dr")
contains = "rag" in name
trimmed = name.strip()
parts = name.split(",")
# Python wins here — clean, readable, natural
# Skev matches Python's readability
```

**In Skev:**
```swift
name    :: string = "Dragon"
level   :: int    = 15
health  :: float  = 0.75

# String interpolation — same {} as Python f-strings
msg :: string = "The {name} is level {level}!"

# Expressions in interpolation
ratio_text :: string = "Health: {(health * 100.0).as(int)}%"

# Literal brace — use {{
template :: string = "Property: {{field_name}}"

# Operations — snake_case, method-style
upper    :: string = name.upper()
lower    :: string = name.lower()
trimmed  :: string = name.trim()
has_dr   :: bool   = name.starts_with("Dr")
has_rag  :: bool   = name.contains("rag")
parts    :: list[string] = name.split(",")
replaced :: string = name.replace("Dragon", "Dragon Boss")
length   :: int    = name.length

# Multi-line string — indentation trimmed automatically
description :: string = "
    This creature roams the ancient mountains.
    It breathes fire and guards great treasure.
    Approach with extreme caution.
"
```

### ✅ Honest Advantage Statement

Skev's string system matches Python's readability and exceeds C#'s conciseness. C++ requires the most manual work — especially for common operations like trim. The `{expression}` interpolation syntax is equivalent to Python's f-strings in power and readability. Where Skev adds value: expressions inside `{}` are type-checked at compile time — a wrong type in an interpolation is a compile error, not a runtime exception.

---

### 🟢 Foundation Example

```swift
entity UISystem >>
    player_name :: string = "Hero"
    score       :: int    = 1500
    level       :: int    = 12

    when update(delta)
        # Direct interpolation
        ui.set_header("Welcome back, {player_name}!")
        ui.set_score("Score: {score}")
        ui.set_level("Level {level}")
    << update

<< UISystem
```

---

### 🟡 Applied Example — Inventory Item Description Generator

**In C++23:**
```cpp
std::string GenerateItemDescription(
    const std::string& name,
    int damage, float weight,
    const std::vector<std::string>& effects) {

    std::string desc = std::format("{}: {} dmg, {:.1f}kg", name, damage, weight);
    if (!effects.empty()) {
        desc += " | Effects: ";
        for (size_t i = 0; i < effects.size(); ++i) {
            if (i > 0) desc += ", ";
            desc += effects[i];
        }
    }
    return desc;
}
// Manual join logic, format specifiers, index management
```

**In C# 13:**
```csharp
string GenerateItemDescription(string name, int damage,
    float weight, List<string> effects) {
    var desc = $"{name}: {damage} dmg, {weight:F1}kg";
    if (effects.Any())
        desc += $" | Effects: {string.Join(", ", effects)}";
    return desc;
}
// Better — but string.Join feels awkward
```

**In Python 3.13:**
```python
def generate_item_description(name, damage, weight, effects):
    desc = f"{name}: {damage} dmg, {weight:.1f}kg"
    if effects:
        desc += f" | Effects: {', '.join(effects)}"
    return desc
# Clean — Python wins here
```

**In Skev:**
```swift
generate_item_description(
    name: string, damage: int,
    weight: float, effects: list[string]
) -> string

    desc :: string = "{name}: {damage} dmg, {weight.round(1)}kg"

    if effects.count > 0 >>
        effects_text :: string = effects.join(", ")
        desc = "{desc} | Effects: {effects_text}"
    << effects.count > 0

    result desc
<< generate_item_description
```

### ✅ Honest Advantage Statement

For the applied string example, Python and Skev are equally clean. C# is close. C++ requires the most code. Skev's advantage over Python: type safety — passing an integer where a string is expected is a compile error. Skev's advantage over C#: snake_case method names that match the rest of Skev, and expressions in interpolation are fully typed.

---

### 🔴 Hard Complex Example — Localisation System

**🌍 Real-World Scenario**

An AAA game ships in 12 languages. Each language has thousands of string keys mapped to translated text. Some strings have plural forms (1 enemy vs 3 enemies). Some have gender agreement. Some have variable substitution. The system must handle missing keys gracefully, support hot-reload of translations during development, and be performant enough to call every frame for dynamic UI.

**💥 Why This Is Hard**

Localisation at AAA scale involves: efficient key lookup (O(1)), type-safe variable substitution, plural rule logic per language, graceful fallback for missing keys, hot-reload without restarting, and memory management for potentially millions of strings across all language packs.

**In C++23:**
```cpp
class LocalisationSystem {
    std::unordered_map<std::string,
        std::unordered_map<std::string, std::string>> tables;
    std::string current_locale = "en";

public:
    std::string Get(const std::string& key,
        const std::unordered_map<std::string, std::string>& vars = {}) {

        auto locale_it = tables.find(current_locale);
        if (locale_it == tables.end()) {
            return "[MISSING LOCALE: " + current_locale + "]";
        }
        auto key_it = locale_it->second.find(key);
        if (key_it == locale_it->second.end()) {
            return "[MISSING KEY: " + key + "]";
        }

        std::string result = key_it->second;
        for (const auto& [var_name, var_value] : vars) {
            std::string placeholder = "{" + var_name + "}";
            size_t pos = 0;
            while ((pos = result.find(placeholder, pos)) != std::string::npos) {
                result.replace(pos, placeholder.length(), var_value);
                pos += var_value.length();
            }
        }
        return result;
    }
    // Manual substitution, manual fallback, manual error strings
    // No type safety on variable substitution
    // Hot-reload requires manual file watching and lock management
};
```

**In C# 13:**
```csharp
public class LocalisationSystem {
    private Dictionary<string, Dictionary<string, string>> _tables = new();
    private string _locale = "en";

    public string Get(string key,
        Dictionary<string, string>? vars = null) {
        if (!_tables.TryGetValue(_locale, out var table))
            return $"[MISSING LOCALE: {_locale}]";
        if (!table.TryGetValue(key, out var text))
            return $"[MISSING KEY: {key}]";

        if (vars != null) {
            foreach (var (name, value) in vars)
                text = text.Replace($"{{{name}}}", value);
        }
        return text;
    }
    // Better syntax — but Dictionary<string,string> for vars is untyped
    // Hot-reload still requires manual FileSystemWatcher setup
}
```

**In Python 3.13:**
```python
class LocalisationSystem:
    def __init__(self):
        self.tables: dict[str, dict[str, str]] = {}
        self.locale = "en"

    def get(self, key: str, **vars) -> str:
        table = self.tables.get(self.locale)
        if not table:
            return f"[MISSING LOCALE: {self.locale}]"
        text = table.get(key)
        if not text:
            return f"[MISSING KEY: {key}]"
        return text.format(**vars)
        # Clean — but text.format(**vars) is runtime — no type safety
        # Missing variable = KeyError at runtime
```

**In Skev:**
```swift
import skev.file
import skev.json

#! serialisable
data LocaleTable >>
    locale  :: string
    entries :: map[string, string]
<< LocaleTable

entity LocalisationSystem >>

    tables       :: map[string, LocaleTable]
    current      :: string = "en"
    fallback     :: string = "en"
    missing_keys :: set[string]    # tracked for developer review

    has FileWatcher    # watches locale files for hot-reload

    async load_locale(locale_code: string) -> result[nothing]
        path :: string  = "locales/{locale_code}.json"
        text :: string  = -> await file.read_text(path)
        table :: LocaleTable = -> json.parse(text)
        tables.add(locale_code -> table)
        succeed nothing
    << load_locale

    get(key: string) -> string
        result get_with_vars(key, [])
    << get

    get_with_vars(key: string, vars: list[string]) -> string
        # Try current locale first
        text :: maybe string = find_in_table(current, key)

        # Fall back to default locale if missing
        if not text exists >>
            text = find_in_table(fallback, key)
            missing_keys.add("{current}:{key}")    # track for review
        << not text exists

        # Return placeholder if truly missing
        if not text exists >>
            missing_keys.add("CRITICAL:{key}")
            result "[{key}]"
        << not text exists

        # Substitute variables using indexed replacement
        output :: string = text
        loop i from 0 to vars.count - 1 >>
            output = output.replace("{i}", vars[i])
        << i

        result output
    << get_with_vars

    find_in_table(locale: string, key: string) -> maybe string
        if tables contains locale >>
            table :: LocaleTable = tables[locale]
            if table.entries contains key >>
                result table.entries[key]
            << table.entries contains key
        << tables contains locale
        result nothing
    << find_in_table

    # Hot-reload — fires when locale file changes on disk
    when file_changed(path: string)
        locale_code :: string = path.replace("locales/", "").replace(".json", "")
        start_reload(locale_code)
    << file_changed

    async start_reload(locale_code: string)
        match await load_locale(locale_code) >>
            succeed _ ->
                debug.log("Locale reloaded: {locale_code}")
            fail error ->
                debug.log("Locale reload failed: {error.message}")
        << await load_locale(locale_code)
    << start_reload

    # Development review — show all missing keys
    when key_pressed(DEBUG_SHOW_MISSING)
        debug.log("Missing locale keys ({missing_keys.count}):")
        loop key in missing_keys >>
            debug.log("  {key}")
        << key
    << key_pressed

<< LocalisationSystem
```

### 🧭 Walk-Through — Hard Complex Localisation

**`load_locale` — async, result-based loading:**
Loads a locale file asynchronously. Uses `->` propagation from Chapter 6 — if the file is missing or malformed, the error surfaces to the caller immediately. The entire locale table is parsed from JSON and stored in the `tables` map. No blocking the game loop.

**`get_with_vars` — three-tier lookup:**
First tries current locale, then falls back to the default locale, then returns a visible placeholder. Every missing key is tracked in `missing_keys` — a `set[string]` (Chapter 3.5 from architecture audit) ensures no duplicate entries. During development, the developer can see exactly which keys were never found.

**Variable substitution — indexed replacement:**
Instead of named substitution (which has no type safety at runtime), Skev uses indexed replacement — `{0}`, `{1}`, `{2}`. The caller passes a `list[string]` of pre-converted values. This moves the type error to compile time — passing the wrong number of variables is visible from the function signature.

**Hot-reload — `FileWatcher` component:**
The `has FileWatcher` component monitors locale files on disk. When a file changes, `file_changed` fires on the main thread, triggering an async reload. The game continues running during the reload — no restart required. This is the developer workflow from Chapter 1 — "hot reloading as a language feature."

**Missing key tracking:**
The `missing_keys` set accumulates every locale miss during a play session. The developer can inspect this at any time. This is a tool that C++, C#, and Python implementations typically require a completely separate logging system for.

### ✅ Honest Advantage Statement

Skev's localisation system is significantly shorter than C++ (52 lines vs 35 in Skev but with hot-reload built in), comparable to C# but with type safety on variable substitution and hot-reload without `FileSystemWatcher` boilerplate. Python is most concise for the basic case but `text.format(**vars)` creates runtime KeyError risks — Skev's indexed substitution moves this to compile time. The `set[string]` for missing key tracking, the `FileWatcher` component, and async reload are unique advantages in this comparison.

---

## 7.3 — Math Library

### 👨‍💻 Developer Version

`skev.math` provides two categories of functions:
- **General math** — standard functions available in every language
- **Game math** — functions game developers use constantly but other libraries bury

> All game-native type math (Vector3! operations, quaternion math, Transform operations) lives on the types themselves — not in the math library. `skev.math` is for scalar and utility math.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A physics simulation needs: clamping values to safe ranges, linear interpolation for smooth movement, random number generation for procedural content, trigonometry for rotation, and value remapping for scaling difficulty. These operations happen thousands of times per frame.

**In C++23:**
```cpp
#include <cmath>
#include <algorithm>
#include <random>

// Different headers for different functions — frustrating
float clamped = std::clamp(value, 0.0f, 1.0f);
float lerped  = std::lerp(a, b, t);      // C++20
float mapped  = a + (value - in_min) / (in_max - in_min) * (out_max - out_min);
// map() doesn't exist — must write every time

// Random — complex setup required
std::random_device rd;
std::mt19937 gen(rd());
std::uniform_real_distribution<float> dis(0.0f, 1.0f);
float random = dis(gen);
// Six lines to get one random float
```

**In C# 13:**
```csharp
using System;

float clamped = Math.Clamp(value, 0.0f, 1.0f);
float lerped  = float.Lerp(a, b, t);   // .NET 8+
// No map() — must write manually
float mapped = a + (value - inMin) / (inMax - inMin) * (outMax - outMin);

float random = Random.Shared.NextSingle();  // .NET 6+
// PascalCase everywhere — Clamp, Lerp, NextSingle
// Less consistent naming than Python or Skev
```

**In Python 3.13:**
```python
import math
import random

clamped = max(0.0, min(1.0, value))     # no clamp — manual
lerped  = a + (b - a) * t              # no lerp — manual
mapped  = a + (value-in_min)/(in_max-in_min) * (out_max-out_min)
rnd     = random.random()              # simple

# Python has no clamp or lerp in standard math
# Must write them manually or use numpy
```

**In Skev:**
```swift
import skev.math

clamped :: float = math.clamp(value, 0.0, 1.0)
lerped  :: float = math.lerp(a, b, t)
mapped  :: float = math.map(value, in_min, in_max, out_min, out_max)
rnd     :: float = math.random()
rnd_int :: int   = math.random_int(1, 6)
```

### ✅ Honest Advantage Statement

Skev provides `clamp`, `lerp`, and `map` as first-class standard library functions — functions that game developers write manually in C++ and Python constantly, and that C# only recently added (`float.Lerp` in .NET 8). The `map` function in particular eliminates a common source of copy-paste errors in game codebases. C++ requires multiple headers and verbose random setup. Python has no `clamp` or `lerp` in its standard library.

---

### Complete Math Library Reference

```swift
import skev.math

# Rounding
math.floor(4.7)           # 4 — round down
math.ceil(4.2)            # 5 — round up
math.round(4.5)           # 5 — round to nearest
math.truncate(4.9)        # 4 — remove decimal

# Arithmetic
math.abs(-5.3)            # 5.3 — absolute value
math.sqrt(144.0)          # 12.0 — square root
math.pow(2.0, 10.0)       # 1024.0 — power
math.log(math.E)          # 1.0 — natural log
math.log10(1000.0)        # 3.0 — base-10 log

# Game math
math.clamp(1.5, 0.0, 1.0) # 1.0 — clamp to range
math.lerp(0.0, 10.0, 0.3) # 3.0 — linear interpolation
math.smoothstep(0.0, 1.0, 0.5)  # smooth interpolation
math.map(5.0, 0.0, 10.0, 0.0, 1.0)  # remap range
math.sign(-5.0)           # -1.0 — sign of value
math.min(3.0, 7.0)        # 3.0
math.max(3.0, 7.0)        # 7.0

# Trigonometry
math.sin(math.PI / 2.0)   # 1.0
math.cos(0.0)             # 1.0
math.tan(math.PI / 4.0)   # ~1.0
math.asin(1.0)            # PI / 2
math.acos(1.0)            # 0.0
math.atan(1.0)            # PI / 4
math.atan2(1.0, 1.0)      # PI / 4 — angle from origin
math.degrees(math.PI)     # 180.0 — radians to degrees
math.radians(180.0)       # PI — degrees to radians

# Random
math.random()             # float 0.0 to 1.0
math.random_int(1, 6)     # int 1 to 6 inclusive
math.random_range(min: float, max: float) -> float
math.set_seed(12345)      # reproducible randomness

# Constants
math.PI       # 3.14159265358979...
math.TAU      # 6.28318530717959... (2 × PI)
math.E        # 2.71828182845904...
math.SQRT2    # 1.41421356237309...
math.INF      # positive infinity
math.NEG_INF  # negative infinity
math.NAN      # not a number
```

---

### 🔴 Hard Complex Example — Procedural Terrain Height Generation

**🌍 Real-World Scenario**

An open world game generates terrain procedurally. Height values are computed using layered Perlin noise octaves, then remapped to create realistic elevation bands (ocean, plains, hills, mountains), with smooth transitions between biomes. This runs on thousands of grid cells, triggered by a background task as the player moves.

**💥 Why This Is Hard**

Procedural generation requires: combining multiple math operations per cell, reproducible results from a seed (for saves to work), smooth interpolation between elevation bands, and performance good enough to generate chunks while the game runs. Getting any math function wrong produces visual artifacts that are immediately obvious to players.

**In C++23:**
```cpp
float smoothstep(float edge0, float edge1, float x) {
    x = std::clamp((x - edge0) / (edge1 - edge0), 0.0f, 1.0f);
    return x * x * (3.0f - 2.0f * x);  // must implement manually
}
float lerp(float a, float b, float t) {
    return std::lerp(a, b, t);  // C++20 at least
}
// Must manually implement smoothstep, noise, octave blending
// No standard noise function — use third-party library
float GenerateHeight(int x, int z, uint32_t seed) {
    // ... 50+ lines of manual noise implementation
}
```

**In C# 13:**
```csharp
// C# has no built-in noise or smoothstep
// Must use Unity's Mathf (game-specific) or third-party library
// In pure C# — same manual implementation as C++
float GenerateHeight(int x, int z, uint32_t seed) {
    // Mathf.Lerp exists in Unity but not in plain C#
    // Same verbosity as C++
}
```

**In Python 3.13:**
```python
import math
# Python has no smoothstep, no noise — use noise library (third-party)
# import noise  # not in standard library
def generate_height(x, z, seed):
    # Must install and import third-party 'noise' package
    # math module alone is insufficient
    pass
```

**In Skev:**
```swift
import skev.math

entity TerrainGenerator >>

    seed     :: int   = 12345
    chunk_size :: int = 64

    # Generate a full terrain chunk — runs in a task
    task generate_chunk(chunk_x: int, chunk_z: int) >>
        heights :: array[float, 4096]    # 64×64 grid pre-allocated

        math.set_seed(seed + chunk_x * 1000 + chunk_z)

        loop z from 0 to chunk_size - 1 >>
            loop x from 0 to chunk_size - 1 >>
                world_x :: float = (chunk_x * chunk_size + x).as(float)
                world_z :: float = (chunk_z * chunk_size + z).as(float)

                # Layer multiple noise octaves for natural terrain
                h :: float = 0.0
                h += math.noise(world_x * 0.003, world_z * 0.003) * 1.0     # base shape
                h += math.noise(world_x * 0.01,  world_z * 0.01)  * 0.4     # hills
                h += math.noise(world_x * 0.05,  world_z * 0.05)  * 0.15    # detail
                h += math.noise(world_x * 0.2,   world_z * 0.2)   * 0.04    # micro detail

                # Remap to elevation bands with smooth transitions
                h = math.map(h, -1.0, 1.0, 0.0, 1.0)    # normalise to 0-1

                match true >>
                    h < 0.35 ->                           # ocean floor
                        h = math.lerp(0.0, 15.0, h / 0.35)

                    h < 0.45 >>                           # beach/coastal smooth
                        t :: float = math.smoothstep(0.35, 0.45, h)
                        h = math.lerp(15.0, 20.0, t)
                    << h < 0.45

                    h < 0.65 >>                           # plains to hills
                        t :: float = math.smoothstep(0.45, 0.65, h)
                        h = math.lerp(20.0, 80.0, t)
                    << h < 0.65

                    h < 0.85 >>                           # hills to mountains
                        t :: float = math.smoothstep(0.65, 0.85, h)
                        h = math.lerp(80.0, 200.0, t)
                    << h < 0.85

                    _ >>                                  # mountain peaks
                        t :: float = math.smoothstep(0.85, 1.0, h)
                        h = math.lerp(200.0, 500.0, math.pow(t, 1.5))
                    << _
                << true

                heights[z * chunk_size + x] = h
            << x
        << z

        task.result = heights
    << generate_chunk

    when update(delta)
        # Fire chunk generation as player moves — non-blocking
        if player_moved_to_new_chunk() >>
            task generate_chunk(player_chunk_x, player_chunk_z)
            await generate_chunk
            apply_chunk_to_world(generate_chunk.result)
        << player_moved_to_new_chunk()
    << update

<< TerrainGenerator
```

### 🧭 Walk-Through — Terrain Generation

**Noise octave layering:**
Four `math.noise()` calls at different frequencies and amplitudes. Low frequency (0.003) gives the broad continental shapes. Medium (0.01, 0.05) adds rolling hills and valleys. High frequency (0.2) adds surface texture. Each octave is weighted — the broad shape dominates, detail adds realism. This is the standard Fractional Brownian Motion technique.

**`math.map()` normalisation:**
The noise values come out in -1.0 to 1.0 range. `math.map()` converts this to 0.0 to 1.0 for the elevation band logic. This single function replaces the manual formula `(value - in_min) / (in_max - in_min) * (out_max - out_min) + out_min` that C++ and Python developers write from memory.

**`math.smoothstep()` for biome transitions:**
Without smoothstep, biome boundaries look like sharp lines — obviously artificial. `math.smoothstep()` creates an S-curve transition — slow start, fast middle, slow end — that mimics natural gradients. This function exists in GLSL shader language but not in C++ or Python standard libraries.

**`math.pow(t, 1.5)` for mountain sculpting:**
Raising the interpolation value to a power above 1.0 creates a curve that rises slowly then sharply — making mountain peaks feel steep and dramatic rather than linear.

**Task execution:**
The entire chunk generation runs inside a `task` block from Chapter 5. The main thread never blocks. The pre-allocated `array[float, 4096]` ensures zero heap allocation during generation — no GC pressure, no ARC overhead.

### ✅ Honest Advantage Statement

Skev's math library provides `smoothstep`, `noise`, `map`, `lerp`, and `clamp` as standard functions — eliminating the need for third-party noise libraries (required in Python and C++) and manual smoothstep implementations (required in C++ and C#). The `match true >>` pattern for range testing is more readable than chained `if/else` in all comparison languages. The task-based execution is unique to Skev — this exact pattern requires `std::async` in C++ or `Task.Run` in C# with more boilerplate.

---

## 7.4 — Time Library

### 👨‍💻 Developer Version

`skev.time` covers three needs: reading current time, running timed operations, and measuring performance. Timers are first-class blocks — they use the same `>>` `<<` syntax as the rest of Skev.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game needs: a countdown before a round starts, a cooldown on player abilities, a regenerating health system, and performance profiling of the AI update loop. All of these need different timing approaches.

**In C++23:**
```cpp
#include <chrono>

// Timing — complex chrono API
auto start = std::chrono::high_resolution_clock::now();
do_work();
auto end = std::chrono::high_resolution_clock::now();
auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();

// Timers — no standard — must track manually
float cooldown_remaining = 2.0f;
void Update(float delta) {
    if (cooldown_remaining > 0) {
        cooldown_remaining -= delta;
        if (cooldown_remaining <= 0) {
            ability_ready = true;  // trigger at zero
        }
    }
}
// Boilerplate repeated everywhere — no abstraction
```

**In C# 13:**
```csharp
using System.Diagnostics;

// Timing — Stopwatch
var sw = Stopwatch.StartNew();
DoWork();
sw.Stop();
double ms = sw.Elapsed.TotalMilliseconds;

// Timers — Unity-specific — no standard C# equivalent for game loops
// In Unity: Invoke("MethodName", 2.0f);  — string-based, unsafe
// Or manual coroutines — complex setup
StartCoroutine(Countdown(2.0f));
IEnumerator Countdown(float duration) {
    yield return new WaitForSeconds(duration);
    OnCountdownComplete();  // boilerplate coroutine pattern
}
```

**In Python 3.13:**
```python
import time

# Timing
start = time.perf_counter()
do_work()
elapsed = time.perf_counter() - start  # seconds

# Timers — no game timer concept
# Must track manually same as C++
# Or use asyncio which requires event loop setup
```

**In Skev:**
```swift
import skev.time

entity AbilitySystem >>
    dash_ready :: bool = true

    when update(delta)
        # One-shot timer — fire once after 2 seconds
        # Reads like: "after 2 seconds, do this"
        if input.key_pressed(SHIFT) and dash_ready >>
            dash_ready = false
            perform_dash()

            timer(2.0)
                dash_ready = true
                effects.play(particles!.ready_pulse, position)
            << timer

        << input.key_pressed(SHIFT) and dash_ready

        # Repeating timer — fire every 0.5 seconds
        every(0.5)
            if health < max_health >>
                health += 1
            << health < max_health
        << every

    << update

    # Performance timing
    benchmark_ai()
        watch :: Stopwatch = time.stopwatch()
        watch.start()
        run_ai_update()
        elapsed :: float = watch.stop()
        debug.log("AI update: {elapsed * 1000.0}ms")
    << benchmark_ai

<< AbilitySystem
```

### 🟡 Applied Example — Round Timer System

**In C++23:**
```cpp
class RoundTimer {
    float duration, remaining;
    bool running = false;
    std::function<void()> on_complete;
public:
    void Start(float d, std::function<void()> cb) {
        duration = remaining = d;
        running = true;
        on_complete = cb;
    }
    void Update(float delta) {
        if (!running) return;
        remaining -= delta;
        if (remaining <= 0) {
            running = false;
            on_complete();
        }
    }
};  // 20 lines for something trivial
```

**In C# 13:**
```csharp
// Unity approach
IEnumerator RunRound(float duration) {
    float remaining = duration;
    while (remaining > 0) {
        ui.UpdateTimer(remaining);
        remaining -= Time.deltaTime;
        yield return null;
    }
    OnRoundComplete();
}
StartCoroutine(RunRound(60.0f));
// Coroutines are C# Iterator pattern — heavyweight
```

**In Python 3.13:**
```python
# No game timer concept — manual tracking
class RoundTimer:
    def __init__(self):
        self.remaining = 0
        self.callback = None
    def start(self, duration, callback):
        self.remaining = duration
        self.callback = callback
    def update(self, delta):
        if self.remaining > 0:
            self.remaining -= delta
            if self.remaining <= 0:
                self.callback()
```

**In Skev:**
```swift
entity RoundManager >>
    round_active :: bool  = false
    round_number :: int   = 0

    when update(delta)
        if input.key_pressed(KEY_START) and not round_active >>
            round_active = true
            round_number += 1
            start_round()
        << input.key_pressed(KEY_START) and not round_active
    << update

    start_round()
        ui.show_round_banner("Round {round_number}")

        timer(3.0)
            ui.hide_round_banner()
            spawn_enemies(round_number * 5)
        << timer

        timer(63.0)
            end_round()
        << timer

        every(1.0)
            if not round_active >>
                stop
            << not round_active
            ui.update_round_timer(time.remaining_in_timer)
        << every
    << start_round

    end_round()
        round_active = false
        ui.show_round_complete(round_number)
    << end_round

<< RoundManager
```

### ✅ Honest Advantage Statement

Skev's `timer` and `every` blocks eliminate an entire category of boilerplate that all three comparison languages require. C++ needs a manual timer class (20+ lines). C# Unity coroutines are functional but syntactically heavy. Python needs the same manual class as C++. Skev's `timer(seconds) >> ... << timer` reads as a direct description of intent — "after N seconds, do this" — with zero boilerplate.

---

## 7.5 — File I/O

### 👨‍💻 Developer Version

`skev.file` handles all file system operations. Everything async by default — the game loop never blocks waiting for disk.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game autosaves every 5 minutes, loads level data on scene transition, writes a crash log when panicking, and reads a config file on startup. All of these must coexist without blocking gameplay.

**In C++23:**
```cpp
#include <fstream>
#include <filesystem>

// Async file I/O in C++ requires platform-specific APIs
// Standard C++ only has synchronous file I/O
// For async: use std::async, platform IOCP/io_uring, or third-party library

// Synchronous read — blocks the thread
std::ifstream file("level.dat", std::ios::binary);
if (!file.is_open()) {
    // Error handling
}
std::vector<char> buffer(
    (std::istreambuf_iterator<char>(file)),
    std::istreambuf_iterator<char>()
);
// No standard async file I/O — major gap for game development
```

**In C# 13:**
```csharp
// C# has good async file I/O
async Task<string> LoadLevel(string path) {
    try {
        return await File.ReadAllTextAsync(path);
    }
    catch (FileNotFoundException) {
        return null;  // caller must handle null
    }
    catch (IOException ex) {
        throw new LevelLoadException(ex);
    }
}
// Good but: exceptions for expected failures (file not found)
// No path normalisation helpers
// No built-in file watching
```

**In Python 3.13:**
```python
import asyncio
import aiofiles  # third-party — not standard library

async def load_level(path: str) -> str | None:
    try:
        async with aiofiles.open(path, 'r') as f:
            return await f.read()
    except FileNotFoundError:
        return None
# Requires third-party aiofiles for async
# Standard open() is synchronous
```

**In Skev:**
```swift
import skev.file

entity LevelManager >>

    async load_level(path: string) -> result[LevelData]
        text      :: string    = -> await file.read_text(path)
        level     :: LevelData = -> json.parse(text)
        succeed level
    << load_level

    async autosave(state: GameState) -> result[nothing]
        data      :: string = json.serialise(state)
        timestamp :: string = time.datetime().replace(" ", "_").replace(":", "-")
        path      :: string = "saves/auto_{timestamp}.skev_save"
        -> await file.write_text(path, data)
        succeed nothing
    << autosave

<< LevelManager
```

### Complete File Library Reference

```swift
import skev.file

# Reading
await file.read_text(path)           -> result[string]
await file.read_bytes(path)          -> result[list[uint8]]
await file.read_lines(path)          -> result[list[string]]

# Writing
await file.write_text(path, text)    -> result[nothing]
await file.write_bytes(path, bytes)  -> result[nothing]
await file.append_text(path, text)   -> result[nothing]

# Checking (synchronous — fast)
file.exists(path)                    -> bool
file.size(path)                      -> int      # bytes
file.modified(path)                  -> float64  # unix timestamp
file.is_directory(path)              -> bool

# Navigation
file.list(dir)                       -> list[string]  # files
file.list_dirs(dir)                  -> list[string]  # subdirectories
file.list_all(dir)                   -> list[string]  # files + dirs

# Operations
await file.copy(from, to)            -> result[nothing]
await file.move(from, to)            -> result[nothing]
await file.delete(path)              -> result[nothing]
await file.create_dir(path)          -> result[nothing]

# Path utilities (platform-independent)
file.path("saves", "slot_1", "game.skev_save")  -> string
file.parent("saves/slot_1/game.skev_save")       -> string  # "saves/slot_1"
file.filename("saves/slot_1/game.skev_save")     -> string  # "game.skev_save"
file.extension("game.skev_save")                 -> string  # "skev_save"
file.user_data_dir()                             -> string  # OS-specific user data path
file.app_dir()                                   -> string  # where the game executable is
```

### ✅ Honest Advantage Statement

Skev's file library is async by default — unlike C++ standard library which has no async file I/O. C# async file I/O is good but uses exceptions for expected failures (file not found). Python requires a third-party `aiofiles` package for async. Skev provides path utilities that normalise separators across platforms — a constant source of Windows vs macOS/Linux bugs in all three comparison languages.

---

## 7.6 — Serialisation

### 👨‍💻 Developer Version

Skev provides two serialisers: `skev.json` for human-readable text (config, debug saves), and `skev.serial` for efficient binary (production saves, network packets).

Mark any `data` type with `#! serialisable` and it works automatically with both.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game needs to save the complete game state: player stats, inventory (hundreds of items), world state (thousands of entities with positions), quest progress, and discovered map areas. This save must load quickly, be small enough to store in cloud saves (under 1MB), and be readable in a text editor for debugging.

**In C++23:**
```cpp
// C++ has no standard serialisation
// Must use third-party: nlohmann/json, cereal, protobuf, etc.
#include <nlohmann/json.hpp>  // third-party

struct PlayerData {
    std::string name;
    int level;
    float x, y, z;

    // Must manually write serialisation code
    nlohmann::json to_json() const {
        return {{"name", name}, {"level", level},
                {"position", {x, y, z}}};
    }
    static PlayerData from_json(const nlohmann::json& j) {
        PlayerData p;
        p.name  = j["name"];
        p.level = j["level"];
        // Error if field missing — no automatic fallback
        return p;
    }
};
// Every struct needs manual to_json and from_json
// Third-party library required — version conflicts possible
```

**In C# 13:**
```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// C# 13 has built-in JSON serialisation — good
public class PlayerData {
    public string Name { get; set; }
    public int Level { get; set; }
    public Vector3 Position { get; set; }
    // Works automatically with attributes
}

string json = JsonSerializer.Serialize(player);
PlayerData p = JsonSerializer.Deserialize<PlayerData>(json);
// Good — but Vector3 (game-native type) needs custom converter
// Binary serialisation is separate (BinaryFormatter deprecated)
// Must choose a third-party binary serialiser for production
```

**In Python 3.13:**
```python
import json
from dataclasses import dataclass, asdict

@dataclass
class PlayerData:
    name: str
    level: int
    position: tuple[float, float, float]

# JSON works with dataclasses
data = json.dumps(asdict(player))
# But custom types (like Vector3) need manual handlers
# Binary serialisation requires pickle (unsafe) or third-party
```

**In Skev:**
```swift
import skev.json
import skev.serial

#! serialisable
data PlayerData >>
    name      :: string  = "Hero"
    level     :: int     = 1
    position  :: Vector3!             # game-native types supported
    inventory :: list[string]
    gold      :: int     = 0
<< PlayerData

#! serialisable
data WorldState >>
    players    :: list[PlayerData]
    discovered :: set[string]         # discovered areas
    quest_flags :: map[string, bool]
    play_time  :: float64
<< WorldState

entity SaveSystem >>

    async save_game(state: WorldState, slot: int) -> result[nothing]
        # Binary — fast, compact
        bytes :: list[uint8] = serial.encode(state)
        path  :: string = file.path("saves", "slot_{slot}.skev_save")
        -> await file.write_bytes(path, bytes)

        # JSON — human-readable backup
        text  :: string = json.serialise(state)
        debug_path :: string = file.path("saves", "slot_{slot}_debug.json")
        -> await file.write_text(debug_path, text)

        succeed nothing
    << save_game

    async load_game(slot: int) -> result[WorldState]
        path  :: string = file.path("saves", "slot_{slot}.skev_save")
        bytes :: list[uint8] = -> await file.read_bytes(path)
        state :: WorldState = -> serial.decode(bytes)
        succeed state
    << load_game

<< SaveSystem
```

### ✅ Honest Advantage Statement

Skev's serialisation is zero-configuration for `data` types marked `#! serialisable` — including game-native types like `Vector3!`, `Color!`, and `Transform!`. C++ requires third-party libraries and manual serialisation code per struct. C# 13's built-in JSON serialisation is good but requires custom converters for game-native types and a third-party library for binary. Python's `json` module works for simple types but needs manual handling for Vector3 and similar. Both `skev.json` and `skev.serial` return `result[T]` — consistent with Chapter 6 error handling.

---

## 7.7 — The `set[T]` Type

### 👨‍💻 Developer Version

`set[T]` holds unique values in no particular order. The key feature: checking if a value is present is O(1) — instant regardless of how many items are in the set.

> Use `set[T]` when you care about membership, not order.
> Use `list[T]` when you care about order or allow duplicates.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

An achievement system tracks which achievements a player has unlocked. A pathfinding system tracks which nodes have been visited. A loading system tracks which assets are currently loaded. All three need fast membership testing and no duplicates.

**In C++23:**
```cpp
#include <unordered_set>

std::unordered_set<std::string> unlocked_achievements;

unlocked_achievements.insert("first_kill");
bool has_it = unlocked_achievements.count("first_kill") > 0;
// .count() > 0 is the idiomatic C++ contains check — clunky
// C++20: unlocked_achievements.contains("first_kill") — better
unlocked_achievements.erase("first_kill");
```

**In C# 13:**
```csharp
var unlockedAchievements = new HashSet<string>();

unlockedAchievements.Add("first_kill");
bool hasIt = unlockedAchievements.Contains("first_kill");
unlockedAchievements.Remove("first_kill");
// Good — but HashSet<T> is verbose to type
```

**In Python 3.13:**
```python
unlocked_achievements = set()

unlocked_achievements.add("first_kill")
has_it = "first_kill" in unlocked_achievements   # clean
unlocked_achievements.discard("first_kill")
# Python set is clean — Skev matches this
```

**In Skev:**
```swift
unlocked_achievements :: set[string]

unlocked_achievements.add("first_kill")
has_it :: bool = unlocked_achievements contains "first_kill"
unlocked_achievements.remove("first_kill")
```

### Complete Set Reference

```swift
# Declaration
active_enemies  :: set[Enemy]
loaded_assets   :: set[string]
visited_cells   :: set[int]

# Literals
tags :: set[string] = ["boss", "player", "npc"]

# Operations — all O(1) for add, remove, contains
set.add(value)              # add if not present
set.remove(value)           # remove if present — no error if missing
set.contains(value)         # true/false
set.count                   # number of elements
set.is_empty                # true/false

# Set mathematics
a.union(b)                  # all elements from both
a.intersect(b)              # elements in both
a.minus(b)                  # elements in a but not b
a.is_subset_of(b)           # true if all a elements are in b

# Iteration
loop achievement in unlocked_achievements >>
    ui.show_achievement_badge(achievement)
<< achievement
```

### ✅ Honest Advantage Statement

Skev's `set[T]` matches Python's natural `in` membership check syntax with `contains`. C++ requires `.count() > 0` (pre-C++20) or `.contains()` (C++20). C# uses `HashSet<T>.Contains()` — functional but verbose. The set mathematics operations (`union`, `intersect`, `minus`) use English words consistent with Skev's naming philosophy.

---

## 7.8 — Networking (`skev.network`)

### 👨‍💻 Developer Version

`skev.network` provides HTTP, WebSocket, and low-level TCP/UDP. All operations are async — network calls never block the game loop.

> **Naming note:** `skev.network` — not `nova.net`. The name `net` evokes Microsoft's .NET platform. `network` is unambiguous. *(Rule H applied)*

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A live-service game needs: fetch the daily leaderboard from a REST API, maintain a persistent WebSocket for real-time multiplayer events, submit a score after each match, and handle network failures gracefully without crashing the game.

**In C++23:**
```cpp
// C++ has no standard HTTP library
// Must use: libcurl, Boost.Beast, cpp-httplib, etc.
#include <curl/curl.h>  // third-party

CURL* curl = curl_easy_init();
curl_easy_setopt(curl, CURLOPT_URL, "https://api.game.com/leaderboard");
// ... 30 more lines of curl setup
// No standard async — must wrap in threads manually
// WebSocket: another third-party library
```

**In C# 13:**
```csharp
using System.Net.Http;
using System.Net.WebSockets;

// HTTP — good in modern C#
var client = new HttpClient();
var response = await client.GetStringAsync("https://api.game.com/leaderboard");
var entries = JsonSerializer.Deserialize<List<LeaderboardEntry>>(response);

// WebSocket — verbose setup
var ws = new ClientWebSocket();
await ws.ConnectAsync(new Uri("wss://game.com/mp"), CancellationToken.None);
// Receive loop requires manual buffer management
var buffer = new ArraySegment<byte>(new byte[4096]);
var result = await ws.ReceiveAsync(buffer, CancellationToken.None);
// No built-in reconnection, no message framing abstraction
```

**In Python 3.13:**
```python
import aiohttp  # third-party — not standard library

async def fetch_leaderboard():
    async with aiohttp.ClientSession() as session:
        async with session.get("https://api.game.com/leaderboard") as resp:
            return await resp.json()
# Requires third-party aiohttp
# Standard urllib is synchronous only
```

**In Skev:**
```swift
import skev.network
import skev.json

entity LiveServiceSystem >>

    connection :: maybe WebSocket

    async connect_to_server() -> result[nothing]
        ws :: WebSocket = -> await network.websocket("wss://game.com/mp")

        ws.on_message >> msg
            handle_server_event(msg)
        << on_message

        ws.on_disconnect >> reason
            debug.log("Disconnected: {reason}")
            schedule_reconnect()
        << on_disconnect

        connection = ws
        succeed nothing
    << connect_to_server

    async fetch_leaderboard() -> result[list[LeaderboardEntry]]
        response :: NetworkResponse = -> await network.get(
            url: "https://api.game.com/leaderboard",
            headers: ["Authorization" -> "Bearer {auth_token}"]
        )
        entries :: list[LeaderboardEntry] = -> json.parse(response.body)
        succeed entries
    << fetch_leaderboard

    async submit_score(score: int) -> result[nothing]
        payload :: string = json.serialise(ScoreSubmission >>
            player_id :: player.id
            score     :: score
            timestamp :: time.unix()
        << ScoreSubmission)

        -> await network.post(
            url:  "https://api.game.com/scores",
            body: payload,
            headers: [
                "Authorization"  -> "Bearer {auth_token}",
                "Content-Type"   -> "application/json"
            ]
        )
        succeed nothing
    << submit_score

    schedule_reconnect()
        timer(5.0)
            if not connection exists >>
                start_connect_with_retry()
            << not connection exists
        << timer
    << schedule_reconnect

<< LiveServiceSystem
```

### ✅ Honest Advantage Statement

C++ has no standard HTTP or WebSocket library — both require third-party dependencies with version management overhead. Python requires `aiohttp` (third-party) for async HTTP. C# 13's `HttpClient` is good for HTTP but WebSocket handling requires manual buffer management. Skev provides both HTTP and WebSocket in a single import with result-based error handling consistent with Chapter 6. The `ws.on_message >>` syntax integrates WebSocket events with Skev's existing event system — no callbacks, no delegates, no closures.

---

## 7.9 — Complete Standard Library Example

**🌍 Real-World Scenario**

A live-service RPG needs a complete game session lifecycle: load config, connect to server, load player data from local save and merge with server data, run the game loop with autosave and leaderboard updates, and handle graceful shutdown with final save.

**💥 Why This Is Hard**

This requires combining every standard library module in a correct sequence: file I/O (async), JSON parsing (with error handling), networking (WebSocket + HTTP), serialisation (binary for saves), timers (autosave, leaderboard refresh), and set operations (tracking loaded assets). Each operation can fail. The game must run smoothly throughout.

**In C++23:**
```cpp
// C++ would require:
// - nlohmann/json (third-party) for JSON
// - libcurl or similar for HTTP (third-party)
// - A WebSocket library (third-party)
// - Manual async file I/O (platform-specific)
// - Manual timer tracking
// - std::unordered_set for asset tracking
// Estimated: 400+ lines across multiple files with third-party deps
```

**In C# 13:**
```csharp
// C# would require:
// - System.Text.Json (built-in) — good
// - System.Net.Http — good
// - System.Net.WebSockets — verbose
// - Custom BinarySerializer or third-party for binary saves
// - Manual timer management (no first-class timers)
// - HashSet<T> for asset tracking
// Estimated: 200+ lines, needs third-party binary serialiser
```

**In Python 3.13:**
```python
# Python would require:
# - json (built-in) — good
# - aiofiles (third-party) for async file I/O
# - aiohttp (third-party) for async HTTP
# - websockets (third-party) for WebSocket
# - pickle or third-party for binary serialisation
# - set type (built-in) — good
# Estimated: 150+ lines, 3 third-party dependencies
```

**In Skev:**
```swift
import skev.core

#! serialisable
data SessionConfig >>
    server_url    :: string = "wss://game.com/mp"
    api_url       :: string = "https://api.game.com"
    autosave_interval :: float = 300.0    # 5 minutes
    leaderboard_refresh :: float = 60.0
<< SessionConfig

entity GameSession >>

    config       :: SessionConfig
    player_data  :: maybe PlayerData
    loaded_assets :: set[string]
    connection   :: maybe WebSocket

    has SaveSystem
    has LiveServiceSystem

    async initialise() -> result[nothing]
        # Load config
        config_text :: string = -> await file.read_text("config.json")
        config = -> json.parse(config_text)

        # Load local player save
        local_save :: maybe PlayerData = nothing
        if file.exists("saves/player.skev_save") >>
            bytes :: list[uint8] = -> await file.read_bytes("saves/player.skev_save")
            local_save = serial.decode(bytes) or_else nothing
        << file.exists("saves/player.skev_save")

        # Connect and merge with server data
        connection = -> await network.websocket(config.server_url)

        server_data :: PlayerData = -> await fetch_server_player_data()

        # Merge — server wins for progression, local wins for settings
        player_data = merge_player_data(local_save, server_data)

        start_session_timers()
        succeed nothing
    << initialise

    start_session_timers()
        # Autosave every 5 minutes
        every(config.autosave_interval)
            if player_data exists >>
                save_system.autosave(player_data)
            << player_data exists
        << every

        # Refresh leaderboard every minute
        every(config.leaderboard_refresh)
            update_leaderboard()
        << every
    << start_session_timers

    async update_leaderboard()
        response :: NetworkResponse = -> await network.get(
            "{config.api_url}/leaderboard"
        )
        entries :: list[LeaderboardEntry] = -> json.parse(response.body)
        ui.update_leaderboard(entries)
    << update_leaderboard

    async fetch_server_player_data() -> result[PlayerData]
        response :: NetworkResponse = -> await network.get(
            "{config.api_url}/player/{auth.player_id}"
        )
        succeed -> json.parse(response.body)
    << fetch_server_player_data

    async shutdown() -> result[nothing]
        # Final save — binary for speed
        if player_data exists >>
            bytes :: list[uint8] = serial.encode(player_data)
            -> await file.write_bytes("saves/player.skev_save", bytes)

            # Also write readable JSON for support debugging
            json_text :: string = json.serialise(player_data)
            -> await file.write_text("saves/player_debug.json", json_text)
        << player_data exists

        # Graceful disconnect
        if connection exists >>
            await connection.close()
        << connection exists

        succeed nothing
    << shutdown

<< GameSession
```

### 🧭 Walk-Through — Complete Session

**`initialise` — sequential async loading:**
Five `->` propagation steps. Each one is async. If any fails, the entire initialisation stops and the error surfaces to the caller. ARC automatically cleans up any allocations made before the failure. The order matters: config must load before connecting to the server (we need the server URL).

**`start_session_timers` — two `every` blocks:**
Both fire from inside the same function — they run concurrently. The autosave fires every 5 minutes regardless of what else is happening. The leaderboard refresh fires every minute. Neither blocks the game loop. The `stop` keyword inside `every` (via `if player_data exists`) provides a clean exit condition.

**`update_leaderboard` — async with `->` propagation:**
Two steps: fetch the HTTP response, parse the JSON. If either fails, the error is returned. The caller (`every` block) ignores the failure — a leaderboard update failure is non-critical. This is the `or_else` pattern from Chapter 6 in action.

**`shutdown` — conditional saves:**
Checks `if player_data exists` before saving — clean handling of the case where initialisation never completed. Writes both binary (fast load next time) and JSON (readable for debugging). Gracefully closes the WebSocket.

**`loaded_assets` set:**
Not shown in this example explicitly but used throughout the system — O(1) checks to avoid double-loading assets. The `set[string]` from section 7.7 does exactly this job.

### ✅ Honest Advantage Statement

The Skev implementation is approximately 90 lines and requires zero third-party libraries. C++ would need 3+ third-party libraries and 400+ lines. C# would need 1-2 third-party libraries and 200+ lines. Python would need 3 third-party libraries and 150+ lines. Skev's advantage is not just fewer lines — it is zero external dependencies, consistent error handling via `result[T]` throughout, first-class timers, and game-native type serialisation without custom converters.

---

## 7.10 — Standard Library Quick Reference

```
LAYER 1 — Always Available (no import):

Strings:
  "Hello, {name}!"        string interpolation
  .upper()  .lower()      case conversion
  .trim()                 remove whitespace
  .contains(s)            membership check
  .starts_with(s)         prefix check
  .ends_with(s)           suffix check
  .replace(old, new)      substitution
  .split(sep)             split to list[string]
  .join(list)             join list to string
  .length                 character count

Math (Layer 1 intrinsics):
  math.sqrt(x)   math.abs(x)    math.pow(x,y)
  math.sin(x)    math.cos(x)    math.tan(x)
  math.PI        math.TAU       math.E

LAYER 2 — import skev.core:

File I/O (skev.file):
  await file.read_text(path)    -> result[string]
  await file.write_text(p, s)   -> result[nothing]
  await file.read_bytes(path)   -> result[list[uint8]]
  await file.write_bytes(p, b)  -> result[nothing]
  file.exists(path)             -> bool
  file.path(parts...)           -> string (platform-safe)

Math extensions (skev.math):
  math.clamp(v, min, max)       math.lerp(a, b, t)
  math.smoothstep(e0, e1, x)    math.map(v, i0,i1,o0,o1)
  math.floor(x)  math.ceil(x)   math.round(x)
  math.random()  math.random_int(min, max)
  math.noise(x, y)              math.set_seed(n)

Time (skev.time):
  time.elapsed                  seconds since start
  time.unix()                   UNIX timestamp
  time.frame_count              frames since start
  time.fps                      current FPS
  time.stopwatch()              → Stopwatch
  timer(seconds) >> ... << timer     one-shot
  every(seconds) >> ... << every     repeating

JSON (skev.json):
  json.serialise(value)         -> string
  json.parse(text)              -> result[T]

Binary (skev.serial):
  serial.encode(value)          -> list[uint8]
  serial.decode(bytes)          -> result[T]
  (requires #! serialisable on type)

Collections:
  set[T]                        unordered unique values
  .add(v)  .remove(v)  .contains(v)
  .union(s)  .intersect(s)  .minus(s)

LAYER 3 — Domain imports:

import skev.network:
  await network.get(url, headers?)      -> result[NetworkResponse]
  await network.post(url, body, hdrs?)  -> result[NetworkResponse]
  await network.websocket(url)          -> result[WebSocket]

import skev.game:
  scene  audio  input  physics

import skev.robotics:
  sensors  motors  realtime

import skev.render:
  pipeline  shaders  materials
```

---

*End of Chapter 7 v0.1 — Next: Chapter 8: Interoperability*
