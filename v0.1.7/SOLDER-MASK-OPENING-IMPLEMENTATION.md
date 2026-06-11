# Solder Mask Opening (Tented vs. Exposed Vias) — Implementation Checklist

Feature: Item 4 from `tests/via/implementation.md`
Status: **DONE**

## What

When a via passes through the top or bottom board face, the solder mask (protective coating) must either:

- **Exposed (default):** Cut a hole in the solder mask, exposing the copper pad for soldering
- **Tented:** Leave the solder mask intact over the via, sealing it shut

Controlled by two properties on `add contact(...)`:

```hw
add contact(ViaCopper) named MyVia at [x: 5mm, y: 5mm] spanning layer: top to bottom:
    net: SIG
    drill_diameter: 0.3mm
    is_tented: false                    # default: false (exposed)
```

---

## Implementation Steps

### Step 1: Engine — Add `SolderMask` Layer Type ✅

**File:** `crates/hwc-engine/src/voxel_grid/substrate_layers.rs`

- [x] Add `SolderMask` variant to `SubstrateLayerType` enum (after `Substrate`)

```rust
pub enum SubstrateLayerType {
    Pour,
    Contact,
    Substrate,
    SolderMask, // solder mask coating on top/bottom board faces
}
```

- [x] Verify all `match` arms across the codebase handle the new variant (no-op in `drill_via_hole` initially, then full implementation)

---

### Step 2: Engine — Extend `ContactMetadata` ✅

**File:** `crates/hwc-engine/src/space.rs`

- [x] Add two fields to `ContactMetadata`:

```rust
pub struct ContactMetadata {
    // ... existing fields ...
    pub is_tented: bool,
    pub mask_clearance_diameter_nm: Option<i64>,
}
```

- [x] Update all places that construct `ContactMetadata` to include the new fields:
  - `crates/hwc-compiler/src/ir/placement/contact.rs`
  - `crates/hwc-engine/src/auto_via_inserter/placement.rs`
  - `crates/hwc-engine/src/unrolling.rs`

---

### Step 3: Compiler — Read `is_tented` and `mask_clearance_diameter` ✅

**File:** `crates/hwc-compiler/src/ir/placement/contact.rs`

- [x] Read `is_tented` and `mask_clearance_diameter` from contact properties via `get_prop_bool`/`get_prop_nm`
- [x] Passed into `ContactMetadata` at end of `place_contact()`
- [x] Read BEFORE the process branch (DrilledPlated/Etched/Deposited) so it's available in all branches

---

### Step 4: Profile-Driven Solder Mask Expansion ✅

**Parser:** `crates/hwc-parser/src/ast/profile.rs`, `crates/hwc-parser/src/parser/definitions/profile/constraints.rs`
**Engine:** `crates/hwc-engine/src/constraint_manager/types.rs`, `crates/hwc-engine/src/constraint_manager/manager_impl/fabrication.rs`, `crates/hwc-engine/src/constraint_manager/manager_impl/symbol_table.rs`
**Materials:** `crates/hwc-materials/src/constraints.rs`
**Compiler:** `crates/hwc-compiler/src/conversions.rs`, `crates/hwc-cli/src/commands/build_cmd/validation/drc.rs`

- [x] Added `solder_mask_expansion: Option<Measurement>` to `ManufacturingConstraints` AST
- [x] Parser handles it in `constraints.rs`
- [x] Added `solder_mask_expansion_nm: i64` to `FabricationConstraints` (engine) and `ConstraintSet` (hwc-materials)
- [x] Extraction in `symbol_table.rs` with IPC-7351 default 75µm
- [x] Loaded in `fabrication.rs` and `conversions.rs`
- [x] Updated DRC construction in `hwc-cli`

---

### Step 5: Engine — Drill Solder Mask Openings ✅

**File:** `crates/hwc-engine/src/voxel_grid/grid/substrate_ops.rs`

- [x] Extended `drill_via_hole()` signature to accept `is_tented`, `pad_diameter_nm`, `solder_mask_expansion_nm`

```rust
pub fn drill_via_hole(
    &mut self,
    hole_bbox: BoundingBox,
    diameter_nm: i64,
    via_net: NetId,
    clearance_nm: i64,
    is_tented: bool,
    pad_diameter_nm: i64,
    solder_mask_expansion_nm: i64,
)
```

- [x] Added `SolderMask` match arm with profile-driven formula:

```rust
SubstrateLayerType::SolderMask => {
    if !is_tented {
        let opening_diameter = pad_diameter_nm + 2 * solder_mask_expansion_nm;
        // Clamp cutout bbox to mask layer Z-range for correct export slicing
        let mask_cutout_bbox = BoundingBox::new(
            Point3D::new(
                hole_bbox.min.x.max(layer.bbox.min.x),
                hole_bbox.min.y.max(layer.bbox.min.y),
                layer.bbox.min.z,
            ),
            Point3D::new(
                hole_bbox.max.x.min(layer.bbox.max.x),
                hole_bbox.max.y.min(layer.bbox.max.y),
                layer.bbox.max.z,
            ),
        );
        layer.add_cylinder_cutout(mask_cutout_bbox, opening_diameter);
    }
}
```

- [x] Updated all call sites:
  - `contact.rs` — DrilledPlated branch
  - `contact.rs` — `drill_tsv` (defaults to exposed)
  - `contact.rs` — Deposited/Etched branches

---

### Step 6: Auto-Creation of Solder Mask Layers ✅

**File:** `crates/hwc-compiler/src/ir/mod.rs`

- [x] Auto-create solder mask substrate layers at board top/bottom faces before placement loop
- [x] Z-positions derived from `stackup_manager.board_thickness_nm()` (not user-defined space depth)
- [x] Skip if user already defined solder mask layers in stackup

```rust
let stackup_height_nm = stackup_manager.board_thickness_nm();
let mask_thickness_nm = 20_000; // 20µm standard

// Top mask: [stackup_height, stackup_height + 20000]
// Bottom mask: [-20000, 0]
```

- [x] Register `SolderMask` material via `space.material_registry.get_or_register("SolderMask")`

---

### Step 7: Export — Gerber Solder Mask Layer ✅

**File:** `crates/hwc-export/src/solder_layers.rs`

- [x] Extended `collect_pads()` to include exposed via pads in Gerber solder mask export

---

### Step 8: Test File ✅

**File:** `tests/via/test_solder_mask.hw`

- [x] Created test with both tented and exposed vias
- [x] Build succeeds with `cargo run --bin hwc -- build tests/via/test_solder_mask.hw`
- [x] GLB output shows green solder mask with hole at exposed via, intact over tented via
- [x] Both solder mask layers show `Cutouts: 1` in export debug output

---

## Bug Fix: Cutout BBox Z-Range

**Root Cause:** The cutout bbox used the via's full Z range `[0, 1365000]`, but the top solder mask was at `[1365000, 1385000]`. The export slicing uses strict `>` comparison (`cutout.bbox.max.z > z_start` → `1365000 > 1365000` → `false`), so the boundary-touching cutout was invisible.

**Fix:** Clamp the cutout bbox Z-range to the mask layer's own Z-range, so it fully spans the mask thickness and the slicing logic matches it correctly.

**File:** `crates/hwc-engine/src/voxel_grid/grid/substrate_ops.rs`

---

## File Change Summary

| File | Change |
|------|--------|
| `crates/hwc-engine/src/voxel_grid/substrate_layers.rs` | Added `SolderMask` variant to `SubstrateLayerType` |
| `crates/hwc-engine/src/space.rs` | Added `is_tented`, `mask_clearance_diameter_nm` to `ContactMetadata` |
| `crates/hwc-engine/src/voxel_grid/grid/substrate_ops.rs` | Extended `drill_via_hole()` with mask drilling + BBox Z-clamping fix |
| `crates/hwc-compiler/src/ir/placement/contact.rs` | Read properties, compute `pad_diameter_nm`, pass to `drill_via_hole` |
| `crates/hwc-compiler/src/ir/mod.rs` | Auto-create solder mask layers using `board_thickness_nm()` |
| `crates/hwc-parser/src/ast/profile.rs` | Added `solder_mask_expansion` to `ManufacturingConstraints` |
| `crates/hwc-parser/src/parser/definitions/profile/constraints.rs` | Parse `solder_mask_expansion` |
| `crates/hwc-engine/src/constraint_manager/types.rs` | Added `solder_mask_expansion_nm` to `FabricationConstraints` |
| `crates/hwc-engine/src/constraint_manager/manager_impl/fabrication.rs` | Extract solder mask expansion |
| `crates/hwc-engine/src/constraint_manager/manager_impl/symbol_table.rs` | `extract_solder_mask_expansion()` function |
| `crates/hwc-materials/src/constraints.rs` | Added `solder_mask_expansion_nm` to `ConstraintSet` |
| `crates/hwc-compiler/src/conversions.rs` | Extract `solder_mask_expansion` from profile, pass to `ConstraintSet` |
| `crates/hwc-cli/src/commands/build_cmd/validation/drc.rs` | Pass `solder_mask_expansion_nm` to `FabricationConstraints` |
| `crates/hwc-export/src/solder_layers.rs` | Include via pads in Gerber mask layer |
| `tests/via/test_solder_mask.hw` | New test file |
| `tests/via/implementation.md` | Item 4 marked done |

---

## Dependencies

- Item 1 (PTH Via) — DONE
- Item 2 (Antipad) — DONE
- Item 3 (Thermal Relief) — DONE
- Parser changes — Generic `properties` HashMap + `solder_mask_expansion` in manufacturing constraints
