# v0.1.7 Component Mounting Abstraction Roadmap

**Goal**: Implement the relational mounting paradigm to resolve physical overlap, Z-fighting, and coordinate mirroring for top/bottom components.

---

## Phase 1: Parser Integration (hwc-parser)

- [x] **1.1 Add MountingSide Enum**: Define `MountingSide` in `hwc-parser/src/ast/component.rs`.
- [x] **1.2 Update ComponentPlacement**: Add `mount` field to `ComponentPlacement` struct.
- [x] **1.3 Update Component Placement Parser**: Modify `parse_component_placement` in `hwc-parser/src/parser/components.rs` to recognize and parse the `mount:` and `standoff:` properties.
- [x] **1.4 Update Layout Block Parser**: Modify `parse_layout_block` in `hwc-parser/src/parser/definitions/component.rs` to parse the `standoff:` property.

## Phase 2: StackupManager API Updates (hwc-compiler)

- [x] **2.1 Boundary Resolution**: Implement `board_surface_z(side: MountingSide) -> i64` in `StackupManager`.
- [x] **2.2 Copper Thickness Lookup**: Implement `outer_copper_thickness_nm(side: MountingSide) -> i64` to find target pad layers.

## Phase 3: Placement & Unrolling Pipeline (hwc-compiler)

- [x] **3.1 Elevation & Body Math**: 
    - Resolve absolute Z for component bodies in `place_component`.
    - Ensure Top-mounted bodies span $[Surface, Surface + Height]$.
    - Ensure Bottom-mounted bodies span $[Surface - Height, Surface]$.
- [x] **3.2 Coordinate Mirroring**: 
    - Implement $X_{mirrored} = -X_{local}$ for bottom-mounted components.
- [x] **3.3 Relational Unrolling of Landing Pads**:
    - Automatically sink pads into the adjacent conductive layer (Top/Bottom copper).

## Phase 4: Downstream & Export Updates (hwc-export)

- [x] **4.1 Update GLB Scene Graph**:
    - Set opaque PBR materials for component bodies.
    - Correctly translate body nodes on the Z-axis.
- [x] **4.2 Segmented Viewport Export (DXF)**:
    - Categorize entities into `TOP_COMPONENTS`, `PCB_LAYERS`, and `BOTTOM_COMPONENTS`.

## Phase 5: Verification & Safety Checks

- [x] **5.1 Verification of "Top" Mount**:
    - Verify U1 package body elevation (positive Z-offset).
    - Verify pads are sunk into top copper.
- [x] **5.2 Verification of "Bottom" Mount**:
    - Verify U2 package body elevation (negative Z-offset).
    - Verify X-axis coordinate mirroring.
- [x] **5.3 Z-Fighting Audit**:
    - Ensure no shared surfaces between body and copper triggers rendering artifacts.
- [x] **5.4 Drill-to-Drill Spacing Check**:
    - Implement `validate_drill_to_drill_clearance` to catch overlapping same-net vias.
    - ✅ **v0.1.7 NATIVE FIX**: Updated to ignore same-net collisions and support vertical stacks.

## v0.1.7 Polish & Bug Fixes (Completed)

- [x] **Robust Z-Axis Intersection**: Replaced legacy `overlaps_start/end` with inclusive `BoundingBox::intersects` logic ($min \le max$).
- [x] **Z-Coordinate Scaling Fix**: Corrected $10^9 \rightarrow 10^6$ scaling error in netlist export and error reporting.
- [x] **Collision-Aware Auto-Via Insertion**: Implemented same-net awareness in `AutoViaInserter` to prevent redundant via placement while supporting vertical stacks.
