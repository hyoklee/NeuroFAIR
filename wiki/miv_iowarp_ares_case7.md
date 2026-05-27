# MiV-Simulator Case 7 (optimization) — IOWarp core vs native HDF5, RAM vs NVMe (ares)

I/O-backend comparison for MiV case 7 on **ares** (SLURM, OpenMPI 5.0.10, CPU).
Unlike the read-once `run-network` studies ([case 6](miv_iowarp_ares_case6.md),
[case 4](miv_iowarp_ares_case4.md)), case 7 runs **`optimize-network`** (dmosopt
weight optimization): each evaluation **re-reads and rebuilds the network**, a
repeated-read pattern — the regime where the IOWarp CTE tier was hypothesized to
help.

## Setup

- **Workload:** `optimize-network`, PR (Bilash) neurons, standard NEURON
  (`use_coreneuron: False`), distwq collective mode.
- `dmosopt`+`distwq` installed from GitHub (not on PyPI); datasets reused from
  case 6. Config `optimize_network_ares_io.yaml`, `tstop=200` (shortened from
  1250 to raise the I/O fraction and keep runs tractable; we measure wall time,
  not optimization quality).
- **Layout:** `mpirun -np 17` = 1 controller + 4 workers × 4 ranks (≤ 20 physical
  cores; the node is 20-core/40-thread, so -np 25 fails OpenMPI BYCORE binding).
- `n-initial=1` → **13 deterministic GLP initial evaluations** (n_initial × 13
  params), identical across all backends. Each eval ≈ 140 s (network build +
  tstop=200 sim).
- IOWarp build `~/core.iowarp`. Driver `~/bin/run_case7.sbatch`; configs
  `~/bin/chimaera_case7.yaml` / `~/bin/chimaera_case7_nvme.yaml`.

## Result (1-node, 13 evals each)

| Config | total | per eval | vs baseline |
|--------|------:|---------:|------------:|
| baseline (native HDF5)      | **1840.1 s** | 142 s | — |
| IOWarp, RAM tier (mean 2)   | **2038.9 s** | 157 s | +10.8% |
| IOWarp, NVMe tier (mean 2)  | **2050.9 s** | 158 s | +11.5% |

- **IOWarp ~11% slower** than native HDF5; **RAM vs NVMe: no difference** (+0.6%,
  noise).
- **Even in the repeated-read regime, CTE does not help.** Each evaluation
  re-reads ~45 MB but spends ~140 s on compute (synapse wiring + integration), so
  the repeated read is <1% of per-eval time. Buffering it in RAM/NVMe saves <1%
  while blanket POSIX interception across 13 builds costs ~11%.

## Conclusion

Case 7 **extends the case-4/6 finding to the optimization (repeated-read)
workload**: the determining factor is the per-evaluation *compute*, not I/O, so
neither the IOWarp adapter nor the storage-tier medium helps. CTE's measured I/O
wins ([perf-clio-core.md](perf-clio-core.md)) require reads that are large
relative to compute and re-read many times with little work between — MiV
evaluations are the opposite (small reads, heavy per-eval compute).

**Side result:** `optimize-network` runs on ares **without deadlock** (standard
NEURON + distwq collective mode) — distinct from the earlier *CoreNEURON*-mode
distwq sub-comm hangs (see `log.md`). The runs used the `megp` surrogate (needs
GPyTorch, not installed), so they stop right after the 13-eval initial-sampling
phase — identical across backends and downstream of the measured I/O, so it does
not affect the comparison; `run_case7.sbatch` now defaults to the `gpr` surrogate
for clean completion.

Jobs 20401–20405. Full report + CSV: `~/MiV-Simulator-Cases/7-optimization/PERFORMANCE.md`,
`results/comparison_case7.csv`. Logs: `~/logs/case7_*.log`.
