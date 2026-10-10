---
name: dokki-automation
description: Create, inspect, revise, run, approve, and delete user-owned Personal or Organization Dokki Automations through MCP. Use for n8n-style WorkflowGraphV1 authoring, trigger/action node planning, revision-safe edits, approvals, and run history.
allowed-tools: mcp__dokki__find mcp__dokki__read mcp__dokki__create mcp__dokki__edit
license: Proprietary
metadata:
  author: Dokki
  version: "1.4.11"
  protocol: dokki-automation@1
---

# Dokki Automation

Create and maintain versioned Dokki workflows through the MCP facade. A visual workflow is a complete `WorkflowGraphV1` graph that the Automation editor renders with React Flow. Legacy code Automations remain supported, but do not translate a visual-workflow request into hidden handler code.

## Choose the operation

| Intent | Facade call |
|---|---|
| Discover exact node types | `find {action:"automation.nodes", args:{intent?, provider?, resource?, query?}}` |
| List Personal workflows | `find {action:"automations", args:{scope_type:"personal"}}` |
| List Organization workflows | `find {action:"automations", organization_id, args:{scope_type:"organization"}}` |
| Inspect one workflow | `read {action:"automation", args:{automation_id}}` |
| Inspect recent runs | `read {action:"automation.runs", args:{automation_id, limit?}}` |
| Inspect approval requests | `read {action:"automation.approvals", args:{automation_id, status?, limit?}}` |
| Approve or reject and resume | `edit {action:"automation.approval.decide", args:{approval_id, decision:"approved"|"rejected", note?}}` |
| Read a configured Webhook URL | `read {action:"automation.webhook", args:{automation_id}}` |
| Initialize or rotate a Webhook URL | `edit {action:"automation.webhook.configure", args:{automation_id, rotate?}, confirm_token?}` |
| Create a Personal visual workflow | `create {action:"automation", args:{scope_type:"personal", name, description?, workflow, enabled?}}` |
| Create an Organization visual workflow | `create {action:"automation", organization_id, args:{scope_type:"organization", name, description?, workflow, enabled?}}` |
| Save a new revision | `edit {action:"automation.update", args:{automation_id, base_revision_id, workflow?, ...}}` |
| Test one step | `edit {action:"automation.test_node", args:{automation_id, node_id, input?, fixtures?, mock_output?}}` |
| Run now | `edit {action:"automation.run", args:{automation_id, revision_id?, idempotency_key?, payload?}}` |
| Delete permanently | `edit {action:"automation.delete", args:{automation_id}}` and repeat with the returned top-level `confirm_token` only after explicit confirmation |

Choose the explicit `personal` or `organization` scope before listing or creating Automations. Every Automation has an `owner_user_id`; currently only that owner can see, edit, run, approve, configure, or delete it. An Organization scope also requires active membership, but membership or an admin role never grants access to another user's Automation. Sharing and delegated permissions are a future ACL layer, not implicit Organization visibility.

A Workspace is only an explicit target or event-source filter on a Dokki node. For every node whose catalog requires `workspace_id`, put an accessible target in the node config. Organization workflows may target only Workspaces in that Organization. Personal workflows may target any Workspace the owner can currently access. Every target is authorized again when the node runs; never infer ownership from a Workspace.

## Author `WorkflowGraphV1`

Call `find automation.nodes` before authoring or revising node types you have not just read from the saved graph. Treat its `type`, `default_config`, `required_config_paths`, `runtime_support`, `input_schema` and `output_schema` fields as authoritative; do not reconstruct action-node types from memory. An `output_schema` marked `speculative` is a guess from the node's kind, not an observed shape; `ai.prompt` emits one scalar string with no fields; `action.dokki.read.table`'s output depends on its branch (`args.format` / `args.mode`): `format:'json'` gives typed row objects keyed by column display name (later duplicates gain `(2)`, `(3)` in definition order; ids remain in `columns[]` metadata); bind `rows[].<column>` to check each name. CSV carries text in `data`, and compact carries text in `markdown`. A branching node answers `selection_status`, `branches` (in the order the runtime tries them) and a `hint`; pass `node_config` to see one branch's `output_schema`.

Send the complete graph, not a partial node patch:

```json
{
  "schemaVersion": 1,
  "nodes": [
    {
      "id": "trigger-1",
      "type": "trigger.manual",
      "typeVersion": 1,
      "label": "When run manually",
      "config": {},
      "position": { "x": 80, "y": 160 }
    }
  ],
  "edges": [],
  "viewport": { "x": 0, "y": 0, "zoom": 1 }
}
```

Every node needs a stable unique `id`, a supported `type`, `typeVersion:1`, a human-readable `label`, object `config`, and finite `position.x/y`. Every edge needs a stable unique `id`, valid `source` and `target` node IDs, and no self-loop. Keep node and edge IDs stable across edits unless the corresponding object is intentionally replaced. Different nodes may target different accessible Workspaces; each target is authorized again when that node runs.

The catalog contains:

- Triggers: `trigger.manual` (declares run-time values in `config.inputs[]` — `{ name, type, required, description, example }` — and the run validates `trigger_payload` against them), `trigger.schedule`, `trigger.webhook`, the legacy `trigger.event`, plus granular `trigger.dokki.<resource>.<event>` nodes for resources, documents, tables, artifacts, files, messages, and publishing
- Actions: `action.add_tag`, `action.remove_tag`, `action.agent.dispatch` (queues an Agent; the run does not wait for its answer), plus every Dokki MCP capability as `action.dokki.<facade>.<action>`; preserve the facade/action/config emitted by the product catalog
- Hidden, unreleased Hire: `action.hire` (see the first-version limits below)
- Conditions: `condition.expression` (edges leave from the `true` / `false` handles) and `condition.switch` (one handle per case, `default` when none matched)
- Flow: `transform.code`, `transform.convert` (one explicit conversion by `operation`: `parse_json`, `parse_csv`, `to_json`, `to_csv`, `markdown_to_html`, `html_to_text`; `input` is a template or binding; strings stay strings, a malformed input fails the node), `flow.parallel` + `flow.join` (branches run concurrently up to `max_concurrency`; a node with two or more inputs must be a join), `flow.for_each`, `flow.while`, `flow.retry`, `flow.catch` (from an upstream error handle), `flow.delay`, `flow.wait_event` (durable suspend, `runtime_support: pause_required`), `flow.call_workflow` (pins a revision), `flow.end`
- AI: `ai.prompt`, `ai.agent_task` (hidden; requires org policy `automation.agent_task`)
- Media: `media.image.generate`, `media.video.generate` (one generated file per item, saved in the folder `parent_id` names; batch summary output only, `items`/`succeeded`/`failed`)
- Human: `human.work_item`, `human.review` (rolling out behind feature flag `automation_human_review_v1`; while off, a write that adds it is refused with `human_review_not_enabled`)

| Node | Runtime contract |
|---|---|
| `ai.agent_task` | Dispatch an isolated Agent task and wait for validated `result` and `artifacts`; route `success` or `error`. Configure `agent_id`, `task`, `input`, `result_schema` using ContractSchema, and `limits`. |
| `human.review` | Freeze a snapshot of `subject`, assign ONE reviewer (`assignee_user_id`: an org member, or the owner in a Personal Automation), optional `form` fields and `deadline_hours` (1–720); routes `approved` / `rejected` / `expired` (all three must be connected; `request_changes` is reserved). Only nodes on `approved` may read `approved_subject` / `form_data`. The assignee decides in Dokki (Needs you); MCP `decide_automation_approval` refuses review requests. |

Use exactly one trigger. Connect every non-trigger node to it, directly or indirectly. Keep the graph acyclic. Put event, schedule, resource, prompt, expression, code, tag, MCP arguments, or work-item settings in `config`; do not encode operational settings into labels. Optional `stages: [{id, name, nodeIds}]` group the canvas and never change execution.

## Hire Agent — unreleased first version

`action.hire` is hidden from the node library pending release verification. Its presence in this catalog does not mean the feature is enabled or deployed; do not offer it as generally available or bypass the rollout gate.

The first version supports only a root workflow buying a fixed-price `hosted_agent` service. The owner selects a fixed payer Agent, listing, confirmed price and credit cap; trigger data and bindings cannot replace those commercial terms or increase the cap. A price change requires explicit confirmation. Delivery and manual-review timeouts are separate, fixed at admission and preserved across retries.

Request exactly one text deliverable: `{name:"report", kind:"text"}`. Other names, multiple deliverables and file results are unsupported. A resource ID does not freeze file bytes. Nested workflows and paid node tests are refused; mock node tests never admit an order or debit credits.

The workflow durably waits for delivery and a human payer-side decision. It succeeds only after manual acceptance and settlement; downstream nodes receive the accepted text snapshot, `engagement_id`, `listing_id`, `settled_credits` and `accepted_at`. A queued dispatch, provider completion or unreviewed report is not success. Canceling or timing out the workflow does not prove the supplier stopped or that a refund completed.

### Runtime limits

The save validates the graph (at most 500 nodes / 2000 edges, fan-out 32, node `timeout` 1–3600 s, `max_iterations` 1–10000, `max_depth` 1–20, one trigger, no unreachable node, no cycle). The values below are **not** validated at save: an out-of-range value is accepted and then replaced by the default at run time — not clamped, and with no error anywhere.

| Config | Range | Default | Out of range means |
|---|---|---|---|
| `flow.delay.duration_ms` | 0–60000 | 1000 | a 300000 ms delay waits one second |
| `flow.retry.delay_ms` / `max_attempts` | 0–60000 / 1–20 | 250 / 3 | |
| `flow.for_each.max_concurrency` | 1–32 | 4 | |
| `flow.for_each` / `flow.while` `max_iterations` | 1–10000 | 100 / 20 | rejected at save |
| `flow.call_workflow.max_depth` | 1–20 | 3 | rejected at save |

A rejected save returns `issues[]`, each with `code`, `node_id`, `path` and a `fix` sentence: a template or binding to a field the selected branch does not carry (`template_path_not_in_branch`), an id bound into another identity space (`binding_id_space_mismatch`; a tables row into a `read.table` target is the one accepted alias), or an unproven type on an automatic binding (`binding_type_unproven`). Apply the `fix` and save again.

A wait longer than 60 s is a chain of `flow.delay` nodes of at most 60000 each, or a `flow.wait_event` that waits for the thing instead of the time. Say the bound to the person rather than writing the number they asked for.

### Data between nodes

A config string may contain `{{path}}`. The roots are `input` (the upstream node's output), `trigger.type`, `trigger.payload.*`, `nodes.<node id>.output`, and `workflow.workspace_id` / `organization_id` / `execution_id`; any other first segment is read as a **node id**, so `{{summary.title}}` means `{{nodes.summary.output.title}}` and a root always wins over a node of the same name. A string that is exactly one template yields the raw value; inside other text it is stringified; **a path that reaches nothing fails silently** — unfilled alone, an empty string inside text — and `ai.prompt` emits one string, so `{{summary.text}}` reaches nothing while `{{summary}}` is the whole answer. An edge may carry `bindings` (source node + path → target config path); the validator accepts only paths the upstream `output_schema` has, and a scalar output is the empty path. A node you save may declare its own `outputSchema: { fields: [{ path, label, type }] }` so downstream steps can bind to what a `transform.code` returns.

## Create safely

1. Use `find automations` to check for an existing workflow with the same purpose.
2. Build and review the complete graph. Name nodes for the user's intent rather than internal implementation details.
3. Default to `enabled:false`; enable only when the user requests activation and every node is executable.
4. Read the returned Automation and verify its stored `workflow`, `current_revision_id`, `scope_type`, `owner_user_id`, optional Organization, and URL.

Manual, webhook, and listed Dokki Event Fabric triggers can run visual graphs when every node is executable. Pure transforms run in the sandbox, AI nodes use the existing metered AI path, and Dokki action nodes use the same permission, billing, and idempotency boundaries as MCP. Human work items and destructive Dokki actions are executable through durable approval checkpoints. Both scopes may use global discovery actions; every Workspace-targeting action still requires an explicit authorized `workspace_id`.

## Design check per step

For every step added or changed, pass a small made-up sample input to `automation.test_node`; use its observed output as the design snapshot. Reads use your access, writes are dry runs, and AI or paid steps are simulated. If testing is disabled, say verification was skipped. Before saving, compare each downstream use with the upstream snapshot. After `create automation` or `automation.update`, read `design_check`; fix each `found:false` or explain why it stays. Show the person one table in their language: 步驟 | 用了什麼測試資料（一句話） | 產出欄位（顯示名稱） | 被哪些步驟使用 | 怎麼驗證的（實測／模擬／試跑／未測）. Use only step labels and display names. Mark simulated, dry-run and untested steps unverified; never call them tested with real output. Keep ids, dotted paths, JSON and template syntax out of the default reply.

## Edit with immutable revisions

Always read the Automation immediately before editing. Use the returned `current_revision_id` as `base_revision_id`, then send the complete replacement `workflow` plus only the metadata fields that should change.

If the save reports a revision conflict, do not blindly retry. Read the new current revision, reconcile the other editor's changes, show any material conflict to the user, and save against the new base revision. After a successful save, read back the Automation and confirm the new revision ID.

## Run and inspect

Before running, read the Automation and inspect the complete graph and runtime-readiness response. Use a stable unique `idempotency_key` for retries of the same intended run so uncertainty does not create duplicate effects. Do not reuse a key for a different intended run. Event-triggered runs are created automatically by the durable outbox and should not be simulated with a manual payload.

`automation.run` only queues the run (the API answers 202); inspect `automation.runs` afterwards and report the persisted status (`pending`, `running`, `waiting`, `success`, `error`, `skipped`, `timeout`, `canceled`) and error honestly. A tool response or HTTP success alone is not execution proof. There is no facade action that stops a run: the owner's Stop in the run panel settles a `pending` or `waiting` run at once and reaches a `running` one at its next heartbeat (15 s), between effects — an effect in flight completes.

If the persisted status is `waiting`, use `read automation.approvals` to list pending checkpoints. Present the exact Automation, node, action, summary, and bounded request details before asking for a decision. Call `edit automation.approval.decide` only after the user authorizes that exact request or explicitly rejects it. The decision resumes the same pinned revision and may reach another approval; inspect the run again until it is terminal or waiting on a new node. Never treat approval of one node as approval of later nodes.

For a Webhook-triggered workflow, call `read automation.webhook` after saving. If it is not configured, call `edit automation.webhook.configure` once and return the credential-bearing URL through an appropriate private channel. Every logical POST must carry a stable `Idempotency-Key`; reusing a key with different JSON is rejected. Rotation invalidates the previous URL immediately: call with `rotate:true`, present the exact Automation and consequence when the tool returns `requires_confirmation`, and repeat only after explicit authorization with the returned top-level `confirm_token` and identical Automation ID.

## Not yet on every environment (checked 2026-09-16)

- **Draft/publish and the run overlay on the canvas are rolling out behind flags.** Step testing is available where `automation.node_tests.admission` is enabled; where it is off, report verification skipped. With `authoring_mode` `draft_v1`, saving does not change the revision runs use until the owner publishes.

## Destructive and external-effect boundaries

- Deletion permanently removes the Automation and its revision/run history. Describe the exact target and obtain explicit confirmation before repeating with `confirm_token`.
- A workflow's dangerous action uses its persisted Automation approval. Do not copy its request into a separate facade call or manually handle the runtime's short-lived confirmation token.
- Enabling schedules, running workflows, sending messages, publishing, sharing, deleting resources, or calling connected systems can create external effects. Require the authority appropriate to that action.
- Never place credentials, API keys, connector tokens, or sensitive payloads in node labels, graph config, logs, or chat output.
- Treat an Automation Webhook URL as a credential. Do not put it in the workflow graph, a document, a public message, or a screenshot; redact it in acceptance media.
- Use only event types listed by the product catalog. Table row events originate from the rows-engine outbox or authoritative CRDT persistence; event payloads carry identifiers and changed-field names, not full private content.

## Return a useful result

Lead with the outcome, then include the Automation name, scope, owner, optional Organization, enabled/draft state, trigger type, node count, targeted Workspaces, and URL. Keep internal ids and raw graph data out of the default reply. For edits, summarize steps added, removed, or changed by label. For blocked execution, name the exact steps and runtime boundary that prevent enablement.
