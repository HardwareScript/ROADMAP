# Hardware Script Compiler - GAP Implementation Plan

**Version:** v0.1.4+  
**Document Type:** Implementation Checklist  
**Purpose:** Clean, actionable implementation guide for all GAP systems  
**Last Updated:** March 29, 2026

---

## Overview

This document provides a structured implementation plan for closing all gaps identified in GAP1-4 documents. Each section references the relevant GAP document sections for detailed context.

**Key Principle:** This is a checklist, not a tutorial. Read the referenced GAP sections to understand the "why" and "how" before implementing.

---

## Phase 1A: Foundation (v0.1.4.1)

### 1.1 Memory Architecture

**Status:** COMPLETED ✅  
**Reference:** GAP1 Section 3.2-3.4, GAP3 Part 1

- [x] FxHashMap Migration (COMPLETED)
  - All HashMap instances replaced with FxHashMap
  - All crates updated with rustc-hash dependency
  - 143+ tests passing

- [x] Memory Allocator Swap (COMPLETED)
  - **Reference:** GAP1 Section 3.4 (Gotcha 1)
  - Added mimalloc dependency to hwc-cli/Cargo.toml
  - Added global allocator declaration to hwc-cli/src/main.rs
  - **Expected Impact:** 20-40% speedup in parallel compilation

- [x] i64 Fixed-Point Coordinates (COMPLETED)
  - **Reference:** GAP1 Section 3.2.A
  - Point3D struct uses i64 nanometers for all coordinates (x, y, z)
  - All spatial calculations use integer math
  - **Why:** Determinism across different CPUs
  - **Verification:** All 67 geometry tests pass

- [x] Morton Encoding Implementation (COMPLETED)
  - **Reference:** GAP1 Section 3.2.B
  - Implemented morton_encode(x, y, z) -> u64
  - Implemented morton_decode(code) -> (x, y, z)
  - Added to hwc-engine/src/morton.rs
  - **Why:** Cache locality for 3D spatial queries
  - **Verification:** All 12 Morton encoding tests pass

- [x] Sparse Voxel Grid (COMPLETED)
  - **Reference:** GAP1 Section 3.2.C-D, GAP3 Part 1
  - Replaced dense array with FxHashMap<u64, Box<VoxelChunk>>
  - Updated SparseVoxelGrid struct (now VoxelGrid)
  - All voxel access uses Morton codes for chunk keys
  - 4x4x4 chunked architecture with u64 bitmasks
  - **Success Criteria:** Empty 1B voxel grid uses ~0 bytes, sparse storage scales with occupancy
  - **Verification:** All voxel_grid tests pass, memory_stats() shows sparse behavior

**FOUNDATION COMPLETE:** All core memory architecture components are implemented and tested. Ready for Phase 1B (Parallel Routing).

---

### 1.2 Module System Core

**Status:** Partially Complete  
**Reference:** GAP1 Section 2.1, MODULE-SYSTEM-FIXES.md

- [x] Module Flattening Core (COMPLETED)
  - Module detection in place_component()
  - Comptime evaluation (for loops, if conditionals)
  - Module component placement with layout blocks
  - Automatic placement fallback

- [x] Array Syntax in Layout Blocks (COMPLETED)
  - Parser supports R[0] syntax in layout blocks
  - Tested with for-loop-generated components

- [x] Module Route Flattening (COMPLETED)
  - Module routes converted to space routes
  - Interface pin detection
  - Array pin indexing in routes

- [x] Import System (@std/ paths) (COMPLETED)
  - **Reference:** GAP1 Section 2.1, Priority 2 Item 6
  - Updated lexer to recognize @ as valid import path start (already done)
  - Wired up stdlib interceptor in compiler
  - Route @std/* to internal hwc-stdlib crate
  - **Test:** `import Copper from @std/materials` works
  - **Verification:** test_stdlib_import.hw compiles successfully

- [x] Array Pin Expansion in Symbol Table (COMPLETED)
  - **Reference:** GAP1 Section 2.1, MODULE-SYSTEM-FIXES.md Item 5
  - Expand array pins into individual pins during registration
  - Store as Bus_Out[0], Bus_Out[1], ..., Bus_Out[63]
  - Update pin resolution to handle array syntax
  - **Test:** `route MainDSP.Bus_Out[0] to Amp.RF_IN` works
  - **Implementation:** Updated `get_pin_positions()` to construct full pin names with indices
  - **Verification:** All 543 tests pass, module array pins resolve correctly

---

### 1.3 Parser Robustness

**Status:** Partially Complete  
**Reference:** GAP1 Section 1.1-1.3

- [x] Mathematical Expression Parser (COMPLETED)
  - Expression AST with literals, variables, binary/unary ops
  - Pratt parser implementation
  - Coordinate fields use Expression enum
  - All parser tests passing

- [x] Whitespace & Comment Handling (COMPLETED)
  - skip_whitespace() calls in all module body parsing loops
  - Parser skips Newline, Comment, BlockComment tokens
  - 410+ tests passing

- [x] Comma-Separated Pin Lists (COMPLETED)
  - Inline syntax: `pins: VCC, GND, SDA, SCL`
  - Block syntax still supported
  - Array support: `pins: DataBus[8], AddressBus[16]`
  - 103 parser tests pass

- [x] Array Indexing in Routes (COMPLETED)
  - PinReference AST includes component_index and pin_index
  - Syntax: Component.Pin[0], Component[0].Pin, Component[0].Pin[1]
  - 107 parser tests pass

- [x] Advanced Definition Types (COMPLETED)
  - signal_group: Signal grouping for impedance-controlled routing
  - pour: Copper pours for ground/power planes
  - polygon: Custom copper shapes for antennas/RF
  - layout blocks: Module-to-physical mapping
  - All features parse and validate correctly

- [x] Scientific Notation Support (COMPLETED)
  - **Reference:** GAP1 Section 1.1
  - Update number regex: `[0-9]+(?:\.[0-9]+)?(?:[eE][+-]?[0-9]+)?`
  - **Test:** `1.68e-8`, `1e9`, `2.4e3` parse correctly
  - **Implementation:** Already implemented in lexer token.rs with Float and Measurement regex patterns
  - **Verification:** Regex patterns support scientific notation for both standalone floats and measurements

- [x] Expression Evaluation in Compiler (COMPLETED)
  - **Reference:** GAP1 Priority 2 Item 4
  - Implement comptime expression evaluation
  - Evaluate expressions during module flattening
  - **Test:** `at [x: 10 + (i*5), y: 20, z: 1]` works in for loops
  - **Implementation:** Expression evaluation implemented in hwc-parser with EvaluationContext
  - **Verification:** coordinate_to_point() uses evaluate_const(), all expression tests pass

---

## Phase 1B: Parallel Architecture (v0.1.4.2)

**Reference:** GAP3 Part 1 (Complete Section)

### 2.1 Domain Partitioning

- [x] Bounding Box Calculation (COMPLETED)
  - **Reference:** GAP3 Section "Phase 1: Partitioning"
  - Calculate module bounding boxes from layout blocks
  - Implement BoundingBox struct with min/max Point3D
  - Add to ConstraintManager::calculate_module_bounding_box()
  - **Implementation:** Added calculate_module_bounding_box() method to ConstraintManager
  - **Verification:** All 3 bounding box tests pass (empty, single component, multiple components)

- [x] Net Classification (COMPLETED)
  - **Reference:** GAP3 Section "Phase 1: Partitioning"
  - Classify nets as "internal" (both pins in same module) or "global"
  - Build interface pin list for each module
  - Implement in ConstraintManager
  - **Implementation:** Added net_classification module with classify_nets() function
  - **Verification:** All 6 net classification tests pass (empty, internal, global, top-level, mixed)

- [x] Domain Structure (COMPLETED)
  - **Reference:** GAP3 Section "1. The Domain Structure"
  - Implement RoutingDomain struct
  - Implement RoutedDomain struct
  - Add domain_id, bounding_box, internal_nets, interface_pins, local_grid
  - **Implementation:** Added domain module with RoutingDomain, RoutedDomain, and Route structs
  - **Verification:** All 4 domain tests pass (creation, dimensions, coordinate conversion, routed domain)

---

### 2.2 Parallel Routing Core

- [x] Rayon Integration (COMPLETED)
  - **Reference:** GAP3 Section "Phase 2: Local Parallel Routing"
  - Rayon dependency already present in Cargo.toml
  - Implemented ParallelRouter::route_domains()
  - Uses .into_par_iter() for domain parallelization
  - **Implementation:** Created parallel_router.rs with ParallelRouter struct
  - **Verification:** All 6 parallel router tests pass

- [x] Isolated Domain Routing (COMPLETED)
  - **Reference:** GAP3 Section "Phase 2: Local Parallel Routing"
  - Creates local A* router per domain
  - Routes within local coordinate space
  - Marks voxels in local_grid (isolated FxHashMap)
  - **Implementation:** route_internal_nets() function with isolated grid per thread
  - **Verification:** test_route_domains_single_domain and test_route_domains_parallel_isolation pass

- [x] Coordinate Translation (COMPLETED)
  - **Reference:** GAP3 Section "Phase 2: Local Parallel Routing"
  - Converts global → local coordinates (subtract bounding_box.min)
  - Converts local → global coordinates (add bounding_box.min)
  - Handles Morton encoding with i64→u32 conversion
  - **Implementation:** Uses domain.global_to_local() and domain.local_to_global()
  - **Verification:** test_coordinate_conversion passes, routes use local coordinates

---

### 2.3 Assembly & Global Routing

- [x] Grid Merging (COMPLETED)
  - **Reference:** GAP3 Section "Phase 3: Assembly & Global Routing"
  - Merges all domain grids into global grid
  - Decodes Morton codes, translates coordinates, re-encodes
  - Inserts into global FxHashMap
  - **Implementation:** assemble_and_route_global() function with coordinate translation
  - **Verification:** test_assemble_and_route_global_empty passes

- [x] Global Net Routing (COMPLETED)
  - **Reference:** GAP3 Section "Phase 3: Assembly & Global Routing"
  - Routes nets that cross domain boundaries
  - Uses global A* router with merged grid
  - Connects interface pins between domains
  - **Implementation:** Global routing loop in assemble_and_route_global()
  - **Verification:** GlobalRoutingResult structure returns all routes

- [x] Performance Testing (COMPLETED)
  - **Reference:** GAP3 Section "Performance Expectations"
  - Benchmarked on various board sizes (2-32 domains, 10-1280 nets)
  - Verified determinism (100 runs produce identical output)
  - **Implementation:** Created parallel_routing_benchmark.rs with 5 benchmark tests
  - **Results:** 
    - Small (4 domains, 40 nets): ~3ms, 100% success
    - Medium (8 domains, 160 nets): ~308ms, 100% success
    - Large (16 domains, 480 nets): ~379ms, 100% success
    - XLarge (32 domains, 1280 nets): ~691ms, 100% success
  - **Verification:** All 4 performance tests pass, determinism verified over 100 runs

---

## Phase 2: Routing Engine Enhancements (v0.1.4.3)

**Reference:** GAP1 Section 3.1, 5.1-5.3

### 3.1 Via System

- [x] Via Penalty and Tracking (COMPLETED)
  - **Reference:** GAP1 Section 1.5
  - A* router discourages layer changes with 10,000-point penalty
  - Via struct tracks position, layer span, diameter, net ID
  - All 21 routing tests pass

- [x] Via Footprint Validation (COMPLETED)
  - **Reference:** GAP1 Section 3.1 (Law 2)
  - Check 3×3 or 5×5 grid column before Z-change
  - Ensure clearance on all intermediate layers
  - Implement can_place_via() in Router
  - **Implementation:** Added can_place_via() and is_circular_area_clear() methods
  - **Verification:** test_via_footprint_validation_success and test_via_footprint_validation_blocked pass

- [x] Via Stamping (COMPLETED)
  - **Reference:** GAP1 Section 3.1 (Law 2)
  - Mark circular area as occupied on all layers via passes through
  - Record via position, diameter, layer span
  - Implement stamp_via() in Router
  - **Implementation:** Added stamp_via() and mark_circular_area_occupied() methods
  - **Verification:** test_via_stamping passes, vias occupy correct footprint on all layers

- [x] Anti-Pad Generation (COMPLETED)
  - **Reference:** GAP1 Section 3.1 (Law 3)
  - Detect when via passes through copper pour
  - Carve clearance void if via net ≠ pour net
  - Implement generate_antipad() in Router
  - **Implementation:** Added generate_antipads() and remove_circular_area() methods
  - **Verification:** test_antipad_generation passes

- [x] Drill File Export (COMPLETED)
  - **Reference:** GAP1 Section 3.1, 4.2
  - Generate .drl (Excellon) file for every via location
  - Include drill diameter and coordinates
  - Add to export pipeline
  - **Implementation:** Created hwc-export/src/excellon.rs with ExcellonExporter
  - **Verification:** All 5 Excellon tests pass, drill files generated in industry-standard format

---

### 3.2 Advanced Routing Features

- [x] Rip-Up and Reroute (COMPLETED ✓)
  - **Reference:** GAP1 Section 5.1
  - **Status:** FULLY IMPLEMENTED
  - **Architecture:** Priority-based iterative routing with conflict detection
  
  **Implementation Complete:**
  - Net priority system (Critical, Power, HighSpeed, DataBus, LowSpeed, GPIO)
  - Automatic priority detection from net names (VCC, CLK, DATA, I2C, GPIO patterns)
  - Route conflict detection and blocking net identification
  - Rip-up capability for lower priority nets
  - Iterative routing with configurable max iterations
  - Routing statistics and completion rate tracking
  
  **Files Created:**
  - `hwc/crates/hwc-engine/src/geometry_router/priority.rs` - Priority system
  - `hwc/crates/hwc-engine/src/geometry_router/ripup.rs` - Rip-up engine
  - `hwc/crates/hwc-engine/tests/ripup_reroute_test.rs` - Test suite
  
  **Router Extensions:**
  - Added `clear_voxel()` method to remove occupied voxels
  - Added `clear_via()` method to remove vias and clear footprints
  - Added `MaxIterationsExceeded` and `InvalidNet` error types
  
  **Test Results:**
  - All 6 rip-up tests passing
  - Priority ordering verified
  - Conflict resolution working correctly
  - Statistics tracking functional
  
  **Verification:** High-priority nets can rip up lower-priority nets to achieve 100% routing completion

- [x] Length Matching & Meandering (COMPLETED ✓)
  - **Reference:** GAP1 Section 5.2, CONSTRAINT-AWARE-ROUTING.md
  - **Status:** FULLY IMPLEMENTED - Core engine + Parser support complete
  - **Core Insight:** Stop post-processing. Feed constraints into A* BEFORE routing starts.
  - **Approach:** Constraint-Aware A* with pattern macro-moves
  
  **Implementation Complete:**
  - **Phase 1:** ✓ Constraint-aware A* with voxel budget tracking (ConstraintNode)
  - **Phase 2:** ✓ Modified heuristic to penalize length mismatches
  - **Phase 3:** ✓ Pattern system with `distance r angle` syntax
  - **Phase 4:** ✓ Parser support for `define pattern` and `define strategy`
  - **Phase 5:** ✓ Router integration with route_net_with_length_constraint()
  
  **Parser Features:**
  - Context-sensitive keywords: `pattern` and `strategy` handled as identifiers
  - Pattern definitions: `define pattern "Name" (params): steps: - distance r angle`
  - Strategy definitions: `define strategy "Name": target: match_longest, tolerance: 0.1mm, pattern: PatternName(args)`
  - Expression support: Measurements in expressions (0.3mm, gap * 2, etc.)
  - Standard library: `hwc/stdlib/routing/patterns.hw` with Zigzag, Trombone, Serpentine
  
  **Files Created:**
  - `hwc/crates/hwc-engine/src/geometry_router/constraint_aware.rs` - Core routing engine
  - `hwc/crates/hwc-engine/src/geometry_router/routing_patterns.rs` - Pattern system
  - `hwc/crates/hwc-parser/src/ast/pattern.rs` - AST definitions
  - `hwc/crates/hwc-parser/src/parser/definitions/pattern.rs` - Parser implementation
  - `hwc/stdlib/routing/patterns.hw` - Standard pattern library
  
  **Test Results:**
  - All 6 pattern routing tests passing with 0mm error on target lengths
  - All 5 parser tests passing (pattern/strategy parsing)
  - Zero warnings in test suite
  
  **Verification:** DDR5 memory interface with trombone meandering achieves exact length matching

- [x] HDI Via Support (COMPLETED)
  - **Reference:** GAP1 Section 5.3
  - Implemented ViaType enum (ThroughHole, Blind, Buried, Microvia)
  - Added automatic via type classification based on layer span and diameter
  - Added cost calculation for different via types (cost multipliers and routing penalties)
  - Implemented ViaTypeCategory for drill file export
  - Added export_hdi_vias() function to generate separate drill files for each via type
  - **Implementation:**
    - Updated Via struct with via_type field and classification logic
    - Added ViaType enum with cost_multiplier() and routing_penalty() methods
    - Created ViaTypeCategory enum for drill file separation
    - Updated ExcellonExporter to support HDI via export
    - Exported Via and ViaType from geometry_router module
  - **Verification:** All 9 HDI via tests pass, all 273 engine tests pass
  - **Test:** Route 10-layer board with mixed via types ✓

---

## Phase 3: Export & Manufacturing (v0.1.4.4)

**Reference:** GAP1 Section 4.1-4.3, 5.4-5.5

### 4.1 Export Decoupling

- [x] Decouple Gerber from 3D Exporters (COMPLETED)
  - **Reference:** GAP1 Section 4.1
  - Missing color property now warns instead of crash
  - Gerber export independent of visual properties
  - Fallback to default color (#FF00FF) for 3D exports
  - **Implementation:** Modified scene_graph.rs extract_color() to return default color with warning
  - **Verification:** All 38 export tests pass

---

### 4.2 Missing Export Formats

- [x] Drill File Generation (COMPLETED)
  - **Reference:** GAP1 Section 4.2
  - Generate Excellon drill files from mechanical mounting_holes
  - Include via locations from routing
  - **Test:** PCB manufacturers can fabricate board
  - **Implementation:** Already completed in Phase 2 (hwc-export/src/excellon.rs)
  - **Verification:** All 6 Excellon tests pass

- [x] Bill of Materials (BOM) (COMPLETED)
  - **Reference:** GAP1 Section 4.2
  - Generate CSV/JSON BOM with component designators, values, quantities
  - **Test:** Can order parts and estimate costs
  - **Implementation:** Already completed (hwc-export/src/bom.rs), now properly populated with IR data
  - **Verification:** All 2 BOM tests pass

- [x] Netlist Export (COMPLETED)
  - **Reference:** GAP1 Section 4.2
  - Generate SPICE netlist from routing graph
  - **Test:** Can perform circuit simulation
  - **Implementation:** Created hwc-export/src/netlist.rs with SPICE format generation
  - **Verification:** All 3 netlist tests pass

- [x] CLI Manufacturing Export Integration (COMPLETED)
  - **Gap Discovered:** Build command was not generating HardwareIR, causing BOM and Netlist to be empty
  - **Fix:** Refactored hwc-cli/src/commands/build.rs to use compile_two_pass() instead of manual compilation
  - **Result:** BOM, Excellon, and Netlist now automatically exported with actual data on every build
  - **Verification:** End-to-end build pipeline tested successfully

---

### 4.3 Advanced Manufacturing

- [x] Pad Shapes (COMPLETED ✓)
  - **Reference:** GAP1 Section 5.4
  - **Status:** Parser and engine support complete
  - **Implementation:**
    - Added PadShape enum to hwc-engine with Circle, Rectangle, Obround, Polygon, RoundedRect variants
    - Updated LayoutBlock AST to include pad_shapes field (FxHashMap<String, String>)
    - Extended parser to handle pad_shapes block in component layout definitions
    - Added parse_pad_shape() function to convert shape strings to PadShape enum
    - Updated all test files to include pad_shapes field in LayoutBlock initializations
    - Created comprehensive test suite (pad_shape_test.rs) with 5 tests covering all shape types
  - **Verification:** All 5 pad shape parser tests pass, all 52+ test suites pass
  - **Next Steps:** Generate solder mask (.gts/.gbs) and paste (.gtp/.gbp) layers in Gerber export

- [x] Polygon Rasterization (COMPLETED ✓)
  - **Reference:** GAP1 Section 4.3, 5.5
  - **Status:** Core rasterization engine complete with thermal relief support
  - **Implementation:**
    - Created polygon_rasterizer.rs with scanline algorithm for arbitrary polygon shapes
    - Implemented Polygon struct with support for outer boundary and holes
    - Added optimized paths for rectangles and circles
    - Created thermal_relief.rs with spoke pattern generation
    - Supports three relief types: Spokes (4-spoke pattern), Direct (no relief), Isolated (clearance gap)
    - Configurable spoke width, count, and gap width
    - Works with both circular and rectangular pads
  - **Verification:** All 7 polygon rasterizer tests pass, all 7 thermal relief tests pass, all 314 engine tests pass
  - **Features:**
    - Scanline algorithm for polygon boundaries
    - Support for arbitrary polygon shapes and holes
    - Circular pad rasterization with midpoint circle algorithm
    - Thermal relief spoke generation at configurable angles
    - Anti-pad clearance gap generation for copper pours
  - **Next Steps:** Integrate with copper pour placement and Gerber export

- [x] Thermal Reliefs (COMPLETED ✓)
  - **Reference:** GAP1 Section 5.5
  - **Status:** Full thermal relief system implemented and tested
  - **Implementation:**
    - Created thermal_relief.rs with ThermalReliefGenerator
    - Three relief types: Spokes (4-spoke pattern), Direct (no relief), Isolated (clearance gap)
    - Configurable spoke width (default 0.3mm), spoke count (default 4), gap width (default 0.2mm)
    - Spoke pattern generation at configurable angles (90° intervals for 4 spokes)
    - Works with both circular and rectangular pads
    - Generates clearance gaps around pads in copper pours
    - Adds spoke connections back to maintain electrical connection while restricting heat flow
  - **Verification:** All 7 thermal relief tests pass, 5 integration tests pass including multi-layer board test
  - **Features:**
    - Spoke pattern generation for copper pours ✓
    - Pad detection in pour area ✓
    - Spoke or clearance pattern generation ✓
    - Configurable relief parameters ✓
  - **Test Results:** 4-layer board test with ground/power planes validates correct thermal relief generation

---

## Phase 4: Advanced Features 

**Reference:** GAP2 (Complete Document)

### 5.1 Procedural Routing

- [x] Strategy Keyword (COMPLETED ✓)
  - **Reference:** GAP2 Section 1
  - **Implementation:** Parser supports `define strategy` with target, tolerance, and pattern fields
  - **AST:** StrategyDefinition in hwc-parser/src/ast/pattern.rs
  - **Execution:** Integrated with constraint-aware A* router
  - **Approach:** Used constraint-aware routing instead of post-processing - patterns are macro-moves injected into A* neighbor generator before routing starts

- [x] Standard Library Strategies (COMPLETED ✓)
  - **Reference:** GAP2 Section 1
  - **Implementation:** hwc/stdlib/routing/patterns.hw with Zigzag, Trombone, Serpentine patterns
  - **Engine:** StandardPatterns struct in routing_patterns.rs provides programmatic access
  - **Syntax:** All patterns use `distance r angle` polar notation (e.g., `gap r 45`)
  - **Note:** Implemented core patterns (Zigzag, Trombone, Serpentine) - additional patterns (BGA_Fanout, DifferentialPair, ArcRoute) can be added as needed

- [x] Strategy Integration (COMPLETED ✓)
  - **Reference:** GAP2 Section 1
  - **Implementation:** Integrated via route_net_with_length_constraint() in GeometryRouter
  - **Validation:** Pattern collision detection using can_place() checks against occupied voxels
  - **Architecture:** Constraint-aware pathfinding with modified heuristic that penalizes length mismatches
  - **Test Results:** All 6 pattern routing tests passing with 0mm error on target lengths
  - **Verification:** DDR5 memory interface with trombone meandering achieves exact length matching

---

### 5.2 Stability Architecture

- [x] Net Priority System (COMPLETED ✓)
  - **Reference:** GAP2 Section 2 (Pillar A)
  - **Status:** Fully integrated with priority-based routing
  - **Implementation:**
    - NetPriority enum with 6 levels (Critical, Power, HighSpeed, DataBus, LowSpeed, GPIO) in priority.rs
    - Automatic priority detection from net names (from_net_name) with pattern matching
    - Priority comparison (can_rip_up) for rip-up routing
    - Integrated in RipUpRouter with priority-based sorting
    - Added route_all_nets_with_priority() to GeometryRouter for production routing
    - Nets sorted highest-to-lowest priority before routing (Critical → Power → HighSpeed → DataBus → LowSpeed → GPIO)
  - **Test Results:** test_route_all_nets_with_priority passes - clock nets routed before GPIO nets
  - **Verification:** Moving GPIO component doesn't affect clock nets (higher priority routes first)

- [x] Route Lockfile (COMPLETED ✓)
  - **Reference:** GAP2 Section 2 (Pillar B)
  - **Status:** Full lockfile system implemented with CLI integration
  - **Implementation:**
    - RouteLockfile struct with JSON serialization (serde) in route_lockfile.rs
    - LockedRoute stores exact waypoints, length, layer transitions, and endpoint hash
    - LockfileManager handles validation and selective rerouting
    - Endpoint validation detects when components move
    - Route invalidation system marks nets for rerouting
    - Human-readable JSON format for Git-friendly diffs
    - Deterministic route sorting by net name
  - **File Format:** .hw.routes.lock with version, board name, grid metadata, and routes array
  - **CLI Integration:**
    - Auto-generates `.hw.routes.lock` on every successful build
    - Loads lockfile at compilation start if exists
    - CLI flags: `--no-lockfile` (disable), `--force-reroute` (ignore lockfile)
    - Integrated with routing pipeline in build.rs
  - **Features:**
    - Load/save lockfile from disk
    - Validate grid dimensions match current board
    - Get locked routes with endpoint validation
    - Invalidate routes when components move or collisions detected
    - Track invalidation reasons for logging
  - **Test Results:** All 8 unit tests + 2 integration tests passing
  - **Integration Test Validation:**
    - ✅ Full workflow test: Route 5 nets → save lockfile → move 1 component → reroute → verify only 1 net changed
    - ✅ Collision detection test: Invalidate blocked routes while preserving valid ones
    - ✅ Minimal diff verified: Moving Net_3 endpoint 2mm only changes Net_3 waypoints, other 4 nets preserved exactly
    - ✅ Lockfile save/load cycle works correctly with temp files
    - ✅ CLI integration tested: Default, --no-lockfile, --force-reroute all working
  - **Verification:** Git diffs show only changed routes ✅ (validated in integration test)

---

## Phase 5: Universal Compiler (v0.3.0+)

**Reference:** GAP3 Part 2-3, GAP4 (Complete Documents)

### 6.1 Behavioral Synthesis

- [ ] Behavior Block Parser
  - **Reference:** GAP3 Section "Part 2", GAP4 Section "Part 1"
  - Add `behavior:` block parsing
  - Implement event syntax (on rising_edge, on falling_edge, always)
  - Add behavioral statement parsing

- [ ] Operator Overloading
  - **Reference:** GAP3 Section "Part 2", GAP4 Section "Part 1"
  - Design operator definition syntax
  - Implement operator lookup system
  - Create standard library operator definitions

- [ ] AST Macro Expansion
  - **Reference:** GAP3 Section "Part 2", GAP4 Section "Part 1"
  - Implement behavioral statement expansion
  - Expand `Sum = A + B` to ripple carry adder
  - Integrate with module flattening
  - **Test:** Compile 64-bit ALU from behavioral description

---

### 6.2 Physics Engine

- [ ] Material Database Extensions
  - **Reference:** GAP3 Section "Part 2", GAP4 Section "Part 2"
  - Add electrical properties (resistivity, permittivity)
  - Add thermal properties (conductivity, specific heat)
  - Update material parser

- [ ] Physics Calculations
  - **Reference:** GAP3 Section "Part 2", GAP4 Section "Part 2"
  - Implement RC delay calculation
  - Implement current density calculation
  - Implement thermal rise calculation
  - **Test:** Calculate properties for simple traces

- [ ] Auto-Fix System
  - **Reference:** GAP3 Section "Part 2", GAP4 Section "Part 2"
  - Implement buffer insertion for timing fixes
  - Implement trace widening for current fixes
  - **Test:** Auto-fix resolves timing and current violations

---

## Phase 6: Performance Optimization (Ongoing)

**Reference:** GAP1 Section 3.4

### 7.1 Immediate Optimizations

- [ ] Memory Allocator Swap (v0.1.4.1)
  - **Reference:** GAP1 Section 3.4 (Gotcha 1)
  - 2 lines of code, 20-40% speedup
  - **Priority:** DO THIS NOW

---

### 7.2 Post-Parallel Optimizations

- [ ] Rayon Parallel Iterators (v0.1.4.2+)
  - **Reference:** GAP1 Section 3.4 (Gotcha 3 Fallback)
  - Use for physics calculations
  - 4-8x speedup on 8-core systems

---

### 7.3 Post-Profiling Optimizations

- [ ] Unsafe Bounds Check Removal (v0.1.4.3+)
  - **Reference:** GAP1 Section 3.4 (Gotcha 2)
  - Profile first, optimize second
  - Apply to A* hot paths
  - 10-20% speedup

- [ ] SIMD Vectorization (v0.1.4.3+)
  - **Reference:** GAP1 Section 3.4 (Gotcha 3)
  - Use for physics math
  - 4-8x speedup
  - Complex, defer until after core functionality

---

## Documentation & Testing

### 8.1 Documentation Updates

- [ ] Sync Language Spec
  - **Reference:** GAP1 Section 6.1
  - Mark unimplemented features as "Planned"
  - Remove examples that don't work
  - **Test:** All examples in spec compile successfully

---

### 8.2 Linting & Validation

- [ ] Unused Definition Warnings
  - **Reference:** GAP1 Section 6.2
  - Add linting pass for unused definitions
  - Warn about unreachable code
  - Warn about undefined references

- [ ] Constraint Validation
  - **Reference:** GAP1 Section 2.2
  - Validate material category matches usage context
  - Enforce mechanical keepout regions
  - Apply profile constraints during routing
