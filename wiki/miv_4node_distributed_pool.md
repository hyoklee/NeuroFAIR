# A 4-node distributed CTE memory pool: nene + jelly + raven + macmini-m1

**Date:** 2026-07-14 · client/controller: `jelly`

This page extends the 2-node distributed CTE study
([miv_jelly_distributed_pool.md](miv_jelly_distributed_pool.md)) to a **4-node,
3-OS, 2-ISA** IOWarp / clio-core Content Transfer Engine (CTE) cluster, and stages
the real MiV-Simulator case-6 (gap-junctions) HDF5 inputs onto every node with
byte-integrity verification — the 4-node analog of commit
[`43efca2`](https://github.com/hyoklee/NeuroFAIR/commit/43efca2099dd7bf047de9aae9302d4a25f2833db).

## The four nodes

All built at clio-core **`dev` @ `b1e087c3`**; ZeroMQ over TCP (Thallium/libfabric
off), single RPC port **9413**, `swim.enabled: false`.

| node | role | OS / arch | HW | LAN IP | repo / build |
|------|------|-----------|----|--------|--------------|
| **0** | nene | macOS 12 / **x86_64** | iMac17,1, i5-6500, 32 GB | 10.10.10.154 | `~/src/clio-core`, `build/bin` |
| **1** | jelly | CentOS 7 / **x86_64** | 2×Xeon 56c, 125 GB | 10.10.10.120 | `/mnt/wrk/hyoklee/clio-core` (conda gcc-13, install `/mnt/wrk/hyoklee/clio-install`) |
| **2** | raven | Windows 10 / **x86_64** | — | 10.10.10.237 | `D:\clio-core-dev`, `build\bin` (icx 2025.3) |
| **3** | macmini-m1 | macOS / **arm64** (Apple Silicon) | 16 GB | 10.10.10.142 | `/Users/hdftest/test/hyoklee/src/clio-core` |

Same `10.10.10.0/24` LAN. The pool therefore spans **three operating systems**
(macOS, Linux, Windows) and **two instruction-set architectures** (x86-64 ×3 +
aarch64 ×1). Both ISAs are little-endian, so the ZMQ/msgpack wire protocol is
identical on the wire.

## Topology / config

Byte-identical `/tmp/clio_4node_hostfile` on all nodes (line offset = node id):

```
10.10.10.154   # nene       -> node 0
10.10.10.120   # jelly      -> node 1
10.10.10.237   # raven      -> node 2
10.10.10.142   # macmini-m1 -> node 3
```

Compose (`/tmp/clio_4node.yaml`; raven uses `C:\clio\dmp\clio_conf.yaml`): CTE core
at pool `512.0`, **one 4 GiB RAM tier per node** (16 GiB aggregate),
`dpe.dpe_type: max_bw`, `targets.neighborhood: 4`,
`default_target_timeout_ms: 300000`. The 300 s target timeout is required at 4
nodes: 60 s produced `PutBlob rc=11` timeouts through jelly's slow link under load
(see Result 3).

## Result 1 — the 4-node cross-ISA cluster forms

- Every pool is created with **4 containers, one per node** — `admin`,
  `ram::chi_default_bdev`, `clio_cte_core`, and the four per-node tiers
  `ram::cte_ram_tier1_node0..node3` (12 pools × 4 containers on raven's log).
- All **6 inter-node connections** establish over TCP:9413.
- The macOS↔Linux↔Windows split **and** the x86-64 ↔ **aarch64** split do not
  break the wire protocol — the arm64 node (macmini) participates as a full peer,
  and case-6 bytes round-trip through it verified-identical (Result 2).

**Gotcha (unchanged from 2 nodes):** co-start all four daemons within the
`wait_for_restart` window or a late node's cross-node container-creation RPCs
expire. Restart order used: raven first, then jelly → macmini → nene.

## Result 2 — real case-6 inputs stored on all 4 nodes (integrity-verified)

`cte_stage_file <path> 1 <query>` (ported from commit `43efca2` to the `b1e087c3`
API: `chi::CHIMAERA_INIT`→`clio::run::CLIO_INIT`, `chi::PoolQuery`→
`clio::run::PoolQuery`) stores a real case-6 HDF5 file as 1 MiB blobs from the
**jelly** client and reads every blob back, comparing byte-for-byte.
`directN` = `PoolQuery::DirectHash(N)` routes each op to node N's CTE container.

**`MiV_Connections_Microcircuit_Small_20220410.h5` — 8.90 MB (9 × 1 MiB):**

| routed to | PUT MB/s | GET MB/s | integrity |
|-----------|---------:|---------:|-----------|
| jelly — node1 (local) | 4.19 | 8.96 | **PASS** 0/9 |
| nene — node0 (LAN) | 4.34 | 5.52 | **PASS** 0/9 |
| **raven — node2 (Windows, LAN)** | 3.88 | 4.81 | **PASS** 0/9 |
| **macmini — node3 (arm64, LAN)** | 7.22 | 9.78 | **PASS** 0/9 |
| dynamic — DPE spread (all 4) | 4.93 | 6.43 | **PASS** 0/9 |

**`MiV_Cells_Microcircuit_Small_20220410.h5` — 36.48 MB (35 × 1 MiB):**

| routed to | PUT MB/s | GET MB/s | integrity |
|-----------|---------:|---------:|-----------|
| **raven — node2 (Windows, LAN)** | 4.22 | 5.20 | **PASS** 0/35 |
| **macmini — node3 (arm64, LAN)** | 7.69 | 9.85 | **PASS** 0/35 |
| dynamic — DPE spread (all 4) | 5.44 | 5.89 | **PASS** 0/35 |

The simulation's real cell and connectivity data live correctly in the pool on
**every** node, including — for the first time — the **Windows (x86-64)** and
**Apple-Silicon (arm64)** nodes, verified byte-identical to the source files. This
is the rigorous end-to-end demonstration; a *live* `run-network` driven through the
pool is still gated by the HDF5-VOL connector's cache-reassembly gaps (see
[miv_jelly_iowarp.md](miv_jelly_iowarp.md)), and case-6 is compute-bound on a
read-once ~45 MB input, so the pool changes its wall time negligibly.

## Result 3 — throughput is LAN-bound for *every* routing mode

Unlike the 2-node study — where a jelly-`local` put hit **442 MB/s** — at 4 nodes
**every** placement, including `local`, ran at **~4–10 MB/s**. An isolated check
confirms it is not the staging tool:

```
clio_cte_bench --op Put --query-type local --io-size 1m --io-count 50 --depth 1
  → Aggregate bandwidth: 6.12 MB/s,  avg latency 163 ms/op
```

163 ms for a 1 MiB "local" put is a full LAN round-trip, not a shared-memory copy.
The cause is the composition **`targets.neighborhood: 4` + `max_bw` DPE**: the CTE
distributes each blob's placement across the whole 4-node neighborhood, so the
bytes leave the client node onto remote tiers regardless of the client's routing
query. In other words, at neighborhood 4 the pool genuinely *uses all four nodes*
— at the cost of being interconnect-bound end-to-end.

macmini (node3) was consistently the **fastest** target (~7–10 MB/s) and
nene/raven the slowest (~4–6 MB/s), consistent with nene sitting on 100baseTX Fast
Ethernet (measured raw TCP ~11 MB/s in the 2-node study) and the slowest path
governing each op.

## Result 4 — a 5 GB synthetic fill spreads across all 4 nodes

Before the case-6 runs, a `clio_cte_bench --op Put --threads 8 --depth 2
--io-size 16m --io-count 40 --query-type dynamic` (320 × 16 MB = **5120 MB**, larger
than any single node's 4 GB tier) completed clean (exit 0, 320/320 ops) at **9.18
MB/s aggregate**. `clio_run` RSS grew on **all four** nodes, proving genuine 4-node
placement:

| node | RSS before | RSS after | Δ |
|------|-----------:|----------:|---:|
| nene (0) | 27 MB | 2168 MB | **+2.04 GB** |
| jelly (1) | 26 MB | 1373 MB | **+1.28 GB** |
| raven (2) | ~1.1 GB* | 2236 MB | **+1.08 GB** |
| macmini (3) | 37 MB | 1751 MB | **+1.63 GB** |

*raven's baseline already includes pre-attached shared-memory pages on Windows.
The 5 GB set demonstrably occupies all four tiers (Σ deltas ≈ 6 GB with replication
overhead), exceeding any single node's 4 GB capacity.

## Result 5 — real case-6 reads through the **HDF5 VOL adapter** (not just raw blobs)

Results 2–4 use `cte_stage_file` / `clio_cte_bench`, which call the CTE **client
API directly** on raw bytes — the `.h5` file is opaque, no HDF5 involved. This
result exercises the actual **HDF5 access path**: the `clio` VOL connector
(`libclio_hdf5_vol.so`, built at `b1e087c3`), driven by `h5dump` from the same
HDF5 (1.14.6) the connector links against, attached to the 4-node pool.

The connector loads with **no application change** via the standard mechanism
(the `H5PLget_plugin_*` exports are present in `dev`):

```sh
export HDF5_VOL_CONNECTOR=clio                     # connector name is "clio"
export HDF5_PLUGIN_PATH=/mnt/wrk/hyoklee/clio-install/lib
export CLIO_SERVER_CONF=/tmp/clio_4node.yaml CLIO_WITH_RUNTIME=0
/scr/hyoklee/hdf5-1.14.6/bin/h5dump -d <dataset> <case6_file.h5>
```

Reading two datasets of `MiV_Connections_…Small` through the VOL, compared against
a native (no-VOL) read of the same datasets:

| dataset | type | VOL vs native |
|---------|------|---------------|
| `/H5Types/Populations` | compound committed datatype | **IDENTICAL** (38/38 lines) |
| `/Projections/OLM/PVBC/Connections/distance` | numeric array | **IDENTICAL** (5160/5160 lines) |

- On `H5Fopen` the connector **initializes a CTE client and connects to the 4-node
  runtime** — its `ClientInit` log (`server_pid` = the running daemon) appears only
  in the VOL run, never the native one, proving the `clio` connector (not HDF5's
  native VOL) served the open.
- It then reads the **pre-existing native** case-6 file correctly, including
  compound committed types and numeric datasets — the read-through-with-native-
  fallback path works in `dev`. This closes the gap noted in Result 2: real
  case-6 HDF5 files are now readable *through the HDF5 API* over the pool, not only
  as raw staged blobs.

Since `h5dump` uses the same HDF5 C library that `h5py`/`neuroh5` link, the result
is library-level and carries to those tools (h5py is not installed on jelly, so
`h5dump` stands in).

> **Fixed (2026-07-15):** originally the CTE runtime logged Debug/Info/Success to
> **stdout** (`context-transport-primitives/.../util/logging.h`), so ~12 INFO lines
> corrupted h5dump's data output and had to be stripped. Logs now go to **stderr**
> for all levels (stdout is reserved for a program's data). After the fix, a
> full-file `h5dump` through the VOL is **byte-identical to native with no
> stripping** (0 log lines on stdout; the runtime logs land on stderr).

**Did the VOL read actually use the tier?** No — `CLIO_VOL_TRACE=<dir>` (a
directory the connector fills with per-access `.jsonl` + a summary `.json`
carrying a `served` classification: `cache` / `native` / `uncacheable`) settles
it. Two passes of the numeric `distance` dataset (chunked `[1000]`, hyperslab,
float — cacheable in principle) both report:

```
read_served: {cache: 0, native: 1, uncacheable: 0}   cache_hit_rate: 0
read_bytes_from_native: 145952   read_bytes_from_cache: 0   (both passes)
```

So the VOL reads of **pre-existing native** case-6 files are a **correctness
passthrough to the native file** — the bytes never traverse CTE blobs and the
pool's tier stays unused, even on a second pass (the connector delegates to
native and does not populate the tier on read). This matches the earlier ares
finding that the tier stays at 0 bytes for read-existing-file workloads. To place
HDF5 data *on* the pool you must **write through** the connector, not read
pre-existing files.

## Result 6 — the POSIX interceptor adapter (write-through tier, case-6 round-trip)

The **POSIX** adapter (`libclio_cte_posix.so`) is the complementary path: an
`LD_PRELOAD` interceptor of libc I/O that routes paths carrying a `clio::` marker
(as a path component) to the tier via the `clio_cte_filesystem` chimod. It is
**write-through** (store bytes, then serve them back from the tier) — the opposite
of the VOL's read-existing passthrough. That chimod is
[clio-fs](concept-clio-fs-vs-hdf5-vfd.md), the shared filesystem layer the POSIX,
STDIO, MPI-IO, FUSE and HDF5-VFD front ends all delegate to; the `clio::` marker
is its routing rule (`IsPathTracked` == `HasClioPrefix`).

**Building it on jelly (CentOS 7 + conda toolchain) took `-DCLIO_CORE_ENABLE_ELF=ON`
plus three portability shims**, all under `/mnt/wrk/hyoklee/pkgshim` — worth
recording, as none are obvious:

1. **`libelf.pc`** — the CMake requires `pkg_check_modules(libelf REQUIRED)`, but
   CentOS 7's elfutils ships `libelf.so` + headers with **no `.pc` file**. Hand-write one.
2. **`CPLUS_INCLUDE_PATH`** to a dir of symlinked ELF headers — the conda `g++`
   does not search `/usr/include`, and pkg-config *strips* `-I/usr/include` as a
   "standard" path, so `<gelf.h>` is invisible. A **non-standard** include dir
   (symlinks to `gelf.h`/`libelf.h`) survives.
3. **`LIBRARY_PATH`** to a dir with a `libelf.so` symlink — the conda `ld` does not
   search `/usr/lib64`, so `-lelf` fails to link otherwise.

With those, the daemon loads the `clio_cte_filesystem` chimod and creates pool
`560.0` with **4 containers (one per node)** — the tier-backed filesystem spans
all four nodes.

**Result:** the 8.90 MB `MiV_Connections_…Small` file, written to a
`clio::/tmp/…` path and read back through the adapter, is **byte-identical**
(`POSIX_INTEGRITY_PASS`, 8,899,461 bytes). The write got CTE fd **8192** and the
read fd **8193** (both ≥ `kCfsFdBase`), confirming interception routed through the
tier rather than the native filesystem.

**Caveat:** the adapter hands out **synthetic fds (≥ 8192)** that are not real
kernel descriptors, so standard tools that `fstat()` the fd (coreutils `dd`,
`cat`) fail with `EBADF`, and `openat()` is not intercepted (only
`open`/`open64`/`creat`). A purpose-built `open`/`read`/`write` harness (no
`fstat`) exercises it cleanly — which is how the round-trip above was measured.

**VOL vs POSIX, side by side:** on this 4-node pool, the **POSIX** adapter
genuinely routes case-6 data *through* the tier (write-through, verified), while
the **VOL** over pre-existing files serves *native* (read passthrough, tier
unused). They are complementary: POSIX for staging bytes into the pool, VOL for
transparent correct reads of existing HDF5 without app changes.

## Result 7 — read-performance comparison of the three access methods

Reading the same case-6 payload back three ways, from the jelly client against the
4-node pool. **The methods do not all read from the same place**, which is the
whole point of the table:

| method | what it reads | backing store | read MB/s |
|--------|---------------|---------------|----------:|
| raw blob — `cte_stage_file` GET, `local` | 36.48 MB file as 1 MiB blobs | **CTE tier** | 8.5 |
| raw blob — `dynamic` | same | **CTE tier** (spread ×4) | 5.7 |
| POSIX adapter — `clio::` path | 36.48 MB file via filesystem chimod | **CTE tier** | 14.0 |
| HDF5 VOL — `-d` dataset | 2.67 MB dataset | **native file (tier bypassed)** | ~39 |
| *native baseline* (reference) | 36.48 MB plain read | page cache | *7268* |

**Only raw-blob and POSIX are an apples-to-apples pair** — both genuinely read from
the distributed CTE tier over the LAN, so both are interconnect-bound (single
digits to ~14 MB/s), consistent with the ~11 MB/s Fast-Ethernet ceiling and the
`neighborhood:4` spread. Between them, **POSIX was ~1.6–2.5× faster** (14.0 vs
8.5/5.7): `cte_stage_file` issues strictly synchronous one-blob-at-a-time GETs
(depth 1), whereas the POSIX filesystem-chimod read path is more efficient.

**The VOL number is not a tier read.** `CLIO_VOL_TRACE` confirms every VOL read was
`served: native` — for pre-existing files the connector delegates to the native
file (page cache + HDF5 stack + passthrough), never touching the pool. Its ~39
MB/s is therefore local-file access minus HDF5/VOL overhead (far below the 7268
MB/s raw native; still several × the tier methods only because it stays off the
network). To make the VOL read *from* the tier you must write the HDF5 file
*through* the connector first (write-through) — a different workload.

Two caveats on the VOL measurement: (1) the 39 MB/s came from an explicit
single-dataset `-d` read (2.67 MB `syn_types`), because the `CLIO_VOL_TRACE`
instrumentation only records reads on datasets opened **by name** (`-d`), not those
opened via full-file link/object *iteration* — a **trace under-count**, not a data
gap. A full-file `h5dump` through the VOL in fact reads **all** data correctly
(output byte-identical to native — 237/62 DATA blocks on the Cells/Connections
files), the reads simply aren't tallied in the trace summary. *(This corrects an
earlier version of this page that mis-reported "0 data reads / full-traversal
gap".)* (2) per-pass VOL rates were noisy (24.7 / 62.3 / 49.3 MB/s) — small
payload, per-op overhead dominates. Numbers are single-run on the live cluster.

**Takeaway:** for data resident in the distributed pool, **POSIX is the fastest
read path (~14 MB/s), ahead of raw blob (~6–8.5 MB/s)**, both LAN-bound; the VOL
over existing files reads the native copy and bypasses the pool, so its rate is not
a distributed-tier measurement.

## Result 8 — VOL read-performance *with the compound/whole-read patch* (goes through CTE)

The VOL's "reads native, bypasses the pool" behaviour above was then fixed
([iowarp/clio-core#750/#751](https://github.com/iowarp/clio-core/pull/751)): fixed
compound/array datasets are made read-cacheable, and a full-extent hyperslab (what
`h5dump`/`H5Dread` issue) is treated as a whole read so it populates and serves the
tier instead of falling into the serve-only partial path. Re-benchmarked with the
patched connector on a 10.69 MB float dataset
(`/Populations/PYR/Synapse Attributes/syn_locs/Attribute Value`), reading it four
times in one process (`H5Dread`, `CLIO_VOL_TRACE` confirms the `served` field):

| path | throughput | `served` |
|------|-----------:|----------|
| native, no VOL — cold | 59.6 MB/s | (RAID) |
| native, no VOL — warm | **~131 MB/s** | (page cache) |
| VOL patched — pass 1 | 2.7–3.0 MB/s | `native` (miss: native read **+ stage into tier**) |
| VOL patched — pass 2 → 4 | 8 → 16 → **~21 MB/s** | **`cache`** (served from the CTE tier) |

The patch achieves its goal — **reads are now served from the CTE tier**
(`served: cache`, trace-confirmed; before the patch this and every compound dataset
read `native`, tier bypassed). But on **this** cluster it is a *functional*
enablement, not a speedup: steady-state tier-served read is **~20 MB/s vs ~131 MB/s
native page-cache (~6× slower)**, and the first touch is expensive (~2.7 MB/s) —
it stages the whole dataset as blobs across the pool. Same root cause as every
result here: the tier is LAN-bound (~11 MB/s Fast Ethernet) while native reads hit
local page cache, so CTE only *wins* when the backing store is slower than the
fabric (slow shared FS + ≥10–40 GbE) — not here. Of note, the VOL cache read
(~20 MB/s) is nonetheless the **fastest of the three tier paths** (VOL ~20 > POSIX
~14 > raw blob ~6–8.5). Single-run on the live cluster.

## Verdict

The distributed CTE pool scales cleanly from 2 to **4 nodes across 3 OSes and 2
ISAs**: pools form, real case-6 data is stored and read back byte-identical on every
node (including Windows/x86-64 and macOS/arm64), and a >single-tier working set
spreads over all four. The performance story is unchanged and reinforced from the
2-node finding — **interconnect bandwidth, not capacity, is the ceiling.** At
`neighborhood: 4` the pool distributes placement cluster-wide, so throughput tracks
the LAN (~4–10 MB/s, gated by nene's Fast Ethernet) for every routing mode. The
pool adds RAM capacity a single node cannot, but only *accelerates* access when the
peers sit on a fast fabric (≥10–40 GbE) — exactly the conclusion the 2-node study
reached, now confirmed at twice the scale.

## How to reproduce

```sh
# 1. All 4 nodes at dev b1e087c3; identical /tmp/clio_4node_hostfile +
#    /tmp/clio_4node.yaml (raven: C:\clio\dmp\). jelly needs the CentOS-7
#    memfd_create syscall shim in context-transport-primitives/src/system_info.cc.
# 2. Co-start all daemons within wait_for_restart (raven, then jelly, macmini, nene):
#    <node>$ CLIO_SERVER_CONF=/tmp/clio_4node.yaml clio_run runtime start
# 3. From jelly (client), stage a real case-6 file onto a chosen node & verify:
export CLIO_WITH_RUNTIME=0 CLIO_SERVER_CONF=/tmp/clio_4node.yaml
cte_stage_file <case6_file.h5> 1 direct3    # -> macmini (node3); PASS = byte-identical
cte_stage_file <case6_file.h5> 1 dynamic    # -> spread across all 4 nodes
```

`cte_stage_file.cc` (b1e087c3 port) lives at
`context-transfer-engine/benchmark/cte_stage_file.cc` on jelly's tree, built as a
CMake target alongside `clio_cte_bench`.
