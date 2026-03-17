# Compilation Flow

This document describes exactly what happens when you run `dltc-make` to compile an article or issue.

## Entry point: dltc-make

`dltc-make` is a shell script installed in the Docker image at `/usr/local/bin/dltc-make`:

```bash
#!/usr/bin/env bash
maker="/home/copyeditor/dltc-workhouse/template/1.2/scripts/make.lua"
if [ -f "${maker}" ]; then
    pandoc lua "${maker}" $@
else
    echo "Error: can't find the make.lua script."
    exit 1
fi
```

It runs `make.lua` as a Pandoc Lua script, passing all arguments through. There is also `scripts/make.sh`, a POSIX wrapper that locates `make.lua` by relative path (for use outside Docker).

## Step 1: Argument parsing

`make.lua` parses the command-line arguments into an `Options` object:

- **Mode**: offprint, issue, book, bare, refs (default: offprint)
- **Format**: html, pdf, latex, epub, jats (default: html for offprint, pdf for others)
- **Article number**: which article to compile for offprints (e.g., `off2` for article 2)
- **Flags**: `--proof`, `--quiet`, `--verbose`
- **Key-values**: `master=<file>`, `pandoc=<cmd>`, etc.

## Step 2: Template location

`findDialoa()` searches for the template directory by looking for `template/1.2/dialoa.yaml` relative to the current working directory, walking up to 3 parent directories. See [overview.md](overview.md) for details.

## Step 3: Read master.md

The `master.md` file is read and parsed. Its YAML frontmatter provides:

```yaml
---
volume: 75
issue: 2
date: December 2023
doi: 10.48106/dial.v75.i2
first-page: 1
imports:
  - article-1/source.md
  - article-2/source.md
  - article-3/source.md
---
```

Key fields:
- **`imports`**: list of article Markdown files to include
- **`volume`**, **`issue`**, **`date`**: journal metadata passed to templates
- **`doi`**: used to derive the collection name (output filename base)

## Step 4: Prepare metadata

### Collection name
Derived from `master.md` metadata:
1. If DOI exists: strip the registrant prefix (e.g., `10.48106/dial.v75.i2` becomes `v75.i2`)
2. Else if date exists: `issue-<date>` (e.g., `issue-December-2023`)
3. Fallback: `untitled`

### Article files
The `imports` list maps articles by index. `off2` selects the second import, `off3` the third, etc. Each article file becomes the basename for offprint output (e.g., `article-1/source.md` → `source.html`).

### Temporary collection metadata
`make.lua` reads `settings/collection.yaml` and fills in runtime values to create a temporary `_collection.yaml` file:

```yaml
publication-type: "issue"           # or "book"
templatefolder: "/path/to/template/1.2"
collection:
  defaults: "/path/to/paper-in-issue-nix.yaml"
  mode: raw
  isolate: true
  gather: [header-includes, references]
  pass: [volume, issue]
offprints:
  defaults: "/path/to/paper-in-issue-nix.yaml"
  gather: [title, author, references, ...]
  replace: [first-page, last-page, doi]
  pass: [volume, issue]
```

The `collection` block configures how `collection.lua` handles multi-article documents:
- **`mode: raw`** — include articles as-is
- **`isolate: true`** — prefix IDs to prevent conflicts between articles
- **`gather`** — metadata fields collected from articles into the parent document
- **`pass`** — metadata fields passed from parent to articles
- **`replace`** — metadata fields from articles that override parent values

## Step 5: Assemble pandoc command

`renderOne()` builds the full pandoc command. Here is the exact order:

```bash
pandoc \
  # 1. Collection filter (always first, loaded as library)
  -L template/1.2/filters/collection.lua \

  # 2. Common defaults
  -d template/1.2/d_common.yaml \

  # 3. Mode-specific defaults (adds extra filters)
  -d template/1.2/d_offprint.yaml \

  # 4. Temporary collection metadata
  --metadata-file _collection.yaml \

  # 5. Settings metadata files (in order)
  --metadata-file template/1.2/settings/common.yaml \
  --metadata-file template/1.2/settings/common-latex.yaml \
  --metadata-file template/1.2/settings/offprint-latex.yaml \

  # 6. Mode-specific metadata flags
  -M offprint-mode=1 \

  # 7. Proof flag (if --proof)
  -M proofmode=true \

  # 8. Output format and filename
  -t pdf -o source.pdf \

  # 9. Source file
  master.md
```

### Metadata file load order

`getMetadataFiles()` loads settings files in this pattern for each of `{common, <mode>}` crossed with `{'', '-<format>'}`:

1. `settings/common.yaml` (always)
2. `settings/common-<format>.yaml` (e.g., `common-latex.yaml` for PDF)
3. `settings/<mode>.yaml` (if exists)
4. `settings/<mode>-<format>.yaml` (e.g., `offprint-latex.yaml`)

Later files override earlier ones. The article's own YAML frontmatter overrides everything.

## Step 6: Pandoc processing

When pandoc runs with the assembled command:

### 6a. Defaults files load

`d_common.yaml` sets basic options:
```yaml
standalone: true
pdf-engine: lualatex
template: ${.}/templates/dialectica    # ${.} = template root dir
number-sections: true
```

`d_offprint.yaml` adds post-gather filters:
```yaml
filters:
- ${.}/filters/scholarly-metadata.lua
- ${.}/filters/dialectica-meta-format.lua
```

### 6b. How `${.}` works

In Pandoc defaults files, `${.}` resolves to the directory containing that defaults file. So if the defaults file is at `/path/to/template/1.2/d_common.yaml`, then `${.}/templates/dialectica` resolves to `/path/to/template/1.2/templates/dialectica`.

This makes the template relocatable — all internal paths are relative.

### 6c. Collection filter processes imports

`collection.lua` (loaded via `-L`) reads the `imports` list from `master.md` and processes each article:

1. Reads each imported Markdown file
2. If `isolate: true`, prefixes all identifiers (`fig:`, `sec:`, `tbl:`, `eq:`, etc.) with a unique hash to prevent conflicts between articles
3. For offprint mode: extracts only the specified chapter
4. For issue/book mode: merges all articles into one document
5. Gathers/passes/replaces metadata fields as configured

### 6d. Per-paper filter pipeline

Each imported paper is processed through the filter pipeline defined in the platform-specific defaults file (e.g., `paper-in-issue-nix.yaml`). This is a chain of 20 filters — see [filters.md](filters.md) for the complete reference.

### 6e. Template rendering

The processed document is rendered through the output template:

- **PDF**: `templates/dialectica.latex` (KOMA-Script via pandokoma) → LuaLaTeX → PDF
- **HTML**: `templates/dialectica.html` + `templates/dialectica.css`

For papers within issues, the chapter-level templates are used:
- `templates/paper-in-issue.latex` / `paper-in-issue.html`
- `templates/paper-in-book.latex`

## Platform differences

The only difference between platform-specific YAML files is the pandoc-crossref binary name:

| Platform | File | Binary |
|----------|------|--------|
| Linux | `paper-in-issue-nix.yaml` | `${.}/filters/pandoc-crossref-nix` |
| macOS | `paper-in-issue-mac.yaml` | `${.}/filters/pandoc-crossref` |
| Windows | `paper-in-issue-win.yaml` | `${.}/filters/pandoc-crossref.exe` |

Everything else is identical.

## Output files

### Offprint
- `<article-basename>.html` or `<article-basename>.pdf`
- With `--proof`: `<article-basename>.proof.html` or `.proof.pdf`

### Issue
- `<collection-name>.pdf` (e.g., `v75.i2.pdf`)

### Book
- `<collection-name>-book.pdf`

### Refs
- `<article-basename>.bib.txt` for each article (plain-text bibliography)

## Example: compiling a single offprint

```bash
cd /dltc-workhouse/2024/75-02/
dltc-make off1pdf --proof
```

1. Parse args → mode: offprint, format: pdf, article: 1, proof: true
2. Find template at `../../template/1.2/` (2 levels up)
3. Read `master.md`, get imports list
4. Article 1 = first import (e.g., `smith-2024/source.md`)
5. Create `_collection.yaml` with offprint settings
6. Run pandoc with full command (defaults + metadata + filters + template)
7. Output: `smith-2024.proof.pdf`

## Example: compiling a full issue

```bash
cd /dltc-workhouse/2024/75-02/
dltc-make volpdf
```

1. Parse args → mode: issue, format: pdf, all articles
2. Find template, read master.md
3. Create `_collection.yaml` with issue settings
4. Pandoc merges all imported articles via collection.lua
5. Each article's IDs are prefixed for isolation
6. Issue-level metadata (cover, TOC, editorial board) from `issue-latex.yaml`
7. Output: `v75.i2.pdf` (complete issue)
