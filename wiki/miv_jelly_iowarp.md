# IOWarp / clio-core on `jelly` — CTE built from source + a measured I/O win

Built **IOWarp's Context Transfer Engine (CTE)** from source on the HDF Group
**jelly** node (CentOS 7, glibc 2.17, kernel 3.10) and measured where it helps,
using large locally-generated NeuroH5/HDF5 data. Companion to the native-HDF5
baseline in [miv_jelly_case6.md](miv_jelly_case6.md). Date 2026-06-20.

## Building clio-core on CentOS 7

IOWarp targets a newer OS baseline than jelly, so two things were needed:

- **Toolchain:** IOWarp needs **C++20 / GCC ≥ 11**; jelly has system GCC 4.8.5
  and spack GCC 10.2.0 (both too old), and no prebuilt `iowarp-core` wheel works
  on glibc 2.17. Used **conda-forge GCC 13.4.0** (works on glibc 2.17) plus
  conda-forge `yaml-cpp, cereal, msgpack, libboost, zeromq, libsodium, libaio`.
- **Two portability patches** for CentOS 7:
  1. `libaio.h` missing → installed conda-forge `libaio`.
  2. `memfd_create` not declared (glibc 2.17 lacks the wrapper; added in 2.27) →
     patched `context-transport-primitives/src/system_info.cc` to call
     `syscall(SYS_memfd_create, …)` directly (RHEL-7's 3.10 kernel has the
     syscall backported; memfd is what lets the RAM tier exceed the 4 GB
     `/dev/shm`).
- **Config:** minimal CMake (runtime + CTE + benchmarks, ZeroMQ transport;
  **io_uring OFF** — needs kernel ≥5.1, jelly has 3.10; CUDA/ROCm/MPI/thallium/
  ADIOS2 all OFF). Builds `clio_run` (runtime), `clio_cte_bench`, `chimaera`,
  etc. into `/mnt/wrk/hyoklee/clio-install`. Env: `/mnt/wrk/hyoklee/clio_env.sh`.

The runtime starts cleanly (`clio_run start`): RAM bdev (memfd-backed, ~100 GB =
80% DRAM), CTE core pool, ZeroMQ server on 127.0.0.1.

## Storage reality on jelly

jelly has **no fast NVMe tier** and `/dev/shm` is only 4 GB; the working storage
`/mnt/wrk` is an **HDD RAID** (`/dev/md10`, 11 TB). Measured device bandwidth
(`dd`, 8 GB): **write (fdatasync) ~250–300 MB/s**, **uncached read (O_DIRECT)
~280–294 MB/s**. Only warm page-cache reads are fast (2.9–5.6 GB/s). So jelly
*does* have a slow backing store for CTE to beat — but also free RAM page cache,
which competes.

## Measured result — CTE RAM tier vs the RAID filesystem (8 GB)

Generated a 3.2 GB NeuroH5-style forest (`datasets/large/MiV_Cells_large.h5`,
200 M points) plus matched 8 GB `dd`/`clio_cte_bench` runs:

| Operation (8 GB) | `/mnt/wrk` RAID | IOWarp CTE RAM tier | speedup |
|------------------|----------------:|--------------------:|--------:|
| Write (device-synced)        | 248–302 MB/s | **1.86–2.45 GB/s** | **~7–10×** |
| Read (uncached, O_DIRECT)    | 280–294 MB/s | **1.88–2.05 GB/s** | **~7×** |
| Read (warm page cache, h5py) | 2.93 GB/s    | 1.88 GB/s          | cache wins |

`clio_cte_bench` (16 MB blobs, 8 threads) over the ZeroMQ-TCP client path; the CTE
RAM tier sustains ~1.9–2.4 GB/s (TCP IPC is the cap — true shared-memory IPC would
be higher).

**IOWarp helps, in the regime it is designed for.** For **I/O-bound** work —
writing/staging large data, or reading data that does **not** fit in (or is not
resident in) page cache — the CTE RAM tier is **~7–10× faster** than jelly's slow
HDD RAID. For **small, cache-resident, read-once** data the OS page cache (free
RAM) wins, so the CTE adds overhead — which is exactly why the MiV *simulation*
(compute-bound, ~45 MB read once) did **not** benefit, here or on ares
([miv_iowarp_ares_case6.md](miv_iowarp_ares_case6.md)). The two findings are
consistent: same hardware truth, opposite ends of the I/O/compute spectrum.

## How to reproduce
```bash
source /mnt/wrk/hyoklee/clio_env.sh
clio_run start                      # starts runtime (RAM tier)
clio_cte_bench --op Put --io-size 16m --threads 8 --io-count 64 --query-type local
clio_cte_bench --op Get --io-size 16m --threads 8 --io-count 64 --query-type local
clio_run stop
# baseline: dd if=/dev/zero of=/mnt/wrk/hyoklee/x bs=1M count=8192 conv=fdatasync
#           dd if=/mnt/wrk/hyoklee/x of=/dev/null bs=1M iflag=direct
```

## Routing HDF5/neuroh5 reads through the CTE — the VOL connector

Built the **HDF5 VOL connector** (`libiowarp_hdf5_vol.so`) on jelly against the
same HDF5 1.14.6, with `CLIO_CTE_ENABLE_HDF5_VOL=ON`. The ares fixes are now in
`iowarp/clio-core` **main**: HDF5≥1.14 floor, `H5PLget_plugin_*` exports, and the
read-through-cache `dataset_read` (native fallback + stage-to-tier) — so no source
patching was needed beyond enabling it.

**Status — reads route through the CTE; tier engages; tier-hit data has a bug.**

- The connector **loads** via `HDF5_VOL_CONNECTOR=iowarp` + `HDF5_PLUGIN_PATH`,
  and on first use **lazily attaches to the running CTE runtime**
  (`CLIO_CTE_CLIENT_INIT()` → "Successfully connected to runtime"). Unmodified
  **h5py routes its HDF5 calls through the IOWarp connector into the CTE**.
- **File open, dataset open, `H5Dget_type`, and reads all work**, returning
  **correct data == native** (tiny float dataset and the 3.2 GB
  `MiV_Cells_large.h5` `Populations/PYR/Trees/X Coordinate`, 200 M floats,
  checksum 504.3932). Verified end-to-end through the connector.
- **One build fix was required:** jelly's `/scr/hyoklee/hdf5-1.14.6` (which h5py
  links) has `assert()`s active, so the read tripped
  `H5T_patch_vlen_file: Assertion 'file' failed` (a no-op for non-vlen types).
  Built a drop-in **NDEBUG parallel HDF5 1.14.6** (`/mnt/wrk/hyoklee/hdf5-rel2`,
  38 MPI-IO symbols, Asserts OFF) and `LD_PRELOAD` it — reads then complete.
- **The CTE read-through cache engages** for `H5Dopen`+`H5S_ALL` whole reads
  (h5py low-level `h5d.open` → connector marks the dataset *cacheable*; the
  high-level `f[path]` uses `H5Oopen`, which currently yields a *non-cacheable*
  wrapper → native passthrough). On the cacheable path the dataset is **chunked
  and staged into the RAM tier** — `cte_search` lists the blobs
  (`hdf5:/…/X Coordinate/chunk_231`, `chunk_688`, …) — and the **tier-hit second
  read is ~2× faster** (0.98 s vs 2.14 s native+stage).
- h5py group iteration (`list(keys())`, `for k in g`) uses the deprecated
  `H5Literate_by_name1` (native-VOL-only) and is unsupported under any connector;
  **explicit-path** dataset access is the working pattern.

**Both connector gaps were fixed** (patched `iowarp_vol.cc`, rebuilt):
1. **High-level reads now cache.** `iowarp_is_whole_read` accepted only literal
   `H5S_ALL`; h5py high-level reads (`read_direct`, `ds[()]`) pass a *full
   selection* (`H5S_SEL_ALL`) instead. Widened the check to accept `H5S_SEL_ALL`
   (still contiguous/linear, so the chunk reassembly stays valid).
2. **Tier-hit correctness.** The per-chunk blobs `chunk_i` were written/read with
   blob-internal offset `i*chunk_size` instead of `0` — making each blob sparse
   and `(i+1)*chunk_size` long, so total tier use was **O(n²)**, the tier filled,
   high chunks silently failed to stage, and the hit served garbage. Fixed to
   blob-offset `0` (each `chunk_i` is its own `this_size`-byte blob).

**Verified after the fix:** high-level `read_direct` through the VOL stages the
full dataset (763 contiguous `chunk_0…chunk_762` blobs, no gaps), the **tier-hit
read is ~2.6× faster** (0.82 s vs 2.13 s native+stage), and the tier-served data
**matches native exactly** (checksum 100004032, `MATCH=True`). So the read-through
cache now works end-to-end: first read stages native→tier, subsequent reads served
correctly and faster from the CTE RAM tier.

## Does IOWarp help the case-6 *simulation*? — measured: no

Two independent reasons, both measured on jelly.

**(a) neuroh5 can't route reads through the VOL yet, and wouldn't benefit if it
could.** jelly's `neuroh5` uses the native-VOL-only `H5Literate(… H5_ITER_NATIVE …)`
/ `H5Gget_num_objs` APIs in **14 sites** (none ported to `H5Literate2`), so
`run-network` under `HDF5_VOL_CONNECTOR=iowarp` aborts immediately at
`cell_populations.cc:83` (`H5Literate … >= 0` assertion). Even after the ares-style
`H5Literate1→2` port, neuroh5's bulk reads are **collective MPI-IO**, which the
connector passes straight through to native — the CTE never caches them.

**(b) The simulation is compute-bound; its I/O is negligible.** Measured input and
phase breakdown (np=8, tstop=10 ms):

| run-network phase | wall | dominated by |
|-------------------|-----:|--------------|
| created cells     | 0.76 s  | read Cells 36 MB + cell setup |
| **connected cells** | **45.30 s** | read Connections 8.9 MB + instantiate **3.27 M synapses (compute)** |
| created gap junctions | 0.02 s | 102 edges |
| ran simulation (10 ms) | 2.22 s | NEURON integration (pure compute) |
| **total** | **51.98 s** | |

The **entire 45 MB input** (Cells 36 + Connections 8.9 + GapJ 0.05) is read **once**:

| read all 45 MB input | bandwidth | time | % of 52 s total |
|----------------------|----------:|-----:|----------------:|
| jelly RAID, cold (O_DIRECT) | ~217 MB/s | **0.21 s** | 0.40% |
| OS page cache, warm | ~5.6 GB/s | **0.04 s** | 0.08% |
| IOWarp CTE RAM tier | ~1.9 GB/s | ~0.024 s | 0.05% |

So even with **infinitely fast I/O**, the simulation could save **≤0.4 %** of wall
time. And for this 45 MB read-once, cache-resident working set the **OS page cache
(5.6 GB/s) already beats the CTE tier (1.9 GB/s)** — IOWarp would *add* overhead,
not remove it. This independently reproduces the ares end-to-end finding (IOWarp
POSIX adapter **+15 % slower** on this exact workload,
[miv_iowarp_ares_case6.md](miv_iowarp_ares_case6.md)).

**Verdict:** IOWarp's CTE genuinely helps **I/O-bound, large, repeated-read** work
on jelly's slow RAID (the `clio_cte_bench` ~7–10× and the VOL read-through cache
~2.6× above). It does **not** help the case-6 **simulation**, which is ~99.6 %
compute with a 45 MB read-once cache-resident input.

## A >125 GB NeuroH5 read benchmark — defeating the page cache

To force the I/O-bound regime, generated **150 GB** of NeuroH5 forest files
(14 × 10.7 GB, `Populations/PYR/Trees/X Coordinate`) in `/mnt/wrk/hyoklee/nh5_bench/`
— larger than jelly's **125 GB RAM**, so the OS page cache cannot hold it.

**Native read (`cat`, jelly `/mnt/wrk` RAID):**

| working set | pass 1 (cold) | pass 2 (repeat) | regime |
|-------------|--------------:|----------------:|--------|
| 10 GB (< RAM) | 389 MB/s | **4778 MB/s** | page-cache served on re-read |
| **150 GB (> RAM)** | 384 MB/s | **412 MB/s** | **RAID-bound — page cache defeated** |

The control proves the cache works *when the set fits* (re-read 12× faster); at
150 GB the repeat read stays at RAID speed (~400 MB/s, O_DIRECT device = 349 MB/s)
— the genuine I/O-bound regime IOWarp targets.

**Can the IOWarp CTE accelerate it on jelly? No — its only fast tier is RAM:**

| CTE RAM tier condition | bandwidth | vs RAID |
|------------------------|----------:|--------:|
| working set ≤ tier, RAM free (staging) | ~1.9–2.4 GB/s | **7–10×** |
| working set > tier (64 GB vs 32 GB tier) | ~0.39 GB/s | ≈ RAID (thrash/spill) |
| smaller ops under the 150 GB memory pressure | ~0.37–0.68 GB/s | ≈ RAID (RAM contention) |

The decisive structural fact: **the CTE RAM tier and the OS page cache compete for
the same DRAM** — the tier adds no capacity, it *reallocates* RAM. After the 150 GB
benchmark filled the page cache, the CTE tier came up at only 32 GB, and a working
set larger than the tier collapsed CTE bandwidth to ~0.4 GB/s, no better than the
RAID it was meant to beat. So on jelly a **>125 GB working set exceeds *both* the
page cache *and* the CTE RAM tier** → both are RAID-bound, and IOWarp cannot help.

**What it would take:** IOWarp's read win needs the working set to fit a *fast tier*.
On a single RAM-only node that tier *is* the page cache, so there is no headroom for
> RAM sets. Accelerating > RAM reads needs a fast **persistent** tier (NVMe) larger
than RAM — which jelly lacks (the ares NVMe-tier / Lustre studies are exactly that
hardware). IOWarp's measured jelly wins (`clio_cte_bench` 7–10×, VOL cache 2.6×)
are all **staging / sub-tier** cases, not > RAM reads.

## Notes / limits
- Globus was **not** used: jelly is not a Globus endpoint (no Globus Connect
  Personal, interactive auth required) and no NeuroH5 files were staged there, so
  the large data was generated locally instead. Globus CLI 3.42.0 is installed
  (conda env `globus`) for future use.
- The CTE bandwidth numbers above are via its native blob API (`clio_cte_bench`);
  the VOL connector (above) is the transparent-HDF5 path. ares VOL background:
  [miv_iowarp_ares_case7_vol_fix.md](miv_iowarp_ares_case7_vol_fix.md).
