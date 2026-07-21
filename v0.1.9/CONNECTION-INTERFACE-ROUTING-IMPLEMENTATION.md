# Roadmap — Connection Interface Routing (CIR) Implementation

**Read:** `Docs/v0.1.9/Connection-Interface-Routing.md`

**Status:** PHASE 1 + PHASE 2.1-2.3 FULLY COMPLETE (excluding benchmarks)

**Source Documents:**
- Connection-Interface-Routing.md (v0.1.9)
- ROUTING-ENGINE-MODERNIZATION.md (v0.1.9)
- Spatial_Synthesis_Abstraction.md (v0.1.9)

---

## Overview

This roadmap implements the Connection Interface Routing (CIR) system, which adds a semantic abstraction layer above the existing port escape and routing infrastructure. CIR enables:

- Multi-interface component footprints (redundant contacts for power/ground)
- Interface capability tracking (current limits, bandwidth, thermal)
- Routing intent abstraction (clock trees, power meshes, differential pairs)
- Connection candidate optimization (selecting best interface pairs before routing)

**Key Philosophy:** Integration over replacement. CIR extends existing systems (`port_escape.rs`, EntityGraph, TopologicalRouter) rather than replacing them.

---

## Phase 1: Core Integration (Critical Path)

### 1.1 EntityGraph Extension for Interface Ownership

**Goal:** Add interface tracking to EntityGraph without breaking existing functionality

- [x] **Define `InterfaceId` type** — Add `#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)] pub struct InterfaceId(pub u32);` to `geometry_router/types.rs` or new `geometry_router/interface_types.rs`
- [x] **Add Interface Database to EntityGraph** — Add `interface_database: FxHashMap<InterfaceId, PhysicalInterface>` field to `EntityGraph` struct in `geometry_router/entity_graph/mod.rs`
- [x] **Implement InterfaceId allocation** — Add `next_interface_id: u32` counter to EntityGraph and `allocate_interface_id() -> InterfaceId` method
- [x] **Add component interface tracking** — Extend component metadata with `Vec<InterfaceId>` mapping logical pins to physical interfaces
- [x] **Implement interface registration** — Add `register_interface(&mut self, component_id: ComponentId, interface: PhysicalInterface) -> InterfaceId` method
- [x] **Implement interface queries** — Add `get_interface(&self, id: InterfaceId) -> Option<&PhysicalInterface>` and `get_component_interfaces(&self, component_id: ComponentId) -> &[InterfaceId]` methods
- [x] **Wire into existing component placement** — Call `register_interface()` during component stamp instantiation

**Files to modify:**
- `hwc-engine/src/geometry_router/entity_graph/mod.rs`
- `hwc-engine/src/geometry_router/entity_graph/types.rs`
- `hwc-engine/src/geometry_router/types.rs` (or create `interface_types.rs`)

**Success criteria:**
- EntityGraph can allocate and store InterfaceIds
- Components track their interfaces
- Existing tests still pass

### 1.2 Interface Type Definitions

**Goal:** Define core interface types with cached geometric properties

- [x] **Create `connection_interface.rs` module** — New file at `hwc-engine/src/geometry_router/connection_interface.rs`
- [x] **Define `Normal2D` type** — Fixed-point integer normal vector scaled by 10^9 for deterministic geometry
- [x] **Define `InterfaceGeometry` enum** — `Point(Point3D)`, `Edge { start, end }`, `Polygon(Vec<Point3D>)` variants
- [x] **Define `Orientation` enum** — `Derived`, `Explicit(Normal2D)`, `None` variants
- [x] **Define `InterfaceCapability` enum** — `CarryCurrent`, `SignalBandwidth`, `CarryHeat`, `OpticalCoupling` variants with constraint derivation
- [x] **Define `DerivedConstraint` enum** — `MinimumTraceWidth`, `MaximumTraceLength`, `ThermalViaRequired`, `None` variants
- [x] **Implement `InterfaceCapability::derive_constraint()`** — Convert capabilities to routing constraints (e.g., current → minimum width)
- [x] **Define `PhysicalInterface` struct** — Contains geometry, capabilities, and cached properties (normals, access regions, keepouts)
- [x] **Define `AccessRegion` struct** — Pre-computed approach zone with entry point, normal, and priority

**Files to create:**
- `hwc-engine/src/geometry_router/connection_interface/` (modular: mod.rs, types.rs, geometry.rs, capability.rs, access_region.rs, physical.rs, testing.rs)

**Files to modify:**
- `hwc-engine/src/geometry_router/mod.rs` (add module declaration)

**Success criteria:**
- All interface types compile
- Capabilities can derive constraints
- Types integrate with existing Point3D and BoundingBox

### 1.3 Integer Geometry Operations

**Goal:** Implement deterministic normal derivation for all geometry types

- [x] **Implement `integer_sqrt()`** — Deterministic Newton-Heron square root in `hwc-engine/src/geometry/math.rs` (may already exist)
- [x] **Implement `Normal2D::new()`** — Constructor with validation
- [x] **Implement `InterfaceGeometry::derive_normals()`** — Main normal derivation dispatch method
- [x] **Implement edge normal computation** — `compute_edge_normal()` for Edge geometry using perpendicular rotation
- [x] **Implement polygon normal computation** — Per-edge normal derivation using signed area method
- [x] **Add polygon normal test** — Verify normals point outward for convex and concave polygons
- [x] **Verify determinism** — Test that same geometry produces identical normals across multiple runs

**Files to create/modify:**
- `hwc-engine/src/geometry_router/geometry_math.rs` (created)
- `hwc-engine/src/geometry_router/connection_interface/geometry.rs` (impl block)

**Success criteria:**
- Normal derivation works for rectangles, octagons, and arbitrary polygons
- No `todo!()` placeholders remain
- Integer-only math, no floating-point operations
- Bit-identical results across platforms

### 1.4 Bridge to Existing Port Escape System

**Goal:** Integrate interfaces with existing CardinalPort/EdgeOffset system

- [x] **Study existing port_escape.rs** — Understand `CardinalPort`, `EdgeOffset`, `calculate_rect_escape()` API
- [x] **Add interface parameter to escape functions** — Extend `calculate_rect_escape()` to accept `InterfaceId` (optional initially)
- [x] **Implement geometry-based dispatch** — Create `calculate_interface_escape()` that routes to appropriate strategy:
  - Rectangle → `calculate_rect_escape()` with CardinalPort
  - Circle → `calculate_circular_escape()`
  - Polygon → New edge-based escape (select best edge segment)
- [x] **Reuse CardinalPort for rectangles** — No changes needed to existing rectangular escape logic
- [x] **Implement polygon edge selection** — Choose edge closest to routing target or with best clearance
- [x] **Add access region → port escape mapping** — Convert AccessRegion into CardinalPort/EdgeOffset parameters

**Files to create:**
- `hwc-engine/src/geometry_router/interface_escape.rs` (new dispatch layer)

**Success criteria:**
- Existing rectangular components use CardinalPort unchanged
- Polygon interfaces select appropriate edge for escape
- All existing port escape tests still pass

### 1.5 Access Region Generation

**Goal:** Pre-compute and cache interface approach zones

- [x] **Define `AccessRegion` struct** — Entry point, normal direction, approach zone polygon, priority
- [x] **Implement `AccessRegion::generate()`** — For edge geometry, compute stub extension and approach corridor
- [x] **Implement rectangular access regions** — Generate 4 access regions (one per cardinal direction) for rectangles
- [x] **Implement polygon access regions** — Generate one access region per polygon edge
- [x] **Implement circular access regions** — Generate radial access regions at key angles (N/S/E/W minimum)
- [x] **Add Minkowski inflation** — Expand access regions by `(trace_width / 2) + min_clearance`
- [x] **Store as immutable Arc** — Cache `Arc<Vec<AccessRegion>>` in PhysicalInterface to enable lock-free concurrent access
- [x] **Wire into interface creation** — Call `generate_access_regions()` once during interface registration

**Files to modify:**
- `hwc-engine/src/geometry_router/connection_interface/access_region.rs`
- `hwc-engine/src/geometry_router/connection_interface/physical.rs`

**Success criteria:**
- Access regions generated for all geometry types
- Regions pre-computed and cached (not regenerated on queries)
- Memory overhead acceptable (<100 bytes per interface)

### 1.6 Capability Constraint Validation

**Goal:** Enforce interface capabilities as routing constraints

- [x] **Wire capabilities into RoutingParams** — Extend existing `RoutingParams` struct with `interface_constraints: Vec<DerivedConstraint>`
- [x] **Implement constraint checking** — Validate trace width against `MinimumTraceWidth` from `CarryCurrent` capability
- [x] **Add constraint derivation to routing setup** — Call `capability.derive_constraint()` before routing starts
- [x] **Emit compile error on violation** — If `CarryCurrent(100mA)` but `min_trace_width` too narrow, fail with diagnostic
- [x] **Add capability to materials integration** — Query material current density from `MaterialRegistry`
- [x] **Test current density calculation** — Verify 100mA through 180nm trace with copper resistivity produces correct width requirement

**Files to modify:**
- `hwc-engine/src/geometry_router/pathfinding/types.rs` (extend RoutingParams)
- `hwc-engine/src/geometry_router/constraints.rs` (add validation)
- `hwc-compiler/src/compiler.rs` (emit diagnostics)

**Success criteria:**
- Capabilities enforce physical constraints
- Compile fails if trace too narrow for current
- Material properties used in calculations

---

## Phase 2: Optimization (Performance & Quality)

### 2.1 Connection Candidate Selection

**Goal:** Pre-select best interface pairs before routing to reduce search space

- [x] **Define `ConnectionCandidate` struct** — Source/sink InterfaceIds, estimated cost, via requirement, layer span
- [x] **Implement candidate generation** — For each terminal pair, enumerate all interface combinations
- [x] **Implement heuristic scoring** — Euclidean distance + via penalty + capability mismatch cost
- [x] **Implement candidate selection** — Sort by score and return top N candidates (e.g., 10)
- [x] **Wire into global routing** — Call `select_connection_candidates()` before pathfinding
- [x] **Add multi-interface component test** — Component with 4 VDD contacts routing to power mesh with 200 vias
- [ ] **Measure performance improvement** — Benchmark routing time reduction (target: 10-100× for redundant contact cases)

**Files to create:**
- `hwc-engine/src/geometry_router/connection_candidate.rs`

**Success criteria:**
- Router explores 10 candidates instead of 800 for 4×200 interface combinations
- Routing time significantly reduced for power mesh connections
- Quality metrics unchanged (same path quality, fewer candidates explored)

### 2.2 Routing Intent System

**Goal:** Enable intent-driven cost evaluation (Clock, PowerRail, Signal, DifferentialPair)

- [x] **Define `RoutingIntent` enum** — Clock, PowerRail, Signal, DifferentialPair, HighSpeed variants
- [x] **Add intent to PhysicalInterface** — Store routing intent hint per interface
- [x] **Implement intent-specific cost weights** — Clock avoids vias (50000 penalty), Power prefers vias (10 penalty)
- [x] **Add syntax support** — Parser support for `intent: Clock` in route property blocks
- [x] **Wire into PDK profiles** — Load intent cost weights from profile `cost_weights:` block via routing heuristics
- [x] **Update route declarations** — Support `route X to Y:` with `intent: <Name>` syntax
- [x] **Test intent behavior** — Verify clock routes avoid vias, power routes use vias liberally

**Files to create:**
- `hwc-engine/src/geometry_router/routing_intent.rs`

**Success criteria:**
- Clock nets route with minimal vias
- Power nets use vias freely
- Intent policy loaded from PDK profiles

### 2.3 Cost Composer with Enum Dispatch

**Goal:** Replace fixed penalty fields with extensible enum-based cost evaluation

- [x] **Define `CostEvaluator` enum** — GeometricMove, ViaTransition, Direction, Thermal, Electromigration, Crosstalk, ReferenceVoid
- [x] **Implement `CostEvaluator::evaluate()`** — Inline enum match dispatch (no dynamic trait objects)
- [x] **Define `CostComposer` struct** — Uses `SmallVec<[CostEvaluator; 8]>` for stack allocation
- [x] **Implement `CostComposer::calculate_step_cost()`** — Sum all evaluator costs with saturating_add
- [x] **Add thermal evaluator** — Query local temperature from database, penalize if above threshold
- [x] **Add electromigration evaluator** — Query current density, penalize if above limit
- [x] **Add crosstalk evaluator** — Query nearest parallel trace distance, penalize if too close
- [x] **Wire into pathfinder** — CostComposer built from RoutingParams and per-net RoutingIntent; stored on GeometryRouter with intent_composers map
- [ ] **Benchmark performance** — Verify zero overhead vs. current explicit penalty approach

**Files to create:**
- `hwc-engine/src/geometry_router/pathfinding/cost_evaluator.rs`

**Success criteria:**
- Cost evaluation is extensible without performance loss
- No dynamic dispatch in routing inner loop
- Thermal and EM evaluators work (even if data sources are mocked initially)

### 2.4 Salsa Query Integration

**Goal:** Add incremental memoization for interface queries

- [ ] **Define Salsa database trait** — Add `RoutingDatabase: salsa::ParallelDatabase` trait
- [ ] **Add tracked interface query** — `#[salsa::tracked] fn get_physical_interface(db, id) -> Arc<PhysicalInterface>`
- [ ] **Add tracked access region query** — `#[salsa::tracked] fn get_interface_access_regions(db, id) -> Arc<Vec<AccessRegion>>`
- [ ] **Add input types** — `#[salsa::input] struct InterfaceInput` for geometry and capabilities
- [ ] **Wire into EntityGraph** — EntityGraph provides RoutingDatabase implementation
- [ ] **Test incremental behavior** — Modify one interface, verify only affected queries recompute
- [ ] **Measure cache hit rate** — Add metrics to track Salsa query cache effectiveness

**Files to create:**
- `hwc-engine/src/geometry_router/query_engine/interface_queries.rs`

**Files to modify:**
- `hwc-engine/src/geometry_router/query_engine/mod.rs`

**Dependencies:**
- Requires Salsa integration (may already exist from v0.1.9 modernization)

**Success criteria:**
- Interface queries are memoized
- Incremental compilation only recomputes changed interfaces
- Cache hit rate >95% for typical design iterations

---

## Phase 3: Advanced Features (Future Work)

### 3.1 Thermal-Aware Routing

**Goal:** Avoid thermal hotspots during routing

- [ ] **Implement thermal field database** — Store temperature gradient field from thermal simulation
- [ ] **Add `RoutingDatabase::get_local_temperature_at(pos)` query** — Return temperature at grid position
- [ ] **Enable thermal cost evaluator** — Wire `CostEvaluator::Thermal` to database
- [ ] **Add thermal penalty tuning** — Profile-driven threshold and penalty values
- [ ] **Test thermal avoidance** — Route near simulated hotspot, verify path deviates
- [ ] **Integrate with physics engine** — Use junction temperature estimates from device models

**Files to modify:**
- `hwc-engine/src/geometry_router/query_engine/mod.rs` (add thermal queries)
- `hwc-physics/` (thermal simulation integration)

**Success criteria:**
- Routes avoid regions above thermal threshold
- Penalty tunable via PDK profile
- Works with both simulated and measured thermal data

### 3.2 Electromigration Analysis

**Goal:** Enforce EM reliability constraints during routing

- [ ] **Add current density database** — Store per-layer current density estimates
- [ ] **Implement `RoutingDatabase::get_current_density_at(pos)` query** — Return current density at position
- [ ] **Enable EM cost evaluator** — Wire `CostEvaluator::Electromigration` to database
- [ ] **Add material-specific EM limits** — Query from MaterialRegistry (e.g., Aluminum: 1mA/μm², Copper: 2mA/μm²)
- [ ] **Test EM avoidance** — High-current net avoids narrow congested regions
- [ ] **Integrate with existing EM checker** — Reuse EM validation from v0.1.8

**Files to modify:**
- `hwc-engine/src/geometry_router/query_engine/mod.rs` (add EM queries)
- `hwc-engine/src/geometry_router/em_thermal_check.rs` (reuse existing)

**Success criteria:**
- Current density limits enforced during routing
- Material-specific EM rules applied
- EM violations caught at synthesis time

### 3.3 Timing-Driven Routing

**Goal:** Optimize critical path timing during routing

- [ ] **Add delay estimation** — Implement RC delay calculation for trace segments
- [ ] **Implement `CostEvaluator::TimingCritical`** — High penalty for increased delay on critical paths
- [ ] **Add net criticality annotation** — Mark critical paths in netlist (from STA)
- [ ] **Implement skew minimization** — For clock nets, minimize path length variation
- [ ] **Test timing optimization** — Critical path routes with minimal length/vias
- [ ] **Integrate with STA engine** — Use static timing analysis to identify critical nets

**Files to create:**
- `hwc-engine/src/geometry_router/timing_estimation.rs`

**Files to modify:**
- `hwc-engine/src/geometry_router/pathfinding/cost_evaluator.rs` (add timing evaluator)

**Success criteria:**
- Critical paths routed with minimal delay
- Clock nets have balanced skew
- Non-critical nets deprioritized for critical net resources

### 3.4 Photonics Extension

**Goal:** Enable waveguide interface routing for photonics

- [ ] **Add `InterfaceGeometry::WaveguideFace`** — Face normal and modal properties
- [ ] **Implement waveguide mode matching** — Ensure compatible modes at interface boundaries
- [ ] **Add modal coupling constraint** — Penalize mode mismatch during routing
- [ ] **Implement `calculate_waveguide_escape()`** — Face-normal alignment for optical coupling
- [ ] **Add wavelength-aware routing** — Consider wavelength in cost calculation
- [ ] **Test photonics routing** — Route optical interconnect between waveguide faces

**Files to modify:**
- `hwc-engine/src/geometry_router/connection_interface.rs` (add waveguide geometry)
- `hwc-engine/src/geometry_router/interface_escape.rs` (add waveguide escape)

**Success criteria:**
- Waveguide interfaces supported
- Mode matching enforced
- Optical routing maintains coupling requirements

---

## Summary

| Phase | Section | Tasks | Completed | Remaining |
|-------|---------|-------|-----------|-----------|
| **1** | **Core Integration** | | **40/40** | **0** |
| | 1.1 EntityGraph Extension | 7 | 7 | 0 |
| | 1.2 Interface Types | 9 | 9 | 0 |
| | 1.3 Integer Geometry | 7 | 7 | 0 |
| | 1.4 Port Escape Bridge | 6 | 6 | 0 |
| | 1.5 Access Regions | 8 | 8 | 0 |
| | 1.6 Capability Validation | 6 | 6 | 0 |
| **2** | **Optimization** | | **22/25** | **3** |
| | 2.1 Connection Candidates | 7 | 6 | 1 (benchmark) |
| | 2.2 Routing Intent | 7 | 7 | 0 |
| | 2.3 Cost Composer | 9 | 8 | 1 (benchmark) |
| | 2.4 Salsa Integration | 7 | 0 | 7 |
| **3** | **Advanced Features** | | **0/23** | **23** |
| | 3.1 Thermal-Aware | 6 | 0 | 6 |
| | 3.2 Electromigration | 6 | 0 | 6 |
| | 3.3 Timing-Driven | 6 | 0 | 6 |
| | 3.4 Photonics | 6 | 0 | 6 |
| **TOTAL** | | **88** | **62** | **26** |

---

## Dependency Graph

```
Phase 1: Core Integration (Critical Path)
  ├── 1.1 EntityGraph Extension ────────────────┐
  │     └── Required by: ALL Phase 1 tasks     │
  │                                             │
  ├── 1.2 Interface Types ──────────────────────┤
  │     └── Required by: 1.3, 1.4, 1.5, 1.6    │
  │                                             │
  ├── 1.3 Integer Geometry ─────────────────────┤
  │     └── Required by: 1.5 (access regions)  │
  │                                             │
  ├── 1.4 Port Escape Bridge ───────────────────┤
  │     └── Required by: Phase 2 routing       │
  │                                             │
  ├── 1.5 Access Regions ───────────────────────┤
  │     └── Required by: 2.1, 2.4              │
  │                                             │
  └── 1.6 Capability Validation ────────────────┘
        └── Required by: 2.2, 3.1, 3.2

Phase 2: Optimization (Builds on Phase 1)
  ├── 2.1 Connection Candidates ────────────────┐
  │     └── Uses: 1.1, 1.2, 1.5                │
  │                                             │
  ├── 2.2 Routing Intent ───────────────────────┤
  │     └── Uses: 1.2, 1.6, 2.3                │
  │                                             │
  ├── 2.3 Cost Composer ────────────────────────┤
  │     └── Uses: 1.2 (capabilities)           │
  │     └── Required by: 2.2, 3.1, 3.2, 3.3    │
  │                                             │
  └── 2.4 Salsa Integration ────────────────────┘
        └── Uses: 1.1, 1.2, 1.5 (queries)

Phase 3: Advanced Features (Optional)
  ├── 3.1 Thermal-Aware ────────────────────────┐
  │     └── Uses: 2.3 (cost composer)          │
  │                                             │
  ├── 3.2 Electromigration ─────────────────────┤
  │     └── Uses: 2.3, 1.6 (capabilities)      │
  │                                             │
  ├── 3.3 Timing-Driven ────────────────────────┤
  │     └── Uses: 2.3, 2.2 (intent)            │
  │                                             │
  └── 3.4 Photonics ────────────────────────────┘
        └── Uses: 1.2, 1.3, 1.4 (geometry)
```

---

## Implementation Order

### Critical Path (Must Complete for Usability) — ✅ ALL COMPLETE
1. **1.1 EntityGraph Extension** — Foundation for everything ✅
2. **1.2 Interface Types** — Core data structures ✅
3. **1.3 Integer Geometry** — Deterministic math for normals ✅
4. **1.5 Access Regions** — Pre-computed approach zones ✅
5. **1.4 Port Escape Bridge** — Connect to existing router ✅
6. **1.6 Capability Validation** — Enforce physical constraints ✅

### Optimization Path (Improves Quality & Performance) — ✅ ALL COMPLETE (except benchmarks)
7. **2.1 Connection Candidates** — Reduces search space dramatically ✅
8. **2.3 Cost Composer** — Extensible cost architecture ✅
9. **2.2 Routing Intent** — Intent-driven routing policies ✅
10. **2.4 Salsa Integration** — Incremental compilation (DEFERRED to later)

### Advanced Path (Future Enhancements)
11. **3.1 Thermal-Aware** — Thermal avoidance
12. **3.2 Electromigration** — EM reliability
13. **3.3 Timing-Driven** — Timing optimization
14. **3.4 Photonics** — Optical interconnect

---

## Testing Strategy

### Phase 1 Tests
- **Unit tests:** Interface type creation, normal derivation, capability constraints
- **Integration tests:** EntityGraph stores/retrieves interfaces, port escape dispatch works
- **Regression tests:** All existing routing tests still pass with CIR infrastructure

### Phase 2 Tests
- **Performance tests:** Connection candidate selection reduces routing time
- **Quality tests:** Cost composer produces equivalent or better routes
- **Intent tests:** Clock routes avoid vias, power routes use vias

### Phase 3 Tests
- **Thermal tests:** Routes avoid hotspots
- **EM tests:** High-current nets get adequate width
- **Timing tests:** Critical paths have minimal delay

---

## Files to Create

### New Modules (Created)
1. `hwc-engine/src/geometry_router/connection_interface/` — Interface types (modular: mod.rs, types.rs, geometry.rs, capability.rs, access_region.rs, physical.rs, testing.rs)
2. `hwc-engine/src/geometry_router/interface_escape.rs` — Geometry-based escape dispatch
3. `hwc-engine/src/geometry_router/connection_candidate.rs` — Connection optimization
4. `hwc-engine/src/geometry_router/routing_intent.rs` — Intent system
5. `hwc-engine/src/geometry_router/pathfinding/cost_evaluator.rs` — Extensible cost system
6. `hwc-engine/src/geometry_router/geometry_math.rs` — Integer math utilities

### New Modules (Pending)
7. `hwc-engine/src/geometry_router/query_engine/interface_queries.rs` — Salsa queries
8. `hwc-engine/src/geometry_router/timing_estimation.rs` — Timing analysis

### Modified Files
1. `hwc-engine/src/geometry_router/entity_graph/mod.rs` — Added interface database + allocation/query methods
2. `hwc-engine/src/geometry_router/entity_graph/impls.rs` — Updated Clone impl for new fields
3. `hwc-engine/src/geometry_router/pathfinding/types.rs` — Extended RoutingParams with interface_constraints and routing_intent
4. `hwc-engine/src/geometry_router/pathfinding/mod.rs` — Added cost_evaluator module declaration
5. `hwc-engine/src/geometry_router/pathfinding/cost_evaluator.rs` — Added `from_routing_params()` and `from_intent_overrides()` constructors
6. `hwc-engine/src/geometry_router/mod.rs` — Added all new module declarations + re-exports
7. `hwc-engine/src/lib.rs` — Re-exports all CIR types
8. `hwc-engine/src/geometry_router/router/core/types.rs` — Added `cost_composer` and `intent_composers` fields to GeometryRouter
9. `hwc-engine/src/geometry_router/router/core/initialization.rs` — Initialized new GeometryRouter fields
10. `hwc-engine/src/geometry_router/router/core/engine.rs` — Initialized new GeometryRouter fields in hierarchical routing clones
11. `hwc-engine/src/geometry_router/constraints.rs` — Added `check_interface_constraints()` and `InterfaceViolation` enum
12. `hwc-compiler/src/ir/errors.rs` — Added `InterfaceConstraintViolation` diagnostic (CIR1)
13. `hwc-compiler/src/ir/placement/component/mod.rs` — Register physical interfaces for each component pin during placement
14. `hwc-compiler/src/ir/routing/global/builder.rs` — Extract intent from Route AST into RoutingData
15. `hwc-compiler/src/ir/routing/global/config.rs` — Added `net_intents` field to RouterConfig
16. `hwc-compiler/src/ir/routing/global/engine.rs` — Register per-net intent cost composers
17. `hwc-compiler/src/ir/compilation/routing_phase.rs` — Pass `net_intents` to AutoRouter
18. `hwc-parser/src/lexer/token.rs` — Added `Intent` token
19. `hwc-parser/src/ast/space/routes.rs` — Added `intent: Option<CompactString>` field to Route struct
20. `hwc-parser/src/parser/routing.rs` — Parse `intent:` in route property blocks
21. `hwc-compiler/src/ir/routing/mod.rs` — Updated test Route constructions with `intent: None`
22. `hwc-compiler/src/ir/parametric_unroller/substitution/unroll_routes.rs` — Propagate `route.intent`
23. `hwc-compiler/src/ir/placement/module.rs` — Updated Route construction with `intent: None`

---

## Progress Tracking

Update this section as tasks are completed:

**Current Status:** PHASE 1 + PHASE 2.1-2.3 FULLY COMPLETE (excluding benchmarks)

**Last Updated:** 2026-07-20

**Next Milestone:** Salsa query integration (2.4), Phase 3 advanced features

**Blockers:** None

**Completed This Session:**
- Phase 1.1: EntityGraph extension (7/7 — wired into component placement)
- Phase 1.2: All interface type definitions (9/9)
- Phase 1.3: Integer geometry operations (7/7)
- Phase 1.4: Port escape bridge (6/6)
- Phase 1.5: Access region generation (8/8)
- Phase 1.6: Capability constraint validation (6/6 — constraint checking + CIR1 diagnostic)
- Phase 2.1: Connection candidate selection (6/7 — wired into global router, benchmark pending)
- Phase 2.2: Routing intent system (7/7 — parser/AST/intent keyword/routing pipeline complete)
- Phase 2.3: Cost composer with enum dispatch (8/9 — wired into GeometryRouter, benchmark pending)

**Build & Test:** 480/480 unit tests pass. Integration test (two-pad-relational) builds and routes successfully.

**Key Files Modified (this session):**
- Parser: token.rs (Intent token), routes.rs (intent field), routing.rs (parse intent:)
- Engine: cost_evaluator.rs (from_routing_params, from_intent_overrides), router/core/types.rs (cost_composer, intent_composers), constraints.rs (check_interface_constraints), entity_graph/mod.rs (register_interface in placement)
- Compiler: errors.rs (CIR1), placement/component/mod.rs (register interfaces), routing/global/ (intent wiring), routing_phase.rs (net_intents)

---

## References

- [Connection-Interface-Routing.md](../../Docs/v0.1.9/Connection-Interface-Routing.md) — Main specification
- [ROUTING-ENGINE-MODERNIZATION.md](../../Docs/v0.1.9/ROUTING-ENGINE-MODERNIZATION.md) — Context on v0.1.9 architecture
- [Spatial_Synthesis_Abstraction.md](../../Docs/v0.1.9/Spatial_Synthesis_Abstraction.md) — Syntax integration
- [port_escape.rs](../../hwc/crates/hwc-engine/src/geometry_router/port_escape.rs) — Existing escape system
- [entity_graph/mod.rs](../../hwc/crates/hwc-engine/src/geometry_router/entity_graph/mod.rs) — EntityGraph implementation
