# Wiki Index

## Overview

- [Overview](overview.md) — high-level synthesis of neuroh5 HDF5 samples

## Sources — dentate gyrus samples

- [example.h5](source-example_h5.md) — minimal GC→GC projection (1 population, 14 edges)
- [dentate_test.h5](source-dentate_test_h5.md) — dentate gyrus circuit (11 populations, 40 projections)

## Sources — MiV Microcircuit

- [MiV_h5types.h5](source-MiV_h5types_h5.md) — population/projection registry (4 pops, 10 projections)
- [Microcircuit_coords.h5](source-Microcircuit_coords_h5.md) — arc distances + generated coordinates for all 4 populations
- [MiV_input_features.h5](source-MiV_input_features_h5.md) — STIM place-field features (peak rate 20 Hz, selectivity type)
- [MiV_input_spikes.h5](source-MiV_input_spikes_h5.md) — STIM spike trains (586K spikes) + spatial trajectory
- [OLM_forest.h5](source-OLM_forest_h5.md) — OLM morphology forest (438 cells, ~954 pts/cell)
- [OLM_forest_syns.h5](source-OLM_forest_syns_h5.md) — OLM synapse attribute table (1.59M synapses, 6 attributes)
- [OLM_connections.h5](source-OLM_connections_h5.md) — OLM outgoing connectivity (360K→PVBC, 1.23M→PYR edges)
- [OLM_tree.h5](source-OLM_tree_h5.md) — single OLM cell exemplar (GID 0, 954 points)
- [PVBC_forest.h5](source-PVBC_forest_h5.md) — PVBC morphology forest (1,474 cells, 1820 pts/cell)
- [PVBC_forest_syns.h5](source-PVBC_forest_syns_h5.md) — PVBC synapse attribute table (~12.1M synapses, 6 attributes)
- [PVBC_connections.h5](source-PVBC_connections_h5.md) — PVBC outgoing connectivity (12.1M total edges, 4 projections)
- [PVBC_tree.h5](source-PVBC_tree_h5.md) — single PVBC cell exemplar (GID 0, 1820 points)
- [PYR_forest.h5](source-PYR_forest_h5.md) — PYR forest stub (80k cell registry, no coord data)
- [PYR_forest_compressed.h5](source-PYR_forest_compressed_h5.md) — full PYR morphology (80k cells, gzip, 4.4 GB)
- [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md) — full PYR connectivity (2.68B edges, gzip, 6.2 GB)
- [PYR_tree.h5](source-PYR_tree_h5.md) — single PYR cell exemplar (GID 0, 10346 points)
- [MiV_Connections_Microcircuit_20220410.h5](source-MiV_Connections_Microcircuit_20220410_h5.md) — all-population connectivity snapshot (2022-04-10, 6.4 GB)
- [MiV_Cells_Microcircuit_20220410.h5](source-MiV_Cells_Microcircuit_20220410_h5.md) — full-circuit cell attributes snapshot (2022-04-10, 15.7 GB)
- [MiV_Cells_Microcircuit_20220412.h5](source-MiV_Cells_Microcircuit_20220412_h5.md) — full-circuit cell attributes snapshot (2022-04-12, 18.4 GB)
- [MiV_Connections_Microcircuit_20220412.h5](source-MiV_Connections_Microcircuit_20220412_h5.md) — all-population connectivity snapshot (2022-04-12, 6.9 GB)
- [PYR_forest_syns_compressed.h5](source-PYR_forest_syns_compressed_h5.md) — PYR synapse attribute table (2.68B synapses, 7 attributes, gzip, 11.2 GB)

## Sources — Motoneuron Model (Motoneuron_model/)

- [dmosopt_Motoneuron.h5](source-dmosopt_Motoneuron_h5.md) — DMOSOPT optimization results (42K evaluations, 11 params, 5 objectives, 121 MB)

## Sources — MiV Microcircuit Small (Microcircuit_Small/)

- [MiV_Cells_Microcircuit_Small_20220410.h5](source-MiV_Cells_Microcircuit_Small_20220410_h5.md) — small-scale cell coordinates (44 OLM cells, ~10% subset)
- [MiV_Connections_Microcircuit_Small_20220410.h5](source-MiV_Connections_Microcircuit_Small_20220410_h5.md) — small-scale connectivity (36K OLM→PVBC, 124K OLM→PYR edges)

## Concepts

- [Destination Block Sparse (DBS) Format](concept-dbs-format.md) — sparse adjacency representation used for all connectivity
- [Populations](concept-populations.md) — GID-ranged neuron groups; registry in /H5Types/Populations
- [Projections](concept-projections.md) — directed connectivity sets between populations; per-edge distance attributes
- [H5Types](concept-h5types.md) — committed HDF5 type registry (Population labels, range, projections)
- [Dentate Gyrus Cell Types](concept-dentate-gyrus.md) — GC, MC, AAC, BC, HC, NGFC in dentate_test.h5
- [MiV Microcircuit](concept-miv-microcircuit.md) — 4-population CA1 model (STIM/PYR/PVBC/OLM, 82912 neurons)
- [Morphology SWC Format](concept-morphology-swc.md) — per-cell tree morphology layout in forest/tree files
- [Cell Coordinates and Arc Distances](concept-cell-coordinates.md) — curvilinear hippocampal coordinate system

## Optimization Experiments

- [MiV-Simulator 7-optimization on Aurora](source-MiV_Optimizer_test.md) — PBS jobs 8448630–8452563: dmosopt NSGA-II synaptic weight optimization; Runs 1–8 documented; cells firing confirmed in Run 8; Run 9 running

## LLM / AI

- [IPEX-LLM on Aurora](concept-ipex-llm-aurora.md) — conda env `ipex-llm`; torch 2.8.0 + IPEX 2.8.10 (Aurora wheelhouse) + ipex-llm 2.2.0; INT4 inference on Intel GPU XPU

## Performance Studies

- [MiV case 6 (gap junctions) — native parallel HDF5 on jelly](miv_jelly_case6.md) — full from-scratch bring-up on the HDF Group **jelly** node (gcc 10.2.0 + OpenMPI 5.0.5 + parallel HDF5 1.14.6, NEURON 9.0a0, neuroh5 0.1.18, miv_simulator 0.3.0). Case 6 (187 cells, ~3.27M synaptic edges, 102 PYR↔PYR gap junctions) **builds and runs end-to-end**; **near-ideal single-node rank scaling** (tstop=50): total wall 105.9 s (4) → 58.8 s (8) → 33.1 s (16), connected-cells phase 82.6→43.3→23.4 s (88–95% par. eff.). Native-HDF5 baseline (no IOWarp); 8 bring-up blockers fixed; STIM input spikes omitted (fork `generate-input-features` API bug)
- [MiV-Simulator GPU vs CPU Benchmark](miv_gpu_bench.md) — Polaris A100; standard NEURON CPU baseline **481–520 s** (tstop=500ms, 1 rank, Microcircuit_Small); CoreNEURON CPU blocked by SIGABRT inside psolve; GPU requires NEURON rebuild with `CORENRN_ENABLE_GPU=ON`; 4 bugs fixed in neuroh5/MiV-Simulator during debugging (2026-05-20/21)
- [MiV-Simulator + IOWarp CTE Benchmark](miv_iowarp_bench.md) — IOWarp Context Transfer Engine RAM-tier buffering; scatter_read_trees + optimize-network comparison; Chimaera ZMQ bind fails on all Polaris compute node IPs; VecStim namespace fix confirmed n_active=80/53/44; bench6 27 evals, ~486s/eval
- [clio-core CTE buffering for MiV optimization](perf-clio-core.md) — PBS jobs 8452562/8452563: Lustre vs /dev/shm (CTE RAM tier) I/O benchmark; baseline setup=25.84 s; full VFD integration path documented
- [clio-core CTE: GPU memory vs CPU DRAM for MiV](concept-clio-core-gpu-memory.md) — why GPU HBM2e does not help current workload (no CoreNEURON, no GPU CTE backend); path to full GPU benefit documented
- [MiV Case 6 (gap junctions) perf comparison](perf-case6-gapjunctions.md) — PBS 8473164: 4/4 conditions blocked. CPU runs SIGSEGV in `init_network` (gap-junction yaml suspected); `special-core` build fails on `nrnunits.lib` resolution. Pre-stage 129 MB in 8.288 s recorded.
- [MiV case 6 — IOWarp core vs native HDF5 (ares, end-to-end)](miv_iowarp_ares_case6.md) — whole-application `run-network` comparison: IOWarp POSIX adapter ~15% slower on 1 node (compute-bound, read-once), multi-node deadlocks; native HDF5 scales to 2 nodes (1.76×). Fork `~/core` vs upstream `iowarp/clio-core` v2.0.0 (`~/core.iowarp`) within 0.4% — no build difference; CTE RAM tier vs node-local NVMe tier within 0.3% — no tier difference
- [MiV case 4 (opsin) — IOWarp core vs native HDF5, RAM vs NVMe (ares)](miv_iowarp_ares_case4.md) — confirms the case-6 finding on a second use case: IOWarp ~17.5% slower (compute-bound), RAM vs NVMe tier no difference; case 4 = case 6 minus gap junctions (opsin is a TODO no-op in run-network)
- [MiV case 7 (optimization) — IOWarp core vs native HDF5, RAM vs NVMe (ares)](miv_iowarp_ares_case7.md) — `optimize-network` (dmosopt, repeated-read regime): IOWarp ~11% slower, RAM vs NVMe no difference. Even with repeated network reads, per-eval compute (~140 s) dwarfs the ~45 MB read so CTE can't help. optimize-network runs on ares without deadlock (standard NEURON + distwq)
- [Designing a MiV workload where CTE beats baseline (ares)](miv_iowarp_ares_caseio.md) — write-bound benchmark (dense recording + checkpoints) built to favor CTE; still ~11% slower on the small circuit (compute-dominated + adapter overhead). Measured ares I/O hierarchy (NFS 590/215, NVMe 1.5G/1.0G, 46 GB cache) + the recipe + what's needed for a real win (full-scale data / multi-node)
- [Read-bound CA1 benchmark — IOWarp CTE vs native HDF5 (ares)](miv_iowarp_ares_caseread.md) — full CA1 staged via Globus (~69 GB on NFS); 48.5 GB working set defeats the 46 GB page cache. **Plain read**: CTE = baseline within 0.05% across 3 iterations (~4030 s each); the NVMe tier stayed at 0 bytes used because the adapter only intercepts paths with the `clio::` marker. **Explicit pre-staging** (`/tmp/.../clio::file.h5`): writes land in the tier (6 GB blob), but read-back is **~4× slower** than NFS direct (36 MB/s vs 142 MB/s) and emits `GetBlob` errors on the last pages
- [Can IOWarp/clio-core help MiV case 7 on ares? — feasibility analysis](miv_iowarp_ares_case7_feasibility.md) — synthesis across case 7 / case 7 full / caseread / caseio: three independent blockers (MiV emits unprefixed paths; tier read path is 4× slower than NFS [#470]; case 7 is compute-bound). Bounded above by "= NFS" and below by "4× slower with errors"; no knob in between
- [MiV case 7 (optimization) on full CA1 — IOWarp vs native HDF5 (ares)](miv_iowarp_ares_case7_full.md) — `optimize-network` against the staged full CA1 dataset. Whole-population init OOMs at `Reading trees for population PYR` (~64 GB needed vs 46 GB node); the cell_selection bypass is blocked by two MiV-Simulator code-path bugs (PRN templates lack `gid` kwarg; HOC templates lack `mech_dict`). Across every attempt the NVMe tier stayed at **0 bytes used** — MiV passes unprefixed NFS paths so the adapter never intercepts. 11 jobs logged (20436–20456)
- [MiV case 7 with the HDF5 **VOL** CTE adapter (ares)](miv_iowarp_ares_case7_vol.md) — evaluates the VOL connector (`adapter/hdf5_vol/iowarp_vol.cc`) that blocker #2 pointed to, instead of the POSIX adapter. **No performance number possible**, for a stronger reason than POSIX: unbuilt (`CLIO_CTE_ENABLE_HDF5_VOL=OFF`, CMake guard demands HDF5≥2.0 but the API matches 1.14.6 — removable); no `H5PLget_plugin_*` export so it can't load without modifying MiV/neuroh5; and **decisively** its `dataset_read` serves data only from CTE blobs previously written through the connector — no native-file fallback, hyperslab `file_space` ignored. Pointed at case 7's pre-existing native `.h5` files it returns zero-filled buffers → neuroh5 crashes at init. It's a write-through-cache prototype; case 7 is a read-existing-files workload. **→ now fixed, see next**
- [Fixing the VOL CTE adapter — A/B/C resolved, read-path measured (ares)](miv_iowarp_ares_case7_vol_fix.md) — fixes all three blockers across `iowarp/clio-core` + `hyoklee/neuroh5` (branches `vol-cte-case7-fix`): lowered the CMake HDF5 floor to 1.14 (A); added `H5PLget_plugin_*` exports so `HDF5_VOL_CONNECTOR=iowarp` loads with no app change (B); rewrote `dataset_read` as a read-through cache with native fallback + hyperslab/collective passthrough + `datatype_cls` (C); plus fixed a client-bootstrap segfault and a fatal `H5Oopen` dataset mis-cast. **Validated**: unmodified h5py reads pre-existing native files correctly (whole + hyperslab + compound committed types). neuroh5 also ported off native-VOL-only deprecated APIs (`H5Literate1→2`, `H5_ITER_NATIVE→INC`, `H5Gget_num_objs→H5Gget_info`). **Remaining**: full case-7 init still needs passthrough link-iteration object-wrapping (a 4th, deeper gap). **Measurement**: 256 MB whole-read, native HDF5 190.6 ms vs CTE VOL 249.3 ms = **CTE ~31% slower** (page-cache-resident data on ares; no slow tier for the cache to win back) — consistent with all prior POSIX-adapter findings

## Build and Test Reports

- [MiV-Simulator build and test on Aurora](source-MiV_Simulator_build_test.md) — PBS job results: neuroh5 cmake build, 208 tests (170+38 pass), CoreNEURON GPU, known issues
- [MiV-Simulator build and test on Polaris](miv_polaris_build_test.md) — PBS jobs 7097199–7097215: neuroh5 cmake (gcc/GNU MPICH), 204/223 tests pass, CoreNEURON A100 GPU confirmed; GNU Cray MPICH SHM transport blocks multi-rank neuroh5 I/O
- [MiV-Simulator PR #103 on Polaris](miv_pr103_polaris_test.md) — PBS job 7097324: patch fixes np.float_ (test_coding.py 3/3 pass), mpi_env check requires MIV_SKIP_MPI_CHECK=1; 207/226 pass
- [neuroh5 shared-memory crash fix](neuroh5_shm_crash_fix.md) — int overflow in alltoallv sdispls for >2 GB datasets; P2P Isend/Irecv fix; **12/12 I/O tests pass** (job 7097644); PR: iraikov/neuroh5#19
- [MiV-Simulator 7-optimization on Polaris](miv_opt_7_polaris.md) — Case 7-optimization; dmosopt MOASMO; epoch 0 completed; tstop tuning + preemptable queue for 3h runs

## Schema

- [CLAUDE.md](CLAUDE.md) — wiki schema and workflows for LLM maintenance
