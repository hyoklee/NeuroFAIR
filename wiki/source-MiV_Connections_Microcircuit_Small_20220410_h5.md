# MiV_Connections_Microcircuit_Small_20220410.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/Microcircuit_Small/MiV_Connections_Microcircuit_Small_20220410.h5`
**Size:** (in `Microcircuit_Small/` subdirectory)
**Added:** 2026-04-14

## Summary

Small-scale connectivity snapshot for the MiV Microcircuit, dated 2022-04-10. Contains all projections for a downsampled subset of the circuit (44 OLM cells, ~10% of the full circuit). Destination Pointer arrays have shape (45,), matching the 44 OLM cells in the companion [MiV_Cells_Microcircuit_Small_20220410.h5](source-MiV_Cells_Microcircuit_Small_20220410_h5.md). Used for testing and validation of neuroh5 I/O.

## Population Registry (`/H5Types`)

Same 4-population MiV Microcircuit registry (metadata only).

## Projections

### OLM → PVBC

| Dataset | Shape | Notes |
|---|---|---|
| Connections/distance | (36,488,) float32 | |
| Connections/Source Index | (36,488,) uint32 | |
| Edges/Destination Block Index | (2,) uint32 | only 2 blocks (small subset) |
| Edges/Destination Block Pointer | (3,) uint64 | |
| Edges/Destination Pointer | (45,) uint64 | 44 OLM destination cells |
| Synapses/syn_id | (36,488,) uint32 | |

### OLM → PYR

| Dataset | Shape | Notes |
|---|---|---|
| Connections/distance | (124,040,) float32 | |
| Connections/Source Index | (124,040,) uint32 | |
| Edges/Destination Block Index | (2,) uint32 | |
| Edges/Destination Block Pointer | (3,) uint64 | |
| Edges/Destination Pointer | (45,) uint64 | |
| Synapses/syn_id | (124,040,) uint32 | |

Other population projections (PVBC, PYR source) also present but not detailed here.

## Scale comparison with full circuit

| Metric | Small | Full (OLM_connections.h5) |
|---|---|---|
| OLM cells | 44 | 438 |
| OLM→PVBC edges | 36,488 | 360,893 |
| OLM→PYR edges | 124,040 | 1,233,575 |
| Ratio | ~10% | 100% |

## Related files

- [MiV_Cells_Microcircuit_Small_20220410.h5](source-MiV_Cells_Microcircuit_Small_20220410_h5.md) — cell coordinates for same small circuit
- [MiV_Connections_Microcircuit_20220410.h5](source-MiV_Connections_Microcircuit_20220410_h5.md) — full-scale connectivity
- [OLM_connections.h5](source-OLM_connections_h5.md) — full OLM connectivity
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
