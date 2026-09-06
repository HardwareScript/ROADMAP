> [!CAUTION]
> ## ⚰️ PROJECT PERMANENTLY TERMINATED — SEPTEMBER 2026
>
> **HardwareScript has been permanently shut down and archived.**
>
> After 9 months of development, the project was permanently terminated. All roadmaps, gap analyses, and implementation plans in this directory are **permanently abandoned** — none of the work tracked here will ever be completed.
>
> - These documents describe a **cancelled compiler** and a **cancelled product**.
> - **No further development, issues, or PRs will be accepted.**
>
> For the full technical post-mortem, see [`Docs/End/Final-Status.md`](../Docs/End/Final-Status.md).
>
> *HardwareScript Architecture Team — September 2026*

---

# Hardware Script — ROADMAP

This directory contains the **official implementation roadmaps, gap analyses, and architectural guides** for the [Hardware Script](https://github.com/hardware-script) compiler toolchain and its ecosystem of tools (HSM, HWC, etc.).

Each version directory tracks a specific release cycle's technical goals, completed tasks, discovered gaps, and architectural decisions.

---

## Directory Structure

```
ROADMAP/
├── README.md                     ← This file
├── PTH_IMPLEMENTATION_GUIDE.md   ┐ Plated Through-Hole (PTH) implementation
│                                  │ guide — shared across versions
│
├── v0.1.2/  — Core Compiler Foundation
│   ├── SYSTEM-1-IMPLEMENTATION-PLAN.md    Front-End / Logic & Control Flow
│   ├── SYSTEM-2-IMPLEMENTATION-PLAN.md    Bit-Stream Engine (God-Tier Storage)
│   ├── SYSTEM-3-IMPLEMENTATION-PLAN.md    Physical Compiler / Voxel Engine
│   ├── SYSTEM-4-IMPLEMENTATION-PLAN.md    Collaborative Prefab System
│   ├── SYSTEM-5-IMPLEMENTATION-PLAN.md    Build System & Package Manager
│   ├── SYSTEM-FRAGMENTATION.md            Cross-cutting fragmentation analysis
│   └── TESTING-AND-BUILD-STANDARDS.md     Testing & build conventions
│
├── v0.1.4/  — Constraint-Aware Routing & Logic Synthesis
│   ├── SYSTEM-1-IMPLEMENTATION-PLAN.md    Front-End / Control Flow
│   ├── SYSTEM-2-IMPLEMENTATION-PLAN.md    Bit-Stream Engine
│   ├── SYSTEM-3-IMPLEMENTATION-PLAN.md    Physical Compiler
│   ├── SYSTEM-4-IMPLEMENTATION-PLAN.md    Prefab System
│   ├── SYSTEM-5-IMPLEMENTATION-PLAN.md    Build System
│   ├── SYSTEM-6-IMPLEMENTATION-PLAN.md    CLI / UX / Tooling
│   ├── SYSTEM-FRAGMENTATION.md            Updated fragmentation analysis
│   ├── TESTING-AND-BUILD-STANDARDS.md     Updated standards
│   ├── CONSTRAINT-AWARE-ROUTING.md        Constraint-driven routing design
│   ├── GAP-IMPLEMENTATION-PLAN.md         Gap closure plan
│   ├── GAp.md, GAP2.md, Gap3.md, gap4.md  Identified gaps (sequenced)
│   ├── gap4Soc.md                         SoC-specific gap analysis
│   ├── Gap5-Logic-Compiler-Integration.md Logic comp → physical comp bridge
│   ├── LOGIC-SYNTHESIS-SPECIFICATION.md   Logic synthesis specification
│   ├── MODULE-SYSTEM-FIXES.md             Module system bug fixes
│   └── SLIPPED-GAPS-IMPLEMENTATION.md     Slipped gap remediation
│
├── v0.1.5/  — Routing God-Tier Migration & IDE Integration
│   ├── BGA-STRESS-TEST-GAPS.md            BGA routing stress-test findings
│   ├── COMPILER-GAPS-FOUND.md             Discovered compiler gaps
│   ├── DUAL-COORDINATE-SYSTEM.md          Dual (logical/physical) coordinate system
│   ├── IDE-INTEGRATION-ARCHITECTURE.md    VS Code / LSP integration design
│   ├── implementation-tasks.md            Master task tracker (v0.1.5)
│   ├── implementation-tasks-2.md          Secondary task tracker (v0.1.5)
│   ├── language-update.md                 Language specification updates
│   ├── ROUTING-GOD-TIER-MIGRATION.md      Router → 2.5D analytic migration
│   └── silicon-performance.md             Silicon-aware performance targets
│
├── v0.1.6/  — Assembly Completeness & Bridge Layer
│   ├── ASSEMBLY-COMPLETENESS-ANALYSIS.md  Assembly completeness audit
│   ├── ASSEMBLY-COMPLETENESS-CHECKLIST.md Assembly completeness checklist
│   ├── AUTHORITY-IMPLEMENTATION-PLAN.md   Authority/trust system
│   ├── BRIDGE-IMPLEMENTATION.md           Compiler bridge layer design
│   ├── BRIDGE-IMPLEMENTATION-CHECKLIST.md Bridge checklist
│   ├── connections.md                     Connection semantics
│   ├── CONTEXT-AWARE-PARSING.md           Context-aware parser improvements
│   ├── DEFERRED-WORK.md                   Work deferred from v0.1.6
│   ├── ERROR-HANDLING-IMPLEMENTATION.md   Error handling architecture
│   ├── EXPORT-STRATEGY.md                 Export pipeline strategy
│   ├── EXPORT-TEST-ROUTING-GAP.md         Export-router gap
│   ├── GAP-SUBSTRATE-SPARSE-ARCHITECTURE.md Sparse substrate architecture
│   ├── GAP3-NAMESPACE-RESOLUTION.md       Namespace resolution gaps
│   ├── GAP7-PROGRESSIVE-LVS.md            Progressive LVS (Layout vs. Schematic)
│   ├── GAPS-ANALYSIS.md                   Comprehensive gap analysis
│   ├── GAPS-CLOSED.md                     Gap closure log
│   ├── IMPLEMENTATION-ROADMAP.md          Master roadmap document
│   ├── implementation-tasks.md            Task tracker
│   ├── NATIVE-IR-MIGRATION.md             Native IR migration plan
│   ├── POST-BRIDGE-LIMITATIONS.md         Post-bridge known limitations
│   ├── PROOF-OF-WORK-IMPLEMENTATION.md    Proof-of-work implementation
│   ├── ROUTER-ARCHITECTURAL-GAPS.md       Router architecture gaps
│   ├── SPRINT-2.3-IMPLEMENTATION-STATUS.md Sprint 2.3 status
│   ├── SPRINT-2.4-COMPONENT-LEVEL-CONTINUITY.md Component continuity
│   ├── STAGE1-CRITICAL-GAPS.md            Stage 1 critical gap review
│   ├── STDLIB-ARCHITECTURE-PLAN.md        Standard library architecture
│   ├── THE-COMMIT-GATE-ARCHITECTURE.md    Commit gate design
│   ├── THE-COMMIT-GATE-ARCHITECTURE-info.md Commit gate reference info
│   ├── VISUAL-API.md                      Visual API specification
│   └── VULNERABILITY-1-VOXEL-SMOOTHING.md Voxel smoothing vuln analysis
│
└── v0.1.7/  — 2.5D Reality as Code & HSM Desktop
    ├── BASE-IMPLEMENTATION-ROADMAP.md      Core 2.5D analytic engine roadmap
    ├── COMPILER-BUG-TRACKER.md             Live compiler bug tracker
    ├── HIGH-FIDELITY-OPTICS-PLAN.md        High-fidelity optical inspection
    ├── HSM-IMPLEMENTATION-PLAN.md          HSM (Hardware Script Monitor) plan
    ├── MANIFOLD-EXPORT-RETHINK.md          Manifold export redesign
    ├── PHYSICAL-TRUTH-MIGRATION.md         Physical truth migration
    ├── PTH_IMPLEMENTATION_GUIDE.md         Plated Through-Hole guide
    ├── Routing-&-Manufacturing-Roadmap.md  Routing & manufacturing roadmap
    ├── The-SoC-Engine.md                   System-on-Chip engine design
    ├── Verification-&-Validation-Roadmap.md Verification & validation roadmap
    └── Z-AXIS-ABSTRACTION-IMPLEMENTATION.md Z-axis abstraction impl.
```

---

## Version Summaries

### v0.1.2 — Core Compiler Foundation
The earliest structured roadmap. Defines the 5-system compiler architecture:
1. **Front-End** — Parser, lexer, control flow (MUX trees, index math)
2. **Bit-Stream Engine** — God-tier bit storage with Morton-code spatial indexing
3. **Physical Compiler** — Voxel engine, SDF, obstacle mapping
4. **Collaborative Prefab System** — Component prefabs and composition
5. **Build System / Package Manager** — CLI, caching, dependency resolution

Also introduces the **System Fragmentation** model — a cross-cutting analysis of how the 5 systems interact and where they misalign.

### v0.1.4 — Constraint-Aware Routing & Logic Synthesis
Major expansion with System 6 (CLI/UX/Tooling) and deep gap analysis:
- Constraint-aware routing with Minkowski obstacle inflation
- Logic synthesis specification (Hardware Script → gates)
- Gap closure plans (GAp through Gap5) addressing:
  - Module system fixes
  - SoC-specific architecture
  - Logic compiler ↔ physical compiler integration
- Testing & build standards formalized

### v0.1.5 — Routing God-Tier Migration & IDE Integration
Focus on the **analytic router transition** from voxel-crawling to 2.5D shape-based pathfinding:
- Dual coordinate system (logical indices + physical nanometers)
- BGA stress-test gap analysis
- IDE integration architecture (VS Code / LSP)
- Silicon-aware performance modeling
- Language specification updates

### v0.1.6 — Assembly Completeness & Bridge Layer
The most comprehensive roadmap version. Covers:
- **Assembly completeness** — full PCB/electronic assembly representation
- **Bridge layer** — compiler data interchange between front-end, engine, and exporters
- **Export strategy** — GLB, IDF, ODB++, Gerber, IPC-2581
- **Router architecture gaps** — sparse substrates, progressive LVS, namespace resolution
- **Commit Gate architecture** — CI/CD integrity checks
- **Standard library architecture** — built-in component library design
- **Visual API** — programmatic access to geometry and scene data
- Stage 1 critical gaps and post-bridge limitations documented

### v0.1.7 — 2.5D Reality as Code & HSM Desktop
Current release. Two major thrusts:

**Compiler Side (2.5D Analytic Engine):**
- Planar-locked A* pathfinder with fixed Z-height
- Minkowski obstacle inflation (O(1) collision)
- Z-axis abstraction / Stackup Manager (layer → absolute nm)
- Explicit substrate syntax with cutouts
- Manifold face culling for zero-flicker GPU rendering
- glTF depth bias metadata (polygonOffset, renderOrder)
- 5-stage compiler pipeline (Topological Sort → Obstacle Blitting → 2.5D Routing → Via Stamping → Export)
- Plated Through-Hole (PTH) implementation guide
- SoC engine design
- Verification & validation roadmap
- Compiler bug tracker (live)

**HSM Desktop (Hardware Script Monitor):**
- Tauri v2 native application with Rust Data Factory
- Babylon.js 3D viewport (lazy-loaded, orbit controls, wireframe, orthographic)
- Canvas 2D DXF viewport (layer visibility, measurement tool, unit switching)
- SPICE simulation viewport (uPlot waveform charts)
- Netlist browser with search/filter
- Zero-copy `.hwsb`/`.hsx` parsing pipeline (memmap2 → bytemuck → rkyv)
- PourID → DeviceBinding picking handshake
- GLB augmentation with PourID mesh injection

---

## Key Concepts Referenced Across Versions

| Concept | Description |
|---------|-------------|
| **System Fragmentation** | Systematic analysis of misalignments between the 5 compiler systems |
| **Minkowski Obstacle Inflation** | Compute inflated AABBs = Width/2 + Clearance for O(1) collision |
| **Planar Lock** | Lock A* pathfinder to a specific Z-height from StackupManager |
| **Dual-Paradigm Elevation** | Assembly (Absolute Z) and High-Level (Layer-Based) Z resolution |
| **Commit Gate** | CI/CD architecture enforcing integrity checks before merging |
| **Progressive LVS** | Incremental Layout vs. Schematic verification during routing |
| **Aggregate Engine** | Rust Data Factory + JS Beauty Layer (Babylon/Pixi/Three/uPlot) |
| **PourID Picking Handshake** | Babylon.js raycast → mesh name → Rust resolve → DeviceBinding |
| **Zero-Copy Pipeline** | Memory-mapped `.hwsb` → bytemuck cast → rkyv archive → GLB augmentation |

---

## How to Use These Documents

1. **Start with a version directory** matching the release you're working on.
2. **Read the `IMPLEMENTATION-ROADMAP.md`** or **`BASE-IMPLEMENTATION-ROADMAP.md`** for the high-level plan.
3. **Drill into specific systems** via `SYSTEM-N-IMPLEMENTATION-PLAN.md`.
4. **Check gap analyses** (`GAPS-ANALYSIS.md`, `GAP*.md`, `COMPILER-GAPS-FOUND.md`) for known issues.
5. **Reference the task trackers** (`implementation-tasks.md`, `COMPILER-BUG-TRACKER.md`) for granular status.
6. **For HSM-specific work**, see `v0.1.7/HSM-IMPLEMENTATION-PLAN.md`.

---

## Cross-Cutting Documents

| Document | Location | Scope |
|----------|----------|-------|
| `PTH_IMPLEMENTATION_GUIDE.md` | `ROOT/` | Plated Through-Hole via hot-air solder leveling — technique and use with Hardware Script |
| `TESTING-AND-BUILD-STANDARDS.md` | `v0.1.2/`, `v0.1.4/` | Build conventions, test infrastructure, and quality gates |
| `SYSTEM-FRAGMENTATION.md` | `v0.1.2/`, `v0.1.4/` | Cross-system fragmentation analysis with evolution across versions |

---

## License

This documentation is licensed under the **Creative Commons Attribution 4.0 International License (CC-BY-4.0)**.

You are free to:
- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material for any purpose, even commercially

Under the following terms:
- **Attribution** — You must give appropriate credit, provide a link to the license, and indicate if changes were made

See the [LICENSE](LICENSE) file for the full legal text, or visit [https://creativecommons.org/licenses/by/4.0/](https://creativecommons.org/licenses/by/4.0/)

**Copyright © 2024-2026 Olowookere Olamide and HardwareScript Contributors**

**Note**: This license applies to the roadmap documentation only. The Hardware Script compiler toolchain (`hwc`) and related software are licensed separately under AGPLv3 + Commercial License. See the main repository for software licensing details.