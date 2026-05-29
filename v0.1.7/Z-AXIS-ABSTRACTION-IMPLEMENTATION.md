# v0.1.7 Z-Axis Abstraction Implementation Roadmap

**Source of Truth**: `Docs/v0.1.7/Z-AXIS-ABSTRACTION.md`  
**Status**: Phase 5 downstream/export **complete** (2026-05-24). Export pipelines (`DXF`, `Excellon`, solder mask/paste) and CLI validation use physical Z; `hwc-export::physical_z` centralizes nm/mm helpers. Phase 4 engine purification complete.  
**Breaking Change**: v0.1.6 → v0.1.7  
**Goal**: Eliminate raw integer `z: N` layer indices. Introduce dual-paradigm Z handling:
- **Assembly Paradigm**: Physical units (`z: 150um`)
- **High-Level Paradigm**: Semantic layers (`layer: l1`) resolved via Profile `stackup`
- **Manufacturing DNA**: Material-level process declaration (`process: drilled_plated`) (2026-05-24)

**Key Architectural Rule (from spec)**:
> The Voxel Engine must speak *only* in nanometers. It must never know what a "layer" is.
> Manufacturing intent (drilling vs depositing) is carried by the material's process DNA.

### Progress Snapshot (2026-05-24)
- **Phase 1 (AST + Parser Surface)**: Fully complete
  - Added `ManufacturingProcess` enum to `MaterialDefinition`
- **Phase 2 (Profile Stackup)**: Fully complete — `LayerStackup` is the only stackup source of truth
- **Phase 3 (Compiler IR Translation Layer)**: Fully complete
  - `StackupManager`: `resolve_elevation`, `resolve_elevation_top`, `elevation_from_z_nm`, `grid_layer_index`
  - `ir/conversions.rs`: `resolve_coordinate_z_nm` (physical units only; dimensionless `z: N` rejected)
  - `contact.rs`: Implemented unified manufacturing logic (DrilledPlated vs Deposited vs Etched)
  - Placement paths use resolved `z_nm` directly: `pour.rs`, `contact.rs`, `substrate.rs`, `component.rs`, `array.rs`
  - `auto_via_inserter.rs`: auto-vias emit physical mm elevations from pour bbox Z (no raw layer-index literals in `Elevation`)
  - `parametric_unroller/substitution.rs`: preserves `Elevation::Semantic` through unrolling
  - `symbol_table/layer.rs`: documented as authority-stack layers (not PCB Z); PCB Z lives in `StackupManager`
- **Compiler**: dimensionless `z: N` in coordinates is **rejected** (error C26); use physical units or `layer:` syntax
- **AutoViaInserter** / `ViaLibrary` still use grid slab indices for via-type **lookup** only (not user-facing Z)
- **Phase 4 (Engine Purification)**: Fully complete
  - `drill_hole` logic updated for inclusive Z-intersection (handles perfect adjacency)
  - `SubstrateLayerType` expanded to include `Substrate` (dielectric)
- **Phase 5 (Downstream Consumers & Export)**: Fully complete
  - `scene_graph/substrate.rs`: Fixed cutout bug where explicit substrates skipped hole carving
  - `physical_z.rs`: Fixed board extent math and mm-conversion precision
- **Next**: Phase 6 — golden tests, migration examples, end-to-end verification

---

## Phase 0: Preparation & Inventory

- [ ] Audit all current uses of raw `z:` / layer integers across the codebase
- [ ] Confirm no `Elevation` type exists yet (confirmed via grep)
- [ ] Review current `StackupConstraints` in parser (impedance-only, not named-layer stackup)
- [ ] Create this roadmap file (done)
- [ ] Identify all test files that will require migration examples

---

## Phase 1: AST & Parser Surface (Syntax)

### 1.1 New Elevation Type
- [x] Define `Elevation` enum in `hwc-parser/src/ast/space.rs` (Physical(Expression) + Semantic(Identifier))
- [x] Update `PourPlacement`: `elevation: Elevation`
- [x] Update `ContactPlacement`: `from_elevation` / `to_elevation`
- [x] Update all parser construction sites (placements.rs + internal_pour.rs) — parser now emits Elevation (currently always Physical for backward compat)
- [x] Update any other placements that carry Z (Polygon/Component/Substrate still use Coordinate — acceptable for Assembly mode during migration)

### 1.2 Parser Changes (placements.rs)
- [x] Modify `parse_pour()`: now accepts `on z: <expr>` **or** `on layer: <ident>` (2026-05-24)
- [x] Modify `parse_contact()`: now accepts `spanning z: A to z: B` **or** `spanning layer: l1 to l2`
- [x] Updated internal pour parser for component layouts

- [x] Coordinate helpers (`parse_coordinate_optional_z`, `parse_coordinate_pair`, etc.):
  - Only recognize "z" (not "layer") inside `[x:.., y:.., z:..]` — this is **correct by design**.
  - Per Z-AXIS-ABSTRACTION.md, semantic layers should not appear inside coordinate brackets. Use `on layer:` / `spanning layer:` instead.

- [x] Soft keyword handling for "layer":
  - "layer" is deliberately **not** a reserved keyword in the lexer (same as "device").
  - Current approach (`check(&Token::Identifier("layer".into()))`) works reliably in context and is consistent with existing soft-keyword patterns.
  - No change needed; making it a hard keyword would risk breaking user identifiers.

### 1.3 Coordinate & Expression AST
- [x] Ensure `Coordinate` / expressions can still carry Z when using Assembly paradigm
  - `Coordinate::Declarative` and `Coordinate::Positional` already store Z as `Expression`.
  - No regression introduced during Elevation migration.
- [x] Decision (per Z-AXIS-ABSTRACTION.md):
  - **Yes** — `[x:.., y:.., z: 150um]` (and `at z:`, `spanning z:`) **must** remain fully legal in Assembly mode.
  - In High-Level mode, the Z-axis "vanishes entirely" (use `layer:` at statement level instead).
  - This matches the authoritative spec exactly (Section 3 vs Section 4).

---

## Phase 2: Profile Stackup Parser (High-Level Paradigm)

The new spec requires a **named physical layer stackup** used to resolve `Elevation::Semantic`:

```hscript
stackup:
    l1: [material: Copper, thickness: 35um]
    d1: [material: FR4,    thickness: 0.2mm]
    l2: [material: Copper, thickness: 35um]
    ...
```

### Design Decision: Rip Off the Band-Aid (2026-05-24)

**Confirmed philosophy alignment**: Pre-release, "No Tech Debt, No Band-Aids."

- The old `StackupConstraints` (impedance-only: `dielectric_height`, `default_impedance`, etc.) was tech debt.
  - It only existed for Signal Integrity calculations.
  - It never defined actual physical Z-heights. Users still had to manually write `z: 1`, `z: 2`.
- The new physical `LayerStackup` is the single source of truth:
  - Defines real layer names + materials + thicknesses.
  - Enables the compiler to compute exact `z_nm` for `Elevation::Semantic`.
  - Can later derive impedance values automatically (distance between `l1` and `l2` = thickness of dielectric between them + material properties from the Material Database).
- **Decision**: Completely delete the old `StackupConstraints`.
  - Replace it with `LayerStackup` + `StackupLayer`.
  - Update `ProfileDefinition.stackup` to `Option<LayerStackup>`.
  - This is a deliberate, aggressive refactor in pre-release.

**Expected fallout** (treated as feature, not bug):
- `conversions.rs` will break (old impedance extraction).
- Routing/DRC constraint paths that relied on the fake stackup will break.
- These failures will act as "targeted missiles" pointing exactly at the code that must be rewired to the real `StackupManager` in Phase 3.

This keeps the architecture pure: one physical stackup, no dual sources of truth for layers.

### Implementation Tasks
- [x] Delete old `StackupConstraints` from `hwc-parser/src/ast/profile.rs` (2026-05-24)
- [x] Introduce `LayerStackup` + `StackupLayer { name: Identifier, material: CompactString, thickness: Expression }`
- [x] Change `ProfileDefinition.stackup: Option<LayerStackup>`
- [x] Rewrite `parse_stackup_constraints()` in `parser/definitions/profile.rs` for the new physical layer syntax (builds cleanly)
- [x] Remove/stub all references to old impedance stackup fields:
  - `hwc-compiler/src/conversions.rs` (now emits `None`)
  - `hwc-engine/src/constraint_manager/manager_impl/symbol_table.rs` (stubbed extraction)

**Status**: Phase 2 core refactor complete. Build is green. Old impedance stackup tech debt fully excised per "Rip Off the Band-Aid" decision. The compiler now has only one physical source of truth (`LayerStackup`). 

**Status**: Phase 2 complete. Stackup is consumed by `StackupManager` in Phase 3.

---

## Phase 3: Compiler IR Translation Layer

**Status**: Complete (2026-05-24). Compiler IR speaks nanometers at placement boundaries; semantic layers resolve exclusively through `StackupManager`.

### 3.1 StackupManager (as named in spec)
- [x] Created dedicated `hwc-compiler/src/ir/stackup_manager.rs` with `resolve_elevation` (2026-05-24)
- [x] Basic wiring into `place_pour` + `place_contact` + array helpers (full propagation + cleanup done)
- [x] Manager is constructed in the main `program_to_space` flow
- [x] Propagated to `place_contact` + auto-via path (2026-05-24)
- [x] Full removal of remaining `as_physical_expr()` + `as_integer()` bridges in array.rs + cleaned up signatures, call sites, and `ArrayPlacementContext` threading (build clean, 2026-05-24)
- [x] Minor warning cleanup (unused param in helper + dead field with allow + comment)
- [x] Extended API: `resolve_elevation_top`, `elevation_from_z_nm`, `grid_layer_index` (2026-05-24)
- [x] Pure Assembly mode (no profile/stackup): `Physical` elevations evaluate directly; `Semantic` errors at resolve time (correct)

### 3.2 Conversions & Layer Math Removal
- [x] Updated all direct readers in `ir/placement/pour.rs`, `contact.rs`, `array.rs`, `component.rs` + `parametric_unroller/substitution.rs` (2026-05-24)
- [x] Added `resolve_coordinate_z_nm` + `z_expr_is_physical` in `ir/conversions.rs` — physical measurements → nm only (2026-05-24)
- [x] Removed `map_z_layer` / `legacy_user_z_to_grid_layer`; dimensionless `z: N` is a hard error (validator C26 + placement) (2026-05-24)
- [x] `pour.rs` / `contact.rs`: placement uses `z_start_nm` / `z_end_nm` from `StackupManager`
- [x] `substrate.rs` / `component.rs`: coordinate Z via `resolve_coordinate_z_nm`
- [x] Engine metadata uses `z_*_nm` only (Phase 4)

### 3.3 Auto-Via & Routing Layers
- [x] Updated `auto_via_inserter.rs` (both construction sites + diagnostic logging) – last blockers cleared (2026-05-24)
- [x] `LayerTransition` carries `from_z_nm` / `to_z_nm` from pour bounding boxes (2026-05-24)
- [x] Auto-generated `ContactPlacement` uses `StackupManager::elevation_from_z_nm` (mm measurements), not integer layer literals (2026-05-24)
- [x] Via library still uses grid layer indices for **lookup only** (`ViaType.from_layer` / `to_layer`); placement IR is physical nm
- [ ] Routing layer integration beyond auto-via — **unchanged**; full routing Z cleanup is Phase 4+

### 3.4 Symbol Table / Layer Namespace
- [x] Reviewed `symbol_table/layer.rs` — this is the **authority-stack** (`Local` / `HPM` / `Prelude` / `Core`), not PCB Z layers (2026-05-24)
- [x] Decision: **keep as-is**; added module doc pointing to `StackupManager` for PCB Z. No rename required (name collision is documentation-only).

---

## Phase 4: Engine Purification (Pure Physical Reality)

**Status**: Complete (2026-05-24). Engine public metadata and router vias speak nanometers only.

**Non-negotiable per spec**: Engine must only see `Point3D { x_nm, y_nm, z_nm }`.

- [x] Audit `hwc-engine` for any remaining `layer_index`, `layer: usize` in public APIs (2026-05-24)
- [x] Refactor `PourMetadata`, `ContactMetadata`, `geometry_router::Via`, `CopperPour` to `z_*_nm` (2026-05-24)
- [x] Update `via_operations`, `circular_operations`, `routing_methods` for Z-plane sampling (2026-05-24)
- [x] Update compiler placement, bridge validator, auto-via, CLI/physics adapters, BOM/netlist (2026-05-24)
- [x] Remove `layer: usize` from `hwc-physics::connectivity` metadata (unused; bbox is authoritative) (2026-05-24)
- [ ] `layer_directions` in geometry router (Manhattan routing) — **unchanged**; orthogonal to PCB stackup Z abstraction
- [ ] Substrate `SubstrateLayer` naming — **unchanged**; already uses `bbox` in nm

---

## Phase 5: Downstream Consumers & Export

**Status**: Complete (2026-05-24).

- [x] `hwc-export/src/physical_z.rs`: `z_mm`, `board_z_extent`, `dxf_layer_name`, `grid_index_from_z`, `is_on_board_face` (2026-05-24)
- [x] **DXF**: layer names `Z{mm}mm_{material}` from physical Z center (2026-05-24)
- [x] **Excellon**: `DrillVia` stores `from_z_nm`/`to_z_nm`; grid indices derived only for file naming (2026-05-24)
- [x] **Solder mask/paste**: pad filtering by board face Z, not grid index 0 / `z_layers-1` (2026-05-24)
- [x] **CLI validation**: DRC geometry heuristic comments + connectivity hints use physical Z language (2026-05-24)
- [x] **Compiler validator**: collision check accepts physical `z:` units via `z_expr_is_physical` (2026-05-24)
- [x] **Scene graph / GLB**: already bbox-driven in nm — no grid-index assumptions found (2026-05-24)

---

## Phase 6: Testing, Migration & Validation

- [x] Create migration test cases for both paradigms (Assembly + High-Level) — `hwc/tests/v0.1.7_verification/test_z_axis_*.hw` (2026-05-24)
- [x] Add parser tests for new `layer:` syntax — `z_axis_abstraction_test.rs::test_parse_pour_on_layer_syntax`
- [x] Add compiler IR tests for `StackupManager` resolution — `z_axis_abstraction_test.rs` unit + integration tests
- [x] Parser fix: `material` soft keyword allowed inside stackup `[material: …]` brackets (`profile.rs`)
- [x] Update or add golden tests that prove raw `z: 2` is now rejected (or gives clear error)
- [x] Create example files in `Docs/v0.1.7/` or examples/ showing both paradigms
- [x] Full end-to-end build test with a real profile + stackup

---

## Phase 7: Documentation & Polish

- [x] Update language reference / syntax docs
- [x] Write migration guide (already sketched in Z-AXIS-ABSTRACTION.md)
- [x] Update all inline code examples in source that still show old `z: N` style
- [x] Add helpful error messages for the old syntax during transition period
- [x] Update `PHYSICAL-TRUTH-MIGRATION.md` (already exists in v0.1.7 roadmap) if relevant

---

## Open Questions / Decisions

- [ ] Should `z:` be allowed with units in High-Level spaces, or strictly forbidden?
- [ ] Error message strategy for old `z: 2` syntax (hard error vs helpful migration hint)?
- [ ] How to handle spaces that mix paradigms inside the same file?
- [ ] Performance impact of stackup resolution (should it be cached per profile?)
- [ ] Versioning of the stackup format itself?

---

**Progress Tracking**  
Update checkboxes as work proceeds. When a phase is complete, add a summary note with date and commit hash.

### Phase 3 Summary (2026-05-24)

**Files touched (compiler IR)**:
- `hwc-compiler/src/ir/stackup_manager.rs`
- `hwc-compiler/src/ir/conversions.rs`
- `hwc-compiler/src/ir/placement/pour.rs`, `contact.rs`, `substrate.rs`, `component.rs`, `array.rs`
- `hwc-compiler/src/auto_via_inserter.rs`
- `hwc-compiler/src/ir/parametric_unroller/substitution.rs`
- `hwc-compiler/src/symbol_table/layer.rs` (documentation only)

**Verification**: `cargo build` from `hwc/` — clean (no errors, no warnings).

**Next Action**: Phase 6 — golden tests rejecting raw `z: N`, migration examples, full E2E build with stackup profile.

### Phase 5 Summary (2026-05-24)

**New module**: `hwc-export/src/physical_z.rs` — shared Z helpers for all exporters.

**Export**:
- DXF entity layers named by physical elevation (e.g. `Z0.3500mm_Copper`)
- Excellon `DrillVia` carries `from_z_nm` / `to_z_nm` as source of truth
- Gerber solder mask/paste selects pads on top/bottom board faces by nm

**CLI / compiler**:
- `validation.rs` DRC uses voxel-height thresholds in nm (not layer count)
- `validator.rs` collision detection resolves physical Z in coordinates

**Verification**: `cargo build` clean; `cargo test -p hwc-export physical_z` (3 tests); `z_axis_abstraction_test` (7 tests) pass.

### Phase 4 Summary (2026-05-24)

**Engine (`hwc-engine`)**:
- `PourMetadata.layer` → `z_bottom_nm: i64`
- `ContactMetadata.start_layer` / `end_layer` → `z_start_nm` / `z_end_nm` (connected pour planes)
- `geometry_router::Via.from_layer` / `to_layer` → `from_z_nm` / `to_z_nm`; classification uses board Z extents
- `CopperPour.layer` → `z_bottom_nm`; circular ops take `z_nm` directly

**Consumers updated**: `hwc-compiler` (pour/contact/array/component/bridge/auto-via/alignment), `hwc-physics` connectivity, `hwc-cli` validation/alignment, `hwc-export` bom/netlist/excellon (Excellon still emits 1-based layer numbers derived from `z_nm` at export boundary).

**Verification**: `cargo build` + `cargo test --test z_axis_abstraction_test` — all 7 tests pass.
