# MiV-Simulator Build and Test on Aurora

**Date**: 2026-04-22  
**Platform**: Aurora (Intel GPU, Aurora MPICH 5.0, SYCL/oneAPI)  
**Job**: PBS debug queue, `gpu_hack` allocation  
**Source repo**: `~/MiV-Simulator` (GazzolaLab/MiV-Simulator v0.3.0)  
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
| Compiler | Intel icx (oneAPI, Aurora) |

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
still pass; parallel HDF5 I/O correctness is unverified.

---

## Test results (PBS job 8445030, 2026-04-22)

### Step [6] — Pure Python tests

```
18 failed, 38 passed, 19 warnings  (16.28s)
```

**Passed** (all tests): `test_coding.py`, `test_connections.py`

**Failed**:

| Test | Error |
|---|---|
| `test_input_features.py::test_visual_feature_encoding` | `TypeError` |
| `test_eval_network.py` (17 tests) | `AttributeError` — likely machinable API mismatch |

### Step [7] — Generate spike trains

```
0 items collected, 19 warnings
```

Test module imports fine but collects no tests in the PBS single-rank environment
(likely requires `mpirun`/multi-rank invocation).

### Step [8] — NEURON-dependent tests

```
2 failed, 170 passed, 20 warnings  (19.56s)
```

**Passed**: `test_synapses.py`, `test_distribute_synapses.py`, `tests/mechanisms/` (others)

**Failed**:

| Test | Error |
|---|---|
| `test_mechanisms.py::test_mechanisms_compile_and_load` | `AssertionError` — `isfile` checks a directory path (upstream test bug) |
| `test_gfluct3.py::test_run_with_Gfluct3` | `LookupError: 'tstop' is not a defined hoc variable` |

**`test_run_with_Gfluct3` root cause**: `Gfluct3.mod` *does* compile and link
successfully.  The failure is at `h.tstop = simdur` (line 43).  In NEURON 9.x,
`stdrun.hoc` is no longer auto-loaded; `h.tstop`, `h.run()`, etc. are undefined
until `h.load_file('stdrun.hoc')` is called explicitly.  The test needs a one-line
fix upstream: add `h.load_file('stdrun.hoc')` before `h.tstop = simdur`.

**`test_mechanisms_compile_and_load` root cause**: The test calls `isfile()` on
a path that is a directory (`compiled/<hash>/`).  Should be `isdir()`.  Upstream
test bug, unrelated to the build environment.

**Fix for `nrnivmodl`** (PBS job 8445183 confirmed): export
`NRNHOME=<miv_env>/lib/python3.12/site-packages/neuron/.data` before running
`nrnivmodl`; the script then uses `${NRNHOME}/bin/nrnmech_makefile` instead of
the stale pip temp path.

### Step [9] — `show-h5types`

```
Name       Start    Count
====       =====    =====
STIM       0        1000
PYR        1000     80000
PVBC       81000    1474
OLM        82474    438
```

4 populations confirmed.  Correct option is `--input-path` (not `--input-file`).

### Step [10] — `query-cell-attrs --populations OLM`

Run via `mpiexec -n 1` (bare execution crashes with `internal_Comm_dup` unless
Aurora MPICH appears before OpenMPI in `LD_LIBRARY_PATH`).

```
Population OLM; Namespace: Arc Distances
    Attribute: U Distance — GIDs 143–186 (44 cells)
    Attribute: V Distance — GIDs 143–186 (44 cells)
Population OLM; Namespace: Generated Coordinates
    Attribute: L Coordinate — GIDs 143–186
    Attribute: U Coordinate — GIDs 143–186
    Attribute: V Coordinate — GIDs 143–186
    Attribute: X Coordinate — GIDs 143–186
    Attribute: Y Coordinate — GIDs 143–186
    (Z Coordinate also present)
```

OLM cells in the Small dataset occupy GIDs 143–186 (44 cells), storing arc
distances and generated Cartesian/curvilinear coordinates.  Correct options are
`--populations` (not `--population`) and `--input-path` (not `--input-file`).

### Step [11] — CoreNEURON GPU check

```
CoreNEURON: <neuron.coreneuron.coreneuron object at ...>
GPU attr present: True
GPU mode set OK
```

CoreNEURON GPU backend confirmed available on Aurora compute nodes.

---

## Known issues and next steps

1. **nrnivmodl broken**: NEURON 9.x pip wheel embeds the build-time temp path in
   `nrnmech_makefile`.  Workaround: build NEURON from source on Aurora, or find
   and patch `nrnmech_makefile` reference inside the installed wheel.

2. **mpi4py / OpenMPI conflict** (resolved in PBS job 8445212): The `miv` conda
   env contains OpenMPI's `libmpi.so.40` (pulled in by conda-forge dependencies).
   With `${MIV_PREFIX}/lib` first in `LD_LIBRARY_PATH`, OpenMPI is loaded instead
   of Aurora MPICH — causing `Fatal error in internal_Comm_dup` crashes.  Fix:
   prepend `${AURORA_MPICH}/lib` to `LD_LIBRARY_PATH` so the correct library is
   found first.  The `query-cell-attrs` CLI tool also requires `mpiexec -n 1` to
   set up the PMI process-management environment before MPI can initialize.

3. **`test_eval_network.py` failures**: 17 tests fail with `AttributeError`,
   suggesting `machinable` or mock API has diverged from the test expectations.
   These are unit tests with mocks; not a runtime failure.

4. **CLI option drift**: `show-h5types` and `query-cell-attrs` renamed their
   options between versions.  PBS script updated to use `--input-path` and
   `--populations`.

---

## Related files

- [MiV_h5types.h5](source-MiV_h5types_h5.md) — population type registry
- [MiV_Cells_Microcircuit_Small_20220410.h5](source-MiV_Cells_Microcircuit_Small_20220410_h5.md) — cell attributes used by `query-cell-attrs`
- PBS script: `/home/hyoklee/wrp/run/miv_build_test.pbs`
- Full build log: `/home/hyoklee/wrp/run/miv_build_test_run.log`
