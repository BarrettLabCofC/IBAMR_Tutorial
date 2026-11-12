# Tutorial

Follow the tutorials in order. Tutorials 00–11 cover the fundamentals; tutorials 12 onward explore more detailed models. Each tutorial explains a software task, guides you through it, and describes what you should have before continuing.

| Tutorial | Activity |
| --- | --- |
| [00. Find your way around](00-orientation.md) | Set up the repository and identify the source, input, executable, and build directory |
| [01. Run a fluid simulation](01-first-run.md) | Run the baseline and higher-viscosity cases |
| [02. See what the flow is doing](02-first-visualization.md) | View saved fields in VisIt and export comparison images |
| [03. Compute kinetic energy](03-kinetic-energy.md) | Read the fluid application and add a diagnostic using patch loops and the weighted `dot` operation |
| [04. Put an elastic rubber band in the fluid](04-elastic-loop.md) | Run the rubber band, inspect its structure files, and synchronize fluid and structure plots |
| [05. Change the model through its input](05-input-change.md) | Follow a stiffness override into the force calculation and compare separate runs |
| [06. Use finer cells near the rubber band](06-refinement.md) | Inspect grid levels, match structure spacing, and compare resolutions |
| [07. Run a finite-element elastic ring](07-finite-element-ring.md) | Run and visualize an elastic structure represented by finite elements |
| [08. Change the ring’s initial stretch](08-change-initial-stretch.md) | Change the initial stretch and compare geometry and motion |
| [09. Hold a cylinder in a flow](09-target-forces-and-boundaries.md) | Use target forces, follow physical boundary conditions, and explore consistent units and Reynolds number |
| [10. Build your own rubber-band application](10-your-rubber-band-application.md) | Create a standalone CMake project, generate points and springs in C++, and measure the enclosed area |
| [11. Save and restart a simulation](11-restarting.md) | Save a checkpoint, resume from it, and compare with an uninterrupted run |
| [12. Bend a finite-element strip in a flow](12-finite-element-strip.md) | Run a strip held at both ends and add a spatially varying material stiffness in C++ |
| [13. Solve a solid mechanics problem in 3D](13-cooks-membrane.md) | Run solid-only Cook’s membrane and modify its surface-force callback to twist it with an uneven load |

Begin with [tutorial 00](00-orientation.md), using a terminal opened at the repository's top level. The tutorials define `EXAMPLES_SRC` for this repository, `EXAMPLES_BUILD` for its build directory, and `IBAMR_ROOT` for the installed library. Keep those variables set as you work through the tutorials.

The current examples and figures have been checked with IBAMR v0.19.0.

Return to the [repository overview](../README.md) for requirements and layout.
