<!--
Copyright © 2026 AJ. All Rights Reserved.
Skev Programming Language Specification.
Unauthorized reproduction, distribution, or use of this material
without explicit written permission from the author is prohibited.
skev.dev | skev.org
-->

# Skev Language Specification
## Chapter 6: Error Handling
**Version:** 0.1 — DRAFT
**Authors:** AJ (Copyright © 2026)
**Status:** In Progress
**Depends On:** Chapter 3 (types), Chapter 4 (ARC), Chapter 5 (async/tasks)
**Use Cases Tested:** All 7 — Games, XR/VR/AR, Simulations, Robotics, Animation/VFX, Interactive Apps, Real-time Networking
**Parameters Verified:** Speed ✅ Reliability ✅ Security ✅ Readability ✅

---

## 6.1 — Overview

### 👨‍💻 Developer Version

Every program fails. Files go missing. Networks time out. Players give invalid input. Save data gets corrupted. The question is never *if* things go wrong — it is *how your code responds when they do.*

Skev's error handling is built on one principle from Chapter 1:

> **Errors should be impossible to ignore.**

In C++ you can forget to check a return code. In Python an exception can silently propagate past code you expected to run. In C# you can swallow exceptions with an empty catch block and never know.

In Skev — you cannot ignore an error. The compiler will not let you. Every fallible operation must be handled, propagated, or explicitly given a fallback. There is no silent path. There is no forgetting.

**Skev has two categories of failure:**

```
Expected Failures — result[T]
  Things that CAN go wrong in normal program flow.
  File not found. Network timeout. Invalid input.
  YOU decide how to respond.
  These are part of your program's logic.

Programming Errors — panic
  Things that should NEVER happen in correct code.
  Array out of bounds. Division by zero.
  These mean YOU made a mistake.
  The program stops. You fix the code.
```

**The rule of thumb:**
```
If it could happen to a correct program  →  result[T]
If it only happens due to a bug          →  panic
```

### ⚙️ Technical Version

Skev implements a Result type system inspired by Rust's `Result<T,E>` and Swift's `throws`, but with game-specific design priorities: zero overhead on the success path, automatic cleanup via ARC on failure paths, and compiler-enforced exhaustive error handling. The system uses three mechanisms: `result[T]` as a discriminated union (success value or error), `->` propagation for clean error chaining, and `panic` for unrecoverable programming errors. No runtime exception infrastructure is required — `result[T]` compiles to a tagged union with the same memory layout as `maybe T`, and `->` propagation compiles to conditional branches with zero overhead when no error occurs.

---

## 6.2 — The `result[T]` Type

### 👨‍💻 Developer Version

`result[T]` is Skev's way of saying: *this operation might succeed with a value of type T, or it might fail with an error.*

> `result[T]` means: I tried to give you a T, but something might have gone wrong.

**Declaring a fallible function:**

```swift
# This function might succeed (returns LevelData)
# or might fail (returns an error)
load_level(path: string) -> result[LevelData]

# This function might succeed (returns int)
# or might fail (returns an error)
parse_damage(raw: string) -> result[int]

# This function might succeed (no value returned)
# or might fail
save_game(slot: int) -> result[nothing]
```

**The difference between `result[T]` and `maybe T`:**

```swift
# maybe T — value might not exist (not an error)
target :: maybe Enemy       # no target is normal — not an error

# result[T] — operation might fail (an error occurred)
load_level() -> result[LevelData]   # failure needs a response
```

```
maybe T:
  → Some:    value is present
  → Nothing: value is absent — this is normal
  → No error information carried

result[T]:
  → Succeed: operation worked, here is the value
  → Fail:    operation failed, here is why
  → Error information always carried on failure
```

### 🟢 Foundation Example

```swift
# A simple validation function
validate_player_name(name: string) -> result[string]
    if name.length == 0 >>
        fail ValidationError.empty_name
    << name.length == 0
    if name.length > 24 >>
        fail ValidationError.name_too_long with max: 24, actual: name.length
    << name.length > 24
    succeed name
<< validate_player_name
```

### ⚙️ Technical Version

`result[T]` is a discriminated union implemented as a tagged struct:

```
Offset 0: uint8_t tag        — 0x00 = fail, 0x01 = succeed
Offset 1: [padding]          — alignment to T's requirement
Offset N: [T or ErrorUnion]  — value or error, never both
```

The success and failure values occupy the same memory region (union semantics). Size of `result[T]` is `max(sizeof(T), sizeof(ErrorUnion)) + alignment_padding`. For small types this is typically 8-16 bytes. ARC does not apply to `result[T]` itself — it applies to any ARC-managed types contained within. The compiler generates branch-on-tag code at each handling site. LLVM's branch predictor optimises for the success case being taken more frequently.

---

## 6.3 — Creating Results: `succeed` and `fail`

### 👨‍💻 Developer Version

Two keywords create results:

```swift
succeed value    # operation worked — here is the value
fail ErrorType   # operation failed — here is why
```

**`succeed` — returning a successful result:**

```swift
parse_health(raw: string) -> result[int]
    value :: maybe int = raw.as(maybe int)

    if value exists >>
        if value < 0 >>
            fail ParseError.negative_value
        << value < 0
        if value > 9999 >>
            fail ParseError.value_too_large with max: 9999
        << value > 9999
        succeed value    # ← success — return the parsed int
    << value exists
    else >>
        fail ParseError.not_a_number with raw: raw
    << else
<< parse_health
```

**`fail` — returning a failed result:**

```swift
# Simple fail — kind only
fail FileError.not_found

# Fail with inline context
fail FileError.not_found with path: attempted_path

# Fail with a data error (rich context)
fail NetworkError >>
    code    :: 503
    message :: "Service temporarily unavailable"
    url     :: request_url
    retry_after :: 5.0
<< NetworkError
```

### 🟡 Applied Example — Weapon Validation System

```swift
entity WeaponValidator >>

    validate_weapon_config(config: WeaponConfig) -> result[WeaponConfig]
        # Chain multiple validations — each can fail
        if config.damage < 0 >>
            fail WeaponError.invalid_damage with
                value:   config.damage,
                message: "Damage cannot be negative"
        << config.damage < 0

        if config.fire_rate <= 0.0 >>
            fail WeaponError.invalid_fire_rate with
                value:   config.fire_rate,
                message: "Fire rate must be positive"
        << config.fire_rate <= 0.0

        if config.magazine_size < 1 >>
            fail WeaponError.invalid_magazine with
                value:   config.magazine_size,
                message: "Magazine must hold at least 1 round"
        << config.magazine_size < 1

        if config.damage > 9999 and not config.is_boss_weapon >>
            fail WeaponError.balance_violation with
                value:   config.damage,
                message: "Non-boss weapons cannot exceed 9999 damage"
        << config.damage > 9999 and not config.is_boss_weapon

        # All checks passed
        succeed config
    << validate_weapon_config

<< WeaponValidator
```

### ⚙️ Technical Version

`succeed value` compiles to: set tag byte to 0x01, copy `value` into the union storage. `fail ErrorType` compiles to: set tag byte to 0x00, copy `ErrorType` into the union storage. Both are single store instructions followed optionally by a memcpy for non-trivial types. No heap allocation occurs. No exception table is generated. `fail with context` compiles to an inline struct construction in the union storage — context is stack-allocated and copied, never heap-allocated. This is zero-overhead on the success path: a program with no failures at runtime executes identically to a program without result types.

---

## 6.4 — Error Types Using `kind` and `data`

### 👨‍💻 Developer Version

Skev error types use the same `kind` and `data` keywords you already know. There is no separate `error` keyword — errors are just values.

> **Errors are values. Values have types. Types use `kind` and `data`.**

**Simple errors — `kind` only:**
Use when the category of error is all the caller needs to know:

```swift
kind FileError >>
    not_found
    permission_denied
    corrupted
    too_large
    disk_full
<< FileError

kind NetworkError >>
    timeout
    connection_refused
    invalid_response
    rate_limited
    server_error
<< NetworkError

kind ParseError >>
    invalid_format
    missing_field
    type_mismatch
    version_mismatch
    unsupported_feature
<< ParseError
```

**Rich errors — `data` with context:**
Use when the caller needs details to respond meaningfully:

```swift
data SaveError >>
    kind       :: SaveErrorKind
    slot       :: int
    path       :: string
    message    :: string
    recoverable :: bool = true
<< SaveError

kind SaveErrorKind >>
    file_corrupted
    version_incompatible
    insufficient_space
    serialisation_failed
<< SaveErrorKind

data ValidationError >>
    field    :: string
    message  :: string
    value    :: string
    expected :: string
<< ValidationError
```

**`fail with` — inline context for kind errors:**
When you need just one piece of context without a full data type:

```swift
# Instead of defining a data type for every error:
fail FileError.not_found with path: attempted_path
fail ParseError.type_mismatch with expected: "int", found: raw_value
fail NetworkError.timeout with duration_ms: elapsed.as(int)
```

**Why `kind` and `data` instead of a special `error` keyword:**

```
Skev already has:
→ kind  for named variants
→ data  for structured values

Errors ARE named variants with optional structure.
Adding a special "error" keyword would:
→ Increase language surface area unnecessarily
→ Create inconsistency — why are errors special?
→ Violate Practice 8 (Minimal Surface Area)

Using kind + data means:
→ Exhaustive match works on errors automatically
→ Same syntax developers already know
→ Adding a variant = compiler reports all affected sites
→ Errors are first-class values — not second-class exceptions
```

### ⚙️ Technical Version

Error types defined with `kind` compile to a uint32 discriminant — identical to any other `kind`. Error types defined with `data` compile to plain LLVM structs — identical to any other `data`. No special error handling infrastructure is generated. The `result[T]` union stores the error in a fixed-size region — if the error type exceeds the region size, it is promoted to scoped heap allocation (same rules as `data` in Chapter 3). `fail with` context compiles to an anonymous struct construction inline at the call site — the compiler generates a temporary struct type for each unique `with` context signature.

---

## 6.5 — The `->` Propagation Operator

### 👨‍💻 Developer Version

The `->` operator is Skev's solution to the most common frustration with result types: verbosity when chaining multiple fallible operations.

Without propagation, loading a level requires deeply nested match blocks. With `->`, it reads as a linear sequence of steps — exactly how you think about the operation:

> `->` means: if this fails, stop and return the failure immediately. If it succeeds, give me the value and continue.

### Rule F — The Problem in Other Languages

**🌍 Real-World Scenario**

A game loading system needs to: find the save file, verify it is not corrupted, parse the data, validate the game version, load the required assets, and finally build the game state. Every one of these steps can fail — and when one fails, the whole operation should fail cleanly without leaking memory or leaving the game in a broken state.

**💥 Why This Is Hard**

This is called "the railway problem." You have a sequence of operations where each one depends on the previous succeeding. If any step fails, you need to:
- Stop immediately
- Return the error to the caller
- Clean up everything that was allocated so far
- Not execute any of the remaining steps

In languages with exceptions this requires try/finally blocks. In languages with result types this creates deeply nested matches. Neither is readable. Neither is safe by default.

---

**In C++23:**
```cpp
// C++23 with std::expected (the modern approach)
std::expected<GameState, std::error_code> LoadGame(const std::string& path) {
    auto fileData = LoadFile(path);
    if (!fileData) return std::unexpected(fileData.error());

    auto validated = ValidateChecksum(*fileData);
    if (!validated) return std::unexpected(validated.error());

    auto parsed = ParseSaveData(*validated);
    if (!parsed) return std::unexpected(parsed.error());

    auto assets = LoadAssets(*parsed);
    if (!assets) return std::unexpected(assets.error());

    return GameState::Build(*parsed, *assets);
}
// Every step requires manual unwrapping with * and error checking
// Developer must remember to check every return value
// Forgetting one check = silent bug = wrong game state loaded
// No compiler enforcement that all errors are handled at call site
```

**In C# 13:**
```csharp
// C# uses exceptions — no result types in standard library
async Task<GameState> LoadGameAsync(string path) {
    try {
        var fileData = await File.ReadAllBytesAsync(path);
        var validated = ValidateChecksum(fileData);
        var parsed = await ParseSaveDataAsync(validated);
        var assets = await LoadAssetsAsync(parsed);
        return GameState.Build(parsed, assets);
    }
    catch (FileNotFoundException ex) {
        // Handle file not found
        throw new GameLoadException("Save file not found", ex);
    }
    catch (ChecksumException ex) {
        // Handle corruption
        throw new GameLoadException("Save corrupted", ex);
    }
    catch (Exception ex) {
        // Catch-all — what does caller do with this?
        throw new GameLoadException("Load failed", ex);
    }
}
// Hidden control flow — exceptions jump past code invisibly
// Caller must know to catch GameLoadException
// Easy to forget — no compiler enforcement
// Performance cost of exception infrastructure even when no error
```

**In Python 3.13:**
```python
async def load_game(path: str) -> GameState:
    try:
        file_data = await load_file(path)
        validated = validate_checksum(file_data)
        parsed = await parse_save_data(validated)
        assets = await load_assets(parsed)
        return GameState.build(parsed, assets)
    except FileNotFoundError:
        raise GameLoadError(f"Save file not found: {path}")
    except ChecksumError as e:
        raise GameLoadError(f"Save corrupted: {e}")
    except Exception as e:
        raise GameLoadError(f"Load failed: {e}")
# Same hidden control flow as C#
# No compile-time enforcement
# Exception can propagate past any caller that does not catch it
# Dynamic typing means error types are not verified
```

**In Skev:**
```swift
async load_game(path: string) -> result[GameState]
    file_data :: SaveFileData  = -> await file.load(path)
    validated :: ValidatedSave = -> validate_checksum(file_data)
    parsed    :: ParsedSave    = -> parse_save_data(validated)
    assets    :: AssetBundle   = -> load_assets(parsed)
    succeed GameState.build(parsed, assets)
<< load_game
```

### 🧭 Walk-Through — What Each Part Does

**Line 1 — `async load_game(path: string) -> result[GameState]`**
This function is declared as async (can yield during file loading) and returns a result. The caller MUST handle both success and failure — the compiler enforces this. No exceptions. No surprises.

**Lines 2-5 — The `->` propagation lines**
Each line follows the same pattern: `value = -> operation`. Reading left to right: "try this operation, and if it fails, stop the whole function and return that failure to whoever called me." If it succeeds, put the value in the variable and continue to the next line. No nesting. No if-checks. No manual cleanup — ARC handles all of it.

**Line 6 — `succeed GameState.build(parsed, assets)`**
Only reached if every previous step succeeded. Returns a successful result containing the fully built game state.

**What Skev does automatically that others don't:**
If line 3 (`parse_save_data`) fails — Skev automatically decrements the ARC count of `file_data` and `validated`, freeing their memory. The developer never writes cleanup code. This is guaranteed regardless of which step fails.

---

**In Skev — the caller:**
```swift
when scene_load
    load_game_with_recovery()
<< scene_load

async load_game_with_recovery()
    match await load_game(save_path) >>

        succeed state ->
            scene.restore(state)
            ui.show_message("Game loaded successfully")

        fail FileError.not_found ->
            ui.show_message("No save found — starting new game")
            scene.load_new_game()

        fail SaveError.corrupted ->
            try_backup_recovery()

        fail ParseError.version_mismatch ->
            try_migration_recovery()

        fail error ->
            # Catch-all for unexpected errors
            analytics.report("save_load_failure", error.message)
            ui.show_error("Could not load save — starting new game")
            scene.load_new_game()

    << await load_game(save_path)
<< load_game_with_recovery
```

### 🧭 Walk-Through — The Caller

Every possible failure is handled explicitly. The compiler verifies this — if you add a new error kind to `FileError`, every match on `FileError` across your entire project will generate a compile error until you handle the new case. This is the same exhaustive matching from `kind` — applied to errors automatically.

### ✅ Honest Advantage Statement

Skev's `->` propagation achieves in 5 lines what C++ needs 8+ lines for and what C#/Python need try/catch blocks for. More importantly: Skev's version is the only one where forgetting to handle an error is a compile error, where cleanup is guaranteed automatically by ARC, and where the caller sees exactly which errors can occur in the function's type signature. The trade-off: Skev requires explicit error handling at every call site — there is no "throw it upward and hope someone catches it" escape hatch. This is the intended design.

### ⚙️ Technical Version

`->` propagation compiles to: evaluate the right-hand side, test the tag byte of the result, if tag = fail then store the error into a new `result[T]` with the outer function's error union and execute a return instruction, if tag = succeed then extract the value into the left-hand binding and fall through. The test-and-branch is a single conditional jump instruction. LLVM's backend marks the failure branch as cold — branch predictor learns to skip it. ARC decrements for all bindings in scope fire in reverse declaration order before the return instruction, identical to a normal function return. No separate unwind table or exception frame is generated.

---

## 6.6 — Handling Results

### 👨‍💻 Developer Version

Three patterns for handling a result — choose based on how much control you need:

---

**Pattern 1 — `match` (full control, recommended for complex failures)**

```swift
match load_config("game.skev_config") >>

    succeed config >>
        apply_config(config)
        ui.show_message("Config loaded")
    << succeed config

    fail FileError.not_found >>
        apply_default_config()
        ui.show_message("Using default settings")
    << fail FileError.not_found

    fail FileError.corrupted >>
        delete_config("game.skev_config")
        apply_default_config()
        ui.show_warning("Config corrupted — reset to defaults")
    << fail FileError.corrupted

    fail error >>
        log.error("Config load failed: " + error.message)
        apply_default_config()
    << fail error

<< load_config("game.skev_config")
```

---

**Pattern 2 — `or_else` (fallback, for simple cases)**

```swift
# If loading fails — use the default config instead
config :: Config = load_config("game.skev_config") or_else Config.default()

# If parsing fails — use 0 as the damage value
damage :: int = parse_damage(raw_input) or_else 0

# Can chain or_else
level :: Level = load_level(primary_path)
              or_else load_level(backup_path)
              or_else Level.default()
```

---

**Pattern 3 — `if succeed` / `if fail` (quick checks)**

```swift
# When you only care about one outcome
if load_achievement_data() succeed achievements >>
    ui.show_achievements(achievements)
<< load_achievement_data() succeed achievements

# When you only care about failure
if save_game(slot: 1) fail error >>
    ui.show_error("Save failed: " + error.message)
<< save_game(slot: 1) fail error
```

---

**Compiler enforcement — cannot ignore a result:**

```swift
# This is a compile error in Skev:
load_level("level_2.skev_level")    # ❌ result not handled
```

```
Error: Unhandled result type
  load_level() returns result[LevelData]
  The result must be handled, propagated, or given a fallback.

  Options:
  1. Handle with match:
     match load_level("level_2") >> ...
  2. Propagate with ->:
     data = -> load_level("level_2")
  3. Use fallback with or_else:
     data = load_level("level_2") or_else LevelData.default()

  Line 8 — level_loader.skev
```

### ⚙️ Technical Version

At each result-returning call site, the compiler inserts a liveness check during the type-checking pass. A `result[T]` value that is created but not consumed by a `match`, `->`, `or_else`, or `if succeed/fail` expression is a `UNHANDLED_RESULT` compile error. This check operates on the SSA value graph — a result value that flows into a binding without being consumed before the binding goes out of scope is flagged. The `or_else` expression compiles to a conditional move (CMOV) instruction on architectures that support it — branchless code for the common case.

---

## 6.7 — Panic — Unrecoverable Programming Errors

### 👨‍💻 Developer Version

A panic is different from a result failure. A result failure is something your program expects might happen. A panic is something that should *never* happen — and if it does, it means the code has a bug.

> **result failure** = "the file was not found — let me handle that"
> **panic** = "I tried to access array index 10 but the array only has 4 elements — this is a bug"

**What causes a panic:**

```swift
# Array out of bounds — bug in the code
scores :: array[int, 4] = [10, 20, 30, 40]
bad_access :: int = scores[10]    # PANIC — index 10 out of bounds for array[int, 4]

# Integer division by zero — bug in the code
ratio :: int = total / count      # PANIC — division by zero

# Stack overflow — infinite recursion — bug in the code
```

**Panics cannot be caught.** They are not exceptions. You cannot write a catch block for a panic. If your program panics — you have a bug. Fix the bug. Do not try to recover from it at runtime.

**`assert` — developer-defined invariants:**

```swift
entity CombatSystem >>

    apply_damage(target: Entity, amount: int)
        # Assert things that MUST be true for correct code
        assert amount >= 0 "Damage amount must be non-negative. Got: " + amount.as(string)
        assert target.health > 0 "Cannot damage a dead entity"

        target.health -= amount
    << apply_damage

<< CombatSystem
```

If an `assert` fails — it panics with your message. Use `assert` to document and enforce assumptions in your code. They are your safety net against bugs.

**What a panic looks like:**

In **development builds** — full diagnostic, program halts, debugger can attach:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Skev PANIC — Unrecoverable Error
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Type:     ArrayIndexOutOfBounds
Message:  Index 10 is out of bounds for array[int, 4]
          Valid range: 0 to 3

Location: combat_system.skev  Line 24  Column 12
          bone_weights[bone_index]

Call Stack:
  1.  combat_system.skev:24   update_bone_weights(entity: Dragon)
  2.  animator.skev:156       animate_frame(delta: 0.016)
  3.  game_loop.skev:89       update_all_entities(delta: 0.016)

Entity at panic:
  Dragon "Boss_Dragon_01"
  health:   450 / 1000
  position: (12.3, 0.0, -8.7)
  state:    combat

ARC Cleanup:
  3 entities released cleanly
  0 memory leaks detected
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

In **release builds** — graceful exit, player-friendly message, crash report saved:
```
[Player sees]: "Something went wrong. Your progress has been saved."
[Developer gets]: crash_report_2024_05_02_23_44.skev_crash
[Analytics receives]: automated crash report with sanitised stack trace
```

**The panic hook — preparing for the worst:**

```swift
# Register what happens before graceful exit in release builds
engine.on_panic >> info
    # Try to save player progress
    if save_system.can_emergency_save() >>
        save_system.emergency_save()
        log.write("Emergency save completed before crash")
    << save_system.can_emergency_save()

    # Report to your analytics service
    analytics.report_crash >>
        message:   info.message
        location:  info.location
        build:     build_info.version
    << analytics.report_crash

    # Show player-friendly message
    ui.show_fatal_error(
        "Something went wrong.\n" +
        "Your progress has been saved.\n" +
        "Please restart the game."
    )
<< on_panic
```

### 🔴 Complex Example — Full Error + Panic Separation

**🌍 Real-World Scenario**

A multiplayer RPG server processes thousands of player actions per second. Some actions are invalid input — these should return errors. Some represent actual server bugs — these should panic and generate a crash report. Getting this distinction wrong in either direction is dangerous: treating bugs as handleable errors hides them; treating valid failures as panics crashes the server unnecessarily.

**💥 Why This Is Hard**

Distinguishing "this is a normal failure" from "this is a bug" requires careful design. Too aggressive with panics = server crashes on bad input. Too lenient with results = bugs are silently swallowed. The system must be explicit, compiler-enforced, and readable.

**In C++23:**
```cpp
// C++ mixes exceptions and return codes haphazardly
// Developers decide ad-hoc which approach to use
// No consistency, no compiler enforcement

class ActionProcessor {
    GameState ProcessAction(PlayerId id, const Action& action) {
        // Some functions throw, some return error codes, some return nullopt
        // The developer must know which is which for every function
        try {
            auto player = playerMap.at(id);      // throws std::out_of_range
            if (!action.IsValid()) {
                return GameState::Invalid();      // return code
            }
            auto result = ApplyAction(player, action);
            if (!result.has_value()) {            // optional
                throw ActionException("Apply failed");  // exception
            }
            return *result;
        } catch (std::out_of_range&) {
            return GameState::PlayerNotFound();
        } catch (ActionException& e) {
            LogError(e.what());
            return GameState::Error();
        }
        // Inconsistent. Unmaintainable. No compiler help.
    }
};
```

**In C# 13:**
```csharp
// C# uses exceptions for everything
// No distinction between "expected failure" and "bug"
async Task<GameStateDto> ProcessActionAsync(
    Guid playerId, ActionDto action) {
    try {
        var player = await _playerRepo.GetAsync(playerId)
            ?? throw new PlayerNotFoundException(playerId);

        if (!action.IsValid())
            throw new InvalidActionException(action);

        var result = await _engine.ApplyAsync(player, action);
        return result.ToDto();
    }
    catch (PlayerNotFoundException) {
        return GameStateDto.NotFound();
    }
    catch (InvalidActionException ex) {
        _logger.LogWarning("Invalid action: {Error}", ex.Message);
        return GameStateDto.Invalid(ex.Message);
    }
    // Note: no catch for ArgumentNullException, NullReferenceException etc.
    // Those are BUGS — they should crash, not be caught
    // But C# can't enforce this distinction
}
```

**In Python 3.13:**
```python
async def process_action(player_id: UUID, action: Action) -> GameState:
    try:
        player = await player_repo.get(player_id)
        if player is None:
            return GameState.not_found()
        if not action.is_valid():
            return GameState.invalid(action.validation_error())
        result = await engine.apply(player, action)
        return result
    except Exception as e:
        # Catches EVERYTHING — bugs and expected failures alike
        # How does the caller know if this is recoverable?
        logger.error(f"Action processing failed: {e}")
        return GameState.error(str(e))
        # Bugs are silently swallowed as "errors"
```

**In Skev:**
```swift
kind ActionError >>
    player_not_found
    invalid_action
    action_rate_limited
    insufficient_resources
<< ActionError

entity ActionProcessor >>

    process_action(player_id: uint32, action: GameAction) -> result[GameState]

        # Expected failure — player might not exist
        player :: Player = -> find_player(player_id)

        # Expected failure — action might be invalid
        validated :: GameAction = -> validate_action(player, action)

        # Expected failure — player might be rate limited
        -> check_rate_limit(player)

        # Expected failure — player might lack resources
        -> check_resources(player, validated)

        # Apply the action — succeed or fail
        new_state :: GameState = -> apply_action(player, validated)
        succeed new_state

    << process_action

    apply_action(player: Player, action: GameAction) -> result[GameState]

        # Programming invariant — must always be true
        # If this fires — there is a bug in our validation code
        assert action.is_validated "apply_action called with unvalidated action"
        assert player.is_alive "apply_action called on dead player"

        # The actual game logic
        match action.type >>
            ActionType.move     -> succeed apply_move(player, action)
            ActionType.attack   -> succeed apply_attack(player, action)
            ActionType.use_item -> succeed apply_item_use(player, action)
            _                   -> fail ActionError.invalid_action
        << action.type

    << apply_action

<< ActionProcessor
```

### 🧭 Walk-Through — What Each Part Does

**`process_action` — the chain of expected failures:**
Five `->` lines represent five things that can normally go wrong. Each one automatically returns the appropriate error to the caller if it fails. No try/catch. No manual error propagation. No nesting. The function reads as a linear description of the happy path.

**`apply_action` — where bugs are separated from failures:**
Two `assert` statements document invariants that must always be true in correct code. If they fire, something is broken in our code — not in the player's input. These will panic in development (stop the server, show the bug) and write a crash report in production. The `match` below handles the last remaining expected failure case.

**The critical distinction:**
`->` + `result` = "this might fail legitimately — caller must handle it"
`assert` = "this must always be true — if not, we have a bug"

In C++, C#, and Python — these two categories are often mixed. In Skev — the type system enforces the separation.

### ✅ Honest Advantage Statement

Skev is the only language in this comparison where the distinction between "expected failure" and "programming bug" is enforced at compile time through the type system. C++23's `std::expected` is closest but still allows mixing with exceptions and provides no enforcement at call sites. C# and Python treat everything as exceptions — developers must impose their own discipline. Skev's trade-off: more explicit code at every fallible call site. The gain: a codebase where bugs are structurally separated from expected failures, and where ignoring either is impossible.

### ⚙️ Technical Version

Panics are implemented via LLVM's `unreachable` intrinsic in release builds with a preceding call to the panic handler function. In debug builds, panic sites compile to a call to the Skev runtime's `skev_panic()` function which prints the diagnostic and calls `SIGABRT`. Array bounds checks compile to a compare-and-branch before every array access with a constant index — LLVM eliminates these when the index is provably in bounds at compile time. `assert` compiles to a conditional branch to the panic handler — the branch is marked cold and is eliminated entirely in release builds compiled with `--release-no-asserts` (for maximum performance in shipping builds). The panic hook is a single function pointer stored in the engine runtime's global state.

---

## 6.8 — Errors in Async Functions and Tasks

### 👨‍💻 Developer Version

Async functions and tasks integrate naturally with the result system:

**Async functions that can fail:**

```swift
# Declare the return type — async + result combine cleanly
async load_full_game() -> result[GameState]
    config    :: Config    = -> await file.load_config("game.skev_config")
    save_data :: SaveData  = -> await file.load_save(current_slot)
    assets    :: Assets    = -> await assets.preload(save_data.required_assets)
    succeed GameState.build(config, save_data, assets)
<< load_full_game

# Calling it — same patterns as any result
when scene_load
    handle_load()
<< scene_load

async handle_load()
    match await load_full_game() >>
        succeed state ->
            scene.restore(state)
        fail FileError.not_found ->
            scene.start_new_game()
        fail error ->
            ui.show_error("Load failed: " + error.message)
            scene.start_new_game()
    << await load_full_game()
<< handle_load
```

**Tasks that can fail:**

```swift
task compute_navmesh >>
    raw_mesh :: RawMesh = -> geometry.extract_walkable()
    optimised :: OptMesh = -> navmesh.optimise(raw_mesh)
    task.result = optimised
<< compute_navmesh

await compute_navmesh

# Check task result with same patterns
if compute_navmesh succeed mesh >>
    ai_director.set_navmesh(mesh)
<< compute_navmesh succeed mesh
else if compute_navmesh fail NavError.geometry_too_complex >>
    ai_director.use_simplified_navigation()
<< compute_navmesh fail NavError.geometry_too_complex
```

**ARC cleanup on failure — always automatic:**

```swift
async load_expensive_assets() -> result[AssetBundle]
    # Each of these might be large heap allocations
    textures :: list[Texture]   = -> await textures.load_all()    # 200MB
    meshes   :: list[Mesh]      = -> await meshes.load_all()      # 150MB
    audio    :: list[AudioClip] = -> await audio.load_all()       # 50MB

    # If meshes.load_all() fails:
    # → textures ARC count decremented → 200MB freed automatically
    # → audio never allocated
    # → failure returned to caller
    # Developer writes ZERO cleanup code

    succeed AssetBundle.build(textures, meshes, audio)
<< load_expensive_assets
```

### Rule F — Real-World Example: Robotics

**🌍 Real-World Scenario**

A surgical robot receives commands from a controller. Every command must be validated, and failures must be handled in specific ways depending on their type. A communication timeout means wait and retry. An invalid command means reject and alert the operator. A hardware fault means emergency stop immediately. Getting any of these wrong has serious consequences.

**In Skev — surgical robot command processing:**

```swift
kind CommandError >>
    communication_timeout
    invalid_command
    hardware_fault
    safety_boundary_exceeded
    actuator_not_ready
<< CommandError

data CommandContext >>
    command_id  :: uint64
    timestamp   :: float64
    operator_id :: uint32
<< CommandContext

entity SurgicalRobotController >>

    shared emergency_stop_active :: bool = false

    process_command(cmd: RobotCommand, ctx: CommandContext) -> result[CommandResult]

        # Safety check — always first
        if emergency_stop_active >>
            fail CommandError.hardware_fault with
                command_id: ctx.command_id,
                message: "Emergency stop is active — commands rejected"
        << emergency_stop_active

        # Expected failures — each handled differently by caller
        validated  :: RobotCommand  = -> validate_command_bounds(cmd)
        authorised :: RobotCommand  = -> check_operator_authority(cmd, ctx)
        safe       :: RobotCommand  = -> run_safety_simulation(authorised)
        result     :: CommandResult = -> execute_on_hardware(safe)

        succeed result

    << process_command

    execute_on_hardware(cmd: RobotCommand) -> result[CommandResult]

        # Programming invariants — bugs if these fail
        assert not emergency_stop_active "Hardware execute called during emergency stop"
        assert cmd.is_safety_validated   "Unvalidated command reached hardware layer"

        # Expected hardware failure
        response :: HardwareResponse = -> hardware_bus.send_command(cmd)

        if response.fault_detected >>
            emergency_stop_active = true    # set shared flag — all threads see this
            fail CommandError.hardware_fault with
                fault_code: response.fault_code,
                message: response.fault_description
        << response.fault_detected

        succeed CommandResult.from_response(response)

    << execute_on_hardware

<< SurgicalRobotController
```

### 🧭 Walk-Through — Robotics Example

**The shared emergency stop:** Uses `shared` from Chapter 5 — the main thread, network thread, and sensor thread can all read this flag atomically. Any thread that detects a hardware fault sets it true. Any subsequent command check sees it immediately.

**The assert invariants:** Two lines that document things that must always be true. `assert not emergency_stop_active` catches a bug where the emergency stop check was bypassed. `assert cmd.is_safety_validated` catches a bug where the validation chain was short-circuited. In a surgical context, these bugs are serious — asserting them means they will be caught during development and testing, never in the operating theatre.

**The failure chain:** Five expected failures. Each one uses `->` propagation — if any step fails, execution stops, the error is returned to the caller, and the caller decides the appropriate response (retry timeout, reject invalid, emergency stop on fault). The hardware layer never receives an unsafe command.

### ⚙️ Technical Version

Async functions returning `result[T]` compile their coroutine state machine with an additional error field in the coroutine frame. `->` inside an async function compiles to: test the result tag, if fail then store the error into the coroutine frame's error field and transition to the `coro.end` state, if succeed then extract the value and continue. When the coroutine reaches `coro.end` with the error flag set, it resumes the awaiting continuation with a `result[T]` tagged as failure. Task results are stored in the task's result struct — the task completion callback checks the tag and sets either `task.result` or `task.error` on the task handle.

---

## 6.9 — Error Propagation Across Systems

### 👨‍💻 Developer Version

Real games have many systems. Errors in one system often need to surface in another. Skev's result system handles this through type-safe error combination:

```swift
# Each system defines its own errors
kind SaveError  >> corrupted  << SaveError
kind AssetError >> not_found  << AssetError
kind NetError   >> timeout    << NetError

# A function that calls across systems
# Skev infers the error union automatically
async load_multiplayer_session() -> result[Session]
    save    :: SaveData  = -> await save_system.load_current()   # SaveError
    assets  :: Assets    = -> await asset_system.preload()       # AssetError
    session :: Session   = -> await net_system.connect()         # NetError
    succeed Session.build(save, assets, session)
<< load_multiplayer_session

# Caller handles all possible errors
match await load_multiplayer_session() >>
    succeed session ->
        start_multiplayer(session)

    fail SaveError.corrupted ->
        ui.show_error("Save data corrupted")
        offer_new_game()

    fail AssetError.not_found ->
        ui.show_error("Game files missing — verify installation")

    fail NetError.timeout ->
        ui.show_error("Could not connect to server")
        offer_offline_mode()

    fail error ->
        ui.show_error("Could not start session: " + error.message)

<< await load_multiplayer_session()
```

---

## 6.10 — Complete Error Handling Example

**🌍 Real-World Scenario**

An open-world RPG loads a game session: reads the save file, validates game version compatibility, loads all required assets, connects to the multiplayer server if applicable, and restores the full game state. Any step can fail for different reasons. Each failure needs a different response. Memory must be clean if anything fails midway.

```swift
#! game_session_loader.skev
#! Handles the full game session loading pipeline.
#! Covers: file I/O, version migration, assets, networking, state restore.

kind SessionError >>
    save_not_found
    save_corrupted
    version_too_old
    assets_missing
    server_unreachable
    state_invalid
<< SessionError

data MigrationError >>
    from_version :: string
    to_version   :: string
    reason       :: string
<< MigrationError

entity GameSessionLoader >>

    current_version :: string = "2.4.1"
    loading         :: bool   = false

    has NetworkManager
    has AssetManager
    has SaveManager

    #{ Public entry point }#
    when collision(other: LoadTrigger)
        if not loading >>
            loading = true
            ui.show_loading_screen()
            begin_load(other.save_slot)
        << not loading
    << collision: LoadTrigger

    async begin_load(slot: int)
        match await load_full_session(slot) >>

            succeed session >>
                scene.restore(session.state)
                if session.is_multiplayer >>
                    net.join_session(session.net_id)
                << session.is_multiplayer
                ui.hide_loading_screen()
                ui.show_message("Welcome back, " + session.player_name + "!")
                loading = false
            << succeed session

            fail SessionError.save_not_found >>
                ui.hide_loading_screen()
                ui.show_new_game_prompt()
                loading = false
            << fail SessionError.save_not_found

            fail SessionError.save_corrupted >>
                # Try backup before giving up
                match await load_backup_session(slot) >>
                    succeed session ->
                        scene.restore(session.state)
                        ui.hide_loading_screen()
                        ui.show_warning("Restored from backup save")
                        loading = false
                    fail error ->
                        ui.hide_loading_screen()
                        ui.show_error("Save data unrecoverable — starting new game")
                        scene.start_new_game()
                        loading = false
                << await load_backup_session(slot)
            << fail SessionError.save_corrupted

            fail SessionError.version_too_old >>
                # Try migration
                match await migrate_and_load(slot) >>
                    succeed session ->
                        scene.restore(session.state)
                        ui.hide_loading_screen()
                        ui.show_message("Save migrated to version " + current_version)
                        loading = false
                    fail MigrationError error ->
                        ui.hide_loading_screen()
                        ui.show_error(
                            "Cannot migrate save from version " +
                            error.from_version + " to " + error.to_version +
                            "\nReason: " + error.reason
                        )
                        scene.start_new_game()
                        loading = false
                << await migrate_and_load(slot)
            << fail SessionError.version_too_old

            fail SessionError.assets_missing >>
                ui.hide_loading_screen()
                ui.show_verify_installation_prompt()
                loading = false
            << fail SessionError.assets_missing

            fail SessionError.server_unreachable >>
                # Offer offline mode
                match await load_offline_session(slot) >>
                    succeed session ->
                        scene.restore(session.state)
                        ui.hide_loading_screen()
                        ui.show_warning("Playing offline — progress will sync when reconnected")
                        loading = false
                    fail error ->
                        ui.hide_loading_screen()
                        ui.show_error("Could not connect — check your internet connection")
                        loading = false
                << await load_offline_session(slot)
            << fail SessionError.server_unreachable

            fail error >>
                analytics.report("session_load_failure", error.message)
                ui.hide_loading_screen()
                ui.show_error("Unexpected error — please restart the game")
                loading = false
            << fail error

        << await load_full_session(slot)
    << begin_load

    #{ Core loading pipeline }#
    async load_full_session(slot: int) -> result[GameSession]
        save_data :: SaveData     = -> await save_manager.load(slot)
        _         :: nothing      = -> validate_version(save_data.version)
        assets    :: AssetBundle  = -> await asset_manager.preload(save_data.required_assets)
        net_state :: maybe NetState = await try_connect_multiplayer(save_data)
        state     :: GameState    = -> restore_game_state(save_data, assets)
        succeed GameSession.build(state, net_state, save_data.player_name)
    << load_full_session

    validate_version(version: string) -> result[nothing]
        if version == current_version >>
            succeed nothing
        << version == current_version
        if version_is_newer(version) >>
            fail SessionError.version_too_old with
                found:    version,
                required: current_version
        << version_is_newer(version)
        succeed nothing    # older compatible versions are fine
    << validate_version

    async try_connect_multiplayer(save: SaveData) -> maybe NetState
        if not save.was_multiplayer >>
            # Not a multiplayer save — return nothing (not an error)
            result nothing
        << not save.was_multiplayer

        # Try to connect — if it fails, return nothing (handled by caller as offline)
        match await net.connect(save.last_server) >>
            succeed net_state -> result net_state
            fail _            -> result nothing
        << await net.connect(save.last_server)
    << try_connect_multiplayer

<< GameSessionLoader
```

### 🧭 Walk-Through — Complete Example

**`begin_load` — the decision tree:**
Every failure path has a specific response. The structure reads like a specification document. Each `fail X >>` block is a named decision: "if this specific thing went wrong, here is what we do." No silent swallowing. No generic catch-all (except the last `fail error` which logs and gracefully degrades).

**`load_full_session` — the happy path:**
Four `->` lines describe the steps in order. The function reads as a recipe. Any step can fail — if one does, ARC cleans up what was allocated before that step, and the appropriate error surfaces to `begin_load` which decides how to respond.

**`try_connect_multiplayer` — error vs absence:**
This function returns `maybe NetState` not `result[NetState]`. A failed connection is not an error in this context — it means "play offline." The caller (`load_full_session`) treats nothing as "start in offline mode." This demonstrates the `maybe` vs `result` distinction: absence is normal, failure requires a response.

**`validate_version` — result[nothing]:**
A function that either succeeds (version is compatible) or fails (version is too old or unrecognised). Returns `result[nothing]` because success has no meaningful value — only the failure matters. The `_ = ->` discard pattern handles the nothing value cleanly.

---

## 6.11 — Error Handling Quick Reference

```
RESULT TYPE:
  result[T]                   operation that can succeed or fail
  -> await operation()        async fallible operation
  -> sync_operation()         sync fallible operation

CREATING RESULTS:
  succeed value               operation succeeded with value
  succeed nothing             operation succeeded with no value
  fail ErrorKind.variant      operation failed — simple kind
  fail ErrorKind.variant      operation failed — with context
    with key: value
  fail DataError >>           operation failed — rich context
      field :: value
  << DataError

HANDLING RESULTS:
  match operation() >>        full control — all cases handled
      succeed value >> ...
      fail ErrorX >> ...
      fail error   >> ...
  << operation()

  value = op() or_else dflt   use default value on failure
  value = op1() or_else op2() try op1, fall back to op2

  if op() succeed val >>      only care about success
  if op() fail err >>         only care about failure

PROPAGATION:
  value :: T = -> operation()  propagate failure, unwrap success
  Only valid inside result-returning functions
  NOT valid directly in when blocks

PANICS:
  assert condition "message"  programming invariant — panics if false
  engine.on_panic >> info     release build graceful exit hook
      ...
  << on_panic

ERROR TYPES:
  kind ErrorName >>           simple error categories
      variant_one
      variant_two
  << ErrorName

  data ErrorName >>           error with context
      field :: Type
      message :: string
  << ErrorName

  fail ErrorKind.variant      attach context inline
      with key: value

KEY RULES:
  → Unhandled result = compile error
  → result[T] is zero-cost on success path
  → ARC cleans up automatically on -> propagation
  → Panics cannot be caught — fix the code
  → kind errors = exhaustive match enforced
  → -> not valid in when blocks directly
```

---

*End of Chapter 6 v0.1 — Next: Chapter 7: Standard Library*
