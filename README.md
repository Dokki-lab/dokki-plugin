# Dokki for Claude Code

> **[Dokki](https://dokki.one) — agent-native collaboration OS.** The workspace where AI agents
> work like teammates, reading, writing, reviewing, and publishing alongside people. This plugin
> brings it into Claude Code — create and edit documents, build tables and artifacts, publish
> public sites, connect 1000+ external apps, and search everything you've written, without
> leaving the terminal.

This plugin bundles:

- **One connector** (`.mcp.json`) — Dokki's hosted MCP, using the **facade** surface:
  - **`dokki`** (`/mcp/v2`) — **10 high-level tools** (each takes an `action` + `args`) plus
    `preview_resource`, instead of ~39 flat tools. Fewer tools = the model picks the right one
    and spends fewer tokens. Publishing, external integrations, Skills and Agents are built in.
- **Ten skills** that orchestrate those tools into real workflows (routing, decision trees,
  templates), so natural-language requests map to the right action sequence.

The `dokki` facade tools:

| Tool | What it does |
|------|--------------|
| `find` | list workspaces/resources/Automations/node types, semantic search, exact grep, knowledge-graph (tag/type/date filters) |
| `read` | read a document (`view`/`outline`/`edit` modes + pagination), table (`where`/`sort`/`columns`/paging), artifact, file |
| `create` | workspace, folder, document, table, artifact, Dokki App, Automation, file upload |
| `edit` | rename/move/tag/delete (set an emoji **or Lucide icon**), doc/table/artifact edits (op-arrays, markdown, anchor/section targeting, header-name columns) |
| `share` | share with a user, or set public access |
| `message` | a workspace channel for human confirmations & notifications |
| `skills` | folder-backed Skills: draft, validate, publish, install, bind to an Agent |
| `publish` | publish/unpublish resources to a public site + custom domains |
| `connect` | **connect & use 1000+ external integrations** (GitHub, Slack, Gmail, Notion, Google Workspace, Linear, …) through Dokki |
| `agent` | manage and run your Dokki Agents (roster, teams, schedules); query and cancel delegated runs |
| `preview_resource` | inline rendered preview of a doc/table/artifact |

The facade is **self-teaching**: call a tool with no `action` to list its actions; a partial
action returns the subtree; missing args return a hint with an example. Dangerous actions
(`edit resource.delete`, `share public`, `publish add`, a `table.edit` column delete) return a
`confirm_token` you re-send to proceed.

| Skill | Command | Scope |
|-------|---------|-------|
| Entry / router | `/dokki:dokki <intent>` | Interprets intent, routes to the right skill, orchestrates multi-skill workflows |
| Workspace | `/dokki:dokki-workspace` | Browse, search, organize, coordinate, connect — `find`, `edit resource.*`, `share`, `message`, `connect` (1000+ integrations), upload |
| Document | `/dokki:dokki-document` | Rich-text docs and inline images — `create doc`, `read doc`, `edit doc.edit`/`doc.rewrite` |
| Table | `/dokki:dokki-table` | Structured data — `create table`, `read table`, `edit table.edit` |
| Artifact | `/dokki:dokki-artifact` | HTML or JSX artifacts, charts, interactive UI — `create artifact`, `edit artifact.*` |
| Publish | `/dokki:dokki-publish` | Public sites & custom domains — `publish site/add/remove/domain.*` |
| Automation | `/dokki:dokki-automation` | Versioned visual workflows — list/read/create/revise/run/history/delete |
| App | `/dokki:dokki-app` | Multi-page Dokki Apps — Screens, reusable Components, data bound to a Table; validate, release, install |
| Ad film | `/dokki:ad-film-production` | One-line brief → a generatable shot table: proposition, three-second hook, per-shot video prompts, continuity |
| Sales call debrief | `/dokki:sales-call-debrief` | Spoken meeting recap → evidence-backed report with account history, stage/BANT, and only the material gaps |

## Install

From this repository:

```
/plugin marketplace add Dokki-lab/dokki-plugin
/plugin install dokki@dokki-plugin
```

Update later with `/plugin marketplace update dokki-plugin`.

On first use, Claude Code connects to `https://dokki.one/mcp/v2`, then runs Dokki's OAuth flow
in your browser — no API key to paste. During OAuth, choose the Personal and Org workspaces this
MCP connection can access.

## Authentication

The server supports three auth methods:

1. **OAuth** (default) — nothing to configure; Dokki's `.well-known` discovery endpoints drive
   the browser sign-in. Keep the URL as `/mcp/v2`; Org and workspace scope are selected in the
   Dokki consent screen. Older OAuth connections created before scoped consent may only see
   Personal workspaces until you reconnect. (External integrations connected via `connect` are
   scoped to your user account.)
2. **API key** — set an `Authorization: Bearer dk_...` header if you prefer static credentials.
   A `dk_...` key is scoped to Personal or a single Org; use separate server entries/keys when
   you need multiple Org scopes:
   ```json
   {
     "mcpServers": {
       "dokki": {
         "type": "http",
         "url": "https://dokki.one/mcp/v2",
         "headers": { "Authorization": "Bearer dk_your_api_key_here" }
       }
     }
   }
   ```
3. **Connector token** — workspace-scoped tokens for embedded/integration use.

`https://dokki.one/api/mcp` remains as a compatibility alias for existing configurations and
serves the same facade tools as `/mcp/v2`; it is not a separate flat surface. The fixed-Org
`https://dokki.one/api/mcp/org/<orgId>` endpoint remains for older integrations. New
configurations should always use `https://dokki.one/mcp/v2`.

## Local development

```bash
# Load the plugin without installing:
claude --plugin-dir .

# Validate before submitting:
claude plugin validate .
```

Try it:

```
/dokki:dokki Write a PRD about rate limiting and publish it to our docs site
/dokki:dokki Pull my open GitHub issues into a Dokki table
/dokki:dokki-document "API Guide"
```

## Self-hosting

If you run your own Dokki instance, change the `url` in `.mcp.json` to your origin's `/mcp/v2`
endpoint.

## License

[MIT](LICENSE)
