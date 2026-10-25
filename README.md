# Workload Cost Optimizer

Predict query runtimes across compute sizes, then schedule dependent workloads at the lowest cost.

## 1. Data

- Historical query runs, dependency graphs (DAGs)
- **Pre-execution features only:** SQL complexity (joins, CTEs, aggregations, window functions), query plan traits, table stats, estimated data volume, compute size
- **Target (Y):** actual execution time (seconds)
- **Excluded (leakage):** actual bytes scanned, spilling, cache hits from the same run
- Controlled benchmarks across compute sizes, not just historical correlations

## 2. Runtime prediction

- XGBoost regression: runtime as a function of query features × compute size
- **Evaluation:**
  - Grouped train/test splits by model identity, to test on unseen workloads
  - Held-out compute-size changes, to test predicted speedup/slowdown
- **Next:** add intermediate models (bytes scanned, spilling, cache use) only if they improve accuracy

## 3. Schedule optimization

- CP-SAT solver, given predicted runtimes + the dependency graph
- **Decides:** when each workload runs, when to scale compute up or down
- **Objective:** minimize total compute cost
- **Constraints:** dependencies, concurrency limits, completion-time SLAs

## 4. Simulation & validation

- Discrete-event simulator models billing, scaling overhead, resource contention
- Validate the simulator against historical runs first
- Then test optimized schedules in controlled experiments

## Pipeline

```
history + benchmarks → features → XGBoost runtime model
                                          ↓
                    dependency graph → CP-SAT scheduler → simulator → controlled rollout
```
