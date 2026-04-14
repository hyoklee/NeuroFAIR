# OLM_forest_syns.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/OLM_forest_syns.h5`
**Size:** 30.8 MB
**Added:** 2026-04-14

## Summary

Per-synapse attribute table for OLM cells. Stores six synapse attributes for each of the 438 OLM cells under `/Populations/OLM/Synapse Attributes`. Each attribute uses a (Cell Index, Attribute Pointer, Attribute Value) triplet — the standard neuroh5 ragged-array layout.

## Population Registry (`/H5Types`)

Same 4-population MiV Microcircuit registry as other files in this set.

## Synapse Attributes (`/Populations/OLM/Synapse Attributes`)

All 6 attributes span **1,594,468 synaptic entries** across 438 OLM cells (~3,640 synapses/cell on average).

| Attribute | Value dtype | Description |
|---|---|---|
| `syn_ids` | uint32 | Synapse identifier (sequential 0…N) |
| `syn_types` | uint8 | Synapse type code (0 = inhibitory in observed data) |
| `swc_types` | uint8 | SWC compartment type of the postsynaptic site (3 = basal dendrite) |
| `syn_layers` | int8 | Cortical/hippocampal layer index (5 = stratum lacunosum-moleculare) |
| `syn_locs` | float32 | Location along section (0–1 normalized) |
| `syn_secs` | uint32 | Section (compartment) index on the target cell |
| `syn_cdists` | float32 | Cable distance from soma (µm; range ~20–22 µm in first cells) |

### Indexing structure (per attribute)

- **Cell Index** shape (438,) — maps ragged blocks back to GIDs
- **Attribute Pointer** shape (439,) uint64 — start offset per cell (e.g. cell 0: 0, cell 1: 3681 → 3,681 synapses)
- **Attribute Value** shape (1,594,468,) — flat array of attribute values

## Related files

- [OLM_forest.h5](source-OLM_forest_h5.md) — OLM morphology
- [OLM_connections.h5](source-OLM_connections_h5.md) — OLM outgoing connections
- [OLM_tree.h5](source-OLM_tree_h5.md) — single OLM cell exemplar
- [concept: Morphology SWC Format](concept-morphology-swc.md)
- [concept: MiV Microcircuit](concept-miv-microcircuit.md)
