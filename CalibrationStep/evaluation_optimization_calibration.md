# Evaluation, Optimization, and Calibration in SYMFLUENCE

Notes gathered from `SYMFLUENCE/src/symfluence` (optimization, evaluation, workflow) and `docs/source/calibration.rst`.

This file explains what each piece does and how they work together. It does **not** reimplement Symfluence — it documents the existing path so HydroAgent / CalibHydroAgent can sit on top of it.

---

## 1. Big picture

In Symfluence, these three words are related but not identical:

| Concept | Main role | Typical home in code |
|--------|-----------|----------------------|
| **Optimization** | Search for better parameter sets using an algorithm | `optimization/` → `OptimizationManager`, algorithms, workers |
| **Calibration** | The workflow step that *runs* iterative optimization for a model | Workflow step `calibrate_model` |
| **Evaluation** | Score sim vs obs (metrics, targets, post-run analysis) | `evaluation/` + targets used *inside* the optimization loop; also post-calibration analysis |

**Short version:**

- **Calibration** = the experiment (periods, params, target, budget).
- **Optimization** = the search engine that proposes candidates.
- **Evaluation** = how each candidate (and the final run) is scored and later analyzed.

They share the same metrics (KGE, NSE, …) and observation data, but run at different times in the workflow.

```text
Config (YAML)
  ├─ periods, params, algorithm, metric, target
  │
  ▼
calibrate_model  ──► OptimizationManager.calibrate_model()
  │                     │
  │                     ├─ ParameterManager  (bounds, apply params)
  │                     ├─ Algorithm         (DDS, DE, PSO, …)
  │                     ├─ Worker            (run model once)
  │                     └─ CalibrationTarget / Evaluators
  │                           └─ metrics (KGE, NSE, …)  ← evaluation inside the loop
  │
  ├─ Final evaluation (best params → full/eval run, save results)
  │
  ▼
Post-calibration analysis (optional workflow steps)
  ├─ run_benchmarking
  ├─ run_sensitivity_analysis
  └─ run_decision_analysis
```

---

## 2. Calibration (workflow meaning)

### What it is

Calibration is the Symfluence workflow step that enables iterative parameter estimation:

- CLI / step name: `calibrate_model`
- Wired in: `project/workflow_orchestrator.py` → `managers['optimization'].calibrate_model`
- Declared in: `workflow_steps.py` (“Run model calibration and parameter optimization”)

### Enablement

Calibration only runs if optimization methods include iterative search, typically:

```yaml
OPTIMIZATION_METHODS: [iteration]
```

If `iteration` is not enabled, `calibrate_model` effectively does nothing useful (returns without a real search).

### What you configure (typical keys)

From docs and `core/config/models/optimization.py`:

| Config idea | Example keys |
|-------------|--------------|
| Periods | `CALIBRATION_PERIOD`, `EVALUATION_PERIOD`, `SPINUP_PERIOD`, `CALIBRATION_TIMESTEP` |
| Parameters | `PARAMS_TO_CALIBRATE`, `BASIN_PARAMS_TO_CALIBRATE`, plus model-specific lists |
| Target / variable | `OPTIMIZATION_TARGET` / `CALIBRATION_VARIABLE` (e.g. streamflow, swe, et) |
| Metric | `OPTIMIZATION_METRIC` (KGE, NSE, RMSE, PBIAS, R2, …) |
| Algorithm | `ITERATIVE_OPTIMIZATION_ALGORITHM` / `OPTIMIZATION_ALGORITHM` |
| Budget | `NUMBER_OF_ITERATIONS`, `POPULATION_SIZE`, parallel / MPI settings |

### Typical calibration procedure (scientific sequence)

1. Choose parameters and bounds.
2. Choose objective (metric + target variable).
3. Choose algorithm and budget.
4. Run iterative search on the **calibration** period.
5. Assess performance on an independent **evaluation** period (and optionally validation).

Symfluence automates steps 4–5 once the config is filled; steps 1–3 are still expert/agent decisions.

### Outputs (typical)

- Parallel / iteration results CSV under `optimization/` (e.g. `*_parallel_iteration_results.csv`)
- Best parameter set and objective history
- Reporting plots (convergence, default vs calibrated, etc.) via reporting / calibration orchestrators

---

## 3. Optimization (engine meaning)

### What it is

Optimization is the **numerical search layer**. The facade is:

- `optimization/optimization_manager.py` → `OptimizationManager`
  - `run_optimization_workflow()` — reads `optimization.methods` and runs enabled methods
  - `calibrate_model()` — selects the model-specific optimizer from the registry and runs it

Model-specific optimizers inherit from:

- `optimization/optimizers/base_model_optimizer.py` → `BaseModelOptimizer`

That base class centralizes:

- algorithm execution
- parallel / MPI / local-scratch workers
- results tracking / retries
- optional gradient methods (Adam, L-BFGS)
- **final evaluation** after the search finishes

### Inner loop (one candidate)

```text
Algorithm proposes θ
   → ParameterManager applies θ to model files / config
   → Worker runs the model (isolated directory)
   → Target / evaluator computes metrics vs observations
   → Objective turns metrics into a scalar to minimize (e.g. 1 − KGE)
   → Algorithm updates search state
```

### Algorithms (registered)

Under `optimization/optimizers/algorithms/` (registry includes, among others):

| Family | Examples |
|--------|----------|
| Heuristic / evolutionary | DDS, Async-DDS, DE, PSO, SCE-UA, GA, CMA-ES |
| Multi-objective | NSGA-II, MOEA/D |
| Bayesian / uncertain | DREAM, GLUE, ABC, Bayesian Opt |
| Local / classical | Nelder-Mead, Basin-Hopping, Simulated Annealing |
| Gradient (differentiable models) | Adam, L-BFGS |

Config types live in `core/config/models/optimization.py` (`OptimizationAlgorithmType`, metric types, PSO/DE/DDS/SCE/NSGA2 knobs, etc.).

### Objectives

- Plugin API: `optimization/objectives/` (`BaseObjective`, registries)
- Workers compute metric dicts; objectives convert them to a **minimization** scalar
- Supports single-metric, weighted multi-metric, and multivariate setups (e.g. streamflow + SWE/ET)

### Calibration targets vs evaluation evaluators

- Model-specific **targets** (e.g. SUMMA streamflow/snow/ET) live under:
  - `models/<model>/calibration/targets.py`
  - and/or `optimization/calibration_targets/`
- Shared **evaluators** for variables live under:
  - `evaluation/evaluators/` — streamflow, snow, ET, soil moisture, groundwater, TWS, …

During calibration, targets wrap or call these evaluators so each trial is scored consistently.

### Final evaluation (inside optimization)

After the search, `FinalEvaluationOrchestrator` (`optimization/optimizers/final_evaluation/`) typically:

1. Applies best parameters.
2. Updates file managers for the appropriate period (often full / evaluation period).
3. Re-runs the model once with best params.
4. Saves final results and may update model decisions / reporting inputs.

So “evaluation” happens **inside** optimization at the end, not only as a later workflow step.

---

## 4. Evaluation (scoring and post-analysis)

Evaluation appears in **two layers**:

### A. In-loop evaluation (during calibration)

Used every iteration:

- Compare simulated vs observed for the chosen target.
- Compute metrics (KGE, NSE, RMSE, PBIAS, R², …).
- Feed the objective for the optimizer.

Key packages:

- `evaluation/metrics*.py`, `evaluation/utilities/streamflow_metrics.py`
- `evaluation/evaluators/*`
- Also mirrored helpers under `optimization/workers/utilities/` for workers

Observation config (gauges, SNOTEL, FluxNet, GRACE, SMAP, …) is modeled in:

- `core/config/models/evaluation.py` → `EvaluationConfig` and related obs configs

### B. Post-calibration analysis (workflow steps)

Orchestrated by `evaluation/analysis_manager.py` → `AnalysisManager`, wired as separate steps:

| Workflow step | Purpose |
|---------------|---------|
| `run_benchmarking` | Compare calibrated model to simple benchmarks (mean, seasonality, persistence); write scores under `evaluation/` |
| `run_sensitivity_analysis` | Parameter importance (Morris / Sobol / FAST-style workflows) |
| `run_decision_analysis` | Impact of structural / decision choices |

These are **not** the optimizer. They answer: *How good is the calibrated model relative to baselines? Which parameters matter? Which decisions change skill?*

Other evaluation-related tooling in the same package includes sensitivity helpers, likelihood utilities, structure ensembles, Koopman analysis, etc. — used for analysis/research, not for the basic DDS/DE loop.

---

## 5. How they work together (end-to-end)

### Recommended mental sequence

1. **Domain + data ready**  
   Forcings, attributes, observed streamflow/SWE/ET processed (`process_observed_data`, etc.).

2. **Model runnable**  
   `model_specific_preprocessing` → optional `run_model` smoke test with default params.

3. **Calibration design (config)**  
   Periods, param list, metric, target, algorithm, iterations / population, `OPTIMIZATION_METHODS: [iteration]`.

4. **Optimization / calibration execution**  
   `calibrate_model` → iterative evaluate–propose–update loop → final evaluation with best params.

5. **Independent assessment**  
   Metrics on `EVALUATION_PERIOD` (and optional validation period). Reporting plots (default vs calibrated, hydrographs).

6. **Optional deeper evaluation**  
   Benchmarking, sensitivity, decision analysis.

### Shared glue

- **Same observations** feed calibration scoring and later benchmarking.
- **Same metrics** are used as optimization objectives and as reporting scores (objective form may invert them for minimization).
- **Same model adapters** (parameter files, runners) are used for trial runs and the final run.
- **Config is the contract**: HydroAgent should write/validate YAML; Symfluence executes.

### Periods — important distinction

| Period | Typical use |
|--------|-------------|
| Spinup | Warm states; often not scored |
| Calibration | Window the optimizer fits to |
| Evaluation | Independent check after calibration |
| Validation (optional) | Extra holdout / split-sample test |

Mixing these up is a common source of overconfident “calibrated” results.

---

## 6. Entry points (how you invoke it)

### CLI

```bash
symfluence workflow step calibrate_model --config my_project.yaml
symfluence workflow step run_benchmarking --config my_project.yaml
symfluence workflow run --config my_project.yaml
```

### Python

```python
# Via managers (pattern used across Symfluence / notebooks)
symfluence.managers['optimization'].calibrate_model()
symfluence.managers['analysis'].run_benchmarking()
```

### GUI / TUI

Calibration panels enqueue `calibrate_model` with the same config-driven path.

### HydroAgent / CalibHydroAgent role (note)

- Symfluence owns algorithms, workers, metrics, and file I/O.
- The agent should **design and validate** the calibration config, ensure prerequisites, call `calibrate_model`, then **interpret** results and optionally replan.
- Dangerous / expensive steps (`run_model`, `calibrate_model`) should remain gated.

---

## 7. Key source map

| Concern | Path |
|---------|------|
| Workflow steps | `src/symfluence/workflow_steps.py` |
| Orchestrator wiring | `src/symfluence/project/workflow_orchestrator.py` |
| Optimization facade | `src/symfluence/optimization/optimization_manager.py` |
| Unified optimizer | `src/symfluence/optimization/optimizers/base_model_optimizer.py` |
| Algorithms | `src/symfluence/optimization/optimizers/algorithms/` |
| Objectives | `src/symfluence/optimization/objectives/` |
| Final evaluation | `src/symfluence/optimization/optimizers/final_evaluation/` |
| Variable evaluators | `src/symfluence/evaluation/evaluators/` |
| Metrics | `src/symfluence/evaluation/metrics*.py` |
| Post-cal analysis | `src/symfluence/evaluation/analysis_manager.py` |
| Opt / eval config models | `src/symfluence/core/config/models/optimization.py`, `evaluation.py` |
| Docs | `docs/source/calibration.rst` |
| Per-model stacks | `src/symfluence/models/<model>/calibration/` (see companion note) |

---

## 8. Takeaways for CalibHydroAgent

1. Do **not** rebuild DDS/DE/PSO or metric math — call Symfluence.
2. Treat **calibration** as config + workflow orchestration.
3. Treat **evaluation** as (a) scoring inside the loop and (b) post-run analysis steps.
4. Treat **optimization** as the registered algorithm + worker machinery behind `calibrate_model`.
5. Highest agent value: choose params/metric/algorithm/periods, validate prerequisites, diagnose failures, and revise the plan after results.

See also: `models_calibration.md` for model-specific calibration packages under `symfluence/models`.
