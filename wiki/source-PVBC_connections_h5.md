# PVBC_connections.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/PVBC_connections.h5`
**Size:** 146 MB
**Added:** 2026-04-14

## Summary

Outgoing connectivity for all 1,474 PVBC cells, covering 4 projections: PVBC→OLM, PVBC→PVBC, PVBC→PYR, and PVBC→STIM. Each projection uses DBS sparse format with `distance` (float32) and `syn_id` (uint32) as edge attributes.

## Population Registry (`/H5Types`)

Same 4-population MiV Microcircuit registry (STIM/PYR/PVBC/OLM, 10 projections).

## Projections (`/Projections/PVBC`)

### PVBC → OLM

| Dataset | Shape | Notes |
|---|---|---|
| Connections/distance | (142,955,) float32 | ~9.4 µm (short-range inhibition) |
| Connections/Source Index | (142,955,) uint32 | |
| Edges/Destination Block Index | (1,474,) uint32 | |
| Edges/Destination Pointer | (1,475,) uint64 | |
| Synapses/syn_id | (142,955,) uint32 | |

**142,955 edges**. Very short arc distances (~9.4 µm) reflect local inhibition of OLM by PVBC.

### PVBC → PVBC

| Dataset | Shape | Notes |
|---|---|---|
| Connections/distance | (3,095,801,) float32 | ~0.0 µm (same-population) |
| Connections/Source Index | (3,095,801,) uint32 | |
| Edges/Destination Block Index | (1,474,) uint32 | |
| Synapses/syn_id | (3,095,801,) uint32 | |

**3,095,801 edges** — dense recurrent PVBC↔PVBC inhibition (~2,100 connections/cell).

### PVBC → PYR

| Dataset | Shape | Notes |
|---|---|---|
| Connections/distance | (4,443,585,) float32 | 90–317 µm |
| Connections/Source Index | (4,443,585,) uint32 | |
| Edges/Destination Block Index | (1,474,) uint32 | |
| Synapses/syn_id | (4,443,585,) uint32 | |

**4,443,585 edges** — PVBC perisomatic inhibition onto PYR (~3,014 targets/cell).

### PVBC → STIM

| Dataset | Shape | Notes |
|---|---|---|
| Connections/distance | (4,445,143,) float32 | ~154 µm |
| Connections/Source Index | (4,445,143,) uint32 | |
| Edges/Destination Block Index | (1,474,) uint32 | |
| Synapses/syn_id | (4,445,143,) uint32 | |

**4,445,143 edges** — feedback inhibition of STIM inputs.

## Summary statistics

| Projection | Edges | Avg edges/PVBC cell |
|---|---|---|
| PVBC→OLM | 142,955 | ~97 |
| PVBC→PVBC | 3,095,801 | ~2,100 |
| PVBC→PYR | 4,443,585 | ~3,014 |
| PVBC→STIM | 4,445,143 | ~3,015 |
| **Total** | **12,127,484** | |

## Related files

- [PVBC_forest.h5](source-PVBC_forest_h5.md) — PVBC morphology
- [PVBC_forest_syns.h5](source-PVBC_forest_syns_h5.md) — PVBC synapse attributes
- [PVBC_tree.h5](source-PVBC_tree_h5.md) — single PVBC exemplar
- [concept: DBS Format](concept-dbs-format.md)
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
