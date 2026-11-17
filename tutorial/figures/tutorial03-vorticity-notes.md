# Tutorial 03: computed vorticity figure

Created 2026-11-17 from the tutorial 03 extension, built against IBAMR v0.19.0 optimized libraries. This is simulation output, rendered in VisIt 3.4.2.

- File: tutorial03-computed-vorticity.png.
- Purpose: show the scalar field produced by the optional postprocessing function.
- Input: tutorial 01 baseline, N=64, MAX_LEVELS=1, END_TIME=1, OUTPUT_OMEGA=FALSE.
- Program: the tutorial's manual/dot energy application plus the vorticity extension.
- Database: first-flow-vorticity/viz_INS2d/dumps.visit.
- Pseudocolor: computed_vorticity, cell-centered scalar; physical time 1, step 287.
- View: unit square; color table hot_desaturated; fixed limits [-80,80].
- No structure or mesh overlay; dimensionless example coordinates; 900 by 900 pixels.
- Source and simulation attribution: IBAMR developers; see repository COPYRIGHT and UPSTREAM.json.
- Capture and validation records are maintained with the tutorial authoring materials.
