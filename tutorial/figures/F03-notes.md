# Rubber-band stiffness comparison

`F03-baseline.png` and `F03-stiff.png` show actual IBAMR 0.20.0-pre results rendered with VisIt 3.4.2. Created for tutorial 05 on 2026-10-29 to compare one input change at equal physical time.

- Starting input and structure files: `examples/IB/explicit/ex1/input2d`, `curve2d_64.vertex`, and `curve2d_64.spring`.
- Baseline uses the supplied input, `K = 1.0`. The comparison changes only `K` to `2.0`; the existing `DT` expression supplies a smaller step bound. Both end at time `0.1`.
- Source and executable are unchanged between runs. The example source and input provenance is recorded in `../../UPSTREAM.json`.
- Fluid: `viz_IB2d/dumps.visit`, Pseudocolor `P`. Structure: `viz_IB2d/lag_data.visit`, black Mesh `curve2d_64_mesh`.
- Both fluid and structure are at physical time `0.1`, synchronized through a Time correlation. Baseline state index `20`; stiffness-comparison state index `40`.
- Shared view: full unit square, pressure limits `[-10,10]`, color table `hot_desaturated`, 900 x 900 PNG. Values use the example's numerical scales; dimensional units have not been assigned.
- Baseline final pressure range approximately `[-1.6977,4.0242]`; comparison range approximately `[-3.2281,6.7793]`.
- Baseline height/width approximately `1.4336`; comparison approximately `1.0332`. These are transient results at time `0.1`.

The baseline figure was freshly rerun with the same files as tutorial 04. The two captions and the corresponding input change are given in tutorial 05. Both images were visually inspected. Complete capture settings and numerical query results are retained in the author validation records.
