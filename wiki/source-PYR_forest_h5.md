# Source: PYR_forest.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/PYR_forest.h5`
**Dataset:** MiV Microcircuit — PYR population morphological forest (stub)

## Summary

Population-level morphology file for the PYR population (80,000 cells). Unlike [OLM_forest.h5](source-OLM_forest_h5.md), the coordinate datasets here are **empty** (shape `(0,)`) — only the Cell Index is populated. This indicates the file is a stub or placeholder where the cell registry is written but morphology data has not yet been filled in. Uses gzip compression on datasets.

## Top-level structure

```
/
├── H5Types/
│   ├── Populations                    ← 3 populations (no STIM)
│   └── Valid population projections   ← 9 projections
└── Populations/
    └── PYR/
        └── Trees/
            ├── Cell Index              (80000 uint32, IDs 79999→0 descending, gzip)
            └── X Coordinate/
                ├── Attribute Pointer   (0 elements — empty)
                ├── Attribute Value     (0 elements — empty)
                └── Cell Index          (80000 uint32, gzip)
```

## Notable: empty coordinate data

The `X Coordinate` group (and by extension all other coordinates) has zero-length `Attribute Pointer` and `Attribute Value` arrays. This is a known pattern when the forest skeleton is written before morphology assignment. Compare with [OLM_forest.h5](source-OLM_forest_h5.md) which has 417,852 points for 438 cells.

## Related

- [PYR_tree.h5](source-PYR_tree_h5.md) — single PYR exemplar with full morphology
- [OLM_forest.h5](source-OLM_forest_h5.md) — complete forest example
- [MiV Microcircuit](concept-miv-microcircuit.md)
- [Morphology SWC Format](concept-morphology-swc.md)
