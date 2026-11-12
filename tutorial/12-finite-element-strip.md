# Tutorial 12: Bend a finite-element strip in a flow

## Introduction

A thin elastic strip stands vertically across the middle of a rectangular fluid domain. Its top and bottom edges are fixed, leaving the middle free to bend. The pressure is higher at the left side of the domain than at the right, so the fluid pushes the strip to the right. The strip pushes back on the fluid as it deforms.

Watch the strip bend and swing back between its held ends. Then add a spatially varying stiffness in C++ to make the material stiffer near the supports, and compare the motion.

## Before you start

Use the configured build and the finite-element visualization workflow from [tutorial 07](07-finite-element-ring.md). This example is `examples/IBFE/explicit/ex2` and uses the same IBAMR installation with libMesh and ExodusII support.

Line numbers below refer to the supplied repository files before the edits in this tutorial. Use the named function or input block to find the same location after adding code.

## Running the example

Configure the repository to include the strip application, then build it.

```bash
cmake -S "$EXAMPLES_SRC" -B "$EXAMPLES_BUILD"
cmake --build "$EXAMPLES_BUILD" --target IBFE-ex2-2d --parallel 4
```

Copy the input into a new run directory. The program generates the finite-element mesh when it starts.

```bash
cd "$EXAMPLES_BUILD/examples/IBFE/explicit/ex2"
mkdir strip
cp "$EXAMPLES_SRC/examples/IBFE/explicit/ex2/input2d" strip/
cd strip
../main2d input2d
```

The supplied input sets `END_TIME = 10.0`. Check that the last advance reaches time `10`.

## Results

Open `dumps.visit` and `output.ex2` from `strip/viz_IB2d`. Follow the overlay steps from tutorial 07. Use **Pseudocolor → P** for the fluid and a black **Mesh → Mesh** for the strip. Set the pressure limits to `[-0.12,0.12]` and use `hot_desaturated`. The color scale clips values beyond those limits. For both cases, use the full view with horizontal bounds `0` to `2` and vertical bounds `0` to `1`.

Apply **Displace** to the Mesh plot with a multiplier of `1` and the expression below. This draws the elements at their current physical coordinates.

```text
{<X_0>, <X_1>, 0} - coord(<Mesh>)
```

Create a **Time** correlation for the two databases. Both sequences have 81 states from time `0` to `10`. The input uses `DT = 0.0625*DX` and writes every 128 steps, so the saved states are spaced `0.125` apart. Play the sequence to see the middle bend and swing back while the top and bottom remain in place. Select time `2` for a clear view of the first bend.

![Baseline strip and pressure at time 2](figures/F09-strip.png)

*Baseline strip at time 2. The black mesh shows how the solid's interior deforms between its held ends.*

## The input and program

Follow the excerpts alongside the [application](../examples/IBFE/explicit/ex2/example.cpp) and [input](../examples/IBFE/explicit/ex2/input2d).

### Build the strip and hold its ends

The input sets the mesh element type and spacing relative to the finest fluid grid.

**Read:** [`input2d`, lines 12–14](../examples/IBFE/explicit/ex2/input2d#L12-L14).

```text
DX = L/NFINEST
MFAC = 2.0
ELEM_TYPE = "QUAD9"
```

The application reads these values and generates a rectangular reference mesh.

**Read:** [`example.cpp`, lines 165–167](../examples/IBFE/explicit/ex2/example.cpp#L165-L167).

```cpp
const double dx = input_db->getDouble("DX");
const double ds = input_db->getDouble("MFAC") * dx;
string elem_type = input_db->getString("ELEM_TYPE");
```

**Read:** [`example.cpp`, lines 169–176](../examples/IBFE/explicit/ex2/example.cpp#L169-L176).

```cpp
MeshTools::Generation::build_square(mesh,
                                    static_cast<int>(ceil(0.1 / ds)),
                                    static_cast<int>(ceil(1.0 / ds)),
                                    0.95,
                                    1.05,
                                    0.0,
                                    1,
                                    Utility::string_to_enum<ElemType>(elem_type));
```

The strip spans `0.95` to `1.05` in $x$ and `0` to `1` in $y$. `ds` sets the target element spacing. With `DX = L/NFINEST = 1/64`, the ceiling expressions give four elements across the width and 32 along the length. A `QUAD9` is a quadrilateral with nine nodes, including one at its center.

The application then visits exterior sides of the mesh. For this rectangle, boundary IDs `0` and `2` label the bottom and top edges.

**Read:** [`example.cpp`, lines 202–205](../examples/IBFE/explicit/ex2/example.cpp#L202-L205).

```cpp
if (boundary_info.has_boundary_id(elem, side, 0) || boundary_info.has_boundary_id(elem, side, 2))
{
    boundary_info.add_side(elem, side, FEDataManager::ZERO_DISPLACEMENT_XY_BDRY_ID);
}
```

The added boundary ID marks both displacement components as zero there. When IBAMR sets up the finite-element systems, these nodes are constrained to their reference positions. The remaining nodes can move, which lets the strip bend between its supports.

### Drive the fluid with a pressure difference

The fluid boundary inputs use the component and face numbering from [tutorial 09](09-target-forces-and-boundaries.md). `VelocityBcCoefs_0` sets the horizontal component and `VelocityBcCoefs_1` sets the vertical component.

**Read:** [`input2d`, lines 50–82](../examples/IBFE/explicit/ex2/input2d#L50-L82).

```text
VelocityBcCoefs_0 {
   acoef_function_0 = "0.0"
   acoef_function_1 = "0.0"
   acoef_function_2 = "0.0"
   acoef_function_3 = "0.0"

   bcoef_function_0 = "1.0"
   bcoef_function_1 = "1.0"
   bcoef_function_2 = "1.0"
   bcoef_function_3 = "1.0"

   gcoef_function_0 = "-1.0e-1"
   gcoef_function_1 = " 1.0e-1"
   gcoef_function_2 = "0.0"
   gcoef_function_3 = "0.0"
}

VelocityBcCoefs_1 {
   acoef_function_0 = "1.0"
   acoef_function_1 = "1.0"
   acoef_function_2 = "1.0"
   acoef_function_3 = "1.0"

   bcoef_function_0 = "0.0"
   bcoef_function_1 = "0.0"
   bcoef_function_2 = "0.0"
   bcoef_function_3 = "0.0"

   gcoef_function_0 = "0.0"
   gcoef_function_1 = "0.0"
   gcoef_function_2 = "0.0"
   gcoef_function_3 = "0.0"
}
```

The horizontal component uses `a = 0` and `b = 1`. On the left and right faces that component is normal to the boundary. For these inputs, the v0.19.0 staggered solver uses `g` as the value of the Cartesian normal stress component on each face:

$
\sigma_{xx}=-p+2\mu_f\frac{\partial u_x}{\partial x}.
$

Here $\mu_f$ is the fluid viscosity, set by `MU = 0.01` in the input. `VelocityBcCoefs_1` sets $u_y=0$ along both vertical faces. Since $u_y$ is constant as you move along either face in the $y$ direction, $\partial u_y/\partial y=0$ there. Incompressibility gives $\partial u_x/\partial x=-\partial u_y/\partial y=0$. The normal stress condition therefore reduces to $p=-g$ on those faces. The left pressure is $0.1$ and the right pressure is $-0.1$. IBAMR applies these conditions through its normal-stress boundary treatment. [v0.19.0 staggered normal-stress implementation](https://github.com/IBAMR/IBAMR/blob/v0.19.0/src/navier_stokes/INSStaggeredPressureBcCoef.cpp#L276)

On the bottom and top faces, `VelocityBcCoefs_1` sets the normal velocity to zero. The horizontal component is tangential there, so its zero stress values prescribe zero shear stress.

### Follow both contributions to the elastic stress

In `namespace ModelData`, the structural coefficient is

**Read:** [`example.cpp`, line 54](../examples/IBFE/explicit/ex2/example.cpp#L54).

```cpp
static const double mu = 10.0;
```

The two functions below it each fill the output tensor `PP` from the deformation gradient `FF`.

**Read:** [`example.cpp`, line 68](../examples/IBFE/explicit/ex2/example.cpp#L68).

```cpp
PP = mu * FF;
```

**Read:** [`example.cpp`, line 83](../examples/IBFE/explicit/ex2/example.cpp#L83).

```cpp
PP = -mu * tensor_inverse_transpose(FF, NDIM);
```

Writing the structural coefficient as $\mu_s$, the contributions add to the first Piola–Kirchhoff stress

$
\mathbf P^e=\mu_s(\mathbf F-\mathbf F^{-T}).
$

The second term uses the inverse transpose of the deformation gradient. When the strip is undeformed, $\mathbf F=\mathbf I$ and the two terms cancel. This stress comes from a **compressible neo-Hookean** strain energy per reference volume. With $J=\det\mathbf F$, the two-dimensional energy is

$
W=\frac{\mu_s}{2}(\mathbf F:\mathbf F-2)-\mu_s\log J.
$

This is the logarithmic form with the first Lamé parameter $\lambda=0$. Differentiating $W$ with respect to $\mathbf F$ gives the two stress contributions above. [FEniCS compressible neo-Hookean model](https://docs.fenicsproject.org/dolfinx/main/cpp/demos/demo_hyperelasticity.html)

The program registers the two contributions separately so each can use its own integration rule.

**Read:** [`example.cpp`, lines 270–277](../examples/IBFE/explicit/ex2/example.cpp#L270-L277).

```cpp
IBFEMethod::PK1StressFcnData PK1_dev_stress_data(PK1_dev_stress_function);
IBFEMethod::PK1StressFcnData PK1_dil_stress_data(PK1_dil_stress_function);
PK1_dev_stress_data.quad_order =
    Utility::string_to_enum<libMesh::Order>(input_db->getStringWithDefault("PK1_DEV_QUAD_ORDER", "THIRD"));
PK1_dil_stress_data.quad_order =
    Utility::string_to_enum<libMesh::Order>(input_db->getStringWithDefault("PK1_DIL_QUAD_ORDER", "FIRST"));
ib_method_ops->registerPK1StressFunction(PK1_dev_stress_data);
ib_method_ops->registerPK1StressFunction(PK1_dil_stress_data);
```

The input sets `PK1_DEV_QUAD_ORDER = "FIFTH"` and `PK1_DIL_QUAD_ORDER = "THIRD"` at [`input2d`, lines 15–16](../examples/IBFE/explicit/ex2/input2d#L15-L16). These orders control how each contribution is integrated over the elements. During the advance, `IBFEMethod` evaluates the callbacks and uses the resulting stresses in the weak force calculation from tutorial 07. Changing the shared `mu` scales both contributions together.

## Further exploration: vary the material stiffness in C++

Make the strip stiffer near its supports while keeping the middle at the baseline stiffness. You will supply the local coefficient from the reference position passed to each stress callback.

### Define the material profile

In `$EXAMPLES_SRC/examples/IBFE/explicit/ex2/example.cpp`, keep `mu = 10.0`. Add this helper immediately below it, inside `namespace ModelData`.

**Add to:** `example.cpp`, after `ModelData::mu` at [`example.cpp`, line 54](../examples/IBFE/explicit/ex2/example.cpp#L54).

```cpp
double
local_shear_modulus(const libMesh::Point& s)
{
    const double distance_from_middle = 2.0 * s(1) - 1.0;
    return mu * (1.0 + distance_from_middle * distance_from_middle);
}
```

The reference coordinate `s(1)` runs from `0` at the bottom support to `1` at the top. The expression `2.0 * s(1) - 1.0` is zero in the middle and has magnitude one at either end. Squaring it gives the same variation above and below the middle. The helper returns `10` at the middle and `20` at the supports, with a smooth variation between them.

The stiffness is assigned by material position, so it stays with that part of the strip as the strip moves.

### Use the profile in both stress contributions

In the signatures of `PK1_dev_stress_function` and `PK1_dil_stress_function`, replace the argument line `const libMesh::Point& /*s*/,` with

**Replace in:** `example.cpp`, the reference-position arguments in `PK1_dev_stress_function` and `PK1_dil_stress_function` ([`example.cpp`, line 61](../examples/IBFE/explicit/ex2/example.cpp#L61) and [`example.cpp`, line 76](../examples/IBFE/explicit/ex2/example.cpp#L76) before editing).

```cpp
const libMesh::Point& s,
```

This gives a name to the reference-position argument that the callbacks already receive. In `PK1_dev_stress_function`, replace `PP = mu * FF;` with

**Replace in:** `example.cpp`, `PK1_dev_stress_function` ([`example.cpp`, line 68](../examples/IBFE/explicit/ex2/example.cpp#L68) before editing).

```cpp
PP = local_shear_modulus(s) * FF;
```

In `PK1_dil_stress_function`, replace `PP = -mu * tensor_inverse_transpose(FF, NDIM);` with

**Replace in:** `example.cpp`, `PK1_dil_stress_function` ([`example.cpp`, line 83](../examples/IBFE/explicit/ex2/example.cpp#L83) before editing).

```cpp
PP = -local_shear_modulus(s) * tensor_inverse_transpose(FF, NDIM);
```

Both terms use the same coefficient at each material point. Their sum is now $\mathbf P^e=\mu_s(s)(\mathbf F-\mathbf F^{-T})$. Using the helper in both callbacks preserves the cancellation at $\mathbf F=\mathbf I$.

### Rebuild and compare the motion

Rebuild the existing target and use the baseline input for the new run.

```bash
cmake --build "$EXAMPLES_BUILD" --target IBFE-ex2-2d --parallel 4
cd "$EXAMPLES_BUILD/examples/IBFE/explicit/ex2"
mkdir strip-graded
cp strip/input2d strip-graded/
cd strip-graded
../main2d input2d
```

Open the new fluid and solid output in VisIt and create a Time correlation for this pair. Use the same view, pressure scale, and displaced Mesh plot as the baseline. Compare the motion at time `2`, then play both sequences through time `10`.

![Strip with spatially varying stiffness at time 2](figures/F09-strip-graded.png)

*The material coefficient varies smoothly from 10 in the middle to 20 at the supported ends.*

At time `2`, the strip with stiffer ends bends less than the baseline. Play the full sequences to see how the stiffness distribution changes the subsequent motion.

## Before moving on

You ran a strip held at both ends, followed the fluid loading and elastic stress through the input and program, and added a local material coefficient in both stress callbacks. You should have two runs you can play in VisIt and compare at matching physical times.

The next tutorial shows how a solid deforms under a prescribed surface load in a solid-only calculation. [Tutorial 13](13-cooks-membrane.md)
