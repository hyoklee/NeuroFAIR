# MiV case 7 from clio-fs via clio-core's **MPI-IO adapter** (ares, `dev` `5ac2747d` + local patch)

Follow-up to [case 7 with every input in clio-fs](miv_iowarp_ares_case7_clio_fs.md).
That run reached clio-fs through Open MPI's OMPIO + the POSIX interceptor and
needed an OMPIO knob to avoid an un-intercepted `preadv`. clio-core also has an
**MPI-IO adapter** (`libclio_cte_mpiio.so`, `CLIO_CTE_ENABLE_MPIIO_ADAPTER`),
which intercepts `MPI_File_*` directly and hands `clio::` files to the clio-fs
client, never touching OMPIO. Same workload: `optimize-network`, 13 GLP initial
evaluations, 1 node, `mpirun -np 17`, inputs staged into `…/clio::datasets`.

**Result: out of the box the adapter cannot serve parallel HDF5 at all. It
crashes at file open, and it ignores file views, so it would return wrong
data on neuroh5's reads without any error. With a local patch (~470-line
diff) MiV case 7 runs fully through it with bit-identical objectives at
+5.0 %, the same cost as the OMPIO path (+4.8 %). The per-operation cost is
in clio-fs itself, not in the interception route.**

> **Note (2026-10-20).** The "identical objectives" check on this page compared all-zero vectors: the `tstop=200` timing config simulates a silent network (STIM input starts at 256–407 ms). The timings are unaffected; the byte-level read checks are the correctness evidence. See [case 7 on the updated stack](miv_iowarp_ares_case7_1020.md#the-timing-workload-simulates-a-silent-network).

## Build

`~/bin/build_core_dev_miv.sh` now takes `SRC=`, `BUILD=`, `PREFIX=` and
`EXTRA_CMAKE=` overrides.

- Stock: `EXTRA_CMAKE=-DCLIO_CTE_ENABLE_MPIIO_ADAPTER=ON` →
  `~/core_dev_miv_mpiio/install`. The build detects Open MPI 5.0.10, the same
  version as the miv env.
- Patched: `git archive 5ac2747d` → `~/core_dev_miv_mpiio_patch/src`, so the
  daily-updated source tree is untouched → `~/core_dev_miv_mpiio_patch/install`.
  The patch is in `~/core_dev_miv_mpiio_patch/mpiio_hdf5.patch`.

## Gaps in the stock adapter

The adapter wraps open/close/seek/get_position, the read/write family
(`_at`, `_all`, `_shared`, `_ordered`, `i*`) and `sync`. Parallel HDF5 1.14.6
imports 14 `MPI_File_*` symbols (`nm -D libhdf5.so.310`):

| gap | effect | found by |
|---|---|---|
| `MPI_File_get_info`, `get_size`, `set_view`, `set_size`, `get/set_atomicity`, `delete` not wrapped | a `clio::` handle is a heap token, not an `MPI_File`, so real Open MPI dereferences it: **segfault in `H5Fopen`** (`MPI_File_get_info` → `opal_info_dup_public`) | smoke job 24625 (h5py `driver="mpio"`, np=1 and 4) |
| **file views and derived datatypes ignored** (stated in the source header: "a datatype only contributes its size") | neuroh5's collective filtered-chunk reads (`H5D__mpio_collective_filtered_vec_io`) set a **non-contiguous file view**. The adapter would read the wrong bytes with no error | patched adapter with a fail-loud guard: `CLIO_MPIIO_UNSUPPORTED: non-contiguous file view` (job 24627) |
| RPATH pins `~/mc3/envs/iowarp-build/lib` | preloading the adapter loads iowarp-build's `libmpi.so.40`/`libopen-pal` into MiV (same 5.0.10, but a second build) | backtrace of job 24625 |
| preload order | if `libmpi` comes before the adapter in `LD_PRELOAD`, libmpi's `MPI_File_open` wins and the adapter is silently bypassed. "Identical" results then prove nothing | job 24626: OMPIO `preadv` EBADF despite the adapter being loaded |

What works, and what fixes the last two rows:
`LD_PRELOAD=libclio_cte_mpiio.so:$CONDA_PREFIX/lib/libmpi.so.40:libclio_cte_posix.so`.
The POSIX interceptor is still needed for h5py's serial opens and for staging.

## The local patch (`mpiio_hdf5.patch`)

1. **Wraps** `get_size` (`SizeFd`), `set_size` (`FtruncateFd`), `get_info`
   (empty info), `set_info`, `get_amode`, `get/set_atomicity` (stored per
   handle), `delete` (`RemovePath`) and `set_view`. Each one falls back to
   `PMPI_*` for untracked handles.
2. **File views:** `set_view` flattens the filetype into byte blocks. The
   decoder handles named, dup, resized, contiguous, (h)vector, (h)indexed,
   (h)indexed_block and struct types, and fails loudly on anything else.
   Explicit-offset reads and writes (`*_at`, `*_at_all`, `i*_at`) map the data
   stream `[offset·etype, +n)` through the tiled view to file segments. The
   adapter reads the covering span in **one** `PreadFd` when it is dense
   (≤ 2n + 4 MiB, i.e. data sieving), and otherwise reads segment by segment.
3. **Derived memory datatypes** are scattered and gathered through the same
   flattener.

Not done: views on the sequential and shared-pointer calls. HDF5 does not use
them.

## Correctness

Smoke test (`~/bin/cfs_mpiio_smoke.sbatch`, job 24629). Every check compares
against a native read:

- h5py `driver="mpio"`, whole-file read (60 datasets, md5 `fddb49e9…`), on
  the Connections file at np=1 and np=4 and on the Cells file at np=4:
  **identical**.
- neuroh5 bulk reads (`~/bin/cfs_nh5_bulk.py`: every cell-attribute namespace
  plus trees, 748 cells) at np=1: **identical** (md5 `8b53db84…`). This path
  uses the non-contiguous views. (np=4 is omitted because this synthetic read
  fails *natively* in HDF5's filtered collective I/O.)
- Control with only the POSIX interceptor: crashes with OMPIO `preadv` EBADF.
  So the adapter is what makes the read work.
- **Provenance check** (`strace`): with the adapter there are **0** `statfs`
  calls on `clio::` paths and **0** `preadv`. With the POSIX-only path there
  are 18 `statfs` calls and an EBADF `preadv`. The adapter served the reads,
  not OMPIO.
- Digest fix found on the way: hashing neuroh5's `section_topology` (a dict)
  as a numpy object array hashes memory addresses, so the native digest
  changed run to run. The fixed digest recurses into dicts and is
  deterministic.

Full simulation: the sorted `objective …:` lines have the **same md5** in all
runs: MPI-IO adapter ×2, fresh native ×2, and the earlier native, clio-fs/POSIX
and VOL runs.

## Result — end-to-end (1 node, 13 evaluations, default OMPIO settings)

| Config | jobs (node) | reps (s) | mean | vs baseline |
|---|---|---:|---:|---:|
| baseline (native) | 24631 (comp-17), 24633 (comp-11) | 1830.3 / 1841.0 | **1835.6 s** | — |
| **clio-fs via MPI-IO adapter** (`IOWARP=5`) | 24630 (comp-16), 24632 (comp-10) | 1925.5 / 1929.4 | **1927.4 s** | **+5.0 %** |
| *(reference)* clio-fs via OMPIO + POSIX (`IOWARP=4`, gap 1 G) | 24619/24620 | 1933.2 / 1925.0 | 1929.1 s | +4.8 % vs its own baseline (1841.0) |

| phase (rank 0) | native | MPI-IO adapter | OMPIO + POSIX |
|---|---|---|---|
| cells created | 1.89–1.96 s | **1.67–1.75 s (−11 %)** | 1.71–1.76 s |
| configure 33,436 synapses for gid 10 | 8.51–9.14 s | **9.05–9.65 s (+6 %)** | 9.00–9.57 s |

There were 0 `CLIO_MPIIO_UNSUPPORTED` hits in either full run.

**Reading.** The two clio-fs routes cost the same, to within noise. One route
goes through OMPIO's `fbtl_posix` → POSIX interceptor and issues ~49 k
failing `fcntl` lock calls per file. The other goes `MPI_File_*` → adapter →
clio-fs client and issues no `fcntl` or `statfs` at all. So the earlier
hypothesis that OMPIO's failed `fcntl` calls contributed to the +4.8 % is
**ruled out**. The cost is per-operation overhead in clio-fs (many small
reads in the synapse/connectivity phase). Bulk cell reads are ~10 % faster
from the RAM tier either way. The verdict is the same as in every ares case-7
study: the workload is compute-bound and reads ~92 MB in total, so there is
nothing for clio-fs to win.

## Bottom line

1. **The stock MPI-IO adapter is not usable with parallel HDF5.** It segfaults
   at `H5Fopen` because `MPI_File_get_info` and other calls are not wrapped.
   Even with those wrapped, it ignores file views and derived datatypes, which
   neuroh5's collective reads use, so it would return wrong data without any
   error.
2. **With the patch it serves MiV correctly** (bit-identical objectives; reads
   proven to go through the adapter) and needs **no OMPIO knob**. It also
   avoids the `preadv` and `fcntl` gaps and the ROMIO `clio::` filename
   collision, because OMPIO and ROMIO never see `clio::` files.
3. **Cost +5.0 %**, the same as the OMPIO route. The interception route
   doesn't matter; clio-fs's small-read overhead does.
4. Upstream asks for clio-core:
   - wrap the remaining `MPI_File_*` calls HDF5 uses;
   - implement file views and derived datatypes, or fail loudly instead of
     ignoring them;
   - drop the iowarp-build RPATH from `libclio_cte_mpiio.so`;
   - document that the adapter must precede `libmpi` in `LD_PRELOAD`.

   **Upstreamed as [iowarp/clio-core#1272](https://github.com/iowarp/clio-core/pull/1272)**
   (2026-10-13, branch `hyoklee:mpiio-hdf5-compat` off `dev` `515598ca`). It is
   a cleaned-up rewrite of the local patch: wrapped calls go through `MpiioApi`;
   undecoded constructors, non-`native` datarep and file-pointer ops under a
   view return `MPI_ERR_UNSUPPORTED_OPERATION`. It adds 4 unit test cases
   (9 cases / 379 assertions pass; `dev` 5 / 37). The MiV smoke test was
   re-run against the PR build (job 24639): all reads identical, and `strace`
   confirms the adapter served them. The RPATH is unchanged in the PR.

## Related pages

- [Case 7 with every input in clio-fs (OMPIO + POSIX route)](miv_iowarp_ares_case7_clio_fs.md)
- [Case 7 on clio-core `dev` `5ac2747d` — POSIX vs VOL vs VFD](miv_iowarp_ares_case7_dev_1011.md)
- [clio-fs and the CLIO HDF5 VFD](concept-clio-fs-vs-hdf5-vfd.md)

## Artifacts

- PR branch: `~/src/clio-core-mpiio-pr` (`mpiio-hdf5-compat`), build
  `~/clio_mpiio_pr/build`; unit tests `~/bin/mpiio_pr_diag.sbatch`. Running
  them on ares needs `LD_LIBRARY_PATH` **without** `~/mc3/lib`, because its
  legacy `libchimaera_*_runtime.so` are auto-loaded by the runtime and
  segfault it, and needs `CLIO_SERVER_CONF` set to a config without CAE when
  CAE is not built.

- builds: `~/core_dev_miv_mpiio/install` (stock), `~/core_dev_miv_mpiio_patch/{src,install,mpiio_hdf5.patch}`;
  logs `~/logs/build_core_dev_miv_mpiio{,_patch}.log`
- harness: `~/bin/run_case7_dev.sbatch` `IOWARP=5` (needs `CORE_INSTALL` with
  `libclio_cte_mpiio.so` and `OPTCONF=optimize_network_ares_cfs.yaml`);
  submit `~/bin/submit_case7_mpiio.sh`
- smoke: `~/bin/cfs_mpiio_smoke.sbatch` (jobs 24625 stock, 24626 wrong preload
  order, 24627 guard hit, 24628/24629 view support), `~/bin/cfs_nh5_bulk.py`
- logs: `~/logs/case7_o_{mpiio,mpiio2,basem,basem2}_*.log` (24630–24633)
- data: `~/MiV-Simulator-Cases/7-optimization/results/comparison_case7_dev_1011.csv`
