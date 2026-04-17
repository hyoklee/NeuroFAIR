# dmosopt_Motoneuron.h5

**Path:** `/lus/flare/projects/gpu_hack/iowarp/neuroh5/Motoneuron_model/motoneuron_20230210/dmosopt_Motoneuron.h5`
**Size:** 121 MB
**Added:** 2026-04-17

## Summary

Distributed multi-objective optimization (DMOSOPT) results for a biophysical motoneuron model. Contains 42,000 parameter configurations evaluated across 8 constraints, 10 features, and 5 objectives. Unlike other files in this collection, this is not a neuroh5 circuit file — it is an optimization database recording the parameter search space for a single-cell conductance model.

## Structure (`/dmosopt_Motoneuron_neuron`)

### Evaluation records (`/dmosopt_Motoneuron_neuron/0`)

42,000 rows, each representing one model evaluation:

| Dataset | Shape | dtype | Description |
|---|---|---|---|
| parameters | (42000,) | 11-field struct | Conductance/morphology parameters |
| features | (42000,) | multi-field struct | Electrophysiological features measured |
| objectives | (42000,) | 5-field struct | Optimization objective error values |
| predictions | (42000,) | 5-field struct | Predicted objective values |
| constraints | (42000,) | 8-field struct | Constraint satisfaction values |
| epochs | (42000,) | uint32 | Optimization epoch index |

### Parameter space (11 dimensions)

| Parameter | Description |
|---|---|
| gc | Gap conductance |
| soma_gmax_Na | Somatic maximum Na conductance |
| soma_gmax_K | Somatic maximum K conductance |
| soma_gmax_KCa | Somatic maximum KCa conductance |
| soma_gmax_CaN | Somatic maximum CaN conductance |
| soma_g_pas | Somatic passive conductance |
| dend_gmax_CaL | Dendritic maximum CaL conductance |
| dend_gmax_CaN | Dendritic maximum CaN conductance |
| dend_gmax_KCa | Dendritic maximum KCa conductance |
| dend_g_pas | Dendritic passive conductance |
| cm_ratio | Membrane capacitance ratio |

### Objectives (5)

`rn_error`, `tau_error`, `fI_error`, `spike_amplitude_error`, `ISI_adaptation_error`

### Features

Input resistance (`rn`), membrane time constant (`tau`), f–I curve (7 current levels), ISI statistics (first, last, ratio, mean, std, N — 7 levels), spike threshold (7 levels), spike amplitude (7 levels).

### Constraints (8)

`monotonic_fI`, `rn_constr`, `tau_constr`, `spike_amplitude_constr`, `first_ISI_constr`, `ISI_adaptation_constr`, `pre_spk_count`, `initial_v_constr`

### Metadata

Optimization targets stored as structured arrays: `fI_target` (3×7), `ISI_adaptation_target` (3×7), `rn_target` (2,), `spike_amplitude_target` (3×7), `tau_target` (2,).

### Spec datasets

`constraint_spec` (8,), `feature_spec` (10,), `objective_spec` — enumerate the names of constraints, features, and objectives respectively.

## Notes

- This is the only non-circuit, single-cell optimization file in the collection.
- The Motoneuron model is distinct from the MiV hippocampal microcircuit populations.
- Successfully retrieved via Globus Transfer API on 2026-04-17.

## Related files

- [concept: MiV Microcircuit](concept-miv-microcircuit.md) — broader circuit context
