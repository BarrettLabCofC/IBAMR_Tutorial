# Tutorial

Follow the tutorials in order. Each tutorial explains a software task, guides you through it, and describes what you should have before continuing.

| Tutorial | Activity |
| --- | --- |
| [00. Find your way around](00-orientation.md) | Set up the repository and identify the source, input, executable, and build directory |
| [01. Run a fluid simulation](01-first-run.md) | Run the baseline and higher-viscosity cases |
| [02. See what the flow is doing](02-first-visualization.md) | View saved fields in VisIt and export comparison images |
| [03. Compute kinetic energy](03-kinetic-energy.md) | Read the fluid application and add a diagnostic using loops and a hierarchy operation |
| [04. Put an elastic rubber band in the fluid](04-elastic-loop.md) | Run the rubber band, inspect its structure files, and synchronize fluid and structure plots |
| [05. Change the model through its input](05-input-change.md) | Follow a stiffness override into the force calculation and compare separate runs |
| [06. Use finer cells near the rubber band](06-refinement.md) | Inspect grid levels, match structure spacing, and compare resolutions |
| [07. Run a finite-element elastic ring](07-finite-element-ring.md) | Run and visualize an elastic structure represented by finite elements |
| 08. Modify the finite-element model (planned) | Change the initial stretch and compare geometry and motion |
| [09. Hold a cylinder in a flow](09-target-forces-and-boundaries.md) | Use target forces, follow physical boundary conditions, and explore consistent units and Reynolds number |
| 10. Create a named application (planned) | Organize a personal rubber-band application and its build target |
| 11. Implement a custom spring law (planned) | Add and compare a small C++ model change |

Entries marked planned are not yet available.

Begin with [tutorial 00](00-orientation.md), using a terminal opened at the repository's top level. The tutorials define `EXAMPLES_SRC` for this repository, `EXAMPLES_BUILD` for its build directory, and `IBAMR_ROOT` for the installed library. Keep those variables set as you work through the tutorials.

The current examples and figures have been checked with IBAMR v0.19.0.

Return to the [repository overview](../README.md) for requirements and layout.
