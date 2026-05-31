# v0.1.7 Z-Axis Abstraction Implementation Roadmap

**Source of Truth**: `Docs/v0.1.7/Z-AXIS-ABSTRACTION.md`  
**Status**: Phase 5 downstream/export **complete** (2026-05-24). Export pipelines (`DXF`, `Excellon`, solder mask/paste) and CLI validation use physical Z; `hwc-export::physical_z` centralizes nm/mm helpers. Phase 4 engine purification complete.  
**Breaking Change**: v0.1.6 → v0.1.7  
**Goal**: Eliminate raw integer `z: N` layer indices. Introduce dual-paradigm Z handling:
- **Assembly Paradigm**: Physical units (`z: 150um`)
- **High-Level Paradigm**: Semantic layers (`layer: l1`) resolved via Profile `stackup`
- **Manufacturing DNA**: Material-level process declaration (`process: drilled_plated`) (2026-05-24)

**Key Architectural Rule (from spec)**:
> The Voxel Engine must speak *only* in nanometers. It must never know what a "layer" is.
> Manufacturing intent (drilling vs depositing) is carried by the material's process DNA.

### Progress Snapshot (2026-05-24)
- **Phase 1 (AST + Parser Surface)**: Fully complete
  - Added `ManufacturingProcess` enum to `MaterialDefinition`
- **Phase 2 (Profile Stackup)**: Fully complete — `LayerStackup` is the only stackup source of truth
- **Phase 3 (Compiler IR Translation Layer)**: Fully complete
  - `StackupManager`: `resolve_elevation`, `resolve_elevation_top`, `elevation_from_z_nm`, `grid_layer_index`
  - `ir/conversions.rs`: `resolve_coordinate_z_nm` (physical units only; dimensionless `z: N` rejected)
  - `contact.rs`: Implemented unified manufacturing logic (DrilledPlated vs Deposited vs Etched)
  - Placement paths use resolved `z_nm` directly: `pour.rs`, `contact.rs`, `substrate.rs`, `component.rs`, `array.rs`
  - `auto_via_inserter.rs`: auto-vias emit physical mm elevations from pour bbox Z (no raw layer-index literals in `Elevation`)
  - `parametric_unroller/substitution.rs`: preserves `Elevation::Semantic` through unrolling
  - `symbol_table/layer.rs`: documented as authority-stack layers (not PCB Z); PCB Z lives in `StackupManager`
- **Compiler**: dimensionless `z: N` in coordinates is **rejected** (error C26); use physical units or `layer:` syntax
- **AutoViaInserter** / `ViaLibrary`: **Fail-Fast v0.1.7 Implementation Complete** (2026-05-31). Removed all hardcoded measurement defaults from the compiler. Vias are now exclusively driven by the active profile.
- **Phase 4 (Engine Purification)**: Fully complete
- **Phase 7 (Fail-Fast Vias & Smart Selection)**: Fully complete (2026-05-31)
  - [x] Stripped hardcoded measurement defaults (0.3mm/0.15mm) from compiler.
  - [x] Implemented `Smart Selection` (prefer small for signal, large for power).
  - [x] Added `Self-Enclosure` filtering in analytic DRC.
  - [x] Moved generic via definitions to `@std/profiles/generic.hw`.

### Progress Snapshot (2026-05-31)
- **Phase 1 (AST + Parser Surface)**: Fully complete
  - Added `ManufacturingProcess` enum to `MaterialDefinition`
