# Wiki Index

## Overview

- [Overview](overview.md) — high-level synthesis of neuroh5 HDF5 samples

## Sources

- [example.h5](source-example_h5.md) — minimal GC→GC projection (1 population, 14 edges)
- [dentate_test.h5](source-dentate_test_h5.md) — dentate gyrus circuit (11 populations, 40 projections)

## Concepts

- [Destination Block Sparse (DBS) Format](concept-dbs-format.md) — sparse adjacency representation used for all connectivity
- [Populations](concept-populations.md) — GID-ranged neuron groups; registry in /H5Types/Populations
- [Projections](concept-projections.md) — directed connectivity sets between populations; per-edge distance attributes
- [H5Types](concept-h5types.md) — committed HDF5 type registry (Population labels, range, projections)
- [Dentate Gyrus Cell Types](concept-dentate-gyrus.md) — GC, MC, AAC, BC, HC, NGFC in dentate_test.h5

## Schema

- [CLAUDE.md](CLAUDE.md) — wiki schema and workflows for LLM maintenance
