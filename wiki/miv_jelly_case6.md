# MiV-Simulator Case 6 (gap junctions) — native parallel HDF5 on `jelly`

End-to-end `run-network` of MiV-Simulator **case 6** (`6-gapjunctions`,
`Microcircuit_Small` + gap junctions) on the HDF Group **jelly** machine
(CentOS 7, 56 cores, 125 GB RAM), single node, CPU, **native parallel HDF5
(MPI-IO)** — no IOWarp. The entire toolchain, datasets, and run were built from
scratch on jelly on 2026-06-20. This is the jelly **baseline** complementing the
ares end-to-end study in [miv_iowarp_ares_case6.md](miv_iowarp_ares_case6.md).

## Software stack (one consistent spack gcc-10.2.0 toolchain)

- **gcc 10.2.0 + OpenMPI 5.0.5 + parallel HDF5 1.14.6** (prebuilt in spack under
  `/scr/hyoklee`), cmake 3.26.3. The system HDF5 (`/usr/hdf`, 1.8.7) is too old;
  the spack 1.14.6 install (`Parallel HDF5: ON`) is reused.
- miniconda3 (`/mnt/wrk/hyoklee/miniconda3`, glibc-2.17-compatible py311_23.11.0),
  env `miv` Python 3.11: **mpi4py 4.1.2** (OpenMPI 5.0.5), **h5py 3.16.0**
  (HDF5 1.14.6, MPI=on), **NEURON 9.0a0**, **neuroh5 0.1.18** (built from
  `hyoklee/neuroh5`, C++ io.so + `neurotrees_import`/`neurotrees_copy` CLI),
  **miv_simulator 0.3.0** (`hyoklee/MiV-Simulator`).
- All three layers (neuroh5, mpi4py, h5py) link the **same** MPI + HDF5 — the
  prerequisite for parallel correctness. Env script: `/mnt/wrk/hyoklee/miv_env.sh`.

## Model

- 187 cells: **PYR 80, PVBC 53, OLM 44, STIM 10** (matches ares).
- Synaptic connectivity: **PYR 2,671,902 + PVBC 435,557 + OLM 160,528 edges**
  (~3.27M synaptic edges).
- Gap junctions: **102 PYR↔PYR edges** (identical to the ares result).
- Datasets (`datasets/Microcircuit_Small/`): Cells **36 MB**, Connections
  **8.9 MB**, GapJunctions **51 KB** — same sizes as ares.
- **Datasets constructed from scratch** (no pre-built data on jelly): make-h5types
  → generate-soma-coordinates → measure-distances → `neurotrees_import`/`copy`
  (forests from PYR/PVBC/OLM `.swc`) → distribute-synapse-locs →
  generate-distance-connections → generate-gapjunctions → finalize (h5py + h5copy).

## Rank scaling (single node, tstop=50 ms, dt=0.025, native HDF5)

| ranks | created cells | **connected cells** | gap junctions | ran simulation (50 ms) | **total wall** | speedup | max RSS |
|------:|-------------:|--------------------:|--------------:|-----------------------:|---------------:|--------:|--------:|
| 4  | 1.52 s | 82.64 s | 0.02 s | 16.77 s | **105.9 s** | 1.00× | 1.53 GB |
| 8  | 0.76 s | 43.28 s | 0.02 s | 10.96 s | **58.8 s**  | 1.80× | 838 MB |
| 16 | 0.42 s | 23.40 s | 0.02 s | 5.67 s  | **33.1 s**  | 3.20× | 496 MB |

**Scaling is near-ideal on one node.** The dominant phase — *connected cells*
(synapse instantiation + edge-attribute reads from the 8.9 MB Connections file via
MPI-IO) — scales **1.91× (4→8)** and **1.85× (8→16)**, i.e. 88–95% parallel
efficiency; total wall is **3.20×** faster on 16 vs 4 ranks. The pure-compute
*ran simulation* phase scales too (16.77→5.67 s). Per-rank RSS halves as ranks
double (work is partitioned), so case 6 is not memory-bound on jelly. Gap-junction
construction is negligible (0.02 s, 102 edges) and cell creation is sub-second.

This breakdown mirrors the ares phase structure (created → connected → ran), and
the connected-cells phase is again the I/O+setup bottleneck while the simulation
is pure compute — the same compute-bound, read-once regime documented on ares,
where the IOWarp POSIX CTE adapter was ~15% slower. jelly carries no IOWarp build;
this page establishes the **native-HDF5 baseline** and that case 6 builds and runs
end-to-end on jelly with the spack HDF5 1.14.6 stack.

## Longer simulation (np=8, tstop=500 ms) — the compute phase dominates

| phase | tstop=50 ms | **tstop=500 ms** |
|-------|------------:|-----------------:|
| created cells | 0.76 s | 0.76 s |
| connected cells | 43.28 s | 44.44 s |
| created gap junctions | 0.02 s | 0.02 s |
| **ran simulation** | 10.96 s | **167.68 s** |
| **total wall** | 58.8 s | **219.1 s** |

The *connected cells* phase is **tstop-independent** (43.3→44.4 s, noise), while
*ran simulation* grows with integration length and at tstop=500 ms becomes the
**dominant cost (167.7 of 219 s, 77%)**. So at a production tstop the workload is
**compute-bound, not I/O-bound** — the regime where the ares study found an I/O
accelerator (IOWarp CTE) cannot help, since it can't speed a pure-compute phase
and the ~45 MB of input is read once. Same conclusion, independently reproduced on
jelly with native HDF5.

## Blockers fixed during bring-up

- **Stale-binary artifact** made OpenMPI 5.0.5 appear to give `MPI_COMM_WORLD`
  size 0; a fresh recompile showed correct ranks (np=4 → 0–3). Toolchain is sound.
- **glibc 2.17**: latest Miniconda requires glibc ≥2.28; used py311_23.11.0. Latest
  scipy/wheels need glibc 2.28 too — pinned **scipy 1.13.1** (manylinux_2_17).
- **libstdc++ conflict**: neuroh5 `io.so` has DT_RPATH to gcc-10.2 libstdc++
  (GLIBCXX_3.4.28), which shadowed conda's libspatialindex (needs 3.4.34) →
  `rtree` failed. Fixed with `LD_PRELOAD` of conda's newer libstdc++ (a superset).
- **`pkg_resources` missing**: setuptools 82 removed it (breaks `nrnivmodl`) →
  pinned setuptools <81.
- **`cacum already exists`**: NEURON auto-loads a root `x86_64/` *and* MiV loads
  `mechanisms/compiled/<hash>/` → mechanism double-registration. `run-network`
  needs **no** root `x86_64/`; `generate-gapjunctions` needs it (it does not use
  MiV's loader) — so they are run in opposite mechanism states.
- **`MPI_Abort(0)` at exit**: MiV MPI scripts abort on teardown and prterun then
  signals the parent shell; each step is wrapped in `setsid` to isolate it.
- **Data path**: `run-network` reads `<prefix>/<Dataset Name>/` =
  `datasets/Microcircuit_Small/`, not `datasets/`.
- **lfpout collision**: the post-sim LFP write fails if `results/` already holds a
  prior run's group — clean `results/runs` per run.

## Known limitation

**STIM input spikes were omitted.** The fork's `generate-input-features` CLI is
API-incompatible with its own `input_features.py` (`unexpected keyword argument
'config_prefix'`), so STIM carries no spike train and the network has no external
drive. The measured I/O + setup + integration phases are unaffected (the network
is fully wired, gap junctions included), but spike output is near-silent. A
biologically faithful run needs that script fixed in the fork.

Full report + logs: `~/MiV-Simulator-Cases/6-gapjunctions/PERFORMANCE.md`,
`logs/sweep_np{4,8,16}_t50.log`.
