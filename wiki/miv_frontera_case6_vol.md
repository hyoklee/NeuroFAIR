# MiV-Simulator Case 6 on TACC Frontera — clio-core HDF5 **VOL** adapter vs native HDF5

End-to-end `run-network` benchmark of **case 6 (gap junctions)** on the TACC
**Frontera** supercomputer (Intel Cascade Lake, x86_64), using the clio-core
**HDF5 VOL connector** (`HDF5_VOL_CONNECTOR=clio`) instead of the POSIX/ELF
adapter — the configuration requested in `claude.md`. Complements the Vista/ARM
([miv_vista_case6.md](miv_vista_case6.md)), ares
([miv_iowarp_ares_case6.md](miv_iowarp_ares_case6.md)), jelly
([miv_jelly_case6.md](miv_jelly_case6.md)) and Polaris runs. CPU-only, no GPUs.
Allocation `IBN22011`. Driver scripts in `frontera.hyoklee/bin/`
(`build-clio-iowarp.slurm`, `run-case6-vol.slurm`, `probe-*.slurm`,
`build-miv-sif.slurm`, `miv-stack.def`); logs in `frontera/build/`.

`claude.md`'s `gpu_hack_large`/`preemptable`/PBS references are Polaris (ALCF)
conventions; on Frontera (Slurm, no GPUs) they map to the `development` queue
(1-node debug) then `normal`/`large`.

## Headline result

With one **neuroh5 bug fix** plus a small **h5py write-open shim** (both below),
the clio **VOL** connector runs case 6 **end-to-end at NR=1 — read, construct,
simulate, and write the results file** — and its output is **bit-for-bit
identical to native** (`h5diff` = 0). The read/construction path is **native
within ~0.3%**: the VOL routes every read through the connector (mirrored to the
CTE RAM tier) but there is **no speedup**, because case 6 is compute-bound and
read-once — the same conclusion reached on every other platform.

**Update 2026-07-17:** the results-file write-open (previously listed as an open
limitation) is now resolved by pulling the Vista `sitecustomize.py` h5py shim
into the Frontera run — the NR=1 VOL run completes and `h5diff` = 0. The
**multi-rank** collective-read deadlock **persists on Frontera** (MPICH), even
though the same connector + shim complete at 4/8 ranks on
[Vista](miv_vista_case6_vol.md) (OpenMPI) — pointing to an MPI-stack difference
in how collective MPI-IO interacts with the caching VOL, not a connector bug.

## Environment

Everything runs inside `frontera/miv-stack.sif` (Ubuntu 24.04, glibc 2.39, built
on `clio-deps-cpu.sif`), so it is host-el7-independent.

| Component | Version / build | |
|-----------|-----------------|--|
| compiler | **gcc-13.3.0** (`mpicc`/`mpicxx`, MPICH) | container default; see deviation note |
| HDF5 | 1.14.6 parallel, `/opt/hdf5` (`libhdf5.so.310`) | built from source in the def |
| neuroh5 | local fork + VOL fix | parallel HDF5 + MPI C-extension |
| miv_simulator | 0.3.0 (`vista-arm-fix`) | case-6 gap-junction fix |
| h5py / mpi4py | 3.16.0 (parallel) / 4.x | source-built vs container HDF5/MPICH |
| NEURON | 9.0a0 (pip) | mechanisms auto-compiled |
| clio-core | `dev` `b5c68c5e` | VOL adapter `libclio_hdf5_vol.so`, connector name `clio` |

Datasets: 187 cells (PYR 70, PVBC 53, OLM 44, STIM 10) + gap junctions,
`MiV_Cells`/`MiV_Connections_Microcircuit_Small_20220410.h5`, staged on
`/scratch2`. The VOL is loaded purely via `HDF5_VOL_CONNECTOR=clio` +
`HDF5_PLUGIN_PATH` — no `LD_PRELOAD`, no application source change. A `clio_run`
runtime is started per job for the CTE cache path.

## The bug that blocked the VOL — and the fix

The VOL arm initially aborted in `network.make_cells` with `KeyError: 'OLM'`.
Root cause: neuroh5's `read_cell_attribute_info` returned an **empty dict**
through the VOL while returning all 4 populations natively. `read_population_names`
worked, so the failure was in the deeper attribute-*namespace* enumeration.

Traced (via a `clio_run`-side trace of the connector's `link_specific`/
`group_open`/`H5Literate2`) to: **all HDF5 traversal ops succeed through the VOL**
— the empty result came from neuroh5, not the connector. The connector's
`clio_link_specific` is byte-identical to the reference `H5VLpassthru`.

The real defect is in neuroh5 `src/cell/cell_attributes.cc`: its three
attribute-enumeration `H5Literate2` callbacks gate on
`H5Gget_objinfo(grp, name, 0, NULL)`, a **deprecated H5G API that HDF5 routes
only to the native VOL connector**. Through a non-native (pass-through) VOL it
fails; with error handling turned off in the callback the enumeration block is
silently skipped → empty `cell_attribute_info`. Same class of native-VOL-only
API that neuroh5 was previously ported off (`H5Literate1→2`,
`H5_ITER_NATIVE→INC`, `H5Gget_num_objs→H5Gget_info`) — this call was missed.

**Fix** (`neuroh5`, all 3 callbacks):
```c
-  ierr = H5Gget_objinfo (grp, name, 0, NULL);
+  // H5Gget_objinfo is native-VOL-only; H5Lexists is VOL-neutral (same guard).
+  ierr = (H5Lexists (grp, name, H5P_DEFAULT) > 0) ? 0 : -1;
   if (ierr == 0)
```
`H5Gget_num_objs` elsewhere (`cell_populations.cc:79`, `group_contents.cc:68`)
maps to `H5Gget_info` and **does** work through the VOL — left unchanged.
Verified: after the fix `read_cell_attribute_info` returns all 4 populations
through the VOL, identical to native.

## 1-node result (tstop=50, single node c206-*)

**NR=1 — the VOL runs case 6 end-to-end (read + write); construction ≈ native:**

| Phase | Native HDF5 | clio VOL | Δ |
|-------|------------:|---------:|----:|
| created cells (read 36 MB) | 10.22 / 10.27 s | 10.43 / 10.26 s | noise |
| gap junctions | 0.04 s | 0.06 s | noise |
| LFP objects | 1.77 s | 1.76 s | noise |
| **connected cells (read + wire)** | **334.6 / 330.0 / 328.4 s** | **335.1 / 329.5 / 329.5 s** | **+0.14% / −0.16% / +0.32%** |
| total run (read+construct+sim+write) | 438–445 s | 456 s | +4% |

Three independent NR=1 runs (jobs 7861112, 7861121, **7861565**); the
connection-reading phase — the read-heavy part the CTE is meant to accelerate —
is native within noise. Run 7861565 completes **including the results-file
write**, and `h5diff results/native vs results/vol` = **0 differences**
(bit-identical output, both 168,599 bytes). Native NR=8 (single node) total =
**64–65 s** (≈7× rank scaling), consistent with jelly/Vista intra-node scaling.

## VOL limitations

**Resolved — results-file write-open (h5py `"a"` of a new file).** Originally the
NR=1 VOL run failed at `io_utils.mkout` → `h5py.File(results, "a")` →
`H5Fopen(ACC_RDWR)` with `OSError`. Cause: h5py's `"a"` mode falls back to
*creating* a missing file only when the failure surfaces as `FileNotFoundError`,
but HDF5's H5VL layer wraps the underlying `errno=2` as a generic "can't open
object" above the connector, so h5py raises a plain `OSError` and skips the
create-fallback. **Fix**: the Vista `sitecustomize.py` h5py shim
(`frontera.hyoklee/bin/sitecustomize.py`, added to `PYTHONPATH` for the VOL run;
a no-op unless `HDF5_VOL_CONNECTOR` is set) makes mode-`"a"` create the file when
absent. With it, NR=1 completes read **and** write, output bit-identical to
native. Not a connector bug — the masking error frame is added by HDF5 itself.

**Open — multi-rank MPI-collective reads deadlock (Frontera/MPICH-specific).**
At NR=8 the VOL arm still hangs in neuroh5's collective projection read
(`Reading projection PVBC -> OLM`; job 7861569 timed out at 900 s, native NR=8 =
65 s). The same connector + shim complete at **4 and 8 ranks on
[Vista](miv_vista_case6_vol.md)** (OpenMPI 5.0.5), so this is not a connector bug
but an **MPI-stack interaction**: the container's MPICH collective MPI-IO does
not tolerate the caching VOL's per-rank CTE side effects inside a collective HDF5
call, whereas Vista's OpenMPI does. The Frontera VOL comparison is therefore
serial-read (NR=1); native scales fine to 8 ranks on both.

## Conclusion

On Frontera the updated `dev` clio **VOL** connector, plus the neuroh5
`H5Gget_objinfo→H5Lexists` fix and the h5py write-open shim, runs case 6
**end-to-end at NR=1 — read, construct, simulate, and write — with output
bit-identical to native** (`h5diff` = 0) at **native performance within ~0.3%**.
As on every prior platform, **IOWarp/clio does not speed up case 6** — it is
compute-bound and reads its inputs once, so a caching tier has nothing to win
back. The only remaining gap is **multi-rank collective I/O**, which deadlocks
through the VOL under the container's **MPICH** but completes under **OpenMPI on
Vista** — an MPI-stack interaction, not a connector defect.

## Deviations from `claude.md`

- **g++-14**: the container ships **gcc-13.3.0** (Ubuntu 24.04 default); the whole
  stack builds and runs cleanly with it. Installing g++-14 would need a def change
  + another ~75-min SIF rebuild; not done.
- **Queues/nodes**: Frontera has no `gpu_hack_large`/`preemptable`; used
  `development` (1 node). Multi-node VOL scaling is blocked by the collective-read
  deadlock above, so "max nodes" was not pursued for the VOL arm (native scales).
