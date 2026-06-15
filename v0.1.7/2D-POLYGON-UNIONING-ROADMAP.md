# v0.1.7 2D Polygon Unioning Roadmap (Strategy A)

**Status**: Planning / Implementation Ready  
**Goal**: Implement "Physical Reality First" by merging all copper elements on the same net/layer into single, manifold 3D meshes using 2D polygon clipping.

---

## Phase 1: Infrastructure & Dependencies

- [x] **1.1 Add Clipper2-Rust & Earcut**: Update `hwc-export/Cargo.toml` with `clipper2-rust` (pure Rust port) and `earcut` (GeoRust zero-allocation triangulator).
- [x] **1.2 Create Geometry Module**: Implement `hwc-export/src/geometry_union.rs` for polyline conversion helpers.
- [x] **1.3 Implement Extrusion Engine**: Create `hwc-export/src/mesh_extrusion.rs` to handle 3D extrusion of complex polygons with holes using optimized `earcut`.

## Phase 2: Primitive to Polygon Conversion

- [x] **2.1 BBox to Polygon**: Implement `rect_to_path` for rectangular pours and traces.
- [x] **2.2 Cylinder to Polygon**: Implement `circle_to_path` for via landing pads and cylindrical contacts.
- [x] **2.3 Ribbon to Polygon**: Update substrate logic to use 2D contours for unioning instead of legacy ribbons.

## Phase 3: The Unioning Pipeline (hwc-export)

- [x] **3.1 Cluster Collection**: Update `add_substrate_with_net_clustering` to collect all 2D contours for a given (net, material, z-range).
- [x] **3.2 Clipper Weld**: Perform `Union` operation with **Non-Zero Winding Rule** to ensure solid manifolds for overlapping shapes.
- [x] **3.3 Manifold Extrusion**: Replace individual mesh generation calls with optimized `earcut` extrusion for the unified result.

## Phase 4: Verification & Manufacturing Alignment

- [x] **4.1 Visual Verification**: Confirmed zero Z-fighting in GLB exports for complex intersections in `strategy_a_union_test.hw`.
- [x] **4.2 Manifold Audit**: Verified that exported GLB files contain clean, unified islands.
- [x] **4.3 Performance Handshake**: Integrated `clipper2-rust` and `earcut` for maximum compilation and runtime efficiency.

---

## Phase 5: 4-Mode Shape Design System

- [x] **5.1 AST Extensions**: Added CsgExpression, CsgPrimitive, ShapeGenerator, GeometryBlock, GeometryStatement types to `crates/hwc-parser/src/ast/shape.rs`
- [x] **5.2 Parser Extensions**: Added geometry block parsing, for-loop parameterization, let bindings, Point(x,y) expressions, CSG expressions, procedural generator calls to `crates/hwc-parser/src/parser/definitions/shape.rs`
- [x] **5.3 Shape Generators**: Implemented star_generator_contour() and gear_generator_contour() in `crates/hwc-compiler/src/shape_generators.rs`
- [x] **5.4 Via Integration**: Implemented evaluate_shape_points(), evaluate_nm_expr(), evaluate_pure_math() (with paren fix) in `crates/hwc-compiler/src/auto_via_inserter/library.rs`
- [x] **5.5 Multi-Shape Stitching**: Multiple shapes in one space via custom contacts with collision detection
- [x] **5.6 Parameterized Shapes**: All via shapes use width parameter to scale to via diameter

### References
- [SHAPE-SYSTEM-ARCHITECTURE.md](../../Docs/v0.1.7/SHAPE-SYSTEM-ARCHITECTURE.md)

## References
- [2D-POLYGON-UNIONING-IMPLEMENTATION.md](../../Docs/v0.1.7/2D-POLYGON-UNIONING-IMPLEMENTATION.md)
- [MANIFOLD-EXPORT-RETHINK.md](./MANIFOLD-EXPORT-RETHINK.md)
- [Z_FIGHTING_FIX.md](../../Docs/v0.1.7/Z_FIGHTING_FIX.md)
