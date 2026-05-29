# Hardware Script v0.1.7: The "Physical Truth" Release

**Status**: Release Candidate / Feature Freeze  
**Theme**: Transitioning from "Minecraft" voxels to high-fidelity, manufacturing-ready geometry and optics.

---

## 1. High-Fidelity Optics & Materials ✅ COMPLETE
*Achieved photorealistic CAD quality via advanced PBR extensions.*

- [x] **Refractive Optics**: Implemented `ior` (Index of Refraction) and `transmission` for realistic substrate depth.
- [x] **Clearcoat Finishing**: Added `clearcoat` and `clearcoat_roughness` for polished solder mask effects.
- [x] **Subsurface Scattering**: Implemented `subsurface` scattering for realistic "milky" FR4/Prepreg appearance.
- [x] **Anisotropy**: Added mathematical anisotropy to simulate surface grain (brushed metal/fiberglass) without external assets.

---

## 2. Geometric Precision (Limitation 7 Fixes) ✅ COMPLETE
*Moving from square approximations to true circular primitives.*

- [x] **The Perfect Hole**: Implemented "Cylindrical Handshake" between `VoxelGrid` and `hwc-export` for true PTH geometry.
- [x] **Circular DRC**: Updated DRC engine to use analytic distance checks for annular rings, eliminating false positives on circular pads.
- [x] **Single-Wall Rendering**: Fixed "Double Cylinder" artifact in GLB export for cleaner 3D views.
- [x] **Auto-Plating Butler**: Implemented automatic generation of copper tubes (plating) and annular rings (donuts) based on profile constraints.
- [x] **Quadrant Partition Triangulation**: Eliminated starburst artifacts and normal inversion in circular mesh generation.

---

## 3. Automation & Constraints ✅ COMPLETE
*Reducing hardcoded values in favor of profile-driven design.*

- [x] **Constraint-Driven Vias**: `AutoViaInserter` now respects `default_diameter` and `min_annular_ring` from user profiles.
- [x] **Pin Stitching**: Automatically detect and connect footprint pins to internal nets via auto-inserted vias.
- [x] **Coordinate Unification**: Fixed Z-axis floating gaps and implemented 1-based indexing consistency.

---

## 4. Hardware Script Monitor (HSM) 🚧 IN PROGRESS
*Professional visual feedback via Tauri v2 + Babylon.js.*

- [x] **Hybrid Data Factory**: Rust backend serving extracted GLB/DXF data to JS rendering engines.
- [x] **Babylon Viewer Integration**: Adopted high-level viewer for superior camera and environment handling.
- [x] **Multi-Viewport UX**: Unified light/dark mode and cross-viewport grid synchronization.
- [ ] **TSV Layer-Spanning Logic**: Coordinate Through-Silicon Vias across multi-die stacks (Targeting v0.1.7-final).
- [ ] **Spatial Net Selection**: Highlight nets across 3D/2D views by clicking geometry.

---

## 5. Remaining v0.1.7 Tasks
- [ ] **Fiberglass Preset**: Fine-tune anisotropy rotation for standard FR4 weave simulation.
- [ ] **TSV Keep-Out Zones (KOZ)**: Integrate silicon-via stress regions into DRC validation.
- [ ] **Final Build Performance Audit**: Ensure sub-second build times are maintained with new triangulation logic.
