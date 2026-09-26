<p>
  <a href="https://dokki.one">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/Dokki-lab/.github/main/assets/dokki-dark.svg">
      <img src="https://raw.githubusercontent.com/Dokki-lab/.github/main/assets/dokki-light.svg" alt="Dokki" width="180">
    </picture>
  </a>
</p>

# Dokki for Claude Code

**You lead. Agents do the work.** [Dokki](https://dokki.one) brings your team, agents and work into one workspace—from the first goal to shared, editable results.

Bring your Dokki workspace into Claude Code. Search shared knowledge, create and edit documents, build tables and artifacts, and publish results without leaving your agent workflow.

[Install](#install) · [Documentation](https://dokki.one/pub/docs) · [Local development](#local-development) · [MIT license](LICENSE)

You need Claude Code with plugin support and a [Dokki account](https://dokki.one). Hosted usage follows your [Dokki plan](https://dokki.one/plans).

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

## Your first result

Start with a read-only request:

```text
/dokki:dokki List my workspaces and show the resources in the workspace I choose.
```

Then try a small document:

```text
/dokki:dokki-document Create a Launch brief in my chosen workspace with audience, message and next steps. Return its link.
```

Open the returned document in Dokki to review and edit it. Publishing is a separate action and should be requested explicitly.

## Included workflows

This plugin bundles:

- **One connector** (`.mcp.json`) — Dokki's hosted MCP, using the **facade** surface:
  - **`dokki`** (`/mcp/v2`) — high-level tools (each takes an `action` + `args`) plus
    `preview_resource`, with discoverable actions. Publishing, external integrations, Skills and Agents are built in.
- **Focused skills** that orchestrate those tools into real workflows (routing, decision trees,
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
| `connect` | **connect & use external integrations** (GitHub, Slack, Gmail, Notion, Google Workspace, Linear, …) through Dokki |
| `agent` | manage and run your Dokki Agents (roster, teams, schedules); query and cancel delegated runs |
| `preview_resource` | inline rendered preview of a doc/table/artifact |

The facade is **self-teaching**: call a tool with no `action` to list its actions; a partial
action returns the subtree; missing args return a hint with an example. Dangerous actions
(`edit resource.delete`, `share public`, `publish add`, a `table.edit` column delete) return a
`confirm_token` you re-send to proceed.

| Skill | Command | Scope |
|-------|---------|-------|
| Entry / router | `/dokki:dokki <intent>` | Interprets intent, routes to the right skill, orchestrates multi-skill workflows |
| Workspace | `/dokki:dokki-workspace` | Browse, search, organize, coordinate, connect — `find`, `edit resource.*`, `share`, `message`, `connect` (external integrations), upload |
| Document | `/dokki:dokki-document` | Rich-text docs and inline images — `create doc`, `read doc`, `edit doc.edit`/`doc.rewrite` |
| Table | `/dokki:dokki-table` | Structured data — `create table`, `read table`, `edit table.edit` |
| Artifact | `/dokki:dokki-artifact` | HTML or JSX artifacts, charts, interactive UI — `create artifact`, `edit artifact.*` |
| Publish | `/dokki:dokki-publish` | Public sites & custom domains — `publish site/add/remove/domain.*` |
| Automation | `/dokki:dokki-automation` | Versioned visual workflows — list/read/create/revise/run/history/delete |
| App | `/dokki:dokki-app` | Multi-page Dokki Apps — Screens, reusable Components, data bound to a Table; validate, release, install |
| Ad film | `/dokki:ad-film-production` | One-line brief → a generatable shot table: proposition, three-second hook, per-shot video prompts, continuity |
| Sales call debrief | `/dokki:sales-call-debrief` | Spoken meeting recap → evidence-backed report with account history, stage/BANT, and only the material gaps |

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

## Contributing and support

Bug reports, examples and focused improvements are welcome. Read the [contribution guide](https://github.com/Dokki-lab/.github/blob/main/CONTRIBUTING.md), use this repository's Issues for reproducible problems, and follow [private security reporting](https://github.com/Dokki-lab/.github/blob/main/SECURITY.md) for vulnerabilities.

[Dokki](https://dokki.one) · [Documentation](https://dokki.one/pub/docs) · [All projects](https://github.com/Dokki-lab) · [Support](https://github.com/Dokki-lab/.github/blob/main/SUPPORT.md)
