# Roadmap 07 — Language Syntax & AST

**Read:** Syntax-&-Definition.md, Architectural-Specification.md

---

## 7.1 Syntax Transitions (Leaving to Entering)

- [ ] **Replace `grid: <X> by <Y> by <Z>`** with `resolution: <Measurement>` — continuous database snap-step; engine always uses 1pm internally
- [ ] **Replace `z: <Measurement>` / `z: 1`** with `on layer: <Identifier>` — strict semantic layer abstraction
- [ ] **Replace `spanning z: <Start> to <End>`** with `spanning layer: <Start> to <End>` — prevent via-to-pad floating gap errors
- [ ] **Replace `pour` with `cutouts:`** with semantic separation:
  - `substrate(Material)` for non-conductive dielectric (FR4, Si, Glass)
  - `plane(Material)` for conductive sheet (Copper, Aluminum) with subtractive `cutouts:` (antipads, plane voids)
- [ ] **Replace `points:`** (manual coordinate lists) with `geometry:` blocks — parametric loops, math functions, procedural generators
- [ ] **Replace manual route `path:` arrays** with port-aware logical `route` (`exit:`, `enter:`) — bypass "inside-routing" bug
- [ ] **Replace `current_limit: <Value>`** with `current_limit: [rms: <Value>, peak: <Value>]` — separate AC declarations for EM vs thermal

## 7.2 Parametric Geometry Blocks

- [ ] **Implement `geometry:` block parser** — supports `for i in 0..N` loops, local variables (`let`), trig functions (`sin`, `cos`, `tan`), constants (`PI`, `DEG_TO_RAD`)
- [ ] **Support procedural generator calls** — e.g., `StarGenerator(points: 24, outer: ..., inner: ...)`

## 7.3 AST Simplification

- [ ] **Delete soft keyword hacks** — property keys (`tolerance`, `clearance`, `trace`, `via`) tokenized directly as `IDENTIFIER`; reduce lexer token variants by ~20%
- [ ] **Remove string-reparsing loop** — geometry expressions parsed directly into recursive `Expr` AST nodes; eliminate parsing crashes from token spacing variations
- [ ] **Enforce strict semantic separation** — colons (`:`) inside property blocks for declarative facts; equals (`=`) inside logic blocks for behavioral actions

## 7.4 Resolution & Coordinate System

- [ ] **Implement picometer (pm) internal coordinates** — absolute 64-bit integers; max addressable +/-9,220 km
- [ ] **Implement user-facing `resolution:`** as snapping constraint on parser — engine always executes in picometers
