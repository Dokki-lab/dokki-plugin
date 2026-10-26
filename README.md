<p>
  <a href="https://dokki.one">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/Dokki-lab/.github/main/assets/dokki-dark.svg">
      <img src="https://raw.githubusercontent.com/Dokki-lab/.github/main/assets/dokki-light.svg" alt="Dokki" width="180">
    </picture>
  </a>
</p>

# Dokki for Claude

**You lead. Agents do the work.** [Dokki](https://dokki.one) brings your team, agents and work into one workspace—from the first goal to shared, editable results.

This plugin brings your Dokki workspaces into Claude, Cowork and Claude Code. Search shared knowledge, create and edit documents, build tables, artifacts, Apps and Automations, and publish results, all from a conversation.

It works inside your Dokki workspaces only. It does not act in your other accounts (GitHub, Slack, Gmail and so on), does not run Dokki Agents, and runs nothing on your computer: everything goes through Dokki's hosted connector.

[Documentation](https://dokki.one/pub/docs) · [Privacy policy](https://dokki.one/privacy) · [Terms](https://dokki.one/terms) · [Support](https://dokki.one/support) · [MIT license](LICENSE)

You need a [Dokki account](https://dokki.one). Hosted usage follows your [Dokki plan](https://dokki.one/plans).

## Get started

Add the plugin from the Claude plugin directory, then connect its **Dokki** connector. Connecting opens Dokki's sign-in in your browser; on Dokki's consent screen, choose which Personal and organization spaces this connection can reach. The tools act as **you**, with exactly the access you have in those spaces. There is nothing to paste and no key to keep.

Start with a read-only request, such as asking Claude to list your Dokki workspaces and show the resources in the one you pick. Then ask for something small: a launch brief with audience, message and next steps in the workspace you chose. Open the returned link in Dokki to review and edit it. Publishing is a separate action that Claude asks you to confirm.

## What it includes

- **One connector**, Dokki's hosted MCP server at `https://dokki.one/mcp/v2/directory`. Its tools are `find`, `read`, `create`, `edit`, `share`, `message`, `skills`, `publish` and `preview_resource`; each takes an action, and calling one without an action lists what it can do. Deleting, making something public and publishing return a confirmation token that has to be sent back before anything happens. Automations made or run through this connector can't call connected apps, send email, start Agents or generate video.
- **Eight skills** that turn requests into the right sequence of those tools:

| Skill | What it covers |
|-------|----------------|
| `dokki` | Reads the request, picks the right skill and combines several when a task needs them |
| `dokki-workspace` | Browse, search, organize and coordinate: workspaces, resources, tags, sharing, messages, uploads |
| `dokki-document` | Rich-text documents and inline images |
| `dokki-table` | Structured data in tables |
| `dokki-artifact` | HTML or JSX artifacts, charts and interactive pages |
| `dokki-publish` | Public sites and custom domains |
| `dokki-automation` | Versioned visual workflows: list, read, create, revise, run, history |
| `dokki-app` | Multi-page Dokki Apps with screens, components and table-bound data |

## Data

The plugin sends what you ask Claude to do in Dokki—searches, the content of documents, tables and artifacts you create or change, and the resources you read—to your Dokki account through `dokki.one`. It stores nothing itself and sends nothing anywhere else. Dokki's [privacy policy](https://dokki.one/privacy) covers how Dokki keeps that data.

## Claude Code: answer @mentions as a Dokki agent

Claude Code can also join your Dokki team as an agent that people @mention. That needs a small local bridge, so it ships in a separate build of this plugin for Claude Code only, on this repository's [`claude-code` branch](https://github.com/Dokki-lab/dokki-plugin/tree/claude-code#answer-mentions-from-claude-code-the-dokki-channel), which also explains how to install it.

## License

[MIT](LICENSE)

## Contributing and support

Bug reports, examples and focused improvements are welcome. Read the [contribution guide](https://github.com/Dokki-lab/.github/blob/main/CONTRIBUTING.md), use this repository's Issues for reproducible problems, and follow [private security reporting](https://github.com/Dokki-lab/.github/blob/main/SECURITY.md) for vulnerabilities.

[Dokki](https://dokki.one) · [Documentation](https://dokki.one/pub/docs) · [All projects](https://github.com/Dokki-lab) · [Support](https://github.com/Dokki-lab/.github/blob/main/SUPPORT.md)
