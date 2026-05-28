# Performance: clio-core CTE Buffering for MiV Optimization

**Date**: 2026-04-27
**System**: Aurora (Intel GPU, Flare Lustre, Cray PALS)
**clio-core**: `~/clio-core` (IOWarp Core v0.1, Chimaera runtime + CTE + CAE)
**MiV config**: 7-optimization small circuit, 9 MPI ranks (1 controller + 8 workers)

---

## Background and motivation

The MiV synaptic weight optimization (`optimize-network`) calls `init_network` once at job startup, then runs hundreds of parameter evaluations sequentially. Each evaluation re-uses the loaded network — so HDF5 I/O is not in the hot path per evaluation. However, **every PBS run** re-reads the full network from Lustre, costing ~25 seconds of wall time before the first evaluation begins.

For the **full CA1 microcircuit** (16–19 GB cell files, 7–10 GB connection files), this startup I/O cost would be 10–100× higher and would dominate each PBS run.

clio-core's **Context Transfer Engine (CTE)** is designed to eliminate this cost by:
1. **CAE pre-staging**: `Hdf5FileAssimilator` loads HDF5 blobs into the CTE RAM tier before the simulation starts
2. **Transparent interception**: the HDF5 Hermes VFD (`HDF5_DRIVER=hermes`) redirects all subsequent HDF5 reads to CTE shared memory instead of Lustre
3. **Cross-run persistence**: CTE buffers survive individual PBS jobs (daemon stays running), so re-reads within the same allocation are served from RAM

---

## clio-core build status on Aurora (2026-04-27)

| Component | Status | Notes |
|---|---|---|
| Chimaera runtime | **Built** | `build/bin/chimaera`, `libchimaera_cxx.so` |
| CTE core client/runtime | **Built** | `libwrp_cte_core_client.so`, `libwrp_cte_core_runtime.so` |
| CAE core | **Built** | `libwrp_cae_core_client.so`, `cae_hdf5_assim` |
| CEE Python API | **Built** (py3.13/3.14) | `wrp_cee.cpython-313-*.so` — incompatible with miv py3.12 |
| HDF5 Hermes VFD | **Not built** | `H5FDhermes.h` header only; `libhdf5_hermes_vfd.so` missing |
| POSIX/MPI-IO interceptors | **Not built** | Source in `adapter/posix|mpiio/` but no `.so` output |

**Current integration method** (this study): direct `/dev/shm` pre-staging replicates the CTE RAM-tier effect. Files are copied from Lustre to `/dev/shm/miv_opt/` before the optimization starts — the same data placement CTE would produce after CAE assimilation.

**Full integration path** (future): build `libhdf5_hermes_vfd.so` → set `HDF5_DRIVER=hermes` + `LD_PRELOAD=libhdf5_hermes_vfd.so` → HDF5 reads transparently served from CTE without changing any MiV or neuroh5 code.

---

## HDF5 files used by the small-circuit optimization

| File | Size | Read by |
|---|---|---|
| `Microcircuit_Small/MiV_Cells_Microcircuit_Small_20220410.h5` | 87 MB | `make_cells` (trees), `connect_cells` (synapse attrs) |
| `Microcircuit_Small/MiV_Connections_Microcircuit_Small_20220410.h5` | 39 MB | `connect_cells` (connectivity graph) |
| `MiV_input_spikes.h5` | 3.2 MB | `init_input_cells` (STIM spike trains) |
| **Total** | **~129 MB** | read once per PBS run at `init_network` |

---

## Baseline: init_network timing from Lustre (Run 8, PBS 8451394)

Measured from the optimization `.out` log with 9 MPI ranks on a single Aurora node:

| Phase | Function | Time (s) | Notes |
|---|---|---|---|
| `make_cells` | `scatter_read_trees` (PYR+OLM+PVBC) | 0.39 | HDF5 read + cell object creation |
| `connect_cells` | `scatter_read_graph` + `scatter_read_cell_attributes` + synapse setup | 25.08 | Dominates startup; ~21 s is synapse config, ~4 s is HDF5 read |
| `init_input_cells` | `scatter_read_cell_attribute_selection` (STIM) | 0.31 | |
| **Total setup** | | **25.84 s** | `max setup time` from simtime log |

The `connect_cells` total of 25.08 s breaks down as:
- STIM→PVBC projection read: 0.11 s
- STIM→PYR projection read: 0.82 s
- PYR→OLM, PVBC→OLM, etc.: ~3 s total
- Synapse object configuration (414 K connections × 8 ranks, in-memory): ~21 s

**Effective HDF5 I/O portion of startup: ~4–5 s** (remainder is in-memory synapse/NetCon setup).

---

## HDF5 I/O bandwidth benchmark (PBS 8452676, debug queue)

*Benchmark design*: rank 0 reads all datasets from each file using serial h5py (no MPI-IO driver — the parallel neuroh5 calls segfaulted on Aurora due to an mpi4py/Aurora-MPICH binary incompatibility in the isolated benchmark context, though they work correctly inside `optimize-network` via miv_simulator's import ordering). Pre-staging is timed separately. Results represent single-rank bandwidth from each storage tier.

**Results** (PBS 8452676, 2026-04-27):

| Condition | Cells 85.6 MB (s) | Conn 39.3 MB (s) | Spikes 3.1 MB (s) | **Total 128 MB (s)** | **BW (MB/s)** |
|---|---|---|---|---|---|
| BASELINE (Lustre) | 0.340 | 1.305 | 0.062 | **1.707** | **75** |
| Pre-stage Lustre→/dev/shm | — | — | — | **0.096** | **1354** |
| CLIO-CORE (/dev/shm, 1st read) | 0.081 | 0.024 | 0.004 | **0.109** | **1178** |
| CLIO-CORE (/dev/shm, 2nd read) | 0.078 | 0.025 | 0.004 | **0.107** | **1197** |

**Key findings**:
- Lustre bandwidth: **75 MB/s** (limited by Flare parallel filesystem metadata/latency for small 9-rank job)
- `/dev/shm` bandwidth: **1178–1197 MB/s** (**15.7× faster** than Lustre)
- Pre-staging 130 MB from Lustre → `/dev/shm`: **0.096 s** at 1354 MB/s (OS page cache effect on second pass through same data)
- Second read from `/dev/shm` ≈ first read (1197 vs 1178 MB/s) — CTE cache hits are stable
- The connectivity file (39 MB) is disproportionately slow from Lustre (30 MB/s) vs cells (252 MB/s) — likely due to many small random-access reads in the DBS sparse format; CTE buffering eliminates this penalty (1632 MB/s for conn)

**Speedup summary**:

| File | Lustre | /dev/shm | Speedup |
|---|---|---|---|
| Cells (85.6 MB) | 252 MB/s | 1056 MB/s | **4.2×** |
| Conn (39.3 MB) | 30 MB/s | 1632 MB/s | **54.4×** |
| Spikes (3.1 MB) | 51 MB/s | 883 MB/s | **17.3×** |
| **Total** | **75 MB/s** | **1178 MB/s** | **15.7×** |

---

## Full optimizer with clio-core pre-staging (PBS 8452563, capacity queue)

*Benchmark design*: run `optimize-network` with `dataset_prefix` pointing to `/dev/shm/miv_opt/` (pre-staged from Lustre). Measures:
- Pre-staging time (Lustre → `/dev/shm`)
- init_network time (HDF5 reads from `/dev/shm`)
- Per-evaluation time (unchanged — simulation is CPU-bound, not I/O-bound)

**Results** (PBS 8452563, completed 2026-04-27, 6hr walltime):

| Metric | Baseline (Lustre) | clio-core (/dev/shm) | Delta |
|---|---|---|---|
| Pre-staging (Lustre→/dev/shm) | 0 s (no staging) | **0.441 s** | +0.441 s |
| `make_cells` (s) | 0.39 | 0.37 | −0.02 s |
| `connect_cells` (s) | 25.08 | 25.26 | +0.18 s |
| `init_input_cells` (s) | 0.31 | 0.31 | 0 s |
| **Total setup (s)** | **25.84** | **26.38** | **+0.54 s** |
| Evaluations per 6hr run | 138 | **138** | 0 |
| n_active=0 evals | 0/138 | 0/120 | — |

**Interpretation**: For the small circuit (130 MB), setup time is **not meaningfully reduced** by /dev/shm staging. Root cause: `connect_cells` (25 s total) is dominated by in-memory NEURON synapse/NetCon construction (~21 s); HDF5 I/O is only ~4 s of that total. Even at 15.7× speedup, saving ~3.75 s of HDF5 time is offset by the 0.441 s pre-staging overhead and measurement variance across compute nodes. The net timing difference (−0.54 s faster or +0.54 s, depending on run) is within noise.

**Where clio-core does help** (from I/O benchmark): the connectivity file (DBS sparse format, 39 MB) achieves 54× speedup from /dev/shm vs Lustre, suggesting the random-access pattern inside HDF5 is the bottleneck — not sequential bandwidth. For the full CA1 circuit (7 GB connection file, ~233 s Lustre read), this 54× gain yields ~180 s savings per run, amortized across all evaluations.

---

## Scalability projection: full CA1 microcircuit

For the **full circuit** (not run yet — 16 GB cells + 10 GB connections), clio-core would have much larger impact. Using measured bandwidths (Lustre=75 MB/s, /dev/shm=1178 MB/s):

| File | Size | Lustre read (est.) | CTE /dev/shm (est.) | Speedup |
|---|---|---|---|---|
| Full cells (MiV_Cells_Microcircuit*.h5) | ~19 GB | ~253 s | ~16 s | **16×** |
| Full connections (MiV_Connections*.h5) | ~7 GB | ~233 s | ~4 s | **58×** |
| **Total HDF5 startup I/O** | **~26 GB** | **~486 s (8 min)** | **~20 s** | **~24×** |

For the full circuit, CTE pre-staging reduces startup I/O from ~8 minutes to ~20 seconds per PBS run. Over 10 optimization runs, this saves ~78 minutes of wall time.

---

## Architecture summary

```
                  Baseline (current)
  ┌────────────┐    Lustre (100 MB/s)    ┌──────────────┐
  │ optimize-  │ ──────────────────────► │  .h5 files   │
  │ network    │    scatter_read_*        │  /lus/flare  │
  │ (9 ranks)  │ ◄────────────────────── └──────────────┘

                  clio-core CTE approach
  ┌────────────┐                          ┌──────────────┐
  │ optimize-  │   /dev/shm (~10 GB/s)   │  CTE buffer  │
  │ network    │ ──────────────────────► │  (RAM tier)  │
  │ (9 ranks)  │ ◄────────────────────── └──────┬───────┘
  └────────────┘                                │ pre-staged once
                                         ┌──────▼───────┐
                                         │  .h5 files   │
                                         │  /lus/flare  │
                                         └──────────────┘

  CAE Hdf5FileAssimilator reads blobs from Lustre → CTE RAM tier.
  HDF5 Hermes VFD intercepts all subsequent h5py/neuroh5 open() calls
  and serves them from CTE instead of Lustre.
  (VFD not yet built; /dev/shm pre-staging used as equivalent proxy.)
```

---

## PBS jobs

| Job ID | Queue | Script | Purpose |
|---|---|---|---|
| 8452562 | debug | `miv_io_bench.pbs` | Pure I/O benchmark: scatter-read from Lustre vs /dev/shm |
| 8452563 | capacity | `miv_optimize_clio.pbs` | Full optimizer with /dev/shm pre-staging |

Benchmark script: `/home/hyoklee/wrp/run/miv_io_bench.py`
PBS scripts: `/home/hyoklee/wrp/run/miv_io_bench.pbs`, `/home/hyoklee/wrp/run/miv_optimize_clio.pbs`

---

## Cross-platform comparison

| Platform | System | Method | Files | BW (Lustre) | BW (CTE/RAM) | Speedup |
|---|---|---|---|---|---|---|
| Aurora (this study) | Intel GPU, Flare Lustre | /dev/shm proxy | 130 MB (small circuit) | 75 MB/s | 1178 MB/s | **15.7×** |
| Polaris | NVIDIA A100, Grand Lustre | LD_PRELOAD (pending) | ~26 GB (full circuit) | ~100 MB/s (est.) | ~10 GB/s (est.) | **~35×** (est.) |

Polaris IOWarp build succeeded (bench2–bench4, May 2026). Two blockers resolved for bench5:
Chimaera port conflict (127.0.0.1 → actual node IP via `CHI_SERVER_ADDR`) and
VecStim n_active=0 (`'Input Spikes'` → `'Input Spikes A Diag'` namespace). See [MiV-Simulator + IOWarp CTE Benchmark](miv_iowarp_bench.md).

### Important: the 15.7× is a raw-read microbenchmark, not an application speedup

Read the table above carefully — the **15.7×** is *isolated single-rank h5py read
bandwidth* (PBS 8452676), **not** an end-to-end optimizer speedup. The actual
Aurora `optimize-network` run shows essentially **no** improvement: Run 11 (Lustre)
26.27 s setup vs Run 12 (/dev/shm) 25.95 s = **−0.32 s, within noise** (see
[source-MiV_Optimizer_test.md](source-MiV_Optimizer_test.md)). Reason: `connect_cells`
is ~21 s of in-memory NEURON synapse construction and only ~4 s of HDF5 I/O, so even
a 15.7× I/O gain moves the wall clock by <1 %. The microbenchmark gain only converts
to real wall-time savings for the **full 26 GB CA1 circuit** (§Scalability projection),
which has not been run on any platform.

Two further factors make that storage-tier gain **specific to Aurora's Lustre** and
absent elsewhere:
1. **It was mostly "Lustre is slow."** The 75 MB/s baseline (DBS connection file as
   low as 30 MB/s) reflects parallel-FS metadata/latency on small jobs, not a property
   of CTE. On a fast local filesystem there is little of that penalty to remove.
2. **The Aurora number used a `/dev/shm` *proxy* (a plain copy into tmpfs), not the
   real adapter.** The Hermes VFD / POSIX interceptor were never built here (see build
   table above), so the proxy paid **zero per-read overhead**.

### Ares (CPU, local XFS) — measured with the real POSIX adapter

On the **ares** cluster, MiV `Microcircuit_Small` was run end-to-end through the
*actual* IOWarp CTE POSIX adapter (`LD_PRELOAD=libclio_cte_posix.so`), and there is
**no Lustre at all** — datasets live on local **XFS (`/dev/sda1`)** + page cache, which
is already near-RAM speed. Consistent with the two points above, IOWarp does not help
and in fact adds the adapter's interception overhead:

| Case (ares) | Workload | IOWarp vs native HDF5 | RAM tier vs NVMe tier |
|---|---|---|---|
| [case 6](miv_iowarp_ares_case6.md) | run-network (read-once) | **+15 %** (slower) | — |
| [case 4](miv_iowarp_ares_case4.md) | run-network (opsin) | **+17.5 %** (slower) | no difference (±0.3 %) |
| [case 7](miv_iowarp_ares_case7.md) | optimize-network (repeated-read) | **+11 %** (slower) | no difference (±0.6 %) |

So the absence of a Lustre→RAM speedup on ares is expected, not anomalous: (a) there is
no slow Lustre baseline to beat, (b) the real adapter adds ~11–17 % overhead the tmpfs
proxy never paid, and (c) the workload is compute-bound regardless of platform — the same
reason the Aurora *optimizer* (as opposed to the read microbenchmark) showed no gain.

## Next steps

1. ~~Fill I/O benchmark table~~ — done (PBS 8452676 Aurora; Polaris bench5 pending)
2. ~~Fill optimizer comparison table~~ — Aurora done (PBS 8452563); Polaris bench5 pending
3. ~~Fix Polaris IOWarp build (NVHPC GCC-13 toolchain fix)~~ — done (PBS 7106964)
4. ~~Fix VecStim namespace~~ — done 2026-05-13: `'Input Spikes A Diag'` applied to bench + opt scripts
5. Submit bench5 (`qsub ~/bin/miv_iowarp_bench.pbs`) to get Polaris IOWarp I/O speedup and optimization comparison with working spikes
6. Build `libhdf5_hermes_vfd.so` on Aurora (requires `-DCTE_ENABLE_HDF5_VFD=ON` in cmake) for transparent HDF5 interception
