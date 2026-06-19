# Roadmap 04 — Verification & DRC

**Read:** Core-System-Architecture.md, PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md, Architectural-Specification.md, Engineering-Specification.md, Syntax-&-Definition.md

---

## 4.1 G-Cell-Local Unified Sweep Verification (DRC + Same-Net Topology)

- [ ] **Implement G-Cell partitioning with Boundary-Halo Expansion** — segments within C_max_clearance of boundary registered as Ghost Segments in both adjacent G-cells
- [ ] **Implement Morton ordering** — Z-order curve sort within each G-cell for cache-friendly SIMD access
- [ ] **Implement local flat active interval sweep** — no BST, no pointer chasing; vertical sweep-line with flat array
- [ ] **Implement unified overlap dispatch:**
  - Different-Net (`net_id_A != net_id_B`): evaluate clearance rules via SIMD
  - Same-Net (`net_id_A == net_id_B`): assert overlap lands on VirtualJunctionNode or component port bbox
- [ ] **Implement SIMD 8-wide AVX-512 / 4-wide NEON** bounding box overlap checks
- [ ] **Implement Rayon parallelism** — each G-cell on separate thread; no global memory locks; linear scaling
- [ ] **Verify complexity:** O(N log N + K) per G-cell

## 4.2 Incremental DRC (Planned)

- [ ] **Implement local windowing** — define bounding box around edit for DRC validation
- [ ] **Implement targeted queries** — R*-Tree query for geometries intersecting local bounding box only
- [ ] **Implement local rule validation** — clearance, width, keepout checks on retrieved local geometries only
- [ ] **Verify DRC time reduction** — >90% reduction by skipping unchanged regions

## 4.3 Connectivity Verification

- [ ] **Implement graph reachability analysis** — prove all pins connected, no broken nets or un-waived shorts

## 4.4 Wheeler + Sakurai + Greenhouse BEM Parasitic Extraction

- [ ] **Implement Wheeler's effective permittivity** — accounts for field lines through substrate and air
- [ ] **Implement Sakurai coupling capacitance (C12)** — ground-plane-aware with effective permittivity
- [ ] **Implement Sakurai ground capacitance (C1g)**
- [ ] **Implement series resistance (Rs)** — `Rs = rho * L / (W * t)`
- [ ] **Implement via self-inductance (Lvia)** — analytical cylinder inductance formula
- [ ] **Implement mutual inductance (M12)** — Greenhouse approximation for parallel trace runs
- [ ] **Implement Virtual Junction modeling** — C_junc, L_junc for T-junctions; emit lumped three-port subcircuits in SPICE
- [ ] **Embed R/C values into SPICE netlist export** — K_coupling cards for mutual inductance
- [ ] **Verify extraction time** — <50 ms on SoC-scale designs

## 4.5 Current-Density & Thermal-Rise Verification (AC-Aware)

- [ ] **Implement electromigration limit check (Silicon)** — `A_min = I_peak / J_limit`
- [ ] **Implement IPC-2152 temperature rise check (PCBs)** — with G-cell-local thermal maps
- [ ] **Support separate AC current declarations** — `current_limit: [rms: ..., peak: ...]`; DC-only pins: single value = both
- [ ] **Auto-scale trace width** in thermal hotspots
- [ ] **Flag P21 (EM) and P22 (thermal) violations** — halt build

## 4.6 Manufacturing Verification

- [ ] **Validate via drill aspect ratios** — board thickness to drill diameter
- [ ] **Validate lamination cycle limits**
- [ ] **Validate solder mask expansion rules**
- [ ] **Enforce technology-specific via constraints** — PCB: stacked microvias per IPC Class 3; ASIC: layer-local
