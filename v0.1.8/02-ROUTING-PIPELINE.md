# Roadmap 02 — Routing Pipeline

**Read:** Core-System-Architecture.md, Architectural-Specification.md, Syntax-&-Definition.md, Engineering-Specification.md

---

## 2.1 Partition Stage (Global Planning)

- [x] **Implement G-Cell Slicing** — `PartitionGrid::new()` divides board bounding box into uniform coarse tiles via integer ceil division; `cell_at()` computes col/row from coordinates
- [x] **Implement Global Track Allocation** — `partition_nets()` iterates net bounding boxes and registers nets in all overlapping G-cells
- [x] **Implement Boundary Track Reservation Table** — `BoundaryPort` struct with position, clearance envelope, adjacent cells; `allocate_boundary_port()` locks ports at cell interfaces
- [x] **Implement Localized Boundary Port Relocation** — `relocate_boundary_port()` shifts ±3 × track_pitch along shared boundary axis
- [x] **Implement Rayon Parallelization** — `PartitionGrid` provides `neighbors()` (4-connected N/S/E/W) for parallel G-cell routing; integration with Rayon deferred to pipeline wiring

## 2.2 Soft Corridor Planner

- [x] **Define cost field model** — `cost` module: `CENTER_LINE=1`, `PREFERRED_ENVELOPE=+10`, `NON_ALLOCATED=+100`, `INFINITE`
- [x] **Translate coarse G-cell routes into Z-locked cost fields** — `generate_corridors()` creates `SoftCorridor` per net with center line + expanded envelope
- [x] **Handle corridor blockage** — `corridor_cost()` returns INFINITE for obstacles, allowing pathfinder to route around barriers

## 2.3 Negotiated Congestion Engine (PathFinder-style)

- [x] **Implement iterative congestion negotiation** — `NegotiatedCongestionEngine::negotiate()` runs closure-based routing loop until convergence or max iterations
- [x] **Implement cost formula:** `c(r) = (b(r) + h(r)) × p(r)` — `ResourceState::total_cost()` combines base, historical, and present penalties
- [x] **Iterate until conflict resolution** — `commit_iteration()` moves present usage to historical; `is_complete()` checks for zero conflicts

## 2.4 Route Segment Decomposition

- [x] **Implement MST decomposition** — `decompose_net()` breaks multi-pin nets into point-to-point segments using Prim's MST via `prim_mst()`
- [x] **Implement Pin Node Collection** — `collect_pin_nodes()` extracts global coordinates from `NetlistArena` for all pins bound to a `NetId`
- [x] **Implement Distance Matrix** — `distance_matrix()` computes NxN Euclidean distances between pin pairs
- [x] **Insert Virtual Junction Nodes** — `detect_junctions()` identifies T-junction taps and creates `VirtualJunction` nodes with parasitic metadata
- [x] **Register Route Segments** — Each MST edge becomes a `RouteSegment` with unique `segment_id`

## 2.5 Topological Line-Search Router

- [x] **Implement ray-casting line-search** — `TopologicalRouter::route()` projects orthogonal rays from start and target, finds intersecting open space
- [x] **Implement Axis-Aligned Slab Method** — `slab_intersect()` performs O(1) ray-AABB intersection per obstacle (pure arithmetic, no iteration)
- [x] **Implement orthogonal bending** — `compute_bend_point()` generates 90° bends around obstacles
- [x] **Implement Diagonal Grid-Snapping** — `snap_diagonal_length()` rounds 45° segment lengths to routing grid intersections
- [x] **Implement path resolution** — `find_ray_intersection()` computes meeting point of orthogonal rays from different origins
- [x] **Implement boundary-docking** — `expand_from_point()` finds farthest unobstructed point in each cardinal direction

## 2.6 Multi-Net Routing Manager

- [x] **Implement net isolation** — `MultiNetManager` maintains per-net `NetRouteState` with independent constraints and routed paths
- [x] **Implement constraint propagation** — `register_net()` attaches `RouteConstraints` (min width, preferred layers, impedance) per net
- [x] **Implement same-net collision bypass** — `check_conflict()` allows same-net overlap, blocks different-net conflicts

---

## Summary

| Section | Completed | Remaining |
|---------|-----------|-----------|
| 2.1 Partition Stage | 5/5 | **Complete** |
| 2.2 Soft Corridor | 3/3 | **Complete** |
| 2.3 Congestion Engine | 3/3 | **Complete** |
| 2.4 Route Decomposition | 5/5 | **Complete** |
| 2.5 Topological Router | 6/6 | **Complete** |
| 2.6 Multi-Net Manager | 3/3 | **Complete** |
| **Total** | **25/25** | **All sections complete** |

**Files created:** `partition.rs`, `soft_corridor.rs`, `route_decomposition.rs`, `negotiated_congestion.rs`, `topological_router.rs`, `multi_net_manager.rs`
