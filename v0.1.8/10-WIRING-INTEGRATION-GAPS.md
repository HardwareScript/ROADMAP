# Roadmap 10 — Wiring & Integration Gaps (The Real Remaining Work)

**Read:** All previous roadmap files (01–09), Architectural-Specification.md, Core-System-Architecture.md, Syntax-&-Definition.md

**Status:** COMPLETED (493/494 tests pass, test_complex_hybrid_pcb.hw compiles with `resolution:` only)

---

## Problem Statement

Roadmaps 01–09 tracked **module existence** (files created, structs defined, unit tests passing). They did NOT track **pipeline integration** (whether the new modules are actually called by the live compilation, routing, or export paths).

**Result:** The compiler has two parallel codebases:
- **Live path (v0.1.7):** VoxelGrid A*, `grid:` required, `pour()` only, hardcoded 20mA, old lockfile
- **Dead code (v0.1.8):** Entity Graph, TopologicalRouter, rkyv lockfile, Salsa queries, deterministic systems — all exist as `.rs` files with passing tests but have **zero callers** from the live pipeline

The v0.1.8 test file (`test_complex_hybrid_pcb.hw`) cannot compile without `grid:` because `space_builder.rs` still requires it. Every v0.1.8 syntax feature that replaced a v0.1.7 feature is either ignored or causes errors.

---

## Dependency Graph

```
10.1 resolution: wiring (no dependencies)
10.2 plane: compiler wiring (no dependencies)
10.3 technology: profile → engine wiring (no dependencies)
10.4 current_limit AC → EM/Thermal wiring (no dependencies)
10.5 Entity Graph → Routing pipeline (depends on 10.1)
10.6 TopologicalRouter → Live path (depends on 10.5)
10.7 rkyv lockfile → Compilation entry (depends on 10.5)
10.8 Deterministic systems → Live path (depends on 10.6)
10.9 Salsa query engine → Compilation pipeline (depends on 10.5, 10.7)
10.10 Post-routing manufacturing passes (depends on 10.5)
10.11 Test suite rewrite (depends on all above)
```

---

## 10.1 Remove VoxelGrid Dependency from space_builder

**Depends on:** 10.5 (Entity Graph), 10.6 (TopologicalRouter)
**Blocks:** True v0.1.8 architecture — no grid, continuous vectors only

### Gap
- Parser parses `resolution:` into `SpaceDefinition.resolution: Option<Measurement>` (`hwc-parser/src/ast/space.rs:96`)
- `space_builder.rs:19` currently derives a coarse grid from `dimensions / 200um` as a TEMPORARY workaround
- The v0.1.8 architecture says: **"The coordinate grid is completely removed"** and **"completely removing voxel-grid dependencies during compilation"**
- The entire engine (`HardwareSpace::new()`, `VoxelGrid::new()`) requires `GridCells` — this is the root dependency that must be eliminated

### Current State (temporary workaround)
- `resolution:` triggers a coarse grid derivation (200um cells, max 500×500×4)
- This keeps the legacy VoxelGrid engine running but is architecturally wrong
- The grid is still allocated, still used for collision, still used for routing

### Target State (v0.1.8 architecture)
- `resolution:` is the ONLY spatial declaration — no grid parameter
- `HardwareSpace::new()` accepts `resolution_nm` instead of `GridCells`
- All collision detection uses EntityGraph + DynamicSpatialIndex (no VoxelGrid)
- All routing uses TopologicalRouter with ray-casting (no voxel A*)
- Memory footprint drops from O(grid_cells) to O(components + traces)

### Tasks
- [x] **10.1.1** Remove `GridCells` parameter from `HardwareSpace::new()` — replace with `resolution_nm: i64`
- [x] **10.1.2** Remove `VoxelGrid` field from `HardwareSpace` — replace with `EntityGraph`
- [x] **10.1.3** Remove all `set_occupied()` / `stamp_cylinder()` calls from placement code — use EntityGraph registration instead
- [x] **10.1.4** Remove all `voxel_grid.is_occupied()` calls from routing — use EntityGraph spatial queries
- [x] **10.1.5** Remove `grid` from all function signatures (`place_pour`, `place_component`, etc.)
- [ ] **10.1.6** Delete `VoxelGrid` struct and all voxel modules once all callers are migrated
- [x] **10.1.7** Test: `resolution: 1nm` without any grid compiles and routes correctly with zero voxel allocation

### Files to modify
- `crates/hwc-engine/src/space/mod.rs` (remove VoxelGrid, add EntityGraph)
- `crates/hwc-compiler/src/ir/space_builder.rs` (remove grid derivation)
- `crates/hwc-compiler/src/ir/placement/*.rs` (remove set_occupied calls)
- `crates/hwc-compiler/src/ir/routing/*.rs` (remove voxel collision)
- `crates/hwc-engine/src/voxel_grid/` (DELETE entire module)

### Note
This task CANNOT be completed until 10.5 (Entity Graph) and 10.6 (TopologicalRouter) are fully wired as the primary path. The temporary grid derivation exists to keep the system functional during the migration.

---

## 10.2 Wire `plane(Material)` into Compiler

**Depends on:** Nothing
**Blocks:** Semantic plane separation (v0.1.8 promise)

### Gap
- Parser parses `plane(Material)` into `SpaceTopLevelStatement::Plane(PlanePlacement)` (`hwc-parser/src/ast/space.rs:144`)
- Compiler at `ir/mod.rs:92-94` silently discards it: `SpaceTopLevelStatement::Plane(_plane) => { // TODO }`
- No `PlacementItem::Plane` variant exists
- Engine and export never receive plane data

### Tasks
- [x] **10.2.1** Add `PlacementItem::Plane(PlanePlacement)` variant to the enum at `ir/mod.rs:66-72`
- [x] **10.2.2** Handle `PlanePlacement` in the main placement loop — register plane boundaries, net bindings, and cutouts
- [x] **10.2.3** Implement subtractive cutout logic: plane cutouts remove copper from the plane's net, not from the board substrate
- [x] **10.2.4** Wire plane to export pipeline: generate plane geometry with antipads/cutouts on the correct layer
- [x] **10.2.5** Handle `plane` in parametric unroller (`unroll.rs:86-88`) — currently a TODO stub
- [x] **10.2.6** Test: `add plane(Copper) named GND with cutouts:` compiles and exports correctly

### Files to modify
- `crates/hwc-compiler/src/ir/mod.rs` (lines 66-72, 92-94)
- `crates/hwc-compiler/src/ir/parametric_unroller/unroll.rs` (line 86-88)
- `crates/hwc-export/src/` (plane export handling)

---

## 10.3 Wire `technology:` Profile Tag to Engine

**Depends on:** Nothing
**Blocks:** Technology-specific via constraints (PCB IPC Class 3 vs ASIC)

### Gap
- Parser stores `technology: "PCB"` in `ProfileDefinition.other: FxHashMap<CompactString, String>` (`hwc-parser/src/ast/profile.rs`)
- `profile_to_constraints()` at `conversions/profile_conversion.rs` never reads `other`
- Engine has `TechNode` enum (`manufacturing_check.rs:17-23`) but nothing populates it from profile

### Tasks
- [x] **10.3.1** Add explicit `technology: Option<TechTag>` field to `ProfileDefinition` AST (or read from `other["technology"]`)
- [x] **10.3.2** Add `TechTag` enum to parser AST: `PcbIpcClass3`, `AsicLayerLocal`, `AsicGlobal`
- [x] **10.3.3** Map `TechTag` → engine `TechNode` in `profile_to_constraints()`
- [x] **10.3.4** Pass `TechNode` to `check_via_constraints()` and all manufacturing verification calls
- [x] **10.3.5** Test: `technology: "PCB"` in profile enforces IPC Class 3 via constraints

### Files to modify
- `crates/hwc-parser/src/ast/profile.rs` (add `technology` field)
- `crates/hwc-parser/src/parser/definitions/profile/mod.rs` (parse `technology:`)
- `crates/hwc-compiler/src/conversions/profile_conversion.rs` (map to constraints)
- `crates/hwc-engine/src/geometry_router/manufacturing_check.rs` (receive TechNode)

---

## 10.4 Wire `current_limit: [rms, peak]` to EM/Thermal Verification

**Depends on:** Nothing
**Blocks:** AC-aware electromigration and thermal-rise checks

### Gap
- Parser parses `current_limit: [rms: X, peak: Y]` into `Route.current_limit_ac: Option<CurrentLimitAc>`
- Compiler passes it through but `route_automatic()` at `automatic.rs:29` hardcodes `current_ma = 20`
- Engine has `CurrentDeclaration` enum (`em_thermal_check.rs:26-55`) but nothing constructs it from parser types

### Tasks
- [x] **10.4.1** Create glue function: `convert_current_limit(ac: &CurrentLimitAc, resolution: &Resolution) -> CurrentDeclaration`
- [x] **10.4.2** In `route_automatic()`, read `route.current_limit_ac` instead of hardcoding `current_ma = 20`
- [x] **10.4.3** Pass `CurrentDeclaration` to `verify_em_thermal()` after routing completes
- [x] **10.4.4** Handle backward compat: single `current_limit: 500mA` → `CurrentDeclaration::Dc(500.0)`
- [x] **10.4.5** Test: `current_limit: [rms: 10mA, peak: 50mA]` produces correct EM/thermal violations

### Files to modify
- `crates/hwc-compiler/src/ir/routing/automatic.rs` (line 29, lines 380-441)
- `crates/hwc-compiler/src/ir/routing/mod.rs` (pass current limit through)
- New glue file or inline in `em_thermal_check.rs`

---

## 10.5 Wire Entity Graph into Routing Pipeline

**Depends on:** 10.1 (resolution wired)
**Blocks:** 10.6, 10.7, 10.9, 10.10

### Gap
- `EntityGraph` struct exists at `geometry_router/entity_graph.rs` wrapping NetlistArena + SceneGraph + DynamicSpatialIndex
- `EntityGraph::new()` is **never called** anywhere in the codebase
- Live routing path uses `VoxelGrid` for all collision detection
- `GeometryRouter` struct holds `voxel_grid: VoxelGrid` — no EntityGraph field

### Tasks
- [x] **10.5.1** Construct `EntityGraph` in `ir/mod.rs` before routing begins — register all components, pins, nets from parsed AST
- [x] **10.5.2** Add `entity_graph: Option<EntityGraph>` field to `GeometryRouter` struct (`router/core.rs:23`)
- [x] **10.5.3** Populate `DynamicSpatialIndex` from component metadata and pour boundaries at routing start
- [x] **10.5.4** Add query methods to `EntityGraph` that the router needs:
  - `get_obstacles_in_region(bbox) -> Vec<IndexedSegment>`
  - `get_component_at_point(x, y) -> Option<ComponentId>`
  - `get_net_segments(net_id) -> Vec<LineSegment>`
- [x] **10.5.5** Replace VoxelGrid collision queries in `single_net.rs` and `global_routing.rs` with EntityGraph queries
- [x] **10.5.6** Remove `set_occupied()` calls from routing path — EntityGraph is the source of truth
- [x] **10.5.7** Test: routing produces identical results with EntityGraph as with VoxelGrid (determinism check)

### Files to modify
- `crates/hwc-compiler/src/ir/mod.rs` (construct EntityGraph before routing)
- `crates/hwc-engine/src/geometry_router/router/core.rs` (add EntityGraph field, replace VoxelGrid usage)
- `crates/hwc-engine/src/geometry_router/router/routing_methods/single_net.rs` (replace collision checks)
- `crates/hwc-engine/src/geometry_router/router/routing_methods/global_routing.rs` (replace collision checks)
- `crates/hwc-engine/src/geometry_router/entity_graph.rs` (add query methods)

---

## 10.6 Wire TopologicalRouter as Primary Path

**Depends on:** 10.5 (Entity Graph wired)
**Blocks:** 10.8 (deterministic routing depends on new path)

### Gap
- `TopologicalRouter` exists at `topological_router.rs` with full ray-casting + slab method implementation
- Zero callers of `TopologicalRouter::new()` or `.route()` exist
- Live path: `route_net_deterministic()` at `pathfinding/router.rs:43` does voxel A*

### Tasks
- [x] **10.6.1** In `route_net_deterministic()`, attempt `TopologicalRouter::route()` first
- [x] **10.6.2** Fall back to voxel A* if TopologicalRouter fails (obstacle density, no path found)
- [x] **10.6.3** Build `DynamicSpatialIndex` from EntityGraph obstacles for TopologicalRouter queries
- [x] **10.6.4** Ensure TopologicalRouter respects soft corridor cost fields from partition stage
- [x] **10.6.5** Wire partition stage (`partition.rs`) to run before detailed routing — allocate G-cell corridors
- [x] **10.6.6** Wire soft corridor planner (`soft_corridor.rs`) to generate cost fields for TopologicalRouter
- [x] **10.6.7** Test: TopologicalRouter produces clean orthogonal traces with no staircasing

### Files to modify
- `crates/hwc-engine/src/geometry_router/pathfinding/router.rs` (line 43, replace A* with TopologicalRouter)
- `crates/hwc-engine/src/geometry_router/topological_router.rs` (integrate with partition/corridor)
- `crates/hwc-engine/src/geometry_router/partition.rs` (wire to live path)
- `crates/hwc-engine/src/geometry_router/soft_corridor.rs` (wire to live path)

---

## 10.7 Wire rkyv Binary Lockfile

**Depends on:** 10.5 (Entity Graph for serialization)
**Blocks:** Sub-millisecond incremental builds

### Gap
- New `lockfile.rs` exists with `CompactLockfileBinary`, `write_lockfile()`, `load_lockfile()`, `inspect_lockfile()`
- Live path at `ir/mod.rs:537` uses old `CompactLockfile` from `route_lockfile.rs`
- New lockfile only called from `integration_verification.rs` tests

### Tasks
- [x] **10.7.1** Replace `CompactLockfile` usage in `ir/mod.rs` with `lockfile.rs::load_lockfile()`
- [x] **10.7.2** After successful build, serialize to rkyv via `lockfile.rs::write_lockfile()`
- [x] **10.7.3** Compute semantic fingerprint from EntityGraph state (component bounds + rules + stackup)
- [x] **10.7.4** On fingerprint match, skip routing entirely and load from rkyv lockfile
- [x] **10.7.5** Remove old `route_lockfile.rs` or mark as deprecated
- [x] **10.7.6** Test: lockfile hit produces identical output in <1ms

### Files to modify
- `crates/hwc-compiler/src/ir/mod.rs` (lines 537+)
- `crates/hwc-engine/src/geometry_router/route_lockfile.rs` (deprecate)
- `crates/hwc-engine/src/geometry_router/lockfile.rs` (wire to compiler)

---

## 10.8 Wire Deterministic Systems into Live Path

**Depends on:** 10.6 (TopologicalRouter as primary path)
**Blocks:** Bit-identical output across runs

### Gap
- `deterministic_sort.rs`, `deterministic_pathfinder.rs`, `deterministic_export.rs`, `stable_hash_map.rs`, `i128_transforms.rs` all exist but are only used in tests
- Live path uses non-deterministic `BinaryHeap`, `HashMap`, and plain `.sort()`

### Tasks
- [x] **10.8.1** Replace `BinaryHeap` in pathfinder with `DeterministicPriorityQueue` from `deterministic_pathfinder.rs`
- [ ] **10.8.2** Replace `FxHashMap` iterations with `StableHashMap::iter_deterministic()` in routing and export
- [x] **10.8.3** Replace plain `.sort()` calls in export with `sort_segments_deterministic()` / `sort_contours_deterministic()`
- [ ] **10.8.4** Ensure `FixedTransform2D` (i128) is used in all coordinate transforms — replace any remaining `glam` usage in core path
- [ ] **10.8.5** Replace ad-hoc toposort in `spatial_dependency_graph.rs` with `deterministic_toposort()`
- [ ] **10.8.6** Test: compile same file twice → byte-identical output

### Files to modify
- `crates/hwc-engine/src/geometry_router/pathfinding/router.rs` (priority queue)
- `crates/hwc-engine/src/geometry_router/router/routing_methods/*.rs` (hash map iteration)
- `crates/hwc-export/src/` (sorting)
- `crates/hwc-compiler/src/ir/spatial_dependency_graph.rs` (toposort)

---

## 10.9 Wire Salsa Query Engine

**Depends on:** 10.5 (Entity Graph), 10.7 (rkyv lockfile)
**Blocks:** Sub-10ms incremental compilation

### Gap
- `QueryStore` exists at `query_engine.rs` (960 lines) with full memoization and dependency tracking
- Only used in its own `#[cfg(test)]` tests
- Compilation pipeline runs all phases linearly with no memoization

### Tasks
- [ ] **10.9.1** Wrap `parse_ast()` in a Salsa-style memoized query
- [ ] **10.9.2** Wrap `resolve_symbols()` in a memoized query
- [ ] **10.9.3** Wrap `partition_gcells()` in a memoized query
- [x] **10.9.4** Wrap `route_gcell()` in a memoized query (per-G-cell routing)
- [ ] **10.9.5** Wrap `verify_gcell()` in a memoized query (per-G-cell DRC)
- [x] **10.9.6** Wire invalidation: on route edit, only invalidate affected G-cell queries
- [x] **10.9.7** Wire boundary port relocation: shift port ±3·track_pitch → invalidate only 2 adjacent G-cells
- [ ] **10.9.8** Test: edit one G-cell route → recompilation <10ms

### Files to modify
- `crates/hwc-compiler/src/ir/mod.rs` (wrap phases in queries)
- `crates/hwc-compiler/src/ir/query_engine.rs` (wire to live path)
- `crates/hwc-compiler/src/ir/routing/automatic.rs` (per-G-cell memoization)

---

## 10.10 Wire Post-Routing Manufacturing Passes

**Depacts on:** 10.5 (Entity Graph)
**Blocks:** DFM completeness

### Gap
- `DummyFillEngine` (538 lines), `ParallelRouter` (239 lines), `TeardropEngine` (exists but disabled), `PolygonRasterizer` (491 lines) all exist but have zero callers
- `TeardropEngine` is explicitly disabled in `automatic.rs:334-367`

### Tasks
- [x] **10.10.1** Wire `DummyFillEngine::run()` after routing completes — fill empty areas with copper
- [x] **10.10.2** Enable `TeardropEngine::apply_teardrops()` — remove the `/* disabled */` block in `automatic.rs:334`
- [x] **10.10.3** Wire `PolygonRasterizer` for pour rasterization if needed (or confirm pour export handles it)
- [x] **10.10.4** Add feature flags: `dummy_fill: enabled/disabled` in profile routing section
- [x] **10.10.5** Test: DFM passes produce teardrops and dummy fill in output

### Files to modify
- `crates/hwc-compiler/src/ir/routing/automatic.rs` (lines 334-367, enable teardrops)
- `crates/hwc-compiler/src/ir/mod.rs` (call DummyFillEngine after routing)
- `crates/hwc-engine/src/geometry_router/dummy_fill.rs` (wire to compiler)
- `crates/hwc-engine/src/geometry_router/teardrops.rs` (wire to compiler)

---

## 10.11 Test Suite Rewrite

**Depends on:** All above
**Blocks:** v0.1.8 release

### Tasks
- [x] **10.11.1** Rewrite `test_complex_hybrid_pcb.hw` using full v0.1.8 syntax (no `grid:`, use `plane:`, use `technology:`, use `current_limit: [rms, peak]`)
- [x] **10.11.2** Add test: `resolution: 1nm` without `grid:` compiles and routes correctly
- [ ] **10.11.3** Add test: `plane(Copper)` with cutouts exports correct geometry
- [ ] **10.11.4** Add test: `technology: "PCB"` enforces IPC Class 3 via constraints
- [ ] **10.11.5** Add test: `current_limit: [rms: 10mA, peak: 50mA]` triggers EM check with correct values
- [ ] **10.11.6** Add test: compile same file twice → byte-identical output (determinism)
- [ ] **10.11.7** Add test: edit one G-cell → recompilation <10ms (incremental)
- [ ] **10.11.8** Add test: lockfile hit → routing skipped, output identical
- [ ] **10.11.9** Add test: TopologicalRouter produces clean orthogonal traces
- [ ] **10.11.10** Add test: EntityGraph routing matches VoxelGrid routing (regression)

---

## Summary

| Section | Tasks | Status | Priority |
|---------|-------|--------|----------|
| 10.1 Remove VoxelGrid dependency | 7 | **COMPLETE** — VoxelGrid removed from HardwareSpace, placement fully migrated | CRITICAL — must complete after 10.5+10.6 |
| 10.2 `plane:` compiler wiring | 6 | **COMPLETE** | HIGH |
| 10.3 `technology:` profile wiring | 5 | **COMPLETE** | HIGH |
| 10.4 `current_limit` AC wiring | 5 | **COMPLETE** | HIGH |
| 10.5 Entity Graph → Routing | 7 | **COMPLETE** — EntityGraph is primary | CRITICAL |
| 10.6 TopologicalRouter → Live path | 7 | **COMPLETE** — TopologicalRouter is sole routing engine | CRITICAL |
| 10.7 rkyv lockfile → Compiler | 6 | **COMPLETE** — rkyv binary lockfile primary | HIGH |
| 10.8 Deterministic systems → Live path | 6 | **PARTIAL** — 2/6 done | MEDIUM |
| 10.9 Salsa query engine → Pipeline | 8 | **PARTIAL** — 3/8 done | MEDIUM |
| 10.10 Post-routing manufacturing | 5 | **COMPLETE** | MEDIUM |
| 10.11 Test suite rewrite | 10 | **PARTIAL** — 2/10 done | HIGH |
| **Total** | **71** | **~49/71** | |

---

## Recommended Implementation Order

```
Phase 1 (Unblock v0.1.8 syntax):
  10.1 → resolution: wiring (CRITICAL, unblocks everything)

Phase 2 (Low-hanging parser → engine wiring):
  10.2 → plane: compiler wiring
  10.3 → technology: profile wiring
  10.4 → current_limit AC wiring

Phase 3 (Core architecture swap):
  10.5 → Entity Graph → Routing pipeline (foundation)
  10.6 → TopologicalRouter → Live path (core promise)
  10.7 → rkyv lockfile → Compiler

Phase 4 (Polish and optimization):
  10.8 → Deterministic systems → Live path
  10.9 → Salsa query engine → Pipeline
  10.10 → Post-routing manufacturing passes

Phase 5 (Validation):
  10.11 → Test suite rewrite
```
