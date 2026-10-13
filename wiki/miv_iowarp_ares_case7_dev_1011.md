# MiV case 7 on clio-core `dev` @ `5ac2747d` (Oct 2026) — the VFD overhead is gone, the VOL's is not (ares)

Third run of the same MiV-Simulator **case 7** workload against clio-core, this
time on `dev` **`5ac2747d`** (2026-10-11), 1,486 commits after the
[`3357ac99` run of 2026-09-01](miv_iowarp_ares_case7_dev.md). Those commits
include a batch of HDF5 VOL/VFD performance and correctness fixes landed in
mid-September, so the question is simply: did they move the end-to-end number?

> **Correction (2026-10-12).** The POSIX-adapter (`IOWARP=1`) runs on this page
> routed **no** simulation I/O through clio-fs: the interceptor only handles paths
> containing a `clio::` component, and the case-7 configs use plain paths. The
> number measures pass-through interception only. MiV I/O actually served by
> clio-fs is measured in [case 7 with inputs in clio-fs](miv_iowarp_ares_case7_clio_fs.md) (+4.8 %).

## Setup — unchanged, so the runs are directly comparable

Identical to the [September run](miv_iowarp_ares_case7_dev.md): `optimize-network`
on `Microcircuit_Small`, `optimize_network_ares_io.yaml` (`tstop=200`), 1 node,
`mpirun -np 17` (1 distwq controller + 4 workers × 4 ranks), `n-initial=1`,
`--surrogate-method=megp` → **13 deterministic GLP initial evaluations**, then a
deliberate stop (megp needs GPyTorch, not installed — `exit=1` is expected and
identical in every run). `--exclusive`, one job per node. Every run below was
checked to have completed **exactly 13 evaluations** (13 × `broker N: sending
task complete`).

**Build.** `~/src/iowarp/clio-core/dev/conda` @ `5ac2747d`, built on a compute
node with the unchanged `~/bin/build_core_dev_miv.sh` (now taking `BUILD=` /
`PREFIX=` overrides) into a **new prefix `~/core_dev_miv_1011/install`**, so the
September build at `~/core_dev_miv/install` is kept for A/B. Same flags: ELF +
POSIX adapter + HDF5 VOL (`clio`) + HDF5 VFD (`clio_vfd`), against the miv env's
parallel HDF5 1.14.6 (`ldd` → `~/mc3/envs/miv/lib/libhdf5.so.310` for both
`.so`s). Clean build, no patches needed.

**Correctness re-verified on the new build** before any timing:

- h5py whole-file read of `Microcircuit_Small_connections.h5` (60 datasets,
  39.2 MB): md5 `fddb49e9…` **identical** native / VOL / VFD
  (`~/bin/case7_dev_smoke.sh`).
- neuroh5's own C++ read API: output **byte-identical** native / VOL / VFD, and
  `CLIO_VFD_DEBUG=1` shows the VFD actually driving the reads (82 traced
  `[vfd] READ` ops for the script) (`~/bin/case7_backend_verify.sh`).
- Real case-7 payload (4 files, 43.3 MB numeric): md5 `5d80841e…` identical
  across all 10 backend × build combinations (Result 2).

## Environment issue found on the way — UCX hangs `MPI_Init` on ares (Oct 2026)

Not a clio-core problem, but it blocks every MiV run, so it is recorded here.
As of 2026-10-12 **`from mpi4py import MPI` hangs indefinitely** in the miv env —
on the login node and on compute nodes, with or without `mpirun`, with or
without clio-core on `LD_LIBRARY_PATH`. The miv env itself is unchanged since
2026-05-25, so the change is system-side.

Isolation (30 s timeout each): `OMPI_MCA_coll=^ucc` → hang; `OMPI_MCA_osc=^ucx`
→ hang; `OMPI_MCA_pml=ob1` → OK; **`UCX_TLS=self,sm,tcp` → OK**. UCX 1.20.0 is
hanging while probing the nodes' RDMA devices (`rocep47s0`, iWARP `iwo1`/`iwo2`).

**Fix applied:** `export UCX_TLS="${UCX_TLS:-self,sm,tcp}"` (and `-x UCX_TLS` to
`mpirun`) in `~/bin/run_case7_dev.sbatch` and `~/bin/case7_backend_verify.sh`.
For these single-node runs MPI traffic is intra-node shared memory either way,
and the setting applies equally to the baseline and every backend, so it does
not bias the comparison. Multi-node MiV runs on ares will need the same
workaround (or a fixed UCX/RDMA stack) — over TCP rather than RDMA.

## Result 1 — end-to-end case 7 (1 node, 13 evaluations each)

| Config | reps (s) | mean | per eval | vs baseline | Sept (`3357ac99`) |
|---|---:|---:|---:|---:|---:|
| **baseline** (native HDF5) | 1864.4 / 1865.4 | **1864.9 s** | 143.5 s | — | — |
| CTE **POSIX** adapter | 1875.1 / 1852.4 | **1863.7 s** | 143.4 s | **−0.1 %** | +1.0 % |
| HDF5 **VOL** (`clio`, cache on) | 1949.5 / 1947.4 | **1948.5 s** | 149.9 s | **+4.5 %** | +4.6 % |
| HDF5 **VOL**, `CLIO_VOL_CACHE=0` | 1883.0 | 1883.0 s | 144.8 s | +1.0 % | −0.8 % |
| HDF5 **VFD** (`clio_vfd`, cache on) | 1875.5 / 1877.5 | **1876.5 s** | 144.3 s | **+0.6 %** | **+4.2 %** |
| HDF5 **VFD**, `cache=0` | 1881.1 | 1881.1 s | 144.7 s | +0.9 % | −0.1 % |

Jobs 24592–24605. Data: `~/MiV-Simulator-Cases/7-optimization/results/comparison_case7_dev_1011.csv`.
The baseline itself is within 0.3 % of September's (1860.1 s). The noise floor
between same-config reps on different nodes is about ±1 % (POSIX reps differ
by 1.2 %), so the single-rep cache-off controls (+0.9 / +1.0 %) are best read
as "at native", not as a regression from September's −0.8 / −0.1 %.

**One run excluded.** The first POSIX rep (job 24593) took **5303 s** on
`ares-comp-18`. Every phase was ~3× slower, including pure NEURON integration
(1.85 s vs 0.60 s per step) that no I/O adapter can touch. The node itself is
degraded: a single-thread Python CPU benchmark takes **32.0 s on comp-18 vs
10.1 s on comp-12**, and comp-18 showed load 5.6 while otherwise idle with no
user processes. Its clio runtime behaved exactly like the other runs (one
elastic worker spawn, 5 workers). It was rerun with `--exclude=ares-comp-18`
(job 24605, 1852.4 s). **Avoid `ares-comp-18` for timing work** until it is
looked at.

### Per-phase proxy agrees

Rank-0 time to configure the 33,436 synapses of gid 10 (4 samples per run):

| backend | Oct (`5ac2747d`) | Sept (`3357ac99`) |
|---|---|---|
| native | 8.56–8.68 s | 8.57–8.59 s |
| POSIX | 8.56–8.64 s | 8.59–8.62 s |
| VOL, cache on | **9.00–9.25 s** | 9.05–9.09 s |
| VOL, cache off | 8.54–8.67 s | 8.56–8.64 s |
| **VFD, cache on** | **8.55–8.68 s** | **9.05–9.09 s** |
| VFD, cache off | 8.59–8.64 s | 8.56–8.64 s |

The VFD with its tier on has dropped onto the native band; the VOL with its
tier on is still the one config outside it.

### What changed, and what did not

- **VFD: +4.2 % → +0.6 % (at native).** This is consistent with
  `656e96e7` *perf(vfd): stop advertising vectored I/O, so the data sieve is
  used again* — implementing `read_vector`/`write_vector` had switched HDF5's
  selection I/O on, which bypasses the data sieve the driver itself requested;
  the commit measures the driver at parity with `sec2` once withheld. Case 7's
  read path is many small metadata and chunk reads, exactly where the sieve
  matters. (Attribution is by commit content, not a bisect.) `85165e57`
  (*track residency, so the read tier stops serving zeros it never held*) is a
  correctness fix for scattered access; case 7 was already byte-identical
  before it.
- **POSIX adapter: +1.0 % → −0.1 %.** Within noise both times — but see the
  correction above: with no `clio::` paths this is the cost of loading the
  interceptor, not of clio-fs.
- **VOL: +4.6 % → +4.5 %, unchanged.** The September admission work
  (`85c00bf0`, `f3547661`: admit to the tier *only when it can beat the store it
  replaces*) does not switch the cache off for case 7, and correctly so: the
  data is hundreds of small **gzip-chunked** datasets, native reads of which
  are measured slower than the tier, so the gate admits them (Result 2 shows
  the cache still winning on the payload). The cache-off control is back at
  native, so the remaining ~4.5 % is still the cost of the VOL's
  cache path itself, paid across every dataset/metadata access of a
  compute-bound run whose total I/O is ~0.1 % of its runtime.

## Result 2 — read path on the real case-7 payload, Sept vs Oct build, same node

`~/bin/case7_payload_bench.sh` (now `CORE=`-selectable), both builds in one job
on `ares-comp-16` (job 24606), payload staged to node-local `/tmp`, 8 passes,
median:

| backend | Sept `3357ac99` | Oct `5ac2747d` |
|---|---:|---:|
| native | 503.4 ms | 495.6 ms |
| **VOL, cache on** | **278.0 ms** (first 709.9) | **288.6 ms** (first 705.0) |
| VOL, cache off | 521.9 ms | 509.0 ms |
| VFD, cache on | 511.7 ms (first **605.5**) | 507.2 ms (first **525.5**) |
| VFD, cache off | 534.1 ms | 507.7 ms |

- The **VOL cache's read win is unchanged: ~1.7× faster than native** on the
  real compressed payload (it serves the decompressed image; see the
  [mechanism](miv_iowarp_ares_case7_dev.md#c-mechanism--the-cache-serves-a-decompressed-image)).
  It still does not become an application win, for the same reason as before:
  ~0.2 s saved per evaluation out of ~143 s.
- The **VFD's first-pass (populate) cost fell from +20 % to +6 %** over native
  (605 → 525 ms), and its warm reads stay at native. That cheaper populate
  path is the read-side face of the end-to-end improvement above.

### Not measured here — the VFD's new opt-in read tier

`5ac2747d` also gives the VFD an **opt-in read tier** (`CLIO_VFD_READ_TIER=1`,
default off; residency-tracked since `85165e57`) and makes vectored I/O opt-in
(`CLIO_VFD_VECTOR_IO`). All VFD runs above used the defaults — populate-only,
no vectored I/O — matching September. A `CLIO_VFD_READ_TIER=1` run of case 7
is the obvious next measurement: it is the first configuration in which the
VFD *could* serve reads from the CTE.

## Bottom line

1. **The HDF5 VFD is now free end-to-end with its tier on** (+0.6 %, vs +4.2 %
   on `3357ac99`), and its per-phase time sits inside the native band. All
   three non-VOL configurations — POSIX adapter, VFD, VFD cache-off — are now
   indistinguishable from native HDF5 on case 7.
2. **The VOL connector's +4.5 % did not move.** Its cache still wins 1.7× on
   the read path and still cannot pay for itself in a compute-bound
   application. Its cache-off mode remains free.
3. **Nothing in clio-core makes case 7 faster** — as in every prior ares study,
   the workload, not the adapter, is the limit (see
   [case 7 feasibility](miv_iowarp_ares_case7_feasibility.md)).
4. Two ares environment findings: **UCX hangs `MPI_Init`** (workaround
   `UCX_TLS=self,sm,tcp`), and **`ares-comp-18` is degraded** (~3× slow CPU).

## Related pages

- [Case 7 with every input in clio-fs](miv_iowarp_ares_case7_clio_fs.md) — the first run with I/O actually served by clio-fs
- [Case 7 on clio-core `dev` `3357ac99` (Sept 2026)](miv_iowarp_ares_case7_dev.md) — the run this compares against, including the VOL cache mechanism
- [Case 7 — IOWarp core vs native HDF5, RAM vs NVMe (May 2026)](miv_iowarp_ares_case7.md)
- [Fixing the HDF5 VOL CTE adapter for case 7 (A–F)](miv_iowarp_ares_case7_vol_fix.md)
- [clio-fs and the CLIO HDF5 VFD](concept-clio-fs-vs-hdf5-vfd.md)
- [Case 7 feasibility synthesis](miv_iowarp_ares_case7_feasibility.md)

## Artifacts

- clio-core: `~/src/iowarp/clio-core/dev/conda` @ `5ac2747d`; build
  `BUILD=~/core_dev_miv_1011/build PREFIX=~/core_dev_miv_1011/install ~/bin/build_core_dev_miv.sh`
  (log `~/logs/build_core_dev_miv_1011.log`)
- run harness: `~/bin/run_case7_dev.sbatch` (+ `UCX_TLS` workaround; backup of
  the pre-change script at `run_case7_dev.sbatch.bak.20261012`); matrix
  submitted by `~/bin/submit_case7_dev_1011.sh`
- logs: `~/logs/case7_o_{base,posix,vol,vfd}{,2}_*.log`, `case7_o_posix3_24605.log`,
  `case7_o_{volnc,vfdnc}_*.log` (jobs 24592–24605)
- verification: `~/bin/case7_verify_1011.sbatch` → `~/logs/case7_verify_1011_24591.log`
- read-path A/B: `~/bin/case7_payload_cmp_1011.sbatch` → `~/logs/case7_payload_cmp_1011_24606.log`
- UCX diagnosis: `~/bin/neuroh5_hang_diag.sbatch` → `~/logs/neuroh5_hang_diag_24590.log`
