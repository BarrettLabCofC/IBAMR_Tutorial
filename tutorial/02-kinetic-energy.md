# Lesson 02: Compute kinetic energy

Compute kinetic energy by looping over grid levels, patches, and side indices. Then consult the side-operations documentation and use `dot` to perform the same calculation.

## Before you start

Complete [tutorial 01](01-first-run.md), including the baseline and higher-viscosity runs. Keep `EXAMPLES_SRC` and `EXAMPLES_BUILD` set in your terminal. Source changes go in `examples/navier_stokes/ex1` under the source directory; the new executable and run files go in the corresponding build directory.

## The quantity to compute

For the constant-density, two-dimensional case, compute

$$
E(t)=\frac{\rho}{2}\int_\Omega\left(u_x^2+u_y^2\right)\,dA.
$$

This is kinetic energy per unit depth. The density comes from the same input entry used by the fluid solver:

```text
RHO = 1.0
```

The first implementation reads the velocity values directly and accumulates their contributions. This activity uses the staggered solver selected in tutorial 01.

## Compute energy by looping over patches

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

IBAMR uses a `Variable` to identify a quantity, such as velocity, and a `VariableContext` to identify which stored version of that quantity to use. Here, `getVelocityVariable()` and `getCurrentContext()` select the current velocity. `VariableDatabase` maps this pair to `u_idx`, which each patch uses to locate its velocity data.

The next lookup obtains `wgt_idx`, the data index for the side-centered integration weights maintained by `HierarchyMathOps`. These weights include the integration volumes and account for shared patch boundaries and coarse/fine coverage.

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

### Build and run the manual version

Append this block to `examples/navier_stokes/ex1/CMakeLists.txt` in the source directory:

```cmake
add_executable(navier_stokes-ex1-energy-2d kinetic_energy.cpp)
target_link_libraries(navier_stokes-ex1-energy-2d PRIVATE IBAMR::IBAMR2d)
set_target_properties(navier_stokes-ex1-energy-2d PROPERTIES OUTPUT_NAME main2d_energy)
```

This adds a build target for `kinetic_energy.cpp` and names its executable `main2d_energy`. Regenerate the example repository’s build configuration to register the new target, then compile it:

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

## Look at the side-operations documentation

The [IBAMR API documentation](https://ibamr.github.io/IBAMR-docs/ibamr/) describes class methods, their arguments, and default values. For this calculation, open the [`HierarchyDataOpsReal` class reference](https://ibamr.github.io/IBAMR-docs/ibamr/classSAMRAI_1_1math_1_1HierarchyDataOpsReal.html). It defines the common operations that `HierarchySideDataOpsReal` implements for side-centered data, including methods for computing useful measurements. In particular, read the [`dot` method description](https://ibamr.github.io/IBAMR-docs/ibamr/classSAMRAI_1_1math_1_1HierarchyDataOpsReal.html#a3c304a331cc9e8ad1894850e1c948ba0). We can use this method as an alternative to looping over patches manually. Match the documented arguments to the values we already have:

| Argument | Used here | Meaning |
| --- | --- | --- |
| `data1_id` | `u_idx` | First field's data index |
| `data2_id` | `u_idx` | Second field's data index; using velocity twice gives its square |
| `vol_id` | `wgt_idx` | Integration weights; the default `-1` would leave the sum unweighted |
| `local_only` | Default `false` | Combine contributions across MPI processes |

Use these method descriptions to check which data indices an operation expects and whether it already performs the parallel reduction.

## Use dot to compute the same energy

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

`dot` also performs the MPI reduction because `local_only` defaults to `false`. Its result is already global; do not apply another `sumReduction`. Every process still calls the function. The manual version shows the individual data-access and accumulation steps that this hierarchy operation handles.

### Compare the two implementations

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

This lesson computed kinetic energy by looping over levels, patches, and side indices, then used the documented `dot` operation to perform the same weighted calculation. You should have the modified `kinetic_energy.cpp`, the rebuilt `main2d_energy` executable, a `kinetic_energy_manual.curve` file, and a comparison run's `stdout.log` containing both manual and `dot` results at matching times. Keep these results with their run inputs and visualization output.

In [tutorial 03](03-first-visualization.md), you will open the fluid output in VisIt, select fields and saved times, and compare the baseline and higher-viscosity runs.
