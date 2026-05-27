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

## Multi-node IOWarp — not viable

The distributed chimaera runtime **started cleanly** (one daemon/node via
`srun --overlap`; `runtime ready (2/2 nodes)`), but `run-network` then
**deadlocked at MPI startup** under the distributed CTE — 0 progress in 49 min
vs 2.9 min baseline, killed. Reproduces the multi-node chimaera instability from
earlier PBS runs (port conflict / sub-comm deadlock).

## Conclusion

- For MiV case 6, the IOWarp POSIX adapter **did not help** (1-node ~15% slower);
  multi-node **hangs** the application.
- It is the wrong tool for a **compute-bound, small, read-once** workload.
- CTE's measured I/O wins ([perf-clio-core.md](perf-clio-core.md)) apply to
  **bandwidth-bound, large, repeated-read** regimes — the opposite end.
- To target neuroh5's actual bulk transfers, an **MPI-IO / HDF5-VFD** adapter is
  needed rather than blanket POSIX interception.

Full report + raw CSVs: `~/MiV-Simulator-Cases/6-gapjunctions/PERFORMANCE.md`,
`results/comparison_1node.csv`, `results/scaling.csv`. Logs: `~/logs/case6_*.log`.
