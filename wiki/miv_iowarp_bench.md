# MiV-Simulator + IOWarp CTE Performance Benchmark

**Date**: 2026-04-26  
**Platform**: Polaris (NVIDIA A100, Cray MPICH 9.0.1)  
**IOWarp**: `~/core.iowarp` (Context Transfer Engine — Hermes RAM-tier buffering)  
**PBS script**: `~/bin/miv_iowarp_bench.pbs`  

---

## Motivation

Repeated optimization epochs re-read the same HDF5 files from Lustre on every
evaluation. With IOWarp CTE (Context Transfer Engine), the first read stages data
into a node-local RAM tier; subsequent reads bypass Lustre entirely.

### Identified Bottlenecks (from prior jobs)

| Phase | Baseline time | Nature |
|---|---|---|
| `scatter_read_trees PYR_forest_compressed.h5` | ~304 s | Pure Lustre I/O (4.4 GB, gzip) |
| `connected cells` per evaluation | ~224 s | NEURON netcon build + HDF5 reads |
| `ran simulation` per evaluation | ~1073 s (tstop=1250ms) | Compute (NEURON ODE steps) |
| Per-evaluation wall time (tstop=500ms) | ~12 min | I/O + compute |

The `scatter_read_trees` call is the clearest I/O bottleneck.  
The `connected cells` phase includes both HDF5 reads and NEURON setup; IOWarp
can reduce the HDF5 portion.  
The simulation itself is compute-bound and not accelerated by CTE.

---

## IOWarp CTE Integration Strategy

```
Polaris Lustre (/lus/grand/...)
        │  HDF5 parallel reads (first access)
        ▼
 IOWarp CTE RAM tier (64 GB, node-local DRAM)
        │  HDF5 reads served from RAM (subsequent accesses)
        ▼
  neuroh5 / miv_simulator
```

**Integration method**: `LD_PRELOAD=$IOWARP_INSTALL/lib/libhermes_posix.so`  
Intercepts POSIX file I/O calls transparently — no changes to neuroh5 or
miv_simulator source code required.

**CTE configuration** (`$CASE_DIR/config/cte_config.yaml`):
- Runtime: Chimaera, 8 scheduler threads
- Storage tier: `ram::cte_miv_storage`, 64 GB, score=1.0
- Network port: 9413 (ZMQ disabled for HPC; local shared-memory IPC only)

**Build** (once, saved to `/lus/grand/projects/gpu_hack/iowarp/iowarp-install/`):
```bash
cmake ~/core.iowarp -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_COMPILER=nvc -DCMAKE_CXX_COMPILER=nvc++ \
    -DWRP_CORE_ENABLE_RUNTIME=ON -DWRP_CORE_ENABLE_CTE=ON \
    -DWRP_CORE_ENABLE_MPI=ON    -DWRP_CORE_ENABLE_HDF5=ON \
    -DWRP_CORE_ENABLE_ZMQ=OFF   -DWRP_CORE_ENABLE_IO_URING=ON \
    -DMPI_C_COMPILER=<nvidia_mpich>/bin/mpicc \
    -DHDF5_ROOT=<nvidia_hdf5> \
    -DNVHPC_GCC_TOOLCHAIN=/opt/cray/pe/gcc/13.x.x/snos
```

---

## NVHPC + GCC-13 Build Fix

**Problem**: NVHPC 25.5 could not resolve `<filesystem>` in the CTE adapter headers
(`filesystem.h:44`, `filesystem_io_client.h:39`) because its default GCC toolchain
predates C++17 `std::filesystem`.

**Files changed**: `~/core.iowarp/CMakeLists.txt`

### Fix 1 — GCC-13 toolchain detection (line ~665, NVHPC coroutines block)

When `CMAKE_CXX_COMPILER_ID` is `NVHPC`, CMake now:

1. Checks `-DNVHPC_GCC_TOOLCHAIN=<path>` cache variable
2. Falls back to `$GCC_PATH` env var (set by `module load gcc/13.x`)
3. Auto-probes Polaris Cray PE paths:
   - `/opt/cray/pe/gcc/13.3.0/snos`
   - `/opt/cray/pe/gcc/13.2.0/snos`
   - `/opt/cray/pe/gcc/13.1.0/snos`
   - `/usr/lib/gcc/x86_64-linux-gnu/13`
4. Passes `--gcc-toolchain=<path>` to every compile unit via `add_compile_options`
5. Adds rpath for `<path>/lib64` so the runtime linker finds `libstdc++.so.6`

### Fix 2 — NVHPC linker flags (line ~771, Compiler Flags block)

```cmake
else()  # NVHPC
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -noswitcherror")
    set(CMAKE_EXE_LINKER_FLAGS    "${CMAKE_EXE_LINKER_FLAGS} -lstdc++ -lm")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -lstdc++ -lm")
endif()
```

- `-noswitcherror`: suppresses NVHPC aborting on unrecognised compiler switches
- `-lstdc++ -lm`: ensures `std::filesystem` symbols (e.g. `path`, `file_size`,
  `absolute`) link from GCC-13's libstdc++ rather than NVHPC's stub

### PBS script update (`~/bin/miv_iowarp_bench.pbs`)

- Probes Cray PE gcc-13 paths at job start; exports `GCC13_ROOT`
- Passes `-DCMAKE_C_COMPILER=nvc -DCMAKE_CXX_COMPILER=nvc++` explicitly
- Passes `-DNVHPC_GCC_TOOLCHAIN=${GCC13_ROOT}` to cmake when found

---

## Benchmark Design

### Benchmark A — I/O throughput (`scatter_read_trees`)

| Run | Description | Expected time |
|---|---|---|
| Baseline | Read from Lustre (1 run) | ~305 s |
| IOWarp cold | Read from Lustre + stage to RAM | ~305 s |
| IOWarp warm | Read from CTE RAM tier | ~5–15 s |

**Expected I/O speedup (warm vs. baseline)**: 20–60×  
Governed by DDR4 bandwidth (~50 GB/s) vs. Lustre effective throughput for
gzip-compressed HDF5 (~1–2 GB/s sequential).

### Benchmark B — Optimization wall time (1 iter × 1 gen, tstop=500ms)

| Run | Description | Expected time |
|---|---|---|
| Baseline | All reads from Lustre | ~25 min |
| IOWarp | HDF5 reads served from CTE RAM | ~22–24 min |

Smaller relative gain because the optimization bottleneck is NEURON compute
(`config_cell_syns`, `ran simulation`), not I/O. I/O benefit concentrates in
the `connected cells` HDF5-read portion (~5–10 s out of 224 s total).

---

## Results

### Polaris — IOWarp build status (PBS 7100495, 2026-04-26)

**FAILED — `libhermes_posix.so` not built.** NVHPC 25.5 could not compile `<filesystem>` headers (error: `cannot open source file "filesystem"` in `module_manager.cc`, `config_manager.cc`, `pool_manager.cc`). The GCC-13 toolchain fix documented above was applied during the PBS job but the job was killed by 6hr walltime before the baseline benchmark ran. See `miv_iowarp_bench_test.log`.

**Impact**: No Polaris IOWarp performance numbers available. The POSIX interceptor (`libhermes_posix.so`) must be built for the transparent `LD_PRELOAD` approach to work.

**Next step**: Apply the CMakeLists.txt fix, rebuild outside of PBS, verify `libhermes_posix.so` exists, then resubmit benchmark.

---

### Aurora — `/dev/shm` proxy benchmark (PBS 8452676, 2026-04-27)

The Aurora benchmark used `/dev/shm` as a CTE RAM-tier proxy (serial h5py on rank 0, 130 MB total). Results for the **small circuit** files:

| Condition | Cells 85.6 MB | Conn 39.3 MB | Spikes 3.1 MB | Total 128 MB | **BW** |
|---|---|---|---|---|---|
| Lustre (baseline) | 0.340 s | 1.305 s | 0.062 s | 1.707 s | **75 MB/s** |
| /dev/shm 1st read | 0.081 s | 0.024 s | 0.004 s | 0.109 s | **1178 MB/s** |
| /dev/shm 2nd read | 0.078 s | 0.025 s | 0.004 s | 0.107 s | **1197 MB/s** |
| **Speedup** | **4.2×** | **54.4×** | **17.3×** | — | **15.7×** |

Pre-staging (Lustre → /dev/shm): 0.096 s at 1354 MB/s for 130 MB.  
Connectivity file (DBS sparse): 54× gain — random-access penalty on Lustre eliminated.

See `perf-clio-core.md` for full Aurora analysis.

---

### Benchmark A — I/O benchmark (`scatter_read_trees`): **Polaris pending**

| Metric | Baseline (Lustre) | IOWarp cold | IOWarp warm | Speedup |
|---|---|---|---|---|
| OLM_forest.h5 | — | — | — | — |
| PVBC_forest.h5 | — | — | — | — |
| PYR_forest_compressed.h5 (~304 s est.) | — | — | — | — |
| **Total** | **—** | **—** | **—** | **—** |

### Benchmark B — Optimization wall time: **Polaris baseline measured**

From `miv_opt_7d_test.log` (PBS job, 1hr walltime, full circuit, tstop=1250ms, zero-spike issue present — VecStim fix not yet applied on Polaris):

| Metric | Baseline (Lustre) | IOWarp | Speedup |
|---|---|---|---|
| `connected cells` | **224 s** | — | — |
| `ran simulation` (tstop=1250ms) | **1073 s** | — | — |
| Per-evaluation total | **~21 min** | — | — |
| **Completed evals (1hr job)** | **2 evals** | — | — |

*Note: n_active=0 for all populations on Polaris (VecStim not registered — same root cause as Aurora Runs 6–7). Fix: apply the `make_cells` VecStim patch from Aurora Run 8 to the Polaris miv_simulator installation.*

---

## Environment

```
Python:    3.11.7 (cray-python venv)
NEURON:    9.0.1
mpi4py:    3.1.4 (custom, libmpi_nvidia.so)
IOWarp:    ~/core.iowarp (Chimaera runtime + CTE/Hermes)
Install:   /lus/grand/projects/gpu_hack/iowarp/iowarp-install/
Config:    CASE_DIR/config/cte_config.yaml (64 GB RAM tier)
POSIX:     LD_PRELOAD=libhermes_posix.so
Data:      /lus/grand/projects/gpu_hack/iowarp/
```

---

## Related

- [MiV-Simulator 7-optimization on Polaris](miv_opt_7_polaris.md)
- [neuroh5 shared-memory crash fix](neuroh5_shm_crash_fix.md)
- [Polaris build/test report](miv_polaris_build_test.md)
