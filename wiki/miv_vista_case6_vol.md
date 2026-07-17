# MiV-Simulator Case 6 on TACC Vista (ARM) — clio-core HDF5 **VOL** connector vs native vs POSIX

End-to-end `run-network` benchmark of **case 6 (gap junctions)** on the TACC
**Vista** supercomputer (NVIDIA **Grace**, ARM **aarch64**), using the clio-core
**HDF5 VOL connector** (`HDF5_VOL_CONNECTOR=clio`) — the configuration requested
in `NeuroFAIR.md`. Complements the POSIX/ELF Vista study
([miv_vista_case6.md](miv_vista_case6.md)) and the x86 Frontera VOL study
([miv_frontera_case6_vol.md](miv_frontera_case6_vol.md)). CPU-only (`gg`
partition, Grace-Grace); no GPUs. Allocation `IBN22011`. Driver scripts in
`vista/bin/`: `build_clio_vol.slurm`, `run_case6_vol.slurm`, `clio_vol_env.sh`,
`sitecustomize.py`, `rebuild_neuroh5.slurm`, `vol_smoke.{py,sh}`.

## Headline result

The clio **VOL** connector runs case 6 **end-to-end on Vista/ARM, including
writing the results file**, at **native performance within noise**, and its
output is **bit-for-bit identical to native** (`h5diff` = 0 differences). This
goes **past the Frontera VOL study**, which listed both the results-file
write-open *and* multi-rank collective reads as unresolved: on Vista, with two
targeted MiV-stack fixes (below), both work — case 6 completes at **4 and 8
ranks**, reads *and* writes through the VOL.

Unlike the POSIX interceptor (which routes only paths carrying a literal
`clio::` marker, so neuroh5's MPI-IO reads bypass it entirely), the VOL
intercepts at the HDF5 dataset-I/O layer and **actually sees neuroh5's reads**,
mirroring whole-dataset atomic transfers to the CTE RAM tier. It is transparent
and correct — just not faster, because case 6 is compute-bound and read-once
(the same conclusion reached on every platform).

## Environment

| Component | Version / build | |
|-----------|-----------------|--|
| toolchain | `gcc/14.2.0` + `openmpi/5.0.5` | Vista modules |
| HDF5 | **`phdf5/1.14.6`** parallel (`libhdf5.so.310`) | module; VOL links THIS, not conda's 2.1.0 |
| neuroh5 | 0.1.18, branch `fix/vol-h5literate2` | rebuilt this session (VOL-neutral iteration) |
| miv_simulator | 0.3.0 (`vista-arm-fix`) | + `sitecustomize.py` VOL shim |
| h5py / mpi4py | 3.16.0 (parallel) / 4.1.2 | on phdf5/1.14.6, openmpi/5.0.5 |
| NEURON | 9.0.1 | mechanisms auto-compiled |
| clio-core | `dev` `b5c68c5e` | VOL adapter `libclio_hdf5_vol.so`, connector name `clio` |

The VOL is built with `-DCLIO_CTE_ENABLE_HDF5_VOL=ON -DHDF5_DIR=<phdf5 cmake>` so
the connector links the **same** HDF5 (`1.14.6`, soname 310) that h5py/neuroh5
link — a VOL connector is ABI-coupled to the `libhdf5` of the app that
`dlopen`s it (`ldd libclio_hdf5_vol.so` → `libhdf5.so.310` from the module).
Loaded purely via `HDF5_VOL_CONNECTOR=clio` + `HDF5_PLUGIN_PATH` — no
`LD_PRELOAD` of an interceptor, no application source change. A `clio_run`
runtime is started per job for the CTE cache path.

## 1-node three-way result (4 ranks, tstop=50, mean; SAME dev build)

| Phase | native (3 reps) | POSIX (2) | clio VOL (3) |
|-------|----------------:|----------:|-------------:|
| created cells (read ~36 MB) | 2.01 s | 2.13 s | 1.95 s |
| connected cells (read + wire) | 45.66 s | 46.38 s | 46.02 s |
| ran simulation (50 ms compute) | 14.78 s | 14.62 s | 14.99 s |
| **total wall** | **70.10 s** | **71.21 s** | **70.88 s** |

Δ vs native on the read-heavy *connected* phase: **POSIX +1.57 %, VOL +0.79 %**;
total POSIX +1.59 %, VOL +1.11 % — all inside the ~1–2 % run-to-run noise. All
three I/O paths perform **the same**. Jobs: native 837944/838383/838385, VOL
838332/838384/838386, POSIX 838390/838391.

## Multi-rank collective reads through the VOL — no deadlock on Vista

| config | connected | ran | total | rc |
|--------|----------:|----:|------:|---:|
| VOL, 1 node × 4 ranks | 46.02 s (mean) | 14.99 s | 70.88 s | 0 |
| VOL, 1 node × 8 ranks | 27.20 s | 9.66 s | 46.88 s | 0 |

neuroh5's projection reads are MPI-collective (all ranks participate). Both 4-
and 8-rank VOL runs complete cleanly (`--io-size=1`, OpenMPI 5.0.5). The **NR=8
collective-read deadlock reported on Frontera** (MPICH container) **does not
reproduce here** — likely MPI-implementation / `io-size` dependent. Job 838399.

## Two VOL-incompatibilities in the MiV stack — found, fixed, neither a connector bug

HDF5 enforces both restrictions in its public-API / VOL-dispatch layer **above**
the connector, so the clio connector cannot address them and stays pristine.

### 1. neuroh5 used deprecated version-1 HDF5 iteration APIs

HDF5 forbids the *version-1* link/group iteration APIs (`H5Literate1`,
`H5Literate_by_name1`, `H5Gget_objinfo`) on any **non-native** VOL — the block
fires before the connector runs. Proven with a C probe on the real Cells file:
through the connector `H5Literate2`/`H5Gget_info`/`H5Gget_num_objs` return rc=0
and iterate correctly, but `H5Literate1`/`H5Literate_by_name1` return **rc=−1**.
The deployed `neuroh5/io.so` predated the fix and aborted in
`read_population_names`.

**Fix** — neuroh5 branch `fix/vol-h5literate2`, rebuilt + reinstalled via
`bin/rebuild_neuroh5.slurm`: `H5Literate`→`H5Literate2` (commit 28b6d83) and the
three `cell_attributes.cc` attribute-enum guards `H5Gget_objinfo`→VOL-neutral
`H5Lexists` (commit b62051b — the same fix landed on Frontera).
`H5Gget_num_objs` works through the VOL and is left unchanged.

### 2. h5py `File(path, "a")` create-fallback skipped through the VOL — **now resolved**

`h5py.File(path, "a")` opens read-write and, on failure, creates the file — but
only when the failure surfaces as `FileNotFoundError`, which h5py derives from
the top HDF5 error frame. Through a non-native VOL, HDF5's H5VL layer wraps the
underlying `errno=2` as a generic *"Virtual Object Layer / Can't open object"*
error, so h5py raises plain `OSError`, skips the create fallback, and
`File(path, "a")` fails on a not-yet-existing file. This is exactly the
"results-file write-open fails" limitation left open on Frontera; it breaks
MiV's `io_utils.mkout` and ~10 other `h5py.File(..., "a")` sites.

**Fix** — `bin/sitecustomize.py`, a VOL-compatibility shim auto-imported at
interpreter startup (`PYTHONPATH=vista/bin`, added by `clio_vol_env.sh` and
forwarded to MPI ranks). When mode is `"a"` and the file is absent it creates
with `"x"` (a lost create race falls back to append). It is a **no-op unless
`HDF5_VOL_CONNECTOR` is set**, so native/POSIX runs are unaffected. With it, the
NR=4/NR=8 VOL runs write their full spike/intracellular/LFP results.

## Correctness

`h5diff` native-vs-VOL results file → **0 differences**; identical 167 881-byte
size; `h5ls`/`h5dump` with the connector *off* read the VOL-written file as a
valid standard native HDF5 file (H5Types, LFP electrodes, per-population spike +
intracellular datasets). The native file is always authoritative; CTE mirroring
is additive.

## Conclusion

- Case 6 **builds and runs end-to-end on Vista/ARM through the clio HDF5 VOL
  connector**, reading *and* writing, at 4 and 8 ranks, with bit-identical output.
- **native ≈ POSIX ≈ VOL within noise** — compute-bound, read-once; no I/O
  speedup expected or seen, consistent with all prior MiV studies.
- Advances past Frontera: the h5py `"a"` write-open and multi-rank collective
  reads (open there) both work here, via a neuroh5 rebuild + a MiV-side h5py
  shim. **clio-core `dev` stays pristine** — both fixes are outside it.

Full summary + reproduce steps: `vista/PERFORMANCE_VOL.md`. Raw timings:
`/scratch/11623/hyoklee/miv/results/case6_vol_timing.csv` (native/VOL) and
`case6_timing.csv` (POSIX).
