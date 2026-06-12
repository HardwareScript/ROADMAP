# Manufacturing Yield Implementation Checklist

This document tracks the implementation of ohmic bridge sandwiches, junction tapers, dummy metal fill, via stub back-drilling, and the consolidated 6-stage compile pipeline for the v0.1.7 compiler.

**Source Docs:** `Docs/v0.1.7/Adaptive-Heuristic.md`, `Docs/v0.1.7/Unified-2.5D-3D-Routing-and-Placement.md`

---

## List 1: Ohmic Bridge Sandwich in Route Committer

> **Target file:** `hwc/crates/hwc-compiler/src/ir/space_builder.rs` (add `commit_routes` method to `HardwareSpace`)

- [ ] **1. Trace Segment Commitment**
  - *What it is:* Commit horizontal copper trace segments from the router's `RouteResult` into the voxel grid.
  - *Implementation:*
    - Add `pub fn commit_routes(&mut self, results: RouteResult, symbol_table: &SymbolTable) -> Result<(), IrError>` to `HardwareSpace`.
    - For each `(net_id, path)` in `results.paths`, iterate over `path.windows(2)` and stamp each segment into the grid using `VoxelGrid::set_occupied()` or the existing stamp utilities.
  - *Note:* The existing `VoxelGrid` (`hwc/crates/hwc-engine/src/voxel_grid\grid\core.rs`) already has `set_occupied(x, y, z, material, handle)` and `stamp_cylinder()`.

- [ ] **2. Via Commitment with Bridge Sandwich Rule**
  - *What it is:* When committing a via, query the `BridgeTable` to determine if an ohmic interface material is needed at the boundary between semiconductor and metal layers.
  - *Implementation:*
    - For each via in `results.vias`, call `active_profile.bridge_table.lookup(start_mat, end_mat)` (from `hwc/crates/hwc-compiler/src\bridge_resolver.rs`).
    - If a `BridgeStack` is returned:
      - Stamp 1 voxel layer of `interface_material` at the boundary Z position.
      - Fill the remaining vertical column with `fill_material` (typically Tungsten).
    - If no bridge: fill the entire via column with `default_via_fill`.

- [ ] **3. BridgeStack Material Resolution**
  - *What it is:* Look up the correct interface and fill materials for a given layer-to-layer transition.
  - *Implementation:*
    - Use `BridgeTable::lookup(from_material, to_material) -> Option<&BridgeStack>` (already implemented in `hwc/crates/hwc-compiler/src/bridge_resolver.rs`).
    - `BridgeStack` contains `interface_material`, `interface_thickness_nm`, and `fill_material`.
    - Use `VoxelGrid::stamp_cylinder()` or `set_occupied()` to write the bridge material at the boundary voxel.
  - *Verify:* A via from Silicon to Copper produces a Cobalt_Silicide boundary layer + Tungsten fill.

---

## List 2: Junction Taper & Teardrop Integration

> **Target files:**
> - `hwc/crates/hwc-engine/src/geometry_router/teardrops.rs` (extend `TeardropEngine`)
> - `hwc/crates/hwc-compiler/src/ir/space_builder.rs` (invoke after route commitment)

- [ ] **1. Wide Junction Detection**
  - *What it is:* After routes are committed, identify T-junctions where a thick trunk trace meets a thin branch trace and apply mitered tapers.
  - *Implementation:*
    - Add `pub fn apply_junction_tapers(&mut self, grid: &mut VoxelGrid, constraints: &ConstraintRulebook) -> Result<(), RoutingError>` to `TeardropEngine`.
    - Scan the voxel grid for junction nodes where segments of different widths intersect.
    - Generate triangular transition wedges at each junction.

- [ ] **2. Profile Strategy Gate**
  - *What it is:* The taper pass only runs when `strategy: WideJunction` is enabled in the active profile.
  - *Implementation:*
    - In the compile pipeline, check `active_profile.constraints.strategy == Strategy::WideJunction` before invoking `TeardropEngine`.
    - The existing `TeardropEngine::apply_teardrops()` handles endpoint fillets; the new `apply_junction_tapers()` handles mid-segment T-junctions.

- [ ] **3. Clipper Union for Overlapping Geometries**
  - *What it is:* The teardrop/taper stamps overlap with existing traces. The 2D Clipper engine must weld them into a single non-overlapping copper manifold.
  - *Implementation:*
    - Leverage existing polygon rasterizer at `hwc/crates/hwc-engine/src\geometry_router\polygon_rasterizer.rs`.
    - Ensure taper polygons are added to the per-layer copper pool before final union.
  - *Verify:* A thick-trunk / thin-branch junction produces a smooth mitered transition with no acid traps.

---

## List 3: Dummy Metal Fill Engine (Thieving Pass)

> **Target file:** `hwc/crates/hwc-compiler/src/ir/placement/dummy_fill.rs` (new module)

- [ ] **1. DummyFillEngine Struct & Constructor**
  - *What it is:* Engine that injects isolated, non-functional copper squares into low-density zones to maintain uniform copper density for CMP.
  - *Implementation:*
    - Create `hwc/crates/hwc-compiler/src/ir/placement/dummy_fill.rs`.
    - Define `pub struct DummyFillEngine { target_density: f64, dummy_size_nm: i64, dummy_spacing_nm: i64, clearance_nm: i64 }`.
    - Constructor reads from profile: `target_density` (default 0.45), `dummy_size_nm` (default 2000), `dummy_spacing_nm` (default 4000), `clearance_nm` (default 3000).

- [ ] **2. Zone-Based Density Analysis**
  - *What it is:* Divide the grid into coarse zones (100µm), calculate copper density per zone, and identify under-filled regions.
  - *Implementation:*
    - Add `pub fn apply_thieving(&mut self, grid: &mut VoxelGrid, copper_material_id: MaterialId) -> Result<(), ThievingError>`.
    - Iterate over 100µm zones on each metal layer.
    - For each zone, call `grid.calculate_copper_density(&zone_bbox, copper_material_id)` (new helper needed on `VoxelGrid`).

- [ ] **3. Dummy Stamp with Clearance Check**
  - *What it is:* Stamp isolated dummy squares in under-filled zones while maintaining clearance from active routed nets.
  - *Implementation:*
    - Add private `fn fill_zone_with_dummies(&self, grid, zone, z_nm, material) -> Result<(), ThievingError>`.
    - Grid-scan the zone at `dummy_spacing_nm` intervals.
    - At each candidate position, check `grid.has_active_nets_in_radius(center, clearance_nm)` (new helper needed on `VoxelGrid`).
    - If clear, stamp a solid box using `VoxelGrid::set_occupied()` or a new `stamp_solid_box()` helper with `NetId::UNCONNECTED`.

- [ ] **4. Module Registration**
  - *What it is:* Register the new module in the placement module tree.
  - *Implementation:*
    - Add `pub mod dummy_fill;` to `hwc/crates/hwc-compiler/src/ir/placement/mod.rs`.
    - Re-export `DummyFillEngine`.
  - *Verify:* A design with large empty copper zones receives isolated dummy squares that do not touch active traces.

---

## List 4: Via Stub Detection & Back-Drill Scheduling

> **Target file:** `hwc/crates/hwc-compiler/src/ir/placement/back_drill.rs` (new module)

- [ ] **1. BackDrillAnalyzer Struct**
  - *What it is:* Analyzes routed vias on high-frequency nets and identifies dangling stubs that must be back-drilled to prevent signal reflections.
  - *Implementation:*
    - Create `hwc/crates/hwc-compiler/src/ir/placement/back_drill.rs`.
    - Define `pub struct BackDrillAnalyzer { pub frequency_threshold_hz: f64 }` (default 1 GHz = `1_000_000_000.0`).

- [ ] **2. Stub Length Calculation**
  - *What it is:* For each via on a high-speed net, determine if the unused vertical portion (stub) exceeds the critical threshold (200µm at GHz frequencies).
  - *Implementation:*
    - Add `pub fn analyze_via_stubs(&self, vias: &[Via], net_frequencies: &FxHashMap<NetId, f64>, stackup: &StackupManager) -> Vec<BackDrillDirective>`.
    - For each via: if `net_freq >= threshold` and via is through-hole (starts at top, ends at inner layer), compute `stub_length = |end_z - board_bottom_z|`.
    - If `stub_length > 200_000` nm, emit a `BackDrillDirective`.

- [ ] **3. BackDrillDirective Type**
  - *What it is:* Describes a back-drill operation for manufacturing export (Excellon format).
  - *Implementation:*
    - Define `pub struct BackDrillDirective { pub pos: Point3D, pub drill_depth_nm: i64, pub drill_diameter_nm: i64 }`.
    - `drill_diameter_nm` = via drill diameter + 100µm clearance.
    - Store directives on `HardwareSpace::back_drill_schedule: Vec<BackDrillDirective>`.

- [ ] **4. StackupManager Integration**
  - *What it is:* The analyzer needs layer Z positions and board boundaries from the stackup.
  - *Implementation:*
    - Use `StackupManager::get_layer_start_z(layer_name) -> Option<i64>` and `board_thickness_nm() -> i64`.
    - Import `StackupManager` from `hwc/crates/hwc-compiler/src/ir/stackup_manager.rs`.
  - *Verify:* A through-hole via on a 2.5 GHz net with a 300µm stub produces a `BackDrillDirective` with depth=300µm.

---

## List 5: Consolidated 6-Stage Compile Pipeline

> **Target file:** `hwc/crates/hwc-compiler/src/ir/mod.rs` (`program_to_space` function)

- [ ] **1. Pass 1 & 2: Topological Placement (Existing)**
  - *What it is:* Components are placed from the AST. Already implemented in `program_to_space`.
  - *No changes needed* — verify existing flow.

- [ ] **2. Pass 3: Adaptive Routing**
  - *What it is:* Initialize routing constraints from the active profile and invoke the adaptive router with full physical context.
  - *Implementation:*
    - Build `RoutingConstraints` from `active_profile` (angle_restriction, track_pitch, layer_directions, etc.).
    - Extract `net_frequencies` from `space.netlist.extract_frequencies()` before routing.
    - Call `route_space(bbox, nets, obstacles, &space.substrate_layers, &net_frequencies)` — context propagation ensures the cost evaluator can detect reference-plane voids and classify high-speed nets.
    - Store `RouteResult` for subsequent passes.

- [ ] **3. Pass 4: Geometric Realization & Ohmic Bridge Sandwiches**
  - *What it is:* Commit routes to the voxel grid with bridge material stamping.
  - *Implementation:*
    - Call `space.commit_routes(route_results, symbol_table)?` (from List 1 above).
    - This stamps traces, vias, and ohmic bridge sandwiches in one pass.

- [ ] **4. Pass 5.1: Junction Taper & Teardrop Generation**
  - *What it is:* Apply mitered tapers at T-junctions when `strategy: WideJunction` is active.
  - *Implementation:*
    - Gate on `active_profile.constraints.strategy == Strategy::WideJunction`.
    - Call `TeardropEngine::new().apply_junction_tapers(&mut space.voxel_grid)?`.

- [ ] **5. Pass 5.2: Dummy Metal Fill (Thieving Pass)**
  - *What it is:* Inject isolated dummy copper in low-density zones.
  - *Implementation:*
    - Gate on `active_profile.constraints.dummy_fill.unwrap_or(false)`.
    - Call `DummyFillEngine::new(profile).apply_thieving(&mut grid, copper_id)?`.

- [ ] **6. Pass 5.3: Via Stub Analysis & Back-Drill Scheduling**
  - *What it is:* Identify high-frequency via stubs and schedule back-drilling for Excellon export.
  - *Implementation:*
    - Reuse `net_frequencies` extracted before routing (no duplicate extraction).
    - Call `BackDrillAnalyzer::new().analyze_via_stubs(&vias, &net_frequencies, &space.substrate_layers)`.
    - Store directives on `space.back_drill_schedule`.
  - *Verify:* The full pipeline compiles end-to-end and the 6 stages execute in order.

---

## Verification Checklist

- [ ] Ohmic bridge sandwich stamps 1 voxel layer of interface material at semiconductor-metal boundaries.
- [ ] Junction tapers generate mitered wedges at thick/thin T-junctions when `strategy: WideJunction` is set.
- [ ] Dummy fill stamps isolated squares in zones below target copper density.
- [ ] Dummy fill respects clearance from active routed nets.
- [ ] Back-drill analyzer identifies stubs >200µm on nets ≥1 GHz.
- [ ] Back-drill directives are stored on `HardwareSpace` for Excellon export.
- [ ] `commit_routes` handles both trace segments and via bridges in a single pass.
- [ ] `substrate_layers` and `net_frequencies` are extracted once before routing and propagated to both the router and back-drill analyzer.
- [ ] The 6-stage pipeline executes in order: Placement → Routing → Realization → Tapers → Fill → Back-Drill.
- [ ] All new modules compile (`cargo build`) and pass clippy (`cargo clippy`).
