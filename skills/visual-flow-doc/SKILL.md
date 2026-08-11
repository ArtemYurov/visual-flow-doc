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
  version: "1.6"
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

## Step 4. Trace — first pass

**First establish the ecosystem** — read the manifest (`composer.json`, `package.json`, `pyproject.toml`, `go.mod`, …) and open the matching guide under [references/tracing/](references/tracing/) if one exists. Which connections the import graph misses depends entirely on the ecosystem, and that is what makes traces wrong.

The first pass follows the trunk: what happens when everything goes well. These rules are mandatory:

1. **Do not invent connections.** Every node and every arrow must follow from real code: a method call, constructor injection, `dispatch`, an HTTP request, a database query, an event. Either verify a suspected connection or mark it explicitly as a hypothesis.
2. **Stop at external service boundaries.** A third-party API, an external site, someone else's database is a terminal node — do not go inside.
3. **On every significant node** — path and line (`app/Services/Baxi/CatalogDiffer.php:88`), what comes in and what goes out.
4. **Show branching explicitly.** A condition is a decision node; every branch carries a label ("yes" / "no" / "mismatch").

General tracing method — [references/TRACING.md](references/TRACING.md). Per-ecosystem detail — [references/tracing/](references/tracing/): [php-laravel](references/tracing/php-laravel.md), [node-typescript](references/tracing/node-typescript.md).

---

## Step 5. Second pass — sweep for depth

**Mandatory. Do not skip it, and do not merge it into the first pass.**

The first pass answers "what happens when everything works". It structurally misses everything else. Go back into the code you already read and hunt specifically for the following. Each item is a question with a yes/no answer — if the answer is yes, the diagram or the prose is missing something.

| what to hunt | why the first pass misses it |
|---|---|
| **Early exits** — `return`, `continue`, `throw` inside the methods you called | they live one level below the call you drew as a single node |
| **`catch` blocks** — what happens to the data on failure | the happy path never enters them |
| **How many distinct outcomes share one exit code** — silent data loss, user cancellation, an interrupt signal and real success may all return the same code | from the outside they are indistinguishable, and a caller cannot react to what it cannot tell apart |
| **Cascades** — one skip that removes a whole branch downstream | visible only when you ask "what depends on this id / this cache entry?" |
| **Caching** — every cache turns one path into two: hit and miss | the first pass usually draws only the miss |
| **Flags and config** that change behavior, not just values | they are read far from where they take effect |
| **Non-boolean states** — a field with three or more meanings (`true` / `false` / absent) | the first pass reads it as a flag |
| **Injected but never called** — a dependency in the constructor that this flow never uses | it looks like part of the flow because it is in the signature |
| **Second run** — overwrite, skip, duplicates, no request at all | idempotency cannot be seen by reading a single run |

### Which black boxes to open

**The subject of the document is the algorithm and its branches — not implementation detail.** A node you drew as one box is either a transformation or a decision point, and only the second kind is worth opening.

| the box | leave it closed? |
|---|---|
| **transforms** data — mapper, formatter, serializer, parser internals | **yes** — its input and output are the whole story |
| **decides the fate** of a record — matcher, differ, reconciler, planner, scheduler | **no — open it**: the branches inside are the content of the diagram |

Stop rule: descend while you keep finding decisions, stop at the level where only transformation remains. Never descend for detail's sake — how a string is split is not part of the flow; *which of two keys is used to match a record* is.

Two rules for what you find:

1. **A missed branch is a defect of the diagram, not a detail.** Add the node, do not add a sentence.
2. **What you deliberately left as a black box must be named** — see the completeness checklist in Step 9.

If the second pass finds nothing at all, that is a signal you did not actually run it — a subsystem of any size always has at least a cache branch or an error path.

---

## Step 5b. Third sweep — properties of the whole

The two passes above walk *a* flow. Some of the most useful facts about a codebase are not on any flow — they are visible only when you look at the whole of it. Collect these before writing:

| what to establish | how to get it |
|---|---|
| **The purpose, and what this is *not*** | the README, the package description, the commands' own descriptions. State the job in one sentence, then name two or three neighbouring jobs it deliberately does **not** do — the negations rule out false expectations faster than any description |
| **Whose scenarios these are** | quote them from the user's side: *"I want my local copy to match production"*. A diagram answers *how*; a scenario answers *why anyone runs it* |
| **Destructiveness** | for anything that writes: what is irreversibly lost if it goes wrong. Rank the entry points by it and say what exactly does the damage (`DROP SCHEMA … CASCADE`) |
| **Declared but unused** | grep every DTO, enum, exception and interface for uses outside its own file. "Imported but never applied", "zero throws" are facts about the design, not trivia |
| **The same thing three times** | one behaviour implemented separately in several places — confirmation prompts, argument parsing, adapter resolution. Name the copies and the drift between them |
| **How dependencies are assembled** | container bindings or bare `new`? State the consequence: what can and cannot be substituted in tests |
| **Intent versus implementation** | a docblock, README or config promising something the code does not do. "The mechanism exists but is effectively a stub" is a finding |

Not all seven apply to every subject — but each is worth one question. Findings go into the document as prose and tables, not as diagram nodes: they describe the system, not its flow.

---

## Step 6. Build the document

Start from [templates/skeleton.html](templates/skeleton.html) — it carries the CSS variables, theming, print rules and the full class vocabulary.

**Hard requirements for the output:**

- zero `<script>` tags;
- zero external URLs in `src`/`href` for resources (links to live sources in prose are fine and welcome);
- no Mermaid, no CDN;
- flow layout built from `div` elements, never hand-computed coordinates; `<svg>` only where CSS cannot express the geometry (crossing edges, cycles, complex relations);
- light and dark themes: `prefers-color-scheme` plus `data-theme="dark"` and `data-theme="light"`;
- a sticky `nav.toc` from three sections upward;
- **the first section states the purpose** — what the subject is for, what it deliberately is not, and whose scenarios it serves. A diagram without that hangs in the air;
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

## Step 6b. Distil — cut without losing a fact

**Mandatory. Run it on the finished draft, before reporting.**

A diagram document earns its keep by density. Every sentence must carry a fact that is **not already in a diagram, a table or a code reference**. Prose that restates the picture is worse than no prose: it costs reading time and teaches the reader to skim past the parts that do matter.

Go through the draft and apply the drop test to every sentence: **remove it — is any fact lost?** If nothing is lost, it stays removed.

### Cut on sight

| cut | replace with |
|---|---|
| meta-text — "this section describes…", "as the diagram shows…", "it is worth noting that…" | the fact itself |
| a description of a property | the property: not "the cache speeds things up" but "TTL 86400 s; on a miss it goes to the network" |
| adjectives where a number exists | the number: not "a large file" but "916 lines" |
| hedging — "generally", "in some cases", "may sometimes" | either the condition that makes it true, or an explicit "not verified" |
| a caption retelling the nodes above it | nothing, or the one thing the diagram cannot show — why it is built that way |
| a recap section at the end | nothing; the reader has just read it |
| three synonyms for one idea | the sharpest one |
| a paragraph longer than four lines with parallel items | a table or a list |

### Keep

Causes, consequences and constraints — anything answering **why it is like this** and **what breaks otherwise**. A diagram shows what happens; it cannot show that partial writes are possible because the sink is someone else's API. That sentence stays.

### Shape

- one thought per sentence;
- lists ranked, not enumerated — if a list passes five items, split it into "matters now" and "the rest";
- a table whenever three or more items share a structure;
- no preamble before a section and no summary after it.

The target is not a shorter document. It is the same facts in fewer words — and a reader who can trust that every line was worth printing.

---

## Step 7. Where it goes

By default `docs/` at the project root, named after the subject: `docs/baxi.html`, `docs/payments-sync-flow.html`.

A path named by the user always wins. If the document is by its role a plan or a research note, it belongs next to plans (`plans/`, `research/`) rather than in `docs/`.

When splitting into several files — one directory plus `index.html` with a table of contents and links.

---

## Step 8. Export (on request)

`--pdf` and `--md` only when asked. Commands and print requirements — [references/EXPORT.md](references/EXPORT.md).

This skill deliberately **does not pre-approve shell access**: rendering a PDF launches a browser, and that should be confirmed each time. Markdown is written with ordinary Write and needs no confirmation.

---

## Step 9. Report

Do not open the file automatically (`open` does not exist everywhere — worktrees, ssh). Print the path as a link:

```
Done: file:///absolute/path/docs/baxi.html

Scale:     subsystem
Input:     product_catalog.xml, service.baxi.ru
Output:    backend-app service API
Sections:  overview, layers, stage A, stage B, caching
```

### Completeness checklist — run it before reporting

- [ ] input and output are named explicitly, at the chosen scale
- [ ] the second pass actually ran, and what it found is **in the diagram**, not only in prose
- [ ] every cache, flag and early exit that changes the outcome is a branch
- [ ] at least one reference table: files, config keys, or command flags
- [ ] the purpose is stated, including what the subject is *not*
- [ ] anything that writes carries a destructiveness note — what is irreversibly lost
- [ ] declared-but-unused types were checked systematically, not incidentally
- [ ] a section naming **what is left as a black box** and what the document does not cover
- [ ] more than one diagram, or more than one table, in a section → each gets its own `h3` with a meaningful name (subheadings reach the table of contents and are what make the document navigable months later)
- [ ] whatever could not be traced is said out loud, with the place named
- [ ] the distillation pass ran: no sentence survives that restates a diagram, and no adjective survives where a number was available

The black-box section is not an apology — it is what keeps the reader from believing the document is more complete than it is.

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
