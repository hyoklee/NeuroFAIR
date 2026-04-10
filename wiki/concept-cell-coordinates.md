# Cell Coordinates and Arc Distances

## Definition

neuroh5 represents neuron positions in two complementary coordinate systems stored under `/Populations/<POP>/Arc Distances/` and `/Populations/<POP>/Generated Coordinates/`. These appear in [Microcircuit_coords.h5](source-Microcircuit_coords_h5.md) for all four MiV populations.

## Arc Distances

Arc distances measure each cell's position along the curved surface of the hippocampal layer using two orthogonal axes:

- **U Distance** — transverse (septo-temporal) axis, range ±2000.01 µm
- **V Distance** — longitudinal axis, range ±2000.01 µm

Each `Arc Distances` HDF5 group carries scalar attributes defining the reference surface bounds:

```
Reference U Min/Max = ±2000.01 µm
Reference V Min/Max = ±2000.01 µm
```

Data layout: one float32 value per cell, stored in the standard neuroh5 `Attribute Pointer` + `Attribute Value` + `Cell Index` triple.

## Generated Coordinates

Six coordinate datasets under `Generated Coordinates/`:

| Dataset | Description |
|---------|-------------|
| L Coordinate | Layer (depth) coordinate |
| U Coordinate | Curvilinear U axis |
| V Coordinate | Curvilinear V axis |
| X Coordinate | Cartesian X (µm) |
| Y Coordinate | Cartesian Y (µm) |
| Z Coordinate | Cartesian Z (µm) |

## Cell counts in Microcircuit_coords.h5

| Population | Cells |
|------------|-------|
| STIM | 1,000 |
| PYR | 80,000 |
| PVBC | 1,474 |
| OLM | 438 |

## Where it appears

- [Microcircuit_coords.h5](source-Microcircuit_coords_h5.md)

## Related

- [MiV Microcircuit](concept-miv-microcircuit.md)
- [Populations](concept-populations.md)
