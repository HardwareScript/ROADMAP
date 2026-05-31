# v0.1.7 Manufacturing Fidelity Rethink Roadmap

**Goal**: Transition the Hardware Script compiler from a functional prototype to a professional-grade EDA tool by eliminating voxel approximations and visual artifacts in manufacturing exports.

---

## 1. Visual & Rendering Fidelity (GLB/3D)

### 1.1 Manifold Mesh Culling (Anti Z-Fighting)
- [ ] Implement precedence-based face culling in `hwc-export::scene_graph`.
- [ ] Ensure component pads (higher precedence) "own" the shared surface with copper rails.
- [ ] Eliminate messy triangular clipping and striping on top/bottom faces.

### 1.2 The Cylindrical Handshake
- [ ] Update `SceneGraph` mesh generation to support true circular voids.
- [ ] Implement Quadrant Partition Triangulation for smooth circular cutouts.
- [ ] Replace square "Minecraft-style" bounding box cutouts with analytic cylinders.

---

## 2. Manufacturing Layout Fidelity (DXF/CAD)

### 2.1 Layer De-Pollution
- [ ] Refactor DXF exporter to map physical slices back to logical named layers (Top, Inner1, Inner2, Bottom).
- [ ] Create a dedicated, global `DRILL` layer for CNC coordinate export.
- [ ] Eliminate the "15-layer mechanical mess" in the CAD layer tree.

### 2.2 Analytic Primitives (Arcs & Circles)
- [ ] Use `AnalyticTrace` definitions to export true AutoCAD `CIRCLE` and `ARC` entities.
- [ ] Eliminate segmented, blocky polygons for annular rings and drill holes.

---

## 3. Electrical Extraction Fidelity (SPICE)

### 3.1 Device Classification Hardening
- [ ] Update `device_extractor.rs` to filter out registered IC packages from the MOSFET sweep.
- [ ] Only extract MOSFETs/Transistors when active semiconductor regions (Diffusion/Poly) are physically present.
- [ ] Eliminate "ghost" MOSFETs with shorted terminals (0 0 0 0) in the SPICE netlist.

---

## 4. Manufacturing Documentation Fidelity (BOM/CSV)

### 4.1 Virtual Anchor Filtering
- [ ] Update BOM extractor to filter out internal routing anchors (e.g., `Pour(Copper)`, `Contact()`).
- [ ] Ensure the Bill of Materials only contains physically orderable components.
- [ ] Eliminate duplicate entries for copper pours.

---

## Verification Gauntlet
- [ ] Verify U1/U2 pad rendering in GLB without Z-fighting.
- [ ] Verify circular drill holes in DXF (AutoCAD compatible).
- [ ] Verify SPICE netlist contains only logical components and true extracted silicon.
- [ ] Verify BOM count matches physical component count (e.g., 2 ICs, not 31 items).
