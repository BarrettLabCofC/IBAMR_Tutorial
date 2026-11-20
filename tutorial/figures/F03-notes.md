# Classical IB stiffness comparison

Both frames are at time 0.6. Pressure limits are [-10,10]. The stiffness case changes K from 1 to 2.

- Baseline at time 0.6, state 120 of 121 saved states. Width 0.430722, height 0.571779, height/width 1.327490. Pressure range [-1.452588, 3.578816].
- Stiff at time 0.6, state 240 of 241 saved states. Width 0.567682, height 0.438566, height/width 0.772555. Pressure range [-2.941667, 7.273967].

Rendered from IBAMR v0.19.0 opt output in VisIt 3.4.2. Fluid and structure times were checked for each image. The pressure plots use hot_desaturated with a black structure overlay.


## Combined view (2026-11-20)

F03-side-by-side.png shows the same actual v0.19.0 outputs together in one VisIt 3.4.2 window at time 0.6. Baseline K=1 is left; stiff K=2 is right. Transform translates both stiff pressure and stiff mesh by (1.2,0,0). Shared pressure limits [-10,10] and hot_desaturated colors, black meshes, one pressure legend, view x [0,2.2], y [0,1], 1600 by 850 pixels. All four plot times and translated pressure extents were checked. Translation changes displayed position but not shape or pressure values. The image was visually inspected.
