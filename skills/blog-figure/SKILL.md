---
name: blog-figure
description: Draw a diagram for a Laminar blog post as an SVG in the house style (rounded charcoal panels, General Sans, one spacing grid, color only on tracked IDs). Use when asked to make, add, or redraw a figure, diagram, or illustration for a blog article.
---

# Blog figure

A figure explains one mechanism in a post: what is stored, where a request goes, what changes. The style follows laminar.sh's landing cards: soft, dark, lots of air, very little ink. Shipped figures live in lmnr-blog-renderer's `public/figures/`. When you can, look at a few before drawing: your figure must look like it belongs next to them.

## Look

- **Flat and dark.** Charcoal panels with rounded corners on the page's dark background. No borders, shadows, gradients, icons, or emoji.
- **Less is the style.** Show only what the figure proves. Cut columns, rows, notes, and tags the prose already covers. A panel with three rows beats one with six. Leave generous space inside panels.
- **Text does the work.** Short labels and real identifiers from the post (`trace_id`, `signal_events`, `c7`). No line longer than a short phrase.
- **Color marks identity, and nothing else.** Most of a figure is grey. Color goes only on the IDs the post tracks across figures (the join keys, the new fields), and at most one accent per row. Field names stay grey. Don't tint whole rows to highlight them. To dim something, like a row nothing points at, use `textDim`.
- **One idea per figure,** read left to right or top to bottom. At most three panels across.
- **Titles are written as the thing is named.** Tables and identifiers are lowercase (`traces`, `signal_events`, `clusters_dict`). Names keep their spelling (`ClickHouse`, `Postgres`, `Coding agent`).

<!-- generated:tokens (generated in lmnr-blog-renderer by `pnpm figure-kit` from components/blog/query-flow/layout.ts; copy it over when that changes) -->
## Colors

| Name | Hex | Use |
| --- | --- | --- |
| `stroke` | `#484848` | Connectors, stations, tree elbows, and resent (not new) blocks. |
| `text` | `#b8b8b8` | Body text, values, captions. |
| `textDim` | `#777777` | Field names, notes, anything that should step back. |
| `title` | `#ffffff` | Panel titles, step labels, and lane labels. |
| `body` | `#212121` | Panel bodies and pill-shaped operations. |
| `pill` | `#2c2c2c` | Anything set on a panel or beside it: column-name bands, cards, resting chips, the join badge. |
| `glyph` | `#858585` | The symbol inside a join badge. |
| `ch` | `#ecbb4f` | Trace IDs and ClickHouse. |
| `pg` | `#6aa4f5` | Postgres and the names it holds. |
| `accent` | `#c66945` | Event IDs and the coding agent. |
| `engine` | `#1f7bd8` | The query engine. |
| `cluster` | `#ef6aac` | Cluster IDs. |
| `pulse` | `#ec7b4f` | The dash that travels along a wire; brighter than `accent` so it is not read as an event. |

Identity colors (`ch`, `pg`, `accent`, `engine`, `cluster`) mean the same thing in every figure: reuse them for the same concepts, and keep everything else in the greys. A new concept that needs its own color is a question for the author, not a new hex. `pulse` is only for animated dashes.

## Text

One face, General Sans, via `font-family:var(--figure-font)`; letter-spacing `0.2px` on everything. Sizes are viewBox units (px at 720 wide).

| Role | Size | Weight | Color | Use |
| --- | --- | --- | --- | --- |
| `panelTitle` | 16 | 500 | `title` | Panel title, inside the panel's top-left |
| `text` | 15 | 460 | `text` | Values, table cells, entry names, station labels, step captions |
| `columnHeader` | 13 | 460 | `textDim` | Column names above a table's rows; small right-aligned tags (`new`) |
| `note` | 13 | 460 | `textDim` | Wire labels, notes beside a row, axis labels, legends; the smallest size |

Letter-spacing `0.2px` on every piece of text. No bold beyond these weights, no italics, no all caps, no underlines. Anchor with `text-anchor` (`start`, `middle`, `end`), never by guessing widths.

## Spacing

Every panel sits on one vertical grid, measured from the panel's top edge. Use these numbers; don't eyeball.

| Token | Value | Rule |
| --- | --- | --- |
| `PAD` | 16 | Text inset from a panel's left and right edges; also the gap from its top edge to the title's caps |
| `TITLE_DY` | 27 | Title baseline, from the panel's top |
| `ROW_STEP` | 32 | One row. Rows sit on this pitch below the title |
| `FIRST_ROW_DY` | 59 | First row's baseline (`TITLE_DY + ROW_STEP`) |
| `COLUMN_DY` | 59 | A table's column names, on the first row |
| `TABLE_FIRST_ROW_DY` | 91 | A table's first data row, one row under its column names |
| `LINE_STEP` | 24 | Lines of one block of text (SQL, a two-line card), tighter than rows |
| `BOTTOM_PAD` | 20 | Under a panel's last baseline to its bottom edge |
| `LINE_BOX_H` | 44 | Height of any one-line box (an entry point, a pill); also a panel's title block |
| `GAP` | 16 | Between panels with nothing in the gap; between stacked cards |
| `WIRE_GAP` | 32 | Minimum gap a wire or arrow crosses |
| `INSET` | 8 | A card, band, or tint inside a panel, from the panel's edges |
| `CAPTION_DY` | 32 | Numbered caption baseline, below the panels' bottom edge |
| `RADIUS` | 6 | Panel corners |
| `RADIUS_SM` | 4 | Corners of anything nested in a panel, and of chips |
| `WIRE_WIDTH` | 1.5 | Connector stroke, round caps and joins |
| `panel height` | 59 + (rows − 1) × 32 + 20 | A panel of `rows` rows; a table starts from 91 instead of 59 |

Canvas: 720 wide, 16 gutter each side, so content spans x 16–704.
<!-- /generated:tokens -->

## Shapes

Coordinates are relative to the shape's top-left `(x, y)`, width `w`. Colors and tokens are the names in the tables above. The complete example at the end of this section shows the exact markup.

**Panel**, the unit everything is built from. The title sits inside the body; there is no title strip:
```svg
<rect x y width=w height=h rx=RADIUS fill=body/>
<circle cx=x+PAD+3.5 cy=y+TITLE_DY-5 r=3.5 fill=<identity>/>     <!-- only if it has an identity color -->
<text x=x+PAD+14 y=y+TITLE_DY fill=title style=panelTitle>traces</text>   <!-- x+PAD without a dot -->
```
Rows follow at `y+FIRST_ROW_DY`, then every `ROW_STEP`. Height = `panel height` from the spacing table.

**Key/value rows**: the key at `x+PAD` in `textDim`, and the value at one shared x in `text` or its identity color. Line up the value column across panels that show the same rows.

**Table**: column names (`columnHeader`) on a band at `y+COLUMN_DY`, then data from `y+TABLE_FIRST_ROW_DY`:
```svg
<rect x=x+INSET y=y+COLUMN_DY-17 width=w-2·INSET height=24 rx=RADIUS_SM fill=pill/>
```
Only the join keys are colored. A row nothing points at gets its ID in `textDim`.

**Group tint**, when a few rows form one thing (a tree, an expanded list): one block behind the group, not one per row. `<rect x=x+INSET y=firstBaseline-21 width=w-2·INSET height=lastBaseline-firstBaseline+32 rx=RADIUS_SM fill=<identity> fill-opacity=0.07/>`.

**Right-aligned tag** on a row or on the title line (`new`, `per trace`, `12 hashes`): `small` at `x+w-PAD`, `text-anchor=end`, in `textDim`, or in the row's identity color when it marks that row.

**Card**, a sub-block inside a panel (an agent's steps, stored messages): `<rect x=x+INSET width=w-2·INSET rx=RADIUS_SM fill=pill/>`. Cards sit `GAP` apart, and the first starts `LINE_BOX_H` below the panel top. A card is laid out like a panel: label on `TITLE_DY`, data from `FIRST_ROW_DY`, then `LINE_STEP` per extra line, then `BOTTOM_PAD`. So a one-line card and a one-row panel are the same height. Cards can also stand alone on the page (see `pipeline.svg`); there they are two `small` lines, the role in its color at y+19 and the content in `textDim` at y+37, 46 high.

**Entry box**, a one-line untitled source (a client, a surface): `LINE_BOX_H` high, text at `x+PAD`, baseline `y+h/2+5`. Stack them `GAP` apart.

**Chip**, a short ID or hash as a token: 24–30 high, `rx=RADIUS_SM`, `small` text centered. New or tracked: `fill=<identity> fill-opacity=0.12–0.14` with the text in that color. Repeated or background: `fill=pill` with `textDim` text.

**Pill**, an operation or a decision (canonicalize, lookup, "new to trace?"): `rx=h/2`, `fill=body`, `LINE_BOX_H` high (64 for two lines), with the text centered. Pills are steps; panels and cards are data. Don't use circles or diamonds.

**Wire**: `stroke=stroke stroke-width=WIRE_WIDTH stroke-linecap=round stroke-linejoin=round fill=none`. Straight when both ends are level. Otherwise draw one S-curve that leaves and arrives level: from `a` to `b`, `C a.x+0.34·dx a.y, a.x+0.57·dx b.y, b.x b.y`. Fanning sources each get their own curve. Draw wires before panels and end them at panel edges. A gap a wire crosses is at least `WIRE_GAP`.

**Arrowhead**, an open chevron at the tip `(tx, ty)` pointing right: `M tx-5 ty-4 L tx ty L tx-5 ty+4`, same stroke. Rotate it for other directions. Stop the line 1–2 short of the panel.

**Request and reply**: the request is solid, with its label (`small`, centered) 10 above the wire. The reply is dashed (`stroke-dasharray="3 4"`), 14 below, unlabelled, pointing back.

**Station**, a stage a wire passes through inside a panel: `<rect width=6 height=6 rx=1 fill=stroke/>` centered on the wire, with its label (`text`, centered) one row below.

**Join**: a 30×30 `pill` square with `rx≈7`, centered in the gap on the wire, holding the ⋈ glyph `<path d="M10 13V31L34 13V31L10 13Z" fill=none stroke=glyph stroke-width=2.5 stroke-linejoin=round/>` scaled by 30/44.

**Tree** (parent → child): indent children 18. Elbow `M px+6 py+6 V cy-5 H cx-2` in `stroke`, width 1, round caps.

**Step caption**, numbered order under a row of panels: `<text>` with `<tspan fill=textDim>1.</tspan><tspan dx=8>Write one ID per trace</tspan>` in `text` size, at the panel's left edge, baseline `CAPTION_DY` below the panels.

**Blocks and bars**: 24×12 rects with `rx=2`, 3 apart, bottom-aligned on a shared baseline so their areas compare. New blocks in `ch`, repeated ones in `stroke`. Axis labels are `small`/`textDim`, centered under each column.

**Legend**, only when color alone carries meaning: a 10×10 `rx=2` square and a `small`/`textDim` label, below the panels, left-aligned. Center the square on the lowercase letters: square `y = baseline − 8.5`, label `x = square + 18`.

**Lane label** (`Write`, `Read`), when one figure has two lanes: `text` in `title`, at x 16, above the lane.

**Colored runs in one line**: `<tspan fill=…>` inside the `<text>`, e.g. `<tspan fill=pg>“Failure Detector”</tspan>: <tspan fill=ch>28 events</tspan>`.

**Complete example**, one table panel, centered. It shows the exact markup every figure uses (inline styles, no `<style>`, colors as hex). Copy it and change it:
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="16 50 688 175" aria-label="The signal_events table: each event points at a trace and a cluster.">
<rect x="216" y="50" width="288" height="175" rx="6" fill="#212121"/>
<circle cx="235.5" cy="71.7" r="3.5" fill="#c66945"/>
<text x="246" y="77" fill="#ffffff" style="font-family:var(--figure-font);font-weight:500;font-size:16px;letter-spacing:0.2px">signal_events</text>
<rect x="224" y="92" width="272" height="24" rx="4" fill="#2c2c2c"/>
<text x="232" y="109" fill="#777777" style="font-family:var(--figure-font);font-weight:460;font-size:13px;letter-spacing:0.2px;white-space:pre">id</text>
<text x="312" y="109" fill="#777777" style="font-family:var(--figure-font);font-weight:460;font-size:13px;letter-spacing:0.2px;white-space:pre">trace_id</text>
<text x="396" y="109" fill="#777777" style="font-family:var(--figure-font);font-weight:460;font-size:13px;letter-spacing:0.2px;white-space:pre">cluster_id</text>
<text x="232" y="141" fill="#b8b8b8" style="font-family:var(--figure-font);font-weight:460;font-size:15px;letter-spacing:0.2px;white-space:pre">ev_91</text>
<text x="312" y="141" fill="#ecbb4f" style="font-family:var(--figure-font);font-weight:460;font-size:15px;letter-spacing:0.2px;white-space:pre">t1</text>
<text x="396" y="141" fill="#ef6aac" style="font-family:var(--figure-font);font-weight:460;font-size:15px;letter-spacing:0.2px;white-space:pre">c4</text>
<text x="232" y="173" fill="#b8b8b8" style="font-family:var(--figure-font);font-weight:460;font-size:15px;letter-spacing:0.2px;white-space:pre">ev_92</text>
<text x="312" y="173" fill="#ecbb4f" style="font-family:var(--figure-font);font-weight:460;font-size:15px;letter-spacing:0.2px;white-space:pre">t1</text>
<text x="396" y="173" fill="#ef6aac" style="font-family:var(--figure-font);font-weight:460;font-size:15px;letter-spacing:0.2px;white-space:pre">c7</text>
<text x="232" y="205" fill="#b8b8b8" style="font-family:var(--figure-font);font-weight:460;font-size:15px;letter-spacing:0.2px;white-space:pre">ev_93</text>
<text x="312" y="205" fill="#ecbb4f" style="font-family:var(--figure-font);font-weight:460;font-size:15px;letter-spacing:0.2px;white-space:pre">t2</text>
<text x="396" y="205" fill="#ef6aac" style="font-family:var(--figure-font);font-weight:460;font-size:15px;letter-spacing:0.2px;white-space:pre">c7</text>
</svg>
```

## Layout

- **Canvas.** 720 wide, drawing inside x 16–704, with `viewBox="16 <top> 688 <height>"` cropped to the content, nothing empty above or below. Content starts at y 50. It renders about 612px wide (×0.89) in the article, smaller on phones.
- **Rows of panels share one top and one height** (the tallest one's). Widths plus gaps add up exactly to 688. Size each panel to its longest line and give the extra to the panel with the longest text. A single panel can be narrower and centered.
- **Align across panels.** The same data sits on the same baselines, captions share one baseline, and stores line up with the step they answer.
- **Vertical rhythm comes only from the spacing table.** Rows are `ROW_STEP` apart and lines within a block `LINE_STEP` apart. Everything else is `PAD`, `INSET`, `GAP`, or `BOTTOM_PAD`. If you need a number that isn't there, the layout is wrong. Change the layout, don't invent the number.

## Fit (where figures break)

Budget every line before placing it. Widths per character, tracking included (measured in General Sans; IDs and hashes come out the same):

| Role | Units per character |
| --- | --- |
| `text` (15) | ~7.5 |
| `small` (13) | ~6.5 |
| `panelTitle` (16) | ~8 |

For example, `Search timeouts` (15 chars, `text`) needs ~113, plus `PAD` on each side.

Check the tightest spots: the longest value in each column, a right-aligned tag meeting a left column, step captions (panel width + gap), labels over short wires, text inside chips. If it doesn't fit:
1. Cut words.
2. Widen the panel.
3. Drop that text to `small`, which is the smallest size there is.

## Write, finish, check

**In lmnr-blog-renderer, prefer the JSX route.** The scene imports the real tokens (`components/blog/query-flow/layout.ts`) and building blocks (`primitives.tsx`: `Node`, `Text`, `ColumnHeaders`, `GroupTint`, `Step`), so it can't drift from the style.
1. Write a scene `components/blog/<post-figures>/scenes/<name>.tsx` that returns a `<g>`. Compute every coordinate from the tokens, as the existing scenes do, and export the scene's bottom edge for the viewBox crop.
2. Register it in that folder's `figures.tsx`: add it to `SCENES`, put its alt text in `LABELS`, and give its `{ top, bottom }` in `BOUNDS`. A new post gets a new folder and an entry in `scripts/export-figures.tsx` and `app/figures/page.tsx`.
3. Look at it live on `/figures?only=<name>` (`pnpm dev`). `?width=360` shows the phone width, and `?headers=plain` hides the column bands.
4. `pnpm export-figures` writes `public/figures/<post>-<name>.svg`, with the font embedded. A hand-written SVG there goes through `pnpm finish-figure figures-src/<name>.svg` instead.
5. If you changed `layout.ts`, run `pnpm figure-kit` and copy the regenerated token tables into this skill.

**Anywhere else, hand-write the SVG:**
1. Start from the complete example above. Keep `<svg xmlns="http://www.w3.org/2000/svg" viewBox="…" aria-label="<alt text>">`. Every `<text>` carries its style inline, exactly as in the example. No `<style>`, fonts, images, or external references while drawing.
2. **Finish it** before it ships, because an SVG shown through `<img>` can't use the page's fonts. Insert these right after the opening `<svg>` tag:
   - `<title>` holding the `aria-label` text.
   - A `<style>` containing:
     - `@font-face{font-family:"Figure Sans";font-weight:200 700;src:url(data:font/woff2;base64,<…>) format("woff2")}`, with the General Sans variable woff2 (`app/fonts/general/GeneralSans-Variable.woff2` in lmnr-blog-renderer, or the free download from fontshare.com) base64-encoded.
     - `svg{--figure-font:"Figure Sans",ui-sans-serif,sans-serif;color-scheme:dark}`. Without the color scheme, an `<object>` paints a white canvas.
     - `svg{-webkit-font-smoothing:antialiased;-moz-osx-font-smoothing:grayscale}`. Without it, text renders heavier on macOS.
   - Also add `width` and `height` (the viewBox's width and height) and `role="img"` to the `<svg>` tag.

**Either way, check it.** Load the finished file in headless Chrome at 612px wide (the article column) and 360px (a phone), and look at both screenshots. For example: `chrome --headless=new --window-size=612,<h> --screenshot=out.png page.html`, where `page.html` is `<body style="margin:0;background:#1a1a1a"><img src="<file>" style="width:612px">`. Fix anything clipped, touching, overlapping, off the grid, or misaligned, and repeat until the 612px render is clean. At 360px, text only has to stay legible; if it doesn't, say so and offer a stacked `<name>-compact.svg`.

The `aria-label` says what the figure shows in one or two sentences; it is the alt text. Report the file path and the embed line. Don't claim the figure is clean without having looked at the screenshots.

## Embed

```html
<img src="<storage-url>/<name>.svg" alt="<the aria-label>" className="not-typeset my-8 w-full" />
```
Don't use markdown `![]()`: that route adds a border and rounded corners. An animated figure uses `<object data="…" type="image/svg+xml" aria-label="…" className="not-typeset my-8 block w-full" />`, since animation inside `<img>` stutters in Chrome. Only animate when asked.
