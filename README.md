# Learning IBAMR

A guided introduction to building, running, and modifying IBAMR applications. The tutorial is for readers familiar with the basic fluid and immersed boundary equations who want practical experience using the software. Each lesson combines a hands-on task with explanations of the input settings and C++ code involved.

## Prerequisites

This tutorial assumes basic familiarity with using the terminal and C++ syntax. Before starting, have the following installed:

- **IBAMR**, installed either with autoibamr or manually, with 2D and Silo support for the current lessons.
- **VisIt**, for opening simulation output and saving figures.
- **Git**, to clone this repository locally.
- **CMake 3.15 or newer** and the compiler/MPI environment matching your IBAMR installation.

The example applications link to your installed IBAMR; you do not need its source checkout or original build directory.

## Get the tutorial

Clone this repository onto your machine and enter its top-level directory:

```bash
git clone https://github.com/BarrettLabCofC/IBAMR_Tutorial.git
cd IBAMR_Tutorial
```

This gives you the tutorial Markdown, figures, and starting example files. Run the setup commands in lesson 00 from this directory.

Set the IBAMR installation path in your terminal:

```bash
export IBAMR_ROOT=$HOME/sfw/ibamr/linux-dbg
```

Replace `$HOME/sfw/ibamr/linux-dbg` with your installation directory, whether installed using autoibamr or manually. Lesson 00 explains the configuration and build commands, including matching the build type to your installation.

## Using the tutorial

Start with [lesson 00](tutorial/00-orientation.md) and work through the lessons in order. Use this repository as your source directory, keep compiled programs and simulation runs under `build/`, and retain separate run directories for comparisons.

| Lesson | What you will do |
| --- | --- |
| [00. Find your way around](tutorial/00-orientation.md) | Configure and build the example, identify its files, and follow how the application starts |
| [01. Run a fluid simulation](tutorial/01-first-run.md) | Run the fluid example and change viscosity through its input file |
| [02. Compute kinetic energy](tutorial/02-kinetic-energy.md) | Add a calculation using patch loops, then use the documented `dot` operation |
| [03. See what the flow is doing](tutorial/03-first-visualization.md) | Open output in VisIt, compare runs, and save figures |
| [04. Put an elastic rubber band in the fluid](tutorial/04-elastic-loop.md) | Run an elastic rubber band and view it together with the fluid |
| [05. Change the model through its input](tutorial/05-input-change.md) | Change rubber-band stiffness through the input and compare results at the same physical time |

These are the currently written lessons. More lessons and their examples will be added as the tutorial develops.

## Repository layout

- `tutorial/` contains the lessons and their figures.
- `examples/` contains the application source, inputs, structure data, and CMake build instructions used by the tutorials.
- `build/` is created when you configure the examples and holds compiled programs and run files. It is ignored by Git.

The example source and input come from the [IBAMR 0.20.0-pre development version](https://github.com/IBAMR/IBAMR/tree/cb650c845f2a78f0afaf0221b055250ce79da8ea). See [UPSTREAM.json](UPSTREAM.json) for provenance and [COPYRIGHT](COPYRIGHT) for the upstream license. Other IBAMR releases still need compatibility checks. For the external build interface, see the [IBAMR application documentation](https://ibamr.github.io/applications).
