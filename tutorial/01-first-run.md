# Lesson 01: Run a fluid simulation

Build and run a small two-dimensional fluid example, check its progress, and locate its output. The shear-layer model provides a starting case for learning the input and visualization workflow.

## Before you start

Complete [lesson 00](00-orientation.md). Keep its terminal open so that `EXAMPLES_SRC` and `EXAMPLES_BUILD` remain set. This activity uses `examples/navier_stokes/ex1` in the source and build directories.

## The starting model

In the supplied `input2d`, find `L` near the top:

```text
L = 1.0
```

Then find the domain geometry block:

```text
CartesianGeometry {
   domain_boxes = [ (0,0),(N - 1,N - 1) ]
   x_lo = 0,0
   x_up = L,L
   periodic_dimension = 1,1
}
```

`x_lo` and `x_up` set the physical bounds. With `L = 1.0`, the domain is the square $0\leq x,y\leq1$. `domain_boxes` gives the grid-index bounds, here `N` cells in each direction. The two entries in `periodic_dimension` enable periodicity in $x$ and $y$: material leaving one side re-enters through the opposite side. These settings give periodic boundaries rather than walls, an inlet, or an outlet.

The horizontal velocity changes direction across thin layers near $y=0.25$ and $y=0.75$. A small vertical disturbance starts their motion away from a perfectly straight arrangement. Find this block in the supplied `input2d`:

```text
VelocityInitialConditions {
   rho = 80.0
   delta = 0.05
   function_0 = "X1 < 0.5 ? tanh(rho*(X1-0.25)) : tanh(rho*(0.75-X1))"
   function_1 = "delta*sin(2.0*pi*(X0+0.25))"
}
```

`X0` and `X1` mean $x$ and $y$. `function_0` is horizontal velocity and `function_1` is vertical velocity. The expression before `?` selects the first horizontal formula below the middle of the square and the second above it. The local `rho = 80.0` controls layer sharpness; it is distinct from the top-level fluid density `RHO = 1.0`.

## Build and run the starting case

First build this example using the example repository’s configured build. You can run this command from any working directory:

```bash
cmake --build "$EXAMPLES_BUILD" --target navier_stokes-ex1-2d --parallel 4
```

The target is the name CMake uses for building; the resulting executable is `examples/navier_stokes/ex1/main2d` in the build directory. If the build reports an error, resolve it before continuing.

Create a fresh run subdirectory under the example's build directory and copy in the supplied input. If `first-flow` already exists, choose a new name rather than overwrite an earlier run.

```bash
cd "$EXAMPLES_BUILD/examples/navier_stokes/ex1"
mkdir first-flow
cp "$EXAMPLES_SRC/examples/navier_stokes/ex1/input2d" first-flow/
cd first-flow
```

Open this run directory's `input2d` in your editor. Make the following four changes for a short, single-level run. Keep the other settings unchanged.

| Setting | Supplied value | Value for this run |
| --- | --- | --- |
| `MAX_LEVELS` | `3` | `1` |
| `N` | `16` | `64` |
| `END_TIME` | `2.0` | `1.0` |
| `Main.viz_dump_interval` | `NFINEST/8` | `10` |

`Main.viz_dump_interval` means the `viz_dump_interval` entry inside the `Main` block. Save the input, then run from the same `first-flow` directory:

```bash
../main2d input2d
```

This uses one MPI process. The program reads `input2d` in the current directory and writes its output there. Look for repeated time-step messages. The last completed step should report simulation time `1`; the step number can vary with numerical settings.

After the run, check the files in this same directory:

```bash
ls
tail -n 15 INS2d.log
head -n 4 viz_INS2d/dumps.visit
```

`INS2d.log` contains the application log. `viz_INS2d` contains saved fluid fields. `dumps.visit` is a small list telling VisIt where the saved states are; keep the entire visualization directory. Do not open just one numbered state if you want the time sequence. The reference run reached $t=1$, wrote 30 saved states, and produced about 2.8 MB in the run directory.

## How it works: from input to fluid solver

Keep `examples/navier_stokes/ex1/example.cpp` from the source directory open alongside the run's `input2d` from the build directory. The excerpts below come from that existing application. Read them in place; they omit surrounding setup and are not standalone programs. This source-reading activity does not require editing or rebuilding anything.

### Follow a viscosity value into the solver

The input selects `SOLVER_TYPE = "STAGGERED"`. The application constructs `INSStaggeredHierarchyIntegrator`, which stores pressure at cell centers and velocity components at cell faces. This layout supports the discrete pressure and incompressibility calculation. The file also contains a collocated-solver block; it is not the selected solver in this run.

The connection is explicit in C++. Inside the `STAGGERED` branch, find:

```cpp
time_integrator = new INSStaggeredHierarchyIntegrator(
    "INSStaggeredHierarchyIntegrator",
    app_initializer->getComponentDatabase("INSStaggeredHierarchyIntegrator"));
```

The first argument names this solver object. The second supplies its input database. Inside that input block, `mu = MU` connects the top-level value to the solver's viscosity setting. The path is therefore `MU` → the block's `mu` entry → the constructed fluid integrator. Top-level names such as `MU` are conveniences chosen by this input file; a solver does not automatically read every top-level setting.

The selected block receives `mu = MU`, `rho = RHO`, and the time settings. `CFL_MAX = 0.3` limits how far flow moves relative to a cell during a step, while `DT_MAX = 0.5/NFINEST` gives an upper bound on step size. The integrator chooses the actual step, so output every ten steps does not necessarily mean equally spaced physical times. The input selects `ADAMS_BASHFORTH` for the convection term. The viscous term uses the default `TRAPEZOIDAL_RULE` (Crank–Nicolson), since this input does not specify `viscous_time_stepping_type`.

The default is defined in IBAMR’s `INSHierarchyIntegrator.h` (installed under `$IBAMR_ROOT/include/ibamr`):

```cpp
TimeSteppingType d_viscous_time_stepping_type = TRAPEZOIDAL_RULE;
```

### Supply the initial field, then initialize it

Find the initial-condition setup:

```cpp
Pointer<CartGridFunction> u_init = new muParserCartGridFunction(
    "u_init", app_initializer->getComponentDatabase("VelocityInitialConditions"), grid_geometry);
time_integrator->registerVelocityInitialConditions(u_init);
```

The expression object receives the velocity database and domain geometry. The registration call tells the integrator which object will supply initial velocity. Registration connects a provider to a solver; it does not itself advance time. The later `initializePatchHierarchy` call builds the initial hierarchy and initializes its data using the registered setup.

This separation lets the fluid solver request velocity data without knowing whether the application obtains it from expressions or custom C++. For an initial field that needs more than prescribed expressions, the extension is a custom `IBTK::CartGridFunction`, providing `setDataOnPatch`. `setDataOnPatch` receives the destination data index, variable, patch, requested time, and initialization flag, along with an optional level pointer. Its job is to fill the requested patch data in the variable's layout. The same `registerVelocityInitialConditions` call connects the replacement. We will implement a provider later when adding forces.

### Build the grid and advance its data

The solver works over a `PatchHierarchy`: grid cells grouped into rectangular patches and refinement levels. Here `N = 64` and `MAX_LEVELS = 1` give a single level with 64 cells per direction. Later, smaller cells can cover selected regions without refining the whole domain. The application's `advanceHierarchy` call advances all the data it manages; you do not write a separate loop over cells in the input file.

Follow these objects through the setup in `example.cpp`:

| Object | What this application asks it to do |
| --- | --- |
| `CartesianGridGeometry` | Describe the physical bounds and periodic directions |
| `PatchHierarchy` | Hold the grid levels, their patches, and associated data |
| `StandardTagAndInitialize` | Connect initialization/refinement requests to the integrator |
| `BergerRigoutsos` | Group cells marked for refinement into rectangular boxes |
| `LoadBalancer` | Distribute patch work among processes |
| `GriddingAlgorithm` | Assemble the hierarchy using those helpers and the grid input |

The application connects the hierarchy and gridding algorithm with:

```cpp
time_integrator->initializePatchHierarchy(patch_hierarchy, gridding_algorithm);
```

Our one-level run still uses this infrastructure, but creates no finer levels. `REF_RATIO` only affects finer levels when they exist. The same application setup can therefore support a uniform teaching grid and a later locally refined run.

`periodic_dimension = 1,1` specifies the two periodic directions. The application recognizes this geometry and does not register physical-boundary coefficient objects for these directions. A later cylinder lesson introduces actual inlet/outlet coefficients and their meanings.

Now find these lines inside the time loop:

```cpp
iteration_num = time_integrator->getIntegratorStep();
loop_time = time_integrator->getIntegratorTime();
dt = time_integrator->getMaximumTimeStepSize();
time_integrator->advanceHierarchy(dt);
loop_time += dt;
```

The first two calls read the current step and physical time. The application then asks for an allowed time step and passes it to `advanceHierarchy`. That call performs the integrator's time advance, including its configured numerical operations and solver calls. The local `loop_time` is updated for the application's reporting and output. Adding a number to that local variable alone would not update the fluid state.

The application's `while` condition checks the end time and `stepsRemaining()`. The numerical work stays inside the integrator, while this outer loop handles progress messages and decides when to save data. For an IB application, an IB hierarchy integrator will occupy this coordinating role and receive both a fluid integrator and a structure strategy.

Trace `END_TIME` through the input's `end_time` entry, then locate `getEndTime()` in the application. This shows where the application reads the integrator's stopping time; `advanceHierarchy(dt)` performs the actual update of the fields.

## Prepare one change

To practice changing an input parameter and keeping results separate, increase viscosity tenfold. Keep the baseline and make a second run directory:

```bash
cd "$EXAMPLES_BUILD/examples/navier_stokes/ex1"
mkdir first-flow-viscous
cp first-flow/input2d first-flow-viscous/
```

Open `first-flow-viscous/input2d` in the example's build directory. Change only the top-level viscosity assignment from

```text
MU  = 0.0001
```

to

```text
MU  = 0.001
```

Keep density, grid, final time, and all other values unchanged. No rebuild is needed. Run the same program from the new directory:

```bash
cd "$EXAMPLES_BUILD/examples/navier_stokes/ex1/first-flow-viscous"
../main2d input2d
```

The comparison should also reach $t=1$. In [lesson 03](03-first-visualization.md), compare the two flows at that same physical time.

If the program says it cannot open `input2d`, check `pwd` and `ls`: run from the directory containing the copied input. If the visualization directory is missing, inspect the log before moving on.

## Before moving on

This lesson built and ran the fluid example, followed how input settings reach the solver, and changed viscosity without recompiling. You should have separate `first-flow` and `first-flow-viscous` run directories, each containing its input, `INS2d.log`, and complete `viz_INS2d` output directory with a `dumps.visit` file.

In [tutorial 02](02-kinetic-energy.md), you will add a kinetic-energy calculation to this application. You will access the velocity data and save a measurement after each time step.
