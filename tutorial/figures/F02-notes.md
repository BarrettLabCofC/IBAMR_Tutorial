# Elastic rubber-band figure notes

`F02-initial.png` and `F02-later.png` are VisIt 3.4.2 renders of the tutorial 04 IBAMR 0.20.0-pre run. Their purpose is to show the moving rubber band together with the surrounding pressure field.

- Input: `examples/IB/explicit/ex1/input2d`, with its supplied vertex/spring files; no additional run edits.
- Fluid source: the run's `viz_IB2d/dumps.visit`, Pseudocolor `P`.
- Structure source: the run's `viz_IB2d/lag_data.visit`, Mesh `curve2d_64_mesh`, black lines.
- Initial image: both datasets at time `0`, state `0`.
- Later image: both datasets at time `0.1`, state `20`.
- A Time database correlation synchronizes the datasets.
- Display: full unit square, pressure limits `[-5,5]`, color table `hot_desaturated`, 900 × 900 PNG.
- Values follow the example's numerical scales; no dimensional units have been assigned.

The saved initial pressure is zero. At time `0.1`, the saved pressure ranges approximately from `-1.6977` to `4.0242`. The initial rubber-band bounds are approximately `[0.3214,0.6786] × [0.15,0.85]`; later bounds are `[0.2978,0.7022] × [0.2101,0.7899]`. These describe the short teaching run, not an equilibrium or convergence result.
