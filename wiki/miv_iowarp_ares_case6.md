# MiV-Simulator Case 6 — IOWarp core vs native HDF5 (ares, end-to-end)

End-to-end `run-network` comparison of the IOWarp Context Transfer Engine (CTE)
POSIX adapter against native parallel HDF5, on the **ares** SLURM cluster
(OpenMPI 5.0.10, CPU). This complements the Aurora/Polaris I/O **microbenchmarks**
in [perf-clio-core.md](perf-clio-core.md) and [miv_iowarp_bench.md](miv_iowarp_bench.md):
those measured raw read bandwidth in isolation; this measures whole-application
wall time.

## Setup

- **Model:** `Microcircuit_Small` + gap junctions — PYR 80, PVBC 53, OLM 44,
  STIM 10 (177 cells), 102 PYR↔PYR gap-junction edges, ~160k synaptic edges.
- **Datasets:** Cells 36 MB, Connections 8.9 MB, GapJunctions 52 KB (total ~45 MB).
- **baseline:** neuroh5 → native parallel HDF5 (MPI-IO).
- **IOWarp:** `LD_PRELOAD=libclio_cte_posix.so` + chimaera runtime, RAM storage tier.
- Driver `~/bin/run_case6.sbatch` (`IOWARP=0|1`), config `~/bin/chimaera_case6.yaml`.
- `#SBATCH --exclusive` (one job per node → isolated, cold cache). 4 ranks/node.

## 1-node result (tstop=50, 2 reps each; reps within <1%)

| Phase | Baseline | IOWarp | Δ |
|-------|---------:|-------:|----:|
| created cells (read 36 MB) | 6.52 s | 7.23 s | +10.9% |
| connected cells (read 8.9 MB + wire) | 214.8 s | 250.0 s | +16.4% |
| ran simulation (50 ms, **pure compute**) | 52.4 s | 59.2 s | +13.0% |
| **total wall time** | **304.9 s** | **350.3 s** | **+14.9%** |

**IOWarp was ~15% slower.** The decisive signal: the pure-compute simulation
phase (no application I/O) is +13% slower — an I/O accelerator cannot slow a
compute-only phase. The `LD_PRELOAD` POSIX adapter intercepts *every* syscall
(Python imports, NEURON `.so`, mechanism/hoc/config reads) and the runtime's
worker threads compete for CPU, so all phases slow by a uniform ~11–16%.

**Why no I/O win here:** the workload is **compute-bound** — synapse setup +
integration dominate; I/O is ~45 MB, read once, no reuse for the RAM tier to
exploit. Contrast with [perf-clio-core.md](perf-clio-core.md), where a 26 GB
full-circuit, repeated-read, bandwidth-bound microbenchmark showed 15.7× on
Lustre — that regime is exactly where CTE helps and this one is not.

## Node scaling (baseline, native HDF5, 4 ranks/node)

| Nodes (ranks) | total | speedup | par. eff. |
|--------------:|------:|--------:|----------:|
| 1 (4)  | 304.9 s | 1.00× | 100% |
| 2 (8)  | **173.5 s** | **1.76×** | 88% |
| 4 (16) | 253.1 s | 1.20× | 30% |

The 177-cell network **scales to 2 nodes then regresses at 4** (too little work
per rank). Practical max-useful size = 2 nodes; a real large-scale study needs
the Large/Full microcircuit.

## Fork vs upstream: `~/core` vs `~/core.iowarp` (1-node, 2 reps each)

Re-ran the IOWarp path against two different IOWarp builds under identical
current conditions (same node generation, same batch, `--exclusive`):

- **`~/core`** — `hyoklee/core` fork (carries Polaris GPU fixes).
- **`~/core.iowarp`** — upstream `iowarp/clio-core` **v2.0.0**. Its POSIX adapter
  is not built by default (upstream defaults `ELF=OFF`); built here with the same
  recipe as the fork (ELF=ON, MPI from `iowarp-build`, g++) → `~/core.iowarp/install`.

| Build | created | connected | sim | total | vs baseline |
|-------|--------:|----------:|----:|------:|------------:|
| baseline (native HDF5) | 6.54 s | 216.0 s | 52.3 s | **313.9 s** | — |
| `~/core` (mean of 2)   | 7.23 s | 250.4 s | 59.2 s | **358.0 s** | +14.0% |
| `~/core.iowarp` (mean of 2) | 7.26 s | 252.2 s | 58.6 s | **359.5 s** | +14.5% |

**No meaningful difference between the two IOWarp builds: 358.0 s vs 359.5 s
(+0.4%), inside run-to-run noise** (reps: core 357.99/357.98 s; core.iowarp
359.01/360.01 s). Both add the same ~14% POSIX-interception overhead over native
HDF5. This is expected — the fork's changes are Polaris GPU fixes that don't touch
the CPU POSIX-adapter path, and both share the same chimaera/CTE lineage. Jobs
20368–20372; data in `results/comparison_core_vs_iowarp.csv`.

## Storage tier: RAM vs node-local NVMe (`~/core.iowarp`, 1-node, 2 reps each)

Swapped the CTE storage tier from DRAM to a **file-backed bdev on the compute
node's local NVMe** (`/mnt/nvme`, 239 GB SSD) — `bdev_type: file`, 8 GB cap,
config `~/bin/chimaera_case6_nvme.yaml`; `run_case6.sbatch` creates the tier dir
per run (`NVME_DIR`) for a cold start. The required `chi_default_bdev` (the
runtime's shared-memory allocator) stays RAM; only the CTE tier moved to NVMe.
All runs in one batch on the same node generation.

| Tier | created | connected | sim | total | vs baseline |
|------|--------:|----------:|----:|------:|------------:|
| baseline (native HDF5) | 6.39 s | 213.7 s | 52.3 s | **287.8 s** | — |
| RAM tier (mean of 2)   | 7.18 s | 251.5 s | 59.2 s | **338.9 s** | +17.7% |
| NVMe tier (mean of 2)  | 7.18 s | 250.4 s | 58.9 s | **337.9 s** | +17.4% |

**NVMe makes no difference: 337.9 s vs 338.9 s (−0.3%, i.e. noise)**; both ~17.5%
over baseline. The tier file (`cte_tier1_node0`, 8 GB) was created and active, so
the tier was genuinely engaged. The decisive observation: **if data flowed
heavily through the tier, NVMe (the slower medium) would be measurably slower
than RAM — it isn't.** That means almost no payload actually transits the CTE
tier for this workload; the overhead is POSIX-syscall interception on the CPU,
independent of tier medium. Same root cause as the RAM-tier result: the workload
is compute-bound with ~45 MB read-once I/O, so neither a RAM nor an NVMe buffer
has anything to accelerate. Jobs 20376–20380; data in
`results/comparison_ram_vs_nvme.csv`.

(Note: baseline here is 287.8 s vs 304.9/313.9 s in earlier batches — cluster
conditions drift between batches, which is exactly why RAM and NVMe were run in
the *same* batch for a valid within-batch comparison.)

## Multi-node IOWarp — not viable

The distributed chimaera runtime **started cleanly** (one daemon/node via
`srun --overlap`; `runtime ready (2/2 nodes)`), but `run-network` then
**deadlocked at MPI startup** under the distributed CTE — 0 progress in 49 min
vs 2.9 min baseline, killed. Reproduces the multi-node chimaera instability from
earlier PBS runs (port conflict / sub-comm deadlock).

## Conclusion

- For MiV case 6, the IOWarp POSIX adapter **did not help** (1-node ~15% slower);
  multi-node **hangs** the application.
- The result is **build-independent**: the `hyoklee/core` fork and upstream
  `iowarp/clio-core` v2.0.0 (`~/core.iowarp`) perform within 0.4% of each other.
- It is also **tier-independent**: a node-local **NVMe** CTE tier performs the
  same as the RAM tier (within 0.3%) — confirming little payload transits the
  tier, so the medium is irrelevant.
- It is the wrong tool for a **compute-bound, small, read-once** workload.
- CTE's measured I/O wins ([perf-clio-core.md](perf-clio-core.md)) apply to
  **bandwidth-bound, large, repeated-read** regimes — the opposite end.
- To target neuroh5's actual bulk transfers, an **MPI-IO / HDF5-VFD** adapter is
  needed rather than blanket POSIX interception.

Full report + raw CSVs: `~/MiV-Simulator-Cases/6-gapjunctions/PERFORMANCE.md`,
`results/comparison_1node.csv`, `results/scaling.csv`. Logs: `~/logs/case6_*.log`.
