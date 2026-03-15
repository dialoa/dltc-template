# Architecture Overview

The Dialectica template (dltc-template) is a Pandoc-based document processing system that compiles academic philosophy articles from Markdown into PDF and HTML. It is used to produce the journal [Dialectica](https://dialectica.philosophie.ch/).

## What it produces

From a single set of Markdown source files, the template can produce:

- **Offprint PDFs** — individual article PDFs with title page, headers, and bibliography
- **Offprint HTML** — standalone HTML articles for web publication
- **Issue PDFs** — complete journal issues with cover, table of contents, and all articles
- **Book PDFs** — collected volumes with book-style title pages
- **Reference lists** — plain-text bibliographies for OJS (Open Journal Systems)

## How it works

```
                   master.md
                      |
                      v
  dltc-make  -->  make.lua  -->  pandoc command
                                     |
                     +---------------+---------------+
                     |               |               |
               defaults files   filter pipeline   templates
              (d_common.yaml)  (20 Lua filters)  (dialectica.latex)
              (d_offprint.yaml) (collection.lua)  (dialectica.html)
              (settings/*.yaml)                    (paper-in-issue.*)
                     |               |               |
                     +-------+-------+-------+-------+
                             |
                             v
                     output (PDF / HTML)
```

1. **`dltc-make`** is a shell script baked into the Docker compilation environment. It calls `pandoc lua make.lua` with user arguments.
2. **`make.lua`** parses arguments (mode, format, flags), locates the template directory, reads `master.md`, and assembles a full `pandoc` command.
3. **Pandoc** runs with the assembled defaults files, metadata files, filter pipeline, and template to produce the final output.

## Directory layout

```
dltc-template/
|-- dialoa.yaml              # Template marker file (used by findDialoa)
|-- d_common.yaml            # Pandoc defaults: all modes
|-- d_offprint.yaml          # Pandoc defaults: offprint mode
|-- d_issue.yaml             # Pandoc defaults: issue mode
|-- d_book.yaml              # Pandoc defaults: book mode
|-- paper-in-issue-{nix,mac,win}.yaml   # Platform-specific entry points
|-- paper-in-book-{nix,mac,win}.yaml    # Platform-specific entry points
|-- Makefile                 # Filter update tool (pulls from GitHub repos)
|
|-- scripts/
|   |-- make.lua             # Main build orchestrator
|   |-- make.sh              # POSIX shell wrapper for make.lua
|   |-- refs.lua             # Reference list generator for OJS
|   |-- cleanmakefiles.lua   # Utility to clean generated Makefiles
|   `-- cleanRprojectfiles.sh
|
|-- filters/                 # Pandoc Lua filters (+ pandoc-crossref binary)
|   |-- collection.lua       # Always loaded first (-L flag): handles imports
|   |-- dialectica-pre-format.lua
|   |-- sections-to-meta.lua
|   |-- ... (20+ filters)
|   `-- image-embedder.lua
|
|-- templates/               # Pandoc output templates
|   |-- dialectica.latex     # Main LaTeX template (via pandokoma)
|   |-- dialectica.html      # Main HTML template
|   |-- dialectica.css       # HTML stylesheet
|   |-- dialectica.less      # CSS source (Less preprocessor)
|   |-- paper-in-issue.latex # Chapter template for papers in issues
|   |-- paper-in-issue.html  # HTML template for papers in issues
|   |-- paper-in-book.latex  # Chapter template for papers in books
|   `-- dublincore.html      # Dublin Core metadata partial
|
|-- settings/                # YAML metadata files
|   |-- common.yaml          # Universal settings
|   |-- common-html.yaml     # HTML-specific metadata
|   |-- common-latex.yaml    # LaTeX-specific metadata (fonts, colors, geometry)
|   |-- offprint-latex.yaml  # Offprint title page and layout
|   |-- issue-latex.yaml     # Issue cover page and editorial board
|   |-- book-latex.yaml      # Book title pages and contributors
|   |-- collection.yaml      # Template for collection metadata (used by make.lua)
|   `-- paper-in-issue.yaml  # Per-paper settings (crossref options, filter config)
|
|-- csl/                     # Citation Style Language files
|   |-- dialectica.csl       # Main citation style (author-date, Chicago variant)
|   |-- dialectica-full-citation.csl
|   `-- minimal.csl
|
`-- media/                   # Logo TeX files
    |-- cc-by-logo.tex
    |-- esap-logo.tex
    |-- openaccess-logo.tex
    `-- ORCID-iD-icon.svg
```

## How `findDialoa()` locates the template

When `make.lua` runs, it needs to find the template root directory. The function `findDialoa()` searches for `template/{version}/dialoa.yaml` starting from the current working directory and walking up to 3 parent directories:

```
Working directory: /dltc-workhouse/2024/75-02/03/
Search path:
  ./template/1.2/dialoa.yaml            (not found)
  ../template/1.2/dialoa.yaml           (not found)
  ../../template/1.2/dialoa.yaml        (not found)
  ../../../template/1.2/dialoa.yaml     (FOUND -> template root)
```

This means article directories can be nested up to 3 levels deep inside a workhouse that has `template/1.2/` at its root.

The version string (`1.2`) is hardcoded in `make.lua` as `VERSION = '1.2'`.

## Compilation modes

| Mode | Command | Output | Use case |
|------|---------|--------|----------|
| `offprint` / `off` | `dltc-make off` | Single article HTML/PDF | Web publication, author proofs |
| `issue` / `vol` | `dltc-make vol` | Full issue PDF | Print publication |
| `book` | `dltc-make book` | Book PDF | Collected volumes |
| `bare` | `dltc-make bare` | Book without imports | Testing |
| `refs` | `dltc-make refs` | Plain-text bibliographies | OJS upload |
| `all` | `dltc-make all` | Issue + all offprints | Full production run |

See [CLI Reference](cli-reference.md) for complete argument documentation.

## Dependencies

The template requires:

- **Pandoc** 3.5+ (provides the document processing engine)
- **LuaLaTeX** via TeXLive (full scheme) for PDF output
- **pandoc-crossref** binary for cross-reference processing
- **Inkscape** and **librsvg2** for image format conversion
- **Fonts**: STIX Two Text, Libertinus family, VenusSBOP-BoldExtended

All dependencies are bundled in the `dltc-env` Docker image.

## Related repositories

| Repository | Purpose |
|------------|---------|
| [`dialoa/dltc-template`](https://github.com/dialoa/dltc-template) | This repo: templates, filters, settings |
| [`dialoa/dialectica-filters`](https://github.com/dialoa/dialectica-filters) | Upstream source for Lua filters (submodules) |
| [`dialoa/pandokoma`](https://github.com/dialoa/pandokoma) | Pandoc + KOMA-Script LaTeX template (upstream for `dialectica.latex`) |
| [`dialoa/dltc-csl`](https://github.com/dialoa/dltc-csl) | Citation style development |
| Individual filter repos | `dialoa/statement`, `dialoa/columns`, `dialoa/first-line-indent`, `dialoa/labelled-lists`, `dialoa/recursive-citeproc`, `dialoa/imagify`, `dialoa/collection` |
