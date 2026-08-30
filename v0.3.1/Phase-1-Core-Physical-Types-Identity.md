# Phase 1: Core Physical Types, Identity & Base Database

**Target Crate:** `crates/hwc-engine`  
**Pre-requisites:** Phase 0 (`hwc-parser`)  
**Blocks Downstream:** Phase 2 (`hwc-compiler::eval`), Phase 3 (`hwc-synthesis`), Phase 5 (`hwc-physics`), Phase 6 (`hwc-router`), Phase 7 (`hwc-export`)  
**Specification References:**
* [Stable-Structural-Identity.md  2](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md#L59) (Merkle Path Hash-Consing, `EntityId`, `HierarchicalPath`)
* [Stable-Structural-Identity.md  4.5](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md#L570) (`BaseSiliconLock` artifact & checksums)
* [Digital-Logic-Synthesis.md  7.4](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L450) (`StackupManager` Dielectric Context)

---

## 1. Core Responsibilities & Tasks

- [x] **1. Merkle Path Identity Engine (`identity.rs`)**
  - [x] Implement `EntityId(pub u64)` with 100% span-independent hashing (purging `SourceSpan` lines/bytes from identity computation).
  - [x] Implement `HierarchicalPath` and `PathSegment` enum (`Space`, `Module`, `Instance`, `ScopeIndex`, `SemanticKey`, `SubCell`).
  - [x] Implement `EntityId::compute(parent_path, node_kind, semantic_key, declaration_index_in_scope)`.

- [x] **2. Identity Registry (`registry.rs`)**
  - [x] Implement `IdentityRegistry` for bi-directional $O(1)$ lookup between `EntityId` and canonical hierarchical path strings.

- [x] **3. Base Silicon Snapshot Lock (`freeze_lock.rs`)**
  - [x] Implement `BaseSiliconLock` struct (128-bit Blake3 checksum of Layers 1–20 geometries, `frozen_entity_ids` set, `spare_ga_filler_ids`, `locked_layers`).
  - [x] Implement `is_entity_locked(id)` and `is_layer_locked(layer_name)` validation methods.

- [x] **4. Stackup & Dielectric Context (`stackup/`)**
  - [x] Provide `StackupManager::get_stackup_dielectric_context(layer)` returning $(\varepsilon_r, z_{\text{ground}})$.
  - [x] Provide `StackupManager::get_layer_routing_z(layer)` for vertical height calculations.

---

## 2. Verification Gate

- [x] Unit test: Mutating `SourceSpan` offsets (simulating adding whitespace/comments) produces bit-identical `EntityId` values.
- [x] Unit test: `IdentityRegistry` resolves paths to `EntityId` and reverse in $O(1)$.
- [x] Unit test: `BaseSiliconLock` correctly flags modified entities in frozen layers.

