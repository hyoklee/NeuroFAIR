# Can IOWarp/clio-core help MiV case 7 on ares? — feasibility analysis

After running case 7 in five different regimes — small circuit
([case7](miv_iowarp_ares_case7.md)), full CA1
([case7_full](miv_iowarp_ares_case7_full.md)), read-bound
([caseread](miv_iowarp_ares_caseread.md)), write-bound
([caseio](miv_iowarp_ares_caseio.md)), and core build comparison
([case6](miv_iowarp_ares_case6.md)) — and never measuring a CTE win, this
page records why. It's the synthesis question: **what would have to change
for CTE to beat baseline on `optimize-network`?**

## Three independent blockers — each fatal on its own

### 1. MiV-Simulator emits unprefixed paths

The LD_PRELOAD POSIX adapter only intercepts paths containing the literal
`clio::` marker
(`core/context-transfer-engine/adapter/filesystem/filesystem.h:87`,
`kClioPrefix`). MiV builds HDF5 paths via
`Env.data_file_path = os.path.join(dataset_path, "Cell Data")`
([env.py:358-365](https://github.com/MiV-Sim/MiV-Simulator/blob/main/src/miv_simulator/env.py))
without ever embedding the marker. On every case 7 run logged so far
(jobs 20436–20456, see [case7_full](miv_iowarp_ares_case7_full.md)) the
NVMe tier ended at **0 bytes used** — the adapter sees nothing and
contributes only LD_PRELOAD intercept overhead.

### 2. The forced-marker path is slower than NFS, not faster

The [caseread pre-staging follow-up](miv_iowarp_ares_caseread.md#follow-up-explicit-pre-staging-into-a-cte-managed-path)
exercised the only path that does drive bytes through the tier: rename
target files to `clio::MiV_Cells_…h5` and let the adapter route both write
and read. The write side worked (24.7 s to stage 47 GB into the NVMe tier).
The read-back came in at **~36 MB/s logical** vs NFS direct at ~142 MB/s
(≈4× slower), and emitted `Tag GetBlob failed for page N` errors at the
last pages of every file. Filed upstream as
[iowarp/clio-core#470](https://github.com/iowarp/clio-core/issues/470).

So even with the marker, you'd be measuring "CTE = 4× slower with read
errors" rather than a win.

### 3. Case 7 is compute-bound on every data size that actually runs

| circuit | per-eval NEURON | per-eval I/O | I/O fraction |
|---|---|---|---|
| Microcircuit_Small (case7) | ~140 s | ~45 MB read | < 0.4% |
| Microcircuit_Full (case7_full) | — | 30 GB once at init | OOMs before any eval |

Even a hypothetical 10× CTE speed-up on reads only moves wall time by a
fraction of a percent on the small circuit, which is dominated by NEURON
`fadvance` time, not file I/O. On the full circuit the question is moot
because the node OOMs at the PYR forest read (~64 GB needed vs 46 GB
ceiling) before any eval starts. The `cell_selection` bypass (~50 cells)
is blocked by two independent MiV-Simulator code-path bugs:

- PRN/Bilash templates: `network.py:1102` passes `gid=…` to
  `PRN.__init__`, which only accepts `params=None`.
- HOC templates: `synapses.py:3576` (`init_syn_mech_attrs`) accesses
  `cell.mech_dict` on raw `hoc.HocObject` instances that have no such attr.

## Workloads that could in principle benefit, and why they don't here

| Hypothetical regime | Could CTE help? | Why not on this build |
|---|---|---|
| Working set > 46 GB cache, many repeated reads | Yes | (a) MiV reads each HDF5 once and holds it in Python objects (`cache_queries: True`); (b) tier-read path is broken ([#470](https://github.com/iowarp/clio-core/issues/470)) |
| Cold-cache first-read across allocations served from persistent NVMe tier | Yes | Tier isn't persisted between jobs; even if it were, tier reads are 4× slower than NFS |
| Many small metadata-heavy opens | Yes | optimize-network opens ~5 files, not thousands |
| Write-bound checkpoints (caseio) | Yes | Already measured: ~11% slower because simulation is still compute-bound on small circuit; full data OOMs before checkpoint |
| Multi-node where `/dev/shm` is the bottleneck | Yes | Multi-node MiV deadlocks on ares chimaera build (case 6 finding) |
| Async prefetch overlapping init I/O with NEURON compute | Yes | clio-core has no async-prefetch API exposed; MiV blocks on init reads |

## The one experiment that would add a datapoint but not flip the verdict

You can route MiV through the adapter **without modifying MiV** by adding
the marker via a symlink directory:

```
ln -s MiV_CA1_full /home/hyoklee/dataset-staging/clio::MiV_CA1_full
# then optimize_network_ares_full.yaml:
#   dataset_prefix: /home/hyoklee/dataset-staging/clio::MiV_CA1_full
```

`kClioPrefix` is matched as a substring, so this routes every MiV HDF5
open through the adapter. Outcome is predictable from the
[caseread §pre-staging](miv_iowarp_ares_caseread.md#follow-up-explicit-pre-staging-into-a-cte-managed-path)
result: the tier read path is 4× slower than NFS and drops `GetBlob`
errors at file tails, so case 7 would either crash on `H5Dread` past a
broken page or run measurably slower than baseline. Adds a datapoint;
doesn't flip the verdict.

## What would actually unlock case-7 ≥ baseline

All three are required:

1. **clio-core [#470](https://github.com/iowarp/clio-core/issues/470)
   fix** so staged-tier reads match or beat NFS bandwidth and don't error
   at file tails.
2. **MiV-Simulator change**: either emit `clio::`-prefixed paths when the
   adapter is loaded, OR link against a CTE-aware HDF5 VFD so the marker
   isn't needed at the path level. (The symlink hack above is a
   workaround for this single requirement only.)
3. **A workload where I/O actually matters** — repeated-read amortization
   (turn off MiV's in-process caching, or run many separate allocations
   against a persistent tier), or async prefetch overlapping init reads
   with NEURON compute. On the small circuit, init reads are ~0.4% of
   wall time; on full CA1 the node OOMs before reads finish.

## Bottom line

Until (1) and (2) land upstream, case 7 with CTE on ares is **bounded
above** by "≈ NFS within 0.05%" (no marker → adapter inert) and **below**
by "≈ 4× slower with `GetBlob` errors" (marker forced → tier read path
broken). There's no configuration knob between those bounds.

The answer to "can IOWarp/clio-core help case 7 performance on ares?" is
currently **no measurable benefit possible**, with the path to changing
that answer documented above.

## Related pages

- [Case 7 small-circuit](miv_iowarp_ares_case7.md) — IOWarp 11% slower
- [Case 7 full CA1](miv_iowarp_ares_case7_full.md) — 11 jobs, OOM/crash matrix
- [Read-bound caseread](miv_iowarp_ares_caseread.md) — adapter-inert and forced-marker results
- [Write-bound caseio](miv_iowarp_ares_caseio.md) — checkpoint-heavy still ~11% slower
- [Case 6 build/tier comparison](miv_iowarp_ares_case6.md) — fork vs upstream, RAM vs NVMe tier
