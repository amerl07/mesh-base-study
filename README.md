# Mesh Base Size Study

CFD mesh convergence study for the full car (straight-line condition), evaluating how the volume mesh **base size** affects aerodynamic force outputs. The goal is to find a base size that is fine enough to give mesh-independent results but coarse enough to keep run time/cell count reasonable.

## Contents

| File | Description |
|---|---|
| `mesh_base_study.ipynb` | Main analysis notebook: parses force reports, builds tables, generates plots, exports results. |
| `force_reports_{30,35,40,50,60}mm.txt` | Raw BFR CFD force report exports, one per mesh base size tested. |
| `mesh_walltime_summary.xlsx` | Run time per mesh size (start / solve end / job complete times, iterations run), derived from file mtimes on the cluster. |
| `mesh_study_results.xlsx` | Exported results — `Raw Values`, `Percent Delta` and `Wall Time` sheets. |
| `plot_*.png` | Generated plots (see below). |

## Method

1. **Parse** each `force_reports_*mm.txt` file (label/value/unit lines) into a dict of metrics, keyed by mesh size (e.g. `30mm`, `60mm`).
2. **Assemble** a `Metric x Mesh Size` table (`raw_df`), ordered from finest (30mm) to coarsest (60mm) mesh.
3. **Plot raw values** for downforce components, drag components, CoP%, and a combined full-car summary (DF, Drag, CoP) vs. mesh size.
4. **Compute percent delta** of every metric at each mesh size relative to the finest mesh (30mm, treated as the reference/baseline):

   ```
   % delta = (value_at_mesh - value_at_30mm) / |value_at_30mm| * 100
   ```

5. **Plot percent deltas** for DF components, drag components, the full-car summary, and cell count, to visualize how far each mesh size drifts from the 30mm baseline as the mesh is coarsened.
6. **Export** both the raw values and percent-delta tables to `mesh_study_results.xlsx`.
7. **Load wall time** from `mesh_walltime_summary.xlsx` (footnote rows dropped), key it by mesh size to match the force reports, and join in cell count.
8. **Derive cost deltas** — solve time per iteration, total wall time and cell count as % change vs. the 30mm baseline.
9. **Plot wall time** (by mesh size, per-iteration vs. cell count, and cost vs. accuracy against the full-car force deltas) and add a `Wall Time` sheet to `mesh_study_results.xlsx`.

## Plots generated

- `plot_df_components.png` — Component downforce (N) vs. mesh size
- `plot_drag_components.png` — Component drag (N) vs. mesh size
- `plot_cop_percentage.png` — Full car CoP (%) vs. mesh size
- `plot_full_car_summary.png` — Full car DF, Drag (N) and CoP (%) vs. mesh size (dual axis)
- `plot_delta_df_components.png` — % change in DF components vs. 30mm baseline (Endplate DF excluded — its near-zero raw force makes percent change noisy/misleading)
- `plot_delta_drag_components.png` — % change in drag components vs. 30mm baseline
- `plot_delta_full_car_summary.png` — % change in full car DF, Drag, CoP vs. 30mm baseline
- `plot_delta_cell_count.png` — % change in mesh cell count vs. 30mm baseline
- `plot_walltime.png` — Solve + post-processing wall time per mesh size, with iterations run annotated
- `plot_time_per_iteration.png` — Solve time per iteration (s) vs. cell count
- `plot_cost_vs_accuracy.png` — % change in full car DF, Drag, CoP vs. % change in solve time per iteration (both vs. 30mm)

## Notes on the input data

- Reports follow the `BFR_CFD_Standards` format: half-car, undoubled raw values, with downforce read as positive.
- Each report's filename encodes its mesh base size in mm (e.g. `force_reports_60mm.txt` → 60 mm base size).
- The notebook checks that all report files contain the same set of metrics and warns if any mesh size is missing or has extra metrics.

- Wall times come from file mtimes on the cluster (`ls -la --time-style=full-iso`), not PBS accounting logs, so they carry some noise from node allocation/load (roughly ±10% on time per iteration).
- **The 40mm run stopped at 909 iterations** (the others ran 1250), so its total and solve time are not comparable. Cost comparisons use **solve time per iteration**. It also means the 40mm forces come from a shorter run — the reason it stopped has not been confirmed.
- Convergence is assumed for all five runs and has not been verified from residual/monitor data.

## Requirements

- Python 3
- `pandas`, `matplotlib`, `openpyxl`
- `mesh_walltime_summary.xlsx` in this directory (needed by the Wall Time section of the notebook)

## Usage

Run all cells in `mesh_base_study.ipynb` from top to bottom. It will re-parse the `force_reports_*mm.txt` files and `mesh_walltime_summary.xlsx` in this directory, regenerate all `plot_*.png` files, and overwrite `mesh_study_results.xlsx`. Run the wall time export cell after the main export cell — the main export rewrites the file, and the wall time cell appends its sheet.

## Observations

**Cost (from wall time data):**

- Coarsening buys little: 30mm → 60mm cuts cell count by ~11% and solve time per iteration by ~15% (~12 min on a ~93 min job).
- Post-processing is roughly constant (~11–13 min) regardless of mesh size.
- Time per iteration is not monotonic in cell count (e.g. 35mm is faster than 40mm), consistent with the timing noise noted above.

_Recommended mesh base size: TBD — to be decided from `plot_cost_vs_accuracy.png` together with the force deltas above, once convergence of each run (especially 40mm) is confirmed._
