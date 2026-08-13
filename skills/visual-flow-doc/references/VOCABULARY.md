# Markup vocabulary

Diagrams are assembled from `div` elements with classes. The browser lays them out — no coordinates are computed, and nothing needs recalculating when you edit.

All CSS for these classes already lives in [../templates/skeleton.html](../templates/skeleton.html).

---

## Flow nodes

| class | meaning | appearance |
|---|---|---|
| `node` | an ordinary step | bordered rounded rectangle |
| `node io` | **input or output** | dashed border |
| `node dec` | decision, branching | `--warn` color |
| `node ext` | external service (terminal) | `--b` color |
| `node db` | storage, sink | rounded "barrel", thick top |
| `node ok` | successful outcome | `--ok` color |
| `node stop` | halt, rejection | `--a` color |
| `node acc-a`, `node acc-b` | belongs to stage A or B | colored bar on the left |

Inside a node: `<b>` is the title, `<small>` is the gray explanation.

```html
<div class="flow-wrap"><div class="flow">
  <div class="node io"><b>product_catalog.xml</b><small>models, specs, links</small></div>
  <div class="arr"></div>
  <div class="node acc-a"><b>Stage A · baxi:boilers</b><small>diff / upsert</small></div>
  <div class="arr ext"><span>HTTP (Saloon, JWT)</span></div>
  <div class="node db"><b>Service API</b><small>series → boilers → versions</small></div>
</div></div>
```

---

## Structure

| class | purpose |
|---|---|
| `flow-wrap` | outer frame of a diagram, provides horizontal scrolling |
| `flow` | a column of nodes, centered |
| `branch` | several `flow` columns side by side — parallel paths |
| `arr` | an arrow between nodes — the line is drawn in CSS, **put no `│` or `▼` in the markup**. A label goes inside an element (`<b>` or `<span>`), never as bare text, so it gets the backing that keeps the line from running through it |
| `arr dashed` | conditional or optional transition |
| `arr ext` | the arrow leaves the subsystem — same colour as `node ext` |
| `arr stop` | the arrow ends in a halt or refusal |
| `arr tall` | taller gap when a label needs the room |
| `callout` | an aside in the margin: italic serif, dashed rule. Signals "explanation", not "flow". **Max two per diagram**, and never for something the diagram should label directly |
| `lane-note` | a small caption under a branch |
| `grid2` | a card grid that adapts to width on its own |
| `card` | a card with an `h4` heading |
| `tbl-wrap` | a scrollable table wrapper |

A labeled branch:

```html
<div class="branch">
  <div class="flow">
    <div class="arr"><b>yes</b></div>
    <div class="node ok"><b>Write the version</b></div>
  </div>
  <div class="flow">
    <div class="arr"><b>no</b></div>
    <div class="node stop"><b>Into the orphan report</b></div>
  </div>
</div>
```

---

## Layers of responsibility

| class | purpose |
|---|---|
| `layers` | a column of layers |
| `layer l1`..`l4` | a layer, colored bar on the left by number |
| `layer cross` | a cross-cutting layer (dashed) |
| `layer-arr` | an arrow between layers |
| `.num` | layer number and name in caps |
| `.path` | directory path in gray |
| `.nope` | "does not know about…" — in red |

```html
<div class="layers">
  <div class="layer l1">
    <span class="num">1 · SCENARIO</span><span class="path">app/Console/Commands/</span>
    <small>Order of steps, flags, progress.<br>
    <span class="nope">Does not know:</span> how the XML is shaped, how to call the API.</small>
  </div>
  <div class="layer-arr"></div>
  <div class="layer l2">…</div>
</div>
```

The rule of layers: **each layer knows only the layer beneath it.** Data rises, decisions descend. Always state what a layer does **not** know — that is where responsibility ends.

---

## Current and planned

For documents about change — both states on one diagram, plus a `legend` explaining the notation.

| class | meaning |
|---|---|
| `now` / `fut` | a "now" / "will become" block |
| `box add` | being added |
| `box dead` | being removed, dying |
| `box hot` | hot spot, bottleneck |
| `call new` | a new call |
| `call warn` / `call bad` | questionable / broken |
| `step newp` | a new process step |
| `step danger` / `step exit` | dangerous step / exit |
| `step mut` | muted, secondary |
| `legend` | notation key — **mandatory** if any modifier is used |

---

## Header and asides

Used in every document, so worth stating the rules rather than copying by example.

| class | purpose |
|---|---|
| `hero` | the header block: `h1`, a one-sentence `p.sub`, then `cmd-row` |
| `cmd-row` | the row of commands that actually run this thing |
| `cmd` | one command plate; put the command in `<code>` inside it |
| `cmd > .dot` + `dot-a` / `dot-b` | a coloured dot on the plate. **The colour must match the stage the command belongs to** — a `dot-a` plate next to a `stage-a` section, otherwise the colour is decoration and teaches the reader nothing |
| `footnote` | a small grey aside under a diagram or table — for the caveat that would break the flow of the paragraph above |
| `page-break` | manual page break for PDF output, see [EXPORT.md](EXPORT.md) |

`footnote` and `callout` are not interchangeable: a footnote qualifies what was just said, a callout points at one element of a diagram. If the text applies to the whole section, it is neither — it is a sentence in the body.

---

## Chips and accents

`chip` with one of `chip-ok`, `chip-warn`, `chip-mut`, `chip-a`, `chip-b` — small pill labels for statuses in headings and tables.

`h2 > span.stage` with `stage-a` / `stage-b` — a stage marker next to a section heading.

---

## When SVG is needed

A `div` flow will not do when you need:

- crossing edges, arrows that route around;
- cyclic dependencies;
- a diagram on a coordinate grid (component placement on a board, a topology).

Then use inline `<svg viewBox>` with `role="img"` and `aria-label`. Always a `viewBox` and no fixed `width`/`height`, so it scales. Take colors from `currentColor` or CSS variables, otherwise dark mode breaks.

---

## Document sections

The order that emerged in practice:

1. `header.hero` — title, a one-sentence subtitle, `cmd-row` with the commands that run it
2. `nav.toc` — sticky table of contents (from three sections up)
3. **Purpose** — what this is for, what it deliberately is **not**, whose scenarios it serves, and — for anything that writes — a destructiveness column ranking the entry points by what they can irreversibly destroy
4. **Overview** — the whole picture: all inputs → transformation → output
5. **Layers of responsibility** — for subsystem scale and above
6. One section per stage, each with its own diagram
7. **Properties of the whole** — declared but unused, duplicated behaviour, how dependencies are assembled, intent versus implementation
8. Reference — tables: files, config, command flags
9. **Next** — what is missing, what is planned

Do not force all nine into a small document. Purpose, an overview and at least one diagram is the minimum — the purpose section is never the one to drop.
