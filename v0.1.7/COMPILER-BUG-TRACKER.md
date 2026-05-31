# v0.1.7 Compiler Stabilization & Bug Fix Roadmap

This document tracks the technical debt and compiler gaps identified during the v0.1.7 "Physical Truth" migration, specifically regarding Z-Axis Abstraction and Engineering Mode validation.

## 1. Z-Axis Abstraction Gaps

### 1.1 The "Pile Paradox" (Double-Addition Bug)
- **Status**: Fixed (v0.1.7)
- **Description**: The compiler incorrectly accumulates Z-heights when a component is placed on a semantic layer. 
- **Example**: `Component at layer: top` (1.9mm) + `Internal Pour on layer: top` (1.9mm) = `3.8mm` (Exceeds board boundary).
- **Fix**: 
  - [x] Refactor `unroll_component` in `hwc-compiler` to treat internal semantic layer references as relative to the parent's base elevation, not as absolute offsets.
  - [x] Ensure `untransformed_origin.z` is correctly propagated during unrolling.

### 1.2 Missing "Semantic Relative" Elevation Mode
- **Status**: Fixed (v0.1.7)
- **Description**: Parser lacks a keyword to bind internal geometry to the "current" layer of the parent component.
- **Proposed Syntax**: `add pour(Material) on layer: self:` or `on z: relative:`
- **Fix**:
  - [x] Update `Elevation` enum in `hwc-parser` to include `Relative` variant.
  - [x] Update parser to recognize `on layer: self` and `on z: relative`.
  - [x] Implement resolution logic in `StackupManager` to handle `Relative` elevations.

### 1.3 Automatic Substrate & Pour Carving (Auto-Drill)
- **Status**: Partially Fixed (v0.1.7)
- **Description**: Conductive geometry (pads/vias) that overlaps with insulating substrates or different-net pours causes interpenetration errors or short circuits.
- **Fix**:
  - [x] Implement `Auto-Carve` for Pours in `hwc-compiler/src/ir/placement/pour.rs`.
  - [x] Implement `Auto-Carve` for Components in `hwc-compiler/src/ir/placement/component.rs`.
  - [x] Fix `SubstrateLayerType` classification in `pour.rs` (registered as `Pour` instead of `Substrate`).
  - [x] Implement net-aware `drill_hole` in `hwc-engine` and `HardwareSpace`.
  - [ ] Complete `Auto-Drill` for Contacts in `hwc-compiler/src/ir/placement/contact.rs` to prevent short circuits with internal planes.

## 2. Professional Mode (DRC) Inconsistencies

### 2.1 Via Enclosure vs. Trace Width Logic
- **Status**: Fixed (v0.1.7)
- **Description**: `AutoViaInserter` fails if a via's physical diameter (e.g., 0.6mm) is wider than the intersecting trace (e.g., 0.4mm), even if the trace is intended to be a landing pad.
- **Fix**:
  - [x] Implemented "Dielectric Skip" in `validate_via_enclosure_analytic` to avoid false positives on insulating layers.
  - [x] Implemented "Self-Enclosure" filtering to prevent a via from failing against its own geometry.
  - [x] Implemented "Smart Via Selection" to prefer smaller vias for signals, avoiding overhang on narrow traces.

### 2.2 Short-Circuit False Positives in 3D Booleans
- **Status**: Identified
- **Description**: The continuity validator sometimes flags physical overlaps between a component pad and its own landing rail as a short circuit if the bounding boxes intersect before the boolean merge is complete.
- **Required Fix**: Update `ContinuityValidator` to prioritize `merge: true` flags during the initial net-island extraction phase.

## 3. Physical Continuity & Net Assignment

### 3.1 P43 Unassigned Conductor for Component Pads
- **Status**: In-Progress
- **Description**: Unrolled internal pours for components (VCC, GND, IO pads) have `net: None` to avoid redundant anchors, causing P43 errors.
- **Required Fix**: Dynamically resolve net name in `place_pour` by querying the `NetlistArena` for the bound component pin.

## 4. Standard Library & Profile Transition

### 4.1 Dynamic Via Definitions in Profiles
- **Status**: Fixed (v0.1.7)
- **Description**: Hardcoded `ViaLibrary` causes mapping mismatches with physical stackups.
- **Fix**:
  - [x] Updated `hwc-parser` to support `via` blocks in `profile` blocks.
  - [x] Updated `AutoViaInserter` to query the active profile's via library.
  - [x] Stripped all hardcoded via measurements from the compiler (v0.1.7 "Fail-Fast" architecture).

## 5. Verification Gauntlet
- [x] Implement `Elevation::Relative` in `hwc-parser/src/ast/space.rs`
- [x] Fix `unroll_pour` Z-accumulation in `hwc-compiler/src/ir/parametric_unroller/substitution.rs`
- [x] Update `StackupManager` to handle relative elevation resolution
- [x] Fix analytic via enclosure for 3D multi-layer boards.
- [x] Resolve P41 Disconnected Net issue via correct `SubstrateLayerType` classification.
- [x] Verify passing build for `complex_vias_advanced.hw` in Engineering Mode without hacks (2026-05-31).
