# Tutorial

Follow the lessons in order. Each lesson explains a software task, guides you through it, and describes what you should have before continuing.

| Lesson | Activity |
| --- | --- |
| [00. Find your way around](00-orientation.md) | Set up the repository and identify the source, input, executable, and build directory |
| [01. Run a fluid simulation](01-first-run.md) | Run the baseline and higher-viscosity cases |
| [02. See what the flow is doing](02-first-visualization.md) | View saved fields in VisIt and export comparison images |
| [03. Compute kinetic energy](03-kinetic-energy.md) | Read the fluid application and add a diagnostic using loops and a hierarchy operation |
| [04. Put an elastic rubber band in the fluid](04-elastic-loop.md) | Run the rubber band, inspect its structure files, and synchronize fluid and structure plots |
| [05. Change the model through its input](05-input-change.md) | Follow a stiffness override into the force calculation and compare separate runs |
| [06. Use finer cells near the rubber band](06-refinement.md) | Inspect grid levels, match structure spacing, and compare resolutions |

Begin with [lesson 00](00-orientation.md), using a terminal opened at the repository's top level. The lessons define `EXAMPLES_SRC` for this repository, `EXAMPLES_BUILD` for its build directory, and `IBAMR_ROOT` for the installed library. Keep those variables set as you work through the lessons.

The examples target IBAMR v0.19.0.

Return to the [repository overview](../README.md) for requirements and layout.
