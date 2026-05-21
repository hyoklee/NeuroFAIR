# MiV-Simulator GPU vs CPU Benchmark

**Date**: 2026-05-20 / 2026-05-21  
**Platform**: Polaris (NVIDIA A100-SXM4-40GB, 40 GB, 4× per node)  
**Queue**: `debug` (1h), 1 node, 1 MPI rank  
**Circuit**: Microcircuit_Small (PYR 80, PVBC 53, OLM 44, STIM 10)  
**Simulation**: tstop=500ms, dt=0.025ms, v_init=-77mV  
**PBS scripts**: `~/polaris/pbs/miv_gpu_bench_build.pbs`  
**Bench script**: `~/bin/miv_gpu_bench.py`

---

## Results (bench18, PBS 7163402)

| Mode | Wall time | Exit | Notes |
|---|---|---|---|
| **[A] Standard NEURON CPU** | **481 s** | **0 ✓** | Baseline — fully working |
| [B] CoreNEURON CPU | 254 s | 134 (SIGABRT) | Crashes inside `psolve` — see below |
| [C] CoreNEURON GPU | 225 s | 42 | GPU not enabled at build time — see below |

**Node**: x3110c0s37b1n0 (NVIDIA A100-SXM4-40GB, 40960 MiB × 4)

Consistent CPU baseline across bench13–bench18: **481–520 s** for tstop=500ms, 1 rank.

---

## Mode A: Standard NEURON CPU — Baseline

Setup (network.init including HDF5 reads + synapse cache): ~240 s  
Simulation (psolve 500ms): ~240 s  
Total: ~481 s

This is the authoritative CPU baseline for the Microcircuit_Small circuit.

---

## Mode B: CoreNEURON CPU — SIGABRT in psolve

CoreNEURON runs ~2× faster for setup (~15 s) but crashes inside `psolve` at
approximately the tstop wall-clock equivalent (~240 s of simulation ≈ 254 s total).

**Root cause**: The pip-installed NEURON 9.0.1 CoreNEURON backend is incompatible
with one or more of the circuit's ion-accumulation mechanisms (`cad`, `CaconcPR`,
`K_conc`, `Na_conc`, `kahpPR`, `ksPR`, etc.). A C-level assertion fires inside
CoreNEURON's `psolve` — confirmed via `strings` on the core dump
(`Assertion failed.` with no mechanism-specific message).

**What was ruled out** (bench13–bench18 debugging iterations):
- Not an HDF5 output conflict (`network.run(output=False)` still crashed)
- Not a post-simulation timing call (`env.pc.psolve()` direct call still crashed)
- Not a mechanism compilation issue (`special-core` builds and loads correctly)
- Not a path/cache bug in `compile_and_load` (positional-arg keyword fix applied)

**Next steps to enable CoreNEURON CPU**:
1. Test with a simpler circuit (fewer/different ion channels) to isolate which mechanism triggers the assertion
2. Build NEURON from source with debug symbols to get a symbolic stack trace from the core dump
3. Try NEURON 9.0.0 or another version — this may be a version-specific CoreNEURON bug

---

## Mode C: CoreNEURON GPU — Build-time limitation

Error: `GPU support was not enabled at build time but GPU execution was requested.`

The NEURON 9.0.1 wheel in the venv was compiled without `CORENRN_ENABLE_GPU=ON`.
Even though `nrnivmodl -coreneuron` successfully creates `special-core`, the
CoreNEURON runtime binary itself lacks GPU code paths.

**To enable GPU mode**, NEURON must be built from source:
```bash
cmake ... -DCORENRN_ENABLE_GPU=ON \
          -DCMAKE_CXX_COMPILER=nvc++ \
          -DCMAKE_C_COMPILER=nvc
```
This requires NVHPC (nvc++) on Polaris — the same toolchain used for neuroh5
(see `wiki/miv_iowarp_bench.md`).

---

## Bug Fixes Applied During Benchmarking

Several bugs were found and fixed in the course of getting this benchmark to run:

### neuroh5: `append_rank_attr_map` assertion (PBS 7159587–7161529)
**File**: `/home/hyoklee/neuroh5/src/data/append_rank_attr_map.cc`  
**Fix**: Replaced 7× `throw_assert(it != node_rank_map.end(), ...)` with
`if (it == node_rank_map.end()) continue;`  
**Root cause**: STIM spike input file contains 1000 gids but Small circuit has
only 10 STIM cells; gids 10–999 were not in the rank map → fatal assertion.

### neuroh5 build: cmake not found on compute nodes
**Fix**: Symlinked `~/miniconda3/bin/cmake` into the venv bin directory.  
Set `CC=gcc CXX=g++` to avoid NVHPC `-Wno-unknown-pragmas` rejection.

### MiV-Simulator: `init_input_cells` KeyError with 1 rank
**File**: `/home/hyoklee/MiV-Simulator/src/miv_simulator/network.py:1419`  
**Fix**: Added `if gid not in env.artificial_cells[pop_name]: continue`  
**Root cause**: With 1 MPI rank, `pc.gid_exists()` returns True for all gids
including PYR gid 10, but `artificial_cells["STIM"]` only has gids 0–9.

### MiV-Simulator: `compile_and_load` positional arg bug
**File**: `/home/hyoklee/MiV-Simulator/src/miv_simulator/mechanisms.py:135`  
**Fix**: `compile(..., coreneuron=coreneuron)` (keyword arg instead of positional)  
**Root cause**: `compile()` has `return_hash` as 5th param before `coreneuron`;
passing `coreneuron` positionally → `return_hash=True` → returned hash string
instead of full path → FileNotFoundError.

---

## Environment

```
Python:   3.11.7 (cray-python venv)
NEURON:   9.0.1 (pip, no GPU support)
neuroh5:  0.1.18 (rebuilt with GCC 7.5 + append_rank_attr_map fix, PBS 7161529)
Platform: Polaris, x86_64, NVIDIA A100-SXM4-40GB
MPI:      Cray MPICH 9.0.1 (nvidia/23.3)
HDF5:     1.14.3.7 (cray parallel)
```

---

## Related

- [neuroh5 append_rank_attr_map fix](neuroh5_shm_crash_fix.md)
- [MiV-Simulator IOWarp CTE benchmark](miv_iowarp_bench.md)
- [MiV-Simulator optimization on Polaris](miv_opt_7_polaris.md)
