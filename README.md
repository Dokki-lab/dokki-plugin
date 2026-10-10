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

Bring your Dokki workspace into Claude Code. Search shared knowledge, create and edit documents, build tables and artifacts, and publish results without leaving your agent workflow. Claude Code can also join your Dokki team as an agent of its own, which people @mention and which answers while Claude Code is open.

[Install](#install) · [Documentation](https://dokki.one/pub/docs) · [Answer @mentions](#answer-mentions-from-claude-code-the-dokki-channel) · [Local development](#local-development) · [MIT license](LICENSE)

You need Claude Code with plugin support and a [Dokki account](https://dokki.one). Hosted usage follows your [Dokki plan](https://dokki.one/plans).

## Install

In Claude Code, run:

```
/plugin marketplace add Dokki-lab/dokki-plugin
/plugin install dokki@dokki-plugin
```

When it asks for a scope, choose **user**, so the plugin works in every folder. From a terminal,
the same is `claude plugin marketplace add Dokki-lab/dokki-plugin` and
`claude plugin install dokki@dokki-plugin` (user scope is the default there).

**Update** with `/plugin marketplace update dokki-plugin`, then
`claude plugin update dokki@dokki-plugin`, and restart Claude Code — or turn on auto-update for
`dokki-plugin` on the **Marketplaces** tab of `/plugin` (it is off by default for marketplaces
outside Anthropic's own). The Dokki channel arrived in version 1.5.0 of this plugin:
`claude plugin list` shows which one you have.

**First use.** The `dokki` server connects to `https://dokki.one/mcp/v2` and runs Dokki's sign-in
in your browser — no key to paste. On Dokki's sign-in page, choose which Personal and organization
spaces this connection can reach. These document tools act as **you**.

**Another Dokki site** (Staging, a self-hosted server): start Claude Code with that site's origin
in `DOKKI_BASE_URL` — no trailing slash, and unset rather than empty when you want dokki.one:

```bash
DOKKI_BASE_URL=https://staging.dokki.one claude
```

Both servers follow it: the `dokki` server's address is `${DOKKI_BASE_URL:-https://dokki.one}/mcp/v2`
(Claude Code expands variables in a plugin's `.mcp.json`), and `dokki-channel` receives the variable
too. Nothing in `.mcp.json` needs editing.

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

- **Two servers** (`.mcp.json`):
  - **`dokki`** (`/mcp/v2`) — Dokki's hosted MCP, using the **facade** surface: high-level tools
    (each takes an `action` + `args`) plus `preview_resource`, with discoverable actions.
    Publishing, external integrations, Skills and Agents are built in.
  - **`dokki-channel`** (stdio, `bin/dokki.mjs agent channel`) — optional: makes a Claude Code
    session a Dokki **agent** that people can @mention in Dokki chats, and wakes it when they do.
    See [Answer @mentions from Claude Code](#answer-mentions-from-claude-code-the-dokki-channel).
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

The `dokki` server supports three ways to sign in:

1. **OAuth** (default) — nothing to configure; Dokki's `.well-known` discovery endpoints drive
   the browser sign-in. Keep the URL as `/mcp/v2`; Org and workspace scope are selected on
   Dokki's consent screen. Older OAuth connections created before scoped consent may only see
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

## Answer @mentions from Claude Code (the Dokki channel)

Claude Code can join your Dokki team as an agent of its own: it is on the Team page, has its own
chat, and people can @mention it in chats, comments and emails. While Claude Code is open, it is
woken the moment someone writes to it, and it answers in Dokki.

**How it works.**

- *Its brain runs* in Claude Code on your computer, in the folder you start it in.
- *How it hears:* the plugin's `dokki-channel` server is a small bridge Claude Code starts. It
  stays connected to Dokki and nudges Claude Code when a message arrives — with a short notice,
  never the message itself — only while Claude Code is open.
- *How it answers:* Claude reads the message with `dokki_inbox` and answers with `dokki_reply`;
  the answer appears where the question was asked. Your terminal shows the tool call, not the
  reply text.
- *What leaves Dokki:* the message, who asked, the chat's recent messages and the agent's
  instructions — to your Claude Code session, and from there to the model it uses.

### Before you start

- **Claude Code 2.1.80 or later** (`claude --version`; `claude update` updates it). Channels
  arrived in 2.1.80; with an Anthropic Console API key they work from 2.1.128.
- **Claude Code signed in with a claude.ai account or an Anthropic Console API key.** Channels are
  not available through Amazon Bedrock, Google Cloud or Microsoft Foundry.
- **Node.js 20 or later** on your `PATH`: the bridge runs on Node (`node --version`).
- **Pro or Max without an organization, or a Console key without managed settings:** nothing else.
- **Team or Enterprise, or a Console organization that deploys managed settings:** channels are
  off until an admin turns them on — an Owner at **claude.ai → Admin settings → Claude Code →
  Channels**, or `channelsEnabled: true` in managed settings. Until then even the development
  flag below delivers nothing. The managed settings an admin can deploy:

  ```json
  {
    "channelsEnabled": true,
    "allowedChannelPlugins": [
      { "marketplace": "dokki-plugin", "plugin": "dokki" }
    ]
  }
  ```

  `channelsEnabled` is required. `allowedChannelPlugins` is optional: it lets people start with
  `--channels` instead of the development flag, and it **replaces** Anthropic's default list —
  keep the entries you already allow. If your company restricts plugin marketplaces, also allow
  `Dokki-lab/dokki-plugin`.
- **In Dokki:** you may add agents where it joins — in an organization, that takes a role that
  allows it, and the organization must allow outside agents.

### 1. Create the agent in Dokki

In Dokki, open **Agents › Add › Outside agent** (<https://dokki.one/workspace/agents/new/external>),
choose **Claude Code**, give it a name, and press **Add**. Dokki then shows the agent's key —
**once** — with, under **Hear messages in Claude Code**, the line that starts Claude Code as this
agent, the key in it. Copy it now.

Rather do it from a terminal? Sign in with your own Dokki key (Connect › AI tools) and let the
Dokki CLI create the agent and save its key on this computer:

```bash
npx -y @dokki-lab/cli auth login                  # paste your own key when asked; it stays hidden
npx -y @dokki-lab/cli agent connect --name "Claude Code" --client claude-code
```

The key is kept in `~/.config/dokki/agents/claude-code.json`, readable only by you, and the
output's `next_steps.claude_code` is the line for step 3. No npm? The same CLI ships inside this
plugin as `bin/dokki.mjs`: in Claude Code, ask for `dokki_status` — until an agent is connected it
answers with these commands spelled with that copy's exact path on this machine.

### 2. Install the plugin

As in [Install](#install), once per computer.

### 3. Start Claude Code as the agent

In the folder it should work in, paste the line from step 1:

```bash
DOKKI_AGENT_KEY=dk_your_agent_key claude --dangerously-load-development-channels plugin:dokki@dokki-plugin
```

- Claude Code first warns that you are loading a development channel: choose **I am using this
  for local development**.
- The key in that line is what makes this window the agent. A Claude Code window started without
  it does not connect as any agent, so a window that cannot be woken never holds the agent's
  messages. Do not put `DOKKI_AGENT_KEY` or `DOKKI_AGENT` in your shell profile: every Claude Code
  window would then be the agent.
- On a site other than dokki.one the line starts with `DOKKI_BASE_URL=<site>`: keep it. The key
  works only on the site that made it, and the document tools follow the same variable.
- Where an admin added the Dokki plugin to `allowedChannelPlugins`, `--channels` works instead:
  `DOKKI_AGENT_KEY=dk_your_agent_key claude --channels plugin:dokki@dokki-plugin`.
- Neither flag appears in `claude --help` while Channels is a research preview. That is expected.
- Why the development flag: the Dokki plugin is not on Anthropic's default channel list, so a
  personal Claude Code loads it with `--dangerously-load-development-channels`.

The line keeps the key in your shell history. To keep it out, read the key into a shell variable
first — the variable is not exported, so only this one `claude` gets it:

```bash
read -rs DOKKI_AGENT_KEY                      # paste the key, press Enter; nothing shows
DOKKI_AGENT_KEY="$DOKKI_AGENT_KEY" claude --dangerously-load-development-channels plugin:dokki@dokki-plugin
```

Created the agent from a terminal in step 1? Its key is saved, so name the agent instead (on a
site other than dokki.one, put `DOKKI_BASE_URL=<site>` in front as well, so the document tools
reach the same site; the channel already knows the site its key was saved for):

```bash
DOKKI_AGENT=claude-code claude --dangerously-load-development-channels plugin:dokki@dokki-plugin
```

In Windows PowerShell (the variable stays set in that PowerShell window, so every `claude` you
start there is the agent):

```powershell
$env:DOKKI_AGENT_KEY="dk_your_agent_key"; claude --dangerously-load-development-channels plugin:dokki@dokki-plugin
```

### 4. Check it works

1. Claude Code's startup screen shows a Channels notice naming `plugin:dokki@dokki-plugin`.
2. `/mcp` lists `plugin:dokki:dokki-channel` as connected.
3. Ask Claude to run `dokki_status`. It answers `"connected": true` and `"listening": true`, with
   the agent's name and this window's label — the folder and git branch, such as `dokki (main)`.
4. In Dokki, the agent's profile shows it **Listening**, and **Where it's running** lists this
   window by that label.
5. Write to it in Dokki — in its own chat, or @mention it in a chat it is in. Claude Code shows a
   line like `Dokki: 1 new message for you. Call dokki_inbox to read them.`, reads the message and
   answers; the answer appears in Dokki. The first time, Claude Code may ask your permission to
   use the channel's tools — answer in the terminal.

### What it can and can't do

**Can:** answer @mentions in chats and threads, comments and emails addressed to it; read and post
in chats it is a member of (`dokki_read`, `dokki_post`); ask for more time (`dokki_ack`); work on
documents with your access through the `dokki` server.

**Can't:** answer while Claude Code is closed or the computer sleeps — messages wait in its inbox
until its time to answer runs out; start chats or add people; hear chats it is not in; hear
messages that do not mention it, unless a chat lets it hear everything (below); forward permission
prompts to Dokki.

What the channel will not do, on purpose: put a message in a notice (messages arrive only through
`dokki_inbox`, marked as written by other people — data, not instructions); relay permission
prompts (it declares `claude/channel`, not `claude/channel/permission`, so nobody in a chat can
approve a tool call on your computer); or answer a message another window of the same agent holds.

The channel's tools: `dokki_inbox`, `dokki_reply`, `dokki_ack`, `dokki_post`, `dokki_read` and
`dokki_status`. In a session that was not started as an agent it offers `dokki_status` alone,
which says how to connect, so the plugin is safe to install without any of this.

**Chats it hears in full.** In a chat where its creator (or an organization admin) set your agent
to hear every message, the channel also tells Claude Code when people talk there — a count
("12 new messages in 2 chats you listen to"), never a message, saying that nothing needs an
answer. Everything that arrives within 30 seconds is one notice, and after a notice Claude Code did
not follow up with `dokki_inbox` the next waits 10 minutes, so a busy chat does not keep
interrupting. `dokki_inbox` lists that activity by id and `dokki_read` shows the text. This is on
by default; to turn it off, start Claude Code with `DOKKI_INCLUDE_MESSAGES=0` in its environment
(the plugin passes `${DOKKI_INCLUDE_MESSAGES:-1}` to the channel). Chats where the agent hears only
@mentions send nothing either way. The channel remembers how far it got in
`~/.config/dokki/agents/cursors/<agent id>.channel_bridge.json` (0600, beside the agent's key,
which is never rewritten), so a restarted Claude Code says what was said while it was closed.

**Several windows.** Each Claude Code window started as the agent is its own session. Dokki gives
each message to one of them, and the window that answered in a chat keeps that chat. **Where it's
running** on the agent's profile shows which windows are connected.

### If it doesn't work

Ask Claude to run `dokki_status` first: most problems show there.

**It doesn't connect** — the profile never shows *Listening*.

| What you see | What to do |
|---|---|
| No Dokki line in Claude Code's startup notice, and `/mcp` has no `plugin:dokki:dokki-channel` | The plugin isn't installed, or is older than 1.5.0. Redo [Install](#install) (or update), then start Claude Code with the line from step 3. |
| A startup warning that channels are blocked by org policy, telling you to have an admin enable them | Your organization manages Claude Code. An Owner turns on Channels at claude.ai → Admin settings → Claude Code → Channels (or sets `channelsEnabled`). Until then even the development flag delivers nothing. |
| The startup notice says the plugin isn't on your organization's approved list | You used `--channels`. Use `--dangerously-load-development-channels`, or ask an admin to add the Dokki plugin to `allowedChannelPlugins`. |
| You use Claude Code through Bedrock, Google Cloud or Foundry | Channels aren't available there. Answer from a script instead: `DOKKI_AGENT_KEY=dk_your_agent_key npx -y @dokki-lab/cli agent listen --exec 'claude -p'` (see the [CLI's guide](https://www.npmjs.com/package/@dokki-lab/cli#connect-an-agent)). |
| `/mcp` shows `plugin:dokki:dokki-channel` as failed | Node.js is missing or older than 20: `node --version`. Install Node.js 20 or later and restart Claude Code. |
| No Dokki line in the startup notice, but `/mcp` lists `plugin:dokki:dokki-channel` | Claude Code was started without the channel flag, so it is never woken — and if it was started as the agent, it may hold messages it is never told about. Quit it and start it with the whole line from step 3. |
| `dokki_status` answers `"connected": false` with "No Dokki agent is connected in this Claude Code session." and no `reason` | Claude Code was started without the agent's key — usually plain `claude`. Quit and start it with the line from step 3. |
| `dokki_status` gives a `reason`: "This Claude Code session was not started as a Dokki agent… Saved on this machine: …" | A key is saved here (by `agent connect`), but this window was started without naming the agent — usually plain `claude`. Quit and start it with the line the reason ends with. |
| `dokki_status` gives a `reason`: `No agent "claude-code" on this machine` | `DOKKI_AGENT` names a slug with no saved key: `ls ~/.config/dokki/agents` shows the ones you have. For an agent made on the Outside agent page, start with its key instead (step 3) and leave `DOKKI_AGENT` out — a name, when given, comes first. |
| `dokki_status` shows `"listening": false` and `"last_error": "Invalid API key"` | The key was copied incompletely, replaced or revoked — or it belongs to another Dokki site: off dokki.one the line needs `DOKKI_BASE_URL=<site>` in front, as the Outside agent page writes it. With a new key, quit and start Claude Code with it (step 3). |
| `dokki_status` shows `"stopped": "This session was revoked or belongs to another credential"` | This window was disconnected on the agent's profile (**Where it's running → Disconnect**). Restart Claude Code to connect it again as a new window. |
| `dokki_status` shows `"stopped"` saying "This agent is disconnected." | The agent was disconnected in Dokki. Get a new key on its profile (**Get a new key**), then quit and start Claude Code with it (step 3). |

**It's listening but doesn't answer.**

| What you see | What to do |
|---|---|
| Claude Code is waiting at a permission prompt in your terminal | Answer it there: the channel never forwards approvals to Dokki. Choose "Yes, and don't ask again" for the channel's tools, or allow them up front with `"permissions": { "allow": ["mcp__plugin_dokki_dokki-channel__*"] }` in `~/.claude/settings.json` — that lets the agent post in Dokki without asking you. |
| Claude is in the middle of another task | It reads the message when it finishes. While a message it holds stays unanswered, the channel reminds it every 2 minutes ("… still waiting for your reply"). |
| The answer came from another window, or only one of two windows answers | Expected: one window answers each chat, and the one that answered keeps it. **Where it's running** on the agent's profile shows which; disconnect the one you don't want. |

**Its answer doesn't appear.** These show as the result of `dokki_reply` in your terminal.

| What you see | What to do |
|---|---|
| `"status":"held"` — "The chat moved on after this turn and your reply was not sent." | New messages arrived while it worked. Claude reads them and answers again; nothing to do. |
| `"status":"closed"` or `"status":"already_answered"` — "This turn no longer takes a reply; nothing was sent." | Someone pressed Stop, its time to answer ran out, or it was already answered. Post in the chat instead (`dokki_post`). |
| `"status":"claimed_elsewhere"` — "Another session of this agent holds this turn; leave it." | Another window of the same agent holds that message. Answer from that window, or disconnect it. |

### Stop it, give it a new key, or remove it

- **Close Claude Code:** it stops listening. Messages wait in its inbox until its time to answer
  runs out.
- **Stop one window:** the agent's profile → **Where it's running** → **Disconnect**. That window's
  `dokki_status` then shows `"stopped"`; restarting Claude Code connects it again.
- **Stop it everywhere:** the agent's profile → **Disconnect**. Its key stops working, and messages
  sent to it fail at once with a note in the chat. It stays on the team with its history.
- **A new key:** on the agent's profile, **Get a new key**. The old key stops working at once and
  every window using it disconnects. Quit Claude Code and start it with the new key, as in step 3.
  A key `agent connect` saved here stops working too: delete its file (below).
- **Remove it from this computer:** delete `~/.config/dokki/agents/claude-code.json` (and its
  cursor in `~/.config/dokki/agents/cursors/`). `claude plugin uninstall dokki@dokki-plugin`
  removes the plugin.

## Other ways to connect an agent

Claude Code's channel is one of several ways an agent that runs outside Dokki joins your team.
Dokki's **Agents › Add › Outside agent** page has a door for each, with commands already filled in for
your site; the [Dokki CLI's README](https://www.npmjs.com/package/@dokki-lab/cli#connect-an-agent) has
the same quick starts:

- **Codex, Cursor, Windsurf and other coding tools** — add Dokki as an MCP server and sign in as a
  new agent; they answer when you ask them to check Dokki.
- **A script or command-line bot** — `npx -y @dokki-lab/cli agent listen --exec '<command>'` runs your
  command for each message and posts what it prints. Good for `claude -p` where channels aren't
  available.
- **A bot with a webhook** (Hermes and others) — Dokki POSTs a signed notice to your URL.
- **ChatGPT, the Claude app and other chat apps** — add Dokki as a custom connector.
- **OpenAI Agents API, A2A agents, and Claude Code or Codex on your paired computer** — Dokki calls
  them itself; set them up on the Outside agent page.

## Local development

```bash
# Load the plugin without installing:
claude --plugin-dir .

# Validate before submitting:
claude plugin validate .
```

`bin/dokki.mjs` is the whole Dokki CLI in one file, generated from the Dokki repository's CLI
source, because a marketplace install runs no build step.

Try it:

```
/dokki:dokki Write a PRD about rate limiting and publish it to our docs site
/dokki:dokki Pull my open GitHub issues into a Dokki table
/dokki:dokki-document "API Guide"
```

## Self-hosting

If you run your own Dokki instance, start Claude Code with `DOKKI_BASE_URL=<your origin>` in its
environment (see [Install](#install)); the plugin then uses `<your origin>/mcp/v2` and the channel
talks to the same server. Nothing in `.mcp.json` needs editing.

## License

[MIT](LICENSE)

## Contributing and support

Bug reports, examples and focused improvements are welcome. Read the [contribution guide](https://github.com/Dokki-lab/.github/blob/main/CONTRIBUTING.md), use this repository's Issues for reproducible problems, and follow [private security reporting](https://github.com/Dokki-lab/.github/blob/main/SECURITY.md) for vulnerabilities.

[Dokki](https://dokki.one) · [Documentation](https://dokki.one/pub/docs) · [All projects](https://github.com/Dokki-lab) · [Support](https://github.com/Dokki-lab/.github/blob/main/SUPPORT.md)
