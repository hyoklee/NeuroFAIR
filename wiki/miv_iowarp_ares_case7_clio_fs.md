# MiV case 7 with every input in **clio-fs** (ares, clio-core `dev` `5ac2747d`)

First run of a MiV simulation whose inputs actually **live in clio-fs** — the
filesystem ChiMod over the CTE (see [clio-fs and the CLIO HDF5 VFD](concept-clio-fs-vs-hdf5-vfd.md)).
Same case-7 workload as the [Oct backend matrix](miv_iowarp_ares_case7_dev_1011.md):
`optimize-network`, 13 GLP initial evaluations, 1 node, `mpirun -np 17`.

**Result: it works end to end, produces bit-identical science, and costs
+4.8 %** — but only after working around three gaps in the POSIX interceptor,
two of which crash MiV out of the box.

## First, a correction: the "POSIX adapter" runs never used clio-fs

The POSIX interceptor routes a path into clio-fs **only if it contains a
`clio::` path component** (`IsPathTracked` → `HasClioPrefix`,
`filesystem_client.h`); every other path falls straight through to libc. No
case-7 config ever used that marker. So the `IOWARP=1` / "CTE POSIX adapter"
numbers on `dev` — September's **+1.0 %** and October's **−0.1 %** — measured
only the cost of *having the interceptor loaded* (plus a running runtime), with
**zero simulation bytes going through clio-fs**. (The May **+10.8 %** used
clio-core v2.0.0, `~/core.iowarp`, whose source is no longer on disk, so its
path-tracking rule could not be checked.) The
September headline "the POSIX adapter got ten times cheaper" is really "the
pass-through interception got cheaper". Notes have been added to those pages.

## How the inputs get into clio-fs

clio-fs is a filesystem, not a cache in front of host files: a `clio::` path *is*
a file stored in CTE blobs (1 MiB pages), and the host path behind it need not
exist (no host directory is created). The tier here is RAM, so the inputs are
**staged per job**, after the runtime starts:

- `~/bin/cfs_stage.py SRC DST` — under `LD_PRELOAD=libclio_cte_posix.so`, copies
  with plain `os.open/read/write` (not `shutil`, which uses `sendfile`/
  `copy_file_range`, neither intercepted), then reads each file back through
  clio-fs and checks md5 + `stat` size.
- Staged: the 14 top-level files of `datasets/` **and** the 3 files of
  `datasets/Microcircuit_Small/` — the latter are what the network really reads
  (`MiV_Cells_Microcircuit_Small_20220410.h5` 36.5 MB,
  `MiV_Connections_…` 8.9 MB, gap junctions). **92 MB in ~0.9 s**, all
  byte-exact.
- Config `config/optimize_network_ares_cfs.yaml` = `optimize_network_ares_io.yaml`
  with `dataset_prefix` and `spike_input_path` pointing at
  `/home/hyoklee/MiV-Simulator-Cases/7-optimization/clio::datasets`.
- Harness: `~/bin/run_case7_dev.sbatch` **`IOWARP=4`** (stage, then run under
  the interceptor). It **aborts rather than falling back to native** if the
  runtime, adapter or staging fails.

## Interceptor gaps found (and what MiV needed)

`libclio_cte_posix.so` interposes `open/open64/creat/read/write/pread/pwrite/
lseek/stat/fstat/__xstat*/ftruncate/fsync/close/unlink/remove/flock`. clio-fs
descriptors are numbered from **8192** and are not kernel fds, so any other call
on them reaches the kernel and fails. Found with `strace` (intercepted calls
never reach the kernel, so anything on a `clio::` path or an fd ≥ 8192 that
*does* appear leaked past the interceptor) — `~/bin/cfs_romio_diag.sbatch`:

| gap | who calls it | effect on MiV | workaround |
|---|---|---|---|
| **`preadv`** | Open MPI OMPIO `fbtl_posix` (`mca_fbtl_posix_preadv_generic`) | **crash** at cell creation: `(p)readv: Bad file descriptor` → `H5Dread` fails → neuroh5 assertion, exit 134 | `OMPI_MCA_fbtl_posix_max_gap_size=1073741824` keeps OMPIO on its data-sieving path, which uses plain `pread` |
| **`clio::` vs ROMIO filename syntax** | ROMIO (`OMPI_MCA_io=romio341`) parses `prefix:` in a filename as a file-system-type selector (`ufs:`, `nfs:`…) | **`MPI_File_open` fails** (`MPI_ERR_IO`) before any `open` is issued | none short of a path-syntax change; use OMPIO |
| `fcntl` (`F_SETLK`/`F_SETLKW`) | OMPIO byte-range locking | ~49 k failed (EBADF) lock calls per file read; **ignored on reads** | none needed for read-only; *unlocked* for concurrent MPI-IO writes |
| `statfs` | OMPIO file-system detection | ENOENT on the (non-existent) host path; tolerated | none needed |

Not intercepted and not hit here, but likely to matter elsewhere: `openat`,
`statx`, `lstat`, `access`, `readv/writev`, `pwritev`, `mmap`, `sendfile`,
`copy_file_range`. clio-core's **MPI-IO adapter**
(`CLIO_CTE_ENABLE_MPIIO_ADAPTER`, intercepting `MPI_File_*` and so sidestepping
OMPIO's syscalls) is the designed answer for MPI apps; it is **off** in this
build. **Tested since** ([MPI-IO adapter page](miv_iowarp_ares_case7_mpiio.md)):
stock it crashes parallel HDF5 and ignores file views; patched, it runs case 7
at +5.0 %.

## Correctness

- Before the full run: h5py whole-file read (60 datasets, md5 `fddb49e9…`) and
  neuroh5's C++ API at np=1 and np=4 — **identical** from clio-fs and native
  (`~/bin/cfs_smoke.sbatch`, job 24610; np=4 output identical once rank print
  order is ignored).
- Full simulation: all **117 objective values** (13 evaluations × 9 objectives)
  are **identical** between native, the matched baseline, both clio-fs runs and
  the VOL run (same md5 of the sorted `objective …:` lines).

## Result — end-to-end (1 node, 13 evaluations)

The gap-size knob changes the *native* MPI-IO path too, so the baseline was
re-run with the same setting:

| Config | reps (s) | mean | vs matched baseline |
|---|---:|---:|---:|
| baseline, `max_gap_size=1G` | 1847.0 / 1835.1 | **1841.0 s** | — |
| **clio-fs** (`IOWARP=4`), `max_gap_size=1G` | 1933.2 / 1925.0 | **1929.1 s** | **+4.8 %** |
| *(reference)* baseline, default OMPIO | 1864.4 / 1865.4 | 1864.9 s | +1.3 % |

The knob itself is within noise for native (−1.3 %, identical synapse-phase
times). Staging adds ~0.9 s per job (0.05 %) and is included in the clio-fs wall
time.

### Where the +4.8 % comes from

| phase (rank 0) | native | clio-fs |
|---|---|---|
| cells created (reads Cells file trees/coords) | 1.90–1.95 s | **1.71–1.76 s (−9 %)** |
| configure 33,436 synapses for gid 10 | 8.54–8.63 s | **9.00–9.57 s (+6 %)** |

Bulk cell reads are a little *faster* out of clio-fs's RAM tier (the
zero-IPC shared-memory read fast path is active: `clio-fs: serving reads from
shared memory`), but the synapse/connectivity phase — many small attribute
reads per cell — is slower, and it dominates. The likely contributors are the
per-request clio-fs overhead and OMPIO's failed `fcntl` lock calls (seen in the
strace diagnostic), but this was not profiled further. *(Update 2026-10-13: the
`fcntl` part is ruled out — the [MPI-IO adapter route](miv_iowarp_ares_case7_mpiio.md)
issues no `fcntl` at all and costs the same, +5.0 %.)* The same shape as the
[VOL cache](miv_iowarp_ares_case7_dev_1011.md) (+4.5 %): a per-operation
cost spread over a compute-bound run whose total I/O is tiny.

## Bottom line

1. **MiV case 7 runs entirely out of clio-fs** on clio-core `dev` `5ac2747d`,
   with bit-identical objectives — but **only with OMPIO and
   `OMPI_MCA_fbtl_posix_max_gap_size` raised**. Out of the box it crashes
   (`preadv` not intercepted), and under ROMIO it cannot open the file at all
   (`clio::` is ROMIO filename-prefix syntax).
2. **Cost: +4.8 %**, from small-read overhead in the connectivity phase; bulk
   reads are ~9 % faster. As in every ares study, the workload is compute-bound,
   so there is nothing for clio-fs to win.
3. **The Sept/Oct "POSIX adapter" numbers measured pass-through only.** This page
   is the first measurement of MiV I/O actually served by clio-fs.
4. Upstream asks for clio-core: intercept `preadv`/`pwritev`/`readv`/`writev`
   and `fcntl` locks (or return EBADF-safe emulation), and reconsider the
   `clio::` marker's collision with ROMIO's `prefix:` syntax. MPI-IO adapter:
   see [its page](miv_iowarp_ares_case7_mpiio.md).

## Related pages

- [Case 7 from clio-fs via the MPI-IO adapter](miv_iowarp_ares_case7_mpiio.md) — same inputs, `MPI_File_*` interception instead of OMPIO + POSIX

- [Case 7 on clio-core `dev` `5ac2747d` — POSIX vs VOL vs VFD](miv_iowarp_ares_case7_dev_1011.md) — same day, same build, same workload
- [clio-fs and the CLIO HDF5 VFD](concept-clio-fs-vs-hdf5-vfd.md) — what clio-fs is
- [Case 7 on clio-core `dev` `3357ac99` (Sept 2026)](miv_iowarp_ares_case7_dev.md)
- [Case 7 feasibility synthesis](miv_iowarp_ares_case7_feasibility.md)

## Artifacts

- harness: `~/bin/run_case7_dev.sbatch` `IOWARP=4`; config
  `~/MiV-Simulator-Cases/7-optimization/config/optimize_network_ares_cfs.yaml`;
  stager `~/bin/cfs_stage.py`
- logs: `~/logs/case7_o_cfsg{,2}_*.log` (jobs 24619, 24620), matched baselines
  `~/logs/case7_o_baseg{,2}_*.log` (24621, 24622); failed runs
  `case7_o_cfs{,2}_*.log` (24612/3, `preadv`), `case7_o_cfsr_24614.log` (ROMIO)
- diagnostics: `~/bin/cfs_smoke.sbatch` (job 24610),
  `~/bin/cfs_romio_diag.sbatch` (job 24618, strace in `~/logs/cfs_romio_diag/`)
