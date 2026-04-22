# Wiki Log

## [2026-04-22] build-test | MiV-Simulator on Aurora — 208 tests run, neuroh5 cmake OK

Built MiV-Simulator v0.3.0 on Aurora compute node (PBS job 8445030, debug queue, `gpu_hack`). neuroh5 C++ extension compiled from source (`~/neuroh5`) against Aurora parallel HDF5 1.14.6 and MPICH 5.0 via manual cmake — resolving previous `H5Pset_dxpl_mpio` failures with serial or HDF5 2.x builds. Test results: pure-Python 38 passed / 18 failed (`test_eval_network` AttributeErrors, likely machinable API drift); NEURON tests 170 passed / 2 failed (Gfluct3 mechanism — `nrnivmodl` broken in NEURON 9.x pip wheel); CoreNEURON GPU confirmed (`GPU mode set OK`). mpi4py ABI mismatch warnings at runtime (struct sizes differ between build headers and runtime MPICH). CLI option names corrected: `--input-path` / `--populations`. Created `source-MiV_Simulator_build_test.md`.

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
