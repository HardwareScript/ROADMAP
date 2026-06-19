# Roadmap 09 — Integration, Benchmarks & Release

**Read:** Engineering-Specification.md, Architectural-Specification.md

---

## 9.1 Library Integration Verification

- [ ] **Verify `rstar` integration** — dynamic R*-tree for macro floorplanning; O(log N) queries
- [ ] **Verify `geo-index` integration** — static packed R*-tree for per-layer obstacles; 5-10x faster queries
- [ ] **Verify `clarabel` integration** — interior-point solver for macro-scale legalization
- [ ] **Verify `clipper2-rust` integration** — 2D boolean union with Non-Zero Winding Rule
- [ ] **Verify `earcut` integration** — zero-allocation triangulation on clean unioned contours
- [ ] **Verify `rkyv` + `memmap2` + `AlignedVec` integration** — zero-copy binary lockfile with 16-byte alignment
- [ ] **Verify `salsa` integration** — demand-driven incremental query engine
- [ ] **Verify `glam` isolation** — forbidden in core path; only in GLB viewport renderer

## 9.2 Performance Benchmarks

### Small-Scale (2-Layer PCB, 15 Nets, 20mm x 20mm)
- [ ] **Cold build time** — target <10 ms (v0.1.7: 156 ms, 16.9x speedup)
- [ ] **Lockfile hit build time** — target <1 ms (v0.1.7: 13 ms, 16.2x speedup)

### SoC-Scale (TSMC 180nm, 50K Gates, 4K Nets, 100um x 100um)
- [ ] **Cold build time** — target <0.5 s (v0.1.7: 21.64 s, 67.6x speedup)
- [ ] **Lockfile hit build time** — target <0.02 s (v0.1.7: 2.2 s, 110x speedup)

### Memory
- [ ] **100M-transistor design memory** — target <80 MB (down from 1.6 GB)

### Incremental Compilation
- [ ] **Minor edit re-compilation** — target <10 ms

### DRC & Parasitic Extraction
- [ ] **DRC pass time** — target <5 ms on SoC-scale
- [ ] **Parasitic extraction time** — target <50 ms on SoC-scale

## 9.3 Test Suite Validation

- [ ] **Run `test_complex_hybrid_pcb.hw`** — full PCB pipeline validation
- [ ] **Run `test_complex_hybrid_asic.hw`** — full ASIC pipeline validation

## 9.4 Weekend Milestone Checklist

- [ ] **Weekend 1:** Entity Graph, rstar/geo-index integration, O(log N) queries verified
- [ ] **Weekend 2:** Line-search pathfinder, geo-index slab method, boundary-docking, Manhattan/Octilinear paths
- [ ] **Weekend 3:** clarabel IPM, active-set/DAG solver, localized legalization, signal-aware compaction
- [ ] **Weekend 4:** Clipper2 union, earcut triangulation, dependency DAG, rkyv lockfile
- [ ] **Weekend 5:** ComponentStamp OBVH, pre-transformed global bounds, Salsa queries, <10 ms incremental
- [ ] **Weekend 6:** G-Cell SIMD DRC, Sakurai BEM solver, SPICE embedding, <5 ms DRC / <50 ms extraction
- [ ] **Weekend 7:** Full test suite, SoC build <0.5 s, memory <80 MB, v0.1.8 release finalized
