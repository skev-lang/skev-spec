<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language — Weakness Analysis
## Complete Category-by-Category Gap Register
**Version:** 1.0
**Authors:** AJ (Copyright © 2026) & Claude (Anthropic)
**Source:** Skev_Language_Comparison_v2.md
**Purpose:** Internal audit — every weakness documented, nothing softened
**Date:** 2026

---

> **Purpose of this document:** Every weakness found in the full language
> comparison, organised by category. This is the internal gap register —
> the honest list of what needs to be built, fixed, or accepted.
> Nothing in this document is softened. Nothing is omitted.

---

## PRIORITY STACK — Highest Impact First

```
1.  COMPILER            — blocks everything else
2.  ECOSYSTEM           — the largest practical adoption barrier
3.  TOOLING             — IDE, debugger needed for developer experience
4.  HOT RELOAD          — Odin has it today in production
5.  SAFETY CERTIFICATION — needed for robotics/medical/embedded
6.  SOA                 — needed for serious game performance work
7.  GPU COMPUTE         — needed for simulation/VFX domains
8.  GENERICS POWER      — Zig comptime gap is real
9.  COMMUNITY           — follows from compiler shipping
10. SHIPPED PRODUCTS    — follows from community
```

---

## Category 1 — MATURITY

**The root problem. Every other weakness flows from this.**

```
→ Real compiler (Rust/LLVM): NOT YET BUILT — Milestone 3 pending
→ This comparison document compares Skev's DESIGN against other
  languages' IMPLEMENTATIONS — a genuine and stated limitation
→ Skev's performance claims (90-95% of C++) are estimates only —
  unproven until the real compiler exists and is benchmarked
→ Every other language in the comparison has a working compiler
  that has shipped real products
```

**Resolution path:** Milestone 3 (Rust/LLVM compiler). No shortcut.

---

## Category 2 — SHIPPED PRODUCTS

**Zero shipped products across all domains.**

```
→ Zero shipped games
→ Zero shipped XR/VR/AR applications
→ Zero shipped simulations in production
→ Zero shipped robotics deployments
→ Zero shipped anything

Battle-testing catches things that specifications miss.
Skev has had no battle-testing.

Comparison (honest):
  Odin:  multiple indie games shipped and in production
  Zig:   Bun (JavaScript runtime) — real-world production use
  Vale:  nothing shipped — but Vale is more experimental than Skev
  Skev:  nothing shipped
```

**Resolution path:** Follows from compiler (Milestone 3) → community forms
→ early adopters ship projects → proof of concept becomes proof of production.

---

## Category 3 — ECOSYSTEM

**The largest practical adoption barrier. Rated 1/5 across all metrics.**

### Missing Core Infrastructure

```
→ Zero packages in any registry
→ Zero third-party libraries
→ Zero community (does not exist yet)
→ Every team using Skev must build from scratch
→ "A language without libraries is an island" — stated directly
   in the comparison document
```

### Missing Domain-Specific Libraries

```
Networking:
  → skev.network: specified in Chapter 7, not yet built
  → No equivalent to Go's net/http or Rust's Tokio

XR / VR / AR:
  → No XR-specific runtime library
  → No OpenXR binding
  → No spatial audio library
  → No hand-tracking library

Graphics / Rendering:
  → No GPU compute pipeline
  → No CUDA equivalent
  → No offline rendering pipeline
  → No shader pipeline

Simulation / Science:
  → No numerical computing library (NumPy equivalent)
  → No scientific library (SciPy equivalent)
  → No statistical computing (R/MATLAB equivalent)
  → No data visualisation (Matplotlib equivalent)
  → No data analysis (Pandas equivalent)

Animation / VFX:
  → No DCC tool API (Maya/Houdini expect Python or C++)
  → VFX library ecosystem does not exist

AI / ML:
  → No ML framework (PyTorch/TensorFlow/JAX equivalent)
  → Python is irreplaceable here for the foreseeable future
```

### Scoring Matrix Results

| Metric           | Score | Context                              |
|------------------|-------|--------------------------------------|
| Package registry | 1/5   | Does not exist                       |
| Community size   | 1/5   | Does not exist                       |
| Game libraries   | 1/5   | Does not exist                       |
| IDE support      | 1/5   | Specified but not built              |

**Resolution path:** Ecosystem follows compiler. Compiler enables early
adopters. Early adopters contribute libraries. No shortcut exists.
The realistic timeline: years, not months.

---

## Category 4 — TOOLING

**Specified in Chapter 11. Not yet built.**

### Items Specified But Unbuilt

```
→ IDE support (VS Code extension, LSP server):
    Specified in Chapter 11. Not yet built. Rated 1/5.

→ Debugger:
    Specified in Chapter 11. Not yet built. Rated 1/5.

→ Hot reload:
    Specified in Chapter 11.
    Not yet implemented in any compiler.
    The design is correct. The implementation is pending.

→ SIMD auto-vectorisation:
    Specified — game-native types auto-SIMD.
    Not yet implemented (requires LLVM compiler).

→ Build system (skev build/run/test):
    Specified in Chapter 9. Validated via transpiler.
    Real compiler pending.
```

### Honest Comparison

```
Odin hot reload:  works today in production. Production-tested.
                  Multiple shipped games use it.
Skev hot reload:  correctly designed. Implementation pending.

Rust (rust-analyzer + VS Code): strong, working today.
Skev LSP:                       specified, not built.
```

**Resolution path:** Milestone 3 (compiler) enables LSP, debugger, SIMD.
Hot reload is a parallel workstream — can potentially start during
compiler development. TextMate grammar is buildable right now (no compiler
needed) — this is the highest ROI near-term tooling task.

---

## Category 5 — PERFORMANCE GAPS

### Gap 1 — ARC Overhead vs Manual Memory

```
→ ARC atomic reference counts have a small overhead vs manual free
→ In microbenchmarks, Zig and C++ will beat Skev
→ Execution speed estimated at 90-95% of C++ — not 100%
→ Skev concurrency performance rated 4/5, not 5/5

Honest ranking by estimated speed:
  C++:   100% (baseline)
  Zig:   97-100%
  Rust:  95-100%
  Odin:  90-97%
  Skev:  90-95%
```

### Gap 2 — Embedded Specifically

```
→ ARC atomic operations cost matters on microcontrollers
  where every CPU cycle has a real power and timing cost
→ Zig wins on embedded: no runtime at all, smaller binary,
  explicit allocators — zero hidden overhead
→ Skev's ARC runtime has a minimum footprint that may be
  unacceptable on very constrained devices
```

### Gap 3 — GPU Compute

```
→ No GPU compute pipeline
→ No CUDA equivalent (CUDA is fundamentally C++)
→ GPU-parallel simulations not supported
→ Offline rendering (path tracing, ray tracing) not supported
→ This rules Skev out of the GPU-heavy simulation and VFX domains
```

**What this means:** Skev is not trying to beat C++ on raw speed.
The 90-95% estimate is accepted. ARC overhead is a conscious trade-off
for developer safety. The embedded gap is a domain limitation, not a
design flaw. GPU compute is a future work item.

---

## Category 6 — SAFETY GAPS

### Gap 1 — Weaker than Rust

```
→ Data races: Skev = 🟡 partial. Rust = ✅ proven at compile time.
→ Integer overflow: Skev = 🟡 partial. Zig = ✅ fully checked.
→ Memory safety: Skev = 4/5. Rust = 5/5.
→ Concurrency safety: Skev = 4/5. Rust = 5/5.

Stated directly in the comparison:
  "Rust is technically safer. The borrow checker proves more than ARC can."
  "For safety-critical certification: Rust is the better choice today."
```

### Gap 2 — No Safety Certification Path

```
→ Skev has no certified toolchain
→ ISO 26262 / IEC 62443 certification requires a certified compiler
→ Not possible until Milestone 3 is complete and subsequently
  submitted for certification (years of additional work)

Comparison:
  Rust: Ferrocene — ISO 26262 certified Rust toolchain (exists today)
  C++:  MISRA-C++ — widely used in automotive/medical
  Ada:  DO-178C certified for aerospace
  Skev: No path until after Milestone 3
```

**Impact:** Skev cannot legally be used in certified medical devices,
certified automotive systems, or certified aerospace systems today.
This is a real domain restriction for robotics/medical/embedded.

---

## Category 7 — GENERICS POWER

**A deliberate trade-off that is still a trade-off.**

```
→ Generics power rated 3/5 — joint lowest with C# 13 and Odin
→ Zig's comptime is the most powerful metaprogramming in the comparison
  and is genuinely more powerful than Skev's [T where T: Comparable]
→ C++ templates are turing-complete — Skev cannot match this
→ Skev's generics are more readable but LESS FLEXIBLE

Stated directly in the comparison:
  "Zig's comptime is more powerful than [T where T: Comparable].
   C++ templates are turing-complete.
   Skev's generics are more readable but less flexible.
   This is a deliberate trade-off. It is still a trade-off."

What Skev cannot do that Zig comptime can:
  → Run arbitrary code at compile time
  → Use types as first-class values
  → Generate code based on runtime data structures known at compile time
  → Hardware register definitions via comptime
  → Compile-time reflection beyond what the spec defines
```

**Assessment:** This is accepted. The trade-off (readability over
metaprogramming power) is right for a game language. A game developer
should not need turing-complete templates to write game logic.
The gap vs Zig matters more in engine code than game code.

---

## Category 8 — DATA-ORIENTED DESIGN

**A missing feature that matters for game performance.**

```
→ Skev does not have built-in SOA (Structure of Arrays)
→ Odin: SOA is first-class — a primary design goal
→ Jai (demo): SOA transformations at the language level
→ Skev: SOA is a pattern, not a language feature

Why SOA matters:
  AOS (Array of Structs — Skev default):
    entities[i].x, entities[i].y, entities[i].z
    → Cache misses when only processing one field across many entities

  SOA (Structure of Arrays — Odin native):
    positions.x[i], positions.y[i], positions.z[i]
    → Cache-friendly for SIMD operations on one field at a time
    → Significant performance difference in particle systems, physics

Stated directly:
  "SOA (Skev does not have built-in SOA)"
  This appears in the Odin vs Skev honest comparison.
```

**Assessment:** This is a real gap for high-performance game systems
(particle systems, physics, animation) where SOA gives measurable
speedups. Future spec addition to consider.

---

## Category 9 — LEARNING CURVE

**Higher than Odin. Lower than Rust and C++.**

```
→ 7-8 concepts required before productive
  (more than Odin's 5-6, less than Rust's 6-8)

→ The >> << block system is unfamiliar to every developer
  from every other language in the comparison
  → "First contact is jarring"
  → After 30 minutes: value becomes apparent
  → Upfront cost is real

→ Odin deliberately resists adding features — fewer concepts
  Skev adds the entity/event system — more concepts to learn

→ Concept count rated 3/5 in the scoring matrix

Concepts required before productive in Skev:
  entity, data, kind, when, >>, <<, result[T], maybe T, ARC basics
  = 7-8 minimum
```

**Assessment:** Acceptable for the target. The entity/event system
adds concepts but removes boilerplate. The >> << syntax pays dividends
at scale. The learning curve is higher than GDScript but serves a
different (more capable) developer target.

---

## Category 10 — DOMAIN-SPECIFIC WEAKNESSES

### Domain: Games

```
→ No ecosystem — every library must be built from scratch
→ No shipped games — no proof that real games can be built with Skev
→ Hot reload: specified but unbuilt
  (Odin's hot reload is production-tested today)
→ Odin is the realistic current choice for the same target audience
```

### Domain: XR / VR / AR

```
→ No XR-specific runtime library
→ No OpenXR binding (OpenXR is the standard XR API)
→ No shipped XR applications
→ Library gap described as "real and significant"
→ Architecture is correct (no GC, game-native types)
  but library reality prevents actual XR development
```

### Domain: Simulations

```
→ Cannot compete for scientific HPC
  Python/NumPy/PyTorch ecosystem is irreplaceable — stated directly
→ Statistical simulations: R, MATLAB, Julia win
→ GPU-parallel simulations: CUDA is C++, not Skev
→ Legacy simulations: typically Fortran — no migration path
→ Julia wins on linear algebra vs Skev
```

### Domain: Robotics / Embedded

```
→ No microcontroller support
  (requires LLVM target port + runtime port — not done)
→ ARC overhead is a real cost on embedded targets
→ No certified toolchain (blocks medical/automotive robotics)
→ Document verdict: "For deep embedded (microcontrollers, bare metal):
  Zig or C. Not Skev."
→ Zig wins: no runtime, explicit allocators, comptime for register
  definitions, smaller binary, better C toolchain
```

### Domain: Animation / VFX

```
→ Maya/Houdini plugins require Python or C++ APIs — not Skev
→ No offline rendering pipeline
→ No GPU compute pipeline
→ VFX library ecosystem does not exist
→ Skev does not compete with Python in DCC tool scripting
→ Relevant only for game-adjacent / real-time VFX work
```

### Domain: Interactive Applications

```
→ Chapter 11 UI capabilities are specified but minimal
→ Not designed for standard business software (CRUD apps)
→ C# 13 (MAUI) wins for Microsoft ecosystem desktop apps
→ JavaScript wins for web applications
→ Swift wins for iOS/macOS native apps
→ Skev adds value only for game-like interactive experiences,
  not for conventional application development
```

### Domain: Real-time Networking

```
→ No networking library built yet
  (skev.network is specified in Chapter 7, not implemented)
→ Go's networking ecosystem is irreplaceable today
  (goroutines, net/http, standard library — world-class)
→ Rust's Tokio (proven high-performance async server stack) wins
→ Skev cannot compete with Go's networking ecosystem today
→ Document verdict: "Skev is the application layer, not the
  infrastructure layer — at least until the ecosystem grows"
```

---

## Category 11 — COMMUNITY

**The chicken-and-egg problem.**

```
→ Community does not exist
→ Community size rated 1/5 — lowest possible rating
→ Language success requires community
→ Community requires a working language
→ Skev must solve these sequentially — one unlocks the other
→ The path exists: Go grew from 0 to millions of developers
→ The timeline: years, not months
→ Cannot be shortcut or bypassed
```

**The sequential dependency chain:**
```
Compiler ships (M3)
  → Early adopters can use Skev
    → First libraries appear
      → Community forms around shared problems
        → More libraries
          → More developers
            → First shipped games
              → Credibility established
                → Broader adoption
```

---

## Category 12 — THE DOCUMENT'S OWN VERDICT

**Part 7, Part 8 of the comparison document — stated directly.**

### "Do Not Choose Skev When" — Verbatim

```
→ You need a working compiler today (Milestone 3 is pending)
→ Ecosystem matters (libraries, packages, community)
→ Your project starts in the next 6-12 months
→ You cannot afford to be on a language at day 1
→ Odin exists, ships today, and matches your needs
```

### The Honest Positioning Statement — Verbatim

```
"Skev is the best-designed language for games and real-time systems
that does not yet have a working compiler.

Among languages that exist today and target the same space:
  Odin is the closest and most practical alternative.
  Rust offers stronger safety at higher complexity cost.
  C++ offers maximum ecosystem at maximum complexity cost."
```

### Scoring Matrix Honest Reality

```
Skev as designed (spec score):   93/125
Skev as available today (score): ~72/125

Today's honest ranking: between Zig (61) and C# (82)
Not between GDScript (86) and perfect (125)

The gap between 93 and 72 = the unbuilt items
Closing that gap = Milestone 3 + ecosystem development
```

---

## RESOLUTION SUMMARY

| Category              | Fixable Now? | Milestone | Notes                              |
|-----------------------|--------------|-----------|------------------------------------|
| Compiler              | No           | M3        | The critical path                  |
| Shipped products      | No           | Post-M3   | Follows from compiler              |
| Ecosystem             | No           | Post-M3   | Follows from compiler + community  |
| IDE / Debugger        | No           | M3        | Requires real compiler             |
| TextMate grammar      | **YES**      | Now       | No compiler needed                 |
| Hot reload            | No           | M3        | Requires compiler                  |
| SIMD auto             | No           | M3        | Requires LLVM                      |
| ARC overhead          | No           | Accepted  | Conscious trade-off                |
| GPU compute           | No           | Future    | Post-M3 research item              |
| Safety certification  | No           | Post-M3+  | Years of additional work           |
| Generics power        | Partially    | Accepted  | Deliberate trade-off               |
| SOA                   | Spec only    | Future    | Spec addition possible             |
| Data races (partial)  | Spec only    | Future    | Could be strengthened in spec      |
| Integer overflow       | Spec only    | Future    | Could be strengthened in spec      |
| Learning curve        | Partially    | Accepted  | >> << is by design                 |
| Community             | No           | Post-M3   | Follows from compiler + time       |

---

*Copyright © 2026 AJ. All Rights Reserved. skev.dev | skev.org*
*Source: Skev_Language_Comparison_v2.md — Full Honest Assessment*
