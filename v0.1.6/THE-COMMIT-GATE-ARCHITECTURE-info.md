# The Commit Gate Architecture: From Passive Guard to Active Enforcer

**Document Type**: Architectural Gap Analysis & Roadmap  
**Status**: Critical - Defines v0.1.6 Direction  
**Date**: May 2026  
**Context**: Response to "Floating Bricks" and "Rubbish Export" Problem

---

## Executive Summary: The Current State

Hardware Script is at a **critical architectural crossroad**. The compiler currently acts as a "Passive Guard" - it reports violations but still exports broken designs. This document defines the path to becoming an "Active Enforcer" that refuses to produce physically invalid output.

**The Problem**: Users can build designs with 25 physics violations and still get exported files (GLB, DXF, SPICE) showing "floating transistors" and "disconnected nets."

**The Solution**: Implement the **Commit Gate Architecture** - a strict validation pipeline that blocks all exports if physical integrity fails.

---

## Part 1: Where We Are (The "Physics is Honest" Phase)

### The "Floating Bricks" Reality

When you see "floating bricks" in the 3D viewer, the compiler is showing you **Physical Truth**:

1. **The Floating Problem**: User places NMOS at `z:4`, substrate ends at `z:2` → literal vacuum at `z:3`
2. **The Embedding Problem**: Bulk pours at `z:1` inside substrate (`z:0-2`) → P42 short circuit detected
3. **The Auto-Via Failure**: "No XY overlap" or "Overlap region too small" → electrons can't flow between layers

**Current Behavior**: The compiler runs Sprint 2.3 Island Builder, finds 25 violations, prints a report, and **still exports the broken design**.

### Why This Happens

The compiler currently has **no Commit Layer**. It treats "Building the Voxel Grid" and "Writing Export Files" as the same operation:

```
Current Pipeline (BROKEN):
1. Parse source code
2. Build voxel grid in RAM
3. Run physics validation (finds violations)
4. Export files anyway ← THE PROBLEM
5. User sees "floating bricks" in viewer
```

**The Zen Reality Check**: In Altium or Cadence, you could design this exact same oscillator and the tool would show a "flat green board" that looks perfect. You'd pay $10,000 for fabrication and get back a chip that doesn't work. **Hardware Script caught the error in 334 milliseconds** - but then it exported the broken design anyway.

---

## Part 2: The Commit Gate Architecture (The Solution)

### The Memory-First Pipeline

We move exporters (GLB, DXF, SPICE) to the **very end** of the chain. They become "Validated Triggers."

```
New Pipeline (CORRECT):
┌─────────────────────────────────────────────────────────┐
│ Phase 1: Virtual Build (Parallel, In-Memory)           │
├─────────────────────────────────────────────────────────┤
│ • Synthesize logic                                      │
│ • Unroll geometry into Voxel Grid (RAM only)            │
│ • Status: Everything in memory, ZERO files written      │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ Phase 2: Physical Integrity Scan (Parallel)            │
├─────────────────────────────────────────────────────────┤
│ • Run Island Builder (Sprint 2.3)                       │
│ • Run DRC (clearance, shorts)                           │
│ • Run Alignment Layer (Triple-Check Architecture)       │
│ • Status: Compiler has list of violations               │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ Phase 3: THE COMMIT GATE (The Decision)                │
├─────────────────────────────────────────────────────────┤
│ IF (violations == 0) OR (mode == Artist):              │
│   → PROCEED to Phase 4                                  │
│ ELSE:                                                   │
│   → ABORT. Print error report. STOP.                    │
│   → Result: NO FILES CREATED. Folder stays clean.       │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ Phase 4: Physical Realization (Export)                 │
├─────────────────────────────────────────────────────────┤
│ • NOW write .glb, .dxf, .sp files                       │
│ • Status: Only perfect designs reach this phase         │
└─────────────────────────────────────────────────────────┘
```

### Why This Preserves Speed

**Speed comes from memory-parallelism, not early-writing.**

- Disk I/O is the **slowest** part of a computer
- Checking islands and DRC simultaneously using multiple CPU cores while design is "hot" in RAM is **fast**
- By waiting until the end to build files, we **save time** - never waste CPU cycles generating complex 3D meshes for broken designs

**Performance Impact**: None. Sub-second builds maintained because validation happens in parallel on in-memory data.

---

## Part 3: Artist Mode vs. Architecture Mode

### The Two Modes

Hardware Script must support two distinct workflows:

#### Artist Mode (No `implements` keyword)
```hw
space MyExperiment:
    # Just sketching, testing ideas
    add pour(Silicon_N) at [x: 2mm, y: 4mm, z: 4]  # Might be floating
```

**Compiler Behavior**:
- Acts as "Helpful Lab Assistant"
- Allows "rubbish" because user might be testing a new idea
- Prints warnings but still exports files
- Result: **Build successful (with warnings). Files generated.**

#### Architecture Mode (`implements` keyword present)
```hw
module RingOscillator:
    # Logical specification
    add NMOS named M1
    route M1.gate to VIN

space RingOscillatorLayout implements RingOscillator:
    # Physical implementation - MUST match module
    add NMOS named M1 at [x: 2mm, y: 4mm, z: 4]  # Floating!
```

**Compiler Behavior**:
- Acts as "Draconian Site Inspector"
- User has made a **Logical Claim** (implements)
- 1 Violation = 0 Files
- Result: **Build failed. Zero files generated.**

### The Contract

When you write `space X implements Y`, you sign a **Physical Integrity Contract**:

1. **Requirement**: Logic says Net `STAGE1_OUT` connects M1 to M3
2. **Validation**: Island Builder must find **exactly one** physical island for `STAGE1_OUT`
3. **Failure**: If it finds 2 islands (a gap), the Alignment Check **FAILS** and the Commit Gate **CLOSES**

---

## Part 4: Immediate Z-Axis Integrity (Gap 2 Fix)

### The Problem

Currently, the compiler waits until the **end** of the build to check connectivity. That's too late.

```hw
space Test:
    substrate: Silicon from [0, 0, 0] to [10mm, 10mm, 2]  # Ends at z:2
    add NMOS at [x: 2mm, y: 4mm, z: 4]  # Floating at z:4!
    # Compiler says nothing until Phase 3 (too late)
```

### The Fix: Substrate Validation at Placement Time

Move substrate validation into the `add` statement itself:

```rust
// In component placement code:
fn place_component(component: &Component, position: Position, space: &Space) -> Result<()> {
    let z = position.z;
    let substrate = &space.substrate;
    
    // IMMEDIATE CHECK
    if z < substrate.min_z {
        return Err(Error::ComponentBuriedInSubstrate {
            component: component.name.clone(),
            z_position: z,
            substrate_min: substrate.min_z,
        });
    }
    
    if z > substrate.max_z {
        return Err(Error::ComponentFloatingInAir {
            component: component.name.clone(),
            z_position: z,
            substrate_max: substrate.max_z,
            gap: z - substrate.max_z,
        });
    }
    
    // Only proceed if Z is valid
    Ok(())
}
```

**Result**: In Architecture Mode, the compiler **forbids** any Z-coordinate that creates a physical gap. You won't see floating bricks because the compiler will **refuse to place them**.

---

## Part 5: The "Structured Assembly" Philosophy

### Zero Hidden Magic

Hardware Script follows the **Zig/Assembly philosophy**: The programmer is always right, even when they are wrong.

**We do NOT implement "Automatic Snapping"** - that would make us exactly like the bloated legacy tools we're trying to kill.

### From Hardcoded to Structured Assembly

Instead of hiding geometry, we give users **Spatial Variables**:

```hw
# HARDCODED (Current - Error-Prone)
add NMOS at [x: 2mm, y: 4mm, z: 4]
# Problem: If substrate changes, this number is wrong. It floats.

# STRUCTURED (Next Step - Explicit Control)
add NMOS at [x: 2mm, y: 4mm, z: substrate.max_z]
# This is NOT a black box. User explicitly chose to use a variable.
# If they want 1 micron higher: z: substrate.max_z + 1um
```

**Key Insight**: This is still Assembly. The user has 100% control. But now they can write **more accurate code** because they have access to spatial context.

### The "Linker" is an Error Message, Not a Mover

In C, the linker doesn't change your code; it tells you: "You asked for function X, but I can't find it in the memory map."

Hardware Script's "Geometric Linker" works the same way:

1. User defines geometry: `NMOS at z: 4`
2. Compiler runs Physical Continuity: Sees gap at `z: 3`
3. **Result**: Refuses to build. Does NOT snap. Throws error:
   ```
   Error P44: Disconnected Atom
   NMOS at z:4 is floating. Substrate ends at z:2.
   Gap of 2 layers detected.
   
   Suggested fix: Change z:4 to z:2 (substrate.max_z)
   ```

**The Zen Way**: User retains 100% control, but compiler provides Physical Truth so user can make informed decisions.

---

## Part 6: The Three Critical Gaps

Before implementing the Commit Gate, we must address three gaps that would make the developer experience frustrating instead of "Zen":

### Gap 1: The "Ghost View" (Visual Debugger)

**The Problem**: If the Commit Gate blocks the `.glb` export because of a floating transistor, the user is **blind**. They can read the error message, but they can't see the spatial relationship that caused the error.

**The Fix**: Diagnostic Export Target

- In Architecture Mode, if build fails, compiler refuses to generate "Real" production files (`board.glb`)
- Instead, it generates `build/debug/integrity_report.glb`
- **The Difference**: This file only renders the **Violations**
  - Floating components highlighted in **glowing Yellow**
  - Short circuits highlighted in **glowing Red**
  - Disconnected islands shown with **gap measurements**
- **Result**: User sees a "Ghost View" of their error. They can visually inspect the rubbish to understand the fix, but they cannot use this file for manufacturing.

**Debug Output Structure**:
```
build/
├── RingOscillatorLayout/
│   ├── board.glb          # Only created if build succeeds
│   ├── board.dxf          # Only created if build succeeds
│   └── netlist.sp         # Only created if build succeeds
└── debug/                 # Only created if --debug flag used OR build fails
    ├── integrity_report.glb    # Violation visualization
    ├── violation_log.txt       # Detailed error report
    └── physics_state.json      # Full physics validation data
```

### Gap 2: Intentional Embedding (The "Silicon Law")

**The Problem**: In silicon design, "Bulk" pours are **supposed to be** embedded in the "Substrate" (doping happens inside the wafer). If the Commit Gate sees this as a "Short Circuit" (P42) and stops the build, the user is blocked from designing real chips.

**The Fix**: Explicit Overlap Waivers

- **Default**: Overlap is a Fatal Error
- **Explicit Intent**:
  ```hw
  add pour(Silicon_P) named Bulk:
      at: [x: 2mm, y: 4mm, z: 1]
      attributes: [allow_substrate_overlap]  # Whitelists this violation
  ```
- **Result**: Maintains "Zero Hidden Magic" rule. Overlapping is forbidden unless the user explicitly signs a waiver for that specific atom.

**Waiver Types**:
- `allow_substrate_overlap` - Pour can be inside substrate (for doping wells)
- `allow_component_overlap` - Component can overlap with another (for merged regions)
- `allow_floating` - Component can be disconnected from substrate (for air-gap devices)

### Gap 3: Batch Placement Validation (The "Cascade" Problem)

**The Problem**: If the compiler stops at the **first** floating component it finds in a `for` loop, the user has to fix 64 errors one-by-one. This is the opposite of speed.

**The Fix**: Batch Gate

- Compiler must **finish the entire unrolling pass** (Phase 1) even if it finds errors
- Collects **every** floating component into a Violation Vector
- Only after unrolling is done does it check the vector and decide whether to close the gate
- **Result**: User gets a report of **all 64 floating transistors at once**. They fix the loop math once, and the gate opens.

**Example Output**:
```
❌ BUILD FAILED - 64 violations found

Violations by type:
  • P44 (Floating Component): 64 instances
    - Adder[0] at z:4 (substrate ends at z:2, gap: 2 layers)
    - Adder[1] at z:4 (substrate ends at z:2, gap: 2 layers)
    - Adder[2] at z:4 (substrate ends at z:2, gap: 2 layers)
    ... (61 more)
    
💡 Pattern detected: All violations in loop at line 45
   Suggested fix: Change 'z: 4' to 'z: substrate.max_z'
```

---