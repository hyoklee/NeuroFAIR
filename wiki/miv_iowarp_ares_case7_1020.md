# MiV case 7 on the updated stack (2026-10-20) — MiV-Simulator 0.4.1, neuroh5 upstream, clio-core `dev` `e7e409d0` (ares)

Re-run of the case 7 backend matrix after pulling and rebuilding every piece
of code the MiV simulation uses on ares. The questions: does the new
application stack change the native baseline, and what do the latest
clio-core changes do to each I/O route?

**Result: the stack update is performance-neutral (native 1847.1 s, inside
the 1835.6–1864.9 s range of the last two rounds). The clio-core numbers are
unchanged: VFD +0.5 % and VOL cache-off +0.7 % (at native), VOL +4.3 %,
clio-fs via the now-upstream MPI-IO adapter +4.5 %. The first run of the
VFD's opt-in read tier costs +4.2 %, the same as the VOL cache. Separately,
the timing config has always simulated a silent network (see
[below](#the-timing-workload-simulates-a-silent-network)), so "identical
objectives" across backends is weak evidence; the read-path checks are the
real correctness evidence.**

## What was updated

| component | before | after | how |
|---|---|---|---|
| MiV-Simulator (`~/MiV-Simulator`, editable install) | `f279b9c` (0.3.0) | **`e78aa8b`** (0.4.1), 13 commits | fast-forward to `GazzolaLab/main`; the 4 local robustness edits (connections / geometry / gap junctions / `np.seterr`) re-applied from a stash. The fork `origin` (hyoklee) was not pushed |
| MiV-Simulator-Cases | `db413ca` | **`619dd1e`** (PR #8, modular optimizer config) | `git pull` |
| neuroh5 (`~/neuroh5`, `build/` + `io.so` copied to the miv env) | branch `vol-cte-case7-fix` `10d9881` | **`master` = `iraikov/master` `f784c4a`** (0.2.0), 21 commits | upstream now carries its own version of our VOL fixes (`H5Lexists`, version-gated `H5Literate2`), so the case-7 branch is superseded. Rebuilt with `~/bin/neuroh5_ares`; previous build saved to `~/neuroh5_prev_20261020/` |
| clio-core (`~/src/iowarp/clio-core/dev/conda`) | `5ac2747d` | **`e7e409d0`** (2026-10-19), 84 commits | already pulled by the daily cron; built into a new prefix `~/core_dev_miv_1020/install` with `CLIO_CTE_ENABLE_MPIIO_ADAPTER=ON` |
| NeuroFAIR | — | already up to date | |

Relevant clio-core commits since `5ac2747d`: **#1272 MPI-IO adapter HDF5
compatibility (merged)**, `c6819155` VOL tier chunk default 64 KiB,
`5c78b5ee` VOL admits on read miss, `7e0d7e77` VFD fsync at close is now
opt-in, plus clio-fs inode-table and directory-journal work.

### Config migration (required)

MiV-Simulator PR #104 replaced the flat `objective_names` /
`target_populations` keys with a modular `Network Optimization:` block
(Features / Objectives / Constraints). `load_network_opt_config` has **no
fallback**: an old-style config yields an empty optimization config. New
files `config/optimize_network_ares_{io,cfs}_v2.yaml` take the
Features/Objectives/Constraints block verbatim from upstream
`optimize_network.yaml` (5 objectives, 6 constraints, 15 features). Their
`kwargs` are **byte-identical** to the old `optimize_network_ares_{io,cfs}.yaml`
(`tstop=200`, same paths). Wall time is therefore comparable with earlier
rounds, but objective values are not.

Harness change: `~/bin/run_case7_dev.sbatch` now forwards
`CLIO_VFD_READ_TIER` to the ranks (backup `run_case7_dev.sbatch.bak.20261020`).

## Correctness gate (job 24969, `ares-comp-16`)

`~/bin/case7_verify_1020.sbatch`, new builds:

- h5py whole-file read of `Microcircuit_Small_connections.h5` (60 datasets,
  39.2 MB): md5 `fddb49e9…` **identical** native / VOL / VOL warm / VFD / VFD warm.
- neuroh5's C++ read API with the **new** neuroh5: output identical
  native / VOL / VFD. `CLIO_VFD_DEBUG` shows the VFD driving the reads (82 ops).
- **Stock upstream MPI-IO adapter** (no local patch any more): h5py
  `driver="mpio"` on Connections np=1/np=4 and Cells np=4, and neuroh5 bulk
  reads (`8b53db84…`, which go through non-contiguous file views), are all
  identical to native. `strace` shows 0 `statfs(clio::)` and 0 `preadv`, so
  the adapter served the reads. The POSIX-only control still fails as it
  should (OMPIO `preadv` EBADF).

## Result — end-to-end (1 node, `mpirun -np 17`, 13 evaluations each)

Same workload shape as every round since September: `optimize-network`,
`Microcircuit_Small`, `NINIT=1 POP=2 SURR=megp` → 13 GLP evaluations, then the
expected megp/GPyTorch stop (`exit=1`, identical in every run). All 10 runs
completed exactly 13 evaluations. `--exclusive`, `ares-comp-18/19` excluded.

| Config | jobs (node) | reps (s) | mean | per eval | vs baseline | Oct 11–13 |
|---|---|---:|---:|---:|---:|---:|
| **baseline** (native HDF5) | 24970 (c32), 24974 (c29) | 1861.9 / 1832.2 | **1847.1 s** | 142.1 s | — | 1864.9 / 1835.6 s |
| HDF5 **VFD** (`clio_vfd`, cache on) | 24972 (c27), 24976 (c32) | 1855.5 / 1857.7 | **1856.6 s** | 142.8 s | **+0.5 %** | +0.6 % |
| HDF5 **VFD**, `CLIO_VFD_READ_TIER=1` *(new)* | 24978 (c27) | 1924.0 | 1924.0 s | 148.0 s | **+4.2 %** | not measured |
| HDF5 **VOL** (`clio`, cache on) | 24971 (c16), 24975 (c30) | 1927.5 / 1925.5 | **1926.5 s** | 148.2 s | **+4.3 %** | +4.5 % |
| HDF5 **VOL**, `CLIO_VOL_CACHE=0` | 24979 (c30) | 1859.4 | 1859.4 s | 143.0 s | +0.7 % | +1.0 % |
| **clio-fs via MPI-IO adapter** (`IOWARP=5`, stock) | 24973 (c28), 24977 (c29) | 1926.8 / 1932.7 | **1929.7 s** | 148.4 s | **+4.5 %** | +5.0 % (patched) |

Noise floor between same-config reps on different nodes is about ±1 %
(baseline reps differ by 1.6 %). The POSIX adapter alone (`IOWARP=1`) was not
re-run, because without `clio::` paths it routes no I/O
([correction of 2026-10-12](miv_iowarp_ares_case7_dev_1011.md)).
Data: `~/MiV-Simulator-Cases/7-optimization/results/comparison_case7_1020.csv`.

### Per-phase proxies (rank 0)

| backend | configure 33,436 synapses, gid 10 | cells created |
|---|---|---|
| native | 8.51–8.66 s | 1.87–2.01 s |
| VFD | 8.56–8.68 s | 1.90–1.98 s |
| **VFD read tier** | **9.02–9.09 s** | 2.02–2.12 s |
| **VOL**, cache on | **9.00–9.53 s** | 1.97–2.05 s |
| VOL, cache off | 8.56–9.12 s | 1.94–1.97 s |
| **clio-fs / MPI-IO** | **9.00–9.59 s** | **1.70–1.75 s (−9 %)** |

The three ~+4.5 % configurations share one signature: the many-small-read
synapse/connectivity phase is about 0.45 s (+5 %) slower per evaluation.
Every configuration that keeps reads on the native path (VFD default, VOL
cache-off) sits in the native band. Over 13 evaluations that per-phase gap
accounts for most of the end-to-end difference. clio-fs still serves the
bulk cell reads ~9 % faster from its RAM tier, but that is too small a share
of the run to matter.

## Reading

1. **Application stack update: neutral.** MiV-Simulator 0.4.1 (new modular
   objectives) and neuroh5 0.2.0 change no measurable cost. The native
   baseline (1847.1 s) and synapse phase (8.51–8.66 s) fall inside the
   previous rounds' ranges.
2. **VFD (default): still free (+0.5 %).** Making fsync-at-close opt-in
   (`7e0d7e77`) does not matter for this read-only workload.
3. **VFD read tier: +4.2 %, the first measurement.** Turning on reads from
   the CTE moves the VFD from the native band to exactly the VOL-cache band
   (synapse phase 9.02–9.09 s). Serving case 7's small reads from the tier
   costs the same whichever adapter does it.
4. **VOL: +4.5 % → +4.3 %, within noise.** The 64 KiB tier chunk default
   and the new read-miss admission rule do not move case 7. Cache-off is
   still at native.
5. **MPI-IO adapter: stock upstream now works.** With #1272 merged it serves
   parallel HDF5 and neuroh5's view-based reads with no local patch. Its
   cost (+4.5 %) is the same as the patched build (+5.0 %) and the OMPIO
   route (+4.8 %).
6. As in every ares study, **nothing in clio-core makes case 7 faster**: the
   run is compute-bound and its total input is ~92 MB. Any path that serves
   reads from the CTE tier pays ~4.5 % in per-operation overhead and has no
   slow store to win back from.

## The timing workload simulates a silent network

Checking the result files turned up a problem with all case-7 rounds on
ares, not only this one. **Every evaluation has every feature = 0**
(`n_active = 0` for PYR, PVBC and OLM). This was also true of the old-code
runs of 2026-10-13 (jobs 24631/24633).

**Cause:** the `_io`/`_cfs` timing configs set `tstop: 200.` ms to keep runs
short, but every STIM spike train in
`Microcircuit_Small_input_spikes.h5` starts between **256 and 407 ms** (e.g.
STIM gid 4: 256.3–1174.0 ms, gid 0: 406.5–1243.6 ms). No input reaches the
network inside the simulated window. Upstream uses `tstop: 1250.`.

Consequences:

- **The timings stand.** The cost is setup (cell and synapse construction
  plus HDF5 reads) and 200 ms of NEURON integration, both performed in full
  whatever the firing. All 11 runs compared (10 new plus 24631) evaluate the
  **same GLP lattice**: per-column sorted parameter md5 `d8c71b1f…` in all
  of them. Only the random pairing of columns into rows differs, as it did
  before.
- **"Bit-identical objectives" claims on earlier pages are weak.** They
  compared all-zero vectors, so they could not detect a wrong read. The
  byte-level read checks (h5py md5, neuroh5 C++ output, MPI-IO smoke) remain
  the evidence that each backend returns correct data, and they pass.
- **To make objectives a real check**, a future round should use
  `tstop ≥ ~1250` (≈6× the integration time per evaluation) or shift the
  input onset into the window, and fix the dmosopt seed so the rows match
  too.

Digest tool: `~/bin/case7_dmosopt_digest.py`. It hashes the sorted per-row
parameters, objectives, features and constraints of a `dmosopt.*.h5`.

## Related pages

- [Case 7 on clio-core `dev` `5ac2747d` — POSIX vs VOL vs VFD](miv_iowarp_ares_case7_dev_1011.md) — previous round
- [Case 7 via the MPI-IO adapter](miv_iowarp_ares_case7_mpiio.md) — the patch that became #1272
- [Case 7 with every input in clio-fs](miv_iowarp_ares_case7_clio_fs.md)
- [clio-fs and the CLIO HDF5 VFD](concept-clio-fs-vs-hdf5-vfd.md)
- [Case 7 feasibility synthesis](miv_iowarp_ares_case7_feasibility.md)

## Artifacts

- builds: `~/core_dev_miv_1020/{build,install}` (log
  `~/logs/build_core_dev_miv_1020.log`); neuroh5 `~/neuroh5/build` (log
  `~/logs/build_neuroh5_20261020.log`), previous build in `~/neuroh5_prev_20261020/`
- configs: `~/MiV-Simulator-Cases/7-optimization/config/optimize_network_ares_{io,cfs}_v2.yaml`
- verification: `~/bin/case7_verify_1020.sbatch` → `~/logs/case7_verify_1020_24969.log`
- matrix: `~/bin/submit_case7_1020.sh` (`SKIP_BASE1=1`; the first baseline
  was submitted alone as 24970) → `~/logs/case7_n_*_249{70..79}.log`,
  results in `~/MiV-Simulator-Cases/7-optimization/results/n_*`
- data: `~/MiV-Simulator-Cases/7-optimization/results/comparison_case7_1020.csv`
