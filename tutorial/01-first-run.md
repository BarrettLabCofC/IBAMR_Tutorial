# Tutorial 01: Run a fluid simulation

## Introduction

Build and run a small two-dimensional fluid example, check its progress, and locate its output. The shear-layer model provides a starting case for learning the input and visualization workflow.

## Before you start

Complete [tutorial 00](00-orientation.md). Keep its terminal open so that `EXAMPLES_SRC` and `EXAMPLES_BUILD` remain set. This activity uses `examples/navier_stokes/ex1` in the source and build directories.

## Running the example

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

Open `$EXAMPLES_BUILD/examples/navier_stokes/ex1/first-flow/input2d` in your editor. The table gives each setting's line in the supplied [`input2d`, lines 1–120](../examples/navier_stokes/ex1/input2d#L1-L120). Make the following four changes for a short, single-level run.

| Setting | Line | Supplied value | Value for this run |
| --- | --- | --- | --- |
| `MAX_LEVELS` | 7 | `3` | `1` |
| `N` | 9 | `16` | `64` |
| `END_TIME` | 17 | `2.0` | `1.0` |
| `Main.viz_dump_interval` | 106 | `NFINEST/8` | `10` |

`Main.viz_dump_interval` means the `viz_dump_interval` entry inside the `Main` block. Save the input, then run from the same `first-flow` directory:

```bash
../main2d input2d
```

This uses one MPI process. The program reads `input2d` in the current directory and writes its output there. Look for repeated time-step messages. The last completed step should report simulation time `1`; the step number can vary with numerical settings.

## Results

After the run, check the files in this same directory:

```bash
ls
tail -n 15 INS2d.log
head -n 4 viz_INS2d/dumps.visit
```

`INS2d.log` contains the application log. `viz_INS2d` contains saved fluid fields. `dumps.visit` is a small list telling VisIt where the saved states are; keep the entire visualization directory. Do not open just one numbered state if you want the time sequence. The reference run reached $t=1$, wrote 30 saved states, and produced about 2.8 MB in the run directory.

## How the input file is read

When the application starts, `AppInitializer` passes `input2d` to SAMRAI's input parser. The parser reads named assignments and blocks such as `Main { ... }` and stores their values in an input database. The application and its solver objects then retrieve settings from that database. See the [SAMRAI parser documentation](https://ibamr.github.io/IBAMR-docs/ibamr/classSAMRAI_1_1tbox_1_1Parser.html) for the input-reading interface.

Numeric assignments can contain simple arithmetic expressions, using `+`, `-`, `*`, `/`, `^` for powers, and parentheses. They can also use names defined earlier in the file. SAMRAI keeps integer values such as `1` distinct from floating-point values such as `1.0`, so `1/2` uses integer division and gives `0`, whereas `1.0/2` promotes the operation to floating point and gives `0.5`. For example, the supplied input calculates the finest grid resolution from three settings:

**Read:** [`input2d`, lines 7–10](../examples/navier_stokes/ex1/input2d#L7-L10).

```text
MAX_LEVELS = 3
REF_RATIO = 4
N = 16
NFINEST = (REF_RATIO^(MAX_LEVELS - 1))*N
```

After the edits for this run, `MAX_LEVELS = 1` and `N = 64`, so the parser evaluates `NFINEST` as `(4^0)*64 = 64`. Later, it evaluates the time-step bound from that result:

**Read:** [`input2d`, line 15](../examples/navier_stokes/ex1/input2d#L15).

```text
DT_MAX = 0.5/NFINEST
```

For this run, `DT_MAX` becomes `0.5/64 = 0.0078125`. Keeping dependent settings as expressions lets them follow changes to parameters such as `N`. SAMRAI evaluates these assignments when reading the file at startup and stores the resulting numbers.

## Further exploration: compare two viscosities

Complete this comparison before tutorial 02, which opens both runs in VisIt.

To practice changing an input parameter and keeping results separate, increase viscosity tenfold. Keep the baseline and make a second run directory:

```bash
cd "$EXAMPLES_BUILD/examples/navier_stokes/ex1"
mkdir first-flow-viscous
cp first-flow/input2d first-flow-viscous/
```

Open `first-flow-viscous/input2d` in the example's build directory. Change only the top-level viscosity assignment from

**Run input:** `first-flow-viscous/input2d`; viscosity is at [`input2d`, line 2](../examples/navier_stokes/ex1/input2d#L2) in the supplied file.

```text
MU  = 0.0001
```

to

**Replacement in the same run input:** change the viscosity entry identified above.

```text
MU  = 0.001
```

Keep density, grid, final time, and all other values unchanged. No rebuild is needed. Run the same program from the new directory:

```bash
cd "$EXAMPLES_BUILD/examples/navier_stokes/ex1/first-flow-viscous"
../main2d input2d
```

The comparison should also reach $t=1$. In [tutorial 02](02-first-visualization.md), compare the two flows at that same physical time.

If the program says it cannot open `input2d`, check `pwd` and `ls`: run from the directory containing the copied input. If the visualization directory is missing, inspect the log before moving on.

## Before moving on

This tutorial built and ran the fluid example, prepared its run inputs, and changed viscosity without recompiling. You should have separate `first-flow` and `first-flow-viscous` run directories, each containing its input, `INS2d.log`, and complete `viz_INS2d` output directory with a `dumps.visit` file.

In [tutorial 02](02-first-visualization.md), you will open the fluid output in VisIt, select fields and saved times, and compare the baseline and higher-viscosity runs.
