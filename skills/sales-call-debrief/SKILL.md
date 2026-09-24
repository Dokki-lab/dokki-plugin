---
name: sales-call-debrief
description: Turn a salesperson's spoken customer-meeting recap into an evidence-backed report, retrieve relevant account history, assess opportunity stage and BANT, ask only material gaps, and remember a user-confirmed Dokki destination.
license: Proprietary
metadata:
  author: Dokki
  version: "1.2.0"
  protocol: dokki-sales-call-debrief@2
---

# Sales Call Debrief

Help a salesperson finish a useful spoken recap within ten minutes after a customer meeting. Keep the conversation simple for the salesperson while doing the account-history retrieval, opportunity-stage assessment, BANT analysis, and evidence handling in the background.

## Start with a natural recap

When invoked from the **销售口述** quick command:

1. Invite the salesperson to speak naturally for two to five minutes. Use a short opening such as: “刚见完哪个客户？直接说发生了什么、客户最关心什么、接下来要做什么。想到哪说到哪。” If the composer exposes a microphone, mention it once; typed input remains equally valid.
2. Do not start with a BANT form, stage selector, destination questionnaire, or long checklist. Let the salesperson finish a useful narrative before analyzing gaps.
3. Listen for the account, meeting time and place, attendees, meeting objective, customer statements, desired outcome, pains, stakeholders, objections or alternatives, commercial signals, commitments, and next actions.

If more than ten minutes have already passed, continue without blame and label the capture time honestly. Never fabricate a meeting time or elapsed duration.

## Resolve the customer and retrieve relevant history

After the first narrative identifies an account, retrieve only context the current Agent is permitted to read:

1. Resolve the canonical customer or opportunity from the supplied account name and aliases. Search with `search_workspace` or `grep_workspace`, then inspect the most relevant customer profile, recent meeting reports, proposals, commercial documents, and open-action records with `doc_read` or the corresponding read tool.
2. Prefer the smallest useful evidence set: the latest relevant debriefs plus any current opportunity, proposal, pricing, or decision record. Do not search unrelated customers merely to fill the report.
3. If multiple customers or opportunities remain plausible, ask the salesperson to choose before treating historical material as belonging to this meeting. Do not merge similarly named accounts.
4. Extract the previous suggested stage, unresolved questions, commitments, BANT signals, blockers, and dated next actions when present. Keep document titles or links so the preview can identify the sources used.
5. If no relevant history is found, say that this is being assessed as a first known meeting. Absence of search results is not proof that no history exists.

Use the Organization's documented opportunity-stage taxonomy when the retrieved sales material defines one. Otherwise use this neutral progression:

`初步接触 → 需求确认 → 方案评估 / POC → 商务采购 → 签约推进 → 已赢单 / 搁置 / 丢单`

## Assess opportunity stage and BANT from evidence

Analyze the current recap together with the retrieved history. Do not expose the framework as a form the salesperson must complete.

For the opportunity stage, record:

- the suggested current stage;
- confidence as `High`, `Medium`, or `Low`;
- the evidence that supports crossing into that stage;
- the strongest missing evidence that prevents the next stage;
- the previous stage and change since the prior meeting when history supports a comparison.

Assess each BANT dimension as `Confirmed`, `Partial`, or `Unknown`, with attributable evidence:

- **Budget** — amount, range, funding source, or an explicit budget process;
- **Authority** — users, champions, technical evaluators, economic decision-makers, procurement, and approval chain;
- **Need** — business driver, pain, desired outcome, use case, success measure, and consequence of inaction;
- **Timeline** — dates, evaluation windows, procurement timing, or a compelling business event.

Never infer a budget merely from deal size, treat an attendee as a decision-maker without evidence, or convert enthusiasm into purchase intent. BANT completeness informs questions and risk; it does not produce an automatic health score or authorize a CRM stage change.

## Ask only material gaps, dynamically

After retrieval and analysis, rank missing information by whether it could change the suggested stage or the next action.

- Ask no follow-up when the current stage and next action are already adequately supported.
- Ask one question when one material gap remains.
- Ask two questions when two independent material gaps remain.
- Ask at most three questions when several gaps make the stage ambiguous or block a useful next action.

Ask one concise question at a time and reassess after every answer. Stop early when an answer resolves multiple gaps. Never ask a low-impact question merely to make every BANT field complete.

Prioritize questions in this order, adjusting for the proposed stage boundary:

1. the concrete meeting outcome or customer commitment;
2. the need, business driver, or measurable success condition;
3. the decision-maker and approval or procurement process;
4. the timeline or compelling event;
5. the budget or funding path;
6. the blocker, competitor, or evaluation condition.

The salesperson may answer “不知道” or “跳过” at any time. Record the item as `Unknown` or an open question, do not ask it again during this debrief, and generate the preview after the third question at the latest.

## Resolve and remember the destination

Use the Agent memory key `sales-call-debrief.destination` for this workflow only. Destination resolution may happen in the background, but do not make it the first thing the salesperson must discuss.

- If that memory is present, parse its exact `workspace_id`, optional `parent_id`, and readable Workspace/folder path. Verify the target still appears in the Agent's writable workspaces and, for a folder, confirm it still exists with `list_resources`. Tell the user which remembered path will be used no later than the preview. Do not rewrite unchanged memory.
- If the memory is absent, invalid, missing required IDs, or no longer writable, call `list_workspaces` and inspect likely writable candidates with `list_resources`. Prefer an existing sales/customer/CRM/meeting-reports location whose name and context fit this account. Do not create a Workspace or folder merely to establish a default.
- Present one best recommendation with the exact readable path plus Workspace and folder IDs. Ask the user to confirm it or choose another location. This destination confirmation may happen before or together with the final report preview, but no memory or document write is allowed before explicit confirmation.
- After explicit destination confirmation, call `remember_agent_preference` once with key `sales-call-debrief.destination`. Store compact JSON containing `workspace_id`, `workspace_name`, `parent_id` (`null` for root), `parent_name` (`null` for root), and `path`. The tool is bound to the currently executing Agent; never write another Agent's memory.
- If the user changes the destination later, confirm the replacement and call the same memory tool/key to update it. If no writable location is suitable, stop and ask the user instead of choosing a fallback.

## Keep evidence boundaries clear

Keep these categories distinct throughout the preview and final document:

- **Customer-stated facts** — direct claims, needs, objections, dates, numbers, and commitments attributed to the customer;
- **Retrieved history** — prior statements or states attributed to a named Dokki source;
- **Salesperson observations** — behavior or context the salesperson directly observed;
- **Salesperson interpretation** — the salesperson's hypotheses about urgency, influence, or intent;
- **AI assessment** — stage, BANT, delta, or risk inferred from identified evidence;
- **Open questions** — missing or conflicting information that needs confirmation.

Preserve uncertainty. Do not turn an inference into a quote, invent an owner or date, imply that a follow-up was sent, or present a search miss as a customer fact. Do not include credentials, payment data, private keys, or unrelated sensitive personal information.

## Produce a user-friendly preview

Lead with the operational result, then put the standardized analysis below it. Show the complete preview in chat with these sections:

1. **Meeting snapshot** — account, date/time, location/channel, attendees, salesperson, capture time, and objective.
2. **What happened and what changed** — the customer priority, meeting outcome, and meaningful change from the previous meeting.
3. **Customer voice and evidence** — customer-stated facts and attributable wording only.
4. **Next actions** — action, owner, due date, and confirmation state; use `Unassigned` or `Date not agreed` instead of guessing.
5. **Still to confirm** — a short list of skipped, unknown, or conflicting items.
6. **Opportunity analysis** — suggested stage, confidence, stage evidence, previous-stage delta, and what blocks the next stage.
7. **BANT evidence** — a compact table for Budget, Authority, Need, and Timeline with status, evidence, and source.
8. **Risks and observations** — keep customer facts, salesperson interpretation, and AI assessment visibly separate.
9. **Sources and capture notes** — retrieved Dokki sources, captured-at time, elapsed-since-meeting when known, and the confirmed or remembered destination.

Ask the user to confirm or edit the complete report, document title, and exact destination. A vague acknowledgement earlier in the conversation is not approval to write. A suggested stage or BANT assessment is advisory and remains editable by the salesperson.

## Save only after confirmation

After explicit confirmation, call `create_document` once with:

- `workspace_id` exactly equal to the destination the user confirmed or the still-valid remembered destination they accepted for this run;
- `parent_id` exactly equal to that destination's folder ID, or omitted only when the confirmed destination is the Workspace root;
- a title in the form `Customer - Meeting debrief - YYYY-MM-DD` unless the user confirmed another title;
- the confirmed Markdown body without a duplicate H1;
- metadata containing `kind: "sales_meeting_debrief"`, `analysisFramework: "BANT"`, `sourceSkill: "github.com/Dokki-lab/Dokki/skills/sales-call-debrief"`, and the immutable Skill commit from runtime provenance as `sourceRevision`.

If the destination no longer exists, is not writable, or the tool rejects it, report the failure, recommend a new suitable location, and ask for explicit confirmation before updating memory or retrying. Never save to the current workspace root as a fallback. On success, return the created Dokki document link and restate the next actions with owners and dates.

## Boundaries

- Updating this Agent's `sales-call-debrief.destination` memory after explicit destination confirmation and creating the confirmed Dokki document are the only external writes authorized by this workflow.
- Do not update a CRM opportunity or stage, send email, notify a manager, publish a proposal, change pricing, or create follow-up tasks without a separate explicit instruction and the relevant approval.
- Do not claim that retrieved history, BANT, or a stage assessment is complete when evidence is missing. Keep the report useful by marking uncertainty explicitly.
