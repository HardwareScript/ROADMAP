# v0.1.7 Manufacturing Fidelity Rethink Roadmap

**Goal**: Transition the Hardware Script compiler from a functional prototype to a professional-grade EDA tool by eliminating voxel approximations and visual artifacts in manufacturing exports.

---

## 1. Visual & Rendering Fidelity (GLB/3D)

### 1.1 Manifold Mesh Culling (Anti Z-Fighting)
- [x] Implement precedence-based face culling in `hwc-export::scene_graph`.
- [x] Ensure component pads (higher precedence) "own" the shared surface with copper rails.
- [x] Eliminate messy triangular clipping and striping on top/bottom faces.

### 1.2 The Cylindrical Handshake
- [x] Update `SceneGraph` mesh generation to support true circular voids.
- [x] Implement Quadrant Partition Triangulation for smooth circular cutouts.
- [x] Replace square "Minecraft-style" bounding box cutouts with analytic cylinders.

---

## 2. Manufacturing Layout Fidelity (DXF/CAD)

### 2.1 Layer De-Pollution
- [x] Refactor DXF exporter to map physical slices back to logical named layers (Top, Inner1, Inner2, Bottom).
- [x] Create a dedicated, global `DRILL` layer for CNC coordinate export.
- [x] Eliminate the "15-layer mechanical mess" in the CAD layer tree.
- [x] **Audit Fix**: Linked `StackupManager` to DXF exporter to use semantic profile names instead of hardcoded Z-division.

### 2.2 Analytic Primitives (Arcs & Circles)
- [x] Use `AnalyticTrace` definitions to export true AutoCAD `CIRCLE` and `ARC` entities.
- [x] Eliminate segmented, blocky polygons for annular rings and drill holes.

---

## 3. Electrical Extraction Fidelity (SPICE)

### 3.1 Device Classification Hardening
- [x] Update `device_extractor.rs` to filter out registered IC packages from the MOSFET sweep.
- [x] Only extract MOSFETs/Transistors when active semiconductor regions (Diffusion/Poly) are physically present.
- [x] Eliminate "ghost" MOSFETs with shorted terminals (0 0 0 0) in the SPICE netlist.
- [x] **Audit Fix**: Explicitly filter `IC_Package` and `Component` types from the `M` prefix transistor extraction.

---

## 4. Manufacturing Documentation Fidelity (BOM/CSV)

### 4.1 Virtual Anchor Filtering
- [x] Update BOM extractor to filter out internal routing anchors (e.g., `Pour(Copper)`, `Contact()`).
- [x] Ensure the Bill of Materials only contains physically orderable components.
- [x] Eliminate duplicate entries for copper pours.

### 4.2 Manufacturing Drill Registration (Excellon)
- [x] **Audit Fix**: Register all placed contacts/vias in `space.vias` during the IR placement phase to ensure non-empty `.drl` files.

---

## Verification Gauntlet
- [x] Verify U1/U2 pad rendering in GLB without Z-fighting.
- [x] Verify circular drill holes in DXF (AutoCAD compatible).
- [x] Verify SPICE netlist contains only logical components and true extracted silicon.
- [x] Verify BOM count matches physical component count (e.g., 2 ICs, not 31 items).
- [x] **Audit Verification**: Confirmed `.drl` file contains all via/drill hits.
- [x] **Audit Verification**: Confirmed DXF layers match stackup profile names (e.g., `GndPlane_COPPER`).
