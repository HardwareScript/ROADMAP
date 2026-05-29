# High-Fidelity Optics Roadmap (v0.1.7)

> **Context**: v0.1.6 achieved basic PBR rendering (Metal/Roughness). v0.1.7 aims for 
> photorealistic CAD quality by implementing advanced optical behaviors specifically 
> for electronics materials (solder mask gloss, substrate depth, and refractive optics).

**Status**: Completed (v0.1.7 Alpha)  
**Priority**: High (Artist Mode Excellence)  
**Depends On**: `POST-BRIDGE-LIMITATIONS.md` Limitation 8

---

## Phase 1: Language & AST Expansion ✅

**Goal**: Enable engineers to define advanced optical properties in the `material` block.

- [x] **Property Addition**: Add `ior`, `clearcoat`, `clearcoat_roughness`, and `subsurface` to `MaterialDefinition` in `hwc-parser`.
- [x] **Validation**: Ensure `ior` is >= 1.0 and other factors are 0.0-1.0.
- [x] **Default Values**:
    - `ior`: 1.5 (Standard plastic/glass)
    - `clearcoat`: 0.0
    - `subsurface`: 0.0

---

## Phase 2: Scene Graph & Internal API ✅

**Goal**: Propagate these properties from the compiler to the export engine.

- [x] **MaterialNode Update**: Update `MaterialNode` in `hwc-export/src/scene_graph/types.rs`.
- [x] **Symbol Resolution**: Update `add_materials` in `hwc-export/src/scene_graph/mod.rs` to extract these new properties from the `SymbolTable`.

---

## Phase 3: GLB Exporter (glTF Extensions) ✅

**Goal**: Map internal properties to industry-standard glTF 2.0 extensions.

- [x] **KHR_materials_ior**: Implement IOR extension for realistic light bending.
- [x] **KHR_materials_clearcoat**: Implement clearcoat for that "polished epoxy" look.
- [x] **KHR_materials_transmission**: Wire `opacity` to physical transmission for realistic glass/dielectrics.
- [x] **KHR_materials_volume**: Implemented for physical depth and thickness (1mm default).
- [x] **KHR_materials_anisotropy**: Implemented for surface grain/brushed metal effects.

---

## Phase 4: Procedural Normal Mapping (Pivoted) ✅

**Goal**: Simulate surface texture (fiberglass weave, brushed metal) without external images.

- [x] **Mathematical Anisotropy**: Replaced image-based normal mapping with `KHR_materials_anisotropy` to maintain "Zero Asset" philosophy.
- [ ] **Fiberglass Preset**: (Follow-up) Fine-tune anisotropy rotation for standard FR4 weave.

---

## Phase 5: Verification & Calibration ✅

- [x] **Test Case**: `tests/contact-test/high_fidelity_optics.hw`
    - Verified "Deep Gloss" on Black Solder Mask.
    - Verified "Internal Refraction" with IOR extension.
    - Verified "Anisotropic Highlights" on Gold/Copper.
- [x] **Artist Mode Benchmarking**: Achieved photorealistic results in Babylon.js and other glTF viewers.

---

## Phase 6: Standard Library Update ✅

- [x] **New Visual Library**: Created `hwc/stdlib/materials/visual_optics.hw` with comprehensive definitions for `SolderMask`, `FR4_Visual`, and `Brushed_Metals`.
- [x] **Documentation**: The library file now serves as living documentation for the `ior`, `clearcoat`, `subsurface`, and `anisotropy` properties.

---

## Success Criteria
1. **The "Gloss" Test**: Black solder mask reflects sharp highlights while maintaining a dark base.
2. **The "Depth" Test**: Copper traces inside a 4-layer board appear slightly shifted/distorted based on viewing angle (IOR).
3. **Zero Bloat**: All of this must be achieved via math and glTF extensions, keeping `.hw` files purely textual.
