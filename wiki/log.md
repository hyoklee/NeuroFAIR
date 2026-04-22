# Wiki Log

## [2026-04-22] build-test | MiV-Simulator on Aurora — all components verified (PBS jobs 8445030–8445212)

Built MiV-Simulator v0.3.0 on Aurora (debug queue, `gpu_hack`). Key findings: (1) neuroh5 requires Aurora parallel HDF5 1.14.6 (`hdf5-1.14.6-ehlefog`) — HDF5 2.x removed `H5Pset_dxpl_mpio`, serial builds lack MPI; (2) `nrnivmodl` NEURON 9.x pip wheel fix: export `NRNHOME=<miv_env>/lib/.../neuron/.data` before calling it; (3) OpenMPI (`libmpi.so.40`) in conda env shadows Aurora MPICH — fix by prepending `${AURORA_MPICH}/lib` to `LD_LIBRARY_PATH`; (4) `query-cell-attrs` needs `mpiexec -n 1`. Tests: pure-Python 38 pass/18 fail (test_eval_network machinable API drift — upstream bug); NEURON 170 pass/2 fail (test_gfluct3 needs `h.load_file('stdrun.hoc')` in NEURON 9.x — upstream bug); CoreNEURON GPU confirmed. CLI: `show-h5types` shows 4 populations (STIM/PYR/PVBC/OLM); `query-cell-attrs --populations OLM` returns arc distances + coordinates for GIDs 143–186. Updated `source-MiV_Simulator_build_test.md`.

## [2026-04-17] ingest | 5 files recovered via Globus Transfer API

Previously corrupted/truncated files were re-transferred from the MiV Globus guest collection (`0028aea1-ffc7-44b6-aec9-dd748ac839c4`) to Aurora Flare (`f39a7a0f-5bfc-46ce-9615-ba9f8592814f`) using CAE OMNI with Globus-to-Globus Transfer API. Created source pages: `source-MiV_Cells_Microcircuit_20220410_h5.md` (15.7 GB), `source-MiV_Cells_Microcircuit_20220412_h5.md` (18.4 GB), `source-MiV_Connections_Microcircuit_20220412_h5.md` (6.9 GB), `source-PYR_forest_syns_compressed_h5.md` (11.2 GB, 2.68B synapses), `source-dmosopt_Motoneuron_h5.md` (121 MB, DMOSOPT optimization). Added Motoneuron Model section to `index.md`.

## [2026-04-14] ingest | MiV Microcircuit HDF5 files — new additions (10 files)

Used `~/miniconda3/bin/h5md` to generate markdown from 10 newly added HDF5 files. Created source pages: `source-OLM_connections_h5.md`, `source-OLM_forest_syns_h5.md`, `source-PVBC_forest_h5.md`, `source-PVBC_forest_syns_h5.md`, `source-PVBC_connections_h5.md`, `source-PYR_forest_compressed_h5.md`, `source-PYR_connections_compressed_h5.md`, `source-MiV_Connections_Microcircuit_20220410_h5.md`, `source-MiV_Cells_Microcircuit_Small_20220410_h5.md`, `source-MiV_Connections_Microcircuit_Small_20220410_h5.md`. Updated `index.md`. Failed files (corrupted/truncated): `MiV_Cells_Microcircuit_20220410.h5`, `MiV_Cells_Microcircuit_20220412.h5`, `MiV_Connections_Microcircuit_20220412.h5`, `PYR_forest_syns_compressed.h5`, `Motoneuron_model/motoneuron_20230210/dmosopt_Motoneuron.h5`.

## [2026-04-10] ingest | MiV Microcircuit HDF5 files (9 files)

Converted 9 HDF5 files from `/lus/flare/projects/gpu_hack/iowarp/neuroh5/` using `h5md`. Created source pages: `source-MiV_h5types_h5.md`, `source-Microcircuit_coords_h5.md`, `source-MiV_input_features_h5.md`, `source-MiV_input_spikes_h5.md`, `source-OLM_forest_h5.md`, `source-OLM_tree_h5.md`, `source-PVBC_tree_h5.md`, `source-PYR_forest_h5.md`, `source-PYR_tree_h5.md`. Created concept pages: `concept-miv-microcircuit.md`, `concept-morphology-swc.md`, `concept-cell-coordinates.md`. Updated `index.md` and `overview.md`. 12 files failed to convert due to filesystem truncation (MiV_Cells/Connections 2022*, OLM/PVBC/PYR connections+forest_syns+compressed variants, Microcircuit_Small files).

## [2026-04-08] ingest | example.h5

Ingested `/home/hyoklee/neuroh5/data/example.h5`. Single GC→GC projection, 14 edges, legacy `Connectivity/` layout. Created `source-example_h5.md`. Wrote SMD to all HDF5 objects. Updated index.

## [2026-04-08] ingest | dentate_test.h5

Ingested `/home/hyoklee/neuroh5/data/dentate_test.h5`. Dentate gyrus circuit: 11 populations, 40 valid projections across 6 cell types (GC, MC, AAC, BC, HC, NGFC). Created `source-dentate_test_h5.md` and `concept-dentate-gyrus.md`. Wrote SMD to key HDF5 objects. Updated index.

## [2026-04-08] init | wiki created

Initial wiki created using llm-wiki pattern. Schema defined in `CLAUDE.md`. Concept pages written for DBS format, populations, projections, H5Types, and dentate gyrus cell types.
