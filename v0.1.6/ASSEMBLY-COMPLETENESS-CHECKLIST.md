# Hardware Script v0.1.6: Assembly Completeness Implementation Checklist

**Reference Document**: `ASSEMBLY-COMPLETENESS-ANALYSIS.md` (read for detailed explanations)  
**Current Completion**: 92%  
**Target**: 100% Assembly Completeness  
**Time Estimate**: 2 weeks

---

## CRITICAL PATH (Priority Order)

### ✅ WEEK 1: Gap 7 - Spatial Topological Sorting (COMPLETE)
**Status**: 100% Complete | **Blocks**: None

#### Phase 1: Spatial Dependency Graph (3 days)
- [x] Create `hwc-compiler/src/ir/spatial_dependency_graph.rs`
- [x] Implement `SpatialDependencyGraph` struct
- [x] Implement `extract_anchor_references()` from coordinate expressions
- [x] Implement `add_component_dependency()` method
- [x] Implement `detect_circular_dependencies()` using DFS
- [x] Add error type: `SpatialDependencyError::CircularReference`

#### Phase 2: Topological Sort Integration (2 days)
- [x] Update `hwc-compiler/src/ir/mod.rs` (Integrated into placement pipeline)
- [x] Implement `topological_sort_components()` method
- [x] Modify placement pipeline to use sorted order (not textual order)
- [x] Handle components with no dependencies (absolute positions)
- [x] Handle components with dependencies (relative positions)

#### Phase 3: Testing (2 days)
- [x] Create `hwc/tests/sprint_spatial_sorting/test_forward_reference.hw`
- [x] Create `hwc/tests/sprint_spatial_sorting/test_circular_dependency.hw`
- [x] Create `hwc/tests/sprint_spatial_sorting/test_complex_chain.hw`
- [x] Verify: M2 can reference M1 even if M1 is defined later
- [x] Verify: Circular dependencies are detected and reported
- [x] Verify: Complex chains (M1→M2→M3→M4) work correctly

**Deliverable**: ✅ Can reference components in any textual order

---

### 🟠 WEEK 2: Gap 2 - Finish Parametric Unrolling (HIGH)
**Status**: 100% Complete | **Blocks**: None

- [x] Update `hwc-compiler/src/ir/parametric_unroller.rs`
- [x] Implement `unroll_pour_loop()` method
- [x] Evaluate loop variable in `boundary:` expressions
- [x] Evaluate loop variable in `net:` expressions
- [x] Handle `named:` with array syntax: `Trace[i]`
- [x] Generate unique pour instances per iteration
- [x] Implement `unroll_contact_loop()` method
- [x] Evaluate loop variable in position expressions
- [x] Evaluate loop variable in `spanning` expressions
- [x] Evaluate loop variable in `net:` expressions
- [x] Generate unique contact instances per iteration
- [x] Track previous component in loop context
- [x] Resolve `last` to previous component's bounding box (Resolved in Constraint Solver)
- [x] Create `hwc/tests/sprint_parametric/test_pour_loop.hw` (3D Staircase Test)
- [x] Verify: 64 pours stamped in 10 lines of code
- [x] Verify: `last` keyword chains components correctly

**Deliverable**: ✅ Can write 64-bit bus in 10 lines of code

---

### 🟡 WEEK 3: Gap 3 - Multi-Layer Via Stacks (HIGH)
**Status**: 100% Complete | **Blocks**: None

#### Phase 1: Via Stack Algorithm (3 days)
- [x] Update `hwc-compiler/src/ir/auto_via_inserter.rs`
- [x] Implement `insert_via_stack()` method
- [x] Detect layer span > 1 (e.g., z:2 → z:10)
- [x] Query via library for intermediate layer vias
- [x] Calculate via positions for each layer transition
- [x] Stamp via chain: z:2→z:3, z:3→z:4, ..., z:9→z:10
- [x] Verify overlap at each layer transition

#### Phase 2: Via Stack Validation (2 days)
- [x] Implement `validate_via_stack()` method
- [x] Check enclosure requirements for each via
- [x] Check layer compatibility (material constraints)
- [x] Error if via library missing required via type
- [x] Generate detailed error messages with layer info

#### Phase 3: Testing (2 days)
- [x] Create `hwc/tests/sprint_via_stacks/test_silicon_to_metal5.hw`
- [x] Create `hwc/tests/sprint_via_stacks/test_via_stack_validation.hw`
- [x] Create `hwc/tests/sprint_via_stacks/test_10_layer_stack.hw`
- [x] Verify: Via stack from z:2 to z:10 (8 vias)
- [x] Verify: GDSII export shows all intermediate vias
- [x] Verify: Build time < 100ms for via stack insertion

**Deliverable**: ✅ Can route from Silicon to top metal layer automatically

---

## MEDIUM PRIORITY (Post-SoC Readiness)

### 🟢 Gap 1 - Relative Positioning Enhancements
**Status**: 100% Complete | **Improves**: Usability

#### Pour Relative Positioning (3 days)
- [x] Update `hwc-parser/src/parser/definitions/space/pours.rs`
- [x] Parse relative coordinates in `boundary:` expressions
- [x] Support: `boundary: P1.right to [x: P1.right + 5mm, y: 10mm]`
- [x] Update `hwc-compiler/src/ir/constraint_solver.rs`
- [x] Resolve pour boundaries using bbox tracker
- [x] Create `tests/sprint_relative_positioning/test_relative_pour.hw`

#### Circular Dependency Detection (1 day)
- [x] Update `hwc-compiler/src/ir/constraint_solver.rs`
- [x] Build dependency graph during resolution
- [x] Detect cycles using DFS
- [x] Error: "Circular dependency: C1 → C2 → C1"
- [x] Create `tests/sprint_relative_positioning/test_circular_dependency.hw`

**Deliverable**: ✅ Can position pours relative to other pours

---

### 🟢 Gap 3 - Via Enhancements
**Status**: 100% Complete | **Improves**: High-current routing

#### Via Array Generation (3 days)
- [x] Update `hwc-compiler/src/ir/auto_via_inserter.rs`
- [x] Implement `generate_via_array()` method
- [x] Calculate required via count from overlap area
- [x] Generate via grid pattern (e.g., 16×16 array)
- [x] Ensure minimum via spacing from profile
- [x] Create `hwc/tests/sprint_via_arrays/test_power_via_array.hw`
- [x] Implement constraint-driven logic (power/ground → arrays, signal → single)

#### Profile-Based Via Rules (2 days)
- [x] Via constraints read from profile definitions
- [x] Support `min_spacing` for via array generation
- [x] Net classification drives via insertion strategy

**Deliverable**: ✅ Can generate via arrays for power distribution (COMPLETE)

