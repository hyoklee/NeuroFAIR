# MiV_Cells_Microcircuit_20220410.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/MiV_Cells_Microcircuit_20220410.h5`
**Size:** 15.7 GB
**Added:** 2026-04-17

## Summary

Full-circuit cell attribute snapshot for the MiV Microcircuit dated 2022-04-10. Stores per-cell coordinates, arc distances, synapse attributes, and morphology tree attributes for three neuron populations (OLM, PVBC, PYR). This is the companion to [MiV_Connections_Microcircuit_20220410.h5](source-MiV_Connections_Microcircuit_20220410_h5.md) and represents the earliest available full-circuit snapshot.

## Population Registry (`/H5Types`)

4 populations, 10 valid projections — same standard MiV Microcircuit registry:

| Population | Start GID | Count | ID |
|---|---|---|---|
| STIM | 0 | 1000 | 0 |
| PYR | 1000 | 80000 | 100 |
| PVBC | 81000 | 1474 | 101 |
| OLM | 82474 | 438 | 102 |

## Per-population data (`/Populations`)

Each of OLM, PVBC, and PYR contains:

| Group | Contents |
|---|---|
| Arc Distances / U Distance | Per-cell curvilinear U arc-distance (Attribute Pointer + Value + Cell Index) |
| Arc Distances / V Distance | Per-cell curvilinear V arc-distance |
| Generated Coordinates | L, U, V, X, Y, Z coordinates per cell |
| Synapse Attributes | swc_types, syn_cdists, syn_ids, syn_layers, syn_locs, syn_secs, syn_types |
| Tree Attributes | Morphology skeleton data |

Arc distance bounds for OLM: U/V ±2000.01 µm.

## Related files

- [MiV_Cells_Microcircuit_20220412.h5](source-MiV_Cells_Microcircuit_20220412_h5.md) — later snapshot (2022-04-12, 18.4 GB)
- [MiV_Connections_Microcircuit_20220410.h5](source-MiV_Connections_Microcircuit_20220410_h5.md) — connectivity snapshot same date
- [MiV_Cells_Microcircuit_Small_20220410.h5](source-MiV_Cells_Microcircuit_Small_20220410_h5.md) — 10% subset of this file
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
- [concept: Cell Coordinates and Arc Distances](concept-cell-coordinates.md)
