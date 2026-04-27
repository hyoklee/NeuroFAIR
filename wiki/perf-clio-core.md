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

## neuroh5 scatter-read I/O benchmark (PBS 8452562, debug queue)

*Benchmark design*: 9 MPI ranks call the same neuroh5 scatter-read functions used by `init_network`, with files read from Lustre (baseline) then from `/dev/shm` (clio-core CTE RAM tier), then repeated from `/dev/shm` (cache hit).

Phases benchmarked:
1. `scatter_read_trees` — PYR + OLM + PVBC morphology
2. `scatter_read_cell_attributes` — Synapse Attributes + Generated Coordinates (OLM, PVBC, PYR)
3. `scatter_read_cell_attribute_selection` — STIM spike trains (GIDs 0–9)
4. `scatter_read_graph` — 10 connectivity projections

**Results** (PBS 8452562, pending):

| Condition | Trees (s) | Cell Attrs (s) | Spike Trains (s) | Graph (s) | **Total (s)** | **BW (MB/s)** |
|---|---|---|---|---|---|---|
| BASELINE (Lustre) | — | — | — | — | — | — |
| CLIO-CORE (/dev/shm, 1st read) | — | — | — | — | — | — |
| CLIO-CORE (/dev/shm, 2nd read) | — | — | — | — | — | — |

*Table to be filled once PBS 8452562 completes.*

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

For the **full circuit** (not run yet — 16 GB cells + 10 GB connections), clio-core would have much larger impact:

| Phase | Small circuit | Full circuit (est.) | Full circuit + CTE |
|---|---|---|---|
| `scatter_read_trees` PYR | ~0.1 s | ~100 s | ~2 s |
| `scatter_read_cell_attributes` | ~1 s | ~50 s | ~1 s |
| `scatter_read_graph` | ~3 s | ~60 s | ~3 s |
| **Total HDF5 I/O** | **~4 s** | **~210 s** | **~6 s** |
| Estimated speedup | — | — | **~35×** |

Estimates based on 87 MB → 26 GB scale-up at same Lustre bandwidth (~100 MB/s parallel read), CTE RAM at ~10 GB/s.

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

## Next steps

1. Fill results tables once PBS 8452562 and 8452563 complete
2. Build `libhdf5_hermes_vfd.so` (requires `-DCTE_ENABLE_HDF5_VFD=ON` in cmake) to enable transparent interception
3. Test VFD-based approach: `HDF5_DRIVER=hermes LD_PRELOAD=libhdf5_hermes_vfd.so mpiexec ...`
4. Extend benchmark to full CA1 circuit files for large-scale validation
