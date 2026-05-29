read C:\Users\olowo\Downloads\Code\Hardware-Script\Docs\v0.1.6\THE-COMMIT-GATE-ARCHITECTURE-info.md for more info

## Part 7: Implementation Roadmap

### Sprint 5: The Commit Gate (CRITICAL - 1 week)

**Goal**: Never export broken designs in Architecture Mode

#### Task 5.1: Implement Phantom Buffer
- [x] Voxel Engine works in "Phantom Buffer" (already in RAM)
- [x] Results only "Realized" (exported) if physics check returns `Ok(())`
- [x] Add `BuildResult` enum: `Success`, `FailedValidation`, `FailedParse`

#### Task 5.2: The Commit Gate
- [x] Add gate between Phase 2 (validation) and Phase 4 (export)
- [x] Check: `if violations.is_empty() || mode == ArtistMode`
- [x] If gate closes: Delete any partial exports, print error report, exit with code 1

#### Task 5.3: Mode Detection
- [x] Detect Artist Mode: `space.implements_module.is_none()`
- [x] Detect Architecture Mode: `space.implements_module.is_some()`
- [x] Add `--force-export` flag for debugging (overrides gate)

#### Task 5.4: Error Reporting Enhancement
- [x] When gate closes, show **actionable** error messages
- [x] Include code snippets with line numbers (foundation laid for future integration)
- [x] Show suggested fixes (e.g., "Change z:4 to z:2")
- [ ] Add "Physical Integrity Error" screen in viewer (instead of broken 3D)

### Sprint 5.5: Immediate Z-Axis Validation (HIGH - 3 days)

**Goal**: Catch floating/buried components at placement time

#### Task 5.5.1: Substrate Bounds Checking
- [x] Add `validate_z_position()` to component placer
- [x] Check `z < substrate.min_z` → Error: Component buried
- [x] Check `z > substrate.max_z` → Error: Component floating
- [x] Include gap size in error message

#### Task 5.5.2: Error Types
- [x] Add `ComponentBuriedInSubstrate` error variant
- [x] Add `ComponentFloatingInAir` error variant
- [x] Include substrate bounds and component position in error

#### Task 5.5.3: Testing
- [x] Test: Component at `z:4` with substrate ending at `z:2` → Error
- [x] Test: Component at `z:2` (substrate.max_z) → Success
- [x] Test: Component at `z:-1` with substrate starting at `z:0` → Error

### Sprint 6: Spatial Variables (MEDIUM - 1 week)

**Goal**: Enable structured assembly with spatial context

#### Task 6.1: Anchor System Enhancement
- [x] Add `substrate.min_z` and `substrate.max_z` as anchors
- [x] Add `last.top`, `last.bottom` for Z-axis relative positioning
- [x] Support arithmetic: `substrate.max_z + 1um`

#### Task 6.2: Constraint Solver Updates
- [x] Resolve `substrate.max_z` to actual Z coordinate
- [x] Support expressions: `substrate.max_z + offset`
- [x] Validate that result is still within valid bounds

#### Task 6.3: Documentation
- [x] Document spatial variables in language spec
- [x] Provide examples of structured vs hardcoded assembly
- [x] Explain when to use each approach

#### Task 6.4: Modular View Orientation (COMPLETE)
- [x] Add `view` property to `render` block in `space` AST
- [x] Implement `SpaceView` enum: `vertical_standing` (Y-up) vs `horizontal_floor` (Z-up)
- [x] Update DXF exporter for orientation-aware projection (XY vs XZ planes)
- [x] Update GLB exporter for automatic mesh axis-swapping (Z-up vs Y-up)
- [x] Verified cross-perspective consistency with dedicated test files

### Sprint 7: Visual Integrity Debugger (CRITICAL - 1 week)

**Goal**: Provide visual feedback when Commit Gate closes

#### Task 7.1: Integrity Mesh Generator
- [ ] Create `IntegrityMeshGenerator` that renders violations as low-poly bounding boxes
- [ ] Floating components (P44): Glowing **Yellow** boxes
- [ ] Short circuits (P42): Glowing **Red** boxes
- [ ] Disconnected islands (P41): **Blue** boxes with gap measurements
- [ ] Include substrate outline as semi-transparent reference

#### Task 7.2: Debug Export System
- [ ] Create `build/debug/` folder only when:
  - Build fails in Architecture Mode, OR
  - `--debug` flag is used
- [ ] Export `integrity_report.glb` with violation visualization
- [ ] Export `violation_log.txt` with detailed error descriptions
- [ ] Export `physics_state.json` with full validation data

#### Task 7.3: Debug Output Structure
- [ ] Normal builds: `build/SpaceName/board.glb` (flat structure)
- [ ] Failed builds: `build/debug/integrity_report.glb` (debug subfolder)
- [ ] Ensure debug files are **never** in production output folder
- [ ] Add `.gitignore` entry for `build/debug/`

#### Task 7.4: Studio Integration
- [ ] Update Hardware Script Studio to detect `integrity_report.glb`
- [ ] Show "Integrity Violation View" mode in viewer
- [ ] Display violation annotations (labels, measurements)
- [ ] Link violations to source code line numbers

### Sprint 8: Intentional Overlap Waivers (COMPLETE)

**Goal**: Allow intentional substrate embedding for silicon design

#### Task 8.1: Waiver Attributes (Parser) [DONE]
- [x] Add `waivers` field to `PourPlacement` AST
- [x] Add `waivers` field to `ComponentPlacement` AST
- [x] Parse unified merge syntax: `merge: true` (Global)
- [x] Support granular list merging: `merge: [source, drain]`

#### Task 8.2: Waiver Types (Engine) [DONE]
- [x] Define `MergeWaiver` enum (None, All, Specific)
- [x] Implement additional intent booleans: `floating`, `isolated`, `virtual`, `locked`
- [x] Persist `waivers` in `PourMetadata` and `PlacementParams`

#### Task 8.3: Collision Engine Updates [DONE]
- [x] Update P12/P42 checkers to respect `merge: true`
- [x] Implement surgical waiver logic for `merge: [list]`
- [x] Enforce Vacuum Rule (P44 Floating requires explicit `floating: true`)
- [x] Update P12 (Collision) checker to skip waivered overlaps
- [x] Update P44 (Floating) checker to skip waivered components
- [x] Native Reporting: Replaced legacy `println!` with `WaiverApplied` diagnostics

#### Task 8.4: Testing [DONE]
- [x] Test: Bulk pour with `allow_substrate_overlap` → Success
- [x] Test: Bulk pour without waiver → P42 error
- [x] Test: Merged regions with `allow_component_overlap` → Success
- [x] Test: Air-gap device with `allow_floating` → Success (Verified in `tests/sprint8_waivers/test_waivers.hw`)

#### Task 8.5: Documentation [DONE]
- [x] Native Waiver Diagnostic header implementation (`waiver[W001]`)
- [x] Document waiver system in [WAIVER-SYSTEM.md](./WAIVER-SYSTEM.md)
- [x] Explain when each waiver type is appropriate
- [x] Provide silicon design examples (doping wells)
- [x] Warn about waiver misuse (defeats safety checks)


### Sprint 9: Batch Validation (MEDIUM - 2 days)

**Goal**: Collect all violations before closing Commit Gate

#### Task 9.1: Violation Collector
- [x] Create `ViolationCollector` that accumulates errors during Phase 1
- [x] Don't abort on first error - continue unrolling
- [x] Collect violations by type (P41, P42, P43, P44)
- [x] Track source location for each violation

#### Task 9.2: Pattern Detection
- [x] Detect repeated violations in loops
- [x] Group violations by pattern (e.g., "All Adder[i] floating")
- [x] Suggest loop-level fixes instead of per-instance fixes
- [x] Show first 10 violations, summarize rest

#### Task 9.3: Error Reporting Enhancement
- [x] Show violation count by type
- [x] Highlight patterns: "64 violations in loop at line 45"
- [x] Provide single fix suggestion for patterns
- [x] Include code snippet with suggested change

#### Task 9.4: Testing
- [x] Test: 64-bit bus with floating errors → Show individual snippets + summary note
- [x] Test: Verify pattern detection groups manual names (Adder0, Adder1) correctly
- [x] Test: Validate dynamic suggestions match actual source code lines
- [x] Test: Verify error cap (50 errors) prevents terminal flood while remaining informative

