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
- [OLM_tree.h5](source-OLM_tree_h5.md) — single OLM cell exemplar (GID 0, 954 points)
- [PVBC_tree.h5](source-PVBC_tree_h5.md) — single PVBC cell exemplar (GID 0, 1820 points)
- [PYR_forest.h5](source-PYR_forest_h5.md) — PYR forest stub (80k cell registry, no coord data)
- [PYR_tree.h5](source-PYR_tree_h5.md) — single PYR cell exemplar (GID 0, 10346 points)

## Concepts

- [Destination Block Sparse (DBS) Format](concept-dbs-format.md) — sparse adjacency representation used for all connectivity
- [Populations](concept-populations.md) — GID-ranged neuron groups; registry in /H5Types/Populations
- [Projections](concept-projections.md) — directed connectivity sets between populations; per-edge distance attributes
- [H5Types](concept-h5types.md) — committed HDF5 type registry (Population labels, range, projections)
- [Dentate Gyrus Cell Types](concept-dentate-gyrus.md) — GC, MC, AAC, BC, HC, NGFC in dentate_test.h5
- [MiV Microcircuit](concept-miv-microcircuit.md) — 4-population CA1 model (STIM/PYR/PVBC/OLM, 82912 neurons)
- [Morphology SWC Format](concept-morphology-swc.md) — per-cell tree morphology layout in forest/tree files
- [Cell Coordinates and Arc Distances](concept-cell-coordinates.md) — curvilinear hippocampal coordinate system

## Schema

- [CLAUDE.md](CLAUDE.md) — wiki schema and workflows for LLM maintenance
