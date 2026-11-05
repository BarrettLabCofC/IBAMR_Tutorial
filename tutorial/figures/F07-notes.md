# Finite-element ring figures

Rendered from the IBAMR v0.19.0 opt application using VisIt 3.4.2 for tutorial 07. The fluid and structure times were checked at all 65 saved states.

- Source: `examples/IBFE/explicit/ex1/example.cpp`, unchanged release copy. The tutorial input sets `END_TIME = 2.0` and `Main.viz_dump_interval = 16`. Provenance is in `../../UPSTREAM.json`.
- Baseline uses one 128-by-128 fluid level, MFAC 4, QUAD9 elements, the source's gamma 0.15 and structural mu 1, fluid MU 0.01, and END_TIME = 2.0.
- Fluid: `viz_IB2d/dumps.visit`, Pseudocolor P, hot_desaturated, fixed limits [-5,15]. Structure: `viz_IB2d/output.ex2`, black Mesh plot of Mesh.
- Structure operator: Displace, variable `ring_displacement`, expression `{<X_0>, <X_1>, 0} - coord(<Mesh>)`, multiplier 1. This displays current physical coordinates from the saved fields.
- Time correlation matches the two databases. F07-initial is state 0 at time 0; F07-later is state 64 at time 2.0. Each database has 65 states.
- Full unit-square view, 900-by-900 PNGs. Values use the example's consistent numerical scales; rendering applies no unit conversion.
- Initial saved pressure is zero. Final pressure range is approximately [-2.506982,12.520104].
- Actual displaced-mesh widths/heights: initially 0.6250000/0.9245613; finally 0.7984205/0.7215631. Queries use Actual data so the displacement is included.

These show the baseline's initial and later configurations. They are not a controlled comparison with the classical spring model. Complete capture settings, sessions, metadata, and measurements are retained in the author validation records.
