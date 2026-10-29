# Lesson 03: See what the flow is doing

Open the two runs from lesson 01 in VisIt, select fields and saved times, and export comparable images.

## Before you start

Finish both runs in [lesson 01](01-first-run.md). Use VisIt with its SAMRAI reader available. The output has been opened and rendered with VisIt 3.4.2; these graphical steps are a pilot awaiting a learner walkthrough.

Open `examples/navier_stokes/ex1/first-flow/viz_INS2d/dumps.visit` under `EXAMPLES_BUILD`, the example repository's build directory.

## Open and draw the fluid output

1. Start VisIt. Click **Open** in the main controls, select the baseline `dumps.visit`, and open it.
2. Under **Plots**, choose **Add → Pseudocolor → Omega**. Click **Draw**. If a grid-level plot was added automatically, hide or delete it for this first view.
3. Use the time slider to select the first saved state. Check the displayed physical time is `0`.
4. Double-click the Pseudocolor plot to open its attributes. Enable explicit minimum and maximum limits, set them to `-80` and `80`, choose `hot_desaturated` as the color table, and apply.
5. Move to the last state. Check the displayed time is `1`, then return to the first state and use the animation controls to play the sequence.

## How it works: what the colors mean

In this two-dimensional example, `Omega` is the out-of-plane vorticity,

$$\omega=\frac{\partial u_y}{\partial x}-\frac{\partial u_x}{\partial y}.$$

Selecting `Omega` colors the plot by vorticity. The legend shows its signed values; use the same color limits when comparing saved times or separate runs.

The input requests velocity, pressure, and vorticity through `output_U`, `output_P`, and `output_Omega` in the selected fluid integrator block. The application uses a `VisItDataWriter` and writes initial data, then data at the requested step interval and at the last step. VisIt reads those saved fields; it does not rerun the fluid solver when you move the time slider.

### Follow a saved state from the application to VisIt

Open `examples/navier_stokes/ex1/example.cpp` in the source directory to follow how the application writes visualization files. First, locate the call that registers the visualization writer:

```cpp
time_integrator->registerVisItDataWriter(visit_data_writer);
```

Further down in the same file, locate the calls that prepare and write each saved state:

```cpp
time_integrator->setupPlotData();
visit_data_writer->writePlotData(patch_hierarchy, iteration_num, loop_time);
```

Registering the writer lets the integrator connect its requested plot quantities to the output system. `setupPlotData()` prepares data for plotting; `writePlotData` saves a hierarchy state with a step number and physical time. The application calls this pair once before the time loop, then at selected completed steps. That first call explains why a frame exists at $t=0$.

The input uses `viz_dump_interval = 10`. Inside the loop, the application writes when `iteration_num % viz_dump_interval == 0` or this is the last step. `%` means the remainder after division: every tenth step has remainder zero. The last-step condition saves the endpoint even when its step number is not a multiple of ten. The `.visit` list points to the resulting states; the displayed time comes from the saved state, not from treating the frame number as elapsed time.

There are two different kinds of derived field in this activity. `Omega` is already in the saved IBAMR output. The `speed` expression below is evaluated by VisIt from saved `U`. Changing that expression changes what you view without changing the simulation or creating a new solved state.

The coordinates, time, and fields in this tutorial follow the example's numerical scales; no dimensional length or time unit has been assigned. Vorticity has inverse-time units under any consistent physical interpretation. Pressure has an arbitrary additive reference in this periodic case, with mean normalization requested in the input.

![Initial vorticity](figures/F01-initial.png)

*IBAMR result at $t=0$, with vorticity limits $[-80,80]$. Two narrow bands mark the initial shear layers.*

![Later vorticity](figures/F01-later.png)

*The same run at $t=1$, using the same limits. The bands have rolled into rotating regions. This deliberately coarse teaching grid also leaves visible small-scale numerical texture; the picture is not an accuracy assessment.*

## Look at speed

Speed is the magnitude of velocity, $|\mathbf u|=\sqrt{u_x^2+u_y^2}$. To give it a clear scalar name, open **Controls → Expressions**, create a scalar expression called `speed`, and enter:

```text
magnitude(<U>)
```

Apply the expression. Hide the `Omega` plot and add **Pseudocolor → speed**, then **Draw**. Let this speed plot choose its own limits: the signed vorticity range is inappropriate for speed. The same expression has been checked in the automated VisIt capture.

Compare the speed and vorticity views at $t=1$. The fastest fluid and the strongest local rotation need not occur in the same places. For flow direction, a vector plot of `U` would add information that speed alone cannot provide.

## Compare the viscosity change

Return to `Omega`. Open `examples/navier_stokes/ex1/first-flow-viscous/viz_INS2d/dumps.visit` under the build directory, make a Pseudocolor plot of `Omega`, and select its final physical time `1`. Set the same color table and limits `-80` and `80`. Save an image of each run at that time before switching between them.

![Higher-viscosity result](figures/F01-viscous.png)

*IBAMR result with `MU = 0.001` at $t=1$, with the same vorticity limits. The structures are broader and their peak vorticity is weaker than in the baseline.*

In the reference output, the maximum absolute saved vorticity at $t=1$ was approximately `56.02` for `MU = 0.0001` and `23.78` for `MU = 0.001`. These are VisIt MinMax queries on the saved field, not analytic predictions. Larger viscosity smooths velocity gradients. The numbers support the observation for this grid and short run; they do not establish a converged result.

Before saving the comparison, check that both plots show physical time `1`, variable `Omega`, and color limits `-80` to `80`. These settings let the images show differences between the runs consistently.

## Save a figure

With the desired plot and time visible, choose **File → Set save options**, select PNG and the corresponding run directory under the build directory, then use **File → Save window**. Keep the legend and time visible. Name the image so that you can identify the run and time, for example `first-flow-Omega-t1.png`.

The figures shown here were exported at 900 × 900 pixels. Keep the input, log, and visualization files with the saved images in the build directory so the results can be revisited.

If `Omega` is absent, check that you opened the fluid `dumps.visit` from the intended run. If you only see a grid outline, add and draw the Pseudocolor plot.

## Before moving on

This lesson opened a saved time sequence in VisIt, plotted vorticity and speed, and compared two runs at the same physical time with consistent display settings. You should have saved comparison images for the baseline and higher-viscosity runs at $t=1$, with the input, log, and complete visualization output retained in each run directory.

The next tutorial will add an elastic loop to the fluid simulation. You will view the fluid and structure outputs together at matching times.
