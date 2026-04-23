# MiV-Simulator Build and Test on Polaris

**Date**: 2026-04-23  
**Platform**: Polaris (NVIDIA A100 GPU, Cray MPICH 9.0.1, NVHPC 25.5)  
**Jobs**: 7097199, 7097203, 7097206, 7097208, 7097215  
**Source repos**: `~/MiV-Simulator` (v0.3.0), `~/neuroh5` (v0.1.18)  
**Data**: `/lus/grand/projects/gpu_hack/iowarp/neuroh5/`

---

## Build Environment

| Component | Version / Path |
|---|---|
| Python | 3.11.7 (cray-python via venv with `--system-site-packages`) |
| cray-hdf5-parallel (build) | 1.14.3.7/gnu/12.3 (GNU variant, links `libmpi_gnu_123.so`) |
| cray-mpich (build) | 9.0.1/ofi/gnu/12.3 (`libmpi_gnu_123.so`) |
| mpi4py | 3.1.4 (inherited from cray-python, links `libmpi_gnu_123.so`) |
| numpy | 2.4.4 |
| h5py | 3.16.0 (serial wheel — parallel build fails, see Issues) |
| NEURON | 9.0.1 (pip wheel) |
| neuroh5 cmake | gcc/g++ 7.5.0, GNU HDF5 1.14.3.7, GNU MPICH 9.0.1 |
| Venv | `/lus/grand/projects/gpu_hack/iowarp/miv_polaris/venv` |

---

## GPU Environment

```
NVIDIA A100-SXM4-40GB, driver 570.124.06, 40960 MiB  (×4 per node)
```

---

## Import Checks (all pass)

```
OK  mpi4py: 3.1.4
OK  h5py: 3.16.0
OK  numpy: 2.4.4
OK  neuroh5.io: ok
OK  miv_simulator: ok
OK  neuron: 9.0.1
```

---

## Test Results

### [3] Parallel neuroh5 I/O (4 MPI ranks) — FAIL

All `scatter_read_*` functions crash during the inter-rank data distribution phase:

```
process_vm_readv: Bad address
Assertion failed in cray_common/cray_common_memops.c:446
→ PMPI_Wait → neuroh5::mpi::alltoallv_vector<char>
```

**Root cause**: The GNU Cray MPICH variant (`libmpi_gnu_123.so`) uses a Cray custom shared-memory transport (`cray_common`) for intra-node MPI messages. This transport uses `process_vm_readv(2)` to copy memory across process virtual address spaces. On Polaris compute nodes this call fails with `EFAULT (Bad address)`, likely due to kernel security policy or address-space layout differences with the GPU-enabled job environment.

**Note**: `MPICH_CH4_SHM_ENABLE=0` and `MPICH_SHM_BYPASS=1` do not disable this transport in cray-mpich 9.0.1.

**Workaround path**: Rebuild mpi4py from source against the NVIDIA Cray MPICH variant (`libmpi_nvidia.so`). This requires resolving the nvc compiler flag issue: `mpicc` from `nvidia/23.3` wraps `nvc` which rejects the `-march=x86-64` flag added by pip. Fix: pass `CFLAGS=-march=x86-64-v3` or use gcc as the CC for the mpi4py C extension while linking against the NVIDIA MPICH.

**Single-rank success**: `show-h5types` (via `mpiexec -n 1`) reads and lists all populations correctly, confirming that single-rank neuroh5 I/O is functional.

### [4a] miv-simulator pure Python tests — 62/63 PASS

| Test file | Result |
|---|---|
| `test_connections.py` | 32/32 PASS ✓ |
| `test_distribute_synapses.py` | 30/30 PASS ✓ |
| `test_input_features.py::test_temporal_feature_population` | PASS ✓ |
| `test_input_features.py::test_visual_feature_encoding` | **FAIL** |
| `test_coding.py` | Collection error (skipped) |

**FAIL details:**
- `test_visual_feature_encoding`: `TypeError: FeatureSpace.__init__() got unexpected keyword 'dimensions'` — upstream API change in miv-simulator between test and implementation.
- `test_coding.py`: `AttributeError: np.float_ was removed in NumPy 2.0` — miv_simulator/coding.py uses deprecated NumPy API; venv has NumPy 2.4.4.

### [4b] miv-simulator synapses + eval_network — 143/160

| Test file | Result |
|---|---|
| `test_synapses.py` | **142/142 PASS** ✓ |
| `test_eval_network.py` | 1/18 PASS, 17 FAIL |

**eval_network FAIL detail**: All 17 failures share the same root cause:
```
AttributeError: module 'miv_simulator' has no attribute 'eval_network'
```
The test uses `unittest.mock.patch('miv_simulator.eval_network.xxx')` but the `eval_network` module is not exposed as a top-level attribute of `miv_simulator`. This is a test/package structure mismatch (same as Aurora result).

### [5] CoreNEURON GPU backend — PASS ✓

```
CoreNEURON: <neuron.coreneuron.coreneuron object at ...>
GPU attr present: True
GPU mode set: True
```

NEURON 9.0.1 (pip wheel) confirms CoreNEURON GPU backend is available on Polaris A100 nodes. Full GPU-accelerated simulation requires compiling NEURON MOD mechanisms (`nrnivmodl`), which fails for the pip wheel (hardcoded build-time path in `nrnmech_makefile`).

### [6] MiV CLI tools (single-rank)

```
show-h5types → MiV_h5types.h5:
  Name       Start    Count
  STIM       0        1000
  PYR        1000     80000
  PVBC       81000    1474
  OLM        82474    438
```

`show-h5types` works correctly via `mpiexec -n 1`.  
`query-cell-attrs` fails with MPI_Barrier error when run with 2 ranks (same intra-node SHM issue).

---

## Known Issues

| # | Issue | Severity | Fix |
|---|---|---|---|
| 1 | GNU Cray MPICH `cray_common` SHM `process_vm_readv` fails | Blocks all multi-rank neuroh5 I/O | Rebuild mpi4py against NVIDIA MPICH variant |
| 2 | `PYR_forest.h5`: `Attribute Pointer` dataset has 0 elements | H5Dread bounds error | Use `PYR_forest_compressed.h5` instead |
| 3 | h5py parallel build fails | Minor (serial wheel works) | Fix numpy.distutils/msvccompiler conflict |
| 4 | `np.float_` removed in NumPy 2.0 | `test_coding.py` collection fails | miv-simulator upstream fix needed |
| 5 | `miv_simulator.eval_network` not a top-level attribute | 17 test_eval_network.py tests fail | Package structure / `__init__.py` export |
| 6 | `FeatureSpace(dimensions=...)` API changed | 1 test_input_features test fails | Upstream fix needed |
| 7 | NEURON `nrnivmodl` embeds build-time temp path | Can't compile MOD files | Build NEURON from source on Polaris |

---

## Build Notes

### Critical: MPI library variant matching

neuroh5/io.so must link the **same** Cray MPICH variant as mpi4py, or MPI initialization isn't shared:

```
cray-python mpi4py → libmpi_gnu_123.so.12  (GCC variant)
cray-hdf5-parallel/nvidia → libmpi_nvidia.so.12  (wrong — MPI_Init not shared)
cray-hdf5-parallel/gnu/12.3 → libmpi_gnu_123.so.12  (correct match)
```

neuroh5 cmake flags used:
```bash
cmake ~/neuroh5 \
  -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ \
  -DHDF5_ROOT=/opt/cray/pe/hdf5-parallel/1.14.3.7/gnu/12.3 \
  -DMPI_C_COMPILER=/opt/cray/pe/mpich/9.0.1/ofi/gnu/12.3/bin/mpicc \
  -DMPI_CXX_COMPILER=/opt/cray/pe/mpich/9.0.1/ofi/gnu/12.3/bin/mpicxx \
  -DPython_EXECUTABLE=$VENV/bin/python3 \
  -DMPI4PY_INCLUDE_DIR=/opt/cray/pe/python/3.11.7/lib/python3.11/site-packages/mpi4py/include \
  -DBUILD_TESTS=OFF
```

Note: `nvc++` (NVHPC) does not accept `-Wno-unknown-pragmas`. Must use `gcc/g++`.

### h5py build failure (documented for reference)

Parallel h5py build fails due to:
1. h5py ≥ 3.13 requires mpi4py ≥ 4.0; building mpi4py from source fails because `mpicc` (wrapping `nvc`) rejects `-march=x86-64`.
2. With `--no-build-isolation`, `numpy.distutils` tries to import `distutils.msvccompiler` (removed from newer setuptools).
3. **Mitigation**: Use serial h5py from PyPI wheel. neuroh5's C extension accesses HDF5 directly (C API) and doesn't use h5py for parallel I/O.

---

## Build and Test Scripts

- Build script: `~/bin/miv_polaris_build.sh`
- PBS test script: `~/bin/miv_polaris_test.pbs`
- neuroh5 I/O test: `~/bin/miv_neuroh5_io_test.py`
- Raw PBS log: `~/NeuroFAIR/wiki/miv_polaris_test.log`
- Raw results: `/lus/grand/projects/gpu_hack/iowarp/miv_polaris/results/`

---

## Related files (data)

- [source-MiV_Cells_Microcircuit_Small_20220410_h5.md](source-MiV_Cells_Microcircuit_Small_20220410_h5.md)
- [source-MiV_Connections_Microcircuit_Small_20220410_h5.md](source-MiV_Connections_Microcircuit_Small_20220410_h5.md)
- [Aurora results (comparison)](source-MiV_Simulator_build_test.md)
