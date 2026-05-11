<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev — Weaknesses Solvable Before the Compiler
## Pre-Compiler Action Plan: What Can Be Fixed Now
**Version:** 1.0
**Authors:** AJ (Copyright © 2026)
**Source:** Skev_Weakness_Analysis.md + Skev_Language_Comparison_v2.md
**Purpose:** Identify every weakness addressable before Milestone 3 (compiler)

---

> **The core question:** The compiler is the critical path. It cannot be
> shortcut. But not every weakness requires a compiler to fix.
> This document identifies exactly what can be addressed right now —
> and in what order to maximise return per hour of work.

---

## SUMMARY — 11 Solvable Items

```
TIER 1 — BUILD IT (deliverable artifacts):
  1. TextMate grammar      skev.tmLanguage.json
  2. VS Code extension     skev.vscode-skev (syntax highlighting only)
  3. GitHub Linguist PR    .skev file recognition on GitHub
  4. Prism.js plugin       skev.dev documentation highlighting

TIER 2 — WRITE IT (spec additions and documentation):
  5. SOA syntax            spec addition to Chapter 3
  6. Integer overflow      strengthen spec to fully defined
  7. Data race detection   strengthen Chapter 5 safety rules
  8. Cookbook / examples   common game patterns in Skev
  9. skev.dev content      landing page, roadmap, spec published
 10. Transpiler programs   grow the compliance test suite
 11. Community setup       GitHub org, Discord, social presence
```

---

## PRIORITY ORDER — Return Per Hour of Work

```
1. TextMate grammar        → fixes every editor immediately, enables 2/3/4
2. VS Code extension       → developer-facing, publishable, permanent
3. GitHub Linguist PR      → .skev recognised on GitHub forever
4. SOA spec addition       → closes design gap before compiler locks it in
5. Integer overflow spec   → closes safety gap at zero cost
6. Data race spec          → closes safety gap at zero cost
7. Prism.js plugin         → skev.dev code blocks look right
8. Transpiler programs     → grows compliance test suite
9. skev.dev content        → somewhere to send interested developers
10. Cookbook / examples    → learning resource for early adopters
11. Community setup        → ready to receive early adopters when ready
```

---

## TIER 1 — BUILD IT

### Item 1 — TextMate Grammar

```
File:     skev.tmLanguage.json
Repo:     github.com/skev-lang/skev-vscode
Scope:    source.skev
```

**What weakness it fixes:**
```
The >> << :: when has kind syntax highlights as nothing in every
editor today. The spec uses ```swift as a temporary approximation.
Skev-specific syntax (>>, <<, ::) is not highlighted.
Skev keywords (when, has, kind) only partially match swift.
```

**What you get:**
```
→ Correct Skev syntax highlighting in: VS Code, Sublime Text,
  Zed, Nova, Atom, Neovim, and any TextMate-compatible editor
→ All spec documents switch from ```swift to ```skev
→ The locked colour scheme (Skev_Color_Scheme.md) applies everywhere
→ GitHub renders Skev code correctly via the linguist entry
→ Enables items 2, 3, and 4 — grammar is the foundation
```

**Token categories to implement (all colours locked):**

| Token                  | Scope                     | Colour    | Hex       |
|------------------------|---------------------------|-----------|-----------|
| Keywords               | keyword.control.skev      | Warm Red  | `#FF7B72` |
| Game-native types (!)  | support.type.game-native  | Bright Blue | `#79C0FF` |
| Operators (>> << :: ->) | keyword.operator.skev    | Warm Orange | `#FF9E64` |
| Block labels (<< text) | entity.name.label.skev    | Bright Green | `#7EE787` |
| Strings                | string.quoted.double.skev | Light Blue | `#A5D6FF` |
| Numbers                | constant.numeric.skev     | Bright Blue | `#79C0FF` |
| Line comments (#)      | comment.line.skev         | Muted Grey | `#8B949E` |
| Doc comments (#!)      | comment.block.documentation.skev | Green | `#3FB950` |
| Type names (PascalCase)| entity.name.type.skev     | Lavender  | `#D2A8FF` |
| Properties (after ::)  | variable.other.property.skev | Amber  | `#FFA657` |

**Compiler needed?** ❌ No. Grammar is pure regex pattern matching
against source text. Zero language semantics required.

**Effort:** One session.

---

### Item 2 — VS Code Extension

```
Extension ID:  skev.vscode-skev
Display name:  Skev Language Support
Publisher:     skev-lang
Marketplace:   marketplace.visualstudio.com
```

**What weakness it fixes:**
```
IDE support rated 1/5 — lowest possible rating.
No IDE presence at all.
A developer who finds Skev has no tooling to work with.
```

**What you get:**
```
→ Published to VS Code Marketplace — searchable and installable
→ Syntax highlighting via the TextMate grammar (Item 1)
→ .skev file association — files open with correct icon and highlight
→ Snippet support — common Skev patterns as snippets
→ Professional presence — "Skev Language Support" exists
```

**What this extension does NOT include yet:**
```
→ LSP (autocomplete, go-to-definition, type errors) — requires compiler
→ Debugger integration — requires compiler
→ Hot reload integration — requires compiler
These features are added to the extension after Milestone 3.
The extension structure is already correct — features are added,
not rebuilt.
```

**Compiler needed?** ❌ No. Syntax highlighting only. LSP client is
added to the same extension after the compiler exists.

**Effort:** Grammar first, then package.json wrapper + publish.
One session after Item 1 is complete.

---

### Item 3 — GitHub Linguist PR

```
Target repo:  github/linguist
File:         lib/linguist/languages.yml
Entry name:   Skev
Extension:    .skev
Grammar:      grammar from Item 1
```

**What weakness it fixes:**
```
.skev files on GitHub show as "unknown" — grey, unrecognised.
The spec documents, transpiler tests, example programs, future compiler
repo — every file with .skev code shows as unrecognised.
The comparison document notes Skev does not exist in any syntax
highlighter. This fixes it permanently at the source.
```

**What you get:**
```
→ GitHub detects and highlights .skev files correctly
→ Language statistics in repos (% Skev)
→ .skev appears in GitHub's language filter/search
→ Every spec example, every future repo looks correct
→ Permanent — applies retroactively to all .skev files on GitHub
```

**The languages.yml entry:**
```yaml
Skev:
  type: programming
  color: "#FF7B72"
  extensions:
    - ".skev"
  tm_scope: source.skev
  ace_mode: text
  language_id: [assigned by GitHub]
```

**Compiler needed?** ❌ No. Linguist only needs the file extension
and the TextMate grammar scope.

**Effort:** One PR. Review time depends on GitHub's queue (days to weeks).

---

### Item 4 — Prism.js Plugin

```
File:  prism-skev.js
Usage: skev.dev documentation site
```

**What weakness it fixes:**
```
Documentation rated 🟡* (partially defined).
All code examples on skev.dev currently render as plain text
or use a mismatched highlighter.
The site should use the locked colour scheme from day one.
```

**What you get:**
```
→ Every code block on skev.dev highlights correctly
→ Uses the same locked colour scheme as the VS Code extension
→ Consistency: one colour scheme, all tools — per the spec requirement
→ Professional documentation site from launch day
```

**Compiler needed?** ❌ No. Prism.js is pure client-side JavaScript
pattern matching — same approach as TextMate, different format.

**Effort:** One session after Item 1. Prism follows TextMate's
token structure closely — largely a format translation.

---

## TIER 2 — WRITE IT

### Item 5 — SOA Syntax Spec Addition

**What weakness it fixes:**
```
"Skev does not have built-in SOA (Structure of Arrays)" —
stated directly in the Odin vs Skev comparison.
Odin: SOA is first-class. A primary design goal.
Jai (demo): SOA transformations at the language level.
Skev: SOA is currently a pattern — not a language feature.

Why SOA matters for games:
  AOS (default): entities[i].x, entities[i].y, entities[i].z
    → Cache misses when processing one field across many entities

  SOA (Odin native): positions.x[i], positions.y[i], positions.z[i]
    → Cache-friendly for SIMD — measurable speedup in:
       particle systems, physics, animation, crowd simulation
```

**What you get:**
```
→ SOA syntax locked in the spec before the compiler is written
→ Compiler implements SOA correctly from day one
→ Gap vs Odin addressed at the design level
→ The weakness disappears from the comparison table
```

**Proposed spec addition:**
```swift
# SOA declaration — compiler lays out as Structure of Arrays
soa entity ParticleSystem >>
    position :: Vector3!     # stored as: x[], y[], z[]
    velocity :: Vector3!     # stored as: x[], y[], z[]
    lifetime :: float        # stored as: lifetime[]
    active   :: bool         # stored as: active[]
<< ParticleSystem

# AOS (default, unchanged):
entity Player >>
    position :: Vector3!
    health   :: int
<< Player
```

**Compiler needed?** ❌ No. Spec-level design only. No implementation
required until Milestone 3.

**Effort:** Design decisions document + Chapter 3 addition.
Same process as every previously locked decision.

---

### Item 6 — Integer Overflow — Strengthen the Spec

**What weakness it fixes:**
```
Integer overflow currently rated 🟡 partial in the safety matrix.
Zig: ✅ fully defined — wrapping, saturating, and trapping modes.
Skev: the spec does not fully define overflow semantics.

This means:
  → The compiler author must make a decision we haven't made
  → Different compiler authors may make different decisions
  → The comparison table permanently shows 🟡 until the spec locks it
```

**What you get:**
```
→ Fully locked integer overflow semantics
→ Compiler implements exactly what the spec says
→ Rating improves from 🟡 to ✅ in the safety matrix
→ No ambiguity for the Milestone 3 compiler author
```

**Proposed spec addition (3 modes, matches Zig's approach):**
```swift
# Default: panic on overflow in debug, wrapping in release
health :: int = 200 + 200    # debug: PANIC  release: wraps

# Explicit wrapping (developer opts in):
score :: int = score +% delta    # always wraps — no panic

# Explicit saturating:
volume :: int = volume +| delta  # clamps at max, never overflows

# Explicit checked (always panics):
health :: int = health +! damage  # panic if overflow — any build
```

**Compiler needed?** ❌ No. Spec and design decisions only.

**Effort:** One design decisions document. Chapter 3 type system update.

---

### Item 7 — Data Race Detection — Strengthen the Spec

**What weakness it fixes:**
```
Data races rated 🟡 partial.
Rust: ✅ data races proven impossible at compile time.
Skev: channel[T] prevents data races structurally for the
      standard pattern, but the spec does not formally define
      ALL the rules. Shared mutable state outside channels
      is not fully governed.
```

**What you get:**
```
→ Formal rules for what the compiler must reject
→ The channel[T] guarantee is explicit and specified
→ shared property access rules are locked
→ Rating improves from 🟡 toward ✅
→ The compiler author has clear rules to implement
```

**Proposed spec rules to lock:**
```
Rule 1: Any shared mutable state accessed from multiple
        when handlers MUST be declared shared.

Rule 2: shared properties are atomic — compiler generates
        atomic read/write operations.

Rule 3: Passing an entity reference across a channel[T]
        requires the entity to be the sole writer —
        compiler enforces the transfer.

Rule 4: Mutable local variables are never shared — they
        live on the calling entity's stack only.
```

**Compiler needed?** ❌ No. Spec and Chapter 5 update only.

**Effort:** Chapter 5 additions + design decisions document.

---

### Item 8 — Cookbook / Extended Examples

**What weakness it fixes:**
```
Documentation rated 🟡* (partially defined).
The spec chapters show examples but answer "what does Skev do?"
not "how do I solve X in Skev?"

A developer trying Skev via the transpiler today has:
  → The spec (formal, dense)
  → The 417 test programs (good but not tutorial-style)
  → Nothing cookbook-style

This is a real friction point for early adopters.
```

**What you get:**
```
→ 20-30 common game patterns solved in Skev
→ Every example validated by the transpiler (417 tests grow)
→ Searchable, linkable, shareable
→ The language at its best — each example chosen to show a win
```

**Proposed cookbook sections:**
```
Chapter A: Entity Patterns
  → Health system with events
  → Inventory with weight limits
  → State machine with guards
  → Observer pattern via when

Chapter B: Error Handling Patterns
  → Multi-step validation chain
  → Error recovery with or_else
  → Propagation with ->
  → Custom error kinds with context

Chapter C: Concurrency Patterns
  → Producer/consumer via channel[T]
  → Task-based loading
  → Shared state with shared properties

Chapter D: Data Patterns
  → Immutable game config with fixed
  → Save/load with serialisable
  → Migration with migration block

Chapter E: Game-Specific Patterns
  → Camera follow with lerp
  → Damage calculation with clamp
  → AI state machine
  → Achievement tracking
```

**Compiler needed?** ❌ No. The Python transpiler validates every
example. These programs run today.

**Effort:** Ongoing. One example per session, validated by transpiler.

---

### Item 9 — skev.dev Content

**What weakness it fixes:**
```
Community does not exist partly because there is nowhere to
send a developer who discovers Skev.

Today, if someone finds Skev:
  → No website
  → No clear explanation of what it is
  → No roadmap
  → No place to follow progress
  → No way to engage

This is a near-zero cost weakness to address.
```

**What you get:**
```
→ A home for Skev that exists before the compiler
→ Clear "what is Skev and why does it exist?"
→ Honest roadmap: spec complete → transpiler done → compiler next
→ Links to spec, transpiler, comparison document
→ Early interest can accumulate before compiler ships
→ The comparison document and weakness analysis published here
```

**Proposed skev.dev sections:**
```
/           Landing page — what Skev is, one-sentence pitch
/why        The honest case — why another language, who it's for
/spec       The full specification (Chapters 1-11 + 3.5)
/compare    The language comparison v2
/roadmap    Honest milestones with status
/play       (post-compiler) Online playground
/docs       (post-compiler) API documentation
```

**Compiler needed?** ❌ No. Writing and web work only.

**Effort:** Writing. No technical prerequisite.

---

### Item 10 — Transpiler Test Suite — More Programs

**What weakness it fixes:**
```
Shipped products = zero.
The Python transpiler is the only runnable Skev that exists.
Every additional end-to-end program:
  → Validates more of the spec
  → Grows the compliance suite the real compiler must pass
  → Demonstrates real Skev programs work correctly
  → Is a "shipped" Skev program in the only available sense
```

**What you get:**
```
→ More real Skev programs proven to work via the transpiler
→ The 417 test count grows toward a larger compliance suite
→ The compiler author has more test cases to pass
→ More cookbook material (Items 8 and 10 overlap)
```

**Proposed new program targets:**
```
Program 9:  Entity Component System (full ECS pattern in Skev)
Program 10: Network Message Handler (channel + async)
Program 11: Level Loading Pipeline (async + result[T] chain)
Program 12: AI Behaviour Tree (state + event + match)
Program 13: Save File System (serialisable + migration)
Program 14: Audio Manager (shared + event + entity)
Program 15: Physics Integration (Vector3! + delta time)
```

**Compiler needed?** ❌ No. Python transpiler runs all of these.

**Effort:** One session per program. Directly continues Milestone 2.

---

### Item 11 — Community Infrastructure

**What weakness it fixes:**
```
Community does not exist.
There is currently no place for interested developers to:
  → Ask questions about Skev
  → Follow progress
  → Contribute early feedback
  → Build in public alongside us

This infrastructure costs almost nothing to create and
can accumulate interest before the compiler ships.
```

**What you get:**
```
→ GitHub organisation: github.com/skev-lang
  → skev-spec repo (specification documents)
  → skev-transpiler repo (Python transpiler, 417 tests)
  → skev-vscode repo (VS Code extension)
  → skev-examples repo (cookbook programs)
  → skev-compiler repo (Milestone 3, when work begins)

→ Discord server or GitHub Discussions
  → #general, #spec-questions, #transpiler, #roadmap

→ Social presence
  → Where the Skev community will announce milestones
```

**Compiler needed?** ❌ No. Setup and maintenance work only.

**Effort:** One session to create the GitHub org and repos.
Ongoing: community management.

---

## WHAT CANNOT BE FIXED BEFORE THE COMPILER

This section exists to be explicit. Everything below requires a working
Rust/LLVM compiler. No workaround exists.

| Item                     | Why it needs the compiler                        |
|--------------------------|--------------------------------------------------|
| IDE LSP                  | Type information requires type checking          |
| Autocomplete             | Requires type checker and symbol resolution      |
| Go-to-definition         | Requires symbol table from compilation           |
| Inline error display     | Requires compiler diagnostics                    |
| Debugger                 | Requires debug info in compiled output           |
| Hot reload               | Requires shared library compilation model        |
| SIMD auto-vectorisation  | Requires LLVM vectoriser                         |
| Ecosystem (libraries)    | Developers need compiler to build libraries      |
| Performance validation   | 90-95% claim requires real benchmarks            |
| Safety certification     | Requires certified compiler toolchain            |
| Community at scale       | Follows compiler — not before it                 |
| Shipped products         | Requires compiler to build real applications     |

---

## RESOLUTION TABLE — Complete Picture

| Weakness                 | Solvable Now? | Action                           |
|--------------------------|---------------|----------------------------------|
| No syntax highlighting   | ✅ YES        | TextMate grammar                 |
| No VS Code extension     | ✅ YES        | Build + publish (syntax only)    |
| No GitHub recognition    | ✅ YES        | Linguist PR                      |
| No doc site highlighting | ✅ YES        | Prism.js plugin                  |
| SOA not in language      | ✅ YES        | Spec addition (Chapter 3)        |
| Integer overflow partial | ✅ YES        | Spec addition (Chapter 3)        |
| Data races partial       | ✅ YES        | Spec addition (Chapter 5)        |
| Thin documentation       | ✅ YES        | Cookbook + examples              |
| No web presence          | ✅ YES        | skev.dev content                 |
| Small test suite         | ✅ YES        | More transpiler programs         |
| No community home        | ✅ YES        | GitHub org + Discord setup       |
| No IDE LSP               | ❌ No         | Requires compiler (M3)           |
| No debugger              | ❌ No         | Requires compiler (M3)           |
| No hot reload            | ❌ No         | Requires compiler (M3)           |
| No SIMD                  | ❌ No         | Requires LLVM (M3)               |
| No ecosystem             | ❌ No         | Requires compiler + community    |
| No performance proof     | ❌ No         | Requires compiler + benchmarks   |
| No safety certification  | ❌ No         | Requires certified compiler      |
| Generics power gap       | ❌ No         | Deliberate trade-off, accepted   |
| ARC overhead             | ❌ No         | Deliberate trade-off, accepted   |
| No shipped products      | ❌ No         | Requires compiler + community    |

---

## THE HONEST FRAMING

```
These 11 items do not close the largest weaknesses.

The largest weaknesses are:
  1. No compiler
  2. No ecosystem
  3. No community

None of these 11 items fixes those.

What these 11 items do:
  → Remove friction for developers who discover Skev early
  → Ensure the spec is fully tight before the compiler is built
    (spec fixes cost an afternoon — compiler fixes cost weeks)
  → Establish Skev's presence before the compiler ships
  → Grow the compliance test suite for Milestone 3
  → Start the community clock early

The compiler is still the critical path.
These items run in parallel — not instead of the compiler.
```

---

*Copyright © 2026 AJ. All Rights Reserved. https://skev.dev | https://skev.org*
*Derived from: Skev_Weakness_Analysis.md + Skev_Language_Comparison_v2.md*
