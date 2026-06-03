# v0.1.7 2D Polygon Unioning Roadmap (Strategy A)

**Status**: Planning / Implementation Ready  
**Goal**: Implement "Physical Reality First" by merging all copper elements on the same net/layer into single, manifold 3D meshes using 2D polygon clipping.

---

## Phase 1: Infrastructure & Dependencies

- [ ] **1.1 Add Clipper2 & Earcutr**: Update `hwc-export/Cargo.toml` with `clipper2` (for boolean operations) and `earcutr` (for triangulation).
- [ ] **1.2 Create Geometry Module**: Implement `hwc-export/src/geometry_union.rs` for polyline conversion helpers.
- [ ] **1.3 Implement Extrusion Engine**: Create `hwc-export/src/mesh_extrusion.rs` to handle 3D extrusion of complex polygons with holes.

## Phase 2: Primitive to Polygon Conversion

- [ ] **2.1 BBox to Polygon**: Implement `rect_to_path` for rectangular pours and traces.
- [ ] **2.2 Cylinder to Polygon**: Implement `circle_to_path` for via landing pads and cylindrical contacts.
- [ ] **2.3 Ribbon to Polygon**: Update `create_extruded_ribbon` to optionally return 2D contours for unioning.

## Phase 3: The Unioning Pipeline (hwc-export)

- [ ] **3.1 Cluster Collection**: Update `add_substrate_with_net_clustering` to collect all 2D contours for a given (net, material, z-range).
- [ ] **3.2 Clipper Weld**: Perform `Union` operation on all collected contours for each cluster.
- [ ] **3.3 Manifold Extrusion**: Replace individual mesh generation calls with a single call to the new extrusion engine for the unified result.

## Phase 4: Verification & Manufacturing Alignment

- [ ] **4.1 Visual Verification**: Confirm zero Z-fighting in GLB exports for complex trace-to-via intersections.
- [ ] **4.2 Manifold Audit**: Verify that exported GLB files contain no internal faces or overlapping volumes.
- [ ] **4.3 3D Print Readiness**: Test exported meshes in 3D slicer software to ensure clean toolpath generation.

---

## References
- [2D-POLYGON-UNIONING-IMPLEMENTATION.md](../../Docs/v0.1.7/2D-POLYGON-UNIONING-IMPLEMENTATION.md)
- [MANIFOLD-EXPORT-RETHINK.md](./MANIFOLD-EXPORT-RETHINK.md)
- [Z_FIGHTING_FIX.md](../../Docs/v0.1.7/Z_FIGHTING_FIX.md)
