# Model Calibration Packages in SYMFLUENCE

Notes on calibration support under `SYMFLUENCE/src/symfluence/models/*/calibration/`.

Most hydrological models follow the same pattern:

```text
models/<model>/calibration/
  ├── optimizer.py           # *ModelOptimizer ← BaseModelOptimizer
  ├── parameter_manager.py   # bounds, normalize, write params into model files
  ├── worker.py              # one trial: apply params → run → metrics
  ├── targets.py             # (optional) streamflow / snow / ET targets
  └── __init__.py
```

Shared engine (algorithms, parallel execution, final evaluation) lives in `symfluence/optimization/`. Model packages only customize **how parameters are written** and **how outputs are read**.

Routing-only packages (`mizuroute`, `troute`) are thinner (often optimizer-focused).

Models **without** a `calibration/` package in this tree: `wmfire` (and non-model helper dirs such as `base`, `adapters`, `config`, …).

---

## Land-surface / process-based hydrology

### SUMMA
**Path:** `models/summa/calibration/`

Structure for Unifying Multiple Modeling Alternatives. Full stack: optimizer, parameter manager (local HRU + basin GRU params, depth-related params), worker (optional mizuRoute), and targets for **streamflow**, **snow (SWE/SCA/depth)**, and **ET**. Also has regionalization helpers (`summa_regionalization.py`). Common calib params include snow/soil hydraulic knobs (`k_snow`, `theta_sat`, `vGn_*`, routing gamma params, etc.).

### FUSE
**Path:** `models/fuse/calibration/`

Framework for Understanding Structural Errors. Supports **lumped** and **distributed + mizuRoute** modes. Rich package: NetCDF parameter application, multi-gauge metrics, transfer functions / regionalization, streamflow and snow targets. Can use Symfluence algorithms; FUSE may also run **internal SCE** when configured (`FUSE_RUN_INTERNAL_CALIBRATION`). Typical params: storage/baseflow/percolation-style conceptual parameters.

### GR (GR4J / GR5J / GR6J)
**Path:** `models/gr/calibration/`

Génie Rural conceptual models, optional CemaNeige snow. Parameters are passed via config/runner (not classic trial-file edits). Worker runs isolated GR instances in parallel. Streamflow target handles lumped CSV and distributed NetCDF, including mm/day → m³/s conversion.

### HYPE
**Path:** `models/hype/calibration/`

SMHI semi-distributed HYPE. Parameter manager updates `par.txt`. Worker + streamflow target read `timeCOUT.txt` or routed NetCDF and select outlet subbasins. Supports multi-gauge metrics and regionalization helpers.

### MESH
**Path:** `models/mesh/calibration/`

Modélisation Environnementale Communautaire – Surface Hydrology. Optimizer + parameter manager (`.ini` updates) + worker for trial runs and streamflow metrics. Land-surface / hydrology calibration through MESH settings files.

### VIC
**Path:** `models/vic/calibration/`

Variable Infiltration Capacity. Standard optimizer / parameter-manager / worker trio: update VIC parameter files, run, score streamflow (and related outputs as configured).

### Noah-MP
**Path:** `models/noahmp/calibration/`

Noah-MP (noah-owp-modular style). Parameters split across `namelist.input` (scalar knobs) and `SOILPARM.TBL` (soil hydraulics). Worker scores runoff-related outputs (e.g. surface + underground runoff contributions).

### CLM
**Path:** `models/clm/calibration/`

CLM5 calibration across **three** file targets: namelist hydrology knobs, `params.nc` (snow/PFT), and `surfdata` soil multipliers (~26 params in the manager design). Worker runs via CESM/CLM executable and scores runoff (e.g. `QRUNOFF`).

### CRHM
**Path:** `models/crhm/calibration/`

Cold Regions Hydrological Model. Parameters in text `.prj` project files (key–value). Worker applies params, runs CRHM, metrics from CSV output — suited to cold-region / snow process calibration.

### PRMS
**Path:** `models/prms/calibration/`

Precipitation Roadmapoff Modeling System. Updates structured `params.dat` blocks; worker runs PRMS and scores `statvar` (or equivalent) streamflow output.

### SWAT
**Path:** `models/swat/calibration/`

SWAT TxtInOut text parameters (`.bsn`, `.gw`, `.hru`, `.sol`, `.mgt`, …) with relative / absolute / replace-style updates. Worker scores reach output (`output.rch`).

### mHM
**Path:** `models/mhm/calibration/`

mesoscale Hydrological Model. Fortran namelist (`.nml`) regex updates; worker scores discharge output.

### CWatM
**Path:** `models/cwatm/calibration/`

Community Water Model. Calibratable knobs live in the settings INI `[CALIBRATION]` section (no separate external param-file family). Optimizer / parameter manager / worker follow the unified pattern.

### PCR-GLOBWB
**Path:** `models/pcrglobwb/calibration/`

Global water balance model. Parameters stored as generated NetCDF maps under settings; worker applies maps and scores the configured target (typically discharge).

### LISFLOOD
**Path:** `models/lisflood/calibration/`

Parameters as individual NetCDF map files under a `maps/` directory (e.g. `ksat1.nc`). Worker updates maps, runs LISFLOOD, computes metrics.

### Wflow
**Path:** `models/wflow/calibration/`

Wflow (Deltares) stack: optimizer, parameter manager, worker for file updates and trial evaluation against observations.

### WATFLOOD
**Path:** `models/watflood/calibration/`

Canadian WATFLOOD. Parameters in `.par` files with per-land-class blocks. Worker notes Wine-based execution for the Windows binary path.

### WRF-Hydro
**Path:** `models/wrfhydro/calibration/`

Coupled land / routing. Namelist updates for `hydro.namelist` (routing) and `namelist.hrldas` (Noah-MP side). Worker scores channel output (e.g. CHRTOUT).

### RHESSys
**Path:** `models/rhessys/calibration/`

Regional Hydro-Ecologic Simulation System. Definition-file parameter updates; streamflow target handles CSV / `basin.daily` and daily resampling of observations. Ecohydrology-oriented calibration.

---

## NextGen and modular frameworks

### NextGen (ngen)
**Path:** `models/ngen/calibration/`

Next Generation Water Resources Modeling Framework (BMI modules). Parameter manager spans **CFE**, **NOAH-OWP**, and **PET** config JSON. Worker applies JSON params and scores nexus-style streamflow (`NgenStreamflowTarget`). Config lists often look like `NGEN_NOAH_PARAMS_TO_CALIBRATE`, etc.

---

## Integrated / physically coupled systems

### ParFlow
**Path:** `models/parflow/calibration/`

3D Richards + overland flow. Updates `.pfidb` (van Genuchten, Ksat, Manning, optional Snow-17). Worker isolates dirs; streamflow target converts overland `.pfb` through a two-component linear reservoir for gauge comparison.

### CLM–ParFlow
**Path:** `models/clmparflow/calibration/`

ParFlow coupled with CLM land surface. Same `.pfidb`-centric calibration idea, plus CLM process params (snow, ET, vegetation context). Streamflow target analogous to ParFlow (scaled overland + reservoir routing for lumped domains).

### PIHM
**Path:** `models/pihm/calibration/`

Penn State Integrated Hydrologic Model. Writes `.soil`, `.calib`, `.lc` (and optional Snow-17 forcing params). Streamflow from `.rivflx` river flux output.

### MODFLOW (coupled GW)
**Path:** `models/modflow/calibration/`

**Joint** land-surface + MODFLOW calibration. Land surface model chosen via `LAND_SURFACE_MODEL`; its parameter manager/worker are loaded dynamically and combined with MODFLOW groundwater params. Streamflow target merges surface runoff + MODFLOW drain/baseflow. Uses dCoupler graph coupling when available.

### GSFLOW
**Path:** `models/gsflow/calibration/`

PRMS + MODFLOW-NWT integrated package. Parameter manager covers PRMS `params.dat` and MODFLOW UPW-style groundwater parameters; joint surface–subsurface calibration trials.

---

## Routing-focused calibration

### mizuRoute
**Path:** `models/mizuroute/calibration/`

River routing. Thinner package (optimizer-focused) for routing parameters (e.g. impulse response / channel params). Often used **with** a land model (SUMMA/FUSE) rather than alone.

### T-Route
**Path:** `models/troute/calibration/`

NOAA OWP T-Route. Optimizer for channel / reservoir routing parameters. Same “routing companion” role as mizuRoute in NextGen-style workflows.

---

## Machine learning models

### LSTM
**Path:** `models/lstm/calibration/`

LSTM rainfall–runoff. Uses generic `MLParameterManager` (config-defined bounds) and `LSTMWorker` for trial evaluation. Optimizer hooks into the unified algorithm interface (training-loop style calibration / hyperparameter search depending on config).

### GNN
**Path:** `models/gnn/calibration/`

Graph neural network hydrology models. Same ML parameter-manager pattern as LSTM; worker evaluates GNN predictions for optimization objectives (architecture / hyperparameter oriented).

---

## Fire / specialty

### IGNACIO
**Path:** `models/ignacio/calibration/`

Fire Behavior Prediction (FBP) parameter calibration. Objective is **spatial** (IoU / Dice) between simulated and observed fire perimeters — not streamflow KGE. Core params include FFMC, DMC, DC, FMC, etc. Evolutionary algorithms (DDS, PSO, SCE, DE); gradient methods not used for this spatial objective.

---

## Quick reference table

| Model | Calib package | Notable specialty |
|-------|---------------|-------------------|
| SUMMA | full + targets | Multi-target (Q, SWE, ET); regionalization |
| FUSE | full + multi-gauge / TF | Lumped or distributed; optional internal SCE |
| GR | full + targets | In-memory/config params; GR4J/5J/6J |
| HYPE | full + multi-gauge | `par.txt`; subbasin outlets |
| MESH | full | `.ini` land-surface/hydro |
| VIC | full | Classic VIC param files |
| Noah-MP | full | namelist + SOILPARM.TBL |
| CLM | full | 3 file families (nl / params.nc / surfdata) |
| CRHM | full | `.prj` cold-regions |
| PRMS | full | `params.dat` blocks |
| SWAT | full | TxtInOut relative/absolute edits |
| mHM | full | Fortran namelists |
| CWatM | full | INI `[CALIBRATION]` section |
| PCR-GLOBWB | full | NetCDF parameter maps |
| LISFLOOD | full | Per-parameter NetCDF maps |
| Wflow | full | Wflow settings/maps |
| WATFLOOD | full | `.par` land-class; Wine runner |
| WRF-Hydro | full | hydro + hrldas namelists |
| RHESSys | full + targets | Ecohydrology daily outputs |
| NextGen | full + targets | CFE / Noah-OWP / PET JSON |
| ParFlow | full + targets | `.pfidb` + reservoir routing for Q |
| CLMParFlow | full + targets | Coupled CLM + ParFlow |
| PIHM | full + targets | `.soil` / `.calib` / `.lc` |
| MODFLOW | coupled full | Joint LSM + GW parameter space |
| GSFLOW | full | PRMS + MODFLOW-NWT |
| mizuRoute | optimizer | Routing params |
| T-Route | optimizer | Channel/reservoir routing |
| LSTM | ML stack | Config-defined ML params |
| GNN | ML stack | Graph model / hyperparams |
| IGNACIO | specialty | Spatial IoU/Dice fire perimeters |

---

## Shared pattern (for agent design)

For almost every model, CalibHydroAgent only needs to:

1. Know the **model name** and its **param list config keys**.
2. Ensure the correct **target** (streamflow vs snow vs spatial fire, etc.).
3. Write YAML and call Symfluence `calibrate_model`.
4. Read the same style of optimization results CSV / reports.

Model differences are mostly **file formats and output parsers**, not a different optimization theory.

Companion note: `evaluation_optimization_calibration.md` (how evaluation, optimization, and calibration work together).
