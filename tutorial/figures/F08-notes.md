# Initial-stretch comparison figures

Rendered from IBAMR v0.19.0 opt output in VisIt 3.4.2. All four images were visually inspected.

- Source is examples/IBFE/explicit/ex1/example.cpp. The baseline uses gamma 0.15. The comparison changes only ModelData::gamma to 0.05 and rebuilds the existing IBFE-ex1 target.
- Both runs use the same input, with END_TIME 2, DT 1/512, and visualization every 16 steps. Each produces 65 states.
- Fluid P from viz_IB2d/dumps.visit, hot_desaturated, limits [-5,15]. Black Mesh from viz_IB2d/output.ex2, displaced by the saved X_0 and X_1 fields using the tutorial 07 expression with multiplier 1.
- Initial figures are at time 0, state 0. Later figures are at time 2, state 64. Fluid and structure times were checked at every saved state.
- Full unit-square view, 900 x 900 PNGs. Rendering applies no unit conversion.

- Gamma 0.15, initial width/height 0.6250000/0.9245613, final 0.7984205/0.7215631. Final pressure range [-2.5069816,12.5201044].
- Gamma 0.05, initial width/height 0.6250000/0.7246561, final 0.6861719/0.6601022. Final pressure range [-1.9929208,13.3258009].

The coordinate map changes the initial material area as well as the aspect ratio. The tutorial explains this while comparing geometry and motion with the same stress law and coefficients.
