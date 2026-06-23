# 13 — Physical Synthesis Guardrails: Three Compiler-Side Protections for ASIC Layout Integrity

**Status:** APPROVED
**Priority:** CRITICAL
**Complexity:** HIGH
**Impact:** The compiler silently accepts physically impossible ASIC layouts — metal routed on active silicon, coplanar forbidden junctions undetected, router wandering into transistor interiors.

**Discovered:** 2026-06-23 (CMOS Inverter build succeeded with illegal Copper-on-Silicon routing)
**Depends on:** 04-VERIFICATION-DRC.md (G-Cell sweep engine), 10-WIRING-INTEGRATION-GAPS.md (Entity Graph wired)
**Blocks:** Foundry-grade ASIC compilation — any design with semiconductor stackups

**Cross-references:**
- `Z-AXIS-ABSTRACTION-IMPLEMENTATION.md` — stackup layer model
- `Unified-2.5D-3D-Routing-and-Placement.md` — interior lockout rule
- `BRIDGE-IMPLEMENTATION.md` — silicide bridge requirements
- `BRIDGE-IMPLEMENTATION-CHECKLIST.md` — P45 Forbidden Junction checking
- `Core-System-Architecture.md` — G-Cell DRC sweep architecture
- `11-ZERO-MAGIC-COMPILER.md` — zero-fallback philosophy
- `The-SoC-Engine.md` — layer-aware keep-out zones for standard cells
- `ROUTING-AND-MANUFACTURING-ARCHITECTURE.md` — Escape Exemption filter
- `2D-POLYGON-UNIONING-IMPLEMENTATION.md` — Clipper2 same-material welding
- `ROUTING-LOCK-SYSTEM-IMPLEMENTATION.md` — route lock system

---

## Problem Statement

The CMOS Inverter test (`tests/ASIC/CMOS-Inverter/inverter.hw`) compiled and exported
successfully despite containing three physically impossible conditions:

1. **Copper traces routed directly on the `active` (semiconductor) layer** — in a real
   TSMC 180nm process, routing is only permitted on `metal1` and above. The `active` layer
   is reserved for device diffusion and wells.

2. **Coplanar Copper-to-Silicon_N junctions without a silicide bridge** — the router drew
   a Copper trace that horizontally touched the Silicon_N source/drain diffusion regions.
   No Titanium_Silicide interface was inserted. In silicon, this causes aluminum spiking
   and Schottky-barrier failures.

3. **The router entered the interior bounding box of the transistor** — the pathfinder
   targeted the geometric center of the logical pin, passing through the transistor's
   physical body on the active layer. Traces must terminate at boundary ports only.

The compiler's verification engine reported zero violations because none of these three
checks exist in the current codebase.

---

## Root Cause Analysis

### Gap A: Missing "Routable Layer" Constraints

The stackup profile declares layers with material and thickness but has no concept of
whether a layer permits routing. The pathfinder treats every layer as a valid routing
plane.

**Current stackup model** (`hwc-parser/src/ast/profile.rs`):
```rust
pub struct LayerEntry {
    pub material: Identifier,
    pub thickness: Measurement,
    // No routable flag
}
```

**What exists but is unused:**
- `profile.routing: Option<RoutingConstraints>` — parsed but never consulted by the
  pathfinder during trace placement.
- The pathfinder selects layers based purely on geometric proximity to the start/end
  Z-coordinates, with no material-category filter.

### Gap B: Coplanar P45: Forbidden Junction Blindness

The Forbidden Junction checker (`P45`) is implemented only in the via-unroller
(`unrolling.rs`), which checks **vertical** transitions (vias connecting different Z-layers).
It does not check **horizontal** (coplanar) material interfaces where two different materials
touch on the same Z-layer.

**Current P45 scope:**
- ✅ Vertical via stacks: Copper via touching Silicon substrate → flagged
- ❌ Horizontal traces: Copper trace touching Silicon_N diffusion on same layer → silent

The DRC sweep engine (`gcell_sweep.rs`) checks clearance violations between different nets
but does not classify material categories (conductor vs. semiconductor) at touching
boundaries.

### Gap C: Missing "Interior Lockout" Rule

The pathfinder (`topological_router.rs`) uses A*/ray-casting to find paths between pins.
When a route targets a pin inside a component, the pathfinder enters the component's
bounding box to reach the pin's coordinates.

**What should happen:**
- The component's interior volume must be marked `Cost::INFINITE` during routing.
- Traces must terminate exactly at the component's boundary ports (using `exit:`/`enter:`
  cardinal heuristics).
- The router must never penetrate the component body.

**What actually happens:**
- The router enters the transistor's 1.2µm × 1.2µm bounding box on the active layer.
- It draws Copper through the Polysilicon gate region and Silicon diffusion areas.
- No violation is raised.

---

## Edge Case Analysis: Four Hidden Gaps in the Three-Fix Model

A strict architectural review of the proposed three-fix model reveals four edge cases
that, if unaddressed, will cause routing deadlocks, false positives, or mesh defects
during compile-time verification.

```
    ┌──────────────────────┐              ┌──────────────────────┐
    │ Gap 1: Blunt Keepout │              │ Gap 2: LocalOnly     │
    │ - Over-chip routing  │              │ - Gate ties blocked  │
    │   is blocked in 3D   │              │   outside bboxes     │
    └──────────────────────┘              └──────────────────────┘

    ┌──────────────────────┐              ┌──────────────────────┐
    │ Gap 3: Self-Collision│              │ Gap 4: Clipper/Same- │
    │ - Start/end nodes    │              │        Net Overlaps  │
    │   rejected as INF    │              │ - Different mats are │
    │   cost on bbox edge  │              │   unwelded, overlapping│
    └──────────────────────┘              └──────────────────────┘
```

### Edge Case 1: The "Blunt Keepout" Trap (Fix 3)

**The Issue:** Marking the component's entire 3D bounding box as `Cost::INFINITE` across
all Z-planes prevents the router from routing traces *over* the component on higher metal
layers. In a standard-cell ASIC, metal1/metal2 lines routinely run directly over the
active-layer transistor bodies. A blanket 3D keepout would block all over-cell routing,
causing deadlocks on dense designs.

**The Solution:** The `Cost::INFINITE` penalty must be **layer-aware**. The compiler must
restrict the infinite cost penalty to the specific layers where the component actually has
physical material or declared keep-out zones (active, poly, substrate), leaving upper metal
layers free for global routing.

```rust
fn mark_component_interiors_layer_aware(
    cost_field: &mut CostField,
    entity_graph: &EntityGraph,
    current_layer_name: &str,
    stackup_manager: &StackupManager,
) {
    for component in entity_graph.iter_components() {
        // Only block if the component has physical material on the current routing layer
        if component.has_material_on_layer(current_layer_name, stackup_manager) {
            let bbox = component.bounding_box();
            cost_field.set_region_cost(bbox, Cost::INFINITE);
        }
    }
}
```

### Edge Case 2: The "LocalOnly" Routing Layer Loophole (Fix 1)

**The Issue:** Polysilicon is frequently used as a local interconnect to tie adjacent
transistor gates together (e.g., `route M1.gate to M2.gate` on the poly layer). If
`routable: local_only` strictly forbids routing outside component bounding boxes, a local
polysilicon trace connecting M1.gate to M2.gate will be rejected the moment it exits
their respective bounding boxes into the "empty space" of the active layer.

**The Solution:** Instead of prohibiting out-of-boundary routing entirely, the compiler
should enforce a **Max Local Route Length** constraint. Local-only routes are permitted to
exit component boundaries, but must not exceed a configurable maximum length (default:
10µm) and must not cross G-Cell boundaries. This allows gate-to-gate ties while preventing
local layers from being used for global routing.

```rust
// hwc-engine/src/geometry_router/pathfinding/cost.rs
if stackup_manager.is_routable(layer_name) == RoutableMode::LocalOnly {
    let current_length = current_path.length_nm();
    let max_local_len = profile.routing.max_local_route_length.unwrap_or(10_000); // 10um

    if current_length > max_local_len
        && !is_inside_any_component_bbox(current_pos, entity_graph)
    {
        return Cost::INFINITE; // Halt expansion if local routing exceeds length limit
    }
}
```

**Profile syntax addition:**
```hw
profile Silicon_180nm:
    routing:
        max_local_route_length: 10um   # Max length for routable: local_only layers
```

### Edge Case 3: Pathfinder Port Self-Collision (Fix 3)

**The Issue:** If the component interior is marked as `Cost::INFINITE`, a boundary port
lying exactly on the co-planar outer face of the bounding box will be rejected as an
obstacle by the A* expansion algorithm. The router will fail to initialize the path because
the start node itself is inside the INFINITE-cost region.

**The Solution:** The pathfinder must implement the **Escape Exemption filter**. During the
start and end steps of pathfinding, the specific bounding boxes of the source and target
components must be temporarily exempted from collision checks. The router treats the
boundary port as a valid origin/destination even though it lies on the INFINITE-cost face.

```rust
// In pathfinder initialization:
fn find_path_with_exemptions(
    &self,
    start: Point3D,
    end: Point3D,
    source_component_bbox: Option<BoundingBox>,
    target_component_bbox: Option<BoundingBox>,
) -> Result<Vec<Point3D>, RoutingError> {
    // Temporarily exempt source and target component bboxes from collision
    self.cost_field.exempt_regions(&[
        source_component_bbox,
        target_component_bbox,
    ].into_iter().flatten().collect::<Vec<_>>());

    let path = self.a_star(start, end);

    // Remove exemptions after pathfinding
    self.cost_field.clear_exemptions();

    path
}
```

### Edge Case 4: Same-Net Different-Material Interpenetration (Fix 2)

**The Issue:** The 2D Clipper union engine only welds geometries of the same material
group (e.g., Copper to Copper). If a Copper trace physically overlaps a Silicon_N active
region on the same net, Clipper2 will not weld them. This leaves intersecting, overlapping
3D meshes in the final GLB output — a mesh defect that produces invalid manufacturing
data.

**The Solution:** The same-net branch of the G-Cell sweep-line checker must check for
**intersections** (AABB overlaps), not just touching faces, between different material
categories on the same layer. Any overlapping conductor-semiconductor pair on the same net
must trigger P45 immediately unless they are joined by a registered via/contact.

```rust
// hwc-engine/src/geometry_router/gcell_sweep.rs
if net_id_a == net_id_b && material_id_a != material_id_b {
    // Different materials on the same net — check for spatial intersection
    if aabb_a.intersects(&aabb_b) {
        let mat_a = material_registry.get(material_id_a);
        let mat_b = material_registry.get(material_id_b);
        let junction = classify_junction(mat_a, mat_b, bridge_rules);
        if let JunctionClassification::Forbidden = junction {
            return Some(DrcError::ForbiddenJunction {
                pos: aabb_a.intersection_center(&aabb_b),
                mat_a: mat_a.name.clone(),
                mat_b: mat_b.name.clone(),
            });
        }
    }
}
```

This also means the `OverlapResult` enum must distinguish between face-touching (boundary
contact) and volumetric intersection (body overlap):

```rust
pub enum OverlapResult {
    DifferentNet { overlap_area: f64, required_clearance: i64 },
    SameNetFaceTouch { is_valid_junction: bool },
    SameNetIntersection {                        // NEW — volumetric overlap
        mat_a: MaterialId,
        mat_b: MaterialId,
        intersection_volume: f64,
    },
    MaterialJunction { classification: JunctionClassification },
    NoOverlap,
}
```

### Edge Case 5: Via-Portal Exemption (Fix 3 Extension)

**The Issue:** When routing a net from an active-layer pin (e.g., `M1.drain` on
`layer: active`) to a routing trace on a higher metal layer (e.g., `layer: metal1`), the
router must drop a vertical via tower through the intermediate dielectric and poly layers.
If the component's interior is marked as `Cost::INFINITE` on its active and poly layers,
the vertical via tower will intersect these infinite-cost zones. Even though the Escape
Exemption allows the pathfinder to start at the 2D boundary port, the vertical via column
directly above or below that coordinate will be rejected as an obstacle, causing a routing
deadlock.

```
   UNBIASED VIA BLOCK (Deadlock)             VIA-PORTAL EXEMPTION (Pass)

      Metal 1 Layer                              Metal 1 Layer
      ┌──────────────┐                           ┌──────────────┐
      │   Trace      │                           │   Trace      │
      └──────┬───────┘                           └──────┬───────┘
             │ Via Blocked                              │ Via Passes
    INFINITE │ by Component                    INFINITE │ through
    BBox     ▼ active/poly                     BBox     ▼ local
      ┌──────────────┐                           ┌──────X───────┐
      │  Transistor  │                           │  Transistor  │
      └──────────────┘                           └──────────────┘
```

**The Solution:** The Escape Exemption filter must be extended to support vertical via
paths. The infinite-cost penalty of a component's interior must be bypassed within a tiny
cylindrical radius (matching the via's drill diameter) directly centered on the active
pin's XY coordinate. This allows the via tower to punch through the component's internal
layers to reach the pin, while still blocking unrelated traces from passing through.

**Implementation in the Pathfinder Cost Evaluator:**

```rust
// hwc-engine/src/geometry_router/pathfinding/cost.rs

impl CostField {
    /// Check if a cell position is blocked for routing.
    ///
    /// The Via-Portal Exemption allows vertical via towers to penetrate
    /// component keep-out zones at the exact XY coordinate of an active pin
    /// belonging to the current net.
    pub fn is_cell_blocked(&self, pos: Point3D, current_net: NetId) -> bool {
        if self.is_inside_component_keepout(pos) {
            // VIA-PORTAL EXEMPTION:
            // If the coordinate is directly above/below a pin belonging
            // to this net, allow vertical via penetration
            if self.is_co_located_with_active_pin(pos.x, pos.y, current_net) {
                return false; // Exemption granted: via can pass through
            }
            return true; // Blocked: unrelated trace
        }
        false
    }

    /// Check if an (x, y) position is co-located with any active pin of the given net.
    /// "Co-located" means within the via drill diameter tolerance.
    fn is_co_located_with_active_pin(&self, x: i64, y: i64, net: NetId) -> bool {
        let tolerance_nm = self.via_drill_diameter_nm; // e.g., 220nm for ASIC
        for pin in self.net_pins(net) {
            if (pin.x - x).abs() <= tolerance_nm / 2
                && (pin.y - y).abs() <= tolerance_nm / 2
            {
                return true;
            }
        }
        false
    }
}
```

**Profile integration:**
The via drill diameter used for the portal tolerance is read from the profile's
`via.min_diameter` constraint. This ensures the portal radius matches the physical via
size declared in the PDK.

## The Three Fixes

### Fix 1: Routable Layer Attributions (Profile Level)

**Extend the stackup syntax** to declare whether each layer permits routing.

**New profile syntax:**
```hw
profile Silicon_180nm:
    technology: "ASIC"
    stackup:
        substrate: [material: Silicon_P, thickness: 300um, routable: false]
        active:    [material: Silicon_N, thickness: 200nm, routable: false]
        poly:      [material: Polysilicon, thickness: 150nm, routable: local_only]
        d1:        [material: SiO2,      thickness: 500nm, routable: false]
        metal1:    [material: Aluminum,  thickness: 400nm, routable: true]
```

**Routable modes:**
| Mode | Meaning |
|------|---------|
| `true` | Full routing permitted (metal layers) |
| `false` | No routing permitted (substrate, active, oxide) |
| `local_only` | Local interconnects permitted with max length limit; may exit component boundaries briefly for gate ties |

**Compiler enforcement:**
- During Pass 3 (Routing), if the pathfinder places a trace segment on a layer where
  `routable: false`, the compiler halts with **Error R25: Non-Routable Layer**.
- If `routable: local_only`, traces are permitted to exit component boundaries for short
  gate-to-gate ties, but must not exceed `routing.max_local_route_length` (default: 10µm)
  and must not cross G-Cell boundaries. See Edge Case 2 for the length-limit enforcement.

**Profile syntax addition** (inside `routing:` block):
```hw
profile Silicon_180nm:
    routing:
        max_local_route_length: 10um   # Max length for routable: local_only layers
```

**AST change** (`hwc-parser/src/ast/profile.rs`):
```rust
pub struct LayerEntry {
    pub material: Identifier,
    pub thickness: Measurement,
    pub routable: Option<RoutableMode>,  // NEW
}

pub enum RoutableMode {
    True,
    False,
    LocalOnly,
}

// Addition to RoutingConstraints:
pub struct RoutingConstraints {
    pub per_layer: Option<FxHashMap<CompactString, LayerRouting>>,
    pub max_local_route_length: Option<Measurement>,  // NEW — for local_only layers
}
```

**Parser change** (`hwc-parser/src/parser/definitions/profile/mod.rs`):
- Parse `routable: true|false|local_only` inside stackup layer entries.

**Engine change** (`hwc-engine/src/space/stackup.rs`):
- `StackupManager` exposes `is_routable(layer_name) -> RoutableMode`.

**Router change** (`hwc-engine/src/geometry_router/`):
- Before placing a trace segment, query `stackup_manager.is_routable(layer)`.
- If `false` → reject segment, emit R25.
- If `local_only` → reject unless segment is inside a component's bounding box.

**New error** (`hwc-compiler/src/ir/errors.rs`):
```rust
pub enum IrError {
    // ... existing errors ...
    NonRoutableLayer {
        layer: CompactString,
        material: CompactString,
        span: Option<Span>,
    },
}
```

### Fix 2: Global Coplanar Interface Checking (DRC Level)

**Move P45 Forbidden Junction checking** from the via-unroller into the global G-Cell-local
sweep-line DRC engine.

**Scope expansion:**
- Current: Only vertical via stacks (Z-transition checks).
- New: All face-touching and overlapping boundaries in 3D space (XY coplanar + Z-vertical).

**Classification rule:**
If a routing trace containing a material of category `conductor` (Copper, Aluminum)
physically touches a material of category `semiconductor` (Silicon_N, Silicon_P) **without
an intermediate material of category `ohmic_contact`** (Titanium_Silicide, Cobalt_Silicide),
the DRC engine raises **P45: Forbidden Junction**.

**Implementation in `gcell_sweep.rs`:**

```rust
/// Classify a material junction between two touching geometries
fn classify_junction(
    mat_a: &MaterialDefinition,
    mat_b: &MaterialDefinition,
    bridge_rules: &BridgeTable,
) -> JunctionClassification {
    let cat_a = &mat_a.category;
    let cat_b = &mat_b.category;

    match (cat_a, cat_b) {
        // Conductor touching Semiconductor without bridge → FORBIDDEN
        (MaterialCategory::Conductor, MaterialCategory::Semiconductor)
        | (MaterialCategory::Semiconductor, MaterialCategory::Conductor) => {
            let key = format!("{}:{}", mat_a.name, mat_b.name);
            if bridge_rules.contains_key(&key) {
                JunctionClassification::BridgeRequired { bridge: key }
            } else {
                JunctionClassification::Forbidden
            }
        }
        // Same category or insulator involved → OK (no junction violation)
        _ => JunctionClassification::Allowed,
    }
}
```

**Integration with existing DRC:**
- The `FlatIntervalSweep` in `gcell_sweep.rs` already detects overlapping bounding boxes.
- Extend `OverlapResult` to include material classification and same-net intersection:
  ```rust
  pub enum OverlapResult {
      DifferentNet { overlap_area: f64, required_clearance: i64 },
      SameNetFaceTouch { is_valid_junction: bool },
      SameNetIntersection {                  // NEW — volumetric overlap (Edge Case 4)
          mat_a: MaterialId,
          mat_b: MaterialId,
          intersection_volume: f64,
      },
      MaterialJunction { classification: JunctionClassification },  // NEW
      NoOverlap,
  }
  ```
- When two geometries on the **same net** touch coplanarly, classify the material junction.
  If `Forbidden` → emit P45 violation and halt.
- When two geometries on the **same net** have different materials and their AABBs
  **intersect** (not just touch), run the same junction classification. This catches
  Copper traces that overlap Silicon diffusions — a condition Clipper2 cannot weld
  (Edge Case 4).
- When two geometries on **different nets** overlap, the existing clearance check applies.

**Profile integration:**
- The bridge table from the profile (`profile.bridges: Vec<BridgeRule>`) is passed to the
  DRC engine. If a conductor-semiconductor pair has a declared bridge, the junction is
  flagged as `BridgeRequired` (warning, not error). If no bridge is declared → `Forbidden`
  (error).

### Fix 3: Strict Boundary-Docking Box Model (Pathfinder Level)

**Enforce the Interior Lockout rule** from `Unified-2.5D-3D-Routing-and-Placement.md`,
with layer-aware keep-out zones and Escape Exemption filtering.

**Implementation in `topological_router.rs` and `entity_graph.rs`:**

1. **Mark component interiors as INFINITE cost (layer-aware):**
   When the router initializes its cost field, query all placed component bounding boxes
   from the `EntityGraph`. For each component, set the interior volume to `Cost::INFINITE`
   **only on layers where the component has physical material**. Upper metal layers remain
   free for over-cell routing (Edge Case 1).

   ```rust
   fn mark_component_interiors_layer_aware(
       cost_field: &mut CostField,
       entity_graph: &EntityGraph,
       current_layer_name: &str,
       stackup_manager: &StackupManager,
   ) {
       for component in entity_graph.iter_components() {
           if component.has_material_on_layer(current_layer_name, stackup_manager) {
               let bbox = component.bounding_box();
               cost_field.set_region_cost(bbox, Cost::INFINITE);
           }
       }
   }
   ```

2. **Define boundary ports at component edges:**
   Each component pin has a position. The boundary port is the intersection of the pin's
   position with the component's bounding box surface (not the interior center).

   ```rust
   fn compute_boundary_port(
       pin_position: Point3D,
       component_bbox: BoundingBox,
       exit_direction: CardinalDirection,
   ) -> Point3D {
       match exit_direction {
           North => Point3D::new(pin_position.x, component_bbox.max.y, pin_position.z),
           South => Point3D::new(pin_position.x, component_bbox.min.y, pin_position.z),
           East  => Point3D::new(component_bbox.max.x, pin_position.y, pin_position.z),
           West  => Point3D::new(component_bbox.min.x, pin_position.y, pin_position.z),
       }
   }
   ```

3. **Escape Exemption filter (Edge Case 3):**
   During pathfinding initialization, the start and end component bounding boxes are
   temporarily exempted from INFINITE-cost collision checks. This prevents the pathfinder
   from rejecting boundary ports that lie on the co-planar face of the keep-out zone.

   ```rust
   fn find_path_with_exemptions(
       &self,
       start: Point3D,
       end: Point3D,
       source_bbox: Option<BoundingBox>,
       target_bbox: Option<BoundingBox>,
   ) -> Result<Vec<Point3D>, RoutingError> {
       // Temporarily exempt source and target component bboxes
       let exempts: Vec<BoundingBox> = [source_bbox, target_bbox]
           .into_iter().flatten().collect();
       self.cost_field.exempt_regions(&exempts);

       let path = self.a_star(start, end);

       self.cost_field.clear_exemptions();
       path
   }
   ```

4. **Force route termination at boundary ports:**
   The router's start and end points for each route segment are the boundary ports, not
   the pin's interior coordinates. The `exit:` and `enter:` cardinal directions select
   which face of the bounding box the trace docks onto.

5. **Validation in the unroller:**
   After routing completes, verify that no trace segment's midpoint lies inside any
   component's bounding box (except at the terminal endpoint). If violated → emit
   **Error R30: Route Penetrates Component Interior**.

---

## Implementation Tasks

### Phase 1: Routable Layer Attributes

- [x] **13.1.1** Add `RoutableMode` enum to parser AST (`hwc-parser/src/ast/profile.rs`)
- [x] **13.1.2** Parse `routable: true|false|local_only` in stackup layer entries
  (`hwc-parser/src/parser/definitions/profile/stackup.rs`)
- [x] **13.1.3** Store routable mode in `StackupLayer` and propagate through
  `profile_to_constraints()` (`hwc-compiler/src/conversions/profile_conversion.rs`)
- [x] **13.1.4** Add `is_routable(layer_name) -> RoutableMode` to `StackupManager`
  (`hwc-engine/src/geometry_router/stackup_slicing.rs`)
- [x] **13.1.4a** Add `max_local_route_length: Option<Measurement>` to `RoutingConstraints`
  in parser AST and profile conversions
- [x] **13.1.5** Add `NonRoutableLayer` error variant (R25) and `LocalRouteExceeded` to `IrError`
  (`hwc-compiler/src/ir/errors.rs`)
- [x] **13.1.6** Query `is_routable()` in pathfinder before placing trace segments —
  reject with `i64::MAX` cost if `false` (`hwc-engine/src/geometry_router/pathfinding/cost.rs`)
- [x] **13.1.7** Handle `local_only` mode: permit traces up to `max_local_route_length`
  that exit component boundaries for gate ties; reject if length exceeded
  (`hwc-engine/src/geometry_router/pathfinding/cost.rs`)
- [x] **13.1.8** Add `routable: false` to all non-routing layers in
  `tests/ASIC/CMOS-Inverter/foundry_pdk.hw`
- [ ] **13.1.9** Test: routing a trace on `active` layer → Error R25
- [ ] **13.1.10** Test: `local_only` trace exceeding max length → rejected

### Phase 2: Coplanar P45 Forbidden Junction Checking

- [x] **13.2.1** Add `JunctionClassification` enum to DRC engine
  (`hwc-engine/src/geometry_router/gcell_sweep.rs`)
- [x] **13.2.1a** Extend `OverlapResult` with `SameNetIntersection` variant for
  volumetric different-material overlaps on same net (Edge Case 4)
- [x] **13.2.2** Extend `OverlapResult` with `MaterialJunction` variant
- [x] **13.2.3** Implement `classify_junction(mat_a, mat_b, registry, bridge_table)` function
  using material registry lookup tables
- [x] **13.2.4** During sweep overlap detection, query material categories from
  `MaterialRegistry` via `layer_to_material` map (table-driven, no hard-coding)
- [x] **13.2.5** If same-net coplanar overlap involves conductor+semiconductor,
  classify the junction using the profile's bridge table
- [x] **13.2.5a** If same-net AABB **intersection** (not just face-touch) involves
  different material categories, run junction classification — catches Copper-over-Silicon
  overlaps that Clipper2 cannot weld (Edge Case 4)
- [x] **13.2.6** If `Forbidden` → emit `ViolationType::ForbiddenJunction` with material names
- [x] **13.2.7** If `BridgeRequired` → no violation emitted (bridge_validator handles it)
- [x] **13.2.8** `verify_gcell_sweep()` accepts `layer_to_material`, `material_registry`,
  `bridge_table` parameters — all table-driven
- [ ] **13.2.9** Test: Copper trace touching Silicon_N without bridge → P45 error
- [ ] **13.2.10** Test: Copper trace touching Silicon_N with declared bridge → warning
- [ ] **13.2.11** Test: same-net Copper overlapping Silicon_N AABB → P45 error (Edge Case 4)

### Phase 3: Strict Interior Lockout

- [x] **13.3.1** Layer-aware interior lockout via `component_keepouts` in `MoveCostParams` —
  only block layers where component has physical material (Edge Case 1)
  (`hwc-engine/src/geometry_router/pathfinding/cost.rs`)
- [x] **13.3.1a** Implement `has_material_on_z_range()` on `ComponentMetadata` in
  `EntityGraph` (`hwc-engine/src/geometry_router/substrate_types.rs`)
- [x] **13.3.2** Implement `boundary_port()` on `ComponentMetadata` using component bbox +
  `CardinalDirection` (`hwc-engine/src/geometry_router/substrate_types.rs`)
- [x] **13.3.3** `compute_boundary_port()` on `EntityGraph` delegates to component
  (`hwc-engine/src/geometry_router/entity_graph.rs`)
- [x] **13.3.3a** Escape Exemption handled externally by pathfinder (start/end bbox temporary
  exemption) — `is_inside_component_keepout()` is the hard block
- [x] **13.3.4** Add `R30: RoutePenetratesComponent` error variant
  (`hwc-compiler/src/ir/errors.rs`)
- [x] **13.3.5** Post-route validation: `validate_no_route_penetration()` on `EntityGraph`
  scans all trace midpoints against component bounding boxes
- [ ] **13.3.6** Test: route targeting interior pin → forced to boundary port
- [ ] **13.3.7** Test: route midpoint inside component → Error R30
- [ ] **13.3.8** Test: metal2 trace over active-layer component → permitted (Edge Case 1)
- [ ] **13.3.9** Test: boundary port on INFINITE-cost face → Escape Exemption allows
  pathfinding to start (Edge Case 3)
- [x] **13.3.10** Implement Via-Portal Exemption: `is_via_portal_exempt()` bypasses
  INFINITE-cost within via drill diameter of active pins belonging to current net
  (Edge Case 5) (`hwc-engine/src/geometry_router/pathfinding/cost.rs`)
- [x] **13.3.11** `is_via_portal_exempt()` uses half `via_drill_diameter_nm` as tolerance
  (Edge Case 5)
- [ ] **13.3.12** Test: vertical via through component keep-out zone at pin XY → passes
  (Edge Case 5)
- [ ] **13.3.13** Test: vertical via through component keep-out zone at non-pin XY →
  rejected (Edge Case 5)

### Phase 4: Integration & Regression

- [x] **13.4.1** Update `foundry_pdk.hw` with `routable:` attributes on all stackup layers
  and `routing: max_local_route_length: 10um`
- [x] **13.4.2** Update `inverter.hw` — added missing `width: 180nm` to physical routes, verify clean compile
- [ ] **13.4.3** Run full test suite — no regressions
- [ ] **13.4.4** Verify GLB/DXF/Netlist exports still produce correct geometry
- [ ] **13.4.5** Add `tests/ASIC/CMOS-Inverter/` as a CI regression test
- [ ] **13.4.6** Document new error codes in `hwc-diagnostics/` error reference

---

## Files to Modify

| Phase | File | Change |
|-------|------|--------|
| 1 | `crates/hwc-parser/src/ast/profile.rs` | Add `RoutableMode` enum, `routable` field to `StackupLayer`, `max_local_route_length` to `RoutingConstraints` |
| 1 | `crates/hwc-parser/src/parser/definitions/profile/stackup.rs` | Parse `routable:` in stackup layers |
| 1 | `crates/hwc-parser/src/parser/definitions/profile/constraints.rs` | Parse `max_local_route_length` in routing constraints |
| 1 | `crates/hwc-materials/src/constraints.rs` | Add `RoutableMode`, `layer_routability`, `max_local_route_length_nm` to `ConstraintSet` |
| 1 | `crates/hwc-materials/src/lib.rs` | Re-export `RoutableMode` |
| 1 | `crates/hwc-compiler/src/conversions/profile_conversion.rs` | Propagate routable mode and max local length to constraints |
| 1 | `crates/hwc-engine/src/geometry_router/stackup_slicing.rs` | Engine-side `RoutableMode`, `StackupManager.is_routable()` |
| 1 | `crates/hwc-compiler/src/ir/errors.rs` | Add `NonRoutableLayer` (R25), `LocalRouteExceeded` (R25) errors |
| 1 | `crates/hwc-engine/src/geometry_router/pathfinding/cost.rs` | Query routable before placing traces; enforce local_only length limit |
| 2 | `crates/hwc-engine/src/geometry_router/gcell_sweep.rs` | Add `JunctionClassification`, `BridgeTable`, `SameNetIntersection`, `MaterialJunction` variants, `classify_junction()` |
| 2 | `crates/hwc-materials/src/lib.rs` | `MaterialRegistry` lookup for junction classification (table-driven) |
| 2 | `crates/hwc-engine/src/geometry_router/incremental_drc.rs` | Pass `layer_to_material`, `material_registry`, `bridge_table` to sweep |
| 3 | `crates/hwc-engine/src/geometry_router/substrate_types.rs` | `has_material_on_z_range()`, `boundary_port()`, `CardinalDirection` |
| 3 | `crates/hwc-engine/src/geometry_router/entity_graph.rs` | `compute_boundary_port()`, `validate_no_route_penetration()`, `get_components_on_z_range()` |
| 3 | `crates/hwc-engine/src/geometry_router/pathfinding/cost.rs` | `MoveCostParams` extension, `is_inside_component_keepout()`, `is_via_portal_exempt()` |
| 3 | `crates/hwc-compiler/src/ir/errors.rs` | Add `RoutePenetratesComponent` (R30), `ForbiddenJunction` (P45) errors |
| 4 | `tests/ASIC/CMOS-Inverter/foundry_pdk.hw` | Add `routable:` attributes and `max_local_route_length` |

---

## Error Code Registry

New error variants for `hwc-compiler/src/ir/errors.rs`:

| Code | Name | Trigger |
|------|------|---------|
| **R25** | `NonRoutableLayer` | Pathfinder places trace on layer with `routable: false` |
| **R25** | `LocalRouteExceeded` | `local_only` trace exceeds `max_local_route_length` |
| **R30** | `RoutePenetratesComponent` | Post-route validation finds trace midpoint inside component bbox |
| **P45** | `ForbiddenJunction` | Coplanar conductor-semiconductor contact without declared bridge, or same-net volumetric different-material intersection |

P45 already existed for vertical via checks. This extension adds coplanar (same-layer)
scope and same-net volumetric intersection checks to the same error code.

---

## Success Criteria

- [x] `foundry_pdk.hw` declares `routable: false` on substrate, active, and oxide layers
- [ ] Routing a trace on the `active` layer produces Error R25 (not a silent pass) — implementation complete, test pending
- [ ] A Copper trace coplanarly touching Silicon_N without a bridge produces P45 — implementation complete, test pending
- [ ] A Copper trace coplanarly touching Silicon_N with a declared bridge produces no violation — implementation complete, test pending
- [ ] A same-net Copper trace overlapping Silicon_N AABB produces P45 (Edge Case 4) — implementation complete, test pending
- [ ] The router terminates traces at component boundary ports, never entering interiors — implementation complete, test pending
- [ ] Metal2 traces route freely over active-layer components (Edge Case 1 — no false keepouts) — implementation complete, test pending
- [ ] `local_only` polysilicon traces connecting gate pins succeed within length limit (Edge Case 2) — implementation complete, test pending
- [ ] `local_only` polysilicon traces exceeding max length are rejected (Edge Case 2) — implementation complete, test pending
- [ ] Boundary ports on INFINITE-cost faces are valid pathfinding start/end nodes (Edge Case 3) — implementation complete, test pending
- [ ] Vertical vias through component keep-out zones at pin XY coordinates pass (Edge Case 5) — implementation complete, test pending
- [ ] Vertical vias through component keep-out zones at non-pin XY coordinates are rejected (Edge Case 5) — implementation complete, test pending
- [ ] The CMOS Inverter compiles cleanly with all guardrails active — `cargo build --lib` passes
- [ ] No false positives on valid designs (metal-on-metal routing, proper bridges, over-cell routing)
- [ ] GLB/DXF exports are unchanged for valid designs
- [ ] Full test suite passes with zero regressions

---

## Design Principles

This gap enforces three principles from the Hardware Script Manifesto:

1. **Zero Magic** — The compiler never silently permits a physically impossible condition.
   If a rule is violated, the build halts with a specific error and actionable suggestion.

2. **Explicit Intent** — Every routable layer must be declared. Every material junction
   must have a bridge rule. The compiler does not guess.

3. **Foundry-Grade Verification** — The DRC engine checks the same conditions a real
   foundry DRC deck would check: layer usage rules, material interface rules, and
   component boundary rules.
