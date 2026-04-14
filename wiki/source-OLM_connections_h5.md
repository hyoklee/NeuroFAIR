# OLM_connections.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/OLM_connections.h5`
**Size:** 19.5 MB
**Added:** 2026-04-14

## Summary

OLM cell outgoing connectivity file. Contains two projections from OLM (pop 102, 438 cells) to its two target populations: PVBC (101) and PYR (100). Each projection is stored in DBS format with synapse IDs as the edge attribute.

## Population Registry (`/H5Types`)

Matches the shared MiV Microcircuit registry (4 populations):

| Population | Start GID | Count | ID |
|---|---|---|---|
| STIM | 0 | 1000 | 0 |
| PYR | 1000 | 80000 | 100 |
| PVBC | 81000 | 1474 | 101 |
| OLM | 82474 | 438 | 102 |

10 valid projections registered (same as `MiV_h5types.h5`).

## Projections

### OLM → PVBC (`/Projections/OLM/PVBC`)

| Dataset | Shape | Notes |
|---|---|---|
| Edges/Destination Block Index | (438,) uint32 | DBS block index per OLM destination |
| Edges/Destination Block Pointer | (439,) uint64 | block pointer array |
| Edges/Destination Pointer | (439,) uint64 | cumulative edge offsets |
| Connections/Source Index | (360893,) uint32 | source GIDs |
| Connections/distance | (360893,) float32 | arc distance per edge |
| Synapses/syn_id | (360893,) uint32 | target synapse IDs on PVBC cells |

**360,893 edges** from 438 OLM cells onto PVBC cells. Representative distances ~315 µm.

### OLM → PYR (`/Projections/OLM/PYR`)

| Dataset | Shape | Notes |
|---|---|---|
| Edges/Destination Block Index | (438,) uint32 | |
| Edges/Destination Pointer | (439,) uint64 | |
| Connections/Source Index | (1233575,) uint32 | |
| Connections/distance | (1233575,) float32 | distances vary (24–367 µm) |
| Synapses/syn_id | (1233575,) uint32 | |

**1,233,575 edges** from OLM onto PYR cells. Wider distance spread reflecting OLM axon range into stratum radiatum/lacunosum.

## Related files

- [OLM_forest.h5](source-OLM_forest_h5.md) — OLM morphology forest (438 cells)
- [OLM_forest_syns.h5](source-OLM_forest_syns_h5.md) — synapse attribute table for OLM cells
- [MiV_h5types.h5](source-MiV_h5types_h5.md) — population/projection registry
- [concept: DBS Format](concept-dbs-format.md)
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
