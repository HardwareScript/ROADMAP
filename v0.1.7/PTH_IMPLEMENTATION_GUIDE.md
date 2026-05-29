# Physically Accurate Plated Through-Hole (PTH) Implementation Guide

Based on an external audit and industry standards, this document outlines the corrected "Additive Surface + Subtractive Hole" model and the solution to the "Triangulation Star" problem.

## 1. The Physical Anatomy

A standard PTH consists of three distinct geometric components that do NOT overlap in the same Z-space:

*   **Substrate (Laminate/FR4)**:
    *   **Action**: Subtractive.
    *   **Geometry**: A solid box with a hole of `Drill Diameter + 1μm` (Epsilon Offset) passing through it.
    *   **Partitioning**: Uses "Donut-to-Square" grid partitioning to avoid long, thin triangles.
*   **Surface Copper (Pads / Annular Rings)**:
    *   **Action**: Additive.
    *   **Geometry**: A thin cylinder sitting *on top* or *at the bottom* of the substrate.
    *   **Inner Hole**: The pad itself has a hole matching the `Drill Diameter`.
*   **Plating (The Tube)**:
    *   **Action**: Additive.
    *   **Geometry**: A hollow tube lining the hole.
    *   **Z-Range**: Spans `Z_max - 1μm` to prevent Z-fighting with the pad surfaces.

---

## 2. Technical Challenges & Solutions

### A. The "Triangulation Star" Problem
*   **Issue**: Naive "Fan" triangulation connects a central circle to the board's outer corners, creating long, thin triangles. This causes normal inversion (black artifacts) and GPU flickering.
*   **Fix**: **Quadrant Partition Method**. We divide the area around the hole into a local 3x3 grid (8 outer quads, 1 central hole zone). Only the central zone is triangulated to the circle.

### B. Z-Fighting (Copper Piercing)
*   **Issue**: Co-planar surfaces (FR4 top and Copper Pad bottom) cause flickering.
*   **Fix**: Apply a **1-micron Epsilon Offset**.
    *   Hole in FR4 is `Radius + 1μm`.
    *   Plating Tube height is slightly reduced or offset from the pads.

### C. "Void" Material Redundancy
*   **Issue**: Rendering "Void" objects creates "inner rings" and visual clutter.
*   **Fix**: The exporter MUST explicitly skip any layer or cutout with the "Void" material. The geometry is already carved into the FR4 mesh.

---

## 3. Implementation Checkboxes

### A. Compiler Layer (`hwc-compiler`)
- [x] **Refactor `place_contact`**: Stop drilling pad-sized holes into the substrate.
- [x] **Standardized Drilling**: Only drill the `drill_diameter` through the substrate Z-span.
- [x] **The "Butler" Rule**: Automate the addition of SubstrateCutout, Plating Tube, and Pad Donuts from a single `add contact` call.

### B. Engine Layer (`hwc-engine`)
- [x] **Unified Statements**: Preserved textual order of substrates and components to support `last` keyword and stacking.

### C. Exporter Layer (`hwc-export`)
- [ ] **Quadrant Partitioning**: Refactor `create_box_with_holes_mesh` to use the 3x3 grid method.
- [ ] **Epsilon Offsets**: Implement the 1μm offset in `create_box_with_holes_mesh` and `create_tube_mesh`.
- [ ] **Void Skipping**: Ensure `substrate.rs` skips rendering "Void" materials.

---

## 4. Handshake Verification

1.  **DXF**: Must show a Circle (Drill) and a Circle (Annular Ring).
2.  **Drill File**: Must show a single hit at the via coordinate.
3.  **GLB (3D)**: Must show a clean hole in the green FR4, lined with a brown copper tube, with no star-pattern lines or black artifacts.
