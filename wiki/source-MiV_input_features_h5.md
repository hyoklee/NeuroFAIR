# Source: MiV_input_features.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/MiV_input_features.h5`
**Dataset:** MiV Microcircuit — STIM population input features

## Summary

Stores place-field selectivity attributes for the STIM population (1000 cells). Two feature types are recorded: `Peak Rate` (float32, Hz) and `Selectivity Type` (uint8). All 1000 STIM cells have a constant peak rate of 20.0 Hz and selectivity type 1. Uses the standard neuroh5 cell-attribute layout: `Attribute Pointer` + `Attribute Value` + `Cell Index`.

## Top-level structure

```
/
├── H5Types/
│   ├── Populations                    ← 4 populations (shared registry)
│   └── Valid population projections   ← 10 projections
└── Populations/
    └── STIM/
        └── Constant Selectivity A/
            ├── Peak Rate/
            │   ├── Attribute Pointer   (1001 uint64)
            │   ├── Attribute Value     (1000 float32, all 20.0 Hz)
            │   └── Cell Index          (1000 uint32)
            └── Selectivity Type/
                ├── Attribute Pointer   (1001 uint64)
                ├── Attribute Value     (1000 uint8, all 1)
                └── Cell Index          (1000 uint32)
```

## Key datasets

| Dataset | Shape | dtype | Notes |
|---------|-------|-------|-------|
| Peak Rate / Attribute Value | (1000,) | float32 | Uniform 20.0 Hz |
| Selectivity Type / Attribute Value | (1000,) | uint8 | All = 1 (constant) |
| Cell Index | (1000,) | uint32 | Sparse GID mapping |

## Related

- [MiV Microcircuit](concept-miv-microcircuit.md)
- [MiV_input_spikes.h5](source-MiV_input_spikes_h5.md)
- [Populations](concept-populations.md)
