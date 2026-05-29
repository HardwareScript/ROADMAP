# v0.1.7 Compiler Stabilization & Bug Fix Roadmap

This document tracks the technical debt and compiler gaps identified during the v0.1.7 "Physical Truth" migration, specifically regarding Z-Axis Abstraction and Engineering Mode validation.

## 1. Z-Axis Abstraction Gaps

### 1.1 The "Pile Paradox" (Double-Addition Bug)
- **Status**: Identified
- **Description**: The compiler incorrectly accumulates Z-heights when a component is placed on a semantic layer. 
- **Example**: `Component at layer: top` (1.9mm) + `Internal Pour on layer: top` (1.9mm) = `3.8mm` (Exceeds board boundary).
- **Required Fix**: Refactor `unroll_component` in `hwc-compiler` to treat internal semantic layer references as relative to the parent's base elevation, not as absolute offsets.

### 1.2 Missing "Semantic Relative" Elevation Mode
- **Status**: Planned
- **Description**: Parser lacks a keyword to bind internal geometry to the "current" layer of the parent component.
- **Proposed Syntax**: `add pour(Material) on layer: self:` or `on z: relative:`
- **Impact**: Removes the need for the "Hybrid Hack" of using `0mm` absolute offsets in component definitions.

## 2. Professional Mode (DRC) Inconsistencies

### 2.1 Via Enclosure vs. Trace Width Logic
- **Status**: Identified
- **Description**: `AutoViaInserter` fails if a via's physical diameter (e.g., 0.6mm) is wider than the intersecting trace (e.g., 0.4mm), even if the trace is intended to be a landing pad.
- **Required Fix**: Implement "Teardrop" generation or allow "Enclosure Breakout" in `hwc-physics` when the intersection net is identical.

### 2.2 Short-Circuit False Positives in 3D Booleans
- **Status**: Identified
- **Description**: The continuity validator sometimes flags physical overlaps between a component pad and its own landing rail as a short circuit if the bounding boxes intersect before the boolean merge is complete.
- **Required Fix**: Update `ContinuityValidator` to prioritize `merge: true` flags during the initial net-island extraction phase.

## 3. Parser & Lexer Stabilization

### 3.1 Soft-Keyword Ambiguity
- **Status**: Partially Fixed (Refactored `module.rs`)
- **Description**: Keywords like `layer`, `device`, and `on` are context-dependent. The parser occasionally fails to differentiate between a property name and a syntax keyword.
- **Required Fix**: Standardize `expect_identifier_or_keyword` across all definition parsers to remove reliance on `continue` jumps.

### 3.2 Component Body Logic Block
- **Status**: Planned
- **Description**: Components currently only support static `layout:` blocks. They lack the `logic:` and `add` capabilities of Modules.
- **Goal**: Full HSM (Hardware Script Module) unification where Components are just Modules with physical shapes.

## 4. Verification Gauntlet
- [ ] Fix `unroll_pour` Z-accumulation in `hwc-compiler/src/ir/unroll.rs`
- [ ] Add `Teardrop` support to `hwc-physics/src/drc/enclosure.rs`
- [ ] Verify passing build for `complex_vias_advanced.hw` in Engineering Mode without hacks.
