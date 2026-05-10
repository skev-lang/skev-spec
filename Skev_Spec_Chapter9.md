<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 9: Build & Distribution
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026) & Claude (Anthropic)
**Status:** In Progress
**Depends On:** Chapter 8 (FFI/interop), Chapter 6 (errors), Chapter 7 (standard library)
**Use Cases Tested:** All 7 — Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅
**Rule H Verified:** skev.pkg, skev.dev, skev.org — all names confirmed ✅
**Rule I Applied:** Package consumption both directions considered ✅

---

## 9.1 — Overview

### 👨‍💻 Developer Version

Building a game and shipping it are two completely different challenges. Writing code is the creative part. Building, packaging, versioning, and distributing that code so it runs on Windows, macOS, PlayStation, and mobile — that is the engineering part that most languages leave entirely to third-party tools.

Skev treats the build system as a first-class part of the language. The same syntax you use to write game code is the syntax you use to describe your package. No XML. No TOML. No YAML. No separate format to learn.

> **One language. One syntax. Everything from entity code to package definition.**

Skev's build and distribution system is built on four principles:

```
1. Zero configuration for simple projects
   A single skev.pkg file is all you need.
   skev build just works.

2. Reproducible builds everywhere
   skev.pkg.lock guarantees your game builds
   identically on every machine, every time.
   No "works on my machine" problems.

3. Supply chain safety by design
   Every package is cryptographically signed.
   Tampered packages are rejected automatically.
   No trust required — verification is automatic.

4. Code written today must compile in 10 years
   The edition system ensures backward compatibility
   without freezing the language in amber.
```

### ⚙️ Technical Version

Chapter 9 specifies the complete build and distribution infrastructure for Skev: the `skev.pkg` package manifest format, semantic versioning with constraint operators, the Skev Package Hub registry, the `skev` CLI build tool, the edition-based backward compatibility system, three-layer supply chain security (signing + ownership protection + lock files), static and dynamic linking, all supported deployment targets, and save-file serialisation versioning via `#! serialisable(version: N)` with migration blocks. All of these build on Chapter 8's interop model (for C library linking) and Chapter 6's error system (for build-time and runtime failure handling).

---

## 9.2 — The `skev.pkg` Package File

### 👨‍💻 Developer Version

Every Skev project has one `skev.pkg` file in its root directory. This file describes everything about your package: its name, version, dependencies, and how to build it.

The key design decision: `skev.pkg` uses Skev's own syntax. Not TOML. Not JSON. Not XML. If you know Skev — you already know how to write a package file.

> **You learn one syntax. It works for everything.**

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game studio is building an open-world RPG. They need to declare: the game's version, five library dependencies with version constraints, a development-only testing library, a release build target with link-time optimisation, a debug build with hot reload, and a console build that extends the release configuration. This package file will be read by every developer on the team every day.

**💥 Why This Is Hard**

Every language uses a different format for its package file. C++ has three competing systems with no consensus. C# uses XML that is readable by compilers but not humans. Rust's Cargo.toml is the best of the comparison set — but it is still a different format from Rust itself. Every format means a context switch: you stop thinking in your language and start thinking in a configuration language.

---

**In C++23 (CMakeLists.txt + vcpkg):**
```cpp
cmake_minimum_required(VERSION 3.28)
project(OpenWorldRPG VERSION 2.4.1)

find_package(PhysX CONFIG REQUIRED)
find_package(FMOD CONFIG REQUIRED)
find_package(Steamworks CONFIG REQUIRED)
find_package(PathfinderLib CONFIG REQUIRED)
find_package(AudioEngine CONFIG REQUIRED)

add_executable(OpenWorldRPG src/main.cpp)

target_link_libraries(OpenWorldRPG PRIVATE
    PhysX::PhysX
    FMOD::FMOD
    Steamworks::Steamworks
    PathfinderLib::PathfinderLib
    AudioEngine::AudioEngine
)

set_target_properties(OpenWorldRPG PROPERTIES
    CXX_STANDARD 23
    INTERPROCEDURAL_OPTIMIZATION_RELEASE TRUE
)
// 22 lines total — and this is simplified
// Real CMakeLists files run to hundreds of lines
// vcpkg.json is a SEPARATE file in a DIFFERENT format
```

**In C# 13 (.csproj):**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <Version>2.4.1</Version>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <Optimize Condition="'$(Configuration)'=='Release'">true</Optimize>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="PhysicsLib" Version="4.0.*" />
    <PackageReference Include="NetworkLib" Version="2.0.*" />
    <PackageReference Include="RenderLib" Version="3.1.*" />
    <PackageReference Include="AudioLib" Version="1.8.*" />
    <PackageReference Include="PathfinderLib" Version="2.0.*" />
  </ItemGroup>
  <ItemGroup Condition="'$(Configuration)'=='Debug'">
    <PackageReference Include="TestingLib" Version="*" />
  </ItemGroup>
</Project>
// 19 lines total — XML, verbose, no comments in PackageReference
// Console targets require separate project files or MSBuild conditions
```

**In Python 3.13 (pyproject.toml):**
```toml
[project]
name = "open-world-rpg"
version = "2.4.1"
requires-python = ">=3.13"
dependencies = [
    "physics-lib>=4.0.0",
    "network-lib>=2.0.0",
    "render-lib>=3.1.0",
    "audio-lib>=1.8.0",
    "pathfinder-lib>=2.0.0",
]

[project.optional-dependencies]
dev = ["testing-lib>=1.0.0"]

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.pytest.ini_options]
testpaths = ["tests"]
// 19 lines total — TOML format, separate from Python syntax
// Build profiles (debug/release) not natively supported
// Console targets not applicable
```

**In Skev:**
```swift
package OpenWorldRPG >>
    version     :: "2.4.1"
    description :: "Open world RPG — Skev game engine showcase"
    author      :: "AJ Studios"
    license     :: "Proprietary"
    edition     :: "2026"

    needs skev.physics    :: "^4.0.0"
    needs skev.network    :: "^2.0.0"
    needs skev.render     :: "^3.1.0"
    needs skev.audio      :: "^1.8.0"
    needs skev.pathfinder :: "^2.0.0"

    dev_needs skev.testing :: ">=1.0.0"

    target pc_release >>
        entry    :: "src/main.skev"
        platform :: [windows-x64, macos-arm64, linux-x64]
        link     :: static
        lto      :: true
    << pc_release

    target pc_debug >>
        extends  :: pc_release
        lto      :: false
        debug_info :: true
        hot_reload :: true
        assertions :: true
    << pc_debug

    target console >>
        extends  :: pc_release
        platform :: [ps5, xbox, switch]
    << console

<< OpenWorldRPG
// 30 lines total — Skev syntax throughout — no new format to learn
```

### 🧭 Walk-Through — `skev.pkg`

**`package OpenWorldRPG >>`:** Opens the package definition block. Same `>>` `<<` syntax as every other Skev block. The package name is PascalCase by convention.

**`needs` and `dev_needs`:** Runtime and development-only dependencies. `needs` links into the final binary. `dev_needs` is only available during development and testing — stripped from release builds automatically.

**`target` blocks:** Each build configuration is its own block. `extends` allows one target to inherit all settings from another and override specific values. The `console` target extends `pc_release` — it inherits all its settings and only changes `platform`.

**`edition :: "2026"`:** Declares which Skev edition this package uses. Explained fully in section 9.5.

### ✅ Honest Advantage Statement

Skev's `skev.pkg` uses zero new syntax — developers already know everything needed to write a package file. C++23 requires learning CMake (a language in itself) plus a separate vcpkg format. C# `.csproj` is XML — readable by compilers, not humans. Python's `pyproject.toml` is TOML — simple but still a third format to learn. Rust's `Cargo.toml` is the closest comparison — clean and effective. Skev's advantage over Rust's TOML: the same `>>` `<<` block structure, `::` property declarations, and `extends` inheritance that appear everywhere else in the language. The package file feels like Skev code because it is Skev code.

---

### 🔴 Hard Complex Example — Multi-Platform AAA Studio Package

**🌍 Real-World Scenario**

A AAA studio ships the same game on PC (Windows, macOS, Linux), three console platforms (PS5, Xbox, Switch), and mobile (iOS, Android). They have a proprietary engine library hosted on a private internal registry, an NDA console SDK that must not appear in public source, multiple build profiles per platform, a tools build for the level editor, and a standalone C library export for their in-house middleware team.

**💥 Why This Is Hard**

Multi-platform AAA builds require: platform-specific dependencies, private registries that coexist with the public hub, console SDK integration under NDA, conditional builds per platform, multiple output targets from the same source, and a tools build that shares most code but has a different entry point and output format.

**In C++23:**
```cpp
// This requires:
// → CMakeLists.txt (main build — 200+ lines)
// → vcpkg.json (public dependencies)
// → vcpkg-private.json (private dependencies)
// → platform/ps5/CMakeLists.txt (console — under NDA)
// → platform/xbox/CMakeLists.txt (console — under NDA)
// → platform/switch/CMakeLists.txt (console — under NDA)
// → tools/CMakeLists.txt (editor tools)
// Total: 6+ files, 500+ lines, multiple formats
// ~500 lines total [abbreviated — typical AAA CMake setup]
```

**In C# 13:**
```xml
<!-- Requires separate .csproj per platform -->
<!-- GamePC.csproj, GameConsole.csproj, GameMobile.csproj -->
<!-- GameTools.csproj -->
<!-- Directory.Build.props for shared properties -->
<!-- NuGet.Config for private registry -->
<!-- Total: 5+ files, 300+ lines -->
// ~300 lines total [abbreviated — typical .NET multi-platform]
```

**In Python 3.13:**
```toml
# Python is not used for AAA game builds
# Not applicable to this use case
// N/A — Python not used for multi-platform native game builds
```

**In Skev:**
```swift
package AAAGameEngine >>
    version     :: "4.0.0"
    edition     :: "2026"
    author      :: "AJ Studios"
    license     :: "Proprietary"

    # Public dependencies — from packages.skev.dev
    needs skev.physics    :: "^4.0.0"
    needs skev.render     :: "^3.1.0"
    needs skev.network    :: "^2.0.0"
    needs skev.audio      :: "^1.8.0"

    # Private dependency — from studio's internal registry
    needs ajstudios.engine_core :: "^12.0.0"
        registry :: "https://packages.ajstudios.internal"
    << needs

    dev_needs skev.testing  :: ">=1.0.0"
    dev_needs skev.profiler :: ">=2.0.0"

    # PC release — primary platform
    target pc_release >>
        entry    :: "src/main.skev"
        platform :: [windows-x64, windows-arm64,
                     macos-arm64, macos-x64,
                     linux-x64, linux-arm64]
        link     :: static
        lto      :: true
        output   :: "AAAGame"
    << pc_release

    # PC debug — extends release, adds dev tools
    target pc_debug >>
        extends    :: pc_release
        lto        :: false
        debug_info :: true
        hot_reload :: true
        assertions :: true
        output     :: "AAAGame_Debug"
    << pc_debug

    # Mobile — extends release, mobile platforms
    target mobile >>
        extends  :: pc_release
        platform :: [ios-arm64, android-arm64]
        output   :: "AAAGame_Mobile"
    << mobile

    # Console — extends release, NDA platforms
    # Console SDK details in private registry under NDA
    target console >>
        extends  :: pc_release
        platform :: [ps5, xbox, switch]
        needs ajstudios.console_sdk :: "^4.0.0"
            registry :: "https://packages.ajstudios.console"
        << needs
        output   :: "AAAGame_Console"
    << console

    # Level editor tools — different entry point
    target tools >>
        entry    :: "tools/editor_main.skev"
        platform :: [windows-x64, macos-arm64]
        link     :: dynamic
        output   :: "SkevEditor"
    << tools

    # C library export — for middleware team
    target middleware_lib >>
        entry    :: "src/middleware_api.skev"
        platform :: [windows-x64, linux-x64]
        link     :: static
        export   :: "C"
        output   :: "libajengine"
    << middleware_lib

<< AAAGameEngine
// 56 lines total — one file, one format, every platform
```

### 🧭 Walk-Through — AAA Package

**Private registry per dependency:** `needs ajstudios.engine_core :: "^12.0.0"` with a nested `registry ::` block. The public Package Hub is used for `skev.*` dependencies automatically. Private registries only apply to the specific dependency that declares them.

**`extends` inheritance:** `pc_debug` extends `pc_release` — inherits all 5 fields and overrides 4. `mobile` and `console` both extend `pc_release` — each adds only what differs. No duplication. If `pc_release` changes, all targets that extend it update automatically.

**Console under NDA:** The console target uses `extends :: pc_release` and adds a private registry dependency. The console SDK function names and APIs are in the private registry under NDA — the public `skev.pkg` shows the dependency exists but not its contents. Same pattern Unity and Unreal use.

**`export :: "C"` target:** The `middleware_lib` target compiles Skev code and generates C headers automatically — exactly the `export "C"` mechanism from Chapter 8, integrated directly into the build system.

### ✅ Honest Advantage Statement

Skev's single 56-line `skev.pkg` replaces 500+ lines across 6+ files in C++. C# requires separate `.csproj` files per major platform. The `extends` inheritance is the critical advantage — it eliminates the duplication that makes multi-platform build files so hard to maintain. When you change a shared setting, you change it in one place. Every target that extends it updates. Honest trade-off: Skev is new and has fewer third-party packages than C++ (vcpkg) or C# (NuGet). The build system quality exceeds the ecosystem size at this stage.

---

## 9.3 — Version Specification Syntax

### 👨‍💻 Developer Version

Skev uses semantic versioning — the same standard used by npm, Cargo, and NuGet. A version number has three parts:

```
2  .  4  .  1
↑     ↑     ↑
│     │     └── PATCH:  Bug fix. Nothing changes for callers.
│     └──────── MINOR:  New feature added. Old code still works.
└────────────── MAJOR:  Breaking change. Old code may not compile.
```

**Version constraints — what you write in `needs`:**

```swift
# Exact version — only this
needs skev.physics :: "2.4.1"

# Minimum version — this or higher
needs skev.physics :: ">=2.0.0"

# Compatible — same major, any minor/patch
needs skev.physics :: "^2.0.0"     # 2.0.0 to <3.0.0

# Patch-level — same major and minor
needs skev.physics :: "~2.4.0"     # 2.4.0 to <2.5.0

# Range
needs skev.physics :: ">=2.0.0 <3.0.0"
```

**The `^` operator — the most common choice:**

```
^2.0.0 means: "I need at least 2.0.0 but I trust
               the author not to break things in 2.x"

This is right for most dependencies.
The library author promised: minor versions don't break.
^ holds them to that promise.
```

### ✅ Honest Advantage Statement

Skev's version syntax is identical to npm and Cargo — the two most successful package ecosystems in programming. This is intentional. Developers who know either system know Skev's versioning immediately. No new concepts to learn here.

---

## 9.4 — Skev Package Hub

### 👨‍💻 Developer Version

The Skev Package Hub at `packages.skev.dev` is the central home for all public Skev packages. Think of it like npm for JavaScript or crates.io for Rust — but built with lessons learned from both.

**What makes it different from older registries:**

```
1. Content-addressed — immutable packages
   Once you publish skev.physics 2.4.1 —
   it is that version forever.
   Nobody can replace it. Nobody can change it.
   The npm "left-pad" incident cannot happen.
   (A tiny package was deleted and broke 300,000 projects.)

2. Cryptographic signing — verified packages
   Every package is signed with the author's keypair.
   Skev Hub verifies the signature when you install.
   A tampered package = installation refused.
   Automatic. Silent. Always on.

3. Reserved namespaces
   "skev.*" is reserved for official Skev packages only.
   Nobody can publish "skev.physics2" and trick developers.
   
4. Dependency audit built in
   skev audit scans your dependencies for known
   security vulnerabilities. Skev Studio shows
   warnings inline as you code.
```

**Publishing to the Hub:**

```swift
# In your terminal:
skev publish

# What Skev does:
# 1. Builds your package in release mode
# 2. Signs it with your registered keypair
# 3. Uploads to packages.skev.dev
# 4. Verifiable immediately by anyone
```

### ⚙️ Technical Version

The Skev Package Hub uses content-addressed storage: each package version is stored by the SHA-256 hash of its contents. The version string (e.g. `skev.physics 2.4.1`) is a pointer to this hash. Once a version is published, the hash is immutable — the pointer cannot be redirected. Package signing uses Ed25519 keypairs registered with the Hub at account creation. The Hub validates signatures server-side at upload time and publishes the signature alongside the package. `skev install` validates signatures client-side before extracting — a mismatched signature causes installation failure with a clear error.

---

## 9.5 — The `skev build` CLI

### 👨‍💻 Developer Version

The entire Skev build system is one command: `skev build`. Everything else is an optional flag or subcommand.

**Complete CLI reference:**

```
skev build              Build current package (debug profile)
skev build --release    Build with optimisations
skev build --target T   Cross-compile for target T
skev run                Build and run
skev run --release      Build release and run
skev test               Build and run all tests
skev package            Build distributable package
skev publish            Sign and publish to Package Hub
skev audit              Check all dependencies for vulnerabilities
skev audit --abi        Check binary ABI compatibility
skev audit --security   Check for known CVEs only
skev clean              Clear build cache and output
skev migrate            Upgrade code to new edition
skev lock               Regenerate skev.pkg.lock
skev lock --resolve     Resolve lock file merge conflicts
```

**Build profiles — in `skev.pkg`:**

```swift
build >>

    debug >>
        optimise   :: false
        debug_info :: true
        assertions :: true     # assert() checks active
        hot_reload :: true     # Skev Studio live editing
    << debug

    release >>
        optimise   :: true
        debug_info :: false
        assertions :: false    # stripped — maximum speed
        hot_reload :: false
        lto        :: true     # link-time optimisation
    << release

<< build
```

**Zero configuration for simple projects:**

```swift
# Minimal skev.pkg — this is all you need:
package HelloGame >>
    version :: "0.1.0"
<< HelloGame

# skev build — works immediately
# skev run   — works immediately
# No configuration required
```

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A solo developer wants to start a new Skev game project, add a physics dependency, and build a release binary for Windows and macOS. They want this to take under 5 minutes of setup.

**In C++23:**
```bash
# C++ — must choose: CMake? Meson? Bazel? xmake?
# Using CMake + vcpkg (most common):
mkdir game && cd game
git init
git submodule add https://github.com/microsoft/vcpkg.git
./vcpkg/bootstrap-vcpkg.sh
echo '{ "dependencies": ["physx"] }' > vcpkg.json
# Write CMakeLists.txt — 20+ lines minimum
# Configure, generate, build — 3 separate steps
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=vcpkg/scripts/...
cmake --build build --config Release
# Time: 15-30 minutes first time (vcpkg downloads + compiles deps)
// ~8 lines total (just the commands — CMakeLists is additional)
```

**In C# 13:**
```bash
# C# — better than C++ but still multiple steps
dotnet new console -n HelloGame
cd HelloGame
dotnet add package PhysicsLib
# Edit .csproj for platform targets
dotnet build -c Release -r win-x64
dotnet build -c Release -r osx-arm64
# Time: 5-10 minutes (NuGet download is fast)
// 6 lines total
```

**In Python 3.13:**
```bash
# Python — simple but not for native game binaries
mkdir game && cd game
python -m venv venv
source venv/bin/activate
pip install pygame
# Python does not produce native optimised game binaries
# Not applicable for performance game development
// 4 lines total — but produces interpreted code, not native binary
```

**In Skev:**
```bash
skev new HelloGame
cd HelloGame
# Edit skev.pkg — add: needs skev.physics :: "^4.0.0"
skev build --release --target windows-x64
skev build --release --target macos-arm64
# Time: 2-3 minutes (Skev Package Hub download is fast)
// 5 lines total — produces native optimised binaries
```

### ✅ Honest Advantage Statement

`skev new` + `skev build` is the fastest path to a native game binary among all comparison languages. C++ requires choosing a build system, setting up a package manager separately, and learning CMake syntax. C# is good — `dotnet` is clean — but the `.csproj` XML adds friction. Python cannot produce optimised native binaries. Skev's honest advantage is the combination of: zero build system knowledge required, cross-compilation with a single flag, and the same Skev syntax in the package file. The time comparison (2-3 minutes vs 15-30 minutes for C++) reflects real dependency management speed differences, not artificial optimisation.

---

## 9.6 — Backward Compatibility — The Edition System

### 👨‍💻 Developer Version

Every programming language faces the same dilemma eventually:

```
The language needs to improve.
But millions of lines of existing code must keep working.
How do you do both?
```

Three approaches languages have tried:

```
Java:      Never break anything.
           Result: language accumulates technical debt.
           Methods deprecated for 20 years but never removed.

Python 2→3: Break everything, migrate the ecosystem.
            Result: 10+ years of painful migration.
            Some projects never moved.

Rust:      Edition system — opt in to new behaviour.
           Old code compiles in its edition forever.
           New code can use improved syntax.
           This works. Skev uses the same approach.
```

**How Skev editions work:**

```swift
# Every package declares its edition
package MyGame >>
    edition :: "2026"    # Skev Edition 2026
    version :: "1.0.0"
<< MyGame
```

```
Skev Edition 2026:  The current edition. All features as designed.

Skev Edition 2026:  May introduce improved syntax for some patterns.
                    Code written in Edition 2025 still compiles.
                    Edition 2025 code is never broken.

Rules:
→ An edition NEVER breaks existing code in that edition.
→ New editions may offer improved alternatives — never remove old ones.
→ Skev compiler always supports the last 3 editions.
→ skev migrate upgrades code automatically before any deprecation.
→ Deprecation announced minimum 2 editions in advance.

The guarantee:
"Code written today in Skev Edition 2026 must compile in 10 years."
```

**`skev migrate` — automatic edition upgrade:**

```
skev migrate --to 2026

What it does:
→ Analyses all .skev files in the project
→ Finds patterns deprecated in Edition 2026
→ Rewrites them to the new syntax
→ Creates a git commit with all changes
→ Shows diff before applying — you approve first
→ Inserts # TODO: manual migration needed
  for any case it cannot resolve automatically
→ Prints a migration report: X automatic, Y manual
```

### ✅ Honest Advantage Statement

Skev's edition system is directly inspired by Rust editions — which have proven successful. The honest position: Rust has been doing editions since 2018 and has worked out many edge cases. Skev benefits from that experience. The key addition in Skev: `skev migrate` runs automatically before any deprecation is enforced — developers are never surprised by a breaking change without a migration path.

---

## 9.7 — Supply Chain Safety

### 👨‍💻 Developer Version

The most dangerous attacks on software today do not exploit your code. They exploit your dependencies.

```
npm event-stream (2018):
  Popular package with 2 million weekly downloads.
  Maintainer transferred ownership.
  New owner added cryptocurrency theft code.
  Shipped to millions of projects.
  Took months to discover.

PyPI typosquatting (ongoing):
  "requesrs" instead of "requests" — malware.
  "colourama" instead of "colorama" — malware.
  Developers install typos. Malware runs.

Skev is designed to make both of these impossible.
```

**Three-layer defence:**

**Layer 1 — Cryptographic package signing**

```
Every package author registers a public/private keypair
with the Skev Package Hub.

When a package is published:
→ It is signed with the author's private key
→ The signature is stored with the package on the Hub

When you install a package:
→ skev install verifies the signature automatically
→ Mismatched signature = installation refused
→ No manual step needed — always on

What this prevents:
→ A compromised Hub cannot serve modified packages
  (signature would not match)
→ A man-in-the-middle attack cannot inject code
  (signature would not match)
```

**Layer 2 — Ownership transfer protection**

```
If a package maintainer wants to transfer ownership:
→ 72-hour cooling-off period before transfer completes
→ Email confirmation required from BOTH parties
→ Public announcement added to package changelog
→ Skev Studio shows warning to all developers using
  the package for 7 days after transfer

What this prevents:
→ The npm event-stream attack:
  Attacker convinces maintainer to transfer.
  72-hour window allows community to notice.
  7-day Studio warning gives developers time to audit.
```

**Layer 3 — `skev.pkg.lock` — reproducible builds**

```swift
# skev.pkg.lock — generated automatically
# ALWAYS committed to source control
# NEVER manually edited

lock >>
    skev.physics    :: "2.4.1" hash: "sha256:a3b2c1d4e5..."
    skev.network    :: "1.8.0" hash: "sha256:f6g7h8i9j0..."
    skev.render     :: "3.1.2" hash: "sha256:k1l2m3n4o5..."
    skev.pathfinder :: "1.4.2" hash: "sha256:p6q7r8s9t0..."
<< lock
```

```
What skev.pkg.lock does:
→ Records the exact version AND exact bytes of every dependency
→ Every developer on the team builds with identical binaries
→ If a dependency changes (even a patch version) — the hash
  changes — lock file mismatch — build fails
→ Developer sees: "Dependency changed. Review and approve."
→ Deliberate update = run skev lock to regenerate
→ Accidental or malicious change = caught immediately

What this prevents:
→ "Works on my machine" — same bytes everywhere
→ Surprise updates that break things
→ Silent supply chain substitution
```

**Handling lock file merge conflicts:**

```
When two developers update different dependencies
and both commit skev.pkg.lock — merge conflict.

Resolution:
skev lock --resolve

This re-runs dependency resolution from skev.pkg
(not from the conflicted lock file).
Both developers' updates are evaluated together.
Version conflicts surface as clear errors with fix suggestions.
```

### 🔴 Hard Complex Example — Studio Under Attack

**🌍 Real-World Scenario**

A game studio's CI pipeline starts producing builds with unexpected network calls. Security team investigates and finds a compromised dependency three levels deep in the dependency tree (a dependency of a dependency of a dependency). They need to identify the compromised package, block it, and continue shipping without it.

**💥 Why This Is Hard**

Transitive dependency attacks are nearly invisible. The compromised code is not in your direct dependencies — it is hidden three levels down. Without content-addressed hashing, you cannot even verify when the change was introduced. Without build reproducibility, you cannot distinguish "this was always here" from "this appeared today."

**In C++23:**
```cpp
// C++ has no standard supply chain tooling
// vcpkg has no built-in security audit
// Developer must:
// → Manually audit CMakeLists for all transitive deps
// → Check each library's source manually
// → Hope the library author noticed
// → No automated detection possible
// ~0 lines of tooling — manual investigation only
```

**In C# 13:**
```csharp
// NuGet has some audit support in .NET 8+
dotnet list package --vulnerable
// Shows packages with known CVEs
// But: no content-addressed storage
// A package CAN be silently modified
// NuGet audit only catches KNOWN CVEs — not novel attacks
// 1 line total — limited scope
```

**In Python 3.13:**
```python
# pip-audit exists (third-party)
pip install pip-audit
pip-audit
# Same limitation as NuGet: only known CVEs
# No content-addressed storage
# PyPI has had multiple supply chain attacks
// 2 lines total — third-party, known CVEs only
```

**In Skev:**
```bash
# Step 1 — audit detects the anomaly
skev audit --security
# Output:
# ⚠️  SECURITY ALERT
# skev.pathfinder 1.4.2 (transitive dependency via skev.ai)
# Hash mismatch detected:
#   Expected: sha256:p6q7r8s9t0...
#   Got:      sha256:DIFFERENT...
# This package has been modified since your lock file was created.
# DO NOT USE until verified.

# Step 2 — Studio shows inline warning on all files using it
# [Skev Studio already flagged this 20 minutes ago]

# Step 3 — pin to verified version
# Edit skev.pkg:
needs skev.pathfinder :: "1.4.1"    # known-good version

# Step 4 — regenerate lock with verified hashes
skev lock

# Step 5 — build proceeds safely
skev build --release
// 6 lines total — attack detected, contained, resolved
```

### 🧭 Walk-Through — Supply Chain Attack Response

**Hash mismatch detection:** `skev.pkg.lock` stores `sha256:p6q7r8s9t0...` for `skev.pathfinder 1.4.2` when the project was last built cleanly. `skev audit` recomputes the hash of the currently installed package. If the package was modified after being published — even one byte changed — the hash is different. The mismatch is caught immediately. No CVE database lookup needed — pure cryptographic verification.

**Transitive depth:** `skev.pathfinder` is three levels deep. Skev's audit traces the full dependency graph. The compromised package is found regardless of depth.

**Studio integration:** Skev Studio runs `skev audit` in the background and shows the warning as an inline code annotation — developers see it before running any code, not after a production incident.

**Pinning to safe version:** Pinning to `1.4.1` (the last known-good version) and running `skev lock` regenerates the lock file with the `1.4.1` hash. The compromised `1.4.2` cannot be used — even accidentally.

### ✅ Honest Advantage Statement

**Why don't established languages have this already?**

This is the right question to ask — and the honest answer is more interesting than "Skev is simply better."

```
Reason 1 — They were designed before the threat existed:
  C++ standard:    1998  (no internet package ecosystem)
  Python pip:      2008  (supply chain attacks were rare)
  NuGet (C#):      2010  (same era)
  Rust/Cargo:      2014  (first to learn from npm's mistakes)

  The biggest attacks happened AFTER these systems launched:
  npm event-stream attack:  2018
  SolarWinds:               2020
  Log4Shell:                2021
  PyPI mass typosquatting:  2022-2023

  Skev is designed in 2025.
  We saw every one of those attacks before writing the spec.
  This is timing — not cleverness.

Reason 2 — Retrofitting security breaks existing users:
  Python has 500,000+ packages published WITHOUT signing.
  If PyPI made signing mandatory tomorrow:
  → Every existing package becomes invalid
  → Every CI pipeline breaks
  → Abandoned package maintainers must be tracked down
  → Millions of production projects break

  So PyPI makes signing optional (they did in 2023).
  But optional signing does not prevent attacks —
  attackers just publish unsigned packages.

  Skev has zero packages right now.
  Zero users who would be broken.
  Mandatory signing costs nothing when you start from zero.

Reason 3 — Established languages move slowly by design:
  To add mandatory signing to PyPI:
  → Write a PEP
  → Get consensus from thousands of developers
  → Implement across pip, poetry, pdm, conda...
  → Ship across multiple Python versions
  Timeline: typically 3-5 years

  Skev's security decisions: one conversation, locked immediately.

Reason 4 — They ARE fixing it — just slowly:
  Rust/Cargo already has content-addressed storage ✅
  npm added SLSA provenance attestations in 2023 (optional)
  NuGet has package signing since 2018 (optional, rarely used)
  The direction is right. The pace is constrained by inertia.
```

**What Skev actually did — honest:**

```
Every layer of Skev's security is borrowed from existing best practice:

Content-addressed storage  →  borrowed from Cargo/crates.io
Cryptographic signing      →  borrowed from Debian apt (since 2001)
Ownership transfer rules   →  learned from npm's failed policies
Lock files with hashes     →  borrowed from Cargo.lock

Skev takes best practices scattered across different ecosystems,
makes ALL of them mandatory from day one,
in a single coherent system,
for a language with zero legacy to protect.

That is the real advantage:
Not invention — synthesis.
Not superiority — timing.
Not better engineers — a blank slate.
```

The honest limitation: `skev.pkg.lock` must be committed to source control. Teams that skip this lose the hash-mismatch detection capability — the same way Git cannot protect uncommitted files. The system is only as strong as the team's discipline to commit the lock file.

---

## 9.8 — Linking & Loading

### 👨‍💻 Developer Version

When Skev compiles your game, it needs to combine your code with all your dependencies into something that can actually run. This process is called linking.

**Two kinds of linking:**

```
Static linking — everything in one file:
  All dependencies compiled into your game binary.
  Result: one large executable file.
  Advantage: no missing dependencies at runtime.
             copy the .exe — it runs anywhere.
  Disadvantage: larger file.
  Best for: console games, mobile apps, embedded.

Dynamic linking — dependencies as separate files:
  Your game is small. Dependencies are .dll/.so files.
  Result: small executable + separate library files.
  Advantage: multiple programs can share one library.
             update a library without recompiling the game.
  Disadvantage: library files must be present at runtime.
  Best for: PC games with plugins, tools, editors.
```

**Declaring in `skev.pkg`:**

```swift
target pc_game >>
    link :: static     # everything in one binary
<< pc_game

target editor_tools >>
    link :: dynamic    # plugins load at runtime
<< editor_tools
```

**The Skev binary loading sequence:**

```
When your game launches:
1. OS maps the Skev binary into memory
2. Skev runtime initialises:
   → ARC system ready
   → Thread pool created (hardware_concurrency - 2 threads)
   → Main entity system initialised
3. Entry point entity fires when scene_load event
4. Game runs

No hidden global state.
No static initialisation order problems.
(A notorious C++ issue where global objects
 initialise in unpredictable order.)
Deterministic every single time.
```

### ✅ Honest Advantage Statement

Static vs dynamic linking works identically in C++ and C# — Skev adds no unique value here. The genuine advantage is the declaration syntax: `link :: static` in `skev.pkg` is a single property — no CMake `STATIC`/`SHARED` keywords, no `.csproj` output type XML. The deterministic loading sequence eliminates an entire class of C++ bugs — static initialisation order fiascos — by having no static initialisation at all.

---

## 9.9 — Deployment Targets

### 👨‍💻 Developer Version

Skev targets are the platforms your game can run on. Cross-compilation is a first-class feature — build for any target from any machine:

**All supported targets:**

```
DESKTOP:
  windows-x64      Windows 10+  (x86-64)
  windows-arm64    Windows 11   (ARM — Surface Pro X, Snapdragon)
  macos-arm64      macOS 11+    (Apple Silicon — M1/M2/M3)
  macos-x64        macOS 10.15+ (Intel — legacy support)
  linux-x64        Linux        (x86-64, glibc 2.31+)
  linux-arm64      Linux        (ARM64 — Raspberry Pi 4+)

MOBILE:
  ios-arm64        iOS 14+      (iPhone, iPad)
  android-arm64    Android 8.0+ (API level 26+)

CONSOLE (developer program required — under NDA):
  ps5              PlayStation 5
  xbox             Xbox Series X/S
  switch           Nintendo Switch

WEB:
  wasm32           WebAssembly  (all modern browsers)

EMBEDDED / ROBOTICS:
  linux-arm64-bare Linux ARM64  (no GUI — robots, controllers)
  rtos             Real-time OS (vendor-specific — contact Skev team)
```

**Cross-compilation from any host:**

```bash
# Build for a robot controller, from your Mac:
skev build --target linux-arm64-bare --release

# Build for browser, from Windows:
skev build --target wasm32 --release

# Build for all desktop platforms:
skev build --target windows-x64 --release
skev build --target macos-arm64 --release
skev build --target linux-x64   --release
```

---

## 9.10 — Save File Compatibility

### 👨‍💻 Developer Version

Games get updates. Updates change data structures. Players have save files from version 1.0. When version 2.0 adds new fields — what happens to old saves?

**The two bad outcomes to avoid:**

```
Game crashes loading old save     →  terrible player experience
Game silently loses player data   →  also terrible
```

**Skev's solution — versioned serialisation with migration:**

```swift
#! serialisable(version: 3)
data PlayerData >>
    name       :: string
    level      :: int
    position   :: Vector3!
    health     :: int    = 100     # added in version 2
    abilities  :: list[string]     # added in version 3
    # note: "gold" was in version 1 and 2 — removed in version 3
    # the migration handler below converts it
<< PlayerData
```

**Migration handlers — required when the version increments:**

```swift
migration PlayerData >>

    # Load a version 1 save (no health, no abilities, had gold)
    from_version_1 >> old
        result PlayerData >>
            name      :: old.name
            level     :: old.level
            position  :: old.position
            health    :: 100         # not in v1 — use default
            abilities :: []          # not in v1 — start empty
            # gold (old.gold) discarded — not in v3
        << PlayerData
    << from_version_1

    # Load a version 2 save (has health, no abilities, had gold)
    from_version_2 >> old
        result PlayerData >>
            name      :: old.name
            level     :: old.level
            position  :: old.position
            health    :: old.health  # exists in v2 — carry forward
            abilities :: []          # not in v2 — start empty
            # gold (old.gold) discarded — not in v3
        << PlayerData
    << from_version_2

<< migration
```

**Compiler enforcement:**

```
If you increment the version number in #! serialisable —
the compiler checks for a matching migration handler.

If the migration is missing:

Error: Missing migration handler
  PlayerData version was incremented from 2 to 3.
  A migration handler is required for from_version_2.

  Add to your migration block:
  from_version_2 >> old
      result PlayerData >> ... << PlayerData
  << from_version_2

  Without this — old saves cannot be loaded safely.

  player_data.skev  Line 1
```

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A live-service RPG ships version 1.5 which adds a new `abilities` field to player saves. Players who saved in version 1.0 (no abilities field) load the game. The game must not crash, must not lose their other data, and must give them a sensible default for the new field.

**In C++23:**
```cpp
// C++ has no standard solution — developer writes everything
struct PlayerData {
    int version;
    std::string name;
    int level;
    // v1.5 addition:
    std::vector<std::string> abilities;
};

PlayerData LoadPlayer(std::ifstream& file) {
    PlayerData p;
    file.read(reinterpret_cast<char*>(&p.version), sizeof(int));

    if (p.version == 1) {
        // Manual v1 migration
        file.read(/* name */);
        file.read(/* level */);
        p.abilities = {};    // default — must remember manually
    } else if (p.version == 2) {
        // full load
    }
    return p;
}
// ~25 lines total — manual, error-prone, no compiler enforcement
// Developer must remember to handle every old version
// Forgetting one = crash for players on that version
```

**In C# 13:**
```csharp
// C# uses JSON with [JsonIgnore] or versioned DTOs
// No built-in migration — developer writes manually
[Serializable]
public class PlayerData {
    public int Version { get; set; }
    public string Name { get; set; }
    public int Level { get; set; }
    // Added in v1.5:
    public List<string> Abilities { get; set; } = new();
}

// JsonSerializer handles missing fields with default values
// But: no compiler enforcement that migrations exist
// Adding a field silently "works" — but removed fields
// cause deserialization errors with no migration path
// ~12 lines total — works for additions, fragile for removals
```

**In Python 3.13:**
```python
# Python — same manual pattern as C++/C#
import json

def load_player(file_path: str) -> dict:
    with open(file_path) as f:
        data = json.load(f)

    version = data.get("version", 1)
    if version < 2:
        data.setdefault("health", 100)
    if version < 3:
        data.setdefault("abilities", [])
    return data
# 9 lines total — clean for additions but:
# No compiler enforcement
# Removed fields silently ignored (data loss)
# No type safety on migrated values
```

**In Skev:**
```swift
#! serialisable(version: 3)
data PlayerData >>
    name       :: string
    level      :: int
    position   :: Vector3!
    health     :: int          = 100
    abilities  :: list[string]
<< PlayerData

migration PlayerData >>
    from_version_1 >> old
        result PlayerData >>
            name :: old.name   level :: old.level
            position :: old.position
            health :: 100      abilities :: []
        << PlayerData
    << from_version_1

    from_version_2 >> old
        result PlayerData >>
            name :: old.name   level :: old.level
            position :: old.position
            health :: old.health   abilities :: []
        << PlayerData
    << from_version_2
<< migration
// 20 lines total — compiler enforces migration exists
// Removed fields handled explicitly — no silent data loss
// Type-safe migration — compiler verifies field types
```

### ✅ Honest Advantage Statement

Skev's versioned serialisation is the only approach where missing migrations are compile errors. C++ and Python rely entirely on developer discipline — forgetting a migration version is silent until a player with an old save reports a crash. C# handles field additions automatically but fails silently on field removals. Skev requires explicit handling of every version transition — verbose for simple cases, essential for complex ones. The honest trade-off: 20 lines vs 9 lines for Python. The gain: a player who saved in version 1.0 cannot crash the game two years later.

---

## 9.11 — Private Package Registries

### 👨‍💻 Developer Version

Not everything belongs on the public Package Hub. Proprietary engine code. Licensed middleware. Console SDKs under NDA. Internal tools that are not ready for public release.

Private registries work identically to the public Hub — same format, same signing, same verification — but hosted on your own infrastructure:

```swift
# Package hosted entirely on private registry
package StudioEngine >>
    registry :: "https://packages.ajstudios.internal"
    version  :: "4.0.0"
<< StudioEngine

# Mixed: most deps public, one dep private
package OpenWorldRPG >>
    needs skev.physics :: "^4.0.0"          # public Hub
    needs skev.network :: "^2.0.0"          # public Hub

    needs ajstudios.engine :: "^12.0.0"     # private registry
        registry :: "https://packages.ajstudios.internal"
    << needs

<< OpenWorldRPG
```

**Enterprise and offline support:**

```
Air-gapped studios (no internet access — common in console dev):
→ Mirror the public Hub internally
→ All skev.* packages served from the mirror
→ Zero internet access required for any build
→ Console certification environments fully supported

Console platform holders:
→ Require private registries for platform SDKs
→ Same mechanism — different URL
→ SDK details remain under NDA
→ Public package file shows dependency exists — not its contents
```

---

## 9.12 — Emergency Security Response

### 👨‍💻 Developer Version

When a critical vulnerability is found in a Skev package — the response process is fast, transparent, and developer-friendly:

```
Step 1 — Report
  Researcher reports to security@skev.dev
  Skev team acknowledges within 24 hours

Step 2 — Verify & Assign
  Skev security team verifies the vulnerability
  CVE number assigned
  Package maintainer notified privately

Step 3 — Fix Window
  Maintainer has 30 days to publish a fix
  Skev team provides technical assistance

Step 4 — Public Advisory
  If unfixed after 30 days:
  → Public advisory published on skev.dev/security
  → Package Hub marks package as "security risk"
  → skev install shows prominent warning
  → skev audit --security flags it in all projects

Step 5 — Emergency Yank (critical vulnerabilities only)
  For actively exploited vulnerabilities:
  → Skev team can force-yank the package from the Hub
  → All installations receive warning from Skev Studio
  → skev build fails with security error if yanked package used
  → Developer must explicitly acknowledge before proceeding
  → Clear instructions provided for safe alternative
```

---

## 9.13 — Build & Distribution Quick Reference

```
PACKAGE FILE:
  skev.pkg               package definition in Skev syntax
  package Name >>        open package block
      version  :: "X.Y.Z"
      edition  :: "2025"
      needs X  :: "^1.0.0"    runtime dependency
      dev_needs X :: ">=1.0.0" dev-only dependency
      target name >>     build target
          entry    :: "src/main.skev"
          platform :: [windows-x64, macos-arm64]
          link     :: static / dynamic
          extends  :: other_target
          lto      :: true / false
      << name
  << Name

VERSION CONSTRAINTS:
  "2.4.1"        exact version
  ">=2.0.0"      minimum version
  "^2.0.0"       compatible (2.x.x)
  "~2.4.0"       patch-level (2.4.x)

CLI COMMANDS:
  skev build              debug build
  skev build --release    release build
  skev build --target T   cross-compile
  skev run                build + run
  skev test               build + test
  skev publish            sign + publish to Hub
  skev audit              vulnerability scan
  skev migrate            edition upgrade
  skev lock               regenerate lock file
  skev lock --resolve     fix merge conflicts

SUPPLY CHAIN:
  skev.pkg.lock           commit this — always
                          content-addressed hashes
                          hash mismatch = build fails

EDITIONS:
  edition :: "2026"       in skev.pkg
  skev migrate --to 2026  upgrade code
  Old code always compiles in its declared edition

SAVE FILE VERSIONING:
  #! serialisable(version: N)    on data type
  migration TypeName >>           migration block
      from_version_N >> old       handler per version
          result TypeName >> ...
      << from_version_N
  << migration
  Missing handler = compile error

DEPLOYMENT TARGETS:
  Desktop:  windows-x64 windows-arm64 macos-arm64
            macos-x64 linux-x64 linux-arm64
  Mobile:   ios-arm64 android-arm64
  Console:  ps5 xbox switch (NDA — developer program)
  Web:      wasm32
  Embedded: linux-arm64-bare rtos
```

---

*End of Chapter 9 v0.1 — Next: Chapter 3.5: Generics*
