# Export

Only on explicit request. By default the output is a single HTML file.

---

## PDF (`--pdf`)

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu \
  --no-pdf-header-footer --print-to-pdf="NAME.pdf" "file://$PWD/NAME.html"
```

Nothing else needs installing — Chrome is already there. On Linux use `google-chrome` or `chromium` instead of the application path.

The PDF goes next to the HTML, under the same name.

---

## Printing breaks diagrams — do not ship without this CSS

Nodes and tables get torn in half across pages. The template already sets up `@media print`; when hand-writing markup, make sure it includes:

```css
@media print {
  .flow-wrap, .flow, .node, .branch,
  .layers, .layer, .tbl-wrap, .card,
  pre, svg, figure  { break-inside: avoid; }

  h2, h3            { break-after: avoid; }   /* heading stays with its text */
  nav.toc           { display: none; }        /* a sticky ToC is noise in print */
  a[href^="http"]::after { content: " (" attr(href) ")"; font-size: .85em; }
}

@page { margin: 14mm 12mm; }

.page-break { break-before: page; }
```

`@page` is mandatory: without it Chrome applies its own margins and clips wide diagrams.

`.page-break` is for a manual break where the content calls for one (before a major section, before an appendix).

---

## Theme when printing

Printing is always in the light theme. Even if the document is open in dark mode, the headless render takes light — `prefers-color-scheme` defaults to light in headless. No separate check is needed, but `body` must set its background explicitly, otherwise the result is transparent.

---

## Markdown (`--md`)

Goes next to the HTML, under the same name: `docs/baxi.md`.

Markdown is **not a mechanical conversion** of the HTML but a condensed version for reading in git and in diffs:

- diagrams become fenced code blocks with ASCII arrows, not HTML markup;
- tables carry over as they are;
- cards and chips unfold into plain lists;
- links to live sources are preserved.

The point is that the markdown shows up in code review while the HTML is what you read with your eyes.

---

## Multiple files

When a document has outgrown itself and gets split:

```
docs/service-api/
├── index.html       <- table of contents and a short overview, links to the rest
├── read.html
├── write.html
└── permissions.html
```

`index.html` carries the overall diagram (input → output as a whole) and one-line descriptions linking to each file. Every file links back to the index in its header.

With `--pdf` over a set of files, the PDF is built for the index only, unless stated otherwise.
