# v0.1.7 Base Implementation Roadmap: Reality as Code

This roadmap focuses on the transition to the **Unified 2.5D Routing, Stacking, and Manufacturing Architecture**. It ensures the compiler treats hardware as mathematically exact physical layers rather than a discrete voxel grid.

## Phase 1: Core 2.5D Analytic Engine

The goal is to transition the router from voxel-crawling to a 2.5D Shape-Based Pathfinder operating on continuous i64 coordinates.

### 1.1 The "Planar Lock" Mechanism
- **Goal**: Lock the A* pathfinder to a specific Z-height provided by the `StackupManager`.
- [x] Modify `route_net_sdf_accelerated` to accept a `fixed_z_nm` parameter.
- [x] Update `SdfGenerator` to handle Z-locking at substrate boundaries.

### 1.2 Minkowski Obstacle Inflation
- **Goal**: Implement O(1) collision overhead using inflated AABBs.
- [x] Implement Minkowski Sum logic for obstacle inflation: $Inflation = \frac{Width}{2} + Clearance$.
- [x] Create `BoundingBoxTracker` with layer-indexed obstacle storage, inflated AABB computation, and SDF integration.

### 1.3 The "Escape Exemption" Logic
- **Goal**: Allow traces to escape source/destination pins without self-collision.
- [x] Add `exempt_component_names` to `RoutingParams`.
- [x] Update `sdf.get_distance` logic to return `SAFE` for exempt components.

## Phase 2: Z-Axis Abstraction & Stackup Manager

### 2.1 Dual-Paradigm Elevation System
- **Goal**: Support both Assembly (Absolute Z) and High-Level (Layer-Based) paradigms.
- [x] Implement `StackupManager` to resolve `layer:` keywords to absolute Z-nanometers.
- [x] Support `origin: bl by b` (Bottom-Up) and `origin: tl by t` (Top-Down) resolution.

### 2.2 Explicit Substrate Syntax
- [x] Implement parser support for `add substrate(...) spanning [start] to [end]`.
- [x] Implement `cutouts:` property for manual solder mask/dielectric openings.

## Phase 3: Zero-Flicker GPU Handshake

### 3.1 Manifold Face Culling
- [x] Implement precedence-based face culling for coincident conductive/dielectric surfaces.
- [x] Propagate culling bitmasks to `create_box_with_holes_mesh` and `create_cylinder_mesh`.

### 3.2 glTF Depth Bias Metadata
- [x] Update `export_glb` to calculate dynamic `polygonOffset` based on stackup precedence.
- [x] Write `polygonOffset` and `renderOrder` to the glTF `extras` block.
- [x] Implement pattern-based material inference for unknown materials using lookup table approach.

## Phase 4: The 5-Stage Compiler Pipeline

1.  **Pass 1**: Topological Sort & Structural Placement.
2.  **Pass 2**: Obstacle Blitting & Keepout Map (Minkowski).
3.  **Pass 3**: Parallel 2.5D Auto-Routing & Auto-Via Stamping.
4.  **Pass 4**: Verification (P45/R16) & Dummy Metal Fill.
5.  **Pass 5**: Realization & Zero-Flicker Export (GLB Extras).

## Phase 5: Remaining Features

### 5.1 Portal Costs for Vias
- [x] Implement high-penalty costs for vertical layer transitions (vias).

### 5.2 ASIC H-Tree Synthesis
- [x] Implement recursive symmetrical H-Tree coordinate generation for clock nets.
- [x] Automatically identify H-Tree split nodes for buffer insertion.

### 5.3 DFM: Teardrops & Filleting
- [x] Implement filleted transitions at trace-to-pad junctions for `AnalyticTrace` primitives.
