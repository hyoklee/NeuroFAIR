# Source: OLM_tree.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/OLM_tree.h5`
**Dataset:** MiV Microcircuit — single OLM cell morphology (GID 0)

## Summary

Single-cell exemplar morphology for OLM (oriens-lacunosum-moleculare) interneuron GID 0. Identical layout to [OLM_forest.h5](source-OLM_forest_h5.md) but with only 1 cell and 954 morphology points. The H5Types table here omits the STIM population (3 populations: PYR/PVBC/OLM, 9 projections). Used for single-cell inspection and testing.

## Top-level structure

```
/
├── H5Types/
│   ├── Populations                    ← 3 populations (no STIM)
│   └── Valid population projections   ← 9 projections
└── Populations/
    └── OLM/
        └── Trees/
            ├── Cell Index              (1 entry: GID 0)
            ├── X Coordinate/           (954 float32, all 0.0)
            ├── Y Coordinate/           (954 float32, 0 to ~9.47 µm)
            ├── Z Coordinate/           (954 float32, all 0.0)
            ├── Radius/                 (954 float32, all 5.0 µm)
            ├── SWC Type/               (954 int8, all 1)
            ├── Point Layer/            (954 int8, all 5)
            ├── Parent Point/           (954 int32)
            ├── Section/                (959 uint16)
            ├── Source Section/         (3 uint16, all 3)
            └── Destination Section/    (3 uint16: 1, 2, 0)
```

## Populations (3, no STIM)

| Index | Start GID | Count  | Pop ID |
|-------|-----------|--------|--------|
| 0     | 0         | 80000  | 100 (PYR)  |
| 1     | 80000     | 1474   | 101 (PVBC) |
| 2     | 81474     | 438    | 102 (OLM)  |

## Related

- [OLM_forest.h5](source-OLM_forest_h5.md)
- [MiV Microcircuit](concept-miv-microcircuit.md)
- [Morphology SWC Format](concept-morphology-swc.md)
