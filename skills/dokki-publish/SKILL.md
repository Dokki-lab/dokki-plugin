---
name: dokki-publish
description: Publish Dokki content as a public site — manage published sites, publish or unpublish resources, configure custom domains and site settings.
argument-hint: <action> [workspace-or-site] [details]
allowed-tools: mcp__dokki__publish mcp__dokki__find
---

# Dokki Publish — Sites, Resources & Custom Domains

> **Server note:** publishing is now the **`publish` facade on the single `dokki` MCP server**
> (`https://dokki.one/mcp/v2`) — the old separate `dokki-publish` server (`/api/publish-mcp`) has
> been folded in and should be dropped. Resource browsing/reading come from the same server via the
> `find` and `read` facades. Every call here is `publish {action, <ids>, args:{…}}`.

## Mental Model

- Each **workspace** can have **one published site** at `dokki.one/pub/{slug}`
- `publish {action:"site.create"}` defaults to `is_active: true` if omitted; pass
  `args:{is_active: false}` when staging a site before launch.
- **Resources** (documents, tables, artifacts) are published **individually** into a site — the site is a curated subset
- Published content is a **frozen snapshot** — editing the source doesn't auto-update the public version; you must re-publish
- **Custom domains** can replace the `dokki.one/pub/...` URL (e.g., `docs.example.com`)
- **One page on its own domain** — a single resource a person published to the web
  (Share → Publish to web, `dokki.one/p/<id>`) can also answer at its own domain, at the root,
  on its own: `publish {action:"page.domain.set", resource_id, args:{domain}}`. No Site needed,
  so a one-page event page does not take the workspace's one Site.
- `publish {action:"add"}` and `share {action:"public"}` expose content publicly, so they return a
  `confirm_token` — re-send the same call with the token to execute.

---

## Action Decision Tree

```
What does the user want?
│
├── Start publishing from scratch
│     → Full Setup Flow (below)
│
├── An official website, a landing page, an event page
│     → Website Mode (below) — never a docs site for this
│     → ONE page on its own domain, no Site: Page Domain Flow (below)
│
├── Add a doc/table to an existing site
│     → publish {action:"add", site_id, resource_id}   ⚠️ confirm
│
├── Remove a doc/table from a site
│     → publish {action:"remove", site_id, resource_id}
│
├── Refresh an already-published doc after edits
│     → publish {action:"add", …} again (re-freezes snapshot)   ⚠️ confirm
│
├── Make site public / private
│     → publish {action:"site.update", site_id, args:{is_active}}
│
├── Change site slug, homepage, nav tags
│     → publish {action:"site.update", site_id, args:{settings|slug}}
│
├── Add / verify / remove custom domain
│     → a Site: publish {action:"domain.set"/"domain.status"/"domain.remove", site_id}
│     → one published page: publish {action:"page.domain.set"/"page.domain.status"/"page.domain.remove", resource_id}
│
└── "Is anything published?" / status check
      → publish {action:"site"} + publish {action:"resources", site_id}
```

---

## Full Setup Flow (New Site)

Use this when the user says "I want to publish these docs as a public site".

1. **Check existing state**
   `publish {action:"site", workspace_id}` — skip to step 3 if a site already exists.

2. **Create the site**
   ```
   publish:
     action: "site.create"
     workspace_id
     args:
       slug: "acme-docs"     # lowercase, hyphens only
       is_active: false      # explicit: start hidden until content is ready
   ```
   Save the returned `site_id`.

3. **Select resources to publish**
   If not specified, ask. Or call `find {action:"resources", workspace_id}` (via `dokki-workspace`) and let the user pick.

4. **Publish each resource**
   ```
   publish { action:"add", site_id, resource_id }   # confirm — exposes it publicly
   ```
   Loop per resource. It returns a `confirm_token` the first time — re-send with the token to
   execute. Slugs are auto-generated from names; override with `args:{slug}` if needed.

5. **Configure site settings (optional)**
   ```
   publish:
     action: "site.update"
     site_id
     args:
       settings:
         home:
           title: "Acme Documentation"
           description: "Everything about our product"
           resource_id: <id>     # optional: pin a specific doc as homepage
         nav_tags: [<tag_id>, …]  # up to 5
         site_url: "https://acme.com"  # logo click destination
   ```

6. **Flip it live**
   ```
   publish { action:"site.update", site_id, args:{ is_active: true } }
   ```

7. **(Optional) Custom domain**
   See flow below.

8. **Report**
   - Public URL: `https://dokki.one/pub/<slug>` (+ custom domain if set)
   - Number of resources published
   - Next steps (add custom domain? share URL with team?)

---

## Website Mode — official sites, landing pages, event pages

A normal site looks like documentation: page tree, titles, breadcrumbs, article
list. When the user wants something that should look like **a website**, build a
**website-mode site** instead: every page is an HTML artifact shown full-page with
no Dokki chrome, and every word, list and image on it comes from Dokki tables and
documents that a person edits later — no code.

A workspace has one site, so a customer's website gets **its own workspace**.

### Build it in this order

1. **Content first.** Put everything a person might change into tables and
   documents: a `Site copy` document (hero title, taglines, section text under
   headings), and tables such as `Speakers`, `Agenda`, `Products`, `FAQ`,
   `Pricing`. Images go in table image cells or the documents.
2. **One HTML artifact per page** (`dokki-artifact`). Read content at render time
   with the literal resource UUIDs written in the source — the publisher finds
   the ids by scanning for them:
   ```js
   const { columns, rows } = await window.dokki.readTable("<table uuid>")  // rows keyed by column id
   const { title, markdown } = await window.dokki.readDocument("<document uuid>")
   ```
   Handle a rejected read with a quiet fallback, never a blank page.
3. **Navigation lives in the pages** — a shared header/footer component, or the
   same markup in each page. Links are **relative**: `href="./"` (home),
   `href="about"`, `href="blog"`, `href="#agenda"`. Never `/pub/...`, never an
   absolute dokki.one URL: the site is served at `/site/<slug>` and at its own
   domain, and relative links work in both.
4. **Publish data before pages.** `publish add` every table/document the pages
   read, then each page artifact. A page's `add` result lists its
   `data_sources` and a `next_step` if any are not published yet — do it.
5. **Switch the site to website mode and pick the home page** (a published artifact):
   ```
   publish { action:"site.update", site_id,
             args:{ settings:{ layout:"website",
                               website:{ home_resource_id:"<home artifact id>" } } } }
   ```
6. **SEO**: `publish resource.metadata.update` with `seo_title` / `seo_description` per page.
7. **Report** the URL (`https://dokki.one/site/<slug>`, or the custom domain), and
   which table/document holds the content of which page, with this sentence:
   edits go live after **Update publish** (the button in site settings, or
   `publish add` again on the changed table/document).

### A blog or news section inside the website

- Articles are ordinary documents in one **folder**, written in the Dokki editor.
- A **template** artifact renders one article; the **list page** is an artifact
  published with the collection's path as its slug (`publish add`, `args:{slug:"blog"}`).
  ```js
  // template — /blog/<article slug>
  const { items } = await window.dokki.readCollection("<folder uuid>")
  const id = window.dokki.page?.articleId ?? items[0]?.id   // no page in the editor preview
  const article = await window.dokki.readArticle(id)       // { id, title, html, excerpt, date, cover, url }
  main.innerHTML = article.html                            // Dokki-rendered HTML: style h2, p, img, ul inside main

  // list page — /blog
  const { items } = await window.dokki.readCollection("<folder uuid>")  // [{ id, title, excerpt, date, cover, url, tags }]
  // link each item to item.url (null in the editor preview, where no article page exists yet)
  ```
- Publish every article document, the template and the list page, then
  ```
  publish { action:"site.update", site_id, args:{ settings:{ website:{
    home_resource_id:"<home>", collections:[{ path:"blog", folder_id:"<folder>", template_id:"<template>" }] } } } }
  ```
  (`website` is replaced as a whole — always send `home_resource_id` with it.)
- A new article = a new document in the folder, then `publish add` it.

### What a website page cannot do

It runs sandboxed: no cookies, `localStorage` throws (keep state in variables),
`window.dokki.state` / `call` are unavailable, and it can read only tables and
documents published on the **same site**. For sign-ups, link to a Dokki form's
public URL.

---

## Custom Domain Flow

```
publish { action:"domain.set", site_id, args:{ domain:"docs.example.com" } }
    ↓
status: pending  ← the domain is attached; TLS is issued once DNS points here
    ↓
User adds the DNS record the reply names (dns_hint)
    ↓
publish { action:"domain.status", site_id }  ← check until verified
    ↓
status: active  ← HTTPS works, the site answers at the domain
```

### Instructions to give the user

After `domain.set` (or `page.domain.set`) succeeds, pass on the record in the reply's
`dns_hint` — do not make one up:

> Domain registered with status `pending`. To activate:
> 1. Go to your DNS provider (Cloudflare, Namecheap, Route53, etc.)
> 2. Add the record: a **CNAME** from the subdomain (e.g. `docs`) to `cname.vercel-dns.com`,
>    or, for a bare domain like `example.com`, an **A** record from `@` to `76.76.21.21`
> 3. Wait a few minutes for DNS propagation (it can take up to 48 hours)
> 4. Run `publish {action:"domain.status"}` (or `page.domain.status`) to verify

A domain is used by one Site or one page, never both; a taken domain is refused.
Dokki's own hostnames (`*.dokki.one`) cannot be used.

### Page Domain Flow — one page, its own domain

For a single page (an event page, a landing page) that should live at its own domain without
a Site:

1. The person publishes the resource to the web: **Share → Publish to web**. (There is no
   agent action for this step; ask them to do it.) An artifact is served as a whole page,
   like a website-mode page; a document or table as its read-only published page.
2. `publish { action:"page.domain.set", resource_id, args:{ domain:"event.example.com" } }`
3. Pass on the `dns_hint`; then `publish { action:"page.domain.status", resource_id }`.
4. Report `https://event.example.com`. The `dokki.one/p/<id>` link keeps working;
   search engines follow the page's own "Let search engines find it" switch.

Unpublishing the page (or trashing / making the resource private) removes its domain too.
A multi-page website still needs a Site (Website Mode).

### Domain Statuses

| Status | Meaning |
|--------|---------|
| `none` | No domain configured |
| `pending` | Attached, waiting for the DNS record and TLS |
| `active` | Verified and serving HTTPS |
| `error` | Reserved; not set today |

---

## Publishing vs Sharing

Easy to confuse — they're different:

| Need | Call | Where |
|------|------|-------|
| "Send a link to Alice" | `share {action:"user", args:{email, role}}` | `dokki-workspace` |
| "Let anyone with the URL view it" | `share {action:"public", args:{public_access:"view"}}` | `dokki-workspace` |
| "Build a browsable docs website" | `publish {action:"add", site_id, resource_id}` | **this skill** |

A shared link = single-resource access, hosted on the app.
A published site = curated collection of resources on a public, navigable website (with homepage, nav, SEO, custom domain).

---

## Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| Publishing then editing, expecting auto-refresh | Published content is **frozen** — live edits invisible to the public | Re-call `publish {action:"add"}` to refresh the snapshot |
| Omitting `is_active` when creating a staging site | Defaults to live, so an empty URL may be public | Pass `args:{is_active:false}`, then flip it after publishing resources |
| Using marketing-y slug with caps/spaces | Slug is lowercased + hyphenated, may surprise user | Confirm cleaned slug with user before creating |
| Removing domain when user meant to disable site | Detaches the hostname — re-adding means waiting for DNS/TLS again | Confirm intent: disable (`site.update` is_active=false) vs `domain.remove` |
| Not waiting for SSL on custom domain | User sees cert warnings | Tell user to run `publish {action:"domain.status"}` and wait for `active` |
| Forgetting `nav_tags` are tag IDs | Passing tag names silently breaks navigation | Look up tag IDs first |
| `home.resource_id` set to a non-published resource | Homepage 404s | Ensure the homepage resource is also published |
| Ignoring the `confirm_token` on `publish {action:"add"}` | Nothing gets published | Re-send the same call with the returned token |
| Building a website or event page as a docs site | Visitors see a documentation site | Website Mode |
| Website page reads a table that is not published on the site | That section stays empty on the web | Follow the page's `next_step`: `publish add` the table/document |
| Hard-coding the words of a website page in its HTML | The customer cannot change them without code | Read every changeable word from a table or document |

## Cross-Skill Follow-Ups

- User wants to update content → `dokki-document` / `dokki-table` / `dokki-artifact`, then re-`publish {action:"add"}`
- User wants to see what's publishable → `dokki-workspace` `find {action:"resources"}`
- User wants SEO for a published doc → `publish {action:"site.update"}` + consider the doc's tags and headings
