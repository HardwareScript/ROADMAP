# Advanced Routing Implementation Checklist

This document tracks the implementation of adaptive routing, Steiner net tapping, via tower unrolling, and substrate-aware cost evaluation for the v0.1.7 compiler.

**Source Docs:** `Docs/v0.1.7/Adaptive-Heuristic.md`, `Docs/v0.1.7/Unified-2.5D-3D-Routing-and-Placement.md`

---

## List 1: Adaptive Router Orchestrator

> **Target file:** `hwc/crates/hwc-engine/src/geometry_router/router/core.rs` (extend `GeometryRouter`)

- [x] **1. Scale Detection & Mode Selection**
  - *What it is:* Before running any pathfinding, evaluate net count and total area to choose between Pass-Through (small design) and Hierarchical (SoC-scale) modes.
  - *Implementation:*
    - Add `area_threshold_nm2: i64` (default `1_000_000_000_000` = 1mm²) and `net_count_threshold: usize` (default `100`) fields to `GeometryRouter` or a new `AdaptiveRouter` wrapper struct.
    - Add `pub fn route_space(grid_bbox, nets, obstacle_bboxes) -> Result<RouteResult, RoutingError>` that checks `area < threshold && net_count < threshold` before delegating.
  - *Pass-Through Mode:* Single G-Cell, no global routing, `GeometryRouter` runs once over entire board.
  - *Hierarchical Mode:* Partition into G-Cells, run global router, then parallel detailed routing via `rayon`.

- [x] **2. RouteResult & LocalRoute Types**
  - *What it is:* Unified result container for both routing modes.
  - *Implementation:*
    - Define `pub struct RouteResult { pub paths: FxHashMap<NetId, Vec<Point3D>>, pub vias: Vec<Via> }` in `hwc/crates/hwc-engine/src/geometry_router/types.rs`.
    - Add `pub fn merge(&mut self, other: RouteResult)` method.
    - Reuse existing `Via` struct from `types.rs` (already has `position`, `from_z_nm`, `to_z_nm`, `net_id`, `via_type`).

- [x] **3. Parallel G-Cell Routing (Rayon)**
  - *What it is:* In hierarchical mode, route each G-Cell independently in parallel.
  - *Implementation:*
    - Partition grid using `CoarseGrid` (already exists at `hwc/crates/hwc-engine/src/geometry_router\coarse_grid.rs`).
    - Use `rayon::par_iter()` over cells, each spawning a `GeometryRouter` instance.
    - Stitch results back into a unified `RouteResult`.

---

## List 2: Steiner Net Tapping & Dynamic Target Expansion

> **Target file:** `hwc/crates/hwc-engine/src/geometry_router/router/routing_methods.rs`

- [x] **1. Dynamic Target Set**
  - *What it is:* When routing Pin N of a multi-pin net, all previously routed segments of the same net become valid targets (not just original pins). This enables Steiner Minimum Tree branching via physical T-junctions.
  - *Implementation:*
    - In the routing loop for multi-pin nets, maintain `net_paths: Vec<Vec<Point3D>>` per net.
    - Replace nearest-pin heuristic with `find_nearest_target_on_net(new_pin, &net_paths)` that searches all existing path segments.
    - Terminate A* the moment it intersects any coordinate registered to that net.

- [x] **2. find_nearest_target_on_net Method**
  - *What it is:* Finds the absolute closest point on any currently routed segment of the net.
  - *Implementation:*
    - Add `fn find_nearest_target_on_net(&self, new_pin: Point3D, existing_paths: &[Vec<Point3D>]) -> Point3D` to the router.
    - Iterate over `existing_paths.iter().flatten()`, minimize Euclidean distance squared.
    - Return `new_pin` if `existing_paths` is empty.

- [x] **3. Integration with Existing A* Pathfinder**
  - *What it is:* Wire the dynamic target selection into `route_net_deterministic` or the SDF router.
  - *Implementation:*
    - After each sub-route completes, push the resulting path into `net_paths`.
    - The next sub-route's target is computed via `find_nearest_target_on_net` instead of the next unconnected pin.
  - *Verify:* Multi-pin nets (3+ pins) produce branching T-junctions, not daisy chains.

---

## List 3: Via Tower Unroller (ASIC Multi-Layer Transitions)

> **Target file:** `hwc/crates/hwc-engine/src/geometry_router/router/via_operations.rs`

- [x] **1. Single-Layer Via Unrolling for ASIC**
  - *What it is:* ASIC profiles forbid direct multi-layer via transitions (e.g., M1→M4). A vertical step must be unrolled into single-layer vias (M1→M2, M2→M3, M3→M4) with intermediate landing pads.
  - *Implementation:*
    - Add `pub fn unroll_via_tower(pos, start_layer_idx, end_layer_idx, profile_layers) -> Vec<Via>` to `via_operations.rs`.
    - For Manhattan angle restriction: step one layer at a time, emit a `Via` per adjacent layer pair.
    - For Octilinear (PCB): emit a single through-hole via spanning the full depth.

- [x] **2. StackupManager Layer Query Integration**
  - *What it is:* The unroller needs the ordered layer names from the profile to generate correct `from_layer`/`to_layer` labels on each via.
  - *Implementation:*
    - Use `StackupManager` (`hwc/crates/hwc-compiler/src/ir/stackup_manager.rs`) to get `ordered_layers` and `get_layer_start_z()`.
    - The router calls `unroll_via_tower` when `constraints.angle_restriction == Manhattan` and `dz > 0`.

- [x] **3. Intermediate Landing Pad Generation**
  - *What it is:* Each intermediate layer in the via tower needs a small copper landing pad (annular ring) to anchor the via physically.
  - *Implementation:*
    - Query `profile.via.enclosures[layer_name]` for the enclosure size per layer.
    - Stamp a copper disc of radius `(via_diameter/2 + enclosure)` at each intermediate Z using `VoxelGrid::stamp_cylinder()`.
  - *Verify:* ASIC M1→M4 route produces 3 discrete vias and 2 intermediate landing pads.

---

## List 4: Substrate & Reference-Plane Aware Routing

> **Target file:** `hwc/crates/hwc-engine/src/geometry_router/pathfinding/cost.rs`

- [x] **1. Reference Plane Void Detection**
  - *What it is:* When routing a high-speed signal, crossing a split or void in the ground/power reference plane causes signal reflections. The cost evaluator must detect this and penalize the step.
  - *Implementation:*
    - Extend `MoveCostParams` (already in `cost.rs`) with `substrate_layers: Option<&SubstrateLayerManager>` and `is_high_speed_net: bool`.
    - In `calculate_move_cost()`, after existing cost logic, add: if `is_high_speed_net`, look up the reference layer below the current routing layer via `profile.get_underlying_reference_layer(layer_name)`.
    - Check `substrate_layers.is_void_at(Point3D { x, y, z_ref })`. If void detected, add `cost += 5_000_000` (extreme penalty to force deviation).

- [x] **2. Profile Integration for High-Speed Net Classification**
  - *What it is:* The router needs to know which nets are high-speed to apply the SI penalty selectively.
  - *Implementation:*
    - In the profile parser (`hwc/crates/hwc-parser/src/parser/definitions/profile/constraints.rs`), ensure `net_class` or `frequency` metadata can be attached to nets.
    - Alternatively, check if the net's frequency exceeds a threshold (e.g., ≥1 GHz) from the netlist metadata.

- [x] **3. SubstrateLayerManager Access from Pathfinder**
  - *What it is:* The pathfinder currently operates on the voxel grid; it needs a reference to the substrate layer data.
  - *Implementation:*
    - Pass `&SubstrateLayerManager` through `RoutingParams` (in `hwc/crates/hwc-engine/src/geometry_router/pathfinding/types.rs`).
    - `SubstrateLayerManager` is already stored in `VoxelGrid::substrate_layers` (see `hwc/crates/hwc-engine/src/voxel_grid\grid\core.rs`).
  - *Verify:* A high-speed trace routed over a ground-plane void reroutes around the void instead of crossing it.

---

## List 5: Parser Profile Extensions

> **Target file:** `hwc/crates/hwc-parser/src/parser/definitions/profile/constraints.rs`

- [x] **1. Track Pitch & Grid Snapping Fields**
  - *What it is:* ASIC profiles need `track_pitch` and `grid_snapping` to enforce gridded routing.
  - *Implementation:*
    - Add `pub track_pitch: Option<Expression>`, `pub grid_snapping: Option<bool>` to `ManufacturingConstraints` or a new `RoutingConstraints` struct in the profile AST.
    - Parse in `parse_manufacturing_constraints()`.

- [x] **2. Layer Direction Preferences**
  - *What it is:* Each metal layer can have a preferred routing direction (horizontal/vertical) to maximize density.
  - *Implementation:*
    - Add `pub layer_directions: FxHashMap<String, RoutingDirection>` where `RoutingDirection` is `Horizontal | Vertical | Any`.
    - Parse from `routing:` sub-block in the profile syntax.

- [x] **3. Via Enclosures & Stacking Rules**
  - *What it is:* Per-layer enclosure (annular ring) constraints and stacked-via permissions.
  - *Implementation:*
    - Ensure `parse_via_constraints()` in `hwc/crates/hwc-parser/src/parser/definitions/profile/via.rs` captures `enclosures`, `allow_stacked_vias`, `min_stagger_offset`.
    - These are already partially implemented; verify they surface into the `ViaConstraints` AST.

- [x] **4. Dummy Fill Configuration Fields**
  - *What it is:* Profile-level toggle and parameters for the thieving pass.
  - *Implementation:*
    - Add `pub dummy_fill: Option<bool>`, `pub dummy_fill_density: Option<f64>`, `pub dummy_fill_size: Option<Expression>`, `pub dummy_fill_spacing: Option<Expression>` to `ManufacturingConstraints`.
  - *Verify:* A profile with `dummy_fill: true` enables the thieving pass in `compile_space`.

---

## List 6: Context Propagation (Substrate Layers & Net Frequencies)

> **Target files:**
> - `hwc/crates/hwc-engine/src/geometry_router/router/core.rs` (update `GeometryRouter` signatures)
> - `hwc/crates/hwc-engine/src/geometry_router/pathfinding/cost.rs` (consume context in `calculate_move_cost`)
> - `hwc/crates/hwc-compiler/src/ir/mod.rs` (propagate from `program_to_space`)

- [x] **1. Update `route_space` Signature on GeometryRouter**
  - *What it is:* The orchestrator must accept `substrate_layers` and `net_frequencies` so the pathfinder can query reference-plane voids and classify high-speed nets.
  - *Implementation:*
    - Add `substrate_layers: &SubstrateLayerManager` and `net_frequencies: &FxHashMap<NetId, f64>` parameters to `pub fn route_space(...)`.
    - Forward both into `route_flat()` and `route_detailed_parallel()`.
  - *Verify:* `cargo check` passes with the updated signature.

- [x] **2. Update `route_flat` and `route_local_cell` Signatures**
  - *What it is:* Both routing modes must receive the physical context so the cost evaluator can query substrate layers per step.
  - *Implementation:*
    - Add `substrate_layers: &SubstrateLayerManager` and `net_frequencies: &FxHashMap<NetId, f64>` to `route_flat()` and `route_local_cell()`.
    - In `route_flat`, extract `net_freq = net_frequencies.get(&net_id).copied().unwrap_or(0.0)` per net and pass into the A* call.

- [x] **3. Thread Context into A* Pathfinder**
  - *What it is:* The A* loop's cost evaluator needs `substrate_layers` and `is_high_speed_net` to apply the SI penalty.
  - *Implementation:*
    - Update `run_leap_frog_astar()` to accept `net_id`, `net_freq`, and `substrate_layers`.
    - Inside the loop, call `calculate_move_cost()` with `is_high_speed_net: net_freq >= 1_000_000_000.0` and `substrate_layers: Some(substrate_layers)`.
    - The existing `MoveCostParams` struct in `cost.rs` already has `occupied_voxels` and `clearance_zones`; add `substrate_layers` and `is_high_speed_net` fields.

- [x] **4. Compiler Pipeline Context Extraction**
  - *What it is:* `program_to_space` must extract `net_frequencies` and pass `&space.substrate_layers` into `route_space`.
  - *Implementation:*
    - In `hwc/crates/hwc-compiler/src/ir/mod.rs`, before calling `route_space`, add:
      ```rust
      let net_frequencies = space.netlist.extract_frequencies();
      ```
    - Pass `&space.substrate_layers` and `&net_frequencies` to `route_space()`.
  - *Verify:* The compiler compiles end-to-end with propagated context.

---

## Verification Checklist

- [x] Pass-Through mode routes small designs (<100 nets, <1mm²) with zero G-Cell pre-processing.
      *Verified: `test_pad_to_pad_automatic.hw` routes successfully with 2 components, 1 net.*
- [ ] Hierarchical mode partitions large designs and routes G-Cells in parallel via Rayon.
      *Implemented but requires >100 nets to trigger; no large design test yet.*
- [x] Multi-pin nets produce Steiner T-junctions (not daisy chains) via dynamic target expansion.
      *Verified: `test_pad_to_vias_escape.hw` routes 8 nets from 1 pad to 8 vias (3-pin nets via SDF A*).*
- [ ] ASIC via transitions (M1→M4) unroll into single-layer vias with intermediate landing pads.
      *Implemented but requires ASIC profile test case.*
- [ ] PCB via transitions emit a single through-hole via.
      *Implemented in `unroll_via_tower` for non-Manhattan angle restriction.*
- [ ] High-speed traces avoid ground-plane voids (cost penalty forces rerouting).
      *Implemented in `cost.rs` with +5M penalty; requires SI test case.*
- [x] `substrate_layers` and `net_frequencies` are propagated from `compile_space` → `route_space` → `route_flat` → A* cost evaluator.
      *All signatures updated and threaded through.*
- [x] Profile `track_pitch`, `grid_snapping`, `layer_directions`, `enclosures` are parsed and surfaced to the router.
      *Parser additions complete in `profile.rs`, `constraints.rs`, `via.rs`.*
- [x] All new types compile without warnings (`cargo clippy` clean).
      *All new code compiles with zero new clippy warnings (115 pre-existing in hwc-compiler, 22 in hwc-export, 4 in hwc-cli).*

---

## v0.1.7 Port Escape Routing (Implemented)

> **Spec:** `Docs/v0.1.7/VIA-AND-PAD-PORT-ESCAPE-SPECIFICATION.md`

### Implementation Summary

The port escape system enables directional `exit:`/`enter:` syntax on `route` statements, allowing designers to control which pad edge traces depart from and arrive at. This eliminates the "dummy bounding box gap" where the router routed to pin centers instead of actual pad boundaries.

### Components Implemented

1. **Parser AST** (`hwc-parser/src/ast/space.rs`):
   - `RouteEscape` struct with `CardinalDirection` (North/South/East/West) and `EdgeOffsetSpec`
   - `exit_escape` / `enter_escape` fields on `Route` struct
   - `Exit` / `Enter` lexer tokens
   - `parse_route_escape()` and `parse_edge_offset()` parser functions

2. **Escape Position Calculator** (`hwc-engine/src/geometry_router/port_escape.rs`):
   - `CardinalPort` enum with direction vectors
   - `EdgeOffset` (Center, Percentage, Measurement, Named)
   - `smart_corner_clamp()` — prevents trace overhang at pad corners
   - `calculate_rect_escape()` — rectangular pad escape with clearance
   - `calculate_circular_escape()` — radial projection for circular pads/vias
   - `parse_port_escape()` — string-to-spec parser
   - 4 unit tests passing

3. **AutoRouter Integration** (`hwc-compiler/src/ir/routing/global.rs`):
   - `RouteEscapeSpec` struct for compiler-side escape data
   - `route_escape_specs` HashMap keyed by `(start_pin, goal_pin)` pairs
   - Escape-aware clipping in direct-route path (2-pin same-layer)
   - Escape-aware start/goal positions in SDF path (multi-pin nets)
   - Spatial pour bbox fallback for `contact(Copper)` vias via `get_pour_bbox_at_position()`

4. **Substrate Layer Spatial Lookup** (`hwc-engine/src/voxel_grid/grid/substrate_ops.rs`):
   - `get_pour_bbox_at_position()` — finds any pour layer at given coordinates
   - Enables Pin anchors (no pour) to resolve bbox from co-located contact(Copper) vias

### Test Coverage (6 test files, all passing)

| Test File | Geometry | Routes | Routing Path |
|-----------|----------|--------|-------------|
| `test_pad_to_pad_escape.hw` | Rect → Rect | 4 | Direct route (2-pin) |
| `test_ring_escape.hw` | Rect → Circle (ring) | 4 | Direct route (2-pin) |
| `test_pad_to_ring_escape.hw` | Rect → Circle (mixed) | 4 | Direct route (2-pin) |
| `test_pad_to_vias_escape.hw` | Rect → Vias | 8 | SDF A* (3-pin nets) |
| `test_ring_to_vias_escape.hw` | Circle → Vias | 4 | SDF A* (multi-pin) |
| `test_big_via_to_vias_escape.hw` | Big Via → Vias | 4 | SDF A* (5-pin net) |

### Key Bug Fixes

1. **Pour Alignment**: Rectangular pours with `device: N` anchor now resolve `[from, to]` coordinates relative to anchor edge point correctly.
2. **Circle Placement**: Circular pours with `device: N` use offset from pin anchor to center the circle at the correct component-relative position.
3. **Escape Clipping in AutoRouter**: The naive `goal_pos.x > start_pos.x` heuristic was replaced with escape-spec-aware directional clipping. Exit/enter specs now determine which pad edge traces clip to.
4. **Multi-Pin Escape**: Escape specs now work for SDF-routed multi-pin nets (not just 2-pin direct routes) via the `(start_pin, goal_pin)` keyed lookup.
5. **Spatial Pour Bbox**: Pin anchors co-located with `contact(Copper)` vias now resolve their bbox via spatial proximity in substrate layers.
