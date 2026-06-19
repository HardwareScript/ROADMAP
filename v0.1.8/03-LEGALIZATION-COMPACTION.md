# Roadmap 03 — Legalization & Compaction

**Read:** Core-System-Architecture.md, Architectural-Specification.md, Engineering-Specification.md

---

## 3.1 Localized Legalization Engine (QP Window Solver)

- [ ] **Implement collision detection** — detect trace/trace and trace/pad clearance violations
- [ ] **Implement window bounding** — define small bounding box strictly around affected area when collision detected
- [ ] **Implement convex QP formulation** — convert trace vectors inside window into QP variables; lock surrounding geometry as hard obstacles

### 3.1.1 Macro-Scale Solver: `clarabel` IPM

- [ ] **Integrate `clarabel` crate** — interior-point solver for complex multi-variable constraints
- [ ] **Implement QP objective function:** `min: a(displacement)^2 + b(via_count) + c(length)^2 + d(crosstalk)^2`
- [ ] **Wire for macro-scale floorplanning** — large windows with many variables (block floorplanning, macro-corridor legalization)

### 3.1.2 Local Micro-Adjustment Solver (Active-Set / DAG)

- [ ] **Integrate lightweight active-set solver** (e.g., PIQP) — for small sparse QP problems (N < 20)
- [ ] **Implement 1D DAG graph compaction solver** — longest-path constraint solver for 1D trace compaction
- [ ] **Verify microsecond solve times** with zero heap allocations — avoid O(N^3) matrix factorization overhead

### 3.1.3 Nudge Application

- [ ] **Implement smooth trace shifting** — traces shifted sideways within window boundaries maintaining layout integrity
- [ ] **Prevent global re-routing avalanches** — localized windows prevent small edits from cascading

## 3.2 Constraint-Aware Compaction Engine

- [ ] **Implement impedance-constraint evaluation** — query profile for characteristic impedance targets (50 ohm single-ended, 100 ohm differential)
- [ ] **Implement crosstalk spacing check** — calculate max parallel run length, determine minimum spacing
- [ ] **Implement signal-aware squeeze** — slide traces closer, cap movement at signal integrity limits
