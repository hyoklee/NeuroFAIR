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

- [MiV-Simulator 7-optimization on Aurora](source-MiV_Optimizer_test.md) — PBS jobs 8448630–8449551: dmosopt NSGA-II synaptic weight optimization; Run 1 failed (OpenMPI PRRTE slot error); Run 2 pending (Cray PALS mpiexec fix)

## Build and Test Reports

- [MiV-Simulator build and test on Aurora](source-MiV_Simulator_build_test.md) — PBS job results: neuroh5 cmake build, 208 tests (170+38 pass), CoreNEURON GPU, known issues
- [MiV-Simulator build and test on Polaris](miv_polaris_build_test.md) — PBS jobs 7097199–7097215: neuroh5 cmake (gcc/GNU MPICH), 204/223 tests pass, CoreNEURON A100 GPU confirmed; GNU Cray MPICH SHM transport blocks multi-rank neuroh5 I/O
- [MiV-Simulator PR #103 on Polaris](miv_pr103_polaris_test.md) — PBS job 7097324: patch fixes np.float_ (test_coding.py 3/3 pass), mpi_env check requires MIV_SKIP_MPI_CHECK=1; 207/226 pass
- [neuroh5 shared-memory crash fix](neuroh5_shm_crash_fix.md) — int overflow in alltoallv sdispls for >2 GB datasets; P2P Isend/Irecv fix; **12/12 I/O tests pass** (job 7097644); PR: iraikov/neuroh5#19

## Schema

- [CLAUDE.md](CLAUDE.md) — wiki schema and workflows for LLM maintenance
