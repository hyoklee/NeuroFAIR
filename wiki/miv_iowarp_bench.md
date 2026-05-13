# MiV-Simulator + IOWarp CTE Performance Benchmark

**Date**: 2026-04-27 (updated; Lustre /lus service restored and stable)  
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

### Benchmark A — I/O benchmark (`scatter_read_trees`): **baseline measured; IOWarp pending**

From bench3 (PBS 7108868) and bench4 (PBS 7110606) — Chimaera failed in both runs:

| File | Baseline (Lustre) | IOWarp cold | IOWarp warm | Speedup |
|---|---|---|---|---|
| OLM_forest.h5 | **0.37 s** | — (Chimaera failed) | — | — |
| PVBC_forest.h5 | **0.76 s** | — | — | — |
| PYR_forest_compressed.h5 | **311 s** | — | — | — |
| **Total** | **322 s** | **—** | **—** | **—** |

*Chimaera fix applied for bench5; IOWarp numbers pending.*

### Benchmark B — Optimization wall time: **baseline measured; IOWarp pending**

From bench3 (PBS 7108868) and bench4 (PBS 7110606), `tstop=500ms`, 2 MPI ranks:

| Metric | Baseline (Lustre) | IOWarp | Speedup |
|---|---|---|---|
| `connected cells` (one-time setup) | **225 s** | — | — |
| `ran simulation` per eval (tstop=500ms) | **~216 s** | — | — |
| Per-evaluation total | **~441 s (~7.3 min)** | — | — |
| Evals in 6h job | **22 evals** | — | — |
| n_active (all populations) | **0** (namespace bug) | — | — |

*VecStim namespace fix applied for bench5; actual spike results pending.*

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
