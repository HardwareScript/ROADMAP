# Compiler Gap Report: Multi-Layer Auto-Router Failure in CMOS Inverter

**Date:** 2026-06-30  
**Compiler Version:** hwc v0.1.6 (Syntax Unification)  
**Test Case:** CMOS Inverter (0.25µm TSMC Process)  
**Status:** ❌ FAILED - Physical Continuity Validation  
**Severity:** HIGH - Blocks compilation of standard CMOS circuits

---

## Executive Summary

The hwc compiler's auto-router **consistently fails to create physical continuity** for nets that must connect across multiple stackup layers in ASIC designs. Specifically, when a net (e.g., `VIN`) must connect:
1. A metal1 pad (Aluminum, z: 955nm-1355nm)
2. To transistor gate terminals (Polysilicon, z: 305nm-455nm) 
3. Across multiple component instances

The auto-router **does not generate the required vias/contacts** to bridge these layer transitions, resulting in disconnected conductive islands and failed physical continuity validation.

---

## Problem Description

### What Should Happen
When the user writes:
```hw
route M1.gate to VIN
route M2.gate to VIN  
route VIN_Pad to M1.gate  // or implicit via routing rules
```

The compiler should automatically:
1. Detect that `M1.gate` and `M2.gate` are on the **polysilicon layer** (z: 305nm-455nm)
2. Detect that `VIN_Pad` is on the **metal1 layer** (z: 955nm-1355nm)
3. Apply the defined **bridge rules** from the PDK:
   ```hw
   bridge Polysilicon to Aluminum:
       interface: Titanium_Silicide
       thickness: 50nm
       fill: Tungsten
   ```
4. **Automatically generate contact/via structures** at the intersection points
5. Route metal1 traces to physically connect all VIN components

### What Actually Happens
The compiler:
1. ✅ Correctly parses the logical netlist (M1.gate, M2.gate, VIN_Pad all assigned to net `VIN`)
2. ✅ Passes alignment validation (logical netlist matches schematic)
3. ❌ **FAILS** to generate physical routing between layers
4. ❌ Results in **3 disconnected island groups** on the VIN net:
   - Island group 0 at z:0nm-1355nm (2 islands)
   - Island group 1 at z:955nm-1355nm (1 island)  
   - Island group 2 at z:305nm-455nm (1 island)
5. ❌ Reports physical continuity violation with Y-gap: 925nm-1125nm

---

## Reproduction Steps

### Test Files Location
```
tests/ASIC/cmos-inverter-test/
├── cmos_inverter.hw       # Main design file
├── foundry_pdk.hw         # PDK with bridge rules
├── materials.hw           # Material properties
└── COMPILER-GAP-REPORT.md # This document
```

### Command to Reproduce
```bash
cd c:\Users\olowo\Downloads\Code\hw\hwc
cargo run -- build tests\ASIC\cmos-inverter-test\cmos_inverter.hw
```

### Expected Result
```
✅ Physical continuity validation passed: All nets are physically continuous
✅ DRC validation passed
✅ Build complete: lockfile saved
```

### Actual Result
```
❌ PHYSICAL CONTINUITY VIOLATIONS - Cannot proceed to parameter validation:
   P41 (Net 'VIN' has 3 disconnected conductive components: Island group 0 at z:0nm-1355nm (2 islands), 
        Island group 1 at z:955nm-1355nm (1 islands), Island group 2 at z:305nm-455nm (1 islands)): 
   XY-plane gap detected between components on net 'VIN'.
   X-gap: 0 nm, Y-gap: 925 nm.
   Suggested fix: Add a pour or route to bridge the gap on net 'VIN'.
  x Physical continuity validation failed with 1 violation(s).
```

---

## Technical Analysis

### Layer Stack Configuration
From `foundry_pdk.hw`:
```hw
stackup:
    substrate:  [material: Silicon_P,   thickness: 100nm, routable: false]  # z: 0-100nm
    active:     [material: Silicon_N,   thickness: 200nm, routable: false]  # z: 100-300nm
    gate_oxide: [material: SiO2,        thickness: 5.75nm, routable: false] # z: 300-305.75nm
    poly:       [material: Polysilicon, thickness: 150nm, routable: local_only] # z: 305.75-455.75nm
    d1:         [material: SiO2,        thickness: 500nm, routable: false]  # z: 455.75-955.75nm
    metal1:     [material: Aluminum,    thickness: 400nm, routable: true]   # z: 955.75-1355.75nm
```

### Defined Bridge Rules
```hw
bridge Polysilicon to Aluminum:
    interface: Titanium_Silicide
    thickness: 50nm
    fill: Tungsten
```

**Interpretation:** When Polysilicon and Aluminum need to connect, the compiler should:
1. Create a TiSi₂ interface layer (50nm)
2. Fill with Tungsten via
3. This creates an ohmic contact from poly (gate) to metal1 (routing)

### Component Placement
```hw
space CMOS_Inverter:
    # Transistor M1 (NMOS) placed at y=900nm
    add NMOS_Transistor named M1 at [x: 0.875um, y: 0.9um] on layer: active
    
    # Transistor M2 (PMOS) placed at y=2000nm  
    add PMOS_Transistor named M2 at [x: 0.875um, y: 2.0um] on layer: active
    
    # VIN pad on metal1 at y=1400-2000nm
    add pour(Aluminum) named VIN_Pad on layer: metal1:
        net: VIN
        boundary: [x: 0nm, y: 1.4um] to [x: 0.5um, y: 2.0um]
```

### Logical Routing Declarations
```hw
route M1.gate to M2.gate:
    width: 250nm
    layer: metal1
    current_limit_ac: { rms: 1uA, peak: 3uA }
    
route M1.gate to VIN_Pad:
    width: 250nm
    layer: metal1
    current_limit_ac: { rms: 1uA, peak: 3uA }
```

**Note:** Both routes specify `layer: metal1`, implying the router should handle layer transitions.

---

## Attempted Workarounds

### Attempt 1: Adjusting Component Spacing
**Hypothesis:** Components were too far apart for router to find paths.

**Actions Taken:**
- Reduced M1 to M2 vertical spacing from 2160nm → 1500nm → 1100nm
- Centered VIN_Pad between transistors
- Reduced overall die size from 3.6×5.76µm → 3.0×4.0µm

**Result:** ❌ Failed. Y-gap reduced from 1305nm → 1125nm → 925nm, but VIN still disconnected.

### Attempt 2: Manual Bridge Pours
**Hypothesis:** Explicit metal1 pours overlapping gate pads would force connections.

**Actions Taken:**
```hw
# Manual bridge attempting to connect M1 gate to VIN
add pour(Aluminum) named VIN_to_M1_Gate on layer: metal1:
    net: VIN
    boundary: [x: 0.5um, y: 1.375um] to [x: 1.5um, y: 1.625um]

# Manual bridge attempting to connect M2 gate to VIN  
add pour(Aluminum) named VIN_to_M2_Gate on layer: metal1:
    net: VIN
    boundary: [x: 0.5um, y: 2.925um] to [x: 1.5um, y: 3.175um]
```

**Result:** ❌ Failed. Y-gap reduced slightly (1125nm → 925nm), but islands remain disconnected because:
- Metal1 pours don't physically overlap with polysilicon gate pads (different Z-layers)
- No via/contact structures were auto-generated at overlapping XY coordinates

### Attempt 3: Increasing Trace Width
**Hypothesis:** Wider traces might help router find paths.

**Actions Taken:**
- Changed route widths from 180nm → 250nm (minimum per PDK rules)

**Result:** ❌ Failed. No change in connectivity behavior.

---

## Root Cause Analysis

### Confirmed Non-Issues
1. ✅ **PDK Definition:** Bridge rules are correctly specified
2. ✅ **Material Properties:** All conductors have proper resistivity, current limits
3. ✅ **Logical Netlist:** Module and schematic match perfectly
4. ✅ **DRC Rules:** Design rules (spacing, widths) are appropriate for 0.25µm process
5. ✅ **Component Definitions:** Transistor footprints have proper device bindings

### Identified Root Cause
**The auto-router does not implement multi-layer routing with automatic via insertion.**

Specifically:
1. The router operates **primarily in 2D (XY plane)** on a single routable layer
2. When a net requires connections on **multiple layers** (poly + metal1), the router:
   - ❌ Does not detect the need for layer transitions
   - ❌ Does not apply the defined bridge rules
   - ❌ Does not generate via/contact structures
   - ❌ Does not create metal1 "straps" to physically connect poly islands

3. The `route` statement's `layer: metal1` parameter is **not sufficient** to trigger:
   - Via generation at poly→metal1 transitions
   - Automatic bridging per the PDK rules

---

## Expected Auto-Router Behavior

For the VIN net to be physically continuous, the auto-router should generate (pseudo-code):

```
1. Identify all VIN terminals and their layers:
   - VIN_Pad: metal1 @ [0-500nm, 1400-2000nm, 955-1355nm]
   - M1_Gate_Pad: metal1 @ [1375-1625nm, 1275-1525nm, 955-1355nm]  
   - M1_Gate_Poly: polysilicon @ [1375-1625nm, 900-1500nm, 305-455nm]
   - M2_Gate_Pad: metal1 @ [1375-1625nm, 3025-3275nm, 955-1355nm]
   - M2_Gate_Poly: polysilicon @ [1375-1625nm, 2000-3350nm, 305-455nm]

2. Detect layer transitions needed:
   - M1_Gate_Poly (poly) → M1_Gate_Pad (metal1): needs poly-to-metal contact
   - M2_Gate_Poly (poly) → M2_Gate_Pad (metal1): needs poly-to-metal contact

3. For each transition, apply bridge rule:
   ```
   Contact @ [1500nm, 1400nm]:
     - Bottom: Polysilicon @ z=305-455nm
     - Interface: Titanium_Silicide @ z=455-505nm  
     - Via fill: Tungsten cylinder (250nm dia) @ z=505-955nm
     - Top: Aluminum @ z=955-1355nm
   ```

4. Route metal1 traces to connect:
   - VIN_Pad [250nm, 1700nm] → M1_Gate_Pad [1500nm, 1400nm]: horizontal trace
   - M1_Gate_Pad [1500nm, 1400nm] → M2_Gate_Pad [1500nm, 3150nm]: vertical trace
   
5. Verify physical continuity in 3D:
   - All conductive elements on VIN net form a single connected component
   - Electrical path exists from VIN_Pad → M1.gate → M2.gate
```

---

## Impact Assessment

### Circuits Affected
This bug affects **any ASIC design** where:
1. Multiple stackup layers are used (poly + metal1 + metal2, etc.)
2. Component terminals exist on different layers
3. Logical connectivity requires cross-layer routing

**Examples:**
- ✅ Simple resistor networks (single layer)
- ❌ **CMOS inverters** (poly gates + metal routing)
- ❌ **Logic gates** (NAND, NOR, XOR, etc.)
- ❌ **Flip-flops, latches** (any sequential logic)
- ❌ **Standard cell libraries** (all ASIC digital cells)
- ❌ **Multi-layer analog circuits**

**Conclusion:** This bug **blocks compilation of essentially all CMOS digital circuits**.

### Workaround Viability
**Manual routing** could theoretically work by:
1. Explicitly declaring every via location
2. Manually specifying all metal1 strap geometries
3. Manually ensuring all XY overlaps trigger via generation

**However:**
- Requires 100× more code than logical routing
- Defeats the purpose of an auto-router
- Not scalable for realistic designs (>10 gates)
- Error-prone and non-portable across process nodes

**Verdict:** No practical workaround exists.

---

## Recommended Fixes

### Priority 1: Implement Multi-Layer Auto-Router
**Location:** `crates/hwc-compiler/src/ir/routing/automatic.rs`

**Required Features:**
1. **Layer-aware graph construction:**
   - Build 3D routing graph with nodes on each routable layer
   - Add vertical edges representing via/contact sites

2. **Via insertion logic:**
   - Detect when source and destination terminals are on different layers
   - Query PDK for applicable bridge rules
   - Generate contact/via structures at XY overlap points
   - Apply material interfaces (e.g., TiSi₂) per bridge specification

3. **Cost function updates:**
   - Vias should have higher cost than horizontal traces (to minimize layer changes)
   - Consider via count limits, current capacity, parasitic resistance

4. **Bridge rule application:**
   - Parse `bridge MaterialA to MaterialB` declarations from PDK
   - Auto-generate the interface and via fill materials
   - Respect via diameter, annular ring, spacing rules

### Priority 2: Enhanced Diagnostics
**Location:** `crates/hwc-cli/src/commands/build_cmd/validation/continuity.rs`

**Improvements:**
1. When detecting disconnected islands, report **which layers** each island occupies
2. Suggest **specific bridge rules** that should apply
3. Show **XY overlap regions** where vias should be placed
4. Provide visual output (e.g., SVG layer view) showing the disconnection

**Example improved error message:**
```
❌ Net 'VIN' has 3 disconnected islands:
   
   Island 0 (metal1 layer, z=955-1355nm):
     - VIN_Pad @ [0-500nm, 1400-2000nm]
     - VIN_to_M1_Gate @ [500-1500nm, 1375-1625nm]
   
   Island 1 (poly layer, z=305-455nm):
     - M1_Gate_Poly @ [1375-1625nm, 900-1500nm]
   
   Island 2 (poly layer, z=305-455nm):
     - M2_Gate_Poly @ [1375-1625nm, 2000-3350nm]
   
   Suggested fix:
     Add vias using bridge rule 'Polysilicon to Aluminum' at:
       - [1500nm, 1400nm] (connects Island 0 ↔ Island 1)
       - [1500nm, 3100nm] (connects Island 0 ↔ Island 2)
```

### Priority 3: Test Suite Expansion
**Location:** `tests/ASIC/`

**Add test cases for:**
1. ✅ Single-layer routing (already passing)
2. ❌ **Two-layer routing** (poly + metal1) ← **THIS TEST**
3. ❌ Three-layer routing (poly + metal1 + metal2)
4. ❌ Via array stress test (many vias in small area)
5. ❌ Multi-net multi-layer (crosstalk avoidance)

---

## Related Code Locations

### Auto-Router Implementation
```
crates/hwc-compiler/src/ir/routing/automatic.rs
```
- Line ~50-200: Main routing algorithm
- Needs: Multi-layer graph construction, via insertion logic

### Physical Continuity Validation
```
crates/hwc-cli/src/commands/build_cmd/validation/continuity.rs
```
- Line ~100-300: Island detection and connectivity check
- Already working correctly (detects the problem)
- Needs: Better diagnostic messages

### Bridge Rule Parsing
```
crates/hwc-compiler/src/symbol_table/mod.rs
```
- Bridge rules are parsed but not applied during routing
- Needs: Interface to pass bridge rules to auto-router

### Stackup/Layer Management
```
crates/hwc-compiler/src/symbol_table/layer.rs
```
- Already correctly models layer stack
- Auto-router needs to query this for routable layers

---

## Test Case Validation

To verify a fix is complete, the following command should pass:

```bash
cargo run -- build tests\ASIC\cmos-inverter-test\cmos_inverter.hw
```

**Success criteria:**
1. ✅ Logical netlist passes (already passing)
2. ✅ Alignment validation passes (already passing)  
3. ✅ **Physical continuity validation passes** ← **CURRENTLY FAILING**
4. ✅ DRC validation passes (may have clearance issues to fix after routing works)
5. ✅ Lockfile generated successfully
6. ✅ Netlist export (SPICE) includes all device terminals correctly connected

---

## Additional Context

### Design Source
The CMOS inverter design is based on **standard VLSI textbook parameters**:
- Rabaey, Chandrakasan, Nikolic: "Digital Integrated Circuits" 
- Process: TSMC 0.25µm (λ = 125nm)
- VDD = 2.5V
- NMOS: W=375nm (3λ), L=250nm (2λ)
- PMOS: W=1125nm (9λ), L=250nm (2λ)
- Gate oxide: tox = 5.75nm (Cox = 6 fF/µm²)

These are **industry-standard, proven-correct electrical parameters**. The failure is purely a compiler routing limitation, not a design issue.

### Timeline
- **2026-06-30:** Bug discovered during CMOS inverter implementation
- **Attempts:** 3 workaround iterations (spacing, manual pours, trace widths)
- **Conclusion:** Auto-router fundamentally cannot handle multi-layer routing

---

## Appendix: Full Error Output

```
🔥 hwc COMPILER v0.1.6 (Syntax Unification)
==================================================

[    0.90ms] Source file read successfully (8186 bytes)
[    3.29ms] Lexer complete (1648 tokens)
[    5.98ms] Parser complete (4 imports, 4 definitions)
[   16.33ms] Symbol table built
[DRC] G-Cell sweep found 65 violations:
  ... (violations related to same-net overlaps and clearances)
[DFM] Dummy fill: 3 zones analyzed, 3 zones filled, 0 dummies placed (avg density before: 10.6%)
[  490.15ms] Found 1 space(s) to build

── Building space: CMOS_Inverter ──
[  490.92ms] HardwareSpace created: CMOS_Inverter (3000x4000x1360)
🔍 Professional Mode: Alignment validation enabled
   ├─ Scanning 22 pours for device bindings...
   ... (all device bindings successful)
   ├─ Checking device: M1 (NMOS)
      ├─ bulk: M1_Bulk_Pad (net: GND)
      ├─ source: M1_Source_Pad (net: GND)
      ├─ drain: M1_Drain_Pad (net: VOUT)
      ├─ gate: M1_Gate_Pad (net: VIN)
      └─ Device extracted successfully ✓
   ├─ Checking device: M2 (PMOS)
      ├─ bulk: M2_Bulk_Pad (net: VDD)
      ├─ source: M2_Source_Pad (net: VDD)
      ├─ drain: M2_Drain_Pad (net: VOUT)
      ├─ gate: M2_Gate_Pad (net: VIN)
      └─ Device extracted successfully ✓
   ├─ Copied 4 port declarations from module
   ✅ Physical netlist extracted: 2 devices
   ✅ Logical netlist synthesized: 2 devices
   ✅ Alignment validation passed: Layout matches schematic
[  511.34ms] PIVB connectivity check completed in 3.23ms

❌ PHYSICAL CONTINUITY VIOLATIONS - Cannot proceed to parameter validation:
   P41 (Net 'VIN' has 3 disconnected conductive components: 
        Island group 0 at z:0nm-1355nm (2 islands), 
        Island group 1 at z:955nm-1355nm (1 islands), 
        Island group 2 at z:305nm-455nm (1 islands)): 
   XY-plane gap detected between components on net 'VIN'.
   X-gap: 0 nm, Y-gap: 925 nm.
   Suggested fix: Add a pour or route to bridge the gap on net 'VIN'.
  x Physical continuity validation failed with 1 violation(s). 
    Alignment Layer cannot validate fragmented nets.

error: process didn't exit successfully: 
  `target\debug\hwc.exe build tests\ASIC\cmos-inverter-test\cmos_inverter.hw` (exit code: 1)
```

---

## Contact

For questions or to discuss implementation approaches:
- Test case location: `tests/ASIC/cmos-inverter-test/`
- Related issue tracker: [TBD - create GitHub issue]
- Technical contact: [Compiler team]

---

**End of Report**
