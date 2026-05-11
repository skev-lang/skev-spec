<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Comparison v2
## Complete Honest Assessment — All Domains, All Competitors
**Version:** 2.0
**Authors:** AJ (Copyright © 2026)
**Supersedes:** Skev_Language_Comparison.md (v1)
**Languages assessed:** 12
**Domains assessed:** 7
**Honesty level:** Complete — including where Skev loses

---

> **The honest approach:** This document exists to help developers make real
> technology decisions. Every rating includes a reason. Every weakness is stated
> plainly. Where Skev loses, it says so.

---

## PART 1 — THE LANDSCAPE

### Why This Comparison Exists

Skev competes in a space with 30+ years of established incumbents and several
serious new challengers. A developer evaluating Skev deserves a clear,
honest answer to: "Should I use this, or something else?"

This document answers that question for seven domains:

```
Games & Interactive Entertainment
XR / VR / AR
Simulations
Robotics & Embedded Systems
Animation / VFX
Interactive Applications
Real-time Networking
```

### The Honest Constraint

```
⚠️  CRITICAL DISCLOSURE — READ FIRST:

Skev at the time of this writing:
  → Specification complete (Chapters 1-11 + 3.5)
  → Python transpiler complete (417 tests passing)
  → Real compiler (Rust/LLVM): NOT YET BUILT (Milestone 3)
  → Shipped products: ZERO
  → Package ecosystem: ZERO
  → Community: DOES NOT EXIST YET

Every other language in this comparison has a working compiler,
shipped products, and an active community.

This document compares Skev's DESIGN against other languages'
IMPLEMENTATIONS. That is a genuine limitation. It is stated here
and not repeated to avoid the comparison becoming a disclaimer.
```

### The 12 Languages

**Tier 1 — Full comparison (complete code examples):**
```
C++23       — the incumbent. Everything runs on it.
C# 13       — Unity's language. Largest game dev community.
Rust 1.7x   — memory safe, no GC, the modern challenger.
Zig 0.13    — lean, explicit, no hidden magic.
Odin        — game-native. Closest philosophical neighbour to Skev.
```

**Tier 2 — Detailed summary (no full code examples):**
```
Jai*        — Jonathan Blow's language. Designed for games.
Go 1.22     — concurrency focus. Backend and networking.
GDScript 4  — Godot's scripting language. Engine-specific.
Lua 5.4     — embedded scripting. LuaJIT near-native speed.
Vale 0.2†   — memory safe without GC or borrow checker.
```

**Tier 3 — Brief mention:**
```
Python 3.13  — scripting layer. Not a systems language.
AngelScript  — used in engines for modding. Niche.
```

```
* Jai has never been publicly released. All information in this
  document comes from conference talks (2014–2024) and developer
  demos. Ratings marked [JAI-DEMO] are based on observed behaviour,
  not verified compilation.

† Vale 0.2 is pre-production. Generational references are partially
  implemented. Ratings reflect the stated design, not full production
  behaviour.
```

---

## PART 2 — LANGUAGE PROFILES

### C++23

```
Philosophy:    Zero-cost abstractions. You don't pay for what you don't use.
First release: 1985 (C with Classes). C++23: 2023.
Memory model:  Manual. Smart pointers mitigate (not eliminate) risk.
Compiler:      Clang, GCC, MSVC — all mature, heavily optimised.
Package mgmt:  CMake, vcpkg, Conan — fragmented, no standard.
Primary use:   Games, OS, databases, embedded, finance.
```

**Honest strengths:**
- Raw performance ceiling. 100% baseline — everything else measured against it.
- 30-year ecosystem. Any library you need exists.
- Maximum platform support. If it has a CPU, C++ runs on it.
- Templates are turing-complete. Generics power nothing else matches.
- Zero-overhead principle genuinely delivered.

**Honest weaknesses:**
- Memory unsafety. Buffer overflows, use-after-free, dangling pointers — all silent.
- Build system chaos. No standard. Every project invents its own.
- Compile times catastrophic. Large C++ projects take minutes to hours.
- Null everywhere. `nullptr` dereference crashes with no helpful message.
- Learning curve enormous. "Effective C++" is a 300-page book for reason.
- Header files. A 1972 solution still causing pain in 2026.
- Error messages. Template errors produce pages of incomprehensible output.

**Honest verdict:** C++ is the reference implementation of systems programming.
Use it when you need maximum performance, maximum control, and can afford the
team expertise cost. For solo developers or small teams: the cognitive cost is real.

---

### C# 13 / .NET 9

```
Philosophy:    Productive systems programming. Managed but fast.
First release: 2002. C# 13 / .NET 9: 2024.
Memory model:  Garbage collected. Native AOT reduces but doesn't eliminate GC.
Compiler:      Roslyn. Fast incremental compilation.
Package mgmt:  NuGet — the best package manager in this comparison.
Primary use:   Games (Unity), enterprise, desktop, web.
```

**Honest strengths:**
- Unity integration. The most popular game engine uses C#.
- NuGet ecosystem. Excellent. Consistent. Searchable.
- Language design quality. LINQ, async/await, nullable reference types — well designed.
- Tooling. Visual Studio and Rider are world-class.
- Incremental hot reload. Better than C++, not as complete as Skev targets.
- `async/await` is genuinely the best async design in any mainstream language.
- AOT compilation (Native AOT) makes deployment simpler and faster.

**Honest weaknesses:**
- GC pauses. Even with Native AOT, the GC still runs. Pauses still happen.
  Unity's GC has caused frame rate drops in shipped AAA games.
- Runtime overhead. The .NET runtime is not optional.
- No game-native types. Vector3 is a struct, not a primitive.
- Value type boxing. `object` boxes structs — hidden allocation.
- Nullable inconsistency. `?` nullable reference types only with `enable` pragma.
- Null still exists. `NullReferenceException` is still the most common runtime error.

**Honest verdict:** C# is excellent for teams that use Unity or Microsoft's ecosystem.
The GC is a real limitation for performance-critical realtime code.
For new projects not tied to Unity: Rust or Skev are worth evaluating.

---

### Rust 1.7x

```
Philosophy:    Fearless concurrency. Memory safety without GC.
First release: 2010 (alpha). Rust 1.0: 2015. 1.7x: 2024-2025.
Memory model:  Ownership + borrow checker. Compiler-proven memory safety.
Compiler:      rustc (LLVM backend). Notoriously slow.
Package mgmt:  Cargo + crates.io — the second-best in this comparison.
Primary use:   Systems, WebAssembly, game engines, networking.
```

**Honest strengths:**
- Memory safety formally proven. The borrow checker is not heuristic — it's a proof.
- No GC. Zero-cost memory management without manual malloc/free.
- crates.io. Large, high-quality ecosystem.
- Cargo. Excellent build system. Reproducible builds.
- Fearless concurrency. Data races are compile errors.
- WebAssembly. Rust → WASM is the best-supported path.
- Growing game ecosystem. Bevy ECS is impressive.
- Error handling. `Result<T, E>` is excellent — close to Skev's design.

**Honest weaknesses:**
- Learning curve. The borrow checker is the steepest wall in any language here.
  Average time to "productive with Rust": 3-6 months.
- Compile times. Notoriously slow. 60-second builds are common in large projects.
- Verbosity. `Option<Box<dyn Error>>` is real code that real people write.
- Lifetime annotations. `'a, 'b, 'static` add cognitive overhead constantly.
- No game-native types. Vector3 comes from a library (glam, nalgebra).
- No entity system. ECS (Bevy) is third-party and architecturally opinionated.
- Async/await complexity. `Pin<Box<dyn Future<Output=T>>>` is genuinely hostile.
- Hot reload. Effectively impossible in the current architecture.

**Honest verdict:** Rust is the right choice when memory safety proofs matter more
than developer velocity. For game engines: Bevy is serious. For game logic: the
borrow checker fights you more than it helps. Rust is a systems language doing
game development — not a game language.

---

### Zig 0.13

```
Philosophy:    No hidden control flow. No hidden allocations. Simple.
First release: 2016. Zig 0.13: 2024.
Memory model:  Manual with explicit allocators. Allocator is a parameter.
Compiler:      Self-hosted (since 0.10). Fast.
Package mgmt:  zig build — built in. No central registry yet.
Primary use:   Systems, embedded, C replacement, game engines.
```

**Honest strengths:**
- `comptime`. Zig's compile-time execution is the most powerful metaprogramming
  in this list. Functions run at compile time. Types are first-class values.
  This is genuinely more powerful than C++ templates AND Skev's generics.
- No hidden control flow. No exceptions. No implicit allocations. What you see is what runs.
- Explicit allocators. Every allocation takes an allocator parameter. Memory is visible.
- C interop. Better than C++. Zero overhead. No headers needed.
- Compile speed. Among the fastest compiled languages here.
- Cross-compilation. Zig ships a complete C/C++ cross-compilation toolchain.
- Small language. Fewer concepts than C++, Rust, or Skev.
- Error unions. `!T` — clean, explicit, integrated into the type system.

**Honest weaknesses:**
- Pre-1.0. Zig 0.13 is not stable. Breaking changes still happen.
- No standard package registry. Package management is improving but not mature.
- Small community. Growing fast but still small relative to C++/Rust.
- No game-native types. Vector3 is a library concern.
- No entity system. No event system.
- No hot reload.
- `comptime` power comes with complexity. Duck-typed comptime is hard to
  understand when it fails. Error messages are improving but still rough.
- Allocator discipline is correct but verbose. Every allocation decision is explicit.

**Honest verdict:** Zig is the best C replacement. For embedded and systems work
where allocation control is critical — Zig wins. For games specifically, it has
no game-native abstractions. Developers use Zig to write game engines in,
not game logic. The comptime system is genuinely exceptional.

---

### Odin

```
Philosophy:    Joy of programming. Simple, fast, data-oriented.
First release: 2016 (by Bill Hall / gingerBill).
Memory model:  Manual + allocator contexts. Context-based.
Compiler:      Self-hosted. Fast.
Package mgmt:  Core library. Vendor directory. No registry.
Primary use:   Games, graphics, tools, simulations.
```

**Honest strengths:**
- Closest to Skev's stated philosophy. "Fast like C, readable like Python."
- Game-oriented from day one. The creator works on game tools.
- Fewer concepts than Skev. Odin deliberately resists adding features.
- Context system. Thread-local allocator context — clean implicit allocator passing.
- SIMD built-in. `#simd` type. Games benefit directly.
- Hot reload. Native support via shared libraries. Production-tested.
- `using` keyword. Struct embedding without inheritance — similar to Skev's `has`.
- SOA (Structure of Arrays). First-class. Cache performance for game loops.
- Excellent foreign function interface. C interop with zero friction.
- Compile speed. Very fast.

**Honest weaknesses — and this is where Skev must be honest:**
- Smaller community than Rust. Larger than Skev (which is zero).
- No package registry. The vendor directory model is explicit but manual.
- No memory safety. Manual memory — same risk profile as C.
- Null exists. `nil` pointer dereference is a runtime crash.
- Error handling is `(value, error)` tuples — less structured than Skev's result[T].
- No built-in entity/event system. These are patterns, not language features.
- Less safe than Skev. No ARC, no maybe, no result[T] enforcement.
- Generics exist but limited. Less powerful than Zig comptime, comparable to Skev.

**Honest Skev vs Odin assessment:**
```
Odin wins:   Simpler to learn (fewer concepts)
             Proven (ships real games today)
             Hot reload (production-tested)
             SOA (Skev does not have built-in SOA)
             No safety overhead (faster in theory)

Skev wins:   ARC memory (no manual free)
             result[T] enforcement (compiler-forced error handling)
             maybe T (no null crashes)
             Built-in entity/event system
             Readability at scale (>> << labels)
             Safety by default without GC

Honest call: If you need to ship a game TODAY — use Odin.
             If safety matters more than simplicity — Skev.
             If you've never programmed before — Skev is designed for you.
             If you're a C programmer wanting better — Odin is designed for you.
```

---

### Jai [JAI-DEMO]

```
⚠️  All ratings below are based on public conference talks and demos.
    Jai has NOT been publicly released as of this writing.
    Information from: Jonathan Blow's streams and GDC/conferences 2014-2024.
```

```
Philosophy:    Games first. Iteration speed above all.
Designed by:   Jonathan Blow (creator of Braid, The Witness).
Memory model:  Manual + pools. "The game knows when to free."
Compiler:      Exists. Not public. Reported to be extremely fast.
Package mgmt:  Modules system. Not public.
Primary use:   Games. Only games.
```

**Observed strengths [JAI-DEMO]:**
- Metaprogramming. Compile-time code execution on par with Zig comptime.
- Iteration speed. The compiler is reported to be extraordinarily fast.
- Data-oriented design built in. SOA transformations at language level.
- Build system in the language itself. No separate build tool.
- Practical game focus. Every design decision asks "does a game developer need this?"
- Hot reload. Built-in.
- Minimal ceremony. No boilerplate.

**Observed weaknesses [JAI-DEMO]:**
- Not released. The largest weakness. It has been "coming soon" for 10 years.
- Manual memory. No ARC, no GC.
- Null exists.
- Unknown community (will exist when released, but unknown scale).
- Untested at industry scale.

**Honest call:** Jai is the most interesting language in the game development
space that nobody can actually use. If it ships, it will be a serious competitor
to Skev in the pure-games domain. Skev's advantage: it exists.

---

### Go 1.22

```
Philosophy:    Simplicity at scale. Fast compile. Easy concurrency.
First release: 2009. Go 1.22: 2024.
Memory model:  Garbage collected. Short pauses. Not realtime-safe.
Compiler:      gc (the Go compiler). Very fast.
Package mgmt:  go modules — good, built in.
Primary use:   Web services, CLI tools, DevOps, game backends.
```

**Honest strengths:**
- Goroutines. The easiest concurrency model in this list.
- Compile speed. The fastest compiled language here.
- Standard library. Comprehensive and high quality.
- Deployment. Single binary, trivial cross-compilation.
- Readability. Deliberately minimal syntax.

**Honest weaknesses:**
- GC pauses. Short but present. Not suitable for realtime game loops.
- No generics power. Go 1.18 added generics — still limited.
- Null everywhere. `nil` pointer dereference is the #1 Go runtime error.
- No game-native types. Vector math is library-only.
- No entity system. No event system.
- Verbose error handling. `if err != nil` repeated thousands of times.
- Not for games. Go is a cloud/networking language.

**When Go wins over Skev:** Game backend servers, matchmaking services,
analytics pipelines, DevOps tooling. Go's goroutine model and deployment
simplicity win in these cases. Skev wins on the game client side.

---

### GDScript 4.x

```
Philosophy:    Python-like scripting inside Godot engine.
First release: 2014 (Godot 1.0). GDScript 4: 2023.
Memory model:  Reference counted (Godot's refcount, not language-level).
Compiler:      Interpreted + optional bytecode compilation.
Package mgmt:  Godot Asset Library. Engine-coupled.
Primary use:   Game logic INSIDE Godot. Not standalone.
```

**Honest assessment:**
GDScript is not a general-purpose language. Comparing it to C++ or Skev is
like comparing SQL to Python — different tools for different jobs.

What GDScript does well:
- Python-like syntax. Beginners learn it in hours.
- Deep Godot integration. Access to every engine feature natively.
- Signals (events). First-class. More ergonomic than most languages.
- Node tree. The entity model is the engine architecture.

What GDScript cannot do:
- Run outside Godot.
- Compile to native code (standard GDScript).
- Compete on performance with C++/Rust/Skev.

**When GDScript beats Skev:** You are making a Godot game and want the fastest
path from idea to prototype. GDScript wins on iteration speed inside Godot.
Skev targets developers who want this experience without engine lock-in.

---

### Lua 5.4 / LuaJIT

```
Philosophy:    Lightweight embeddable scripting. Small and fast.
First release: 1993. Lua 5.4: 2020. LuaJIT: 2.1 (2023).
Memory model:  Garbage collected. Incremental.
Compiler:      Interpreted (Lua 5.4) / JIT (LuaJIT — near-native speed).
Package mgmt:  LuaRocks — exists, inconsistent quality.
Primary use:   Embedded scripting in C++ games. Modding.
```

**Honest assessment:**
LuaJIT is the fastest scripting language in this list. Roblox uses it.
World of Warcraft uses it. The modding API for many major games uses it.

Lua wins where:
- Embedding in an existing C++ engine is the goal.
- Mod authoring for end users (non-programmers).
- Lightweight scripting that calls into C++ for heavy work.

Lua loses where:
- Type safety matters (Lua is dynamically typed).
- The code grows large (untyped code is hard to maintain).
- First-class game-native types are needed.

**Skev vs Lua:** Not the same category. Skev is the host language.
Lua is the scripting language. In Skev's model, Lua/WASM mods run
inside Skev programs via the Chapter 10 sandbox.

---

### Vale 0.2

```
⚠️  Vale is pre-production. This rating reflects stated design goals.
    Generational references are partially implemented as of 0.2.
```

```
Philosophy:    Memory safety without GC, without borrow checker.
Designed by:   Evan Ovadia. Started 2019.
Memory model:  Generational references. Single ownership + fast checks.
Compiler:      valec. Early stage.
Primary use:   Games, systems. Stated goal.
```

**Interesting design:** Vale's generational references are a novel approach.
Instead of a borrow checker or GC, every pointer has a generation number.
Accessing a freed pointer is detected at runtime (not compile time) with
near-zero overhead. This is a genuinely different model from Rust and Skev.

**Honest status:** Vale is fascinating but pre-production. It has not
shipped any real products. It is more experimental than Skev.

**Honest comparison to Skev:**
- Vale: generational references — no GC, no borrow checker, no ARC
- Skev: ARC — no GC, no borrow checker, deterministic
- Both target the same space (memory safe, no GC, readable)
- Vale's model may have lower overhead than ARC's atomic operations
- Skev is further along in specification and implementation planning

---

### Python 3.13

```
Philosophy:    Readability above all. Everything else is secondary.
Memory model:  Reference counted + cyclic GC.
Primary use:   Scientific computing, AI/ML, scripting, tooling.
```

**In game/simulation context:**
Python itself is too slow for realtime game loops (5-10% of C++).
Python's dominance in simulation comes from NumPy/PyTorch — C++ libraries.

Where Python genuinely wins over Skev TODAY:
- AI/ML ecosystem. PyTorch, TensorFlow, JAX — no equivalent in Skev.
- Scientific libraries. NumPy, SciPy, Matplotlib — decades of development.
- Prototyping speed. A simulation prototype in Python takes hours.
- Data analysis. Pandas, Jupyter — no equivalent in Skev.

**Honest call:** Python is not a game language. It is the best scripting and
scientific language in the world. Skev will eventually host Python as a
scripting tier (Chapter 8, Tier 2). The languages are not competitors.

---

### AngelScript

```
Philosophy:    C-like scripting that embeds in game engines.
Primary use:   Modding in established engines. Niche.
```

AngelScript is used in a small number of game engines as a modding
language (Eternal Lands, some Unreal projects). It occupies the same
niche as Lua — embedded scripting — but with C-like syntax.

It is not a general-purpose language and is not a competitor to Skev.
Users wanting C-like mod scripting in a Skev game would use Skev's
WASM sandbox (Chapter 10) rather than embedding a scripting engine.

---

## PART 3 — DOMAIN ANALYSIS

### Domain 1: Games & Interactive Entertainment

```
PRIMARY COMPETITORS:  C++23, C# 13, Odin
SECONDARY:            Rust, Zig, GDScript
NICHE:                Lua (modding), Jai (if released)
SKEV TARGET:          ✅ Primary target domain
```

**Current landscape:**
```
C++:    Powers Unreal, custom engines. Industry standard.
        Crew of 100+ engineers can manage the complexity.
        Solo developers: significant pain.

C# 13:  Powers Unity. Millions of developers.
        GC pauses are real. Frame spikes happen.
        Unity's C# is slightly behind C# 13 — 2026 still catching up.

Odin:   Serious contender for solo/indie. Proven.
        Several shipped indie games. Growing community.
        No safety guarantees — manual memory.

Rust:   Bevy ECS is impressive. Growing.
        Borrow checker fights common game patterns.
        Entity ownership + mutation is a known Rust pain point.

Zig:    Used for game engines (not game logic).
        No game-native abstractions.
        Comptime is exceptional for engine work.

GDScript: Best for Godot-only projects. Not portable.
```

**Where Skev positions:**
```
✅ ARC beats C# GC for realtime — no pauses
✅ Game-native types beat all of the above (Vector3!, Color!)
✅ Built-in entity/event system beats C++ manual implementation
✅ Safer than Odin/C++ with similar readability goal
✅ More readable than Rust for game logic
❌ No ecosystem — team must build from scratch
❌ No shipped games — proof of concept only
❌ Odin hot reload exists today — Skev's is specified but unbuilt
```

**Domain verdict:**
Skev is the best-designed language for games that does not yet exist as
a working compiler. Odin is the best actual choice for a game developer
today who wants what Skev promises.

---

### Domain 2: XR / VR / AR

```
PRIMARY COMPETITORS:  C++23, C# 13 (Unity/MRTK)
SECONDARY:            Rust (wgpu), Zig
SKEV TARGET:          ✅ Primary target domain
```

**XR-specific requirements:**
```
→ 90fps minimum (11ms budget per frame)
→ No frame drops (VR sickness)
→ Spatial math everywhere (Transform!, Quat!, Ray!)
→ Left/right eye rendering synchronisation
→ Hand tracking events
→ Mixed reality occlusion
```

**Current landscape:**
```
C# 13 (Unity):  Most XR development happens here.
                MRTK2/MRTK3 for Microsoft HoloLens.
                GC pauses cause visible artifacts in VR.
                Teams spend significant effort on GC avoidance.

C++ (Unreal):   Used for high-fidelity VR.
                No GC pauses.
                Very high developer cost.
                OpenXR integration mature.

Rust:           wgpu + OpenXR is a working combination.
                Memory safety valuable for XR robustness.
                Still assembling libraries.
```

**Where Skev positions:**
```
✅ No GC pauses — XR's most critical requirement
✅ Game-native types — Transform!, Quat! are spatial-math primitives
✅ ARC deterministic memory — predictable frame timing
✅ More readable than C++ at equivalent performance
❌ No XR-specific libraries yet
❌ No OpenXR binding yet
❌ No shipped XR applications
```

**Domain verdict:** Skev is architecturally well-suited for XR. The no-GC,
game-native-types combination addresses XR's primary pain points. The
library gap is real and significant.

---

### Domain 3: Simulations

```
PRIMARY COMPETITORS:  C++23, Python (NumPy/SciPy), Julia, MATLAB
SECONDARY:            Rust, Zig, Fortran (legacy HPC)
SKEV TARGET:          🟡 Strong candidate — not primary
```

**Simulation types where Skev fits:**
```
→ Real-time physics simulations (robot, vehicle, fluid)
→ Agent-based simulations (crowd, AI behaviour)
→ Interactive simulations (training systems, digital twins)
→ Game-adjacent simulations (battle system, economy model)
```

**Simulation types where Skev is weaker:**
```
→ Scientific HPC (Python/NumPy/PyTorch ecosystem dominates)
→ Statistical simulations (R, MATLAB, Julia)
→ GPU-parallel simulations (CUDA is C++, not Skev)
→ Legacy simulations (often Fortran — no path to Skev)
```

**Honest assessment by competitor:**
```
Python: Loses on speed (10x slower). Wins on library ecosystem.
        NumPy/SciPy are irreplaceable. Python is not leaving simulation.

C++:    Wins on raw speed. Loses on development time.
        A simulation in C++ takes 3x longer to write than Skev.

Julia:  Interesting — JIT compiled, scientific-first.
        For numerical simulation: Julia vs Skev is genuine competition.
        Julia wins on linear algebra. Skev wins on real-time safety.

Rust:   Similar speed profile to Skev. Less readable for simulation
        logic. Better for simulation infrastructure (server, scheduler).
```

**Domain verdict:** Skev is a strong choice for interactive/real-time
simulations and agent-based simulations. For scientific HPC and data-heavy
simulations: Python/Julia's library ecosystems are irreplaceable today.

---

### Domain 4: Robotics & Embedded Systems

```
PRIMARY COMPETITORS:  C, C++, Rust (growing fast), Python (ROS2)
SECONDARY:            Zig (excellent fit), Ada (safety-critical)
SKEV TARGET:          🟡 Strong candidate via safety-critical profile
```

**Robotics-specific requirements:**
```
→ Real-time control loops (1ms budget common)
→ Deterministic memory (no GC)
→ Hardware interrupt handling
→ Safety-critical certification (ISO 26262, IEC 62443)
→ Low-power embedded targets
→ C interop (everything talks C)
```

**Language positions in robotics:**
```
C:      Still dominant in embedded. Low-level control.
        Unsafe. No abstractions. But: runs everywhere.

C++:    ROS2 primary language. Dominant in robot middleware.
        Memory unsafe. Complex. But: the industry standard.

Rust:   Growing fast in robotics. ROS2-Rust exists.
        Memory safety matters for safety-critical robots.
        Borrow checker is painful for real-time control loops.
        No realtime guarantees in standard Rust.

Python: ROS2 Python API. Used for high-level coordination.
        Too slow for motor control. Good for task planning.

Zig:    Excellent fit for embedded.
        Explicit allocators — critical for embedded.
        No hidden allocation is exactly what embedded needs.
        Comptime for hardware register definitions.
        Cross-compilation built in.
```

**Where Skev positions:**
```
✅ #! safety_critical annotation (Chapter 10)
✅ Realtime paths — no GC, no hidden allocation
✅ result[T] enforcement — safety-critical error handling
✅ C ABI interop — talks to any hardware driver
✅ Deterministic ARC — no GC pauses in control loops
❌ No microcontroller support (needs LLVM target + runtime port)
❌ ARC overhead on embedded (atomic operations cost matters)
❌ No certified toolchain (safety certification requires it)
❌ Zig wins on: smaller runtime, explicit allocators, comptime
```

**Honest Zig vs Skev in embedded:**
```
Zig wins: no runtime at all, explicit allocators, comptime for
          register definitions, smaller binary, better C toolchain.

Skev wins: safety guarantees (ARC, maybe, result[T]), readability,
           entity model for robot subsystems, hot reload for iteration.

Verdict: For deep embedded (microcontrollers, bare metal):
         Zig or C. Not Skev.
         For robot application logic (above OS, above drivers):
         Skev is a serious option. Zig is also strong.
```

---

### Domain 5: Animation / VFX

```
PRIMARY COMPETITORS:  C++23, Python (Maya/Houdini scripting)
SECONDARY:            Rust (emerging)
SKEV TARGET:          🟡 Strong candidate
```

**Animation/VFX contexts:**
```
→ DCC tool plugins (Maya, Houdini, Blender) — Python scripting
→ Render engine cores — C++
→ Procedural generation — C++, Houdini VEX, Python
→ Animation system logic — C++ (in engines)
→ Export/pipeline tools — Python, C#
```

**Where Skev fits in the pipeline:**
```
✅ Animation system logic inside Skev game/XR applications
✅ Procedural content generation (game-adjacent)
✅ Pipeline tools where Python is too slow
✅ Real-time VFX within a game engine context
❌ Maya/Houdini plugins — those expect Python or C++ APIs
❌ Offline rendering — no GPU compute pipeline yet
❌ VFX library ecosystem — does not exist
```

**Honest assessment:**
Skev is relevant for the game-adjacent end of VFX (real-time VFX,
in-engine procedural generation). For the DCC pipeline (offline render,
compositing, Maya/Houdini scripting) — Python is entrenched and
irreplaceable. Skev does not compete with Python in DCC tool scripting.

---

### Domain 6: Interactive Applications

```
PRIMARY COMPETITORS:  C# 13 (WPF/MAUI), JavaScript/TypeScript,
                      Swift, Kotlin, Flutter/Dart
SECONDARY:            Rust (Tauri), Go
SKEV TARGET:          🟡 Secondary target
```

**Interactive app contexts:**
```
→ Desktop apps (Windows, macOS, Linux)
→ Mobile apps (iOS, Android)
→ Web apps (WASM target)
→ Hybrid apps (Electron-style)
→ XR interfaces (overlapping with Domain 2)
```

**Honest position:**
Skev's Chapter 11 UI capabilities are specified but minimal.
The language is designed for interactive applications in the sense of
"game-like interactive experiences" — not business software.

```
C# 13 wins: For desktop apps in the Microsoft ecosystem. MAUI is mature.
JavaScript wins: For web apps. The WASM target helps Rust/Skev but
                 the JS ecosystem is irreplaceable.
Swift wins:  For iOS/macOS native apps.
```

**Where Skev adds value in apps:**
Interactive apps with game-like real-time requirements:
simulations displayed in a UI, interactive data visualisation,
XR-adjacent experiences. For standard CRUD apps: C# or JavaScript
is the right choice.

---

### Domain 7: Real-time Networking

```
PRIMARY COMPETITORS:  C++ (custom), Go 1.22, Rust, Erlang
SECONDARY:            C# 13 (SignalR), Node.js
SKEV TARGET:          🟡 Secondary target
```

**Real-time networking contexts:**
```
→ Game servers (authoritative simulation)
→ Matchmaking services
→ Low-latency message brokers
→ Multiplayer state synchronisation
→ Real-time analytics
```

**Language positions:**
```
Go:    Goroutines make Go the easiest high-concurrency networking
       language. Go dominates: microservices, game backends,
       matchmaking, analytics pipelines. Go's net/http and net are
       world-class. Skev cannot compete with Go's ecosystem here.

Rust:  Tokio makes Rust competitive for high-performance networking.
       More complex than Go but: no GC, better performance.
       Used in Discord, Cloudflare, game infrastructure.

C++:   Custom networking for latency-critical paths.
       Game server engines are almost all C++.
       libuv, ASIO, custom event loops.

Erlang: Telco-grade reliability. 99.9999% uptime systems.
        Different model — not relevant for most game networking.
```

**Where Skev positions:**
```
✅ channel[T] — native typed channels for game event routing
✅ async/await — non-blocking I/O for server logic
✅ No GC — consistent latency, no GC spikes
✅ Entity model — game server objects as Skev entities
❌ No networking library yet (skev.network is specified, not built)
❌ Go wins on: ecosystem, simplicity, community for server work
❌ Rust wins on: proven high-performance server ecosystem (Tokio)
```

**Honest verdict:** Skev's game server is the game logic layer.
The infrastructure (load balancers, matchmaking, analytics) should
use Go or Rust today. Skev is the application layer, not the
infrastructure layer — at least until the ecosystem grows.

---

## PART 4 — DIMENSION ANALYSIS

### Dimension 1: Syntax & Readability

**The test:** Write a character with health, damage handling, and death.
Same logic. All Tier 1 languages.

```cpp
// C++23 — character with health system
#include <algorithm>
#include <optional>

class Character {
public:
    int health;
    int max_health;
    bool alive;

    explicit Character(int hp) : health(hp), max_health(hp), alive(true) {}

    void take_damage(int amount) {
        health = std::clamp(health - amount, 0, max_health);
        if (health == 0) alive = false;
    }

    void heal(int amount) {
        if (alive)
            health = std::clamp(health + amount, 0, max_health);
    }

    [[nodiscard]] float health_percent() const {
        return static_cast<float>(health) / static_cast<float>(max_health);
    }
};
// 18 lines total — explicit types, explicit cast, [[nodiscard]] optional
```

```csharp
// C# 13 — character with health system
public class Character {
    public int Health { get; private set; }
    public int MaxHealth { get; }
    public bool Alive { get; private set; }

    public Character(int hp) {
        Health = MaxHealth = hp;
        Alive = true;
    }

    public void TakeDamage(int amount) {
        Health = Math.Clamp(Health - amount, 0, MaxHealth);
        if (Health == 0) Alive = false;
    }

    public void Heal(int amount) {
        if (Alive) Health = Math.Clamp(Health + amount, 0, MaxHealth);
    }

    public float HealthPercent() =>
        Math.Clamp((float)Health / MaxHealth, 0f, 1f);
}
// 16 lines total — property accessors, PascalCase convention
```

```rust
// Rust 1.7x — character with health system
pub struct Character {
    pub health: i32,
    pub max_health: i32,
    pub alive: bool,
}

impl Character {
    pub fn new(hp: i32) -> Self {
        Self { health: hp, max_health: hp, alive: true }
    }

    pub fn take_damage(&mut self, amount: i32) {
        self.health = (self.health - amount).clamp(0, self.max_health);
        if self.health == 0 { self.alive = false; }
    }

    pub fn heal(&mut self, amount: i32) {
        if self.alive {
            self.health = (self.health + amount).clamp(0, self.max_health);
        }
    }

    pub fn health_percent(&self) -> f32 {
        (self.health as f32 / self.max_health as f32).clamp(0.0, 1.0)
    }
}
// 20 lines total — struct + impl split, &mut self, as f32 cast
```

```zig
// Zig 0.13 — character with health system
const std = @import("std");

pub const Character = struct {
    health: i32,
    max_health: i32,
    alive: bool,

    pub fn init(hp: i32) Character {
        return .{ .health = hp, .max_health = hp, .alive = true };
    }

    pub fn takeDamage(self: *Character, amount: i32) void {
        self.health = std.math.clamp(self.health - amount, 0, self.max_health);
        if (self.health == 0) self.alive = false;
    }

    pub fn heal(self: *Character, amount: i32) void {
        if (self.alive) {
            self.health = std.math.clamp(self.health + amount, 0, self.max_health);
        }
    }

    pub fn healthPercent(self: Character) f32 {
        return std.math.clamp(
            @as(f32, @floatFromInt(self.health)) /
            @as(f32, @floatFromInt(self.max_health)), 0.0, 1.0);
    }
};
// 22 lines total — @floatFromInt cast is verbose but explicit
```

```odin
// Odin — character with health system
package main
import "core:math"

Character :: struct {
    health:     int,
    max_health: int,
    alive:      bool,
}

character_init :: proc(hp: int) -> Character {
    return Character{health = hp, max_health = hp, alive = true}
}

take_damage :: proc(c: ^Character, amount: int) {
    c.health = clamp(c.health - amount, 0, c.max_health)
    if c.health == 0 do c.alive = false
}

heal :: proc(c: ^Character, amount: int) {
    if c.alive do c.health = clamp(c.health + amount, 0, c.max_health)
}

health_percent :: proc(c: Character) -> f32 {
    return clamp(f32(c.health) / f32(c.max_health), 0, 1)
}
// 20 lines total — struct + separate procedures, pointer for mutation
```

```swift
// Skev — character with health system
entity Character >>
    health     :: int  = 100
    max_health :: int  = 100
    alive      :: bool = true

    take_damage(amount: int) -> nothing
        health = math.clamp(health - amount, 0, max_health)
        if health == 0 >>
            alive = false
        << health == 0
    << take_damage

    heal(amount: int) -> nothing
        if alive >>
            health = math.clamp(health + amount, 0, max_health)
        << alive
    << heal

    health_percent() -> float
        result math.clamp(health / max_health, 0.0, 1.0)
    << health_percent

<< Character
// 20 lines total — entity block, >> << structure, no self., no cast
```

**Readability Ratings:**

| Metric              | C++23 | C# 13 | Rust | Zig | Odin | Skev |
|---------------------|-------|-------|------|-----|------|------|
| Beginner clarity    | 2     | 3     | 2    | 3   | 3    | 5    |
| Noise per feature   | 2     | 3     | 3    | 3   | 4    | 5    |
| Cast verbosity      | 3     | 4     | 2    | 1   | 3    | 5    |
| Large file nav      | 2     | 3     | 3    | 3   | 3    | 5    |
| Intent clarity      | 3     | 4     | 3    | 4   | 4    | 5    |

**Honest Skev note:** Skev's `>> <<` block system is unfamiliar to developers
from every other language here. First contact is jarring. After 30 minutes: the
labels make large files significantly more navigable than any competitor.

---

### Dimension 2: Memory Model

| Language | Model          | GC Pauses | Manual Free | Developer Effort |
|----------|---------------|-----------|-------------|-----------------|
| C++23    | Manual         | None      | Yes         | High            |
| C# 13    | GC + Native AOT| Reduced   | No          | Low             |
| Rust     | Borrow checker | None      | No (auto)   | High (checker)  |
| Zig 0.13 | Manual + alloc | None      | Yes         | High            |
| Odin     | Manual + ctx   | None      | Yes         | Medium          |
| Go 1.22  | GC             | Short     | No          | Low             |
| GDScript | Refcount + GC  | Occasional| No          | Low             |
| Lua 5.4  | GC             | Yes       | No          | Low             |
| Python   | Refcount + GC  | Occasional| No          | Low             |
| Skev     | ARC            | None      | No          | Low             |
| Vale 0.2 | Gen. refs      | None      | No (design) | Low (design)    |

**Honest scoring (5 = best for realtime game/simulation work):**

| Metric              | C++23 | C# 13 | Rust | Zig | Odin | Skev |
|---------------------|-------|-------|------|-----|------|------|
| No GC pauses        | 5     | 3     | 5    | 5   | 5    | 5    |
| No manual free      | 1     | 5     | 5    | 1   | 1    | 5    |
| Predictable timing  | 3     | 3     | 5    | 4   | 4    | 5    |
| Developer overhead  | 1     | 4     | 2    | 2   | 3    | 5    |
| Safety guarantees   | 1     | 3     | 5    | 2   | 2    | 4    |

**Skev honest note:** ARC atomic reference counts have a small overhead
vs manual free. In microbenchmarks, Zig and C++ will beat ARC.
For typical game code (not hot loop level), the difference is noise.
The developer overhead saving is real and significant.

---

### Dimension 3: Execution Performance

**Estimates relative to hand-optimised C++ (100% baseline).**
These are NOT benchmarks. They are informed estimates based on:
- Language implementation strategy
- Known overhead sources
- Published benchmark results where available

| Language  | Speed     | Notes                                        |
|-----------|-----------|----------------------------------------------|
| C++23     | 100%      | Baseline. Optimal with expert use.           |
| Zig 0.13  | 97-100%   | Near-identical. Explicit alloc beats ARC.    |
| Rust 1.7x | 95-100%   | Comparable. Monomorphisation + LLVM.         |
| Odin      | 90-97%    | Fast. No GC. Some abstraction overhead.      |
| Skev      | 90-95%    | ARC atomic ops. LLVM optimised. Honest est.  |
| C# 13 AOT | 80-90%    | Good but GC overhead remains.                |
| Go 1.22   | 70-80%    | GC pauses are the primary limiter.           |
| GDScript  | 10-30%    | Interpreted. C# backend faster.             |
| Lua 5.4   | 10-30%    | Interpreted. LuaJIT: 60-80%.                |
| Python 3.13| 5-15%   | Interpreted. NumPy ops: back to C++ speed.   |

**GC worst-case frame impact:**

| Language  | GC Pause (worst case) | Impact at 60fps (16ms budget) |
|-----------|-----------------------|-------------------------------|
| C++23     | 0ms                   | None                          |
| Rust      | 0ms                   | None                          |
| Zig       | 0ms                   | None                          |
| Odin      | 0ms                   | None                          |
| Skev      | 0ms                   | None                          |
| Go 1.22   | 0.1-1ms               | Occasional micro-stutter      |
| C# 13     | 0.5-5ms               | Noticeable spikes             |
| C# Unity  | 1-50ms (Gen2)         | Visible stutter in games      |
| Python    | 1-100ms               | Not realtime-usable           |

---

### Dimension 4: Error Handling

**The test:** Read a file, validate its content, parse an integer,
validate the range, return result or specific error.

```cpp
// C++23 — using std::expected
#include <expected>
#include <fstream>
#include <string>

enum class ParseError { FileNotFound, EmptyFile, NotANumber, OutOfRange };

std::expected<int, ParseError> parse_health_file(const std::string& path) {
    std::ifstream file(path);
    if (!file) return std::unexpected(ParseError::FileNotFound);
    std::string line;
    if (!std::getline(file, line) || line.empty())
        return std::unexpected(ParseError::EmptyFile);
    try {
        int val = std::stoi(line);
        if (val < 0 || val > 9999)
            return std::unexpected(ParseError::OutOfRange);
        return val;
    } catch (...) {
        return std::unexpected(ParseError::NotANumber);
    }
}
// 18 lines total — std::expected is C++23 only. try/catch still needed.
```

```csharp
// C# 13 — using discriminated union pattern
public enum ParseError { FileNotFound, EmptyFile, NotANumber, OutOfRange }
public record Result<T>(T? Value, ParseError? Error) {
    public bool IsSuccess => Error is null;
}

public static Result<int> ParseHealthFile(string path) {
    if (!File.Exists(path))
        return new(default, ParseError.FileNotFound);
    var line = File.ReadAllText(path).Trim();
    if (string.IsNullOrEmpty(line))
        return new(default, ParseError.EmptyFile);
    if (!int.TryParse(line, out var val))
        return new(default, ParseError.NotANumber);
    if (val < 0 || val > 9999)
        return new(default, ParseError.OutOfRange);
    return new(val, null);
}
// 16 lines total — custom Result because C# has no built-in
```

```rust
// Rust 1.7x — using Result<T, E>
use std::fs;
use std::num::ParseIntError;

#[derive(Debug)]
enum ParseError { FileNotFound, EmptyFile, NotANumber, OutOfRange }

fn parse_health_file(path: &str) -> Result<i32, ParseError> {
    let content = fs::read_to_string(path)
        .map_err(|_| ParseError::FileNotFound)?;
    let line = content.trim();
    if line.is_empty() { return Err(ParseError::EmptyFile); }
    let val: i32 = line.parse().map_err(|_| ParseError::NotANumber)?;
    if val < 0 || val > 9999 { return Err(ParseError::OutOfRange); }
    Ok(val)
}
// 14 lines total — ? operator is clean. map_err adds noise.
```

```zig
// Zig 0.13 — using error unions
const std = @import("std");

const ParseError = error{ FileNotFound, EmptyFile, NotANumber, OutOfRange };

fn parseHealthFile(path: []const u8, alloc: std.mem.Allocator) ParseError!i32 {
    const content = std.fs.cwd().readFileAlloc(alloc, path, 1024)
        catch return ParseError.FileNotFound;
    defer alloc.free(content);
    const line = std.mem.trim(u8, content, "\n\r ");
    if (line.len == 0) return ParseError.EmptyFile;
    const val = std.fmt.parseInt(i32, line, 10)
        catch return ParseError.NotANumber;
    if (val < 0 or val > 9999) return ParseError.OutOfRange;
    return val;
}
// 14 lines total — error union is clean. Allocator param adds noise.
```

```odin
// Odin — using multiple return values
import "core:os"
import "core:strconv"
import "core:strings"

ParseError :: enum { None, FileNotFound, EmptyFile, NotANumber, OutOfRange }

parse_health_file :: proc(path: string) -> (int, ParseError) {
    data, ok := os.read_entire_file(path)
    if !ok do return 0, .FileNotFound
    line := strings.trim_space(string(data))
    if len(line) == 0 do return 0, .EmptyFile
    val, parse_ok := strconv.parse_int(line)
    if !parse_ok do return 0, .NotANumber
    if val < 0 || val > 9999 do return 0, .OutOfRange
    return val, .None
}
// 12 lines total — clean. Multiple return is Odin's pattern.
// Callers CAN ignore the error — Odin does not enforce handling.
```

```swift
// Skev — using result[T] with enforcement
kind ParseError >>
    file_not_found
    empty_file
    not_a_number
    out_of_range
<< ParseError

parse_health_file(path: string) -> result[int]
    content :: string = -> await file.read_text(path)
        or fail ParseError.file_not_found
    line :: string = content.trim()
    if line == "" >>
        fail ParseError.empty_file
    << line == ""
    value :: int = line.as(int) or_else fail ParseError.not_a_number
    if value < 0 or value > 9999 >>
        fail ParseError.out_of_range
    << value < 0 or value > 9999
    succeed value
<< parse_health_file
// 16 lines total — result[T] enforced by compiler. Unhandled = error.
```

**Error handling ratings:**

| Metric                | C++23 | C# 13 | Rust | Zig | Odin | Skev |
|-----------------------|-------|-------|------|-----|------|------|
| Compiler enforcement  | 2     | 3     | 5    | 4   | 2    | 5    |
| Verbosity             | 3     | 3     | 4    | 4   | 5    | 4    |
| Ignore-ability        | Bad   | OK    | Good | OK  | Bad  | Best |
| Type expressiveness   | 4     | 3     | 5    | 4   | 3    | 5    |
| Learning curve        | 3     | 4     | 3    | 4   | 5    | 4    |

**Critical honest note on Odin:** Odin's multiple-return error pattern
is clean to write but callers CAN ignore the error return silently.
The compiler does not enforce handling. This is a real safety gap.
Skev's `result[T]` is designed so unhandled results are compile errors.
Rust's `Result<T,E>` with `#[must_use]` is the strongest enforcement.

---

### Dimension 5: Game-Native Features

| Feature              | C++23 | C# 13 | Rust | Zig | Odin | GDScript | Skev |
|----------------------|-------|-------|------|-----|------|----------|------|
| Vector3 built-in     | ❌    | ❌    | ❌   | ❌  | ❌   | ✅       | ✅   |
| Color built-in       | ❌    | ❌    | ❌   | ❌  | ❌   | ✅       | ✅   |
| Transform built-in   | ❌    | ❌    | ❌   | ❌  | ❌   | ✅       | ✅   |
| Entity system        | ❌    | ❌    | ❌   | ❌  | ❌   | ✅       | ✅   |
| Event system         | ❌    | ❌    | ❌   | ❌  | ❌   | ✅       | ✅   |
| Hot reload           | ❌    | 🟡    | ❌   | ❌  | ✅   | ✅       | ✅*  |
| WASM mod support     | ❌    | ❌    | ✅   | ✅  | ❌   | ❌       | ✅   |
| SIMD auto            | 🟡    | 🟡    | 🟡   | ✅  | ✅   | ❌       | ✅*  |
| ARC memory           | ❌    | ❌    | ❌   | ❌  | ❌   | 🟡       | ✅   |

```
✅  = supported natively
🟡 = partial / library required / workaround
❌  = not supported
*   = specified but not yet implemented in the compiler
```

**Honest Skev column note:** Every ✅ in the Skev column represents a
specification. The Python transpiler validates the semantics. The real
compiler is Milestone 3. GDScript's ✅ marks are working today.

**Honest overall:** Skev and GDScript are the only languages in this
comparison with game-native types and entity systems as first-class
language features. All others require libraries or engine scaffolding.
The difference: GDScript is engine-locked. Skev compiles standalone.

---

### Dimension 6: Safety Guarantees

| Safety Type         | C++23 | C# 13 | Rust | Zig | Odin | Skev |
|---------------------|-------|-------|------|-----|------|------|
| Buffer overflow      | ❌    | ✅    | ✅   | 🟡  | ❌   | ✅   |
| Use-after-free       | ❌    | ✅    | ✅   | ❌  | ❌   | ✅   |
| Null deref           | ❌    | 🟡    | ✅   | ❌  | ❌   | ✅   |
| Data races           | ❌    | 🟡    | ✅   | 🟡  | ❌   | 🟡   |
| Integer overflow     | ❌    | 🟡    | 🟡   | ✅  | ❌   | 🟡   |
| Unhandled errors     | ❌    | ❌    | ✅   | 🟡  | ❌   | ✅   |
| Type safety          | 🟡    | ✅    | ✅   | ✅  | 🟡   | ✅   |

**Honest Rust vs Skev safety comparison:**
```
Rust wins:
  → Memory safety is formally proven by the borrow checker
  → Data race prevention is proven at compile time
  → Integer overflow is checked in debug mode
  → Larger surface area of safety guarantees

Skev wins:
  → ARC is easier to use than borrow checker
  → result[T] enforcement matches Rust's must_use
  → maybe T is cleaner than Option<T>
  → Less cognitive overhead for equivalent safety

Honest call: Rust is technically safer. The borrow checker proves more
than ARC can. Skev is safer than C++/Odin/Zig without the complexity cost.
For safety-critical certification: Rust is the better choice today.
```

---

### Dimension 7: Ecosystem & Tooling

This is the most important dimension for a technology decision.
It is also where Skev is most honest about its limitations.

| Metric               | C++23  | C# 13   | Rust  | Zig    | Odin  | Go    | Skev   |
|----------------------|--------|---------|-------|--------|-------|-------|--------|
| Package registry     | 🟡     | ✅✅    | ✅✅  | 🟡     | ❌    | ✅    | ❌     |
| Build system         | 🔴     | ✅      | ✅✅  | ✅     | ✅    | ✅✅  | ✅*    |
| IDE support          | ✅✅   | ✅✅    | ✅    | 🟡     | 🟡    | ✅    | ❌*    |
| Testing built-in     | ❌     | ❌      | ✅    | ✅     | ❌    | ✅    | ✅*    |
| Debugger             | ✅✅   | ✅✅    | ✅    | 🟡     | 🟡    | ✅    | ❌*    |
| Community size       | ✅✅✅ | ✅✅✅  | ✅✅  | 🟡     | 🟡    | ✅✅  | ❌     |
| Game libraries       | ✅✅✅ | ✅✅    | 🟡    | ❌     | 🟡    | ❌    | ❌     |
| Documentation        | 🟡     | ✅✅    | ✅✅  | 🟡     | 🟡    | ✅✅  | 🟡*    |

```
❌*  = specified in Chapter 11 but not yet built
✅*  = specified and validated via transpiler — real compiler pending
🟡*  = partially defined
```

**The honest ecosystem summary:**

```
C++:  30 years. Everything exists. Build systems are a disaster.
      NuGet: N/A. vcpkg/Conan: functional. CMake: necessary evil.

C# 13: NuGet is the best package manager in this comparison.
       Visual Studio + Rider: world-class. No equals.
       Game libraries: Unity Asset Store is unmatched.

Rust: crates.io is excellent. Cargo is excellent.
      VS Code + rust-analyzer: strong.
      Game ecosystem: growing but thin vs C++/C#.

Zig:  No central registry yet. zig build is good.
      IDE support: early. Community: growing.

Odin: No registry. Vendor directory model.
      IDE support: limited. Community: small but active.

Go:   go modules: solid. go doc: excellent.
      IDE: VS Code + gopls: strong.
      Game libraries: thin.

Skev: NOTHING EXISTS YET.
      This is the largest single gap vs every language here.
      A language without libraries is an island.
```

---

### Dimension 8: Learning Curve

**Time to write a working game entity (rough estimate, zero prior experience):**

| Language | Time to first working entity | Primary obstacle          |
|----------|------------------------------|---------------------------|
| GDScript | 2-4 hours                    | Learning Godot editor     |
| Python   | 1-2 hours                    | No game entity concept    |
| C# Unity | 4-8 hours                    | Unity architecture        |
| Odin     | 1-2 days                     | Manual memory             |
| Go       | 1-2 days                     | No entity concept         |
| Skev     | 4-8 hours                    | >> << block syntax        |
| Zig      | 2-3 days                     | Allocators + comptime     |
| Rust     | 1-2 weeks                    | Borrow checker            |
| C++      | 2-4 weeks                    | Everything                |

**Concepts a developer must learn before being productive:**

```
C++:   Pointers, references, RAII, templates, undefined behaviour,
       build systems, header files, linking. 12+ concepts minimum.

C# 13: Classes, properties, delegates, async/await, GC pressure.
       5-6 concepts. Unity adds its own layer on top.

Rust:  Ownership, borrowing, lifetimes, traits, Result, async.
       6-8 concepts. The borrow checker fights intuition initially.

Zig:   Explicit allocators, comptime, error unions, defer.
       5-6 concepts. Comptime is powerful but non-obvious.

Odin:  Procedures, structs, context, pointers, multiple returns.
       5-6 concepts. Cleaner than C++ but still manual memory.

Skev:  entity, data, kind, when, >>, <<, result[T], maybe, ARC basics.
       7-8 concepts. The >> << syntax is the primary learning hurdle.
       Once learned: significantly easier than everything above it.
```

**Honest learning curve verdict:**
Skev's initial learning curve is higher than GDScript and C# Unity
but lower than Rust and C++. The >> << block syntax is unfamiliar
to every developer. After the first hour: the value becomes apparent.
The entity/when/has model reduces the conceptual load compared to
manually wiring up an ECS in C++ or Rust.

---

## PART 5 — CODE COMPARISON TABLES

### Problem C — Concurrent Producer/Consumer

**Context:** A game event queue. One system produces collision events.
Another consumes them. Must be thread-safe. No data races.

```cpp
// C++23 — concurrent event queue
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <optional>

struct CollisionEvent { int entity_a; int entity_b; float force; };

class EventQueue {
    std::queue<CollisionEvent> queue;
    std::mutex mtx;
    std::condition_variable cv;
public:
    void push(CollisionEvent e) {
        std::lock_guard<std::mutex> lock(mtx);
        queue.push(e);
        cv.notify_one();
    }
    std::optional<CollisionEvent> pop() {
        std::unique_lock<std::mutex> lock(mtx);
        if (queue.empty()) return std::nullopt;
        auto e = queue.front(); queue.pop();
        return e;
    }
};
// 21 lines total — manual mutex, condition variable, lock_guard ceremony
// Data race if mutex omitted — compiler does NOT catch this
```

```csharp
// C# 13 — concurrent event queue
using System.Collections.Concurrent;

public record CollisionEvent(int EntityA, int EntityB, float Force);

public class EventQueue {
    private readonly ConcurrentQueue<CollisionEvent> _queue = new();

    public void Push(CollisionEvent e) => _queue.Enqueue(e);

    public CollisionEvent? Pop() {
        _queue.TryDequeue(out var e);
        return e;
    }
}
// 10 lines total — ConcurrentQueue handles thread safety
// Cleaner than C++ but: no compiler guarantee of correct usage
```

```rust
// Rust 1.7x — concurrent event queue
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct CollisionEvent { entity_a: u32, entity_b: u32, force: f32 }

struct EventQueue {
    inner: Arc<Mutex<Vec<CollisionEvent>>>,
}

impl EventQueue {
    fn new() -> Self { Self { inner: Arc::new(Mutex::new(Vec::new())) } }

    fn push(&self, event: CollisionEvent) {
        self.inner.lock().unwrap().push(event);
    }

    fn pop(&self) -> Option<CollisionEvent> {
        let mut q = self.inner.lock().unwrap();
        if q.is_empty() { None } else { Some(q.remove(0)) }
    }
}
// 17 lines total — Arc<Mutex<>> is the standard pattern
// Data races are compile-time impossible — this is Rust's strongest point
```

```zig
// Zig 0.13 — concurrent event queue
const std = @import("std");

const CollisionEvent = struct { entity_a: u32, entity_b: u32, force: f32 };

const EventQueue = struct {
    queue: std.ArrayList(CollisionEvent),
    mutex: std.Thread.Mutex = .{},
    allocator: std.mem.Allocator,

    pub fn init(alloc: std.mem.Allocator) EventQueue {
        return .{ .queue = std.ArrayList(CollisionEvent).init(alloc),
                  .allocator = alloc };
    }

    pub fn push(self: *EventQueue, event: CollisionEvent) !void {
        self.mutex.lock(); defer self.mutex.unlock();
        try self.queue.append(event);
    }

    pub fn pop(self: *EventQueue) ?CollisionEvent {
        self.mutex.lock(); defer self.mutex.unlock();
        return if (self.queue.items.len == 0) null
               else self.queue.orderedRemove(0);
    }
};
// 20 lines total — explicit allocator, mutex is value type
// Zig does not prevent data races — discipline required
```

```odin
// Odin — concurrent event queue
import "core:sync"

CollisionEvent :: struct { entity_a, entity_b: int, force: f32 }

EventQueue :: struct {
    queue:  [dynamic]CollisionEvent,
    mutex:  sync.Mutex,
}

queue_push :: proc(q: ^EventQueue, e: CollisionEvent) {
    sync.mutex_lock(&q.mutex); defer sync.mutex_unlock(&q.mutex)
    append(&q.queue, e)
}

queue_pop :: proc(q: ^EventQueue) -> (CollisionEvent, bool) {
    sync.mutex_lock(&q.mutex); defer sync.mutex_unlock(&q.mutex)
    if len(q.queue) == 0 do return {}, false
    e := q.queue[0]
    ordered_remove(&q.queue, 0)
    return e, true
}
// 16 lines total — clean. defer for unlock is nice.
// No data race protection — discipline required (same as Zig/C++)
```

```swift
// Skev — concurrent event queue using built-in channel[T]
data CollisionEvent >>
    entity_a :: int
    entity_b :: int
    force    :: float
<< CollisionEvent

entity PhysicsSystem >>
    collision_channel :: channel[CollisionEvent]

    when physics_update(delta)
        detect_collisions(delta)
    << physics_update

    detect_collisions(delta: float) -> nothing
        event :: CollisionEvent = CollisionEvent >>
            entity_a :: 1
            entity_b :: 2
            force    :: 45.5
        << CollisionEvent
        collision_channel.send(event)
    << detect_collisions

<< PhysicsSystem

entity AudioSystem >>
    collision_channel :: channel[CollisionEvent]

    when scene_load
        task process_collisions >>
            loop while true >>
                event :: CollisionEvent = -> await collision_channel.receive()
                audio.play_at(sounds.collision, event.force)
            << while true
        << task
    << scene_load

<< AudioSystem
// 34 lines total — channel[T] is typed, thread-safe by construction
// Data races are structurally impossible: channel enforces ordering
// No mutex visible. No lock_guard. Thread safety is the default.
```

**Concurrency ratings:**

| Metric              | C++23 | C# 13 | Rust | Zig | Odin | Skev |
|---------------------|-------|-------|------|-----|------|------|
| Data race safety    | 1     | 3     | 5    | 2   | 2    | 4    |
| Verbosity           | 2     | 4     | 3    | 3   | 4    | 4    |
| Mental model        | 2     | 3     | 3    | 3   | 4    | 5    |
| Performance         | 5     | 4     | 5    | 5   | 5    | 4    |
| Deadlock prevention | 1     | 2     | 3    | 1   | 1    | 4    |

---

### Problem D — Generic Algorithm

**Context:** Find the maximum value in a list of any comparable type.

```cpp
// C++23
template<typename T>
requires std::totally_ordered<T>
std::optional<T> find_max(const std::vector<T>& items) {
    if (items.empty()) return std::nullopt;
    return *std::max_element(items.begin(), items.end());
}
// 5 lines total — requires concept is C++20. Clean.
// Template error messages: still notorious.
```

```rust
// Rust 1.7x
fn find_max<T: Ord>(items: &[T]) -> Option<&T> {
    items.iter().max()
}
// 3 lines total — trait bound, iterator. Concise. Returns reference (lifetime implied).
```

```zig
// Zig 0.13 — comptime
fn findMax(comptime T: type, items: []const T) ?T {
    if (items.len == 0) return null;
    var max = items[0];
    for (items[1..]) |item| {
        if (item > max) max = item;
    }
    return max;
}
// 7 lines total — comptime T is a value, not a type parameter annotation.
// More verbose than Rust but: comptime is more powerful than trait bounds.
```

```odin
// Odin
find_max :: proc(items: []$T) -> (T, bool) where intrinsics.type_is_ordered(T) {
    if len(items) == 0 do return {}, false
    max := items[0]
    for item in items[1:] {
        if item > max do max = item
    }
    return max, true
}
// 6 lines total — $T is Odin's polymorphic parameter. Clean.
```

```swift
// Skev
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
// 11 lines total — [T where T: Comparable] is explicit constraint.
// More verbose than Rust (3 lines) but more readable than Zig (7 lines).
```

**Generics ratings — honest:**

| Metric              | C++23 | C# 13 | Rust | Zig  | Odin | Skev |
|---------------------|-------|-------|------|------|------|------|
| Power               | 5     | 3     | 4    | 5    | 3    | 3    |
| Readability         | 3     | 4     | 4    | 3    | 4    | 5    |
| Error messages      | 1     | 3     | 4    | 3    | 3    | 4*   |
| Verbosity           | 3     | 3     | 5    | 3    | 4    | 3    |
| Compile performance | 1     | 3     | 2    | 4    | 4    | 4*   |

```
*Estimated — real compiler not yet built.

Honest call: Zig's comptime is the most powerful generics system in this list.
C++ templates are powerful but hostile. Rust traits are the best balance of
power and readability. Skev's [T where T: Constraint] is more readable than
all of them but LESS POWERFUL than Zig and C++.

This is a conscious Skev trade-off: readability over metaprogramming power.
For a game language, this is the right trade-off.
```

---

## PART 6 — THE FULL SCORING MATRIX

**Scale: 5 = best-in-class. 1 = significant weakness. All ratings honest.**

| Dimension              | C++23 | C# 13 | Rust | Zig  | Odin | Go  | GDScript | Lua | Skev |
|------------------------|-------|-------|------|------|------|-----|----------|-----|------|
| **SYNTAX**             |       |       |      |      |      |     |          |     |      |
| Beginner clarity       | 1     | 3     | 2    | 3    | 3    | 4   | 5        | 4   | 4    |
| Noise per feature      | 2     | 3     | 3    | 3    | 4    | 4   | 5        | 4   | 5    |
| Large file nav         | 2     | 3     | 3    | 3    | 3    | 4   | 3        | 2   | 5    |
| **MEMORY**             |       |       |      |      |      |     |          |     |      |
| No GC pauses           | 5     | 3     | 5    | 5    | 5    | 2   | 2        | 2   | 5    |
| No manual free         | 1     | 5     | 5    | 1    | 1    | 5   | 5        | 5   | 5    |
| Developer overhead     | 1     | 4     | 2    | 2    | 3    | 5   | 5        | 5   | 5    |
| **PERFORMANCE**        |       |       |      |      |      |     |          |     |      |
| Execution speed        | 5     | 4     | 5    | 5    | 4    | 3   | 1        | 1   | 4    |
| Compile speed          | 1     | 3     | 1    | 4    | 5    | 5   | 5        | 5   | 4    |
| Hot reload             | 1     | 2     | 1    | 1    | 4    | 1   | 5        | 3   | 4*   |
| **ERROR HANDLING**     |       |       |      |      |      |     |          |     |      |
| Compiler enforcement   | 2     | 3     | 5    | 4    | 2    | 2   | 2        | 1   | 5    |
| Verbosity              | 3     | 3     | 4    | 4    | 5    | 2   | 3        | 3   | 4    |
| **GAME FEATURES**      |       |       |      |      |      |     |          |     |      |
| Game-native types      | 1     | 1     | 1    | 1    | 1    | 1   | 5        | 1   | 5    |
| Entity system          | 1     | 1     | 1    | 1    | 1    | 1   | 5        | 1   | 5    |
| Event system           | 1     | 2     | 1    | 1    | 1    | 1   | 5        | 1   | 5    |
| **SAFETY**             |       |       |      |      |      |     |          |     |      |
| Memory safety          | 1     | 4     | 5    | 2    | 2    | 4   | 4        | 3   | 4    |
| Null safety            | 1     | 3     | 5    | 1    | 1    | 1   | 3        | 1   | 5    |
| Concurrency safety     | 1     | 3     | 5    | 2    | 2    | 3   | 3        | 2   | 4    |
| **ECOSYSTEM**          |       |       |      |      |      |     |          |     |      |
| Package registry       | 2     | 5     | 5    | 2    | 1    | 4   | 3        | 2   | 1    |
| Community size         | 5     | 5     | 4    | 2    | 2    | 5   | 3        | 3   | 1    |
| Game libraries         | 5     | 4     | 2    | 1    | 2    | 1   | 4        | 3   | 1    |
| IDE support            | 5     | 5     | 4    | 2    | 2    | 4   | 4        | 2   | 1*   |
| **LEARNING**           |       |       |      |      |      |     |          |     |      |
| Beginner ramp          | 1     | 3     | 1    | 2    | 3    | 4   | 5        | 4   | 4    |
| Concept count          | 1     | 3     | 2    | 3    | 4    | 5   | 4        | 5   | 3    |
| **TOTALS**             | **54**|**82** |**73**|**61**|**64**|**71**|**86**|**57**|**93***|

```
* Skev's total includes 4* ratings (specified but unbuilt).
  If those become 1 (not available): Skev total = ~72.
  As designed and specified: 93. As currently available: ~72.
  Honest Skev today: between Zig and C# in the ranking.

This table should be read as: "What does Skev promise?"
not "What can you use Skev for today?"
```

---

## PART 7 — WHEN TO CHOOSE EACH LANGUAGE

### Choose C++23 when:
```
✅ Maximum raw performance is non-negotiable (AAA game engine)
✅ You have a large team with C++ expertise
✅ You need to integrate with existing C++ codebases
✅ You need maximum control over every allocation
✅ Your target platform requires it (console certification, legacy systems)

❌ Do not choose C++ when:
   → You are a solo developer or small team
   → Developer velocity matters more than 5% extra performance
   → Memory safety requirements exist
   → You are starting a new project from scratch
```

### Choose C# 13 when:
```
✅ You are making a Unity game
✅ Your team already knows C#
✅ NuGet ecosystem access matters
✅ You prefer managed memory and accept GC trade-offs
✅ You need world-class IDE support (Visual Studio, Rider)

❌ Do not choose C# when:
   → GC pauses are unacceptable (VR, realtime simulation)
   → You are not in the Unity/Microsoft ecosystem
   → Maximum performance is required
```

### Choose Rust when:
```
✅ Memory safety proofs are required (safety-critical systems)
✅ WebAssembly is a primary target
✅ You need the strongest concurrency safety guarantees
✅ You are building infrastructure (game engine, server)
✅ Your team can invest in the learning curve

❌ Do not choose Rust when:
   → Developer velocity is the primary concern
   → Team has no Rust experience and deadline is tight
   → You want hot reload during development
   → Game logic (not engine) is the primary concern
```

### Choose Zig 0.13 when:
```
✅ You are replacing C in an existing C project
✅ You are writing a game engine (not game logic)
✅ Explicit allocation control is critical (embedded, microcontrollers)
✅ You need comptime metaprogramming power
✅ Cross-compilation is essential

❌ Do not choose Zig when:
   → Zig 1.0 stability is required (breaking changes still possible)
   → You need a rich library ecosystem
   → You want game-native abstractions
   → Your team is not comfortable with manual memory
```

### Choose Odin when:
```
✅ You want what Skev promises but need it to work TODAY
✅ You are a C programmer who wants a better experience
✅ Data-oriented design is a primary architecture goal
✅ Hot reload during development is essential
✅ You want game development without engine lock-in

❌ Do not choose Odin when:
   → Memory safety matters (Odin is manual memory)
   → Compiler-enforced error handling is required
   → Community size is a factor in your decision
```

### Choose Go when:
```
✅ You are building a game backend (matchmaking, analytics, lobby)
✅ Concurrency is the primary design concern
✅ Compile speed is critical for team iteration
✅ You want the simplest possible deployment (single binary)
✅ Your team already knows Go

❌ Do not choose Go for:
   → Game client/rendering code (GC pauses)
   → Realtime simulation loops
   → Anywhere null safety matters
```

### Choose GDScript when:
```
✅ You are making a Godot game
✅ Rapid prototyping is the goal
✅ The team includes non-programmers
✅ Engine integration matters more than portability

❌ Do not choose GDScript when:
   → You need to run code outside Godot
   → Performance is critical (use GDExtension C++ instead)
   → Type safety across large codebases matters
```

### Choose Lua / LuaJIT when:
```
✅ You need an embeddable scripting language in a C++ engine
✅ Modding by end users is the goal
✅ The script runtime must be tiny and portable
✅ LuaJIT performance is acceptable (60-80% of C++)

❌ Do not choose Lua when:
   → Type safety matters at scale
   → The codebase will grow beyond 10K lines of script
```

### Choose Skev when:
```
✅ You want game-native types (Vector3!, Color!) as language primitives
✅ ARC memory (no GC, no manual free) is the right trade-off
✅ Compiler-enforced error handling (result[T]) matters
✅ Null safety (maybe T) is required
✅ You are starting a new project and can wait for Milestone 3
✅ The team wants Python-like readability at C-like performance
✅ You want a language designed FOR your domain, not adapted to it
✅ You are willing to build the ecosystem or wait for it to grow

❌ Do not choose Skev when:
   → You need a working compiler today (Milestone 3 is pending)
   → Ecosystem matters (libraries, packages, community)
   → Your project starts in the next 6-12 months
   → You cannot afford to be on a language at day 1
   → Odin exists and ships today — and matches your needs
```

---

## PART 8 — SKEV'S HONEST POSITION

### What Skev Wins — No Hedging

```
1. GAME-NATIVE TYPES — unique
   Vector3!, Color!, Transform! as language primitives.
   Auto-SIMD. Always available. No import.
   No other general-purpose language in this comparison has these.

2. ARC MEMORY — best balance
   No GC pauses (C++ parity).
   No manual free (C# parity).
   No borrow checker complexity (unique).
   The combination does not exist anywhere else.

3. RESULT[T] ENFORCEMENT — joint strongest with Rust
   Unhandled results are compile errors.
   Odin/C++/Zig all allow silent error dropping.
   Skev and Rust are the only languages where ignoring errors
   is impossible. Skev's syntax is cleaner.

4. NULL SAFETY — stronger than C++/Odin/Zig/Go
   maybe T removes null from the language.
   No null pointer dereference is possible.
   Joint with Rust and C# (with nullable enabled).

5. READABILITY AT SCALE — unique mechanism
   >> << labels at every nesting level.
   At 500+ lines: you always know where you are.
   No other language has this specific mechanism.

6. PURPOSE-BUILT — not adapted
   C++ was built for systems. Adapted to games.
   C# was built for enterprise. Adapted to games (Unity).
   Rust was built for systems. Adapting to games.
   Go was built for cloud. Not adapting to games.
   Skev was built for games, XR, simulations from day one.

7. ENTITY/EVENT SYSTEM — built in
   Not a library. Not a pattern. Part of the language.
   entity, when, has are keywords.
   No other general-purpose compiled language has this.
```

### What Skev Loses — No Hedging

```
1. ECOSYSTEM — worst in this comparison
   Zero packages. Zero libraries. Zero community.
   Every team must build their own standard library usage from scratch.
   This is the single largest practical barrier to adoption.
   No spin makes this smaller. It is a real and significant gap.

2. COMPILER — not yet built
   The Python transpiler validates semantics.
   The real compiler (Rust/LLVM) is Milestone 3.
   Every other language here has a compiler.
   Skev's performance claims are estimates until proven.

3. SHIPPED PRODUCTS — zero
   Odin: multiple indie games shipped.
   Zig: Bun (JavaScript runtime) — production use.
   Vale: nothing shipped.
   Skev: nothing shipped.
   Battle-testing catches things specifications miss.

4. GENERICS POWER — weaker than Zig and C++
   Zig's comptime is more powerful than [T where T: Comparable].
   C++ templates are turing-complete.
   Skev's generics are more readable but less flexible.
   This is a deliberate trade-off. It is still a trade-off.

5. HOT RELOAD — specified but unbuilt
   Odin's hot reload works today in production.
   Skev's hot reload is in Chapter 11 of the spec.
   The design is correct. The implementation is pending.

6. SAFETY CERTIFICATION — no path yet
   Rust has a path to ISO 26262 certified use (Ferrocene).
   C++ has MISRA-C++.
   Skev has no certified toolchain. Not possible until Milestone 3+.

7. COMMUNITY — the chicken-and-egg problem
   Language success requires community.
   Community requires a working language.
   Skev must solve this sequentially.
   Every language on this list went through this phase.
   Go's community grew from 0 to millions.
   The path exists. It takes years.
```

### The Honest Positioning Statement

```
Skev is the best-designed language for games and real-time systems
that does not yet have a working compiler.

Among languages that exist today and target the same space:
  Odin is the closest and most practical alternative.
  Rust offers stronger safety at higher complexity cost.
  C++ offers maximum ecosystem at maximum complexity cost.

Skev's bet is: readability + safety + game-native types in one package
will be compelling enough to build an ecosystem from scratch.

Every successful language made this bet.
Every successful language went through this phase.
The design is solid. The work is ahead of us.
```

---

## APPENDIX A — VERSION REFERENCE

| Language  | Version assessed | Release date  | Notes                    |
|-----------|-----------------|---------------|--------------------------|
| C++       | C++23           | 2023          | ISO/IEC 14882:2024       |
| C#        | 13 / .NET 9     | 2024          |                          |
| Rust      | 1.7x            | 2024-2025     | Stable channel           |
| Zig       | 0.13            | 2024          | Pre-1.0. Breaking changes possible |
| Odin      | Latest          | Ongoing       | No version system        |
| Jai       | Demo            | Never released| [JAI-DEMO] throughout    |
| Go        | 1.22            | 2024          |                          |
| GDScript  | 4.x             | 2023          | Godot 4 series           |
| Lua       | 5.4 / LuaJIT 2.1| 2020/2023     |                          |
| Vale      | 0.2             | 2023          | Pre-production           |
| Python    | 3.13            | 2024          |                          |
| AngelScript| 2.36           | 2023          |                          |
| Skev      | 1.0 spec        | 2026          | Compiler: Milestone 3    |

---

## APPENDIX B — DOMAIN WINNER TABLE

| Domain               | Best today    | Best designed  | Skev position |
|----------------------|---------------|----------------|---------------|
| Games                | C++ / C# / Odin | Skev / Jai  | Primary target |
| XR/VR/AR             | C++ / C#      | Skev           | Primary target |
| Simulations          | C++ / Python  | Skev / Julia   | Strong candidate |
| Robotics/Embedded    | C / Zig       | Zig / Skev     | Strong candidate |
| Animation/VFX        | C++ / Python  | Skev (partial) | Partial fit |
| Interactive Apps     | C# / JS       | Skev (partial) | Secondary |
| Real-time Networking | Go / Rust     | Rust / Go      | Secondary |

---

*Copyright © 2026 AJ. All Rights Reserved. https://skev.dev | https://skev.org*
*This document may be reproduced with attribution to AJ and skev.dev.*
