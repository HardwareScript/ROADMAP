# Roadmap 08 — Determinism & Build Pipeline

**Read:** Core-System-Architecture.md, Engineering-Specification.md

---

## 8.1 Deterministic Topological Sorting

- [ ] **Implement topological sort on dependency graph** — elements compiled in exact same order regardless of textual position in source

## 8.2 Fixed-Point Coordinate Transforms (i128)

- [ ] **Ensure all coordinate transforms use pure i128 intermediate arithmetic** — no floating-point in core pathfinder, collision, or coordinate engines
- [ ] **Verify overflow safety** — 200mm PCB coordinates reach 2e11 pm; intermediate products with trig factors reach ~1.414e20 which exceeds i64::MAX
- [ ] **Verify bit-identical results** across AMD x86_64 and Apple Silicon ARM64

## 8.3 Stable Hash Map Iteration

- [ ] **Use `indexmap` or `rustc_hash` crates** with deterministic seed initializers instead of standard `HashMap`

## 8.4 Tie-Breaking Pathfinder Heuristics

- [ ] **Implement deterministic tie-breaking** — when A* encounters identical movement costs, prefer horizontal before vertical; sort by coordinate values

## 8.5 Bit-Identical Serialization

- [ ] **Sort export files by coordinate and Net ID before writing** — GLB meshes, DXF geometries, SPICE netlists
- [ ] **Verify byte-identical output** across multiple runs of identical input
