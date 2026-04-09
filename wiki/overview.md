# neuroh5 HDF5 Samples — Overview

neuroh5 is a parallel HDF5-based library for storing and processing large-scale neural network graphs and cell model attributes. Its HDF5 format encodes synaptic connectivity as directed graphs (adjacency lists) where vertices are neurons identified by integer global IDs (GIDs) and edges carry float attributes such as synaptic distance.

## Sample files

| File | Size | Description |
|------|------|-------------|
| [example.h5](source-example_h5.md) | tiny | Minimal synthetic GC→GC projection; one population, one projection |
| [dentate_test.h5](source-dentate_test_h5.md) | small | Dentate gyrus circuit; 11 populations, ~40 valid projections across 6 cell types |

## Key concepts

- [Destination Block Sparse (DBS) Format](concept-dbs-format.md) — the core sparse adjacency representation
- [Populations](concept-populations.md) — sets of neurons sharing a biological type, identified by GID ranges
- [Projections](concept-projections.md) — directed connectivity sets between two populations
- [H5Types](concept-h5types.md) — committed HDF5 compound/enum datatypes used across files
- [Dentate Gyrus Cell Types](concept-dentate-gyrus.md) — the six cell populations in dentate_test.h5

## Format structure (both files)

```
/
├── H5Types/          ← committed datatypes + registry datasets
│   ├── Populations   ← GID start/count/label per population
│   └── Valid population projections
└── Projections/      ← one group per source population
    └── <SRC>/
        └── <DST>/
            ├── Edges/        ← DBS arrays (Source Index, Destination Block *, Destination Pointer)
            └── Attributes/   ← per-edge float attributes (distance etc.)
```

> Note: `example.h5` uses a slightly older layout where the projection group is named `<SRC>to<DST>` and the DBS arrays live under `Connectivity/` rather than `Edges/`.
