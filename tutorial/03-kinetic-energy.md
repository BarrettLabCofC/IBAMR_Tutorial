# Lesson 03: Compute kinetic energy

## Introduction

Follow the setup and time loop of the fluid application you have already run and visualized. Then compute kinetic energy by looping over grid levels, patches, and side indices, consult the side-operations documentation, and use `dot` to perform the same calculation.

## Before you start

Complete [tutorial 02](02-first-visualization.md), using the baseline and higher-viscosity runs from tutorial 01. Keep `EXAMPLES_SRC` and `EXAMPLES_BUILD` set in your terminal. Source changes go in `examples/navier_stokes/ex1` under the source directory; the new executable and run files go in the corresponding build directory.

## Start with the existing result

Use the completed fluid run from tutorial 01 as the starting example. Before editing its source, check the output in the build directory:

```bash
cd "$EXAMPLES_BUILD/examples/navier_stokes/ex1/first-flow"
tail -n 15 INS2d.log
head -n 4 viz_INS2d/dumps.visit
```

You already have the simulated velocity field and its saved time sequence. This lesson adds a measurement of that velocity to the program. The energy calculation must be implemented before you can build and run the new executable; the original fluid application remains available throughout.

## Follow the existing fluid application

The flow you ran in tutorial 01 and viewed in tutorial 02 now provides a concrete example for reading the program. Follow its model settings, initialization, and time advance before adding the measurement.

### The starting model

In the supplied `input2d`, find `L` near the top:

```text
L = 1.0
```

The tutorial 01 run uses `N = 64` cells per direction. Find the domain geometry block:

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

Read the complete [fluid application](../examples/navier_stokes/ex1/example.cpp) alongside its [input file](../examples/navier_stokes/ex1/input2d). The excerpts below come from that existing application. Read them in place; they omit surrounding setup and are not standalone programs. This source-reading activity does not require editing or rebuilding anything.

### Follow a viscosity value into the solver

The input selects `SOLVER_TYPE = "STAGGERED"`. The application constructs `INSStaggeredHierarchyIntegrator`, which stores pressure at cell centers and velocity components at cell faces. This layout supports the discrete pressure and incompressibility calculation. The file also contains a collocated-solver block; it is not the selected solver in this run.

Inside the `STAGGERED` branch, find:

```cpp
time_integrator = new INSStaggeredHierarchyIntegrator(
    "INSStaggeredHierarchyIntegrator",
    app_initializer->getComponentDatabase("INSStaggeredHierarchyIntegrator"));
```

The first argument names this solver object. The second supplies its input database. Inside that input block, `mu = MU` connects the top-level value to the solver's viscosity setting. The application chooses which database to pass into this constructor, so the input-block name must match the string in the lookup. The solver object then reads the entries it understands from that block. The path is therefore `MU` → the block's `mu` entry → the constructed fluid integrator. Top-level names such as `MU` are conveniences chosen by this input file; a solver does not automatically read every top-level setting.

The constructor configures the integrator before the hierarchy exists. It does not yet allocate a velocity field or advance a time step. Those operations happen during initialization and advancement below.

The relevant top-level assignments are:

```text
RHO = 1.0
CFL_MAX = 0.3
DT_MAX = 0.5/NFINEST
CONVECTIVE_TS_TYPE = "ADAMS_BASHFORTH"
```

Inside `INSStaggeredHierarchyIntegrator`, find the corresponding entries:

```text
mu = MU
rho = RHO
cfl = CFL_MAX
dt_max = DT_MAX
convective_time_stepping_type = CONVECTIVE_TS_TYPE
```

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

The expression object receives the velocity database and domain geometry. The registration call tells the integrator which object will supply initial velocity. The three constructor arguments name the provider, supply its expression database, and supply the physical grid geometry. `function_0` and `function_1` are interpreted as components of the requested velocity field. The provider can evaluate those expressions when the integrator asks it to initialize a patch.

Registration connects a provider to a solver; it does not itself advance time. The later `initializePatchHierarchy` call builds the initial hierarchy and initializes its data using the registered setup.

This separation lets the fluid solver request velocity data without knowing whether the application obtains it from expressions or custom C++. For an initial field that needs more than prescribed expressions, the extension is a custom `IBTK::CartGridFunction`, providing `setDataOnPatch`. `setDataOnPatch` receives the destination data index, variable, patch, requested time, and initialization flag, along with an optional level pointer. Its job is to fill the requested patch data in the variable's layout. The same `registerVelocityInitialConditions` call connects the replacement. We will implement a provider later when adding forces.

### Build the grid and advance its data

The solver works over a `PatchHierarchy`: grid cells grouped into rectangular patches and refinement levels. The tutorial 01 run uses `N = 64` and `MAX_LEVELS = 1` to give a single level with 64 cells per direction. Later, smaller cells can cover selected regions without refining the whole domain. The application's `advanceHierarchy` call advances all the data it manages; you do not write a separate loop over cells in the input file.

The geometry and hierarchy are connected explicitly:

```cpp
Pointer<CartesianGridGeometry<NDIM>> grid_geometry = new CartesianGridGeometry<NDIM>(
    "CartesianGeometry", app_initializer->getComponentDatabase("CartesianGeometry"));
Pointer<PatchHierarchy<NDIM>> patch_hierarchy = new PatchHierarchy<NDIM>("PatchHierarchy", grid_geometry);
```

The geometry reads physical bounds and periodic directions from its database. The hierarchy receives that geometry and will hold the levels, patches, and numerical data created during initialization. Constructing these objects establishes their relationship; the later initialization call builds the actual grid and fills the initial fields.

Follow the remaining grid helpers through the setup in `example.cpp`:

| Object | What this application asks it to do |
| --- | --- |
| `CartesianGridGeometry` | Describe the physical bounds and periodic directions |
| `PatchHierarchy` | Hold the grid levels, their patches, and associated data |
| `StandardTagAndInitialize` | Connect initialization/refinement requests to the integrator |
| `BergerRigoutsos` | Group cells marked for refinement into rectangular boxes |
| `LoadBalancer` | Distribute patch work among processes |
| `GriddingAlgorithm` | Assemble the hierarchy using those helpers and the grid input |

`StandardTagAndInitialize` receives `time_integrator`, which lets grid creation ask the solver how to initialize data and identify cells for refinement. The `GriddingAlgorithm` then receives the input block and the three helpers:

```cpp
Pointer<GriddingAlgorithm<NDIM>> gridding_algorithm =
    new GriddingAlgorithm<NDIM>("GriddingAlgorithm",
                                app_initializer->getComponentDatabase("GriddingAlgorithm"),
                                error_detector,
                                box_generator,
                                load_balancer);
```

The arguments keep separate responsibilities together: `error_detector` connects the solver's requests, `box_generator` forms rectangular patch boxes, and `load_balancer` assigns the work. The application supplies these objects once; grid construction uses them as needed.

The application connects the hierarchy and gridding algorithm with:

```cpp
time_integrator->initializePatchHierarchy(patch_hierarchy, gridding_algorithm);
```

At this call the gridding algorithm creates patches and the integrator initializes their data using providers such as `u_init`. This is the point at which the configured model becomes a stored numerical state on a grid.

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

## The input and program

Read the supplied [fluid application](../examples/navier_stokes/ex1/example.cpp) as the starting point. In this lesson you will create a source copy and add a complete diagnostic function to it.

### The quantity to compute

For the constant-density, two-dimensional case, compute

$$
E(t)=\frac{\rho}{2}\int_\Omega\left(u_x^2+u_y^2\right)\,dA.
$$

This is kinetic energy per unit depth. The density comes from the same input entry used by the fluid solver:

```text
RHO = 1.0
```

The first implementation reads the velocity values directly and accumulates their contributions. This activity uses the staggered solver described above.

### Create a version of the example to edit

In the source directory, copy the application's source:

```bash
cd "$EXAMPLES_SRC/examples/navier_stokes/ex1"
cp example.cpp kinetic_energy.cpp
```

Make the following changes in `kinetic_energy.cpp`. Add these headers beside the existing includes, before `app_namespaces.h`:

```cpp
#include <Patch.h>
#include <PatchLevel.h>
#include <SideData.h>
#include <SideIterator.h>
#include <ibtk/HierarchyMathOps.h>
#include <iomanip>
```

The patch and side-data headers provide the types used in the loops. `HierarchyMathOps` supplies integration weights, and `<iomanip>` supplies the output precision setting.

Add this declaration beside the existing function prototypes, before `main`:

```cpp
double compute_kinetic_energy_manual(Pointer<PatchHierarchy<NDIM>> patch_hierarchy,
                                     Pointer<INSHierarchyIntegrator> ins_integrator,
                                     double rho);
```

The function receives the hierarchy containing the grid, the fluid integrator identifying its velocity state, and the constant density. It returns one number, the total kinetic energy; it does not modify the velocity. Passing the integrator lets the function find its data through the public interface rather than assume a fixed data index.

### Retrieve the patch data and accumulate energy

Add the complete function at the end of the file, after the existing `output_data` function:

```cpp
double
compute_kinetic_energy_manual(Pointer<PatchHierarchy<NDIM>> patch_hierarchy,
                              Pointer<INSHierarchyIntegrator> ins_integrator,
                              const double rho)
{
    auto* var_db = VariableDatabase<NDIM>::getDatabase();
    const int u_idx = var_db->mapVariableAndContextToIndex(
        ins_integrator->getVelocityVariable(), ins_integrator->getCurrentContext());
    const int wgt_idx = ins_integrator->getHierarchyMathOps()->getSideWeightPatchDescriptorIndex();

    double local_energy = 0.0;
    for (int ln = 0; ln <= patch_hierarchy->getFinestLevelNumber(); ++ln)
    {
        Pointer<PatchLevel<NDIM>> level = patch_hierarchy->getPatchLevel(ln);
        for (PatchLevel<NDIM>::Iterator p(level); p; p++)
        {
            Pointer<Patch<NDIM>> patch = level->getPatch(p());
            Pointer<SideData<NDIM, double>> u_data = patch->getPatchData(u_idx);
            Pointer<SideData<NDIM, double>> weights = patch->getPatchData(wgt_idx);

            for (int axis = 0; axis < NDIM; ++axis)
            {
                for (SideIterator<NDIM> s(patch->getBox(), axis); s; s++)
                {
                    const SideIndex<NDIM>& side = s();
                    const double u = (*u_data)(side);
                    const double weight = (*weights)(side);
                    local_energy += 0.5 * rho * u * u * weight;
                }
            }
        }
    }
    return IBTK_MPI::sumReduction(local_energy);
}
```

IBAMR uses a `Variable` to identify a quantity, such as velocity, and a `VariableContext` to identify which stored version of that quantity to use. Here, `getVelocityVariable()` retrieves the variable for the fluid velocity and `getCurrentContext()` select the current data. `VariableDatabase` maps this pair to `u_idx`, which is a patch data index that each patch uses to locate its velocity data. 

The next lookup obtains `wgt_idx`, the data index for the side-centered integration weights maintained by `HierarchyMathOps`. These weights include the integration volumes and account for shared patch boundaries and coarse/fine coverage.

Coarse cells covered by finer cells have zero integration weight, so the sum counts the represented fluid volume once. This remains important when the same function is used on a refined hierarchy, even though the present example has one level.

The outer loop selects each level with `getPatchLevel(ln)`. `PatchLevel::Iterator` then visits the patches owned by this MPI process. On each patch, `getPatchData(u_idx)` retrieves the velocity arrays, and `getPatchData(wgt_idx)` retrieves the weights. `SideData<NDIM, double>` identifies their storage layout and numerical type.

The `axis` loop selects a velocity component: axis `0` visits locations storing $u_x$, and axis `1` visits those storing $u_y$. `SideIterator` enumerates the side indices associated with the patch's interior box for that component. `s()` returns a `SideIndex` containing a grid location and axis. `(*u_data)(side)` reads the velocity there.

At each index, add `rho*u*u*weight/2` to `local_energy`. Summing over axes supplies the squared components in the energy formula. The weight already includes the volume contribution, so do not multiply by cell area again or include the ghost region. This calculation reads the current state without changing it and needs no spatial derivatives or additional ghost-cell fill.

The loops accumulate this process's contribution. `IBTK_MPI::sumReduction` adds contributions from all processes and returns the global energy. Every process must call this function.

### Call the function after initialization and each step

In `main`, immediately after the line that gets `input_db`, read the density:

```cpp
const double rho = input_db->getDouble("RHO");
```

After the declaration of `solver_type`, before the solver-selection `if` statement, add this check for the layout used by the function:

```cpp
if (solver_type != "STAGGERED")
{
    TBOX_ERROR("The kinetic-energy example requires SOLVER_TYPE = STAGGERED.\n");
}
```

Find `// Main time step loop.`. Immediately before that comment, add:

```cpp
const double initial_manual_energy = compute_kinetic_energy_manual(patch_hierarchy, time_integrator, rho);
pout << "KINETIC_ENERGY_MANUAL " << loop_time << " " << std::setprecision(17) << initial_manual_energy << "\n";
```

The hierarchy and initial velocity have been initialized at this point. Inside the time loop, immediately after `loop_time += dt;`, add:

```cpp
const double manual_energy = compute_kinetic_energy_manual(patch_hierarchy, time_integrator, rho);
pout << "KINETIC_ENERGY_MANUAL " << loop_time << " " << manual_energy << "\n";
```

The preceding `advanceHierarchy(dt)` call has completed the step, so the current context holds updated velocity. Each output line contains a label, physical time, and global energy. SAMRAI's `pout` prints on the primary process while the calculation runs on all processes.

## Running the example

Append this block to `examples/navier_stokes/ex1/CMakeLists.txt` in the source directory:

```cmake
add_executable(navier_stokes-ex1-energy-2d kinetic_energy.cpp)
target_link_libraries(navier_stokes-ex1-energy-2d PRIVATE IBAMR::IBAMR2d)
set_target_properties(navier_stokes-ex1-energy-2d PROPERTIES OUTPUT_NAME main2d_energy)
```

`add_executable` creates a separate target from the source copy, so the original fluid application remains available. `target_link_libraries` supplies the same installed 2D IBAMR interface as before. `OUTPUT_NAME` names the new executable `main2d_energy`; use the target name for building and the output name for running. Regenerate the example repository’s build configuration to register the new target, then compile it:

```bash
cmake -S "$EXAMPLES_SRC" -B "$EXAMPLES_BUILD"
cmake --build "$EXAMPLES_BUILD" --target navier_stokes-ex1-energy-2d --parallel 4
```

Create a fresh run directory under the example's build directory and copy the baseline input from tutorial 01:

```bash
cd "$EXAMPLES_BUILD/examples/navier_stokes/ex1"
mkdir first-flow-energy
cp first-flow/input2d first-flow-energy/
cd first-flow-energy
../main2d_energy input2d > stdout.log 2>&1
tail -n 10 stdout.log
```

## Results

The redirection saves the program's messages in `stdout.log`. Extract the energy lines into a two-column file containing time and energy:

```bash
awk '/^KINETIC_ENERGY_MANUAL / { print $2, $3 }' stdout.log > kinetic_energy_manual.curve
head -n 3 kinetic_energy_manual.curve
tail -n 3 kinetic_energy_manual.curve
```

Check that the first row is at time `0` and the last at time `1`. Keep this file with the run's input, log, and visualization output. To collect the higher-viscosity case, repeat these commands in a fresh `first-flow-viscous-energy` run directory, copying its input from `first-flow-viscous/input2d`.

The checked runs give these reference values:

| Run input | Energy at time `0` | Energy at time `1` |
| --- | --- | --- |
| Baseline, `MU = 0.0001` | `0.475918774876` | `0.457566778945` |
| Comparison, `MU = 0.001` | `0.475918774876` | `0.396848493426` |

Both runs start with the same velocity and density, so their initial energies match. These values provide a check on the function and the selected input; they are results for this grid and run, rather than a numerical-accuracy benchmark.

## Further exploration: use a hierarchy operation

Continue with the same source file to compare your manual calculation with the library operation.

### Look at the side-operations documentation

The [IBAMR API documentation](https://ibamr.github.io/IBAMR-docs/ibamr/) describes class methods, their arguments, and default values. For this calculation, open the [`HierarchyDataOpsReal` class reference](https://ibamr.github.io/IBAMR-docs/ibamr/classSAMRAI_1_1math_1_1HierarchyDataOpsReal.html). It defines the common operations that `HierarchySideDataOpsReal` implements for side-centered data, including methods for computing useful measurements. In particular, read the [`dot` method description](https://ibamr.github.io/IBAMR-docs/ibamr/classSAMRAI_1_1math_1_1HierarchyDataOpsReal.html#a3c304a331cc9e8ad1894850e1c948ba0). We can use this method as an alternative to looping over patches manually. Match the documented arguments to the values we already have:

| Argument | Used here | Meaning |
| --- | --- | --- |
| `data1_id` | `u_idx` | First field's data index |
| `data2_id` | `u_idx` | Second field's data index; using velocity twice gives its square |
| `vol_id` | `wgt_idx` | Integration weights; the default `-1` would leave the sum unweighted |
| `local_only` | Default `false` | Combine contributions across MPI processes |

Use these method descriptions to check which data indices an operation expects and whether it already performs the parallel reduction.

### Use dot to compute the same energy

Add the side-operations header before `app_namespaces.h`:

```cpp
#include <HierarchySideDataOpsReal.h>
```

Add this declaration beside the manual function's prototype:

```cpp
double compute_kinetic_energy(Pointer<PatchHierarchy<NDIM>> patch_hierarchy,
                              Pointer<INSHierarchyIntegrator> ins_integrator,
                              double rho);
```

Keep the manual function for comparison and add this second function at the end of the file:

```cpp
double
compute_kinetic_energy(Pointer<PatchHierarchy<NDIM>> patch_hierarchy,
                       Pointer<INSHierarchyIntegrator> ins_integrator,
                       const double rho)
{
    auto* var_db = VariableDatabase<NDIM>::getDatabase();
    const auto u_var = ins_integrator->getVelocityVariable();
    const auto current_ctx = ins_integrator->getCurrentContext();
    const int u_idx = var_db->mapVariableAndContextToIndex(u_var, current_ctx);

    const int wgt_idx = ins_integrator->getHierarchyMathOps()->getSideWeightPatchDescriptorIndex();
    HierarchySideDataOpsReal<NDIM, double> side_ops(
        patch_hierarchy, 0, patch_hierarchy->getFinestLevelNumber());
    return 0.5 * rho * side_ops.dot(u_idx, u_idx, wgt_idx);
}
```

The velocity and weight lookups identify the same data used in the manual function. `side_ops` is configured to operate from level `0` through the current finest level. `dot(u_idx, u_idx, wgt_idx)` multiplies velocity by itself, applies the weights, and sums across components, patches, and levels. Multiplying by `rho/2` gives the energy.

`dot` also performs the MPI reduction because `local_only` defaults to `false`. Its result is already global; do not apply another `sumReduction`. The manual version shows the individual data-access and accumulation steps that this hierarchy operation handles.

#### Compare the two implementations

Immediately after the initial manual-energy output line, add:

```cpp
const double initial_energy = compute_kinetic_energy(patch_hierarchy, time_integrator, rho);
pout << "KINETIC_ENERGY " << loop_time << " " << std::setprecision(17) << initial_energy << "\n";
```

Inside the time loop, immediately after the manual-energy output line, add:

```cpp
const double energy = compute_kinetic_energy(patch_hierarchy, time_integrator, rho);
pout << "KINETIC_ENERGY " << loop_time << " " << std::setprecision(17) << energy << "\n";
```

Rebuild the same target and run in a fresh build subdirectory:

```bash
cmake --build "$EXAMPLES_BUILD" --target navier_stokes-ex1-energy-2d --parallel 4
cd "$EXAMPLES_BUILD/examples/navier_stokes/ex1"
mkdir first-flow-energy-dot
cp first-flow/input2d first-flow-energy-dot/
cd first-flow-energy-dot
../main2d_energy input2d > stdout.log 2>&1
grep '^KINETIC_ENERGY' stdout.log | head -n 6
```

Each time now has a `KINETIC_ENERGY_MANUAL` line from the loops and a `KINETIC_ENERGY` line from `dot`. Compare values at matching times. Small differences in the last digits can occur because the sums are accumulated in a different order.

## Before moving on

This lesson followed the fluid application from input settings through initialization and time advancement, computed kinetic energy by looping over levels, patches, and side indices, then used the documented `dot` operation to perform the same weighted calculation. You should have the modified `kinetic_energy.cpp`, the rebuilt `main2d_energy` executable, a `kinetic_energy_manual.curve` file, and a comparison run's `stdout.log` containing both manual and `dot` results at matching times. Keep these results with their run inputs and visualization output.

In [tutorial 04](04-elastic-loop.md), you will add an elastic rubber band to the fluid simulation and view the fluid and structure outputs together at matching times.
