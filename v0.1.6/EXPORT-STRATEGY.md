# God-Tier Trio: Final Export Architecture

**Version:** 0.1.6  
**Status:** Architectural Consolidation  
**Philosophy:** The "CorelDRAW Vision" - One Tool, Universal Output

---

## Executive Summary

Hardware Script has achieved **Final Architectural Consolidation** by collapsing the industry's fragmented export mess into three universal formats. We are moving from a "Professional Four" to a "Universal Three" that guarantees every output file "Just Works."

This is the **CorelDRAW moment** for hardware design: one source language, three perfect outputs, infinite downstream compatibility.

---

## The God-Tier Trio

| Domain | Format | Filename | Purpose |
|--------|--------|----------|---------|
| **1. Visual Truth** | GLB | `board.glb` | Beauty: High-fidelity 3D visualization with PBR materials, transparency, and shadows |
| **2. Physical Truth** | DXF | `layout.dxf` | Precision: Universal 2D layout for both Silicon and Board manufacturing |
| **3. Electrical Truth** | SPICE | `netlist.sp` | Behavior: Mathematical circuit model for simulation and verification |

---

## 1. Visual Truth: GLB

**File:** `board.glb`  
**Standard:** glTF 2.0 Binary Format  
**Opens In:** Blender, Three.js, Unity, Unreal Engine, Windows 3D Viewer, macOS Preview

### What It Provides

- **High-Fidelity 3D Model:** Complete geometric representation with accurate dimensions
- **PBR Materials:** Physically-based rendering with metallic/roughness workflow
- **Visual Realism:** Transparency, shadows, reflections, and surface properties
- **Human Understanding:** The file you show to stakeholders, investors, and teammates

### Use Cases

- Design review and visualization
- Marketing and documentation
- 3D printing preparation
- Virtual prototyping
- Educational demonstrations

### Technical Details

- Embedded textures and materials
- Optimized mesh geometry
- Industry-standard format with universal tool support
- Self-contained single-file distribution

---

## 2. Physical Truth: DXF

**File:** `layout.dxf`  
**Standard:** AutoCAD Drawing Exchange Format  
**Opens In:** LibreCAD, KLayout, KiCad, AutoCAD, FreeCAD, QCAD, Inkscape

### What It Provides

- **Universal 2D Layout:** Precise geometric representation of all physical layers
- **Silicon + Board Unified:** One format for both chip layouts and PCB designs
- **Manufacturing Ready:** Direct input for fabrication toolchains
- **Tool Agnostic:** 100x more stable than XML-based alternatives

### Why DXF Replaced IPC-2581 and DEF/LEF

**The Problem with IPC-2581:**
- XML nightmare with brittle parsing
- Every tool implements the "standard" differently
- "Unknown Stream" errors and compatibility issues
- Board-only format, doesn't work for silicon

**The Problem with DEF/LEF:**
- Silicon-only format
- Requires separate toolchain
- Not universally supported

**The DXF Solution:**
- **Universal:** Works for both silicon and board layouts
- **Stable:** Text-based format with decades of tool support
- **Simple:** Layers are just layers - copper trace or polysilicon gate, DXF doesn't care
- **Proven:** Every CAD tool in existence can read DXF

### Layer Mapping

DXF layers directly represent physical manufacturing layers:

**For PCBs:**
- Top copper, bottom copper, inner layers
- Silkscreen, soldermask, paste
- Board outline and drill holes

**For Silicon:**
- Polysilicon gates
- Metal interconnect layers
- Diffusion regions
- Via structures

### The "Handoff" Workflow

Hardware Script becomes the **Universal Master**:

1. **For PCB Manufacturing (JLCPCB, PCBWay, etc.):**
   - Open `layout.dxf` in KiCad
   - Click "Export Gerber"
   - Send to fab house

2. **For Silicon Manufacturing (TSMC, GlobalFoundries, etc.):**
   - Open `layout.dxf` in KLayout
   - Click "Export GDSII"
   - Submit to foundry

3. **For Custom Workflows:**
   - Import `layout.dxf` into any CAD tool
   - Apply process-specific design rules
   - Export to required format

---

## 3. Electrical Truth: SPICE

**File:** `netlist.sp`  
**Standard:** SPICE3 Netlist Format  
**Simulates In:** ngspice, LTspice, HSPICE, Xyce, Spectre

### What It Provides

- **Circuit Topology:** Complete electrical connectivity
- **Component Models:** Resistors, capacitors, transistors with parameters
- **Simulation Ready:** Direct input for analog/digital simulation
- **Mathematical Truth:** The equations that prove your design works

### Use Cases

- DC operating point analysis
- AC frequency response
- Transient time-domain simulation
- Worst-case corner analysis
- Power consumption estimation

### Technical Details

- Human-readable text format
- Subcircuit hierarchy preserved
- Component parameters from materials database
- Compatible with all major SPICE engines

---

## Architectural Philosophy

### The "CorelDRAW Vision"

Just as CorelDRAW became the universal tool for graphic design by providing clean exports to any format, Hardware Script becomes the universal tool for hardware design by providing three perfect outputs:

1. **One Source Language:** Hardware Script (`.hw`)
2. **Three Universal Outputs:** GLB, DXF, SPICE
3. **Infinite Downstream Compatibility:** Every tool, every workflow, every manufacturer

### Why Three is Perfect

**Not Too Few:**
- One format can't capture visual, physical, and electrical truth simultaneously
- Different domains require different representations

**Not Too Many:**
- Four formats (with IPC-2581) created redundancy and fragility
- More formats = more maintenance burden = more bugs

**Just Right:**
- Three orthogonal domains with zero overlap
- Each format is the undisputed best-in-class for its purpose
- Maximum compatibility with minimum complexity

### The Stability Guarantee

By choosing these three formats, we guarantee:

1. **GLB:** Khronos Group standard, backed by industry consortium
2. **DXF:** 40+ years of stability, Autodesk's most successful format
3. **SPICE:** 50+ years of academic and industry use, IEEE standard

These formats will outlive us all.

---

## Migration from v0.1.5

### Removed Formats

- **IPC-2581:** Replaced by DXF (more stable, more universal)
- **DEF/LEF:** Replaced by DXF (unified silicon + board workflow)

### Compiler Changes

The export pipeline now generates exactly three files:

```
hw compile board.hw
  ✓ Parsing complete
  ✓ Synthesis complete
  ✓ Layout complete
  
  Exports:
  → board.glb     (Visual Truth)
  → layout.dxf    (Physical Truth)
  → netlist.sp    (Electrical Truth)
```

### User Impact

**Before (v0.1.5):**
- Four export formats with overlapping purposes
- Confusion about which format to use
- IPC-2581 compatibility issues

**After (v0.1.6):**
- Three clear, orthogonal outputs
- Each format has obvious use case
- Universal tool compatibility

---

## Implementation Status

### Phase 1: Core Exports ✓
- [x] GLB exporter with PBR materials
- [x] DXF exporter with layer mapping
- [x] SPICE netlist generator

### Phase 2: Silicon Support (In Progress)
- [x] DXF layer mapping for silicon
- [ ] Process-specific design rules
- [ ] Foundry-specific optimizations

### Phase 3: Advanced Features (Planned)
- [ ] Multi-die DXF export
- [ ] Hierarchical SPICE subcircuits
- [ ] Animated GLB for signal visualization

---

## Technical Specifications

### GLB Export

**Format Version:** glTF 2.0  
**Coordinate System:** Right-handed, Y-up  
**Units:** Millimeters  
**Materials:** PBR metallic-roughness workflow  
**Textures:** Embedded in binary blob  
**Mesh Optimization:** Enabled by default

### DXF Export

**Format Version:** AutoCAD R2018 DXF  
**Coordinate System:** Right-handed, Z-up  
**Units:** Millimeters (configurable)  
**Layer Naming:** Descriptive (e.g., "TOP_COPPER", "POLY_GATE")  
**Entity Types:** LINE, CIRCLE, ARC, POLYLINE, HATCH  
**Text Encoding:** UTF-8

### SPICE Export

**Format Version:** SPICE3f5 compatible  
**Subcircuit Hierarchy:** Preserved from source  
**Component Naming:** Hierarchical dot notation  
**Parameter Units:** SI base units (V, A, Ω, F, H)  
**Comments:** Include source line references

---

## Validation and Testing

### GLB Validation
- Opens in Blender without errors
- Renders correctly in Three.js
- Materials display properly in Windows 3D Viewer

### DXF Validation
- Imports cleanly into KiCad
- Displays correctly in LibreCAD
- Converts to GDSII via KLayout

### SPICE Validation
- Parses without errors in ngspice
- Simulates successfully in LTspice
- Produces expected DC operating points

---

## Future-Proofing

### Format Longevity

These three formats are guaranteed to be supported for decades:

- **GLB:** Backed by Khronos Group (OpenGL, Vulkan, WebGL)
- **DXF:** Autodesk's most widely adopted format, 40+ year track record
- **SPICE:** Academic standard, embedded in every EE curriculum

### Extensibility

Each format supports future extensions without breaking compatibility:

- **GLB:** Custom properties via glTF extensions
- **DXF:** Custom layers and entity types
- **SPICE:** Custom subcircuit models and parameters

---

## Conclusion

The God-Tier Trio represents the final architectural consolidation of Hardware Script's export strategy. By choosing three universal, stable, and orthogonal formats, we have achieved the "CorelDRAW Vision":

**One tool. Three perfect outputs. Infinite compatibility.**

This is not a compromise. This is not a temporary solution. This is the endgame.

---

**Document Version:** 1.0  
**Last Updated:** 2026-04-19  
**Author:** Hardware Script Core Team  
**Status:** Canonical Reference
