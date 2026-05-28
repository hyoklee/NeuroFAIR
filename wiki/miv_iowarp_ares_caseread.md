# Read-bound CA1 benchmark — IOWarp CTE vs native HDF5 (ares)

After cases 4/6/7 ([compute-bound](miv_iowarp_ares_case4.md)) and
[caseio](miv_iowarp_ares_caseio.md) (write-bound) all showed IOWarp's CTE
*slower* than native HDF5, this page tests the remaining design point: a
**read-bound** workload that defeats the OS page cache, where the CTE NVMe tier
is supposed to win on **re-reads**.

## Dataset

Full MiV CA1 microcircuit, staged from the iraikov MiV Globus collection
(`0028aea1-ffc7-44b6-aec9-dd748ac839c4`) to the ares GCP endpoint
(`2967ad7c-a333-11f0-8c5a-0affca67c55f`) — Globus task
`f770e12e-5a55-11f1-8e67-0e9d40238285`, ~73.6 GB on-wire,
~69 GB at `/home/hyoklee/dataset-staging/MiV_CA1_full/` on `/mnt/common` (NFS).

## Working set

48.5 GB of large NeuroH5 files (above the **46 GB** per-node page cache):
`MiV_Cells_Microcircuit_20220412.h5` (19 GB),
`MiV_Connections_Microcircuit_20220412.h5` (7 GB),
`PYR_forest_compressed.h5` (4.2 GB),
`PYR_connections_compressed.h5` (5.9 GB),
`PYR_forest_syns_compressed.h5` (12 GB).

## Method

`~/bin/read_bench_ca1.py` streams every dataset of each file through
`h5py.File(...).visititems()` chunk-by-chunk (1M rows/step), releasing memory
between datasets. `~/bin/run_caseread.sbatch` runs it for `--iterations 3` in
one SLURM allocation. The CTE NVMe tier (55 GB capacity, file-backed on
`/mnt/nvme`) is large enough to hold the full working set — i.e. once iter 1
populates it, iters 2/3 should serve at NVMe (1.5 GB/s) instead of NFS
(590 MB/s).

## Result — CTE is identical to native, all 3 iterations

1 node, 1 rank, exclusive allocation. Jobs 20424 (baseline) / 20425 (IOWarp NVMe).

| iter | baseline (s) | IOWarp NVMe (s) | rel. diff |
|---:|---:|---:|---:|
| 1 (cold) | 1384.99 | 1384.34 | -0.05% |
| 2 (re-read) | 1344.49 | 1343.94 | -0.04% |
| 3 (re-read) | 1291.43 | 1291.72 | +0.02% |
| total | 4020.91 | 4020.00 | -0.02% |

Reported throughput: 142–153 MB/s of **logical** bytes (197 GB read into Python
per iter; HDF5 expands compressed chunks 4× from the 49 GB on-disk working set).

## Why CTE shows no benefit

**The NVMe tier file was never populated.** After both 2× and 3× full
iterations, `du -sh /mnt/nvme/.../cte_tier1_node0` reports **0 bytes used**
(though the sparse file is 55 GB allocated). The LD_PRELOAD POSIX adapter
intercepts reads but routes them straight through to the backing store — it does
**not transparently cache** them into the storage tier. So iter 2 and iter 3
hit NFS exactly the same as iter 1.

The small 7% drop across iterations (1385 → 1291 s) is the OS page cache
holding a few hot dataset chunks, not CTE.

## Throughput is gzip-CPU-bound, not NFS-bound

The 142 MB/s headline is **decompression**, not NFS. PYR_*_compressed.h5 use
HDF5 gzip filters; libz on a single core caps at ~150 MB/s. So even if CTE were
caching, it could only shave the NFS portion of wall time — and at the observed
ratios (49 GB on-disk vs 197 GB logical) the disk fraction is small.

Re-running with uncompressed-only files (`WSET=raw`, jobs 20426/20427) gave the
same iter-1 numbers (1591s baseline vs 1590s CTE) with the same 0-byte NVMe
tier — so the result is not specific to the gzip path. Cancelled before iter 3
since iter 1 already proved the pattern.

## Bottom line

On the ares CTE build, the CTE POSIX adapter is effectively a passthrough for
reads — neither warm nor cold re-reads of a 48-GB working set move bytes into
the NVMe tier, so there is no path to a read-side win on this stack. The win
recipe in [caseio](miv_iowarp_ares_caseio.md) (working set > cache, re-read in
one allocation) is necessary but **not sufficient** without an adapter that
actually populates the tier.

A path to demonstrate it would be either (a) an adapter build with a
write-through read cache enabled, (b) explicit pre-staging
(`cp data/* /cte-namespace/`) and reading from the CTE-managed path, or (c) the
write-bound recipe from caseio with a workload whose write bytes truly dominate
its compute.

## Follow-up: explicit pre-staging into a CTE-managed path

The CTE adapter's [filesystem source](https://github.com/iowarp/iowarp-runtime)
shows reads/writes are only intercepted when a path component starts with
`clio::` (`adapter/filesystem/filesystem.h:87`, `kClioPrefix`). All other paths
fall through untouched — which is why iters 2/3 above never hit the tier.

`~/bin/staged_read_bench.py` + `~/bin/run_caseread_staged.sbatch` do the
explicit-stage path in a single Python process (the chimaera client init logs
flood every subshell, so a multi-process bash flow can't survive). Phase 1
copies each NFS source via `os.read/os.write` to `/tmp/cte_stage_$JOBID/clio::$basename`
— the marker on the trailing component routes the writes into the NVMe tier.
Phase 2 runs h5py read iterations against the same `clio::`-prefixed paths.

### Result

`MiV_Connections_Microcircuit_20220410.h5` (6.0 GB gzip; ~32 GB logical),
single file, 2 iters. Job 20435.

| phase | wall (s) | bytes | MB/s (logical) | MB/s (on-disk) |
|---|---:|---:|---:|---:|
| stage (NFS → tier) | 24.7 | 6.0 GB on-disk | — | 246 |
| iter 1 (read from tier) | 853 | 30.8 GB logical | 36.1 | 7.5 |
| iter 2 (read from tier) | 857 | 30.8 GB logical | 36.0 | 7.5 |

Compare to plain baseline (NFS, no adapter, same gzip CPU bottleneck):
~140 MB/s logical on the big-WSET runs. Staged reads are **~4× slower** than
plain NFS reads on the same file. The NVMe tier blob did grow as expected
(`du -sh` = 6.0 GB after staging), confirming bytes really landed in the tier,
but the read-back path is the bottleneck — every read also produces:

```
ERROR ... BaseRead Tag GetBlob failed for page 6079: GetBlob operation failed
```

repeatedly for the last pages (6079, 6080 — i.e. the tail of the 6.0 GB file),
suggesting either a flush gap between the final `os.close()` and the tier
indexing, or a bug in the tier's GetBlob path. h5py still completes the iter
(returns 32 GB), so the missing pages either come back as zeros or are masked
by retries, but with a large throughput cost.

### Bottom line on this build

Pre-staging makes the *adapter* do its job (writes routed to NVMe), but the
read-back is slower than NFS — so even with the marker the demo doesn't
produce a win on this stack. Reproducing the [perf-clio-core](perf-clio-core.md)
15.7× microbenchmark result on an end-to-end MiV workload remains blocked on
ares.

Artifacts: `~/bin/read_bench_ca1.py`, `~/bin/run_caseread.sbatch`,
`~/bin/staged_read_bench.py`, `~/bin/run_caseread_staged.sbatch`,
`~/bin/chimaera_caseread{,_nvme}.yaml` (also under `~/ares/bin/`).
Logs: `~/logs/caseread_cr_*_20424.log`, `20425.log`, `20426.log`, `20427.log`;
staged: `~/logs/caseread_staged_*_20434.log` (4 files, GetBlob errors),
`20435.log` (1 file).
