# Concept: Destination Block Sparse (DBS) Format

## Definition

The Destination Block Sparse (DBS) format is neuroh5's core sparse adjacency representation for synaptic connectivity. It is optimised for the common case where each destination neuron connects to a small, locally clustered subset of source neurons.

## Four arrays

| Dataset | Dtype | Role |
|---------|-------|------|
| `Source Index` | uint32 | Flat list of all source GIDs (length = total edges) |
| `Destination Block Pointer` | uint32 | Offsets into `Destination Pointer`; length = num\_blocks + 1 |
| `Destination Block Index` | uint32 | First destination GID in each block; length = num\_blocks |
| `Destination Pointer` | uint32 | Offsets into `Source Index`; length = total destinations + 1 |

## Edge lookup algorithm

To find all sources connected to destination GID `d`:
1. Locate the block `i` containing `d` using `Destination Block Index`.
2. Compute destination's offset within the block: `j = d - dst_blk_idx[i]`.
3. Flat pointer offset: `ptr = dst_blk_ptr[i] + j`.
4. Source slice: `Source Index[ dst_ptr[ptr] : dst_ptr[ptr+1] ]`.

## Where it appears

- `example.h5` — `/Projections/GCtoGC/Connectivity/` (legacy path)
- `dentate_test.h5` — `/Projections/<SRC>/<DST>/Edges/` (current path)

## Storage

All DBS datasets use **gzip compression (level 6)** and chunked storage. Chunk sizes vary: `example.h5` uses 1024; `dentate_test.h5` uses per-projection natural chunks (e.g. 46 for a 46-edge projection).

## Related concepts

- [Projections](concept-projections.md)
- [Populations](concept-populations.md)
