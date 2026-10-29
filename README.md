# Learning IBAMR

A guided introduction to building, running, and modifying IBAMR applications. The tutorial is for readers familiar with the basic fluid and immersed boundary equations who want practical experience using the software. Each lesson combines a hands-on task with explanations of the input settings and C++ code involved.

## Using the tutorial

Start with [lesson 00](tutorial/00-orientation.md) and work through the lessons in order. Use this repository as your source directory, keep compiled programs and simulation runs under `build/`, and retain separate run directories for comparisons. You will first run the supplied fluid example, then add a kinetic-energy calculation and visualize the results.

| Lesson | What you will do |
| --- | --- |
| [00. Find your way around](tutorial/00-orientation.md) | Configure and build the example, identify its files, and follow how the application starts |
| [01. Run a fluid simulation](tutorial/01-first-run.md) | Run the fluid example and change viscosity through its input file |
| [02. Compute kinetic energy](tutorial/02-kinetic-energy.md) | Add a calculation using patch loops, then use the documented `dot` operation |
| [03. See what the flow is doing](tutorial/03-first-visualization.md) | Open output in VisIt, compare runs, and save figures |

These are the currently written lessons. More lessons and their examples will be added as the tutorial develops.

## Before you begin

You need an installed IBAMR with 2D and Silo support, CMake 3.15 or newer, an MPI compiler environment matching that installation, and VisIt. The example applications link to installed IBAMR; you do not need its source checkout or original build directory. Basic familiarity with shell commands and C++ syntax is assumed.

Set the installation path in your terminal:

```bash
export IBAMR_ROOT=$HOME/sfw/ibamr/linux-dbg
```

Replace `$HOME/sfw/ibamr/linux-dbg` with your installation directory. It may have been installed using autoibamr or built and installed separately. Lesson 00 explains the remaining configuration and build commands, including matching the build type to your installation.

## Repository layout

- `tutorial/` contains the lessons and their figures.
- `examples/navier_stokes/ex1/` contains the starting fluid application, input, and CMake build instructions.
- `build/` is created when you configure the examples and holds compiled programs and run files. It is ignored by Git.

Lesson 02 walks you through making the source changes yourself. Completed solution applications are not included.

The example source and input come from the [IBAMR 0.20.0-pre development version](https://github.com/IBAMR/IBAMR/tree/cb650c845f2a78f0afaf0221b055250ce79da8ea). See [UPSTREAM.json](UPSTREAM.json) for provenance and [COPYRIGHT](COPYRIGHT) for the upstream license. Other IBAMR releases still need compatibility checks. For the external build interface, see the [IBAMR application documentation](https://ibamr.github.io/applications).
