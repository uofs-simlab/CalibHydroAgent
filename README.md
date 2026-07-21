# CalibHydroAgent

Calibration-focused architecture for **HydroAgent** on top of **SYMFLUENCE**.

CalibHydroAgent does **not** reimplement numerical calibration (DDS, DE, PSO, metrics, workers). SYMFLUENCE already owns that engine. This project defines the **decision and orchestration layer**: design the calibration experiment, validate the config and workflow, launch Symfluence `calibrate_model`, interpret results, and revise the plan.

---

## Problem

SYMFLUENCE can **execute** a filled calibration config:

```text
YAML → calibrate_model → OptimizationManager → algorithm + worker + metrics → results
```

HydroAgent can already **plan and run** workflow steps (including optionally `calibrate_model`). What is still missing is a specialized agent that owns **scientific design + adaptive closed-loop calibration** — choosing parameters, metric, algorithm, periods, checking prerequisites, diagnosing failures, and replanning.

---

## Design principle

| Layer | Owner | Responsibility |
|-------|--------|----------------|
| Optimizers, objectives, workers, bounds | **SYMFLUENCE** | Given a valid config, search parameters and score runs |
| Workflow planner / executor | **HydroAgent** | NL → plan + `config.yaml` → run Symfluence steps |
| Calibration design + diagnose/replan | **CalibHydroAgent** | Decide *what* and *how* to calibrate; adapt after results |

**One-line rule:** Symfluence runs the calibration; CalibHydroAgent designs, validates, launches, diagnoses, and revises it.

---

## High-level architecture

```text
┌────────────────────────────────────────────────────────────┐
│  User (natural language)                                   │
│  e.g. "Calibrate SUMMA on Bow River for streamflow (KGE)"  │
└─────────────────────────────┬──────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│  CalibHydroAgent  (decision layer)                         │
│                                                            │
│  1. CalibrationDesigner                                    │
│     params · metric · target · algorithm · periods · budget│
│  2. ConfigValidator                                        │
│     registries · obs availability · model constraints      │
│  3. PrerequisiteChecker                                    │
│     obs processed · OPTIMIZATION_METHODS · domain ready    │
│  4. Executor (thin adapter)                                │
│     write config → call Symfluence calibrate_model         │
│  5. ResultInterpreter                                      │
│     KGE / failures / equifinality / worker errors          │
│  6. Replanner                                              │
│     shrink params · change algo · warm-start · periods     │
└─────────────────────────────┬──────────────────────────────┘
                              │ config.yaml + calibrate_model
                              ▼
┌────────────────────────────────────────────────────────────┐
│  SYMFLUENCE  (execution layer)                             │
│                                                            │
│  WorkflowOrchestrator.calibrate_model                      │
│       → OptimizationManager                                │
│       → model optimizer (SUMMA, FUSE, GR, …)               │
│       → ParameterManager · Algorithm · Worker · Target     │
│       → final evaluation + results CSV / plots             │
└────────────────────────────────────────────────────────────┘
```

### Closed loop

```text
User intent
   → Design calibration config & workflow steps
   → Validate against Symfluence constraints
   → Ensure prerequisites (plan repair if needed)
   → Execute calibrate_model
   → Interpret metrics / logs
   → Replan (optional next round)
```

---

## What we add vs what we reuse

### Do **not** rebuild (Symfluence)

- Algorithms (DDS, DE, PSO, SCE-UA, NSGA-II, Adam, …)
- Objective math (e.g. minimize `1 − KGE`)
- Model workers and parameter-bound registries
- The `calibrate_model` workflow step itself

### Do build (CalibHydroAgent)

1. **CalibrationDesigner** — From basin type, model, and available observations, propose:
   - `PARAMS_TO_CALIBRATE` (or model-specific lists)
   - `OPTIMIZATION_TARGET` / `OPTIMIZATION_METRIC`
   - `ITERATIVE_OPTIMIZATION_ALGORITHM`, iterations, population
   - `CALIBRATION_PERIOD` / `EVALUATION_PERIOD` / spinup
   - `OPTIMIZATION_METHODS: [iteration]`

2. **ConfigValidator** — Check proposed settings against Symfluence parameter registries and model-specific constraints before expensive runs.

3. **PrerequisiteChecker** — Ensure the HydroAgent plan includes needed steps (e.g. `process_observed_data`) and that domain / obs / periods are coherent before `calibrate_model`.

4. **Thin Symfluence adapter** — Write YAML, invoke CLI/API (`calibrate_model`), read results. No fork of the optimizer.

5. **ResultInterpreter + Replanner** — After a run: summarize skill, detect failure modes, propose one revision (fewer params, different algorithm, warm-start, longer budget, period change).

---

## How this sits with HydroAgent

HydroAgent remains the general **workflow assistant** (NL → plan → execute Symfluence steps). CalibHydroAgent is the **calibration specialist** that:

- fills and repairs the calibration block of the plan / `config.yaml`
- gates dangerous steps (`run_model`, `calibrate_model`) as HydroAgent already does
- turns post-run artifacts into the next scientific decision

Symfluence’s in-tree agent can answer config questions; CalibHydroAgent is about **closed-loop experiment control**, not chat-only help.

```text
HydroAgent (general planner/executor)
        │
        ├── uses Symfluence adapter for all steps
        │
        └── Calibration capability (this repo’s focus)
                └── designs / validates / diagnoses calibration
                        └── Symfluence OptimizationManager
```

---

## MVP (first vertical slice)

A minimal path that already matches the architecture:

1. NL → fill calibration block in `config.yaml` (params from a **model allowlist**, metric, algorithm, periods).
2. Ensure plan includes prerequisites (e.g. `process_observed_data`) before `calibrate_model`.
3. Run Symfluence `calibrate_model`.
4. Summarize results and propose **one** revision.

Start with one proven case (e.g. Bow River / SUMMA streamflow + KGE), then generalize to other models.

---

## Repo layout

```text
CalibHydroAgent/
├── README.md                          ← this architecture overview
├── CalibrationStep/
│   ├── evaluation_optimization_calibration.md
│   └── models_calibration.md
└── calib_hydro_agent/                 ← planned module skeleton
    ├── README.md
    ├── types.py
    ├── designer.py                    # CalibrationDesigner
    ├── validator.py                   # ConfigValidator
    ├── prerequisites.py               # PrerequisiteChecker
    ├── adapter.py                     # SymfluenceAdapter
    ├── interpreter.py                 # ResultInterpreter
    ├── replanner.py                   # Replanner
    └── pipeline.py                    # CalibrationPipeline (wires the loop)
```

Stubs raise `NotImplementedError` until wired to HydroAgent / Symfluence. See [calib_hydro_agent/README.md](calib_hydro_agent/README.md).

---

## Relation to Symfluence concepts

| Term in Symfluence | Role in this architecture |
|--------------------|---------------------------|
| **Calibration** | Experiment the agent designs (`calibrate_model` + periods/params/target) |
| **Optimization** | Engine Symfluence runs (algorithms + workers) — called, not rewritten |
| **Evaluation** | In-loop scoring + post-run analysis the agent interprets for replanning |

Details: [CalibrationStep/evaluation_optimization_calibration.md](CalibrationStep/evaluation_optimization_calibration.md)  
Model packages: [CalibrationStep/models_calibration.md](CalibrationStep/models_calibration.md)

---

## Non-goals

- Replacing Symfluence’s `OptimizationManager` or algorithm registry
- Becoming a new hydrological model
- Auto-running unbounded calibrations without user allow / budget gates

---

## Success criteria

CalibHydroAgent is working when a user can request calibration in natural language and the system:

1. Produces a **valid** Symfluence calibration config for the chosen model.
2. Ensures the **workflow** is ready (obs, periods, `iteration` enabled).
3. Executes via Symfluence and returns an interpretable summary.
4. Suggests a **concrete next action** when results are poor or the run fails.
