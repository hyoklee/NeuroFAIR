# clio-core CTE: GPU Memory vs CPU DRAM for MiV Optimization

**Date**: 2026-04-29
**Platform**: Aurora (Intel GPU Max / Ponte Vecchio, HBM2e)
**Context**: MiV-Simulator 7-optimization, Runs 11–12 clio-core performance comparison

---

## Finding: clio-core Run 12 did not use GPU memory

The clio-core comparison (Run 12, PBS 8453684) staged HDF5 files into **CPU DRAM
(`/dev/shm`)**, not GPU HBM. Two independent reasons prevented GPU memory use:

### 1. Simulation runs on CPU, not GPU

`optimize_network.yaml` has `use_coreneuron: False`. All NEURON simulation
runs on CPU; the Intel GPU on the Aurora compute node is unused by the optimizer.

### 2. No GPU CTE backend compiled in the clio-core build

The clio-core build (`~/clio-core/build/bin/`) contains GPU memory backend
headers (`gpu_malloc.h`, `gpu_shm_mmap.h`) but no corresponding GPU shared
libraries were compiled. No GPU memory tier is available in the current build.

---

## Why GPU HBM would not help this workload anyway

| Storage tier | Bandwidth | Data path to neuroh5 |
|---|---|---|
| Lustre (baseline) | ~75 MB/s | Lustre → CPU DRAM → numpy |
| `/dev/shm` CPU DRAM (Run 12) | ~1178 MB/s | CPU DRAM → numpy |
| GPU HBM2e (hypothetical) | ~3200 GB/s | GPU HBM → PCIe → CPU DRAM → numpy |

neuroh5 and h5py read data into **CPU numpy arrays**. A GPU HBM CTE tier would
require an extra PCIe transfer (GPU HBM → CPU DRAM) before the data is usable,
making reads *slower* than `/dev/shm` for this access pattern.

GPU memory as a CTE buffer is only beneficial if the computation consuming the
data also runs on the GPU (avoiding the PCIe round-trip entirely).

---

## Measured results (Runs 11 vs 12)

| Phase | Run 11 (Lustre) | Run 12 (clio /dev/shm) | Delta |
|---|---|---|---|
| Pre-staging | — | 0.332 s | — |
| `connect_cells` | 25.54 s | 24.96 s | −0.58 s |
| **Total setup** | **26.27 s** | **25.95 s** | **−0.32 s** |

The 0.58 s saving in `connect_cells` reflects the 15.7× raw I/O speedup
(75 MB/s Lustre → 1178 MB/s `/dev/shm`) applied to the ~4 s HDF5 read
portion. NEURON synapse construction (~21 s) dominates the remaining time
and is unaffected by storage tier.

---

## Path to GPU memory benefit

To leverage Aurora GPU HBM2e for the MiV optimization:

1. **Enable CoreNEURON GPU**: set `use_coreneuron: True` in `optimize_network.yaml`
   so NEURON simulation runs on the Intel GPU (eliminating the PCIe bottleneck —
   data consumed on the same device it is stored on).

2. **Build clio-core with Intel oneAPI/SYCL GPU support**:
   ```cmake
   cmake ~/clio-core -DWRP_CORE_ENABLE_ONEAPI=ON \
       -DWRP_CORE_ENABLE_SYCL=ON \
       -DCMAKE_CXX_COMPILER=icpx
   ```
   This enables the `gpu_shm_mmap` backend so CTE can allocate blobs in HBM2e.

3. **Configure CTE with GPU HBM tier** (`cte_config.yaml`):
   ```yaml
   storage:
     - type: gpu_shm
       capacity: 64GB
       score: 2.0   # higher score = preferred over CPU DRAM
   ```

4. **Run with GPU-aware staging**: CTE loads HDF5 blobs into HBM on first access;
   CoreNEURON reads directly from HBM without PCIe transfers.

With this setup, the effective bandwidth for repeated reads would approach
HBM2e peak (~3.2 TB/s), and the simulation would also run faster on GPU.

---

## Summary

| Scenario | I/O tier | Simulation | Expected benefit |
|---|---|---|---|
| Current (Runs 1–12) | Lustre / `/dev/shm` | CPU | `/dev/shm`: 15.7× I/O speedup; 0.32 s net setup saving |
| Full clio-core GPU | GPU HBM2e | GPU (CoreNEURON) | HBM: >100× I/O; GPU sim speedup too |
| Intermediate | `/dev/shm` | GPU (CoreNEURON) | 15.7× I/O + GPU sim speedup |

For the small circuit (130 MB), `/dev/shm` is sufficient and already optimal.
GPU HBM matters most for the full CA1 circuit (26 GB) where I/O dominates,
and only if the simulation also runs on GPU.

---

## Related pages

- [clio-core CTE buffering for MiV optimization](perf-clio-core.md)
- [MiV-Simulator 7-optimization on Aurora](source-MiV_Optimizer_test.md)
