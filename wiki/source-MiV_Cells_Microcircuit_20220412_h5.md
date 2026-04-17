# MiV_Cells_Microcircuit_20220412.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/MiV_Cells_Microcircuit_20220412.h5`
**Size:** 18.4 GB
**Added:** 2026-04-17

## Summary

Full-circuit cell attribute snapshot for the MiV Microcircuit dated 2022-04-12. Identical structure to [MiV_Cells_Microcircuit_20220410.h5](source-MiV_Cells_Microcircuit_20220410_h5.md) but ~2.7 GB larger, suggesting updated or expanded synapse attribute data. Companion to [MiV_Connections_Microcircuit_20220412.h5](source-MiV_Connections_Microcircuit_20220412_h5.md).

## Population Registry (`/H5Types`)

4 populations, 10 valid projections — standard MiV Microcircuit registry:

| Population | Start GID | Count | ID |
|---|---|---|---|
| STIM | 0 | 1000 | 0 |
| PYR | 1000 | 80000 | 100 |
| PVBC | 81000 | 1474 | 101 |
| OLM | 82474 | 438 | 102 |

## Per-population data (`/Populations`)

Same layout as the 20220410 snapshot for OLM, PVBC, and PYR:

| Group | Contents |
|---|---|
| Arc Distances / U Distance | Per-cell curvilinear U arc-distance |
| Arc Distances / V Distance | Per-cell curvilinear V arc-distance |
| Generated Coordinates | L, U, V, X, Y, Z coordinates per cell |
| Synapse Attributes | swc_types, syn_cdists, syn_ids, syn_layers, syn_locs, syn_secs, syn_types |
| Tree Attributes | Morphology skeleton data |

Arc distance bounds for OLM: U/V ±2000.01 µm.

## Notes

- Size difference vs 20220410 (+2.7 GB) likely reflects updated synapse attribute values or additional morphology data.
- Previously could not be opened due to file transfer corruption; successfully retrieved via Globus Transfer API on 2026-04-17.

## Related files

- [MiV_Cells_Microcircuit_20220410.h5](source-MiV_Cells_Microcircuit_20220410_h5.md) — earlier snapshot (2022-04-10, 15.7 GB)
- [MiV_Connections_Microcircuit_20220412.h5](source-MiV_Connections_Microcircuit_20220412_h5.md) — connectivity snapshot same date
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
- [concept: Cell Coordinates and Arc Distances](concept-cell-coordinates.md)
