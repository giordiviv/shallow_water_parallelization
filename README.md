# Shallow Water Parallelization
[![Python 3](https://img.shields.io/badge/python-3-blue.svg?logo=python&logoColor=white)](https://www.python.org/)
[![MPI: mpi4py](https://img.shields.io/badge/MPI-mpi4py-informational.svg)](https://mpi4py.readthedocs.io/)
[![NumPy](https://img.shields.io/badge/NumPy-013243.svg?logo=numpy&logoColor=white)](https://numpy.org/)
[![Scheduler: SLURM](https://img.shields.io/badge/scheduler-SLURM-lightgrey.svg)](https://slurm.schedmd.com/)
[![Course: EPFL PHYS-743](https://img.shields.io/badge/EPFL-PHYS--743%20Parallel%20Programming-red.svg)](https://edu.epfl.ch/)

Parallel tsunami simulation: shallow water equations with MPI.

A 2D **shallow water** solver that simulates tsunami propagation over real topography, and
its **parallelization with MPI** on an HPC cluster. The serial NumPy code was turned into a
domain-decomposed MPI program, profiled, and benchmarked on up to **529 processes across 15
nodes** with grids of up to **8001 × 8001** cells.

Project for the EPFL course *PHYS-743 Parallel Programming* (2023).

## The problem

The shallow water equations describe the water height $`h`$ and the momenta $`hu`$, $`hv`$ on a
500 km × 500 km map with seabed topography. They are integrated with an explicit
finite-volume scheme. The time step is adaptive and set by the CFL condition from the fastest
wave speed $`|u| + \sqrt{g h}`$ on the whole grid.

## Parallelization strategy

- **2D Cartesian domain decomposition.** `MPI.Compute_dims` and `Create_cart` split the grid
  into blocks, one per process, with remainder rows and columns spread evenly.
- **Halo exchange.** Each block keeps ghost cells, which are swapped with its four neighbours
  (`sendrecv` north/south/left/right) at every time step.
- **Global time step.** Each process computes its local maximum wave speed. The root reduces
  them and broadcasts the CFL time step, so all ranks stay in sync.
- **Output.** The blocks are gathered on the root, reassembled and written to disk, together
  with a CSV row of timings (initialisation, main loop, reconstruction) and GFLOP/s.
- **Profiling.** `cProfile` runs, visualised as call graphs with `gprof2dot`, to find the
  serial bottlenecks and communication overhead.

## Results

<!-- TODO(human): add the scaling results (table and/or plot) -->

## Repository layout

```
python/
├── compute.py                 serial solver
├── compute_mpi.py             MPI solver (domain decomposition + halo exchange)
├── compute_profiling.py       serial solver instrumented for profiling
├── compute_mpi_profiling.py   MPI solver instrumented for profiling
├── visualize.py               3D plots of the wave and of the process grid
├── config.yaml                grid size, map size, end time, MPI/SLURM resources
└── run.zsh                    local run (serial or mpiexec)
run_jobscript.sh               generates and submits a SLURM job from config.yaml
profile_jobscript.sh           same, with cProfile + gprof2dot
```

## How to run

Requirements: Python 3, NumPy, Matplotlib, PyYAML, mpi4py (and an MPI library). For profiling
you also need `gprof2dot` and Graphviz. The input grids (initial conditions and topography)
were provided by the course and are not included. Place them in `data/`.

```bash
# Local: 1 process runs the serial code, N > 1 runs mpiexec -n N
cd python && ./run.zsh

# Cluster: set number_of_processes / number_of_nodes / Nx in python/config.yaml, then
./run_jobscript.sh        # writes tsunami.job and submits it with sbatch
./profile_jobscript.sh    # same, with profiling
```
