# Source: PYR_tree.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/PYR_tree.h5`
**Dataset:** MiV Microcircuit — single PYR cell morphology (GID 0)

## Summary

Single-cell exemplar morphology for PYR (pyramidal) neuron GID 0. The most complex of the three single-cell morphology files: 10,346 morphology points, 10,549 sections, and 201 synapse placement records. Point Layer = 6 (stratum pyramidale). Unlike OLM and PVBC, the PYR radius is heterogeneous (1.7–3.7+ µm), reflecting the branching dendritic complexity of pyramidal cells. Y coordinate spans ~0 to ~102 µm.

## Top-level structure

```
/
├── H5Types/
│   ├── Populations                    ← 3 populations (no STIM)
│   └── Valid population projections   ← 9 projections
└── Populations/
    └── PYR/
        └── Trees/
            ├── Cell Index              (1 entry: GID 0)
            ├── X Coordinate/           (10346 float32, all 0.0)
            ├── Y Coordinate/           (10346 float32, 0 to ~102 µm)
            ├── Z Coordinate/           (10346 float32, all 0.0)
            ├── Radius/                 (10346 float32, 1.7–3.7+ µm variable)
            ├── SWC Type/               (10346 int8, all 1)
            ├── Point Layer/            (10346 int8, all 6)
            ├── Parent Point/           (10346 int32)
            ├── Section/                (10549 uint16)
            ├── Source Section/         (201 uint16)
            └── Destination Section/    (201 uint16)
```

## Morphology complexity comparison

| Property | PYR | PVBC | OLM |
|----------|-----|------|-----|
| Points | 10,346 | 1,820 | 954 |
| Sections | 10,549 | 1,838 | 959 |
| Synapse records | 201 | 16 | 3 |
| Radius | 1.7–3.7+ µm (variable) | 5.0 µm | 5.0 µm |
| Point Layer | 6 | 6 | 5 |

## Related

- [PYR_forest.h5](source-PYR_forest_h5.md)
- [MiV Microcircuit](concept-miv-microcircuit.md)
- [PVBC_tree.h5](source-PVBC_tree_h5.md)
- [OLM_tree.h5](source-OLM_tree_h5.md)
- [Morphology SWC Format](concept-morphology-swc.md)
