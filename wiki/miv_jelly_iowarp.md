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

## Notes / limits
- Globus was **not** used: jelly is not a Globus endpoint (no Globus Connect
  Personal, interactive auth required) and no NeuroH5 files were staged there, so
  the large data was generated locally instead. Globus CLI 3.42.0 is installed
  (conda env `globus`) for future use.
- The CTE here is benchmarked via its native blob API. Routing neuroh5/HDF5
  reads transparently through the CTE needs the HDF5 **VFD/VOL** adapter (built
  and fixed on ares — [miv_iowarp_ares_case7_vol_fix.md](miv_iowarp_ares_case7_vol_fix.md));
  not enabled in this minimal jelly build.
