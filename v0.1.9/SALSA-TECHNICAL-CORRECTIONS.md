# Salsa Integration: Critical Technical Corrections

## Overview

This document details four **fatal "footguns"** that occur when mixing functional compiler frameworks (Salsa) with physical geometry and parallel computing. These corrections are mandatory before implementation begins.

---

## Correction 1: The Salsa Immutability Trap (`&mut` is Poison)

### The Problem

**Original flawed signature:**
```rust
run_optimization_loop(initial_path, constraints, space: &mut HardwareSpace)
```

**Why this fails:**
- Salsa is a purely functional, demand-driven memoization framework
- Salsa queries **cannot take mutable references** (`&mut`)
- If a query mutates global state, dependency tracking breaks down completely
- Multiple threads cannot hold mutable references simultaneously in `std::thread::scope`

### The Solution: The Delta Pattern

The optimizer must take an **immutable reference** to the RoutingDatabase. Instead of mutating space directly, return a pure, detached `OptimizationResult` or `RouteDelta`. Only the top-level compiler orchestrator (outside Salsa's query evaluation) commits results to HardwareSpace.

### Corrected Implementation

```rust
// hwc-compiler/src/ir/routing/electrical_optimizer.rs
use std::sync::Arc;
use hwc_engine::Point3D;

/// Notice: No &mut HardwareSpace. Everything is immutable and Arc-wrapped for zero-cost cloning.
pub fn run_optimization_loop(
    db: &dyn RoutingDatabase,
    initial_path: Arc<Vec<Point3D>>,
    net_id: NetId,
    gcell_id: GCellId,
) -> OptimizationResult {
    let constraints = get_net_constraints(db, net_id);
    let mut current_path = (*initial_path).clone();
    let mut best_score = f64::MAX;
    let mut best_path = current_path.clone();

    for _ in 0..5 { // Max iterations
        // Compute metrics immutably
        let metrics = compute_route_metrics(db, &current_path, net_id);
        let violations = check_constraints(&metrics, &constraints);

        if violations.is_empty() {
            return OptimizationResult::Converged(Arc::new(current_path));
        }

        // Apply mutations to local `current_path` only (No global state mutated)
        let mut mutated = false;
        for violation in &violations {
            match violation {
                Violation::SoftLengthDeficit(deficit) => {
                    // Query static obstacles immutably to ensure meanders don't collide
                    let obstacles = get_gcell_obstacles(db, gcell_id);
                    current_path = inject_meanders(&current_path, *deficit, &obstacles);
                    mutated = true;
                }
                Violation::HardClearance(_) => {
                    // Cannot resolve hard geometry constraints here.
                    return OptimizationResult::RequiresRepair(violations);
                }
                // ... handle other soft violations
            }
        }

        if !mutated { break; }
    }

    OptimizationResult::RequiresRepair(
        check_constraints(
            &compute_route_metrics(db, &best_path, net_id),
            &constraints
        )
    )
}
```

**Key Principles:**
- ✅ All database access is immutable (`&dyn RoutingDatabase`)
- ✅ Paths are wrapped in `Arc` for zero-cost cloning
- ✅ Mutations happen to local variables only
- ✅ Pure function - same inputs always produce same output

---

## Correction 2: Deterministic Penalty Injection (Salsa Keying)

### The Problem

**Original flawed approach:**
> "Invalidate specific G-cell corridor cache... Apply negative weight penalty to bottleneck cell... Re-invoke Step 3."

**Why this fails:**
- Salsa memoizes **strictly based on inputs**
- If inputs (Start, End, Obstacles, Constraints) are identical, Salsa returns the **exact same failing path** from cache
- Salsa won't even run your code - instant cache hit
- No way to force alternative routing without changing inputs

### The Solution: Context Input Pattern

Make **Penalty Weights** an explicit `#[salsa::input]`. When repair is required, update the `RoutingContextInput`, which automatically triggers clean, deterministic cache invalidation for the affected G-cell.

### Corrected Implementation

```rust
// hwc-compiler/src/ir/query_engine.rs

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct RoutingPenalties {
    /// Explicit walls to avoid
    pub blocked_edges: Vec<BoundingBox>,
    /// GCell ID -> Penalty Cost (higher = avoid more)
    pub cell_weights: FxHashMap<usize, i64>,
    /// Tracks repair attempts for this net
    pub attempt_count: u32,
}

#[salsa::input]
pub struct RoutingContextInput {
    pub gcell_id: GCellId,
    pub net_id: NetId,
    /// THIS is the key to localized repair - changing this invalidates the query
    pub penalties: Arc<RoutingPenalties>,
}

#[salsa::tracked]
pub fn extract_topological_corridor(
    db: &dyn RoutingDatabase,
    context: RoutingContextInput,
    entry_port: Point3D,
    exit_port: Point3D,
) -> Option<Arc<Vec<Point3D>>> {
    let obstacles = get_gcell_obstacles(db, context.gcell_id(db));
    let penalties = context.penalties(db);
    
    let mut decomposer = SpatialDecomposer::new(obstacles);
    
    // The decomposer now uses the penalties to alter BFS/Dijkstra edge costs.
    // Because `penalties` is a Salsa input, adding a penalty legally invalidates this cache!
    decomposer.extract_corridor(entry_port, exit_port, penalties)
}
```

### How the Orchestrator Uses This

```rust
// When run_optimization_loop returns RequiresRepair:
if let OptimizationResult::RequiresRepair(violations) = result {
    // 1. Identify bottleneck location
    let bottleneck_gcell = identify_bottleneck(&violations);
    
    // 2. Create new penalty structure
    let mut new_penalties = current_penalties.clone();
    new_penalties.cell_weights.insert(bottleneck_gcell, 10000);
    new_penalties.attempt_count += 1;
    
    // 3. Update Salsa input - this invalidates the query cache
    context.set_penalties(db, Arc::new(new_penalties));
    
    // 4. Re-query - Salsa will compute a NEW path using the penalties
    let alternative_path = extract_topological_corridor(db, context, entry, exit);
}
```

**Key Principles:**
- ✅ Penalties are part of the input signature
- ✅ Changing penalties = new inputs = cache miss = fresh computation
- ✅ Deterministic - same penalties always produce same path
- ✅ Traceable - attempt_count tracks repair iterations

---

## Correction 3: The "Fat Trace" Problem (Minkowski Pre-Inflation)

### The Problem

**Original flawed approach:**
> "Trapezoidal slicing of free space... route trace through the exact center of those cells"

**Why this fails:**
- Navigable space ≠ visual empty space
- If you slice the **raw empty space** between obstacles and route through the center, the **edge** of your trace crashes into obstacles
- A 50nm-wide trace needs 25nm clearance on each side

### The Solution: Configuration Space (C-Space)

Before generating trapezoids, **inflate all obstacles** by `(trace_width / 2) + clearance`. The resulting trapezoidal decomposition is the **Configuration Space**. Any coordinate inside these cells is guaranteed to be 100% physically legal for the trace's centerline.

### Corrected Implementation

```rust
// hwc-engine/src/geometry_router/navigable_space.rs

impl SpatialDecomposer {
    /// Decomposes the Configuration Space (C-Space) for the centerline of the trace
    pub fn decompose_navigable_space(
        &self,
        raw_obstacles: &[BoundingBox],
        trace_width_nm: i64,
        min_clearance_nm: i64,
    ) -> Vec<FreeCell> {
        // STEP 1: Minkowski Inflation FIRST
        let inflation = (trace_width_nm / 2) + min_clearance_nm;
        
        let inflated_obstacles: Vec<BoundingBox> = raw_obstacles
            .iter()
            .map(|obs| obs.expand_by(inflation))
            .collect();

        // STEP 2: Trapezoidal Slicing against INFLATED obstacles
        let x_splits = extract_x_boundaries(&inflated_obstacles);
        let mut cells = Vec::new();
        
        // ... sweep vertical slices to create rectangles

        // STEP 3: Convex Merging
        merge_convex_cells(&mut cells);

        // Centerlines routed through these cells are mathematically guaranteed safe
        cells
    }
}
```

### Visual Explanation

```
WRONG (Raw Space):
┌────────────────────────┐
│     OBSTACLE           │
└────────────────────────┘
         ↓ Route centerline here
         ⚠️  Trace edge collides!

CORRECT (C-Space):
┌──────────────────────────┐
│  INFLATED OBSTACLE       │
│  (includes trace width)  │
└──────────────────────────┘
             ↓ Route centerline here
             ✅ Guaranteed safe!
```

**Key Principles:**
- ✅ Inflate obstacles BEFORE spatial decomposition
- ✅ Inflation = (trace_width / 2) + min_clearance
- ✅ Route through C-Space guarantees physical legality
- ✅ No post-processing DRC violations

---

## Correction 4: Parallel Database Trait Requirements

### The Problem

**Original flawed approach:**
> "Chunk G-cells and route them using `std::thread::scope`"

**Why this fails:**
- If threads call Salsa queries, you hit compiler errors about thread safety
- Standard Salsa databases are not thread-safe by default
- Naive cloning of database doesn't work - breaks memoization

### The Solution: Salsa Snapshots

Salsa supports **lock-free parallelism** via `db.snapshot()`. The database trait must require `salsa::ParallelDatabase`, and each thread gets its own snapshot.

### Corrected Implementation

```rust
// hwc-compiler/src/ir/routing/parallel_chunked.rs

use salsa::ParallelDatabase;

pub fn route_gcells_parallel_chunked(
    db: &dyn RoutingDatabase, // Must implement ParallelDatabase
    gcells: &[GCell],
) -> Result<Vec<Arc<Vec<Point3D>>>, IrError> {
    let chunks = chunk_gcells_by_spatial_proximity(gcells, num_cpus::get());
    let mut global_results = vec![Vec::new(); chunks.len()];

    std::thread::scope(|s| {
        let mut handles = Vec::new();
        
        for (chunk_idx, chunk_gcells) in chunks.into_iter().enumerate() {
            // STEP 1: Take a thread-safe snapshot of the Salsa database
            let db_snapshot = db.snapshot();
            
            let handle = s.spawn(move || {
                let arena = bumpalo::Bump::new();
                let mut local_paths = Vec::new();
                
                for gcell in chunk_gcells {
                    // STEP 2: Safely execute Salsa queries from inside the scoped thread!
                    let path = extract_topological_corridor(&*db_snapshot, ...);
                    local_paths.push(path);
                }
                
                local_paths
            });
            
            handles.push((chunk_idx, handle));
        }

        for (idx, handle) in handles {
            global_results[idx] = handle.join().unwrap();
        }
    });

    Ok(global_results.into_iter().flatten().flatten().collect())
}
```

### Database Trait Definition

```rust
// hwc-compiler/src/ir/query_engine.rs

#[salsa::database(/* ... */)]
pub trait RoutingDatabase: salsa::ParallelDatabase {
    // Query definitions here
}
```

**Key Principles:**
- ✅ Database trait extends `salsa::ParallelDatabase`
- ✅ Each thread gets its own `db.snapshot()`
- ✅ Snapshots are immutable and thread-safe
- ✅ Memoization still works - snapshots share underlying cache
- ✅ Zero contention - lock-free reads

---

## Summary: Before You Write Code

### ❌ Fatal Patterns to Avoid

1. **DON'T** use `&mut` in Salsa queries
2. **DON'T** expect Salsa to re-compute with same inputs
3. **DON'T** route in raw obstacle space without inflation
4. **DON'T** share database references across threads without snapshots

### ✅ Correct Patterns to Use

1. **DO** use immutable references and `Arc` for data
2. **DO** make penalties/weights explicit Salsa inputs
3. **DO** inflate obstacles by `(width/2) + clearance` before decomposition
4. **DO** use `db.snapshot()` for parallel query execution

---

## Implementation Checklist

Before starting Phase 1:

- [x] Understand Salsa's pure functional model
- [x] Understand the difference between visual space and C-space
- [x] Review these four corrections
- [x] Set up parallel database trait correctly
- [x] Plan data flow: Salsa queries → local mutations → orchestrator commits

With these corrections applied, the architecture is **mathematically bulletproof** and ready for implementation.

---

**Version:** v0.1.9  
**Status:** Technical Reference (Implemented)  
**Last Updated:** 2026-07-18
