# Performance: MiV Case 6 (Gap Junctions) — clio-core CTE and Intel GPU

**Date**: 2026-05-07
**Platform**: Aurora (Intel GPU Max 1550 / Ponte Vecchio, Flare Lustre)
**Case**: `~/MiV-Simulator-Cases/6-gapjunctions/` — `Microcircuit_Small.yaml` with gap-junction coupling
**Network**: 189 cells (80 PYR + 56 OLM + 53 PVBC), `dt=0.025 ms`, `tstop=500 ms`
**MPI**: 12 ranks on 1 Aurora node, `debug-scaling` queue
**PBS job**: 8473164 (`miv_case6_perf.pbs`) — exit 0, walltime 00:00:46

---

## TL;DR

Case 6 cannot be benchmarked end-to-end against case 7's clio-core methodology
in the current `~/miniconda3/envs/miv` install. The intended 4-condition sweep
(`{Lustre, /dev/shm} × {CPU, Intel GPU}`) collapsed to 0/4 successful runs:

| # | Condition | Result | Blocker |
|---|---|---|---|
| A | CPU + Lustre | **fail** (rank 0 SIGSEGV in `init_network`, total 5.439 s) | run-network/case-6 segfault |
| B | CPU + /dev/shm | **fail** (rank 0 SIGSEGV in `init_network`, total 3.987 s) | same as A |
| C | Intel GPU + Lustre | **skipped** | `special-core` build failed |
| D | Intel GPU + /dev/shm | **skipped** | `special-core` build failed |

The clio-core pre-staging did succeed (129 MB Lustre→/dev/shm in **8.288 s**,
~15.6 MB/s — much slower than case 7's 0.332 s seen on Apr 27, suggesting
Lustre or page-cache cold-start contention this morning). That is the only
useful measurement produced by this run.

---

## Background

Case 6 adds gap-junction (GJ) coupling between PYR cells (per
`config/Gap_Junctions.yaml`) on top of the standard small-microcircuit
network. Compared to case 7-optimization (already documented in
[perf-clio-core.md](perf-clio-core.md)) it differs in:

- One additional HDF5 read: `MiV_gapjunctions_20230408.h5` (20 KB) declared via
  `Gap Junction Data:` in `config/Microcircuit_Small.yaml`.
- One additional mechanism: `mechanisms/ggap.mod` (gap-junction conductance).
- Direct simulation via `run-network` on `MPI_COMM_WORLD` (no `distwq` workers,
  no sub-communicator split). This was expected to avoid the CoreNEURON
  sub-comm deadlock that blocked GPU evaluation in case 7
  (commits 2e34a65 and 45fe9c9), but the run never reached CoreNEURON.

The "Intel GPU vs CPU" axis substitutes for the original NVIDIA-GPU comparison
in `~/NeuroFAIR/claude.md`: Aurora has no NVIDIA hardware, so the available
analog is the Intel GPU Max 1550 reached via oneAPI/SYCL through CoreNEURON.

---

## Inputs (verified)

All five input files were present and pre-staged successfully:

| File | Size | Source path |
|---|---|---|
| `MiV_Cells_Microcircuit_Small_20220410.h5` | 87 MB | `…/Microcircuit_Small/` |
| `MiV_Connections_Microcircuit_Small_20220410.h5` | 39 MB | `…/Microcircuit_Small/` |
| `MiV_gapjunctions_20230408.h5` | 20 KB | `…/Microcircuit_Small/` |
| `MiV_h5types_gj.h5` | 20 KB | `…/Microcircuit_Small/` |
| `MiV_input_spikes.h5` | 3.2 MB | (top-level) |
| **Total** | **129 MB** | `/lus/flare/projects/gpu_hack/iowarp/neuroh5/` |

`/dev/shm` mirror: `/dev/shm/miv_case6/` (cleaned up post-run).

---

## Failure 1 — `nrnivmodl-core` cannot find `nrnunits.lib` (resolved post-run)

**Symptom** (excerpt from `wrp/run/miv_case6_perf_run.log`):

```
terminate called after throwing an instance of 'std::runtime_error'
  what():  Could not find nrnunits.lib in any of:
    /tmp/tmpxzg66zp3/wheel/platlib/share/nmodl/nrnunits.lib
    /project/build_wheel/src/nmodl/share/nmodl/nrnunits.lib
make: *** […/nrnivmodl_core_makefile:185: x86_64/corenrn/mod2c/ggap.cpp] Aborted
```

**Cause**: the `nmodl` translator inside the NEURON pip wheel
(`~/miniconda3/envs/miv/lib/python3.12/site-packages/neuron/.data/bin/nmodl`)
has only **two** path candidates compiled in (verified via
`strings nmodl | grep nrnunits.lib`); both are wheel build paths, neither is
the installed `share/nmodl/`, and the binary does **not** consult `NRNHOME`,
`NEURON_HOME`, `MODLUNIT`, or any other env var. Even `nmodl --help` aborts
the same way — the lookup runs in the static initializer.

**Workaround that works** (verified on login node, post-PBS-8473164):

```bash
NRN_DATA=$CONDA_PREFIX/lib/python3.12/site-packages/neuron/.data
mkdir -p /tmp/tmpxzg66zp3/wheel/platlib/share/nmodl
ln -sfn "$NRN_DATA/share/nmodl/nrnunits.lib" \
        /tmp/tmpxzg66zp3/wheel/platlib/share/nmodl/nrnunits.lib
```

The first hardcoded candidate path is in user-writable `/tmp`, so creating it
unblocks the lookup. After this, `nrnivmodl-core mechanisms` runs to
completion and produces `~/MiV-Simulator-Cases/6-gapjunctions/x86_64/special-core`
(224 KB) with all 41 mechanisms including `ggap.cpp`. The build picks
`icpx` automatically from `oneapi/release/2025.3.1` (loaded module).

**Important caveat — built `special-core` is CPU-only**:

`ldd x86_64/special-core` shows only `libgomp`, `libiomp5`, libstdc++, libm, libc,
libpthread. **No SYCL, Level-Zero, oneMKL, or CUDA runtime is linked**, so
`--coreneuron-gpu` has no GPU backend to dispatch to and would either run on
CPU or fail. The CoreNEURON shipped in this NEURON pip wheel is built without
GPU support at all.

**To get an actual Intel-GPU CoreNEURON** (required by `claude.md`'s "Use
intel oneapi compilers whenever GPU is necessary"), NEURON + CoreNEURON must
be built from source against Aurora oneAPI 2025.3.1 with
`-DNRN_ENABLE_CORENEURON=ON -DCORENRN_ENABLE_GPU=ON` and SYCL backend
configured. The pip wheel cannot be retrofitted.

The CPU `special` (NEURON, not CoreNEURON) was built successfully on the
login node with `nrnivmodl mechanisms` before the job was submitted.

---

## Failure 2 — `run-network` rank-0 SIGSEGV in `init_network`

**Symptom** (identical for A and B, irrespective of storage tier):

```
INFO:miv_simulator:env.dataset_prefix = <prefix>
INFO:miv_simulator:env.cell_selection_path = None
INFO:miv_simulator:env.data_file_path = <prefix>/Microcircuit_Small/MiV_Cells_Microcircuit_Small_20220410.h5
[x4217c0s0b0n0:117623] *** Process received signal ***
… rank 0 died from signal 11
… rank 2 died from signal 15      ← (or rank 5; the other ranks are killed by mpiexec after rank 0 dies)
```

The crash happens **after** the dataset paths have been resolved and **before**
any timing markers (`make_cells`, `connect_cells`, `setup time`) are emitted —
so this is during `init_network`'s first HDF5 access (likely `read_population_ranges`
or the gap-junction file open, neither of which is timed individually).

**Repeatability**: identical crash in the May 2 case-6 run
(`/home/hyoklee/wrp/run/miv_gapjunctions_run.log`, same node-pattern signature).
Two runs, two months apart — deterministic.

**Actual crash site** (confirmed 2026-05-07 via login-node single-rank smoke test
with `mpirun -n 1 run-network ...`, full backtrace in `/tmp/smoke_nogj_full.log`):

```
[ 0] libc.so.6(+0x4ad00)
[ 1] libmpi.so.12(MPI_Comm_dup+0x89)
[ 2] neuroh5/io.so(+0x8673e)        ← here
[ 3] python3.12(+0x21d338)          ← Python C-API call into neuroh5
...
```

The SIGSEGV is inside `MPI_Comm_dup` invoked from `neuroh5/io.so` during
`init_network`'s neuroh5 setup, **not** anywhere near gap-junction code.

**`Gap Junction Data:` line is NOT the trigger** (hypothesis falsified): the
identical crash reproduces with the line removed
(`config/Microcircuit_Small_nogj.yaml`, smoke test B exit 139, same
`MPI_Comm_dup` frame).

**Why case 7 (`optimize-network`) works but case 6 (`run-network`) crashes**
(hypothesis): different import order. `optimize-network` enters via `distwq`,
which imports `mpi4py` first and initializes MPI before any neuroh5 import.
`run-network` imports `miv_simulator` first, which pulls neuroh5; the order
differs and neuroh5's `MPI_Comm_dup` call then dereferences a comm handle
whose internal layout disagrees with what the loaded `libmpi.so.12` expects.
mpi4py's 18 `RuntimeWarning` size-mismatch messages (32 vs 40 bytes, 48 vs 56,
etc.) are the loud evidence of this binary-incompat between the pip-wheel
mpi4py and the dynamically-found Aurora MPICH.

**Note on login-node vs compute-node MPI**:
- Login-node `mpirun` is OpenMPI (`prterun` launcher); the backtrace explicitly
  shows the call going through Aurora MPICH `libmpi.so.12` because neuroh5 was
  linked against it. OpenMPI launcher + MPICH library is a well-known
  unsupported combination, but the crash inside `MPI_Comm_dup` is the same
  symptom we see on compute nodes (PBS 8473164 used Cray PALS mpiexec, no
  OpenMPI). So the launcher choice is not the root cause; the broken MPI
  comm handle is.

**Genuine next step** (requires deeper investigation, not just config tweak):

1. Check the import order in `run-network` script entry point
   (`miv_simulator/scripts/run_network.py`) — does `import mpi4py.MPI` appear
   before any neuroh5 import?
2. Compare with `optimize-network` entry point and replicate its import
   ordering in case 6.
3. If reordering doesn't fix it, this is a neuroh5/mpi4py binary incompat
   that requires rebuilding one of: `neuroh5` against the runtime mpi4py, or
   `mpi4py` (`pip install --no-binary=mpi4py mpi4py`) against the runtime MPI.

---

## What did work

- **Pre-staging**: 129 MB Lustre → `/dev/shm/miv_case6/` in **8.288 s**
  (~15.6 MB/s). Compared with Apr 27's case-7 measurement of 0.332 s for the
  same 129 MB on the same paths, today's pre-stage was 25× slower — likely
  Lustre / OS-page-cache cold-start, not a structural change. (Case 6's
  pre-stage warms the same files case 7 already had cached in past runs.)
- **CPU `special` build** (`nrnivmodl mechanisms` on login node) — produced
  `~/MiV-Simulator-Cases/6-gapjunctions/x86_64/special` with all 41 mechanisms
  including `ggap.cpp` and `ggap.o`, no errors.
- **PBS job lifecycle**: `debug-scaling` queue accepted the job, ran on
  `x4217c0s0b0n0`, exited cleanly with status 0 (the `timeout 900` per-condition
  guards converted each crash to `fail(143)` without aborting the script).

---

## Method (jobscript)

`~/aurora/pbs/miv_case6_perf.pbs`:

1. Verify Lustre input files; record sizes.
2. Pre-stage all five inputs from Lustre to `/dev/shm/miv_case6/`; time the copy.
3. Build `special-core` with `CC=icx CXX=icpx nrnivmodl-core mechanisms`
   (Intel oneAPI). On failure (this run), conditions C and D are skipped.
4. For each of A/B/C/D, run `mpiexec -n 12 run-network …` wrapped in
   `timeout 900` and write per-phase timings (`max setup time`,
   `max simulation time`) plus wall-clock total to
   `miv_case6_perf_timing.csv`.
5. Cleanup `/dev/shm/miv_case6/`.

---

## Known constraints documented during this run

- **Aurora has no NVIDIA hardware** — `claude.md` requested "NVIDIA GPU" but
  the available accelerator is Intel GPU Max 1550. This benchmark substitutes
  Intel GPU via CoreNEURON oneAPI/SYCL.
- **`gpu_hack_large` queue is currently disabled** (`Ena=no` in `qstat -Q`);
  this run uses `debug-scaling` (max 1 h walltime).
- **clio-core has no GPU memory backend built**: no SYCL/oneAPI libs in
  `~/clio-core/build`, no `H5FDhermes` VFD. The /dev/shm pre-staging is the
  CPU-DRAM tier emulation used in case 7
  (see [concept-clio-core-gpu-memory.md](concept-clio-core-gpu-memory.md)).
- **Aurora pip-wheel mpi4py reports 18 binary-incompat warnings against
  Aurora MPICH** — see the run log. This is unrelated to today's blockers
  but is the most likely root cause of the `init_network` SIGSEGV.

---

## Recommended next experiment

Smallest viable test that would prove the gap-junction ingestion is the
trigger and unblock the CPU comparison:

1. `cp config/Microcircuit_Small.yaml config/Microcircuit_Small_nogj.yaml`
2. In `_nogj.yaml`, delete the `Gap Junction Data:` line.
3. Re-submit `miv_case6_perf.pbs` with `--config-file=Microcircuit_Small_nogj.yaml`.
4. If A and B succeed, the SIGSEGV is isolated to `read_gap_junctions` (or
   the equivalent inside `init_network`) and the actual fix is in
   `miv_simulator` source.

Separately, to unblock C and D:

- Set `export MODLUNIT="${MIV_PREFIX}/lib/python3.12/site-packages/neuron/.data/share/nmodl/nrnunits.lib"`
  before `nrnivmodl-core` and rerun the build step in isolation (login node,
  no MPI, fast feedback). If `special-core` builds, re-submit the full
  4-condition jobscript.

---

## Related pages

- [perf-clio-core.md](perf-clio-core.md) — case 7 baseline (Lustre vs /dev/shm)
- [concept-clio-core-gpu-memory.md](concept-clio-core-gpu-memory.md) — why
  GPU HBM doesn't help today's MiV workload
- [source-MiV_Optimizer_test.md](source-MiV_Optimizer_test.md) — case 7
  optimization run history including GPU evaluation hangs
- Run log: `/home/hyoklee/wrp/run/miv_case6_perf_run.log`
- Timing CSV: `/home/hyoklee/wrp/run/miv_case6_perf_timing.csv`
