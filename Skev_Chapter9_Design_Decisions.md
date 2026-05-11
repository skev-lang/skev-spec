<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 9: Build & Distribution — Design Decisions
**Version:** 0.1
**Authors:** AJ (Copyright © 2026)
**Status:** All decisions locked — Chapter 9 ready to write
**Process:** Steps 1–8 completed per Design Practices v2.4
**Rule H Applied:** skev.pkg, novapkg — all names checked ✅
**Rule I Applied:** Package consumption both directions considered ✅

---

## What Chapter 9 Covers

```
→ Package definition format      (skev.pkg)
→ Version specification syntax   (semver + constraints)
→ Skev Package Hub               (central registry)
→ Build system                   (skev build CLI)
→ Backward compatibility policy  (edition system)
→ Code signing & supply chain    (cryptographic safety)
→ Linking & loading              (static/dynamic)
→ Deployment targets             (all platforms)
→ Save file compatibility        (#! serialisable versioning)
→ Private registries             (studio/enterprise)
→ Emergency security response    (CVE + yanking)
```

---

## Decision 1 — Package Definition Format (`skev.pkg`)

**Problem evaluated:**
```
C++:   CMake + vcpkg/Conan — three competing systems, notoriously complex
C#:    NuGet + .csproj — XML, verbose, hard to read manually
Python: pyproject.toml — good direction, but a new format to learn
Rust:  Cargo.toml — the gold standard for simplicity
```

**Decision: `skev.pkg` using Skev's own syntax**

```swift
package DragonAI >>
    version     :: "1.2.0"
    description :: "Advanced dragon AI for Skev games"
    author      :: "AJ Studios"
    license     :: "MIT"
    edition     :: "2026"

    needs skev.physics    :: ">=2.0.0"
    needs skev.network    :: ">=1.0.0"
    needs skev.pathfinder :: "^1.4.0"

    dev_needs skev.testing :: ">=1.0.0"

    target game_plugin >>
        entry    :: "src/dragon_ai.skev"
        output   :: "dragon_ai.skevpkg"
        platform :: [windows, macos, linux]
        link     :: static
    << game_plugin

    target standalone_lib >>
        entry  :: "src/dragon_ai.skev"
        output :: "libdragon_ai"
        export :: "C"
    << standalone_lib

<< DragonAI
```

**Why Skev syntax — not TOML/JSON/YAML:**
```
→ Zero learning curve — Skev developers already know Skev syntax
→ Skev Studio highlights it automatically
→ Same compiler validates it
→ Consistent with the rest of Skev
→ No separate parser needed
```

**Rule H check:**
```
skev.pkg    → no trademark conflict ✅
novapkg     → no trademark conflict ✅
skev.toml   → TOML is associated with Rust/Cargo — avoided ✅
```

**Five Question Test:** ✅ All five passed
**Status: Locked ✅**

---

## Decision 2 — Version Specification Syntax

**Decision: Semantic versioning (MAJOR.MINOR.PATCH) with constraint operators**

```swift
needs skev.physics :: "2.4.1"          # exactly this version
needs skev.physics :: ">=2.0.0"        # minimum version
needs skev.physics :: "^2.0.0"         # compatible: 2.x.x
needs skev.physics :: "~2.4.0"         # patch-level: 2.4.x
needs skev.physics :: ">=2.0.0 <3.0.0" # explicit range
```

**Semantic versioning policy:**
```
MAJOR: breaking changes — existing code may not compile
MINOR: new features — existing code still works
PATCH: bug fixes — no API changes

^ (caret):  >=current, <next major  (^1.4.0 = >=1.4.0 <2.0.0)
~ (tilde):  >=current, <next minor  (~1.4.0 = >=1.4.0 <1.5.0)
```

**Status: Locked ✅**

---

## Decision 3 — Skev Package Hub

**Central registry at packages.skev.dev**

**Key design decisions (learned from npm/PyPI/crates.io):**

```
1. Content-addressed storage:
   Once a version is published — immutable forever.
   "skev.physics 2.4.1" always = same bytes.
   Prevents npm left-pad incident.

2. Cryptographic signing:
   Every package signed with author's key pair.
   Package Hub verifies signatures on download.
   Tampered packages = installation refused.

3. Namespace ownership:
   "skev.*" reserved for official Skev packages.
   Prevents squatting on official names.

4. Dependency audit:
   skev audit scans for known vulnerabilities.
   Skev Studio shows security warnings inline.

5. Mirror support:
   Studios can run private mirrors.
   Console platform holders require this.
```

**Status: Locked ✅**

---

## Decision 4 — Build System (`skev build` CLI)

**Decision: Zero-configuration CLI for simple projects**

```
skev build              # builds current package
skev build --release    # release build (optimised)
skev build --target wasm32  # cross-compile
skev run                # build and run
skev test               # build and run tests
nova package            # build distributable package
skev publish            # publish to Skev Package Hub
skev audit              # check for security vulnerabilities
skev audit --abi        # check package ABI compatibility
skev audit --security   # check for known CVEs
skev clean              # clear build cache
skev migrate            # upgrade to new edition
skev lock               # regenerate skev.pkg.lock
skev lock --resolve     # resolve lock file merge conflicts
```

**Build profiles in skev.pkg:**

```swift
build >>
    debug >>
        optimise   :: false
        debug_info :: true
        assertions :: true
        hot_reload :: true
    << debug

    release >>
        optimise   :: true
        debug_info :: false
        assertions :: false
        hot_reload :: false
        lto        :: true
    << release

    console >>
        extends  :: release
        platform :: [ps5, xbox, switch]
    << console
<< build
```

**Status: Locked ✅**

---

## Decision 5 — Backward Compatibility Policy (Edition System)

**Problem evaluated:**
```
Java:     never break anything → language accumulates technical debt
Python:   broke freely (2→3) → 10+ years of ecosystem fragmentation
Rust:     edition system → opt-in migration, old code still compiles
```

**Decision: Skev Edition System — inspired by Rust editions**

```swift
package MyGame >>
    edition :: "2026"
    version :: "1.0.0"
<< MyGame
```

**How editions work:**
```
An edition NEVER breaks existing code in that edition.
New editions may change surface syntax — never core semantics.
Compiler always supports last 3 editions.
Migration tool provided before any deprecation.

Hard rule:
"Code written today must compile in 10 years."

What CAN change between editions:
→ Surface syntax patterns (improved alternatives)
→ Standard library function names (with aliases)

What NEVER changes:
→ Core type semantics (entity, data, kind, maybe, result)
→ ARC memory model
→ Error propagation model
→ Concurrency model
```

**Status: Locked ✅**

---

## Decision 6 — Code Signing & Supply Chain Safety

**Problem:** npm event-stream (2018), PyPI typosquatting — real threats.

**Decision: Three-layer defence**

```
Layer 1 — Cryptographic package signing:
  Every package signed with author's keypair.
  Public keys registered with Skev Package Hub.
  Mismatched signature = rejected at install time.

Layer 2 — Ownership transfer protection:
  Transfer requires 72-hour cooling-off period.
  Email confirmation from both parties.
  Public announcement in package changelog.
  Skev Studio warns developers within 7 days of transfer.

Layer 3 — Reproducible builds via skev.pkg.lock:
  skev.pkg.lock = exact hash of every dependency.
  Committed to source control.
  Any dependency change = lock file mismatch = build fails.
  Developer must explicitly approve updates.
```

**Lock file format:**
```swift
lock >>
    skev.physics    :: "2.4.1" hash: "sha256:a3b2c1..."
    skev.network    :: "1.8.0" hash: "sha256:d4e5f6..."
    skev.pathfinder :: "1.4.2" hash: "sha256:789abc..."
<< lock
```

**Status: Locked ✅**

---

## Decision 7 — Linking & Loading

**Decision: Both static and dynamic supported — declared in skev.pkg**

```swift
target game >>
    link :: static     # everything compiled into one binary
<< game

target game_with_plugins >>
    link :: dynamic    # plugins load at runtime via ffi.load_platform()
<< game_with_plugins
```

**Dynamic loading:** Already defined in Chapter 8 — `ffi.load_platform()` — consistent ✅

**Loader behaviour (deterministic):**
```
Skev binary loading sequence:
1. OS loader maps binary into memory
2. Skev runtime initialises (ARC system, thread pool)
3. Main entity system initialises
4. Entry point entity fires scene_load event
5. Game runs

No hidden global state.
No static initialisation order problems.
Deterministic every time.
```

**Status: Locked ✅**

---

## Decision 8 — Deployment Targets

**All targets — Rule H verified (no platform trademark conflicts):**

```
Desktop:
  windows-x64      Windows 10+
  windows-arm64    Windows 11 ARM
  macos-arm64      macOS 11+ (Apple Silicon)
  macos-x64        macOS 10.15+ (Intel)
  linux-x64        Linux x86-64 (glibc 2.31+)
  linux-arm64      Linux ARM64

Mobile:
  ios-arm64        iOS 14+
  android-arm64    Android 8.0+ (API 26)

Console (NDA — developer program required):
  ps5              PlayStation 5
  xbox             Xbox Series X/S
  switch           Nintendo Switch

Web:
  wasm32           WebAssembly (browser)

Embedded/Robotics:
  linux-arm64-bare Linux ARM64 (no GUI)
  rtos             Real-time OS (custom per vendor)
```

**Cross-compilation:**
```
skev build --target wasm32       # browser from any host
skev build --target linux-arm64  # robot controller on desktop
```

**Status: Locked ✅**

---

## Decision 9 — Save File Compatibility (Serialisation Versioning)

**Problem:**
```
Player saves in v1.0.
Game updates to v1.5 — new fields added to PlayerData.
Player loads save → crash or silent data loss.
Both outcomes are unacceptable.
```

**Decision: `#! serialisable(version: N)` + `migration` blocks**

```swift
#! serialisable(version: 3)
data PlayerData >>
    name       :: string
    level      :: int
    position   :: Vector3!
    health     :: int      = 100     # added v2 — default for old saves
    abilities  :: list[string]       # added v3 — empty for old saves
    # gold removed in v3 — migration handles it
<< PlayerData

migration PlayerData >>

    from_version_1 >> old
        result PlayerData >>
            name      :: old.name
            level     :: old.level
            position  :: old.position
            health    :: 100
            abilities :: []
        << PlayerData
    << from_version_1

    from_version_2 >> old
        result PlayerData >>
            name      :: old.name
            level     :: old.level
            position  :: old.position
            health    :: old.health
            abilities :: []
        << PlayerData
    << from_version_2

<< migration
```

**Compiler rules:**
```
→ Version number increments when fields change
→ Migration handlers REQUIRED for breaking changes
→ Compiler enforces: version incremented = migration must exist
→ Old save files never crash — always migrate
→ Missing optional fields use declared defaults
```

**Status: Locked ✅**

---

## Decision 10 — Private Package Registries

**Decision: `registry :: "url"` in skev.pkg**

```swift
package StudioEngine >>
    registry :: "https://packages.ajstudios.internal"
    version  :: "4.0.0"
<< StudioEngine

# Per-dependency override
needs ajstudios.render :: "^3.0.0"
    registry :: "https://packages.ajstudios.internal"
<< needs
```

**Rules:**
```
→ Same skev.pkg format and signing requirements as public hub
→ skev.* packages always come from official Package Hub
→ Private packages can shadow public packages for a studio
→ Enterprise: air-gapped installation via local mirror supported
→ Console platform holders can require studio-specific registries
```

**Status: Locked ✅**

---

## Decision 11 — Emergency Security Response

**Decision: Skev Security Advisory System**

```
Step 1: Researcher reports to security@skev.dev
Step 2: Skev team verifies, assigns CVE
Step 3: Maintainer notified — 30-day fix window
Step 4: Unfixed after 30 days → public advisory published
        Package Hub marks as "security risk"
        skev install warns prominently
Step 5: skev audit --security shows all affected packages
        Skev Studio shows inline security warnings

Emergency fast-track (critical only):
→ Skev team can force-yank package from Hub
→ All installations receive warning
→ skev build fails if yanked package present
→ Developer must explicitly acknowledge before proceeding
```

**Status: Locked ✅**

---

## Concern Resolutions

| Concern | Resolution |
|---|---|
| Edition migration tooling | `skev migrate` — rewrites deprecated patterns, git commit, manual TODO for ambiguous cases |
| Private packages | `registry :: "url"` per-package or per-dependency |
| Lock file merge conflicts | `skev lock --resolve` — re-runs resolution from skev.pkg |
| Binary ABI compatibility | Source packages preferred; `skev audit --abi` for binary packages; major version may require recompile |
| Emergency security response | 30-day window + force-yank + `skev audit --security` |

---

## New Keywords — Rule H Verified

| Keyword/Name | Purpose | Rule H Check |
|---|---|---|
| `skev.pkg` | Package definition file | ✅ No conflict |
| `skev.pkg.lock` | Dependency lock file | ✅ No conflict (Cargo uses Cargo.lock — concept not trademark) |
| `needs` | Runtime dependency | ✅ No conflict |
| `dev_needs` | Dev-only dependency | ✅ No conflict |
| `target` | Build target block | ✅ Standard term |
| `edition` | Backward compat version | ✅ Rust uses this — concept not trademark |
| `migration` | Save file migration block | ✅ Standard term |
| `lock` | Lock file block | ✅ Standard term |
| `extends` | Build profile inheritance | ✅ Standard term |
| `skev build` | Build CLI command | ✅ No conflict |
| `skev migrate` | Edition migration tool | ✅ No conflict |
| `skev audit` | Security audit tool | ✅ No conflict |

---

## Bidirectional Check — Rule I

```
Direction A — Skev consuming packages:
  → needs PackageName :: "version" in skev.pkg
  → skev.pkg.lock ensures reproducible builds

Direction B — Skev packages consumed by others:
  → export "C" in skev.pkg target with export :: "C"
  → Generates C headers + compiled library
  → Any language can consume via their FFI mechanism

Direction C — Skev as embeddable library:
  → target standalone_lib with export :: "C"
  → Used for: game engine plugins, Unity integration,
    Python tool libraries, C++ game engine components

All three directions covered ✅
```

---

## Consistency Bible — New Entries

| Decision | Rule |
|---|---|
| Package file | `skev.pkg` — uses Skev syntax |
| Dependency | `needs PackageName :: "version"` |
| Dev dependency | `dev_needs PackageName :: "version"` |
| Version semver | MAJOR.MINOR.PATCH + `^` `~` `>=` `<=` |
| Build target | `target name >> ... << name` |
| Build profile | `debug`, `release`, `console` — in `build >> ... << build` |
| Edition | `edition :: "YYYY"` — opt-in migration |
| Lock file | `skev.pkg.lock` — content-addressed hashes — committed to VCS |
| Migration block | `migration TypeName >> from_version_N >> ... << from_version_N << migration` |
| Serialisable version | `#! serialisable(version: N)` |
| Private registry | `registry :: "url"` in skev.pkg |
| CLI commands | `skev build` `skev run` `skev test` `nova package` `skev publish` `skev audit` `skev migrate` `skev lock` `skev clean` |

---

## Rule C Gap Update

**Resolved by Chapter 9:**
- ✅ Linking & Loading
- ✅ Backward Compatibility
- ✅ Dependency Safety
- ✅ Code Signing & Integrity
- ✅ Serialisation Versioning

**Remaining after Chapter 9:**

| Gap | Chapter | Priority |
|---|---|---|
| Generics | Chapter 3.5 | 🚨 Critical |
| Sandboxing | Chapter 10 | 🟠 High |
| Testing Framework | Chapter 11 | 🟠 High |
| Capabilities/Permissions | Chapter 10 | 🟡 Medium |
| REPL/Playground | Chapter 11 | 🟡 Medium |
| Observability | Chapter 11 | 🟡 Medium |
| Static/Global Memory | Chapter 4.5 | 🟡 Medium |
| Futures/Promises | Chapter 5 | 🟡 Medium |

---

## Full Decision Summary

| Decision | Rule | Status |
|---|---|---|
| Package file format | `skev.pkg` using Skev syntax | Locked ✅ |
| Version constraints | semver + ^ ~ >= <= operators | Locked ✅ |
| Package Hub | packages.skev.dev — content-addressed, signed | Locked ✅ |
| Build CLI | `skev build` — zero-config for simple projects | Locked ✅ |
| Edition system | `edition :: "YYYY"` — structured backward compat | Locked ✅ |
| Code signing | 3-layer: signing + ownership transfer + lock file | Locked ✅ |
| Lock file | `skev.pkg.lock` — hashes, committed, `skev lock --resolve` | Locked ✅ |
| Linking | static/dynamic declared in `target` block | Locked ✅ |
| Deployment targets | 14 targets across desktop/mobile/console/web/embedded | Locked ✅ |
| Save versioning | `#! serialisable(version: N)` + `migration` blocks | Locked ✅ |
| Private registries | `registry :: "url"` per package or dependency | Locked ✅ |
| Security response | CVE process + 30-day window + force-yank | Locked ✅ |

---

*All decisions locked — Chapter 9 ready to write*
*Version 0.1 — Pre-Chapter 9 Design Decisions*
