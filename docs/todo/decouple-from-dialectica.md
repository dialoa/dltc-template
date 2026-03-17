# Plan: Decouple Template from Dialectica

## Prerequisites (from JD)

Two architectural changes should land first — they simplify this plan and address the main barriers to upgrading Pandoc:

1. **Compile chapters independently before assembly.** Currently `collection.lua` merges raw Markdown into one document before running filters. Chapters should be compiled through the filter pipeline individually, then assembled for issue/book layout. This eliminates cross-chapter filter interactions, the fragile ID isolation mechanism, and `statement-isolate.lua` entirely.

2. **Replace pandokoma with Pandoc's default LaTeX template.** The custom template (`dialectica.latex` from `dialoa/pandokoma`) must be manually updated for each Pandoc version (separate branches for 3.5 and 3.6). Instead, use Pandoc's default template and inject formatting via `header-includes`, `include-before`/`include-after`, and KOMA-Script class options. This makes the decoupling below much simpler — journal-specific formatting becomes preamble code in `journal.yaml` rather than a custom template.

## Context

The dltc-template is hardcoded to Dialectica in dozens of places: fonts, colors, title pages, logos, CSS, filters, CSL. To reuse the compilation machine for another journal (or even just to make the Dialectica config easier to reason about), we need to separate the generic infrastructure from journal-specific configuration.

## Current state: what's Dialectica-specific

### Easy (pure YAML values)

| What | Where | Dialectica value |
|------|-------|-----------------|
| Main font | `settings/common-latex.yaml` → `mainfont` | STIX Two Text |
| Sans font | same | Libertinus Sans |
| Mono font | same | Libertinus Mono |
| Math font | same | STIX Two Math |
| Title font | same → `setkomafonts.titlehead` | VenusSBOP-BoldExtended |
| KOMA heading fonts | same → `setkomafonts.*` | STIX Two Text + Libertinus Sans at specific sizes |
| Colors | same → `definecolors` | dialecticablue (0,31,79), dialecticalink (8,65,152) |
| Page size | same → `classoption` | 140mm×210mm |
| Margins | same → `classoption` DIV | 15 |
| CSL file | `paper-in-issue-*.yaml` → `csl` | `dialectica.csl` |
| Language | `settings/common.yaml` → `lang` | `en` |
| French spacing | same | `true` |
| Section depth | same → `secnumdepth` | 3 |
| Indent | same | `true` |
| MathJax URL | `settings/common-html.yaml` | cdn.jsdelivr.net MathJax 3 |
| Crossref prefixes | `settings/paper-in-issue.yaml` | "figure", "table", "section", etc. |
| HTML Google Font | `templates/dialectica.html` | STIX Two Text |
| Imagify zoom | `settings/common-html.yaml`, `settings/paper-in-issue.yaml` | 1.5, 1.6 |

### Medium (templated blocks needing parameterization)

| What | Where | Notes |
|------|-------|-------|
| Offprint title page | `settings/offprint-latex.yaml` → `rawtitlecode` | ~80 lines of raw LaTeX: journal name in VenusSBOP, volume/issue header, CC-BY logo, ESAP logo |
| Issue cover + inside pages | `settings/issue-latex.yaml` → `rawtitlecode` | ~200 lines: blue cover, editorial board, TOC sections, back cover with ISSN/sponsors |
| Book title pages | `settings/book-latex.yaml` → `rawtitlecode` | ~150 lines: half-title, series, main title, contributors, copyright |
| CSS | `templates/dialectica.css` | Font families, link colors, heading sizes, grid baseline |
| HTML template | `templates/dialectica.html` | Google Fonts link, Dialectica CSS import |
| Logo .tex files | `media/cc-by-logo.tex`, `esap-logo.tex`, `openaccess-logo.tex` | Dialectica-specific logos |
| Unicode char mappings | `settings/common-latex.yaml` → `text-unicode-to-latex` | Extensive; may vary per journal |

### Hard (filter logic)

| What | Where | Notes |
|------|-------|-------|
| `dialectica-meta-format.lua` | `filters/` (648 lines) | Formats issue metadata: `\printtitle`, `\printvolume`, editorial board lists, logo macros. Deeply tied to Dialectica's editorial structure |
| `dialectica-pre-format.lua` | `filters/` (39 lines) | Sets up imagify fonts — could be generic if font config is externalized |
| `dialectica-post-format.lua` | `filters/` (115 lines) | ORCID icons, author signature blocks, HTML author display. Partially generic (ORCID is standard), partially Dialectica layout |
| `scholarly-metadata.lua` | `filters/` (199 lines) | Generic — normalizes author metadata |
| `scholarly-format.lua` | `filters/` (62 lines) | Generic — sets `lastand` to '&', checks correspondence email |

## Target architecture

### Layered configuration

```
dltc-template/
├── scripts/                    # Generic — unchanged
│   ├── make.lua
│   ├── make.sh
│   └── refs.lua
│
├── filters/                    # Generic filters (most already are)
│   ├── collection.lua
│   ├── statement.lua
│   ├── columns.lua
│   ├── first-line-indent.lua
│   ├── labelled-lists.lua
│   ├── imagify.lua
│   ├── recursive-citeproc.lua
│   ├── bib-place.lua
│   ├── scholarly-metadata.lua
│   ├── scholarly-format.lua
│   ├── sections-to-meta.lua
│   ├── image-attributes.lua
│   ├── image-embedder.lua
│   ├── display-image.lua
│   ├── not-in-format.lua
│   ├── secnumdepth.lua
│   ├── latex-fixes.lua
│   ├── fix-doi-links.lua
│   └── meta-format.lua         # RENAMED from dialectica-meta-format.lua
│                                # parameterized to read journal config from metadata
│
├── templates/                   # Base templates (parameterized)
│   ├── base.latex               # KOMA base (from pandokoma) — reads fonts/colors from metadata
│   ├── base.html                # Base HTML structure
│   ├── base.css                 # Base CSS with CSS custom properties (variables)
│   ├── paper-in-issue.latex
│   ├── paper-in-issue.html
│   └── paper-in-book.latex
│
├── settings/                    # Base settings (sensible defaults)
│   ├── common.yaml
│   ├── common-html.yaml
│   ├── common-latex.yaml        # No hardcoded fonts/colors — reads from journal config
│   ├── offprint-latex.yaml      # Title page code uses template variables, not hardcoded
│   ├── issue-latex.yaml
│   ├── book-latex.yaml
│   ├── collection.yaml
│   └── paper-in-issue.yaml
│
├── defaults/                    # Base pandoc defaults
│   ├── d_common.yaml
│   ├── d_offprint.yaml
│   ├── d_issue.yaml
│   └── d_book.yaml
│
├── journals/                    # Journal-specific overrides
│   └── dialectica/
│       ├── journal.yaml         # THE main config: fonts, colors, geometry, metadata
│       ├── dialectica.csl       # Citation style
│       ├── dialectica.css       # CSS overrides (colors, font sizes)
│       ├── pre-format.lua       # Journal-specific pre-format filter
│       ├── post-format.lua      # Journal-specific post-format filter
│       ├── title-pages/
│       │   ├── offprint.yaml    # rawtitlecode for offprint
│       │   ├── issue.yaml       # rawtitlecode for issue cover + inside
│       │   └── book.yaml        # rawtitlecode for book
│       └── media/
│           ├── cc-by-logo.tex
│           ├── esap-logo.tex
│           ├── openaccess-logo.tex
│           └── ORCID-iD-icon.svg
│
└── platform/                    # Platform defaults (just crossref binary)
    ├── paper-in-issue-nix.yaml
    ├── paper-in-issue-mac.yaml
    └── paper-in-issue-win.yaml
```

### journal.yaml — the single config file

```yaml
# journals/dialectica/journal.yaml

journal:
  name: "Dialectica"
  issn-print: "0012-2017"
  issn-electronic: "1746-8361"

fonts:
  main: "STIX Two Text"
  main-options:
    - "Extension=.otf"
    - "UprightFont=*-Regular"
    - "BoldFont=*-Bold"
    - "ItalicFont=*-Italic"
    - "BoldItalicFont=*-BoldItalic"
  sans: "Libertinus Sans"
  mono: "Libertinus Mono"
  math: "STIX Two Math"
  title: "VenusSBOP-BoldExtended"
  title-size: "30pt"

colors:
  primary:
    name: "journalblue"
    model: "RGB"
    value: "0,31,79"
  link:
    name: "journallink"
    model: "RGB"
    value: "8,65,152"

geometry:
  paper: "140mm:210mm"
  div: 15
  bcor: "0mm"
  fontsize: "10pt"

csl: "dialectica.csl"
css: "dialectica.css"
lang: "en"

title-pages:
  offprint: "title-pages/offprint.yaml"
  issue: "title-pages/issue.yaml"
  book: "title-pages/book.yaml"

filters:
  pre-format: "pre-format.lua"
  post-format: "post-format.lua"

media:
  logo-cc: "media/cc-by-logo.tex"
  logo-esap: "media/esap-logo.tex"
  logo-oa: "media/openaccess-logo.tex"
  icon-orcid: "media/ORCID-iD-icon.svg"
```

### How make.lua uses it

`make.lua` gets a new `journal` argument or auto-detects the journal config:

```bash
dltc-make offpdf                          # auto-detect journal.yaml
dltc-make offpdf journal=dialectica       # explicit
```

`make.lua` changes:
1. Find `journal.yaml` (in journal dir or passed as arg)
2. Add it as `--metadata-file journal.yaml` early in the pandoc command
3. Title page yamls are loaded from the journal dir
4. Journal-specific filters added to the pipeline

### How settings files change

Pandoc metadata files are static YAML — they don't support variable interpolation. Instead, the journal config provides values directly as metadata, and Pandoc's `--metadata-file` override order handles the rest: later files override earlier ones.

The base `common-latex.yaml` is stripped down to non-journal-specific values (unicode mappings, package includes, structural settings). Journal-specific values (fonts, colors, geometry) move into `journal.yaml`, which is loaded with higher priority:

```bash
pandoc \
  -d d_common.yaml \
  --metadata-file settings/common.yaml \
  --metadata-file settings/common-latex.yaml \       # base: packages, unicode, structure
  --metadata-file journals/dialectica/journal.yaml \  # overrides: fonts, colors, geometry
  --metadata-file settings/offprint-latex.yaml \
  ...
```

### How CSS changes

Base CSS uses CSS custom properties:
```css
:root {
  --journal-font: serif;
  --journal-link-color: #084198;
  --journal-heading-font: serif;
}
body { font-family: var(--journal-font); }
a { color: var(--journal-link-color); }
```

Journal CSS override:
```css
/* journals/dialectica/dialectica.css */
:root {
  --journal-font: 'STIX Two Text', serif;
  --journal-link-color: #084198;
  --journal-heading-font: 'Libertinus Serif Display', serif;
}
```

### How meta-format.lua changes

Currently `dialectica-meta-format.lua` hardcodes Dialectica editorial structure. To generalize:

1. Rename to `meta-format.lua`
2. Read journal metadata from `meta.journal` (populated by journal.yaml)
3. Generate LaTeX macros from whatever fields are present:
   - `meta.journal.name` → `\printjournalname`
   - `meta.journal.editorialboard` → `\printeditorialboard` (if present)
   - etc.
4. Skip fields that don't exist (not all journals have ESAP logos or consulting boards)

The Dialectica-specific parts (exact logo placement, exact title page layout) stay in the journal's title-page yamls and the journal's pre/post-format filters.

## Migration steps

| Step | What | Breaking? | Effort |
|------|------|-----------|--------|
| 1 | Extract font/color/geometry values from `common-latex.yaml` into `journals/dialectica/journal.yaml` | No (additive) | Small |
| 2 | Split `common-latex.yaml` into base (packages, unicode, structure) + journal values | Yes (reorder metadata files in make.lua) | Medium |
| 3 | Move CSL, CSS, logos into `journals/dialectica/` | Yes (update paths) | Small |
| 4 | Parameterize CSS with custom properties | No | Small |
| 5 | Refactor `dialectica-meta-format.lua` → `meta-format.lua` reading from `meta.journal` | Yes (filter rename + metadata structure change) | Medium |
| 6 | Move title page rawtitlecode into journal dir | Yes (path changes) | Small |
| 7 | Refactor `dialectica-pre-format.lua` to read fonts from journal config | Small | Small |
| 8 | Refactor `dialectica-post-format.lua` — split generic (ORCID) from Dialectica-specific | Medium | Medium |
| 9 | Update `make.lua` to find and load journal config | Yes | Medium |
| 10 | Update all platform yamls for new paths | Yes | Small |
| 11 | Test: compile existing articles with new structure, diff output | — | Medium |

**Total estimated effort**: 2–3 focused sessions.

Steps 1–4 are safe and incremental. Steps 5–9 are the breaking changes that should be done together. Step 11 is critical — the output PDFs/HTMLs should be identical before and after.

## What a second journal would need

Once the decoupling is done, adding a new journal requires:

1. Create `journals/newjournal/journal.yaml` with fonts, colors, geometry
2. Create or copy a CSL file
3. Create a CSS override file
4. Create title page yamls (can start from Dialectica's and modify)
5. Optionally: custom pre/post-format filters (or use the generic ones)
6. Provide fonts (in Docker image or system)
7. Provide logos (in media/)

No changes to the base template, filters, or make.lua needed.

## Open questions

1. **Pandoc template variables**: The LaTeX/HTML templates use `$mainfont$`, `$definecolors$`, etc. These are populated from metadata. If journal.yaml provides them, they'll work. But `rawtitlecode` contains raw LaTeX that references specific fonts by name — this can't be parameterized via metadata. Title pages will always be somewhat journal-specific.

2. **Font installation**: Fonts must be installed in the system/Docker image. Each journal may need different fonts. The Docker image would need to either include all fonts or be parameterized per journal.

3. **Template naming**: Currently `templates/dialectica.latex`. Rename to `templates/base.latex` or keep as-is? The `pandokoma` upstream repo calls it `pandokoma-bare.latex`. The Makefile renames it to `dialectica.latex` during `make update`.

4. **Filter pipeline differences**: Different journals might need different filter sets. The platform yamls currently hardcode the full pipeline. A `journal.filters.pipeline` config is possible but complex. Initial assumption: all journals use the same pipeline.
