# Concept: Projections

## Definition

A **projection** is the complete set of directed synaptic connections from one neuron population (source) to another (destination). It is identified by the (source population, destination population) pair and stored as a group under `/Projections/`.

## HDF5 layout (current)

```
/Projections/<SRC>/<DST>/
    Edges/       ← DBS connectivity arrays
    Attributes/  ← per-edge float attributes
```

## HDF5 layout (legacy, example.h5)

```
/Projections/<SRC>to<DST>/
    Connectivity/  ← DBS arrays
    Attributes/Edge/  ← per-edge float attributes
    Source Population
    Destination Population
```

## Edge attributes

Both files carry spatial distance attributes per edge:

| Attribute | Dtype | Description |
|-----------|-------|-------------|
| `Longitudinal Distance` / `Longitudinal` | float32 | Along-axis synaptic distance (µm) |
| `Transverse Distance` / `Transverse` | float32 | Cross-axis synaptic distance (µm) |

In `example.h5` the range is 60–623 µm longitudinal.

## Valid projections registry

`/H5Types/Valid population projections` lists all permitted (Source, Destination) label pairs:
- `example.h5`: shape (1,) — one pair
- `dentate_test.h5`: shape (40,) — 40 pairs

## Related concepts

- [DBS Format](concept-dbs-format.md)
- [Populations](concept-populations.md)
- [Dentate Gyrus Cell Types](concept-dentate-gyrus.md)
