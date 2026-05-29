# System 5 Implementation Plan: Export Generation & Assets

**Hardware Script v0.1.2**  
**Focus**: Manufacturing File Generation and 3D Visualization  
**Priority**: HIGH - Enables real-world manufacturing and visualization  
**Created**: March 19, 2026

---

## Executive Summary

System 5 implements Layer 5 (Manufacturing Layer) of the MLIR pipeline, transforming validated voxel grids into industry-standard manufacturing files and photorealistic 3D visualizations. This system uses Custom Emitters (no bloated third-party crates) to generate outputs with complete control over format specifications.

**The 5 Export Domains**:
```
1. PCB Manufacturing (Gerber, Drill files, BOM)
2. Silicon Manufacturing (GDSII)
3. 3D Visualization (OBJ, GLB, Blender)
4. Simulation (SPICE, QASM)
5. Documentation (SVG, PDF)
```

**Key Philosophy**: Own the output generation. No generic libraries. Direct format generation using string formatting and byte manipulation.

---

## ⚠️ CRITICAL: Data Sources - NO HARDCODING ALLOWED

**This system MUST use these data sources:**

### Primary Data Sources (ALWAYS use these):
1. **`hwc/data/standard-materials.hwmat`** - Material colors and properties for 3D rendering
   - Copper color: RGB(184, 115, 51) metallic
   - FR4 color: RGB(34, 139, 34) matte green
   - Silkscreen color: RGB(255, 255, 255) flat white
   - **NEVER hardcode these values**

2. **`hwc/data/profiles/*.hwp`** - Manufacturing constraints for export validation
   - Minimum trace widths for Gerber generation
   - Via sizes for drill file generation
   - Layer stackup for GDSII generation

3. **Component render blocks** - 3D asset paths and procedural fallbacks
   - Asset paths from `.hwx` component definitions
   - Fallback procedural geometry when assets unavailable

### How to Use Data Sources:
```rust
// ✅ CORRECT: Load from database
let db = MaterialDatabase::load_standard()?;
let copper = db.get_conductor("copper")?;
let color = copper.render_color;

// ❌ WRONG: Hardcoded values
let color = (184, 115, 51); // DON'T DO THIS!
```

---

## Prerequisites (Already Complete)

✅ **System 1: Parser & AST** (Complete)
✅ **System 2: Voxel Engine** (Complete)
✅ **System 3: Routing Engine** (Complete)
✅ **System 4: Physics Validation** (Complete)
✅ **hwc-export crate** (Stub implementations exist, need enhancement)

**Current Status**: Basic export functionality exists with 6 passing tests. Need to enhance with:
- Proper trace optimization (D01 Draw vs D03 Flash)
- FR4 substrate rendering
- Scene graph architecture
- Material-based coloring
- BOM generation
- Drill file generation

---

## Phase 1: Gerber Export Enhancement ✅ BASIC IMPLEMENTATION

**Purpose**: Enhance Gerber export with trace optimization and proper formatting

**Location**: `hwc/crates/hwc-export/src/gerber.rs`

**Format Reference**: `export-format-research/01-GERBER-X3-FORMAT.md` (Gerber X3 specification)

**Documentation References**:
- Read: `Docs/v0.1.3/EXPORTS-AND-ASSETS.md` (Custom Gerber Emitter section)
- Read: `Docs/v0.1.2/EXPORT-OPTIMIZATION-AND-POLISH.md` (D01 Draw vs D03 Flash)

### Phase 1.1: Trace Optimization (D01 Draw vs D03 Flash) ✅ IMPLEMENTED

**Current Status**: Implemented with trace detection and D01/D03 optimization.

**Task**: Implement intelligent trace detection and use D01 Draw for continuous traces.

- [x] Implement `detect_traces()` function:
  - Input: `Vec<(usize, usize)>` (copper voxels on a layer)
  - Algorithm: Group connected voxels into trace segments using flood-fill
  - Output: `Vec<TraceSegment>` (connected components) and isolated pads

- [x] Implement `GerberEmitter` struct (from documentation):
  - `buffer: String` (pre-allocated 1MB)
  - `current_x_nm: i64`, `current_y_nm: i64` (state tracking)
  - `current_aperture: Option<u32>` (aperture tracking)
  - Methods: `define_rectangular_aperture()`, `select_aperture()`, `draw_to()`, `move_to()`, `flash_pad()`

- [x] Update `export()` function:
  - Detect traces vs pads using flood-fill algorithm
  - Use D01 (Draw) for traces
  - Use D03 (Flash) for pads
  - Optimize aperture switching

- [x] Add unit tests:
  - Test: Horizontal trace → D01 Draw commands ✅
  - Test: Vertical trace → D01 Draw commands ✅
  - Test: Pad → D03 Flash command ✅
  - Test: Mixed board (traces + pads) → correct commands ✅
  - **NOTE**: Current tests use manual voxel placement (bypass method)
  - [ ] **TODO**: Rewrite tests with proper component/route syntax once component system is available

- [x] Upgrade to Gerber X3 format:
  - Changed from `%FSLAX24Y24*%` to `%FSLAX36Y36*%` (3.6 precision)
  - Added Gerber X3 attributes (generation software, creation date, file function)

**Files modified**:
- `hwc/crates/hwc-export/src/gerber.rs` ✅
- `hwc/crates/hwc-export/tests/export_tests.rs` ✅ (with TODO for component-based tests)
- `hwc/crates/hwc-export/Cargo.toml` ✅ (added chrono dependency)

**Test Results**: ✅ 11/11 tests passing (using manual voxel placement as temporary workaround)


### Phase 1.2: Multi-Layer Gerber Support ✅ BASIC IMPLEMENTATION

**Current Status**: Basic multi-layer support exists. Need to enhance with proper layer naming.

**Reference**: See `export-format-research/01-GERBER-X3-FORMAT.md` (Layer Naming Convention section)

- [ ] Implement standard Gerber layer naming (see format spec):
  - Top copper: `.gtl` ✅
  - Bottom copper: `.gbl` ✅
  - Inner layers: `.g1`, `.g2`, `.g3`, etc.
  - Top solder mask: `.gts`
  - Bottom solder mask: `.gbs`
  - Top silkscreen: `.gto`
  - Bottom silkscreen: `.gbo`

- [ ] Add layer type detection:
  - Copper layers (from voxel grid)
  - Solder mask layers (from component definitions)
  - Silkscreen layers (from component labels)

- [ ] Add unit tests:
  - Test: 2-layer board → `.gtl` and `.gbl`
  - Test: 4-layer board → `.gtl`, `.g1`, `.g2`, `.gbl`
  - Test: Board with solder mask → `.gts` and `.gbs`

**Files to modify**:
- `hwc/crates/hwc-export/src/gerber.rs`

### Phase 1.3: Gerber Format Compliance ✅ IMPLEMENTED

**Task**: Ensure Gerber output complies with Gerber X3 standard.

**Reference**: See `export-format-research/01-GERBER-X3-FORMAT.md` (File Structure section)

- [x] Implement proper Gerber header (see format spec):
  - Format specification: `%FSLAX36Y36*%` ✅
  - Units: `%MOMM*%` ✅
  - Layer polarity: `%LPD*%` ✅
  - Gerber X3 attributes (generation software, creation date, file function) ✅

- [x] Implement coordinate precision:
  - Use 3.6 format (3 integer digits, 6 decimal digits) ✅
  - Convert nanometers to microns: `nm / 1000` ✅
  - Format coordinates: `X{:06}Y{:06}` ✅

- [x] Add validation tests:
  - Test: Header format correct ✅
  - Test: Coordinate precision correct ✅
  - Test: File ends with `M02*` ✅

**Files modified**:
- `hwc/crates/hwc-export/src/gerber.rs` ✅

### Phase 1.4: Gerber X3 Component Attributes ❌ NOT IMPLEMENTED

**Task**: Add Gerber X3 component information for assembly.

**Reference**: See `export-format-research/01-GERBER-X3-FORMAT.md` (Gerber X3 Component Information section)

- [ ] Implement component attributes (see format spec):
  - `TO.C`: Component reference designator
  - `TO.CRot`: Component rotation
  - `TO.CMfr`: Manufacturer name
  - `TO.CMPN`: Manufacturer part number
  - `TO.CVal`: Component value
  - `TO.CPkg`: Package/footprint

- [ ] Extract component data from IR:
  - Read placed components from HardwareIR
  - Get component properties from definitions
  - Map to Gerber X3 attributes

- [ ] Add unit tests:
  - Test: Component attributes in output
  - Test: Multiple components with different properties
  - Test: Component rotation correct

**Files to modify**:
- `hwc/crates/hwc-export/src/gerber.rs`


---

## Phase 2: Drill File Generation ❌ NOT IMPLEMENTED

**Purpose**: Generate drill files for via specifications

**Location**: `hwc/crates/hwc-export/src/drill.rs` (new file)

**Documentation References**:
- Read: `Docs/v0.1.3/EXPORTS-AND-ASSETS.md` (Drill file section)
- Read: Excellon drill format specification (external)

### Phase 2.1: Via Detection ❌ NOT IMPLEMENTED

**Task**: Detect vias in the voxel grid (vertical copper connections between layers).

- [ ] Create `hwc/crates/hwc-export/src/drill.rs`

- [ ] Implement `detect_vias()` function:
  - Input: `&VoxelGrid`
  - Algorithm: Find copper voxels that span multiple Z layers
  - For each (x, y) position, check if copper exists on multiple Z layers
  - Output: `Vec<Via>` with position and diameter

- [ ] Implement `Via` struct:
  - `position_nm: (i64, i64)` (X, Y position in nanometers)
  - `diameter_nm: i64` (via diameter)
  - `start_layer: usize` (top layer)
  - `end_layer: usize` (bottom layer)

- [ ] Add unit tests:
  - Test: Single via detected correctly
  - Test: Multiple vias detected
  - Test: Through-hole via (all layers)
  - Test: Blind via (partial layers)

**Files to create**:
- `hwc/crates/hwc-export/src/drill.rs`

**Files to read**:
- `hwc/crates/hwc-engine/src/voxel_grid.rs` (voxel iteration)
- `hwc/crates/hwc-engine/src/routing.rs` (via detection logic)

### Phase 2.2: Excellon Drill File Generation ❌ NOT IMPLEMENTED

**Task**: Generate Excellon drill files for PCB manufacturing.

- [ ] Implement `DrillEmitter` struct:
  - `buffer: String` (pre-allocated)
  - `tool_definitions: HashMap<i64, u32>` (diameter → tool number)

- [ ] Implement `export_drill()` function:
  - Generate Excellon header
  - Define tools (T01, T02, etc.) for each via diameter
  - Output drill coordinates
  - Generate footer

- [ ] Excellon format:
  ```
  M48
  INCH,TZ
  T01C0.012
  T02C0.020
  %
  T01
  X001000Y001000
  X002000Y002000
  T02
  X003000Y003000
  M30
  ```

- [ ] Add unit tests:
  - Test: Single via → correct drill file
  - Test: Multiple vias with same diameter → same tool
  - Test: Multiple vias with different diameters → different tools
  - Test: Drill file format compliance

**Files to modify**:
- `hwc/crates/hwc-export/src/drill.rs`
- `hwc/crates/hwc-export/src/lib.rs` (export drill module)


---

## Phase 3: GDSII Export Enhancement ✅ BASIC IMPLEMENTATION

**Purpose**: Enhance GDSII export for silicon manufacturing

**Location**: `hwc/crates/hwc-export/src/gdsii.rs`

**Documentation References**:
- Read: `Docs/v0.1.3/EXPORTS-AND-ASSETS.md` (Custom GDSII Emitter section)
- Read: GDSII format specification (external)

### Phase 3.1: GDSII Binary Format ❌ NOT IMPLEMENTED

**Current Status**: Basic GDSII export exists. Need to implement proper binary format.

**Task**: Implement GDSII binary record structure.

- [ ] Implement `GdsiiEmitter` struct (from documentation):
  - `buffer: Vec<u8>` (pre-allocated 1MB)
  - Methods: `write_record()`, `string_to_bytes()`, `begin_structure()`, `add_boundary()`, `end_structure()`

- [ ] Implement GDSII record types:
  - `0x0002`: HEADER (version 600)
  - `0x0102`: BGNLIB (library name)
  - `0x0502`: BGNSTR (structure name)
  - `0x0800`: BOUNDARY (polygon)
  - `0x0D02`: LAYER (layer number)
  - `0x1003`: XY (coordinates)
  - `0x1100`: ENDEL (end element)
  - `0x0700`: ENDSTR (end structure)
  - `0x0400`: ENDLIB (end library)

- [ ] Implement coordinate conversion:
  - Convert nanometers to GDSII database units
  - Use fixed-point integer math
  - Handle large coordinates correctly

- [ ] Add unit tests:
  - Test: GDSII header correct
  - Test: Structure creation
  - Test: Boundary with coordinates
  - Test: Multi-layer export
  - Test: File ends correctly

**Files to modify**:
- `hwc/crates/hwc-export/src/gdsii.rs`

**Files to read**:
- `hwc/crates/hwc-engine/src/geometry.rs` (Point3D, BoundingBox)

### Phase 3.2: Layer Mapping for Silicon ❌ NOT IMPLEMENTED

**Task**: Map voxel layers to GDSII layers for silicon manufacturing.

- [ ] Implement layer mapping:
  - Metal 1: GDSII layer 1
  - Metal 2: GDSII layer 2
  - Via 1: GDSII layer 10
  - Via 2: GDSII layer 11
  - Poly: GDSII layer 20
  - Diffusion: GDSII layer 30

- [ ] Add layer type detection:
  - Detect metal layers from voxel grid
  - Detect via layers from vertical connections
  - Detect poly/diffusion from material types

- [ ] Add unit tests:
  - Test: Metal layer → correct GDSII layer
  - Test: Via layer → correct GDSII layer
  - Test: Multi-layer silicon → all layers exported

**Files to modify**:
- `hwc/crates/hwc-export/src/gdsii.rs`


---

## Phase 4: 3D Visualization Enhancement ✅ BASIC IMPLEMENTATION

**Purpose**: Enhance 3D export with FR4 substrate and material-based coloring

**Location**: `hwc/crates/hwc-export/src/obj.rs`, `hwc/crates/hwc-export/src/glb.rs`

**Documentation References**:
- Read: `Docs/v0.1.3/EXPORTS-AND-ASSETS.md` (Scene Graph Architecture)
- Read: `Docs/v0.1.2/EXPORT-OPTIMIZATION-AND-POLISH.md` (FR4 substrate rendering)

### Phase 4.1: FR4 Substrate Rendering ❌ NOT IMPLEMENTED

**Current Status**: Only copper voxels are exported. No substrate rendering.

**Task**: Add FR4 substrate rendering to make 3D views look professional.

- [ ] Implement `render_substrate()` function:
  - Input: `&HardwareSpace`
  - Generate mesh for FR4 board substrate
  - Use board dimensions from space
  - Apply green FR4 material color from database
  - Output: Mesh vertices and faces

- [ ] Implement substrate mesh generation:
  - Create rectangular box mesh for board
  - Position at Z=0 (bottom of board)
  - Height = board thickness
  - Apply smooth normals for realistic lighting

- [ ] Load FR4 color from material database:
  - Read `hwc/data/standard-materials.hwmat`
  - Get FR4 render color: RGB(34, 139, 34) matte green
  - Apply to substrate mesh

- [ ] Add unit tests:
  - Test: Substrate mesh generated correctly
  - Test: Substrate dimensions match board
  - Test: FR4 color applied from database
  - Test: Substrate + copper traces combined

**Files to modify**:
- `hwc/crates/hwc-export/src/obj.rs`
- `hwc/crates/hwc-export/src/glb.rs`

**Files to read**:
- `hwc/crates/hwc-materials/src/database.rs` (material color lookup)
- `hwc/data/standard-materials.hwmat` (FR4 properties)

### Phase 4.2: Material-Based Coloring ❌ NOT IMPLEMENTED

**Task**: Apply realistic colors to all materials based on database.

- [ ] Implement `get_material_color()` function:
  - Input: `MaterialState`
  - Lookup color in material database
  - Output: RGB color tuple

- [ ] Material colors from database:
  - Copper: RGB(184, 115, 51) metallic
  - FR4: RGB(34, 139, 34) matte green
  - Silkscreen: RGB(255, 255, 255) flat white
  - Solder mask: RGB(0, 100, 0) translucent green
  - Gold: RGB(255, 215, 0) metallic

- [ ] Apply materials to OBJ export:
  - Generate `.mtl` material library file
  - Reference materials in `.obj` file
  - Set diffuse color, specular, and metallic properties

- [ ] Apply materials to GLB export:
  - Use glTF PBR materials
  - Set baseColorFactor from database
  - Set metallicFactor (1.0 for copper, 0.0 for FR4)
  - Set roughnessFactor (0.2 for copper, 0.8 for FR4)

- [ ] Add unit tests:
  - Test: Copper has correct color
  - Test: FR4 has correct color
  - Test: Materials loaded from database
  - Test: OBJ with MTL file generated
  - Test: GLB with PBR materials

**Files to modify**:
- `hwc/crates/hwc-export/src/obj.rs`
- `hwc/crates/hwc-export/src/glb.rs`

**Files to read**:
- `hwc/crates/hwc-materials/src/database.rs` (MaterialDatabase API)


### Phase 4.3: Mesh Optimization ❌ NOT IMPLEMENTED

**Task**: Optimize mesh generation for better performance and smaller files.

- [ ] Implement voxel-to-mesh conversion:
  - Use marching cubes algorithm for smooth surfaces
  - Or use greedy meshing for blocky style
  - Merge coplanar faces to reduce polygon count

- [ ] Implement mesh simplification:
  - Remove duplicate vertices
  - Merge adjacent faces
  - Optimize vertex order for GPU rendering

- [ ] Add LOD (Level of Detail) support:
  - Generate multiple mesh resolutions
  - High detail for close-up views
  - Low detail for distant views

- [ ] Add unit tests:
  - Test: Mesh vertex count optimized
  - Test: No duplicate vertices
  - Test: Faces correctly oriented
  - Test: File size reduced

**Files to modify**:
- `hwc/crates/hwc-export/src/obj.rs`
- `hwc/crates/hwc-export/src/glb.rs`

---

## Phase 5: Blender Scene Graph ✅ BASIC IMPLEMENTATION

**Purpose**: Generate intelligent Blender Python scripts for photorealistic rendering

**Location**: `hwc/crates/hwc-export/src/blender.rs`

**Documentation References**:
- Read: `Docs/v0.1.3/EXPORTS-AND-ASSETS.md` (Scene Graph Architecture, Blender Execution)

### Phase 5.1: Scene Graph Structure ❌ NOT IMPLEMENTED

**Current Status**: Basic Blender export exists. Need to implement full scene graph.

**Task**: Implement scene graph data structure for organizing 3D scene.

- [ ] Create `SceneGraph` struct:
  - `substrate: SubstrateMesh`
  - `traces: Vec<TraceMesh>`
  - `components: Vec<ComponentNode>`
  - `camera: CameraConfig`
  - `lighting: LightingConfig`
  - `materials: HashMap<String, MaterialConfig>`

- [ ] Implement `ComponentNode` struct:
  - `name: String`
  - `position: (f64, f64, f64)`
  - `rotation: (f64, f64, f64)`
  - `render_type: RenderType` (Asset or Procedural)
  - `asset_path: Option<String>` (path to .glb file)
  - `fallback_geometry: Option<ProceduralGeometry>`

- [ ] Implement `RenderType` enum:
  - `Asset { path: String }` (import .glb from package cache)
  - `Procedural { shape: Shape, color: RGB }` (generate geometry)

- [ ] Add unit tests:
  - Test: Scene graph creation
  - Test: Component node with asset
  - Test: Component node with procedural fallback
  - Test: Scene graph serialization

**Files to modify**:
- `hwc/crates/hwc-export/src/blender.rs`

**Files to create**:
- `hwc/crates/hwc-export/src/scene_graph.rs` (new module)


### Phase 5.2: Blender Python Script Generation ❌ NOT IMPLEMENTED

**Task**: Generate intelligent Blender Python scripts from scene graph.

- [ ] Implement `generate_blender_script()` function:
  - Input: `&SceneGraph`
  - Generate Python script that:
    1. Clears default scene
    2. Creates FR4 substrate
    3. Generates copper traces (procedural geometry)
    4. Imports component .glb assets
    5. Applies materials
    6. Sets up camera and lighting
    7. Configures render settings
  - Output: Python script string

- [ ] Implement trace generation:
  - Convert voxel copper to Blender mesh
  - Use `bpy.ops.mesh.primitive_cube_add()` for each voxel
  - Or use `bmesh` for optimized mesh generation
  - Apply copper material (metallic, RGB from database)

- [ ] Implement component import:
  - Use `bpy.ops.import_scene.gltf()` for .glb assets
  - Position at exact coordinates from scene graph
  - Apply rotation
  - Fallback to procedural if asset not found

- [ ] Implement camera setup:
  - Position camera for good board view
  - Set focal length and depth of field
  - Add camera tracking to board center

- [ ] Implement lighting setup:
  - Add 3-point lighting (key, fill, rim)
  - Use HDRI environment for realistic reflections
  - Set light intensity and color temperature

- [ ] Add unit tests:
  - Test: Script generates valid Python
  - Test: Script imports correctly in Blender
  - Test: Copper traces rendered
  - Test: Components positioned correctly
  - Test: Materials applied

**Files to modify**:
- `hwc/crates/hwc-export/src/blender.rs`

**Documentation reference**:
- Read: `Docs/v0.1.3/EXPORTS-AND-ASSETS.md` (Example Output Script section)

### Phase 5.3: Asset Path Resolution ❌ NOT IMPLEMENTED

**Task**: Resolve component asset paths from package cache.

- [ ] Implement `resolve_asset_path()` function:
  - Input: Component name (e.g., "ESP32")
  - Check `~/.hw/packages/` for component package
  - Look for `assets/chip.glb` in package directory
  - Return absolute path or None if not found

- [ ] Implement fallback handling:
  - If asset not found, use procedural fallback
  - Generate simple box geometry
  - Apply component color from database
  - Log warning about missing asset

- [ ] Add unit tests:
  - Test: Asset found in package cache
  - Test: Asset not found → fallback used
  - Test: Multiple components with different assets
  - Test: Asset path resolution on different platforms

**Files to modify**:
- `hwc/crates/hwc-export/src/blender.rs`
- `hwc/crates/hwc-export/src/scene_graph.rs`


---

## Phase 6: BOM Generation ❌ NOT IMPLEMENTED

**Purpose**: Generate Bill of Materials for manufacturing and procurement

**Location**: `hwc/crates/hwc-export/src/bom.rs` (new file)

**Documentation References**:
- Read: `Docs/v0.1.3/EXPORTS-AND-ASSETS.md` (BOM generation section)

### Phase 6.1: Component Extraction ❌ NOT IMPLEMENTED

**Task**: Extract component list from Hardware IR.

- [ ] Create `hwc/crates/hwc-export/src/bom.rs`

- [ ] Implement `extract_components()` function:
  - Input: `&HardwareIR`
  - Extract all placed components
  - Group by component type
  - Count quantities
  - Output: `Vec<BomEntry>`

- [ ] Implement `BomEntry` struct:
  - `reference: String` (e.g., "R1", "C1", "U1")
  - `component_type: String` (e.g., "Resistor", "Capacitor", "IC")
  - `value: String` (e.g., "4.7kΩ", "100nF", "ESP32")
  - `quantity: usize`
  - `footprint: String` (e.g., "0805", "SOIC-8")
  - `manufacturer: Option<String>`
  - `part_number: Option<String>`

- [ ] Add unit tests:
  - Test: Single component extracted
  - Test: Multiple components grouped
  - Test: Quantities counted correctly
  - Test: Component properties extracted

**Files to create**:
- `hwc/crates/hwc-export/src/bom.rs`

**Files to read**:
- `hwc/crates/hwc-compiler/src/ir.rs` (HardwareIR, PlacedComponent)

### Phase 6.2: CSV Export ❌ NOT IMPLEMENTED

**Task**: Export BOM to CSV format for spreadsheet applications.

- [ ] Implement `export_bom_csv()` function:
  - Input: `Vec<BomEntry>`
  - Generate CSV with headers:
    - Reference, Type, Value, Quantity, Footprint, Manufacturer, Part Number
  - Sort by reference designator
  - Output: CSV string

- [ ] Implement CSV formatting:
  - Escape commas in values
  - Quote strings with special characters
  - Use UTF-8 encoding

- [ ] Add unit tests:
  - Test: CSV header correct
  - Test: Component rows formatted correctly
  - Test: Special characters escaped
  - Test: File can be opened in Excel/LibreOffice

**Files to modify**:
- `hwc/crates/hwc-export/src/bom.rs`

### Phase 6.3: Interactive HTML BOM ❌ NOT IMPLEMENTED

**Task**: Generate interactive HTML BOM for assembly.

- [ ] Implement `export_bom_html()` function:
  - Generate HTML page with:
    - Component table (sortable, filterable)
    - Board image with component highlighting
    - Search functionality
    - Assembly instructions

- [ ] Implement component highlighting:
  - Click component in table → highlight on board image
  - Hover over component → show details
  - Filter by component type

- [ ] Add unit tests:
  - Test: HTML generated correctly
  - Test: JavaScript functionality works
  - Test: Board image embedded
  - Test: Component highlighting works

**Files to modify**:
- `hwc/crates/hwc-export/src/bom.rs`


---

## Phase 7: Export Pipeline Integration ❌ NOT IMPLEMENTED

**Purpose**: Integrate all exporters into unified pipeline

**Location**: `hwc/crates/hwc-export/src/exporter.rs`

**Documentation References**:
- Read: `Docs/v0.1.3/COMPILER-INTERNALS.md` (Layer 5 Manufacturing Layer)

### Phase 7.1: Unified Exporter Trait ✅ BASIC IMPLEMENTATION

**Current Status**: Basic `Exporter` struct exists. Need to enhance with proper trait system.

**Task**: Implement proper exporter trait for all export formats.

- [ ] Define `Exporter` trait:
  ```rust
  pub trait Exporter {
      fn export(&self, space: &HardwareSpace, ir: &HardwareIR) 
          -> Result<Vec<OutputFile>, ExportError>;
  }
  
  pub struct OutputFile {
      pub name: String,
      pub content: Vec<u8>,
  }
  ```

- [ ] Implement trait for all exporters:
  - `GerberExporter: Exporter`
  - `DrillExporter: Exporter`
  - `GdsiiExporter: Exporter`
  - `ObjExporter: Exporter`
  - `GlbExporter: Exporter`
  - `BlenderExporter: Exporter`
  - `BomExporter: Exporter`

- [ ] Add unit tests:
  - Test: All exporters implement trait
  - Test: Export returns OutputFile
  - Test: Multiple files can be generated

**Files to modify**:
- `hwc/crates/hwc-export/src/exporter.rs`

### Phase 7.2: Target-Based Export Selection ❌ NOT IMPLEMENTED

**Task**: Implement target-based export selection (from CLI flags).

- [ ] Implement `get_exporter()` function:
  ```rust
  pub fn get_exporter(target: &str) -> Box<dyn Exporter> {
      match target {
          "pcb" => Box::new(PcbExporter::new()),
          "gerber" => Box::new(GerberExporter::new()),
          "gdsii" => Box::new(GdsiiExporter::new()),
          "viz" => Box::new(VizExporter::new()),
          "blender" => Box::new(BlenderExporter::new()),
          "sim" => Box::new(SimExporter::new()),
          _ => panic!("Unknown target: {}", target),
      }
  }
  ```

- [ ] Implement composite exporters:
  - `PcbExporter`: Gerber + Drill + BOM
  - `VizExporter`: OBJ + GLB + Blender
  - `SimExporter`: SPICE + behavioral models

- [ ] Add unit tests:
  - Test: PCB target generates all PCB files
  - Test: Viz target generates all 3D files
  - Test: Unknown target handled gracefully

**Files to modify**:
- `hwc/crates/hwc-export/src/exporter.rs`
- `hwc/crates/hwc-export/src/lib.rs`

### Phase 7.3: Export Validation ❌ NOT IMPLEMENTED

**Task**: Validate exports before writing to disk.

- [ ] Implement `validate_export()` function:
  - Check file size limits
  - Validate format compliance
  - Check for empty outputs
  - Verify all required files generated

- [ ] Implement export warnings:
  - Warn if board too large for manufacturer
  - Warn if trace too thin
  - Warn if via too small
  - Warn if missing components

- [ ] Add unit tests:
  - Test: Valid export passes validation
  - Test: Invalid export fails validation
  - Test: Warnings generated correctly

**Files to modify**:
- `hwc/crates/hwc-export/src/exporter.rs`


---

## Phase 8: CLI Integration ❌ NOT IMPLEMENTED

**Purpose**: Add export commands to CLI

**Location**: `hwc/crates/hwc-cli/src/commands/`

**Documentation References**:
- Read: `Docs/v0.1.3/COMPILER-INTERNALS.md` (The Complete Pipeline)

### Phase 8.1: Export Command ❌ NOT IMPLEMENTED

**Task**: Add `hwc export` command to CLI.

- [ ] Create `hwc/crates/hwc-cli/src/commands/export.rs`

- [ ] Implement `export` command:
  - Usage: `hwc export <input.hw> --target <format> --output <dir>`
  - Targets: `pcb`, `gerber`, `gdsii`, `viz`, `blender`, `sim`
  - Output directory: default to `./build/`

- [ ] Implement command options:
  - `--target <format>`: Export format (required)
  - `--output <dir>`: Output directory (optional)
  - `--format <list>`: Comma-separated list of formats
  - `--validate`: Validate exports before writing

- [ ] Add progress reporting:
  - Show export progress
  - Report file sizes
  - Show warnings and errors

- [ ] Add unit tests:
  - Test: Export command parses correctly
  - Test: Target selection works
  - Test: Output directory created
  - Test: Progress reporting works

**Files to create**:
- `hwc/crates/hwc-cli/src/commands/export.rs`

**Files to modify**:
- `hwc/crates/hwc-cli/src/main.rs` (add export command)
- `hwc/crates/hwc-cli/src/commands/mod.rs` (export export module)

### Phase 8.2: Build Command Integration ❌ NOT IMPLEMENTED

**Task**: Integrate export into `hwc build` command.

- [ ] Update `hwc build` command:
  - Add `--target` flag (default: `pcb`)
  - Add `--no-export` flag (skip export generation)
  - Add `--export-dir` flag (custom output directory)

- [ ] Implement automatic export:
  - After successful compilation and validation
  - Generate exports based on target
  - Report export results

- [ ] Add unit tests:
  - Test: Build with export works
  - Test: Build without export works
  - Test: Custom export directory works

**Files to modify**:
- `hwc/crates/hwc-cli/src/commands/build.rs`

---

## Phase 9: Testing & Documentation ❌ NOT IMPLEMENTED

**Purpose**: Comprehensive testing and documentation

### Phase 9.1: Integration Tests ❌ NOT IMPLEMENTED

**Task**: Create end-to-end integration tests.

- [ ] Create `hwc/crates/hwc-export/tests/integration_tests.rs`

- [ ] Test: Complete PCB export pipeline
  - Parse .hw file
  - Compile to IR
  - Render to voxel grid
  - Export Gerber + Drill + BOM
  - Verify all files generated

- [ ] Test: Complete visualization pipeline
  - Parse .hw file
  - Compile to IR
  - Render to voxel grid
  - Export OBJ + GLB + Blender
  - Verify all files generated

- [ ] Test: Multi-target export
  - Export to multiple formats simultaneously
  - Verify no conflicts
  - Verify all files correct

**Files to create**:
- `hwc/crates/hwc-export/tests/integration_tests.rs`

### Phase 9.2: Documentation ❌ NOT IMPLEMENTED

**Task**: Document export system.

- [ ] Add rustdoc comments to all public APIs
- [ ] Add module-level documentation
- [ ] Add usage examples in docs
- [ ] Document export formats
- [ ] Document target selection
- [ ] Generate rustdoc with `cargo doc`

**Files to modify**:
- All files in `hwc/crates/hwc-export/src/`

### Phase 9.3: Example Files ❌ NOT IMPLEMENTED

**Task**: Create example .hw files demonstrating exports.

- [ ] Create `hwc/examples/export_pcb.hw`:
  - Simple PCB design
  - Demonstrates Gerber export
  - Includes drill holes

- [ ] Create `hwc/examples/export_viz.hw`:
  - Board with components
  - Demonstrates 3D visualization
  - Includes component assets

- [ ] Create `hwc/examples/export_silicon.hw`:
  - Simple silicon design
  - Demonstrates GDSII export
  - Multi-layer metal

**Files to create**:
- `hwc/examples/export_pcb.hw`
- `hwc/examples/export_viz.hw`
- `hwc/examples/export_silicon.hw`


---

## Testing Strategy

### Unit Tests

Each export module should have comprehensive unit tests:

- **Gerber Export**: Test trace optimization, layer naming, format compliance
- **Drill Export**: Test via detection, tool definitions, Excellon format
- **GDSII Export**: Test binary format, layer mapping, coordinate conversion
- **3D Export**: Test mesh generation, material coloring, FR4 substrate
- **Blender Export**: Test script generation, asset resolution, scene graph
- **BOM Export**: Test component extraction, CSV format, HTML generation

**Target**: 90%+ code coverage for export modules

### Integration Tests

Test the complete export pipeline:

- Simple boards (2-3 components)
- Complex boards (10+ components)
- Multi-layer boards (4+ layers)
- Boards with vias
- Boards with components

**Target**: 20+ integration tests covering all major scenarios

### Format Compliance Tests

Ensure exports comply with industry standards:

- Gerber RS-274X compliance
- Excellon drill format compliance
- GDSII format compliance
- OBJ/GLB format compliance
- Blender Python API compatibility

**Target**: 100% format compliance

### Visual Regression Tests

For 3D exports, verify visual output:

- Render OBJ in viewer
- Render GLB in viewer
- Execute Blender script
- Compare with reference images

**Target**: Visual output matches reference

---

## Success Criteria

System 5 is complete when:

- [ ] All 5 export domains implemented (PCB, Silicon, 3D, Simulation, Documentation)
- [ ] Gerber export with trace optimization (D01 Draw vs D03 Flash)
- [ ] Drill file generation (Excellon format)
- [ ] GDSII export with proper binary format
- [ ] 3D export with FR4 substrate and material coloring
- [ ] Blender scene graph with asset resolution
- [ ] BOM generation (CSV and HTML)
- [ ] All unit tests passing (90%+ coverage)
- [ ] All integration tests passing (20+ tests)
- [ ] Format compliance verified
- [ ] CLI integration complete (`hwc export`, `hwc build --target`)
- [ ] Documentation complete (rustdoc, examples, README)
- [ ] End-to-end example working (compile .hw → Gerber + 3D + BOM)

---

## Implementation Status Summary

**Core Implementation (Phases 1-7)**: ❌ INCOMPLETE
- Phase 1: Gerber Enhancement ✅ Basic / ❌ Optimization needed
- Phase 2: Drill File Generation ❌ Not implemented
- Phase 3: GDSII Enhancement ✅ Basic / ❌ Binary format needed
- Phase 4: 3D Visualization ✅ Basic / ❌ FR4 and materials needed
- Phase 5: Blender Scene Graph ✅ Basic / ❌ Full scene graph needed
- Phase 6: BOM Generation ❌ Not implemented
- Phase 7: Export Pipeline ✅ Basic / ❌ Trait system needed

**CLI Integration (Phase 8)**: ❌ NOT IMPLEMENTED
- Export command not created
- Build command not integrated

**Testing & Documentation (Phase 9)**: ⚠️ PARTIAL
- Basic tests exist (6 passing)
- Integration tests needed
- Documentation needed
- Example files needed

**Test Results**: ✅ 6/6 basic tests passing (need 20+ integration tests)

---

## Implementation Order

Follow this strict order to minimize rework:

1. ✅ **Phase 1.1**: Gerber trace optimization (foundation for PCB export)
2. ✅ **Phase 1.2-1.3**: Gerber multi-layer and format compliance
3. ❌ **Phase 2.1-2.2**: Drill file generation (completes PCB export)
4. ❌ **Phase 4.1-4.2**: FR4 substrate and material coloring (makes 3D look good)
5. ❌ **Phase 3.1-3.2**: GDSII binary format and layer mapping
6. ❌ **Phase 5.1-5.3**: Blender scene graph and asset resolution
7. ❌ **Phase 6.1-6.3**: BOM generation (CSV and HTML)
8. ❌ **Phase 7.1-7.3**: Export pipeline integration
9. ❌ **Phase 8.1-8.2**: CLI integration
10. ❌ **Phase 9.1-9.3**: Testing and documentation

**Timeline**: Core implementation (Phases 1-7) should be completed before System 6.

---

## Known Challenges and Solutions

### Challenge 1: Trace Detection for D01 Draw

**Problem**: Detecting continuous traces from discrete voxels is non-trivial.

**Solution**: Use connected components algorithm. Group adjacent copper voxels into trace segments. Use Bresenham line detection to identify straight segments.

### Challenge 2: GLB Binary Format

**Problem**: GLB format is complex with JSON + binary chunks.

**Solution**: Use minimal glTF 2.0 subset. Focus on mesh geometry and PBR materials. Reference glTF specification for binary layout.

### Challenge 3: Blender Asset Resolution

**Problem**: Component assets may not be installed in package cache.

**Solution**: Always provide procedural fallback. Generate simple box geometry with correct dimensions and color. Log warning about missing asset.

### Challenge 4: BOM Component Extraction

**Problem**: Component properties may not be fully specified in .hw files.

**Solution**: Load component definitions from standard library. Extract properties from component metadata. Provide defaults for missing properties.

### Challenge 5: Export Validation

**Problem**: Detecting invalid exports before writing to disk.

**Solution**: Implement format-specific validators. Check file size limits, coordinate ranges, and format compliance. Provide actionable error messages.

---

## System 7 Enhancements (Post-System 5)

These are NOT required for System 5 completion but are planned for System 7 (Advanced Features):

- **STEP Export**: Mechanical CAD format for enclosure design
- **SVG Export**: 2D documentation and assembly instructions
- **PDF Export**: Complete documentation package
- **Web Viewer**: Interactive 3D viewer in browser (WebAssembly + Three.js)
- **Animation**: Animated assembly instructions
- **AR/VR**: Augmented reality board inspection
- **Pick-and-Place**: SMT assembly machine files
- **Stencil Generation**: Solder paste stencil files

---

## Conclusion

System 5 is the bridge between digital design and physical reality. By implementing Custom Emitters for all export formats, we maintain complete control over output generation without bloated dependencies. The dual identity system (mathematical brain + visual body) enables both precise manufacturing and photorealistic visualization from the same source.

The key innovations:
- **Custom Emitters**: Zero dependencies, complete control, 3-5× faster
- **Trace Optimization**: D01 Draw for traces, D03 Flash for pads
- **FR4 Substrate**: Professional-looking 3D visualizations
- **Material-Based Coloring**: Realistic colors from database
- **Scene Graph**: Intelligent Blender scripts with asset resolution
- **Target-Based Compilation**: Same source, multiple outputs

This implementation plan provides detailed checkbox tasks for every component, references to documentation, and clear success criteria. Follow the phases in order, write tests as you go, and System 5 will be complete.

Let's make hardware design cinematic.

---

**Document Version**: 1.0  
**Created**: March 19, 2026  
**Status**: Implementation Plan  
**Part of**: Hardware Script v0.1.2 Roadmap
