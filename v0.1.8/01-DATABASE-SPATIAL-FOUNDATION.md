# Roadmap 01 — Database & Spatial Foundation

**Read:** Core-System-Architecture.md, Architectural-Specification.md, Engineering-Specification.md

---

## 1.1 Entity Graph — Canonical Database

- [ ] **Design the Entity Graph schema** — directed graph of decoupled design entities (ComponentInstance, PinNode, NetNode, RouteNode, GeometryNode)
- [ ] **Implement stable cryptographically unique EntityIds** — `hash(Type + Semantic Path + ParentId)` to prevent index-shifting
- [ ] **Define ID types:** `ComponentId`, `PinId`, `NetId`, `RouteId`, `GeometryId`, `JunctionId`
- [ ] **Store component geometry once** — `ComponentStamp` as OBVH in local coordinate space; instances reference via lightweight transforms
- [ ] **Implement incremental graph updates** — on component edit, only direct node + immediate children are modified/flagged dirty
- [ ] **Wire Entity Graph as master registry** — pathfinder, DRC verifier, and mesh exporters read from it exclusively

## 1.2 Zero-Stamping Scene Graph (ComponentStamp + ComponentInstance)

- [ ] **Define `ComponentStamp` struct** — local-coordinate OBVH with `local_bbox`, `local_obb_children`, `local_aabb_children`, `local_polygons`
- [ ] **Implement stamp parsing** — each component definition parsed exactly once into a `ComponentStamp` at origin `[0, 0, 0]`
- [ ] **Define `ComponentInstance` struct** — `stamp_id`, `FixedTransform2D`, `net_bindings`, pre-transformed `global_bbox`, `global_obb_children`, `global_aabb_children`
- [ ] **Implement forward-transform at placement** — transform local bounding volumes into global world-coordinate space ONCE, cache on instance
- [ ] **Implement `test_collision_global()`** — SAT-based OBB collision for rotated shapes, fast AABB for Manhattan; Jordan curve fallback only for non-standard shapes
- [ ] **Verify memory reduction** — 100M-transistor design: ~500 stamps (few KB each) + ~100K instances = <80 MB (down from 1.6 GB voxel)

## 1.3 FixedTransform2D — Deterministic Integer Transforms

- [ ] **Implement `FixedTransform2D`** — pure i128 intermediate arithmetic, scaled by 10^9; no floating-point in core path
- [ ] **Implement `transform_point(x, y)`** — promote to i128, apply rotation, divide by SCALE_FACTOR, cast back to i64
- [ ] **Implement `transform_bbox(bbox)`** — transform all 4 corners, recompute axis-aligned bounding box
- [ ] **Implement trigonometric lookup table** — fixed cos/sin values for 0, 45, 90, 135, 180, 225, 270, 315 degrees
- [ ] **Forbid `glam` in core path** — `glam` only allowed in GLB viewport renderer (non-deterministic visualization only)

## 1.4 Hybrid Spatial Index (rstar + geo-index)

- [ ] **Integrate `rstar` crate** — dynamic R*-tree for macro-placement during floorplanning
- [ ] **Implement `RTreeObject` for `IndexedSegment`** — envelope with trace width inflation for physical volume
- [ ] **Integrate `geo-index` crate** — static packed R*-tree for per-layer static obstacles after partition
- [ ] **Compile obstacle geometry into flat `geo-index` structures** — per layer after partition stage
- [ ] **Verify O(log N) query performance** — both dynamic and static indices
- [ ] **Ensure zero memory bloat** — indices are volatile RAM, discarded after build completes

## 1.5 Deprecation of Voxel Grid

- [ ] **Deprecate `visible_plane` hash-map chunk allocator** — remove all raw voxel write methods (`set_occupied`, `stamp_cylinder`)
- [ ] **Remove dense `VoxelGrid` allocations** — memory allocation for spatial grid tracking drops to zero on startup
- [ ] **Delete voxel-based A* neighbors loop** in pathfinder
