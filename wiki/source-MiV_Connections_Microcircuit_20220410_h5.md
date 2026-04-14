# MiV_Connections_Microcircuit_20220410.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/MiV_Connections_Microcircuit_20220410.h5`
**Size:** 6.4 GB
**Added:** 2026-04-14

## Summary

Full all-to-all connectivity snapshot for the MiV Microcircuit dated 2022-04-10. Contains all inter-population projections from three source populations: OLM, PVBC, and PYR (STIM is a drive population, not a source of projections here). Edge counts match exactly the population-specific connection files (`OLM_connections.h5`, `PVBC_connections.h5`), confirming these are consistent snapshots of the same circuit state.

## Population Registry (`/H5Types`)

Same 4-population MiV Microcircuit registry. Metadata only (no data preview in `--no-data` mode).

## Projections

Edge counts (derived from dataset shapes, `--no-data` mode):

### OLM source

| Projection | Edges |
|---|---|
| OLM → PVBC | 360,893 |
| OLM → PYR | 1,233,575 |

### PVBC source

| Projection | Edges |
|---|---|
| PVBC → OLM | 142,955 |
| PVBC → PVBC | 3,095,801 |
| PVBC → PYR | 4,443,585 |
| PVBC → STIM | 4,445,143 |

All PVBC projections use the same Destination Block Index shape (1,474,) confirming 1,474 PVBC source cells.

### PYR source

Edge counts not confirmed from `--no-data` output (shapes not printed). See [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md) for PYR-specific counts (~2.68 billion total PYR edges).

## Notes

- `MiV_Connections_Microcircuit_20220412.h5` (7.4 GB) could not be opened — file signature error at time of ingestion; likely still being written or corrupted.
- This file consolidates what the per-population `*_connections.h5` files contain individually, useful for batch I/O across all projections.

## Related files

- [OLM_connections.h5](source-OLM_connections_h5.md)
- [PVBC_connections.h5](source-PVBC_connections_h5.md)
- [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md)
- [Microcircuit_Small/MiV_Connections_Microcircuit_Small_20220410.h5](source-MiV_Connections_Microcircuit_Small_20220410_h5.md)
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
