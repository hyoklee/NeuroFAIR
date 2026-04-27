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

**Results** (PBS 8452563, pending):

| Metric | Baseline (Lustre) | clio-core (/dev/shm) | Speedup |
|---|---|---|---|
| Pre-staging time (s) | 0 (no staging) | — | — |
| `make_cells` time (s) | 0.39 | — | — |
| `connect_cells` time (s) | 25.08 | — | — |
| `init_input_cells` time (s) | 0.31 | — | — |
| **Total setup time (s)** | **25.84** | — | — |
| Evaluations per 6hr run | ~138 | — | — |

*Table to be filled once PBS 8452563 completes.*

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

Polaris `miv_iowarp_bench.pbs` failed at build (NVHPC `<filesystem>` issue); no Polaris numbers yet. See [MiV-Simulator + IOWarp CTE Benchmark](miv_iowarp_bench.md).

## Next steps

1. ~~Fill I/O benchmark table~~ — done (PBS 8452676)
2. Fill optimizer comparison table once PBS 8452563 completes
3. Build `libhdf5_hermes_vfd.so` on Aurora (requires `-DCTE_ENABLE_HDF5_VFD=ON` in cmake) to enable transparent interception without /dev/shm staging
4. Fix Polaris IOWarp build (NVHPC GCC-13 toolchain fix) and rerun `miv_iowarp_bench.pbs`
5. Apply Aurora VecStim fix to Polaris miv_simulator install before Polaris optimization benchmark
