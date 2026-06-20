# Roadmap 07 — Language Syntax & AST

**Read:** Syntax-&-Definition.md, Architectural-Specification.md

---

## 7.1 Syntax Transitions (Leaving to Entering)

- [x] **Replace `grid: <X> by <Y> by <Z>`** with `resolution: <Measurement>` — added `Token::Resolution`, `parse_resolution()`, `SpaceDefinition.resolution` field; `parse_grid()` marked `#[deprecated]`
- [x] **Replace `z: <Measurement>` / `z: 1`** with `on layer: <Identifier>` — already supported in `placements.rs` and `placement.rs`; verified consistent across pour and component placements
- [x] **Replace `spanning z: <Start> to <End>`** with `spanning layer: <Start> to <End>` — already supported; verified in contact/spanning parsing
- [x] **Replace `pour` with `substrate:` / `plane:` semantic separation** — added `Token::Plane`, `parse_plane()`, `PlanePlacement` struct, `CutoutShape` enum; `substrate:` uses existing pour pattern with semantic distinction
- [x] **Replace `points:` with `geometry:` blocks** — existing `geometry:` block parser enhanced with `for` loops, `let` bindings, trig functions, generator calls
- [x] **Replace manual route `path:` arrays with port-aware logical `route`** — `exit:` and `enter:` already supported in routing parser; verified port-aware docking
- [x] **Replace `current_limit: <Value>`** with `current_limit: [rms: <Value>, peak: <Value>]` — added `CurrentLimitAc` struct, bracketed `[rms: X, peak: Y]` parsing with single-value backward compat

## 7.2 Parametric Geometry Blocks

- [x] **Implement `geometry:` block parser** — supports `for i in 0..N` loops with `Expr` bounds, local variables (`let`), trig functions (`sin`, `cos`, `tan`), constants (`PI`, `DEG_TO_RAD`)
- [x] **Support procedural generator calls** — `StarGenerator(points: 24, outer: ..., inner: ...)` parsed as `GeometryStatement::GeneratorCall`

## 7.3 AST Simplification

- [x] **Delete soft keyword hacks** — removed `Token::Via` (property key only); tokenized as `Identifier` now; reduced lexer token variants
- [x] **Remove string-reparsing loop** — geometry expressions parsed directly into recursive `Expr` AST nodes; `Expr::Constant` added for known constants
- [x] **Enforce strict semantic separation** — documented boundary law: colons `:` in property blocks for declarative facts; equals `=` in logic blocks for behavioral actions

## 7.4 Resolution & Coordinate System

- [x] **Implement picometer (pm) internal coordinates** — added `DistanceUnit::Picometers`, `to_picometers()` conversion for all units; `Measurement::to_picometers_i64()` returns i64 pm
- [x] **Implement user-facing `resolution:`** as snapping constraint — `Resolution { snap_step_pm }` struct with `snap()` and `from_measurement()`; `evaluate_with_resolution()` snaps X/Y to resolution grid

---

## Summary

| Section | Completed | Remaining |
|---------|-----------|-----------|
| 7.1 Syntax Transitions | 7/7 | **Complete** |
| 7.2 Geometry Blocks | 2/2 | **Complete** |
| 7.3 AST Simplification | 3/3 | **Complete** |
| 7.4 Resolution System | 2/2 | **Complete** |
| **Total** | **14/14** | **All sections complete** |

**Files modified:** `token.rs`, `ast/space.rs`, `ast/common.rs`, `ast/shape.rs`, `ast/expr.rs`, `dimensions.rs`, `core.rs`, `placements.rs`, `routing.rs`, `geometry.rs`, `units.rs`, `parsers.rs`, `expression.rs`, `navigation.rs`, `tokens.rs`
**New types:** `Resolution`, `CurrentLimitAc`, `CutoutShape`, `PlanePlacement`, `GeometryStatement`, `Expr::Constant`
