# MyST / Sphinx Rendering Guide

This guide is for implementing MyST/Sphinx-aware rendering of markdown cells on
ALPS or any similar notebook platform.

It is based on:

- the syntax actually used in this repo's markdown cells
- the MyST extensions enabled in [`content/_config.yml`](./content/_config.yml)
- the Jupyter Book / Sphinx rendering model used by this textbook

The objective is not to support "all possible MyST". The objective is to
support the subset required to render this repo correctly and predictably.

## 1. Rendering Goal

The renderer should correctly process markdown cells that contain:

- plain Markdown
- MyST roles
- MyST directives
- MyST labels and cross-references
- MyST math
- MyST extension syntax enabled by the repo config

The renderer should preserve notebook source as-is. Do not rewrite author
content into ALPS-specific markdown. Instead, implement a preprocessor that
parses notebook markdown and emits normalized HTML or a platform-native rich
rendering model.

## 2. Enabled Extensions From This Repo

This repo enables the following MyST extensions in
[`content/_config.yml`](./content/_config.yml):

- `amsmath`
- `colon_fence`
- `deflist`
- `dollarmath`
- `html_admonition`
- `html_image`
- `linkify`
- `replacements`
- `smartquotes`
- `substitution`

It also enables the Sphinx extension:

- `sphinx_multitoc_numbering`

## 3. Syntax Inventory Found In This Repo

A scan of markdown in `content/**/*.ipynb` and `content/**/*.md` found the
following MyST/Sphinx constructs in active content.

### Inline Roles That Must Be Supported

- `{numref}` for numbered references to chapters, sections, figures, and tables
- `{ref}` for label-based cross-references
- `{cite}` for citations

Examples:

```md
{numref}`Figure %s <sample-urn>`
{numref}`Chapter %s <ch:data_scope>`
{ref}`ch:pandas`
{cite}`grotenhuis2018`
```

### Block Directives That Must Be Supported

- `figure`
- `image`
- `note`
- `warning`
- `sidebar`
- `table`

Examples:

````md
```{figure} figures/ds-lifecycle.svg
---
name: ds-lifecycle
---
The four high-level stages of the data science lifecycle.
```
````

````md
```{warning}
This element indicates a warning or caution.
```
````

```md
:::{table} Prefixes for common file sizes.
:name: byte-prefixes
```
```

### Labeling Forms That Must Be Supported

- directive front matter labels via `name:`
- directive option labels via `:name:`
- anchor labels in heading-adjacent form such as `(ch:lifecycle)=`

Examples:

```md
(ch:lifecycle)=
# The Data Science Lifecycle
```

```md
:::{table} Prefixes for common file sizes.
:name: byte-prefixes
```

### Math That Must Be Supported

Because `dollarmath` and `amsmath` are enabled, the renderer should support:

- inline math with `$...$`
- display math with `$$...$$`
- AMS-style math blocks if present

### Extension Features That Should Be Supported

These may be less visible than directives/roles but are enabled in the repo and
should be handled for compatibility:

- `colon_fence`
- `deflist`
- `html_admonition`
- `html_image`
- `linkify`
- `replacements`
- `smartquotes`
- `substitution`

## 4. Capability Matrix

The preprocessor should have the following capabilities.

| Feature | Capability required | Notes |
|---|---|---|
| `{numref}` | Resolve target label and substitute computed number | Must support patterns like `Figure %s`, `Chapter %s`, `Table %s` |
| `{ref}` | Resolve target label and generate hyperlink text | If no explicit text is supplied, use target title/caption |
| `{cite}` | Resolve citation key if bibliography is available | If bibliography is unavailable, degrade gracefully |
| `figure` | Render image as `<figure>` with caption and label | Must create registry entry for numbering |
| `image` | Render standalone image block | May optionally normalize to `<figure>` without numbering |
| `note` / `warning` | Render admonition blocks | Support fenced and colon-fence forms |
| `sidebar` | Render aside/boxed callout | Keep title if supplied |
| `table` | Render captioned table block and register label | Needed for `{numref}` to table labels |
| labels | Register anchors for cross-reference resolution | Labels may come before headings or inside directives |
| math | Forward to MathJax/KaTeX or equivalent | Preserve source fidelity |
| substitutions | Expand configured substitutions if used | If none configured, safe no-op |
| smartquotes/replacements/linkify | Apply final text transformations | These are post-parse formatting behaviors |

## 5. Recommended Rendering Architecture

Use a source-preserving preprocessor with three phases.

### Phase 1: Detect

Only inspect markdown cells.

Mark a cell as MyST-aware if any of the following are present:

- `{numref}`, `{ref}`, `{cite}`
- fenced directives such as ```` ```{figure} ````
- colon fences such as `:::{note}`
- label syntax such as `(label)=`
- directive options such as `:name: some-label`
- math delimiters like `$...$` or `$$...$$`

Do not rely on cell metadata as the primary signal.

Optional override metadata is still useful:

```json
{
  "alps": {
    "render_mode": "myst"
  }
}
```

Use metadata only as an override, not as the main mechanism.

### Phase 2: Parse And Register

Parse markdown into an intermediate structure that recognizes:

- block nodes
- directive nodes
- role nodes
- headings
- labels
- caption text
- math nodes

During this phase, build a document registry:

- `labels[label] -> target node`
- `doc_titles[doc_id] -> heading text`
- `figures[label] -> sequential figure number`
- `tables[label] -> sequential table number`
- `sections[label] -> section/chapter number if applicable`
- `citations[key] -> bibliography entry if available`

This pass must happen before final rendering because `{numref}` and `{ref}`
depend on the full document context.

### Phase 3: Resolve And Render

Transform the parsed nodes into platform output.

Recommended output strategy:

- emit HTML for block directives and structural elements
- emit standard anchor links for references
- emit math nodes for MathJax/KaTeX handling
- emit warning markers in output when references cannot be resolved

## 6. Directive Handling Rules

### `figure`

Input pattern:

````md
```{figure} figures/example.svg
---
name: fig-example
---
Caption text
```
````

Renderer behavior:

- resolve the image path relative to the notebook document
- create a figure registry entry if a label exists
- assign a figure number based on document order
- render `<figure>`
- render `<img>`
- render `<figcaption>`
- expose the label as an anchor target

### `image`

Input pattern:

````md
```{image} vectors.png
```
````

Renderer behavior:

- resolve the image path
- render a simple image block
- if caption text exists, render as figure-like output
- no numbering unless your product explicitly decides to number images

### `note`, `warning`, `sidebar`

Renderer behavior:

- map to admonition/aside UI blocks
- keep the directive title if supplied
- support both fenced and colon-fence variants

Suggested HTML shape:

```html
<aside class="admonition note">
  <div class="admonition-title">Note</div>
  <div class="admonition-body">...</div>
</aside>
```

### `table`

Renderer behavior:

- parse caption and optional `:name:` label
- register table number
- render table wrapper with caption
- expose label as anchor target

## 7. Role Handling Rules

### `{numref}`

This is the most important role in this repo.

Examples:

```md
{numref}`Figure %s <sample-urn>`
{numref}`Chapter %s <ch:data_scope>`
{numref}`Table %s <byte-prefixes>`
```

Renderer behavior:

- extract the target label
- look up the target in the registry
- compute the target number
- replace `%s` with the resolved number
- create a hyperlink to the anchor

If the target type is known:

- figure -> `Figure N`
- table -> `Table N`
- chapter/section -> `Chapter N`, `Section N`, etc.

If the target exists but numbering is unavailable:

- fall back to the caption/title text plus link

If the target does not exist:

- render the raw source visibly or render a styled unresolved-reference marker

### `{ref}`

Examples:

```md
{ref}`ch:pandas`
{ref}`ax:extra_reading`
```

Renderer behavior:

- resolve the label
- use target heading/caption text as link text unless explicit text is present
- hyperlink to the anchor

### `{cite}`

Examples:

```md
{cite}`grotenhuis2018`
{cite}`cdc2021`
```

Renderer behavior:

- if bibliography data is available, render citation text and link to
  bibliography entry
- if bibliography data is not available, render a graceful fallback such as
  `[grotenhuis2018]`

Important note for this repo:

- [`content/_config.yml`](./content/_config.yml) has `bibtex_bibfiles` commented
  out
- therefore citation support in ALPS should be treated as optional unless the
  bibliography source is supplied from elsewhere

## 8. Label And Numbering Strategy

The preprocessor should maintain a label registry for the whole document.

Minimum required registry fields:

| Field | Meaning |
|---|---|
| `label` | symbolic target id like `sample-urn` or `ch:pandas` |
| `kind` | figure, table, section, chapter, doc, citation |
| `number` | computed display number if applicable |
| `title` | heading/caption text |
| `href` | rendered anchor |
| `source_cell` | source markdown cell index |

Recommended numbering policy:

- chapters/sections should follow the textbook TOC or document heading order
- figures should be numbered in document order
- tables should be numbered in document order

Because this repo uses chapter-style labels such as `ch:pandas`, the best
results come from integrating notebook rendering with a TOC-aware document map.

## 9. Failure Handling

Do not silently drop unsupported syntax.

Recommended behavior:

- unresolved reference -> render a visible warning token and log it
- unsupported directive -> render source in a fallback block and log it
- missing image -> render a broken-image placeholder and log it
- citation without bibliography -> render fallback citation token and log it

This is much better than pretending content rendered successfully when it did
not.

## 10. Suggested Detection Rules

Use source-based auto-detection first.

A markdown cell should be treated as MyST-aware if it matches any of:

- `\{numref\}`
- `\{ref\}`
- `\{cite\}`
- `^```{[a-zA-Z0-9_-]+}`
- `^:::{[a-zA-Z0-9_-]+}`
- `^\([a-zA-Z0-9:_-]+\)=$`
- `:name:`
- inline or display math delimiters

Metadata can be used as an override:

- force MyST rendering
- force plain Markdown rendering

But metadata should not be mandatory for existing notebooks.

## 11. Recommended Implementation Model

The most maintainable approach is:

1. Notebook markdown cell input
2. MyST-aware parser / tokenizer
3. registry build pass
4. reference resolution pass
5. HTML or platform-native render output

If ALPS already has a Markdown renderer, do not try to patch raw strings with
regex substitutions only. Regex is acceptable for detection, but not for full
reference resolution.

For correctness, the implementation should behave more like:

- parse -> transform -> render

and less like:

- search/replace raw markdown text

## 12. Minimum Acceptance Criteria

The implementation should be considered ready only if it can render the
following repo patterns correctly:

- `{numref}` to chapters, sections, figures, and tables
- `{ref}` links to chapter/section labels
- `figure` blocks with labels and captions
- `table` blocks with captions and labels
- `note`, `warning`, and `sidebar` admonitions
- inline and display math
- heading labels like `(ch:lifecycle)=`
- colon-fence directives

## 13. Recommended Test Cases

Use these representative files when testing:

- [`content/ch/01/lifecycle_cycle.ipynb`](./content/ch/01/lifecycle_cycle.ipynb)
  for `figure` and `{numref}`
- [`content/ch/08/files_size.ipynb`](./content/ch/08/files_size.ipynb)
  for `table`, `:name:`, and notes
- [`content/ch/07/sql_intro.ipynb`](./content/ch/07/sql_intro.ipynb)
  for `{ref}`
- [`content/preface.md`](./content/preface.md)
  for fenced admonitions
- [`content/ch/12/pa_modeling.ipynb`](./content/ch/12/pa_modeling.ipynb)
  for `sidebar`

## 14. Official References

- MyST roles: https://mystmd.org/guide/roles
- MyST directives: https://mystmd.org/guide/directives
- MyST overview: https://mystmd.org/guide/overview
- Sphinx Markdown / MyST parser support:
  https://www.sphinx-doc.org/en/master/usage/markdown.html

## 15. Team Recommendation

For this textbook and similar notebook content, the recommended product
strategy is:

- keep author source unchanged
- detect MyST from markdown cell content
- preprocess markdown into a structured intermediate representation
- resolve labels and numbering with a registry pass
- render normalized HTML output

That is simpler than inventing ALPS-specific markdown and more reliable than a
metadata-only routing model.
