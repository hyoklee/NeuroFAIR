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

With one **neuroh5 bug fixed** (below), the clio **VOL** connector reads the
**entire case-6 network end-to-end for the first time on any machine** — every
prior study (ares/vista/jelly/Polaris) had the VOL/CTE adapter unable to get
through neuroh5 init. The measured read/construction path is **native within
~0.15%**: the VOL routes every read through the connector (mirrored to the CTE
RAM tier) but there is **no speedup**, because case 6 is compute-bound and
read-once — the same conclusion reached on every other platform, now demonstrated
with the VOL actually doing the full read.

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

**NR=1 (serial reads) — the VOL reads the whole network; construction ≈ native:**

| Phase | Native HDF5 | clio VOL | Δ |
|-------|------------:|---------:|----:|
| created cells (read 36 MB) | 10.22 / 10.43 s | 10.43 / — s | +2% (noise) |
| gap junctions | 0.04 s | 0.06 s | noise |
| LFP objects | 1.77 s | 1.76 s | noise |
| **connected cells (read + wire)** | **334.63 / 330.01 s** | **335.11 / 329.49 s** | **+0.14% / −0.16%** |

Two independent NR=1 runs (jobs 7861112, 7861121); the connection-reading phase
— the read-heavy part the CTE is meant to accelerate — is native within noise.
Native NR=1 total ≈ 439–445 s; NR=8 (single node) total = **64 s** (≈7× rank
scaling), consistent with the near-ideal intra-node scaling seen on jelly/Vista.

## Remaining VOL limitations (characterized, not yet resolved)

1. **Multi-rank MPI-collective reads deadlock.** At NR=8 the VOL arm hangs in
   neuroh5's collective projection read (`Reading projection PVBC -> OLM`) — a
   caching pass-through VOL adding per-rank CTE side effects inside a collective
   HDF5 call breaks MPI collective symmetry. NR=1 (no collective) avoids it. This
   is why the VOL comparison is serial-read; native scales fine to 8 ranks. (Same
   class as the multi-node CTE deadlocks noted on ares/Polaris.)
2. **Results-file write-open fails.** After reading the whole network, the NR=1
   VOL run fails at `io_utils.mkout` → `h5py.File(results, "a")` →
   `H5Fopen(ACC_RDWR)` with `OSError: unable to open`. The VOL does not cleanly
   serve h5py's `"a"` (open-or-create) of a **new** file. The read/construction
   path — the measured quantity — completes before this; the write is a small
   output, so it does not affect the performance conclusion.

## Conclusion

On Frontera the updated `dev` clio **VOL** connector, plus the one neuroh5
`H5Gget_objinfo→H5Lexists` fix, reads case 6 end-to-end (NR=1) at **native
performance within ~0.15%**. As on every prior platform, **IOWarp/clio does not
speed up case 6** — it is compute-bound and reads its inputs once, so a caching
tier has nothing to win back. The VOL adapter is now a viable read path for
single-rank neuroh5 workloads; multi-rank collective I/O and new-file writes
through the VOL remain open.

## Deviations from `claude.md`

- **g++-14**: the container ships **gcc-13.3.0** (Ubuntu 24.04 default); the whole
  stack builds and runs cleanly with it. Installing g++-14 would need a def change
  + another ~75-min SIF rebuild; not done.
- **Queues/nodes**: Frontera has no `gpu_hack_large`/`preemptable`; used
  `development` (1 node). Multi-node VOL scaling is blocked by the collective-read
  deadlock above, so "max nodes" was not pursued for the VOL arm (native scales).
