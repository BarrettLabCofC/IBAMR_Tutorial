# Target-held cylinder figures

Rendered from IBAMR v0.19.0 opt-build runs with VisIt 3.4.2 on 2026-11-01. Both images were visually inspected.

- Application: `examples/IB/explicit/ex2/example.cpp`, unchanged release source; provenance in `../../UPSTREAM.json`.
- Baseline: supplied `input2d.cylinder` and `cylinder2d_128.vertex`/`.spring`; four levels, finest spacing 0.125, final time 1, visualization every ten steps.
- Comparison changes only the top-level `Re` from 200 to 400. The existing `MU = 2.0*R/Re` expression changes viscosity from 0.005 to 0.0025. Inlet velocity, density, geometry, target coefficients, and time-step bound stay fixed.
- Images: `F05-cylinder.png` (baseline) and `F05-Re400.png` (comparison).
- Fluid: `viz_IB2d/dumps.visit`, Pseudocolor `Omega`, `hot_desaturated`, common limits [-5,5]. Values outside that range use the end colors. Final ranges are approximately [-13.6815,13.7177] and [-14.7456,14.7985].
- Structure: `viz_IB2d/lag_data.visit`, black Mesh `cylinder2d_128_mesh`.
- Both sources are at physical time 1, state 13 of 14, synchronized with a Time correlation. View bounds: x [-2,4], y [-2,2]. Actual images are 1000 x 1000 pixels.
- Values use consistent user-chosen units. The tutorial illustrates a possible centimeters/seconds/grams interpretation; rendering applies no unit conversion.

The images show the developing flow. Complete logs, force histories, extents, and capture settings are retained in the author records. The earlier ramp-timing and inlet-speed comparisons have been replaced by this Reynolds-number exercise.
