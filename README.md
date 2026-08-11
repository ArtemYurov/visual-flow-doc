# visual-flow-doc

An agent skill that produces readable HTML documents with block diagrams of algorithms — both current (traced from code) and planned (from discussion).

Not an image generator. A living document: created once, then extended, rewritten and split into files as understanding and code change.

## What you get

A single HTML file that opens with a double click:

- zero `<script>`, zero external URLs, no Mermaid and no CDN;
- diagrams assembled from `div` elements — the browser lays them out, no coordinates to compute;
- light and dark themes;
- on request — PDF (headless Chrome, with correct page breaks) and a markdown version.

## Principles

**Input and output are named explicitly.** If you cannot name them, the algorithm is not understood yet — it is too early to draw.

**Code is the source of truth.** Plans, docs and docblocks go stale; when they disagree with the code, the code wins.

**Connections are never invented.** Every node and every arrow follows from a real call, injection, `dispatch`, HTTP request or database query.

**The subject is the algorithm and its branches, not implementation detail.** A box that only converts data stays closed; a box that decides the fate of a record gets opened, because the branches inside it are the document.

**Three sweeps, not one.** The first follows the trunk. The second hunts what it structurally misses — early exits, `catch` paths, cache hit/miss, cascades, outcomes sharing an exit code. The third looks at the whole codebase rather than any single flow: what the thing is *not* for, how destructive each entry point is, what is declared but never used, what is implemented three times, and where intent and implementation disagree.

**There are no modes.** Whether the file exists is something the skill looks up. Creating, extending, rewriting and splitting off a separate file are one operation.

## Install

As a plugin (recommended — updates through `/plugin`):

```
/plugin marketplace add ArtemYurov/visual-flow-doc
/plugin install visual-flow-doc@visual-flow-doc
```

As a skill, globally:

```bash
ln -s "$PWD/skills/visual-flow-doc" ~/.claude/skills/visual-flow-doc
```

As a skill, in one project:

```bash
cp -R skills/visual-flow-doc <project>/.claude/skills/
```

## Usage

```
/visual-flow-doc show the current import flow, follow the code
/visual-flow-doc record the new stage in docs/baxi.html
/visual-flow-doc split permissions into a separate file, add an index
/visual-flow-doc docs/baxi.html --pdf
```

The skill also triggers on natural phrasing in Russian — that is where it grew up.

## Layout

```
.claude-plugin/
├── plugin.json                 plugin manifest
└── marketplace.json            the repo is its own marketplace
skills/visual-flow-doc/
├── SKILL.md                    main instructions
├── references/
│   ├── VOCABULARY.md           class and markup vocabulary
│   ├── TRACING.md              rules for tracing through code
│   └── EXPORT.md               PDF, markdown, multi-file documents
└── templates/
    └── skeleton.html           skeleton with the complete CSS
```

## Palette

The template's accent colors were validated with the `dataviz` skill's `validate_palette.js`. The light theme passes every check. In the dark theme the warn/ok pair sits below the separation floor — a documented trade-off that holds because every node is labeled with text: color is a hint, never the carrier of meaning.

## License

MIT
