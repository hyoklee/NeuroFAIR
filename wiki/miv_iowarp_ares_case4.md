# MiV-Simulator Case 4 (opsin) — IOWarp core vs native HDF5, RAM vs NVMe (ares)

End-to-end `run-network` I/O comparison for MiV case 4 on the **ares** SLURM
cluster (OpenMPI 5.0.10, CPU), mirroring the case-6 study in
[miv_iowarp_ares_case6.md](miv_iowarp_ares_case6.md). Tests the IOWarp CTE POSIX
adapter against native parallel HDF5, and a DRAM CTE tier against a node-local
NVMe tier.

## Setup

- **Model:** `Microcircuit_Small` — PYR 80, PVBC 53, OLM 44, STIM 10 (177 cells).
- **Case 4 = case 6 minus gap junctions.** Same Cell (36 MB) + Connection
  (8.9 MB) data; `4-opsin/datasets` is a symlink to `6-gapjunctions/datasets`.
- **Opsin is a no-op here:** the ChR2 light protocol is configured in
  `Input_Configuration.yaml`, but `env.opsin_config` is **commented out (TODO)**
  in `miv_simulator/env.py`, so `run-network` runs the base network without
  optogenetic stimulation.
- **IOWarp build:** `~/core.iowarp` (upstream iowarp/clio-core v2.0.0).
- Driver `~/bin/run_case4.sbatch`; configs `~/bin/chimaera_case4.yaml` (RAM tier)
  and `~/bin/chimaera_case4_nvme.yaml` (file bdev on `/mnt/nvme`).
- 1 node, 4 ranks, tstop=50, `--exclusive`, one batch, same node generation.

## Result (1-node, 2 reps each)

| Config | created | connected | sim | total | vs baseline |
|--------|--------:|----------:|----:|------:|------------:|
| baseline (native HDF5)     | 6.39 s | 214.4 s | 52.2 s | **287.3 s** | — |
| IOWarp, RAM tier (mean 2)  | 7.18 s | 250.7 s | 59.0 s | **337.4 s** | +17.5% |
| IOWarp, NVMe tier (mean 2) | 7.11 s | 250.2 s | 59.3 s | **337.4 s** | +17.5% |

- **IOWarp ~17.5% slower** than native HDF5 — same as case 6. The overhead is
  global POSIX-syscall interception + runtime worker threads competing for CPU,
  not I/O (even the pure-compute sim phase slows).
- **RAM vs NVMe tier: no difference** (337.4 s vs 337.4 s). The 8 GB tier file
  (`cte_tier1_node0`) was created and active; if data flowed through it, NVMe
  (the slower medium) would be slower than RAM — it isn't, so almost no payload
  transits the CTE tier and the medium is irrelevant.

## Conclusion

The case-4 result **confirms and generalizes the case-6 finding** across a second
use case: for a **compute-bound, ~45 MB read-once** MiV workload, the IOWarp CTE
POSIX adapter does not help (~17.5% overhead), and the storage-tier medium
(DRAM vs NVMe) makes no difference because little payload transits the tier. CTE's
measured I/O wins ([perf-clio-core.md](perf-clio-core.md)) require a
**bandwidth-bound, large, repeated-read** regime — the opposite of this workload.

Jobs 20382–20386. Full report + CSV: `~/MiV-Simulator-Cases/4-opsin/PERFORMANCE.md`,
`results/comparison_case4.csv`. Logs: `~/logs/case4_*.log`.
