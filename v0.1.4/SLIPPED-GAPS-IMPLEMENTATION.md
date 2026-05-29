# Hardware Script Compiler - Slipped Gaps Implementation

**Version:** v0.1.4+  
**Last Updated:** March 31, 2026

---

## 1. Module Flattening

**Reference:** GAP1 Section 2.1  
**Problem:** `add DSP_64Bit_Core named MainDSP` throws "Unknown component type"

- [x] Implement recursive module expansion in `two_pass_compiler.rs`
- [x] Process layout blocks to determine component positions
- [x] Calculate module bounding boxes for parallel routing
- [x] Prefix component names: `{instance}.{component}`
- [x] Translate module routes to global coordinates
- [x] Handle nested modules (modules containing modules)
- [x] Test: Module instantiation expands all internal components

**Status:** ✅ COMPLETED (March 31, 2026)
**Implementation:** `hwc-compiler/src/module_flattener.rs`, integrated in `two_pass_compiler.rs`
**Details:** See GAP1 Section 2.1 for full implementation documentation

---

## 2. Expression Evaluation in Compiler

**Reference:** GAP1 Priority 2 Item 4  
**Problem:** `at [x: 10 + (i*5), y: 20, z: 1]` doesn't work

- [x] Create `ExpressionEvaluator` in `hwc-compiler/src/expression_evaluator.rs`
- [x] Implement `evaluate()` for literals, variables, binary ops, unary ops
- [x] Integrate with for-loop flattening in `two_pass_compiler.rs`
- [x] Set loop variable in evaluator context
- [x] Evaluate coordinate expressions during comptime
- [x] Test: Arithmetic in coordinates works in for loops

**Status:** ✅ COMPLETED (March 31, 2026)
**Implementation:** Layout blocks now support for loops and if conditionals with full expression evaluation
**Details:** See GAP1 Priority 2 Item 4 for full implementation documentation

---

## 3. Behavioral Synthesis Compiler Integration

**Reference:** GAP3 Part 2, GAP4, GAP5  
**Problem:** Logic blocks validate but don't generate hardware

- [x] Implement operator overloading system in `hwc-compiler`
- [x] Map `+` → RippleCarryAdder from stdlib
- [x] Map `-` → Subtractor from stdlib
- [x] Map `&`, `|`, `^` → Logic gates from stdlib
- [x] Map `<<`, `>>` → Barrel shifters from stdlib
- [x] Map `if/else` → 2-to-1 MUX
- [x] Map `match` → N-to-1 MUX
- [x] Map `Reg()` → D flip-flop with clock/reset routing
- [x] Generate physical routing between synthesized components
- [x] Populate HardwareIR with logic-generated components
- [x] Test: `let result = A + B` generates adder component

**Status:** ✅ COMPLETED (March 31, 2026)
**Implementation:** `hwc-compiler/src/logic_synthesizer/`, integrated in `two_pass_compiler.rs`
**Details:** See GAP5 for full implementation documentation

---

## 4. Physics Engine

**Reference:** GAP3 Part 2, GAP4 Section "Part 2"  
**Problem:** No timing/power validation

- [x] Add electrical properties to `materials.hw` (resistivity, permittivity)
- [x] Add thermal properties (conductivity, specific heat)
- [x] Implement RC delay calculation in `hwc-physics/src/electrical.rs`
- [x] Implement current density calculation in `hwc-physics/src/electrical.rs`
- [x] Implement thermal rise calculation in `hwc-physics/src/thermal.rs`
- [x] Validate against profile limits (max_delay, max_current_density)
- [x] Generate physics violation errors with suggestions
- [x] Implement buffer insertion for timing fixes (auto-fix suggestions)
- [x] Implement trace widening for current fixes (auto-fix suggestions)
- [x] Test: 4GHz clock net validates timing constraints

**Status:** ✅ COMPLETED (March 31, 2026)
**Implementation:** `hwc-physics/src/`, CLI command in `hwc-cli/src/commands/physics.rs`
**Details:** See GAP4 Section "Part 2" for full implementation documentation

---

## 5. HDI Via Support

**Reference:** GAP1 Section 5.3  
**Problem:** All vias drill through entire board

- [x] Create `ViaType` enum: ThroughHole, Blind, Buried, Microvia
- [x] Implement automatic via type classification based on layer span
- [x] Add via type cost calculation to router
- [x] Update via stamping to respect layer ranges
- [x] Generate separate drill files per via type (PTH, NPTH, 1-2, 3-4)
- [x] Test: 10-layer board with mixed via types

**Status:** ✅ COMPLETED (Pre-existing implementation verified March 31, 2026)
**Implementation:** `hwc-engine/src/geometry_router/types.rs`, `hwc-export/src/excellon.rs`
**Details:** See GAP1 Section 5.3 for full implementation documentation
**Test Results:** All 18 unit tests pass (10 via type tests + 8 excellon export tests)

- []End-to-end `.hw` file test deferred pending routing engine coordinate transformation fixes. HDI via functionality is complete and tested at the Rust implementation level. Full integration test required after routing engine completion.

---

## 6. Pad Shapes & Solder Mask

**Reference:** GAP1 Section 5.4  
**Problem:** Only Rectangle pads, no solder mask layers

- [x] Extend `PadShape` enum: Circle, Obround, Polygon, RoundedRect
- [x] Update component definitions in stdlib with pad shapes
- [x] Generate `.gts` (Top Solder Mask) layer
- [x] Generate `.gbs` (Bottom Solder Mask) layer
- [x] Generate `.gtp` (Top Solder Paste) layer
- [x] Generate `.gbp` (Bottom Solder Paste) layer
- [x] Implement solder mask expansion (0.05-0.1mm larger than pad)
- [x] Implement paste reduction (10-20% smaller than pad)
- [x] Test: Component with pad shapes generates all 4 additional layers

**Status:** ✅ COMPLETED (March 31, 2026)
**Implementation:** 
- `hwc-engine/src/placement/component_definition.rs` - PadShape enum (pre-existing)
- `hwc-export/src/solder_layers.rs` - Solder mask/paste layer generation
- `hwc-engine/src/netlist.rs` - Added pad_shape field to PinData
- `hwc-stdlib/components.hw` - Updated with pad_shapes definitions
**Details:** 
- PadShape enum already existed with all required variants (Circle, Rectangle, Obround, Polygon, RoundedRect)
- Parser already supported pad_shapes in layout blocks
- Added pad shape storage in netlist arena for export access
- Created solder_layers module with automatic sizing (mask +0.075mm, paste -15%)
- Integrated into Gerber export workflow
- All 3 unit tests pass
- End-to-end test verified: generates board.gts, board.gbs, board.gtp, board.gbp files

---

## 7. Route Lockfile CLI Integration

**Reference:** GAP2 Pillar B  
**Problem:** Lockfile exists but not used in build pipeline

- [x] Auto-generate `.hw.routes.lock` on every successful build
- [x] Load lockfile at start of compilation
- [x] Validate locked routes against current board state
- [x] Detect endpoint changes (component moved)
- [x] Detect collisions with new components
- [x] Rip up only invalid routes, preserve valid ones
- [x] Save updated lockfile after routing
- [x] Add CLI flags: `--no-lockfile`, `--force-reroute`
- [x] Test: Move one component, verify only 1-3 nets reroute

**Status:** ✅ COMPLETED (March 31, 2026)
**Implementation:** `hwc-cli/src/commands/build.rs`, `hwc-engine/src/geometry_router/route_lockfile.rs`
**Details:** CLI integration complete with all flags working

---

## Success Criteria

**Module Flattening:**
- Module instantiation works end-to-end
- Nested modules expand correctly
- All 543+ tests pass

**Expression Evaluation:**
- Arithmetic in coordinates works
- For loops with expressions generate correct placements

**Behavioral Synthesis:**
- Logic blocks generate physical hardware
- Standard library operators work
- Clock domain tracking functional

**Physics Engine:**
- Timing violations detected
- Current violations detected
- Auto-fix resolves violations

**HDI Vias:**
- Blind/buried vias work
- Separate drill files generated

**Pad Shapes:**
- All pad shapes supported
- Solder mask/paste layers generated

**Route Lockfile:**
- Minimal rerouting on component moves
- Git diffs show only changed routes

