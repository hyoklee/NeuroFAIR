# Concept: H5Types

## Definition

`/H5Types` is a top-level group in every neuroh5 file that serves as a type registry. It contains **committed HDF5 datatypes** (named, reusable type definitions) and **registry datasets** that enumerate populations and valid projections.

## Contents

### Committed datatypes

| Name | Base dtype | Role |
|------|-----------|------|
| `Population labels` | uint16 | Enumerated IDs for each cell-type population |
| `Population projections` | compound (Source u2, Destination u2) | Pair of population labels for a projection |
| `Population range` | compound (Start u8, Count u4, Population u2) | GID range descriptor |
| `Layer tags` | uint8 | Cortical layer tags *(example.h5 only)* |

### Registry datasets

| Name | Shape (example / dentate) | Dtype | Role |
|------|--------------------------|-------|------|
| `Populations` | (1,) / (11,) | Population range | Maps population label → GID start + count |
| `Valid population projections` | (1,) / (40,) | Population projections | Allowed (src, dst) projection pairs |

## Where it appears

- `example.h5` — 6 members (includes `Layer tags`)
- `dentate_test.h5` — 5 members (no `Layer tags`)

## Related concepts

- [Populations](concept-populations.md)
- [Projections](concept-projections.md)
