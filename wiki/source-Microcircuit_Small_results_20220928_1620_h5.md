# Microcircuit_Small_results_20220928_1620.h5

**Path (Globus):** `MiV:/Microcircuit_Small/results/Microcircuit_Small_results_20220928_1620.h5` (collection `0028aea1-ffc7-44b6-aec9-dd748ac839c4`)
**Local copy:** `/work2/11623/hyoklee/NeuroFAIR/globus_dl/Microcircuit_Small_results/Microcircuit_Small_results_20220928_1620.h5` (Vista, via Globus HTTPS)
**Size:** 14,738,912 bytes (14.7 MB)
**Last modified:** 2022-09-28 23:21:55 UTC
**Added:** 2026-10-21

## Summary

Simulation **output** (not input) of the small MiV Microcircuit run on 2022-09-28 16:20: spike trains for all 4 populations, somatic voltage traces for a subset of recorded cells, and 2 local field potential (LFP) electrodes. A single 750 ms trial (Trial Index = 0, Trial Duration = 750 ms), sampled at dt = 0.1 ms. It is the only results file on the MiV collection, and it is the reference shape for the `Microcircuit_Small_results.h5` written by the case-6 runs (see [miv_frontera_case6.md](miv_frontera_case6.md)).

All per-cell data uses the NeuroH5 cell-attribute layout (`Cell Index` / `Attribute Pointer` / `Attribute Value`, chunked 4000, extendible `/Inf`). Values are float32; `Trial Index` and `artificial` are uint8.

## Population Registry (`/H5Types/Populations`)

| Population | GID start | Count |
|---|---|---|
| STIM | 0 | 10 |
| PYR | 10 | 80 |
| PVBC | 90 | 53 |
| OLM | 143 | 44 |

187 cells in total, which matches the small circuit used on Vista (see [miv_vista_case6.md](miv_vista_case6.md)). `/H5Types` also holds the committed `Population labels`/`range`/`projections` types and `Valid population projections` {10}.

## Spike Events (`/Populations/<pop>/Spike Events/{t, Trial Index, Trial Duration, artificial}`)

| Population | Cell entries | Spikes (`t` values) |
|---|---|---|
| STIM | 20 | 662 |
| PYR | 29 | 45 |
| PVBC | 106 | 4,202 |
| OLM | 44 | 217 |

Spike times (ms) include the pre-trial equilibration window. OLM spikes range from −244.15 to 220.95 ms. STIM cells are flagged `artificial = 1`.

## Intracellular soma traces (`/Populations/<pop>/Intracellular soma*/{t, v}`)

| Population | Groups | Cell entries/group | Samples/group |
|---|---|---|---|
| OLM | `Intracellular soma` | 22 | 82,511 |
| PVBC | `Intracellular soma` | 24 | 90,012 |
| PYR | `Intracellular soma.0` … `.12` (13 groups) | 26 | 97,513 |

`t` and `v` are parallel. Traces are 0.1 ms sampled, starting at −250 ms; `Attribute Pointer` shows 5,000-sample segments per cell entry.

## Local Field Potential

`/Local Field Potential Electrode 0` and `/Local Field Potential Electrode 1`, each with `t` and `v` of shape (7503,) float32. `t` runs 0 → 750 ms at 0.1 ms.

## Related files

- [MiV_Cells_Microcircuit_Small_20220410.h5](source-MiV_Cells_Microcircuit_Small_20220410_h5.md): cell input for the same small circuit
- [MiV_Connections_Microcircuit_Small_20220410.h5](source-MiV_Connections_Microcircuit_Small_20220410_h5.md): connectivity input
- [MiV_input_spikes.h5](source-MiV_input_spikes_h5.md): STIM input spike trains
- [concept: Populations](concept-populations.md)
