# neuroh5 HDF5 Wiki — Schema

This wiki documents HDF5 samples from the neuroh5 library. The LLM maintains all files under `wiki/`; raw sources live under `../neuroh5/data/` and are never modified. Semantic metadata (SMD) is written directly to HDF5 files via the `write_smd_batch` / `write_semantic_metadata` MCP tools.

## Directory layout

```
wiki/
  CLAUDE.md              ← this file (schema)
  index.md               ← catalog of all wiki pages
  log.md                 ← append-only event log
  overview.md            ← high-level synthesis
  source-*.md            ← one page per HDF5 source file
  concept-*.md           ← reusable concept / entity pages
```

## Page conventions

- All pages use GitHub-flavored markdown.
- Cross-references use `[[Page Title]]` wikilink style in prose and standard `[text](file.md)` for actual links.
- Every source page lists: file path, size, top-level structure, key datasets, edge attributes.
- Every concept page lists: definition, where it appears (which files / paths), related concepts.

## Workflows

### Ingest a new HDF5 file
1. Run `collect_objects_for_smd` on the file.
2. Write `source-<name>.md` summarising structure and key datasets.
3. Update `index.md` with the new entry.
4. Update or create relevant `concept-*.md` pages.
5. Write SMD to the file via `write_smd_batch`.
6. Append an entry to `log.md`.

### Answer a query
1. Read `index.md` to find relevant pages.
2. Read those pages, synthesise an answer.
3. If the answer is valuable, file it as a new `concept-*.md` or append to an existing page.
4. Log the query in `log.md`.

### Lint
- Check for orphan pages (no inbound links).
- Check for stale SMD vs current file structure.
- Flag contradictions between concept pages.

## SMD format
SMD values are free-text English sentences written as the `semantic_metadata` HDF5 attribute. Keep them concise (1–3 sentences). Focus on *what the object represents* and *how it is used*, not just its dtype.
