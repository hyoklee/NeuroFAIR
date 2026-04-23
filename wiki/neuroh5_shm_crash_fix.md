# neuroh5 Shared-Memory Crash Fix — Polaris

**Date**: 2026-04-23  
**Platform**: Polaris (NVIDIA A100 GPU, Cray MPICH 9.0.1)  
**Final job**: 7097644  
**Status**: **RESOLVED** — 12/12 parallel I/O tests pass  
**Fork**: https://github.com/hyoklee/neuroh5  
**Branch**: `fix/polaris-gpu-alltoallv-int-overflow`  
**PR**: https://github.com/iraikov/neuroh5/pull/19  
**Commit**: `0acad9d`

---

## Root Cause

`scatter_read_trees` and related scatter functions in neuroh5 call
`alltoallv_vector<char>` → `MPI_Ialltoallv` → `MPI_Wait`.  On Polaris GPU
compute nodes the Cray MPICH cray\_common SHM transport uses
`process_vm_readv(2)` to copy large messages between processes on the same
node (CMA rendezvous protocol).

The crash is triggered by **integer overflow in `calculate_chunk_sizes`**:

```cpp
// chunk_info.hh (before fix)
chunk.sdispls[i] = static_cast<int>(full_sdispls[i] + chunk_start);
```

For large datasets such as `PYR_forest_compressed.h5` (80 k cells,
~3 GB serialized), `full_sdispls[3]` exceeds `INT_MAX = 2 147 483 647`.
The `static_cast<int>` wraps to a large negative value.  MPI_Alltoallv then
tries to access `sendbuf + negative_displacement`, landing at an unmapped or
CUDA-managed virtual address.  The CMA path calls `process_vm_readv` for that
invalid address → `EFAULT (Bad address)` → SIGABRT.

**Confirming evidence**: PYR scatter crashed at alltoallv iteration **860**
(chunk\_start = 860 × 65 536 = 56.3 MB).  At that point
`full_sdispls[3] + chunk_start ≈ 2.09 GB + 56 MB > INT_MAX`.

Smaller datasets that fit within 2 GB total (OLM: 438 cells, PVBC: 1 474
cells) never overflow and always passed.

---

## Fix — `alltoallv_template.hh`

Replace `MPI_Ialltoallv` with explicit `MPI_Isend` / `MPI_Irecv` using
`size_t` pointer arithmetic, eliminating the `int` displacement entirely.
Self-sends (rank → rank) are handled with `std::memcpy`.

```
File: ~/neuroh5/include/mpi/alltoallv_template.hh
```

Key changes:
- Remove `MPI_Ialltoallv` call
- Loop over peer ranks; post `MPI_Irecv` then `MPI_Isend` per chunk
- Use `&sendbuf[sdispls[i] + chunk_start]` directly (no int cast)
- Handle self-send (`i == myrank`) with `std::memcpy`
- Chunk size controllable via `NEUROH5_CHUNK_SIZE` env var (default 1 GB)

```
File: ~/neuroh5/include/data/chunk_info.hh
```

```cpp
inline size_t get_chunk_size() {
    const char* env = std::getenv("NEUROH5_CHUNK_SIZE");
    if (env != nullptr) { size_t v = std::strtoull(env,nullptr,10); if(v>0) return v; }
    return 1ULL << 30;  // 1 GB default
}
```

---

## Test Results (job 7097644, 4 ranks, NEUROH5_CHUNK_SIZE=65536)

| Test | Status | Time |
|---|---|---|
| scatter_read_trees OLM_forest.h5 | **PASS** | 0.12 s |
| scatter_read_trees PVBC_forest.h5 | **PASS** | 0.65 s |
| scatter_read_trees PYR_forest_compressed.h5 | **PASS** | 303.84 s |
| scatter_read_cell_attributes PYR (Microcircuit_Small) | **PASS** | 0.01 s |
| scatter_read_cell_attributes OLM | **PASS** | 0.01 s |
| scatter_read_cell_attributes PVBC | **PASS** | 0.01 s |
| scatter_read_cell_attributes STIM | **PASS** | 0.00 s |
| scatter_read_graph (Microcircuit_Small) | **PASS** | 0.08 s |
| pop names (cells + connections) | **PASS** | 0.00 s |
| scatter_read_cell_attrs OLM_forest_syns | **PASS** | 0.22 s |
| scatter_read_cell_attrs PVBC_forest_syns | **PASS** | 1.78 s |
| **Total** | **12/12 PASS** | |

---

## Environment

```
NEUROH5_CHUNK_SIZE=65536          (64 KB per alltoallv round)
LD_LIBRARY_PATH: NVIDIA HDF5 1.14.3.7 + NVIDIA MPICH 9.0.1 + NVHPC libs
mpi4py: 3.1.4 (rebuilt from source against libmpi_nvidia.so.12)
neuroh5: rebuilt with alltoallv_template.hh + chunk_info.hh fix
```

mpi4py was also rebuilt from source (mpi4py-3.1.4 with patched `mpidistutils.py`)
against the NVIDIA Cray MPICH variant (`libmpi_nvidia.so`) to match the
neuroh5 build, avoiding the secondary MPI ABI mismatch from the GNU variant.

---

## Failed MPI bypass attempts (for reference)

| Approach | Outcome |
|---|---|
| `MPICH_CH4_SHM_ENABLE=0` | Variable not recognized |
| `MPICH_SHM_BYPASS=1` | Variable not recognized |
| `MPICH_NO_LOCAL=1` | OFI VNI domain init failure (`Function not implemented`) |
| `MPICH_SMP_SINGLE_COPY_MODE=NONE` | Rank 0 SIGSEGV in MPICH fallback path |
| `MPICH_CH4_XPMEM_ENABLE=0` | No effect on cray_common CMA |
| `MPICH_ALLTOALL_SHM_PER_RANK=0` | Does not affect `MPI_Alltoallv` |
| `CUDA_VISIBLE_DEVICES=""` | No effect |
| `NEUROH5_CHUNK_SIZE=65536` alone | Deferred crash to int-overflow iteration |

---

## Files modified

- `~/neuroh5/include/mpi/alltoallv_template.hh` — P2P implementation, self-copy
- `~/neuroh5/include/data/chunk_info.hh` — `NEUROH5_CHUNK_SIZE` env-var, `get_chunk_size()`
- `~/bin/miv_polaris_test.pbs` — `NEUROH5_CHUNK_SIZE=65536`, NVIDIA MPICH LD paths
- `~/bin/miv_neuroh5_io_test.py` — test API fixes for scatter_read_cell_attributes

---

## Upstream Contribution

- Fork: https://github.com/hyoklee/neuroh5 (branch `fix/polaris-gpu-alltoallv-int-overflow`)
- PR submitted to upstream: https://github.com/iraikov/neuroh5/pull/19
- Commit: `0acad9d fix: replace MPI_Alltoallv with P2P sends to fix int overflow on large datasets`

## Related

- Full Polaris build/test report: `~/NeuroFAIR/wiki/miv_polaris_build_test.md`
- PR #103 test results: `~/NeuroFAIR/wiki/miv_pr103_polaris_test.md`
