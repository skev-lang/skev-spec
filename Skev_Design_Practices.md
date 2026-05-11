<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Design Practices
## Official Design Partnership Guidelines
**Version:** 2.6
**Authors:** AJ (Copyright © 2026)
**Status:** Locked
**Updated:** Rule F v1.9 — comparison on every example. Rule H added — Trademark & Naming Conflict Prevention. skev.net renamed to skev.network as first Rule H application. Chapter Process updated to 8 steps including Rule H.

---

> These practices govern every language design decision made for Skev from this point forward. They exist because good language design is not accidental — it is disciplined, consistent, and honest.

---

## Skev Core Identity — Check This Before Everything Else

> **"Fast like C++ and C#. Easy to read like Python."**

This is the single most important rule in this document. It is checked before any other consideration. If a decision compromises either side of this statement — it is redesigned. No exceptions.

```swift
Fast like C++ and C#:
→ Compiled to native machine code via LLVM
→ No garbage collector — ARC is deterministic
→ Zero hidden runtime overhead
→ SIMD auto-vectorisation for game-native types
→ Stack allocation for value types

Easy to read like Python:
→ Code reads like natural language
→ No cryptic symbols without meaning
→ A non-developer understands intent immediately
→ Visual noise is minimised at every decision
→ Large files stay clear with >> << labels
```

---

## Practice 1 — Proactive Problem Detection

Before proposing any new syntax or feature:

- Analyse what every major language does in this area at its **current version**
- Identify exactly why each approach fails or succeeds
- Design Skev's solution to fix ALL known problems
- Present the solution already battle-tested before it reaches the table
- Test against ALL of Skev's use cases — not just games

```
You should never catch something I already knew.
```

---

## Practice 2 — The Seven Question Test

Before locking any language decision, seven questions must be answered in this exact order:

**Question 0 — Core Identity (answered FIRST)**
Does this maintain Skev's core identity?
- Fast like C++ and C#?
- Easy to read like Python?

If the answer to either is NO — stop. Redesign. Do not proceed to the other questions.

**Question 1 — Is it visually unambiguous?**
Can a developer read it at a glance with zero confusion?

**Question 2 — Is it consistent?**
Does it follow the same patterns already established in Skev? No exceptions. No special cases.

**Question 3 — Is it original?**
Does it feel like Skev — not like Python, C++, or C#?

**Question 4 — Does it scale?**
Does it work in a 10-line file AND a 10,000-line codebase?

**Question 5 — Is it compiler-enforceable?**
Can the compiler catch violations automatically without developer effort?

**Question 6 — Does it work for ALL of Skev's use cases?**
Not just games. Every use case we committed to:
- Game Development
- XR / VR / AR
- Real-time Simulations
- Robotics
- Animation / VFX Tools
- Interactive Applications
- Real-time Networking Applications

```
If any answer is NO — redesign.
We never compromise on these seven.
```

---

## Practice 3 — Lock Before Build

Nothing gets built until it is locked.
Nothing gets locked until it is challenged.

Every decision follows this exact flow:

```
PROPOSE  →  CHALLENGE  →  DEFEND or REVISE  →  LOCK
```

Once locked — that decision is referenced in every future decision that touches it. No contradictions. No inconsistencies. Ever.

---

## Practice 4 — The Consistency Bible

Every locked Skev decision maintained and cross-referenced against all future decisions.

### Core Syntax

| Decision | Rule |
|---|---|
| Block open | `>>` |
| Block close | `<< exact label` |
| Auto-close | `entity` and `component` only — Skev Studio generates immediately |
| Manual close | All inner blocks — compiler enforced |
| Property definition | `::` |
| Event handler | `when` |
| Component attachment | `has` |
| Game-native types | `!` suffix |
| Operators | Mandatory spacing enforced by compiler |
| Indentation | 4 spaces exactly — tabs illegal |

### Event System

| Decision | Rule |
|---|---|
| Duplicate events | Type-qualified labels `<< collision: TypeName` |
| Identical nesting | `:: suffix` disambiguation required |
| Match single arm | `->` arrow — no block needed |
| Match multi arm | `>>` and `<<` with arm name as label |
| kind matching | Exhaustive by default — `_` arm opts out |
| kind exhaustive reporting | Compiler reports ALL affected files across entire project |

### Memory & Types

| Decision | Rule |
|---|---|
| Memory model | ARC — Automatic Reference Counting |
| Compilation target | LLVM → Native machine code |
| ARC operations | Always atomic — thread safe |
| ARC in collections | Entity and component references tracked by ARC |
| data in collections | Always copied — value type — no ARC |
| kind types | Value type — no ARC — no heap allocation |
| Type conversion | Explicit only via `.as(Type)` — no implicit coercion |
| Circular references | `weak` keyword prevents cycles |
| weak implies maybe | `weak T` is always treated as `maybe T` |
| No runtime reflection | Skev has no runtime reflection |
| Compile-time reflection | Opt-in via `#! introspectable` — zero cost when unused |

### Collections

| Decision | Rule |
|---|---|
| Collection membership | `contains` keyword — never `in` (reserved for loops) |
| List literal | `[value1, value2, value3]` |
| Map literal | `["key1" -> value1, "key2" -> value2]` |
| Multi-line literal | Trailing comma allowed — no error |
| Empty literal | `[]` valid for both list and map |

### Function Syntax

| Decision | Rule |
|---|---|
| Function declaration | `name(params) -> ReturnType` — no keyword |
| Function body | No `>>` needed — body starts immediately after signature |
| Function close | `<< function_name` — same as all Skev blocks |
| Return value | `result value` for non-result return types |
| Return from result fn | `succeed value` / `fail ErrorType` |
| Return nothing | `result nothing` or omit — same meaning |
| Default parameters | `param: Type = default_value` |
| Function type syntax | `fn(Type, Type) -> ReturnType` |
| Method vs event | Method: no `when` keyword. Event: with `when` keyword. |
| Standalone function | Outside entity — file scope by default |
| Hidden function | `hidden` modifier — file-private |
| Implicit self | Methods inside entity access properties directly — no `self` |
| Tail call optimisation | Compiler optimises tail recursive calls to loops |

### Type System

| Decision | Rule |
|---|---|
| Pure data containers | `data` keyword — value type, no events, no components |
| Enumerated types | `kind` keyword — variants only, no properties |
| Null safety | `maybe TypeName` — compiler forces handling before access |
| Null check | `if value exists >>` |
| Null fallback | `value or fallback` |
| Clear a maybe | `value = nothing` |
| Generics | NOT YET DEFINED — Chapter 3.5 — critical gap |

### Memory Layout

| Decision | Rule |
|---|---|
| Entity layout | ARC count + dispatch ptr + component mask + properties + component data |
| Dispatch table | Shared per entity TYPE — not per instance |
| Stack allocation | Primitives, game-native, data ≤ 64 bytes, array, kind |
| Heap allocation | Entity, component, string, list, map, data > 64 bytes |
| Large data heap | Scoped heap — freed at block end — not ARC |
| ARC thread safety | All ARC operations are atomic |
| Auto-alignment | Compiler reorders for optimal packing — Studio shows dev order |
| Component inline | ≤ 256 bytes inline in entity — > 256 bytes pointer |
| scene.destroy() | Marks "destroying" — treated as nothing — ARC still tracks |
| Compile-time reflection | `#! introspectable` on data and kind only — never entity |

### Error Handling

| Decision | Rule |
|---|---|
| result[T] | Fallible operation return type — uses [] notation |
| succeed value | Creates successful result — zero-cost on success path |
| fail ErrorType | Creates failed result |
| fail X with ctx | Creates failed result with inline context |
| -> propagation | Propagates failure up — ARC cleans up — reuses -> |
| -> in when | Not permitted — must use async function or inline match |
| or_else | Fallback value or alternative for failed result |
| if succeed / if fail | Simple single-outcome result check |
| kind for errors | Error categories — reuses existing kind |
| data for errors | Error context — reuses existing data |
| Error unions | Inferred by compiler from -> propagation |
| Panic | Unrecoverable — cannot be caught — programming error only |
| assert | Developer invariant — panic if violated |
| engine.on_panic | Release build graceful exit hook |
| ARC + errors | ARC cleans up automatically on -> propagation |

### Concurrency

| Decision | Rule |
|---|---|
| Threading model | Three tiers — game loop, engine systems, developer tasks |
| Main thread | All entity property writes — single-threaded semantics |
| Engine systems | Physics, renderer, audio, networking — engine managed parallel |
| task keyword | Parallel work block — uses >> << syntax |
| task capture | Weak snapshots — does not increment ARC |
| await keyword | Yields async function — never blocks game loop |
| async keyword | Marks functions that can await |
| async in when | Only via async function call — never direct await in when |
| shared modifier | Marks properties safe for cross-thread access — atomic |
| channel[T] | Thread-safe message queue — default capacity 256 |
| channel overflow | Default: drop oldest — configurable |
| Double-buffer transforms | Engine handles automatically — invisible to developer |
| cancel keyword | Cooperative cancellation — sets flag — task must check |
| realtime modifier | Expert real-time tasks — heap/await/channel forbidden |
| Data race prevention | Compiler enforces thread ownership — cross-thread write = error |

### Type Boundaries

| Feature | `entity` | `component` | `data` | `kind` |
|---|---|---|---|---|
| Properties `::` | ✅ | ✅ | ✅ | ❌ |
| Events `when` | ✅ | ✅ | ❌ | ❌ |
| Components `has` | ✅ | ❌ | ❌ | ❌ |
| Engine access | ✅ | ✅ | ❌ | ❌ |
| ARC managed | ✅ | ✅ | ❌ | ❌ |
| Value type | ❌ | ❌ | ✅ | ✅ |
| Variants | ❌ | ❌ | ❌ | ✅ |
| Introspectable | ❌ | ❌ | ✅ | ✅ |

```
Every new decision must not contradict any entry above.
If it does — revise the new decision OR formally
reconsider the locked one. Both require explicit agreement.
```

---

## Practice 5 — Honest Over Exciting

When two options exist:

```
Option A — Exciting but risky
Option B — Solid but less flashy
```

Option B is always presented first with full reasoning. If Option A is genuinely worth the risk, that case is made clearly with complete justification.

```
We never hype a bad idea to seem innovative.
Skev wins on its own merits against current language versions —
not by comparing against outdated behaviour.
```

---

## Practice 6 — Flag Before It Becomes a Problem

If a decision will cause problems three chapters from now — it gets raised now. Not later.

### Mandatory Pre-Decision Checklist

Before presenting ANY proposal, every box must be checked:

```
STEP 0 — Core Identity
□ Fast like C++ and C# — performance maintained?
□ Easy to read like Python — readability maintained?

STEP 1 — Seven Question Test
□ Visually unambiguous?
□ Consistent with locked decisions?
□ Original — not copied from other languages?
□ Scales from 10 to 10,000 lines?
□ Compiler-enforceable?
□ Works for ALL Skev use cases?

STEP 2 — Use Case Checklist
□ Game Development
□ XR / VR / AR
□ Real-time Simulations
□ Robotics
□ Animation / VFX Tools
□ Interactive Applications
□ Real-time Networking

STEP 3 — Four Parameters Check (Rule A)
□ Speed:       Maintained or improved?
□ Reliability: Makes Skev more dependable?
□ Security:    No new vulnerability introduced?
□ Readability: Stays as clean as Python?

STEP 4 — Algorithm & Data Structure Check (Rule E)
□ Does this help common algorithms execute faster?
□ Does this make algorithms clearer to express?
□ Does this enable better data structure design?
□ Does this improve cache-friendliness of data?
□ Does this give the compiler more to optimise?

STEP 5 — Real-World Example Check (Rule F) — EVERY EXAMPLE
□ C++23 comparison written for EVERY real-world example?
□ C# 13 comparison written for EVERY real-world example?
□ Python 3.13 comparison written for EVERY real-world example?
□ No Skev example appears without all three comparisons?
□ Simple foundation example (🟢) written?
□ Applied game development example (🟡) written?
□ Hard complex example (🔴) written — mandatory?
□ Hard complex example meets 3+ complexity checklist items?
□ User-friendly walk-through written for complex example?
□ Plain English scenario explanation written?
□ Non-game use case with comparisons (at 🟡 minimum)?
□ Honest advantage statement on EVERY example?
□ Trade-offs acknowledged honestly — including where C++/C# wins?
□ Line count annotation (// X lines total) on every code block?
□ Implementation depth is EQUAL across all languages?
  (Full Skev = Full others, Abbreviated Skev = Abbreviated others)
□ If Skev is longer — is the reason explicitly stated?

STEP 6 — Naming & Trademark Check (Rule H)
□ Does this name conflict with a Microsoft trademark?
  (.NET, Azure, Windows, C#, DirectX, Xbox)
□ Does this name conflict with an Apple trademark?
  (Swift, Metal, ARKit, SceneKit, SpriteKit, Xcode)
□ Does this name conflict with a game engine name?
  (Unity, Unreal, Godot, GameMaker, CryEngine)
□ Does this name EVOKE an existing product even if not legally identical?
□ Does this name conflict with a common developer tool?
  (git, npm, pip, cargo, gradle, cmake)
□ Would a developer from another language misread this name?
If any YES — rename before presenting.

STEP 7 — Bidirectional Interop Check (Rule I)
□ If this feature involves language boundaries — is BOTH directions considered?
  → Direction A: Skev calling external code
  → Direction B: External code calling Skev
  → Direction C: Skev as an embeddable library
□ Is the interop tier correctly identified?
  (Tier 1: C ABI, Tier 2: Scripting, Tier 3: IPC, Tier 4: WASM)
□ Is the overhead of each direction honestly stated?
□ Are unsafe boundaries clearly marked in both directions?

STEP 8 — Consistency Bible
□ No contradiction with any locked decision
□ No contradiction with any written chapter
□ No interaction with any open gap in Rule C tracker?
```

If anything fails — redesign before presenting.

---

## Practice 7 — Real World Testing

Every syntax decision gets tested against three real scenarios before locking:

**Scenario 1 — Simple beginner code**
Does a developer who has never seen Skev understand this immediately?

**Scenario 2 — Large complex entity**
Does it still work cleanly at 500+ lines without losing clarity?

**Scenario 3 — Edge case abuse**
What happens if someone uses it wrong? Does the compiler catch it cleanly and explain it clearly?

```
If it fails any scenario — fix it before locking.
No exceptions.
```

---

## Practice 8 — Minimal Surface Area

Skev should have the smallest possible number of concepts a developer must learn.

Before adding any new keyword or symbol:
- Can an existing keyword handle this?
- Can an existing symbol handle this?
- Is this genuinely a new concept or a variation of something already in Skev?

Every new feature should feel like it was always part of Skev. Not bolted on. Not borrowed. Native.

---

## Practice 9 — Your Vision, My Expertise

| Who | Owns |
|---|---|
| AJ | The direction — what Skev should feel like |
| Claude | The execution — how to make it work technically |

```
This is partnership — not consulting.
```

---

## Practice 10 — Spec Integrity

The specification document is the single source of truth.

- Nothing enters the spec unreviewed
- Nothing leaves the spec without documented reason
- Every change is versioned
- Contradictions between chapters caught and resolved immediately
- The spec is always the source of truth — not the conversation history

---

## Rule A — The Four Parameters Check

**Speed** — Maintains C++ and C# level execution, zero hidden runtime cost, zero GC pauses.

**Reliability** — Makes Skev more dependable, reduces developer error, gives compiler more to check.

**Security** — No vulnerability introduced, no memory/type/injection risk, no sandboxing regression.

**Readability** — Stays as clean as Python, reduces visual noise, reads aloud naturally.

```
All four must pass. One failure = redesign.
```

---

## Rule B — The Core Identity Statement

```
"Fast like C++ and C#. Easy to read like Python."

Fast side — checked against C++23 and C# 13 / .NET 9
Readable side — checked against Python 3.13

Skev must maintain ~90%+ of C++ execution speed.
Skev must match or exceed Python readability.
Skev must beat Python at large file clarity.
```

---

## Rule C — Gap Tracker

Every known gap in Skev's architecture. Every gap has an assigned chapter. No gap is forgotten.

### 🚨 Critical Gaps — Skev Cannot Ship Without These

| Gap | What's Missing | Chapter | Impact |
|---|---|---|---|
| ~~Error Handling~~ | ~~No exceptions, no result types, no panic model defined~~ | ~~Chapter 6~~ | ✅ **RESOLVED in Chapter 6** |
| ~~Generics~~ | ~~Cannot write generic functions or generic data types~~ | ~~Chapter 3.5~~ | ✅ **RESOLVED in Chapter 3.5** |
| ~~Interoperability / FFI~~ | ~~Cannot call C/C++ libraries, no ABI defined~~ | ~~Chapter 8~~ | ✅ **RESOLVED in Chapter 8** |

### 🟠 High Priority Gaps — Needed Before Public Release

| Gap | What's Missing | Chapter | Impact |
|---|---|---|---|
| Standard Library | Math, string manipulation, file I/O, networking, serialisation — not specified | Chapter 7 | Usability — developers cannot build real programs |
| Sandboxing | No mod isolation, no plugin isolation, no untrusted code model | Chapter 10 | Security — games need mod support |
| Dependency Safety | Package signing, checksum verification, supply chain protection | Chapter 9 | Security — supply chain attacks are real |
| Testing Framework | No unit test system, no integration test model defined | Chapter 11 | Quality — cannot verify Skev programs |
| Linking & Loading | Static/dynamic linking, relocation, loader not specified | Chapter 9 | Build system incomplete |
| Backward Compatibility | No versioning policy, deprecation model, or breaking change rules | Chapter 9 | Evolution — language must be able to grow |

### 🟡 Medium Priority Gaps — Needed for Completeness

| Gap | What's Missing | Chapter | Impact |
|---|---|---|---|
| Static / Global Memory | shared modifier exists — full static/global memory model not specified | Chapter 4.5 | Edge cases for global configuration data |
| REPL / Playground | No interactive exploration mode defined | Chapter 11 | Developer experience — onboarding |
| Serialisation | No JSON, binary, or custom serialisation model | Chapter 7 | Needed for save files, networking, assets |
| Futures / Promises | async/await defined — Futures as first-class values not specified | Chapter 5 | Concurrency completeness |
| Scheduler Specification | Worker thread pool implied — not formally specified | Chapter 5 | Concurrency runtime |
| Memory Ordering | Atomic ARC defined — full memory consistency model not specified | Chapter 5 | Correctness for low-level concurrent code |
| Capabilities / Permissions | No capability model defined | Chapter 10 | Security for plugins and mods |
| Code Signing & Integrity | No code signing model defined | Chapter 9 | Distribution security |
| Observability | Logging and metrics not specified | Chapter 11 | Production debugging |

### Gap Resolution Rules

```
1. Critical gaps block chapter writing past their assigned chapter
   → Cannot write Chapter 7 standard library until
     Chapter 6 error handling is resolved
     (standard library functions need error handling model)

2. High priority gaps must be resolved before public beta

3. Medium priority gaps must be resolved before v1.0 release

4. Every new gap discovered gets added here immediately
   with chapter assignment and impact assessment

5. A gap is only removed from this tracker when:
   → It has been formally written into a chapter
   → That chapter has been approved
   → The Consistency Bible has been updated
```

---

## Rule D — Version Specificity

All language comparisons reference current versions:

| Language | Current Version |
|---|---|
| C++ | C++23 |
| C# | C# 13 / .NET 9 |
| Python | Python 3.13 |
| Rust | Rust 1.7x (reference for ownership/generics comparisons) |
| Go | Go 1.22 (reference for concurrency comparisons) |

```
Skev wins on its own merits against current versions.
Never compare against outdated behaviour
to make Skev look better than it is.
```

---

## Rule E — Algorithm & Data Structure Awareness

Every language design decision must consider how it affects algorithm execution and data structure design.

### Pre-Decision Algorithm Check

```
□ Does this make common algorithms faster to execute?
  → Spatial algorithms — pathfinding, physics, collision
  → State machine algorithms — AI, animation, game flow
  → Parallel algorithms — simulation, networking

□ Does this make algorithms clearer to express?
□ Does this enable better data structure design?
□ Does this give the compiler more to optimise?
```

### The Core Insight

```
Other languages treat algorithms and data structures
as things that LIVE IN the language.

Skev treats them as things the language
was DESIGNED AROUND.

Every Rule E check ensures we never compromise
the thing Skev was designed to do.
```

---

## Rule F — Real-World Examples with Language Comparison

Every significant design decision must be demonstrated through real working code before it can be locked. Examples must be **hard, complex, and real** — not simplified toys. And every complex example must be explained in plain language a non-expert can understand.

> **The two-layer requirement: Make it hard. Make it clear.**
> A trivial example proves nothing. An unexplained complex example helps nobody.
> Every example must be both.

---

### The Non-Negotiable Comparison Rule

> **Every single real-world example — without exception — must show the same problem solved in C++23, C# 13, and Python 3.13 before showing the Skev solution.**

This is not optional for complex examples only. This applies to:
- Every 🟢 Foundation example
- Every 🟡 Applied example
- Every 🔴 Hard Complex example
- Every non-game use case example
- Every section that shows Skev code solving a real problem

```
If Skev code appears in a chapter solving a real problem
→ C++23 version must appear
→ C# 13 version must appear
→ Python 3.13 version must appear
→ Then and only then the Skev version appears

No Skev example without all three comparisons.
No exceptions.
No shortcuts.
```

---

### Mandatory Line Count Annotation

Every code block in every example must end with a line count comment:

```
// X lines total
```

This makes comparisons objective and verifiable. Readers can judge for themselves whether Skev's length is justified. We do not hide line counts when Skev is longer.

```
Format:
// 12 lines total        — at the bottom of every code block
// ~200 lines estimated  — when showing abbreviated version

Rules:
→ Count EVERY line including blank lines and closing labels
→ Do not count the language hint line (```swift) or the closing ```
→ If a code block is abbreviated, mark it [abbreviated] and
  note what the full version would require
→ NEVER count third-party library setup lines for other languages
  but exclude them from Skev too — same rules for all languages
```

**Example format:**

````
### In C++23
```cpp
class Player {
    int health = 100;
    void TakeDamage(int amount) { health -= amount; }
};
// 4 lines total
```

### In Skev
```swift
entity Player >>
    health :: int = 100
    when collision(other: Enemy)
        health -= other.attack_power
    << collision: Enemy
<< Player
// 6 lines total
```

### ✅ Honest Advantage Statement
Skev uses 2 more lines due to block closing labels.
The labels provide navigation clarity in large files.
For this simple case, C++ is more concise.
Skev's advantage appears at scale — not at 4 lines.
````

---

### Implementation Depth Fairness Rule

This rule was violated in Chapter 7 section 7.9 — Skev showed 90 lines of full working code while C++/C# were estimated. That is dishonest and must never happen again.

```
When showing a complex example — the rule is:

Option A — Full implementations in ALL languages
  Show complete, working code in every language
  Same depth. Same completeness. Same features.
  Line counts are accurate and comparable.

Option B — Abbreviated implementations in ALL languages
  Mark every abbreviated block clearly with [abbreviated]
  State what the full version would require
  Estimated line counts clearly marked with ~

NEVER do this:
❌ Full Skev implementation (90 lines, working)
   vs
❌ "C++ would need ~400 lines" (estimated, not shown)

This is the worst form of cherry-picking.
It makes Skev look better by hiding the comparison.

ALWAYS do one of these:
✅ Show comparable abbreviated versions in all languages
   // ~20 lines [abbreviated — see full version in examples repo]

✅ Show full versions in all languages
   // 90 lines total (working, complete)
```

---

### Why Skev Sometimes Uses More Lines — Honest Explanation

Skev code is sometimes longer than equivalent C# or Python code. This is intentional in some cases and a genuine trade-off in others. Every honest advantage statement must acknowledge this:

```
Reasons Skev uses more lines:

1. << block labels — one line per block close
   Cost:   +1 line per nested block
   Gain:   navigation clarity in 500+ line files
   Verdict: worth it at scale, not worth it at 10 lines

2. Explicit error handling (result[T])
   Cost:   more visible lines than silent exceptions
   Gain:   no hidden control flow, compiler-enforced handling
   Verdict: worth it — silent exceptions cause real bugs

3. Explicit type declarations (optional but common)
   Cost:   extra tokens per line
   Gain:   self-documenting, compile-time verified
   Verdict: optional — remove for brevity if clear from context

4. No implicit coercion
   Cost:   .as(float) where C# would convert silently
   Gain:   no accidental precision loss
   Verdict: worth it for safety

Honest rule:
If Skev's example is longer — say so.
Explain which of the above reasons applies.
Let the reader decide if the trade-off is worth it.
Never hide the length difference.
```

---

**Why every example — not just complex ones:**

```
Simple examples prove syntax.
Medium examples prove usability.
Complex examples prove design.
ALL of them prove honesty.

If Skev only shows comparisons on the examples
it knows it wins — that is cherry-picking.
Showing comparisons on every example, including
simple ones, proves that Skev is consistently
better — not selectively better.

A developer evaluating Skev will look at many examples.
If some have comparisons and some don't — they will
wonder what we are hiding in the ones that don't.
Consistency builds trust. Selective comparison destroys it.
```

---

### The Complexity Standard

Examples must meet this bar before they are accepted:

```
❌ NOT acceptable — too simple:
   entity Player >>
       health :: int = 100
   << Player
   "This shows a player with health."

   Why rejected: Any language can do this.
   It proves nothing about Skev's advantages.

✅ ACCEPTABLE — hard and real:
   A full multiplayer combat system with:
   → Concurrent damage processing from multiple sources
   → Rollback netcode for lag compensation
   → State machine for player conditions
   → ARC memory management under entity destruction
   → Real-time performance constraints
   "Here is what each part does and why Skev handles
    it better than the alternatives..."

   Why accepted: This is what real developers build.
   Skev's advantages are visible and verifiable.
```

**The complexity checklist — an example must include at least THREE:**

```
□ Multiple interacting systems (not isolated code)
□ Error conditions and edge cases handled
□ Performance-critical path demonstrated
□ Real data structures with realistic scale
□ Cross-system communication (events, channels, etc.)
□ State management over time (not just one moment)
□ Concurrency or timing constraints
□ Memory lifecycle (creation, use, destruction)
```

---

### The User-Friendly Explanation Standard

Every complex example requires a plain language explanation **before** the code:

```
→ Describe the real-world scenario in one paragraph
  as if explaining to someone who has never programmed

→ Explain what problem is being solved —
  not how the code solves it

→ After the code — walk through what each section does
  in plain English, not technical jargon

→ Make the Skev advantage visible to a non-expert
```

---

### Required Format — Every Real-World Example

```
### 🌍 Real-World Scenario
[One paragraph — plain English — what is happening
 and why it is difficult. No code. No jargon.]

### 💥 Why This Is Hard
[What makes this technically challenging.
 What goes wrong in other languages.]

### In C++23
[Code — fair — modern idioms]
[One sentence: what the developer must manage manually]

### In C# 13
[Code — fair — modern idioms]
[One sentence: what the developer must manage manually]

### In Python 3.13
[Code — fair — modern idioms]
[One sentence: what the developer must manage manually]

### In Skev
[Code — showing the feature solving the same problem]

### 🧭 Walk-Through
[Section by section plain English explanation]
[Only required for 🔴 Hard Complex examples]

### ✅ Honest Advantage Statement
[Specific, verifiable, acknowledges trade-offs]
[Required for every example — even simple ones]
```

---

### Comparison Scale Per Example Level

| Level | Description | Comparison Required |
|---|---|---|
| 🟢 **Foundation** | Single feature in isolation | ✅ All three languages + honest statement |
| 🟡 **Applied** | Feature solving a real problem | ✅ All three languages + honest statement |
| 🔴 **Hard Complex** | Multiple interacting systems under REAL constraints — mandatory | ✅ All three languages + walk-through + honest statement |
| 🌐 **Non-game** | Domain-specific use case | ✅ At least C++ or C# equivalent + honest statement |

---

### Honesty Requirements

```
When C++ or C# is genuinely better in some aspect:
→ Say so directly
→ Explain the trade-off Skev makes
→ Show what Skev gains from the trade-off

When Python is more concise for a simple case:
→ Acknowledge it
→ Explain what Skev adds beyond conciseness

When Skev's example is longer than C++:
→ Acknowledge it
→ Explain what those extra characters prevent
   (bugs, crashes, data races, memory leaks)

A biased comparison destroys trust.
Showing comparisons on every example — not just
the ones we know we win — builds it.
```

---

### Retrofit Requirement for Existing Chapters

Any chapter section that shows a real-world Skev example WITHOUT the three-language comparison must be updated before that chapter is considered complete. This applies retroactively to:

```
Chapter 2 — Syntax:    code examples need comparisons
Chapter 3 — Types:     examples need comparisons
Chapter 4 — Memory:    examples need comparisons
Chapter 5 — Concurrent: partial comparisons — needs completion
Chapter 6 — Errors:    partial comparisons — needs completion
```

All chapter retrofits are tracked and completed before any chapter is marked as final.

---

### Use Case Rotation Table

Each chapter's 🔴 Hard Complex example and non-game example:

| Chapter | Complex Game Example | Non-Game Complex Example |
|---|---|---|
| Chapter 2 | Open world RPG player system | XR hand + body tracking |
| Chapter 3 | Full inventory + crafting system | Robotics multi-sensor data |
| Chapter 4 | Boss fight memory lifecycle | Simulation 10,000 particle ARC |
| Chapter 5 | Multiplayer combat with rollback | Robotics multi-axis real-time |
| Chapter 6 | Corrupted save recovery | Surgical robot command processing |
| Chapter 7 | Full game save/load pipeline | Animation keyframe export |
| Chapter 8 | PhysX integration for destruction | Industrial sensor hardware FFI |
| Chapter 9 | Multi-platform AAA build pipeline | Simulation HPC deployment |
| Chapter 10 | Mod system with sandboxed scripts | Medical device plugin isolation |

---

### Why This Rule Exists

```
Simple examples prove syntax.
Complex examples prove design.
Consistent comparisons prove honesty.

Any language can make "hello world" look clean.
Only a well-designed language makes a 500-enemy
AAA combat system with rollback netcode look clean.

Showing the comparison on every example — not just
the complex ones — is what separates honest advocacy
from marketing.

Skev's core promise — fast like C++, readable like Python —
can only be proven through hard, real, consistently
compared examples across every section of the spec.

If Skev's advantages disappear under comparison —
the design is wrong and must be revised.
If Skev's advantages hold consistently —
that is proof worth showing to the world.
```

---

## Rule G — Syntax Highlighting & Color Scheme

Skev code must always be visually distinct and easy to read. Color is not decoration — it is information. Every token category has a specific color with a specific reason.

### Official Skev Color Scheme — Dark Theme (Primary)

```swift
Background:      #0D1117   deep dark — game developer standard
Default text:    #E6EDF3   soft white — easy on eyes

Keywords:        #FF7B72   warm red
  entity, component, data, kind, when, has, maybe, weak,
  task, async, await, shared, cancel, realtime, loop,
  match, if, else, fixed, hidden, not, and, or, is,
  exists, nothing, stop, skip, result

Game-native (!): #79C0FF   bright blue — these are special
  Vector3!, Color!, Transform!, Quat!, Rect!, Ray!, TypeInfo!

Operators:       #FF9E64   warm orange
  >> << :: -> += -= *= /= = < > <= >=

Strings:         #A5D6FF   light blue
  "Hello World"

Numbers:         #79C0FF   bright blue
  100, 5.0, 0.15

Comments:        #8B949E   muted grey — important but not primary
  # single line comment
  #{ multi line comment }#

Doc comments:    #3FB950   green — documentation stands out
  #! doc comment

Property names:  #FFA657   amber — data is amber
  health, speed, position, damage

Type names:      #D2A8FF   lavender purple — types are purple
  Player, Enemy, Dragon, DamageEvent

Block labels:    #7EE787   bright green — labels guide navigation
  << Dragon, << collision, << update

Punctuation:     #8B949E   muted grey
  ( ) [ ] , .
```

### Official Skev Color Scheme — Light Theme (Documentation Site)

```
Background:      #FFFFFF
Default text:    #24292F
Keywords:        #CF222E   dark red
Game-native (!): #0550AE   dark blue
Operators:       #953800   dark orange
Strings:         #0A3069   dark blue
Numbers:         #0550AE   dark blue
Comments:        #6E7781   grey
Doc comments:    #116329   dark green
Property names:  #953800   dark amber
Type names:      #8250DF   purple
Block labels:    #116329   dark green
Punctuation:     #6E7781   grey
```

### Color Design Rationale

```
Keywords = red:         Commands that DO things — attention-grabbing
Game-native = blue:     The feature unique to Skev — visually distinct
Operators = orange:     The symbols that CONNECT things — warm
Strings = light blue:   Data being stored — cool, calm
Numbers = blue:         Numeric literals — same family as game-native
Comments = grey:        Important but secondary — recedes visually
Doc comments = green:   Documentation — positive, constructive
Properties = amber:     State of entities — warm data
Types = purple:         Classifications — regal, distinct
Block labels = green:   Navigation markers — helpful, visible
```

### MD File Approximation

Skev does not yet exist in any syntax highlighter. Until Chapter 11 defines the TextMate grammar and LSP implementation, MD files use `swift` as the closest approximation:

```
swift language hint gives:
✅ String literals highlighted
✅ Numbers highlighted
✅ Type names highlighted (PascalCase)
✅ Some keywords highlighted
⚠️ Skev-specific syntax (>>, <<, ::) — not highlighted
⚠️ Skev keywords (when, has, kind) — partial match

This is a temporary approximation only.
Proper Skev syntax highlighting arrives in Chapter 11.
```

### Tooling Requirements (Chapter 11)

```
Formal Skev highlighting requires:
→ TextMate grammar (.tmGrammar.json)
→ Language Server Protocol (LSP) implementation
→ VS Code extension
→ Skev Studio native integration
→ Documentation site highlighter (Prism.js plugin)

All of the above are Chapter 11 deliverables.
The color scheme above is locked now so all tools
use identical colors from day one.
```

---

## Rule H — Trademark & Naming Conflict Prevention

Every name introduced into Skev — keyword, library namespace, annotation, or built-in function — must be checked against existing products and trademarks before it is locked. A name that evokes an existing product creates confusion even if it is legally safe.

### The Conflict Check

```
Before locking any name — run all six checks:

□ Microsoft trademarks:
  .NET, Azure, Windows, C#, DirectX, Xbox, Visual Studio,
  TypeScript, PowerShell, MSBuild, NuGet

□ Apple trademarks:
  Swift, Metal, ARKit, SceneKit, SpriteKit, RealityKit,
  Xcode, AppKit, UIKit, SwiftUI, Objective-C

□ Game engine names:
  Unity, Unreal, Godot, GameMaker, CryEngine,
  Lumberyard, Defold, MonoGame, LÖVE, Bevy

□ Evocation check — even if not legally identical:
  Does this name make an experienced developer
  immediately think of another product?
  Example: skev.networkwork → evokes .NET → rename to skev.networkworkwork

□ Common developer tool names:
  git, npm, pip, cargo, gradle, cmake, make,
  docker, kubernetes, terraform, webpack

□ Language-specific confusion check:
  Would a C++ developer misread this?
  Would a C# developer misread this?
  Would a Python developer misread this?
  Would a Rust developer misread this?
```

### The Naming Convention Skev Follows

```
Full descriptive words — never abbreviations:

✅ skev.networkworkwork   not   skev.networkwork    (evokes .NET)
✅ skev.file      not   nova.fs     (unclear)
✅ skev.math      not   nova.m      (unclear)
✅ skev.time      not   nova.t      (unclear)
✅ skev.serial    not   nova.bin    (unclear)

Reasoning:
Full words match "easy to read like Python."
Abbreviations are a C/C++ habit Skev does not inherit.
A developer reading Skev code for the first time
should understand every import immediately.
```

### Applied Example

```
Decision made before Chapter 7:

❌ skev.networkwork    — evokes Microsoft's .NET platform
               — creates friction for C# developers
               — renamed before locking

✅ skev.networkworkwork — unambiguous, full word, zero confusion
               — locked
```

### When a Name Cannot Be Changed

If an unavoidable name conflict exists — document it explicitly:

```
The documentation must state:
→ What the potentially conflicting name is
→ Why the name was chosen despite the similarity
→ How Skev's usage differs from the existing product
→ How to avoid confusion in context
```

---

## Rule I — Bidirectional Interop Thinking

Any feature that involves language boundaries must consider all three directions of interoperability — not just the direction that motivated the feature.

> **Interop is always bidirectional. One direction without the other is an incomplete design.**

### The Three Directions

```
Direction A — Skev calling external code:
  "I want to call a C++ physics library from Skev"
  → extern "C", unsafe block, Tier 1-4 mechanism

Direction B — External code calling Skev:
  "I want to use Skev's pathfinder from my C# game"
  → export "C", native library, P/Invoke, ctypes

Direction C — Skev as an embeddable library:
  "I want to embed Skev as a game scripting engine"
  → Stable ABI, hot-reload DLL, plugin system
```

### The Four Interop Tiers

```
Tier 1 — C ABI (zero overhead):
  Languages: C, C++, Rust, Swift, Zig, Go (with export), C# Native AOT
  Skev calls them: extern "C" Name >> ... << Name
  They call Skev: export "C" function_name()
  Overhead: zero — direct function call

Tier 2 — Embeddable scripting (minimal overhead):
  Languages: Lua, Python, JavaScript
  Skev hosts them: skev.scripting.lua / .python / .javascript
  They call Skev: via registered bridge functions
  Overhead: minimal for Lua, moderate for Python/JS

Tier 3 — IPC-based (high latency):
  Languages: Java, Kotlin, C# (without Native AOT), Ruby
  Skev calls them: skev.ipc (process spawn + pipes)
  They call Skev: skev.ipc (reverse — Skev responds)
  Overhead: milliseconds — not for per-frame use

Tier 4 — WebAssembly (low overhead):
  Skev hosts WASM modules: skev.wasm
  WASM calls Skev: mod.import() registered functions
  Skev calls WASM: module.call()
  Overhead: low — near-native with WASM runtime
```

### Bidirectional Table — Complete Reference

```
Language          Skev → Them              Them → Skev
───────────────────────────────────────────────────────
C                 extern "C"               export "C"
C++               extern "C"               export "C"
Rust              extern "C"               export "C"
Swift             extern "C"               export "C"
C# (Native AOT)   extern "C"               export "C"
C# (Unity)        —                        P/Invoke → export "C"
Python            skev.scripting.python    ctypes/cffi → export "C"
Lua               skev.scripting.lua       lua.register()
JavaScript        skev.scripting.javascript mod.import()
Java              skev.ipc                 skev.ipc (respond)
WebAssembly       skev.wasm                mod.import()
GDScript          extern "C"               export "C" (GDExtension)
```

### The Checklist

Before any interop feature is locked:

```
□ Direction A considered: Skev → other language
□ Direction B considered: other language → Skev
□ Direction C considered: Skev as embeddable library
□ Correct tier identified for each direction
□ Overhead of each direction honestly stated
□ Unsafe boundaries clearly marked
□ Callback direction considered (C calling back into Skev)
□ Thread safety of callbacks considered
□ Data marshaling requirements identified
```

### Why This Rule Exists

```
Every time we only think about one direction:

"How does Skev call Lua?"
— answered
"How does Lua call Skev?"
— not considered until asked

This adds chapters of work discovered late.
Practice 6 exists to catch problems early.
Rule I makes bidirectional thinking mandatory
from the first moment interop is mentioned.
```

---

## Chapter Process — Steps 1 to 8

```
Step 1 — Map everything the chapter needs to cover

Step 2 — Apply the Seven Question Test
          (Question 0 first — always)

Step 3 — Cross-check against Consistency Bible
          AND against Rule C gap tracker —
          does this chapter interact with any open gap?

Step 4 — Real World Test (3 scenarios)

Step 5 — Flag all proactive concerns:
          Use case checklist — all 7 use cases
          Four Parameters Check — Rule A
          Algorithm & Data Structure Check — Rule E
          Naming & Trademark Check — Rule H
            (every new name: keyword, library, annotation)
          Bidirectional Interop Check — Rule I
            (if feature involves language boundaries)
          Gap tracker — add any new gaps discovered

Step 6 — Present complete battle-tested design

Step 7 — Lock approved decisions in Consistency Bible
          Update Rule C gap tracker if gap was resolved
          Confirm all names passed Rule H check
          Confirm bidirectional interop addressed (Rule I)

Step 8 — Write the chapter
          Every significant feature includes
          the full Rule F format:
          → 🟢 Foundation — with C++/C#/Python comparison
          → 🟡 Applied game example — with comparison
          → 🔴 HARD complex example — multiple systems, real constraints, with walk-through and comparison
          → Non-game use case — with comparison
          → Honest advantage statement on every example
```

---

## Architecture Audit Summary

*From the complete programming language architecture diagram — last audited before Chapter 5*

| Section | Status | Primary Chapter |
|---|---|---|
| Lexer | ✅ Complete | Chapter 2 |
| Parser | ✅ Complete | Chapter 2 |
| Semantic Analysis | ✅ Complete | Chapter 2-3 |
| IR / Optimisation | ✅ Via LLVM | Chapter 1 |
| Code Generation | ✅ AOT native | Chapter 1 |
| Execution Model | ✅ AOT decided | Chapter 1 |
| Type System | ⚠️ Generics missing | Chapter 3.5 |
| Memory Model | ✅ Strong | Chapter 4 |
| Concurrency Model | ✅ Core defined | Chapter 5 |
| Error Handling | 🔴 Missing | Chapter 6 |
| Standard Library | ⚠️ Partial | Chapter 7 |
| Interoperability | 🔴 Missing | Chapter 8 |
| Linking & Loading | ⚠️ Not specified | Chapter 9 |
| Ecosystem & Distribution | ⚠️ Vision only | Chapter 9 |
| Security Model | ⚠️ Partial | Chapter 10 |
| Tooling | ⚠️ Vision only | Chapter 11 |
| Cross-Cutting Concerns | ⚠️ Partial | Throughout |
| Backward Compatibility | 🔴 Not defined | Chapter 9 |

```
Foundation is architecturally sound.
All gaps are additive — not structural problems.
Nothing built needs to be torn down.
Every gap has an assigned chapter.
```

---

## Rule J — Governance Model

Skev's governance model is formally locked. Every decision about open source, licensing, revenue, and control must be consistent with this rule.

### The Model — Controlled Open Source

Skev follows the Swift/Kotlin/Go/TypeScript model:

```
AJ controls the language direction — not a committee.
The compiler and specification are open source.
Revenue comes from tooling and services — not the language itself.
```

### What Is Open Source (Apache 2.0)

```
✅ Skev compiler        — every line of compiler code is public
✅ Skev specification   — all chapters are public documents
✅ Skev standard library — skev.math, skev.file, skev.network etc
✅ skev.pkg format       — package format is open and documented
✅ Skev Package Hub (public tier) — free forever for public packages
```

### What Is Commercial

```
💰 Skev Studio Pro      — advanced profiler, team collaboration,
                           cloud build, console dev tools
                           Community edition: free
                           Pro tier: paid subscription

💰 Skev Package Hub (private tier) — private packages for studios
                           Free: public packages only
                           Paid: private packages, enterprise mirror

💰 Skev Console Toolchain — console SDK integration tools
                           Requires paid license (same model as
                           Unity, Unreal, and all game middleware)
```

### Why This Model — Not Full Open Governance

```
Full open governance (Python model) fails because:
→ Design by committee slows decisions to years
→ No single entity can make breaking decisions
→ Funding from donations is chronically insufficient
→ GIL removal took 20+ years of debate
→ Python 2→3 took 15 years because nobody could
  force the ecosystem to move

Controlled open source (Swift/Kotlin/Go model) works because:
→ AJ's vision is the language's biggest asset
→ Distributed control distributes the vision
→ Distributed vision = no vision
→ Money from tooling funds compiler development
→ Community contributes but AJ decides
→ Breaking decisions can be made when needed
```

### Proven Precedents

```
Swift:      Apple controls ✅  Open source ✅  Massive adoption ✅
Kotlin:     JetBrains controls ✅  Open source ✅  Thriving ✅
            JetBrains profits from IntelliJ — community trusts this
Go:         Google controls ✅  Open source ✅  Fast decisions ✅
TypeScript: Microsoft controls ✅  Open source ✅  Dominant ✅
```

### The Governance Backstop (Dead Man's Switch)

```
The main concern with controlled open source:
"What if AJ abandons the project?"

Resolution — written into the Skev license:

"If the Skev Language Foundation ceases to maintain
 the language for 24 consecutive months —
 the compiler, specification, and standard library
 automatically relicense to Apache 2.0 with full
 community governance."

This means:
→ AJ controls while active — direction is clear
→ If AJ walks away — community inherits it cleanly
→ Developers can never be stranded
→ Removes the last objection to controlled open source
```

### What This Means For Every Chapter Decision

```
When designing language features:
→ Features must work in the open source compiler
→ Features must not require the commercial Studio to function
→ Studio can ENHANCE features — not gatekeep them

When designing the standard library:
→ All library code is open source
→ Studio may provide better visualisation of library features
→ But the library itself is always free and open

When designing the Package Hub:
→ Publishing public packages: always free
→ Installing public packages: always free
→ Private packages: paid feature
→ The free tier must be genuinely useful — not crippled

The test for every decision:
"Could a developer use Skev fully without ever
 paying for anything?"
Answer must be: YES — for game development, simulation,
               robotics, all 7 target use cases.
               The commercial tier adds convenience and power.
               It never gates the language itself.
```

### The One-Line Rule

> **Open source the language. Control the direction. Fund through tooling.**

---

## The Commitment

> Every language decision made for Skev will be measured
> against these practices before it is proposed,
> before it is locked, and before it enters the specification.
>
> Real examples ground every decision in developer reality.
> Honest comparisons build trust with the community.
> Every gap is tracked and assigned — none are forgotten.
> Skev wins because it is genuinely better —
> and we can show exactly why, in code, every time.
>
> Open source the language.
> Control the direction.
> Fund through tooling.
> This is how Skev lasts.

---

*Version 1.5*
*Added: Generics to critical gaps, expanded Rule C gap tracker with full architecture audit results, added Rust and Go to Rule D version table, added gap tracker check to Step 6 of mandatory checklist, added architecture audit summary table*
*Changes require explicit agreement from both AJ and Claude*
