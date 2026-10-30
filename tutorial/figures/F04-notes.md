# Rubber-band refinement figures

Created 2026-10-30 from actual IBAMR 0.20.0-pre output, rendered with VisIt 3.4.2. The figures compare a one-level baseline with a two-level locally refined run and show where the fine patches lie.

- Application: `examples/IB/explicit/ex1/example.cpp`, unchanged. Baseline uses the supplied tutorial `input2d` and `curve2d_64` files.
- Refined input: `MAX_LEVELS = 2`, `REF_RATIO = 2`, and both initializer structure names changed to `curve2d_128`. The matching vertex and spring files are unchanged upstream data. Provenance is in `../../UPSTREAM.json`.
- Both runs retain `N = 64`, `K = 1`, and `END_TIME = 0.1`. Existing expressions adjust `ds`, spring stiffness, and the step bound consistently with the finer resolution.
- All figures show physical time `0.1`. Baseline state index `20`; refined index `40`. Time correlations synchronize each fluid and structure sequence.
- Pressure images: `dumps.visit`, Pseudocolor `P`, `hot_desaturated`, limits `[-5,5]`, and black structure Mesh overlays. Baseline mesh `curve2d_64_mesh`; refined mesh `curve2d_128_mesh` from `lag_data.visit`.
- Grid image: fluid Subset `levels`, Wireframe enabled, plus the refined black structure mesh. Red denotes level 0 and green denotes level 1. These are patch/level outlines rather than individual fluid cells. At this state the saved hierarchy has one coarse patch and eight fine patches.
- Full unit-square view; 900 x 900 PNG. Fields follow the example's numerical scales without assigned dimensional units.
- Final width/height ratios: baseline `1.4335747840`; refined `1.4020259745`. This is a two-resolution comparison, not a convergence result.

The three images were visually inspected. Complete run inputs/logs and capture metadata are retained with the author validation records. The figure source is IBAMR simulation data, not an illustrative reconstruction.
