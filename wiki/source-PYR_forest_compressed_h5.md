# PYR_forest_compressed.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/PYR_forest_compressed.h5`
**Size:** 4.4 GB (gzip-compressed)
**Added:** 2026-04-14

## Summary

Full morphology forest for all 80,000 PYR (pyramidal) cells, gzip-compressed. Complements [PYR_forest.h5](source-PYR_forest_h5.md) (which contains only a stub with no coordinate data) by providing the complete morphology tree data for the whole PYR population. All datasets use gzip compression.

## Population Registry (`/H5Types`)

Same 4-population MiV Microcircuit registry (metadata only, `--no-data` mode).

## Morphology Data (`/Populations/PYR/Trees`)

### Cell Index

Shape (80,000,) uint32 — gzip-compressed GID list.

### Per-point attribute arrays (all gzip-compressed)

| Dataset | Pointer shape | Value shape | dtype | Notes |
|---|---|---|---|---|
| X Coordinate | (80,001,) | large | float32 | 3D coordinates |
| Y Coordinate | (80,001,) | large | float32 | |
| Z Coordinate | (80,001,) | large | float32 | |
| Radius | (80,001,) | large | float32 | Section radii |
| SWC Type | (80,001,) | large | int8 | Compartment type codes |
| Point Layer | (80,001,) | large | int8 | Hippocampal layer |
| Parent Point | (80,001,) | large | int32 | Tree topology |
| Section | (80,001,) | large | uint16 | Section IDs |
| Source Section | (80,001,) | large | uint16 | Parent section |
| Destination Section | (80,001,) | large | uint16 | Child section |

Each pointer array has 80,001 entries (one per cell plus sentinel). Per-cell morphology point count can be derived from the Attribute Pointer strides; see [PYR_tree.h5](source-PYR_tree_h5.md) for the single-cell exemplar (GID 0: 10,346 points).

## Notes

- At 4.4 GB compressed, this file is ~55× larger than the 80 KB [PYR_forest.h5](source-PYR_forest_h5.md) stub, which has no actual coordinate values.
- gzip compression on chunked datasets (chunk size 1,000–4,000) allows reasonable sequential read but poor random access.
- `PYR_forest_syns_compressed.h5` (12 GB) remained inaccessible (truncated file) at time of ingestion.

## Related files

- [PYR_forest.h5](source-PYR_forest_h5.md) — uncompressed stub (no coordinate data)
- [PYR_tree.h5](source-PYR_tree_h5.md) — single PYR cell exemplar (GID 0, 10,346 pts)
- [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md) — full PYR connectivity
- [concept: Morphology SWC Format](concept-morphology-swc.md)
