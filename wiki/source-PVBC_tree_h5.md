# Source: PVBC_tree.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/PVBC_tree.h5`
**Dataset:** MiV Microcircuit — single PVBC cell morphology (GID 0)

## Summary

Single-cell exemplar morphology for PVBC (parvalbumin-positive basket cell) interneuron GID 0. Contains 1820 morphology points, 1838 sections, and 16 synapse placement records per Source/Destination Section. Point Layer = 6 throughout (stratum pyramidale). SWC Type = 1 (soma). More complex morphology than OLM (1820 vs 954 points) with richer synapse mapping (16 vs 3 records).

## Top-level structure

```
/
├── H5Types/
│   ├── Populations                    ← 3 populations (no STIM)
│   └── Valid population projections   ← 9 projections
└── Populations/
    └── PVBC/
        └── Trees/
            ├── Cell Index              (1 entry: GID 0)
            ├── X Coordinate/           (1820 float32, all 0.0)
            ├── Y Coordinate/           (1820 float32, 0 to ~9.47 µm)
            ├── Z Coordinate/           (1820 float32, all 0.0)
            ├── Radius/                 (1820 float32, all 5.0 µm)
            ├── SWC Type/               (1820 int8, all 1)
            ├── Point Layer/            (1820 int8, all 6)
            ├── Parent Point/           (1820 int32)
            ├── Section/                (1838 uint16)
            ├── Source Section/         (16 uint16)
            └── Destination Section/    (16 uint16)
```

## Comparison with OLM_tree.h5

| Property | PVBC | OLM |
|----------|------|-----|
| Points   | 1820 | 954 |
| Sections | 1838 | 959 |
| Synapse records | 16 | 3 |
| Point Layer | 6 (str. pyramidale) | 5 (str. oriens) |
| Radius | 5.0 µm | 5.0 µm |

## Related

- [MiV Microcircuit](concept-miv-microcircuit.md)
- [OLM_tree.h5](source-OLM_tree_h5.md)
- [PYR_tree.h5](source-PYR_tree_h5.md)
- [Morphology SWC Format](concept-morphology-swc.md)
