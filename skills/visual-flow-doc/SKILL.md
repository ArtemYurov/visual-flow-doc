---
name: visual-flow-doc
description: >-
  Creates and maintains readable HTML documents with block diagrams of algorithms —
  both current (traced from code) and planned (from discussion). Self-contained HTML
  with no scripts and no external dependencies, light and dark themes, optional PDF
  and markdown. Use when asked to: visualize a data flow, draw a flow, make a visual
  diagram of how something works, an algorithm diagram, a block diagram, layers of
  responsibility, an overview of inputs and outputs, a business process diagram; make
  an HTML flow doc; record something in an existing flow document; rewrite a document
  to match the current state of the code; split part of a document into a separate
  file with a table of contents. Also triggers on Russian phrasing: визуализировать
  поток данных, нарисуй флоу, схема работы, диаграмма алгоритма, блок-схема, слои
  ответственности, входные и выходные данные, HTML-доку flow, зафиксируй в flow
  документе, переоформи под актуальное состояние.
argument-hint: "[what to document or path to a document] [--pdf] [--md]"
allowed-tools: Read Write Edit Glob Grep
license: MIT
metadata:
  author: artemyurov
  version: "1.0"
  category: documentation
---

# visual-flow-doc

A readable HTML document with block diagrams: how the algorithm works now, and what it will become.

This is **not an image generator**. It is a living document — created once, then extended, rewritten and split into files as understanding and code change.

---

## The invariant

**The document must name its input and its output explicitly.**

If you cannot name them, the algorithm is not understood yet — it is too early to draw. Read the code first.

The invariant holds at every scale:

| scale | input | output |
|---|---|---|
| system | data sources | databases, consumers |
| subsystem | XML / HTTP / file | write to an API or a database |
| layers | what arrives from below | what decisions go down |
| method | arguments | return value, side effect |
| business process | what happened | what to do |

---

## Step 1. Decide what you are doing

Whether the file exists is a **fact — look it up**, do not ask. There are no modes: creating, extending and rewriting are one operation — bring the document in line with reality.

```
no file              -> create it
file exists          -> read it in full, check against reality, extend it
                        (keep its structure and style!)
asked to split       -> extend + split into files + an index with a table of contents
"is this documented?" -> read it and answer, create nothing
```

When extending, **never rewrite the whole document** unless explicitly asked. Edit surgically: insert a new section, refresh an outdated block, add a node to a diagram.

---

## Step 2. Decide the source of truth

```
argument names a path, class, command   -> CODE
"current", "how it works now"           -> CODE
"needs to be done", "plan", "problem"   -> DISCUSSION + plan
an existing document is given           -> the document, checked against code
unclear                                 -> ask ONE question and wait
```

**Code always outweighs documentation.** Plans, research notes, old docs and docblocks go stale. When they disagree with the code, the code wins — and the disagreement is worth mentioning in the document.

---

## Step 3. Decide the scale

If the level is named in the argument ("by layers", "high level", "this method") — do not ask.

Otherwise judge the subject yourself, **state a recommendation** and offer the choice through `AskUserQuestion`. Not a bare question — a recommendation with alternatives:

```
SYSTEM       all sources -> all sinks, no implementation detail
SUBSYSTEM    command/endpoint -> services -> sink          <- most often this one
LAYERS       who is responsible for what, what each layer does not know
METHOD       input -> branches -> output, line by line
```

The same subject can be redrawn at another level without starting over.

---

## Step 4. Trace

These rules are mandatory:

1. **Do not invent connections.** Every node and every arrow must follow from real code: a method call, constructor injection, `dispatch`, an HTTP request, a database query, an event. Either verify a suspected connection or mark it explicitly as a hypothesis.
2. **Stop at external service boundaries.** A third-party API, an external site, someone else's database is a terminal node — do not go inside.
3. **On every significant node** — path and line (`app/Services/Baxi/CatalogDiffer.php:88`), what comes in and what goes out.
4. **Show branching explicitly.** A condition is a decision node; every branch carries a label ("yes" / "no" / "mismatch").

Tracing details for PHP/Laravel — [references/TRACING.md](references/TRACING.md).

---

## Step 5. Build the document

Start from [templates/skeleton.html](templates/skeleton.html) — it carries the CSS variables, theming, print rules and the full class vocabulary.

**Hard requirements for the output:**

- zero `<script>` tags;
- zero external URLs in `src`/`href` for resources (links to live sources in prose are fine and welcome);
- no Mermaid, no CDN;
- flow layout built from `div` elements, never hand-computed coordinates; `<svg>` only where CSS cannot express the geometry (crossing edges, cycles, complex relations);
- light and dark themes: `prefers-color-scheme` plus `data-theme="dark"` and `data-theme="light"`;
- a sticky `nav.toc` from three sections upward;
- set the document `lang` attribute to the project's language — the prose follows the language of the project and the conversation, not this file.

Class vocabulary and markup examples — [references/VOCABULARY.md](references/VOCABULARY.md).

### Current and planned on one diagram

When drawing changes, show both layers at once and explain the notation in `.legend`:

```
.now  .fut          now / will become
.box.add            being added
.box.dead           being removed
.box.hot            hot spot, bottleneck
.call.new           new call
.call.warn .bad     questionable / broken
.step.newp          new step
.step.danger .exit  dangerous step / exit
```

---

## Step 6. Where it goes

By default `docs/` at the project root, named after the subject: `docs/baxi.html`, `docs/payments-sync-flow.html`.

A path named by the user always wins. If the document is by its role a plan or a research note, it belongs next to plans (`plans/`, `research/`) rather than in `docs/`.

When splitting into several files — one directory plus `index.html` with a table of contents and links.

---

## Step 7. Export (on request)

`--pdf` and `--md` only when asked. Commands and print requirements — [references/EXPORT.md](references/EXPORT.md).

This skill deliberately **does not pre-approve shell access**: rendering a PDF launches a browser, and that should be confirmed each time. Markdown is written with ordinary Write and needs no confirmation.

---

## Step 8. Report

Do not open the file automatically (`open` does not exist everywhere — worktrees, ssh). Print the path as a link:

```
Done: file:///absolute/path/docs/baxi.html

Scale:     subsystem
Input:     product_catalog.xml, service.baxi.ru
Output:    backend-app service API
Sections:  overview, layers, stage A, stage B, caching
```

If something could not be traced in the code, say so plainly and name the place.

---

## Boundaries: this is not charting

A block diagram is not a data chart. If the document needs a **real chart** (bars, lines, shares, distribution over time, a heatmap, a KPI tile), use the `dataviz` skill for it — it owns chart form, palette and validation.

```
flow, branching, layers, input->output   -> this skill
bars, lines, shares, heatmap, KPI        -> dataviz
```

Do not substitute one for the other: a flow drawn as a chart loses causality; a chart drawn as blocks loses magnitude.

### The palette is validated

The template's accent colors were run through the `dataviz` validator (`scripts/validate_palette.js`).

- Light theme — all checks pass: `--a #B8402C`, `--b #35619B`, `--ok #34794B`, `--warn #A9761F`.
- Dark theme — the `--warn` ↔ `--ok` pair scores ΔE 11.8 against a floor of 15. A known, deliberate trade-off: they cannot be separated further without leaving the muted document palette — the green is squeezed between the blue and the gold. It holds because of secondary encoding: **every node is labeled with text**, and a decision node is also distinguished by its role in the diagram.

Hence the rule: **a node's color is a hint, not the carrier of meaning.** Meaning lives in the text inside the node. A diagram that can only be read by color is built wrong.

---

## AI Factory integration

If the project has `.ai-factory/`:

- read `config.yaml` — it provides the language (`language.ui`, `language.artifacts`) and the paths;
- read `DESCRIPTION.md`, `ARCHITECTURE.md`, `RESEARCH.md` and the active plan as context;
- **but remember: when they disagree with the code, the code wins.**

The skill works without AI Factory too — then the document language follows the project and the conversation.

---

## Artifact ownership

- **Owns:** the HTML document itself, and on explicit request its companions (`.pdf`, `.md`), plus `index.html` when splitting.
- **Read-only:** the code, `.ai-factory/*`, plans, `README.md`, everything else in the repository.
- The skill never edits code and never touches other tools' artifacts.

---

## What not to do

- Do not rewrite an existing document wholesale for one edit.
- Do not invent a new style when a document already exists — follow its markup.
- Do not draw what you did not find in the code when code is the source of truth.
- Do not pull in Mermaid, a CDN, remote fonts or `<script>`.
- Do not hand-compute SVG coordinates where flow layout would do.
- Do not open the file automatically.
- Do not publish through Artifact.
