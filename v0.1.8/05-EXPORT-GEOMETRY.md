# Roadmap 05 — Export & Geometry Refinement

**Read:** Core-System-Architecture.md, PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md, Architectural-Specification.md, Engineering-Specification.md

---

## 5.1 Stackup Slicing Engine

- [ ] **Implement stackup boundary query** — retrieve ordered list of Z-intervals from StackupManager
- [ ] **Implement slicing intersection** — for each geometric entity, intersect Z-span with layer intervals; project 2D polygon onto matching layer
- [ ] **Implement shape registration** — append 2D polygon to layer's local shape registry; no new layer object allocated

## 5.2 2D Polygon Co-Unioning (Copper Welder)

- [ ] **Implement bucketing** — group geometries by NetId and MaterialId within each physical copper layer
- [ ] **Implement polyline conversion:**
  - Rectangles to 4-point paths via `rect_to_path`
  - Circles/Cylinders to 64-sided regular polygons via `circle_to_path`
  - Custom shapes via AST math solver evaluation
- [ ] **Integrate `clipper2-rust`** — boolean union under Non-Zero Winding Rule
- [ ] **Verify overlapping boundaries dissolved** — single continuous outer contour with hollow internal holes

## 5.3 Boundary Canonicalization

- [ ] **Implement collinear edge merging** — discard vertex B if A, B, C are collinear (within 0.001 degree tolerance); up to 70% vertex reduction
- [ ] **Implement sliver cleanup** — remove microscopic self-intersecting polygon loops
- [ ] **Implement winding normalization** — outer contours CCW, inner hole contours CW

## 5.4 Geometry Refinement Engine

- [ ] **Implement 2D Clipper union** — group copper by layer/net, weld using `clipper2-rust` with Non-Zero Winding Rule
- [ ] **Implement boundary canonicalization pass** — merge collinear edges, remove slivers, normalize winding
- [ ] **Integrate `earcut` triangulation** — zero-allocation flat-array ear-clipping on clean unioned contours only at GLB export boundary
- [ ] **Verify 3-5x faster tessellation** than heavier tessellators on clean input

## 5.5 Export Isolation Layer

- [ ] **Enforce strict boundary** — Entity Graph stores only pristine i64 vector coordinates; no mesh data, triangles, or rendering assets
- [ ] **Ensure LVS/DRC/BEM run on raw vector segments** — clipper2 welding and earcut triangulation occur strictly at final export boundary
- [ ] **Implement on-the-fly geometry generation** — triangulation executed only when writing GLB mesh or launching viewport; mesh data discarded from memory after export
- [ ] **Stream-serialize to standard formats** — `.glb` (3D visual), `.dxf` (2D layout), `.sp` (SPICE netlist), `.csv` (BOM)
