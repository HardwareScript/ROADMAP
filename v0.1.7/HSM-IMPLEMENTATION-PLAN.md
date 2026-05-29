# Hardware Script Monitor (HSM) Implementation Plan

**Target**: v0.1.7
**Status**: Revised — Aggregate Engine Strategy
**Goal**: Build the HSM as a Tauri v2 + Rust + SolidJS application with a **Hybrid Data Factory** architecture. Rust is the high-performance backend that ingests `.hsx` binaries and serves extracted data to professional JavaScript rendering engines (Babylon.js, PixiJS, Three.js, uPlot) for best-in-class visual quality.

---

## Current State — v0.1.7 Reality

In v0.1.7, several pragmatic implementation decisions were made that differ from some of the more detailed "build thin custom viewports" plans described later in this document, while remaining fully aligned with the overall **Aggregate Engine** philosophy.

### Key Evolutions Shipped in v0.1.7

| Area                  | Originally Planned in This Document                          | What Was Actually Shipped                                      | Rationale |
|-----------------------|--------------------------------------------------------------|----------------------------------------------------------------|---------|
| **3D Viewport**       | Build on raw `@babylonjs/core` with manual scene, camera, lights and `createDefaultEnvironment()` | Adopted the high-level `@babylonjs/viewer` (official Babylon Viewer) | Dramatically better automatic framing, environment handling, camera behavior and overall user experience with far less custom code. Real `engine.getFps()` is now exposed. |
| **DXF Viewport**      | Build a custom Three.js orthographic DXF viewer from scratch (`DxfViewport.tsx`) | Adopted the mature `dxf-viewer` library (high-level Three.js DXF viewer) | Significantly stronger support for real, complex industrial-grade DXF files (PCBs, masks, etc.). |
| **Cross-Viewport UX** | Not prioritized in early sprints                             | Unified light/dark background color switching and grid toggle now work across both 3D and DXF | Immediate, practical quality-of-life improvement for users inspecting geometry. |
| **DXF File Loading**  | Exclusively through Rust extractors from the `.hsx` binary   | Added temporary direct "Open DXF File" button (DOM picker) for independent testing | Enabled real-world DXF validation while preserving the long-term "Rust hands over data" vision. |
| **FPS Reporting**     | Primarily relied on legacy Rust render-frame counting        | Real Babylon.js `engine.getFps()` exposed for the 3D tab       | More accurate and meaningful telemetry for the actual renderer in use. |

### What Has Not Changed

The strategic foundation remains exactly as described in the rest of this plan:

- **Rust** is the authoritative **Data Factory**.
- Professional JavaScript rendering engines own the **Beauty Layer**.
- The long-term target is still for Rust to pre-process and deliver normalized, enriched data (not raw files) to the viewers — enabling advanced features such as spatial indexing, material-aware color grading, and efficient culling for very large designs.

These v0.1.7 choices were deliberate, pragmatic trade-offs made to deliver professional visual quality and usable real-file support much faster, while keeping the door fully open to evolve toward the higher-performance custom pipeline outlined later.

---

## 1. The "Aggregate Engine" Architecture

### 1.1 Architectural Philosophy

Instead of building a custom WGPU renderer from scratch (requiring months of shader development for PBR, IBL, MSAA, shadows), we use **Proven Visual Giants**:

1. **The Data Factory (Rust)**: Ingest `.hsx` via `memmap2` zero-copy mapping. Extract embedded `.glb`/`.obj` bytes, inject PourID into GLB mesh names using `gltf` crate, serialize netlists via `rkyv` for zero-copy, parallel GLB mesh generation via `rayon`. Serve raw bytes to JavaScript via Binary IPC.

2. **The Beauty Layer (JavaScript)**: Professional rendering engines loaded inside the Tauri WebView handle all visual presentation. Each viewport uses the best engine for its job:
   - **3D**: Babylon.js (`@babylonjs/core` + WebGPU Engine for PBR, IBL, shadows, MSAA, native GLB)
   - **2D**: PixiJS (OffscreenCanvas + Web Worker for 1M+ trace segments without blocking UI)
   - **DXF**: Three.js orthographic (DXF vector primitives)
   - **SPICE**: uPlot (10M points in <1ms)

3. **The Shell (SolidJS)**: Menu bar (Kobalte Tabs), tab routing, sidebar inspector (Kobalte Collapsible), status bar (120Hz coordinate stream via Tauri Channels), ghost view overlay. Completely decoupled from the rendering engines.

### 1.2 Why This Strategy Wins

| Bottleneck | Custom WGPU Approach | Aggregate Engine Approach |
|-----------|---------------------|--------------------------|
| Visual Quality | Raw unlit geometry. Months to write PBR shaders. | **Professional PBR, IBL, shadows, tone mapping, MSAA** — built into Babylon.js/Three.js |
| GLB Format Support | Custom parser for every format | **Native GLB/GLTF loading** — engines load directly from bytes |
| 2D Segment Performance | Must implement vector rendering in WGPU | **PixiJS handles 1M+ segments** with WebGL batched rendering in a Web Worker |
| Development Speed | Months for professional visuals | **Days** to integrate proven libraries + 1-2 weeks to build IPC bridge |
| Maintenance Burden | Every visual bug must be fixed in custom shaders | Visual bugs are fixed by engine maintainers. We fix data extraction bugs. |
| Memory Footprint | Duplicated in Rust GPU + WebView GPU | Rust holds binary data, JS engines hold GPU buffers — efficient |
| Input Latency | Zero (bypassed browser) | **Excellent** — Tauri WebView uses system GPU drivers directly (Metal/Vulkan/DirectX) |

### 1.3 The Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                          Tauri Window                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    SOLIDJS SHELL                         │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │              VIEWPORT AREA                        │   │    │
│  │  │  ┌──────────────┐ ┌──────────────┐ ┌──────────┐ │   │    │
      │  │  │  │  @babylonjs/viewer │ │   PIXIJS    │ │  uPlot   │ │   │    │
      │  │  │  │  (high-level 3D)   │ │  (WebWorker) │ │ (SPICE)  │ │   │    │
      │  │  │  └──────────────┘ └──────────────┘ └──────────┘ │   │    │
      │  │  │  ┌──────────────┐ ┌──────────────┐ ┌──────────┐ │   │    │
      │  │  │  │  dxf-viewer  │ │  SOLIDJS DOM │ │ Sidebar  │ │   │    │
      │  │  │  │  (DXF)       │ │  (Netlist)   │ │ Inspector│ │   │    │
│  │  │  └──────────────┘ └──────────────┘ └──────────┘ │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  │                                                         │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │              STATUSBAR (SolidJS)                  │   │    │
│  │  │   FPS: 60  |  Traces: 1.2M  |  Violations: 0    │   │    │
│  │  │   (120Hz i64 coordinate stream via Tauri Channel) │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
         ▲                          ▲                 ▲
         │  Binary IPC              │  Tauri Events   │  Rust Watcher
         │  (Augmented .glb bytes   │  (item-selected │  (notify crate
         │   with PourID in names,   │   telemetry,    │   watches .hsx)
         │   2D trace coords,       │   build-failed, │
         │   rkyv-serialized        │   hsx-refreshed,│
         │   netlist data,          │   + 120Hz       │
         │   selection state)       │   Channel)      │
```

---

## 2. Six-Pillar Alignment with Hardware Script Compiler DNA

### 2.1 Spatial Math Alignment: The i64 Nanometer Law

**Compiler DNA (COMPILER-INTERNALS.md):** All spatial logic uses i64 fixed-point math in nanometers.

| HSM Context | Math System | Library | Purpose |
|-------------|------------|---------|---------|
| **Viewport Matrices** | f32 SIMD | Babylon.js / Three.js | Projection, model, and view matrices for 60 FPS rendering |
| **Picking / Hit-Testing** | i64 fixed-point | Native i64 (Rust) | Inject `PourID` into GLTF extras as i64 hex string via `gltf` crate. Extract at click time. No floating point jitter. |
| **SDF / Distance Fields** | i64 fixed-point | Native i64 (Rust) | All spatial queries use the same i64 nanometer system as the compiler. |
| **Coordinate Conversion** | i64 → f32 | Rust engine | Conversion only at the final stage when sending data to JavaScript. |
| **Telemetry Streaming** | i64 fixed-point | Tauri v2 Channel | Push i64 nanometer coordinates at 120Hz to status bar without discrete IPC calls. |

### 2.2 Rendering Alignment: Primitives Over Pixels

**Compiler DNA:** The compiler produces `AnalyticTrace` primitives. These are embedded in `.hsx`.

| HSM Render Mode | Engine | Data Source |
|-----------------|--------|-------------|
| **Standard 3D View** | Babylon.js (WebGPU) | Embedded `.glb`/`.obj` in `.hsx`, augmented with PourID mesh names |
| **Standard 2D View** | PixiJS (Web Worker) | 2D trace coordinates from `.hsx` (converted from `AnalyticTrace`) |
| **DXF View** | Three.js (Orthographic) | DXF primitive data from `.hsx` |
| **SPICE View** | uPlot | Waveform data from `.hsx` |
| **Silicon Inspection Mode** | Babylon.js (Rayon-generated) | Realized mesh from Handshake C (parallel generation via `rayon`) |
| **Ghost View (Violations)** | Three.js / Babylon.js | `integrity_report.glb` ingested by Rust |

### 2.3 Memory Alignment: The .hsx Zero-Copy Handshake

**Compiler DNA (LAZY-REALIZATION-ARCHITECTURE.md):** The compiler produces a `.hsx` file designed for zero-copy access.

```
.hsx file on disk
       │
       ▼
┌──────────────────────┐
│  memmap2             │  Memory-map the .hsx file directly
│  (Rust crate)        │  No file reads. No allocation. Just a pointer.
└──────┬───────────────┘
       │
       ▼
┌──────────────────────────────────┐
│  Rust Engine                     │
│  bytemuck: zero-copy cast        │
│  gltf/gltf-json: inject PourID   │
│    into GLB node "extras" field  │
│  rkyv: zero-copy netlist tree    │
│    serialization (no JSON)       │
│  rayon: parallel GLB mesh gen    │
└──────┬───────────────────────────┘
       │ Binary IPC (raw bytes) + Tauri Channels (120Hz telemetry)
       ▼
┌──────────────────────┐
│  JavaScript          │  Babylon.js loads .glb from byte array
│  Viewports           │  PixiJS loads 2D trace data in Web Worker
│                      │  uPlot loads SPICE waveform data
└──────────────────────┘
```

### 2.4 Logic & Selection Alignment: The Binding Law

**Compiler DNA:** Geometry is inert unless bound via the `device:` property.

**Selection Pipeline:**
1. **User clicks** on a 3D model in Babylon.js.
2. **Babylon.js ray-casting** returns `pickedMesh.name` (e.g., `pour_0xAF31`).
3. **Rust** receives only the mesh name string via IPC. Parses hex `PourID`. Uses `fxhash` to instantly look up `DeviceBinding`.
4. **Binary IPC** sends the `DeviceBinding` to SolidJS: `{ pour_id, device, net, is_bound }`.
5. **SolidJS Sidebar** displays:
   - **Bound**: `"This is M1.gate → net: VDD, device: transistor_0032"`
   - **Unbound**: `"Artist Geometry (Inert)"` — if no binding exists.

### 2.5 High-Performance Library Stack

| Component | Library | Purpose |
|-----------|---------|---------|
| **File IO** | `memmap2` | Zero-copy `.hsx` memory mapping |
| **Zero-Copy Casting** | `bytemuck` | Transmute `.hsx` bytes into typed structs |
| **GLB Augmentation** | `gltf` + `gltf-json` | Inject PourID into GLB node extras for Babylon.js picking |
| **Zero-Copy Serialization** | `rkyv` | Netlist trees with 50k+ nodes — no JSON overhead |
| **Parallel Processing** | `rayon` | Visual Realization pass — 100M traces → GLB meshes |
 | **3D Rendering** | `@babylonjs/viewer` (high-level) | Professional auto-framed 3D with PBR/IBL/shadows. Real FPS exposed. |
 | **2D Rendering** | PixiJS (OffscreenCanvas + Web Worker) | Batched WebGL vector rendering, 1M+ segments, non-blocking UI |
 | **DXF Rendering** | `dxf-viewer` (high-level Three.js) | Mature industrial DXF support + background/grid controls |
| **SPICE/Data Visualization** | uPlot | 10M+ point waveform rendering in <1ms |
| **UI Components** | Kobalte (SolidJS) | Accessible Tabs, Collapsible sidebar sections |
| **Styling** | Tailwind CSS | Obsidian theme (backdrop-blur, border-zinc utilities) |
| **Desktop Framework** | SolidJS + Tauri v2 | Reactive UI shell with native capabilities + Channel streaming |
| **Concurrency** | `crossbeam` / `AtomicBool` | Lock-free file watcher signaling |

### 2.6 The "Commit Gate" Visual Debugger

**Compiler DNA (THE-COMMIT-GATE-ARCHITECTURE.md):** Architecture Mode stops builds on violations.

When a build fails, the compiler produces an `integrity_report.glb`. HSM ingests this file and triggers a **Ghost View** overlay:

| Violation Type | Visual Indicator | Engine |
|---------------|------------------|--------|
| **P44 (Floating)** | Glowing Yellow wireframe | Three.js / Babylon.js |
| **P41 (Disconnected)** | Glowing Red wireframe | Three.js / Babylon.js |
| **Clear** | Normal Obsidian theme | — |

---

## 3. Updated Modular Architecture

```
hsm/
├── src-tauri/               # The Rust Core (The "Data Factory")
│   ├── src/
│   │   ├── main.rs          # Tauri entrypoint & Command registration
│   │   ├── lib.rs           # Shared library exports
│   │   ├── engine/          # .hsx Zero-Copy Ingestion
│   │   │   ├── loader.rs    # memmap2-based .hsx memory mapper
│   │   │   ├── parser.rs    # bytemuck zero-copy cast + data extraction
│   │   │   └── watcher.rs   # Hot-reload file watcher (notify)
│   │   ├── telemetry/       # Real-time data streams
│   │   │   └── stats.rs     # FPS, memory, frame timing, 120Hz Channel streaming
│   │   └── extractors/      # Data extractors for each viewport
│   │       ├── mod.rs           # Shared extraction traits
│   │       ├── glb_extractor.rs # Extract + augment GLB (inject PourID into extras via gltf/gltf-json)
│   │       ├── trace_2d.rs      # Extract 2D trace geometry
│   │       ├── netlist.rs       # Extract NetlistArena via rkyv (zero-copy JSON alternative)
│   │       ├── spice.rs         # Extract SPICE waveforms
│   │       ├── drill.rs         # Extract drill coordinates
│   │       └── diagnostics.rs   # integrity_report.glb ingestion
│   └── Cargo.toml           # Dependencies: memmap2, bytemuck, notify, serde_json, gltf, gltf-json, rkyv, rayon, glam
├── src/                     # SolidJS Frontend (The "Shell")
│   ├── index.tsx            # Entrypoint
│   ├── App.tsx              # Core layout, tab router, menu bar (Kobalte Tabs)
│   ├── App.css              # Obsidian dark theme (Tailwind utility classes)
│   ├── components/
│   │   ├── ThreeDViewport.tsx      # Babylon.js 3D viewport canvas (WebGPU backend via @babylonjs/core)
│   │   ├── TwoDViewport.tsx        # PixiJS 2D vector viewport (OffscreenCanvas + Web Worker)
│   │   ├── DxfViewport.tsx         # Three.js orthographic DXF viewer
│   │   ├── SpiceViewport.tsx       # uPlot waveform viewer
│   │   ├── NetlistView.tsx         # SolidJS hierarchical netlist tree (Kobalte Collapsible)
│   │   ├── DrillView.tsx           # SolidJS canvas drill viewer
│   │   ├── Sidebar.tsx             # Property inspector with DeviceBinding (Kobalte Collapsible)
│   │   ├── StatusBar.tsx           # Telemetry display + 120Hz coordinate stream
│   │   └── GhostView.tsx           # PHYSICAL INTEGRITY ALERT overlay (backdrop-blur-md)
│   ├── workers/
│   │   └── pixi-worker.ts     # Web Worker for OffscreenCanvas PixiJS rendering
│   ├── store/
│   │   ├── telemetry.ts     # Reactive telemetry signals (FPS, memory, coordinates)
│   │   └── selection.ts     # Reactive PourID / DeviceBinding state
│   └── bridge/
│       └── ipc.ts           # Binary IPC helpers + event listeners + Channel listeners
├── public/
│   └── assets/
└── package.json             # SolidJS + @babylonjs/core + pixi.js + three + uplot + @kobalte/core + tailwindcss
```

---

## 4. Implementation Sprints

### Sprint 0: Foundation — Rust Data Factory Setup
- [ ] **Rust side**: Set up `engine/loader.rs` — `memmap2`-based `.hsx` memory mapper
- [ ] **Rust side**: Set up `engine/parser.rs` — `bytemuck` zero-copy cast into structs
- [ ] **Rust side**: Set up `engine/watcher.rs` — `notify`-based file watcher with `AtomicBool` flag
- [ ] **Rust side**: Create `extractors/` module structure with traits
- [ ] **Rust side**: Implement `extractors/glb_extractor.rs` — extract embedded `.glb`/`.obj` bytes AND augment them:
  - Use `gltf` / `gltf-json` crate to inject PourID into each GLB node's "extras" field
  - Name conventions: `pour_0xAF31` where hex is the `PourID`
- [ ] **Rust side**: Add `rkyv` for zero-copy structured data serialization (netlist trees, PourMetadata)
- [ ] **Rust side**: Add `rayon` for parallel GLB mesh generation (Visual Realization pass)
- [ ] **Cargo.toml**: Add `gltf = "1"`, `gltf-json = "1"`, `rkyv = "0.8"`, `rayon = "1"`
- [ ] **Result**: Rust can open `.hsx`, watch it, augment GLB with PourID mesh names, and serve data in <5ms

### Sprint 1: Frontend Foundation — SolidJS Shell + NPM Dependencies
- [ ] **npm**: Install `@babylonjs/core` (NOT base `babylonjs` — we need WebGPU Engine), `pixi.js`, `three`, `uplot`, `@kobalte/core`
- [ ] **SolidJS**: Create `App.tsx` with tab navigation using Kobalte `Tabs` (3D, 2D, DXF, SPICE, Netlist, Drill)
- [ ] **SolidJS**: Create `App.css` with Tailwind Obsidian dark theme:
  - `backdrop-blur-md` for overlays
  - `border-zinc-800/50` for thin CAD lines
  - `bg-obsidian-950/30` for depth layering
- [ ] **SolidJS**: Create `Sidebar.tsx` with Kobalte `Collapsible` for expandable sections + `store/selection.ts`
- [ ] **SolidJS**: Create `StatusBar.tsx` with `store/telemetry.ts` for FPS/memory/coordinate display
- [ ] **SolidJS**: Set up Web Worker infrastructure (`workers/pixi-worker.ts` for OffscreenCanvas)
- [ ] **Result**: A responsive desktop app with tab switching, accessible components, and visual shell

### Sprint 2: 3D Viewport — Babylon.js Integration
- [x] **Adopted high-level `@babylonjs/viewer`** (official Babylon Viewer) instead of raw `@babylonjs/core` + manual scene construction. This delivered dramatically better automatic framing, environment, camera behavior, and overall UX with far less custom code.
- [x] Real Babylon `engine.getFps()` exposed for accurate 3D telemetry (replacing legacy Rust frame counting).
- [x] **SolidJS**: Created `ThreeDViewport.tsx` using the high-level Viewer API.
- [x] **Rust + SolidJS**: Wired `get_hsx_3d_layer` — Rust extracts/augments `.glb` (PourID in mesh names) and Babylon Viewer loads it.
- [ ] **Result (achieved)**: Professional 3D rendering with real FPS reporting. Background color control added later for cross-viewport UX.

### Sprint 3: 2D Trace Viewport — PixiJS Integration with Web Worker
- [ ] **SolidJS**: Create `TwoDViewport.tsx` — PixiJS canvas component
- [ ] **SolidJS**: Create `workers/pixi-worker.ts` — Web Worker that receives OffscreenCanvas and renders PixiJS traces
- [ ] **Rust side**: Implement `extractors/trace_2d.rs` — extract 2D trace coordinates from `.hsx`
- [ ] **Rust + SolidJS**: Wire up `get_hsx_2d_layer` invoke command
- [ ] **SolidJS + Worker**: Post trace data to Web Worker via `postMessage`. Worker updates PixiJS graphics on OffscreenCanvas.
- [ ] **SolidJS**: Implement pan/zoom on 2D viewport (send transform to worker)
- [ ] **Result**: 2D vector traces render at 60 FPS with 1M+ segment support. SolidJS remains responsive even during heavy renders because PixiJS runs in a separate thread.

### Sprint 4: SPICE Viewport — uPlot Integration
- [ ] **SolidJS**: Create `SpiceViewport.tsx` — uPlot canvas component
- [ ] **Rust side**: Implement `extractors/spice.rs` — extract waveform data from `.hsx`
- [ ] **Rust + SolidJS**: Wire up `get_spice_data` invoke command
- [ ] **SolidJS**: Configure uPlot with crosshair tooltips and zoom
- [ ] **Result**: SPICE waveforms render 10M points in <1ms

### Sprint 5: DXF + Drill Viewports
- [x] **Adopted high-level `dxf-viewer` library** (Three.js-based, mature industrial DXF support) instead of building a custom orthographic Three.js viewer from scratch. This was a pragmatic shift for robust real-world PCB/mask DXF files.
- [x] **Temporary direct "Open DXF File" button** added (DOM file picker) for independent DXF validation while Rust binary extraction is still in progress. Long-term target remains: Rust hands normalized DXF layer data.
- [x] Unified background color (light/dark) and grid toggle implemented and synced between DXF and 3D viewports.
- [ ] **SolidJS**: `DxfViewport.tsx` now wraps the `dxf-viewer` component.
- [ ] **Rust side**: Still planned — `extractors/` for DXF layer data from `.hsx`.
- [ ] **DrillView.tsx** remains as lightweight SolidJS canvas (unchanged).
- [ ] **Result (achieved for v0.1.7)**: High-quality DXF viewing + practical cross-viewport UX controls + temporary independent file loading.

### Sprint 6: Netlist Inspector
- [ ] **SolidJS**: Create `NetlistView.tsx` — hierarchical collapsible tree (Kobalte Collapsible)
- [ ] **Rust side**: Implement `extractors/netlist.rs` — extract `NetlistArena` via `rkyv` (zero-copy, no JSON overhead for 50k+ nodes)
- [ ] **Rust + SolidJS**: Wire up `get_netlist_data` invoke command
- [ ] **SolidJS**: Render component hierarchy with `DeviceBinding` truth display
- [ ] **Result**: Netlist viewer shows full logical connectivity with instant load

### Sprint 7: Selection & Binding Display (The Picking Handshake)
- [ ] **Babylon.js**: Implement mesh picking (ray-casting) in 3D viewport — returns `pickedMesh.name` (e.g., `pour_0xAF31`)
- [ ] **Rust side**: Create `resolve_device_binding(picked_mesh_name: String) -> DeviceBinding` command
  - Parse hex PourID from mesh name (e.g., `0xAF31` from `pour_0xAF31`)
  - Use `fxhash`-based PourMetadata lookup → instant DeviceBinding resolution
  - Return binding to JS: only a tiny string crosses IPC, Rust does the heavy work
- [ ] **SolidJS**: Update `Sidebar.tsx` to display DeviceBinding truth or "Artist Geometry (Inert)"
- [ ] **Result**: Clicking any trace shows its logical binding in the sidebar. Only a short string crosses IPC.

### Sprint 8: Hot-Reload & Live Refresh
- [ ] **Rust side**: Hook `engine/watcher.rs` into the extractor pipeline
  - On `.hsx` change → re-map via `memmap2`, re-extract all data (GLB + PourID injection, 2D traces, netlist via rkyv, SPICE)
- [ ] **Rust side**: Emit `hsx-refreshed` Tauri event with data availability flags
- [ ] **SolidJS**: Listen for `hsx-refreshed` event, call invoke commands to reload data
- [ ] **SolidJS**: Update each viewport with fresh data (Babylon.js scene clear + re-add, PixiJS Worker repost, uPlot redraw, etc.)
- [ ] **Result**: Saving `.hw` script instantly refreshes all viewports in <50ms

### Sprint 9: High-Frequency Telemetry Streaming
- [ ] **Rust side**: Implement 120Hz coordinate streaming using Tauri v2 `tauri::channel`:
  ```rust
  // Rust opens a channel that pushes i64 coordinates at 120Hz
  let (tx, mut rx) = tauri::channel::channel();
  state.coordinate_tx.lock().unwrap().replace(tx);
  
  // Spawn a streaming thread
  std::thread::spawn(move || {
      loop {
          let coord = get_cursor_nm_position();
          let _ = tx.send(TelemetryPayload { 
              x: coord.x, y: coord.y, z: coord.z 
          });
          std::thread::sleep(Duration::from_millis(8)); // ~120Hz
      }
  });
  ```
- [ ] **Rust side**: Wire up coordinate stream to Tauri command: `subscribe_coordinate_stream` returns the receiver
- [ ] **SolidJS**: Listen to the Channel stream in `StatusBar.tsx`:
  ```typescript
  const stream = await invoke("subscribe_coordinate_stream");
  for await (const payload of stream) {
    setCursorCoord(`${payload.x} nm, ${payload.y} nm, ${payload.z} nm`);
  }
  ```
- [ ] **Result**: Status bar shows i64 nanometer coordinates at 120Hz. No discrete IPC calls per mouse move.

### Sprint 10: Ghost View — Build Violation Overlay
- [ ] **Rust side**: Implement `extractors/diagnostics.rs` — `integrity_report.glb` ingestion
  - Parse GLB markers for P44 (Floating) and P41 (Disconnected) violations
- [ ] **Rust side**: Emit `build-failed` event with violation count payload
- [ ] **SolidJS**: Create `GhostView.tsx` — overlay component for violation rendering
  - Use `backdrop-blur-md` Tailwind class for glass-morphism effect
  - Trigger "PHYSICAL INTEGRITY ALERT" theme transition (Red/Amber accents)
  - Display violation count in sidebar and status bar (Kobalte Collapsible for detail list)
- [ ] **SolidJS**: Render violation geometry in Three.js overlay over the 3D viewport
- [ ] **Result**: Build violations are visually debuggable with professional GLB rendering

### Sprint 11: Polish, Edge Cases & Performance Tuning
- [ ] **SolidJS**: Handle window resize gracefully (Babylon.js/PixiJS auto-handle via engine)
- [ ] **SolidJS**: Handle tab switching — pause/unpause rendering engines, terminate/recreate Web Worker
- [ ] **Rust side**: Handle `.hsx` file not found / corrupt gracefully
- [ ] **SolidJS**: Loading states and error modals for each viewport
- [ ] **SolidJS**: Dark/light mode if needed
- [ ] **Result**: Production-ready application with graceful error handling

---

## 5. Component Responsibility Matrix

| Component | Responsibility | Performance |
| :--- | :--- | :--- |
| **Rust Engine** | `.hsx` Zero-Copy Ingestion (`memmap2` + `bytemuck`), GLB augmentation (PourID injection via `gltf`), netlist serialization (`rkyv`), parallel mesh gen (`rayon`), file watching, hot-reload signal, 120Hz telemetry streaming (`tauri::channel`) | Native (C++ Speed) |
| **Rust Data Extractors** | Extract per-viewport data from `.hsx` (3D mesh bytes with PourID, 2D traces, netlist via rkyv, SPICE data, drill coordinates) | Native (Rust) |
 | **@babylonjs/viewer 3D View** | High-level Babylon Viewer renders 3D board with full PBR, IBL, shadows, MSAA. Loads GLB from Rust bytes. Real `engine.getFps()` exposed. Background color control. | GPU (60 FPS) |
 | **PixiJS 2D View** | Render 2D traces, pads, silkscreen, board outlines. OffscreenCanvas + Web Worker for non-blocking rendering. | GPU (60 FPS) via Web Worker |
 | **dxf-viewer DXF View** | High-level dxf-viewer (Three.js) for robust industrial DXF. Background color + grid toggle synced with 3D. Temporary direct file open for testing. | GPU (60 FPS) |
| **uPlot SPICE View** | Waveform/time-series rendering with crosshair tooltips. 10M points in <1ms. | Canvas (CPU/GPU) |
| **SolidJS Shell** | Menu bar (Kobalte Tabs), tab routing, sidebar inspector (Kobalte Collapsible), status bar (120Hz Channel stream), ghost view overlay (backdrop-blur) | Reactive Web (Low CPU) |
| **The IPC Bridge** | Binary data transfer + lightweight event system + 120Hz Channel streaming | Binary IPC (<1ms) + Channels |

---

## 6. Technical Patterns

### A. Binary Transfer — Augmented GLB from Rust to Babylon.js

```typescript
// In ThreeDViewport.tsx — Rust sends GLB with PourID in mesh names
const glbBytes: number[] = await invoke("get_hsx_3d_layer");

// Convert to Babylon.js usable format
const blob = new Blob([new Uint8Array(glbBytes)]);
const url = URL.createObjectURL(blob);

// Load into Babylon.js scene (WebGPU engine)
const engine = new WebGPUEngine(canvas);
await engine.initAsync();
const scene = new Scene(engine);
SceneLoader.Append("", url, scene, null, null, (error) => {
  console.error("Failed to load GLB:", error);
});
```

### B. Binary Transfer — 2D Trace Data to PixiJS Web Worker

```typescript
// In TwoDViewport.tsx — main thread
const traceData: TraceSegment[] = await invoke("get_hsx_2d_layer");
worker.postMessage({ type: "load-traces", data: traceData });

// In workers/pixi-worker.ts — Web Worker
self.onmessage = (event) => {
  if (event.data.type === "load-traces") {
    const graphics = new PIXI.Graphics();
    graphics.lineStyle(2, 0x3b82f6, 1);
    for (const segment of event.data.data) {
      graphics.moveTo(segment.x1, segment.y1);
      graphics.lineTo(segment.x2, segment.y2);
    }
    app.stage.addChild(graphics);
  }
};
```

### C. The Picking Handshake — PourID Flow

```typescript
// In ThreeDViewport.tsx — Babylon.js click handler
// The mesh name comes from Rust's gltf augmentation: "pour_0xAF31"
canvas.addEventListener("pointerdown", async (evt) => {
  const pickResult = scene.pick(evt.clientX, evt.clientY);
  if (pickResult?.pickedMesh) {
    const meshName = pickResult.pickedMesh.name; // "pour_0xAF31"
    // Only this short string crosses IPC
    const binding = await invoke("resolve_device_binding", { meshName });
    setSelection(binding);
  }
});
```

```rust
// Rust side — fxhash lookup from hex mesh name
#[tauri::command]
fn resolve_device_binding(mesh_name: String, state: State<AppState>) -> Result<DeviceBinding, String> {
    // Parse "pour_0xAF31" → PourID
    let hex = mesh_name.trim_start_matches("pour_");
    let pour_id = i64::from_str_radix(hex, 16).map_err(|_| "invalid PourID")?;
    // fxhash-based O(1) lookup
    let data = state.hsx_data.lock().map_err(|e| e.to_string())?;
    data.lookup_binding(pour_id).ok_or("Artist Geometry (Inert)".to_string())
}
```

### D. Hot-Reload Event Flow

```typescript
// In App.tsx
onMount(() => {
  const unlisten = listen<RefreshPayload>("hsx-refreshed", async (event) => {
    const { has_3d, has_2d, has_spice } = event.payload;
    
    if (has_3d && activeView() === "3d") {
      const glbBytes = await invoke("get_hsx_3d_layer");
      sceneLoader.reloadScene(glbBytes);
    }
    if (has_2d && activeView() === "2d") {
      const traceData = await invoke("get_hsx_2d_layer");
      worker.postMessage({ type: "reload-traces", data: traceData });
    }
    if (has_spice && activeView() === "spice") {
      const spiceData = await invoke("get_spice_data");
      uPlotLoader.reloadData(spiceData);
    }
  });
  onCleanup(() => unlisten.then(fn => fn()));
});
```

### E. 120Hz Telemetry Streaming via Tauri Channel

```typescript
// In StatusBar.tsx — listen to persistent Channel stream
const stream = await invoke("subscribe_coordinate_stream");
for await (const payload of stream) {
  setCursorCoord(`${payload.x} nm, ${payload.y} nm, ${payload.z} nm`);
}
```

---

## 7. Key Dependencies

### Rust (Cargo.toml)
```toml
[dependencies]
tauri = { version = "2", features = [] }
tauri-plugin-opener = "2"
tauri-plugin-dialog = "2"
memmap2 = "0.9"
bytemuck = { version = "1", features = ["derive"] }
notify = "7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
glam = { version = "0.29" }
log = "0.4"
env_logger = "0.11"
dirs-next = "2"

# GLB augmentation — inject PourID into GLTF node extras
gltf = "1"
gltf-json = "1"

# Zero-copy structured data serialization (replaces JSON for netlists)
rkyv = { version = "0.8", features = ["validation"] }

# Parallel GLB mesh generation (Visual Realization pass)
rayon = "1"
```

### JavaScript (package.json)
```json
{
  "dependencies": {
    "@tauri-apps/api": "^2",
    "@tauri-apps/plugin-dialog": "^2",
    "@tauri-apps/plugin-opener": "^2",
    "@babylonjs/viewer": "^7",
    "dxf-viewer": "^1.0",
    "pixi.js": "^8",
    "three": "^0.170",
    "uplot": "^1.6",
    "@kobalte/core": "^0.13",
    "solid-js": "^1.9",
    "lucide-solid": "^1.16"
  },
  "devDependencies": {
    "@tauri-apps/cli": "^2",
    "@tailwindcss/vite": "^4.3",
    "tailwindcss": "^4.3",
    "typescript": "~5.6",
    "vite": "^6.0",
    "vite-plugin-solid": "^2.11"
  }
}
```

### Vite Config — Web Worker + Tailwind
```typescript
// vite.config.ts
import { defineConfig } from "vite";
import solidPlugin from "vite-plugin-solid";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [solidPlugin(), tailwindcss()],
  // Web Workers for OffscreenCanvas — Vite handles worker bundling automatically
  worker: {
    format: "es",
  },
});
```

---

## 8. Definition of Done

1. All Rust crates and SolidJS modules compile successfully (including `gltf`, `gltf-json`, `rkyv`, `rayon`).
2. The user can open HSM and see a premium, dark obsidian UI with tab navigation (Kobalte Tabs).
 3. **3D Viewport**: Uses high-level `@babylonjs/viewer` for professional rendering. Real `engine.getFps()` is exposed. GLB mesh names contain PourID. Background color control works.
 4. **2D Viewport**: PixiJS renders 2D trace vectors with batched WebGL in a Web Worker at 60 FPS. SolidJS remains responsive during heavy renders.
 5. **SPICE Viewport**: uPlot renders waveform data with crosshair tooltips at 60 FPS.
 6. **DXF Viewport**: Uses high-level `dxf-viewer` library. Supports industrial DXF files. Background color + optional grid toggle synced with 3D.
 7. **Drill Viewport**: SolidJS canvas renders drill hole locations.
8. **Netlist View**: Hierarchical tree (Kobalte Collapsible) shows component connectivity with `DeviceBinding` truth. Loaded via `rkyv` zero-copy deserialization.
9. **Picking works**: Clicking any mesh in 3D viewport sends only the mesh name (e.g., `pour_0xAF31`) to Rust, which uses `fxhash` to resolve `DeviceBinding` and display it in sidebar.
10. **Hot-Reload works**: Saving a `.hw` script instantly refreshes all viewports in <50ms.
11. **120Hz Telemetry**: Status bar shows i64 nanometer coordinates at 120Hz via Tauri Channel — no discrete IPC calls per mouse move.
12. **Ghost View**: Build violations render as glowing wireframes with "PHYSICAL INTEGRITY ALERT" UI (backdrop-blur-md).
13. **No custom shader development needed**: All visual quality comes from professional rendering engines.
14. **UI Accessibility**: Kobalte provides keyboard-navigable tabs, collapsible sections, and proper ARIA attributes.
15. System operates at 60 FPS with minimal CPU/GPU overhead.