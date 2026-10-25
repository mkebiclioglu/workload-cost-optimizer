# Workload Cost Optimizer

Predict query runtimes across compute sizes, then schedule dependent workloads at the lowest cost.

## 1. Data

- Historical query runs, dependency graphs (DAGs)
- **Pre-execution features only:** SQL complexity (joins, CTEs, aggregations, window functions), query plan traits, table stats, estimated data volume, compute size
- **Cache-state features (known in advance):** warehouse warm/cold, time since same tables were last read
- **Target (Y):** actual execution time (seconds), modeled as log runtime
- **Excluded (leakage):** actual bytes scanned, spilling, cache hits from the same run
- Controlled benchmarks across compute sizes, not just historical correlations

## 2. Runtime prediction

- XGBoost regression: runtime as a function of query features × compute size
- **Monotone constraint on compute size:** bigger compute never predicts slower
- **Predict speedup vs a baseline size**, since that is what the scheduler needs
- **Quantile outputs:** P50 for cost, P90 for SLA checks
- **Evaluation:**
  - Grouped train/test splits by model identity, to test on unseen workloads
  - Held-out compute-size changes, to test predicted speedup/slowdown
- **Next:** add intermediate models (bytes scanned, spilling, cache use) only if they improve accuracy

## 3. Schedule optimization

- CP-SAT solver, given predicted runtimes + the dependency graph
- **Decides:** when each workload runs, when to scale compute up or down
- **Objective:** minimize total compute cost, including billing rules (minimum billed time per resume, auto-suspend, resize latency)
- **Constraints:** dependencies, concurrency limits, completion-time SLAs (against P90 runtimes)
- **Concurrency slowdown factor** so runtimes reflect shared compute, not isolated runs
- **Scale:** discretize time into buckets; use rolling-horizon solves for large DAGs

## 4. Simulation & validation

- Discrete-event simulator models billing, scaling overhead, resource contention
- Validate the simulator against historical runs first
- Feed simulator contention results back into the optimizer's slowdown factors
- **Baselines:** current schedule and static per-job right-sizing; dynamic scaling must beat both
- Then test optimized schedules in controlled experiments

## Pipeline

```
history + benchmarks → features → XGBoost runtime model (P50/P90)
                                          ↓
                    dependency graph → CP-SAT scheduler ⇄ simulator → controlled rollout
```
