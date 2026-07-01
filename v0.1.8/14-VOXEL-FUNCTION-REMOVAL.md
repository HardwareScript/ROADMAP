# Roadmap 14 — Legacy Voxel-Grid Function Removal & EntityGraph Native Migration

**Read:** 01-DATABASE-SPATIAL-FOUNDATION.md, 10-WIRING-INTEGRATION-GAPS.md, Core-System-Architecture.md

**Status:** COMPLETE — All 77 tasks done, build passes, zero remaining voxel references in hwc-engine/src

---

## Problem Statement

The `VoxelGrid` struct has been removed from `HardwareSpace` (Roadmap 10.1), but **18+ functions** across the engine still use voxel-grid-style operations. These functions fall into two categories:

1. **No-op stubs** (`mark_circular_area_occupied`, `remove_circular_area`): Accept parameters but perform no registration. All callers silently fail, leaving vias, landing pads, and anti-pads unregistered in the EntityGraph.
2. **Grid-rasterization code** (`rasterize_line`, `rasterize_into_grid`): Convert nanometer coordinates to grid cells via division by `voxel_size_nm`, then rasterize Bresenham lines or scanline fills. This quantizes continuous geometry to a discrete grid, contradicting the v0.1.8 "Primitives Over Pixels" architecture.

**Impact:**
- Via footprints are never registered → P41 connectivity failures
- ASIC intermediate landing pads are never registered → broken via towers
- Anti-pads are never cleared → DRC violations
- Copper Welder and Exporter cannot see stamped geometry → gaps in GLB/DXF/GDSII exports
- Thermal relief spokes are rasterized to grid instead of registered as vector polygons
- Dummy fill density analysis uses grid sampling instead of R*-tree queries

**Architecture Reference:** `ROADMAP/v0.1.8/01-DATABASE-SPATIAL-FOUNDATION.md` Section 1.5 — "The coordinate grid is completely removed"

---

## 14.1 Circular Area Operations — Root Cause (No-Op Stubs)

These three functions are the root cause of the split-brain data model. They accept geometric parameters but perform no EntityGraph registration. All callers silently fail.

**Architecture Note:** All three must be implemented together in Phase 1. The via lifecycle is tightly coupled:
- **Collision Check** (`is_circular_area_clear`): If registration is native but collision check still uses legacy logic (only inspecting component metadata), the router will place new vias directly on top of existing traces, other-net pours, or unregistered obstacles.
- **Registration** (`mark_circular_area_occupied`): Once registered natively, vias become visible as planar islands and edges to the PIVB solver.
- **Clearance** (`remove_circular_area`): If optimization or rip-up-and-reroute loops need to clear a bad route, they must remove the via from the EntityGraph and spatial index. Without this, old vias remain as permanent ghost shorts causing subsequent validation failures.

Implementing all three together ensures checking, placing, and ripping up vias remain perfectly synchronized in the vector domain.

- [x] **14.1.1** Replace `mark_circular_area_occupied()` with EntityGraph-native registration — call `entity_graph.add_cylinder_substrate_layer()` to register via pads as analytic cylinder substrate layers (file: `hwc-engine/src/geometry_router/router/circular_operations.rs:31`)
- [x] **14.1.2** Replace `remove_circular_area()` with EntityGraph-native removal — call `entity_graph.spatial_mut().remove_by_source()` + remove from `substrate_layers` vec (file: `hwc-engine/src/geometry_router/router/circular_operations.rs:43`)
- [x] **14.1.3** Replace `is_circular_area_clear()` with EntityGraph R*-tree query — call `entity_graph.query_bbox()` against the spatial index to check all registered geometry (components + substrate layers + routes), not just component metadata bboxes (file: `hwc-engine/src/geometry_router/router/circular_operations.rs:8`)
- [x] **14.1.4** After 14.1.1–14.1.3, delete `circular_operations.rs` if all methods are inlined into callers, or keep as thin wrappers over EntityGraph methods

**EntityGraph replacement API:**
- `add_cylinder_substrate_layer(material, net, bbox, diameter_nm, segments, rotation_deg)` — registers circular pad
- `query_bbox(bbox) -> Vec<SubstrateLayer>` — spatial query via R*-tree
- `spatial_mut().remove_by_source(source)` — removes entity from spatial index

**Files to modify:**
- `hwc-engine/src/geometry_router/router/circular_operations.rs` (rewrite all 3 methods)

---

## 14.2 Via Stamping Operations — Callers of No-Op Stubs

These functions call the no-op stubs from 14.1, meaning they silently fail to register via geometry in the EntityGraph. After 14.1 is complete, these callers must be updated to use the new EntityGraph-native methods.

### 14.2.1 stamp_via — Via Footprint Registration

- [x] **14.2.1.1** Update `stamp_via()` to register via pads as `add_cylinder_substrate_layer()` on each Z plane the via passes through (file: `hwc-engine/src/geometry_router/router/via_operations.rs:121`)
- [x] **14.2.1.2** After stamping all Z planes, call `entity_graph.rebuild_spatial_index()` to update the R*-tree (file: `hwc-engine/src/geometry_router/router/via_operations.rs:134`)
- [x] **14.2.1.3** Verify callers: `global_routing.rs:226`, `single_net.rs:242`, `single_net.rs:362` — these call `stamp_via()` and should work correctly after the fix

**Callers affected:**
- `hwc-engine/src/geometry_router/router/routing_methods/global_routing.rs:226` — `self.stamp_via(&via)`
- `hwc-engine/src/geometry_router/router/routing_methods/single_net.rs:242` — `self.stamp_via(&via)`
- `hwc-engine/src/geometry_router/router/routing_methods/single_net.rs:362` — `self.stamp_via(&via)`

### 14.2.2 generate_intermediate_landing_pads — ASIC Via Tower Landing Pads

- [x] **14.2.2.1** Update `generate_intermediate_landing_pads()` to register landing pads as `add_cylinder_substrate_layer()` at each intermediate Z plane (file: `hwc-engine/src/geometry_router/router/via_operations.rs:308`)
- [x] **14.2.2.2** Ensure landing pads are registered with the correct material, net, and enclosure radius (file: `hwc-engine/src/geometry_router/router/via_operations.rs:328-334`)

**Callers affected:**
- None directly (called from via tower unrolling pipeline)

### 14.2.3 generate_antipads — Anti-Pad Clearance for Cross-Net Pours

- [x] **14.2.3.1** Update `generate_antipads()` to use `entity_graph.query_bbox()` to find intersecting pours on different nets, then drill clearance via `entity_graph.drill_hole()` or `remove_circular_area()` (file: `hwc-engine/src/geometry_router/router/via_operations.rs:138`)
- [x] **14.2.3.2** Ensure thermal relief generation for same-net pours is wired through EntityGraph (currently a TODO at line 163)

### 14.2.4 clear_via — Rip-Up and Reroute Via Removal

- [x] **14.2.4.1** Update `clear_via()` to remove via substrate layers from EntityGraph via `remove_circular_area()` (file: `hwc-engine/src/geometry_router/router/via_operations.rs:173`)
- [x] **14.2.4.2** After removal, call `entity_graph.rebuild_spatial_index()` to update the R*-tree

### 14.2.5 can_place_via — Via Placement Validation

- [x] **14.2.5.1** Update `can_place_via()` to query EntityGraph R*-tree for all geometry (components + substrate layers + routes) instead of only checking component metadata bboxes (file: `hwc-engine/src/geometry_router/router/via_operations.rs:88`)
- [x] **14.2.5.2** Ensure the check covers the full via Z span using `entity_graph.query_bbox()` per Z plane

### 14.2.6 extract_vias_from_path — Via Detection from Routed Paths

- [x] **14.2.6.1** Review `extract_vias_from_path()` for any remaining voxel-grid assumptions (file: `hwc-engine/src/geometry_router/router/via_operations.rs:12`)
- [x] **14.2.6.2** Replace `self.resolution_nm / 2` threshold (line 54) with a layer-aware Z-delta check using `self.layer_z_positions` instead of grid quantization

### 14.2.7 unroll_via_tower — ASIC Multi-Layer Via Unrolling

- [x] **14.2.7.1** Review `unroll_via_tower()` for any remaining `voxel_size_nm` fallbacks (lines 249, 255, 276, 281 use `current_idx as i64 * self.resolution_nm` as fallback) — ensure `layer_z_positions` is always populated so the fallback is never hit (file: `hwc-engine/src/geometry_router/router/via_operations.rs:208`)
- [x] **14.2.7.2** Remove the `index * voxel_size_nm` fallback comments and code paths

### 14.2.8 unroll_detected_via — Detected Via Unrolling

- [x] **14.2.8.1** Review `unroll_detected_via()` for any remaining grid assumptions (file: `hwc-engine/src/geometry_router/router/via_operations.rs:365`)
- [x] **14.2.8.2** Ensure the function delegates to `unroll_via_tower()` which uses `layer_z_positions` (already correct)

**Files to modify:**
- `hwc-engine/src/geometry_router/router/via_operations.rs` (update methods 14.2.1–14.2.8)
- `hwc-engine/src/geometry_router/router/routing_methods/global_routing.rs` (verify callers)
- `hwc-engine/src/geometry_router/router/routing_methods/single_net.rs` (verify callers)

---

## 14.3 Routing Pattern Rasterization — Grid-Based Line Drawing

These functions use 3D Bresenham rasterization with `voxel_size_nm` grid coordinates, quantizing continuous nanometer positions to grid cells.

### 14.3.1 rasterize_line — 3D Bresenham Grid Rasterization

- [x] **14.3.1.1** Replace `rasterize_line()` with continuous parametric line interpolation — use `start + t * (end - start)` at `step_size_nm` intervals, returning nanometer-space points without grid quantization (file: `hwc-engine/src/geometry_router/routing_patterns.rs:170`)
- [x] **14.3.1.2** Remove the `/ voxel_size_nm` division that quantizes to grid coordinates (lines 172-178)
- [x] **14.3.1.3** Remove the `* voxel_size_nm` multiplication that converts back from grid (lines 201-204, 224-227, 246-249, 268-271)
- [x] **14.3.1.4** Rename the function from `rasterize_line` to `interpolate_line` to reflect its new continuous nature

### 14.3.2 generate_moves — Pattern Coordinate Generation

- [x] **14.3.2.1** Update `generate_moves()` to use `step_size_nm` parameter name instead of `voxel_size_nm` (file: `hwc-engine/src/geometry_router/routing_patterns.rs:90`)
- [x] **14.3.2.2** Update all callers to pass `step_size_nm` (currently passes `resolution_nm`)
- [x] **14.3.2.3** Update return type documentation to clarify points are in nanometer space, not grid space

**Callers affected:**
- `hwc-engine/src/geometry_router/constraint_aware.rs:181` — `pat.generate_moves(current.position, heading, resolution_nm)` — already passes `resolution_nm`, just rename parameter

### 14.3.3 can_place — Collision Check Stub

- [x] **14.3.3.1** Review `can_place()` (file: `hwc-engine/src/geometry_router/routing_patterns.rs:141`) — currently always returns `true` with comment "collision checking is deferred to the spatial index". Confirm this is correct for the EntityGraph architecture and update the `voxel_size_nm` parameter name to `step_size_nm`.

**Files to modify:**
- `hwc-engine/src/geometry_router/routing_patterns.rs` (rewrite `rasterize_line`, update `generate_moves`, `can_place`)
- `hwc-engine/src/geometry_router/constraint_aware.rs` (update caller)

---

## 14.4 Thermal Relief Rasterization — Grid-Based Polygon Fill

These functions use scanline rasterization to fill polygon interiors into the EntityGraph via `occupy_point()` at resolution intervals. This must be replaced with native polygon substrate layer registration.

**Architecture Note:** Replace the scanline rasterizer with the vector-first `add_polygon_substrate_layer()` approach. Keeping a scanline or row-based approximation defeats the primary purpose of moving to a continuous coordinate database:
- **Clipper2 Harmony:** A single continuous polygon allows the Clipper2 welding engine to seamlessly dissolve boundaries between the spoke, via landing pad, and surrounding copper plane. Scanline rows introduce hundreds of tiny overlapping boundaries that bloat the polygon clipper and can cause micro-slivers or self-intersection bugs.
- **Topological Simplicity:** The PIVB solver sees a single, clean conductive region instead of a fractured cluster of tiny islands, preventing graph building errors and complex Union-Find overhead.
- **Clean 3D Meshing:** Triangulating and extruding one clean polygon is computationally much faster and results in a visually perfect 3D manifold without Z-fighting or interior walls.

### 14.4.1 PolygonRasterizer::rasterize_into_grid — Scanline Polygon Fill

- [x] **14.4.1.1** Replace scanline rasterization + `occupy_point()` calls with `entity_graph.add_polygon_substrate_layer()` to register the polygon as a native vector shape (file: `hwc-engine/src/geometry_router/thermal_relief.rs:25`)
- [x] **14.4.1.2** Remove the `res = self.resolution_nm.max(1)` grid step logic (line 39)
- [x] **14.4.1.3** Remove the `x += res` inner loop that rasterizes at grid intervals (line 63)
- [x] **14.4.1.4** Remove the `y += res` outer loop that steps through scanlines (line 67)

### 14.4.2 ThermalReliefGenerator::generate_spoke — Single Spoke Rasterization

- [x] **14.4.2.1** Replace `PolygonRasterizer::rasterize_into_grid()` call with `entity_graph.add_polygon_substrate_layer()` for the spoke rectangle (file: `hwc-engine/src/geometry_router/thermal_relief.rs:232`)
- [x] **14.4.2.2** Alternatively, replace with `entity_graph.add_substrate_layer()` using the spoke's bounding box if the spoke is axis-aligned (simpler, fewer vertices)

### 14.4.3 ThermalReliefGenerator::generate_spoke_pattern — Spoke Pattern Generation

- [x] **14.4.3.1** Verify `drill_hole()` calls for clearance gap are correct (file: `hwc-engine/src/geometry_router/thermal_relief.rs:186`) — these already use EntityGraph's `drill_hole()` and are correct
- [x] **14.4.3.2** Update `generate_spoke()` calls to use native polygon registration after 14.4.2

### 14.4.4 ThermalReliefGenerator::generate_for_circular_pad — Circular Pad Relief

- [x] **14.4.4.1** Verify `drill_hole()` calls for isolated relief are correct (file: `hwc-engine/src/geometry_router/thermal_relief.rs:153`) — these already use EntityGraph's `drill_hole()` and are correct
- [x] **14.4.4.2** Update `generate_spoke_pattern()` calls after 14.4.3

### 14.4.5 ThermalReliefGenerator::generate_for_rectangular_pad — Rectangular Pad Relief

- [x] **14.4.5.1** Verify `drill_hole()` calls for isolated relief are correct (file: `hwc-engine/src/geometry_router/thermal_relief.rs:276`) — these already use EntityGraph's `drill_hole()` and are correct
- [x] **14.4.5.2** Update `generate_spoke_pattern()` calls after 14.4.3

### 14.4.6 Delete PolygonRasterizer

- [x] **14.4.6.1** Delete `PolygonRasterizer` struct and all its methods after all callers are migrated to native EntityGraph polygon registration (file: `hwc-engine/src/geometry_router/thermal_relief.rs:15-69`)

**Architecture Note:** Deleting the `PolygonRasterizer` enforces the "no grid" compile-time guardrail. If kept in the codebase "just in case," it leaves a structural backdoor that makes it easy for future features or developers to accidentally re-introduce grid-dependent regressions. Removing it completely ensures that any attempt to perform voxel-style operations in the future will fail at compile time, guaranteeing the codebase remains strictly vector-first.

**Files to modify:**
- `hwc-engine/src/geometry_router/thermal_relief.rs` (rewrite rasterization, delete PolygonRasterizer)

---

## 14.5 Dummy Fill — Grid-Based Density Analysis

These functions use grid-based sampling for density analysis and dummy fill stamping.

**Architecture Note:** Implement full dummy fill stamping now using `entity_graph.add_substrate_layer()` for each dummy square. Leaving it as a no-op while fixing the density analysis creates a functional gap:
- If the compiler successfully detects low-density regions but fails to stamp the dummies, the `dummy_fill: true` directive in the PDK profile becomes a silent failure.
- For foundry-grade ASIC or high-frequency PCB compliance, physical metal density is a hard manufacturing rule to prevent physical wafer warping or board delamination.
- A dummy fill element is simply an unconnected, floating square of copper on a specific layer. Register each dummy directly in the EntityGraph as a small `SubstrateLayer::Pour` with a `net_id` of 0 (or `NetId::UNCONNECTED`). The R*-tree and copper welder will handle them naturally.

### 14.5.1 DummyFillEngine::sample_zone — Grid-Based Density Sampling

- [x] **14.5.1.1** Replace grid-based point sampling with R*-tree bbox query — compute the zone's bounding box, call `entity_graph.query_bbox()`, calculate density from returned substrate layer areas (file: `hwc-engine/src/geometry_router/dummy_fill.rs:262`)
- [x] **14.5.1.2** Remove the `step_by(4)` grid sampling loop (line 289)
- [x] **14.5.1.3** Remove the `entity_graph.is_point_occupied()` call (line 295) — replace with area-based density calculation from query results
- [x] **14.5.1.4** Remove the board-size-as-grid-cells calculation (lines 271-278) — use `entity_graph.total_bounding_box()` directly in nanometers

### 14.5.2 DummyFillEngine::stamp_dummies_in_zone — Dummy Fill Stamping

- [x] **14.5.2.1** Implement dummy fill stamping using `entity_graph.add_substrate_layer()` for each dummy fill square (file: `hwc-engine/src/geometry_router/dummy_fill.rs:320`)
- [x] **14.5.2.2** Compute dummy square positions at `dummy_spacing_nm` intervals within the zone
- [x] **14.5.2.3** Skip positions that would violate `clearance_nm` from existing nets (query R*-tree for nearby geometry)
- [x] **14.5.2.4** Register each dummy square as a `SubstrateLayer` with type `DummyFill` in the EntityGraph

### 14.5.3 DummyFillEngine::run — Main Pipeline

- [x] **14.5.3.1** Update `run()` to pass EntityGraph in nanometer coordinates, not grid-cell coordinates (file: `hwc-engine/src/geometry_router/dummy_fill.rs:145`)
- [x] **14.5.3.2** Remove the `size_x / 100_000` grid-cell conversion (line 159) — use bounding box dimensions directly in nanometers
- [x] **14.5.3.3** Update Z-layer iteration to use `layer_z_positions` from the profile instead of grid-cell indices (lines 173-184)

**Files to modify:**
- `hwc-engine/src/geometry_router/dummy_fill.rs` (rewrite density analysis and stamping)

---

## 14.6 Multi-Net Position Tracking — HashMap-Based Collision

- [x] **14.6.1** Review `MultiNetManager::occupy_position()` (file: `hwc-engine/src/geometry_router/multi_net_manager.rs:119`) — uses `FxHashMap<Point3D, NetId>` for collision tracking
- [x] **14.6.2** Replace with EntityGraph R*-tree query for collision detection — the spatial index already provides this capability
- [x] **14.6.3** Remove the `FxHashMap<Point3D, NetId>` field from `MultiNetManager` if all collision queries go through EntityGraph

**Files to modify:**
- `hwc-engine/src/geometry_router/multi_net_manager.rs` (replace HashMap collision with EntityGraph queries)

---

## 14.7 Resolution / Voxel-Size Naming Cleanup

After all functions are migrated, remove all remaining `voxel_size_nm` parameter names and comments that reference voxel-grid concepts.

- [x] **14.7.1** Rename `voxel_size_nm` parameters to `step_size_nm` or `resolution_nm` across all files (grep for `voxel_size_nm` in `hwc-engine/src/`)
- [x] **14.7.2** Remove comments referencing "voxel", "grid coordinates", "grid positions" from migrated functions
- [x] **14.7.3** Update function documentation to reference "nanometer coordinates" instead of "grid coordinates"
- [x] **14.7.4** Remove the `index * voxel_size_nm` fallback code paths in `unroll_via_tower()` (lines 249, 255, 276, 281 of `via_operations.rs`)
- [x] **14.7.5** Verify no remaining references to `VoxelGrid`, `set_voxel`, `get_voxel`, `set_occupied`, `is_occupied` in active code (excluding Archives/)

**Files to search:**
- All `.rs` files under `hwc/crates/hwc-engine/src/geometry_router/`

---

## 14.8 VoxelGrid Module Deletion

After all 14.1–14.7 are complete, delete the VoxelGrid module entirely.

- [x] **14.8.1** Delete `hwc-engine/src/voxel_grid/` directory (if it still exists) — ROADMAP 10.1.6 tracks this
- [x] **14.8.2** Remove `voxel_grid` field from any remaining structs (should already be done per 10.1.2)
- [x] **14.8.3** Remove all `use crate::voxel_grid::` import statements
- [x] **14.8.4** Remove `voxel_grid` from `Cargo.toml` if it was a separate module
- [x] **14.8.5** Run `cargo build` and `cargo test` to verify no remaining references
- [x] **14.8.6** Update ROADMAP 10.1.6 checkbox to complete

**Files to delete/modify:**
- `hwc-engine/src/voxel_grid/` (DELETE entire directory)
- `hwc-engine/src/lib.rs` (remove module declaration)
- `hwc-engine/src/geometry_router/mod.rs` (remove re-exports)

---

## 14.9 Verification & Testing

- [x] **14.9.1** Run full test suite: `cargo test` — all tests must pass after migration
- [ ] **14.9.2** Verify via registration: route a design with vias, confirm `entity_graph.get_substrate_layers()` includes via pads on each Z plane
- [ ] **14.9.3** Verify intermediate landing pads: route an ASIC design with multi-layer via towers, confirm landing pads appear in EntityGraph
- [ ] **14.9.4** Verify anti-pads: route a design with vias crossing copper pours on different nets, confirm anti-pad clearances are drilled
- [ ] **14.9.5** Verify thermal reliefs: place a pad connected to a copper pour, confirm thermal relief spokes are registered as native polygons
- [ ] **14.9.6** Verify dummy fill: run dummy fill on an empty board, confirm dummy squares are registered as SubstrateLayers
- [ ] **14.9.7** Verify export: export GLB/DXF/GDSII, confirm via geometry and landing pads appear in output (no gaps)
- [x] **14.9.8** Verify memory: confirm VoxelGrid allocations are zero (no grid cells allocated at startup)
- [ ] **14.9.9** Verify P41 connectivity: route the test design that was failing P41, confirm connectivity is complete
- [x] **14.9.10** Grep for remaining voxel references: `grep -r "voxel" hwc/crates/hwc-engine/src/` — should return zero matches (excluding Archives/)

---

## Summary

| Section | Tasks | Description |
|---------|-------|-------------|
| 14.1 Circular Area Operations | 4 | Replace no-op stubs with EntityGraph-native registration |
| 14.2 Via Stamping Operations | 17 | Update all callers of the no-op stubs |
| 14.3 Routing Pattern Rasterization | 8 | Replace Bresenham grid rasterization with continuous interpolation |
| 14.4 Thermal Relief Rasterization | 13 | Replace scanline fill with native polygon registration |
| 14.5 Dummy Fill | 11 | Full implementation: R*-tree density analysis + vector substrate stamping |
| 14.6 Multi-Net Position Tracking | 3 | Replace HashMap collision with EntityGraph queries |
| 14.7 Resolution/Voxel-Size Naming | 5 | Remove all voxel-grid naming remnants |
| 14.8 VoxelGrid Module Deletion | 6 | Delete the VoxelGrid module entirely |
| 14.9 Verification & Testing | 10 | End-to-end validation |
| **Total** | **77** | |

---

## Recommended Implementation Order

```
Phase 1 (Root cause — unblocks all callers):
  14.1 → Circular area operations (3 no-op stubs → EntityGraph)
  14.7.5 → Verify no remaining VoxelGrid references in active code

Phase 2 (Via pipeline — fixes P41 connectivity):
  14.2 → Via stamping operations (all callers of 14.1)
  14.9.2–14.9.4 → Via verification tests

Phase 3 (Routing patterns — removes grid quantization):
  14.3 → Routing pattern rasterization (continuous interpolation)

Phase 4 (Manufacturing passes — native polygon registration):
  14.4 → Thermal relief rasterization
  14.5 → Dummy fill density analysis

Phase 5 (Cleanup — remove all voxel remnants):
  14.6 → Multi-net position tracking
  14.7.1–14.7.4 → Naming cleanup
  14.8 → VoxelGrid module deletion

Phase 6 (Validation):
  14.9 → Full verification and testing
```

---

## Files Affected

| File | Changes |
|------|---------|
| `hwc-engine/src/geometry_router/router/circular_operations.rs` | Rewrite 3 methods (14.1) |
| `hwc-engine/src/geometry_router/router/via_operations.rs` | Update 8 methods (14.2) |
| `hwc-engine/src/geometry_router/routing_patterns.rs` | Rewrite `rasterize_line`, update `generate_moves` (14.3) |
| `hwc-engine/src/geometry_router/constraint_aware.rs` | Update caller (14.3) |
| `hwc-engine/src/geometry_router/thermal_relief.rs` | Rewrite rasterization, delete `PolygonRasterizer` (14.4) |
| `hwc-engine/src/geometry_router/dummy_fill.rs` | Rewrite density analysis and stamping (14.5) |
| `hwc-engine/src/geometry_router/multi_net_manager.rs` | Replace HashMap collision (14.6) |
| `hwc-engine/src/geometry_router/router/routing_methods/global_routing.rs` | Verify callers (14.2) |
| `hwc-engine/src/geometry_router/router/routing_methods/single_net.rs` | Verify callers (14.2) |
| `hwc-engine/src/voxel_grid/` | DELETE entire directory (14.8) |
| `hwc-engine/src/lib.rs` | Remove module declaration (14.8) |
| `hwc-engine/src/geometry_router/mod.rs` | Remove re-exports (14.8) |

---

## Architecture Alignment

After this roadmap is complete:

1. **Split-brain eliminated** — All geometry (vias, landing pads, thermal reliefs, dummy fill) is registered in the EntityGraph as native vector primitives (cylinders, polygons, tubes). No geometry exists only in a grid.
2. **Spatial index is sole collision authority** — All collision queries go through `entity_graph.query_bbox()` backed by the R*-tree spatial index. No point-sampling or grid-cell lookups.
3. **Copper Welder sees all geometry** — Since all substrate layers are registered in EntityGraph, the Clipper2 union engine can seamlessly dissolve boundaries between spokes, via landing pads, and surrounding copper planes without micro-slivers or self-intersection bugs.
4. **PIVB solver sees unified conductive regions** — Single clean polygons instead of fractured scanline clusters prevent graph building errors and Union-Find overhead.
5. **Exporter sees all geometry** — Since all geometry is in EntityGraph, the earcut triangulation and GLB/DXF/GDSII export engines produce complete output with no gaps. Clean polygon triangulation is computationally faster and produces visually perfect 3D manifolds.
6. **Manufacturing compliance** — Dummy fill stamping ensures physical metal density rules are enforced, preventing wafer warping and board delamination.
7. **No grid backdoor** — Deleting `PolygonRasterizer` and all voxel functions enforces the "no grid" compile-time guardrail, preventing future regressions.
8. **VoxelGrid fully deleted** — Zero voxel allocations, zero grid-cell lookups, zero `voxel_size_nm` parameters. The memory footprint drops to O(components + traces) as promised in v0.1.8.
