<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 11: Tooling — Design Decisions
**Version:** 0.1
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked — Chapter 11 written
**Process:** Steps 1–7 completed per Design Practices v2.6
**Rule H Applied:** test, mock, bench, skev.log, skev.vscode-skev — all checked ✅

---

## Decision 1 — Test Syntax: `test` keyword block

```swift
test "description" >> ... << test
test async "description" >> ... << test
test_setup >> ... << test_setup
```

No third-party framework. Built into language. Same `>>` `<<` syntax.
**Status: Locked ✅**

---

## Decision 2 — Test Organisation

Tests inline in source OR in `*_test.skev` files. Both valid. Both stripped from release builds.
**Status: Locked ✅**

---

## Decision 3 — Mock Entities

```swift
mock AudioSystem >> when play_at(...) ... << mock AudioSystem
```

`mock` modifier creates a recording entity for testing.
**Status: Locked ✅**

---

## Decision 4 — Benchmark Tests: `bench` block

```swift
bench "name" >> bench_run >> ... << bench_run
  assert bench.median_ms < N
<< bench
```

Stats available: `bench.median_ms`, `bench.p95_ms`, `bench.p99_ms`, `bench.max_ms`, `bench.iteration_count`
**Status: Locked ✅**

---

## Decision 5 — LSP Features

Standard: completions, diagnostics, hover, go-to-definition, rename, format.
Skev-specific: block pair highlighting, `<<` label validation, kind exhaustion hints, entity graph, ARC preview.
**Status: Locked ✅**

---

## Decision 6 — TextMate Grammar

Scope: `source.skev`. Extension: `.skev`. File: `skev.tmLanguage.json`.
Publication: GitHub linguist → VS Code marketplace → JetBrains → Prism.js.
After Chapter 11: ` ```skev ` replaces ` ```swift ` everywhere.
**Status: Locked ✅**

---

## Decision 7 — REPL: `skev repl`

Terminal REPL + `skev.dev/play` browser playground (Skev compiled to WASM). `.load`, `.type`, `.clear`, `.exit` commands.
**Status: Locked ✅**

---

## Decision 8 — Observability: `skev.log`

```swift
log.debug/info/warn/error/crash("event", >> fields << )
trace "name" >> ... << trace
time.measure >> ... << time.measure → float64
```

Typed fields — not strings. Connects to Skev Studio profiler.
**Status: Locked ✅**

---

## Concern Resolutions

| Concern | Resolution |
|---|---|
| Test isolation | Each `test` block = fresh entity scope |
| Async tests | `test async` variant |
| TextMate → GitHub | linguist PR → VS Code marketplace → Prism.js |
| REPL and ARC | Session = one ARC scope. `.clear` resets. |
| Benchmark tests | `bench` block with `bench_run` + statistical assertions |

---

## New Consistency Bible Entries

| Decision | Rule |
|---|---|
| Test block | `test "desc" >> ... << test` |
| Async test | `test async "desc" >> ... << test` |
| Test setup | `test_setup >> ... << test_setup` |
| Benchmark | `bench "desc" >> bench_run >> << bench_run << bench` |
| Mock | `mock Entity >> ... << mock Entity` |
| Log call | `log.info("event", >> fields << )` |
| Trace block | `trace "name" >> ... << trace` |
| Measure block | `time.measure >> ... << time.measure` |
| TextMate scope | `source.skev` |
| VS Code ext | `skev.vscode-skev` |
| REPL command | `skev repl` |
| Doc command | `skev doc` |

---

## Rule C Gap Update — ALL GAPS RESOLVED

```
Testing Framework:    ✅ RESOLVED — Chapter 11
REPL/Playground:      ✅ RESOLVED — Chapter 11
Observability:        ✅ RESOLVED — Chapter 11
```

**Remaining medium-priority (future editions):**
- 🟡 Static/Global Memory — Chapter 4.5
- 🟡 Futures/Promises — Chapter 5.5
- 🟡 Scheduler Specification — Chapter 5.5
- 🟡 Memory Ordering — Chapter 5.5

These are completeness items — not blockers for v1.0.

---

## The Specification Is Complete

```
Chapters 1–11 + 3.5: All written.
All critical gaps:    Resolved.
All high-priority:    Resolved.
Remaining:            Medium-priority (future editions).

Skev v1.0 specification: COMPLETE.
```

---

*Version 0.1 — All decisions locked — Chapter 11 written*
*The Skev Language Specification is now complete.*
*Copyright © 2026 AJ. All Rights Reserved. https://skev.dev | https://skev.org*
