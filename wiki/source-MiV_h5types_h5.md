# Source: MiV_h5types.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/MiV_h5types.h5`
**Dataset:** MiV Microcircuit

## Summary

Minimal type-registry file for the MiV Microcircuit model. Contains only `/H5Types` — no cell data or connectivity. Serves as the canonical population registry shared across all MiV Microcircuit files.

## Top-level structure

```
/
└── H5Types/
    ├── Populations                    ← 4 populations
    └── Valid population projections   ← 10 directed projections
```

## Populations

| Index | Start GID | Count  | Pop ID | Cell Type |
|-------|-----------|--------|--------|-----------|
| 0     | 0         | 1000   | 0      | STIM      |
| 1     | 1000      | 80000  | 100    | PYR       |
| 2     | 81000     | 1474   | 101    | PVBC      |
| 3     | 82474     | 438    | 102    | OLM       |

## Valid population projections (10)

| Source | Destination | Meaning    |
|--------|-------------|------------|
| 0      | 100         | STIM→PYR  |
| 100    | 100         | PYR→PYR   |
| 101    | 100         | PVBC→PYR  |
| 102    | 100         | OLM→PYR   |
| 100    | 101         | PYR→PVBC  |
| 0      | 101         | STIM→PVBC |
| 101    | 101         | PVBC→PVBC |
| 102    | 101         | OLM→PVBC  |
| 100    | 102         | PYR→OLM   |
| 101    | 102         | PVBC→OLM  |

## Related

- [MiV Microcircuit](concept-miv-microcircuit.md)
- [H5Types](concept-h5types.md)
- [Populations](concept-populations.md)
