# v0.1.7 Verification & Validation Roadmap

This roadmap tracks the functional verification of Hardware Script v0.1.7 features using `.hw` scripts and compiler-level integration tests. The goal is to move from "Internal Testing" to "Canonical Hardware Script Validation."

## Phase 1: Core Physical Infrastructure

### 1.1 Stackup & Elevation Resolution
- **Goal**: Verify that the `StackupManager` correctly resolves semantic layers to absolute nanometers.
- **Test Case**: `tests/v0.1.7/validation/stackup_resolution.hw`
- **Verification**:
  - [x] Bottom-Up Origin (`bl by b`): Verify Layer 1 starts at Z=0.
  - [x] Top-Down Origin (`tl by t`): Verify Layer 1 starts at Total Height.
  - [x] Semantic resolution: `add pour on layer: top_copper` resolves to correct Z.
  - [x] Physical Range: `on z: start to end` syntax correctly sets pour thickness.

### 1.2 Minkowski Obstacle Inflation
- **Goal**: Verify that traces obey inflated clearances without voxel-level checks.
- **Test Case**: `tests/v0.1.7/validation/minkowski_clearance.hw`
- **Verification**:
  - [ ] Trace bends around a component's inflated AABB (Width/2 + Clearance).
  - [ ] O(1) collision detection: Verify routing speed doesn't degrade with complex keepouts.

### 1.3 Planar Lock & Escape Exemption
- **Goal**: Verify traces stay planar and can escape starting pins.
- **Test Case**: `tests/v0.1.7/validation/planar_escape.hw`
- **Verification**:
  - [ ] Trace Z-coordinate is constant (Planar Lock).
  - [ ] Trace successfully starts inside a component's BBox (Escape Exemption).

## Phase 2: Advanced Routing & Patterns

### 2.1 Symmetrical H-Tree (ASIC)
- **Goal**: Verify fractal clock distribution generation.
- **Test Case**: `tests/v0.1.7/validation/htree_clock.hw`
- **Verification**:
  - [ ] Recursive H-Tree coordinates generated for 4, 16, and 64 targets.
  - [ ] Layer-swapping (M3/M4) at branches.

### 2.2 Length Matching (PCB)
- **Goal**: Verify Trombone/Serpentine insertion.
- **Test Case**: `tests/v0.1.7/validation/length_matching.hw`
- **Verification**:
  - [ ] Trace length matches target within tolerance (e.g., 10nm).
  - [ ] Folds are inserted in unobstructed 2D zones.

## Phase 3: Manufacturing & DFM

### 3.1 Ohmic Bridge Auto-Stamping
- **Goal**: Verify bridge material insertion at material transitions.
- **Test Case**: `tests/v0.1.7/validation/ohmic_bridge.hw`
- **Verification**:
  - [ ] 1-voxel layer of `Graphene_Ohmic` inserted between `Silicon_N` and `Copper`.
  - [ ] Remainder of via filled with `Copper`.

### 3.2 Dummy Metal Fill (Thieving)
- **Goal**: Verify density-based metal filling.
- **Test Case**: `tests/v0.1.7/validation/dummy_fill.hw`
- **Verification**:
  - [ ] Isolated squares inserted in low-density zones.
  - [ ] Dummies maintain minimum clearance from functional nets.

### 3.3 DFM Teardrops
- **Goal**: Verify filleted junctions.
- **Test Case**: `tests/v0.1.7/validation/teardrop_dfm.hw`
- **Verification**:
  - [ ] Geometric "teardrops" added at trace-to-pad entries.

## Phase 4: Export & Visual Validation

### 4.1 Zero-Flicker GPU Handshake
- **Goal**: Verify glTF metadata and face culling.
- **Test Case**: `tests/v0.1.7/validation/zero_flicker_handshake.hw`
- **Verification**:
  - [ ] `polygonOffset` extras present in GLB materials.
  - [ ] Shared faces culled (No Z-fighting in Studio).

## Execution Status
- [x] Phase 1.1: Stackup & Elevation Resolution (Verified via `stackup_resolution.hw`)
- [ ] Phase 1.2: Minkowski Obstacle Inflation
- [ ] Phase 1.3: Planar Lock & Escape Exemption
- [ ] Phase 2
- [ ] Phase 3
- [ ] Phase 4
