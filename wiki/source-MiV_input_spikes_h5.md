# Source: MiV_input_spikes.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/MiV_input_spikes.h5`
**Dataset:** MiV Microcircuit — STIM population spike trains and trajectory

## Summary

Contains spike trains for the STIM population (1000 cells) across multiple trials, plus a spatial trajectory used to drive stimulus generation. The spike data (`Input Spikes A Diag`) has 586,979 total spikes across 1000 cells (~587 spikes/cell on average) with timestamps ranging from −222 ms to ~9428 ms. Trial duration is uniformly 9678 ms with 3 trials per cell (3000 total trial records). A `Trajectory A Diag` group stores the animal's spatial path (x, y, d, t) at 9679 timesteps over ~9.68 s.

## Top-level structure

```
/
├── H5Types/
│   ├── Populations                    ← 4 populations
│   └── Valid population projections   ← 10 projections
├── Populations/
│   └── STIM/
│       └── Input Spikes A Diag/
│           ├── Spike Train/
│           │   ├── Attribute Pointer   (1001 uint64)
│           │   ├── Attribute Value     (586979 float32, ms)
│           │   └── Cell Index          (1000 uint32)
│           ├── Trial Duration/
│           │   ├── Attribute Pointer   (1001 uint64)
│           │   ├── Attribute Value     (3000 float32, all 9678.0 ms)
│           │   └── Cell Index          (1000 uint32)
│           └── Trial Index/
│               ├── Attribute Pointer   (1001 uint64)
│               ├── Attribute Value     (586979 uint8)
│               └── Cell Index          (1000 uint32)
└── Trajectory A Diag/
    ├── t    (9679 float32, ms, −250 to ~9428)
    ├── x    (9679 float32, spatial coordinate)
    ├── y    (9679 float32, spatial coordinate)
    └── d    (9679 float32, arc distance, −7.5 to ~22.0)
```

## Key statistics

| Dataset | Shape | dtype | Notes |
|---------|-------|-------|-------|
| Spike Train / Attribute Value | (586979,) | float32 | ~587 spikes/cell average |
| Trial Duration / Attribute Value | (3000,) | float32 | 9678.0 ms uniform |
| Trial Index / Attribute Value | (586979,) | uint8 | Trial ID per spike |
| Trajectory t | (9679,) | float32 | −250 to ~9428 ms |
| Trajectory d | (9679,) | float32 | Arc distance along path |

## Related

- [MiV Microcircuit](concept-miv-microcircuit.md)
- [MiV_input_features.h5](source-MiV_input_features_h5.md)
- [Populations](concept-populations.md)
