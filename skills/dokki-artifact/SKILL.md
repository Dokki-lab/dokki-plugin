---
name: dokki-artifact
description: Create or edit artifacts — self-contained HTML pages, dashboards, charts, diagrams, animations, calculators, prototypes. Use for anything interactive or visual beyond what a document can express.
argument-hint: <description-or-id> [instruction]
allowed-tools: mcp__dokki__create mcp__dokki__read mcp__dokki__edit mcp__dokki__preview_resource
---

# Dokki Artifact — Create & Edit

Artifacts live behind `create {action:"artifact"}`, `read {action:"artifact"}`, and
`edit {action:"artifact.update"/"artifact.patch"}`. Each is `<facade> {action, <ids>, args:{…}}`.

## Is an artifact the right resource?

| Need | Resource |
|------|----------|
| Static prose, lists, simple tables | `dokki-document` |
| Structured rows/columns with sort/filter | `dokki-table` |
| Self-contained page or widget | **artifact** |
| Interactive widget, custom layout, animation | **artifact** |
| Chart / graph from data | **artifact** |
| Flowchart, sequence diagram, state diagram | **artifact** |

---

## Format: write HTML

**Every new artifact is a complete HTML document.** Nothing else.

An artifact renders in a sandboxed iframe — no build step, no compile step that
can fail, no framework assumed. Dokki injects only the `window.dokki` sandbox
bridge before the artifact's scripts. Everything visual goes in the one file:

- styling → a `<style>` tag (or a CDN `<script>`/`<link>`)
- behaviour → a `<script>` tag
- data → inline, or read live from the workspace (see below)

No CSS or framework is injected. A page that relies on Tailwind classes without
loading Tailwind renders unstyled.

### CDN libraries that work

```html
<script src="https://cdn.tailwindcss.com"></script>
<script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdn.jsdelivr.net/npm/d3@7"></script>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@xyflow/react@12/dist/style.css">
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=…&display=swap">
```

---

## Sandbox storage and collaborative Artifact state

An artifact runs on an **opaque origin** (`sandbox="allow-scripts allow-popups"`, no
`allow-same-origin`). Every storage API throws there:

| API | What happens |
|---|---|
| `localStorage` | `SecurityError` |
| `sessionStorage` | `SecurityError` |
| `document.cookie` | `SecurityError` |
| `indexedDB` | `SecurityError` |

It throws **while your script is running**, so it does not disable the save — it
kills the whole page, and the user sees a blank artifact. This is the single most
common way a correct-looking generated artifact renders as nothing.

- Session-only state → a plain JavaScript variable.
- Artifact-owned durable JSON → `window.dokki.state` (below). It is persisted in
  the Artifact's Y.Doc and synchronized between editors.
- Shared business/source-of-truth data → a Dokki **table** or **document**; read
  it through the live workspace data channel below.
- Live workspace data → `window.dokki.readTable` / `readDocument` (below).

`create` and `edit` **reject** an artifact that uses any of the four browser
storage globals above, so a page that reaches for them never reaches the user.

### Persistent state API

```js
const snapshot = await window.dokki.state.get();       // { [key]: JSONValue }
const node = await window.dokki.state.get("node:a");  // value or null
await window.dokki.state.set("node:a", { x: 10, y: 20 });
await window.dokki.state.delete("node:a");

const unsubscribe = window.dokki.state.subscribe((nextSnapshot) => {
  render(nextSnapshot);
});
```

Use one key per independently edited entity (`node:<id>`, `edge:<id>`), not one
giant JSON document. Values must be JSON. A value is capped at 64 KiB and total
Artifact state at 1 MiB. Viewers can read but persistent writes require Artifact
edit permission; Hocuspocus remains the authoritative write boundary.

### AI and MCP Diagram editing

A Diagram is an Artifact variant, but its visible graph is not the `source`
string. Read it with `diagram_read(resource_id)` and update it with
`diagram_update(resource_id, expected_state_hash, mode, nodes, edges)`. Always
pass the latest returned state hash; a stale hash fails instead of overwriting a
human edit. Use `mode: "replace"` to redraw the graph and `mode: "merge"` for
targeted upserts/deletes. Supported node shapes are `rectangle`, `rounded`,
`diamond`, and `circle`; labels remain editable in Dokki and every node renders
top/right/bottom/left connection handles.

If `diagram_read` returns `runtime_compatible: false`, the Diagram was replaced
by legacy static source. Re-read it and call `diagram_update` with
`repair_runtime: true`. This explicitly restores Dokki's collaborative Diagram
runtime and writes the graph in one state-hash-guarded transaction.

Never use `artifact_update` or `artifact_patch` to replace a Diagram with a
static HTML snapshot. Those generic source tools reject Diagram resources. The
AI and MCP tools mutate the same `artifact-live-state` Y.Map as
`window.dokki.state`, so changes persist, broadcast to open collaborators, and
remain subject to the normal Artifact edit permission.

Presence is ephemeral and is never persisted:

```js
await window.dokki.presence.set({ cursor: { x: 10, y: 20 }, selected: ["node:a"] });
const stop = window.dokki.presence.subscribe((peers) => renderCursors(peers));
```

Use presence only for cursors, selections, and transient viewer state. The live
state and presence host is currently available on the Artifact page. Overview
widgets, document embeds, and published pages must handle `unsupported` and show
a deliberate fallback; publishing does not make collaborative state public.

### JSX is deprecated

Dokki still compiles and renders JSX artifacts, and `read` will hand you JSX
source for older ones. It is being retired because the compile step is
unreliable — a source that looks fine can fail to compile or render, and the
author sees an error instead of a page.

- **Never** create a new artifact as JSX.
- Keep JSX **only** when editing an artifact whose current source is already
  JSX and the change is small and local.
- If a JSX artifact keeps failing to compile or render, rewrite it in HTML with
  `artifact.update` rather than patching around the error.
- If the user explicitly asks for a React component, give them JSX — and say
  that HTML is the supported default.

For the JSX runtime (React 18, framer-motion, lucide-react, recharts, mermaid,
Tailwind, `export default`), see **Legacy: JSX artifacts** at the end.

---

## Design it to stand on its own

The artifact renders isolated. It inherits **nothing** from Dokki — not the theme,
not the fonts, not a reset. Whatever you don't set, the browser sets for you, and
it will look like a default.

- **Both themes, at token level.** Define the light palette on `:root`, then
  redefine only those tokens inside `@media (prefers-color-scheme: dark)`. Never
  give a colour its *only* definition inside the media query — that is how a page
  ends up with one theme's text on the other theme's background.
- **`body` needs an explicit `background`.** A transparent body borrows whatever
  is behind the iframe, which is not your palette.
- **Pick type deliberately.** Google Fonts is the one font host that loads; give
  every face a real fallback stack. A system stack is fine — an *unset* font is not.
- **Wide content scrolls itself.** Tables, code and diagrams go in a container with
  `overflow-x: auto` so the page body never scrolls sideways.
- **Semantic markup, real focus states.** Use `<button>`, `<table>`, `<label>`;
  keep it keyboard-navigable; give `:focus-visible` a visible outline; honour
  `prefers-reduced-motion`.
- **Line up your digits** with `font-variant-numeric: tabular-nums` in any column
  of numbers.

Avoid the looks that read as machine-made: cream `#F4F1EA` with a serif display and
a terracotta accent; a purple-to-blue gradient hero on white; emoji as section
markers; everything centred with `border-radius` on every box. If the user names a
direction, follow theirs instead — this list is only for when nothing is specified.

---

## CREATE Mode

### Action

```
create:
  action: "artifact"
  workspace_id: string
  parent_id?: string
  args:
    name: string
    source: string    # Complete HTML document
```

### Required source structure

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>…</title>
<style>
  :root { color-scheme: light dark; --bg: #fff; --fg: #18181b; --muted: #71717a; }
  @media (prefers-color-scheme: dark) {
    :root { --bg: #0b0f14; --fg: #e8edf3; --muted: #94a3b8; }
  }
  body {
    margin: 0;
    background: var(--bg);          /* never leave this unset */
    color: var(--fg);
    font-family: ui-sans-serif, system-ui, sans-serif;
  }
</style>
</head>
<body>
  <!-- markup -->
<script>
  // behaviour — use variables or window.dokki.state, never localStorage
</script>
</body>
</html>
```

Start with `<!doctype html>`. It is what tells Dokki this is HTML and not a JSX
module — a bare fragment that contains `className=`, `{expr}`, `onClick=`, or a
self-closing `<tag/>` is classified as JSX and sent to the compiler.

### Template: Chart Dashboard

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Q1 Revenue</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<style>
  :root { color-scheme: light dark; --bg: #fff; --fg: #18181b; --muted: #71717a; --grid: #e4e4e7; }
  @media (prefers-color-scheme: dark) {
    :root { --bg: #0b0f14; --fg: #e8edf3; --muted: #94a3b8; --grid: #1f2937; }
  }
  body { margin: 0; padding: 1.5rem; font-family: ui-sans-serif, system-ui, sans-serif; background: var(--bg); color: var(--fg); }
  h2 { margin: 0 0 .25rem; font-size: 1.5rem; }
  p  { margin: 0 0 1rem; color: var(--muted); font-size: .875rem; }
  .wrap { max-width: 48rem; margin: 0 auto; }
  /* A responsive canvas fills its parent, so the parent must have a height.
     Without this the chart grows without bound and the bars fall off-screen. */
  .chart { position: relative; height: 20rem; }
</style>
</head>
<body>
  <div class="wrap">
    <h2>Q1 Revenue</h2>
    <p>Monthly revenue vs. expenses</p>
    <div class="chart"><canvas id="chart"></canvas></div>
  </div>
<script>
  const css = (name) => getComputedStyle(document.documentElement).getPropertyValue(name).trim();

  // Canvas inherits no CSS. Feed the chart the page's tokens — and do it BEFORE
  // constructing, because defaults are read at construction time.
  Chart.defaults.color = css("--muted");
  Chart.defaults.borderColor = css("--grid");

  const data = [
    { month: "Jan", revenue: 42000, expenses: 28000 },
    { month: "Feb", revenue: 51000, expenses: 31000 },
    { month: "Mar", revenue: 47000, expenses: 29000 },
  ];

  new Chart(document.getElementById("chart"), {
    type: "bar",
    data: {
      labels: data.map((d) => d.month),
      datasets: [
        { label: "Revenue", data: data.map((d) => d.revenue), backgroundColor: "#6366f1" },
        { label: "Expenses", data: data.map((d) => d.expenses), backgroundColor: "#ef4444" },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      scales: { y: { beginAtZero: true } },
    },
  });
</script>
</body>
</html>
```

### Template: Mermaid Diagram

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<title>Architecture</title>
<script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
<style>
  :root { color-scheme: light dark; --bg: #fff; }
  @media (prefers-color-scheme: dark) { :root { --bg: #0b0f14; } }
  body { margin: 0; padding: 1.5rem; background: var(--bg); font-family: ui-sans-serif, system-ui, sans-serif; }
  .mermaid { overflow-x: auto; }
</style>
</head>
<body>
  <pre class="mermaid">
graph TD
  Client -->|HTTPS| CDN
  CDN -->|Origin| App[Next.js App]
  App -->|RPC| DB[(Supabase)]
  App -->|WebSocket| Hocuspocus
  </pre>
<script>
  const dark = matchMedia("(prefers-color-scheme: dark)").matches;
  mermaid.initialize({ startOnLoad: true, theme: dark ? "dark" : "default" });
</script>
</body>
</html>
```

### Template: Interactive Widget

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Release Checklist</title>
<style>
  :root {
    color-scheme: light dark;
    --bg: #fff; --fg: #18181b; --muted: #71717a; --line: #e4e4e7;
    --chip: #f4f4f5; --done-bg: #f0fdf4; --done-line: #86efac; --done: #16a34a; --done-fg: #fff;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --bg: #0b0f14; --fg: #e8edf3; --muted: #94a3b8; --line: #1f2937;
      --chip: #172033; --done-bg: #0f2a1d; --done-line: #1f5133; --done: #34d399; --done-fg: #07130d;
    }
  }
  body { margin: 0; padding: 1.5rem; font-family: ui-sans-serif, system-ui, sans-serif; background: var(--bg); color: var(--fg); }
  .wrap { max-width: 28rem; margin: 0 auto; }
  h2 { font-size: 1.25rem; margin: 0 0 1rem; }
  ul { list-style: none; margin: 0; padding: 0; display: grid; gap: .5rem; }
  li { display: flex; align-items: center; gap: .75rem; padding: .75rem; border: 1px solid var(--line); border-radius: .5rem; cursor: pointer; transition: transform .15s; }
  li:hover { transform: translateX(4px); }
  li:focus-visible { outline: 2px solid var(--done); outline-offset: 2px; }
  li[aria-checked="true"] { background: var(--done-bg); border-color: var(--done-line); }
  li[aria-checked="true"] .label { text-decoration: line-through; color: var(--muted); }
  .dot { width: 1.5rem; height: 1.5rem; border-radius: 999px; background: var(--chip); display: grid; place-items: center; font-size: .75rem; }
  li[aria-checked="true"] .dot { background: var(--done); color: var(--done-fg); }
  @media (prefers-reduced-motion: reduce) { li { transition: none; } li:hover { transform: none; } }
</style>
</head>
<body>
  <div class="wrap">
    <h2>Release Checklist</h2>
    <ul id="list"></ul>
  </div>
<script>
  const STEPS = ["Plan", "Build", "Test", "Ship"];
  const list = document.getElementById("list");
  STEPS.forEach((step) => {
    const li = document.createElement("li");
    li.setAttribute("role", "checkbox");
    li.setAttribute("aria-checked", "false");
    li.tabIndex = 0;   // a div is not focusable on its own
    li.innerHTML = '<span class="dot">›</span><span class="label"></span>';
    li.querySelector(".label").textContent = step;
    const toggle = () => {
      const next = li.getAttribute("aria-checked") !== "true";
      li.setAttribute("aria-checked", String(next));
      li.querySelector(".dot").textContent = next ? "✓" : "›";
    };
    li.addEventListener("click", toggle);
    li.addEventListener("keydown", (e) => {
      if (e.key === " " || e.key === "Enter") { e.preventDefault(); toggle(); }
    });
    list.appendChild(li);
  });
</script>
</body>
</html>
```

---

## Live workspace data

Instead of pasting a snapshot of a table into the source, read it at render time:

```html
<script>
  (async () => {
    const { columns, rows } = await window.dokki.readTable("<resource-uuid>");
    // rows are keyed by COLUMN ID, not column name
    const revenue = columns.find((c) => c.name === "Revenue").id;
    render(rows.map((r) => Number(r[revenue] ?? 0)));
  })().catch(() => showFallback());
</script>
```

`window.dokki.readDocument(resourceId)` returns `{ title, markdown }`.

Both read as the **current viewer**, and only reach resources in the artifact's
own workspace that both the viewer and the artifact's author can read. Always
handle rejection — the viewer may lack access, and the channel is unavailable on
published pages.

---

## EDIT Mode

### Operation Decision Tree

```
How big is the change?
│
├── New page from scratch / near-total rewrite / JSX → HTML migration
│     └── edit {action:"artifact.update", resource_id, args:{source}}   (full source)
│
└── Localized change (copy tweak, colour, fix a bug)
      └── edit {action:"artifact.patch", resource_id, args:{old_string, new_string}}
```

### Always Read First

`read {action:"artifact", resource_id}` returns the **current** source — which may
differ from what you created earlier if a human edited it, and tells you whether
this artifact is HTML or legacy JSX. Reading first avoids overwriting their changes.

`artifact.patch` uses 3-way merge for concurrent-safe edits, but it needs
`old_string` to match **exactly** — including whitespace.

To see a rendered inline preview, use `preview_resource {resource_id}`.

---

## Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| `localStorage` / `sessionStorage` / cookies / `indexedDB` | Throws on the opaque origin — the **whole page** goes blank, and the write is rejected | Session state in a variable; Artifact UI state in `window.dokki.state`; business data in a table/document |
| Light-only colours | The iframe inherits no theme; half your users read dark-on-dark | Tokens on `:root`, redefined under `prefers-color-scheme: dark` |
| No `background` on `body` | A transparent body shows the host's ground, not your palette | Set `background` from a token |
| Creating a new artifact as JSX | Deprecated path; compile can fail and the author sees an error, not a page | Write a complete HTML document |
| Tailwind classes with no Tailwind | HTML artifacts get no injected CSS — the page renders unstyled | Add the Tailwind CDN tag, or write plain CSS |
| Fragment starting with `<div className=…>` | Classified as JSX and sent to the compiler | Start HTML with `<!doctype html>` and use `class=`, not `className=` |
| Assuming the app's theme applies | The iframe is isolated; it inherits nothing | Set your own colours; `@media (prefers-color-scheme: dark)` for both |
| Forgetting chart dimensions | Canvas/SVG renders at 0px | Give the container an explicit height |
| External data fetching from your own API | Only public CDNs are reachable | Inline the data, or use `window.dokki.readTable/readDocument` |
| `artifact.patch` with fuzzy match | Fails silently or edits the wrong spot | `old_string` must be exact (incl. whitespace & indentation) |
| Giant single-file artifact (>500 lines) | Slow to render, hard to edit | Keep it focused |

---

## Legacy: JSX artifacts

Only for artifacts that are **already** JSX. Do not start here.

The JSX renderer preloads React 18, Tailwind, and Mermaid. Only these imports
resolve — nothing else, no filesystem, no npm install:

```jsx
import { useState, useEffect, useMemo, useRef } from 'react';
import { motion, AnimatePresence } from 'framer-motion';
import { Check, X, ArrowRight } from 'lucide-react';   // any Lucide icon
import { LineChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer } from 'recharts';
import { Mermaid } from 'mermaid';                     // note: not 'mermaid-js'
```

The source must `export default` a component (a bare JSX element is wrapped for
you). Style with Tailwind classes only — there is no CSS pipeline. Wrap Recharts
in `<ResponsiveContainer width="100%" height={N}>` or charts render at 0px.

`window.dokki` is available here too, plus React hook wrappers
`dokki.useTable(resourceId)` / `dokki.useDocument(resourceId)`.

The storage rule applies here identically — JSX artifacts run in the same opaque
origin. Keep session state in `useState`, persistent Artifact state in
`window.dokki.state`, and never use `localStorage`.

When one of these becomes hard to keep working, port it to HTML.

## Cross-Skill Follow-Ups

- Need the data first? → `dokki-table` to create/read the table, then read it live via `window.dokki.readTable`
- Publish the artifact in a docs site? → `dokki-publish`
- Place next to related docs? → `dokki-workspace` `edit {action:"resource.move"}`
