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
