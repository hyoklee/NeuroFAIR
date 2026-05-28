# Designing a MiV workload where IOWarp CTE beats baseline (ares)

Cases 4/6/7 all found IOWarp's CTE *slower* than native HDF5 because MiV
`Microcircuit_Small` is compute-bound with tiny, read-once I/O. This page designs
and tests a workload intended to **invert** that — and documents both the recipe
for a CTE win and why the small circuit can't reach it on ares.

## When CTE can win (the recipe)

CTE's RAM/NVMe tier only beats baseline when **I/O dominates wall time** *and* the
baseline can't serve it from the free OS page cache. Concretely, all three:

1. **I/O ≫ compute** — the read/write bytes (or op latency) must dominate, not
   synapse wiring + integration.
2. **Re-read or write** — CTE only helps on re-reads (the first read fills the tier
   from the backing store) or on writes (which never get a free page-cache pass).
3. **Defeat the page cache** — working set **> RAM** (46 GB/node here), or spread
   across nodes (no shared cache), or writes.

## Measured ares I/O hierarchy (O_DIRECT, uncached)

| Tier | Read | Write |
|---|---:|---:|
| `/mnt/common` (NFS, single server) | 590 MB/s | 215 MB/s |
| `/mnt/nvme` (node-local) | 1.5 GB/s | 1.0 GB/s |
| RAM tier | ~10 GB/s | ~10 GB/s |
| Page cache | — | only **46 GB/node** |

So ares *does* have a slow shared baseline (NFS) and a small cache — there is a
real niche, unlike a fast-local-FS machine.

## The design: write-bound dense recording

`run-network` with a **dense intracellular recording profile** (`IO_heavy`: v from
every section of every biophysical cell, dt=0.025) + a small `--checkpoint-interval`
streams large, periodic HDF5 writes to the results file. Baseline writes to NFS;
IOWarp redirects them to the node-local CTE tier (NVMe/RAM).
Artifacts: `~/bin/run_caseio.sbatch`, `~/bin/chimaera_caseio{,_nvme}.yaml`, and the
`IO_heavy` profile in `4-opsin/config/Recording.yaml`.

## Result — CTE still loses on the small circuit

1 node, 4 ranks, `IO_heavy`, checkpoint every 10 ms. Compared at equal wall time
(~1140 s; runs stopped early — trend unambiguous):

| Config | sim-time reached | checkpoints | rel. speed |
|---|---:|---:|---:|
| baseline (NFS) | **335 ms** | 33 | 1.00 |
| IOWarp RAM tier | 296 ms | 29 | 0.88 (**~11% slower**) |
| IOWarp NVMe tier | 299 ms | 29 | 0.89 (~11% slower); ≈ RAM |

**Why it still loses:** dense recording inflates *compute* (NEURON `Vector.record`
on every section every step), and the POSIX adapter slows that compute by
intercepting every syscall. Meanwhile total write volume stays small (tens of MB),
so the NFS-vs-local write-bandwidth gain (~0.004 s/MB) is negligible next to the
compute. The adapter overhead (~11%) dominates — same story as cases 4/6/7.
RAM ≈ NVMe again (little payload actually transits the tier). Jobs 20418–20420;
data in `4-opsin/results/caseio_writebound.csv`.

## What it would take to actually win on ares

The small circuit can't be pushed into the I/O-bound regime: scaling write volume
(longer tstop / denser recording) scales compute as much or more. A real CTE win
needs the I/O to genuinely dominate:

- **Full-scale circuit** (full CA1: ~26 GB cells+connections, multi-GB forests),
  which makes reads/writes large in absolute terms — **not currently staged on
  ares** (and its synapse-construction compute is also large, so the I/O fraction
  must still be checked).
- **Working set > 46 GB**, re-read across an ensemble within one allocation (CTE
  daemon persistent), so baseline re-reads from NFS (590 MB/s) while CTE serves
  from the NVMe tier (1.5 GB/s) — but the app must *stream* the data, not hold it
  (else OOM at 46 GB RAM).
- **Multi-node distributed cache** — one read into the shared CTE tier vs N nodes
  each reading from NFS; win scales with N. **Blocked on ares** by the multi-node
  chimaera deadlock (cases 6/7); viable on Aurora/Polaris (slow Lustre baseline).

See [perf-clio-core.md](perf-clio-core.md) (15.7× is an *isolated-read*
microbenchmark, not an app speedup) and the read-once results
[case 6](miv_iowarp_ares_case6.md) / [case 4](miv_iowarp_ares_case4.md) /
[case 7](miv_iowarp_ares_case7.md).
