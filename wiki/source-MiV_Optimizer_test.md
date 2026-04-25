# MiV-Simulator 7-Optimization Experiment on Aurora

**Date**: 2026-04-24
**Platform**: Aurora (Intel GPU, Aurora MPICH 5.0, SYCL/oneAPI)
**Job**: PBS debug-scaling queue, `gpu_hack` allocation
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

Compilation is automatic via `miv_simulator.mechanisms.compile_and_load(directory='mechanisms')` when `optimize-network` initializes the Env. Compiled artifacts go to `mechanisms/compiled/<hash>/x86_64/`.

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

**Result: FAILED**

**Changes from Run 1**:
- Replaced bare `mpiexec` with `/opt/cray/pals/1.8/bin/mpiexec`

**Error**: `libnl_3_5` symbol not found in system libnl

```
ImportError: /usr/lib64/libnl-3.so.200: version 'libnl_3_5' not found
(required by /home/hyoklee/miniconda3/envs/miv/lib/libmpi.so.40)
```

**Root cause**: Cray `libfabric.so.1` is in `LD_LIBRARY_PATH` and links to `/usr/lib64/libnl-3.so.200` (system v25.0). Because `libfabric` is loaded early in the dynamic linker, it pulls the system `libnl-3.so.200` into the linker cache. When h5py's OpenMPI (`libmpi.so.40`) then loads, it needs `libnl-route-3.so.200` → `libnl-3.so.200` and gets the cached v25.0 instead of conda's v26.0 (which exports `libnl_3_5`).

Confirmed via:
```
ldd /opt/cray/libfabric/1.22.0/lib64/libfabric.so.1 | grep libnl
# → /usr/lib64/libnl-3.so.200
```

**Fix**: `export LD_PRELOAD=${MIV_PREFIX}/lib/libnl-3.so.200` forces conda v26.0 into the linker cache before `libfabric` can inject the system v25.0.

**Queue context**: Aurora debug queue had persistent zombie jobs running 4–70 hours past walltime limit. Switched to `debug-scaling` queue for subsequent runs.

---

## Run 3 — PBS job 8449571 (2026-04-24 ~18:29, debug-scaling)

**Result: PARTIAL — mechanisms compiled, optimizer started, templates import failed**

**Changes from Run 2**:
- Added `LD_PRELOAD=${MIV_PREFIX}/lib/libnl-3.so.200`
- Changed queue to `debug-scaling`

**What succeeded**:
- h5py 3.16.0 (MPI=True) imported cleanly — LD_PRELOAD fix confirmed working
- Cray PALS mpiexec launched 9 ranks successfully
- distwq initialized 8 collective worker ranks (ranks 1–7) + 1 broker
- MiV-Simulator Env initialized with correct paths:
  - `dataset_prefix = /lus/flare/projects/gpu_hack/iowarp/neuroh5`
  - `data_file_path = .../Microcircuit_Small/MiV_Cells_Microcircuit_Small_20220410.h5`
  - `connectivity_file_path = .../Microcircuit_Small/MiV_Connections_Microcircuit_Small_20220410.h5`
  - `spike_input_path = .../MiV_input_spikes.h5`
  - Population ranges: STIM(0–10), PYR(10–80), PVBC(90–53), OLM(143–44)
  - Projections: OLM→{PVBC, PYR}, PVBC→{OLM, PVBC, PYR, STIM}, PYR→{OLM, PVBC, PYR, STIM}
- All 34 .mod files compiled successfully (`nrnivmodl`, 9 vecevent deprecation warnings, non-fatal)
- `libnrnmech.so` and `special` linked in `mechanisms/compiled/<hash>/x86_64/`
- `Creating cells...` reached — optimizer began network initialization
- dmosopt output file created: `results/network/dmosopt.optimize_network_20260424_1829.h5` (65 KB)

**Error**: `ModuleNotFoundError: No module named 'templates'` on all 9 MPI ranks

```
ModuleNotFoundError: No module named 'templates'
Abort(1) on node 0 (rank 0 in comm 0): application called MPI_Abort(MPI_COMM_WORLD, 1)
...
Abort(1) on node 7 (rank 7 in comm 0): application called MPI_Abort(MPI_COMM_WORLD, 1)
```

**Root cause**: MiV-Simulator's `import_object_by_path("templates.PyramidalCellBilash.PyramidalCell")` and `"templates.PRN_neuron.PRN"` need the `7-optimization/templates/` namespace package in Python's import path. Worker processes launched by Cray PALS `mpiexec` do not inherit the current working directory in `sys.path`, so `templates` is not findable even though it exists at `7-optimization/templates/`.

**Fix**: `export PYTHONPATH=${OPT_DIR}:${PYTHONPATH:-}` prepended before the mpiexec call, so all ranks inherit the path to `7-optimization/`.

---

## Run 4 — PBS job 8450374 (2026-04-24 22:30, debug-scaling)

**Result: FAILED — make_cells: unknown cell configuration type for cell type STIM**

**Changes from Run 3**:
- Added `export PYTHONPATH=${OPT_DIR}:${PYTHONPATH:-}`

**What succeeded**:
- PYTHONPATH fix confirmed: `templates.PyramidalCellBilash.PyramidalCell` and `templates.PRN_neuron.PRN` imported on all ranks — no `ModuleNotFoundError`
- Second dmosopt output file created: `results/network/dmosopt.optimize_network_20260424_2230.h5` (65 KB)

**Error**: `RuntimeError: make_cells: unknown cell configuration type for cell type STIM` on all 9 ranks

```
File "network.py", line 987, in make_cells
    raise RuntimeError(
RuntimeError: make_cells: unknown cell configuration type for cell type STIM
Abort(1) on node 0 (rank 0 in comm 0): application called MPI_Abort(MPI_COMM_WORLD, 1)
...
Abort(1) on node 7 (rank 7 in comm 0): application called MPI_Abort(MPI_COMM_WORLD, 1)
```

**Root cause**: `make_cells` iterates over all `env.celltypes` keys (OLM, PVBC, PYR, STIM). For each population it expects either `Trees` (biophysical morphology) or the `coordinates_ns` namespace (`Generated Coordinates`). STIM has neither — it has only `Coordinates` and `Input Spikes A Diag` namespaces. STIM is a VecStim spike-input population configured with `template: VecStim` and `spike train:` in the YAML; it is intended to be created by `init_input_cells`, not `make_cells`.

**Fix**: Patched `make_cells` in the installed `network.py` to skip populations that have `"spike train"` in their `celltypes` config (i.e., VecStim input populations):

```python
# Spike-input populations (VecStim) are created by init_input_cells, not make_cells
if "spike train" in env.celltypes.get(pop_name, {}):
    continue
```

File patched: `/home/hyoklee/miniconda3/envs/miv/lib/python3.12/site-packages/miv_simulator/network.py` (after `env.pc.barrier()`, before the logging/template loading block).

---

## Run 5 — PBS job 8450380 (2026-04-24 22:41, debug-scaling)

**Result: PARTIAL — cells created, connections loaded, spike init crashed in neuroh5**

**Changes from Run 4**:
- Patched `network.py:make_cells` to skip VecStim (spike train) populations

**What succeeded**:
- STIM skip confirmed: OLM (6 cells/rank), PVBC (6–7 cells/rank), PYR (10 cells/rank) all created from morphology/coordinates
- Connections loaded: ~400K connections per rank across all 7 projections (OLM→{PVBC,PYR}, PVBC→{OLM,PVBC,PYR,STIM}, PYR→{OLM,PVBC,PYR,STIM})
- Third dmosopt output file created: `dmosopt.optimize_network_20260424_2240.h5` (65 KB)

**Error**: neuroh5 assertion failure in `append_rank_attr_map` during STIM spike input initialization

```
append_rank_attr_map: index not found in node rank map:
Assertion 'it != node_rank_map.end()' failed in
file '/home/hyoklee/neuroh5/src/data/append_rank_attr_map.cc' line 53
terminate called after throwing an instance of 'AssertionFailureException'
x4311c6s7b0n0.hsn.cm.aurora.alcf.anl.gov: rank 0 died from signal 6
```

**Root cause**: `init_input_cells` calls `scatter_read_cell_attributes` to read STIM spike trains from `MiV_input_spikes.h5`. That file's H5Types registry records STIM with Count=1000 (full microcircuit). The `ldbal_cell_attr` function builds a `node_rank_map` covering 1000 GIDs, but the neuroh5 `append_rank_attr_map` assertion fails because a GID read from the float attribute data is not found in the rank map — a size mismatch between the spike file (1000 STIM GIDs) and the small circuit cell file (10 STIM GIDs, range 0–9).

**Fix**: Patched `init_input_cells` in `network.py` to use `scatter_read_cell_attribute_selection` with only the 10 GIDs from the cells-file population range (`env.celltypes[pop_name]["start"]` = 0, `["num"]` = 10) instead of `scatter_read_cell_attributes` which reads all GIDs from the spike file:

```python
# Use the cells-file population range so only this circuit's GIDs are read
pop_start = env.celltypes[pop_name].get("start", 0)
pop_num = env.celltypes[pop_name].get("num", 0)
pop_gid_selection = list(range(pop_start, pop_start + pop_num))
vecstim_iter, vecstim_attr_info = scatter_read_cell_attribute_selection(
    input_path, pop_name, pop_gid_selection, namespace=input_ns, ...
)
```

---

## Run 6 — PBS job 8450410 (2026-04-24 23:01, debug-scaling)

**Result: PARTIAL — 28 evaluations completed but all returned zero spikes; killed by walltime at 00:01**

**Changes from Run 5**:
- Patched `init_input_cells` to use `scatter_read_cell_attribute_selection` with cells-file GID range for VecStim populations

**What succeeded**:
- All previous fixes confirmed working: LD_PRELOAD (libnl), PYTHONPATH (templates), make_cells STIM skip, scatter_read_cell_attribute_selection GID selection — no crashes
- Optimization loop launched: dmosopt NSGA-II started, distwq distributed tasks across 8 worker ranks
- 28 evaluations completed (tasks 0–27); task 28 killed in progress by PBS walltime
- dmosopt checkpoint: `results/network/dmosopt.optimize_network_20260424_2301.h5` grew to **127 KB** (vs 65 KB baseline for prior failed runs)
- PBS `.out` file: 4.2 MB of log output

**Critical issue 1 — All evaluations returned zero cell activity**:
- All 28 evaluations: `n_active = 0` for PYR (80 cells), PVBC (53 cells), OLM (44 cells)
- All 10 objectives: `feature: 0.0` (no spikes measured)
- The 1250ms NEURON simulation ran to completion (task 0: ~887 wall-clock seconds) but produced no action potentials
- Hypothesis: `scatter_read_cell_attribute_selection` for STIM GIDs 0–9 may return empty spike trains if GIDs 0–9 are not present in `MiV_input_spikes.h5` with spike data, or if the small circuit STIM GID range (0–9) does not match the spike file's indexed GIDs. The VecStim objects would then have no spike times and drive no input current into PYR/PVBC/OLM cells.

**Critical issue 2 — simtime truncated 27 of 28 evaluations to ~10ms**:
- `miv_simulator.utils.simtime` reported `allocated wall time is 0.50 hours` — only half the actual PBS walltime of 1:00:00
- `max wall time is 1774.06 s` (setup excluded 25.94 s from 1800 s)
- Task 0 ran correctly to t=1250ms (887 wall-clock seconds), consuming ~887 s of simtime budget
- For all subsequent tasks, simtime detected near-zero remaining budget and truncated simulations at t=10ms with warning: `not enough time to complete 10.02 ms simulation, simulation will likely stop around -249.00 ms`
- Tasks 1–27 each ran in ~5–10 wall-clock seconds (vs 887 s for task 0)
- Root cause: simtime reads PBS walltime via some mechanism that returns 0.50 hours instead of 1.00 hours under Cray PALS mpiexec — possibly reading process CPU time limit rather than job walltime, or misinterpreting a PBS resource variable
- Even with simtime fixed, tasks 1–27 would still return zero spikes (same underlying zero-spike issue)

**dmosopt results** (scientific outcome):
- 28 evaluations, all with zero fitness; NSGA-II operated on a degenerate landscape
- No meaningful Pareto front generated
- Checkpoint file preserved for inspection

**Root cause of simtime truncation (clarified)**:
- Synapse initialization calls `h.finitialize()` many times before the first actual `h.run()`; each call adds the inter-call elapsed wall time to `tcsum` as "init time"
- This inflates `tcsum` to ~1789s before any simulation runs, exceeding the 1774s budget (0.5 hours × 3600 - 26s setup)
- All 28 evaluations were truncated to ~10ms; 10ms simulations produce no spikes

**Fixes applied for Run 7**:
1. **simtime fix**: Added `max_walltime_hours: 5.5` to `config/optimize_network.yaml` `kwargs` section — passes directly to `Env(max_walltime_hours=5.5)`, giving 5.5 × 3600 = 19800s budget (vs 1800s default); enough for ~20 evaluations even with synapse setup overhead
2. **Queue/walltime**: Changed PBS queue from `debug-scaling` (1hr max) to `gpu_hack` with `walltime=06:00:00` (6-hour job); `small`/`backfill-small` queues returned "access denied"; `gpu_hack` queue (no walltime limit) accepted
3. **GID verification**: STIM GIDs 0,1,4,6,7,8,9 confirmed present in `MiV_input_spikes.h5` with 21–33 spikes each in [0,1250ms]; GIDs 2,3,5 absent but `scatter_read_cell_attribute_selection` handles them gracefully; spike loading is correct

---

## Run 7 — PBS job 8450989 (2026-04-25, capacity 6hr) — PENDING

**Changes from Run 6**:
- `config/optimize_network.yaml`: added `max_walltime_hours: 5.5`
- PBS queue changed from `debug-scaling` to `capacity`, walltime `01:00:00` → `06:00:00`

**Queue note**: Initially submitted to `gpu_hack` queue (job 8450905) but that routed to reservation `R8428985` (owned by mluczkow, scheduled to start Tue Apr 28 13:00 — 3-day wait). Cancelled and resubmitted to `capacity` queue (168-hr max, 1 node, 6-hr walltime). `small` and `backfill-small` returned "access denied" for gpu_hack allocation.

Expected outcome: simtime budget 19800s >> synapse setup overhead (~1789s) + 15 evals × 887s (~13305s) = ~15094s total; all 15 evaluations (pop=5, gen=3) should complete within 6 hours; cells should produce spikes in full 1250ms simulations; dmosopt NSGA-II Pareto front expected.

---

## Known issues and workarounds

1. **OpenMPI PRRTE slot count mismatch**: Do not use bare `mpiexec` (OpenMPI) for multi-rank jobs on Aurora. Use `/opt/cray/pals/1.8/bin/mpiexec` for PBS-integrated slot allocation.

2. **Cray libfabric libnl version conflict**: `libfabric.so.1` loads `/usr/lib64/libnl-3.so.200` (v25.0, missing `libnl_3_5`) before conda's OpenMPI can load its own v26.0. Fix: `export LD_PRELOAD=${MIV_PREFIX}/lib/libnl-3.so.200`.

3. **templates namespace not in MPI worker sys.path**: Cray PALS mpiexec worker ranks don't inherit cwd in `sys.path`. Fix: `export PYTHONPATH=/path/to/7-optimization:${PYTHONPATH:-}`.

4. **make_cells STIM RuntimeError**: `make_cells` tries to create all `celltypes` populations but STIM (VecStim) has neither `Trees` nor `Generated Coordinates`. Fix: patch `make_cells` in `network.py` to skip populations with `"spike train"` in their `celltypes` config — those are created by `init_input_cells`.

5. **neuroh5 append_rank_attr_map assertion**: `init_input_cells` calls `scatter_read_cell_attributes` on the spike input file, which has 1000 STIM GIDs (full circuit). The `node_rank_map` built from this file causes an assertion failure because the small circuit's actual GIDs (0–9) don't match. Fix: patch `init_input_cells` to use `scatter_read_cell_attribute_selection` with only the 10 GIDs from `env.celltypes[pop_name]["start"]` / `["num"]`.

6. **h5py import order** (same as build-test): `utils/__init__.py` has `import h5py` as first line. Fix is already applied in conda `miv` env.

7. **MIV_SKIP_MPI_CHECK=1**: Required to bypass PR 103's startup h5py parallel check.

8. **NRNHOME**: Must be set to `${MIV_PREFIX}/lib/python3.12/site-packages/neuron/.data` so NEURON 9.x pip wheel finds its binaries.

9. **Aurora debug queue zombie jobs**: PBS does not enforce the 1hr walltime on some jobs. Use `debug-scaling` queue instead of `debug`.

10. **Zero cell activity / silent network** (Run 6, OPEN): All 28 evaluations returned n_active=0 for all populations. The STIM VecStim initialization patch (issue 5 fix) was accepted without crashing, but cells produce no spikes. Likely cause: STIM GIDs 0–9 in `MiV_input_spikes.h5` have no spike data in the `Input Spikes A Diag/Spike Train` namespace, so VecStim objects drive no current. Needs investigation: check what GIDs are indexed in `MiV_input_spikes.h5` and verify spike train data exists for the small circuit's GID range.

11. **simtime walltime misdetection** (Run 6, OPEN): `miv_simulator.utils.simtime` reads `allocated wall time = 0.50 hours` for a 1:00:00 PBS job. After the first 887-second evaluation consumed this budget, simtime truncated all subsequent simulations to ~10ms. Likely cause: simtime reads CPU time limit or a PBS variable that reports half the wall time under Cray PALS. Fix candidates: set `export PBS_WALLTIME=3600` before mpiexec, or disable simtime via its configuration, or use a 2-hour walltime job to work around the factor-of-2 error.

---

## PBS script (Run 4 / current)

Key environment variables:
```bash
AURORA_MPICH=/opt/aurora/26.26.0/spack/unified/1.1.1/install/linux-x86_64/mpich-5.0.0.aurora_test.3c70a61-hlkigtk
MIV_PREFIX=/home/hyoklee/miniconda3/envs/miv
OPT_DIR=/home/hyoklee/MiV-Simulator-Cases/7-optimization
NEUROH5_DIR=/lus/flare/projects/gpu_hack/iowarp/neuroh5

export LD_LIBRARY_PATH=${AURORA_MPICH}/lib:${MIV_PREFIX}/lib:${LD_LIBRARY_PATH:-}
export LD_PRELOAD=${MIV_PREFIX}/lib/libnl-3.so.200
export NRNHOME=${MIV_PREFIX}/lib/python3.12/site-packages/neuron/.data
export PATH=${NRNHOME}/bin:${PATH}
export MIV_SKIP_MPI_CHECK=1
export PYTHONPATH=${OPT_DIR}:${PYTHONPATH:-}
```

Launch command:
```bash
/opt/cray/pals/1.8/bin/mpiexec -n 9 \
    ${MIV_PREFIX}/bin/optimize-network \
    --config-path=./config/optimize_network.yaml \
    --optimize-file-dir=results/network \
    --verbose \
    --nprocs-per-worker=8 \
    --n-epochs=1 \
    --population-size=5 \
    --num-generations=3 \
    "--dataset_prefix=${NEUROH5_DIR}" \
    "--config_prefix=${OPT_DIR}/config" \
    "--spike_input_path=${NEUROH5_DIR}/MiV_input_spikes.h5" \
    "--spike_input_namespaces=Input Spikes A Diag" \
    "--spike_input_attr=Spike Train" \
    "--arena_id=A" \
    "--stimulus_id=Diag" \
    "--coordinates_namespace=Generated Coordinates" \
    "--io_size=1" \
    "--results_path=${OPT_DIR}/results/network"
```

---

## Related files

- PBS script: `/home/hyoklee/wrp/run/miv_optimize_test.pbs`
- Run log: `/home/hyoklee/wrp/run/miv_optimize_test_run.log`
- 7-optimization source: `/home/hyoklee/MiV-Simulator-Cases/7-optimization/`
- Optimization config: `/home/hyoklee/MiV-Simulator-Cases/7-optimization/config/optimize_network.yaml`
- Results dir: `/home/hyoklee/MiV-Simulator-Cases/7-optimization/results/network/`
- [MiV-Simulator build+test](source-MiV_Simulator_build_test.md) — build environment and test results
- [MiV_Cells_Microcircuit_Small_20220410.h5](source-MiV_Cells_Microcircuit_Small_20220410_h5.md) — cell data used
- [MiV_Connections_Microcircuit_Small_20220410.h5](source-MiV_Connections_Microcircuit_Small_20220410_h5.md) — connectivity data used
