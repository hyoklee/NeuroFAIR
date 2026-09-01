# clio-fs and the CLIO HDF5 VFD — how the two relate

**Date**: 2026-09-01
**Source**: clio-core `dev` @ `3357ac99` (`~/src/iowarp/clio-core/dev/conda`)
**Question answered**: *"What's the relation between clio-fs and the CLIO HDF5 VFD?"*

---

## One-line answer

They are not peers. **clio-fs is the layer the VFD sits on**: clio-fs is a POSIX
filesystem implemented as a ChiMod over the CTE, and the HDF5 VFD (`H5FDclio` /
driver `clio_vfd`) is one of five front ends that delegate to it.

```
   application
        |
  +-----+------+------+--------+---------+      <- interceptors / front ends
  |     |      |      |        |         |
 POSIX STDIO MPI-IO  FUSE   HDF5 VFD   (HDF5 VOL - see below, NOT via clio-fs)
  |     |      |      |        |
  +-----+------+------+--------+
        |
   clio-fs  (filesystem ChiMod, pool clio_cte_filesystem / 560.0)
        |     paths -> CTE tags,  offsets -> 1 MiB page blobs
        v
   CTE chain: cache 563 -> compressor 562 -> replication 561 -> core 512
```

---

## What clio-fs is

`context-transfer-engine/filesystem/` — a ChiMod that provides filesystem
semantics over the CTE. From `filesystem/clio_mod.yaml`:

> A filesystem interface over the CTE, sibling to the compressor chimod. The I/O
> interceptors (POSIX, STDIO, libfuse, HDF5 VFD, MPI-IO) delegate to this one
> chimod so filesystem semantics live in one place.

Mechanics:

| Aspect | Value / location |
|---|---|
| Pool | `kCfsPoolId(560, 0)`, `kCfsPoolName = "clio_cte_filesystem"` (`filesystem_tasks.h:61`) |
| Path mapping | path → CTE tag; byte offset → page blob named `off / kFsPageSize` (`PageName`, `filesystem_tasks.h:52`) |
| Page size | **1 MiB** (`kFsPageSize`, `filesystem_tasks.h:40`) — matches the libfuse adapter's blob paging |
| Methods | `kOpen/kClose/kRead/kWrite/kGetattr/kTruncate/kAppend/kReaddir/kMkdir/kRmdir/kUnlink/kRename/kStatSize` (ids 10–22) |
| Exactness | server-side **logical-size metadata**, so `truncate`/`append`/`getattr` are exact rather than page-rounded |
| Client | `clio::cte::filesystem::Client`, reached through the process singleton macro **`CLIO_CFS_CLIENT`** (`filesystem_client.h:1232`); inherits `clio::cte::core::Client` |
| Fast path | since issue #817 a *client* resolves page blobs itself and can serve reads straight from shared memory — logged once as `clio-fs: serving reads from shared memory (zero-IPC fast path active)` (`filesystem_client.h:1147`), with hit/miss counters |

The client's public surface is a POSIX **descriptor API**
(`filesystem_client.h:974+`): `OpenFd`, `ReadFd`/`WriteFd`,
`PreadFd`/`PwriteFd`, `SeekFd`, `TellFd`, `SizeFd`, `SyncFd`, `CloseFd`,
`FtruncateFd`, `TruncatePath`, `RemovePath`, `RenamePath`, `ReaddirPath`.

---

## What the HDF5 VFD is

`context-transfer-engine/adapter/vfd/` — `H5FDclio.cc` / `H5FDclio.h`.

| Aspect | Value |
|---|---|
| Driver name | **`clio_vfd`** (`H5FDclio.h:48`) |
| Driver value | **3200** (`H5FDclio.h:49`) |
| Requires | HDF5 **≥ 1.14** (multi-version `H5FD_class_t`, `read_vector`/`write_vector`) |
| Selection | `H5Pset_fapl_clio(fapl, cache_enabled)` when linked, or `HDF5_DRIVER=clio_vfd` + `HDF5_PLUGIN_PATH` with no source change |
| Authority | the **native on-disk HDF5 file** — plain native format, no driver-info superblock message, so `h5dump`/`h5ls`/`h5diff` read it live with or without CLIO running |
| CTE tier | **populate-only**: writes push in, reads never come out (read-serving is scoped in `translation/VFD_2.1_READ_CACHE_SCOPING.md`, not implemented) |

---

## The link, in code

`adapter/vfd/H5FDclio.cc:62` includes `clio_cte/filesystem/filesystem_client.h`,
and the CTE tier is driven **entirely** through clio-fs's descriptor API
(`H5FDclio.cc:126–128` states this explicitly):

| VFD callback | clio-fs call | `H5FDclio.cc` |
|---|---|---|
| open / create | `CLIO_CFS_CLIENT->OpenFd` | `:759` |
| write | `CLIO_CFS_CLIENT->PwriteFd` | `:1113` |
| truncate | `CLIO_CFS_CLIENT->FtruncateFd` | `:1583` |
| close | `CLIO_CFS_CLIENT->CloseFd` | `:775`, `:795`, `:810`, `:877` |
| delete | `CLIO_CFS_CLIENT->RemovePath` | `:1771` |

**There is no `PreadFd`/`ReadFd` call anywhere in the driver.** That absence *is*
the "populate-only tier" — and it is the code-level reason the VFD tracked native
read performance exactly in the case-7 `dev` measurement (2.38–2.39 GB/s vs
native's 2.39 GB/s, see [case 7 dev](miv_iowarp_ares_case7_dev.md)). The VFD
structurally cannot beat native on reads; only the VOL can.

### Which adapters actually use clio-fs

Counted `CLIO_CFS_CLIENT` call sites under `adapter/` at `3357ac99`:

| Adapter | clio-fs calls | Verdict |
|---|---|---|
| `libfuse` | 36 | via clio-fs |
| `posix` | 33 | via clio-fs |
| `stdio` | 25 | via clio-fs |
| `mpiio` | 8 | via clio-fs |
| `vfd` | 8 | via clio-fs |
| **`hdf5_vol`** | **0** | **not via clio-fs** |

---

## Three consequences worth remembering

### 1. The FUSE mount is a sibling of the VFD, not a dependency

`adapter/libfuse/` (`fuse_cte.cc`, `fuse_cte_ll.cc`) mounts clio-fs into the
kernel namespace — e.g. `/mnt/clio_fs` in the `safe-bdev` integration test
(`context-runtime/test/integration/safe-bdev/`, pool `clio_fs`). The VFD reaches
the *same* ChiMod **in-process**: no kernel mount, no FUSE hop, no root. Two
doors into one room.

### 2. The `clio::` marker is a clio-fs concept — and the two front ends treat it opposite ways

`kClioPrefix = "clio::"` lives in `filesystem_client.h:39`, with
`HasClioPrefix`/`StripClioPrefix` beside it.

- **POSIX adapter**: `IsPathTracked(path) == HasClioPrefix(path)`
  (`filesystem_client.h:915`) — a path is intercepted **iff** it carries the
  marker. This is exactly the mechanism behind the recurring finding in
  [caseread](miv_iowarp_ares_caseread.md) and
  [case 7 full](miv_iowarp_ares_case7_full.md) that the tier stayed at **0 bytes
  used**: MiV/neuroh5 emit unprefixed paths, so the interceptor never fires.
- **VFD**: a marked name is **refused** at open (`H5FDclio.cc:730`). CLIO is
  selected by the driver setting, never by the filename — because a filename that
  has to be rewritten is a source edit at every call site, and because
  `H5FD_FEAT_POSIX_COMPAT_HANDLE` / `H5FD_FEAT_DEFAULT_VFD_COMPATIBLE` promise
  HDF5 a name it can `stat()`.

So the VFD (and the VOL) fix blocker #1 from the
[feasibility analysis](miv_iowarp_ares_case7_feasibility.md) by construction:
no marker needed, no application change.

### 3. VFD and VOL are different routes, and that explains their measured asymmetry

| | HDF5 **VFD** (`clio_vfd`) | HDF5 **VOL** (`clio`) |
|---|---|---|
| Sits at | HDF5's file-driver layer (byte ranges) | HDF5's object layer (datasets) |
| Talks to | **clio-fs** descriptor API | **CTE core** directly (`AsyncPutBlob`/`AsyncGetBlob`) |
| Granularity | 1 MiB page blobs under a path tag | one blob per dataset image |
| Read from tier | **never** (populate-only) | **yes** (selection-aware, serve-only on hit) |
| Measured read path | = native (256 MB uncompressed: 107.7 vs 107.0 ms; case-7 payload: 513.0 vs 497.0 ms, +3.2 %) | 1.65× *faster* than native on gzip-chunked case-7 data (301.5 vs 497.0 ms); 3.9× slower on uncompressed |

The VOL bypassing clio-fs is why it *can* serve reads from the tier while the VFD
cannot, and why the two land on opposite sides of native in
[case 7 on `dev`](miv_iowarp_ares_case7_dev.md). They are complementary, not
redundant: the VFD gives any HDF5 application a correct native file plus a
populated tier; the VOL gives it a read-serving dataset cache.

---

## Portability note

clio-fs's descriptor API is inside `#if !defined(_WIN32)` upstream — it is
specified in `ssize_t`/`off_t` and POSIX descriptor semantics. So
`H5FD_CLIO_HAVE_CACHE_TIER` is compile-time false on Windows
(`H5FDclio.cc:126–135`) and the VFD there is native-only: the authoritative
on-disk file is written exactly as everywhere else, the tier is simply off. This
is the same degradation the driver applies at run time when the CLIO runtime is
unreachable, or when `cache=0` / `cache_enabled=false` is set — in which case no
attach is attempted at all and the driver is a complete, standalone HDF5 driver.

---

## Where clio-fs is exercised directly

- `benchmark/clio_cte_reliability_bench.cc` — mixed metadata + I/O reliability
  benchmark over the eight clio-fs verbs ("Clio-FS Reliability Benchmark");
  fails loudly if the pool is not composed.
- `benchmark/posix_fs_bench.cc` — buffered POSIX throughput; notes clio-fs does
  not fsync.
- `context-runtime/test/integration/safe-bdev/` — one container running runtime +
  safe-bdev + CTE + clio-fs, FUSE-mounted at `/mnt/clio_fs`, with a 32–64 MB
  write smoke test and a scripted member failure.
- The 4-node pool work already touched it: an 8.9 MB case-6 file round-tripped
  byte-identical through the `clio_cte_filesystem` chimod, **pool 560.0** — see
  [4-node distributed pool](miv_4node_distributed_pool.md).

---

## Related pages

- [MiV case 7 on the updated clio-core `dev` — POSIX vs VOL vs VFD](miv_iowarp_ares_case7_dev.md) — the measurements this page explains
- [4-node distributed CTE pool](miv_4node_distributed_pool.md) — clio-fs (pool 560.0) carrying real case-6 bytes
- [Can IOWarp/clio-core help MiV case 7 on ares?](miv_iowarp_ares_case7_feasibility.md) — blocker #1 is the `clio::` marker described above
- [Read-bound CA1 benchmark](miv_iowarp_ares_caseread.md) — the "tier stayed at 0 bytes" symptom of marker-based routing
- [Fixing the VOL CTE adapter](miv_iowarp_ares_case7_vol_fix.md) — the other HDF5 route, which does not go through clio-fs
- [clio-core CTE buffering for MiV optimization](perf-clio-core.md)
