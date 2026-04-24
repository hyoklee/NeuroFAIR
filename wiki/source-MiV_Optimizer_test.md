# MiV-Simulator 7-Optimization Experiment on Aurora

**Date**: 2026-04-24
**Platform**: Aurora (Intel GPU, Aurora MPICH 5.0, SYCL/oneAPI)
**Job**: PBS debug queue, `gpu_hack` allocation
**Source**: GazzolaLab/MiV-Simulator-Cases, `7-optimization/` directory
**MiV-Simulator**: `~/MiV-Simulator` (v0.3.0 + PR 103, conda env `miv`)
**PBS script**: `/home/hyoklee/wrp/run/miv_optimize_test.pbs`

---

## Experiment description

The 7-optimization case runs multi-objective evolutionary optimization of CA1 microcircuit synaptic weight parameters using [dmosopt](https://github.com/iraikov/dmosopt). It optimizes synaptic weights for 3 populations (PYR, PVBC, OLM) to match target firing rates and activity patterns.

**Optimization config**: `config/optimize_network.yaml`
- Objectives (10): firing rate + fraction-active statistics for PYR, PVBC, OLM
- Parameter space: AMPA/NMDA/GABA_A synaptic weights (ranges 0.1–50 × baseline)
- Optimizer: NSGA-II (nsga2), surrogate: mega-GP (megp)
- Network config: `Network_Clamp_Microcircuit_Small_Bilash_PR.yaml` — Bilash CA1_PYR + Pinsky-Rinzel OLM/PVBC
- `tstop=1250 ms`, `dt=0.025 ms`, `v_init=-77 mV`

**Test run parameters** (small, exploratory):
- `--population-size=5`, `--num-generations=3`, `--n-epochs=1`
- `--nprocs-per-worker=8` → 9 total MPI ranks (1 controller + 1 worker of 8)

**Data**: `MiV_Cells_Microcircuit_Small_20220410.h5` (87 MB), `MiV_Connections_Microcircuit_Small_20220410.h5` (39 MB), `MiV_input_spikes.h5` (3.2 MB)

---

## Mechanisms

Three mechanism directories compiled via `miv_simulator.mechanisms.compile_and_load`:
- `mechanisms/ca1_Bilash/` — 18 .mod files (CA1 PYR channels: Ka, Kca, Km, mAHP, cal, car, cat, hha2, etc.)
- `mechanisms/PinskyRinzel/` — 15 .mod files (OLM/PVBC channels: nasPR, ksPR, kcaPR, kahpPR, etc.)
- `mechanisms/synaptic/` — 3 .mod files (lin_exp2syn, lin_exp2synNMDA, NMDA_CA1_pyr_SC)
- `mechanisms/vecevent.mod` — VecStim point process for spike injection

Compilation is automatic via `miv_simulator.mechanisms.compile_and_load(directory='mechanisms')` when `optimize-network` initializes the Env.

---

## Run 1 — PBS job 8448630 (2026-04-24 01:52)

**Result: FAILED**

**Error**: OpenMPI PRRTE "not enough slots"

```
There are not enough slots available in the system to satisfy the 9
slots that were requested by the application:
  /home/hyoklee/miniconda3/envs/miv/bin/optimize-network
```

**Root cause**: `mpiexec -n 9` resolved to the OpenMPI from the conda `miv` env (PRRTE runtime). PRRTE did not detect the PBS allocation's CPU count — it saw fewer than 9 available slots. PBS `select=1` without explicit `ncpus` may only expose 1 slot to OpenMPI, even though `TSK=208` appears in qstat.

**Fix**: Use Cray PALS `mpiexec` (`/opt/cray/pals/1.8/bin/mpiexec`) which integrates directly with the PBS job allocation and correctly exposes all available cores.

---

## Run 2 — PBS job 8449551 (2026-04-24 ~12:27)

**Status**: PENDING (debug queue; zombie jobs blocking queue)

**Changes from Run 1**:
- Replaced bare `mpiexec` with `/opt/cray/pals/1.8/bin/mpiexec`

**Queue context**: Aurora debug queue has persistent zombie jobs running 4–70 hours past the 1hr walltime limit; PBS is not enforcing walltime kill. Job is 6th in queue behind 3 long-stale queued jobs (8293387, 8293399, 8385721) and behind 2 zombie running jobs (8449426, 8449495).

---

## Known issues and workarounds

1. **OpenMPI PRRTE slot count mismatch**: Do not use bare `mpiexec` (OpenMPI) for multi-rank MPI jobs on Aurora. Use `/opt/cray/pals/1.8/bin/mpiexec` for PBS-integrated slot allocation.

2. **h5py import order** (same as build-test): `utils/__init__.py` has `import h5py` as first line — fix is already in place in the conda `miv` env.

3. **MIV_SKIP_MPI_CHECK=1**: Required to bypass PR 103's startup h5py parallel check.

4. **Aurora debug queue zombie jobs**: PBS does not enforce the 1hr walltime on some jobs. Queue can be blocked for hours/days by jobs that exceed their walltime.

---

## Related files

- PBS script: `/home/hyoklee/wrp/run/miv_optimize_test.pbs`
- Run log: `/home/hyoklee/wrp/run/miv_optimize_test_run.log`
- 7-optimization source: `/home/hyoklee/MiV-Simulator-Cases/7-optimization/`
- Optimization config: `/home/hyoklee/MiV-Simulator-Cases/7-optimization/config/optimize_network.yaml`
- Results dir (when run completes): `/home/hyoklee/MiV-Simulator-Cases/7-optimization/results/network/`
- [MiV-Simulator build+test](source-MiV_Simulator_build_test.md) — build environment and test results
- [MiV_Cells_Microcircuit_Small_20220410.h5](source-MiV_Cells_Microcircuit_Small_20220410_h5.md) — cell data used
- [MiV_Connections_Microcircuit_Small_20220410.h5](source-MiV_Connections_Microcircuit_Small_20220410_h5.md) — connectivity data used
