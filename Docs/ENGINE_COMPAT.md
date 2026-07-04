# Monolith — UE Engine Version Compatibility & Porting Notes

**Plugin:** Monolith | **Maintained fork:** [JCSopko/monolith](https://github.com/JCSopko/monolith) | **Verified engine versions:** 5.7.1, 5.8

---

## Purpose

This is the porting log for Monolith — specifically the JCSopko fork — across Unreal Engine major/minor versions. Each engine bump tends to break a handful of call sites in a *pattern*: one root API/ABI change that fans out across every module touching it, plus a couple of unrelated, unlucky API breaks that just happen to surface in the same build pass. This doc exists so the next port doesn't have to rediscover the pattern, the rejected alternatives, or the verification bar from scratch.

Monolith is verified on **UE5.8** as of commit `71648e1` (branch `feat/ue5.8-fjsonobject-port`) — all 18 modules compile, link, and load, confirmed by a real headless-editor MCP round-trip (see [Verification](#verification) below). 5.7.1 support is unaffected; the fix pattern used is additive (a per-site type wrap), not a replacement of 5.7.1-era code paths.

## How this document works

New sections are **appended below**, one per engine version ported — this file reads oldest-to-newest (unlike `CHANGELOG.md`'s newest-first convention), so it doubles as a chronological history of what each engine bump cost. When Monolith is ported to UE5.9 or later:

1. Add a new `## UE5.<minor>` heading below the most recent section — never overwrite or reorganize a prior section.
2. Name the root cause: what the engine changed (API, ABI, behavior), and the *shape* of the breakage (how many sites, how many modules, one pattern or several).
3. Name the fix chosen and why — especially if an engine-provided escape hatch (a legacy-compat macro, a deprecated-but-still-working path) was considered and rejected.
4. Name how it was verified — a real work-product check (a round-trip through the actual code path), not "it compiled" or a clean exit code alone.
5. Note any unrelated API breaks found along the way — easy to lose track of later if they're only mentioned in a commit message.

---

## UE5.8

### The `FJsonObject::Values` key-type rework

UE5.8 changed `FJsonObject`'s internal `Values` map from an `FString`-keyed map to a `UE::FSharedString`-keyed one. `FSharedString` compares against `FStringView` but is not implicitly convertible to/from `FString`, so every module that iterated a `Values` map expecting an `FString` key stopped compiling.

The fix is a minimal, local wrap at the point of use — construct an `FString` from the `FSharedString` key:

```cpp
// Illustrative — the idiom applied at ~50 sites, not a literal diff.

// 5.7.1 — Pair.Key is FString
for (const auto& Pair : JsonObject->Values)
{
    FString FieldName = Pair.Key;
    ProcessField(FieldName, Pair.Value);
}

// 5.8 — Pair.Key is UE::FSharedString; wrap at the point of use
for (const auto& Pair : JsonObject->Values)
{
    FString FieldName = FString(Pair.Key);
    ProcessField(FieldName, Pair.Value);
}
```

This landed across all 18 modules — roughly 50 call sites in 26 files.

### Reverse-wrap cases and the `GetKeys()` / `operator[]` rewrites

Not every site needed the same direction of wrap:

- A couple of sites had the *reverse* shape: an `FSharedString`-typed loop variable needed to key into an ordinary, unrelated `TMap<FString, ...>` (adjacency/room-tag maps in `MonolithMeshSpatialRegistry.cpp`). These wrap the other way — `FString(Key)` — to match the *target* map's key type, not the source.
- One site (`MonolithMeshBuildingValidationActions.cpp`) needed the reverse wrap for the opposite reason: it calls `.FindChecked()` directly on a `Values` map from an `FString`-typed key, so it constructs `UE::FSharedString(Key)` to match `Values`' actual key type.
- Two sites — `MonolithMeshFurnishingActions.cpp` and `MonolithMeshProceduralCache.cpp::SerializeObject` — used `TMap::GetKeys(TArray<FString>&)` or `Values[SomeFString]` directly against a `Values` map. Neither has a heterogeneous-lookup overload for a plain `FString` against an `FSharedString`-keyed map — unlike `FJsonObject`'s own `SetField`/`HasField`, which Epic explicitly gave `FStringView` overloads for this exact transition. Both were rewritten to iterate `Values` directly and carry `FString(Pair.Key)` alongside the value, preserving existing behavior (including alphabetical-sort determinism in the cache serializer).

**Rule of thumb:** the wrap direction is decided by which type the *destination* map or call actually needs — not a single global replace rule. Read each site.

### Why the `UE_JSONOBJECT_LEGACY_STRING_KEYS` escape hatch was rejected

`JsonObject.h` documents a `UE_JSONOBJECT_LEGACY_STRING_KEYS=1` compatibility flag that reverts `FJsonObject` to `FString` keys. It was considered and rejected: the flag is an ABI property of the type, not a per-module define, so it must be set consistently for *every* module that touches `FJsonObject` — flipping it means editing the core `Json` module's `Build.cs`, forcing a full-engine relink and changing behavior for every `FJsonObject` consumer in the tree, not just Monolith. The per-site local wrap has a far smaller blast radius and keeps the fix self-contained to Monolith's own modules.

### The compiler is the oracle, not grep

`Pair.Key` (a loop-variable name used commonly across the codebase) had **230 occurrences across 52 files** in `Source/`. Not all of them were `FJsonObject::Values` iteration — roughly 15-20% were unrelated containers that happened to share the loop-variable name: `Pair.Key.ToString()` on an `FName`-keyed Blackboard map, `Pair.Key.Get()` on a `TWeakObjectPtr`-keyed map, `Pair.Key->NodePosX` on a `UEdGraphNode*`-keyed map. None of those needed touching, and a blind regex sweep across all 230 hits would have introduced real breakage.

Read context to find real candidates, then **let the compiler confirm** — the actual fix sites (~50, across 26 files) are exactly the ones the compiler flags as type errors once the engine header changes. A fatal `#include` error in a file (see the `RigVMAsset.h` removal below) suppresses every later diagnostic in that translation unit, so a single build pass will not surface a file's full defect list — expect more than one compile round before a file reads as clean.

Concretely, this port took four build rounds, each surfacing a genuinely different remainder rather than a shrinking tail of the same errors: round 1 (unmodified baseline) — 61 compile errors, all `Pair.Key`-shaped; round 2 (after the bulk fix) — 2 compile errors, one of them (`FJsonSerializer::Serialize`) only reachable because the `RigVMAsset.h` fix in the same round let the compiler read past its previous fatal error; round 3 — 0 compile errors, but a wholly new failure *class* at link time (the Slate dependency, below) that no earlier round could have shown; round 4 — clean. Don't read an early round's shrinking error count as "almost done" — a clean compile can still hide a link-time surprise.

### Two unrelated API breaks found mid-port

Two 5.8 API breaks surfaced during the compile loop that have nothing to do with the `FJsonObject` rework — a reminder that a version bump's damage isn't always one clean pattern:

- **`RigVMAsset.h` → `RigVMEditorAsset.h`** — 5.8 renamed the header. `MonolithControlRigWriteActions.cpp` updated its `#include` accordingly (a straight rename, no behavior change). This file's fatal include error also hid a second, real error underneath it (an `FJsonSerializer::Serialize` identifier-type mismatch, part of the main port) until the include was fixed and the compiler could read further into the file.
- **`FMeshMergingSettings::bPivotPointAtZero`** — renamed to `bPivotPointAtZero_DEPRECATED` in 5.8. `MonolithMeshQualityActions.cpp` dropped the assignment; the value being set was `false`, which is the type's default, so this is behavior-preserving.

### `MonolithAnimation`'s missing Slate/SlateCore dependency

After the JSON and RigVM fixes, the build reached a new failure class: 7 unresolved `LNK2019` externals in `MonolithAnimation.dll`, all `FSlateAttributeDescriptor` / `SWidget::PrivateRegisterAttributes` symbols, from a locally-instantiated `STableRow<FRetargetChainElement>` widget in the retarget-chain UI. `MonolithAnimation.Build.cs` had never declared direct `Slate`/`SlateCore` dependencies — it relied on a transitive re-export through `UnrealEd` that 5.8 tightened, so the symbols stopped resolving. Fix: add `Slate` and `SlateCore` directly to `MonolithAnimation.Build.cs`'s module dependencies.

General lesson for future ports: a module-dependency tightening elsewhere in the engine can surface as a link error in a module that has nothing to do with the port's expected pattern. Worth a bounded, targeted investigation — not a blind patch-by-analogy, and not an immediate escalation — when a genuinely different error class appears mid-port.

### Verification

Compiling and linking clean is necessary but not sufficient. The port was verified by launching the editor headless (`-nullrhi -unattended`) against a scratch project, polling Monolith's MCP HTTP server on `:9316`, and round-tripping a real `tools/call` request (`editor_query` / `run_python`) with a unique marker through the actual ported JSON pipeline — not just checking the build's exit code. Pass criteria: `tools_registered: 1150`, the marker echoed correctly, all 18 modules loaded with 0 missing/failed entries in the editor log.

This exact e2e shape (launch headless, poll the MCP port, round-trip a real tool call, tear down) already existed as a Linux verification script from an earlier port; the UE5.8 pass was a straight platform port of that pattern (PowerShell process/health-poll in place of `nohup`/`curl`/`kill`) rather than a new verification design. Worth checking for an existing e2e harness to port before inventing a fresh one for the next engine version too.

**Landed:** commit `71648e1`, branch `feat/ue5.8-fjsonobject-port`, pushed to `JCSopko/monolith`.
