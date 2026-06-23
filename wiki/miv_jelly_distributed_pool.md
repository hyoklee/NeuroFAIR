# A distributed CTE memory pool across `nene` (macOS) + `jelly` (Linux)

**Date:** 2026-06-22 · single-writer controller: `nene`

This page extends the single-node IOWarp study ([miv_jelly_iowarp.md](miv_jelly_iowarp.md))
to a **two-node, two-OS distributed memory pool**: an IOWarp / clio-core
**Content Transfer Engine (CTE)** cluster whose aggregate RAM tier spans two
physically separate machines, used as the backing store for the MiV-Simulator
case-6 data on `jelly`.

## The two nodes

| node | role | OS / HW | clio-core | LAN IP |
|------|------|---------|-----------|--------|
| **nene** | controller + remote tier (node 0) | macOS 12, Intel i5-6500, 32 GB | built at HEAD from `~/src/clio-core` (ZMQ transport) | 10.10.10.154 |
| **jelly** | compute host + client (node 1) | CentOS 7, 2× Xeon (56 c), 125 GB | install from `/mnt/wrk/hyoklee/clio-core` (commit `10f0904`) | 10.10.10.120 |

Same `10.10.10.0/24` LAN; both builds link **ZeroMQ over TCP** (Thallium/libfabric
off), so a cross-OS, same-arch (x86-64 LE) cluster is viable.

## Topology / config

Identical compose on both nodes (`dmp/clio_conf.yaml`), node-local hostfile:

```
# hostfile (line offset = node id)
10.10.10.154   # nene  -> node 0
10.10.10.120   # jelly -> node 1
```

- RPC port **9413**; each node binds its real LAN IP (not loopback, so macOS libzmq
  issue #482 — a loopback-only bug — does not apply).
- CTE core at pool `512.0` with one **4 GiB RAM tier per node** and
  `targets.neighborhood: 2` → the aggregate **8 GiB** is the distributed pool.
- Start: `clio_run runtime start` on each, env `CLIO_SERVER_CONF` + `CONTAINER_HOSTFILE`.

## Result 1 — the cluster forms across macOS ↔ Linux

The decisive evidence the pool is genuinely distributed:

- **Bidirectional inter-node RPC** established over TCP — both directions ESTABLISHED:
  `jelly:41852 → nene:9413` *and* `nene:52111 → jelly:9413`.
- Each node identifies itself by local-interface IP and binds `0.0.0.0:9413`;
  CTE creates RAM-tier containers **on both nodes** (`cte_ram_tier1_node0` on nene,
  `cte_ram_tier1_node1` on jelly), plus 2-container `admin`/`bdev`/`cte_main` pools.
- The **2-commit version skew** (jelly `#585` vs nene HEAD `#593`/`#590`) and the
  macOS↔Linux split did **not** break the ZMQ/msgpack wire protocol.

**Gotcha:** start both nodes within the `wait_for_restart` window (30 s). If node 0
is up too long before node 1, its cross-node container-creation RPCs to node 1 expire
(`SendOut task timed out after 30s for node 1`). Co-starting cleanly avoids it.

## Result 2 — cross-node throughput (`clio_cte_bench` on jelly, 1 MiB × 256)

| query | data placement | aggregate BW | avg latency / op |
|-------|----------------|-------------:|-----------------:|
| `local` | jelly's own RAM tier | **442.85 MB/s** | 2,258 µs |
| `direct0` | **nene's RAM tier (over LAN)** | **5.27 MB/s** | 189,927 µs |

`direct0` = `PoolQuery::DirectHash(0)` → routes every op to the CTE container on
node 0 (nene), so the data physically crosses to nene's RAM and back.

## Result 3 — the actual case-6 inputs on the distributed pool (integrity-verified)

A small client (`cte_stage_file`, built against the clio-install CTE client API)
stores a **real case-6 HDF5 file** as 1 MiB blobs and reads it back, comparing
byte-for-byte. `direct0` forces every blob onto **nene's** RAM tier across the LAN.

| file | size | placement | PUT | GET | integrity |
|------|-----:|-----------|----:|----:|-----------|
| `MiV_Connections_…Small` | 8.90 MB | jelly (local) | 417 MB/s | 490 MB/s | **PASS** 0/9 |
| `MiV_Connections_…Small` | 8.90 MB | **nene (remote/LAN)** | 10.96 MB/s | 10.97 MB/s | **PASS** 0/9 |
| `MiV_Cells_…Small` | 36.48 MB | **nene (remote/LAN)** | 11.01 MB/s | 11.05 MB/s | **PASS** 0/35 |

The simulation's real connectivity and cell data live correctly in the distributed
pool, including when physically resident on the **remote** (macOS) node — verified
byte-identical to the source files (matching md5).

> A *live* `run-network` driven through the pool is gated by the HDF5-VOL connector's
> known cache-reassembly gaps (see [miv_jelly_iowarp.md](miv_jelly_iowarp.md)); and
> case-6 at production `tstop` is compute-bound with a read-once ~45 MB input, so the
> pool changes its wall time negligibly. The integrity test above is the rigorous,
> correctness-preserving demonstration that the data path works end-to-end.

## Result 4 — why the remote tier is ~11 MB/s (the link, not the software)

| measurement | rate |
|-------------|-----:|
| `nene` `en0` link (macOS Fast Ethernet) | **100baseTX = 100 Mb/s** |
| raw TCP `jelly → nene` (python socket, 100 MB) | **11.3 MB/s (90 Mbit/s)** |
| CTE remote-tier sequential staging (above) | ~11 MB/s — **link-saturated** |

The old Mac is on **100 Mb Fast Ethernet**; the slower end governs. Sequential
chunked staging hits the link ceiling (~11 MB/s); the bench's depth-16 `PutGet`
reads lower (5.27 MB/s) because `PutGet` moves the payload twice and adds per-op
RPC overhead. The 84× local-vs-remote gap is **the NIC, not CTE**.

## Result 5 — the >125 GB NeuroH5 read benchmark, with the distributed pool

The single-node study concluded a >125 GB set is RAID-bound either way, because the
**CTE RAM tier and the OS page cache compete for the same DRAM** — the tier adds no
capacity. A **distributed** pool changes that premise: nene's RAM is *separate*,
*non-competing* DRAM, so the pool genuinely **adds capacity** beyond jelly's 125 GB.

Fresh native baseline on jelly this session (`/mnt/wrk` RAID, `cat`):

| working set | pass 1 | pass 2 | regime |
|-------------|-------:|-------:|--------|
| 10.7 GB (< RAM) | 415 MB/s | 4359 MB/s | page-cache served on re-read |
| **150 GB (> RAM)** | **442 MB/s** (339.8 s) | — | **RAID-bound — page cache defeated** |

Distributed remote tier on real NeuroH5 forest data (1 GB slice of `cells_00.h5`,
256 × 4 MiB blobs):

| placement | read-back (GET) | integrity |
|-----------|----------------:|-----------|
| jelly local RAM tier | **439 MB/s** | **PASS** 0/256 |
| **nene remote RAM tier (LAN)** | **10.67 MB/s** | **PASS** 0/256 |

**Verdict:** the distributed pool *does* add the capacity the single-node tier
lacked — but on **this** hardware that capacity sits behind a **100 Mb link ~35×
slower than jelly's own RAID** (~11 MB/s vs ~400 MB/s). So spilling a >RAM working
set onto the remote tier is strictly worse than reading from the RAID. A distributed
RAM pool only accelerates >RAM reads when the peer nodes are on a **fast fabric**
(≥10–40 GbE) — exactly the ares-class hardware, and exactly what `jelly`+`nene`
over Fast Ethernet are not. This refines, rather than overturns, the single-node
conclusion: *capacity is no longer the blocker — interconnect bandwidth is.*

## How to reproduce

```sh
# both nodes: identical hostfile + dmp/clio_conf.yaml (4 GiB RAM tier, neighborhood 2)
# co-start within 30 s:
#   jelly:  source clio_env.sh; CLIO_SERVER_CONF=dmp/clio_conf.yaml clio_run runtime start &
#   nene:   CLIO_SERVER_CONF=dmp/clio_conf.yaml ./build/bin/clio_run runtime start &
# verify peers:  ss -tnp | grep 9413   (expect ESTABLISHED both directions)
# cross-node bench (on jelly):
clio_cte_bench --op PutGet --threads 1 --depth 16 --io-size 1m --io-count 256 --query-type direct0
# real-file integrity (on jelly): cte_stage_file <file.h5> <chunkMB> direct0   # VERIFY PASS
```
