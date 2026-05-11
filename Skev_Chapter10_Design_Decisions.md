<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 10: Security & Isolation — Design Decisions
**Version:** 0.1
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked — Chapter 10 written
**Process:** Steps 1–7 completed per Design Practices v2.6
**Rule H Applied:** sandbox, capabilities, skev.sandbox — all checked ✅

---

## Decision 1 — Three-Tier Sandbox Model

```
Tier 1 WASM:    Untrusted (mods, community) — structural isolation
Tier 2 Native:  Trusted-but-isolated (DLC, studio plugins)
Tier 3 Process: External tools via skev.ipc (Chapter 8)
```
**Status: Locked ✅**

---

## Decision 2 — Capability System

Declared in `skev.pkg` under `capabilities >>` block. Host grants subset. Plugin uses only what both declared AND host granted.

Categories: `audio.*` `scene.*` `file.*` `network.*` `skev.*` `ui.*` `system.*`

Fine-grained syntax: `scene.read_entities(kind: [Enemy])` `file.read(path: "mods/x/")`

**Status: Locked ✅**

---

## Decision 3 — `sandbox` Keyword + `#! sandboxed` Annotation

`#! sandboxed` marks functions callable from plugins. Unmarked functions are structurally unreachable from plugin code. Compiler error on any attempt.

**Status: Locked ✅**

---

## Decision 4 — Sandbox Violation Handling

`when sandbox.violation(mod_id, capability)` — default: terminate + log. Host can override.

**Status: Locked ✅**

---

## Decision 5 — `#! safety_critical` Annotation

Enables strict compilation: no heap on safety path, no async/await, bounded execution, deterministic behaviour. `safety >> max_execution_ms :: N << safety` block declares bounds.

**Status: Locked ✅**

---

## Decision 6 — Hot-Reload Security

Production: capabilities cannot expand on update. Expansion requires re-approval. Debug: unrestricted.

**Status: Locked ✅**

---

## Decision 7 — Safety-Critical Profile

Combined `#! safety_critical` + `#! realtime`: strictest mode. No allocation, no async, bounded time, compile-time capability checks only.

**Status: Locked ✅**

---

## Concern Resolutions

| Concern | Resolution |
|---|---|
| WASM + native capability unification | Same `skev.pkg` declaration — different enforcement layer |
| Capability delegation | Explicit only — bounded — `sandbox.delegate(B, [cap])` |
| Capability granularity | Coarse (default) + fine-grained with params |
| Performance overhead | WASM: zero. Native: compile-time. Realtime: compile-time only |
| Cross-plugin communication | Via host events only — never direct |

---

## New Consistency Bible Entries

| Decision | Rule |
|---|---|
| Capability declaration | `capabilities >>` block in `skev.pkg` |
| Sandboxed function | `#! sandboxed` annotation |
| Safety-critical | `#! safety_critical` annotation |
| Sandbox load | `skev.sandbox.load_wasm(path)` / `load_native(path)` |
| Violation event | `when sandbox.violation(mod_id, capability)` |
| Cross-plugin | Via host events only — never direct calls |
| Hot-reload | Capabilities cannot expand in production |

---

## Rule C Gap Update

**Resolved by Chapter 10:**
- ✅ Sandboxing (High priority)
- ✅ Capabilities/Permissions (Medium priority)

**Remaining:**
- 🟠 Testing Framework — Chapter 11
- 🟡 REPL/Playground — Chapter 11
- 🟡 Observability — Chapter 11
- 🟡 Static/Global Memory — Chapter 4.5
- 🟡 Futures/Promises — Chapter 5.5

---

*Version 0.1 — All decisions locked — Chapter 10 written*
