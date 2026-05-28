# MiV case 7 (optimization) with FULL CA1 data — IOWarp CTE vs native HDF5 (ares)

After the [small-circuit case 7](miv_iowarp_ares_case7.md) found IOWarp CTE
~11% slower (compute-bound, tiny I/O), this page tests the same workload
against the **full CA1 microcircuit** staged via Globus
([read-bound benchmark](miv_iowarp_ares_caseread.md)) to see whether the
larger I/O changes the verdict.

## Workload

`optimize-network` (dmosopt, NSGA-II) under distwq.
- Cell Data: `MiV_Cells_Microcircuit_20220412.h5` (19 GB)
- Connection Data: `MiV_Connections_Microcircuit_20220412.h5` (7 GB)
- Geometry: full (80K PYR + 1474 PVBC + 438 OLM + 1000 STIM)
- 1 node × 5 tasks (1 controller + 1 worker × 4 ranks)
- NINIT=1, POP=1, NGEN=1 (smallest non-empty config)

Artifacts:
[`config/Network_Clamp_Microcircuit_Full*.yaml`](https://github.com/iraikov/MiV-Simulator-Cases/blob/main/7-optimization/config),
`config/Geometry_Full.yaml`, `config/optimize_network_ares_full.yaml`,
`config/cell_selection_full.yaml`, `~/bin/run_case7.sbatch`.

## What didn't work, and why

### 1. Whole-population load: OOM at PYR forest read (~46 GB ceiling)

With Geometry_Full and the Bilash/PRN templates, the first thing
`init_network` does is `scatter_read_trees` for each population. OLM and PVBC
complete; PYR (80K cells × ~800 KB tree data ≈ 64 GB) immediately OOMs the
node. Same behavior with 17 tasks (≈3 GB/rank) and 5 tasks (≈11 GB/rank);
both die at the same log line:

```
INFO:miv_simulator.network:*** Reading trees for population PYR
prterun noticed that process rank X exited on signal 9 (Killed).
```

(Jobs 20439/20440 with 17 tasks, 20441/20442 with 5 tasks, 20443/20444 with
the reduced `Geometry_Mid.yaml`; the Geometry counts turn out to be ignored
— actual per-population counts come from the HDF5 `/H5Types/Populations`
registry, e.g. 1474 PVBC observed against the 147 declared in Mid.)

The IOWarp NVMe run consistently died **~70 s earlier than baseline** (308 vs
377 s on the 17-task config, 254 vs 325 s on 5-task) at the same log line.
This is **not** a CTE speed-up — both runs progressed through the same number
of cells; the chimaera daemon's resident set pushes the cgroup over its
allotment sooner, so the OOM-kill fires earlier. NVMe tier blob: 0 bytes used.

### 2. cell_selection (subsetting ~50 cells to bypass PYR OOM)

`Env(cell_selection_path=...)` loads a YAML mapping `pop -> [gids]` and lets
`make_input_cell_selection` treat non-selected cells as spike-source proxies,
which should skip the full PYR forest read. Two distinct code-path bugs in
MiV-Simulator block this on full data:

- **PRN/Bilash Python templates** (jobs 20445/20446):
  `make_cell_selection` (network.py:1102) calls `template_class(gid=..., pop_name=..., env=..., param_dict=...)`,
  but `PRN.__init__` takes only `params=None`:
  `TypeError: PRN.__init__() got an unexpected keyword argument 'gid'`.
- **HOC templates** (jobs 20447–20456): `connect_cell_selection` →
  `init_syn_mech_attrs` (synapses.py:3576) accesses `cell.mech_dict`, which
  doesn't exist on the raw `hoc.HocObject` returned for HOC cells:
  `AttributeError: 'hoc.HocObject' object has no attribute 'mech_dict'`.

Switching to HOC also requires copying `OLMCell.hoc`/`PVBasketCell.hoc`/`PoolosPyramidalCell.hoc`
and the `ca1_Poolos` mod tree from case 4; ca1_Poolos's `cal.mod` then
multiply-defines the same `_cal` symbols as ca1_Bilash's `cal2.mod` so
`nrnivmodl` fails to link unless the Bilash/PinskyRinzel dirs are stashed
elsewhere — leaving you needing a build that supports neither full-circuit
nor the original case-7 PR optimization.

## What this proves regardless of OOM/crash

The NVMe tier blob was checked at the end of **every** run above. In all
cases — successful inits, OOM crashes, MiV code-path crashes — `du -sh
/mnt/nvme/.../cte_tier1_node0` reports **0 bytes used**:

```
=== caseread_staged: ... ===
IOWarp: NVMe tier contents at stop:
0       /mnt/nvme/hyoklee/cte_case7sel
```

The LD_PRELOAD POSIX adapter only intercepts paths containing the `clio::`
marker (`core/context-transfer-engine/adapter/filesystem/filesystem.h:87`,
`kClioPrefix`). MiV-Simulator constructs HDF5 paths via
`Env.data_file_path = os.path.join(dataset_path, "Cell Data")`
([`miv_simulator/env.py:358-365`](https://github.com/MiV-Sim/MiV-Simulator/blob/main/src/miv_simulator/env.py))
— unprefixed. The adapter never sees the reads, the tier is never populated,
and the adapter contributes only LD_PRELOAD intercept overhead.

So even if the full-circuit run had completed, the comparison would be
**bounded above by the plain caseread result**: CTE = baseline within 0.05%
on read iterations, with 0 bytes in the tier
([details](miv_iowarp_ares_caseread.md)). The
[caseread followup](miv_iowarp_ares_caseread.md#follow-up-explicit-pre-staging-into-a-cte-managed-path)
showed that even when you *do* drive bytes into the tier by adding the
marker, the read-back path is ~4× *slower* than NFS direct and emits
`Tag GetBlob failed for page N` errors at the file tail
([upstream issue #470](https://github.com/iowarp/clio-core/issues/470)).

## Bottom line

To make the case-7 / full-CA1 / IOWarp comparison meaningful, two
independent changes are needed:

- **MiV-Simulator**: either (a) emit `clio::`-prefixed paths when the adapter
  is loaded, or (b) link against a CTE-aware HDF5 VFD so the marker isn't
  needed at the path level; AND
- **clio-core**: fix the `GetBlob failed for page N` regression in the
  staged-read path ([#470](https://github.com/iowarp/clio-core/issues/470)),
  AND make the staged-read throughput competitive with NFS direct.

Neither is currently present on the ares build. Without both, the answer
to "can IOWarp/clio-core help case-7 performance?" on ares with full CA1
data is the same as on small data:
**no measurable benefit** (and a small adapter overhead penalty).

## Jobs logged

| job | TAG | config | result |
|---|---|---|---|
| 20436 | full_base | Geometry_Full × Bilash | `cell_populations.cc:72` H5Fopen — dataset_path Microcircuit/ missing |
| 20437 | full_iow_nvme | same + LD_PRELOAD | PMIx `/tmp/ompi.PID` no space |
| 20439/20440 | full_base2/iow_nvme2 | + HDF5_USE_FILE_LOCKING=FALSE + TMPDIR=~/tmp/jobs + Microcircuit/ symlinks | both OOM at PYR; iow died 36s earlier |
| 20441/20442 | full_base3/iow_nvme3 | NTASKS=5 | both OOM at PYR; iow died 70s earlier (308 vs 377 s) |
| 20443/20444 | mid_base/iow_nvme | Geometry_Mid (counts ignored) | same OOM, iow 70s earlier |
| 20445/20446 | sel_base/iow_nvme | + cell_selection_full.yaml | `PRN.__init__() got 'gid'` |
| 20447/20448 | sel_base2/iow_nvme2 | Network_Clamp HOC templates | `OLMCell.hoc` not found |
| 20449/20450 | sel_base3/iow_nvme3 | + copied HOC templates | `ch_KvAolm` not a MECHANISM |
| 20451/20452 | sel_base4/iow_nvme4 | + ca1_Poolos+ca3_SM mod dirs | nrnivmodl link: `cal.mod` vs `cal2.mod` multiple definitions |
| 20453/20454 | sel_base5/iow_nvme5 | `_stashed` inside mechanisms/ — still scanned | same link failure |
| 20455/20456 | sel_base6/iow_nvme6 | Bilash/PinskyRinzel stashed outside | base: `'hoc.HocObject' object has no attribute 'mech_dict'`; iow: HOC load fail |

Logs: `~/logs/case7_c7*_204{36..56}.log`. NVMe tier at stop = 0 bytes in
every IOWarp run.
