# Concept: Dentate Gyrus Cell Types

## Overview

`dentate_test.h5` models a microcircuit of the **dentate gyrus** (DG), the input gateway of the hippocampus. The DG contains tightly regulated inhibitory and excitatory populations that gate information flow from the entorhinal cortex.

## Cell types in dentate_test.h5

| Code | Full name | Role |
|------|-----------|------|
| **GC** | Granule Cell | Principal excitatory neuron; main output of the DG |
| **MC** | Mossy Cell | Excitatory hilar interneuron; recurrent excitation of GCs |
| **AAC** | Axo-Axonic Cell | Inhibitory interneuron targeting axon initial segment of GCs |
| **BC** | Basket Cell | Perisomatic inhibitory interneuron; fast feedforward/feedback inhibition |
| **HC** | HIPP Cell (Hilar Commissural-Associational Pathway cell) | Dendrite-targeting inhibitory interneuron |
| **NGFC** | Neurogliaform Cell | Slow, widespread inhibitory interneuron; volume GABA release |

## Population structure

`dentate_test.h5` has **11 populations** — the 6 cell types above likely subdivided by layer or sub-region — registered in `/H5Types/Populations`.

## Projection graph (partial, from file)

Source groups present under `/Projections/`:
- `AAC → {GC, MC, NGFC}`
- `BC → {…}`
- `GC → {AAC, BC, HC, MC, NGFC}`
- `HC → {…}`
- `MC → {…}`
- `NGFC → {…}`

40 valid projection pairs total.

## Related concepts

- [Populations](concept-populations.md)
- [Projections](concept-projections.md)
- [Source: dentate_test.h5](source-dentate_test_h5.md)
