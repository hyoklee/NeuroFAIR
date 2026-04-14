# PYR_connections_compressed.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/PYR_connections_compressed.h5`
**Size:** 6.2 GB (gzip-compressed)
**Added:** 2026-04-14

## Summary

Full outgoing connectivity for all 80,000 PYR (pyramidal) cells, covering 4 projections: PYR→OLM, PYR→PVBC, PYR→PYR, and PYR→STIM. All datasets are gzip-compressed. This is the compressed counterpart of uncompressed connectivity data; it represents the dominant edge volume in the MiV Microcircuit due to the large PYR population.

## Population Registry (`/H5Types`)

Same 4-population MiV Microcircuit registry (STIM/PYR/PVBC/OLM, 10 projections). Metadata only (no data preview in `--no-data` mode).

## Projections (`/Projections/PYR`)

All Destination Block Index arrays have shape (80,000,) — one entry per PYR source cell. All pointer arrays shape (80,001,).

### PYR → OLM

| Dataset | Shape | Compression |
|---|---|---|
| Connections/distance | (70,230,381,) float32 | gzip |
| Connections/Source Index | (70,230,381,) uint32 | gzip |
| Edges/Destination Block Index | (80,000,) uint32 | gzip |
| Edges/Destination Pointer | (80,001,) uint64 | gzip |
| Synapses/syn_id | (70,230,381,) uint32 | gzip |

**70.2 M edges** from PYR→OLM (~878 targets/cell).

### PYR → PVBC

| Dataset | Shape | Notes |
|---|---|---|
| Connections/distance | (53,222,849,) float32 | gzip |
| Synapses/syn_id | (53,222,849,) uint32 | gzip |

**53.2 M edges** (~665 targets/cell).

### PYR → PYR

| Dataset | Shape | Notes |
|---|---|---|
| Connections/distance | (1,278,524,250,) float32 | gzip |
| Synapses/syn_id | (1,278,524,250,) uint32 | gzip |

**1.28 billion edges** — massive recurrent PYR↔PYR excitation (~15,981 connections/cell). This is the largest projection by edge count in the MiV Microcircuit.

### PYR → STIM

| Dataset | Shape | Notes |
|---|---|---|
| Connections/distance | (1,278,483,520,) float32 | gzip |
| Synapses/syn_id | (1,278,483,520,) uint32 | gzip |

**1.28 billion edges** (~15,981 targets/cell).

## Summary statistics

| Projection | Edges | Avg/PYR cell |
|---|---|---|
| PYR→OLM | 70,230,381 | ~878 |
| PYR→PVBC | 53,222,849 | ~665 |
| PYR→PYR | 1,278,524,250 | ~15,981 |
| PYR→STIM | 1,278,483,520 | ~15,981 |
| **Total** | **~2,680,461,000** | |

## Notes

- All datasets use gzip compression — random access is significantly slower than uncompressed files.
- File does not contain cell morphology; see [PYR_forest_compressed.h5](source-PYR_forest_compressed_h5.md).
- `PYR_forest_syns_compressed.h5` (12 GB) was inaccessible (truncated) at time of ingestion.

## Related files

- [PYR_forest.h5](source-PYR_forest_h5.md) — PYR forest stub
- [PYR_forest_compressed.h5](source-PYR_forest_compressed_h5.md) — full compressed PYR morphology
- [PYR_tree.h5](source-PYR_tree_h5.md) — single PYR cell exemplar
- [concept: DBS Format](concept-dbs-format.md)
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
