# PYR_forest_syns_compressed.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/PYR_forest_syns_compressed.h5`
**Size:** 11.2 GB
**Added:** 2026-04-17

## Summary

Compressed synapse attribute table for all 80,000 PYR (pyramidal) cells in the MiV Microcircuit. Stores 7 per-synapse attributes across 2,680,461,000 synapses (~2.68 billion total), gzip-compressed. This is the PYR equivalent of [PVBC_forest_syns.h5](source-PVBC_forest_syns_h5.md) and [OLM_forest_syns.h5](source-OLM_forest_syns_h5.md), but orders of magnitude larger due to PYR cell count.

## Population Registry (`/H5Types`)

4 populations, 10 valid projections — standard MiV Microcircuit registry (metadata only).

## Synapse Attributes (`/Populations/PYR/Synapse Attributes`)

All datasets use gzip compression with chunk size 4000.

| Attribute | Pointer shape | Value shape | dtype | Description |
|---|---|---|---|---|
| swc_types | (80001,) uint64 | (2,680,461,000,) uint8 | uint8 | SWC compartment type per synapse |
| syn_cdists | (80001,) uint64 | (2,680,461,000,) float32 | float32 | Cable distance along dendrite to synapse |
| syn_ids | (80001,) uint64 | (2,680,461,000,) uint32 | uint32 | Synapse IDs |
| syn_layers | (80001,) uint64 | (2,680,461,000,) uint8 | uint8 | Cortical/hippocampal layer assignment |
| syn_locs | (80001,) uint64 | (2,680,461,000,) float32 | float32 | Local section location (0–1) |
| syn_secs | (80001,) uint64 | (2,680,461,000,) uint32 | uint32 | Section index within cell morphology |
| syn_types | (80001,) uint64 | (2,680,461,000,) uint8 | uint8 | Synapse type (e.g. excitatory/inhibitory) |

Pointer arrays have shape (80001,) — one entry per PYR cell plus a terminal sentinel, providing O(1) lookup of each cell's synapse block.

## Notes

- ~33,506 synapses per PYR cell on average (2.68B / 80,000).
- Compression is essential given raw size would be ~40+ GB uncompressed.
- Previously could not be opened due to file transfer corruption; successfully retrieved via Globus Transfer API on 2026-04-17.

## Related files

- [PYR_forest_compressed.h5](source-PYR_forest_compressed_h5.md) — PYR morphology forest (80k cells)
- [PYR_connections_compressed.h5](source-PYR_connections_compressed_h5.md) — PYR connectivity (2.68B edges)
- [PVBC_forest_syns.h5](source-PVBC_forest_syns_h5.md) — analogous PVBC synapse table (~12.1M synapses)
- [OLM_forest_syns.h5](source-OLM_forest_syns_h5.md) — analogous OLM synapse table
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
- [concept: Morphology SWC Format](concept-morphology-swc.md)
