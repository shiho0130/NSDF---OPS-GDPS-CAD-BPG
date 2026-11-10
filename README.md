[README_particle_comparison (1).md](https://github.com/user-attachments/files/34611578/README_particle_comparison.1.md)# NSDF -> OPS+GDPS / CAD-BPG comparison with diagnostics

## Requirements

```bash
pip install numpy scipy matplotlib trimesh
```

The comparison program requires an NSDF `*.npz` created by the original `STL_SDF_paper.py` with `grid_sign` enabled. `trimesh` is only needed in the comparison stage if `--reference-stl` is set.

## Examples

Create NSDF (using your original NSDF generator):

```bash
python STL_SDF_paper.py Stanford_Bunny.stl --dx 0.01 --band 3 --out bunny_paper
```

Run both methods and save quantitative results and all intermediate VTKs every 20 iterations:

```bash
python STL_particle_comparison_metrics.py bunny_paper.npz --out bunny_comparison --mode both --iters 500 --auto-stop --shift-tol 0.001 --patience 20 --min-iters 100 --metrics-every 10 --print-every 10 --write-every 20 --display-diameter-ratio 1.4 --reference-stl Stanford_Bunny.stl
```

To reproduce a comparison at exactly 150 iterations, **omit** `--auto-stop` and specify `--iters 150`.

## Results

- `bunny_comparison/comparison.csv`: methods compared side by side.
- `bunny_comparison/run_info.json`: VCT time and initial diagnostics.
- `bunny_comparison/initial_before_vct.vtk`: original regular grid.
- `bunny_comparison/initial_after_vct.vtk`: common particle configuration after VCT, with per-particle quality arrays.
- `bunny_comparison/OPS_GDPS/final.vtk`: OPS+GDPS particle results with quality arrays.
- `bunny_comparison/CAD_BPG/final.vtk`: CAD-BPG results with quality arrays.
- `bunny_comparison/OPS_GDPS/history.csv` and `CAD_BPG/history.csv`: convergence and quality metrics by sampled iteration.
- `bunny_comparison/*.png`: convergence history comparisons.

## ParaView

Open `OPS_GDPS/final.vtk` or `CAD_BPG/final.vtk`, click `Apply`, and select one of these arrays in **Color By**:

- `surface_offset_abs_error`: distance from interpolated NSDF offset surface (NOT exact STL distance)
- `nearest_distance`: nearest-neighbor distance
- `spacing_abs_error`: absolute distance from target spacing
- `local_number_density`: neighbors per unit sphere volume (number of neighbors excluding self)
- `density_reference_abs_error`: relative error from regular-lattice SPH kernel sum
- `cumulative_shift`: cumulative absolute displacement through relaxation
- `render_radius`: suggested visual particle sphere radius
- `zone_id`, `mat_id`, `fixed_particle`: labels

Set Representation to **Point Gaussian**, Shader Preset to **Sphere** if available, and adjust `Gaussian Radius`. For a particle spacing `dx=0.01` and display diameter ratio 1.4, suggested radius is `0.007`. Alternatively use Glyph -> Sphere and scale by the `render_radius` scalar (some ParaView versions need `Scale Factor=2` for a unit-radius sphere).

## Definitions and caveats

- `particle_volume_sum = N * dx^3`; it is a *particle representative volume sum*, **not an independent reconstruction of the actual outer surface volume**. When `--reference-stl` is supplied, volume-error percentage compares that sum with watertight STL volume under exactly the NSDF normalization scaling.
- `surface_offset_rmse` is based on the **interpolated NSDF** at particle centers and the designed offset `phi=-dx/2` (a boundary-enforcement error). This can be small by construction and does not establish accuracy against the original STL. Exact STL surface error requires independent distance queries.
- `nearest_distance_cv_percent`, `spacing_rmse`, and local density CV are based on deep interior particles to avoid boundary-support biases. For sparse toy models there may be too few deep-interior particles for meaningful statistics.
- `density_reference_ratio` is the SPH kernel-number density divided by the regular-lattice reference, and is not physical mass density.
- `pseudo_kinetic_energy` is **method-dependent**: OPS+GDPS uses squared normalized shift, while CAD-BPG uses a reset-velocity acceleration proxy. These values should not be compared directly as the same physical kinetic energy. Instead compare `normalized_rms_displacement`.
- `--auto-stop` requires a normalized per-iteration RMS displacement below `--shift-tol` for `--patience` consecutive iterations, after `--min-iters`, and has maximum `--iters`. This is an additional criterion, not a published numerical threshold in Zhu et al. (2021) or Yu et al. (2023).
- Size/positions of physical particles are unchanged by `--display-diameter-ratio`, which only controls the visualization radius field.

## Code architecture

Both original update loops, including neighbor sums and projection, are retained. The program builds one NSDF-based common VCT initial distribution and branches into OPS+GDPS and CAD-BPG from copies of that distribution. Large point-cloud performance is dominated by the original all-particles neighbor-search and kernel-loop algorithms.
