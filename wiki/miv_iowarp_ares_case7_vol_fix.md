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
CTE read-through cache. Further findings emerged and were fixed: neuroh5 needed
its own port to run under any non-native VOL, a deeper passthrough-VOL gap (link
iteration, **D**) was fixed, and a second neuroh5 deprecated-API gap
(`H5Gget_objinfo`, **E**) was fixed. With D and E landed, a **full case-7 init
SLURM run** drives neuroh5's entire network-construction read path through the
VOL successfully; the run then stops at a **write-path** boundary unrelated to
neuroh5 data — dmosopt's own output file (**F**) — documented below.

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

## D. Link-iteration object wrapping (the 4th, deeper blocker) — fixed

After A/B/C, neuroh5's group enumeration still aborted under the VOL with
`H5Literate2 ... is not a VOL connector ID` (and h5py's `visititems` with the
analogous deprecated-v1 restriction). Root cause was in the connector's
**object-wrap machinery**, not the read path:

- `iowarp_get_wrap_ctx` returned the *native* wrap context directly, dropping the
  under-VOL id.
- `iowarp_wrap_object` ignored that context and **skipped `H5VLwrap_object()`**,
  storing the raw iterated object as-is.

During link/object iteration HDF5 re-wraps every visited object through the active
connector's `wrap_object`; with the object only half-wrapped, the `hid_t` handed
to the operator callback was not a valid VOL object, so `H5Literate2` aborted.
neuroh5 enumerates `/Populations`, `/Projections`, and per-cell attribute groups
exactly this way, so init could never complete.

**Fix** (modelled on the reference pass-through connector `H5VLpassthru`): a real
`iowarp_wrap_ctx_t { under_vol_id, under_wrap_ctx }`; `get_wrap_ctx` allocates it
and inc-refs the id; `wrap_object` calls `H5VLwrap_object()` against the under VOL
first, then wraps the result (dispatching datasets to the full
`iowarp_dataset_t`, non-cacheable, so the mis-cast guard from C still holds);
`unwrap_object` made symmetric; `free_wrap_ctx` releases the context and id ref.

**Validated end to end, two ways:**

1. **Synthetic** — a recursive `H5Literate2` (`H5_INDEX_NAME`/`H5_ITER_INC`) walk
   of `dentate_test.h5` that opens every child group/dataset *inside* the
   iteration callback (`~/bin/vol_iter_test.c`): **128 datasets, byte-identical
   content checksum** native vs VOL.
2. **Real neuroh5** — neuroh5's own `io.so` (`read_population_ranges`,
   `read_projection_names`, `read_population_names`,
   `read_cell_attribute_info`; `~/bin/vol_neuroh5_read.py`): **identical** output
   through the VOL vs native — population ranges (compound `/H5Types`), all 21
   projection name pairs (the `/Projections` `H5Literate2` enumeration), etc.

So the connector now drives neuroh5's enumeration path without aborting. (Note:
h5py's `visititems` still can't be used through *any* non-native VOL — it calls
the deprecated v1 `H5Ovisit_by_name1`, which HDF5 hard-restricts to the native
VOL; this is an h5py-internal choice, not a connector gap, and neuroh5 itself uses
the v2 `H5Literate2` path that now works.)

## E. neuroh5 `H5Gget_objinfo` → `H5Lexists` (full case-7 init, part 1)

With D fixed, a full case-7 init SLURM run (`run_case7_vol.sbatch IOWARP=2`,
small microcircuit) got through `read_population_names`, `read_population_ranges`,
`read_projection_names` — but then **every population failed** `make_cells` with
`unknown cell configuration type for cell type OLM` (`network.py:987`).

Root cause: `env.cell_attribute_info` came back **empty** through the VOL
(`Population attributes: {}`) while native returned the full
`{pop: {namespace: [...]}}`. neuroh5's `get_cell_attributes`
(`cell_attributes.cc`) enumerates each namespace group with `H5Literate2`, and
its callbacks (`cell_attribute_cb`, `cell_attribute_index_cb`,
`cell_attribute_index_ptr_cb`) used **`H5Gget_objinfo`** — another deprecated v1
routine HDF5 restricts to the native VOL. Under the connector it returns −1 for
every child, so every attribute was silently skipped → empty map → make_cells
can't classify any cell type.

Fix (neuroh5 `vol-cte-case7-fix`, PR #1): replace
`H5Gget_objinfo(grp, name, 0, NULL)` with
`(H5Lexists(grp, name, H5P_DEFAULT) > 0) ? 0 : -1` at all three sites. Verified:
`read_cell_attribute_info` now returns identical `{pop: {namespace: [...]}}`
through the VOL vs native on `Microcircuit_Small_coords.h5`. The re-run then
**passes make_cells** — populations are classified and it proceeds to read
trees/coordinates (`*** Reading trees for population OLM`) through the VOL.

So **neuroh5's entire case-7 network-construction read path now runs through the
VOL**: population/range/projection enumeration, cell-attribute-info namespace
enumeration, and tree/coordinate reads.

## F. Remaining boundary: dmosopt's output file (write-path, not neuroh5 data)

After E, the run stops at a **different** layer — not reading neuroh5 data, but
**dmosopt creating its own optimization-state file**:

```
dmosopt.py:2304 init_h5 -> f = h5py.File(fpath, "a")
h5f.open(name, ACC_RDWR) -> OSError: Unable to synchronously open file (open failed)
```

This is a write/bookkeeping output file, opened **read-write, create-if-absent**
(`mode="a"`). Mechanism (confirmed in isolation): h5py's `mode="a"` first tries
`H5Fopen(ACC_RDWR)` and, for the **SEC2** driver, falls back to *create* **only
on `FileNotFoundError`**. Native raises `FileNotFoundError` for a missing file;
through the VOL the same open raises a generic **`OSError`** (HDF5's `H5VL`
layer pushes its own "can't open via connector" frame on top of the native
ENOENT error after the connector returns null, masking the errno h5py keys on).
So h5py never falls back to create, and the file is never made.
`mode="w"` (plain create) works through the VOL; only `mode="a"`-on-missing
fails.

This is **not** a neuroh5/data-read issue and not addressed by A–E. Options:
fix error propagation in the connector (no clean connector-only path — the
masking frame is added by HDF5 core, outside the callback), pre-create the
dmosopt output file so `mode="a"` finds it (RDWR-open of an *existing* file
works through the VOL), or keep write-heavy output files off the read-cache VOL.
Note this boundary does not affect the **performance** verdict, which is a
read-path question already answered (CTE ≈ 31% slower on ares; see below).

## Bottom line

- **A, B, C are fixed and validated.** The HDF5 VOL CTE connector now builds
  against the MiV HDF5, loads with no app change, and — the decisive fix — reads
  pre-existing native files **correctly** (read-through cache + native fallback),
  where before it returned zeros and crashed. Two additional crash bugs (client
  bootstrap, dataset mis-cast) were fixed along the way.
- **neuroh5 was ported** off native-VOL-only deprecated APIs so it can run under a
  custom VOL at all.
- **The 4th blocker (D) — passthrough link-iteration wrapping — is fixed and
  validated** (clio-core PR #480). neuroh5's group enumeration runs through the VOL.
- **The 5th blocker (E) — neuroh5 `H5Gget_objinfo` — is fixed** (neuroh5 PR #1).
  A **full case-7 init run** now drives neuroh5's whole network-construction read
  path through the VOL (enumeration + cell-attribute-info + tree/coordinate
  reads); make_cells classifies and builds every population.
- **Remaining boundary (F):** the run stops at dmosopt's own output-file create
  (`h5py.File(..., "a")` on a missing file → `OSError` instead of
  `FileNotFoundError` through the VOL). This is a **write/output-path** issue on a
  non-neuroh5 file, not a data-read gap, and does not affect the perf verdict.
- **Read-path performance: CTE VOL ≈ 31% slower than native HDF5 on ares**
  (page-cache-resident data) — consistent with all prior POSIX-adapter findings.
  clio-core's CTE does not improve over the native baseline here because ares has
  no slow shared storage tier for the cache to win back.

## Related pages

- [Case 7 VOL adapter — pre-fix evaluation](miv_iowarp_ares_case7_vol.md)
- [Case 7 feasibility synthesis (POSIX adapter)](miv_iowarp_ares_case7_feasibility.md)
- [perf-clio-core](perf-clio-core.md) — where CTE's measured I/O wins come from (slow tier + re-read)

## Artifacts

- clio-core A/B/C: `~/clio-core` `main` (PR #475, merged); build `~/bin/build_vol.sh`
- clio-core D (link-iteration): `~/clio-core` branch `vol-cte-link-iter` (PR #480)
- neuroh5 (port + E): `~/neuroh5` branch `vol-cte-case7-fix` (PR #1)
- run harness: `~/bin/run_case7_vol.sbatch` (adds `IOWARP=2` VOL mode)
- full case-7 init logs: `~/logs/case7_c7vol2_20598.log` (VOL, reaches F),
  `~/logs/case7_c7base_20597.log` (native baseline, builds all populations)
- tests/bench: `~/bin/vol_smoke.sh`, `vol_path_sanity.sh`, `vol_n5_api.sh`,
  `vol_bench.sh`; for D `vol_iter_test.{c,sh}` (synthetic recursive `H5Literate2`)
  + `vol_neuroh5_read.{py,sh}` (real neuroh5 `io.so` reads); for E
  `vol_objinfo_test.c` (the `H5Gget_objinfo` repro)
