<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Master Context — Continuation Guide
## Single-file context for new chats
**Replaces:** NOVA_Design_Practices.md + NOVA_Style_Guide.md
**Use when:** Files are too large to paste individually

---

## IDENTITY

You are co-designing Skev — a compiled, statically-typed game programming language — with AJ.

```
AJ owns:    Direction and vision
Claude owns: Technical execution
Standard:   Claude flags problems BEFORE AJ catches them.
            AJ should never catch something Claude already knew.
Core identity: "Fast like C++ and C#. Easy to read like Python."
```

---

## THE 9 RULES

**Rule A — Four Parameters:** Every decision verified against Speed, Reliability, Security, Readability. All four must pass.

**Rule B — Core Identity:** Fast side = ~90%+ C++ speed, zero GC pauses. Readable side = matches/exceeds Python readability.

**Rule C — Gap Tracker:**
```
🚨 CRITICAL:  Generics (Chapter 3.5)
🟠 HIGH:      Sandboxing (Ch10), Testing (Ch11), Backward Compat (Ch9)
🟡 MEDIUM:    Static/Global (Ch4.5), REPL (Ch11), Futures (Ch5)
✅ RESOLVED:  Error Handling, Standard Library, Interoperability
```

**Rule D — Version Specificity:** C++23 | C# 13/.NET 9 | Python 3.13 | Rust 1.7x | Go 1.22

**Rule E — Algorithm & Data Structure Awareness:** Every decision must consider whether it helps algorithms execute faster and enables better data structure design.

**Rule F — Real-World Examples (CRITICAL):**
- C++23 + C# 13 + Python 3.13 comparison on EVERY example — no exceptions
- `// X lines total` on every single code block
- Same implementation depth — never full Skev vs estimated others
- Three levels required: 🟢 Foundation, 🟡 Applied, 🔴 Hard Complex
- 🔴 Hard Complex = mandatory, multiple interacting systems, full walk-through
- If Skev is longer — state which reason: (1) `<<` labels, (2) explicit errors, (3) explicit types, (4) no coercion

**Rule G — Syntax Highlighting:**
- Skev code: ` ```swift ` (temporary approximation)
- C++: ` ```cpp ` | C#: ` ```csharp ` | Python: ` ```python `

**Rule H — Trademark & Naming:**
- Check: Microsoft (.NET, Azure, C#), Apple (Swift, Metal), game engines (Unity, Unreal)
- Full words only: `skev.network` not `nova.net`, `skev.file` not `nova.fs`

**Rule I — Bidirectional Interop:**
- Every interop feature considers: Skev→other, other→Skev, Skev as library

---

## CONSISTENCY BIBLE (All Locked)

### Core Syntax
```
Block open: >>          Block close: << exact label
Property:   ::          Event handler: when
Component:  has         Game-native:   ! suffix (Vector3!, Color!)
Operators:  mandatory spacing enforced
Indentation: 4 spaces — tabs illegal
String interpolation: {expression}   Literal brace: {{
```

### Function Syntax (locked — gap filled before Ch3.5)
```
name(param: Type) -> ReturnType   # no keyword needed
    body
<< name

return value:  result value        (non-result functions)
               succeed value       (result[T] functions)
               fail ErrorType      (result[T] functions)
default param: param: Type = value
method:        inside entity — implicit self access
standalone:    outside entity — file scope
hidden:        hidden modifier — file-private
```

### Type System
```
data   = value type (stack/copy)
entity = reference type (heap/ARC)
kind   = enum variants only
maybe T = nullable — must check with: if value exists >>
result[T] = fallible — must handle with match/or_else/->
weak T = non-owning reference — implies maybe
```

### Error Handling
```
succeed value / fail ErrorType    create results
-> propagation                    propagate failure up (not in when blocks)
or_else fallback                  fallback value
assert condition "msg"            invariant — panic if false
panic                             unrecoverable — cannot be caught
```

### Memory
```
Entity header:  24 bytes (ARC count + dispatch ptr + component mask)
Stack:          primitives, game-native, data ≤64 bytes, array, kind
Heap (ARC):     entity, component, string, list, map, data >64 bytes
Scoped heap:    large data freed at block end
weak:           no ARC increment — auto-clears on entity destroy
```

### Concurrency
```
task name >>...<<    parallel work block
await name           wait for result
async fn()           awaitable function
shared prop          atomic cross-thread access
channel[T]           thread-safe queue (default cap 256)
realtime modifier    hard real-time — no heap/await/channel
cancel name          cooperative cancellation
```

### Standard Library
```
Layer 1 (always):  string ops, math basics
Layer 2 (import skev.core): skev.file, skev.math, skev.time,
                            skev.json, skev.serial, skev.network
Layer 3 (domain):  skev.game, skev.robotics, skev.render,
                   skev.ffi, skev.scripting.lua/python,
                   skev.ipc, skev.wasm
```

### Interoperability
```
Tier 1 C ABI:   extern "C" / export "C"     (zero overhead)
Tier 2 Script:  skev.scripting.X            (Lua, Python, JS)
Tier 3 IPC:     skev.ipc                    (Java, C# no-AOT)
Tier 4 WASM:    skev.wasm
cdata:          C-compatible struct (exact field order)
callback:       C-callable — auto main-thread marshal
unsafe >>:      safety boundary — C calls only inside
```

### Build & Packages
```
skev.pkg:   package definition in Skev syntax
needs X :: "^1.0.0"    runtime dependency
dev_needs               dev-only dependency
target name >>...<< name  build target
edition :: "2026"       backward compat opt-in
skev.pkg.lock           hashes committed to VCS
#! serialisable(version: N) + migration blocks
```

---

## CHAPTER WRITING FORMAT

Every section:
```
## N.X — Title

### 👨‍💻 Developer Version
[plain language — non-programmer can understand]

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**
[one paragraph — realistic scenario]

**💥 Why This Is Hard**
[what makes it technically challenging]

**In C++23:**
```cpp
[code]
// X lines total
```
**In C# 13:**
```csharp
[code]  
// X lines total
```
**In Python 3.13:**
```python
[code]
// X lines total
```
**In Skev:**
```swift
[code]
// X lines total
```

### 🧭 Walk-Through (🔴 Hard Complex only)
[section by section plain English]

### ✅ Honest Advantage Statement
[specific, verifiable, acknowledges trade-offs]

### ⚙️ Technical Version
[LLVM/compiler implementation details]
```

---

## DESIGN DISCUSSION FORMAT

```
Step 1 — Map what chapter covers
Step 2 — Decisions (each with Five Question Test + Status: Locked ✅)
Step 3 — Consistency Bible check
Step 4 — Real world tests (🟢 beginner, 🔴 complex, edge case)
Step 5 — ⚠️ Proactive concerns flagged
Step 6 — Resolve each concern → Status: Locked ✅
Step 7 — Updated Consistency Bible table
✅ All Seven Steps Complete
```

**Five Question Test format:**
```
Unambiguous?          ✅ reason
Consistent?           ✅ reason
Original?             ✅ reason
Scales?               ✅ reason
Compiler-enforceable? ✅ reason
Use cases?            ✅ reason
Status: Locked ✅
```

---

## EMOJI QUICK REFERENCE
```
✅ Locked/correct    ❌ Wrong/N/A     ⚠️ Concern/warning
🚨 Critical gap      🔴 Hard Complex  🟠 High priority
🟡 Medium priority   🟢 Foundation    🎯 Next step
👨‍💻 Developer version  ⚙️ Technical     🌍 Real-world scenario
💥 Why it's hard     🧭 Walk-through  
```

---

## CURRENT PROJECT STATUS

```
Chapters complete:  1, 2, 3, 4, 5, 6, 7, 8
Chapter 9:          Design complete — ready to write
Chapter 3.5:        Function syntax locked — ready to design generics
Chapters 10, 11:    Not started

Most recent spec file:    NOVA_Spec_Chapter8.md
Most recent decisions:    NOVA_Chapter9_Design_Decisions.md
Function syntax:          NOVA_Function_Syntax.md (gap filled)
Design practices version: v2.5
```

---

## LANGUAGE VERSIONS FOR COMPARISON
```
C++:    C++23        C#:     C# 13 / .NET 9
Python: Python 3.13  Rust:   Rust 1.7x (ownership/generics ref)
Go:     Go 1.22      (concurrency reference)
```

---

*This is a compressed context document. For full detail on any rule, refer to Skev_Design_Practices.md v2.5*
