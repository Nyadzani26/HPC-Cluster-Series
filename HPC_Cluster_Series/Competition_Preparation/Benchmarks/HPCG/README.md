# Chapter 5 — Benchmarks: High-Performance Conjugate Gradient (HPCG)

> **Prerequisites:** You must have completed Chapter 3 (Competition Preparation) and Chapter 4 (HPCC) before starting this guide. You need a working OpenMPI compiler environment (`mpicxx`), passwordless SSH, and the shared home directory mounted across all nodes.

---

## What is HPCG?

The **High-Performance Conjugate Gradient (HPCG)** benchmark is designed to complement the High-Performance Linpack (HPL) benchmark. While HPL relies on dense matrix factorisation (which is highly compute-bound and fits modern CPU cache structures perfectly), HPCG uses sparse matrix operations that represent actual scientific applications much more closely.

HPCG is designed to test:
- **Memory Bandwidth**: The speed at which data is moved from RAM to CPU registers.
- **Network Latency**: High quantities of small messages sent via MPI collectives (`MPI_Allreduce`).
- **Preconditioner Performance**: Sparse triangular solvers (SYMGS).

Because scientific codes are usually memory-bound, a supercomputer's HPCG score is typically a tiny fraction of its HPL score (often only 1% to 3% of the theoretical peak). In competitions, maximizing your HPCG efficiency is one of the ultimate tests of system tuning.

---

## Part 1: Installation and Build

### Step 1: Clone the Source

Instead of downloading tarballs, clone the official HPCG benchmark source code from its GitHub repository directly to your headnode:

```bash
cd ~
git clone https://github.com/hpcg-benchmark/hpcg.git
cd hpcg
```

---

### Step 2: Configure the Build Makefile

HPCG utilizes platform configurations inside the `setup/` directory. We will configure a custom platform file named `Make.custom_mpi` to point the build system to your source-compiled OpenMPI compiler wrappers.

Copy the standard MPI configuration template:

```bash
cp setup/Make.Linux_MPI setup/Make.custom_mpi
```

Open the configuration file for editing:

```bash
nano setup/Make.custom_mpi
```

Locate the compiler and directory variables, and update them to use your custom OpenMPI installation. Change the lines to match the following configuration:

```makefile
# ----------------------------------------------------------------------
# HPCG Custom OpenMPI Configuration
# ----------------------------------------------------------------------
MPdir        = $(HOME)/opt/openmpi
MPinc        = -I$(MPdir)/include
MPlib        = -L$(MPdir)/lib -lmpi

# Use custom compiler wrappers
CXX          = $(MPdir)/bin/mpicxx
LINKER       = $(MPdir)/bin/mpicxx

# Compiler flags: -O3 for optimizations, -march=native for host-specific architecture
CXXFLAGS     = -I$(HPCG_DEFS) -O3 -march=native -fopenmp
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

---

### Step 3: Build the Binary

Create a separate build directory to keep the source tree clean, configure it, and compile the code:

```bash
mkdir build
cd build
../configure custom_mpi
make
```

The compilation process will take 1–3 minutes. Once completed, a binary named `xhpcg` will be generated inside the `bin` directory.

#### Verify the Build

```bash
ls -lh bin/xhpcg
ldd bin/xhpcg | grep mpi
```

Ensure the library paths point to your custom OpenMPI directory (`~/opt/openmpi/lib/libmpi.so`).

---

## Part 2: Configuration

### Step 4: Configure the Input (`hpcg.dat`)

Navigate to the binary directory:

```bash
cd bin
```

The parameters inside `hpcg.dat` control the grid dimensions and execution duration. Open it for editing:

```bash
nano hpcg.dat
```

By default, the file looks like this:
```
HPCG benchmark input file
Sandia National Laboratories; University of Tennessee, Knoxville
104 104 104
60
```

#### Modifying the Grid Dimensions for Your Cluster

HPCG defines the problem size using grid dimensions **per MPI process**. The grid size must satisfy the following constraints:
- The local grid dimensions (`nx`, `ny`, `nz`) must be multiples of 8 (e.g. 48, 64, 96, 104).
- The memory consumption scales rapidly: each grid point consumes roughly **2.2 KB of RAM**.

Your 3-node, 3-core-per-node cluster specifications:
- **Total MPI processes:** 9 (3 processes per node)
- **RAM per node:** 4 GB (12 GB total)

To maximize memory utilization without triggering the Linux Out-Of-Memory (OOM) killer, we target utilizing ~60-70% of available RAM per node.

For a local grid of **`64 64 64`**:
$$\text{Memory per process} = 64^3 \times 2.2 \text{ KB} \approx 262{,}144 \times 2200 \text{ bytes} \approx 576 \text{ MB}$$
$$\text{Memory per node (3 processes)} = 3 \times 576 \text{ MB} \approx 1.7 \text{ GB}$$

This is extremely safe and fits comfortably within your 4 GB RAM limit, leaving plenty of memory for the operating system and MPI buffers.

Update `hpcg.dat` to use `64 64 64` and set the runtime to **`180`** seconds (3 minutes) for a thorough validation run:

```
HPCG benchmark input file
Sandia National Laboratories; University of Tennessee, Knoxville
64 64 64
180
```

> **For official submission benchmarks**, the runtime limit must be set to at least **`1800`** seconds (30 minutes). Use `180` for validation testing.

---

## Part 3: Running HPCG

### Step 5: Copy the Hostfile

Ensure you have a copy of your cluster's hostfile in the current directory. If you compiled it during the HPL configuration, you can copy it over:

```bash
cp ~/hpcc-1.5.0/hostfile .
```

Verify that the hostfile correctly defines 3 slots per node:
```
headnode    slots=3
compute-01  slots=3
compute-02  slots=3
```

---

### Step 6: Launch the Benchmark

To prevent core thrashing, we restrict OpenMP threads to 1 per MPI process. Run the following command:

```bash
export OMP_NUM_THREADS=1

mpirun --hostfile hostfile -np 9 \
  --mca btl_tcp_if_include 10.100.0.0/24 \
  --mca oob_tcp_if_include 10.100.0.0/24 \
  -x OMP_NUM_THREADS=1 \
  ./xhpcg
```

---

## Part 4: Reading and Interpreting Results

### Step 7: Inspect the Output

When HPCG completes, it generates a YAML-formatted text file in the same directory, named `HPCG-Benchmark_<version>_<date>.txt`. Open the file:

```bash
cat HPCG-Benchmark_*.txt
```

Verify the following key parameters in the output:

1. **Validation Check**:
   Under `Spectral Convergence Tests::Result`, it must say **`PASSED`**.
   Under `Departure from Symmetry ::Result`, it must say **`PASSED`**.
   Under `Iteration Count Information::Result`, it must say **`PASSED`**.
   Under `Reproducibility Information::Result`, it must say **`PASSED`**.

2. **The GFLOP/s Rating**:
   Look at the final section for your score:
   ```
   Final Summary::HPCG result is VALID with a GFLOP/s rating of = X.XXXX
   ```

3. **Bottleneck Analysis**:
   Look at the **`DDOT Timing Variations`** section. If the max `MPI_Allreduce` time is very high compared to the min time, it points to network load imbalance or latency issues on your private interface.

---

## Troubleshooting

| Symptom | Root Cause | Fix |
|---|---|---|
| `Open RTE was unable to open the hostfile` | Relative path to hostfile is incorrect | Ensure the hostfile exists in the directory you are executing `mpirun` from, or use the absolute path. |
| Process killed / `Out of Memory` | Local grid size is too large for the node RAM | Lower `hpcg.dat` dimensions (e.g. from `104 104 104` to `64 64 64` or `48 48 48`). |
| `Validation FAILED` | Floating point corruption or runtime instability | Ensure `-x OMP_NUM_THREADS=1` is exported and compile flags do not use unsafe math optimizations like `-ffast-math`. |
| Extremely low GFLOP/s | CPU cores oversubscribed | Ensure `OMP_NUM_THREADS` is set to `1` so processes do not compete for cores. |

---

## Summary

| Stage | What You Did |
|---|---|
| **Cloned HPCG** | Downloaded the benchmark source code from GitHub |
| **Created Make.custom_mpi** | Pointed the compiler to your optimized source-built OpenMPI |
| **Compiled `xhpcg`** | Built the customized parallel binary |
| **Tuned `hpcg.dat`** | Sized the grid sizes per core to safely utilize available RAM |
| **Executed Benchmark** | Ran a multi-node MPI execution using network interface pinning |
| **Verified Results** | Checked that all calculations validated successfully |
