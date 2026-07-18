# HardwareScript v0.1.9: Salsa-Driven Constraint Solver Architecture

## Executive Summary

This document outlines the migration of HardwareScript's routing engine from a fixed-sequence heuristic router to a **Salsa-driven, multi-objective constraint solver** that treats physical synthesis as a compiler optimization problem rather than a simple pathfinding task.

## Core Philosophy

> **A physical synthesis tool is not just a router; it is a multi-objective constraint solver.**

The current architecture uses manual cache invalidation and fixed optimization sequences, which causes:
- Infinite loops when optimizations interfere with each other
- Cascading failures from global rip-up strategies
- Loss of Salsa's automatic dependency tracking benefits
- Unclear separation between hard constraints (must satisfy) and soft constraints (optimize towards)

## Implementation Plan

### Phase 1: Salsa Query Infrastructure (Foundation)

**Goal:** Replace manual revision tracking with Salsa's automatic dependency management.

#### Task 1.1: Define Salsa Database
- [x] Create `hwc-compiler/src/ir/query_engine.rs`
- [x] Define `RoutingDatabase` trait with Salsa
- [x] Implement database storage in compiler context
- [x] Add Salsa dependency to `hwc-compiler/Cargo.toml`

#### Task 1.2: Define Salsa Input Structures
- [x] Create `#[salsa::input] NetConstraintsInput`
- [x] Create `#[salsa::input] GCellObstaclesInput`
- [x] Create `#[salsa::input] StackupProfileInput`
- [x] Remove manual `obstacle_revision_id` tracking
- [x] Remove manual `mfg_rule_revision_id` tracking

#### Task 1.3: Define Salsa Tracked Queries
- [x] `#[salsa::tracked] fn extract_topological_corridor(...)`
- [x] `#[salsa::tracked] fn get_gcell_obstacles(...)`
- [x] `#[salsa::tracked] fn get_net_constraints(...)`
- [x] `#[salsa::tracked] fn compute_route_metrics(...)`

**Estimated Effort:** 3-4 days
**Dependencies:** None
**Validation:** Salsa automatically invalidates queries when inputs change

---

### Phase 2: Constraint Type System (Hard vs. Soft)

**Goal:** Formalize the distinction between hard constraints (fail if violated) and soft constraints (optimize to minimize delta).

#### Task 2.1: Define Constraint Types
- [x] Create `hwc-engine/src/geometry_router/constraints.rs`
- [x] Implement `ConstraintKind<T>` enum with `Hard` and `Soft` variants
- [x] Define `NetConstraints` structure
- [x] Define `RouteMetrics` structure
- [x] Define `Violation` enum (Hard vs. Soft)

#### Task 2.2: Constraint Extraction from PDK
- [x] Parse hard constraints from profile (min_clearance, max_via_count)
- [x] Parse soft constraints from profile (target_length, target_impedance)
- [x] Validate constraint coherence (no contradictions)
- [x] Store constraints in Salsa inputs

#### Task 2.3: Metrics Computation Engine
- [x] Implement `RouteMetrics::compute(path, space)`
- [x] Calculate total_length_nm from path waypoints
- [x] Count via transitions
- [x] Count bend angles
- [x] Electrical properties (impedance, delay, coupling) belong in the physics engine (`hwc-physics`), not the router

**Estimated Effort:** 2-3 days
**Dependencies:** Phase 1 complete
**Validation:** Metrics match hand-calculated values for test routes

---

### Phase 3: Convergence Loop (Replace Fixed Sequence)

**Goal:** Implement `Measure → Optimize → Measure` loop that terminates gracefully instead of oscillating.

#### Task 3.1: Optimization Result Types
- [x] Define `OptimizationResult` enum (Converged | RequiresRepair)
- [x] Define `OptimizationStrategy` enum for targeted fixes
- [x] Define penalty scoring function for soft constraints

#### Task 3.2: Core Convergence Loop
- [x] Create `hwc-compiler/src/ir/routing/electrical_optimizer.rs`
- [x] Implement `run_optimization_loop(initial_path, constraints, space)`
- [x] Add iteration limit (default: 5)
- [x] Track best score to detect oscillation
- [x] Apply targeted optimizations based on specific violations

#### Task 3.3: Targeted Optimization Implementations
- [x] `inject_meanders(path, deficit_nm)` for length tuning
- [x] `apply_miters_and_teardrops(path, locations)` for impedance
- [x] `widen_trace_segments(path, sections)` for current capacity
- [x] Early exit on hard constraint violations (return RequiresRepair)

**Estimated Effort:** 4-5 days
**Dependencies:** Phase 2 complete
**Validation:** Loop converges within iteration limit for realistic nets

---

### Phase 4: Localized Repair Mechanism

**Goal:** When optimization fails, re-route through a different corridor instead of global rip-up.

#### Task 4.1: Repair Trigger Logic
- [x] Detect `OptimizationResult::RequiresRepair` in routing pipeline
- [x] Identify bottleneck G-cell causing violation
- [x] Invalidate Salsa query for that specific corridor
- [x] Apply negative weight penalty to bottleneck cell

#### Task 4.2: Alternative Corridor Search
- [x] Re-invoke `extract_topological_corridor` with penalties
- [x] Expand search to adjacent G-cells (combined obstacle set)
- [x] Validate corridor width against trace_width + 2 * min_clearance
- [x] Limit repair attempts (user-declared via `OptimizationConfig`)

#### Task 4.3: Cascading Failure Prevention
- [x] Track repair history per net (`RepairHistory`)
- [x] Detect when same G-cell is repeatedly failing (`gcell_has_repeated_failures`)
- [x] Escalate to global replanning only as last resort (exhaustion check)
- [x] Report actionable error if no solution exists (`RepairFailureError`)

**Estimated Effort:** 3-4 days
**Dependencies:** Phase 3 complete
**Validation:** Localized repair finds alternative route without breaking other nets

---

### Phase 5: Navigable Space Extraction (Advanced Pathfinding)

**Goal:** Replace simple L-bend heuristics with convex-preserving trapezoidal decomposition for complex obstacle avoidance.

#### Task 5.1: Spatial Decomposition Core
- [x] Create `hwc-engine/src/geometry_router/navigable_space.rs`
- [x] Implement trapezoidal slicing of free space
- [x] Implement convex-preserving polygon merging
- [x] Build adjacency graph between navigable regions

#### Task 5.2: Corridor Extraction with Semantic Costs
- [x] Implement BFS/Dijkstra over navigable regions
- [x] Incorporate semantic costs (preferred layers, obstacle avoidance weight)
- [x] Generate smooth waypoint sequence through corridor
- [x] Validate corridor width against trace width + clearance

#### Task 5.3: Integration with Topological Router
- [x] Navigable space extraction wired into `extract_corridor_impl` (Salsa query layer)
- [x] Existing fast paths preserved (LOS, 1-bend, 2-bend in `topological_router::route`)
- [x] Navigable space used via Salsa queries when topological routing fails
- [x] Corridor results cached via Salsa memoization

**Estimated Effort:** 5-6 days
**Dependencies:** Phase 1 complete, Phase 4 optional
**Validation:** Routes successfully around complex multi-obstacle scenarios

---

### Phase 6: Electrical Verification Language

**Goal:** Replace marketing claims with precise, engineering-grade terminology.

#### Task 6.1: Documentation Audit
- [ ] Search codebase for "electromagnetically verified"
- [ ] Replace with "electrically optimized geometry suitable for downstream analysis"
- [ ] Clarify BEM vs. FEM capabilities in documentation
- [ ] Update CLI output messages

#### Task 6.2: Metrics Reporting
- [ ] Emit detailed `RouteMetrics` to build output
- [ ] Report constraint satisfaction status per net
- [ ] Flag nets requiring external EM verification
- [ ] Generate metrics summary CSV for post-processing

**Estimated Effort:** 1 day
**Dependencies:** Phase 2 complete
**Validation:** Documentation accurately reflects capabilities

---

### Phase 7: Integration Testing & Validation

**Goal:** Ensure new architecture works correctly across all existing test cases.

#### Task 7.1: Unit Tests
- [ ] Test constraint evaluation logic
- [ ] Test convergence loop termination
- [ ] Test localized repair mechanism
- [ ] Test navigable space extraction

#### Task 7.2: Integration Tests
- [ ] Run all existing ASIC test cases
- [ ] Run all existing PCB test cases
- [ ] Verify no regressions in simple routes
- [ ] Verify complex obstacle avoidance works

#### Task 7.3: Performance Benchmarking
- [ ] Measure Salsa query overhead
- [ ] Measure convergence loop iterations
- [ ] Compare to v0.1.8 baseline
- [ ] Profile hotspots and optimize

**Estimated Effort:** 3-4 days
**Dependencies:** Phases 1-6 complete
**Validation:** All tests pass, performance acceptable

---

## Final v0.1.9 Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. SEMANTIC LOWERING (Salsa-tracked Inputs)                         │
│    AST → Entity Database, Stackup Resolution, NetConstraints        │
└────────────────────────┬─────────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────────┐
│ 2. GLOBAL TOPOLOGY                                                   │
│    Partitioning → G-Cells → Global Corridor Reservation             │
└────────────────────────┬─────────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────────┐
│ 3. GEOMETRY ROUTING (Salsa Memoized Queries)                        │
│    ├── Tier 0: Direct Line-of-Sight Check (LOS)                     │
│    ├── Tier 1: Fast Axis-Aligned Probing (1-Bend / 2-Bend)          │
│    └── Tier 2: Navigable Space Extraction                           │
│         ├── Free-Space Trapezoidal Slicing                          │
│         ├── Convex-Preserving Merging Heuristic                     │
│         └── Adjacency Corridor BFS (Semantic Cost Aware)            │
└────────────────────────┬─────────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────────┐
│ 4. WAYPOINT THREADING                                                │
│    Convert extracted corridors into raw orthogonal coordinate vectors│
└────────────────────────┬─────────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────────┐
│ 5. GEOMETRY LEGALIZATION                                             │
│    Continuous QP/DAG solvers push vectors to satisfy HARD DRC        │
└────────────────────────┬─────────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────────┐
│ 6. ELECTRICAL OPTIMIZATION & VERIFICATION                            │
│    Measure → Optimize → Measure Convergence Loop:                   │
│    ├── Generate RouteMetrics (Length, Impedance, Delay, Skew)       │
│    ├── Validate against NetConstraints (Hard & Soft)                │
│    └── Apply localized tuning (Meanders, Miters, Teardrops)         │
└────────────────────────┬─────────────────────────────────────────────┘
                         │
                         ├─ SUCCESS → Continue to Manufacturing
                         │
┌────────────────────────▼─────────────────────────────────────────────┐
│ 7. LOCALIZED REPAIR (The Escape Valve)                              │
│    If Hard Constraints fail in Step 6:                              │
│    - Invalidate specific G-cell corridor cache                      │
│    - Apply penalty weight                                           │
│    - Re-invoke Step 3 (No global rip-up)                            │
└────────────────────────┬─────────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────────────┐
│ 8. MANUFACTURING OPTIMIZATION (THE COPPER WELDER)                    │
│    Clipper2 2D Polygon Union + Earcut Triangulation                 │
│    Generate watertight, electrically optimized geometry             │
└──────────────────────────────────────────────────────────────────────┘
```

## Timeline & Resource Estimation

| Phase | Duration | Effort | Dependencies |
|-------|----------|--------|--------------|
| Phase 1: Salsa Infrastructure | 3-4 days | Medium | None |
| Phase 2: Constraint System | 2-3 days | Low | Phase 1 |
| Phase 3: Convergence Loop | 4-5 days | High | Phase 2 |
| Phase 4: Localized Repair | 3-4 days | Medium | Phase 3 |
| Phase 5: Navigable Space | 5-6 days | High | Phase 1 |
| Phase 6: Documentation | 1 day | Low | Phase 2 |
| Phase 7: Testing | 3-4 days | Medium | All |
| **TOTAL** | **21-27 days** | **~4 weeks** | Sequential |

## Success Criteria

- [x] All manual revision tracking removed
- [x] Salsa automatically invalidates stale queries
- [x] Convergence loop terminates gracefully (no oscillation)
- [ ] Localized repair succeeds for 95%+ of routing failures
- [x] Navigable space extraction routes around complex obstacles
- [ ] All v0.1.8 test cases pass
- [ ] Performance within 2x of v0.1.8 baseline
- [ ] Documentation accurately reflects capabilities

## Migration Strategy

This is a **major architectural change**. The migration path:

1. **Clean Slate (Week 1):** Remove conflicting legacy router code immediately - no feature flags, no unnecessary baggage
2. **Salsa Infrastructure (Weeks 1-2):** Build query system from scratch with immutability guarantees
3. **Incremental Enablement (Week 3):** Enable routing tiers one-by-one (LOS → L-bend → Navigable Space)
4. **Validation & Refinement (Week 4):** Run all test cases, fix issues, optimize performance
5. **Production Ready:** Ship v0.1.9 with new architecture as the only option

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Salsa learning curve | Medium | Start with simple queries, add complexity incrementally |
| Performance regression | High | Profile early, optimize hotspaths, consider caching strategies |
| Breaking existing workflows | Critical | Feature flag, extensive testing, clear migration guide |
| Navigable space bugs | Medium | Comprehensive unit tests, visual debugging tools |

## Conclusion

This migration transforms HardwareScript from a "rigid voxel-grid maze-router" to a **Salsa-driven, Continuous-Coordinate Physical Synthesis Compiler** that natively understands geometry, handles constraints gracefully, isolates dependencies via query caching, and models true physical reality.

The result is a **deterministic, cacheable, and formally correct** routing engine that treats physical synthesis as a compiler optimization problem.

---

**Version:** v0.1.9  
**Status:** Implementation In Progress  
**Owner:** Core Compiler Team  
**Last Updated:** 2026-07-18
