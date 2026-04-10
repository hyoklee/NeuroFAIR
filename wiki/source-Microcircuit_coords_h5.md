# Source: Microcircuit_coords.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/Microcircuit_coords.h5`
**Dataset:** MiV Microcircuit — cell spatial coordinates

## Summary

Stores spatial coordinates for all four MiV Microcircuit populations (STIM, PYR, PVBC, OLM). Each population has two coordinate groups: `Arc Distances` (U and V distances along the curved surface of the hippocampal layer) and `Generated Coordinates` (Cartesian X/Y/Z plus L, U, V curvilinear coordinates). The reference surface spans U ∈ [−2000.01, 2000.01] and V ∈ [−2000.01, 2000.01] µm. Arc distance values are float32 and represent each cell's position along the transverse (U) and longitudinal (V) hippocampal axes.

## Top-level structure

```
/
├── H5Types/
│   ├── Populations                    ← 4 populations
│   └── Valid population projections   ← 10 projections
└── Populations/
    ├── OLM/
    │   ├── Arc Distances/             (U/V range: ±2000.01, attrs on group)
    │   │   ├── U Distance/  (438 float32 values, ~−1991 to +1991 µm)
    │   │   └── V Distance/  (438 float32 values)
    │   └── Generated Coordinates/
    │       ├── L Coordinate/
    │       ├── U Coordinate/
    │       ├── V Coordinate/
    │       ├── X Coordinate/
    │       ├── Y Coordinate/
    │       └── Z Coordinate/
    ├── PVBC/   (same layout, 1474 cells)
    ├── PYR/    (same layout, 80000 cells)
    └── STIM/   (same layout, 1000 cells)
```

## Arc Distances attributes

Each `Arc Distances` group carries HDF5 attributes:

| Attribute | Value |
|-----------|-------|
| Reference U Max | 2000.01 µm |
| Reference U Min | −2000.01 µm |
| Reference V Max | 2000.01 µm |
| Reference V Min | −2000.01 µm |

## Cell counts per population

| Population | Count |
|------------|-------|
| STIM       | 1000  |
| PYR        | 80000 |
| PVBC       | 1474  |
| OLM        | 438   |

## Related

- [MiV Microcircuit](concept-miv-microcircuit.md)
- [Cell Coordinates and Arc Distances](concept-cell-coordinates.md)
- [Populations](concept-populations.md)
