# Roadmap 01 — Database & Spatial Foundation

**Read:** Core-System-Architecture.md, Architectural-Specification.md, Engineering-Specification.md

---

## 1.1 Entity Graph — Canonical Database

- [x] **Design the Entity Graph schema** — `EntityGraph` facade struct wrapping `NetlistArena` + `SceneGraph` + `DynamicSpatialIndex` with unified access methods
- [x] **Implement stable cryptographically unique EntityIds** — SHA-256 based `EntityId([u8; 32])` with `generate(type, path, parent)` and typed wrappers
- [x] **Define ID types:** `ComponentGraphId`, `PinGraphId`, `NetGraphId`, `RouteGraphId`, `GeometryGraphId`, `JunctionGraphId` — newtype wrappers over `EntityId`
- [x] **Store component geometry once** — `ComponentStamp` as OBVH in local coordinate space; `stamp_parser.rs` bridges `BakedComponent` → `ComponentStamp`
- [ ] **Implement incremental graph updates** — on component edit, only direct node + immediate children are modified/flagged dirty *(deferred to pipeline migration)*
- [x] **Wire Entity Graph as master registry** — `EntityGraph` provides `netlist()`, `scene()`, `spatial()` accessors + `register_component()`, `register_net()`, `get_net_instances()`, `rebuild_spatial_index()`

## 1.2 Zero-Stamping Scene Graph (ComponentStamp + ComponentInstance)

- [x] **Define `ComponentStamp` struct** — local-coordinate OBVH with `local_bbox`, `local_obb_children`, `local_aabb_children`, `local_polygons`, `local_pin_offsets`
- [x] **Implement stamp parsing** — `bake_stamp()` converts `BakedComponent` → `ComponentStamp` with per-pin AABB children, polygon extraction, pin offsets; `register_baked_stamps()` batch-registers with dedup; `stamp_pin_global_position()` computes global pin coords via `FixedTransform2D`
- [x] **Define `ComponentInstance` struct** — `stamp_id`, `FixedTransform2D`, `net_bindings`, pre-transformed `global_bbox`, `global_obb_children`, `global_aabb_children`
- [x] **Implement forward-transform at placement** — `transform_bbox_to_global()` and `transform_obb_to_global()` called once in `ComponentInstance::new()`
- [x] **Implement `test_collision_global()`** — SAT-based OBB collision via `OrientedBoundingBox::contains_point()`, fast AABB for Manhattan; Jordan curve fallback only for non-standard shapes
- [ ] **Verify memory reduction** — 100M-transistor design: ~500 stamps (few KB each) + ~100K instances = <80 MB (down from 1.6 GB voxel) *(deferred — needs test design)*

**Additional implemented:**
- [x] **`SceneGraph` registry** — stamp + instance storage with FxHashMap name index, `register_stamp()`, `place_instance()`, `estimate_memory_bytes()`
- [x] **`OrientedBoundingBox`** — SAT-based `contains_point()`, `to_aabb()` enclosure

## 1.3 FixedTransform2D — Deterministic Integer Transforms

- [x] **Implement `FixedTransform2D`** — pure i128 intermediate arithmetic, `SCALE_FACTOR = 10^9`; no floating-point in core path
- [x] **Implement `transform_point(x, y)`** — promote to i128, apply rotation, divide by SCALE_FACTOR, cast back to i64
- [x] **Implement `transform_bbox(bbox)`** — `transform_bbox_2d()` transforms all 4 corners, recomputes axis-aligned bounding box
- [x] **Implement trigonometric lookup table** — fixed cos/sin values for 0, 45, 90, 135, 180, 225, 270, 315 degrees via `lookup_trig()`
- [x] **Forbid `glam` in core path** — `glam` only allowed in GLB viewport renderer (non-deterministic visualization only)

**Additional implemented:**
- [x] **`BoundingBox2D`** — 2D axis-aligned bounding box with `contains_point`, `intersects`, `expand`, `union`, `width`, `height`
- [x] **`FixedTransform2D::then()`** — transform composition
- [x] **`FixedTransform2D::inverse()`** — inverse transform

## 1.4 Hybrid Spatial Index (rstar + geo-static)

- [x] **Integrate `rstar` crate** — dynamic R*-tree for macro-placement during floorplanning
- [x] **Implement `RTreeObject` for `IndexedSegment`** — envelope with trace width inflation for physical volume
- [x] **Implement `StaticLayerIndex`** — sorted-array + binary search for per-layer static obstacles after partition
- [x] **Compile obstacle geometry into flat structures** — `StaticLayerIndex::build()` compiles segments into sorted contiguous memory
- [x] **Verify O(log N) query performance** — `DynamicSpatialIndex::query_bbox()` via rstar envelope queries; `StaticLayerIndex::query_bbox()` via `partition_point` binary search
- [x] **Ensure zero memory bloat** — indices are volatile RAM, discarded after build completes

**Additional implemented:**
- [x] **`DynamicSpatialIndex`** — full API: `insert`, `remove`, `query_bbox`, `query_radius`, `query_nearest`, `clear`
- [x] **`query_overlapping_segments()`** — clearance-expanded bbox query with same-net exclusion
- [x] **`PointDistance` impl** — enables `nearest_neighbor` queries on IndexedSegment

## 1.5 Deprecation of Voxel Grid

- [x] **Deprecate `visible_plane` hash-map chunk allocator** — `#[deprecated]` on `set_occupied` and `stamp_cylinder`
- [ ] **Remove dense `VoxelGrid` allocations** — memory allocation for spatial grid tracking drops to zero on startup *(deprecated methods still called in live routing paths — pending pipeline migration)*
- [ ] **Delete voxel-based A* neighbors loop** in pathfinder *(annotated with deprecation comment, pending pipeline migration)*

**Additional completed:**
- [x] **Delete `voxel_stamps/` directory** — 4 files, ~1,380 lines of dead code removed (stamp.rs, library.rs, process_node.rs, tech_mapper.rs)
- [x] **Delete dead collision functions** — removed `is_voxel_available` and `mark_route_occupied` from collision_detection.rs
- [x] **Clean re-exports** — removed all dead pub use statements from lib.rs and geometry_router/mod.rs

---

## Summary

| Section | Completed | Remaining |
|---------|-----------|-----------|
| 1.1 Entity Graph | 5/6 | Incremental graph updates (deferred) |
| 1.2 Scene Graph | 5/6 | Memory verification (deferred) |
| 1.3 FixedTransform2D | 5/5 | **Complete** |
| 1.4 Spatial Index | 6/6 | **Complete** |
| 1.5 Voxel Deprecation | 4/6 | Full removal of live callers (pipeline migration) |
| **Total** | **25/29** | 4 remaining — all deferred to pipeline migration or benchmarking |

**Files created:** `geometry/transform.rs`, `geometry/entity_ids.rs`, `geometry_router/spatial_index.rs`, `geometry_router/geo_static_index.rs`, `geometry_router/scene_graph.rs`, `geometry_router/stamp_parser.rs`, `geometry_router/entity_graph.rs`

**Files deleted:** `voxel_stamps/` (4 files)

**Lines removed:** ~1,380 (dead voxel_stamps code + dead collision functions)

### Remaining 4 tasks (all deferred):
1. **Incremental graph updates** — dirty flagging optimization, implement when routing pipeline needs it
2. **Memory reduction verification** — benchmark with real test design
3. **Remove live `set_occupied` callers** — will happen naturally as routing pipeline migrates to Entity Graph
4. **Delete voxel A* neighbors loop** — will happen when pathfinder switches to spatial index queries
