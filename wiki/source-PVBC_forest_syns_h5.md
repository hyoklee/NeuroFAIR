# PVBC_forest_syns.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/PVBC_forest_syns.h5`
**Size:** 231.6 MB
**Added:** 2026-04-14

## Summary

Per-synapse attribute table for all 1,474 PVBC cells. Mirrors the structure of [OLM_forest_syns.h5](source-OLM_forest_syns_h5.md) but for the larger PVBC population. Six synapse attributes are stored under `/Populations/PVBC/Synapse Attributes` using the standard neuroh5 ragged-array layout (Cell Index + Attribute Pointer + Attribute Value).

## Population Registry (`/H5Types`)

Same 4-population MiV Microcircuit registry.

## Synapse Attributes (`/Populations/PVBC/Synapse Attributes`)

Each attribute pointer array has shape (1,475,). The step between entries is ~8,217 synapses/cell (e.g. Pointer[0]=0, Pointer[1]=8,217), indicating roughly **8,217 synapses per PVBC cell**.

| Attribute | dtype | Description |
|---|---|---|
| `swc_types` | uint8 | SWC compartment type of postsynaptic site |
| `syn_cdists` | float32 | Cable distance from soma (µm) |
| `syn_ids` | uint32 | Synapse identifier |
| `syn_layers` | int8 | Hippocampal layer index |
| `syn_locs` | float32 | Normalized position along section (0–1) |
| `syn_secs` | uint32 | Section (compartment) index |
| `syn_types` | uint8 | Synapse type code |

### Estimated totals

- ~8,217 synapses/cell × 1,474 cells ≈ **12,111,858 total synaptic entries**

## Related files

- [PVBC_forest.h5](source-PVBC_forest_h5.md) — PVBC morphology forest
- [PVBC_connections.h5](source-PVBC_connections_h5.md) — PVBC outgoing connectivity
- [PVBC_tree.h5](source-PVBC_tree_h5.md) — single PVBC cell exemplar
- [OLM_forest_syns.h5](source-OLM_forest_syns_h5.md) — analogous file for OLM cells
