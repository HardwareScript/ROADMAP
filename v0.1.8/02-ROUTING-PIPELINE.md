# Roadmap 02 — Routing Pipeline

**Read:** Core-System-Architecture.md, Architectural-Specification.md, Syntax-&-Definition.md, Engineering-Specification.md

---

## 2.1 Partition Stage (Global Planning)

- [ ] **Implement G-Cell Slicing** — divide board bounding box into uniform coarse tiles (e.g., 10mm x 10mm)
- [ ] **Implement Global Track Allocation** — plan paths of all nets across G-cell boundaries, treating each interface as a virtual port
- [ ] **Implement Boundary Track Reservation Table** — lock coordinate points where each net crosses G-cell boundaries as Immutable Interface Ports with clearance envelopes
- [ ] **Implement Localized Boundary Port Relocation** — shift port +/-3*track_pitch along shared boundary; invalidate only 2 adjacent G-cells in Salsa cache
- [ ] **Implement Rayon Parallelization** — spawn independent detailed routers inside each G-cell concurrently after interface ports are locked

## 2.2 Soft Corridor Planner

- [ ] **Define cost field model** — center line (cost 1), preferred envelope (+10), non-allocated region (+100)
- [ ] **Translate coarse G-cell routes into Z-locked cost fields** over the R*-Tree
- [ ] **Handle corridor blockage** — pathfinder routes around barrier through non-allocated region, snaps back after obstacle

## 2.3 Negotiated Congestion Engine (PathFinder-style)

- [ ] **Implement iterative congestion negotiation** — all nets route simultaneously, overlapping allowed in first iteration
- [ ] **Implement cost formula:** `c(r) = (b(r) + h(r)) x p(r)` — base cost, historical congestion, present congestion penalty
- [ ] **Iterate until conflict resolution** — costs spike to infinity, forcing unique non-overlapping paths

## 2.4 Route Segment Decomposition

- [ ] **Implement MST decomposition** — break multi-pin nets into discrete point-to-point routing jobs using Prim's/Kruskal's
- [ ] **Implement Pin Node Collection** — extract global coordinates of all pins bound to active NetId
- [ ] **Implement Distance Matrix** — complete graph with Euclidean distances between pin pairs
- [ ] **Insert Virtual Junction Nodes** — three-port entities for T-junction taps with stable JunctionId
- [ ] **Register Route Segments** — each MST edge becomes independent `RouteSegment` with own `RouteId`

## 2.5 Topological Line-Search Router

- [ ] **Implement ray-casting line-search** — project orthogonal search rays (horizontal/vertical) along continuous coordinates
- [ ] **Implement Axis-Aligned Slab Method** — single O(log N) ray-AABB intersection query per obstacle over flat-packed `geo-index`
- [ ] **Implement orthogonal bending** — clean 90 or 135 degree bends when ray hits obstacle
- [ ] **Implement Diagonal Grid-Snapping** — `L_snapped = round(N x track_pitch / sin(45))` for 45 degree segments
- [ ] **Implement path resolution** — when ray from start and ray from target intersect in open space, path is found
- [ ] **Implement boundary-docking** — pathfinder targets outer bounding box edge of cardinal port; pad interior marked as impenetrable

## 2.6 Multi-Net Routing Manager

- [ ] **Implement net isolation** — each net gets distinct G-cells and soft corridor cost fields during Partition
- [ ] **Implement constraint propagation** — attach physical rules (min width, preferred layers, target impedance) as metadata to route segments
- [ ] **Implement same-net collision bypass** — allow same-net traces to share pin bboxes; enforce strict clearance against different-net obstacles
