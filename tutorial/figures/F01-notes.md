# Fluid visualization figure notes

`F01-initial.png`, `F01-later.png`, and `F01-viscous.png` were rendered from fresh IBAMR v0.19.0 opt-build runs with VisIt 3.4.2 on 2026-10-30.

- Application and starting input: `examples/navier_stokes/ex1`, with provenance in `../../UPSTREAM.json`.
- Tutorial run edits: `MAX_LEVELS = 1`, `N = 64`, `END_TIME = 1.0`, and `Main.viz_dump_interval = 10`.
- The comparison changes only `MU` from `0.0001` to `0.001`.
- Source: each run's `viz_INS2d/dumps.visit`, Pseudocolor `Omega`, `hot_desaturated`, common limits `[-80,80]`, full unit-square view, 900 x 900 PNG.
- Initial frame: baseline time `0`, index `0`. Later frame: baseline time `1`, index `29`. Comparison frame: viscosity case time `1`, index `26`.
- Maximum absolute saved vorticity at time `1`: approximately `56.01896` for the baseline and `23.77829` for the viscosity comparison.
- The tutorial's `magnitude(<U>)` expression was also evaluated successfully on both runs.
- Values follow the example's numerical scales; dimensional units have not been assigned.

All three images were visually inspected. These are coarse teaching runs, not a convergence study. Complete inputs, logs, and capture records are retained in the author validation records.
