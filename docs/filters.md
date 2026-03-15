# Filter Pipeline Reference

The template uses 23 Pandoc Lua filters. 20 run in the main per-paper pipeline, and 3 are loaded separately for specific modes.

## Main pipeline

These filters run in this exact order for every paper (defined in `paper-in-issue-{nix,mac,win}.yaml`). Order matters — see the dependency notes.

### 1. dialectica-pre-format.lua
**Purpose**: Pre-processing setup for papers in issues.
**What it does**: Configures font settings for the `imagify` filter (STIX Two, unicode-math, XeLaTeX).
**AST**: Meta
**Config**: Reads `meta.imagify` flag.

### 2. sections-to-meta.lua
**Purpose**: Extracts metadata sections from the document body into YAML metadata fields.
**What it does**: Scans document blocks for sections named `Abstract`, `Thanks`/`Acknowledgments`, `Keywords`, or `Review of`. Moves their content into the corresponding metadata fields. Converts keyword bullet lists into MetaList. Stops scanning at the first body-text heading.
**AST**: Blocks, Meta
**Config**: Section header aliases (e.g., "summary" = "abstract", "acknowledgements" = "thanks").
**Dependency**: Must run before pandoc-crossref.

### 3. not-in-format.lua
**Purpose**: Conditional content inclusion/exclusion by output format.
**What it does**: Removes Div blocks with class `not-in-format` if the current output format matches, or keeps only Divs with class `only-in-format` if format matches. Uses regex matching on the FORMAT variable.
**AST**: Div
**Config**: Div classes `not-in-format`, `only-in-format`.

### 4. secnumdepth.lua
**Purpose**: Section numbering depth control for non-LaTeX formats.
**What it does**: Reads `secnumdepth` from metadata (default: 6). Adds `unnumbered` class to headers deeper than this level. Only runs for non-LaTeX formats (LaTeX handles this natively).
**AST**: Header, Meta

### 5. image-attributes.lua
**Purpose**: Image centering and attribute handling.
**What it does**: Wraps images with class `center` in LaTeX `\begin{center}...\end{center}` or HTML `margin:auto; display:block`. Distinguishes inline images from figures via `fig:` title prefix.
**AST**: Image

### 6. first-line-indent.lua
**Purpose**: First-line paragraph indentation.
**What it does**: Inserts `\indent` or `\noindent` raw code in LaTeX, equivalent CSS in HTML. Automatically removes indentation after certain block types (lists, code blocks, block quotes). Highly configurable.
**AST**: Blocks, Inlines, Div, BlockQuote, Meta
**Config**: Metadata fields `indent` (boolean), `remove_after` (block types), `remove_after_class`, `dont_remove_after_class`, `size` (indent length), `recursive`.
**Dependency**: Must run before `statement` and `labelled-lists` (which generate raw code that would break indent detection).

### 7. statement.lua (5929 lines — largest filter)
**Purpose**: Theorem-like environments (statements, principles, definitions, theorems, proofs, exercises, etc.).
**What it does**: Transforms Divs with `statement` class into LaTeX `\newtheorem` environments or styled HTML. Handles numbering, custom names, cross-referencing, QED symbols. Generates LaTeX preamble code for theorem definitions.
**AST**: Div, Header, Meta
**Config**: Extensive `statement` metadata field with environment definitions, styles, numbering schemes.
**Dependency**: Generates raw LaTeX. Must come after first-line-indent.

### 8. labelled-lists.lua
**Purpose**: Custom-labeled lists (e.g., premise numbering like (P1), (P2)).
**What it does**: Converts bullet lists with inline label Spans into semantically formatted labeled lists. Provides HTML (with embedded CSS), LaTeX, and JATS output.
**AST**: BulletList, Span
**Config**: `options.delimiter` (default: `()`), `options.disable_citations`.
**Dependency**: Generates raw code. Must come after first-line-indent.

### 9. columns.lua
**Purpose**: Multi-column layout.
**What it does**: Wraps Divs with `columns` class in LaTeX `multicols` environment or HTML flexbox CSS. Supports nested columns and width specification.
**AST**: Div, Meta
**Config**: Div classes `columns`, `column`; metadata `columns-raggedcolumns`.

### 10. imagify.lua
**Purpose**: Render content as embedded PDF images via XeLaTeX.
**What it does**: Detects Divs with `imagify` class, renders them to PDF via an XeLaTeX subprocess, converts to images, and embeds the result. Used for complex content that doesn't render well in HTML.
**AST**: Meta, Div
**Config**: `meta.imagify` flag (set by dialectica-pre-format).
**Dependency**: Requires dialectica-pre-format to have set up fonts.

### 11. display-image.lua
**Purpose**: Center display images with format-aware styling.
**What it does**: Wraps images with `display` class in appropriate centering code (LaTeX `\begin{center}`, HTML margin CSS, or plain line breaks).
**AST**: Image

### 12. pandoc-crossref (binary — not Lua)
**Purpose**: Cross-reference processing for figures, tables, equations, sections.
**What it does**: Converts `{#fig:name}` identifiers and `@fig:name` citations into formatted, numbered cross-references with hyperlinks.
**AST**: Header, Image, Table, CodeBlock, Link
**Config**: pandoc-crossref options in metadata (autoSectionLabels, prefix labels, etc., set in `settings/paper-in-issue.yaml`).
**Dependency**: Must come after sections-to-meta. Must come before citeproc.

### 13. recursive-citeproc.lua
**Purpose**: Citation processing with recursive bibliography support.
**What it does**: Runs Pandoc's citeproc to resolve `@bibkey` citations into formatted references. Handles self-citing bibliographies and recursive citation chains (a bibliography entry that itself cites other entries).
**AST**: Cite, Link, Meta
**Config**: CSL file, bibliography file from article metadata.
**Dependency**: Must come after pandoc-crossref.

### 14. bib-place.lua
**Purpose**: Move bibliography from document body to metadata for template-controlled placement.
**What it does**: Scans the document backwards for the `refs` Div (generated by citeproc). Extracts it along with any preceding heading. Stores in `meta.referencesblock` so the template can place it anywhere.
**AST**: Pandoc, Div
**Dependency**: Must come after citeproc (which generates the bibliography).

### 15. fix-doi-links.lua
**Purpose**: Improve line-breaking of DOI links in LaTeX.
**What it does**: Detects links matching `https://doi.org/` and wraps their content in LaTeX `\url{}` for better line-breaking behavior. LaTeX-only.
**AST**: Link
**Dependency**: After citeproc (DOI links come from bibliography).

### 16. scholarly-metadata.lua
**Purpose**: Normalize author and affiliation metadata.
**What it does**: Converts author/affiliation data from various input formats into a structured format with `name`, `institute`, `email`, `correspondence`, `ORCID` fields. Handles comma-separated author lists and institute indexing.
**AST**: Meta

### 17. scholarly-format.lua
**Purpose**: Format normalized author metadata for template use.
**What it does**: Maps institute IDs to names. Checks correspondence author has email. Sets localization variable `lastand` to '&'.
**AST**: Meta
**Dependency**: Must come after scholarly-metadata.

### 18. dialectica-post-format.lua
**Purpose**: Generate ORCID icons, author signature blocks, and formatted HTML/LaTeX author displays.
**What it does**: Creates end-of-article author blocks with ORCID links, institution names, correspondence emails. Generates both HTML and LaTeX versions.
**AST**: Meta, Inline
**Dependency**: Must come after scholarly-format.

### 19. latex-fixes.lua
**Purpose**: Fix LaTeX-specific rendering issues.
**What it does**: Adds `\vspace{0em}` after footnotes that end with BlockQuote to prevent spacing issues. Includes a guard against empty content. LaTeX-only.
**AST**: Note

### 20. image-embedder.lua
**Purpose**: Embed external images as base64 data URIs.
**What it does**: Converts image file references to inline base64 data URIs for standalone HTML output (no external file dependencies). Includes a complete base64 encoder.
**AST**: Image

## Filters loaded outside the main pipeline

These are loaded via the mode-specific defaults files (`d_offprint.yaml`, `d_issue.yaml`, `d_book.yaml`), not the per-paper pipeline.

### 21. collection.lua (loaded via `-L` flag — always first)
**Purpose**: Import and merge multiple article sources into unified documents.
**What it does**: Reads the `imports` list from `master.md`. For each imported article: reads the file, optionally prefixes all IDs for isolation, gathers/passes/replaces metadata. For offprint mode, extracts only the specified chapter.
**AST**: Pandoc, Meta, Blocks
**Config**: `imports` (list), `collection` (mode, isolate, gather, pass, replace), `offprints` (same structure).
**Note**: This is the largest infrastructure filter (1398 lines). Always loaded first via the `-L` flag.

### 22. dialectica-meta-format.lua (loaded in d_offprint.yaml, d_issue.yaml, d_book.yaml)
**Purpose**: Format journal/book metadata for title pages, headers, footers.
**What it does**: Generates LaTeX macros and HTML variables for all issue-level presentation: `\printtitle`, `\printvolume`, `\printissue`, `\printdate`, `\printdoi`, editorial board lists, sponsor lists, logo macros. Produces the raw code that title page templates consume.
**AST**: Meta
**Config**: Reads extensive metadata: publication-type, volume, issue, date, ISSN, ISBN, editorial staff, sponsors, indexing services.

### 23. statement-isolate.lua (loaded in d_book.yaml)
**Purpose**: Fix duplicate theorem environment definitions in multi-chapter documents.
**What it does**: When multiple articles define the same theorem environment (e.g., both have a "Theorem" statement), `statement.lua` generates duplicate `\newtheorem` commands. This filter scans raw LaTeX blocks and renames duplicates by appending underscores to prevent compilation errors.
**AST**: RawBlock (LaTeX)
**Dependency**: Runs after statement.lua has processed all chapters in book/issue mode.

## Filter ordering: dependency diagram

```
dialectica-pre-format ──> imagify (font config needed)
sections-to-meta ──> pandoc-crossref (sections must be extracted first)
first-line-indent ──> statement, labelled-lists (raw code breaks indent detection)
pandoc-crossref ──> recursive-citeproc (refs before citations)
recursive-citeproc ──> bib-place (bibliography must exist first)
recursive-citeproc ──> fix-doi-links (DOI links from bibliography)
scholarly-metadata ──> scholarly-format ──> dialectica-post-format (metadata pipeline)
statement ──> statement-isolate (in book mode: dedup theorem defs)
```

## Which filters run in which mode

| Filter | Offprint | Issue | Book |
|--------|----------|-------|------|
| collection.lua (-L) | yes | yes | yes |
| Main pipeline (20 filters) | yes | yes | yes |
| scholarly-metadata (d_offprint) | yes | no | no |
| dialectica-meta-format | yes | yes | yes |
| statement-isolate | no | yes | yes |
