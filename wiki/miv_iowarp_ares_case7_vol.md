# MiV case 7 with the HDF5 **VOL** CTE adapter — can clio-core beat baseline? (ares)

> **Follow-up:** blockers A, B, and C below were subsequently **fixed** and the
> connector validated end to end — see
> [Fixing the VOL CTE adapter](miv_iowarp_ares_case7_vol_fix.md). This page is the
> pre-fix analysis that scoped the work.


Every prior case 7 study ([case7](miv_iowarp_ares_case7.md),
[case7_full](miv_iowarp_ares_case7_full.md), [caseread](miv_iowarp_ares_caseread.md),
[caseio](miv_iowarp_ares_caseio.md), [feasibility](miv_iowarp_ares_case7_feasibility.md))
drove clio-core through the **LD_PRELOAD POSIX adapter**, whose blocker #2 was
"MiV emits unprefixed paths, so the `clio::`-marker interceptor never fires."
The feasibility page named the fix: *"link against a CTE-aware HDF5 VFD/VOL so
the marker isn't needed at the path level."* This page evaluates that exact path
— the **HDF5 VOL connector** shipped in the clio-core tree
(`context-transfer-engine/adapter/hdf5_vol/iowarp_vol.{cc,h}`) — and reports why
a case-7 performance number **cannot be produced with it as implemented**.

Unlike the POSIX studies, this verdict is reached from the connector source, not
from a run: the read path is structurally incapable of serving case 7's workload,
so a run would crash on corrupt data rather than yield a wall-time comparison.

## What the VOL adapter is

`iowarp_hdf5_vol` is a **hybrid passthrough connector** (connector name `iowarp`,
class value 600, `IOWARP_VOL_CONNECTOR_VERSION 1`):

- `iowarp_file_open` opens the real file through the **native VOL**
  (`H5Pset_vol(native_fapl, H5VL_NATIVE, ...)` → `H5VLfile_open`) and attaches a
  CTE tag `hdf5:<path>` (`iowarp_vol.cc:761`+ initializer, `file_open` body).
- `iowarp_dataset_write` chunks the buffer into `chunk_size` (default 1 MB)
  pieces and `AsyncPutBlob`s each as `<dataset_path>/chunk_i` into the CTE tag.
- `iowarp_dataset_read` `AsyncGetBlob`s those same `<dataset_path>/chunk_i` blobs
  back.

So it is a **write-through blob cache**: data must first be *written through this
same connector, in-process*, before a read can find anything.

## Three layers of blocker, top to bottom

### A. Not built, and the CMake guard rejects ares' HDF5

`CLIO_CTE_ENABLE_HDF5_VOL:BOOL=OFF` in `~/core.iowarp/build/CMakeCache.txt` — the
target is never compiled (no `libiowarp_hdf5_vol.so` anywhere under `~/core*`).
Its `CMakeLists.txt` also `return()`s unless `HDF5_VERSION >= 2.0.0`, while ares
has only HDF5 **1.14.6** (conda) / 1.14.5 / 1.14.0 / 1.8.23 (modules); there is
no HDF5 2.0 on the machine.

**That guard is over-conservative, not a real wall.** HDF5 1.14.6 already defines
`H5VL_VERSION 3` (`~/mc3/include/H5VLpublic.h:36`), and the connector's callback
signatures match the 1.14.6 headers exactly — e.g. its
`iowarp_dataset_read(size_t count, void *dset[], hid_t mem_type_id[],
hid_t mem_space_id[], hid_t file_space_id[], hid_t dxpl_id, void *buf[],
void **req)` is byte-for-byte the `H5VL_dataset_class_t::read` prototype in
`~/mc3/include/H5VLconnector.h`. A `-fsyntax-only` compile against the 1.14.6 VOL
headers produced **zero HDF5-version errors** (only unrelated clio_ctp errors from
build-flag defines the CMake harness injects: `-fcoroutines`,
`-DCTP_DEFAULT_THREAD_MODEL`). So relaxing the guard to `1.14.0` would let it
build against ares' HDF5 — blocker A is removable.

### B. No H5PL export → cannot be loaded without modifying MiV

The standard way to insert a VOL connector **without touching the application** is
`HDF5_VOL_CONNECTOR=iowarp` + `HDF5_PLUGIN_PATH=...`, which requires the plugin
`.so` to export `H5PLget_plugin_type` / `H5PLget_plugin_info`. `iowarp_vol.cc`
exports **neither** (grep finds no `H5PLget_plugin_*`). It exposes only
`H5VL_iowarp_register()` → `H5VLregister_connector(&H5VL_iowarp_cls, ...)`, which
an application must call itself and then attach with `H5Pset_vol()`.

So even after building, using it means **modifying and rebuilding MiV-Simulator /
neuroh5** to register the connector and set it on every file-access FAPL —
exactly the kind of upstream change blocker #2 of the
[feasibility analysis](miv_iowarp_ares_case7_feasibility.md) called for, just
relocated from "emit `clio::` paths" to "call `H5Pset_vol`."

### C. **Decisive:** the read path can't serve pre-existing native files

This is the one that makes a case-7 measurement impossible regardless of A and B.

`iowarp_dataset_open` unconditionally sets `dset->file = find_parent_file(obj)`
for any dataset opened inside a connector-opened file. `iowarp_dataset_read` then
takes its `dataset->file != nullptr` branch, which reads **only from CTE blobs**
(`AsyncGetBlob` of `<dataset_path>/chunk_i`) and **never falls back to the native
file** on a blob miss. There is no read-through population from `under_object`.

Case 7 reads **pre-existing native HDF5 files** — `MiV_Cells_Microcircuit_*.h5`
(19 GB), `MiV_Connections_Microcircuit_*.h5` (7 GB) — created offline (Globus
staging / native writers), never written through this connector. Their
`chunk_i` blobs do not exist, so each read returns an **empty/zero-filled
buffer** that `std::memcpy`s straight into the application. neuroh5 receives all
zeros for the `/H5Types/Populations` registry, projection pointers, tree
offsets, etc., and aborts at init on the corrupt structure. That is a guaranteed
crash, **not a wall-time number.**

Two further structural gaps compound it:

- **Hyperslabs ignored.** Both read and write compute `total_size = nelem ×
  type_size` from the *whole* dataspace and stream linear `chunk_i` blobs; the
  `file_space_id[]` selection is never consulted. neuroh5 does partial,
  per-rank, often collective hyperslab reads — even a same-process write→read
  would return the wrong region.
- **Sparse callback table.** `datatype_cls` is all `nullptr`; every `optional`
  slot is `nullptr`. neuroh5's link/object iteration and compound/vlen type
  machinery would hit unimplemented callbacks.

## Why this is worse than the POSIX result, not better

| Adapter | Path interception | Reads existing native files? | Case 7 outcome |
|---|---|---|---|
| POSIX LD_PRELOAD ([feasibility](miv_iowarp_ares_case7_feasibility.md)) | only `clio::`-marked paths | yes (adapter inert, native passes through) | runs; **= NFS within 0.05%**, tier 0 bytes |
| HDF5 VOL (this page) | all opens, if registered | **no** — read path is CTE-blob-only, no native fallback | **crashes on zero-filled reads**; no number |

The POSIX adapter at least lets case 7 *run* (just with no benefit, because the
marker never matches). The VOL adapter, once actually wired in, would *break*
case 7, because its read path assumes a write-through-then-read lifecycle that
the workload doesn't follow.

## What would make a case-7 / VOL measurement possible

All four, in order:

1. **Build it** — flip `CLIO_CTE_ENABLE_HDF5_VOL=ON` and relax the
   `HDF5_VERSION >= 2.0.0` guard to `1.14.0` (the API is already compatible).
2. **Make it loadable without app changes** — add the `H5PLget_plugin_type` /
   `H5PLget_plugin_info` exports so `HDF5_VOL_CONNECTOR=iowarp` works; *or* accept
   modifying MiV/neuroh5 to call `H5VL_iowarp_register` + `H5Pset_vol`.
3. **Fix the read path to read-populate from native on blob miss** (or make it a
   true caching passthrough that delegates to `under_object` and caches on the
   side), and **honor `file_space_id` selections** so partial/collective reads
   return correct data. Without this, correctness — not performance — is the
   blocker.
4. **A workload where caching the file in CTE actually pays back** — i.e. the same
   datasets read many times within a process with the cache warm, and reads large
   relative to compute. Case 7 on the small circuit is ~0.4% I/O (compute-bound,
   per [case7](miv_iowarp_ares_case7.md)); full CA1 OOMs before init finishes
   ([case7_full](miv_iowarp_ares_case7_full.md)). Even a *correct* VOL cache
   inherits the compute-bound ceiling the POSIX studies already measured.

## Bottom line

Can clio-core's **HDF5 VOL CTE adapter** improve case 7 over the native-HDF5
baseline on ares? **Not measurable today**, for a stronger reason than the POSIX
adapter: it is unbuilt (removable), unloadable without source changes (removable),
and — decisively — its read path serves data **only from CTE blobs previously
written through the connector**, with no native-file fallback and no hyperslab
support. Pointed at case 7's pre-existing native HDF5 files it returns zero-filled
buffers and crashes neuroh5 at init, so there is no baseline-vs-clio-core wall
time to report. The adapter is a write-through-cache prototype; case 7 is a
read-existing-files workload. They don't meet until item 3 above lands upstream.

## Related pages

- [Case 7 feasibility synthesis](miv_iowarp_ares_case7_feasibility.md) — POSIX-adapter three-blocker analysis this page extends to the VOL layer
- [Case 7 small-circuit](miv_iowarp_ares_case7.md) — POSIX adapter 11% slower, compute-bound
- [Case 7 full CA1](miv_iowarp_ares_case7_full.md) — OOM/crash matrix, tier 0 bytes
- [Read-bound caseread](miv_iowarp_ares_caseread.md) — adapter-inert + forced-marker (4× slower, [#470](https://github.com/iowarp/clio-core/issues/470))
- [perf-clio-core](perf-clio-core.md) — where CTE's measured I/O wins actually come from
