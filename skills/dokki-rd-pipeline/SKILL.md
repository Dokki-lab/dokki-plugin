---
name: dokki-rd-pipeline
description: Work alongside the Agents of a Dokki Autonomous R&D workspace (the "Autonomous R&D" template) from a coding agent such as Claude Code or Codex. Use when the person asks you to pick up, fix, or hand off an Issue from their Dokki Issues table, to file a bug or request into its Inbox, or to check what the pipeline is doing — so you never duplicate or overwrite work the PM, Lead, Developer or Acceptance Tester Agents are doing.
allowed-tools: mcp__dokki__find mcp__dokki__read mcp__dokki__edit mcp__dokki__create mcp__dokki__agent
license: Proprietary
metadata:
  author: Dokki
  version: "1.0.0"
  protocol: dokki-rd-pipeline@1
---

# Dokki R&D pipeline

A workspace made from the **Autonomous R&D** template runs R&D with Agents: the **PM** triages each new **Inbox** row into an **Issue**, the **Lead** assigns it, a **Developer** opens the pull request, the **Acceptance Tester** checks it on staging and in production, and the **Release Manager** asks a person before production. People and their coding agents (you) work in the same tables and the same repository. These rules keep you from doing someone's work twice or undoing it.

Start by finding the workspace's tables: `find {action:"resources", workspace_id:"<workspace id>", args:{depth:1}}` lists Inbox, Issues, Projects and Pipeline board (`find {action:"workspaces"}` gives the workspace id). Never read a whole table: filter it.

## The columns that matter

| Table | Column | Who writes it |
|---|---|---|
| Issues | **Key** | the table (read-only) |
| Issues | **Status** Planned → In Progress → Staging → Released | the code (a pull request naming the key moves it). People and Agents set only Backlog, Blocked, Done, Canceled |
| Issues | **Owner** | the person who answers for the Issue (from its Project) |
| Issues | **Agent** | whoever is doing the work now. Setting it to an Agent (`agent:<id>`) **starts that Agent** |
| Issues | **Next Action** | the hand-off between people and Agents: who does what next |
| Issues | **Evidence** | append-only: one new line per check, never edit an old line |
| Projects | **Owner**, **Delegation** ("Pipeline may take it" / "Ask the owner first"), **Areas** (code paths) | the people who own each part |

## Before you start on an Issue

1. Read the one row: `read {action:"table", resource_id:<Issues>, args:{where:[{column:"Key", op:"eq", value:"<KEY>"}]}}`.
2. **Is someone already on it?** If **Agent** names an Agent and Status is Planned or In Progress, the pipeline is working on it. If **PR** lists an open pull request, or a branch in the repository carries the key, someone is on it. Do not start: tell the person who it is and ask whether to take over.
3. **Who owns it?** If its Project (or the code it touches, by the Project's Areas) is "Ask the owner first" and the person you work for is not the owner, ask before you change anything.
4. **Say you have it.** Write in **Next Action**: "<person> is working on it with <your tool> in branch <branch>". Leave **Agent** empty while a person works on it, so no Agent is started.

Edit a cell with `edit {action:"table.cells.update", resource_id:<Issues>, args:{updates:[{rowId:"<row id>", columnId:"<column id or header>", value:"..."}]}}` — the row id goes inside each update.

## While you work

- Branch name starts with the key in lower case: `<key>-<short description>`.
- The pull request's description starts with `Issue: <KEY>`. That links it to the Issue and moves Status for you; do not set Status by hand.
- Before you open the pull request, run the collision check below. Two changes to the same file are normal; only a real conflict stops you.
- Never write into **Evidence** except to add a new line, and never set **Done**: the Acceptance Tester does that after checking production.

### Collision check

```bash
git fetch -q origin <target>
mine=$(git diff --name-only origin/<target>...HEAD)
gh pr list --state open --limit 100 --json number,author,files \
  --jq '.[] | "\(.number) \(.author.login) \(.files | map(.path) | join(" "))"' |
while read n who files; do
  hit=; for f in $mine; do case " $files " in *" $f "*) hit=1;; esac; done
  [ -n "$hit" ] || continue
  git fetch -q origin "pull/$n/head:refs/remotes/pr/$n"
  if out=$(git merge-tree --write-tree --name-only HEAD "pr/$n"); then echo "same file #$n $who"
  else echo "CONFLICT #$n $who $(echo "$out" | sed -n '2,6p' | tr '\n' ' ')"; fi
done
```

- Nothing, or only "same file": open the pull request and name the same-file ones in it.
- A conflict only in generated files (lock files, snapshots, bundles): open it anyway; whoever merges second regenerates them.
- A real conflict: do not open it and never change the other pull request. Push your branch, write "Waiting for #<n> (<author>) to merge or close, then rebase" in **Next Action**, and tell the person.

## Handing work to the pipeline

When the owner wants the Agents to do it: write "Owner agreed to hand this to the pipeline" in **Next Action**, then set **Agent** to the Lead (`agent:<the Lead's id>`; `agent {action:"list", workspace_id:"<workspace id>"}` gives its id). The Lead checks ownership and collisions and assigns a Developer.

To stop the pipeline on a pull request, turn it into a draft. The Lead leaves drafts alone.

## Filing a bug or a request

Add a row to **Inbox** (Item, Source, Reporter, Evidence) rather than writing an Issue yourself: the PM checks for duplicates, files it under the right Project and owner, and writes acceptance criteria. Write an Issue directly only when the person asks you to.

## Seeing what the pipeline is doing

Read **Pipeline board** (a document the Lead rewrites on every heartbeat): what is in flight, what was assigned and has no pull request yet, and what needs a person.

## Never

- Start an Agent by writing its name into **Agent**: only `agent:<id>` starts it, and only when the person asked.
- Change an Issue another person or Agent is working on without asking.
- Release to production, delete Issues, or read or repeat a credential.
