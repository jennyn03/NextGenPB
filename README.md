# NextGenPB 
-----------  
Copyright (C) 2021-2025 Vincenzo Di Florio

Copyright (C) 2019-2025 Carlo de Falco

Copyright (C) 2020-2021 Martina Politi

This software is distributed under the terms
the terms of the GNU/GPL licence v3

# Overview
----------

**NextGenPB** is a high-performance solver for the linearized Poisson–Boltzmann equation (PBE), built on an adaptive octree mesh.
It efficiently computes electrostatic potentials in heterogeneous dielectric media using a flexible, hierarchical discretization scheme.

The equation solved is:


$$
-\mathrm{div} \left( \varepsilon_0 \varepsilon_r \nabla \varphi \right) + \kappa^2 \varphi = \rho^f
$$

on a rectangular domain.

# Nonlinear Poisson--Boltzmann Extension
---

This repository also contains a course project extension that adds a
Newton-based solver for the nonlinear Poisson--Boltzmann equation.

The implementation is available in the `nonlinear-solver` branch.

The nonlinear formulation replaces the linear ionic contribution with a
hyperbolic sine term. At every Newton iteration, the Jacobian contains
the corresponding hyperbolic cosine contribution.

The solver mode is selected in the parameter file:

```text
linearized = 1   # original linearized solver
linearized = 0   # nonlinear Newton solver
```

The main modifications are located in:

```text
include/pb_class.h
src/pb_class.cpp
src/poisson_boltzmann.cpp
```

The nonlinear extension adds the following functions:

```text
assemble_newton_system
newton_solve
```

## Build using Docker

The recommended environment for this project is the supplied
`Dockerfile_ubuntu`.

Run the following command from the root of the repository:

```bash
docker build \
  --platform linux/amd64 \
  --build-arg CFLAGS="-O2 -mtune=generic -fno-lto" \
  -f Dockerfile_ubuntu \
  -t nextgenpb .
```

The `--platform linux/amd64` option is required when building the image
through emulation on Apple Silicon. It may be omitted on a native
x86-64 Linux system.

Create and start a development container from the repository root:

```bash
docker run \
  --platform linux/amd64 \
  -it \
  --name nextgenpb-dev \
  -v "$PWD":/App/NextGenPB \
  nextgenpb \
  /bin/bash
```

For later sessions, restart the existing container with:

```bash
docker start -ai nextgenpb-dev
```

## Compile the Project Branch

Inside the container, copy the Ubuntu build configuration and update it
to reference the mounted project directory:

```bash
cd /App/NextGenPB

cp local_setting/local_settings_ubuntu.mk src/local_settings.mk

sed -i \
  's#/usr/local/nextgenPB#/App/NextGenPB#g' \
  src/local_settings.mk
```

Compile the solver:

```bash
cd /App/NextGenPB/src

make distclean
make -j2
```

The resulting executable is:

```text
/App/NextGenPB/src/ngpb
```

Use this executable rather than `/usr/local/nextgenPB/src/ngpb`, which
belongs to the original linear version installed while building the
Docker image.

## Running the Solver

The program is executed with a parameter file:

```bash
/App/NextGenPB/src/ngpb --prmfile options.prm
```

Input paths in the parameter file are interpreted relative to the
directory from which the command is executed. The command should
therefore be run from the test directory, or the input paths must be
adjusted accordingly.

## Reproduce the Nonlinear Sphere Test

The principal nonlinear test contains one unit charge at the origin
inside a sphere with radius 2.0 Å.

Create a test directory from the repository root:

```bash
cd /App/NextGenPB
mkdir -p tests/sphere
```

Create `tests/sphere/sphere_q1.pqr`:

```bash
cat > tests/sphere/sphere_q1.pqr <<'EOF'
ATOM      1  X    X    1       0.000   0.000   0.000  1.000  2.000
EOF
```

Copy the complete standard options file:

```bash
cp data/options.prm tests/sphere/options_nonlinear.prm
```

Update the input file, solver mode, boundary condition, and surface
probe radius:

```bash
sed -i \
  -e 's#^filename = .*#filename = sphere_q1.pqr#' \
  -e 's/^linearized = .*/linearized = 0/' \
  -e 's/^bc_type = .*/bc_type = 3/' \
  -e 's/^surface_parameter = .*/surface_parameter = 0.01/' \
  tests/sphere/options_nonlinear.prm
```

The copied parameter file already contains the remaining settings used
for the test:

```text
ionic_strength = 0.145
molecular_dielectric_constant = 2
solvent_dielectric_constant = 80
T = 298.15

surface_type = 0
stern_layer_surf = 0

mesh_shape = 0
perfil1 = 0.8
perfil2 = 0.5
scale = 2.0

linear_solver = lis
```

Run the test from the test directory:

```bash
cd /App/NextGenPB/tests/sphere

/App/NextGenPB/src/ngpb \
  --prmfile options_nonlinear.prm \
  2>&1 | tee nonlinear_q1.log
```

The relevant output can be displayed with:

```bash
grep -E "Newton|converged|Flux charge|Sum of electro" nonlinear_q1.log
```

The expected results are approximately:

```text
Newton iterations: 5
Electrostatic energy: -68.705158 kT
Flux charge: 1.0000000000
```

Small numerical differences may occur depending on the platform and
library versions.

## Linear Comparison

Create a linear parameter file from the nonlinear configuration:

```bash
cp options_nonlinear.prm options_linear.prm

sed -i \
  's/^linearized = .*/linearized = 1/' \
  options_linear.prm
```

Run the linear solver:

```bash
/App/NextGenPB/src/ngpb \
  --prmfile options_linear.prm \
  2>&1 | tee linear_q1.log
```

The expected linear electrostatic energy is approximately:

```text
-68.669912 kT
```

## Known Limitations

The current nonlinear implementation:

- uses an undamped Newton method;
- does not implement a line search or continuation strategy;
- supports the configuration without a Stern layer;
- has not been compared with an independent nonlinear reference solver;
- may fail to converge for strongly charged configurations.

Exploratory tests with charges `q = 5` and `q = 10` did not converge
within the maximum number of Newton iterations.


# Documentation & Tutorials
---

Comprehensive installation instructions, examples, and usage guides are available here:

[NextGenPB Tutorial and Guide](https://vdiflorio.github.io/nextgenpb_tutorial/)

---

# Citation

If you use **NextGenPB** in your research, please cite the following article:

>Di Florio, V., Ansalone, P., Siryk, S. V., Decherchi, S., De Falco, C., & Rocchia, W. (2025). NextGenPB: An analytically-enabled super resolution tool for solving the Poisson-Boltzmann Equation featuring local (de) refinement. Computer Physics Communications, 109816.
