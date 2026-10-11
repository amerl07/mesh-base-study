# Mesh Base Size Study — Scaled Prism Layers

Follow-up to the mesh base size study in the parent folder. In that study, changing the mesh base/face size in STAR-CCM+ barely changed the total cell count (30mm → 60mm cut cells by only ~11%). This was most likely because the prism-layer settings did not scale with the surface mesh refinement. After the meshing setup was fixed, runs were made at **30, 40, 60 and 100mm** base size. This folder repeats the force and wall time analysis on those runs.

## Contents

| File | Description |
|---|---|
| `mesh_base_study_scaled.ipynb` | Analysis notebook (same steps as `../mesh_base_study.ipynb`, adapted for these runs). |
| `force_reports_{30,40,60,100}mm_scaled.txt` | Raw BFR CFD force report exports. The 30mm job was killed during post-processing, so its report was exported afterwards from the saved `_0700.sim` with `export_scenes.java`. |
| `mesh_walltime_summary_scaled.xlsx` | Run time per mesh size from cluster file mtimes. Same layout as `../mesh_walltime_summary.xlsx`, plus a `Core-Seconds / Iteration` column. |
| `export_scenes.java` | Team post-processing macro (scenes → PNG, reports → `force_reports.txt`); used to post-process the 30mm checkpoint. |
| `mesh_study_results_scaled.xlsx` | Exported results — `Raw Values`, `Percent Delta`, `Original vs Scaled` and `Wall Time` sheets. |
| `plot_*.png` | Generated plots (see below). |

## Differences from the original study

- **The baseline is 30mm**, the finest mesh. It is set by `BASELINE_COL` in the notebook:
  `% delta = (value_at_mesh - value_at_30mm) / |value_at_30mm| * 100`
- **Sign convention.** These reports use *downforce reads negative*, while the original used *positive*. The parser reads the `# DF sign convention` header and flips every `* DF` metric so that downforce reads **positive**. `ClA` is left as reported.
- **Metric set changed.** `EL8 DF/Drag` are no longer reported. All four scaled reports contain the same 45 metrics.
- **The DF %-change plot excludes components below 0.1 N at the baseline** (`Endplate DF`, `Swan Neck DF`).
- **Cost uses core-seconds per iteration.** The runs used different core counts (127 / 95 / 63 / 15), so wall-clock solve time per iteration can't be compared across runs.
- **The notebook recomputes durations from the timestamps.** The wall time sheet's formula columns have no cached values until the file is saved once in Excel/WPS, so the notebook doesn't read them.
- **New section: original vs. scaled comparison** (`plot_compare_original_vs_scaled.png`).

## Plots generated

- `plot_df_components.png`, `plot_drag_components.png`, `plot_cop_percentage.png`, `plot_full_car_summary.png` — raw values vs. mesh size
- `plot_delta_df_components.png`, `plot_delta_drag_components.png`, `plot_delta_full_car_summary.png`, `plot_delta_cell_count.png` — % change vs. 30mm
- `plot_walltime.png` — solve + post-processing wall time, with iterations, cores and core-hours annotated
- `plot_time_per_iteration.png` — core-seconds per iteration vs. cell count
- `plot_cost_vs_accuracy.png` — % change in full car DF / Drag / CoP vs. % change in core-seconds per iteration (vs. 30mm)
- `plot_compare_original_vs_scaled.png` — original vs. scaled setup

## Usage

Run all cells in `mesh_base_study_scaled.ipynb` from top to bottom, from inside this folder. The comparison section also reads `../force_reports_*mm.txt`.

## Notes on the wall time data

- Start time comes from `machinefile`. Solve end comes from the numbered `.sim` file. Job complete comes from the PBS `.o<jobid>` log. All are file mtimes (UTC), not PBS accounting.
- **Iterations Run is an estimate** (yellow in the sheet). It is taken from the numbered `.sim` suffix (`_1200` → 1250, `_0900` → 900, `_0700` → 700, `_0600` → 600). In the original study, `_1200` runs ran 1250 iterations and the `_0900` run ran 909. Verify from each job's `starccm.log`.
- **Cores Used is an estimate** (yellow in the sheet), calculated as machinefile size / 8 bytes per line: 95, 63 and 15. 30mm's 127 is confirmed with `wc -l`, which also confirms the method (1016 B / 8 = 127). Verify with `wc -l machinefile`.
- All four runs ran at the same time on 2026-10-04 (shared cluster load).
- **30mm was killed (SIGTERM) about 5 minutes into post-processing.** Its solve time is complete, but its job-complete time is the kill time, so its post-processing time, total wall time and core-hours are partial.
- Convergence is assumed for all runs and has not been checked against residual/monitor data. The 30, 40 and 60mm runs appear to have stopped early (~700 / ~900 / ~600 iterations).

## Observations

| | 30mm (baseline) | 40mm | 60mm | 100mm |
|---|---|---|---|---|
| Cell count (M) | 111.4 | 60.5 (−46%) | 26.0 (−77%) | 9.5 (−92%) |
| Full Car DF (N) | 248.3 | 255.0 (+2.7%) | 252.7 (+1.8%) | 240.5 (−3.2%) |
| Full Car Drag (N) | 97.1 | 97.6 (+0.5%) | 98.6 (+1.5%) | 96.1 (−1.1%) |
| Full Car CoP (%) | 53.81 | 53.14 (−1.3%) | 53.15 (−1.2%) | 51.59 (−4.1%) |
| Radiator pressure drop | 19.96 | 19.97 | 15.26 | −55.25 |
| Core-s / iteration | 1160 | 500 (−57%) | 274 (−76%) | 84 (−93%) |
| Core-hours (job) | 236 (partial) | 152 | 63 | 33 |

- **The meshing fix worked:** cell count roughly doubles with each refinement step (9.5 → 26 → 60 → 111M).
- **Full-car results don't converge steadily.** Full Car DF goes 248 → 255 → 253 → 240 N from 30 to 100mm, so 40mm is *further* from 30mm than 60mm is. The 30–60mm differences (≤2.7% DF, ≤1.5% drag) are small enough that they could be convergence noise rather than a mesh trend. The 30, 40 and 60mm runs all stopped early, so check their monitors before reading too much into them.
- **40mm and 60mm agree with each other** (DF within 0.9%, CoP identical), and both differ from 30mm in the same way: CoP is about 0.7 points lower and DF is 2–3% higher.
- **Some components are consistently sensitive to the mesh below 40mm:**
  - **Wheel Drag** reads 13–21% lower on every coarser mesh (10.5 N at 30mm vs. about 8.8 N).
  - **FW DF** reads 4–5% higher on every coarser mesh.
  - **Body Drag** is 6–11% higher.
- **Radiator:** 30 and 40mm agree (pressure drop 19.96 vs. 19.97). 60mm underpredicts it by 24%, and 100mm is unphysical (negative inlet pressure). Use ≤40mm if cooling numbers matter.
- **100mm is too coarse:** DF −3.2%, CoP −4.1%, EL7 DF −37%, and the radiator flow breaks down.
- **Cost:** 30mm costs 2.3× as much per iteration as 40mm and 4.2× as much as 60mm.
- **The absolute values differ between the original and scaled studies.** Both RW Area (2.21 → 1.93 m²) and the reported RW CoP changed, so something besides the mesh changed. Check this before comparing absolute numbers across the two studies.

_Recommended mesh base size: **60mm** for full-car aero development. It is within about 2% of 30mm on DF and drag at about a quarter of the cost per iteration, and no less accurate than 40mm on the full-car numbers. Use **40mm** when radiator/cooling results matter. Use **30mm** when wheel drag or front-wing load needs to be absolute. This is pending a convergence check (all three finer runs stopped at 600–900 iterations) and confirmation of the iteration counts._
