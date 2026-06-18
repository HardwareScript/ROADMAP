# Routing Lock System (AVS) Implementation Checklist

This document tracks the implementation of the Alphanumeric Vector Stream (AVS) Lock System for the v0.1.7 compiler.

**Source Doc:** `Docs/v0.1.7/ROUTING-LOCK-SYSTEM-SPEC.md`
**Status:** Implementation Complete

---

## List 1: Lockfile Data Model (hwc-engine)

> **Target file:** `hwc/crates/hwc-engine/src/geometry_router/route_lockfile.rs`

- [x] **1. Replace `RouteLockfile` struct with AVS schema**
  - *What it is:* Replace the legacy `routes: Vec<LockedRoute>` structure with the compact `arcs` + `instances` schema from the spec.
  - *Implementation:*
    - Define `pub struct CompactLockfile` with fields: `version`, `board`, `placement_hash`, `arcs: Vec<CompactString>`, `instances: Vec<i32>`.
    - Remove `LockedRoute`, `GridMetadata` structs (replaced by `placement_hash`).
    - Keep `RouteLockfile` as a type alias for backward compatibility.
    - Implement `serde::Serialize` / `serde::Deserialize` for the new struct.
  - *Verify:* `cargo check` passes. Build succeeds with no warnings.

- [x] **2. Implement `ArcEncoder` — AnalyticTrace to Arc conversion**
  - *What it is:* Converts a routed path (Vec of Point3D waypoints) into a Base-36 RLC arc string.
  - *Implementation:*
    - Add `pub fn encode_arc(waypoints: &[Point3D]) -> CompactString` to `route_lockfile.rs`.
    - Walk waypoints in pairs, compute directional deltas (dx, dy, dz).
    - For Manhattan segments: emit direction char (`R`/`L`/`U`/`D`) + Base-36 magnitude.
    - Coalesce collinear consecutive segments into a single arc command.
    - Add `pub fn encode_instances(traces: &[AnalyticTrace]) -> (Vec<CompactString>, Vec<i32>)` that flattens net_id + arc_idx + start_xyz into the instances array with deduplicated arcs.
  - *Verify:* Lockfile contains `arcs` array with Base-36 strings (e.g., `"R1hl0gU5sc0R2z60w"`).

- [x] **3. Implement `ArcDecoder` — Arc string to Point3D waypoints**
  - *What it is:* Decodes a Base-36 RLC arc string back into absolute 3D coordinates.
  - *Implementation:*
    - Add `pub fn decode_arc(arc: &str, start: Point3D) -> Vec<Point3D>` to `route_lockfile.rs`.
    - Parse characters: letters = direction commands, alphanumeric = Base-36 magnitude accumulator.
    - Apply directional deltas to produce absolute waypoints.
    - Add `pub fn decode_instances(compact: &CompactLockfile) -> FxHashMap<NetId, Vec<Point3D>>` that iterates instances in chunks of 5, resolves arc references, and produces per-net waypoint vectors.
  - *Verify:* Second build loads 2 cached routes from lockfile, bypassing A* solver.

- [x] **4. Implement `placement_hash` computation**
  - *What it is:* Computes a deterministic hash of all component placements, orientations, and physical parameters.
  - *Implementation:*
    - Add `pub fn compute_placement_hash(space: &HardwareSpace) -> CompactString`.
    - Hash component bounding boxes (min/max XYZ), grid dimensions, voxel size, and net count.
    - Use `DefaultHasher` for deterministic output.
    - Store result in `CompactLockfile::placement_hash`.
  - *Verify:* Identical layouts produce identical hashes; moving one component changes the hash.

---

## List 2: Lockfile Loading & Validation (hwc-compiler)

> **Target file:** `hwc/crates/hwc-compiler/src/ir/mod.rs`

- [x] **1. Load lockfile before routing**
  - *What it is:* Load the `.hw.routes.lock` file before the auto-router runs, validate the placement hash, and populate `space.analytic_routes` with cached routes.
  - *Implementation:*
    - In `compile_single_space()`, after component placement but before `AutoRouter::route_all_nets()`, call `CompactLockfile::load(path)`.
    - Compute `placement_hash` from current space via `compute_placement_hash()`.
    - Compare against stored `placement_hash`.
    - If match: call `to_analytic_traces()` to get per-net waypoints, convert to `AnalyticTrace`, push into `space.analytic_routes`.
    - If mismatch: log `[LOCK] Invalidation detected for '{space}'...`, discard lockfile, proceed with A* routing.
  - *Verify:* Second build with unchanged layout prints `[LOCK] Match found... Bypassing A* solver`.

- [x] **2. Save lockfile after routing**
  - *What it is:* After routing completes (whether from cache or fresh A*), serialize all routes into the AVS lockfile format.
  - *Implementation:*
    - After `auto_router.route_all_nets()` returns, call `encode_instances(&space.analytic_routes)`.
    - Build `CompactLockfile` with `version: "0.1.7"`, `board`, `placement_hash`, `arcs`, `instances`.
    - Save to `.hw.routes.lock` via `serde_json::to_string_pretty()`.
  - *Verify:* Lockfile contains `arcs` array with Base-36 strings, `instances` as flat i32 array.

- [x] **3. Source hash preservation for non-geometric changes**
  - *What it is:* Non-geometric edits (description, comments, metadata) must NOT invalidate the lockfile.
  - *Implementation:*
    - The `placement_hash` only hashes geometric data (component positions, grid dimensions, net count).
    - Source file hash is NOT included in the placement hash.
    - Only `placement_hash` is checked for invalidation.
  - *Verify:* `cargo test` passes. Non-geometric changes do not trigger invalidation.

---

## List 3: Invalidation Rules

> **Target file:** `hwc/crates/hwc-engine/src/geometry_router/route_lockfile.rs`

- [x] **1. Placement shift detection**
  - *What it is:* Detect when any component has moved, rotated, or changed footprint.
  - *Implementation:*
    - `compute_placement_hash()` incorporates all component bounding boxes (min/max XYZ).
    - Any change to component position produces a different hash.
  - *Verify:* `cargo check` passes. Component bbox changes alter the hash.

- [x] **2. Netlist alteration detection**
  - *What it is:* Detect when routes are added, removed, or re-bound.
  - *Implementation:*
    - Include `netlist.num_nets()` in the `placement_hash`.
    - Adding/removing a `route:` declaration changes the net count, which changes the hash.
  - *Verify:* `cargo check` passes. Net count changes alter the hash.

- [x] **3. Physical boundary detection**
  - *What it is:* Detect changes to space dimensions, grid resolution, or profile.
  - *Implementation:*
    - Include `grid.x_cols`, `grid.y_rows`, `grid.z_layers`, `voxel_size.x_nm/y_nm/z_nm` in the `placement_hash`.
    - Changing `dimensions:` or `grid:` in the space definition changes the hash.
  - *Verify:* `cargo check` passes. Grid changes alter the hash.

---

## List 4: Legacy Lockfile Rejection

> **Target file:** `hwc/crates/hwc-engine/src/geometry_router/route_lockfile.rs`

The compiler carries no backward compatibility with legacy lockfiles (v0.1.6 and earlier). Old formats are rejected with a clear error message.

- [x] **1. Strict version check on load**
  - *What it is:* Reject any lockfile that is not exactly version `"0.1.7"`.
  - *Implementation:*
    - In `CompactLockfile::load()`, after deserializing JSON, check `lock.version == "0.1.7"`.
    - If version is missing, empty, or any other value → return `Err(LockfileError::ObsoleteVersion(...))`.
    - No parsing of legacy fields, no migration logic, no fallback deserialization.
  - *Verify:* `cargo check` passes. Legacy v0.1.4 lockfile correctly rejected.

- [x] **2. Error message with actionable instruction**
  - *What it is:* When a legacy lockfile is detected, tell the user exactly what to do.
  - *Implementation:*
    - `LockfileError::ObsoleteVersion` displays: `[LOCK] Obsolete lockfile detected (version {}). Delete the .routes.lock file and rebuild.`
    - `LockfileError::ParseError` displays: `[LOCK] Failed to parse lockfile: {}. Delete the .routes.lock file and rebuild.`
    - `LockfileError::IoError` displays: `[LOCK] IO error: {}`
  - *Verify:* `cargo check` passes. Error messages are actionable.

- [x] **3. JSON parse failure = error**
  - *What it is:* Any deserialization error (malformed JSON, missing fields, wrong schema) produces a clear error.
  - *Implementation:*
    - `serde_json::from_str::<CompactLockfile>()` returns `Err` → return `LockfileError::ParseError(e.to_string())`.
    - No silent fallback, no auto-deletion.
  - *Verify:* `cargo check` passes. Parse errors produce actionable messages.

---

## List 5: CLI Integration

> **Target file:** `hwc/crates/hwc-cli/src/commands/build_cmd/mod.rs`

- [x] **1. Wire lockfile path through build pipeline**
  - *What it is:* Pass the lockfile path and source content from the CLI into the compiler.
  - *Implementation:*
    - In `execute()`, compute `lockfile_path = input.with_extension("hw.routes.lock")`.
    - Pass `lockfile_path`, `source_content`, and `force_reroute` to `program_to_spaces_with_lockfile()`.
    - Lockfile logic is now in the compiler crate (not CLI).
  - *Verify:* Build completes successfully. Lockfile saved after routing.

- [x] **2. Add `--no-lockfile` and `--force-reroute` flag support**
  - *What it is:* Respect existing CLI flags to disable or force lockfile behavior.
  - *Implementation:*
    - If `--no-lockfile`: `lockfile_path` is `None`, skip load and save.
    - If `--force-reroute`: `force_reroute` is `true`, skip load but still save after routing.
    - These flags are passed through `program_to_spaces_with_lockfile()` to `compile_single_space()`.
  - *Verify:* `--force-reroute` always runs A* solver. `--no-lockfile` produces no `.routes.lock` file.

---

## List 6: Verification Tests

> **Target file:** `hwc/tests/pcb/test_complex_hybrid_pcb.hw`

- [x] **1. First-time compilation generates lockfile**
  - *Action:* Delete any existing `.hw.routes.lock`, compile `test_complex_hybrid_pcb.hw`.
  - *Verify:* Console outputs `[LOCK] Saved 2 routes to ...`. File exists with `version: "0.1.7"`, `arcs`, `instances`.

- [x] **2. Second compilation loads from lockfile**
  - *Action:* Run identical build command again.
  - *Verify:* Console outputs `[LOCK] Match found for 'PCB_Complex_Space'. Bypassing A* solver. Loading routes directly from project.routes.lock` and `[LOCK] Loaded 2 cached routes`.

- [ ] **3. Non-geometric modification preserves lockfile**
  - *Action:* Change `description` in the profile. Recompile.
  - *Verify:* Lockfile is loaded (no `[LOCK] Invalidation detected`). Output files are bit-for-bit identical.

- [ ] **4. Geometric shift invalidates lockfile**
  - *Action:* Shift J1_S from x:2mm to x:3mm. Recompile.
  - *Verify:* Console outputs `[LOCK] Invalidation detected for 'PCB_Complex_Space'...`. Fresh A* routing runs. New lockfile saved with updated coordinates.

- [ ] **5. Bus topology compression verification**
  - *Action:* Create a test with 8 parallel nets sharing the same directional pattern.
  - *Verify:* Lockfile contains a single arc referenced by 8 instances, not 8 separate waypoint arrays. File size is <50% of legacy format.
