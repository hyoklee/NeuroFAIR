# Morphology SWC Format

## Definition

neuroh5 stores neuronal morphologies in a neuroh5-adapted SWC (Standardized Morphology Data Format) layout under `/Populations/<POP>/Trees/`. Each cell's morphology is a tree of 3D points with typed compartments, stored as per-cell attribute arrays indexed by `Cell Index` and `Attribute Pointer`.

## SWC datasets in Trees groups

| Dataset | dtype | Description |
|---------|-------|-------------|
| Cell Index | uint32 | GID of each cell; one entry per cell |
| X/Y/Z Coordinate | float32 | Cartesian position of each morphology point (µm) |
| Radius | float32 | Compartment radius at each point (µm) |
| SWC Type | int8 | Standard SWC compartment type (1=soma, 3=dendrite, etc.) |
| Point Layer | int8 | Anatomical layer index for each point |
| Parent Point | int32 | Index of parent point in tree; −1 for root |
| Section | uint16 | Section index assigned to each point |
| Section Pointer | uint64 | Offset into Section array per cell |
| Source Section | uint16 | Section indices where axon outputs originate |
| Destination Section | uint16 | Section indices where synaptic inputs arrive |
| Attribute Pointer | uint64 | Start offset in value array for each cell (N_cells+1 entries) |
| Attribute Value | varies | Concatenated per-point values across all cells |

## Observed cell-type morphology sizes (single-cell exemplars)

| Cell type | Points | Sections | Synapse records | Point Layer | Radius |
|-----------|--------|----------|-----------------|-------------|--------|
| PYR | 10,346 | 10,549 | 201 | 6 | 1.7–3.7+ µm (variable) |
| PVBC | 1,820 | 1,838 | 16 | 6 | 5.0 µm uniform |
| OLM | 954 | 959 | 3 | 5 | 5.0 µm uniform |

## Population vs single-cell files

- **`*_tree.h5`** — single exemplar cell (1 cell), used for inspection/testing.
- **`*_forest.h5`** — all cells of the population. OLM_forest has 438 cells with ~954 pts each. PYR_forest is a stub (cell registry present but coordinate data empty).

## Where it appears

- [OLM_forest.h5](source-OLM_forest_h5.md)
- [OLM_tree.h5](source-OLM_tree_h5.md)
- [PVBC_tree.h5](source-PVBC_tree_h5.md)
- [PYR_forest.h5](source-PYR_forest_h5.md)
- [PYR_tree.h5](source-PYR_tree_h5.md)

## Related

- [MiV Microcircuit](concept-miv-microcircuit.md)
- [Populations](concept-populations.md)
