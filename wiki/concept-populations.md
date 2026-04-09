# Concept: Populations

## Definition

A **population** is a set of neurons of the same biological type. Each neuron in a population is identified by a unique integer global identifier (GID). Populations partition the GID space into contiguous ranges.

## HDF5 representation

Stored in `/H5Types/Populations` as a 1-D compound dataset with fields:

| Field | Dtype | Meaning |
|-------|-------|---------|
| `Start` | uint64 | First GID in this population |
| `Count` | uint32 | Number of neurons |
| `Population` | uint16 | Enumerated population label (see `Population labels` type) |

## Where it appears

| File | `/H5Types/Populations` shape | Populations |
|------|------------------------------|-------------|
| `example.h5` | (1,) | 1 population (GC) |
| `dentate_test.h5` | (11,) | 11 populations across 6 cell types |

## Related concepts

- [Projections](concept-projections.md)
- [H5Types](concept-h5types.md)
- [Dentate Gyrus Cell Types](concept-dentate-gyrus.md)
