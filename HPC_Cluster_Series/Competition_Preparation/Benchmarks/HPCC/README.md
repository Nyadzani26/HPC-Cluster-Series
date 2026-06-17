# Chapter 4 — Benchmarks: HPC Challenge (HPCC)

> **Prerequisites:** You must have completed Chapter 3 (Competition Preparation) before starting this guide. Specifically, you need:
> - OpenMPI compiled from source and installed under `~/opt/openmpi/`
> - OpenBLAS compiled from source and installed under `~/opt/openblas/`
> - Your `$PATH` and `$LD_LIBRARY_PATH` set correctly in `~/.bashrc`
> - Passwordless SSH from the headnode to all compute nodes
> - NFS home directory mounted on all compute nodes

---

## What is HPCC?

**HPC Challenge (HPCC)** is a suite of benchmarks designed to stress-test HPC clusters across a wide range of hardware capabilities. It was created by Jack Dongarra and Piotr Luszczek at the University of Tennessee's Innovative Computing Laboratory, and is one of the primary tools used at real competition events like the Student Cluster Competition (SCC).

Unlike HPL, which only tests raw floating-point throughput, HPCC provides a **holistic picture** of your cluster by measuring seven distinct aspects of performance simultaneously in a single run:

| Benchmark | Metric | What It Measures |
|---|---|---|
| **HPL** | Gflops | Floating-point throughput (dense matrix solve) |
| **DGEMM** | Gflops | Double-precision matrix-matrix multiply |
| **STREAM** | GB/s | Memory bandwidth (sustainable memory throughput) |
| **PTRANS** | GB/s | Parallel matrix transposition (network + memory) |
| **RandomAccess** | GUPs | Random memory access (GIGA Updates per Second) |
| **FFT** | Gflops | Fast Fourier Transform throughput |
| **b_eff Latency** | µs / GB/s | MPI latency and bandwidth across the network |

In competition scenarios, judges use the HPCC results table to compare teams' clusters and optimisation strategies.

---

## Understanding the Benchmark Components

### HPL (High-Performance Linpack)

HPL is the same benchmark used to rank the TOP500 supercomputers list. It solves a dense system of linear equations $Ax = b$ using LU factorisation with partial pivoting. The result — **Rmax** — represents your cluster's sustained double-precision floating-point throughput in Gflops (or Tflops at scale).

HPL is compute-bound and tests:
- Your BLAS library quality (OpenBLAS, MKL, etc.)
- MPI communication efficiency
- Memory bandwidth during matrix operations

### DGEMM (Double-precision GEneral Matrix Multiply)

DGEMM runs the core BLAS Level 3 operation in a single-node, non-MPI mode (the "EP" or Embarrassingly Parallel variant). This isolates your BLAS library's single-node matrix multiplication performance without MPI overhead.

### STREAM

STREAM measures **memory bandwidth** — how fast your system can transfer data between the CPU and RAM. It runs four kernels: Copy, Scale, Add, and Triad. In HPC, memory bandwidth is often the real bottleneck for scientific codes, even on high-Gflops hardware.

The formula for the Triad result:
```
STREAM Triad (GB/s) = (3 × N × 8 bytes) / time (seconds)
```

### PTRANS (Parallel Transposition)

PTRANS moves a large matrix across MPI ranks by transposing it. This directly exercises **inter-node network bandwidth** and tests how efficiently your cluster's interconnect handles large collective data movements.

### RandomAccess (GUPS)

RandomAccess is specifically designed to break your hardware. It performs billions of random 64-bit read-modify-write operations on a large table distributed across all MPI ranks. It measures Giga Updates per Second (GUPs) and is expected to score very low on all clusters — this is normal.

### FFT (Fast Fourier Transform)

FFT is run in two modes: **StarFFT** (single-node) and **MPIFFT** (distributed across all ranks). The distributed FFT requires intensive all-to-all communication, making it a sensitive test of both network latency and bandwidth simultaneously.

### Latency and Bandwidth (b_eff)

This tests MPI point-to-point communication performance between nodes. It measures:
- **Latency**: The time (in microseconds) for a tiny message to travel from one rank to another and back
- **Bandwidth**: The peak data rate (GB/s) for large messages

Low latency and high bandwidth are desirable. On a gigabit Ethernet VM cluster, expect 50–200 µs latency and 0.5–2 GB/s bandwidth.

---

## Part 1: Installation and Build

### Step 1: Download HPCC Source

HPCC is downloaded as a single tarball containing all benchmark source trees. Download the latest stable release:

```bash
cd ~
wget https://icl.utk.edu/projectsfiles/hpcc/download/hpcc-1.5.0.tar.gz
tar -xzf hpcc-1.5.0.tar.gz
cd hpcc-1.5.0
```

> **Why 1.5.0?** HPCC 1.5.0 is the most widely used version in competition and research settings. It is compatible with OpenMPI 4.x and modern GCC/gfortran toolchains.

Inspect the directory structure:

```bash
ls
```

You will see:
```
DGEMM/    FFT/    PTRANS/    RandomAccess/    STREAM/    hpl/    include/    src/    Makefile
```

Each directory contains the source for one benchmark component. The top-level `Makefile` orchestrates building them all together using a single architecture configuration file.

---

### Step 2: Understand the Build System

HPCC's build system is inherited from HPL. Every aspect of the build — which compilers to use, which BLAS and MPI libraries to link against, which compiler flags to apply — is controlled by a single **Make architecture file**.

These files live in:
```
hpl/setup/Make.<arch>
```

HPCC ships with many example configurations. Look at what is available:

```bash
ls hpl/setup/
```

You will see files like `Make.Linux_PII_CBLAS`, `Make.UNKNOWN`, etc. None of these will work out-of-the-box for your custom OpenBLAS + OpenMPI build. You need to create your own.

---

### Step 3: Create Your Architecture Configuration File

The configuration file must be placed at `hpl/Make.<arch>` where `<arch>` is a short name you choose. We will use `custom`.

Create the file:

```bash
nano ~/hpcc-1.5.0/hpl/Make.custom
```

Paste the following content exactly:

```makefile
# ----------------------------------------------------------------------
# HPCC Architecture Configuration: custom
# Target: Source-compiled OpenBLAS + OpenMPI on Ubuntu 22.04 x86_64
# ----------------------------------------------------------------------

SHELL        = /bin/sh
CD           = cd
CP           = cp
LN_S         = ln -fs
MKDIR        = mkdir -p
RM           = /bin/rm -f
TOUCH        = touch
ARCH         = custom

# ----------------------------------------------------------------------
# HPL Directory Structure
# ----------------------------------------------------------------------
# NOTE: If you renamed your main directory to ~/hpcc instead of ~/hpcc-1.5.0, 
# change $(HOME)/hpcc-1.5.0/hpl to $(HOME)/hpcc/hpl below.
TOPdir       = $(HOME)/hpcc-1.5.0/hpl
INCdir       = $(TOPdir)/include
BINdir       = $(TOPdir)/bin/$(ARCH)
LIBdir       = $(TOPdir)/lib/$(ARCH)
HPLdir       = $(TOPdir)
HPLlib       = $(LIBdir)/libhpl.a

# ----------------------------------------------------------------------
# MPI Configuration
# Point to your source-compiled OpenMPI installation
# ----------------------------------------------------------------------
MPdir        = $(HOME)/opt/openmpi
MPinc        = -I$(MPdir)/include
MPlib        = -L$(MPdir)/lib -lmpi

# ----------------------------------------------------------------------
# BLAS Configuration
# Point to your source-compiled OpenBLAS installation
# ----------------------------------------------------------------------
LAdir        = $(HOME)/opt/openblas
LAinc        =
LAlib        = -L$(LAdir)/lib -lopenblas

# ----------------------------------------------------------------------
# Compiler Configuration
# Use mpicc from your custom OpenMPI installation
# ----------------------------------------------------------------------
CC           = $(MPdir)/bin/mpicc
CCNOOPT      = $(HPL_DEFS)
CCFLAGS      = $(HPL_DEFS) -O3 -march=native -fomit-frame-pointer -funroll-loops -fopenmp

LINKER       = $(MPdir)/bin/mpicc
LINKFLAGS    = $(CCFLAGS)

ARCHIVER     = ar
ARFLAGS      = r
RANLIB       = echo

# ----------------------------------------------------------------------
# Preprocessor Macros
# ----------------------------------------------------------------------
HPL_OPTS     = -DHPL_CALL_CBLAS
HPL_DEFS     = $(HPL_OPTS) -I$(INCdir) $(MPinc) $(LAinc)

# Runtime linker paths
HPL_LIBS     = $(HPLlib) $(LAlib) $(MPlib) -lm
```

Save and exit: `Ctrl+X`, `Y`, `Enter`.

> **Key variables explained:**
> - `TOPdir` — root of the HPL source subdirectory (must be an absolute path pointing to the `hpl` folder)
> - `HPLlib` — points to the compiled HPL static library
> - `MPdir` — where your OpenMPI was installed
> - `LAdir` — where your OpenBLAS was installed
> - `-march=native` — generates code optimised for the exact CPU in the machine compiling it (enables AVX, SSE4.2, etc.)
> - `-fopenmp` — enables OpenMP threading, required even if you set `OMP_NUM_THREADS=1` at runtime
> - `-DHPL_CALL_CBLAS` — tells HPL to call the C interface to BLAS (CBLAS) rather than the Fortran interface

---

### Step 4: Build HPCC

Before compiling, there is an important compatibility step. HPCC includes legacy code that uses old MPI-1 functions (`MPI_Address` and `MPI_Type_struct`) which were deprecated and removed in MPI-3.0 (and thus fail to compile on modern OpenMPI 4.x/5.x by default). 

#### 4a. Update legacy MPI functions
To make the codebase compatible with modern OpenMPI, run the following `sed` commands from your terminal to update the source files to use modern replacements (`MPI_Get_address` and `MPI_Type_create_struct`):

```bash
# Replace MPI_Address with MPI_Get_address
find ~/hpcc-1.5.0/ -type f \( -name "*.c" -o -name "*.h" \) -exec sed -i 's/MPI_Address/MPI_Get_address/g' {} +

# Replace MPI_Type_struct with MPI_Type_create_struct
find ~/hpcc-1.5.0/ -type f \( -name "*.c" -o -name "*.h" \) -exec sed -i 's/MPI_Type_struct/MPI_Type_create_struct/g' {} +
```
*(If you renamed your main directory to `hpcc`, replace `hpcc-1.5.0` with `hpcc` in the paths above).*

#### 4b. Sync architecture file and compile HPL first
HPCC's main build system does not build the underlying HPL engine library automatically. We must copy the architecture file to the root of the source tree, build the HPL subdirectory first, and then build the top-level binary.

```bash
# 1. Sync the Make.custom file to the root directory
cp ~/hpcc-1.5.0/hpl/Make.custom ~/hpcc-1.5.0/Make.custom

# 2. Build the HPL engine first
cd ~/hpcc-1.5.0/hpl
make arch=custom clean
make arch=custom

# 3. Build the final HPCC binary
cd ~/hpcc-1.5.0
make arch=custom
```

#### Verify the Build Succeeded

```bash
ls -lh ~/hpcc-1.5.0/hpcc
```

Expected output:
```
-rwxrwxr-x 1 ubuntu ubuntu 799K Jun 16 23:22 /home/ubuntu/hpcc-1.5.0/hpcc
```

> **If the build fails with `undefined reference to 'main'`**, the linker ran before all object files were compiled. Clean and rebuild:
>
> ```bash
> rm -rf ~/hpcc-1.5.0/hpl/lib/custom ~/hpcc-1.5.0/hpcc
> make arch=custom
> ```

#### Check Library Dependencies

Verify the binary links against your custom libraries (not system ones):

```bash
ldd ~/hpcc-1.5.0/hpcc | grep -E "openblas|mpi"
```

Expected output (paths will include your home directory):
```
libopenblas.so.0 => /home/ubuntu/opt/openblas/lib/libopenblas.so.0 (0x...)
libmpi.so.40     => /home/ubuntu/opt/openmpi/lib/libmpi.so.40 (0x...)
```

If you see `/usr/lib/...` paths instead, your `$LD_LIBRARY_PATH` is not set correctly. Refer to Section 3.1 in the Competition Preparation guide.

---

## Part 2: Configuration

### Step 5: Set Up the Input File (`hpccinf.txt`)

HPCC reads all benchmark parameters from a file named **`hpccinf.txt`** in the **current working directory** at runtime. Copy the template:

```bash
cp ~/hpcc-1.5.0/_hpccinf.txt ~/hpcc-1.5.0/hpccinf.txt
nano ~/hpcc-1.5.0/hpccinf.txt
```

---

### Step 6: Configure Parameters for Your Cluster

The parameters in `hpccinf.txt` control how large a problem HPCC attempts to solve. Setting them incorrectly results in either a wasted run (problem too small) or an out-of-memory crash (problem too large).

Your cluster specifications:
- **Nodes:** 1 headnode + 2 compute nodes = 3 nodes total
- **Cores per node:** 3
- **Total MPI processes:** 9 (`3 nodes × 3 cores`)
- **RAM per node:** 4 GB
- **Total RAM:** 12 GB

#### The Critical Parameter: N (Problem Size)

`N` is the dimension of the square matrix that HPL solves. The matrix occupies `N² × 8 bytes` of RAM distributed across all MPI ranks.

The standard formula for choosing `N` is to use approximately **80% of total available RAM**:

```
N = sqrt(0.80 × Total_RAM_bytes / 8)
N = sqrt(0.80 × 12 × 1073741824 / 8)
N = sqrt(1,288,490,188)
N ≈ 35,895
```

Round down to the nearest multiple of NB (224):
```
N = floor(35895 / 224) × 224 = 160 × 224 = 35,840
```

> **For a quick initial test run**, use a much smaller N (like 10000 or 15000) to verify the configuration is correct before committing to the full-size problem.

#### The Grid Parameters: P and Q

HPL distributes the matrix across a **P × Q process grid**. The constraint is:
```
P × Q = total MPI processes = 9
```

For best performance, P and Q should be as close to equal as possible (a near-square grid), with Q ≥ P:

| P × Q | Total Processes | Notes |
|---|---|---|
| 1 × 9 | 9 | Very rectangular — poor communication |
| 3 × 3 | 9 | **Square — optimal for HPL** |

Use **P=3, Q=3**.

#### The Block Size: NB

NB is the panel size used during LU factorisation. It controls how the matrix is blocked for cache efficiency. For most modern x86_64 CPUs with OpenBLAS:
- NB between **128 and 256** is a good starting range
- Values that are multiples of 64 align well with cache line sizes
- **NB = 224** is a well-tuned starting point

#### Your Full `hpccinf.txt`

```
HPLinpack benchmark input file
Innovative Computing Laboratory, University of Tennessee
HPL.out      output file name (if any)
8            device out (6=stdout,7=stderr,file)
1            # of problems sizes (N)
35840        Ns
1            # of NBs
224          NBs
0            PMAP process mapping (0=Row-,1=Column-major)
1            # of process grids (P x Q)
3            Ps
3            Qs
16.0         threshold
1            # of panel fact
2            PFACTs (0=left, 1=Crout, 2=Right)
1            # of recursive panel fact.
1            RFACTs (0=left, 1=Crout, 2=Right)
1            # of broadcast
1            BCASTs (0=1rg,1=1rM,2=2rg,3=2rM,4=Lng,5=LnM)
1            # of lookahead depth
1            DEPTHs (0=none,1=optimised)
2            SWAP (0=bin-exch,1=long,2=mix)
64           swapping threshold
0            L1 in (0=transposed,1=no-transposed) form
0            U  in (0=transposed,1=no-transposed) form
1            Equilibration (0=no,1=yes)
8            memory alignment in double (> 0)
##### This line (100 chars) is used to separate sections of this file ###############################
T/V    N    NB     P     Q               Time                 Gflops
#######################################################################################################
```

Save and exit: `Ctrl+X`, `Y`, `Enter`.

---

### Step 7: Parameter Reference Table

| Parameter | Recommended Value | Why |
|---|---|---|
| N | 35840 | 80% of 12 GB total RAM, rounded to NB multiple |
| NB | 224 | Cache-tuned for x86_64 CPUs with OpenBLAS |
| PMAP | 0 | Row-major (standard for square grids) |
| P | 3 | P × Q = 9 total processes |
| Q | 3 | P × Q = 9 total processes |
| PFACT | 2 | Right-looking (fastest for most hardware) |
| RFACT | 1 | Crout factorisation |
| BCAST | 1 | 1-ring modified broadcast |
| DEPTH | 1 | One level of lookahead (hides latency) |

---

## Part 3: Running HPCC

### Step 8: Verify the Environment

Before launching, confirm everything is in order:

```bash
# Verify custom MPI is active on headnode
which mpirun
# Expected: /home/ubuntu/opt/openmpi/bin/mpirun

# Verify it reaches compute nodes
ssh compute-01 "which mpirun && echo OK"
ssh compute-02 "which mpirun && echo OK"
```

If any command fails or returns a system path (`/usr/bin/mpirun`), ensure your `~/.bashrc` exports are placed **before** the `case $-` interactive guard. Refer to Section 3.1.

---

### Step 9: Create the Hostfile

HPCC uses MPI, so it needs a hostfile to know which nodes and how many processes to spawn per node:

```bash
nano ~/hpcc-1.5.0/hostfile
```

For the full 9-process, 3-node run:

```
headnode    slots=3
compute-01  slots=3
compute-02  slots=3
```

Save and exit.

> **`slots`** tells OpenMPI how many processes to spawn on each node. With 3 cores per node × 3 nodes = 9 total processes, matching your P×Q=3×3 grid.

---

### Step 10: Set Environment Variables

Two critical environment variables must be set before launching:

```bash
# Tell OpenBLAS to use exactly 1 thread per MPI process.
# Without this, each of your 9 MPI processes spawns additional threads,
# overloading your 3 physical cores per node with context switching.
export OPENBLAS_NUM_THREADS=1

# Tell OpenMP (used internally by some benchmark components) to do the same
export OMP_NUM_THREADS=1
```

---

### Step 11: Launch HPCC

Run HPCC from the directory containing `hpccinf.txt`:

```bash
cd ~/hpcc-1.5.0

export OPENBLAS_NUM_THREADS=1
export OMP_NUM_THREADS=1

mpirun --hostfile hostfile -np 9 \
  --mca btl_tcp_if_include 10.100.0.0/24 \
  --mca oob_tcp_if_include 10.100.0.0/24 \
  -x OPENBLAS_NUM_THREADS=1 \
  -x OMP_NUM_THREADS=1 \
  ./hpcc
```

| Flag | Purpose |
|---|---|
| `--hostfile hostfile` | Which nodes to use |
| `-np 9` | Total MPI processes (must equal P×Q in hpccinf.txt) |
| `--mca btl_tcp_if_include 10.100.0.0/24` | Pin MPI data transfers to the private cluster network |
| `--mca oob_tcp_if_include 10.100.0.0/24` | Pin MPI daemon messages to the private cluster network |
| `-x OPENBLAS_NUM_THREADS=1` | Export this variable to all remote MPI processes |
| `-x OMP_NUM_THREADS=1` | Export this variable to all remote MPI processes |

> **Why the `--mca` flags?** Your headnode has two network interfaces: the NAT adapter (`192.168.x.x`) and the private cluster adapter (`10.100.0.x`). Without explicit direction, OpenMPI may try to route daemon startup messages through the NAT interface. Compute nodes have no route to `192.168.x.x` and will immediately reject these connections, causing the run to abort. Always pin MPI to `10.100.0.0/24`.

Expected runtime for a full-size N=35840 problem on 9 processes: **5–15 minutes** depending on your VM performance and host laptop hardware.

---

### Step 12: Monitoring the Run

While HPCC is running, open separate SSH sessions to each node and monitor CPU usage with `htop`:

```bash
# Second terminal
ssh ubuntu@10.100.0.11
htop

# Third terminal
ssh ubuntu@10.100.0.12
htop
```

During the HPL phase (matrix factorisation), **all CPU cores on all three nodes should be at 100% utilisation simultaneously**. If some nodes are idle, MPI is not distributing work correctly.

Verify process distribution before the full run using `--display-map`:

```bash
mpirun --hostfile hostfile -np 9 \
  --mca btl_tcp_if_include 10.100.0.0/24 \
  --mca oob_tcp_if_include 10.100.0.0/24 \
  --display-map \
  ./hpcc
```

Expected output at the start:
```
 ========================   JOB MAP   ========================

 Data for node: headnode        Num slots: 3    Num procs: 3
        Process rank: 0
        Process rank: 1
        Process rank: 2

 Data for node: compute-01      Num slots: 3    Num procs: 3
        Process rank: 3
        Process rank: 4
        Process rank: 5

 Data for node: compute-02      Num slots: 3    Num procs: 3
        Process rank: 6
        Process rank: 7
        Process rank: 8

 =============================================================
```

If any node shows `Num procs: 0`, check passwordless SSH and that the hostname in `hostfile` exactly matches the entry in `/etc/hosts`.

---

## Part 4: Reading the Results

### Step 13: The Output File (`hpccoutf.txt`)

When HPCC finishes, all results are written to `hpccoutf.txt` in the current directory:

```bash
cat ~/hpcc-1.5.0/hpccoutf.txt
```

The file contains hundreds of key=value pairs. The most important ones are:

```
HPL_Tflops=0.0480
PTRANS_GBs=0.0195
MPIRandomAccess_GUPs=0.0007
MPIFFT_Gflops=0.0213
StarSTREAM_Triad=2.8158
StarDGEMM_Gflops=23.5681
RandomlyOrderedRingBandwidth_GBytes=0.8421
RandomlyOrderedRingLatency_usec=138.4
```

> Always check the `_Pass` or `_Errors` fields for each benchmark. A non-zero error count invalidates that benchmark's result.

---

### Step 14: Format the Output with the Perl Script

HPCC ships with a Perl formatting script. If you do not already have it, create it:

```bash
nano ~/hpcc-1.5.0/hpccoutf.pl
```

Paste the following:

```perl
#! /usr/bin/perl
#
# Usage: hpccoutf.pl -a -f hpccoutf.txt   To show all parameters
#        hpccoutf.pl -w -f hpccoutf.txt   To show web parameters
#

use strict;
use Getopt::Std;
$Getopt::Std::STANDARD_HELP_VERSION = 1;
our ( $opt_a, $opt_w, $opt_f, $value, $count, $key ) = 0;
getopts("awf:");

unless ( $opt_a || $opt_w && $opt_f ) {
    print "\n";
    print "Usage: $0 -a -f hpccoutf.txt For all parameters\n";
    print "       $0 -w -f hpccoutf.txt For web parameters\n";
    exit;
}

$/ = undef;
open HPPCOUTF, $opt_f or die $!;
my $file     = <HPPCOUTF>;
my %hpccoutf = $file =~ /^(\w+)=(\d.*)$/mg;
close HPPCOUTF;

my @walkorder = (
    'HPL_Tflops',                      'PTRANS_GBs',
    'MPIRandomAccess_GUPs',            'MPIFFT_Gflops',
    'StarSTREAM_Triad*CommWorldProcs', 'StarSTREAM_Triad',
    'StarDGEMM_Gflops',                'RandomlyOrderedRingBandwidth_GBytes',
    'RandomlyOrderedRingLatency_usec'
);

my @walkunits = (
    'Tera Flops per Second',           'Tera Bytes per Second',
    'Giga Updates per Second',         'Tera Flops per Second',
    'Tera Bytes per Second',           'Giga Bytes per Second',
    'Giga Flops per Second',           'Giga Bytes per Second',
    'Micro Seconds'
);

my @walkweb = (
    'G-HPL',              'G-PTRANS',
    'G-RandomAccess',     'G-FFT',
    'EP-STREAM Sys',      'EP-STREAM Triad',
    'EP-DGEMM',           'RandomRing Bandwidth',
    'RandomRing Latency'
);

if ($opt_w) {
    printf "\n";
    printf "-" x 98 . "\n";
    printf "%-40s %-30s %-8s %s\n", "HPCCOUTF NAME", "WEB NAME", "VALUE", "UNITS";
    printf "-" x 98 . "\n";

    for $count ( 0 .. $#walkorder ) {
        $key   = $walkorder[$count];
        $value = $hpccoutf{$key} // 'N/A';
        printf "%-40s %-30s %-8s %s\n",
          $key, $walkweb[$count], $value, $walkunits[$count];
    }
    printf "-" x 98 . "\n\n";
}

if ($opt_a) {
    printf "\n";
    printf "-" x 60 . "\n";
    printf "%-40s %s\n", "HPCCOUTF NAME", "VALUE";
    printf "-" x 60 . "\n";
    for $key ( sort keys %hpccoutf ) {
        printf "%-40s %s\n", $key, $hpccoutf{$key};
    }
    printf "-" x 60 . "\n\n";
}
```

Save and exit, then make it executable:

```bash
chmod +x ~/hpcc-1.5.0/hpccoutf.pl
```

Run the formatter:

```bash
cd ~/hpcc-1.5.0
perl hpccoutf.pl -w -f hpccoutf.txt
```

Expected output format:

```
--------------------------------------------------------------------------------------------------
HPCCOUTF NAME                        WEB NAME                      VALUE   UNITS
--------------------------------------------------------------------------------------------------
HPL_Tflops                           G-HPL                        0.0480   Tera Flops per Second
PTRANS_GBs                           G-PTRANS                     0.0195   Tera Bytes per Second
MPIRandomAccess_GUPs                 G-RandomAccess               0.0007   Giga Updates per Second
MPIFFT_Gflops                        G-FFT                        0.0213   Tera Flops per Second
StarSTREAM_Triad*CommWorldProcs      EP-STREAM Sys                0.0113   Tera Bytes per Second
StarSTREAM_Triad                     EP-STREAM Triad              2.8158   Giga Bytes per Second
StarDGEMM_Gflops                     EP-DGEMM                    23.5681   Giga Flops per Second
RandomlyOrderedRingBandwidth_GBytes  RandomRing Bandwidth         0.8421   Giga Bytes per Second
RandomlyOrderedRingLatency_usec      RandomRing Latency         138.4000   Micro Seconds
--------------------------------------------------------------------------------------------------
```

---

### Step 15: Interpreting Your Results

| Metric | What You Measured | Baseline Expectation (3-node VM cluster) | Bottleneck if Low |
|---|---|---|---|
| **HPL (G-HPL)** | Sustained Gflops across all 9 cores | ~0.04–0.10 Tflops | BLAS library quality, NB tuning, RAM speed |
| **DGEMM (EP-DGEMM)** | Single-node BLAS throughput | ~20–35 Gflops | OpenBLAS build flags, `-march=native` flag |
| **STREAM (EP-STREAM Triad)** | Memory bandwidth per node | ~2–5 GB/s | VM memory allocation, NUMA effects |
| **PTRANS (G-PTRANS)** | Network throughput | ~0.01–0.05 TB/s | NIC speed, MPI collective tuning |
| **RandomAccess (G-RandomAccess)** | Random memory access | ~0.0005–0.002 GUPs | Always very low — expected and normal |
| **FFT (G-FFT)** | FFT throughput | ~0.01–0.05 Gflops | Memory bandwidth, FFT plan size |
| **Ring Latency** | MPI message latency | ~50–200 µs | Network, VM overhead, TCP stack |
| **Ring Bandwidth** | MPI bandwidth | ~0.5–2 GB/s | NIC speed, TCP congestion |

**HPL Efficiency** is the ratio of your actual HPL score to the theoretical peak of your hardware:

```
Efficiency (%) = (R_max / R_peak) × 100

Where:
  R_peak = Total_cores × Clock_speed_GHz × FLOPS_per_cycle
         = 9 cores × ~2.5 GHz × 8 FLOPS/cycle (AVX, double-precision)
         = ~180 Gflops (theoretical maximum)
```

For a VM cluster with custom-compiled OpenBLAS, achieving **60–80% efficiency** is excellent. Below 50% suggests tuning opportunities.

---

## Part 5: Tuning and Optimisation

### Tuning the Block Size (NB)

NB is the most impactful single tunable parameter. To find the optimal value for your hardware, test multiple NB values in one run by editing `hpccinf.txt`:

```
1            # of problems sizes (N)
35840        Ns
3            # of NBs
128 192 224  NBs
```

HPCC runs HPL once for each NB value and reports separate results for each. Compare them and use the NB that gives the highest Gflops.

---

### Reducing N for Quick Tests

For iterative tuning, use a small N to get results in under 1 minute:

```
1            # of problems sizes (N)
6000         Ns
1            # of NBs
128          NBs
```

This is not a production score, but it lets you rapidly compare the effect of changing NB, P×Q, or compiler flags without waiting 10+ minutes per run.

---

### OpenMP + MPI Hybrid Mode (Advanced)

For a cluster with more cores per node, you can switch to a **hybrid MPI+OpenMP** model: launch fewer MPI processes (one per node) and use OpenMP threads to fill the cores within each node:

Update `hpccinf.txt` to use P×Q=1×3 (or 3×1):
```
3            Ps
1            Qs
```

Then launch:
```bash
export OPENBLAS_NUM_THREADS=3
export OMP_NUM_THREADS=3

mpirun --hostfile hostfile -np 3 \
  --mca btl_tcp_if_include 10.100.0.0/24 \
  --mca oob_tcp_if_include 10.100.0.0/24 \
  -x OPENBLAS_NUM_THREADS=3 \
  -x OMP_NUM_THREADS=3 \
  ./hpcc
```

The tradeoff is less MPI communication overhead but more NUMA contention. On small VM clusters this often performs slightly worse than pure MPI, but on real hardware with fast interconnects (InfiniBand) hybrid mode can be significantly faster.

---

## Part 6: Troubleshooting

### Common Errors and Fixes

| Symptom | Root Cause | Fix |
|---|---|---|
| `hpcc: command not found` | Running from wrong directory | `cd ~/hpcc-1.5.0` first |
| `No such file: hpccinf.txt` | Wrong working directory | Run from the directory where `hpccinf.txt` lives |
| `undefined reference to 'main'` | Linker ran before objects compiled | `rm -rf lib/custom hpcc && make arch=custom` |
| `libopenblas.so.0: cannot open shared object` | `$LD_LIBRARY_PATH` not set | `export LD_LIBRARY_PATH=$HOME/opt/openblas/lib:$LD_LIBRARY_PATH` |
| `mpirun: command not found` | `$PATH` not set to custom OpenMPI | `export PATH=$HOME/opt/openmpi/bin:$PATH` |
| `tcp_peer_send_blocking: send() failed` | MPI using wrong network interface | Add `--mca btl_tcp_if_include 10.100.0.0/24` |
| HPL `FAILED` residual check | Wrong P×Q vs `-np` value | Ensure `-np` exactly equals P×Q in `hpccinf.txt` |
| All processes on headnode only | Wrong slot counts in hostfile | Verify `hostfile` has all 3 nodes listed with `slots=3` |
| GFLOPS much lower than DGEMM alone | OpenBLAS spawning extra threads | `export OPENBLAS_NUM_THREADS=1` before `mpirun` |
| Out-of-memory crash | N is too large for available RAM | Reduce N using the formula in Step 6 |
| `Permission denied (publickey)` | SSH key not copied to compute node | `ssh-copy-id ubuntu@compute-02` |
| Remote node shows system `mpirun` | Exports below interactive guard | Move `$PATH` exports to top of `~/.bashrc`, before `case $-` |

---

## Part 7: Quick Reference

### Full Run Sequence (Copy-Paste)

```bash
# 1. Navigate to HPCC directory
cd ~/hpcc-1.5.0

# 2. Set environment variables
export OPENBLAS_NUM_THREADS=1
export OMP_NUM_THREADS=1

# 3. Run HPCC across all 3 nodes (9 MPI processes total)
mpirun --hostfile hostfile -np 9 \
  --mca btl_tcp_if_include 10.100.0.0/24 \
  --mca oob_tcp_if_include 10.100.0.0/24 \
  -x OPENBLAS_NUM_THREADS=1 \
  -x OMP_NUM_THREADS=1 \
  ./hpcc

# 4. View formatted results (web summary)
perl hpccoutf.pl -w -f hpccoutf.txt

# 5. View all parameters
perl hpccoutf.pl -a -f hpccoutf.txt
```

### Parameter Quick Reference for 3-Node, 3-Core/Node Cluster

```
Nodes:       3 (headnode + compute-01 + compute-02)
Cores/node:  3
Total ranks: 9        (= -np 9 in mpirun)
P × Q:       3 × 3    (must equal -np)
N:           35840    (80% of 12 GB total RAM)
NB:          224      (start here, tune between 128–256)
```

### Pre-Run Verification Checklist

```bash
# 1. Binary exists
ls -lh ~/hpcc-1.5.0/hpcc

# 2. Input file exists in run directory
ls ~/hpcc-1.5.0/hpccinf.txt

# 3. Hostfile covers all nodes
cat ~/hpcc-1.5.0/hostfile

# 4. Custom MPI is active
which mpirun           # Must show ~/opt/openmpi/bin/mpirun

# 5. SSH works to all compute nodes
ssh compute-01 "echo OK"
ssh compute-02 "echo OK"

# 6. Environment variables reach compute nodes
ssh compute-01 "which mpirun && echo OK"
ssh compute-02 "which mpirun && echo OK"
```

---

## Summary

| Stage | What You Did |
|---|---|
| **Downloaded HPCC** | Fetched HPCC 1.5.0 source tarball from ICL |
| **Created Make.custom** | Wrote architecture config pointing to your source-compiled OpenMPI and OpenBLAS |
| **Built the binary** | Compiled all 7 benchmark components into a single `hpcc` executable with `-march=native` |
| **Configured hpccinf.txt** | Set N=35840, NB=224, P=3, Q=3 based on 12 GB total RAM and 9 MPI processes |
| **Ran the benchmark** | Launched a 9-process MPI run across 3 nodes with private network pinning |
| **Read the results** | Used the Perl formatter to display all 9 key metrics in a clean table |
| **Understood results** | Interpreted each metric and identified tuning opportunities |

> **Next:** Explore **HPCG (High-Performance Conjugate Gradient)**, which tests sparse iterative solver performance — a better proxy for real scientific workloads and increasingly important in competition scoring rubrics.
