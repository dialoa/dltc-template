# dltc-make CLI Reference

`dltc-make` is the command-line tool for compiling Dialectica articles and issues. It is installed inside the `dltc-env` Docker container.

## Usage

```
dltc-make [MODE][FORMAT] [FLAGS] [KEY=VALUE...]
```

All arguments are optional. Default: compile article 1 as HTML offprint.

## Modes

| Mode | Aliases | Description | Default format |
|------|---------|-------------|----------------|
| `off` | `offprint`, `offprints` | Compile individual article(s) | html |
| `vol` | `iss`, `issue` | Compile complete journal issue | pdf |
| `book` | — | Compile as book | pdf |
| `bare` | — | Book without imports (testing) | pdf |
| `refs` | — | Generate plain-text bibliographies | (text) |
| `all` | — | Issue + all offprints | html + pdf |

## Formats

Append format to mode name (no space):

| Format | Extension | Engine |
|--------|-----------|--------|
| `html` | `.html` | Pandoc HTML |
| `pdf` | `.pdf` | LuaLaTeX |
| `latex` or `tex` | `.tex` | LaTeX source |
| `epub` | `.epub` | Pandoc EPUB |
| `jats` | `.jats` | JATS XML |
| `all` | (multiple) | All applicable |

## Article selection

For offprint mode, specify which article to compile by appending the article number:

- `off` — article 1 (default)
- `off2` — article 2
- `off3` — article 3

## Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--proof` | `-p` | Add `.proof` to output filename |
| `--verbose` | `-v` | Verbose output |
| `--quiet` | `-q` | Suppress output |

Short flags can be combined: `-pv` = proof + verbose.

## Key-value arguments

| Key | Default | Description |
|-----|---------|-------------|
| `master=<file>` | `master.md` | Path to master file |
| `pandoc=<cmd>` | `pandoc` | Pandoc command to use |
| `mode=<modes>` | `offprint` | Comma-separated modes |
| `format=<fmts>` | (per mode) | Comma-separated formats |

## Examples

### Single article

```bash
# Compile article 1 as HTML (default)
dltc-make

# Compile article 1 as PDF
dltc-make offpdf

# Compile article 3 as HTML
dltc-make off3html
# or equivalently:
dltc-make off3

# Compile article 2 as proof PDF
dltc-make off2pdf --proof
# or:
dltc-make off2pdf -p

# Compile all articles as both HTML and PDF
dltc-make offall
```

### Full issue

```bash
# Compile issue as PDF
dltc-make volpdf
# or:
dltc-make vol

# Compile issue as LaTeX source (for debugging)
dltc-make vollatex
```

### Book

```bash
# Compile book as PDF
dltc-make bookpdf

# Compile book as proof PDF
dltc-make bookpdf --proof
```

### Everything

```bash
# Compile issue PDF + all offprint HTMLs
dltc-make all

# Compile issue PDF + all offprint PDFs + all offprint HTMLs
dltc-make allall
```

### Reference lists

```bash
# Generate .bib.txt files for each article (for OJS)
dltc-make refs
```

### Advanced

```bash
# Use a different master file
dltc-make offpdf master=my-master.md

# Use a specific pandoc binary
dltc-make offpdf pandoc=/usr/local/bin/pandoc-3.6

# Compile article 4 as PDF, quiet
dltc-make off4pdf -q
```

## Output files

| Mode | Filename pattern | Example |
|------|-----------------|---------|
| Offprint | `<article-basename>.<ext>` | `smith-2024.html` |
| Offprint (proof) | `<article-basename>.proof.<ext>` | `smith-2024.proof.pdf` |
| Issue | `<collection-name>.<ext>` | `v75.i2.pdf` |
| Book | `<collection-name>-book.<ext>` | `v75.i2-book.pdf` |
| Refs | `<article-basename>.bib.txt` | `smith-2024.bib.txt` |

The collection name is derived from the `doi` field in `master.md` (e.g., `10.48106/dial.v75.i2` becomes `v75.i2`). If no DOI, it uses the date field.

## Running outside Docker

If you have Pandoc and all dependencies installed locally, use `make.sh` instead of `dltc-make`:

```bash
cd /path/to/issue/directory/
/path/to/template/1.2/scripts/make.sh offpdf
```

`make.sh` locates `make.lua` by searching for it relative to the current directory (up to 3 levels).

## Working directory

`dltc-make` must be run from the issue directory — the directory containing `master.md`. The template is located automatically by searching parent directories for `template/1.2/dialoa.yaml`.

```
dltc-workhouse/
|-- template/
|   `-- 1.2/             <-- template found here
|-- 2024/
|   `-- 75-02/
|       |-- master.md    <-- run dltc-make HERE
|       |-- article-1/
|       |   `-- source.md
|       `-- article-2/
|           `-- source.md
```

## Temporary files

`dltc-make` creates a temporary `_collection.yaml` file in the working directory during compilation. This file is regenerated on each run and can be safely deleted.
