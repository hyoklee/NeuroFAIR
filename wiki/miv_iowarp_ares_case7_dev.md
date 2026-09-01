# MiV case 7 re-run on the updated clio-core `dev` — POSIX adapter vs HDF5 **VOL** vs HDF5 **VFD** (ares)

Re-run of the MiV-Simulator **case 7** (`optimize-network`, dmosopt weight
optimization) against the **current** clio-core `dev` tree, and the first
end-to-end case-7 measurement of clio-core's **HDF5 VFD** (`H5FDclio`) alongside
the **HDF5 VOL** connector and the **POSIX** adapter.

Two things changed since the [May 2026 case-7 study](miv_iowarp_ares_case7.md)
and the [June VOL fix page](miv_iowarp_ares_case7_vol_fix.md): clio-core moved on
by ~3 months of `dev`, and it now ships a second HDF5 integration path (the VFD)
next to the VOL connector.

## Setup — identical workload to the previous run

Everything about the workload is unchanged from the May run so the numbers are
directly comparable: `optimize-network` on `Microcircuit_Small`, config
`optimize_network_ares_io.yaml` (`tstop=200`), 1 node, `mpirun -np 17`
(1 distwq controller + 4 workers × 4 ranks), `n-initial=1` → **13 deterministic
GLP initial evaluations**, `--surrogate-method=megp`. All ten runs dispatched all
13 evaluations and stopped at the same point (megp needs GPyTorch, which is not
installed), so each measured wall time covers exactly the same 13-evaluation
initial-sampling phase. `--exclusive`, one job per node, cold page cache.

**What is new:** the clio-core build. Source
`~/src/iowarp/clio-core/dev/conda` at **`3357ac99`** (2026-08-29), built
out-of-source to `~/core_dev_miv/install` with `CLIO_CORE_ENABLE_ELF=ON` plus
`CLIO_CTE_ENABLE_POSIX_ADAPTER`, `CLIO_CTE_ENABLE_HDF5_VOL` and
`CLIO_CTE_ENABLE_VFD` all `ON`, against the **same parallel HDF5 1.14.6**
(`~/mc3/envs/miv`) that neuroh5, h5py and MiV load — verified with
`ldd … | grep hdf5` → `libhdf5.so.310` from that env for both `.so`s. Script:
`~/bin/build_core_dev_miv.sh`. Artifacts: `libclio_cte_posix.so`,
`libclio_hdf5_vol.so` (connector **`clio`**), `libclio_vfd.so` (driver
**`clio_vfd`**), `clio_run`.

Three backends, all selected by environment variable with **no application
change** (`~/bin/run_case7_dev.sbatch`, `IOWARP=0|1|2|3`):

| mode | how it is selected | what it does |
|---|---|---|
| POSIX | `LD_PRELOAD=libclio_cte_posix.so` | ELF syscall interception; routes `clio::`-marked paths into the CTE |
| VOL | `HDF5_VOL_CONNECTOR=clio` | pass-through VOL; mirrors whole-dataset transfers into the CTE and serves reads from the cached image |
| VFD | `HDF5_DRIVER=clio_vfd` | native write-through file driver; the on-disk file stays authoritative, the CTE tier is populate-only |

### One build gotcha worth recording

The `dev` runtime's module manager auto-discovers `*_runtime.so` on
`LD_LIBRARY_PATH`. The legacy `libchimaera_*_runtime.so` modules still sitting in
`~/mc3/envs/iowarp-build/lib` (which the old `run_case7*.sbatch` put on the path)
get loaded alongside the `clio_*` ones and **segfault `clio_run` at startup**.
Nothing in the dev build needs that env — keeping it off `PATH` and
`LD_LIBRARY_PATH` fixes it.

## Correctness first — both HDF5 paths are transparent

- **h5py**, whole-file read of `Microcircuit_Small_connections.h5` (60 numeric
  datasets, 39.2 MB): md5 **identical** native / VOL / VFD
  (`~/bin/case7_dev_smoke.sh`).
- **neuroh5's own C++ API** (`read_population_names`, `read_population_ranges`,
  `read_projection_names`, `read_cell_attribute_info`): output **byte-identical**
  native / VOL / VFD (`~/bin/case7_backend_verify.sh`).
- The VFD is proven to actually *be* the driver under neuroh5, not silently
  bypassed: `CLIO_VFD_DEBUG=1` emits `[vfd] READ addr=… size=…` for **26 read
  operations** per `read_population_ranges`, at **1, 2 and 4 MPI ranks**, with
  correct results (`~/bin/vfd_mpi_probe.sh`).

So both the VOL and the VFD carry the real case-7 read path. The performance
numbers below are measuring the adapters, not a fallback.

## Result 1 — end-to-end case 7 (1 node, 13 evaluations each)

| Config | reps (s) | mean | per eval | vs baseline |
|---|---:|---:|---:|---:|
| **baseline** (native HDF5) | 1864.3 / 1855.9 | **1860.1 s** | 143.1 s | — |
| CTE **POSIX** adapter | 1890.0 / 1867.8 | **1878.9 s** | 144.5 s | **+1.0 %** |
| HDF5 **VOL** (`clio`, cache on) | 1954.3 / 1936.1 | **1945.2 s** | 149.6 s | **+4.6 %** |
| HDF5 **VOL**, `CLIO_VOL_CACHE=0` | 1844.6 | **1844.6 s** | 141.9 s | −0.8 % |
| HDF5 **VFD** (`clio_vfd`, cache on) | 1952.4 / 1924.2 | **1938.3 s** | 149.1 s | **+4.2 %** |
| HDF5 **VFD**, `cache=0` | 1859.2 | **1859.2 s** | 143.0 s | −0.1 % |

Jobs 23786–23807. Data: `results/comparison_case7_dev.csv`.

### Comparison with the previous run

| | previous (2026-05-27, clio-core v2.0.0 `~/core.iowarp`) | now (`dev` `3357ac99`) |
|---|---:|---:|
| baseline | 1840.1 s | 1860.1 s |
| POSIX adapter | 2038.9 s (RAM) / 2050.9 s (NVMe) | 1878.9 s |
| **POSIX overhead vs its own baseline** | **+10.8 % / +11.5 %** | **+1.0 %** |

**The POSIX adapter got roughly ten times cheaper.** The May study's headline —
"IOWarp ~11 % slower" — no longer holds for the current `dev` build: the same
workload, same node type, same 13 evaluations now runs within **1 %** of native.
The two baselines differ by +1.1 %, which is the run-to-run/node noise floor and
does not explain the change.

### The cache-off controls are decisive

Turning only the CTE tier off inside each connector (`CLIO_VOL_CACHE=0`,
`HDF5_DRIVER_CONFIG=cache=0`) — while still starting the clio runtime, so
runtime CPU contention is held constant — lands both at the native baseline
(−0.8 % and −0.1 %). So the **+4.6 % / +4.2 % is entirely the CTE cache path**,
not the VOL/VFD plumbing. As pure pass-throughs both HDF5 integrations are free.

The same split shows up in a per-phase proxy, the rank-0 synapse-configuration
time (33,436 synapses for gid 10, 4 samples per run): 8.57–8.59 s native,
8.59–8.62 s POSIX, **9.05–9.09 s** VOL and VFD with the cache on, and back to
8.56–8.64 s with the cache off.

## Result 2 — read path: where the CTE cache wins and where it loses

Whole-dataset reads through h5py, node-local `/tmp`, page cache warm, medians of
6–8 passes.

### (a) 256 MB `i8`, uncompressed — the CTE tier loses badly

| backend | median / read | throughput |
|---|---:|---:|
| native HDF5 | **107.0 ms** | 2.39 GB/s |
| VOL, cache on | 421.6 ms (first 808.1, staging) | 0.61 GB/s |
| VOL, `CLIO_VOL_CACHE=0` | 110.7 ms | ≈ native |
| VFD, cache on | 107.7 ms | 2.38 GB/s |
| VFD, `cache=0` | 107.2 ms | 2.39 GB/s |

**VOL cache ≈ 3.9× slower than native**; with the cache off the connector costs
~3 %. The VFD is transparent on reads by design — it never serves from the
tier — so it tracks native exactly. (`~/bin/vol_vfd_bench_dev.sh`,
`~/bin/vol_ctrl_dev.sh`.)

### (b) the real case-7 payload — the CTE cache *wins*

Four input files (`Microcircuit_Small_connections/coords/input_spikes`,
`OLM_forest_Small`), 43.3 MB of numeric data, staged to node-local `/tmp`:

| backend | median / pass | vs native |
|---|---:|---:|
| native HDF5 | **497.0 ms** | — |
| **VOL, cache on** | **301.5 ms** (first 748.5) | **1.65× faster** |
| VOL, `CLIO_VOL_CACHE=0` | 520.2 ms | +4.7 % |
| VFD, cache on | 513.0 ms | +3.2 % |
| VFD, `cache=0` | 497.9 ms | ≈ native |

All five md5s identical. (`~/bin/case7_payload_bench.sh`.)

That is the **opposite sign** to (a), and to the June measurement on the older
connector ([+31 % slower](miv_iowarp_ares_case7_vol_fix.md)). The reason is the
data, not the connector.

### (c) mechanism — the cache serves a *decompressed* image

Every real case-7 dataset is **gzip-chunked** (62 datasets in
`…_connections.h5`, 98 in `…_coords.h5`, 56 in `OLM_forest_Small.h5`; median
dataset only 0.2–0.6 KB). The CTE cache stores the decompressed linear image, so
a cache hit skips `inflate` entirely. Same 256 MB array, written twice:

| dataset | native | VOL (cache on) |
|---|---:|---:|
| uncompressed (256 MB on disk) | **125.0 ms** | 414.4 ms — VOL **3.3× slower** |
| gzip-chunked (8.6 MB on disk) | 422.3 ms | **412.2 ms** — VOL ~2 % faster |

The VOL's cached read costs ~412 ms either way: **the CTE tier delivers at a
roughly fixed ~0.62 GB/s**, paying SHM copy + IPC round-trips per chunk. Native
costs 125 ms uncompressed and 422 ms gzip-compressed, because inflate dominates.
So the rule is simply:

> The CTE cache beats native HDF5 exactly when native is slower than ~0.6 GB/s —
> compressed data, or slow/cold storage. On uncompressed page-cache-resident data
> it cannot win, and loses by 3–4×.

The real case-7 payload wins by even more than the synthetic gzip case (1.65×)
because it is hundreds of *small* gzip datasets, where native pays per-chunk
inflate plus per-dataset metadata, while the connector serves whole cached
images. (`~/bin/vol_gzip_test.sh`.)

## Why the read win does not become an application win

The read-path win is real and now measurable, and it still does not move case 7:

- The full 43.3 MB payload takes **497 ms** natively and **301 ms** through the
  VOL cache — a saving of ~0.2 s per evaluation.
- An evaluation costs **~143 s** (synapse wiring + NEURON integration).
- 13 evaluations × 0.2 s ≈ **2.6 s saved out of 1860 s = 0.14 %**, while the
  cache path costs **+4.6 %** in per-operation overhead spread across every
  dataset and metadata access.

The overhead is roughly **35× the benefit**. This is the same compute-bound
verdict as [case 4](miv_iowarp_ares_case4.md), [case 6](miv_iowarp_ares_case6.md)
and the [May case-7 study](miv_iowarp_ares_case7.md) — but for a sharper reason
than before: it is no longer "the CTE read is slower", it is "the CTE read is
faster and the application does not read enough for that to matter".

## Bottom line

1. **The POSIX adapter's cost collapsed**: +10.8 % in May → **+1.0 %** on the
   current `dev`. The prior "IOWarp is ~11 % slower on ares" result is obsolete.
2. **The HDF5 VFD runs case 7 end-to-end for the first time**, transparently
   (byte-identical h5py *and* neuroh5 reads, verified as the active driver at
   1/2/4 ranks), at **+4.2 %** with its tier on and **at native** with it off.
   Because the VFD's tier is populate-on-write only, it can never speed a read up
   today — it is a correctness/durability path, not a performance one.
3. **The VOL connector's CTE cache now genuinely accelerates the case-7 read
   path — 1.65× faster than native** on the real gzip-compressed payload, versus
   31 % *slower* in June. The mechanism is verified: the cache serves the
   decompressed image at a fixed ~0.62 GB/s, so it wins on compressed data and
   loses on uncompressed.
4. **Neither helps the case-7 application**, because the whole input set is
   0.14 % of runtime. Both HDF5 integrations are free as pass-throughs
   (cache-off controls land on the baseline); the +4.2–4.6 % is the CTE tier's
   per-operation cost, unrecovered because there is almost no I/O to recover it
   from.

For clio-core to win on MiV, the workload has to change, not the adapter: I/O
must be a large fraction of runtime (bigger circuits, slow/shared storage, or
re-reads with little compute between them) — the recipe already written up in
[case 7 feasibility](miv_iowarp_ares_case7_feasibility.md) and
[perf-clio-core](perf-clio-core.md).

## Related pages

- [Case 7 — IOWarp core vs native HDF5, RAM vs NVMe (May 2026)](miv_iowarp_ares_case7.md) — the previous run this compares against
- [Fixing the HDF5 VOL CTE adapter for case 7 (A–F)](miv_iowarp_ares_case7_vol_fix.md) — how the VOL path was made to work at all
- [Case 7 feasibility synthesis (POSIX adapter)](miv_iowarp_ares_case7_feasibility.md)
- [perf-clio-core](perf-clio-core.md) — where CTE's measured I/O wins come from

## Artifacts

- clio-core: `~/src/iowarp/clio-core/dev/conda` @ `3357ac99`; build
  `~/bin/build_core_dev_miv.sh` → `~/core_dev_miv/install`
- run harness: `~/bin/run_case7_dev.sbatch` (`IOWARP=0` native, `1` POSIX,
  `2` VOL, `3` VFD; `CLIO_VOL_CACHE` / `VFD_CONFIG` pass through)
- metrics: `~/bin/case7_extract.py`;
  data `~/MiV-Simulator-Cases/7-optimization/results/comparison_case7_dev.csv`
- logs: `~/logs/case7_d_{base,posix,vol,vfd}{,2}_*.log`,
  `~/logs/case7_d_{volnc,vfdnc}_*.log` (jobs 23786–23807)
- benchmarks: `~/bin/vol_vfd_bench_dev.sh` (256 MB synthetic),
  `~/bin/vol_ctrl_dev.sh` (cache on/off control),
  `~/bin/case7_payload_bench.sh` (real payload),
  `~/bin/vol_gzip_test.sh` (compression mechanism)
- verification: `~/bin/case7_dev_smoke.sh`, `~/bin/case7_backend_verify.sh`,
  `~/bin/vfd_mpi_probe.sh`
