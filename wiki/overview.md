# neuroh5 HDF5 Samples — Overview

neuroh5 is a parallel HDF5-based library for storing and processing large-scale neural network graphs and cell model attributes. Its HDF5 format encodes synaptic connectivity as directed graphs (adjacency lists) where vertices are neurons identified by integer global IDs (GIDs) and edges carry float attributes such as synaptic distance. The library also stores neuronal morphologies (SWC trees), spatial coordinates, and stimulus data.

## Sample collections

### Dentate gyrus samples

| File | Size | Description |
|------|------|-------------|
| [example.h5](source-example_h5.md) | tiny | Minimal synthetic GC→GC projection; one population, one projection |
| [dentate_test.h5](source-dentate_test_h5.md) | small | Dentate gyrus circuit; 11 populations, ~40 valid projections across 6 cell types |

### MiV Microcircuit samples

A hippocampal CA1 network model with 4 populations (82,912 neurons total) and 10 directed projections. Files span connectivity, morphology, spatial coordinates, and stimulus data.

| File | Status | Description |
|------|--------|-------------|
| [MiV_h5types.h5](source-MiV_h5types_h5.md) | OK | Population/projection registry only |
| [Microcircuit_coords.h5](source-Microcircuit_coords_h5.md) | OK | Arc distances + generated coords, all 4 pops |
| [MiV_input_features.h5](source-MiV_input_features_h5.md) | OK | STIM place-field features |
| [MiV_input_spikes.h5](source-MiV_input_spikes_h5.md) | OK | STIM spike trains + trajectory |
| [OLM_forest.h5](source-OLM_forest_h5.md) | OK | OLM morphology forest (438 cells) |
| [OLM_tree.h5](source-OLM_tree_h5.md) | OK | Single OLM exemplar |
| [PVBC_tree.h5](source-PVBC_tree_h5.md) | OK | Single PVBC exemplar |
| [PYR_forest.h5](source-PYR_forest_h5.md) | OK (stub) | PYR cell registry, empty coords |
| [PYR_tree.h5](source-PYR_tree_h5.md) | OK | Single PYR exemplar |
| MiV_Cells/Connections (×4), OLM/PVBC/PYR forests+syns+connections | Truncated | Filesystem truncation; unreadable |

See [MiV Microcircuit](concept-miv-microcircuit.md) for the full file inventory and population table.

## Key concepts

- [Destination Block Sparse (DBS) Format](concept-dbs-format.md) — the core sparse adjacency representation
- [Populations](concept-populations.md) — sets of neurons sharing a biological type, identified by GID ranges
- [Projections](concept-projections.md) — directed connectivity sets between two populations
- [H5Types](concept-h5types.md) — committed HDF5 compound/enum datatypes used across files
- [Dentate Gyrus Cell Types](concept-dentate-gyrus.md) — the six cell populations in dentate_test.h5
- [MiV Microcircuit](concept-miv-microcircuit.md) — 4-population CA1 model (STIM/PYR/PVBC/OLM)
- [Morphology SWC Format](concept-morphology-swc.md) — per-cell tree morphology in forest/tree files
- [Cell Coordinates and Arc Distances](concept-cell-coordinates.md) — curvilinear hippocampal coordinate system

## Common format structure (connectivity files)

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

## Morphology file structure (forest/tree files)

```
/
├── H5Types/
└── Populations/
    └── <POP>/
        └── Trees/
            ├── Cell Index
            ├── X/Y/Z Coordinate/   ← Attribute Pointer + Value + Cell Index
            ├── Radius/
            ├── SWC Type/
            ├── Point Layer/
            ├── Parent Point/
            ├── Section/
            ├── Source Section/
            └── Destination Section/
```

> Note: `example.h5` uses a slightly older layout where the projection group is named `<SRC>to<DST>` and the DBS arrays live under `Connectivity/` rather than `Edges/`.
