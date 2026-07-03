# MiV-Simulator Case 6 on TACC Vista (ARM) — IOWarp CTE vs native HDF5

Full from-scratch bring-up and end-to-end `run-network` benchmark of **case 6
(gap junctions)** on the TACC **Vista** supercomputer (NVIDIA **Grace**, ARM
**aarch64**) — a new platform for this study, complementing the x86 Aurora/ares
runs ([miv_iowarp_ares_case6.md](miv_iowarp_ares_case6.md)) and the jelly bring-up
([miv_jelly_case6.md](miv_jelly_case6.md)). CPU-only (`gg` partition,
Grace-Grace, 144 cores/node); no GPUs. Allocation `IBN22011`. Driver scripts in
`vista/bin/` (`env_miv.sh`, `build_miv_stack.slurm`, `gen_datasets.slurm`,
`run_case6.slurm`, `clio_env.sh`, `build_clio_core.slurm`).

## Bring-up (everything compiled natively on aarch64)

| Component | Version | |
|-----------|---------|--|
| NEURON | 9.0.1 | aarch64 wheel; mechanisms auto-compiled |
| neuroh5 | 0.1.18 | local repo, gcc-14, parallel HDF5 |
| miv_simulator | 0.3.0 | branch `vista-arm-fix` (4 patches) |
| h5py | 3.16.0 | **parallel** (`HDF5_MPI=ON`, `phdf5/1.14.6`) |
| mpi4py | 4.1.2 | `openmpi/5.0.5` |
| clio-core | `dev` | CTE `ELF=ON` → `libclio_cte_posix.so` (aarch64) |

Toolchain `gcc/14.2.0` + `openmpi/5.0.5` + `phdf5/1.14.6`. Datasets **generated**
from SWC morphologies (1-construction pipeline, not downloaded): 187 cells
(PYR 70, PVBC 53, OLM 44, STIM 10), Connections **160,528 synaptic edges**,
**107 PYR↔PYR gap-junction edges**, staged to `$SCRATCH/miv/datasets`.

## 1-node result (4 ranks, tstop=50, mean of 3 reps each)

| Phase | Native HDF5 | IOWarp CTE | Δ |
|-------|------------:|-----------:|----:|
| created cells (read 36 MB) | 1.89 s | 2.20 s | +0.31 s |
| connected cells (read 8.9 MB + wire) | 45.17 s | 44.89 s | −0.6% |
| ran simulation (50 ms, pure compute) | 14.72 s | 14.75 s | +0.2% |

**No meaningful difference.** The I/O-inclusive *connected* phase and the
pure-compute *ran* phase are identical within run-to-run noise; the CTE POSIX
interceptor adds only ~0.3 s fixed cost in the create phase. The CTE runtime
**did start and serve** (3 pools composed, DRAM tier `cte_ram_tier1`, server at
127.0.0.1) during the IOWarp runs — this is not a "runtime failed" null result.

**Why no I/O win — two independent reasons, both already documented here:**
1. **Compute-bound, read-once.** ~45 MB read once; wiring + integration dominate.
   Same conclusion as [miv_iowarp_ares_case6.md](miv_iowarp_ares_case6.md),
   [miv_iowarp_ares_case4.md](miv_iowarp_ares_case4.md), and
   [miv_jelly_iowarp.md](miv_jelly_iowarp.md).
2. **The adapter never sees the bulk reads.** This clio-core `dev` CTE POSIX
   adapter routes only paths carrying the literal `clio::` marker (same behavior
   observed in [miv_iowarp_ares_caseread.md](miv_iowarp_ares_caseread.md) and
   [miv_iowarp_ares_case7_full.md](miv_iowarp_ares_case7_full.md)); neuroh5 reads
   via **MPI-IO parallel HDF5**, which bypasses the intercepted POSIX path
   entirely. Transparent acceleration needs the **HDF5 VOL/MPI-IO** adapter
   ([miv_iowarp_ares_case7_vol_fix.md](miv_iowarp_ares_case7_vol_fix.md)), not
   POSIX interception.

Unlike the Aurora POSIX-adapter runs (~15% *slower* from blanket interception of
every syscall), the Vista `dev` adapter's `clio::`-only routing means non-CTE
paths pass straight through, so there is **no whole-application slowdown** — the
overhead is limited to the per-`open` marker check (~0.3 s in create).

## Node scaling (native HDF5, tstop=50, 4 ranks/node)

Vista scales **cleanly 1→2→4 nodes** — contrast the Aurora 177-cell run that
regressed at 4 nodes. Dominant parallel "connected cells" phase:

| Nodes (ranks) | connected | speedup | par. eff. | total wall |
|--------------:|----------:|--------:|----------:|-----------:|
| 1 (4)  | 45.17 s | 1.00× | 100% | 69.4 s* |
| 2 (8)  | 27.36 s | 1.65× | 83%  | 47.5 s |
| 4 (16) | 16.11 s | 2.80× | 70%  | 33.1 s |

\* First 1-node run measured 99.5 s total (one-time NEURON mechanism compilation
into the run dir); warm reps ~69 s. Phase times unaffected.

## Patches

`vista-arm-fix` in MiV-Simulator (4 commits): (1) ignore benign FP underflow in
`np.seterr` (scipy-1.17 RBF matmul on aarch64); (2) restore config-based input
feature/spike-train generation (a WIP commit had gutted the API while leaving
consumers on the old signature); (3) `compile_and_load` mechanisms in
`generate-gapjunctions` before template instantiation (`insert ch_Navaxonp`
aborted otherwise); (4) alias `env.dataset_name`→`datasetName` for `io.mkout`.

clio-core `dev` built on aarch64 with **no source patches** — environment-only:
`ELF=ON` (produces the POSIX adapter; upstream defaults OFF), `elfutils`+`zlib`
added to the deps env, and `sysroot_linux-aarch64=2.34` pinned (conda gcc-15
otherwise bakes GLIBC_2.38, unloadable against Vista's glibc 2.34). ctest 185/259
with 0 failures.

## Conclusion

- Case 6 **builds and runs end-to-end on Vista/ARM** (native HDF5), scaling
  cleanly to 4 nodes (2.80× on the connected phase).
- clio-core CTE and native HDF5 perform **the same within noise** (1 node): the
  workload is compute-bound and read-once, and the POSIX adapter's `clio::`-only
  routing means neuroh5's MPI-IO reads never transit the CTE tier.
- Consistent with the whole prior body of work: for MiV, the win (if any) lives
  in a **bandwidth-bound, large, repeated-read** regime via an **HDF5-VFD/VOL**
  adapter — not the POSIX adapter on this small compute-bound circuit.

Full summary + reproduce steps: `vista/PERFORMANCE.md`. Raw timings:
`/scratch/11623/hyoklee/miv/results/case6_timing.csv`; jobs 803272/803280/803281
(baseline scaling), 803324/803737–803740 (1-node reps).
