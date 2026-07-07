# MiV-Simulator Case 6 (gap junctions) — TACC Frontera build + run

**Date:** 2026-07-07
**Platform:** TACC Frontera (Intel Cascade Lake, 56 cores/node, el7 / glibc 2.17)
**Status:** **PASS** — case 6 built and ran end-to-end (native parallel HDF5)
**Repos:** `hyoklee/neuroh5@master`, `hyoklee/MiV-Simulator@vista-arm-fix`,
`iowarp/clio-core@dev` (container base)

## Approach

Frontera's el7 host (glibc 2.17) cannot run modern conda-forge/manylinux
binaries, so the entire stack runs inside an **Apptainer** container
(`miv-stack.sif`, Ubuntu 24.04 / glibc 2.39) built on top of `clio-deps-cpu.sif`.
Scripts: `frontera.hyoklee/bin/{miv-stack.def, build-miv-sif.slurm,
construct-case6-data.slurm, run-case6.slurm}`.

Stack: MPICH + parallel HDF5 1.14.6 (from source) + Python 3.12 venv with
numpy 1.26.4 (pinned <2), mpi4py 4.1.2, h5py 3.16.0 (parallel, mpi=True),
NEURON (wheel), neuroh5 fork, MiV-Simulator@vista-arm-fix.

## Build issues resolved

1. **Serial vs parallel HDF5** — Ubuntu multiarch apt HDF5 made CMake/h5py pick
   the serial variant (neuroh5 failed: `H5Pset_dxpl_mpio not declared`). Fix:
   build parallel HDF5 from source into `/opt/hdf5`.
2. **NumPy 2 API removals** — MiV uses `np.row_stack`/`np.float_`; pin `numpy<2`
   as the last pip step so deps don't upgrade it.
3. **`%files` 0700 perms** — copied repos were unreadable by the non-root runtime
   user; `chmod -R a+rX /opt` in `%post`.
4. **NEURON mechanism double-load** — a manual `nrnivmodl` + MiV's own
   `compile_and_load` collided (`user defined name already exists: Gfluct3`); let
   MiV own mechanism compilation (vista-arm-fix compiles+loads them).
5. **Missing Arc Distances** — `generate-distance-connections` needs the
   `measure-distances` output (`KeyError: 'PYR'` in `filter_by_distance`); it must
   run after `generate-soma-coordinates`.

## Dataset (regenerated from `.swc`, on `/scratch2`)

The cells file is not in the repo, so it is rebuilt via the case-1 construction
pipeline. Sizes match the ares reference exactly:
`MiV_Cells_..._20220410.h5` = 36.5 MB, `MiV_Connections_..._20220410.h5` = 8.9 MB.

## Case-6 run (1 node, 8 MPI ranks, native parallel HDF5, tstop=50)

| Phase | Time |
|---|---:|
| generate-gapjunctions | 22 s |
| **run-network total** | **65 s** |
| — created cells | 0.78 s |
| — connected cells | 45.04 s |
| — created gap junctions | 0.02 s |
| — ran simulation (50 ms) | 15.41 s |

Load balance 0.98. Output `Microcircuit_Small_results.h5`.

## clio-core (IOWarp) vs native

clio-core rebuilt with the CTE POSIX adapter (`CLIO_CORE_ENABLE_ELF=ON` →
`libclio_cte_posix.so`); IOWarp arm = `clio_run` (RAM CTE tier) +
`LD_PRELOAD=libclio_cte_posix.so`. Both arms same node, 8 ranks, tstop=50:

| Phase | Native | IOWarp |
|---|---:|---:|
| created cells | 0.72 s | 0.72 s |
| connected cells | 43.25 s | 43.54 s |
| ran simulation | 15.14 s | 15.30 s |
| **run-network total** | **64 s** | **63 s** |

Byte-identical results; **IOWarp shows no benefit and no penalty (within noise)**
— consistent with the ares finding ([miv_iowarp_ares_case6.md](miv_iowarp_ares_case6.md))
that CTE doesn't help this compute-bound, ~45 MB read-once workload. Caveat: the
compute phases are unchanged too, so the POSIX adapter was likely not on the
bulk-I/O hot path (LD_PRELOAD/mpirun propagation, or neuroh5 MPI-IO vs POSIX).

## Full report

`frontera.hyoklee/doc/miv-case6-frontera.md`
