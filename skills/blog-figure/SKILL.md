---
name: blog-figure
description: Draw a diagram for a Laminar blog post as an SVG in the house style (rounded charcoal panels, General Sans, one spacing grid, color only on tracked IDs), built as flexbox JSX that Takumi lays out and a small kit paints with real text and optional animated wires. Covers Laminar span trees drawn like the app's tree view. Use when asked to make, add, or redraw a figure, diagram, or illustration for a blog article, including an exported trace or span tree.
---

# Blog figure

A figure explains one mechanism in a post: what is stored, where a request goes, what changes. The style follows laminar.sh's landing cards: soft, dark, lots of air, very little ink. If the post already has figures, look at them first: yours must look like it belongs next to them.

Figures are **flexbox JSX, not hand-placed SVG**. You describe panels, rows, and cards; [Takumi](https://takumi.kane.tw) (`takumi-js`) lays them out; the kit below paints the measured layout as SVG with real `<text>` and General Sans embedded. You compute positions only for wires, and those come from measured boxes. Takumi's own `renderSvg` isn't used, because it turns text into glyph outlines, which render softer than real text.

## Look

- **Flat and dark.** Charcoal panels with rounded corners on the page's dark background. No borders, shadows, gradients, icons, or emoji. The one exception is a figure of something Laminar's app draws, like a span tree: it copies the app, icons included (see [Span trees](#span-trees)).
- **Less is the style.** Show only what the figure proves. Cut columns, rows, notes, and tags the prose already covers. A panel with three rows beats one with six. Leave generous space inside panels.
- **Text does the work.** Short labels and real identifiers from the post (`trace_id`, `signal_events`, `c7`). No line longer than a short phrase.
- **Color marks identity, and nothing else.** Most of a figure is grey. Color goes only on the IDs the post tracks across figures (the join keys, the new fields), and at most one accent per row. Field names stay grey. Don't tint whole rows to highlight them. To dim something, like a row nothing points at, use `textDim`.
- **One idea per figure,** read left to right or top to bottom. At most three panels across.
- **Titles are written as the thing is named.** Tables and identifiers are lowercase (`traces`, `signal_events`, `clusters_dict`). Names keep their spelling (`ClickHouse`, `Postgres`, `Coding agent`).

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

Every panel sits on one vertical grid, measured from the panel's top edge. The kit below already encodes it (as `PAD`, `ROW`, `LINE`, `BOTTOM`, `LINE_BOX`, `GAP`, `WIRE_GAP`, `INSET`); use these numbers for anything you add, don't eyeball.

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

Canvas: figures are 688 wide, the article column; the height is whatever the layout measures to.

## Setup

In a scratch folder (or the blog repo), with Node 18+:

```bash
npm i takumi-js react react-dom subset-font tsx
```

Get General Sans' variable font, `GeneralSans-Variable.woff2`: use the project's copy if it has one, or the free download from fontshare.com/fonts/general-sans. Point `FIGURE_FONT` at it.

Add a `package.json` with `"type": "module"` and a `tsconfig.json` with `"jsx": "react-jsx"`. Then save the kit below as `figure-kit.tsx`, unchanged.

## The kit

```tsx
// figure-kit.tsx — Laminar blog figures: flexbox JSX laid out by Takumi, painted as SVG with real text.
// deps: takumi-js react react-dom subset-font; font: GeneralSans-Variable.woff2 (fontshare.com/fonts/general-sans)
import { readFile } from "node:fs/promises";
import { Children, cloneElement, type CSSProperties, Fragment, isValidElement, type ReactNode } from "react";
import { renderToStaticMarkup } from "react-dom/server";
import subsetFont from "subset-font";
import { fromJsx } from "takumi-js/helpers/jsx";
import { type MeasuredNode, Renderer } from "takumi-js/node";

// ---------- Tokens ----------
export const C = {
  stroke: "#484848", text: "#b8b8b8", dim: "#777777", title: "#ffffff", body: "#212121", pill: "#2c2c2c",
  glyph: "#858585", ch: "#ecbb4f", pg: "#6aa4f5", accent: "#c66945", engine: "#1f7bd8", cluster: "#ef6aac", pulse: "#ec7b4f",
};
export const TYPE = { title: 16, text: 15, small: 13 };
const WEIGHT = 460, TITLE_WEIGHT = 500, TRACKING = 0.2;
export const PAD = 16, ROW = 32, LINE = 24, BOTTOM = 20, LINE_BOX = 44, GAP = 16, WIRE_GAP = 32, INSET = 8;
const RADIUS = 6, RADIUS_SM = 4, WIRE = 1.5, ASCENT = 1.01; // General Sans ascent in em

export type Box = { x: number; y: number; w: number; h: number };
export type Boxes = Record<string, Box>;
export type Pt = { x: number; y: number };
export const right = (b: Box): Pt => ({ x: b.x + b.w, y: b.y + b.h / 2 });
export const left = (b: Box): Pt => ({ x: b.x, y: b.y + b.h / 2 });
export const midY = (b: Box) => b.y + b.h / 2;

export interface FigureDef {
  file: string; // output name without .svg
  label: string; // alt text
  width: number; // 688 = article column
  draw: (b?: Boxes) => ReactNode; // b: measured boxes on the second pass
  overlay?: (b: Boxes) => { under?: ReactNode; over?: ReactNode };
}

// ---------- Static layer (flexbox on the grid: title baseline 27, rows at 59 + 32k, 20 under the last) ----------
const t = (fontSize: number, color: string, fontWeight = WEIGHT): CSSProperties => ({ fontSize, color, fontWeight });
const flex = (s: CSSProperties = {}): CSSProperties => ({ display: "flex", ...s });

export const Root = ({ w, style, children }: { w: number; style?: CSSProperties; children: ReactNode }) => (
  <div style={flex({ position: "relative", width: w, fontFamily: "General Sans", letterSpacing: TRACKING, ...style })}>{children}</div>
);

export const Panel = ({ id, title, dot, w, grow, style, children }: { id?: string; title?: string; dot?: string; w?: number; grow?: boolean; style?: CSSProperties; children?: ReactNode }) => (
  <div id={id} style={flex({ flexDirection: "column", width: w, flexGrow: grow ? 1 : undefined, background: C.body, borderRadius: RADIUS, padding: `11px ${PAD}px 9px`, ...style })}>
    {title ? (
      <div style={flex({ alignItems: "center", gap: 10, height: 22, ...t(TYPE.title, C.title, TITLE_WEIGHT) })}>
        {dot ? <div style={{ width: 7, height: 7, borderRadius: 4, background: dot }} /> : null}
        {title}
      </div>
    ) : null}
    <div style={flex({ flexDirection: "column", marginTop: title ? 5 : 0 })}>{children}</div>
  </div>
);

export const Row = ({ id, h = ROW, style, children }: { id?: string; h?: number; style?: CSSProperties; children?: ReactNode }) => (
  <div id={id} style={flex({ alignItems: "center", height: h, ...t(TYPE.text, C.text), ...style })}>{children}</div>
);

export const Cell = ({ id, w, color = C.text, size = TYPE.text, align = "start", children }: { id?: string; w?: number; color?: string; size?: number; align?: "start" | "center" | "end"; children?: ReactNode }) => (
  <div id={id} style={flex({ width: w, flexGrow: w === undefined ? 1 : undefined, justifyContent: { start: "flex-start", center: "center", end: "flex-end" }[align], ...t(size, color) })}>{children}</div>
);

export const HeaderBand = ({ cols }: { cols: [string, number?][] }) => (
  <Row>
    <div style={flex({ alignItems: "center", height: 24, flexGrow: 1, margin: `0 ${INSET - PAD}px`, padding: `0 ${PAD - INSET}px`, background: C.pill, borderRadius: RADIUS_SM })}>
      {cols.map(([name, w]) => <Cell key={name} w={w} size={TYPE.small} color={C.dim}>{name}</Cell>)}
    </div>
  </Row>
);

export const GroupTint = ({ color, children }: { color: string; children: ReactNode }) => (
  <div style={flex({ flexDirection: "column", margin: `0 ${INSET - PAD}px`, padding: `0 ${PAD - INSET}px`, background: `${color}12`, borderRadius: RADIUS_SM })}>{children}</div>
);

export const LineBox = ({ id, label, h = LINE_BOX, size = TYPE.text, w }: { id?: string; label: ReactNode; h?: number; size?: number; w?: number }) => (
  <div id={id} style={flex({ alignItems: "center", height: h, width: w, padding: `0 ${PAD}px`, background: C.body, borderRadius: RADIUS, ...t(size, C.text) })}>{label}</div>
);

export const Card = ({ id, label, lines, style }: { id?: string; label: ReactNode; lines: ReactNode[]; style?: CSSProperties }) => (
  <div id={id} style={flex({ flexDirection: "column", background: C.pill, borderRadius: RADIUS_SM, padding: `11px ${PAD}px 13px`, ...style })}>
    <div style={flex({ alignItems: "center", height: 22, ...t(TYPE.text, C.title) })}>{label}</div>
    <div style={flex({ flexDirection: "column", marginTop: 9 })}>{lines.map((l, i) => <Row key={i} h={LINE}>{l}</Row>)}</div>
  </div>
);

export const Pill = ({ id, lines, w, h = LINE_BOX }: { id?: string; lines: [string, string?]; w: number; h?: number }) => (
  <div id={id} style={flex({ flexDirection: "column", alignItems: "center", justifyContent: "center", width: w, height: h, background: C.body, borderRadius: h / 2, ...t(TYPE.text, C.text) })}>
    <div style={flex()}>{lines[0]}</div>
    {lines[1] ? <div style={flex(t(TYPE.small, C.dim))}>{lines[1]}</div> : null}
  </div>
);

export const Chip = ({ id, label, color, w, h = 24 }: { id?: string; label: string; color?: string; w: number; h?: number }) => (
  <div id={id} style={flex({ alignItems: "center", justifyContent: "center", width: w, height: h, borderRadius: RADIUS_SM, background: color ? `${color}24` : C.pill, ...t(TYPE.small, color ?? C.dim) })}>{label}</div>
);

export const Caption = ({ n, text, w }: { n: number; text: string; w: number }) => (
  <div style={flex({ gap: 8, width: w, ...t(TYPE.text, C.text) })}><span style={{ color: C.dim }}>{`${n}.`}</span>{text}</div>
);

/** Text placed after measuring; `y` is the baseline. Absolute, so it never moves the layout. */
export const Label = ({ x, y, anchor = "start", size = TYPE.small, color = C.dim, children }: { x: number; y: number; anchor?: "start" | "middle" | "end"; size?: number; color?: string; children: ReactNode }) => {
  const w = 400, lh = size * 1.25;
  return (
    <div style={flex({ position: "absolute", left: anchor === "start" ? x : anchor === "middle" ? x - w / 2 : x - w, top: y - ASCENT * size, width: w, height: lh, justifyContent: { start: "flex-start", middle: "center", end: "flex-end" }[anchor], ...t(size, color) })}>{children}</div>
  );
};

// ---------- Wire layer (plain SVG, placed from measured boxes) ----------
const ctrl = (a: Pt, b: Pt): [Pt, Pt] => [{ x: a.x + (b.x - a.x) * 0.34, y: a.y }, { x: a.x + (b.x - a.x) * 0.57, y: b.y }];
export const wirePath = (pts: Pt[], curve = false) =>
  pts.map((b, i) => {
    if (!i) return `M ${b.x} ${b.y}`;
    const a = pts[i - 1];
    if (!curve || a.y === b.y) return `L ${b.x} ${b.y}`;
    const [c1, c2] = ctrl(a, b);
    return `C ${c1.x} ${c1.y} ${c2.x} ${c2.y} ${b.x} ${b.y}`;
  }).join(" ");
const stroke = { fill: "none", stroke: C.stroke, strokeWidth: WIRE, strokeLinecap: "round", strokeLinejoin: "round" } as const;

/** A connector; `curve` turns each change of height into one level-in, level-out S-curve; `dashed` marks a reply. */
export const Wire = ({ points, curve, dashed }: { points: Pt[]; curve?: boolean; dashed?: boolean }) => (
  <path d={wirePath(points, curve)} {...stroke} strokeDasharray={dashed ? "3 4" : undefined} />
);
export const Chevron = ({ x, y, dx, dy }: { x: number; y: number; dx: number; dy: number }) => (
  <path d={`M ${x - dx * 5 - dy * 4} ${y - dy * 5 + dx * 4} L ${x} ${y} L ${x - dx * 5 + dy * 4} ${y - dy * 5 - dx * 4}`} {...stroke} />
);
/** Axis-aligned connector with a chevron at the end. */
export const Arrow = ({ points, dashed }: { points: Pt[]; dashed?: boolean }) => {
  const [p, q] = points.slice(-2);
  return <g><Wire points={points} dashed={dashed} /><Chevron x={q.x} y={q.y} dx={Math.sign(q.x - p.x)} dy={Math.sign(q.y - p.y)} /></g>;
};

// Animated dashes: an 11-unit pulse sliding along a copy of a wire (SMIL), same speed on every wire.
const DASH = 11, SPEED = 380;
export const SCATTER = [0.2, 3.1, 5.4, 1.7, 4.6, 0.9, 5.8, 2.4, 3.8, 5.1, 1.2, 4.2];
export const wireLength = (pts: Pt[]) =>
  pts.reduce((sum, b, i) => {
    if (!i) return 0;
    const a = pts[i - 1];
    if (a.y === b.y) return sum + Math.abs(b.x - a.x);
    const [c1, c2] = ctrl(a, b);
    let len = 0, prev = a;
    for (let n = 1; n <= 32; n++) {
      const s = n / 32, u = 1 - s;
      const q = { x: u ** 3 * a.x + 3 * u * u * s * c1.x + 3 * u * s * s * c2.x + s ** 3 * b.x, y: u ** 3 * a.y + 3 * u * u * s * c1.y + 3 * u * s * s * c2.y + s ** 3 * b.y };
      len += Math.hypot(q.x - prev.x, q.y - prev.y);
      prev = q;
    }
    return sum + len;
  }, 0);

function Pulse({ points, delay, period, reverse }: { points: Pt[]; delay: number; period: number; reverse?: boolean }) {
  const len = wireLength(points), travel = len / SPEED;
  const [from, to] = reverse ? [-len, DASH] : [DASH, -len];
  const common = { dur: `${period.toFixed(3)}s`, begin: `${delay.toFixed(3)}s`, repeatCount: "indefinite" };
  const arrive = (travel / period).toFixed(4);
  return (
    <path className="flow" d={wirePath(points, true)} {...stroke} stroke={C.pulse} strokeDasharray={`${DASH} ${len + 2 * DASH}`} strokeDashoffset={from} opacity={0}>
      <animate attributeName="stroke-dashoffset" {...common} calcMode="linear" values={`${from};${to};${to}`} keyTimes={`0;${arrive};1`} />
      <animate attributeName="opacity" {...common} calcMode="discrete" values="1;0" keyTimes={`0;${arrive}`} />
    </path>
  );
}

/** Request out and reply back on each wire; `departures[i]` = { at: seconds, path: index }. */
export function Dashes({ paths, departures }: { paths: Pt[][]; departures: { at: number; path: number }[] }) {
  const travels = paths.map((p) => wireLength(p) / SPEED);
  const period = Math.max(...SCATTER) + 2 * Math.max(...travels) + 1;
  return <>{departures.map(({ at, path }, i) => (
    <g key={i}>
      <Pulse points={paths[path]} delay={at} period={period} />
      <Pulse points={paths[path]} delay={at + travels[path] + 0.2} period={period} reverse />
    </g>
  ))}</>;
}

// ---------- Build: Takumi lays out, we paint real <text> and embed the font ----------
type TNode = { type: string; id?: string; text?: string; src?: string; style?: CSSProperties; children?: TNode[] };
type Inh = { color: string; size: number; weight: number };

/** Takumi only reports text positions for text in its own element, so wrap bare strings in flex parents. */
function wrapText(node: ReactNode, inFlex = false): ReactNode {
  if (typeof node === "string" || typeof node === "number") return inFlex ? <span>{node}</span> : node;
  if (Array.isArray(node)) return node.map((n) => wrapText(n, inFlex));
  if (!isValidElement(node)) return node;
  const el = node as React.ReactElement<{ children?: ReactNode; style?: CSSProperties }>;
  if (typeof el.type === "function") return wrapText((el.type as (p: unknown) => ReactNode)(el.props), inFlex);
  if ((el.type as unknown) === Fragment) return Children.map(el.props.children, (c) => wrapText(c, inFlex));
  if (el.type === "svg") return el;
  return cloneElement(el, undefined, Children.map(el.props.children, (c) => wrapText(c, el.props.style?.display === "flex")));
}

const esc = (s: string) => s.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;");
const inherit = (n: TNode, p: Inh): Inh => ({
  color: typeof n.style?.color === "string" ? n.style.color : p.color,
  size: typeof n.style?.fontSize === "number" ? n.style.fontSize : p.size,
  weight: typeof n.style?.fontWeight === "number" ? n.style.fontWeight : p.weight,
});
const leaves = (n: TNode, p: Inh, out: { t: Inh }[] = []) => {
  const own = inherit(n, p);
  if (n.type === "text" && n.text !== undefined) out.push({ t: own });
  n.children?.forEach((c) => leaves(c, own, out));
  return out;
};

function paint(n: TNode, m: MeasuredNode, p: Inh, out: string[], boxes: Boxes) {
  const s = inherit(n, p);
  const [x, y] = [m.transform[4], m.transform[5]];
  if (n.id) boxes[n.id] = { x, y, w: m.width, h: m.height };
  const fill = (n.style?.background ?? n.style?.backgroundColor) as string | undefined;
  if (fill) out.push(`<rect x="${x}" y="${y}" width="${m.width}" height="${m.height}" rx="${Number(n.style?.borderRadius) || 0}" fill="${fill}"/>`);
  if (n.type === "image" && n.src) {
    const open = n.src.match(/<svg\b[^>]*>/)?.[0] ?? "<svg>";
    out.push(n.src.replace(open, open.replace(/\s(x|y|width|height|style)="[^"]*"/g, "").replace("<svg", `<svg x="${x}" y="${y}" width="${m.width}" height="${m.height}"`)));
    return;
  }
  if (m.runs.length) {
    const ls = leaves(n, p);
    m.runs.forEach((r, i) => {
      const { t: f } = ls[i] ?? ls[ls.length - 1];
      out.push(`<text x="${(x + r.x).toFixed(2)}" y="${(y + r.y + ASCENT * f.size).toFixed(2)}" fill="${f.color}" style="font-family:var(--figure-font);font-weight:${f.weight};font-size:${f.size}px;letter-spacing:${TRACKING}px;white-space:pre">${esc(r.text)}</text>`);
    });
    return;
  }
  n.children?.forEach((c, i) => m.children[i] && paint(c, m.children[i], s, out, boxes));
}

export async function buildFigure(def: FigureDef, fontPath: string) {
  const font = await readFile(fontPath);
  const renderer = new Renderer();
  await renderer.registerFont({ name: "General Sans", data: font });
  const layout = async (tree: ReactNode) => {
    const { node } = (await fromJsx(wrapText(tree) as React.ReactElement)) as unknown as { node: TNode };
    const m = await renderer.measure(node as never);
    const boxes: Boxes = {}, out: string[] = [];
    paint(node, m, { color: C.text, size: TYPE.text, weight: WEIGHT }, out, boxes);
    return { m, boxes, out };
  };
  const first = await layout(def.draw()); // pass 1: measure
  const { m, boxes, out } = await layout(def.draw(first.boxes)); // pass 2: place what depends on measurements
  const { under, over } = def.overlay?.(boxes) ?? {};
  const svgOf = (n: ReactNode) => (n ? renderToStaticMarkup(<>{n}</>) : "");
  const body = svgOf(under) + out.join("") + svgOf(over);

  // Embed General Sans, subset to the glyphs this figure draws, one instance per weight used.
  const glyphs = [...new Set([...body.matchAll(/<text[^>]*>([^<]*)<\/text>/g)].map((x) => x[1]).join("").replace(/&amp;/g, "&").replace(/&lt;/g, "<").replace(/&gt;/g, ">").replace(/&quot;/g, '"'))].join("");
  const faces = await Promise.all([WEIGHT, TITLE_WEIGHT].map(async (wght) => {
    const sub = await subsetFont(font, glyphs, { targetFormat: "woff2", variationAxes: { wght } });
    return `@font-face{font-family:"Figure Sans";font-weight:${wght};src:url(data:font/woff2;base64,${sub.toString("base64")}) format("woff2")}`;
  }));
  const style = `${faces.join("")}svg{--figure-font:"Figure Sans",ui-sans-serif,sans-serif;color-scheme:dark;-webkit-font-smoothing:antialiased;-moz-osx-font-smoothing:grayscale}@media (prefers-reduced-motion:reduce){.flow{display:none}}`;
  const h = Math.ceil(m.height), alt = esc(def.label);
  return `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 ${def.width} ${h}" width="${def.width}" height="${h}" role="img" aria-label="${alt}"><title>${alt}</title><style>${style}</style>${body}</svg>\n`;
}
```

## Building a figure

A figure is a `FigureDef`:
- **`draw`** builds the static layer as flexbox JSX.
- **`overlay`** adds the wires and dashes, placed from measured boxes.
- **`label`** is the alt text.

`buildFigure` runs `draw()` once and measures it. It then runs `draw(b)` again with every `id`'d box in `b`, for anything placed from measurements: a store lined up with a step, a label on a wire. It paints the measured tree, puts `overlay.under` beneath it and `overlay.over` on top, and embeds the font subset to the figure's glyphs.

Give an `id` to every box a wire attaches to, and look it up by name (`b.engine`, `b.s0`). Never find boxes by their order.

A complete example: an animated fan-in figure and a table. Start from the closer one.

```tsx
// figures.tsx
import { Cell, C, Dashes, type FigureDef, HeaderBand, LineBox, Panel, Root, Row, right, SCATTER, Wire } from "./figure-kit";

const SURFACES = ["Platform", "CLI", "API · SDK", "MCP · coding agent"];

export const surfaces: FigureDef = {
  file: "example-surfaces",
  label: "Clients sending SQL through one query engine to ClickHouse",
  width: 688,
  draw: () => (
    <Root w={688} style={{ alignItems: "center", gap: 36 }}>
      <div style={{ display: "flex", flexDirection: "column", gap: 16, width: 176 }}>
        {SURFACES.map((s, i) => <LineBox key={s} id={`s${i}`} label={s} />)}
      </div>
      <Panel id="engine" title="Query engine" dot={C.engine} grow>
        <Row />
        <Row>{["validate", "scope", "run"].map((s, i) => <Cell key={s} id={`st${i}`} align="center">{s}</Cell>)}</Row>
      </Panel>
      <Panel id="db" title="ClickHouse" dot={C.ch} w={160}>
        {["spans", "traces", "signal_events", "clusters"].map((t) => <Row key={t}>{t}</Row>)}
      </Panel>
    </Root>
  ),
  overlay: (b) => {
    const y = b.engine.y + 54; // middle of the engine's first row
    const paths = SURFACES.map((_, i) => [right(b[`s${i}`]), { x: b.engine.x, y }, { x: b.engine.x + b.engine.w, y }, { x: b.db.x, y }]);
    return {
      under: paths.map((p, i) => <Wire key={i} points={p} curve />),
      over: (
        <>
          <Wire points={[{ x: b.engine.x, y }, { x: b.engine.x + b.engine.w, y }]} />
          {[0, 1, 2].map((i) => <rect key={i} x={b[`st${i}`].x + b[`st${i}`].w / 2 - 3} y={y - 3} width={6} height={6} rx={1} fill={C.stroke} />)}
          <Dashes paths={paths} departures={SCATTER.map((at, i) => ({ at, path: i % paths.length }))} />
        </>
      ),
    };
  },
};

export const table: FigureDef = {
  file: "example-table",
  label: "The signal_events table: each event points at a trace and a cluster.",
  width: 688,
  draw: () => (
    <Root w={688} style={{ justifyContent: "center" }}>
      <Panel title="signal_events" dot={C.accent} w={288}>
        <HeaderBand cols={[["id", 80], ["trace_id", 84], ["cluster_id"]]} />
        {[["ev_91", "t1", "c4"], ["ev_92", "t1", "c7"], ["ev_93", "t2", "c7"]].map(([e, t, c]) => (
          <Row key={e}><Cell w={80}>{e}</Cell><Cell w={84} color={C.ch}>{t}</Cell><Cell color={C.cluster}>{c}</Cell></Row>
        ))}
      </Panel>
    </Root>
  ),
};
```

```tsx
// build.tsx: FIGURE_FONT=path/to/GeneralSans-Variable.woff2 npx tsx build.tsx
import { mkdir, writeFile } from "node:fs/promises";
import { buildFigure } from "./figure-kit";
import { surfaces, table } from "./figures";

const FONT = process.env.FIGURE_FONT ?? "GeneralSans-Variable.woff2";
await mkdir("out", { recursive: true });
for (const def of [surfaces, table]) {
  const svg = await buildFigure(def, FONT);
  await writeFile(`out/${def.file}.svg`, svg);
  console.log(`out/${def.file}.svg`, `${(svg.length / 1024).toFixed(1)} KB`);
}
```

## Blocks

Every block already sits on the spacing grid, so a panel of rows needs no numbers.

| Block | Use | House rule it encodes |
| --- | --- | --- |
| `Root w` | The figure; a positioned flex container | Set `gap`, `alignItems`, `flexDirection` via `style` |
| `Panel title dot w grow` | The unit everything is built from | Rounded body, title inside on the grid, rows on `ROW`, `BOTTOM` under the last |
| `Row` / `Cell w color size align` | Rows and their columns | 32 high; a `Cell` without `w` takes the rest of the row |
| `HeaderBand cols` | A table's column names | On a `pill` band inset by `INSET`, `small`/`dim` |
| `GroupTint color` | One soft block behind a group of rows | 7% of the identity color; never one tint per row |
| `LineBox label h size` | One-line boxes (entry points, endpoints) | 44 high unless told otherwise |
| `Card label lines` | A step inside a panel | Laid out like a panel: a one-line card and a one-row panel are both 79 high |
| `Pill lines w h` | An operation or decision (canonicalize, lookup, "new to trace?") | Fully rounded: steps, not data |
| `Chip label color w h` | A short ID or hash as a token | Tinted in its color when new or tracked; `pill` grey when resent |
| `Caption n text w` | Numbered caption under a row of panels | Dim `1.`, then the caption |
| `Label x y anchor` | Text placed after measuring (wire labels, yes/no) | Absolute, `small`/`dim`; `y` is the baseline |

For anything else, use a plain `<div style={{ display: "flex", … }}>` with the tokens: padding and gaps from the spacing table, colors from `C`, sizes from `TYPE`. Colored runs in one line go in a `<span style={{ display: "block" }}>` holding the text and `<span style={{ color }}>` pieces.

## Wires and animation

- **`Wire points curve dashed`** draws a connector. With `curve`, each change of height is one S-curve that leaves and arrives level. `dashed` marks a reply.
- **`Arrow points`** draws an axis-aligned connector with a chevron at the end. `Chevron x y dx dy` draws the chevron alone.
- **Request and reply:** the request is solid with its label (`Label`, `small`, centered, 10 above the wire). The reply is dashed, 14 below, unlabelled, pointing back.
- **Stations:** a 6×6 `rect rx=1` in `stroke`, centered on the wire, with its label in the row below.
- **Animated dashes:** `Dashes` sends a pulse out and back along each wire, all at one speed, on the `SCATTER` timetable. It uses SMIL, so it runs inside the SVG file with no script. Only animate when asked.
- **Arrowheads on curves:** a curve that climbs steeply over a short gap is still slanted where it ends, so a level chevron looks broken. End the wire with a short level run: `points={[from, { x: end.x - 12, y: end.y }, end]}`, then the `Chevron` at `end`.
- **Layering:** `under` is for wires (panels cover their ends) and for dashes that should hide behind boxes. `over` is for a wire drawn through a panel, stations, and dashes meant to be seen passing a panel.

## Span trees

A figure of a trace copies Laminar's tree view, so readers recognize the screen. Draw the nesting a real screenshot of the run shows; if the post says otherwise, tell the author.

- One `ROW` per span, indented 22 per level, with a line of elbows in `stroke` from each parent's icon to each child's.
- Before each name, the app's span-type icon: white lucide glyph on the type's color. Names stay `text`; these colors mean span types only.
- Optional, only when it is the point: an LLM span's output under its name (`transfer_to_bookingagent`), or a right-aligned note per row (`@observe root`). Both `small`/`dim`.
- Root as the panel title, or as the first row when every level matters. Cut repeated branches; ten rows is plenty. Panel 420 wide, centered.

```tsx
// span-tree.tsx
import type { ReactNode } from "react";
import { type Boxes, C, midY, Row, TYPE } from "./figure-kit";

export type Span = { name: string; depth: number; kind: "default" | "llm" | "tool"; preview?: string; note?: string };
const INDENT = 22, TILE = 20;
// The app's SpanTypeIcon: lucide Braces, MessageCircle, Bolt; colors flattened onto `body`.
const KINDS: Record<Span["kind"], [string, ReactNode]> = {
  default: ["#4d7db9", <><path d="M8 3H7a2 2 0 0 0-2 2v5a2 2 0 0 1-2 2 2 2 0 0 1 2 2v5c0 1.1.9 2 2 2h1" /><path d="M16 21h1a2 2 0 0 0 2-2v-5c0-1.1.9-2 2-2a2 2 0 0 1-2-2V5a2 2 0 0 0-2-2h-1" /></>],
  llm: ["#7c3aed", <path d="M2.992 16.342a2 2 0 0 1 .094 1.167l-1.065 3.29a1 1 0 0 0 1.236 1.168l3.413-.998a2 2 0 0 1 1.099.092 10 10 0 1 0-4.777-4.719" />],
  tool: ["#d09a0a", <><path d="M21 16V8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16z" /><circle cx="12" cy="12" r="4" /></>],
};
const dim = { fontSize: TYPE.small, color: C.dim };

/** Rows r0, r1, …; pass `from`/`to` to wrap part of the tree in a GroupTint. */
export const SpanRows = ({ spans, from = 0, to = spans.length }: { spans: Span[]; from?: number; to?: number }) => (
  <>{spans.slice(from, to).map((s, k) => (
    <div key={k} style={{ display: "flex", flexDirection: "column" }}>
      <Row id={`r${from + k}`} style={{ paddingLeft: s.depth * INDENT, gap: 8 }}>
        <div style={{ display: "flex", alignItems: "center", justifyContent: "center", width: TILE, height: TILE, borderRadius: 4, background: KINDS[s.kind][0] }}>
          <svg width={14} height={14} viewBox="0 0 24 24" fill="none" stroke="#fff" strokeWidth={2} strokeLinecap="round" strokeLinejoin="round">{KINDS[s.kind][1]}</svg>
        </div>
        <div style={{ display: "flex" }}>{s.name}</div>
        {s.note ? <div style={{ display: "flex", flexGrow: 1, justifyContent: "flex-end", ...dim }}>{s.note}</div> : null}
      </Row>
      {s.preview ? <Row h={20} style={{ paddingLeft: s.depth * INDENT + TILE + 8, marginTop: -6, ...dim }}>{s.preview}</Row> : null}
    </div>
  ))}</>
);

/** In `overlay.over`. */
export const Elbows = ({ spans, b }: { spans: Span[]; b: Boxes }) => {
  const x = (i: number) => b[`r${i}`].x + spans[i].depth * INDENT;
  return (
    <g stroke={C.stroke} fill="none" strokeLinecap="round">
      {spans.map((s, i) => {
        let p = i - 1;
        while (p >= 0 && spans[p].depth >= s.depth) p--;
        return p < 0 ? null : <path key={i} d={`M ${x(p) + TILE / 2} ${midY(b[`r${p}`]) + TILE / 2 + 3} V ${midY(b[`r${i}`])} H ${x(i) - 4}`} />;
      })}
    </g>
  );
};
```

Use it like any panel of rows:

```tsx
draw: () => <Root w={688} style={{ justifyContent: "center" }}><Panel title="airline-support" dot={C.ch} w={420}><SpanRows spans={SPANS} /></Panel></Root>,
overlay: (b) => ({ over: <Elbows spans={SPANS} b={b} /> }),
```

A tree that ends on a preview line gets `paddingBottom: 15` on its panel.

## Layout

- **Canvas.** Width 688, the article column. It renders about 612px wide (×0.89), smaller on phones. The height is whatever the layout measures to. Two figures that sit side by side in the text share one height (`minHeight` on `Root`).
- **Panels in a row share one top and one height:** a flex row's default `alignItems: stretch` does this. Widths plus gaps add up to 688; give one panel `grow` to take the rest.
- **Gaps** are 16 between panels with nothing between them, and at least 32 where a wire crosses.
- **Align across panels:** the same data on the same rows; captions in one row under the panels, with the same widths and gaps.
- **Wide figures on phones:** if the 360px render isn't legible, add a stacked variant at width 336 (`<file>-compact`).

## Fit (where figures break)

Budget every line before placing it. Widths per character, tracking included (measured in General Sans; IDs and hashes come out the same):

| Role | Units per character |
| --- | --- |
| `text` (15) | ~7.5 |
| `small` (13) | ~6.5 |
| `panelTitle` (16) | ~8 |

For example, `Search timeouts` (15 chars, `text`) needs ~113, plus 16 padding on each side. Flexbox won't warn you when text overflows a fixed-width cell, so check the tightest spots: the longest value in each column, captions under narrow panels, labels over short wires, and text in chips. If it doesn't fit:
1. Cut words.
2. Widen the panel.
3. Drop that text to `small`, which is the smallest size there is.

## Gotchas

- **Glyphs General Sans lacks** (`⋈`, arrows, math symbols) silently disappear or fall back to another font. Draw them as a small inline `<svg>` in the row, e.g. `<svg width={16} height={16} viewBox="0 0 44 44" style={{ margin: "0 4px" }}><path d="M10 13V31L34 13V31L10 13Z" fill="none" stroke="#b8b8b8" strokeWidth={3} strokeLinejoin="round" /></svg>` for ⋈.
- **Don't add padding twice.** A row's measured box already starts where its text starts. Offsets for indents and tree elbows go from the row's box, not from the row's box plus `PAD`.
- **Things placed after measuring can only use boxes from the first pass.** A label can be placed from a step's box, but not from a store that is itself placed in the second pass.
- **Icons land on whole pixels.** Takumi rounds an inline `<svg>`'s position, so an icon with an odd inset in its tile (13 in 20) sits half a pixel off-center. Size icons so the inset is even: 14 in 20, or 18 in 32.
- **Bare text in a flex element is fine:** the kit wraps it before layout, so its position is reported and backgrounds stay on the container.

## Check

Load each finished file in headless Chrome at 612px wide (the article column) and at 360px (a phone), and look at both screenshots. For example: `chrome --headless=new --window-size=612,<h> --screenshot=out.png page.html`, where `page.html` is `<body style="margin:0;background:#1a1a1a"><object data="<file>" type="image/svg+xml" style="width:612px"></object>`. Fix anything clipped, touching, overlapping, off the grid, or misaligned, and repeat until the 612px render is clean. At 360px, text only has to stay legible; if it doesn't, say so and offer a stacked `<file>-compact.svg`.

The `label` says what the figure shows in one or two sentences; it is the alt text. Report the file path and the embed line. Don't claim the figure is clean without having looked at the screenshots.

## Embed

```html
<img src="<storage-url>/<file>.svg" alt="<the label>" className="not-typeset my-8 w-full" />
```
Don't use markdown `![]()`: that route adds a border and rounded corners. An animated figure uses `<object data="…" type="image/svg+xml" aria-label="…" className="not-typeset my-8 block w-full" />`, since animation inside `<img>` stutters in Chrome.
