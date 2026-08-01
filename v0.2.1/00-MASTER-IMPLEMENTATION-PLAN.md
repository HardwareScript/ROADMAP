# HardwareScript v0.2.1 - Master Implementation Plan

**Document Type:** Complete Implementation Roadmap  
**Version:** v0.2.1  
**Status:** APPROVED FOR IMPLEMENTATION  
**Date:** August 2026  
**Focus:** Database-Driven Architecture with Context-Aware Parsing and Comptime Anchor Arithmetic  

---

## Executive Summary

This document provides the complete implementation plan for HardwareScript v0.2.1, which merges:

1. **Lexer Token Pruning** - Remove 22+ hardcoded tokens (Section 2 of spec)
2. **Context-Aware Identifier Parsing** - Parse prepositions and origin codes as contextual identifiers (Section 2.3)
3. **Comptime Anchor Arithmetic Engine** - Support mathematical expressions over entity anchors (Section 3-5)
4. **Database-Driven Architecture** - Single source of truth for all spatial data (v0.2.0 carryover)
5. **Technology Strategy Refactoring** - Centralize PCB/ASIC behavior (v0.2.0 carryover)

**Critical Rule**: NO HARDCODING, NO DEFAULTS, NO FALLBACKS, NO SILENT FAILURES. Everything queries the database or fails loudly.

---

## Implementation Philosophy

```
┌──────────────────────────────────────────────────────────────┐
│ THE THREE LAWS OF v0.2.1                                     │
├──────────────────────────────────────────────────────────────┤
│ 1. FAIL LOUDLY: Missing data = compiler error, not warning  │
│ 2. SINGLE SOURCE: Database is truth, not scattered defaults │
│ 3. NO MAGIC: Every decision must be traceable to database   │
└──────────────────────────────────────────────────────────────┘
```

**Before v0.2.1 (BAD)**:
```rust
// Scattered fallbacks and hardcoding
let z = calculate_from_bbox()
    .or_else(|| guess_from_material())
    .unwrap_or(DEFAULT_Z);  // SILENT FAILURE!
```

**After v0.2.1 (GOOD)**:
```rust
// Database query or fail
let z = space.routing_layer_db
    .get_routing_z(layer_name)
    .map_err(|e| CompilerError::MissingLayerData {
        layer: layer_name,
        hint: "Add this layer to your PDK stackup",
    })?;
```

---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    v0.2.1 SYSTEM ARCHITECTURE                   │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────────────────────────────────┐
│  LEXER       │────▶│  Token Stream (Pruned)                   │
│  (Logos)     │     │  - No RightOf, Above, Below, Last, etc.  │
│              │     │  - All prepositions → Identifier         │
└──────────────┘     └──────────────┬───────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│  PARSER (Context-Aware)                                          │
│  - Matches Identifier("right_of") in align: context             │
│  - Matches Identifier("tl") in origin: context                  │
│  - Parses anchor math: (Pad_A.center_x + Pad_B.center_x) / 2   │
└──────────────┬───────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────┐
│  COMPILER (Database-Driven)                                      │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ STAGE 1: Symbol & Dependency DAG Resolution                │ │
│  │ - Build SymbolTable                                        │ │
│  │ - Detect circular dependencies (Error C22)                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ STAGE 2: Comptime Anchor Evaluation                        │ │
│  │ - Evaluate expressions in topological order                │ │
│  │ - Resolve all anchor properties (Pad_A.right, etc.)        │ │
│  │ - Convert to absolute picometers (i64)                     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ DATABASE LAYER (NO HARDCODING)                             │ │
│  │ ┌────────────────────────────────────────────────────────┐ │ │
│  │ │ LayerConnectionDatabase                                │ │ │
│  │ │ - Maps EntityId → RoutingConnectionPoint               │ │ │
│  │ │ - Registers via connections at exact Z elevations      │ │ │
│  │ └────────────────────────────────────────────────────────┘ │ │
│  │ ┌────────────────────────────────────────────────────────┐ │ │
│  │ │ RoutingLayerDatabase                                   │ │ │
│  │ │ - Maps LayerName → exact routing Z (from stackup)      │ │ │
│  │ │ - NO FALLBACKS: Query or fail                          │ │ │
│  │ └────────────────────────────────────────────────────────┘ │ │
│  │ ┌────────────────────────────────────────────────────────┐ │ │
│  │ │ ViaLayerMappingDatabase                                │ │ │
│  │ │ - Maps (from_mat, to_mat) → ViaConnection              │ │ │
│  │ │ - Built from bridge rules + stackup                    │ │ │
│  │ └────────────────────────────────────────────────────────┘ │ │
│  │ ┌────────────────────────────────────────────────────────┐ │ │
│  │ │ TechnologyStrategy (in hwc-types)                      │ │ │
│  │ │ - Set ONCE from PDK profile                            │ │ │
│  │ │ - NO scattered if/else checks                          │ │ │
│  │ └────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────┬───────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────┐
│  ROUTING ENGINE (Query-Based)                                    │
│  - Queries LayerConnectionDatabase for exact connection points   │
│  - Queries RoutingLayerDatabase for exact routing Z              │
│  - Uses TechnologyStrategy for port escape calculations          │
│  - NO hardcoded middle_z calculations                            │
└──────────────┬───────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────┐
│  VALIDATION (Fail-Fast)                                          │
│  - Pre-routing: Verify all connection points exist               │
│  - Post-routing: Verify trace Z matches layer Z                  │
│  - Build FAILS if mismatches detected                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## Implementation Phases

### Phase 0: Preparation & Analysis (2-3 hours)
- [ ] Read and internalize all spec documents
- [ ] Audit current codebase for hardcoded values
- [ ] Document all locations needing database queries
- [ ] Create test files for validation

### Phase 1: Database Infrastructure (6-8 hours)
- [ ] Create LayerConnectionDatabase
- [ ] Create RoutingLayerDatabase
- [ ] Create ViaLayerMappingDatabase
- [ ] Move TechnologyStrategy to hwc-types
- [ ] Integrate databases into HardwareSpace
- [ ] Write comprehensive unit tests

### Phase 2: Lexer Token Pruning (4-6 hours)
- [ ] Remove hardcoded preposition tokens
- [ ] Remove hardcoded origin tokens
- [ ] Remove Last token
- [ ] Update Token enum
- [ ] Update token Display implementation
- [ ] Verify tokenizer still works

### Phase 3: Context-Aware Parser (8-10 hours)
- [ ] Implement contextual identifier matching
- [ ] Update placement parser for new relational syntax
- [ ] Parse comptime anchor arithmetic expressions
- [ ] Parse brace-grouped constraint blocks
- [ ] Update error messages
- [ ] Write parser tests

### Phase 4: Compiler Integration (10-12 hours)
- [ ] Build dependency DAG for anchor expressions
- [ ] Implement topological sort for evaluation order
- [ ] Evaluate anchor arithmetic in comptime
- [ ] Update placement resolution to query databases
- [ ] Update contact placement to register connections
- [ ] Update space builder to set technology strategy
- [ ] Write integration tests

### Phase 5: Router Refactoring (8-10 hours)
- [ ] Update route specification resolution
- [ ] Update boundary resolution to query connection DB
- [ ] Remove hardcoded Z calculations
- [ ] Remove middle_z guessing
- [ ] Update navigable space to use technology strategy
- [ ] Update via operations
- [ ] Write routing tests

### Phase 6: Validation & Error Handling (6-8 hours)
- [ ] Implement pre-routing validation
- [ ] Implement post-routing validation
- [ ] Add fail-fast checks
- [ ] Improve error messages with hints
- [ ] Write validation tests

### Phase 7: Cleanup & Documentation (4-6 hours)
- [ ] Remove all hardcoded defaults
- [ ] Remove all .unwrap_or() fallbacks
- [ ] Remove dead code
- [ ] Update documentation
- [ ] Run full test suite
- [ ] Performance benchmarking

**Total Estimated Time**: 48-63 hours (6-8 full working days)

---

## Critical Success Factors

### 1. Database Queries Replace All Hardcoding

**Search Patterns to Eliminate**:
```rust
// Pattern 1: Hardcoded Z calculations
(bbox.min.z + bbox.max.z) / 2  // FORBIDDEN

// Pattern 2: Fallback chains
.or_else(|| default_value)  // FORBIDDEN
.unwrap_or(DEFAULT)  // FORBIDDEN

// Pattern 3: Scattered technology checks
if min_annular_ring_nm > 0  // REPLACE with strategy.is_pcb()
```

### 2. Single Source of Truth

**Before (BAD)**:
- Via Z calculated from bounding box
- Routing Z guessed from material
- Technology determined ad-hoc

**After (GOOD)**:
- Via Z from ViaLayerMappingDatabase
- Routing Z from RoutingLayerDatabase
- Technology from space.technology_strategy (set once)

### 3. Fail-Fast Validation

**Build Pipeline**:
```
Parse → Build Databases → Validate Databases → Place → Validate Connections
  ↓         ↓                ↓                  ↓         ↓
 AST    Databases        Pre-checks         Register   Post-checks
                                            in DB
```

Every step can fail loudly with clear error messages.

---

## Detailed Implementation Steps

*See following files for detailed breakdown:*
- `01-DATABASE-INFRASTRUCTURE.md` - Phase 1 detailed steps
- `02-LEXER-TOKEN-PRUNING.md` - Phase 2 detailed steps
- `03-CONTEXT-AWARE-PARSER.md` - Phase 3 detailed steps
- `04-COMPILER-INTEGRATION.md` - Phase 4 detailed steps
- `05-ROUTER-REFACTORING.md` - Phase 5 detailed steps
- `06-VALIDATION-ERROR-HANDLING.md` - Phase 6 detailed steps
- `07-CLEANUP-DOCUMENTATION.md` - Phase 7 detailed steps

---

## Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Breaking existing tests | HIGH | Run tests after each phase |
| Performance regression | MEDIUM | Benchmark before/after |
| Missing edge cases in parser | HIGH | Comprehensive parser tests |
| Database query overhead | LOW | Databases built once at compile time |
| Circular dependency bugs | HIGH | Extensive DAG validation tests |
| Migration complexity | HIGH | Detailed step-by-step plans |

---

## Testing Strategy

### Unit Tests (Per Phase)
- [ ] Database operations
- [ ] Parser contextual matching
- [ ] Anchor expression evaluation
- [ ] Technology strategy methods
- [ ] Connection point registration

### Integration Tests
- [ ] PMOS transistor (current failing case)
- [ ] Via stack routing
- [ ] Center alignment with anchor math
- [ ] Radial component arrays
- [ ] Multi-line constraint blocks

### Regression Tests
- [ ] All existing test files must pass
- [ ] No silent failures
- [ ] Error messages are actionable

---

## Success Criteria

✅ **Lexer**:
- [ ] Token enum reduced by 22+ variants
- [ ] No hardcoded preposition tokens
- [ ] 'above', 'below', 'right_of', 'tl', 'last' can be used as identifiers

✅ **Parser**:
- [ ] Context-aware identifier matching works
- [ ] Anchor arithmetic expressions parse correctly
- [ ] Brace-grouped constraints supported

✅ **Compiler**:
- [ ] All databases built from PDK data
- [ ] No hardcoded Z calculations
- [ ] No .unwrap_or() fallbacks
- [ ] Circular dependencies detected

✅ **Router**:
- [ ] Queries connection database for ports
- [ ] Queries routing layer database for Z
- [ ] Uses technology strategy for escape
- [ ] No middle_z guessing

✅ **Validation**:
- [ ] Pre-routing checks pass
- [ ] Post-routing checks pass
- [ ] Build fails loudly on mismatches
- [ ] Error messages provide hints

✅ **Tests**:
- [ ] PMOS transistor test passes
- [ ] All existing tests still pass
- [ ] New anchor math tests pass
- [ ] Performance benchmarks acceptable

---

## Build Verification Command

After implementation, run:

```bash
# 1. Clean build
cargo clean
cargo build --release --all

# 2. Unit tests
cargo test --all

# 3. PMOS test (currently failing)
cargo run --bin hwc -- build tests/ASIC-Minimal/pmos_transistor.hw

# 4. Verify no hardcoding remains
rg "middle_z|unwrap_or\(|or_else\(" --type rust | grep -v "test\|comment"

# 5. Verify databases used everywhere
rg "routing_layer_db|layer_connection_db|via_layer_mapping_db|technology_strategy" --type rust

# 6. Check for TODOs
rg "TODO|FIXME|XXX" --type rust

# 7. Clippy
cargo clippy --all -- -D warnings

# 8. Format
cargo fmt --all -- --check
```

---

## Next Steps

1. **Read this master plan** to understand the full scope
2. **Read Phase 1 plan** (`01-DATABASE-INFRASTRUCTURE.md`) to start implementation
3. **Execute each phase** in order, testing after each
4. **Update this checklist** as you complete each item
5. **Document any deviations** from the plan

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 2026-08-01 | System | Initial draft based on spec v0.2.1 |

---

**Status**: Ready for Implementation  
**Approved By**: System Architect  
**Implementation Priority**: CRITICAL (Blocks all v0.2.x features)
