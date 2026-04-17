# MiV_Connections_Microcircuit_20220412.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/MiV_Connections_Microcircuit_20220412.h5`
**Size:** 6.9 GB
**Added:** 2026-04-17

## Summary

Full all-to-all connectivity snapshot for the MiV Microcircuit dated 2022-04-12. Contains all 10 inter-population projections from three source populations (OLM, PVBC, PYR). Structure and edge counts match [MiV_Connections_Microcircuit_20220410.h5](source-MiV_Connections_Microcircuit_20220410_h5.md) exactly, confirming circuit topology is consistent across snapshots.

## Population Registry (`/H5Types`)

4 populations, 10 valid projections — standard MiV Microcircuit registry.

## Projections (`/Projections`)

### OLM source (438 cells)

| Projection | Edges |
|---|---|
| OLM → PVBC | 360,893 |
| OLM → PYR | 1,233,575 |

### PVBC source (1,474 cells)

| Projection | Edges |
|---|---|
| PVBC → OLM | 142,955 |
| PVBC → PVBC | 3,095,801 |
| PVBC → PYR | 4,443,585 |
| PVBC → STIM | 4,445,143 |

### PYR source (80,000 cells)

| Projection | Edges |
|---|---|
| PYR → OLM | see [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md) |
| PYR → PVBC | see [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md) |
| PYR → PYR | see [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md) |
| PYR → STIM | see [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md) |

Each projection stores Edges (Destination Block Index/Pointer, Destination Pointer, Source Index), Connections (distance float32), and Synapses (syn_id uint32) in DBS format.

## Notes

- Previously could not be opened due to file transfer corruption; successfully retrieved via Globus Transfer API on 2026-04-17.
- Edge counts for OLM and PVBC projections are identical to the 20220410 snapshot.

## Related files

- [MiV_Connections_Microcircuit_20220410.h5](source-MiV_Connections_Microcircuit_20220410_h5.md) — earlier snapshot (2022-04-10, 6.4 GB)
- [MiV_Cells_Microcircuit_20220412.h5](source-MiV_Cells_Microcircuit_20220412_h5.md) — cell attributes same date
- [OLM_connections.h5](source-OLM_connections_h5.md) — OLM-only connectivity file
- [PVBC_connections.h5](source-PVBC_connections_h5.md) — PVBC-only connectivity file
- [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md) — PYR-only connectivity file
- [concept: DBS Format](concept-dbs-format.md)
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
