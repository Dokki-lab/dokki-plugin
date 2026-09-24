---
name: dokki-app
description: Build a multi-page Dokki App — Screens, reusable Components, and data bound to a workspace Table — then validate, release and install it. Use when one artifact is not enough and the result needs routes, a shared component library, or write access to workspace data. Starts with create app.
argument-hint: <what-the-app-should-do>
allowed-tools: mcp__dokki__create mcp__dokki__read mcp__dokki__edit mcp__dokki__publish
---

# Dokki App — Screens, Components, and bound data

A Dokki **App** is a Folder that holds **Screens** (pages, each at a route) and
**Components** (reusable UI), plus a manifest saying which is which, what routes
they answer, and which workspace data the App may touch. Publishing freezes all
of it into an immutable **release**; installing that release is what actually
**grants** the data access.

```
create app                                 ← the package
  └── create artifact {artifact_variant:"component"}   × N   ← your design system
  └── create artifact {artifact_variant:"screen"}      × N   ← the pages
publish app.manifest   ← register them, give Screens routes, bind a Table
publish app.validate   ← get content_digest (and catch mistakes)
publish app.release    ← freeze
publish app.install    ← approve capabilities; the App is now live at /app/<id>/run
```

## Is an App the right thing?

| Need | Use |
|---|---|
| One self-contained page or widget | `dokki-artifact` |
| Rows and columns people edit | `dokki-table` |
| Several pages, shared components, reads/writes a Table | **App** |

---

## The five rules that decide whether your App works

Get any of these wrong and the App still renders — it just shows nothing, with
no error. Read them before writing a line.

### 1. A Screen must `export default`

The renderer mounts the default export and nothing else. Its own error message
is *"No default export found."*

```jsx
export default function Overview() { … }   // ✅ Screen
export function Overview() { … }           // ❌ renders nothing
```

Components are the opposite: they use **named** exports, because Screens import
them by name.

### 2. Data comes from `window.dokki.app`, never from `fetch`

The App runs in a sandboxed frame with no cookies; it cannot call the Dokki API
itself. The host does it, under the viewer's own permissions:

```js
await window.dokki.app.read(alias)                                  // → {columns, rows, total}
await window.dokki.app.create(alias, values)
await window.dokki.app.update(alias, rowId, changes, expectedVersion)
await window.dokki.app.delete(alias, rowId, expectedVersion)
await window.dokki.app.invoke(alias, payload)                       // workspace endpoint
window.dokki.app.navigate("/transactions")                          // route change
```

`alias` is the name you gave the binding in the manifest — not a resource id.

A Screen also has a link of its own: the entry Screen is `/app/<id>/run` and
every other route is `/app/<id>/run<route>` — `/app/<id>/run/transactions`. So
`navigate` and a pasted URL land on the same Screen.

### 3. Cells are keyed by COLUMN ID, not by column name

This is the one that fails silently.

```js
const { columns, rows } = await window.dokki.app.read("transactions")
// columns → [{ id: "col_x7f2", name: "Merchant", type: "text" }, …]
// rows    → [{ id: "row_001", version: 1, col_x7f2: "Amazon", … }]

row.Merchant   // ❌ undefined — table renders, every cell empty, no error
row[byName.Merchant]  // ✅
```

Build the map once and re-key the rows:

```js
const byName = Object.fromEntries(columns.map((c) => [c.name, c.id]))
const items = rows.map((row) => {
  const out = { id: row.id, version: row.version }
  for (const col of columns) out[col.name] = row[col.id]
  return out
})
```

### 4. Every write carries the `version` it was read at

`update` and `delete` are compare-and-set. Send the row's `version`; if someone
changed that row first, the write is **refused** rather than overwriting them.

```js
try {
  await window.dokki.app.update("transactions", item.id, { [byName.Status]: "approved" }, item.version)
} catch (error) {
  // error.code === "conflict" — re-read before telling the person anything.
  await reload()
}
```

Re-read on failure. A message that says "refreshed" while the screen still shows
the stale row is worse than the conflict.

### 5. A Screen imports a Component by resource id

```jsx
import { StatCard } from "dokki:component/id:8f2c…-uuid"
```

Use the **id** form. The slug form breaks when two artifacts slug alike, and a
rename silently unresolves it. Never paste a Component's source into a Screen —
that is what the Component is for.

---

## Build order

**1 · The package**

```
create {action:"app", workspace_id, args:{ name:"Spend" }}
```

Screens and Components must be created **inside** this Folder (`parent_id` = the
App's resource id) — the manifest only offers what lives there.

**2 · Components first**

```
create {action:"artifact", workspace_id, parent_id:<app id>,
        args:{ name:"StatCard", artifact_variant:"component", source:"export function StatCard({…}){…}" }}
```

Build the shared layer before the pages: tokens/colours, then presentational
pieces, then a data hook. Four screens sharing six components beats four screens
each with their own table markup.

**3 · Screens**

```
create {action:"artifact", workspace_id, parent_id:<app id>,
        args:{ name:"Overview", artifact_variant:"screen", source:"export default function Overview(){…}" }}
```

**4 · Wire the manifest**

```
publish {action:"app.manifest", resource_id:<app id>, args:{ changes:[
  { add_screen:    { resource_id:<screen id>, route:"/" } },
  { add_screen:    { resource_id:<screen id>, route:"/transactions" } },
  { add_component: { resource_id:<component id>, export_name:"StatCard" } },
  { add_binding:   { alias:"transactions", resource_type:"table",
                     resource_id:<table id>, operations:["read","update"] } }
]}}
```

Routes are lowercase, slash-prefixed, no trailing slash. The first Screen added
becomes the entry screen unless you set another with `set_entry_screen`.

**5 · Validate, release, install**

```
publish {action:"app.validate", resource_id:<app id>}          → content_digest
publish {action:"app.release",  resource_id:<app id>, args:{ expected_content_digest:"…" }}
publish {action:"app.install",  resource_id:<app id>, args:{ release_id:"…",
         approved_capabilities:{ bindings:[{ alias:"transactions", resourceType:"table",
                                             resourceId:"…", operations:["read","update"] }] } }}
```

**`approved_capabilities` is never implicit.** A release *declares* what it
wants; the installation *grants* what it gets. Omit it and the App installs able
to read nothing. Grant the narrowest set that works.

---

## A Screen that actually reads and writes

```jsx
import { useCallback, useEffect, useState } from "react"
import { StatusPill } from "dokki:component/id:<component uuid>"

export default function Approvals() {
  const [state, setState] = useState({ items: [], byName: {}, note: null })

  const load = useCallback(async () => {
    const { columns, rows } = await window.dokki.app.read("transactions")
    const byName = Object.fromEntries(columns.map((c) => [c.name, c.id]))
    const items = rows.map((row) => {
      const out = { id: row.id, version: row.version }
      for (const col of columns) out[col.name] = row[col.id]
      return out
    })
    setState((s) => ({ ...s, items, byName }))
  }, [])

  useEffect(() => { void load() }, [load])

  async function approve(item) {
    try {
      await window.dokki.app.update(
        "transactions", item.id, { [state.byName.Status]: "approved" }, item.version,
      )
      await load()
      setState((s) => ({ ...s, note: `${item.Merchant} approved.` }))
    } catch (error) {
      await load()   // someone decided it first — show what is true now
      setState((s) => ({ ...s, note: `${item.Merchant} was not approved — ${error.message}` }))
    }
  }

  return (
    <div>
      {state.note ? <p>{state.note}</p> : null}
      {state.items.filter((t) => t.Status === "pending").map((t) => (
        <div key={t.id}>
          <span>{t.Merchant}</span>
          <StatusPill status={t.Status} />
          <button onClick={() => approve(t)}>Approve</button>
        </div>
      ))}
    </div>
  )
}
```

---

## When something looks wrong

| Symptom | Cause |
|---|---|
| Page is blank, no error | Screen has no `export default` |
| Table renders, every cell empty | Reading `row.ColumnName` instead of `row[columnId]` — rule 3 |
| Write throws `forbidden` | The operation is declared but was not **approved** at install |
| Write throws `conflict` | Stale `version` — re-read, then retry or tell the person |
| Import of a Component fails | Wrong id, or the Component is not registered in the manifest |
| Nav does nothing | Route is not in the manifest, or not lowercase/slash-prefixed |
| Runtime says "not installed or enabled" | Published but never installed, or the installation is disabled |

## Iterating

Editing a Screen changes the **draft**; the installed App keeps running the
release it was installed with. To ship a change: `edit artifact.update`, then
validate → release → install again. That is a feature — a half-finished edit
cannot break an App other people are using.

## Limits

128 Screens · 256 Components · 64 bindings · 32 network domains · 500 resources
under the App Folder · 100 fields per write.
