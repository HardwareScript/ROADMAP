# Post-Bridge Limitations Roadmap

> **Context**: After completing the Bridge Implementation (Phases 1–5), a 3D visual
> demonstration (`demo_bridge_3d.hw`) was built and exported to `.glb` and `.dxf`.
> Four concrete limitations were identified from the live build output and visual
> inspection. This checklist tracks the work needed to fix them.

**Status**: COMPLETED (v0.1.7 RC)  
**Priority**: High (Production Readiness)  
**Depends On**: `BRIDGE-IMPLEMENTATION-CHECKLIST.md` Phase 5 ✅

> **Note**: This document is now finalized. Further v0.1.7 development is tracked in [PHYSICAL-TRUTH-MIGRATION.md](file:///c:/Users/olowo/Downloads/Code/Hardware-Script/ROADMAP/v0.1.7/PHYSICAL-TRUTH-MIGRATION.md).

---

## Limitation 1: Geometric Merging / Physical Continuity Bug ✅

> **Problem**: The physics engine's node graph flags auto-inserted vias as "disconnected
> islands" even when they geometrically touch (overlap) the pour boundary.
>
> **Status**: RESOLVED (v0.1.6)
> **Fix**: Implemented inclusive AABB adjacency checks in `island_builder.rs`.

---

## Limitation 2: Cylindrical Mesh Tessellation in 3D Export ✅

> **Problem**: Contacts with a circular `diameter:` property were exported as cuboids.
>
> **Status**: RESOLVED (v0.1.6)
> **Fix**: Implemented `create_cylinder_mesh` in `hwc-export` and wired it to the voxel stamper.

---

## Limitation 3: Component Footprint / Shape Library ✅

> **Problem**: No standard footprint library; parser bugs with negative coordinates.
>
> **Status**: RESOLVED (v0.1.6)
> **Fix**: Created `stdlib/footprints/basic.hw`. Fixed parser `dedent` and negative coordinate handling.

---

## Limitation 4: Material Alias — Symbol Table Integration ✅

> **Problem**: `material_alias` was parsed but not resolved.
>
> **Status**: RESOLVED (v0.1.6)
> **Fix**: Integrated recursive alias resolution into `SymbolTable` and `module_resolver.rs`.

---

## Limitation 5: Z-Axis Floating Gap (Coordinate Offset) ✅

> **Problem**: Components placed on the substrate surface (e.g. `z: 8`) appear to float
> slightly above the copper/substrate in 3D renders.
>
> **Status**: RESOLVED (v0.1.7)
> **Fix**: Implemented origin-aware `map_z_layer` for consistent 1-based indexing. Added `snap_to_surface` property.

---

## Limitation 6: Procedural Mesh Generation (Mathematical Parametrics) ✅

> **Problem**: Components are represented as simple boxes or cylinders. Real EDA users
> expect 3D models for complex packages like TO-220 or BGA, but importing external
> assets (GLB/STEP) introduces dependency bloat and versioning issues.
>
> **Status**: RESOLVED (v0.1.7)
> **Fix**: Implemented `create_to220_meshes` in `procedural.rs`. Added standard procedural material palette.

---

## Limitation 7: Internal Footprint Connectivity (Auto-Stitching & The Perfect Hole) ✅

> **Problem**: Footprints define pins and internal pours, but these are not yet
> automatically "stitched" (connected with vias) to the board-level nets during routing.
> Additionally, 3D renders show "Minecraft" square holes instead of cylindrical PTH.
>
> **Status**: RESOLVED (v0.1.7)
> **Fix**: Implemented the "Quadrant Partition" triangulation method and the "Butler" handshake for physically accurate PTH.

### Audit Feedback (v0.1.7)
*   **Compound Physical Event**: A hole is a sequence: Substrate Cutout (Void) -> Plating (Copper Tube) -> Annular Ring (Donut) -> Drill Hit (Excellon).
*   **Visual Truth**: GLB shows cylindrical voids with copper "sleeves" (Plating). No starburst artifacts or Z-fighting.
*   **Manufacturing Reality**: DXF shows HOLES (circles) and COPPER_TOP (donuts). DRL coordinates are perfectly matched.

### Fix
- [x] Implement "Pin Stitching": automatically add vias between footprint pads and board layers
- [x] Implement "Cylindrical Handshake": Update `VoxelGrid` and `SubstrateLayer` to support `Cylinder` cutouts
- [x] Update `hwc-export` (GLB/DXF) to render cylindrical voids and circular holes
- [x] Fix "Double Cylinder" artifact: Modified `create_tube_mesh` to render only the inner wall, eliminating overlapping surfaces in 3D renders.
- [x] Implement "Auto-Plating Butler": Modify via inserter to automatically stamp Copper Tubes (Plating) inside voids
- [x] Implement "Annular Ring Butler": Automatically generate copper "donuts" around holes based on Profile `min_annular_ring`
- [x] Update `auto_via_inserter` to recognize footprint pins as routing targets
- [x] Implement "Quadrant Partition" Triangulation: Replaced naive "fan" triangulation with local 3x3 grid partitioning to eliminate star artifacts and normal inversion.
- [x] Support for Multi-Layer PTH: Through, Buried, and Blind vias now correctly span and drill multiple substrate/copper layers automatically.

---

## Limitation 8: High-Fidelity Material Optics (v0.1.7) ✅

> **Problem**: Materials use basic PBR (Metal/Roughness). Real hardware exhibits 
> complex optical behavior like light refraction (IOR), clearcoat finishes, and 
> subsurface scattering (SSS) in substrates.
>
> **Status**: RESOLVED (v0.1.7)
> **Fix**: Decoupled standard `opacity` from advanced `subsurface` optics. Implemented KHR extensions for physically accurate "jelly" dielectric looks.

### Fix
- [x] Add `ior` (Index of Refraction) support for physically accurate substrate transparency
- [x] Implement `clearcoat` and `clearcoat_roughness` for polished solder mask/coating effects
- [x] Add `subsurface` scattering properties for realistic "milky" FR4/Prepreg looks
- [x] Implement mathematical anisotropy (Pivoted from normal mapping) to simulate surface grain
- [x] Support `transmission` and `volume` properties for physically accurate dielectric depth
- [x] Decouple `opacity` from `subsurface`: `opacity` now controls standard BLEND alpha, while `subsurface` triggers refractive volume effects.

---

## Cross-Cutting Concerns

### DRC Rules Completeness
- [x] Fix "Trace width violation" false positive for pour-to-pour connections (auto-via net continuity)
- [x] Fix "Via enclosure violation" for auto-inserted vias: Updated `calculate_annular_ring_from_substrate` to use true circular geometry (analytic distance) instead of AABB bounding boxes.
- [x] Fix "Via diameter violation": Integrated profile `default_via_diameter` into `AutoViaInserter` and `FabricationConstraints` to eliminate hardcoded defaults.

### Build Without Force-Export
- [x] Achieve clean `cargo run -- build` (no `--force-export`, no `--skip-*` flags) on `test_bridge_auto_via.hw`
- [x] Achieve clean `cargo run -- build` on `demo_bridge_3d.hw`
- [x] Achieve clean `cargo run -- build --foundry` on `test_bridge_override.hw`

---
