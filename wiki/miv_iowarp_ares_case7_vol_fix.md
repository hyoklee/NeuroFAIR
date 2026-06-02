# Fixing the HDF5 VOL CTE adapter for case 7 — A/B/C resolved, read-path measured (ares)

The [first VOL evaluation](miv_iowarp_ares_case7_vol.md) found three blockers
that made a case-7 measurement impossible with clio-core's HDF5 VOL connector:
**(A)** unbuilt / CMake demands HDF5≥2.0; **(B)** no `H5PLget_plugin_*` export
so it can't load without modifying MiV; **(C)** `dataset_read` served data only
from CTE blobs (no native fallback, hyperslab ignored), so reads of pre-existing
native files returned zero-filled buffers.

This page records **fixing A, B, and C across the relevant repos**, validating
the connector end-to-end, and the read-path performance it yields. All three are
resolved; the connector now reads pre-existing native HDF5 correctly through the
CTE read-through cache. Two further findings emerged: neuroh5 needed its own port
to run under any non-native VOL, and one deeper passthrough-VOL gap (link
iteration) still blocks full case-7 init.

## Branches

| repo | branch | what |
|---|---|---|
| `iowarp/clio-core` (`~/core.iowarp`) | `vol-cte-case7-fix` | fixes A, B, C + two crash fixes in `adapter/hdf5_vol/iowarp_vol.cc` |
| `hyoklee/neuroh5` (`~/neuroh5`) | `vol-cte-case7-fix` | deprecated v1 → VOL-compatible v2 HDF5 APIs (9 sites) |
| `hyoklee/NeuroFAIR` | `vol-cte-case7-fix` | this page |

## A — builds against the MiV HDF5

`adapter/hdf5_vol/CMakeLists.txt` floor lowered `2.0.0 → 1.14.0` and option help
text updated. The VOL API the source targets (`H5VL_VERSION 3`, the multi-dataset
read/write callback signatures) is already ABI-stable in 1.14.x. Built against the
**miv env parallel HDF5 1.14.6** (`~/mc3/envs/miv`, the one neuroh5 links) via
`~/bin/build_vol.sh` → `libiowarp_hdf5_vol.so` (240 KB). Verdict: the ≥2.0 floor
was bogus.

## B — loads with no application change

Added the `H5PLget_plugin_type` / `H5PLget_plugin_info` exports. The connector now
loads via the stock environment mechanism:

```
export HDF5_PLUGIN_PATH=$CORE_INSTALL/lib
export HDF5_VOL_CONNECTOR=iowarp
```

Validated by **unmodified h5py** reading through it.

## C — correct reads of pre-existing native files

`dataset_read` is now a **read-through cache**: whole, independent reads serve
from the CTE tier on a hit (`chunk_0` size > 0) and fall back to the native VOL on
a miss, staging the buffer for next time. Hyperslab/point selections and
collective MPI-IO transfers pass through to native (the blob cache is keyed by
whole-dataset linear offset and can't represent partial/collective transfers);
`dataset_write` got the same guard. Added `datatype_cls` passthrough
(commit/open/get/specific/close) for MiV's committed `/H5Types`.

Two crash fixes were required to make it actually run:

1. **Client bootstrap** — when HDF5 `dlopen()`s the connector there is no
   LD_PRELOAD constructor, so the CTE client singleton was unbound and the first
   tag op segfaulted. Added a lazy `CLIO_CTE_CLIENT_INIT()` (once) in
   `get_cte_client()`.
2. **Dataset mis-cast (fatal)** — h5py opens datasets via `H5Oopen`, so the
   generic `object_open`/`wrap_object` callbacks returned a bare `iowarp_obj_t`;
   HDF5 then routed `dataset_read/close` to it and cast it to the larger
   `iowarp_dataset_t`, reading `std::vector` members out of bounds → "double free
   or corruption", SIGSEGV in `iowarp_dataset_close` (crashed even pure
   passthrough reads). Centralised dataset wrapping in `make_dataset_wrapper`;
   `object_open`/`wrap_object` now dispatch on object type; a `cacheable` flag
   disables the blob cache when the dataset path is unknown.

### Validation (single process, `HDF5_VOL_CONNECTOR=iowarp`, unmodified h5py)

- Pre-existing native file, whole read → **correct data** (`[0 1 2 … 99997 99998
  99999]`), not zeros — the decisive C fix.
- Hyperslab read (`d[10:20]`) → correct (native passthrough).
- Compound committed-type dataset (`/H5Types/Populations`) → md5 matches native.
- Cold read (miss → native + stage) 7.6 ms; warm read (CTE hit) 4.4 ms.

(h5py's `visititems`/`.dtype` introspection hits `H5Ovisit_by_name1` /
"not a VOL connector ID" — deprecated h5py-only paths HDF5 restricts to the
native VOL; neuroh5 reads by explicit path, so this is not on the case-7 path.)

## neuroh5 had to be ported too (`~/neuroh5` branch)

neuroh5's own C++ API aborts under *any* non-native VOL because it uses
deprecated version-1 HDF5 APIs that HDF5 1.14 hard-restricts to the native VOL:

```
H5Literate1 is only meant to be used with the native VOL connector
  → assert in hdf5::group_contents while enumerating /Projections
```

Ported 9 call sites: `H5Literate → H5Literate2` (callbacks `H5L_info_t →
H5L_info2_t`; they use only the link name), `H5_ITER_NATIVE → H5_ITER_INC`,
`H5Gget_num_objs → H5Gget_info`. Behaviour under the native VOL is unchanged.
After this, `read_population_ranges` succeeds through the VOL; iteration advances
to the next gap below.

## Remaining blocker for full case-7 init: passthrough link iteration

With neuroh5 on v2 APIs, `read_projection_names` now reaches `H5Literate2`, which
still fails through the connector:

```
H5VLgroup_close(): not a VOL connector ID            (Invalid arguments)
H5Literate2(): synchronous link iteration failed
  … H5VL__native_link_specific(): error iterating over links
```

`iowarp_link_specific` forwards `H5VL_LINK_ITER` verbatim to the native VOL
without the object re-wrapping the HDF5 passthrough-VOL pattern requires (a
trampoline that wraps the under-object handed to the iteration callback). This is
a **fourth issue, separate from A/B/C** — a known but non-trivial
passthrough-completeness gap, and neuroh5 exercises more such paths downstream
(references, collective MPI-IO through the VOL, variable-length types). Completing
it is the next required step for a full optimize-network run.

## Read-path measurement — does clio-core beat baseline?

What *is* measurable today: a clean A/B of the same whole-dataset reads, native
HDF5 vs the now-working CTE VOL read-through cache (`~/bin/vol_bench.sh`, 256 MB
`i8` dataset, 6 reads, single node, RAM tier):

| backend | median / read | first | warm-min | throughput |
|---|---:|---:|---:|---:|
| **native HDF5** (page cache) | **190.6 ms** | 172.6 | 189.7 | ~1.34 GB/s |
| **IOWarp CTE VOL** (read-through) | **249.3 ms** | 245.7 | 235.6 | ~1.03 GB/s |

**CTE VOL is ~31% slower than native** for a page-cache-resident read. The cause
is the same one the [POSIX-adapter studies](miv_iowarp_ares_case7_feasibility.md)
found: on ares the native data already lives in the page cache (RAM speed), so the
CTE tier's per-1 MB-chunk SHM copy + IPC round-trips add overhead instead of
saving I/O. The warm CTE hit (235–249 ms) never undercuts native (190 ms). The
micro-smoke's 4.4 ms < 7.6 ms was cold-vs-warm on a tiny array, not CTE-vs-native;
on a like-for-like read the verdict matches every prior case: **no measurable CTE
benefit on ares**, where there is no slow shared tier for CTE to hide.

## Bottom line

- **A, B, C are fixed and validated.** The HDF5 VOL CTE connector now builds
  against the MiV HDF5, loads with no app change, and — the decisive fix — reads
  pre-existing native files **correctly** (read-through cache + native fallback),
  where before it returned zeros and crashed. Two additional crash bugs (client
  bootstrap, dataset mis-cast) were fixed along the way.
- **neuroh5 was ported** off native-VOL-only deprecated APIs so it can run under a
  custom VOL at all.
- **Full case-7 init is not yet reachable**: it needs the connector's
  passthrough link-iteration wrapping (a 4th, deeper issue) and likely more
  neuroh5-feature coverage after it.
- **Read-path performance: CTE VOL ≈ 31% slower than native HDF5 on ares**
  (page-cache-resident data) — consistent with all prior POSIX-adapter findings.
  clio-core's CTE does not improve over the native baseline here because ares has
  no slow shared storage tier for the cache to win back.

## Related pages

- [Case 7 VOL adapter — pre-fix evaluation](miv_iowarp_ares_case7_vol.md)
- [Case 7 feasibility synthesis (POSIX adapter)](miv_iowarp_ares_case7_feasibility.md)
- [perf-clio-core](perf-clio-core.md) — where CTE's measured I/O wins come from (slow tier + re-read)

## Artifacts

- clio-core: `~/core.iowarp` branch `vol-cte-case7-fix`; build `~/bin/build_vol.sh`
- neuroh5: `~/neuroh5` branch `vol-cte-case7-fix`
- run harness: `~/bin/run_case7_vol.sbatch` (adds `IOWARP=2` VOL mode)
- tests/bench: `~/bin/vol_smoke.sh`, `vol_path_sanity.sh`, `vol_n5_api.sh`, `vol_bench.sh`
