# Sprint 2.4: Component-Level Continuity Validation

**Status**: 🔄 NOT STARTED  
**Priority**: HIGH (Critical for component library correctness)  
**Estimated Effort**: 2-3 days

---

## Purpose

Validate physical connectivity **inside** each component to ensure:
- Every pin connects to internal geometry
- No unintended internal shorts between pins
- No floating internal geometry
- Component definitions are physically correct

This catches errors in component design, not just board routing.

---

## Violation Types

### C41: Disconnected Pin
Pin exists but doesn't connect to any internal geometry.

**Example**: BGA ball has no via to internal fanout layer.

### C42: Internal Short Circuit
Multiple pins connect to the same internal conductive island.

**Example**: VDD and VSS pins accidentally shorted inside package.

### C43: Floating Internal Geometry
Internal conductive geometry doesn't connect to any pin.

**Example**: Unused internal trace or pour left in component definition.

---

## Implementation Checklist

### Phase 1: Data Structures
- [ ] `ComponentContinuityViolation` enum (C41, C42, C43)
- [ ] `ComponentInternalIsland` struct
- [ ] Component boundary detection (bbox extraction)
- [ ] Pin-to-component coordinate transformation

### Phase 2: Island Building (Per-Component)
- [ ] `build_component_internal_islands()` - Flood-fill within component bbox only
- [ ] Filter geometry nodes by component boundary
- [ ] Reuse existing spatial grid infrastructure
- [ ] Handle component rotation/translation

### Phase 3: Validation
- [ ] **C41 Detection**: Check each pin connects to at least one internal island
- [ ] **C42 Detection**: Check each internal island connects to at most one pin
- [ ] **C43 Detection**: Check each internal island connects to at least one pin
- [ ] Smart diagnostics with suggested fixes

### Phase 4: Integration
- [ ] Add to `hwc-cli/src/commands/build.rs` after board-level checks
- [ ] Iterate through all component instances
- [ ] Report violations with component name and location
- [ ] Error messages with actionable fixes

### Phase 5: Testing
- [ ] Test: Simple resistor (2 pins, 1 internal trace) - should pass
- [ ] Test: Disconnected pin (pin with no internal connection) - C41
- [ ] Test: Internal short (2 pins touching same island) - C42
- [ ] Test: Floating geometry (internal pour with no pins) - C43
- [ ] Test: Complex BGA package with multiple layers
- [ ] Test: Rotated/translated components

---

## Key Design Decisions

### Scope Isolation
Component-level checks run **independently** for each component instance:
- Only examine geometry within component's bounding box
- Transform pin positions to component's local coordinate system
- Ignore board-level routing (that's handled by board-level checks)

### Reuse Existing Infrastructure
- Use same flood-fill algorithm as board-level checks
- Use same spatial grid indexing
- Use same `ConductiveIsland` data structure
- Just filter nodes by component boundary

### When to Run
Run component-level checks **after** board-level checks:
1. Board-level continuity (P41, P42, P43) - Sprint 2.3 ✅
2. Component-level continuity (C41, C42, C43) - Sprint 2.4 🔄

This ensures both the board routing AND component internals are correct.

---

## Files to Modify/Create

### New Files
- `hwc/crates/hwc-physics/src/component_continuity.rs` - Core implementation

### Modified Files
- `hwc/crates/hwc-physics/src/lib.rs` - Export component_continuity module
- `hwc/crates/hwc-cli/src/commands/build.rs` - Add component-level validation loop

### Test Files
- `hwc/tests/sprint2_4_component_continuity/test_c41_disconnected_pin.hw`
- `hwc/tests/sprint2_4_component_continuity/test_c42_internal_short.hw`
- `hwc/tests/sprint2_4_component_continuity/test_c43_floating_internal.hw`
- `hwc/tests/sprint2_4_component_continuity/test_simple_resistor_pass.hw`
- `hwc/tests/sprint2_4_component_continuity/test_bga_package.hw`

---

## Success Criteria

- [ ] All three violation types (C41, C42, C43) detected correctly
- [ ] Works with rotated and translated components
- [ ] Performance: <1ms per component for typical packages
- [ ] Clear error messages with component name and pin location
- [ ] Test coverage: 5+ test cases covering all violation types
- [ ] Integration: Runs automatically during `hwc build`

---

## Benefits

1. **Component Library Correctness**: Catch errors in component definitions before they're used
2. **Custom Package Design**: Validate BGAs, QFNs, and custom IC packages
3. **Early Error Detection**: Find internal shorts/disconnections at compile time
4. **Design Confidence**: Know that component internals are physically correct

---

## Notes

- This is **orthogonal** to board-level checks - both are needed
- Most useful for custom components and package design
- Standard library components should be pre-validated
- Can be disabled for trusted components to improve build speed (future optimization)
