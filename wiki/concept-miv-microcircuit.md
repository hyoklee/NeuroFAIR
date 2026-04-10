# MiV Microcircuit

## Definition

The MiV (Mind-in-Vitro) Microcircuit is a large-scale hippocampal CA1 network model comprising four neural populations and ten directed projections. It is the primary circuit represented in the neuroh5 HDF5 files under `/lus/flare/projects/gpu_hack/iowarp/neuroh5/`.

## Populations

| Pop ID | Name | Count  | GID Range       | Role |
|--------|------|--------|-----------------|------|
| 0      | STIM | 1,000  | 0–999           | Stimulus (place-cell input) |
| 100    | PYR  | 80,000 | 1,000–80,999    | Pyramidal (excitatory) |
| 101    | PVBC | 1,474  | 81,000–82,473   | PV basket cell (fast inhibitory) |
| 102    | OLM  | 438    | 82,474–82,911   | OLM interneuron (slow inhibitory) |

Total: 82,912 neurons.

## Projections (10)

| Source | Destination | Biological meaning |
|--------|-------------|-------------------|
| STIM   | PYR         | Entorhinal drive to pyramidal cells |
| PYR    | PYR         | Recurrent excitation |
| PVBC   | PYR         | Perisomatic inhibition |
| OLM    | PYR         | Distal dendritic inhibition |
| PYR    | PVBC        | Excitatory drive to basket cells |
| STIM   | PVBC        | Direct stimulus to basket cells |
| PVBC   | PVBC        | Basket-cell autoinhibition |
| OLM    | PVBC        | OLM→basket modulation |
| PYR    | OLM         | Excitatory drive to OLM |
| PVBC   | OLM         | Cross-inhibition |

## File inventory

| File | Status | Contents |
|------|--------|----------|
| [MiV_h5types.h5](source-MiV_h5types_h5.md) | OK | Population/projection registry only |
| [Microcircuit_coords.h5](source-Microcircuit_coords_h5.md) | OK | Arc distances + generated coords for all 4 pops |
| [MiV_input_features.h5](source-MiV_input_features_h5.md) | OK | STIM peak rate (20 Hz) + selectivity type |
| [MiV_input_spikes.h5](source-MiV_input_spikes_h5.md) | OK | STIM spike trains + trajectory |
| [OLM_forest.h5](source-OLM_forest_h5.md) | OK | OLM morphology forest (438 cells) |
| [OLM_tree.h5](source-OLM_tree_h5.md) | OK | OLM single-cell exemplar |
| [PVBC_tree.h5](source-PVBC_tree_h5.md) | OK | PVBC single-cell exemplar |
| [PYR_forest.h5](source-PYR_forest_h5.md) | OK (stub) | PYR cell registry, no coords yet |
| [PYR_tree.h5](source-PYR_tree_h5.md) | OK | PYR single-cell exemplar |
| MiV_Cells_Microcircuit_20220410/12.h5 | Truncated | Full cell attribute files |
| MiV_Connections_Microcircuit_20220410/12.h5 | Truncated | Full connectivity files |
| PVBC_forest.h5 | Truncated | PVBC morphology forest |
| PVBC_forest_syns.h5 | Truncated | PVBC synapse data |
| OLM_forest_syns.h5 | Truncated | OLM synapse data |
| PYR_forest_compressed.h5 | Truncated | PYR compressed forest |
| PYR_forest_syns_compressed.h5 | Truncated | PYR compressed synapse data |
| PYR_connections_compressed.h5 | Truncated | PYR compressed connections |
| OLM_connections.h5 | Truncated | OLM connections |

## Related concepts

- [H5Types](concept-h5types.md)
- [Populations](concept-populations.md)
- [Projections](concept-projections.md)
- [Morphology SWC Format](concept-morphology-swc.md)
- [Cell Coordinates and Arc Distances](concept-cell-coordinates.md)
