# Configuration Reference

This document covers every YAML configuration file in the template, what it controls, and the key template variables.

## YAML file hierarchy

The template uses Pandoc's defaults files (`-d`) and metadata files (`--metadata-file`) in layers. Later values override earlier ones, and article frontmatter overrides everything.

```
                    d_common.yaml          (base: standalone, pdf-engine, template)
                         |
                    d_<mode>.yaml          (mode filters: offprint, issue, or book)
                         |
               _collection.yaml            (runtime: template paths, import config)
                         |
          settings/common.yaml             (universal: indent, lang, secnumdepth)
                         |
     settings/common-<format>.yaml         (format: fonts/geometry OR MathJax)
                         |
    settings/<mode>-<format>.yaml          (mode+format: title pages, covers)
                         |
  paper-in-<type>-<platform>.yaml          (per-paper: filter pipeline, CSL, crossref)
                         |
       settings/paper-in-issue.yaml        (per-paper metadata: crossref opts, filter config)
                         |
         article frontmatter               (author's YAML: title, author, abstract, etc.)
```

## Defaults files (loaded with `-d`)

### dialoa.yaml
Template marker file. Contains only the version number:
```yaml
version: 1.2
```
Used by `findDialoa()` to locate the template directory.

### d_common.yaml
Applied to all compilation modes.
```yaml
standalone: true
pdf-engine: lualatex
template: ${.}/templates/dialectica
number-sections: true
```
- `standalone: true` — produce a complete document (not a fragment)
- `pdf-engine: lualatex` — use LuaLaTeX for PDF output
- `template` — the main Pandoc template (resolves to `templates/dialectica.latex` or `.html`)
- `number-sections: true` — enable section numbering

### d_offprint.yaml
Adds filters for offprint mode (individual articles):
```yaml
filters:
- ${.}/filters/scholarly-metadata.lua
- ${.}/filters/dialectica-meta-format.lua
```
These run after the per-paper pipeline. `scholarly-metadata` is only needed for offprints because in issue/book mode the metadata is gathered differently.

### d_issue.yaml
Adds filters for issue mode (complete journal issues):
```yaml
filters:
- ${.}/filters/dialectica-meta-format.lua
- ${.}/filters/statement-isolate.lua
```
`statement-isolate` deduplicates theorem definitions when multiple articles define the same environment.

### d_book.yaml
Same filters as issue, plus a default first page:
```yaml
filters:
- ${.}/filters/dialectica-meta-format.lua
- ${.}/filters/statement-isolate.lua
metadata:
  first-page: 1
```

## Platform entry points

### paper-in-issue-{nix,mac,win}.yaml
The full per-paper configuration. Defines:

1. **Base settings**: standalone, lualatex, `paper-in-issue` template, number-sections
2. **Filter pipeline**: 20 filters in order (see [filters.md](filters.md))
3. **Math**: MathJax 3 for HTML (`tex-chtml-full.js`)
4. **CSL**: `${.}/csl/dialectica.csl`
5. **Chained metadata**: `${.}/settings/paper-in-issue.yaml`

The only difference between platforms is the pandoc-crossref binary path:
- `nix`: `${.}/filters/pandoc-crossref-nix`
- `mac`: `${.}/filters/pandoc-crossref`
- `win`: `${.}/filters/pandoc-crossref.exe`

### paper-in-book-{nix,mac,win}.yaml
Identical to paper-in-issue but uses the `paper-in-book` template.

## Settings metadata files

### settings/common.yaml
Universal settings for all formats:
```yaml
toc: false
secnumdepth: 3
indent: true
lang: en
frenchspacing: true
```
- `toc: false` — no standard table of contents (custom TOC in issue template)
- `secnumdepth: 3` — number headings up to 3 levels deep
- `indent: true` — enable first-line indentation
- `frenchspacing: true` — single space after periods

### settings/common-html.yaml
HTML-specific metadata:
```yaml
html-math-method:
  method: mathjax
  url: "https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-chtml-full.js"
link-citations: true
imagify:
  zoom: 1.5
```

### settings/common-latex.yaml
The largest settings file. Controls LaTeX typography, fonts, colors, and layout.

**Document class and geometry**:
```yaml
documentclass: scrbook
classoption:
  - paper=140mm:210mm    # 2:3 ratio page
  - BCOR=0mm             # no binding correction
  - DIV=15               # KOMA margin calculation
  - fontsize=10pt
  - headings=small
  - numbers=noenddot
```

**Fonts** (requires STIX Two, Libertinus, VenusSBOP):
```yaml
mainfont: STIX Two Text
mainfontoptions:
  - Extension=.otf
  - UprightFont=*-Regular
  - BoldFont=*-Bold
  - ItalicFont=*-Italic
  - BoldItalicFont=*-BoldItalic
mathfont: STIX Two Math
sansfont: Libertinus Sans
monofont: Libertinus Mono
```

**KOMA-Script font definitions** (`setkomafonts`):
- `disposition`: Libertinus Sans, bold (section headings)
- `titlehead`: VenusSBOP-BoldExtended, 30pt (journal name on title pages)
- `pagehead`: STIX Two Text, 8pt (running headers)
- `pagefoot`: STIX Two Text, 8pt (running footers)
- `chapter`: STIX Two Text, 16pt (chapter/article titles)
- `section`: STIX Two Text, 12pt
- `subsection`: STIX Two Text, 10.5pt, italic
- `paragraph`: STIX Two Text, 10pt, italic
- `descriptionlabel`: STIX Two Text, bold

**Colors**:
```yaml
definecolors:
  - name: dialecticablue
    model: RGB
    value: 0,31,79        # #001F4F
  - name: dialecticalink
    model: RGB
    value: 8,65,152       # #084198
```

**Section formatting** (via `redeclaresectioncommand`):
- Chapter: no prefix number, before/after skip
- Section/subsection: customized spacing

**Header/footer**:
```yaml
automark: chapter
ohead: "\\headmark"
ofoot: "\\pagemark"
```

**Unicode character mappings**: Extensive list of Greek and special characters mapped to LaTeX commands (α → `\textalpha`, β → `\textbeta`, etc.).

**Header includes**: LaTeX packages and configuration:
- `enumitem` for tight lists
- `paperabstract` environment
- `reviewof` environment
- `multicol`, `longtable`, `booktabs` for tables
- `graphicx` with bounded scaling (`\setkeys{Gin}{width=\linewidth, height=0.95\textheight, keepaspectratio}`)
- `soul`/`lua-ul` for text underlining
- CSL bibliography commands

### settings/offprint-latex.yaml
Title page configuration for offprints:
```yaml
titlepage: true
```
Contains `rawtitlecode` — raw LaTeX for the title page including:
- Dialectica journal name (VenusSBOP font)
- Volume/issue/date/DOI in header
- Creative Commons CC-BY licensing notice
- ESAP and Open Access logos

### settings/issue-latex.yaml
Title page configuration for issues:
```yaml
titlepage: firstiscover
```
Contains `rawtitlecode` for:
- Cover page with blue background and journal name
- Inside front cover with editorial board, consulting board, journal editors
- Table of contents (3 sections: front, main, back)
- Back cover with ISSN, sponsors, indexing services

### settings/book-latex.yaml
Title page configuration for books:
Contains `rawtitlecode` for:
- Half-title page
- Series title page
- Main title page with blue background
- Contributors page
- Copyright and licensing page

### settings/collection.yaml
Template for the temporary `_collection.yaml` file. Contains placeholders that `make.lua` fills in at runtime:
- `PUBLICATIONTYPE` → `issue` or `book`
- `TEMPLATEFOLDER` → full path to template directory
- `DEFAULTSFILE` → full path to platform-specific defaults file

### settings/paper-in-issue.yaml
Per-paper metadata for filter configuration:

**Pandoc variables**:
```yaml
link-citations: true
lang: en-GB
```

**pandoc-crossref options**:
```yaml
autoSectionLabels: true
cref: false
figPrefix: ["figure", "figures"]
eqnPrefix: ["equation", "equations"]
tblPrefix: ["table", "tables"]
secPrefix: ["section", "sections"]
lstPrefix: ["listing", "listings"]
```

**Statement filter options**:
```yaml
statement:
  define-in-header: false
```

**First-line-indent filter**:
```yaml
first-line-indent:
  remove-after-class:
    - statement
```

**Imagify filter**:
```yaml
imagify:
  zoom: 1.6
```

## Key template variables

These variables are used in the LaTeX and HTML templates. They come from article frontmatter, master.md metadata, or filter-generated metadata.

### Article-level (from article frontmatter)

| Variable | Type | Used in | Description |
|----------|------|---------|-------------|
| `title` | string | paper-in-issue.* | Article title |
| `title-latex` | string | paper-in-issue.latex | LaTeX-safe title (optional) |
| `subtitle` | string | paper-in-issue.* | Article subtitle |
| `author` | list of objects | paper-in-issue.* | Author info (see below) |
| `abstract` | blocks | paper-in-issue.* | Article abstract |
| `abstract-title` | string | paper-in-issue.html | Custom abstract heading |
| `thanks` | blocks | paper-in-issue.latex | Acknowledgments (footnote) |
| `reviewof` | blocks | paper-in-issue.* | Book review metadata |
| `doi` | string | paper-in-issue.* | Article DOI |
| `shorttitle` | string | paper-in-issue.latex | Running header title |
| `bibliography` | string | — | Path to .bib file |
| `nocite` | list | — | Additional bibkeys to include |

### Author object fields

```yaml
author:
  - name: "Jane Smith"
    ORCID: "0000-0001-2345-6789"
    institutename: "University of Example"
    correspondence: true
    email: "jane@example.edu"
```

Template accessors: `$author.name$`, `$author.ORCID$`, `$author.institutename$`, `$author.correspondence$`, `$author.email$`.

For multiple authors: `$author/allbutlast$` and `$author/last$` for "A, B, and C" formatting.

### Issue-level (from master.md)

| Variable | Type | Used in | Description |
|----------|------|---------|-------------|
| `volume` | number | issue/offprint templates | Journal volume |
| `issue` | number | issue/offprint templates | Issue number |
| `date` | string | issue/offprint templates | Publication date |
| `doi` | string | issue/offprint templates | Issue DOI |
| `first-page` | number | issue templates | Starting page number |
| `imports` | list | collection.lua | Article source files |
| `editorialboard` | list | issue-latex.yaml | Editorial board members |
| `journaleditors` | list | issue-latex.yaml | Journal editors |
| `consultingboard` | list | issue-latex.yaml | Consulting board members |
| `issn` | string | issue-latex.yaml | ISSN |
| `sponsors` | list | issue-latex.yaml | Sponsors |
| `indexingservices` | list | issue-latex.yaml | Indexing services |

### Filter-generated variables

| Variable | Set by | Used in | Description |
|----------|--------|---------|-------------|
| `referencesblock` | bib-place.lua | templates | Bibliography HTML/LaTeX blocks |
| `html-printauthor` | dialectica-post-format.lua | paper-in-issue.html | Formatted author HTML |
| `printcovertitle` | dialectica-meta-format.lua | issue-latex.yaml | Cover title LaTeX |
| `printvolume` | dialectica-meta-format.lua | templates | Formatted volume string |
| `printissue` | dialectica-meta-format.lua | templates | Formatted issue string |
| `printdate` | dialectica-meta-format.lua | templates | Formatted date string |
| `proofmode` | CLI flag | templates | Proof mode marker |

## How to customize common things

### Change fonts
Edit `settings/common-latex.yaml`:
- `mainfont`, `sansfont`, `monofont`, `mathfont` for the main font families
- `setkomafonts` entries for heading/caption fonts
- Font files must be installed in the system (or the Docker image)

### Change colors
Edit `settings/common-latex.yaml` → `definecolors` section. The two main colors are:
- `dialecticablue` (RGB 0,31,79) — used for covers
- `dialecticalink` (RGB 8,65,152) — used for hyperlinks

### Change page size/margins
Edit `settings/common-latex.yaml` → `classoption`:
- `paper=140mm:210mm` — page dimensions
- `DIV=15` — KOMA margin calculation (higher = smaller margins)
- `BCOR=0mm` — binding correction

### Change title pages
Edit the `rawtitlecode` field in:
- `settings/offprint-latex.yaml` — offprint title page
- `settings/issue-latex.yaml` — issue cover + inside pages
- `settings/book-latex.yaml` — book title pages

### Change the citation style
Replace `csl/dialectica.csl` with a different CSL file, or change the `csl:` reference in `paper-in-issue-*.yaml`.

### Add or remove a filter
Edit the `filters:` list in `paper-in-issue-{nix,mac,win}.yaml`. Be careful about ordering — see [filters.md](filters.md) for dependencies.

### Change cross-reference labels
Edit `settings/paper-in-issue.yaml` → pandoc-crossref options (`figPrefix`, `eqnPrefix`, `tblPrefix`, `secPrefix`).
