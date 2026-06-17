# Chapter 5 — Benchmarks: High-Performance Conjugate Gradient (HPCG)

> **Prerequisites:** Completed Chapter 3 (Competition Preparation) and Chapter 4 (HPCC). You need OpenMPI compiled from source (`~/opt/openmpi/`), passwordless SSH across all nodes, and NFS home directory mounted.

---

## What is HPCG and Why Does it Exist?

Cast your mind back to HPL. HPL solves a **dense** linear system — a matrix where every element is filled with a number. The computation is a wall-to-wall sequence of floating-point multiplications and additions. Modern CPUs are spectacular at this: their caches can be pre-loaded with the next tile of the matrix, their AVX vector units can process 8 doubles per clock cycle simultaneously, and the arithmetic pipeline never goes hungry. HPL is essentially designed to make your hardware look as fast as possible on paper.

The problem is that real scientific applications almost never look like HPL.

In computational fluid dynamics, finite element structural analysis, climate modelling, seismic simulation, and circuit simulation — the governing equations produce matrices that are almost entirely **zeros**. A typical sparse matrix from a 3D mesh of a billion-element simulation might have fewer than 30 non-zero entries per row out of a billion columns. Storing and computing with all those zeros is wasteful, so these applications use **sparse** data structures instead.

**HPCG — the High-Performance Conjugate Gradient benchmark** — was created by Jack Dongarra (HPL's creator) and Michael Heroux specifically to address this gap. It solves a large sparse system using the Conjugate Gradient iterative method with a Symmetric Gauss-Seidel (SYMGS) multi-grid preconditioner. This is one of the most common solver patterns in all of computational science.

The result is a benchmark that is fundamentally different from HPL:

| Property | HPL | HPCG |
|---|---|---|
| Matrix type | Dense | Sparse (27 non-zeros per row) |
| Memory access | Structured, cache-friendly | Random, cache-unfriendly |
| Performance bottleneck | Compute throughput | **Memory bandwidth** |
| MPI communication | Large panels, infrequent | **Small messages, very frequent** |
| Typical % of peak | 60–90% | **1–5%** |

That last row is the defining reality of HPCG. A machine that scores 10 Pflops on HPL might score only 200 Tflops on HPCG. The gap is not a flaw in the hardware — it is an honest reflection of how hard sparse memory-access patterns are for modern processor architectures, regardless of how many floating-point units they have.

This is why HPCG scores are increasingly weighted in competition rubrics. Any team can tune `N` and `NB` in an HPL config file. Optimizing HPCG requires deep understanding of memory hierarchy, network communication, and solver mathematics.

---

## What HPCG Actually Computes

HPCG constructs a 3D structured sparse matrix from a regular grid and then runs the following computational kernel repeatedly:

```
Solve: A·x = b   (approximately, using CG + multi-grid preconditioning)
```

Each iteration of the solver involves four kernels:

| Kernel | Operation | What it Stresses |
|---|---|---|
| **SpMV** | Sparse Matrix-Vector Multiply: `y = A·x` | Memory bandwidth (random access pattern) |
| **SYMGS (MG)** | Symmetric Gauss-Seidel sweep (preconditioner) | Memory bandwidth + data dependencies |
| **DDOT** | Dot product: `d = xᵀ·y` | Memory bandwidth + `MPI_Allreduce` latency |
| **WAXPBY** | Vector update: `w = α·x + β·y` | Memory bandwidth (streaming) |

The multi-grid preconditioner (MG) is by far the most complex. It recursively restricts the residual to progressively coarser grids (solving a series of smaller problems) before prolongating the result back to the fine grid. This is what makes SYMGS so expensive — the data dependencies in the Gauss-Seidel sweep prevent the solver from parallelizing within a single level, forcing it to execute sequentially through rows of the matrix.

---

## Part 1: Installation and Build

### Step 1: Clone the Source

The HPCG repository is hosted on GitHub. Clone it directly to the headnode:

```bash
cd ~
git clone https://github.com/hpcg-benchmark/hpcg.git
cd hpcg
```

Inspect the structure:

```bash
ls
```

You will see:
```
src/      setup/      bin/      tools/      CMakeLists.txt      configure
```

- `src/` — all C++ source files for the benchmark kernels
- `setup/` — platform configuration files (one per system type)
- `bin/` — where the compiled binary and default input file will live

---

### Step 2: Create Your Platform Configuration

HPCG uses a Makefile-based build system controlled by a platform file in `setup/`. The shipped templates assume system-default compilers. We need to create one that points explicitly to your source-compiled OpenMPI.

Copy the Linux MPI template as your starting point:

```bash
cp setup/Make.Linux_MPI setup/Make.custom_mpi
nano setup/Make.custom_mpi
```

Modify the following variables to match your custom OpenMPI installation:

```makefile
# ----------------------------------------------------------------------
# HPCG Platform Configuration: custom_mpi
# Target: Source-compiled OpenMPI on Ubuntu 22.04 x86_64
# ----------------------------------------------------------------------

MPdir        = $(HOME)/opt/openmpi
MPinc        = -I$(MPdir)/include
MPlib        = -L$(MPdir)/lib -lmpi

CXX          = $(MPdir)/bin/mpicxx
LINKER       = $(MPdir)/bin/mpicxx

CXXFLAGS     = $(HPCG_DEFS) -O3 -march=native -fopenmp
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

> **Why no OpenBLAS here?** Unlike HPL, HPCG does not call any BLAS routines. All of its kernels — SpMV, SYMGS, DDOT, WAXPBY — are implemented directly in C++ inside the `src/` directory. The performance lives in how well the compiler (`-O3 -march=native`) can auto-vectorize those loops, not in a BLAS library. OpenBLAS has nothing to offer here.

---

### Step 3: Build the Binary

Create an isolated build directory, configure it for your platform, and compile:

```bash
mkdir build
cd build
../configure custom_mpi
make
```

The build compiles approximately 40 source files using your custom `mpicxx` wrapper and links them into a single binary called `xhpcg` in `bin/`. This takes 1–3 minutes.

#### Verify the Build

```bash
ls -lh bin/xhpcg
ldd bin/xhpcg | grep mpi
```

The `ldd` output should reference your custom OpenMPI libraries in `~/opt/openmpi/lib/`.

> **About the compiler warnings:** During the build, you will see several warnings from the HPCG source about `printf` format overflow and unused return values. These are pre-existing issues in the HPCG source code itself — they are harmless and do not affect correctness or performance.

---

## Part 2: Configuration

### Step 4: Understanding `hpcg.dat`

Navigate to the binary directory and inspect the default input file:

```bash
cd bin
cat hpcg.dat
```

```
HPCG benchmark input file
Sandia National Laboratories; University of Tennessee, Knoxville
104 104 104
60
```

There are only two parameters that matter:

**Line 3 — Local Grid Dimensions (`nx ny nz`)**

This defines the size of the 3D grid that each individual MPI process owns. The total global problem size scales with the number of MPI processes:

```
Global nx = nx × npx   (npx is automatically determined from MPI process count)
Global ny = ny × npy
Global nz = nz × npz
```

Each grid point holds a double-precision floating-point value and several associated sparse matrix entries. The memory consumed by each MPI process is approximately:

```
Memory per process ≈ (nx × ny × nz) × 2.2 KB
```

The values must be **multiples of 8** (e.g., 32, 48, 64, 96, 104) and must be large enough that the multi-grid coarsening algorithm can subdivide them 3 times (each level halves the grid size). This requires the grid to be divisible by 2³ = 8.

**Line 4 — Execution Time Limit (seconds)**

HPCG runs CG iterations until this time limit is reached. More iterations mean more stable and representative performance numbers.

> **For official TOP500/HPCG-List submission**, the execution time must be at least **1800 seconds** (30 minutes). For validation and tuning, 60–180 seconds is practical.

---

### Step 5: Size the Grid for Your Cluster

Your students' cluster specifications:
- **Nodes:** 3 (headnode + compute-01 + compute-02)
- **Cores per node:** 3
- **Total MPI processes:** 9
- **RAM per node:** 4 GB

**Memory Budget Calculation:**

With 3 MPI processes per node and 4 GB per node, each process can safely use:

```
Available per process ≈ (4 GB × 0.65) / 3 ≈ 880 MB
```

*(Using 65% to leave headroom for the OS, MPI buffers, and the multi-grid coarse levels which also consume RAM)*

**Maximum safe local grid size:**

```
nx³ × 2.2 KB ≤ 880 MB
nx³ ≤ 880 × 1024² / 2200
nx³ ≤ 419,896
nx ≤ 74.8
```

Rounding down to the nearest multiple of 8:
```
nx = ny = nz = 72
```

For conservative validation runs during setup, **`64 64 64`** is safe and comfortable:

```
Memory per process = 64³ × 2.2 KB = 262,144 × 2200 B ≈ 576 MB
Memory per node (3 processes) = 3 × 576 MB ≈ 1.7 GB
```

This leaves over 2 GB of headroom per node. For a competition run where you want to maximize GFLOP/s (bigger grid = better cache utilization), push to `72 72 72`.

Update `hpcg.dat`:

```bash
nano hpcg.dat
```

```
HPCG benchmark input file
Sandia National Laboratories; University of Tennessee, Knoxville
64 64 64
180
```

Save and exit.

| Grid Size | Memory per Node (3 procs) | Use Case |
|---|---|---|
| `48 48 48` | ~0.95 GB | Quick debugging / smoke test |
| `64 64 64` | ~1.73 GB | **Validation — use this first** |
| `72 72 72` | ~2.47 GB | Performance run — good for competition |
| `104 104 104` | ~7.38 GB | **Too large — will OOM crash on 4 GB nodes** |

---

## Part 3: Running HPCG

### Step 6: Prepare the Hostfile

Create or copy the hostfile into the current directory with the correct slot count for your 3-node cluster:

```bash
nano hostfile
```

```
headnode    slots=3
compute-01  slots=3
compute-02  slots=3
```

Save and exit.

---

### Step 7: Set Environment Variables and Launch

```bash
# Prevent OpenMP from spawning extra threads that compete with MPI processes
export OMP_NUM_THREADS=1

# Launch HPCG across all 3 nodes (9 MPI processes total)
mpirun --hostfile hostfile -np 9 \
  --mca btl_tcp_if_include 10.100.0.0/24 \
  --mca oob_tcp_if_include 10.100.0.0/24 \
  -x OMP_NUM_THREADS=1 \
  ./xhpcg
```

HPCG will run silently until the time limit expires, then write its results file and exit. For 180 seconds, expect the run to take approximately 3–4 minutes total (including setup).

#### Verify Process Distribution

Before committing to a long run, verify all 9 processes spread correctly:

```bash
mpirun --hostfile hostfile -np 9 \
  --mca btl_tcp_if_include 10.100.0.0/24 \
  --mca oob_tcp_if_include 10.100.0.0/24 \
  --display-map \
  ./xhpcg
```

---

## Part 4: Reading and Interpreting Results

### Step 8: Open the Output File

When the run finishes, HPCG writes a timestamped results file:

```bash
cat HPCG-Benchmark_*.txt
```

The output is a structured YAML document. Let's walk through every important section.

---

### Problem Configuration Summary

```
Global Problem Dimensions::Global nx=128
Global Problem Dimensions::Global ny=128
Global Problem Dimensions::Global nz=64
Processor Dimensions::npx=2
Processor Dimensions::npy=2
Processor Dimensions::npz=1
```

This tells you how HPCG decomposed 4 MPI processes across the 3D grid. With 4 processes, it chose a 2×2×1 process topology, giving a global grid of 128×128×64. With 9 processes, you will see a 3×3×1 topology and a global grid of 192×192×64 (for `64 64 64` local dimensions).

---

### Memory Usage Summary

```
Memory Use Information::Total memory used for data (Gbytes)=0.750145
Memory Use Information::Bytes per equation (Total memory / Number of Equations)=715.394
```

This reports total distributed memory consumed across all MPI processes. On a 9-process, `64 64 64` run, expect roughly 5 GB total distributed memory (0.56 GB × 9 processes), confirming your grid sizing math is correct.

---

### Validation Summary — Read This First

```
Spectral Convergence Tests::Result=PASSED
Departure from Symmetry |x'Ay-y'Ax|...::Result=PASSED
Iteration Count Information::Result=PASSED
Reproducibility Information::Result=PASSED
```

All four validation checks must say **`PASSED`**. A `FAILED` here means the solver produced mathematically incorrect results and the GFLOP/s rating is meaningless. Common causes of failure:

- Unsafe compiler flags like `-ffast-math` (breaks IEEE floating-point compliance)
- OOM conditions causing silent memory corruption
- Incorrectly pinned MPI communication leading to partial hangs

---

### Performance Time Breakdown — Where Time Actually Goes

```
Benchmark Time Summary::DDOT=8.22525
Benchmark Time Summary::WAXPBY=4.06537
Benchmark Time Summary::SpMV=24.3603
Benchmark Time Summary::MG=151.191
Benchmark Time Summary::Total=187.883
```

Breaking this down as a percentage of total runtime:

| Kernel | Time (s) | % of Total | What it Means |
|---|---|---|---|
| **MG (Multi-Grid)** | 151.19 | **80.5%** | Sparse preconditioner — memory access at every grid level |
| **SpMV** | 24.36 | **13.0%** | Sparse matrix-vector multiply — random memory reads |
| **DDOT** | 8.23 | **4.4%** | Dot products — mostly MPI communication overhead |
| **WAXPBY** | 4.07 | **2.1%** | Vector updates — streaming memory, fastest kernel |

The MG kernel consuming 80% of total runtime is entirely expected and normal. This is the defining characteristic of HPCG across all hardware platforms. The preconditioner is inherently sequential in the Gauss-Seidel sweep direction — there is no way to parallelize within a single level without changing the algorithm. The only way to run MG faster is to have higher memory bandwidth.

---

### The Network Latency Penalty — The Most Important Diagnostic

```
DDOT Timing Variations::Min DDOT MPI_Allreduce time=5.93893
DDOT Timing Variations::Max DDOT MPI_Allreduce time=12.0651
DDOT Timing Variations::Avg DDOT MPI_Allreduce time=7.58217
```

Cross-reference this with the total DDOT time: **8.23 seconds**.

Of 8.23 seconds spent computing dot products, **7.58 seconds (92%)** was spent on `MPI_Allreduce` — waiting for all processes to agree on the global dot product value over the network.

The actual floating-point math of the dot product took only 0.65 seconds. The network was 13× more expensive than the arithmetic.

This is the signature of a high-latency interconnect. Every CG iteration requires multiple `MPI_Allreduce` calls (one per dot product), and each call forces all processes to synchronize. On a VM cluster sharing a laptop's network bridge, these messages travel through the hypervisor's virtual switch, adding significant overhead. On a physical cluster with InfiniBand, this same operation completes in microseconds rather than milliseconds.

The gap between `Min` (5.94s) and `Max` (12.07s) allreduce times is also significant — it shows **MPI jitter**. Some processes are occasionally delayed (by hypervisor scheduling, OS interrupts, or network contention), and all other processes must wait for the slowest one. This is another VM-specific artifact that disappears on bare-metal hardware.

---

### Bandwidth and GFLOP/s Summary

```
GB/s Summary::Raw Total B/W=15.5336
GFLOP/s Summary::Raw Total=2.04738
GFLOP/s Summary::Total with convergence and optimization phase overhead=1.94831
Final Summary::HPCG result is VALID with a GFLOP/s rating of=1.94831
```

**Memory Bandwidth: 15.5 GB/s** across 4 processes (≈3.9 GB/s per process). This is a realistic figure for DDR4 memory accessed through a hypervisor on a laptop — the theoretical bandwidth of DDR4-3200 is around 50 GB/s, so VM overhead is consuming roughly 90% of the theoretical bandwidth before HPCG even sees it.

**Official GFLOP/s Rating: 1.94831** — this is the number you submit and compare against. It intentionally includes all overheads (network latency, OS jitter, etc.) because those are real costs your scientific application would also experience.

---

### The Reference Kernel Warning

```
Final Summary::Reference version of ComputeDotProduct used=Performance results are most likely suboptimal
Final Summary::Reference version of ComputeSPMV used=Performance results are most likely suboptimal
Final Summary::Reference version of ComputeMG used=Performance results are most likely suboptimal
Final Summary::Reference version of ComputeWAXPBY used=Performance results are most likely suboptimal
```

HPCG explicitly tells you when you are using the reference (unoptimized) kernel implementations. This is a feature, not a bug — HPCG is designed to allow vendors to replace these reference implementations with hardware-specific optimized versions.

For example:
- **Intel** ships an optimized HPCG that replaces these kernels with AVX-512 intrinsic-based implementations and NUMA-aware memory allocation, typically achieving 3–5× higher GFLOP/s than the reference code on the same hardware.
- **NVIDIA** ships a GPU-accelerated HPCG (using CUDA) for their HPC GPU systems.

For this competition, you are running the reference CPU implementation, which is the standard baseline. The warning does not invalidate your result — it simply documents that hardware-vendor-specific optimizations have not been applied.

---

### Interpreting Efficiency

HPCG efficiency is measured relative to **memory bandwidth**, not floating-point peak. The key question is: what fraction of your available memory bandwidth are the kernels actually using?

```
Effective bandwidth utilized = Raw Total B/W / Theoretical DRAM peak
```

For a VM on a laptop with DDR4-3200:
- Theoretical bandwidth ≈ 50 GB/s (single channel)
- Measured bandwidth ≈ 15.5 GB/s (distributed across 4 ranks)
- Effective utilization ≈ 31%

The remaining 69% is consumed by hypervisor overhead, cache miss penalties, and the irregular access pattern of sparse matrix storage (which cannot be prefetched by the hardware prefetcher, unlike sequential dense arrays).

---

## Part 5: Tuning

### Increasing Local Grid Size

The single most effective tuning action on any HPCG run is to **increase the local grid size** until you are consuming the maximum amount of RAM safely. A larger grid means:
- More work per MPI collective call (amortizes latency cost)
- Better cache utilization across grid levels
- Higher absolute GFLOP/s rating

For your 3-node, 4 GB-per-node cluster, tune from `64 64 64` toward `72 72 72`:

```
HPCG benchmark input file
Sandia National Laboratories; University of Tennessee, Knoxville
72 72 72
180
```

---

### Running Longer for a Stable Score

More iterations produce a more representative score because transient OS noise averages out. For a final competition run, increase the time limit:

```
HPCG benchmark input file
Sandia National Laboratories; University of Tennessee, Knoxville
72 72 72
1800
```

> Note: A 1800-second run takes 30 minutes. You cannot submit an official HPCG score without at least 1800 seconds of execution time.

---

## Troubleshooting

| Symptom | Root Cause | Fix |
|---|---|---|
| OOM kill / node crashes during run | Grid size too large for node RAM | Reduce grid dimensions to `64 64 64` or `48 48 48` |
| `FAILED` on Spectral Convergence | Unsafe compiler flags or memory issue | Remove `-ffast-math` from `CXXFLAGS`; verify grid fits in RAM |
| All processes on headnode only | Incorrect hostfile or SSH issue | Verify `ssh compute-01 "echo OK"` works passwordlessly |
| `Open RTE: unable to open hostfile` | Wrong path to hostfile | Run from the directory containing `hostfile`, or use full absolute path |
| Very low GFLOP/s despite valid run | OpenMP oversubscribing cores | Confirm `export OMP_NUM_THREADS=1` before `mpirun` |
| `MPI_Allreduce` time ≫ DDOT time | High network latency between VM nodes | Expected on VM clusters. Pin MPI to `--mca btl_tcp_if_include 10.100.0.0/24` to avoid routing through NAT adapters |
| High variance between min/max allreduce | MPI jitter from hypervisor scheduling | Expected on VMs. Mitigated only on bare-metal hardware |

---

## Quick Reference

### Full Run Sequence

```bash
cd ~/hpcg/build/bin

export OMP_NUM_THREADS=1

mpirun --hostfile hostfile -np 9 \
  --mca btl_tcp_if_include 10.100.0.0/24 \
  --mca oob_tcp_if_include 10.100.0.0/24 \
  -x OMP_NUM_THREADS=1 \
  ./xhpcg

cat HPCG-Benchmark_*.txt
```

### Grid Size Reference for 3-Node, 4 GB Cluster

```
Grid       Mem/Node    Use Case
48 48 48   ~0.95 GB    Quick smoke test
64 64 64   ~1.73 GB    Validation and initial run
72 72 72   ~2.47 GB    Competition performance run
```

### What to Record from Results

| Metric | Where in Output |
|---|---|
| Official GFLOP/s rating | `Final Summary::HPCG result is VALID with a GFLOP/s rating of=` |
| Raw total GFLOP/s | `GFLOP/s Summary::Raw Total=` |
| Total memory bandwidth | `GB/s Summary::Raw Total B/W=` |
| MG time fraction | `Benchmark Time Summary::MG` ÷ `Total` |
| Network latency indicator | `DDOT Timing Variations::Avg DDOT MPI_Allreduce time` |
| Validation status | All four `Result=PASSED` lines |

---

## Summary

| Stage | What You Did |
|---|---|
| **Cloned HPCG** | Retrieved source directly from GitHub — no tarball needed |
| **Understood the benchmark** | HPCG tests memory bandwidth and MPI latency, not floating-point peak |
| **Created Make.custom_mpi** | Pointed the C++ compiler to your source-compiled OpenMPI wrappers |
| **Built `xhpcg`** | Compiled with `-O3 -march=native` — no BLAS library required |
| **Sized the grid correctly** | Calculated safe `nx ny nz` values from available RAM per process |
| **Executed and validated** | Confirmed all four mathematical validation checks passed |
| **Interpreted the results** | Understood why MG dominates runtime and why DDOT is network-bound |

> **Next:** Explore application benchmarks where the cluster runs a real scientific code — such as **WRF** (Weather Research and Forecasting) or **GROMACS** (molecular dynamics) — and compare throughput to your HPCC and HPCG scores to understand the gap between synthetic and real-world HPC performance.
