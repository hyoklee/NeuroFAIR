# MiV_Cells_Microcircuit_Small_20220410.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/Microcircuit_Small/MiV_Cells_Microcircuit_Small_20220410.h5`
**Size:** (in `Microcircuit_Small/` subdirectory)
**Added:** 2026-04-14 (directory pre-existed from Apr 9)

## Summary

Small-scale cell coordinate file for the MiV Microcircuit, dated 2022-04-10. Contains arc distances and generated (L/U/V) coordinates for a subset of cells from all four populations (STIM, PYR, PVBC, OLM). The "Small" suffix indicates a downsampled circuit used for testing and validation. Structure is identical to [Microcircuit_coords.h5](source-Microcircuit_coords_h5.md) but with far fewer cells.

## Population Registry (`/H5Types`)

Same 4-population MiV Microcircuit registry (metadata only).

## Cell Data (`/Populations`)

All four populations (OLM, PVBC, PYR, STIM) have:

- **Arc Distances** group with `U Distance` and `V Distance` attributes
  - Reference bounds: U ∈ [-500.01, 500.01], V ∈ [-500.01, 500.01]
- **Generated Coordinates** group with `L Coordinate`, `U Coordinate`, `V Coordinate` attributes

### OLM cell count (observed)

- `Arc Distances/U Distance/Attribute Value` shape **(44,)** — 44 OLM cells in this small subset
- `Attribute Pointer` shape (45,) → 44 entries

OLM in the full circuit has 438 cells; this small file has 44 (10%).

## Notes

- Companion file: [MiV_Connections_Microcircuit_Small_20220410.h5](source-MiV_Connections_Microcircuit_Small_20220410_h5.md) — connectivity for this small circuit.
- Full-scale equivalent: [Microcircuit_coords.h5](source-Microcircuit_coords_h5.md).
- `MiV_Cells_Microcircuit_Small_20220412.h5` does not exist in this directory (only the 20220410 date).

## Related files

- [Microcircuit_coords.h5](source-Microcircuit_coords_h5.md) — full-scale coordinates
- [MiV_Connections_Microcircuit_Small_20220410.h5](source-MiV_Connections_Microcircuit_Small_20220410_h5.md)
- [concept: Cell Coordinates and Arc Distances](concept-cell-coordinates.md)
