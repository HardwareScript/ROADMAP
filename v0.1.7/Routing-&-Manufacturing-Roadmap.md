# v0.1.7 Routing & Manufacturing Roadmap

This roadmap tracks the implementation of the **2.5D Shape-Based Analytic Pathfinder** and the **Pattern Engine** for v0.1.7.

## Phase 1: 2.5D Shape-Based Pathfinder

### 1.1 Analytic Collision Detection
- [x] **Minkowski Inflation**: Inflate obstacle AABBs by $(Width/2 + Clearance)$ for O(1) ray-collision checks. (Implemented in `BoundingBoxTracker`)
- [x] **Swept-Volume Checking**: Implement 3D swept-volume bounding box checks for path segments. (Implemented in `VoxelGrid` operations)
- [x] **Escape Exemption**: Ensure source/destination component AABBs are treated as `Cost::SAFE` during escape/docking steps. (Implemented via `exempt_components` in `RoutingParams`)

### 1.2 The "Planar Lock" & Via Portal Costs
- [x] **Z-Lock**: Pathfinder is restricted to the Z-plane provided by the `StackupManager`. (Implemented via `fixed_z_nm` in `RoutingParams`)
- [x] **Portal Costs**: Implement high-penalty costs for vertical layer transitions (vias). (Implemented in `cost.rs`: via penalty = 50)

## Phase 2: Symmetrical & Length-Matching Pattern Engine

### 2.1 PCB Patterns: Trombone & Serpentine
- [x] **Length Targeter**: Identify the longest trace in a signal group and set as target.
- [x] **Meander Generator**: Insert parameterized Serpentine/Trombone folds to consume length deficits.
  - Implemented `LengthMatchingEngine` with `calculate_deficits()` and `generate_meander()` methods
  - Basic pattern primitives exist in `routing_patterns.rs`: `StandardPatterns::zigzag()`, `StandardPatterns::trombone()`, `StandardPatterns::serpentine()`
  - `RoutedTrace` struct for tracking trace metadata and length

### 2.2 ASIC Patterns: H-Tree Synthesis
- [x] **Fractal Generator**: Implement recursive symmetrical H-Tree coordinate generation for clock nets.
- [x] **Buffer Scheduling**: Automatically identify H-Tree split nodes for buffer insertion.

## Phase 3: Manufacturing Artifacts & Yield

### 3.1 Ohmic Bridge Auto-Via Stamping
- [x] **Bridge Table**: Implement look-up table for material transition bridges (e.g., Silicon -> Graphene -> Copper). (Implemented in `auto_via_inserter.rs`)
- [x] **Compound Via Inserter**: Automatically stamp a 1-voxel bridge material layer at transition boundaries. (Implemented)

### 3.2 Code-Defined Dummy Metal Fill (Thieving)
- [x] **Density Analyzer**: Calculate metal density in coarse grid zones post-routing. (Implemented in `dummy_fill.rs`)
- [x] **Dummy Stamper**: Fill low-density zones with isolated metal squares while respecting net clearances. (Density analysis complete, stamping partially implemented)

### 3.3 DFM: Teardrops & Filleting
- [x] **Teardrop Engine**: Implement filleted transitions at trace-to-pad junctions. (Implemented in `teardrops.rs` with `TeardropEngine::apply_teardrops()`)
- [x] **Analytic Integration**: Hook into `AnalyticTrace` primitive for automatic generation.

## Phase 4: Release Candidate Validation
- [x] **5-Stage Pipeline Integration**: Verify topological sort -> obstacle blitting -> routing -> validation -> export.
- [x] **Deterministic Build Test**: Ensure identical binary outputs for repeated builds.
- [x] **glTF Handshake**: Verify `polygonOffset` metadata correctly resolves Z-fighting in Hardware Script Studio. (Implemented via `get_or_create_material` with pattern-based inference)

## Phase 4.5: Port Escape Routing (v0.1.7)
- [x] **Cardinal Port Escape Syntax**: `exit:` / `enter:` keywords on `route` statements with N/S/E/W directions. (Parser: `parse_route_escape()`, AST: `RouteEscape` struct)
- [x] **Edge-Offset Heuristic**: Percentage, measurement, and named position overrides along pad edges. (Engine: `EdgeOffset` enum, `resolve_offset_to_ratio()`)
- [x] **Smart Corner Clamping**: Prevents trace overhang at pad corners via min/max ratio bounds. (Engine: `smart_corner_clamp()`)
- [x] **Rectangular Pad Escape**: Coordinate-locked escape from rectangular pad bounding boxes. (Engine: `calculate_rect_escape()`)
- [x] **Circular Pad/Ring Escape**: Radial projection mapping for circular pours and vias. (Engine: `calculate_circular_escape()`)
- [x] **AutoRouter Integration**: Escape-aware clipping in both direct-route (2-pin) and SDF (multi-pin) paths. (Compiler: `RouteEscapeSpec` keyed by `(start_pin, goal_pin)`)
- [x] **Spatial Pour Bbox for Vias**: Pin anchors co-located with `contact(Copper)` vias resolve bbox via spatial proximity. (Engine: `get_pour_bbox_at_position()`)
- [x] **Test Coverage**: 6 test files covering pad→pad, pad→ring, ring→vias, pad→vias, big-via→vias geometries. All passing.


### Phase 5: High-Scale Routing Optimization (Post-v1.0)

#### 5.1 Bidirectional A* with Segment Stitching
When scaling to multi-million net SoC layouts where the Leap-Frog router encounters high congestion, the search space can be optimized by transitioning to a Bidirectional A* search:
- **Heuristic Balance**: Use symmetric Euclidean distance estimators for both forward and backward frontiers.
- **Z-Boundary Stitching**: If the frontiers meet on different Z-layers, the meeting node is designated as a mandatory via portal, triggering the via-stamping engine to weld the Z-gap.
- **Asymmetric Constraint Handling**: Keep-out zone (KOZ) exemptions must be evaluated independently for the start and end frontiers to prevent early search termination.