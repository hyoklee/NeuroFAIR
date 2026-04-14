# PVBC_forest.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/PVBC_forest.h5`
**Size:** 64.9 MB
**Added:** 2026-04-14

## Summary

Full morphology forest for all 1,474 PVBC (parvalbumin-positive basket cell) cells. Each cell has 1,820 morphology points (uniform across the population), giving 2,682,680 total points. Analogous in structure to [OLM_forest.h5](source-OLM_forest_h5.md) but for the larger PVBC population.

## Population Registry (`/H5Types`)

Same 4-population MiV Microcircuit registry (STIM/PYR/PVBC/OLM).

## Morphology Data (`/Populations/PVBC/Trees`)

### Cell Index

Shape (1,474,) uint32 — GID list in descending order (1473 … 0).

### Per-point attributes (all shape 2,682,680 values)

| Dataset | dtype | Notes |
|---|---|---|
| X Coordinate / Attribute Value | float32 | All zeros (placeholder; real coords from Microcircuit_coords.h5) |
| Y Coordinate / Attribute Value | float32 | 0–1819 µm range (linear placeholder) |
| Z Coordinate / Attribute Value | float32 | All zeros |
| Radius / Attribute Value | float32 | Uniform 5.0 µm |
| SWC Type / Attribute Value | int8 | 1 (soma) for all observed values |
| Point Layer / Attribute Value | int8 | 6 (stratum oriens) |
| Parent Point / Attribute Value | int32 | -1 at root, sequential thereafter |
| Section / Attribute Value | uint16 | Section IDs |
| Source Section / Attribute Value | uint16 | Parent section IDs (16 bifurcations/cell) |
| Destination Section / Attribute Value | uint16 | Child section IDs |

### Pointer arrays (all shape 1,475)

- **Attribute Pointer** uint64 — per-cell offset; stride = 1,820 pts/cell
- **Section Pointer** uint64 — per-cell offset into section arrays
- **Cell Index** uint32 — repeats the GID list per attribute

## Key statistics

| Metric | Value |
|---|---|
| Cells | 1,474 |
| Points per cell | 1,820 (uniform) |
| Total morphology points | 2,682,680 |
| Sections per cell | ~16 bifurcations (23,584 / 1,474) |

## Related files

- [PVBC_tree.h5](source-PVBC_tree_h5.md) — single PVBC exemplar (GID 0, 1820 pts)
- [PVBC_forest_syns.h5](source-PVBC_forest_syns_h5.md) — synapse attribute table for PVBC cells
- [PVBC_connections.h5](source-PVBC_connections_h5.md) — PVBC outgoing connectivity
- [concept: Morphology SWC Format](concept-morphology-swc.md)
