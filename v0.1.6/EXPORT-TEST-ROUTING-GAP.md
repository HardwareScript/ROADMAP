# Export Test Routing Gap

**Status**: Test Infrastructure Issue (Not a Syntax Bug)  
**Version**: v0.1.6  
**Date**: 2026-04-11

---

## The Issue

Four export tests are failing:
- `test_gerber_horizontal_trace_uses_draw`
- `test_gerber_vertical_trace_uses_draw`
- `test_gerber_multi_layer_export`
- `test_gerber_mixed_board_traces_and_pads`

All failing tests have `route` commands in their Hardware Script source.

## Root Cause Analysis

The tests are failing with:
```
Total Gerber output: 0 lines across 0 layers
Should generate at least one Gerber file
```

### What's Happening

1. **Parsing**: ✅ The Hardware Script source parses correctly
2. **Compilation**: ✅ The AST compiles to `HardwareSpace` correctly
3. **Routing**: ❌ The routing engine is NEVER CALLED
4. **Export**: ❌ No traces exist, so no copper layers are generated

### The Code

```rust
let (space, symbol_table) = parse_and_compile(source).expect("Failed to parse");
let compiled = CompiledOutput {
    space,
    ir: None,  // ← NO IR, NO ROUTING!
    symbol_table,
};
```

The `parse_and_compile()` helper only:
- Lexes the source
- Parses to AST
- Compiles to `HardwareSpace`

It does NOT:
- Create a `GeometryRouter`
- Call `route_all_nets()`
- Generate the IR with routed traces

## Why Tests Pass/Fail

### Passing Tests
- `test_gerber_export` - Empty board, no routing needed
- `test_gerber_isolated_pad_uses_flash` - Just component pads, no routes
- `test_obj_export`, `test_glb_export`, etc. - Don't require routing

### Failing Tests
- All tests with `route` commands - Routing never executes

## The Fix

The `parse_and_compile()` function needs to be extended to:

1. Extract nets from the compiled space
2. Create a `GeometryRouter` with appropriate bounds/constraints
3. Call `router.route_all_nets(&nets)`
4. Generate the IR with routed traces
5. Return `CompiledOutput` with `ir: Some(ir)`

## Example from Engine Tests

```rust
// From hwc/crates/hwc-engine/tests/route_lockfile_integration_test.rs
let bounds = GridBounds::new(100_000_000, 100_000_000, 10_000_000);
let constraints = ConstraintRulebook::new(500_000);
let mut router = GeometryRouter::new(bounds, constraints);

// Route all nets
let routes = router.route_all_nets(&nets).unwrap();
```

## Impact

- **v0.1.6 Syntax Migration**: ✅ COMPLETE - Not blocked by this
- **Export Tests**: ❌ Need routing infrastructure
- **Production Code**: ✅ Works correctly (CLI does full compilation)

## Recommendation

1. Create a `parse_compile_and_route()` helper for tests
2. Update the 4 failing export tests to use it
3. Keep the simple `parse_and_compile()` for tests that don't need routing

This is a test infrastructure gap, not a language or compiler bug.

---

## Verification

The v0.1.6 syntax is working correctly:
- `component Battery:` ✅ (no quotes)
- `space HorizontalTrace:` ✅ (no quotes)
- `add Battery(voltage: 5V)` ✅ (keyword arguments with Measurement values)
- `add substrate(FR4)` ✅ (bare identifier material)
- `route Power.Plus to Light.Anode` ✅ (parsed correctly)

The only issue is that the routing engine is not being invoked in the test harness.
