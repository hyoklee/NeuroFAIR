# MiV-Simulator + IOWarp CTE Performance Benchmark

**Date**: 2026-04-27 (updated 2026-05-14 bench5 results)  
**Platform**: Polaris (NVIDIA A100, Cray MPICH 9.0.1)  
**IOWarp**: `~/core.iowarp` (Context Transfer Engine — Hermes RAM-tier buffering)  
**PBS scripts**: `~/bin/miv_iowarp_build.pbs` → `~/bin/miv_iowarp_bench.pbs`  

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

**Build** (once via dedicated PBS job; installs to `/lus/grand/projects/gpu_hack/iowarp/iowarp-install/`):
```bash
qsub ~/bin/miv_iowarp_build.pbs   # step 1: build (~1-2h, preemptable)
qsub ~/bin/miv_iowarp_bench.pbs   # step 2: benchmark (~2h, preemptable)
```

Uses **GCC** (`g++`) instead of NVHPC to build CTE/chimaera. The CTE layer is
CPU-only buffering code; NVHPC is not needed and caused `<filesystem>` link
failures (job 7100495) because the GCC-13 toolchain path could not be
auto-detected on compute nodes.

---

## Build Approach: GCC instead of NVHPC

**Root cause of previous failure (job 7100495)**: NVHPC 25.5 requires an
explicit `--gcc-toolchain=<path>` to resolve C++17 `<filesystem>`.
The auto-probe paths (`/opt/cray/pe/gcc/13.{1,2,3}.0/snos`) do not exist on
Polaris compute nodes, so the toolchain was never found and all three
`context-runtime` TUs failed with `catastrophic error: cannot open source file "filesystem"`.

**Fix**: Build with GCC (`-DCMAKE_CXX_COMPILER=g++`).  
- GCC has native `<filesystem>` since GCC 8 — no toolchain flag needed  
- GCC 11+ supports `-fcoroutines` (used by chimaera's fiber scheduler)  
- The CTE/Hermes layer is entirely CPU-side; NVHPC provides no benefit here  
- Dedicated build job `miv_iowarp_build.pbs` (3h preemptable) separates the
  ~1-2h build from the benchmark phases, avoiding the 6h total timeout

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

> Next steps (Lustre stable 2026-04-27):  
> Step 1: `qsub ~/bin/miv_iowarp_build.pbs` (log → `miv_iowarp_build_test.log`)  
> Step 2: `qsub ~/bin/miv_iowarp_bench.pbs`  (log → `miv_iowarp_bench2_test.log`)

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

### Blockers resolved for bench5

Two root causes identified and fixed in `~/polaris/pbs/miv_iowarp_bench.pbs` for bench5:

**Blocker 1 — Chimaera TCP bind failure (bench3 + bench4)**  
Chimaera defaults `server_addr` to `127.0.0.1` when no hostfile is configured.  
Port 9413 is occupied on `127.0.0.1` on Polaris compute nodes by another process.  
*Fix*: Get actual node IP at job start (`hostname -I | awk '{print $1}'`), write a
per-job hostfile (`/tmp/chi_hostfile_${PBS_JOBID}`), add `networking.hostfile` to
the CTE YAML config, and export `CHI_SERVER_ADDR=$NODE_IP` before starting Chimaera.

**Blocker 2 — n_active=0 / no input spikes (all Polaris runs)**  
`miv_simulator.network` checks:
```python
set(env.spike_input_namespaces).intersection(
    set(env.spike_input_attribute_info[pop_name].keys())
)
```
The spike file has namespace `'Input Spikes A Diag'` but scripts passed
`--spike_input_namespace='Input Spikes'`. Intersection is empty → `has_spike_train=False`
→ STIM cells get no spike vector → PYR/PVBC/OLM never fire → n_active=0.  
*Fix*: Changed to `--spike_input_namespace='Input Spikes A Diag'` in both bench and
opt scripts.

---

### Benchmark A — I/O benchmark (`scatter_read_trees`)

| File | Bench3 (7108868) | Bench4 (7110606) | Bench5 (7158592) | Notes |
|---|---|---|---|---|
| OLM_forest.h5 | 0.37 s | 0.37 s | **0.43 s** | |
| PVBC_forest.h5 | 0.76 s | 0.76 s | **1.21 s** | |
| PYR_forest_compressed.h5 | 311 s | 311 s | **456 s** | Lustre variance |
| **Total baseline** | **314 s** | **322 s** | **464 s** | Lustre only |
| IOWarp cold | — | — | — | Chimaera failed all runs |
| IOWarp warm | — | — | — | |

*Chimaera ZMQ bind fails on Polaris compute nodes even with dynamic ports and
node IP from PBS_NODEFILE. Root cause under investigation (see below).*

### Benchmark B — Optimization wall time

| Metric | Bench3–4 (pre-fix) | **Bench5 (PBS 7158592)** |
|---|---|---|
| VecStim spike input | **broken** (n_active=0) | **WORKING** (Input Spikes A Diag) |
| `connected cells` (weight setup/eval) | 225–227 s | **228 s** |
| `ran simulation` per eval (tstop=500ms) | 216 s (no spikes) | **258 s** (real spikes) |
| Per-evaluation total | ~441 s | **~486 s** |
| Evals in 1h debug job | N/A | **2 evals** |
| n_active PYR | 0/80 | **80/80** (12–13 Hz) |
| n_active PVBC | 0/53 | **53/53** (14–23 Hz) |
| n_active OLM | 0/44 | **44/44** (53–75 Hz) |

*Per-eval sim time increased from 216s → 258s (+42s) because processing real
spike events adds NEURON event queue overhead. Weight update cost unchanged.*

### Chimaera status (all bench runs)

Chimaera has failed to bind on every Polaris run (bench1–bench5). The ZMQ
`tcp://IP:PORT` bind fails even when:
- Dynamic port (10000–30000, below ephemeral range 32768–60999)
- Node IP resolved from `PBS_NODEFILE` hostname
- 5 retry attempts with different ports

Bench5 error (each attempt):
```
ERROR: Could not start TCP server on any host from hostfile
Port attempted: 28633 (also tried 22254, 29032, 28897, 20520)
Hosts checked: 10.201.1.169
```

The ZMQ ROUTER socket cannot bind to `tcp://10.201.1.169:PORT` on Polaris
compute nodes. Likely cause: the 10.201.x.x interface (management network) does
not accept TCP server binds from within PBS compute jobs (network policy).
Next step: try binding to the Slingshot HSN address or use IPC mode.

IOWarp I/O and optimization sections are skipped until Chimaera is fixed.

---

## Bench run summary

| Job | Date | Queue | Node | I/O (s) | Chimaera | VecStim | Evals | n_active |
|---|---|---|---|---|---|---|---|---|
| 7106964 | 2026-04-30 | preemptable | — | — | Build OK | — | — | — |
| 7107405 (bench2) | 2026-04-30 | preemptable | x3212c0s19b0n0 | — | Port conflict | broken | — | 0 |
| 7108868 (bench3) | 2026-05-01 | preemptable | x3212c0s13b0n0 | 314 | Port conflict | broken | 22 | 0 |
| 7110606 (bench4) | 2026-05-02 | preemptable | x3212c0s19b0n0 | 322 | Port conflict | broken | 22 | 0 |
| 7158592 (bench5) | 2026-05-14 | debug | x3104c0s31b0n0 | 464 | ZMQ bind fails | **FIXED** | 2 | **80/53/44** |

## Environment

```
Python:    3.11.7 (cray-python venv)
NEURON:    9.0.1
mpi4py:    3.1.4 (custom, libmpi_nvidia.so)
IOWarp:    ~/core.iowarp → /lus/grand/projects/gpu_hack/hyoklee/core.iowarp
Install:   /lus/grand/projects/gpu_hack/iowarp/iowarp-install/
Config:    CASE_DIR/config/cte_config.yaml (64 GB RAM tier, dynamic port)
POSIX:     LD_PRELOAD=libwrp_cte_posix.so (skipped when Chimaera fails)
Data:      /lus/grand/projects/gpu_hack/iowarp/
VecStim:   namespace: Input Spikes A Diag (fixed 2026-05-14)
```

---

## Related

- [VecStim namespace fix (n_active=0 root cause)](miv_vecstim_namespace_fix.md)
- [MiV-Simulator 7-optimization on Polaris](miv_opt_7_polaris.md)
- [neuroh5 shared-memory crash fix](neuroh5_shm_crash_fix.md)
- [Polaris build/test report](miv_polaris_build_test.md)
