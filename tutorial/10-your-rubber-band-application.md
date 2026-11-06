# Tutorial 10: Build your own rubber-band application

## Introduction

Create a standalone rubber-band application that generates its points and springs in C++. Run it with geometry parameters from the input, then add a calculation that tracks the area enclosed by the moving band.

## Before you start

Use the installed IBAMR environment from [tutorial 00](00-orientation.md), including `IBAMR_ROOT` and the matching compiler and MPI wrappers. Copy the starting files from the [rubber-band example](../examples/IB/explicit/ex1) using `EXAMPLES_SRC`. Your new project will have its own `CMakeLists.txt` and build directory outside the tutorial repository.

## Create a standalone project

Use a new directory outside the tutorial repository. The commands use `~/my-rubber-band`. If that directory already exists, choose another unused path for `BAND_SRC`.

```bash
export BAND_SRC="$HOME/my-rubber-band"
export BAND_BUILD="$BAND_SRC/build"
mkdir "$BAND_SRC"
cp "$EXAMPLES_SRC/examples/IB/explicit/ex1/"{example.cpp,input2d} "$BAND_SRC/"
```

Create `$BAND_SRC/CMakeLists.txt` with the following contents.

```cmake
cmake_minimum_required(VERSION 3.15)
project(MyRubberBand LANGUAGES C CXX)

find_package(IBAMR 0.19.0 EXACT CONFIG REQUIRED)
if(NOT IBAMR_HAVE_SILO)
  message(FATAL_ERROR "This tutorial requires IBAMR built with Silo for visualization.")
endif()

add_executable(my-rubber-band example.cpp)
target_link_libraries(my-rubber-band PRIVATE IBAMR::IBAMR2d)
```

`find_package` loads the installed IBAMR package. `add_executable` creates the `my-rubber-band` target from your copy of `example.cpp`. Linking to `IBAMR::IBAMR2d` gives the target the headers, libraries, and compile settings for a two-dimensional application.

## Generate the points and springs

### Put the geometry in the input

In `$BAND_SRC/input2d`, replace the whole `IBStandardInitializer { ... }` block, including its nested `curve2d_64` block, with:

```text
IBRedundantInitializer {
   max_levels = MAX_LEVELS
}

RubberBand {
   num_points = 304
   center_x = 0.5
   center_y = 0.5
   radius = 0.25
   beta = 0.35
}
```

`IBRedundantInitializer` needs the number of available hierarchy levels. `RubberBand` is a database defined by this application: it gives your C++ functions the number of points, the center, and the parameters of the ellipse. The vertical semiaxis is `beta`. The horizontal semiaxis is `alpha = radius² / beta`, so `radius` is the radius of a circle with the same area as the continuous ellipse. These settings reproduce the initial geometry from tutorial 04.

### Write the functions that supply the points and springs

In `$BAND_SRC/example.cpp`, replace `#include <ibamr/IBStandardInitializer.h>` with `#include <ibamr/IBRedundantInitializer.h>`. Add the following headers above the comment `// Set up application namespace declarations`:

```cpp
#include <algorithm>
#include <cmath>
#include <map>
#include <vector>
```

Insert this code after `#include <ibamr/app_namespaces.h>` and before `// Function prototypes`:

```cpp
namespace
{
struct RubberBand
{
    int num_points;
    double center_x;
    double center_y;
    double radius;
    double beta;
    double stiffness_scale;
};

void
generate_points(const unsigned int& /*structure_number*/,
                const int& /*level_number*/,
                int& num_vertices,
                std::vector<IBTK::Point>& positions,
                void* ctx)
{
    const auto& band = *static_cast<const RubberBand*>(ctx);
    num_vertices = band.num_points;
    positions.resize(num_vertices);
    const double alpha = band.radius * band.radius / band.beta;
    const double two_pi = 2.0 * std::acos(-1.0);
    for (int i = 0; i < num_vertices; ++i)
    {
        const double theta = two_pi * i / num_vertices;
        positions[i](0) = band.center_x + alpha * std::cos(theta);
        positions[i](1) = band.center_y + band.beta * std::sin(theta);
    }
}

void
generate_springs(
    const unsigned int& /*structure_number*/,
    const int& /*level_number*/,
    std::multimap<int, IBRedundantInitializer::Edge>& spring_map,
    std::map<IBRedundantInitializer::Edge,
             IBRedundantInitializer::SpringSpec,
             IBRedundantInitializer::EdgeComp>& spring_specs,
    void* ctx)
{
    const auto& band = *static_cast<const RubberBand*>(ctx);
    const double ds = 2.0 * std::acos(-1.0) * band.radius / band.num_points;
    const double stiffness = band.stiffness_scale / ds;
    for (int i = 0; i < band.num_points; ++i)
    {
        const int j = (i + 1) % band.num_points;
        const IBRedundantInitializer::Edge edge(std::min(i, j), std::max(i, j));
        spring_map.emplace(edge.first, edge);

        IBRedundantInitializer::SpringSpec spec;
        spec.parameters = { stiffness, 0.0 };
        spec.force_fcn_idx = 0;
        spring_specs.emplace(edge, spec);
    }
}
} // namespace
```

`RubberBand` groups the model parameters so both functions can use them. Each function receives the structure number and hierarchy level from the initializer. This application generates a single structure, so the argument names are commented out. The `ctx` argument carries the address of the `RubberBand` object supplied at registration; the first line of each function recovers it as a reference.

`generate_points` fills the two outputs passed by reference: the point count and the vector of initial positions. It places the points at evenly spaced angles around the ellipse, using `positions[i](0)` and `positions[i](1)` for the two coordinates. As with the vertex file, the order of the points gives them indices starting at zero.

`generate_springs` supplies the data that used to come from the spring file. Each point `i` connects to its successor `j`; the remainder operator `%` connects the last point back to zero. The edge stores the lower endpoint index first, and `spring_map` lists the edges by that first index. `spring_specs` associates each edge with its parameters: stiffness, zero resting length, and force-function index `0` for the standard spring law from tutorial 04.

The stiffness uses the same `K/ds` scaling as the original input. Here `ds` is the circumference of the reference circle divided by the point count. It sets the spring coefficient; it is not the actual distance between neighboring points on the initial ellipse. The 304 points reproduce the discretization used with the original 64-cell grid. When changing resolution, consider the spacing of the generated points alongside the fluid grid, as in [tutorial 06](06-refinement.md).

### Register the callbacks before initializing the hierarchy

In `main`, find the `// Configure the IB solver.` comment. Replace the construction of `IBStandardInitializer` and the following `registerLInitStrategy` call with this block. The existing `IBStandardForceGen` construction and registration follow it:

```cpp
Pointer<Database> band_db = input_db->getDatabase("RubberBand");
RubberBand band{ band_db->getInteger("num_points"),
                 band_db->getDouble("center_x"),
                 band_db->getDouble("center_y"),
                 band_db->getDouble("radius"),
                 band_db->getDouble("beta"),
                 input_db->getDouble("K") };

Pointer<IBRedundantInitializer> ib_initializer = new IBRedundantInitializer(
    "IBRedundantInitializer", input_db->getDatabase("IBRedundantInitializer"));
const int structure_level = input_db->getInteger("MAX_LEVELS") - 1;
ib_initializer->setStructureNamesOnLevel(structure_level, { "rubber_band" });
ib_initializer->registerInitStructureFunction(generate_points, &band);
ib_initializer->registerInitSpringDataFunction(generate_springs, &band);
ib_method_ops->registerLInitStrategy(ib_initializer);
```

`main` reads the required values from the input database and stores them in `band`. It reads `K` from the existing top-level input, so the stiffness scaling still uses the same model parameter as the time-step bound. `setStructureNamesOnLevel` puts the structure named `rubber_band` on level `MAX_LEVELS - 1`, level zero for this input. That name identifies the generated structure in the simulation and in its output.

The two callback registrations pass the functions and the address `&band` to the initializer. Then `registerLInitStrategy` connects the initializer to `IBMethod` and triggers its setup, which calls the functions to collect the starting coordinates and connectivity. Each MPI process generates the same structure description; hierarchy initialization uses that information to place points on the patches that own them. The `band` object in `main` is still in scope during setup.

The force generator has the same job as before. During advancement it computes forces from the current positions and the spring parameters it was given at initialization. The callbacks specify the starting structure and its connections; they do not reset the point positions during the run. See the [`IBRedundantInitializer` documentation](https://ibamr.github.io/IBAMR-docs/ibamr/classIBAMR_1_1IBRedundantInitializer.html) for the initialization interface.

## Build and run the generated model

Configure and build your application with the new initializer. Use `Debug` in place of `Release` if you installed a debug build of IBAMR.

```bash
cmake -S "$BAND_SRC" -B "$BAND_BUILD" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER="$(command -v mpicc)" \
  -DCMAKE_CXX_COMPILER="$(command -v mpicxx)" \
  -DCMAKE_PREFIX_PATH="${IBAMR_ROOT:?Set the installed IBAMR prefix first}"
cmake --build "$BAND_BUILD" --target my-rubber-band --parallel 4
```

Run the generated model from its own directory.

```bash
cd "$BAND_BUILD"
mkdir generated
cp "$BAND_SRC/input2d" generated/
cd generated
../my-rubber-band input2d > stdout.log 2>&1
tail -n 15 stdout.log
```

## Results

Check that the run reaches time `0.6`. Open `dumps.visit` and `lag_data.visit` from `$BAND_BUILD/generated/viz_IB2d` in VisIt. Follow the pressure and mesh overlay steps in tutorial 04 and select `rubber_band_mesh` for the structure. Use **Spatial Extents** on the Mesh plot to measure width and height at the initial and final states. The checked run gave these dimensions.

| Time | Width | Height |
| --- | --- | --- |
| `0` | `0.35714` | `0.70000` |
| `0.6` | `0.43072` | `0.57178` |

These dimensions agree with the original model from tutorial 04.

### Change the initial ellipse

Make a second run with a smaller vertical semiaxis:

```bash
cd "$BAND_BUILD"
mkdir less-stretched
cp generated/input2d less-stretched/
```

In `less-stretched/input2d`, change `beta = 0.35` to `beta = 0.30` in `RubberBand`. Run the same executable:

```bash
cd "$BAND_BUILD/less-stretched"
../my-rubber-band input2d > stdout.log 2>&1
tail -n 15 stdout.log
```

Compare the new `rubber_band_mesh` with the one from `generated` at time `0`. The band starts about `0.4167` wide and `0.6000` tall. The starting ellipse is closer to a circle because the coordinate function uses the new `beta` and recalculates `alpha`. With `radius` held fixed, the continuous ellipses have the same enclosed area, $\pi\alpha\beta$. Play the new sequence to see how the starting shape changes the subsequent motion.

The edit takes effect on the next run without recompiling because the new application reads `beta` at startup and uses it to generate the point coordinates.

## Further exploration: measure the enclosed area

The rubber band is a closed curve. To measure the area it encloses, join consecutive points with straight segments and use the current point positions to calculate the area of that polygon. For $n$ points, let $j=(i+1)\bmod n$ so the last segment connects back to point zero. The **shoelace formula** gives the enclosed area.

$
A=\frac{1}{2}\left|\sum_{i=0}^{n-1}(x_i y_j-x_j y_i)\right|.
$

The sign of the sum depends on the direction you walk around the curve. Taking its absolute value gives the area in squared length units. This calculation measures the region inside the band, while the volume history in tutorial 08 measures the area occupied by the finite-element material.

### Read the current coordinates in point order

Add these headers to `$BAND_SRC/example.cpp` before `// Set up application namespace declarations`.

```cpp
#include <fstream>
#include <iomanip>
#include <petscvec.h>
```

**Get the current positions.** After `generate_springs` and before the closing `} // namespace`, start `write_band_area` with the block below. Append each subsequent block in order to complete the function. The function receives the output stream, the structure's data manager and level, and the time for this measurement. `getLData("X", level)` retrieves the current positions. `getVec()` gives access to the PETSc vector that holds them across the MPI processes.

```cpp
void
write_band_area(std::ostream& output, LDataManager* manager, int level, double time)
{
    Pointer<LData> positions = manager->getLData("X", level);
    Vec distributed = positions->getVec();
```

**Put the coordinates in point order.** PETSc stores the coordinates by process ownership, so neighboring points along the band can be stored on different processes. `VecDuplicate` creates an empty vector with the same layout. `scatterPETScToLagrangian` fills it in the point order from `generate_points`. Then `scatterToZero` collects that ordered vector on rank zero. Every rank participates in both scatter calls. [PETSc scatter documentation](https://petsc.org/release/manualpages/Vec/VecScatterCreateToZero/)

```cpp
    Vec ordered;
    PetscCallAbort(PETSC_COMM_WORLD, VecDuplicate(distributed, &ordered));
    manager->scatterPETScToLagrangian(distributed, ordered, level);

    Vec gathered = nullptr;
    manager->scatterToZero(ordered, gathered);
```

**Sum the polygon contributions on rank zero.** Rank zero has all the coordinates. There are two values per point, with `xy[2*i]` holding $x_i$ and `xy[2*i + 1]` holding $y_i$. The loop takes the next point `j` and adds the term $x_i y_j-x_j y_i$ from the shoelace formula. The remainder operator connects the last point back to point zero.

```cpp
    if (IBTK_MPI::getRank() == 0)
    {
        PetscInt num_coordinates;
        PetscCallAbort(PETSC_COMM_WORLD, VecGetLocalSize(gathered, &num_coordinates));
        const PetscInt num_points = num_coordinates / 2;
        const PetscScalar* xy;
        PetscCallAbort(PETSC_COMM_WORLD, VecGetArrayRead(gathered, &xy));

        double twice_signed_area = 0.0;
        for (PetscInt i = 0; i < num_points; ++i)
        {
            const PetscInt j = (i + 1) % num_points;
            twice_signed_area += PetscRealPart(xy[2*i] * xy[2*j + 1] - xy[2*j] * xy[2*i + 1]);
        }
```

**Write the area and release the temporary vectors.** Rank zero returns its read access to the coordinate array, then writes the time and half the absolute value of the sum. After the rank-zero block, every rank releases the two temporary vectors. The data manager owns the original position vector.

```cpp
        PetscCallAbort(PETSC_COMM_WORLD, VecRestoreArrayRead(gathered, &xy));
        output << time << ' ' << 0.5 * std::abs(twice_signed_area) << '\n';
    }
    PetscCallAbort(PETSC_COMM_WORLD, VecDestroy(&gathered));
    PetscCallAbort(PETSC_COMM_WORLD, VecDestroy(&ordered));
}
```

### Record the initial area and each completed step

In `main`, insert this block immediately before `// Main time step loop.`, after the hierarchy has been initialized and `loop_time` has been set.

```cpp
LDataManager* area_manager = ib_method_ops->getLDataManager();
std::ofstream area_output;
if (IBTK_MPI::getRank() == 0)
{
    area_output.open("band_area.curve");
    if (!area_output)
    {
        TBOX_ERROR("Unable to open band_area.curve\n");
    }
    area_output << "# Enclosed area\n" << std::setprecision(12);
}
write_band_area(area_output, area_manager, structure_level, loop_time);
```

The file starts with a name for the curve, followed by two columns for physical time and area. Every process calls `write_band_area` so the coordinates can be collected, and rank zero opens and writes the file.

Inside the time loop, add the same call immediately after `loop_time += dt;`.

```cpp
write_band_area(area_output, area_manager, structure_level, loop_time);
```

At that point `advanceHierarchy(dt)` has updated the structure and `loop_time` gives the time of the new positions. Calling the function outside the visualization block records every completed step.

### Run the area diagnostic

Rebuild and run with the original generated ellipse from `$BAND_SRC/input2d`.

```bash
cmake --build "$BAND_BUILD" --target my-rubber-band --parallel 4
cd "$BAND_BUILD"
mkdir area
cp "$BAND_SRC/input2d" area/
cd area
../my-rubber-band input2d > stdout.log 2>&1
head -n 3 band_area.curve
tail -n 1 band_area.curve
```

The first data row should be at time `0` and the last at time `0.6`. For the supplied 304-point ellipse, the initial polygon area is about `0.196336`. In the checked run the area reached `0.196687` at time `0.6`, an increase of about 0.18% from its initial value.

## Before moving on

You built a standalone application that generates the rubber band from input parameters. `BAND_BUILD` holds the executable and the `generated` and `less-stretched` runs. If you completed the area calculation, the `area` run also holds `band_area.curve` with the initial and subsequent areas.

The next tutorial shows how to save a simulation checkpoint and resume from it.
