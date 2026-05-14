# VecStim Spike Input Namespace Fix — n_active=0 Root Cause

**Date**: 2026-05-14  
**Platform**: Polaris (NVIDIA A100, Cray MPICH 9.0.1)  
**Status**: Fix applied; bench5 (PBS 7158576) running to verify

---

## Problem

Every optimization run on Polaris (all bench1–bench5, all miv_opt_7 runs) showed
`n_active = 0` for all populations (PYR, PVBC, OLM). Cells never fired despite the
network being fully initialized and the simulation completing.

---

## Root Cause

There were **two independent namespace mismatches**, both pointing to the same
underlying cause: the spike input HDF5 file stores data under the namespace
`"Input Spikes A Diag"`, but the configuration expected `"Input Spikes"`.

### Mismatch 1 — `Microcircuit_Small.yaml` celltypes config (primary fix)

`/lus/grand/projects/gpu_hack/iowarp/miv_opt_case/config/Microcircuit_Small.yaml`
defines the STIM population's VecStim spike train source:

```yaml
# Before (wrong):
  STIM:
    template: VecStim
    spike train:
      namespace: Input Spikes      # ← HDF5 file has "Input Spikes A Diag"
      attribute: Spike Train
```

`init_input_cells` in `miv_simulator/network.py` reads this namespace and uses it
to look up `env.cell_attribute_info[pop_name]`. Since `"Input Spikes"` is not a
key in the file, `has_vecstim` remained `False` and the function never loaded
spike times into the VecStim objects. STIM cells existed but had empty spike
vectors → no input → n_active=0.

**Fix**: changed `namespace: Input Spikes` → `namespace: Input Spikes A Diag` in
both the case directory and `~/MiV-Simulator-Cases/7-optimization/config/`.

### Mismatch 2 — `--spike_input_namespace` PBS argument (secondary fix)

`make_cells` in `network.py` also checks `env.spike_input_namespaces` (set from
the CLI `--spike_input_namespace` argument) against the file's attribute info:

```python
set(env.spike_input_namespaces).intersection(
    set(env.spike_input_attribute_info[pop_name].keys())
)
```

The PBS scripts passed `--spike_input_namespace='Input Spikes'` but the file has
`'Input Spikes A Diag'`. The intersection was empty → `has_spike_train = False` →
STIM cells were created without VecStim objects at all.

**Fix**: changed `--spike_input_namespace='Input Spikes'` to
`--spike_input_namespace='Input Spikes A Diag'` in `miv_iowarp_bench.pbs` and
`miv_opt_polaris.pbs`.

---

## Diagnosis Path

The n_active=0 issue was present in all Polaris runs from the beginning (April 2026)
but was not recognized as a bug — it was assumed to be a VecStim registration error
similar to Aurora Runs 6–7.

**Bench5 (PBS 7158386, 2026-05-14)** provided the decisive clue: after applying the
`--spike_input_namespace` fix (Mismatch 2), n_active was still 0. The log showed:

```
*** Stimuli created in 1.35 s     ← init_input_cells ran
```

but the message `*** Initializing stimulus population STIM from input path ...`
was absent. This message is printed only when `has_vecstim = True` inside
`init_input_cells`, which requires the celltypes `spike train.namespace` to match
the file. Since it was absent, the celltypes config was the remaining blocker.

---

## Code trace

```
init_input_cells(env):
    for pop_name in sorted(env.celltypes.keys()):
        if "spike train" in env.celltypes[pop_name]:          # ← requires YAML fix
            vecstim_namespace = celltypes[pop_name]["spike train"]["namespace"]
            # "Input Spikes" → not in cell_attribute_info → has_vecstim = False
            # "Input Spikes A Diag" → found → has_vecstim = True → loads spikes
            has_vecstim = False
            if env.spike_input_namespaces intersect file_namespaces:
                has_vecstim = True   # ← also requires CLI fix
            if env.cell_attribute_info has vecstim_namespace:
                has_vecstim = True   # ← this path was also failing (YAML mismatch)
            if has_vecstim:
                # load spike times from HDF5 → populate VecStim vectors
```

---

## Files changed

| File | Change |
|---|---|
| `/lus/grand/projects/gpu_hack/iowarp/miv_opt_case/config/Microcircuit_Small.yaml` | `namespace: Input Spikes` → `namespace: Input Spikes A Diag` |
| `~/MiV-Simulator-Cases/7-optimization/config/Microcircuit_Small.yaml` | same |
| `~/polaris/pbs/miv_iowarp_bench.pbs` | `--spike_input_namespace='Input Spikes A Diag'` |
| `~/polaris/pbs/miv_opt_polaris.pbs` | same |

---

## Related fixes (same bench5 cycle)

| Issue | Fix |
|---|---|
| Chimaera port 9413 occupied on 127.0.0.1 | Use actual node IP from `PBS_NODEFILE` hostname + dynamic port (10000–30000, below ephemeral range), verified bindable on node IP before passing to Chimaera |
| core.* dumps from failed IOWarp runs | Guard sections 6 (IOWarp I/O) and 8 (IOWarp optimize) with `CHIMAERA_OK` flag — skip if Chimaera failed to start |
| VecStim "already exists" error (earlier runs) | Removed manual `nrnivmodl` from PBS scripts; `compile_and_load` handles mechanism compilation |

---

## Expected result after fix

With `init_input_cells` now loading actual spike times from `"Input Spikes A Diag"`,
STIM VecStim objects will fire at their recorded rates (~60 Hz average). This drives
AMPA excitation in PYR and PVBC, producing n_active > 0 and meaningful objective
function evaluations for the dmosopt optimizer.

**Verification**: PBS 7158576 (debug queue, 1h walltime, 2026-05-14).

---

## Related

- [MiV-Simulator + IOWarp CTE Benchmark](miv_iowarp_bench.md)
- [MiV-Simulator 7-optimization on Polaris](miv_opt_7_polaris.md)
- [clio-core GPU HBM vs CPU DRAM](concept-clio-core-gpu-memory.md)
