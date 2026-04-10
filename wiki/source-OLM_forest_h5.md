# Source: OLM_forest.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/OLM_forest.h5`
**Dataset:** MiV Microcircuit — OLM population morphological forest (full, 438 cells)

## Summary

Contains SWC-format neuronal morphologies for the entire OLM (oriens-lacunosum-moleculare interneuron) population (438 cells). Each cell has ~954 morphology points. Point Layer values are uniformly 5, indicating all OLM soma/dendrite points reside in layer 5 (stratum oriens). SWC Type = 1 throughout (soma). Section data includes ~959 sections/cell. Synapse placement metadata is stored in `Source Section` and `Destination Section` (3 synapses/cell, all from source section 3).

## Top-level structure

```
/
├── H5Types/
│   ├── Populations                    ← 4 populations (STIM/PYR/PVBC/OLM)
│   └── Valid population projections   ← 10 projections
└── Populations/
    └── OLM/
        └── Trees/
            ├── Cell Index              (438 uint32, IDs 437→0 descending)
            ├── X Coordinate/           (438 cells × ~954 pts, float32, all 0.0)
            ├── Y Coordinate/           (417852 float32, 0 to ~438 µm)
            ├── Z Coordinate/           (417852 float32, all 0.0)
            ├── Radius/                 (417852 float32, all 5.0 µm)
            ├── SWC Type/               (417852 int8, all 1 = soma)
            ├── Point Layer/            (417852 int8, all 5)
            ├── Parent Point/           (417852 int32, −1 for root)
            ├── Section/                (420042 uint16)
            ├── Source Section/         (1314 uint16, all 3)
            └── Destination Section/    (1314 uint16, values 0/1/2)
```

## Key dataset summary

| Dataset | Total elements | dtype | Notes |
|---------|---------------|-------|-------|
| Cell Index | 438 | uint32 | Descending order |
| Y Coordinate | 417,852 | float32 | ~954 pts/cell |
| Radius | 417,852 | float32 | Uniform 5.0 µm |
| Parent Point | 417,852 | int32 | Tree topology |
| Section | 420,042 | uint16 | ~959 sections/cell |
| Source Section | 1,314 | uint16 | 3 synapses/cell |

## Related

- [MiV Microcircuit](concept-miv-microcircuit.md)
- [OLM_tree.h5](source-OLM_tree_h5.md)
- [Morphology SWC Format](concept-morphology-swc.md)
- [Populations](concept-populations.md)
