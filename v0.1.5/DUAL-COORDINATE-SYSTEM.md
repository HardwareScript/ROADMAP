# Physical-Intent Coordinate System (v0.1.5)

## Architectural Decision: Physics-Grounded Coordinates

**Core Principle**: Atoms don't stretch. Physical mass is absolute. Position can be relative.

## The Three Rules

### Rule 1: Physical Mass is Absolute
Anything defining physical mass (component sizes, space dimensions, trace widths, via holes) MUST use absolute SI units.

**Why**: An ESP32 chip is physically 18mm × 25.5mm. If you define it as `10%` and move it from a 100mm board to a 30mm smartwatch, the compiler would generate a shrinking microchip. The factory's pick-and-place machine expects exact dimensions.

**Supported Units**: `m`, `cm`, `mm`, `µm` (or `um`), `nm`, `pm`

### Rule 2: Layers are Logical Indices
The Z-axis represents logical layer indices (1=Top, 2=Inner1, etc.), not physical measurements.

**Why**: You don't control the Z-axis—the factory does. If you route at `z: 1.6mm` and the factory switches from 1.6mm to 1.2mm FR4, your trace floats in air. By using `z: 1`, your design is immune to stackup changes. The `profile.hw` defines physical thickness.

**Supported**: Integer indices only (`1`, `2`, `3`, ...)

### Rule 3: Placement Can Be Relative
X/Y coordinates for placement support BOTH absolute measurements AND percentages.

**Why**: Placing a CPU at `[x: 50mm, y: 50mm]` is brittle. If you resize the board to 80×80mm, it's off-center. Using `[x: 50%, y: 50%]` keeps it centered, like CSS Flexbox for hardware.

**Supported**: SI units (`mm`, `µm`, `nm`) OR percentages (`%`)

## Syntax Examples

### Valid: Absolute Placement
```hw
define space "PCB":
    dimensions: 100mm by 100mm by 1.6mm
    grid: 10000 by 10000 by 2

    # Absolute positioning
    add MCU_ESP32 named Brain at [x: 50mm, y: 50mm, z: 1]
```

### Valid: Relative Placement (The "Flexbox" of Hardware)
```hw
define space "SmartWatch":
    dimensions: 40mm by 40mm by 1.6mm
    grid: 4000 by 4000 by 4

    # Center the MCU (50% of 40mm = 20mm)
    add MCU_ESP32 named Brain at [x: 50%, y: 50%, z: 1]
    
    # Mix absolute and relative
    # "Center on X, but 5mm from top edge on Y"
    add Antenna named WiFi at [x: 50%, y: 5mm, z: 1]
```

### Valid: Math + Loops (Better than CSS)
```hw
define space "LED_Bar":
    dimensions: 100mm by 10mm by 1.6mm
    grid: 10000 by 1000 by 2

    # Distribute 8 LEDs evenly across the board
    # Spacing = 100mm / 8 = 12.5mm per segment
    for i in 0..7:
        add LED_0805 named Light[i] at [x: 12.5mm + (i * 12.5mm), y: 50%, z: 1]
```

### Invalid: Deprecated Syntax
```hw
# ❌ Raw grid indices are deprecated
add Resistor named R1 at [50, 50, 1]

# ❌ Z-axis cannot use measurements
add Resistor named R1 at [x: 10mm, y: 10mm, z: 1.6mm]

# ❌ Positional syntax is deprecated
add Resistor named R1 at [10mm, 10mm, 1]
```

## Implementation Checklist

### 1. Expression System ✅ COMPLETE
- [x] Add `Value::Measurement` variant
- [x] Add `Value::Percentage` variant for relative positioning
- [x] Handle measurement arithmetic (e.g., `10mm + (i * 5mm)`)
- [x] Handle percentage arithmetic (e.g., `50% + 10`)

### 2. Parser Enforcement ✅ COMPLETE
- [x] Reject positional coordinates `[x, y, z]`
- [x] Validate X/Y are measurements OR percentages
- [x] Validate Z is an integer (not measurement, not percentage)
- [x] Smart parser: Intercept `%` measurements in expression context

### 3. Coordinate Conversion ✅ COMPLETE
- [x] Convert measurements to nanometers
- [x] Convert percentages to nanometers (multiply by space dimensions)
- [x] Z-axis: multiply integer by voxel_size.z_nm
- [x] Handle mixed expressions: `50% + 5mm` (via expression evaluation)

### 4. Error Messages ⚠️ DEFERRED
- [ ] S24: Deprecated grid indices (parser already rejects with clear message)
- [ ] S25: Z-axis measurement usage (parser already rejects with clear message)
- [ ] S26: Z-axis percentage usage (parser already rejects with clear message)

### 5. Testing ✅ COMPLETE
- [x] Test absolute positioning: `[x: 10mm, y: 15mm, z: 1]` - WORKS
- [x] Test relative positioning: `[x: 50%, y: 50%, z: 1]` - WORKS
- [x] Test mixed positioning: `[x: 50%, y: 5mm, z: 1]` - WORKS
- [x] Test expressions: `[x: 50% + (i * 5mm), y: 20mm, z: 1]` - WORKS (via expression eval)
- [x] Standard library compatibility: `tolerance: 1%` still works as measurement

## Why This is Better Than CSS Units

CSS has `em`, `rem`, `vh`, `vw` because web layouts are fluid and relative to viewport/font size.

Hardware Script doesn't need these because:

1. **Physical reality is absolute**: A chip is always the same size
2. **Math is better than magic units**: `12.5mm + (i * 12.5mm)` is clearer than `justify-content: space-between`
3. **Percentages are enough**: `50%` for centering, combined with absolute offsets, covers all use cases
4. **Compile-time expressions**: For-loops and arithmetic replace CSS Flexbox/Grid

## The Holy Grail ✅ ACHIEVED

Hardware Script achieves:
- **Strict as a physics simulator**: No impossible geometries
- **Pleasant as a web framework**: Relative positioning, math expressions, loops
- **Scale-invariant**: Same code works from PCBs to 5nm silicon
- **Manufacturing-independent**: Layer indices immune to stackup changes

## Implementation Summary

### Architecture: "Dumb Lexer, Smart Parser"
The key insight was to keep the lexer simple and let the parser assign meaning based on context:

1. **Lexer**: Parses `50%` as `Measurement` with `Custom("%")` unit (generic measurement)
2. **Parser**: Expression parser intercepts `%` measurements and converts to `Expression::Percentage`
3. **Result**: Electrical properties like `tolerance: 1%` remain measurements, while coordinates like `[x: 50%]` become percentages

### What Works Now
- ✅ Absolute positioning: `[x: 10mm, y: 15mm, z: 1]`
- ✅ Relative positioning: `[x: 50%, y: 50%, z: 1]` (centers on any board size)
- ✅ Mixed positioning: `[x: 50%, y: 10mm, z: 1]` (combine absolute and relative)
- ✅ Standard library: `tolerance: 1%` still works as measurement
- ✅ All coordinate conversion functions accept space dimensions for percentage calculations

### Performance Note
For faster iteration during development, skip DRC checks:
```bash
hwc build file.hw --skip-drc
```

This bypasses thermal/electrical validation which can be slow on complex designs.
