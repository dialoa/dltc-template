# Plan: Structured Bibliography JSON Sidecar Filter

## Context

The HTML articles compiled by the Dialectica template lose all structured bibliography data. Pandoc's citeproc renders references into flat presentation markup — `<div class="csl-entry">` with inline text and minimal spans (`smallcaps`, `nocase`, `em`). There's no way to programmatically extract author names, years, titles, volumes, pages, etc. from the rendered HTML.

This blocks post-processing for the philosophie.ch website: reformatting references, generating linked data/metadata (JSON-LD, Dublin Core, OpenURL), and cross-linking to other Dialectica articles or DOIs.

**Solution**: A new Lua filter that, when activated via a metadata flag, embeds the full structured CSL-JSON bibliography data as a `<script type="application/json">` block in the HTML output. The rendered references stay untouched — the JSON block is purely additive.

## How citeproc works (relevant internals)

1. Article frontmatter has `bibliography: path/to/file.bib`
2. `recursive-citeproc.lua` calls `pandoc.utils.citeproc(doc)` which:
   - Reads the .bib file and converts to CSL-JSON internally
   - Resolves all `@bibkey` citations
   - Generates a `<div id="refs" class="references csl-bib-body">` with rendered entries
3. `bib-place.lua` extracts that Div and stores it in `meta.referencesblock`
4. The template inserts `$referencesblock$` into the HTML

The structured CSL-JSON is available to Pandoc internally during citeproc but is **not exposed** in the output. However, we can independently read the .bib file and convert it to CSL-JSON using `pandoc.read()` with the `bibtex` reader.

## Implementation

### New filter: `bib-data.lua`

**Location**: `filters/bib-data.lua`

**When it runs**: After `bib-place.lua` (position 15 in the pipeline), before `scholarly-metadata.lua`. It needs the `#refs` Div to already exist so it can extract the bibkeys that were actually cited.

**Activation**: Only runs when metadata flag `bib-data` is truthy. Default: off.

```yaml
# In article frontmatter or settings:
bib-data: true
```

**Logic**:

1. Check `meta.bib-data` — if falsy or FORMAT is not html, return immediately
2. Read `meta.bibliography` to get the .bib file path(s)
3. Read the .bib file(s) and parse with `pandoc.read(contents, 'bibtex')`
4. Extract CSL-JSON references from the parsed document's metadata (`meta.references`)
5. Collect the set of bibkeys actually cited in the document (scan `meta.referencesblock` for `id="ref-*"` divs, or scan the document for Cite elements)
6. Filter the CSL-JSON to only include cited entries
7. Serialize to JSON string
8. Append a `RawBlock('html', '<script type="application/json" id="bib-data">...</script>')` to the document body (after the references block)

### Key technical details

**Reading BibTeX as CSL-JSON via Pandoc**:
```lua
local bib_content = read_file(bib_path)
local bib_doc = pandoc.read(bib_content, 'bibtex')
local refs = bib_doc.meta.references  -- this is a MetaList of CSL-JSON objects
```

Each entry in `refs` is a Pandoc MetaMap with CSL-JSON fields: `id`, `type`, `author` (list of `{family, given, ...}`), `issued` (date-parts), `title`, `container-title`, `volume`, `issue`, `page`, `DOI`, `URL`, `publisher`, `editor`, etc.

**Serializing to JSON**: Use `pandoc.json.encode()` (available in Pandoc 3.x) or build a custom serializer that walks the MetaMap.

**Finding cited bibkeys**: Scan the `referencesblock` MetaBlocks for Div elements with `id` starting with `ref-`. Extract the bibkey from the id (strip any collection prefix like `c1-`).

### Pipeline position

Current pipeline (from `paper-in-issue-nix.yaml`):
```
...
13. recursive-citeproc.lua
14. bib-place.lua
15. ← bib-data.lua goes HERE (new)
16. fix-doi-links.lua
17. scholarly-metadata.lua
...
```

### Files to create/modify

| File | Action | What |
|------|--------|------|
| `filters/bib-data.lua` | **Create** | New filter (~60-100 lines) |
| `paper-in-issue-nix.yaml` | Modify | Add `${.}/filters/bib-data.lua` after `bib-place.lua` |
| `paper-in-issue-mac.yaml` | Modify | Same |
| `paper-in-issue-win.yaml` | Modify | Same |
| `paper-in-book-nix.yaml` | Modify | Add filter (if books need it too) |
| `paper-in-book-mac.yaml` | Modify | Same |
| `paper-in-book-win.yaml` | Modify | Same |
| `docs/filters.md` | Modify | Document the new filter |

### Output format

```html
<script type="application/json" id="bib-data">
{
  "smith:2024": {
    "id": "smith:2024",
    "type": "article-journal",
    "author": [{"family": "Smith", "given": "Jane"}],
    "issued": {"date-parts": [[2024]]},
    "title": "Some Title",
    "container-title": "Dialectica",
    "volume": "78",
    "issue": "1",
    "page": "1-25",
    "DOI": "10.48106/dial.v78.i1.1234"
  },
  "jones:2020": {
    ...
  }
}
</script>
```

Keyed by bibkey for easy lookup. Only cited entries are included (not the entire .bib file).

## Verification

1. Pick a test article with known references
2. Compile with `bib-data: false` (default) → verify no `<script>` block in HTML
3. Compile with `bib-data: true` → verify `<script id="bib-data">` appears with valid JSON
4. Parse the JSON and verify: all cited references present, correct fields (author, year, title, etc.), no uncited entries leaked
5. Verify the rendered references section is unchanged (diff the HTML before/after, excluding the new script block)
6. Verify PDF output is unaffected (filter should be a no-op for non-HTML formats)
