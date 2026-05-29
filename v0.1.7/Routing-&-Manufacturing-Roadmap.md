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
