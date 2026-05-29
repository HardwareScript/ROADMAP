# Hardware Script v0.1.6: Proof of Work Implementation

**Status**: Planning  
**Priority**: CRITICAL  
**Goal**: Validate every compiler layer against physical reality through escalating complexity tests

---

## Philosophy

Talk is cheap. Compiling to physical reality is the only metric that matters.

This document defines the **Hardware Script Master Proof-of-Work Roadmap** — an escalating gauntlet from microscopic silicon atoms to a fully routed System-on-Chip (SoC). Each stage stress-tests a different compiler subsystem:

- **Lexer/Parser**: Syntax validation, boundary laws, identifier resolution
- **Logic Synthesizer**: Gate-level netlist generation, combinational loop detection
- **Voxel Routing Engine**: 3D placement, Manhattan routing, differential pairs
- **Physics DRC**: Clearance rules, thermal analysis, parasitic extraction
- **Export Targets**: GLB (Visual), DXF (Physical), SPICE (Electrical)

If the compiler has a gap, this gauntlet will expose it.

---

## Roadmap Stages

### Stage 1: The Silicon Foundry (Atoms & Bare-Metal Transistors)

**Testing**: Voxel Engine 3D placement, Materials MVP, Silicon Export

**Artifacts**:
- Custom Silicon Material Definitions (N-Doped, P-Doped, SiO₂)
- Raw NMOS Transistor (Gate, Source, Drain physical geometry)
- Raw PMOS Transistor
- CMOS Inverter (NMOS + PMOS in 3D space)

**Compilation Targets**: 
- `hwc build --target dxf` (Physical layout)
- `hwc build --target glb` (3D visualization)

**Viewer Applications**: 
- **DXF**: KLayout or LibreCAD (for layout verification)
- **GLB**: Blender or Windows 3D Viewer (for 3D visualization)

**Success Criteria**:
- [ ] Materials imported correctly from stdlib (Silicon_N, Silicon_P, SiO2)
- [ ] Transistor geometries render as valid polygons in DXF
- [ ] CMOS inverter shows proper NMOS/PMOS stacking in 3D (GLB)
- [ ] DXF imports cleanly into KLayout for GDSII conversion
- [ ] GLB displays correct physical dimensions with PBR materials (metallic, roughness)
- [ ] Device extraction identifies NMOS and PMOS transistors correctly

**Implementation Files**:
- `examples/pow/stage1_silicon/nmos.hw` - NMOS transistor (imports Silicon_N, Silicon_P from stdlib)
- `examples/pow/stage1_silicon/pmos.hw` - PMOS transistor (imports Silicon_N, Silicon_P from stdlib)
- `examples/pow/stage1_silicon/cmos_inverter.hw` - Combined inverter

**Note**: Use v0.1.6 import system instead of custom material definitions:
```hw
import Silicon_N, Silicon_P, SiO2 from @std/materials/semiconductors
```

---

### Stage 2: The Analog & RF Sandbox (Continuous Physics)

**Testing**: Parasitic Extraction (RCX), Time-Domain simulation, Exotic Materials

**Artifacts**:
- Passive RC Low-Pass Filter (analog math primitives)
- GaN HEMT (Gallium Nitride High Electron Mobility Transistor) for RF
- `test:` block asserting 10ms transient charge curve

**Compilation Target**: `hwc build --target spice`

**Viewer Application**: LTspice or Ngspice

**Success Criteria**:
- [ ] RC filter generates valid SPICE netlist
- [ ] GaN HEMT uses exotic material properties correctly
- [ ] Time-domain simulation shows expected voltage curves
- [ ] Frequency response matches theoretical Bode plot
- [ ] Thermal DRC validates GaN thermal properties

**Implementation Files**:
- `examples/pow/stage2_analog/rc_filter.hw` - Low-pass filter
- `examples/pow/stage2_analog/gan_hemt.hw` - RF transistor
- `examples/pow/stage2_analog/transient_test.hw` - Time-domain assertions

---

### Stage 3: The Package Foundry (Components & HDI Vias)

**Testing**: Universal List Syntax `[]`, Array Indexing, Pad Shapes

**Artifacts**:
- SMD 0805 Resistor (Obround pad shapes, metadata dictionary)
- 256-Pin BGA Package (`for i in 0..255` comptime stamping, Circle pads)
- HDI Microvias (Z-axis layer transitions)

**Compilation Targets**: 
- `hwc build --target dxf` (Physical layout)
- `hwc build --target glb` (3D visualization)

**Viewer Applications**:
- **DXF**: KiCad or LibreCAD (for layout verification, can export to Gerber)
- **GLB**: Blender or Windows 3D Viewer (for 3D representation)

**Success Criteria**:
- [ ] 0805 resistor pads render as proper obrounds in DXF
- [ ] BGA generates exactly 256 circular pads in grid pattern
- [ ] Comptime loop correctly stamps all pad instances
- [ ] HDI microvias show proper layer transitions in 3D (GLB)
- [ ] Substrate cutouts work correctly for mounting holes (v0.1.6 feature)
- [ ] DXF imports into KiCad without errors
- [ ] DXF can be exported to Gerber from KiCad for fab houses
- [ ] GLB model displays correct physical dimensions with PBR materials

**Implementation Files**:
- `examples/pow/stage3_packages/resistor_0805.hw` - SMD resistor
- `examples/pow/stage3_packages/bga_256.hw` - BGA package
- `examples/pow/stage3_packages/hdi_vias.hw` - Microvia structures
- `examples/pow/stage3_packages/mounting_holes.hw` - Substrate cutouts demo (v0.1.6 feature)

---

### Stage 4: Digital Logic Synthesis (Soft IP)

**Testing**: Native Logic Synthesizer, Recursive MuxTrees, Boundary Law (`=` vs `:`)

**Artifacts**:
- 8-bit Ripple Carry Adder (`xor`, `and`, `or` keywords)
- D-Flip-Flop Register (`reg` primitive, `.next` state)
- Traffic Light State Machine (`enum`, `match` statements)

**Compilation Target**: `hwc check` and `hwc sim --mode digital`

**Viewer Application**: Native CLI Output

**Success Criteria**:
- [ ] Adder synthesizes without L02 (Width Mismatch) errors
- [ ] No L03 (Combinational Loop) errors detected
- [ ] Register correctly implements sequential logic using lowercase `reg` primitive
- [ ] State machine transitions validate in simulation
- [ ] Boundary law violations caught at compile time (`:` vs `=`)
- [ ] Logic operators work correctly (`and`, `or`, `xor`, `not`, `mod`)
- [ ] Comparisons use single `=` operator (v0.1.6 syntax)
- [ ] Internal netlist generation succeeds

**Implementation Files**:
- `examples/pow/stage4_logic/ripple_adder.hw` - 8-bit adder
- `examples/pow/stage4_logic/dff_register.hw` - Flip-flop
- `examples/pow/stage4_logic/traffic_light.hw` - State machine

---

### Stage 5: The High-Speed PCB (Constraint-Aware Routing)

**Testing**: Leap-Frog Router, SDF generation, Pattern Constraints

**Artifacts**:
- Differential Pairs (parallel trace coupling, EM clearance)
- DDR5 Data Bus (`for` loops mapping `Data[0..7]` pins)
- "Trombone" Length Matching (all traces exact same length in mm)
- Copper GND Pour with Thermal Reliefs (polygon rasterization, spokes)

**Compilation Targets**: 
- `hwc build --target dxf` (Physical layout)
- `hwc build --target glb` (3D visualization)

**Viewer Applications**: 
- **DXF**: KiCad or LibreCAD (can export to Gerber for manufacturing)
- **GLB**: Blender (for 3D trace visualization)

**Success Criteria**:
- [ ] Differential pairs maintain coupling constraints in DXF
- [ ] DDR5 bus routes all 8 data lines correctly using `[]` array syntax
- [ ] Trombone routing achieves length matching within tolerance
- [ ] GND pour generates 4-spoke thermal reliefs on pads
- [ ] No EM clearance violations
- [ ] Materials imported from stdlib (Copper, FR4) using v0.1.6 import system
- [ ] DXF imports into KiCad and exports to Gerber successfully
- [ ] GLB shows physical copper traces in 3D with PBR materials

**Implementation Files**:
- `examples/pow/stage5_pcb/diff_pairs.hw` - Differential routing
- `examples/pow/stage5_pcb/ddr5_bus.hw` - Memory interface
- `examples/pow/stage5_pcb/length_match.hw` - Trombone routing
- `examples/pow/stage5_pcb/gnd_pour.hw` - Copper pour with reliefs

---

### Stage 6: The Pinnacle - Full System-on-Chip (SoC) Assembly

**Testing**: Hierarchical Floorplanner, Module Flattening, Substrate Memory Scaling

**Artifacts**:
- ALU Module (from Stage 4)
- Register File (Array[32], generic instantiation)
- Instruction Decoder
- Top-Level Space (CPU core on Silicon Substrate)

**Compilation Targets**: 
- `hwc build --target dxf` (Physical layout)
- `hwc build --target glb` (3D visualization)
- `hwc build --target spice` (Electrical verification)

**Viewer Applications**:
- **DXF**: KLayout (for silicon layout, can export to GDSII for foundries)
- **GLB**: Blender (for 3D multi-layered chip structure)
- **SPICE**: ngspice or LTspice (for electrical simulation)

**Success Criteria**:
- [ ] All modules integrate without namespace conflicts (use namespace aliases if needed)
- [ ] Hierarchical floorplan places components correctly
- [ ] DXF imports into KLayout and exports to GDSII successfully
- [ ] Register file array instantiation works at scale using `[]` syntax
- [ ] 3D visualization (GLB) shows complete chip layers with PBR materials
- [ ] SPICE netlist simulates without errors (device extraction working)
- [ ] Memory usage remains O(1) per substrate layer (sparse architecture)
- [ ] Compilation completes in ~1 second (v0.1.6 performance target)
- [ ] Final GDSII (via KLayout) passes DRC checks

**Implementation Files**:
- `examples/pow/stage6_soc/alu.hw` - Arithmetic logic unit
- `examples/pow/stage6_soc/register_file.hw` - Register array
- `examples/pow/stage6_soc/decoder.hw` - Instruction decoder
- `examples/pow/stage6_soc/cpu_top.hw` - Top-level integration

---

## The "Universal Master" Philosophy

Hardware Script adopts the **"CorelDRAW Vision"** for exports: one source language, three perfect outputs, infinite downstream compatibility.

### The God-Tier Trio

1. **Visual Truth (GLB)**: High-fidelity 3D visualization with PBR materials
2. **Physical Truth (DXF)**: Universal 2D layout for both silicon and board manufacturing
3. **Electrical Truth (SPICE)**: Mathematical circuit model for simulation

### The Handoff Workflow

Hardware Script becomes the **Universal Master** that generates DXF, which then hands off to specialized tools:

**For PCB Manufacturing:**
1. Hardware Script generates `layout.dxf`
2. Open in KiCad
3. Export to Gerber
4. Send to fab house (JLCPCB, PCBWay, etc.)

**For Silicon Manufacturing:**
1. Hardware Script generates `layout.dxf`
2. Open in KLayout
3. Export to GDSII
4. Submit to foundry (TSMC, GlobalFoundries, etc.)

This approach provides:
- **Stability**: DXF has 40+ years of tool support
- **Universality**: One format for both silicon and board layouts
- **Flexibility**: Users can apply process-specific rules in their preferred tools
- **Simplicity**: Three orthogonal formats with zero overlap

---

## Execution Strategy

### Phase 1: Setup (Current)
1. Create project directory structure under `examples/pow/`
2. Document expected compiler behavior for each stage
3. Define success criteria and validation procedures

### Phase 2: Implementation (Sequential)
For each stage:
1. Write strictly valid v0.1.6 Hardware Script (no pseudocode)
   - Use stdlib imports for materials: `import Copper from @std/materials/conductors`
   - Use lowercase `reg` primitive for registers
   - Use single `=` for both assignment and comparison
   - Use `mod` keyword instead of `%` operator
   - Use `[]` bracket notation for all lists
   - Use `and`, `or`, `xor`, `not` logic operators
2. Analyze lexer/parser processing requirements
3. Map expected output file contents (`.dxf`, `.glb`, `.sp`)
4. Execute: `hwc build --target <format>`
5. View output in designated application
6. For manufacturing workflows, verify DXF conversion:
   - Silicon: DXF → GDSII via KLayout
   - PCB: DXF → Gerber via KiCad
7. Verify against success criteria
8. If pass: check box, move to next stage
9. If fail: debug compiler gap, document issue, fix, retry

### Phase 3: Documentation
- Screenshot all viewer outputs
- Document any compiler bugs discovered
- Create validation report for each stage
- Update compiler implementation based on gaps found

---

## Compiler Subsystems Under Test

| Stage | Lexer | Parser | Synthesizer | Router | Physics | Export |
|-------|-------|--------|-------------|--------|---------|--------|
| 1. Silicon | ✓ | ✓ | - | ✓ | ✓ | DXF, GLB |
| 2. Analog | ✓ | ✓ | - | - | ✓ | SPICE |
| 3. Packages | ✓ | ✓ | - | - | ✓ | DXF, GLB |
| 4. Logic | ✓ | ✓ | ✓ | - | - | Netlist |
| 5. PCB | ✓ | ✓ | - | ✓ | ✓ | DXF, GLB |
| 6. SoC | ✓ | ✓ | ✓ | ✓ | ✓ | DXF, GLB, SPICE |

---

## Critical Validation Points

### Syntax Validation
- No `define` keyword usage (removed in v0.1.6)
- Correct use of `:` (declaration) vs `=` (assignment/comparison)
- Bare identifiers without quotes
- Universal list syntax `[]` for arrays
- Lowercase `reg` primitive (not `Reg`)
- `mod` keyword for modulo (not `%`)
- Logic operators: `and`, `or`, `xor`, `not` (word forms)
- Stdlib imports for materials (not custom definitions)

### Semantic Validation
- L02: Width mismatch detection
- L03: Combinational loop detection
- Namespace collision prevention
- Type inference correctness

### Physical Validation
- DRC clearance rules enforcement
- Thermal analysis for high-power components
- Parasitic extraction accuracy
- Layer stack validation

### Export Validation
- DXF polygon correctness and layer mapping
- DXF import compatibility (KLayout, KiCad, LibreCAD)
- SPICE netlist syntax and simulation readiness
- GLB/GLTF mesh generation and PBR materials
- DXF-to-GDSII conversion via KLayout
- DXF-to-Gerber conversion via KiCad

---

## Dependencies

### Required Tools
- **hwc**: Hardware Script compiler (v0.1.6+)
- **KLayout**: DXF viewer and GDSII converter (for silicon workflows)
- **KiCad**: DXF viewer and Gerber converter (for PCB workflows)
- **LibreCAD**: Alternative DXF viewer
- **LTspice** or **Ngspice**: SPICE simulation
- **Blender**: 3D model viewer (GLB/GLTF)

### Optional Tools
- **Windows 3D Viewer**: Quick GLB preview
- **FreeCAD**: Alternative DXF viewer
- **QCAD**: Lightweight DXF editor

---

## Next Steps

1. **Immediate**: Create `examples/pow/` directory structure
2. **Stage 1**: Implement Silicon Foundry artifacts
3. **Validation**: Run through first compilation cycle
4. **Iteration**: Debug any compiler gaps discovered
5. **Documentation**: Capture screenshots and results
6. **Progression**: Move to Stage 2 upon Stage 1 completion

---

## Success Metrics

**Project Complete When**:
- All 6 stages pass validation
- All checkboxes marked complete
- All viewer applications display correct output
- Zero compiler crashes
- All DRC checks pass
- Documentation includes visual proof (screenshots)

**Compiler Ready for v0.1.6 Release When**:
- This entire PoW gauntlet passes without manual intervention
- All export targets generate valid output
- Performance remains acceptable at SoC scale (Stage 6)

---

*This is the crucible. Every line of Hardware Script code written here must compile to physical reality.*
