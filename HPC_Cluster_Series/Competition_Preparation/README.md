# Chapter 3: Competition Preparation – Environment Management, HPC Libraries, and Benchmarking

---

## 🏁 Challenge: Add `compute-02` to the Cluster (Self-Guided)

Before you begin this chapter, your cluster must have **two compute nodes** registered and functional. You have all the knowledge you need — this is your first independent engineering task.

**Your objective:**
- Provision a second virtual machine with hostname `compute-02` and assign it the static IP `10.100.0.12` on the Host-Only network.
- Join it to the cluster: register its hostname in `/etc/hosts` on all nodes, ensure it mounts the NFS shared filesystem, receives and verifies the MUNGE authentication key, and is reachable via passwordless SSH from the headnode.

> **Refer to Chapter 1 and Chapter 2.** Every step you need is documented there. Figure it out — that is what competition day looks like.

Your `/etc/hosts` on all nodes should end up looking like this when complete:

```
10.100.0.10 headnode
10.100.0.11 compute-01
10.100.0.12 compute-02
```

Do not continue until both compute nodes pass this verification from the `headnode`:

```bash
ping -c 2 compute-01
ping -c 2 compute-02
munge -n | ssh compute-01 unmunge
munge -n | ssh compute-02 unmunge
ssh compute-01 "df -h | grep home"
ssh compute-02 "df -h | grep home"
```

---

## 3.1. Managing Your Environment

### The Binary Lookup Chain

When you type a command into the terminal and press Enter, the Linux shell does not search your entire hard drive to find that command. Instead, it searches through a specific, ordered list of directories. This list is stored in an environment variable called `$PATH`. Understanding this is not optional in HPC — it is foundational.

To see which directories are currently in your search path, printed as a colon-separated list:

```bash
echo $PATH
```

You will see an output similar to:

```
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

Each of these directories is searched **in order from left to right**. The first matching binary found wins. This is critical when you have multiple versions of the same tool installed (e.g., system GCC at `/usr/bin/gcc` and a custom-compiled GCC at `$HOME/opt/gcc/bin/gcc`). Whichever directory appears first in `$PATH` determines which version gets used.

---

### Identifying Your Default Compiler

To see exactly which binary will be invoked when you type `gcc`, use the `which` command:

```bash
which gcc
```

On a freshly installed Ubuntu system with build tools, you will see:

```
/usr/bin/gcc
```

This tells you that the default GCC binary is provided by the system package manager and lives in `/usr/bin`. Now try this on your **compute node**:

```bash
ssh compute-01 which gcc
```

If you have not installed GCC on the compute node, you will receive no output or an error like `which: no gcc in (...)`. This is an important lesson: **software installed on one node is NOT automatically available on other nodes**, unless it is installed into a location that is shared over NFS (like your home directory).

---

### The NFS Shared Home Directory and `$PATH`

You configured your `/home` directory to be NFS-exported from the headnode and mounted on all compute nodes. This means that any software you compile and install into your home directory (e.g., `~/opt/openblas`) is **physically stored on the headnode's disk** but **accessible at the exact same path on every compute node** simultaneously.

This is the foundation of how HPC clusters manage user software without requiring administrators to install everything on every node manually.

However, two conditions must be satisfied for this to work correctly:

**Condition 1: Your `$PATH` on the headnode must include the directories where you install software.**

For example, if you install OpenMPI into `~/opt/openmpi/bin`, you must add that directory to your path:

```bash
export PATH=$HOME/opt/openmpi/bin:$PATH
```

**Condition 2: The same `$PATH` configuration must exist on your compute nodes.**

Because your home directory is shared, any configuration you write into `~/.profile` or `~/.bashrc` on the headnode is automatically visible on compute nodes. However, when `mpirun` spawns remote processes on compute nodes, it starts a fresh shell. You must ensure these remote shells also pick up your environment configuration.

---

### Installing Build Essentials on Compute Nodes

System-level tools like compilers and linkers must be installed on every node individually. They are not part of the shared home directory. Install the full development toolchain on **all nodes** using APT:

**On BOTH `compute-01` and `compute-02`:**

```bash
sudo apt update
sudo apt install -y build-essential gcc g++ gfortran make git wget nano bc
```

> **Why `gfortran`?** HPL and many scientific applications include Fortran source files. Even if you use `mpicc` (a C compiler wrapper) for most of the build, the linker still needs Fortran runtime libraries at link time.

---

## 3.2. Installing Lmod — Environment Module Management

### What Are Environment Modules?

When you work in an HPC environment, you frequently need to switch between different versions of the same software. You might need GCC 11 for one application and GCC 13 for another. You might need OpenMPI 4.1 for a legacy benchmark and OpenMPI 5.0 for a newer one. Manually editing `$PATH`, `$LD_LIBRARY_PATH`, and `$MANPATH` every time you switch tools is error-prone and tedious.

Environment Modules solve this problem by providing a command-line interface to dynamically load and unload software environments. When you run `module load gcc/13.2`, the module system automatically prepends all the correct paths for that GCC version to your shell environment. When you run `module unload gcc/13.2`, those paths are cleanly removed.

**Lmod** is a modern, Lua-based implementation of Environment Modules. It is the standard tool used on thousands of HPC systems worldwide, including the CHPC national cluster and most TOP500 systems.

---

### Step 1: Install Build Prerequisites (Compute Nodes)

Lmod must be compiled from source. Since your compute nodes have more CPU cores, building there is significantly faster. Install the tools required to compile Lmod:

**On `compute-01` (and `compute-02`):**

```bash
sudo apt update
sudo apt install -y git gcc make pkg-config
```

---

### Step 2: Install Lmod Runtime Dependencies (ALL Nodes)

Lmod is written in Lua. Every node that will **run** Lmod commands (not just the one that compiles it) must have Lua and its associated libraries installed. Since your home directory is NFS-shared, the Lmod binary itself will be accessible everywhere — but the underlying Lua interpreter must be present on each node natively.

**On ALL nodes (headnode, compute-01, compute-02):**

```bash
sudo apt update
sudo apt install -y tcl tcl-dev lua5.3 liblua5.3-dev lua-filesystem-dev lua-posix lua-posix-dev bc
```

| Package | Purpose |
|---|---|
| `tcl` / `tcl-dev` | TCL scripting runtime and development headers (used internally by Lmod) |
| `lua5.3` / `liblua5.3-dev` | The Lua 5.3 interpreter and development header files — Lmod's execution engine |
| `lua-filesystem-dev` | Filesystem library development files for Lua |
| `lua-posix` / `lua-posix-dev` | POSIX system call bindings and development headers for Lua |
| `bc` | Arbitrary precision calculator used in some Lmod scripts |

---

### Step 3: Clone, Configure and Compile Lmod from Source

We will clone the official Lmod repository from the Texas Advanced Computing Center (TACC) at the University of Texas and install it into your home directory. Since your home directory is NFS-mounted, this single installation will be accessible on all cluster nodes.

**On `compute-01`** (where compilation is faster):

**Clone the repository:**

```bash
git clone https://github.com/TACC/Lmod.git
```

**Navigate into the cloned directory:**

```bash
cd Lmod
```

**Run the configuration script, specifying your home directory as the installation target:**

```bash
./configure --prefix=$HOME/lmod
```

> `--prefix=$HOME/lmod` instructs the build system to install all Lmod files under `/home/ubuntu/lmod/`. The `$HOME` variable expands to your home directory path at runtime. This ensures Lmod lands in NFS-shared storage and is available on all nodes without reinstalling.

**Compile and install Lmod using all available CPU cores:**

```bash
make -j$(nproc)
make install
```

> `-j$(nproc)` tells `make` to run as many parallel compilation jobs as there are CPU cores. `$(nproc)` is a shell subcommand that returns the number of available processors. On your compute node this significantly speeds up the build.

---

### Step 4: Activating Lmod in Your Shell

To make the `module` and `ml` commands available in your current terminal session, you must source Lmod's shell initialization script:

```bash
source ~/lmod/lmod/lmod/init/profile
```

To make this permanent so that Lmod loads automatically every time you open a new shell session, append this line to your `~/.profile`:

```bash
echo 'source ~/lmod/lmod/lmod/init/profile' >> ~/.profile
```

> **NFS & Terminal Sessions Note:** Because your `/home` directory is NFS-shared, appending this command to `~/.profile` on one node updates it for **all** nodes. However, any active terminal sessions that were already open (such as your terminal on the `headnode`) will not automatically refresh. You must run the refresh command on **both** the headnode and compute nodes in any currently active terminals:
> ```bash
> source ~/.profile
> ```

---

### Lmod Usage Reference

Once activated, Lmod gives you the `module` command with the following essential subcommands:

| Command | Operation |
|---|---|
| `module avail` | Lists all software modules available on the system |
| `module list` | Lists all modules currently loaded in your environment |
| `module load <name>` | Loads a module and configures its paths into your environment |
| `module unload <name>` | Removes a loaded module and its paths from your environment |
| `module purge` | Unloads all currently loaded modules |

Lmod also provides a convenient shorthand command `ml` that covers all of the above:

| Shorthand | Equivalent |
|---|---|
| `ml` | `module list` |
| `ml avail` | `module avail` |
| `ml <name>` | `module load <name>` |
| `ml -<name>` | `module unload <name>` |
| `ml foo bar` | `module load foo bar` |
| `ml foo -bar` | `module load foo` AND `module unload bar` |

> **Note:** Some software packages (like Intel oneAPI, which we install later) automatically register their own modulefiles with Lmod when installed. Others require you to manually point Lmod to their modulefile directory using `ml use <path>`.

---

## 3.3. System Library Fundamentals

Before you can build or tune HPL, you must understand the two types of software libraries you will be working with and the key technologies HPL depends on. This section is not optional reading — these concepts will come up repeatedly throughout the competition.

---

### Static Libraries (`.a` — Archive Files)

A static library is a collection of pre-compiled object files bundled into a single archive file (identifiable by the `.a` extension, short for "archive"). When you compile a program that uses a static library, the linker **copies** the entire contents of the relevant library directly into your program's binary at compile time.

The resulting executable is entirely self-contained. It carries everything it needs inside itself and has no dependency on external library files at runtime.

**Advantages:**
- Portable — the binary can run on systems without the library installed.
- Slightly faster at startup — no dynamic loading overhead.
- Predictable — the behaviour will not change if a system administrator updates the shared library.

**Disadvantages:**
- Larger binary file size — the library code is duplicated inside every program that uses it.
- To use a newer or better-tuned version of the library, you must recompile your program from scratch.

In HPL's build system, you will see the static HPL library itself referenced as `$(LIBdir)/libhpl.a`.

---

### Dynamic Libraries (`.so` — Shared Object Files)

A dynamic library (shared object, `.so` extension) is not embedded into the program binary at compile time. Instead, the binary contains only a reference to the library. When the program is executed, the **dynamic linker** (`ld.so`) resolves these references at runtime by locating and loading the `.so` file from the system's library paths.

**Advantages:**
- Smaller binary size — the library code exists once on disk and is shared between all programs that use it.
- Updatable — you can swap in a newer version of the library without recompiling the program.

**Disadvantages:**
- Requires the correct library version to be present on any system the program runs on.
- Adds a small startup overhead for the dynamic linking process.
- Library version mismatches (the dreaded "cannot open shared object file") are a common source of errors in HPC environments.

You will encounter both types when configuring HPL. The ATLAS math library, for example, provides both `libsatlas.so` (single-threaded dynamic) and `libtatlas.so` (threaded dynamic), as well as static `.a` variants.

---

### Message Passing Interface (MPI)

MPI is a standardized communication protocol and API for parallel computing. It defines how parallel processes — whether running on the same node (using shared memory) or across different nodes (using the network) — exchange data with one another.

MPI is the backbone of nearly every HPC application, including HPL. Without MPI, a computation can only use the resources of a single process on a single machine. With MPI, the same computation can be spread across thousands of processes running on thousands of machines simultaneously, with each process solving a portion of the problem and communicating results to the others.

There are multiple software implementations of the MPI standard. The most widely used are:

| Implementation | Notes |
|---|---|
| **OpenMPI** | Community open-source implementation, widely used in academia and research |
| **MPICH** | Reference implementation, focused on correctness and standards compliance |
| **Intel MPI** | Intel's highly optimized implementation, included in the Intel oneAPI HPC Toolkit |

In this guide you will progress from **system OpenMPI** (quick to install) to **source-compiled OpenMPI** (optimized) to **Intel MPI** (maximum performance).

---

### Basic Linear Algebra Subprograms (BLAS)

BLAS is a specification (and a family of implementations) of low-level routines for performing fundamental linear algebra operations: vector-vector operations (Level 1), matrix-vector operations (Level 2), and matrix-matrix operations (Level 3).

HPL spends the overwhelming majority of its compute time performing matrix-matrix multiplications — specifically the Level 3 BLAS routine `dgemm` (Double-precision GEneral Matrix Multiply). The speed of your HPL run is therefore almost entirely determined by how fast your BLAS implementation performs `dgemm`.

Implementations you will use in this chapter:

| Library | Source | Optimization |
|---|---|---|
| **ATLAS** | System APT package | Automatically tuned at installation time, generic x86 |
| **OpenBLAS** | Compiled from source | Tuned for your specific CPU microarchitecture |
| **Intel MKL** | Bundled with Intel oneAPI | Heavily optimized for Intel CPUs using AVX-512, AMX |

---

## 3.4. Running HPL with System Libraries

In this section you will compile HPL from source using system-provided OpenMPI and ATLAS libraries. This serves as your baseline benchmark — the reference score you will try to improve upon in subsequent sections.

---

### Step 1: Install System Dependencies

Install OpenMPI and the ATLAS math library from the APT package repository:

**On ALL nodes (headnode, compute-01, compute-02):**

```bash
sudo apt update
sudo apt install -y openmpi-bin libopenmpi-dev libatlas-base-dev wget nano
```

| Package | Provides |
|---|---|
| `openmpi-bin` | `mpirun`, `mpiexec` — MPI job launchers |
| `libopenmpi-dev` | `mpicc` compiler wrapper and MPI header files |
| `libatlas-base-dev` | ATLAS math library (BLAS/LAPACK), both static and dynamic variants |

> **Critical:** Because HPL uses dynamically linked libraries, every node that will participate in an HPL run must have these packages installed. A compute node missing `libatlas-base-dev` will cause the HPL binary to fail with a `libsatlas.so: cannot open shared object file` error at runtime.

---

### Step 2: Download and Extract HPL

From your **headnode** (where the NFS-shared home directory is managed), download the HPL source archive:

```bash
cd ~
wget https://netlib.org/benchmark/hpl/hpl-2.3.tar.gz
```

Extract the archive:

```bash
tar xf hpl-2.3.tar.gz
```

Navigate into the HPL source directory:

```bash
cd hpl-2.3
```

Inspect the available example Makefile templates:

```bash
ls setup/
```

You will see a list of architecture-specific Makefile templates. We will use one as a starting point and customize it for our Ubuntu environment with ATLAS and OpenMPI.

---

### Step 3: Create the HPL Makefile

Copy the generic Linux CBLAS template as your starting configuration:

```bash
cp setup/Make.Linux_PII_CBLAS_gm Make.ubuntu_atlas
```

Open the Makefile for editing:

```bash
nano Make.ubuntu_atlas
```

Replace the **entire contents** of the file with the following. Read every line — each parameter is explained in the comments:

```makefile
# ======================================================================
# HPL Makefile — Ubuntu 22.04, System OpenMPI + ATLAS
# Architecture tag: ubuntu_atlas
# ======================================================================

# ---- Shell and Utilities --------------------------------------------
SHELL        = /bin/sh
CD           = cd
CP           = cp
LN_S         = ln -s
MKDIR        = mkdir
RM           = /bin/rm -f
TOUCH        = touch

# ---- Architecture ---------------------------------------------------
ARCH         = ubuntu_atlas

# ---- Directory Layout -----------------------------------------------
TOPdir       = $(HOME)/hpl
INCdir       = $(TOPdir)/include
BINdir       = $(TOPdir)/bin/$(ARCH)
LIBdir       = $(TOPdir)/lib/$(ARCH)
HPLlib       = $(LIBdir)/libhpl.a

# ---- MPI Configuration -----------------------------------------------
# Ubuntu stores OpenMPI libraries in /usr/lib/x86_64-linux-gnu/openmpi
MPdir        = /usr/lib/x86_64-linux-gnu/openmpi
MPinc        = -I$(MPdir)/include
MPlib        = $(MPdir)/lib/libmpi.so

# ---- BLAS Configuration (ATLAS) --------------------------------------
# Ubuntu stores ATLAS and CBLAS libraries in /usr/lib/x86_64-linux-gnu/atlas
LAdir        = /usr/lib/x86_64-linux-gnu/atlas/
LAinc        =
LAlib        = $(LAdir)/libblas.so $(LAdir)/liblapack.so

# ---- HPL Build Options -----------------------------------------------
F2CDEFS      =
HPL_INCLUDES = -I$(INCdir) -I$(INCdir)/$(ARCH) $(LAinc) $(MPinc)
HPL_LIBS     = $(HPLlib) $(LAlib) $(MPlib)
HPL_OPTS     = -DHPL_CALL_CBLAS
HPL_DEFS     = $(F2CDEFS) $(HPL_OPTS) $(HPL_INCLUDES)

# ---- Compiler Configuration ------------------------------------------
CC           = mpicc
CCNOOPT      = $(HPL_DEFS)
CCFLAGS      = $(HPL_DEFS) -O3 -funroll-loops -fomit-frame-pointer -W -Wall
LINKER       = $(CC)
LINKFLAGS    = $(CCFLAGS)

# ---- Utility Programs ------------------------------------------------
ARCHIVER     = ar
ARFLAGS      = r
RANLIB       = echo
MAKE         = make
```

Save and exit nano (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### Step 4: Compile HPL

Make sure the MPI compiler wrapper is accessible:

```bash
which mpicc
```

You should see `/usr/bin/mpicc`. Now compile HPL using your custom Makefile:

```bash
make arch=ubuntu_atlas
```

> `arch=ubuntu_atlas` tells the HPL build system to use `Make.ubuntu_atlas` as its configuration file and create all output files under the `ubuntu_atlas` architecture subdirectory.

Verify that the HPL executable was successfully produced:

```bash
ls -lh bin/ubuntu_atlas/xhpl
```

You should see an `xhpl` binary file. If the directory or file is missing, the compilation failed — re-read the error output from `make` carefully.

---

### Step 5: Understanding and Configuring `HPL.dat`

Navigate to the binary directory:

```bash
cd bin/ubuntu_atlas
```

Open the default HPL configuration file:

```bash
nano HPL.dat
```

The full default file looks like this:

```
HPLinpack benchmark input file
Innovative Computing Laboratory, University of Tennessee
HPL.out      output file name (if any)
6            device out (6=stdout,7=stderr,file)
1            # of problems sizes (N)
29           Ns
1            # of NBs
232          NBs
0            PMAP process mapping (0=Row-,1=Column-major)
1            # of process grids (P x Q)
1            Ps
1            Qs
16.0         threshold
3            # of panel fact
0 1 2        PFACTs (0=left, 1=Crout, 2=Right)
2            # of recursive stopping criterium
2 4          NBMINs (>= 1)
1            # of panels in recursion
2            NDIVs
3            # of recursive panel fact.
0 1 2        RFACTs (0=left, 1=Crout, 2=Right)
1            # of broadcast
0            BCASTs (0=1rg,1=1rM,2=2rg,3=2rM,4=Lng,5=LnM)
1            # of lookahead depth
0            DEPTHs (>=0)
2            SWAP (0=bin-exch,1=long,2=mix)
64           swapping threshold
0            L1 in (0=transposed,1=no-transposed) form
0            U  in (0=transposed,1=no-transposed) form
1            Equilibration (0=no,1=yes)
8            memory alignment in double (> 0)
```

The parameters that matter most for performance are:

#### N — Problem Size

`N` defines the side length of the square matrix that HPL will factor. Since HPL factorizes an `N × N` matrix of 64-bit double-precision floating point numbers:

- **Memory used** scales as `N² × 8 bytes` (e.g., N=22000 uses ~3.7 GB)
- **Runtime** scales as `(2/3) × N³` floating point operations
- **GFLOPS score** increases with N up to the point where memory bandwidth becomes the bottleneck

> **Rule of thumb:** Set N such that the matrix uses approximately **80% of your available RAM**. Too small and the CPU doesn't have enough work to stay busy; too large and the OS starts swapping to disk and performance collapses.

Calculate a starting value for N from your available RAM:

```bash
free -g
```

If you have 2 GB of RAM and want to use ~80% (1.6 GB for the matrix):
```
N = sqrt(1.6 * 1024^3 / 8) ≈ sqrt(214,748,365) ≈ 14,655
```

Round down to the nearest multiple of your NB value. Set N to approximately **14000** as a starting point on a 2 GB VM.

#### NB — Block Size

`NB` defines the chunk size into which the matrix is divided for distribution and computation. The optimal NB value depends on your CPU's cache architecture. Common good starting values are `128`, `192`, or `232`. For best performance, **N must be a multiple of NB**.

#### P × Q — Process Grid

`P` and `Q` define how MPI processes are arranged in a 2D grid. `P × Q` must equal the total number of MPI processes you launch. For a single-node run with 1 process, `P=1` and `Q=1`. For a 2-node run with 2 total processes, valid options are `P=1, Q=2` or `P=2, Q=1`.

> **Recommendation:** Keep P ≤ Q, and choose values where Q/P is close to 1 (square grid) for best performance.

**Edit `HPL.dat` for a single-node baseline run on a 2 GB VM:**

```
HPLinpack benchmark input file
Innovative Computing Laboratory, University of Tennessee
HPL.out      output file name (if any)
6            device out (6=stdout,7=stderr,file)
1            # of problems sizes (N)
14000        Ns
1            # of NBs
232          NBs
0            PMAP process mapping (0=Row-,1=Column-major)
1            # of process grids (P x Q)
1            Ps
1            Qs
16.0         threshold
3            # of panel fact
0 1 2        PFACTs (0=left, 1=Crout, 2=Right)
2            # of recursive stopping criterium
2 4          NBMINs (>= 1)
1            # of panels in recursion
2            NDIVs
3            # of recursive panel fact.
0 1 2        RFACTs (0=left, 1=Crout, 2=Right)
1            # of broadcast
0            BCASTs (0=1rg,1=1rM,2=2rg,3=2rM,4=Lng,5=LnM)
1            # of lookahead depth
0            DEPTHs (>=0)
2            SWAP (0=bin-exch,1=long,2=mix)
64           swapping threshold
0            L1 in (0=transposed,1=no-transposed) form
0            U  in (0=transposed,1=no-transposed) form
1            Equilibration (0=no,1=yes)
8            memory alignment in double (> 0)
```

---

### Step 6: Run Your Baseline HPL Benchmark

From your `bin/ubuntu_atlas` directory, run the benchmark:

```bash
./xhpl
```

> **Monitor CPU utilization:** Open a second SSH session to the same node and run `htop` or `top` while HPL is executing. You should see at least one CPU core running at or near 100% utilization.

At the end of the run, look for the output line:

```
WR11C2R4       14000  232     1     1       nn.nn       x.xxxe+00
```

The final column is your result in **GFLOPS** (Giga Floating Point Operations per Second). Record this as your **Baseline Score (System ATLAS)**.

---

## 3.5. Compiling OpenBLAS and OpenMPI from Source

The system-installed ATLAS library from APT is compiled to run on any generic x86-64 processor. It cannot take advantage of the advanced instruction sets (like AVX2 or AVX-512) that your specific CPU supports. Compiling these libraries from source on your exact hardware allows the compiler to generate code that uses these wider SIMD units, which can dramatically increase GFLOPS throughput.

> **Competition note:** This is where teams separate themselves. Using source-compiled, architecture-tuned libraries is standard practice for teams that place in the top tier of the SCC.

---

### Step 1: Install Build Dependencies

**On `compute-01` (where we will compile):**

```bash
sudo apt update
sudo apt install -y build-essential gcc g++ gfortran make git wget \
    hwloc libhwloc-dev libevent-dev
```

| Package | Purpose |
|---|---|
| `gfortran` | Fortran compiler required by OpenMPI's build system |
| `hwloc` / `libhwloc-dev` | Hardware locality library — OpenMPI uses it for CPU topology awareness |
| `libevent-dev` | Event notification library required by OpenMPI |

---

### Step 2: Compile OpenBLAS from Source

**Clone the OpenBLAS repository:**

```bash
cd ~
git clone https://github.com/xianyi/OpenBLAS.git
cd OpenBLAS
```

**Check out a stable, tested release version:**

```bash
git checkout v0.3.26
```

**Compile OpenBLAS.** The build system will automatically detect your CPU and apply architecture-specific optimizations:

```bash
make -j$(nproc)
```

**Install OpenBLAS into your home directory:**

```bash
make PREFIX=$HOME/opt/openblas install
```

**Verify the installation:**

```bash
ls ~/opt/openblas/lib/
```

You should see `libopenblas.a` (static) and `libopenblas.so` (dynamic) alongside their symlinks.

---

### Step 3: Compile OpenMPI from Source

**Download the OpenMPI 4.1.4 source archive:**

```bash
cd ~
wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.4.tar.gz
```

**Extract the archive:**

```bash
tar xf openmpi-4.1.4.tar.gz
cd openmpi-4.1.4
```

**Determine your CPU microarchitecture** before configuring the build:

```bash
lscpu | grep -E "Model name|Architecture|Flags"
```

Look at the `Flags` field for entries like `avx`, `avx2`, `avx512f`. If you see `avx2`, use `-march=native` or the specific architecture name. For most modern Intel Xeon CPUs in the CHPC Sebowa environment, `cascadelake` is the appropriate target. If you are unsure, use `native` which tells the compiler to auto-detect your CPU.

**Configure OpenMPI with architecture-specific compiler flags:**

```bash
CFLAGS="-Ofast -march=native -mtune=native" ./configure --prefix=$HOME/opt/openmpi
```

> `CFLAGS="-Ofast -march=native"` — These flags tell GCC to apply the most aggressive safe optimizations (`-Ofast`) and to target your exact CPU model (`-march=native`), enabling all instruction set extensions your CPU supports.

**Compile using all available cores:**

```bash
make -j$(nproc)
```

**Install to your home directory:**

```bash
make install
```

**Verify the installation:**

```bash
ls ~/opt/openmpi/bin/
```

You should see `mpicc`, `mpirun`, `mpiexec`, and other OpenMPI tools.

---

### Step 4: Configure Your Environment for Custom Libraries

Export paths so the shell can find your custom-compiled tools:

```bash
export MPI_HOME=$HOME/opt/openmpi
export PATH=$MPI_HOME/bin:$PATH
export LD_LIBRARY_PATH=$MPI_HOME/lib:$HOME/opt/openblas/lib:$LD_LIBRARY_PATH
```

Verify that your custom `mpicc` is now the first one found:

```bash
which mpicc
```

It should now report `~/opt/openmpi/bin/mpicc` instead of `/usr/bin/mpicc`.

---

### Step 5: Create a New HPL Makefile for Custom Libraries

Navigate back to the HPL source directory:

```bash
cd ~/hpl-2.3
```

Copy your previous Makefile as the starting point for the custom build:

```bash
cp Make.ubuntu_atlas Make.custom_blas_mpi
```

Open the new Makefile:

```bash
nano Make.custom_blas_mpi
```

Update the architecture identifier and library paths to point to your custom-compiled tools:

```makefile
# ======================================================================
# HPL Makefile — Ubuntu 22.04, Custom OpenMPI + OpenBLAS (source-built)
# Architecture tag: custom_blas_mpi
# ======================================================================

# ---- Shell and Utilities --------------------------------------------
SHELL        = /bin/sh
CD           = cd
CP           = cp
LN_S         = ln -s
MKDIR        = mkdir
RM           = /bin/rm -f
TOUCH        = touch

# ---- Architecture ---------------------------------------------------
ARCH         = custom_blas_mpi

# ---- Directory Layout -----------------------------------------------
TOPdir       = $(HOME)/hpl
INCdir       = $(TOPdir)/include
BINdir       = $(TOPdir)/bin/$(ARCH)
LIBdir       = $(TOPdir)/lib/$(ARCH)
HPLlib       = $(LIBdir)/libhpl.a

# ---- Custom OpenMPI --------------------------------------------------
MPdir        = $(HOME)/opt/openmpi
MPinc        = -I$(MPdir)/include
MPlib        = $(MPdir)/lib/libmpi.so

# ---- Custom OpenBLAS (static for maximum performance) ----------------
LAdir        = $(HOME)/opt/openblas
LAinc        =
LAlib        = $(LAdir)/lib/libopenblas.a

# ---- HPL Build Options -----------------------------------------------
F2CDEFS      =
HPL_INCLUDES = -I$(INCdir) -I$(INCdir)/$(ARCH) $(LAinc) $(MPinc)
HPL_LIBS     = $(HPLlib) $(LAlib) $(MPlib)
HPL_OPTS     = -DHPL_CALL_CBLAS
HPL_DEFS     = $(F2CDEFS) $(HPL_OPTS) $(HPL_INCLUDES)

# ---- Compiler with aggressive optimization flags ---------------------
CC           = mpicc
CCNOOPT      = $(HPL_DEFS)
CCFLAGS      = $(HPL_DEFS) -O3 -march=native -mtune=native -fopenmp \
               -fomit-frame-pointer -funroll-loops -W -Wall
LDFLAGS      = -O3 -fopenmp

LINKER       = $(CC)
LINKFLAGS    = $(LDFLAGS)

# ---- Utility Programs ------------------------------------------------
ARCHIVER     = ar
ARFLAGS      = r
RANLIB       = echo
MAKE         = make
```

---

### Step 6: Compile HPL with Custom Libraries

From the HPL source root:

```bash
make arch=custom_blas_mpi
```

> If you encounter errors and need to recompile from scratch (e.g., after editing the Makefile), first clean the previous build artifacts:
> ```bash
> make clean arch=custom_blas_mpi
> ```

**Verify the new binary:**

```bash
ls -lh bin/custom_blas_mpi/xhpl
```

---

### Step 7: Run HPL with Custom Libraries

Navigate to the new binary directory:

```bash
cd ~/hpl-2.3/bin/custom_blas_mpi
```

Copy the HPL.dat file from your baseline run (adjust N for your available RAM):

```bash
cp ../ubuntu_atlas/HPL.dat .
```

Run the benchmark:

```bash
./xhpl
```

Record your new GFLOPS score as **Score 2: Custom OpenBLAS + OpenMPI**.

---

## 3.6. Intel oneAPI Toolkits — Maximum Performance

Intel's oneAPI Toolkit is a comprehensive collection of compilers, math libraries, and profiling tools developed and maintained by Intel. The key components for HPL benchmarking are:

| Component | Purpose |
|---|---|
| **Intel C/C++ Compiler (`icx`)** | Generates highly optimized machine code for Intel CPUs, often outperforming GCC for vectorizable workloads |
| **Intel Math Kernel Library (MKL)** | Intel's BLAS implementation, specifically tuned for Intel microarchitectures using AVX-512, VNNI, and AMX instruction sets |
| **Intel MPI** | Intel's MPI implementation, tuned for both shared-memory and network communication patterns on Intel hardware |

> **⚠️ Time Warning:** The Base and HPC toolkit offline installers are each several gigabytes. Allow significant download and installation time. Skip this section if you have fallen behind the recommended pace — you can return to it.

---

### Step 1: Install Optional GUI Prerequisites

These packages enable Intel's VTune Profiler graphical interface (optional but useful for performance analysis):

```bash
sudo apt install -y libdrm2 libgtk-3-0 libnotify4 xdg-utils \
    libxcb-dri3-0 libgbm1 libatspi2.0-0
```

---

### Step 2: Download the Intel oneAPI Offline Installers

Download both the Base Toolkit and the HPC Toolkit into your home directory. These are large files — run these in a persistent session (use `tmux` or `screen` to prevent disconnection from interrupting the download):

**Intel oneAPI Base Toolkit (includes MKL and `icx`):**

```bash
cd ~
wget https://registrationcenter-download.intel.com/akdlm/IRC_NAS/9a98af19-1c68-46ce-9fdd-e249240c7c42/l_BaseKit_p_2024.2.0.634_offline.sh
```

**Intel oneAPI HPC Toolkit (includes Intel MPI and `ifort`):**

```bash
wget https://registrationcenter-download.intel.com/akdlm/IRC_NAS/d4e49548-1492-45c9-b678-8268cb0f1b05/l_HPCKit_p_2024.2.0.635_offline.sh
```

---

### Step 3: Make the Installer Scripts Executable

```bash
chmod +x l_BaseKit_p_2024.2.0.634_offline.sh
chmod +x l_HPCKit_p_2024.2.0.635_offline.sh
```

---

### Step 4: Run the Base Toolkit Installer

```bash
./l_BaseKit_p_2024.2.0.634_offline.sh -a --cli --eula accept
```

| Flag | Meaning |
|---|---|
| `-a` | Pass subsequent arguments to the installer engine |
| `--cli` | Run in Command Line Interface mode (no graphical window required) |
| `--eula accept` | Automatically accept the End User License Agreement |

The installer will display CLI text prompts. Navigate through them and confirm the installation. By default, Intel oneAPI is installed into `~/intel/oneapi/`.

---

### Step 5: Run the HPC Toolkit Installer

```bash
./l_HPCKit_p_2024.2.0.635_offline.sh -a --cli --eula accept
```

Again, navigate the CLI prompts and confirm.

---

### Step 6: Configure Your Environment for Intel oneAPI

The `setvars.sh` script sets up all required environment variables for the Intel compiler suite in one step:

```bash
source ~/intel/oneapi/setvars.sh
```

You will see output confirming that components like `mpiicx`, `ifort`, `mkl`, and `mpi` have been loaded. To apply this configuration automatically every time you log in:

```bash
echo 'source ~/intel/oneapi/setvars.sh' >> ~/.profile
```

---

### Step 7: Set Up Intel Lmod Modulefiles (Optional but Recommended)

If you successfully installed Lmod in Section 3.2, Intel oneAPI can register itself as loadable modules:

```bash
cd ~/intel/oneapi/
./modulefiles-setup.sh
```

Make the newly created modulefiles available to Lmod:

```bash
ml use $HOME/modulefiles
```

Verify the modules appear:

```bash
ml avail
```

You should see Intel compiler, MKL, and MPI modules listed. You can now load Intel tools with:

```bash
ml intel/2024.2
ml mpi/2024.2
ml mkl/2024.2
```

---

### Step 8: Configure the HPL Makefile for Intel oneAPI

Navigate to the HPL source directory:

```bash
cd ~/hpl-2.3
```

HPL ships with a Linux Intel64 template. Copy it:

```bash
cp setup/Make.Linux_Intel64 ./Make.Linux_Intel64
```

Open it for editing:

```bash
nano Make.Linux_Intel64
```

Ensure the following key lines are configured correctly (update them if needed):

```makefile
# Use Intel's MPI-aware C compiler wrapper
CC       = mpiicx

# Enable OpenMP threading for multi-core utilization
OMP_DEFS = -qopenmp

# Intel optimization flags
CCFLAGS  = $(HPL_DEFS) -O3 -w -ansi-alias -z noexecstack -z relro -z now -Wall
```

The MKL and MPI paths will be resolved automatically from the environment set by `setvars.sh`.

---

### Step 9: Compile HPL with Intel oneAPI

```bash
make arch=Linux_Intel64
```

Verify the Intel-compiled binary:

```bash
ls -lh bin/Linux_Intel64/xhpl
```

---

### Step 10: Run HPL with Intel Compilers and MKL

```bash
cd bin/Linux_Intel64
cp ../ubuntu_atlas/HPL.dat .
./xhpl
```

Record your result as **Score 3: Intel MKL + Intel MPI (`icx`)**.

---

## 3.7. Theoretical Peak Performance (R_Peak)

Before comparing your benchmark results, you need to understand what your hardware is theoretically capable of. This maximum theoretical performance is called **R_Peak** and is calculated from your CPU's specifications.

### The R_Peak Formula

```
R_Peak = CPU Base Frequency (GHz) × Number of CPU Cores × FLOPS per Cycle
```

The "FLOPS per Cycle" term depends on which vector instruction set your CPU supports. Modern CPUs can perform multiple floating-point operations in a single clock cycle using SIMD (Single Instruction, Multiple Data) vector units:

| CPU Instruction Extension | FP64 Operations per Cycle (per core) |
|---|---|
| **SSE4.2** | 4 |
| **AVX / AVX2** | 8 |
| **AVX-512** | 16 |

> Note: AVX-512 is present on Intel Xeon Scalable (Skylake-SP and newer), including Cascade Lake CPUs common in HPC clusters. Consumer CPUs typically only have AVX2.

---

### Step 1: Determine Your CPU Architecture

```bash
lscpu
```

Key fields to note:

- `Model name` — The full CPU model string (e.g., `Intel Xeon Processor (Cascadelake)`)
- `CPU(s)` — Total number of logical CPUs
- `Core(s) per socket` — Physical cores per CPU package
- `Flags` — Instruction set extensions (look for `avx`, `avx2`, `avx512f`)

---

### Step 2: Determine the CPU's Base Frequency

Go to **[ark.intel.com](https://ark.intel.com)**, search for your exact CPU model name, and look up the **Processor Base Frequency** (not Turbo Boost). Use the base frequency for R_Peak calculations because HPL is a sustained workload — the CPU will run at base clock, not boost.

---

### Step 3: Calculate R_Peak

**Example calculation** for a hypothetical 2-core VM running at 2.5 GHz with AVX2 support:

```
R_Peak = 2.5 GHz × 2 cores × 8 FLOPS/cycle (AVX2 FMA) × 2 (FMA has 2 FP ops)
       = 2.5 × 2 × 16 = 80 GFLOPS/s
```

> **Note on FMA:** Fused Multiply-Add (FMA) instructions combine a multiply and an add in a single cycle, effectively doubling the FLOPS per cycle for workloads that can use it. HPL is specifically designed to exploit FMA, so multiply your vector width by 2 if your CPU supports FMA (check `Flags` for `fma`).

---

### Step 4: Calculate Efficiency

The efficiency of your HPL run is:

```
Efficiency = R_Max / R_Peak × 100%
```

Where `R_Max` is your best measured GFLOPS score. For a well-tuned Intel system, an efficiency of **75% or above** is considered adequate for competition purposes. Top competing teams often achieve 85–95% efficiency.

---

### Step 5: Benchmark Results Table

Populate this table as you complete each run. Record your `R_Max` for each configuration and calculate efficiency:

| Configuration | Cores | N | NB | R_Max (GFLOPS) | R_Peak (GFLOPS) | Efficiency |
|---|---|---|---|---|---|---|
| System ATLAS + System OpenMPI | | | | | | |
| Custom OpenBLAS + Custom OpenMPI | | | | | | |
| Intel MKL + Intel MPI (`icx`) | | | | | | |
| Multi-node (compute-01 + compute-02) | | | | | | |

---

### Step 6: Compare Against the TOP500

Visit **[top500.org](https://top500.org/lists/top500/)** and compare your `R_Max` against the entries on the current TOP500 list. The top entry as of recent lists:

| Rank | System | Cores | R_Max (GFLOPS) | R_Peak (GFLOPS) |
|---|---|---|---|---|
| 1 | Frontier (HPE Cray EX — United States) | 8,699,904 | 1,206,000,000 | 1,714,810,000 |
| Your cluster | headnode + compute-01 + compute-02 | | | |

---

## 3.8. Running HPL Across Multiple Nodes

With both compute nodes registered, NFS-mounted, and MUNGE-authenticated, you can now distribute the HPL workload across the entire cluster. This requires coordinating MPI across the private network fabric.

---

### Step 1: Verify Cluster Prerequisites

Before proceeding, confirm all the following from the `headnode`:

```bash
# Passwordless SSH to both compute nodes
ssh compute-01 "echo compute-01 OK"
ssh compute-02 "echo compute-02 OK"

# NFS home is mounted on both
ssh compute-01 "df -h | grep home"
ssh compute-02 "df -h | grep home"

# Required libraries present on all nodes
ssh compute-01 "ls ~/opt/openblas/lib/libopenblas.a"
ssh compute-02 "ls ~/opt/openblas/lib/libopenblas.a"
```

All commands must succeed without password prompts before continuing.

---

### Step 2: Create the MPI Hosts File

The MPI hosts file (also called a machinefile) tells `mpirun` which machines to use and how many processes to allocate on each.

Create the file in your HPL binary directory:

```bash
cd ~/hpl-2.3/bin/custom_blas_mpi
nano hosts
```

Add your compute nodes. The `slots` value specifies the maximum number of MPI processes `mpirun` may launch on that node:

```text
compute-01 slots=2
compute-02 slots=2
```

> Adjust the `slots` value to match the number of physical CPU cores on each compute node. Check with `ssh compute-01 nproc`.

---

### Step 3: Configure Environment for Remote MPI Shells

When `mpirun` spawns processes on remote compute nodes, it opens a fresh SSH shell. This shell must have the correct `$PATH` and `$LD_LIBRARY_PATH` configured to find your custom OpenMPI and OpenBLAS libraries. Configuration in `~/.profile` is loaded by login shells but not always by non-interactive SSH shells.

Append the following to your `~/.profile` on the headnode (which is NFS-shared to all nodes):

```bash
nano ~/.profile
```

Add these lines at the bottom:

```bash
# Custom OpenMPI and OpenBLAS environment
export MPI_HOME=$HOME/opt/openmpi
export OPENBLAS_HOME=$HOME/opt/openblas
export PATH=$MPI_HOME/bin:$PATH
export LD_LIBRARY_PATH=$MPI_HOME/lib:$OPENBLAS_HOME/lib:$LD_LIBRARY_PATH
```

Apply the configuration to your current session:

```bash
source ~/.profile
```

---

### Step 4: Configure HPL.dat for Multi-Node Execution

The `P × Q` grid must now equal the total number of MPI processes across all nodes. If you use 2 processes per node across 2 nodes, you need 4 total processes and a valid `P × Q` decomposition.

```bash
nano HPL.dat
```

Update the process grid parameters:

```
1            # of process grids (P x Q)
2            Ps
2            Qs
```

And increase N to fill the combined memory of both nodes (e.g., if each node has 2 GB, total ~3.2 GB usable):

```
28000        Ns
```

> A larger N across more nodes increases your R_Max score by giving the CPUs more work to do relative to the communication overhead between nodes.

---

### Step 5: Launch the Multi-Node HPL Run

Run HPL across both compute nodes using the hosts file with 4 total MPI processes:

```bash
mpirun -np 4 --hostfile hosts ./xhpl
```

| Flag | Meaning |
|---|---|
| `-np 4` | Launch 4 MPI processes total |
| `--hostfile hosts` | Distribute processes according to the `hosts` file |
| `./xhpl` | The HPL executable to run on each process |

Monitor CPU utilization on both nodes from separate terminal sessions:

```bash
# On compute-01
ssh compute-01 "htop"

# On compute-02
ssh compute-02 "htop"
```

You should see all CPU cores on both machines running at full utilization during the matrix factorization phase.

Record your final result as **Score 4: Multi-Node HPL (compute-01 + compute-02)** and enter it into your benchmark results table from Section 3.7.

---

## 3.9. Summary

You have successfully transformed your cluster from a bare distributed system into a competition-ready HPC benchmarking platform.

| Achievement | Detail |
|---|---|
| **compute-02 Added** | Second compute node provisioned and fully integrated into the cluster independently |
| **Lmod Installed** | Dynamic environment module management deployed across all nodes from NFS-shared home |
| **Library Theory** | Static vs dynamic linking, MPI fundamentals, and BLAS architecture understood |
| **HPL Baseline** | System ATLAS + OpenMPI benchmark score recorded as performance baseline |
| **Source-Compiled Stack** | OpenBLAS and OpenMPI compiled from source with architecture-specific optimizations |
| **Intel oneAPI** | Intel MKL and `icx` compiler suite installed and configured for maximum CPU performance |
| **RPeak Calculated** | Theoretical peak performance derived from CPU architecture and compared against RMax |
| **Multi-Node Run** | HPL distributed across both compute nodes via MPI hosts file |

> **Next:** We will deploy the **Conductor** — the Slurm Workload Manager — to formally schedule and manage these HPL jobs as production workloads across the cluster.
