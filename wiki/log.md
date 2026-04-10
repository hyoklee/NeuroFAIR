# Wiki Log

## [2026-04-10] ingest | MiV Microcircuit HDF5 files (9 files)

Converted 9 HDF5 files from `/lus/flare/projects/gpu_hack/iowarp/neuroh5/` using `h5md`. Created source pages: `source-MiV_h5types_h5.md`, `source-Microcircuit_coords_h5.md`, `source-MiV_input_features_h5.md`, `source-MiV_input_spikes_h5.md`, `source-OLM_forest_h5.md`, `source-OLM_tree_h5.md`, `source-PVBC_tree_h5.md`, `source-PYR_forest_h5.md`, `source-PYR_tree_h5.md`. Created concept pages: `concept-miv-microcircuit.md`, `concept-morphology-swc.md`, `concept-cell-coordinates.md`. Updated `index.md` and `overview.md`. 12 files failed to convert due to filesystem truncation (MiV_Cells/Connections 2022*, OLM/PVBC/PYR connections+forest_syns+compressed variants, Microcircuit_Small files).

## [2026-04-08] ingest | example.h5

Ingested `/home/hyoklee/neuroh5/data/example.h5`. Single GC→GC projection, 14 edges, legacy `Connectivity/` layout. Created `source-example_h5.md`. Wrote SMD to all HDF5 objects. Updated index.

## [2026-04-08] ingest | dentate_test.h5

Ingested `/home/hyoklee/neuroh5/data/dentate_test.h5`. Dentate gyrus circuit: 11 populations, 40 valid projections across 6 cell types (GC, MC, AAC, BC, HC, NGFC). Created `source-dentate_test_h5.md` and `concept-dentate-gyrus.md`. Wrote SMD to key HDF5 objects. Updated index.

## [2026-04-08] init | wiki created

Initial wiki created using llm-wiki pattern. Schema defined in `CLAUDE.md`. Concept pages written for DBS format, populations, projections, H5Types, and dentate gyrus cell types.
