# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this repo is

A read-only mirror of Onshape's **FeatureScript Standard Library** (`onshape/std`), exported tab-by-tab from
the public Onshape std document. Each Feature Studio tab is one `.fs` file at the repo root. The version the
files came from is recorded in `_export_manifest.json` (`versionName`, e.g. `3083.0`). The published
documentation for this repo is the DeepWiki at https://deepwiki.com/yepher/feature_script_std.

Nothing here is built, compiled, or tested locally. FeatureScript only runs inside Onshape. Most tasks are
reading and explaining the library, answering "how do I…" questions for custom features, or keeping the mirror
and its docs up to date.

## Layout

| Path | What it is |
|---|---|
| `*.fs` | Hand-written std modules (~170), all flat at the root |
| `*.gen.fs` | Generated enums (~100), marked `/* Automatically generated file -- DO NOT EDIT */` |
| `_export_manifest.json` | Export metadata: document id, version, per-file element ids, `failures` |
| `LICENSE`, `README.pdf` | Uploaded-file tabs from the Onshape document (MIT license; std README) |
| `README.md` | Repo readme (links to the DeepWiki); not part of the export |
| `.devin/wiki.json` | DeepWiki page structure and notes (see below) |
| `tools/` | Export script and update instructions. **Gitignored**, so it's local only |

## Rules

- **Do not edit library files** (`*.fs`, `*.gen.fs`, `_export_manifest.json`, `LICENSE`, `README.pdf`). They
  are overwritten on every sync. Upstream changes belong in Onshape, not here.
- Library updates come only from re-exporting with `tools/LATEST_SCRIPT.js`, following `tools/README.md`.
  Commit a sync on its own, with a message like `Version: <versionName>`.
- The sync uses `rsync --delete`, which would delete repo-owned files. Exclude `.devin/`, `README.md`,
  `AGENTS.md` and `CLAUDE.md` as well as `.git`, `.gitignore` and `tools/`:
  ```sh
  rsync -av --delete \
    --exclude .git --exclude .gitignore --exclude tools/ --exclude .devin/ \
    --exclude README.md --exclude AGENTS.md --exclude CLAUDE.md \
    ~/Downloads/onshape-fs-<version>/ ./
  ```
- The files you can edit are `README.md`, `AGENTS.md`, `CLAUDE.md` and `.devin/wiki.json`.

## Navigating the library

- Every file starts with `FeatureScript <N>; /* Automatically generated version */`. Not every file is bumped
  each release, so older numbers in the headers are normal.
- Modules import each other as `import(path : "onshape/std/<file>.fs", version : "<ver>")`.
  `geometry.fs` re-exports the full feature library (what most Feature Studios import), and `common.fs`
  re-exports the core runtime without the full feature set.
- Find the source of an Onshape UI feature by grepping its annotation:
  `grep -n '"Feature Type Name" : "Extrude"' *.fs`.
- Features are declared as
  `export const name = defineFeature(function(context is Context, id is Id, definition is map) precondition { … } { … })`.
  The mechanics live in `feature.fs`, and `extrude.fs` is the best full example.
- Core modules to know:
  - `query.fs` and `evaluate.fs`: selecting and measuring geometry (`q*` functions, `evaluate*`)
  - `geomOperations.fs`: the `op*` operations that wrap the `@op*` built-ins
  - `error.fs`: `regenError`, `reportFeatureWarning`, `reportFeatureInfo`
  - `debug.fs`: `debug()` draws values in the graphics area
  - `units.fs`, `valueBounds.fs`: `ValueWithUnits` and the ranges for parameters
  - `attributes.fs`, `context.fs`: attributes and variables
  - `boolean.fs`: `processNewBodyIfNeeded`
- Subsystem prefixes: `sheetMetal*` (sheet metal), `frame*`/`gusset`/`cutlist*` (frames and weldments),
  `hole*` (holes and hole tables), `*Pattern`/`pattern*` (patterns), `sm*.gen.fs` (sheet metal enums).
- Doc comments use `@param`, `@field`, `@ex`, `@requiredif`, `@optional`. They're the best source for
  what a parameter means.

## Answering questions and writing examples

- Cite the file and function for any claim (e.g. `feature.fs` `callSubfeatureAndProcessStatus`).
- Check signatures against the source before using a function in an example. Many functions are overloaded
  (e.g. `regenError`, `debug`), so pick the overload that matches.
- Custom-feature examples should `import(path : "onshape/std/geometry.fs", version : "<versionName>")` and
  use the version from `_export_manifest.json`.

## DeepWiki (`.devin/wiki.json`)

- Schema: `repo_notes` (array of `{content}`) and `pages` (array of `{title, purpose, parent?, page_notes?}`).
  DeepWiki generates exactly the pages listed.
- Limits: 30 pages (Standard plan), 100 notes in total, 10,000 characters per note. Titles must be unique and
  non-empty, and each `parent` must match an existing title.
- After a sync, check that every `.fs` file named in the wiki still exists and that new modules are covered by
  some page. A quick check:
  ```sh
  python3 -c "import json,re,glob;t=json.dumps(json.load(open('.devin/wiki.json')));[print('missing',r) for r in sorted(set(re.findall(r'(?<![\w*.])\w+\.(?:gen\.)?fs\b',t))) if not glob.glob(r)]"
  ```
