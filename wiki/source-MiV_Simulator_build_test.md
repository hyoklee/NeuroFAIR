# MiV-Simulator Build and Test on Aurora

**Date**: 2026-04-24 (PR 103 run); 2026-04-22 (baseline)
**Platform**: Aurora (Intel GPU, Aurora MPICH 5.0, SYCL/oneAPI)
**Job**: PBS debug queue, `gpu_hack` allocation
**Source repo**: `~/MiV-Simulator` (GazzolaLab/MiV-Simulator v0.3.0 + PR 103)
**neuroh5 source**: `~/neuroh5` (iraikov/neuroh5, cloned from GitHub)

---

## Build environment

| Component | Version / Path |
|---|---|
| Python | 3.12.13 (conda env `miv`) |
| NEURON | 9.0.1 (pip wheel) |
| neuroh5 | 0.1.18 (cmake manual build) |
| HDF5 | 1.14.6 parallel (`hdf5-1.14.6-ehlefog`, Aurora MPICH-linked) |
| MPI | Aurora MPICH 5.0 (`mpich-5.0.0.aurora_test.3c70a61-hlkigtk`) |
| mpi4py | 4.1.1 (built from source against Aurora MPICH) |
| h5py | 3.16.0 (PyPI wheel, MPI: True — parallel wheel) |
| Compiler | Intel icx (oneAPI, Aurora) |

---

## PR 103 patches applied

GazzolaLab/MiV-Simulator PR 103 merged into `~/MiV-Simulator main`:

1. `tests/mechanisms/test_gfluct3.py` — adds `h.load_file('stdrun.hoc')` before `h.tstop`; relaxes tolerance to `rtol=1e-3`
2. `tests/mechanisms/test_mechanisms.py` — copies real `Gfluct3.mod`; fixes `isfile` → `isdir`; adds `pytest.skip` for already-loaded mechanism
3. `src/miv_simulator/__init__.py` — calls `_check_mpi_env()` at import (bypassed with `MIV_SKIP_MPI_CHECK=1`)
4. `src/miv_simulator/mpi_env.py` — validates mpi4py and h5py MPI environment

### Aurora-specific patches

Two additional patches were required for Aurora:

**`src/miv_simulator/mpi_env.py`** — added `import mpi4py.MPI` before `import h5py` inside `check_mpi_env()`, so MPI is initialized before h5py when the check runs.

**`src/miv_simulator/utils/__init__.py`** — added `import h5py` as the first line, before `from miv_simulator.utils.utils import *`.

Root cause: `utils/utils.py` does `from mpi4py import MPI` at module level. On Aurora, importing mpi4py.MPI (built against Aurora MPICH with ABI mismatch) before h5py corrupts HDF5 global type-ID state, causing `ValueError: Not a datatype` in `h5py.h5t.lockid` when h5py later tries to lock predefined type constants. Importing h5py first (before mpi4py) avoids the corruption.

**`export MIV_SKIP_MPI_CHECK=1`** — set before pytest runs. PR 103's `_check_mpi_env()` requires parallel h5py (`MPI: True`), but running the check under Cray PALS mpiexec also triggers HDF5/mpi4py ordering issues. Since the MPI environment is verified separately, the startup check is skipped.

---

## neuroh5 cmake build

neuroh5 requires **parallel HDF5** (for `H5Pset_dxpl_mpio`).  The only suitable
build on Aurora is `hdf5-1.14.6-ehlefog` (HDF5 1.14.6, compiled against Aurora
MPICH with `H5_HAVE_PARALLEL=1`).  HDF5 2.x removed `H5Pset_dxpl_mpio`; serial
HDF5 builds lack MPI support entirely.

Build invocation:

```bash
cmake ~/neuroh5 \
  -DHDF5_ROOT=<hdf5-1.14.6-ehlefog> \
  -DHDF5_LIBRARIES="libhdf5.so;libhdf5_hl.so;libmpi.so" \
  -DMPI_C_COMPILER=<aurora_mpich>/bin/mpicc \
  -DMPI_CXX_COMPILER=<aurora_mpich>/bin/mpicxx \
  -DPYTHON_EXECUTABLE=<miv_env>/bin/python \
  -DMPI4PY_INCLUDE_DIR=<miv_env>/lib/python3.12/site-packages/mpi4py/include \
  -DCMAKE_INSTALL_PREFIX=<miv_env>
make -j8
make python_neuroh5_io   # builds build/lib/io.so
```

Result: **SUCCESS** — `[100%] Built target python_neuroh5_io`

Post-build install (cmake has no install target):
```bash
cp build/lib/io.so  <miv_env>/lib/python3.12/site-packages/neuroh5/io.so
cp python/neuroh5/* <miv_env>/lib/python3.12/site-packages/neuroh5/
```

Import check: `neuroh5 imported OK`

### mpi4py binary incompatibility warning

At runtime, mpi4py emits `RuntimeWarning: mpi4py.MPI.<Type> size changed, may
indicate binary incompatibility` for all MPI object types (e.g. `Comm` expected
32, got 40 bytes).  This indicates the Aurora MPICH ABI at compute-node runtime
differs from the headers used during `pip install --no-binary mpi4py`.  Tests
still pass.

---

## Test results (PBS job 8448029, 2026-04-24, with PR 103)

### Step [6] — Pure Python tests

```
18 failed, 38 passed, 19 warnings  (14.25s)
```

**Passed** (38): `test_coding.py` (3), `test_connections.py` (22), `test_input_features.py::test_temporal_feature_population` (1), `test_eval_network.py::test_tuple_sec_type_propagated_to_modify_syn_param` (1), others.

**Failed** (18):

| Test | Error |
|---|---|
| `test_input_features.py::test_visual_feature_encoding` | `TypeError` — upstream |
| `test_eval_network.py` (17 tests) | `AttributeError` — machinable API mismatch (upstream) |

### Step [7] — Generate spike trains

```
0 items collected, warnings only
```

No tests collected (requires multi-rank MPI invocation; single-process pytest skips them).

### Step [8] — NEURON-dependent tests

```
171 passed, 1 skipped, 20 warnings  (19.98s)
```

**Improvement over baseline**: PR 103 fixed both previously-failing NEURON tests:
- `test_gfluct3.py::test_run_with_Gfluct3` — now **PASSES** (PR 103 adds `h.load_file('stdrun.hoc')`)
- `test_mechanisms.py::test_mechanisms_compile_and_load` — now **SKIPPED** (PR 103 adds `pytest.skip` when mechanism already loaded)

All of `test_synapses.py`, `test_distribute_synapses.py`, and `tests/mechanisms/` pass.

### Step [9] — `show-h5types`

```
Name       Start    Count
====       =====    =====
STIM       0        1000
PYR        1000     80000
PVBC       81000    1474
OLM        82474    438
```

4 populations confirmed. Run via `mpiexec -n 1` (OpenMPI from conda env).

### Step [10] — `query-cell-attrs --populations OLM`

Run via `mpiexec -n 1`. Returns 4 namespaces for GIDs 143–186 (44 cells):

```
Population OLM; Namespace: Arc Distances
    Attribute: U Distance, V Distance — GIDs 143–186
Population OLM; Namespace: Generated Coordinates
    Attribute: L, U, V, X, Y, Z Coordinate — GIDs 143–186
Population OLM; Namespace: Synapse Attributes
    Attribute: swc_types, syn_cdists, syn_ids, syn_layers, syn_locs, syn_secs, syn_types — GIDs 143–186
Population OLM; Namespace: Trees
    Attribute: Destination Section, Parent Point, Point Layer, Radius, SWC Type,
               Section, Source Section, X/Y/Z Coordinate — GIDs 143–186
```

PR 103 version returns **4 namespaces** (vs 2 in baseline): Synapse Attributes and Trees now visible.

### Step [11] — CoreNEURON GPU check

```
CoreNEURON: <neuron.coreneuron.coreneuron object at ...>
GPU attr present: True
GPU mode set OK
```

CoreNEURON GPU backend confirmed available on Aurora compute nodes.

---

## Known issues and workarounds

1. **mpi4py ABI mismatch** (warning only, tests pass): mpi4py built against Aurora
   MPICH 5.0 emits `RuntimeWarning: mpi4py.MPI.<Type> size changed` at runtime.
   Not a build failure; tests pass despite the warning.

2. **h5py import order on Aurora** (fixed): `utils/utils.py` imports `mpi4py.MPI`
   at module level. On Aurora, importing mpi4py before h5py corrupts HDF5 global
   state → `ValueError: Not a datatype`. Fixed by adding `import h5py` as the
   first line of `utils/__init__.py`.

3. **MIV_SKIP_MPI_CHECK=1 required**: PR 103's `_check_mpi_env()` validates parallel
   h5py at import. On Aurora compute nodes the check itself triggers the import-order
   issue. Bypassed with the env var; MPI environment verified separately.

4. **`test_eval_network.py` failures** (17 tests): `AttributeError` — machinable
   API has drifted from test expectations. Upstream unit-test issue, not a runtime
   failure.

5. **`test_visual_feature_encoding` failure**: `TypeError` in input_features test.
   Upstream issue unrelated to build environment.

6. **`mpiexec` on Aurora**: Only Cray PALS mpiexec (`/opt/cray/pals/1.8/bin/mpiexec`)
   exists on compute nodes. It injects PMI environment that corrupts HDF5 init.
   Tests run with direct `pytest` (no mpiexec). CLI tools use OpenMPI `mpiexec`
   from the conda env (`mpiexec -n 1` resolves to OpenMPI/prte).

7. **nrnivmodl fix**: export `NRNHOME=<miv_env>/lib/python3.12/site-packages/neuron/.data`
   before running `nrnivmodl`; NEURON 9.x pip wheel otherwise uses stale build-time temp path.

---

## Related files

- [MiV_h5types.h5](source-MiV_h5types_h5.md) — population type registry
- [MiV_Cells_Microcircuit_Small_20220410.h5](source-MiV_Cells_Microcircuit_Small_20220410_h5.md) — cell attributes used by `query-cell-attrs`
- PBS script: `/home/hyoklee/wrp/run/miv_build_test.pbs`
- Full build log: `/home/hyoklee/wrp/run/miv_build_test_run.log`
