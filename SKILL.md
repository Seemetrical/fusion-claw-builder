---
name: fusion-claw-builder
description: "Final step after an AI Agent Studio agentic app is built: add Fusion Claw autonomous outcomes by coding the capabilities into the app's workflows and the Autonomous Outcomes settings (goals, skills, additional agents, examples) into the app file, then saving and reading back through the workspace CLI. No computer use."
---

# Fusion Claw Builder

Invoke this **after** the agentic app is built as usual. It adds Fusion Claw (autonomous outcomes) to that app by editing artifacts in code:

- **Part A:** code the capabilities into the workflows (the read capabilities and action capabilities Claw will call).
- **Part B:** code the Autonomous Outcomes settings into the app file, then save and read back through the workspace CLI.

No step uses computer use or screen clicking. Work only in a test environment. Stop for human confirmation before every **Publish**; the workspace CLI does not publish workflows, so publishing is done by a person in Studio.

## How Claw uses what you build

- Claw is one runtime. The app's Autonomous Outcomes settings make it a specialist. Claw reads and changes Fusion **only through capabilities**.
- **Capabilities** are defined on workflows, in the workflow's CAPABILITY trigger. They reach Claw in two ways: from agents mapped to the app's sections, or from workflows registered as **additional autonomous agents**. The app's Capabilities tab only displays them.
- **Research mode** uses read-only capabilities only, so nothing changes. **Full Auto** adds action capabilities within the user's permissions. Anything Claw cannot resolve comes back as **Needs Action**. The list of action capabilities is the complete set of changes Claw can make.
- Expected Full Auto behaviour: re-read the record, check which actions are allowed now, make **one** change, read it back, then mark it Verified. Design every action capability for this.
- Every run produces an **Outcome Receipt** that lists the mode, skills loaded, each capability call with its input and output, and the before and after state of any change.

## Rules

- Never invent Fusion REST resources, delivered agents or allowed values. Check them on the tenant, or mark them [UNVERIFIED].
- If the app is Oracle-delivered, copy it in Applications and work in the copy.
- Keep capabilities narrow: one job each, with an action-verb snake_case name (`get_invoices_on_hold`, `release_invoice_hold`).
- Every action capability needs a read partner, a pre-check, an idempotency rule and a readback.
- Write goals and examples as business results, never as screen steps.
- Name artifacts for reuse: no client name in capability names, skill names, workflow codes or descriptions. Capability and skill descriptions use plain business names, never internal codes (cohort, scenario or record IDs).
- Before editing any app or workflow file: fetch the server DRAFT, compare it with the local file, and back up both. If the server copy is newer, merge onto it; never overwrite it blindly.
- Write only the fields listed in the field map below. A field marked [UNVERIFIED] stays unset unless the user supplies a confirmed example of its shape.

## Field map: where each component lives

Observed on release-26C (server copies of working apps). Verify against the current workspace skill before use.

| Component | Artifact and field | Shape |
|---|---|---|
| Agentic Apps compatibility | Workflow `.wf`: top-level `aiAppsCompatibleFlag` | `true` for workflows exposed to the app, including additional autonomous agents |
| Enable autonomous outcomes | App `.apps`: `specification.applicationMetadata.enableAutonomousActivities` | `true` |
| Goals | App: `applicationMetadata.goals` | One string, Markdown allowed: the result, who it applies to, what done means, evidence |
| Skills | App: `applicationMetadata.skills[]` | `{ "id": "skill_<uuid v4>", "name", "description", "prompt" }` |
| Additional agents | App: `applicationMetadata.additionalAutonomousAgents[]` | `{ "workflowCode", "title", "description", "useDraftWorkflowWhileDeveloping" }` (documented in the workspace skill as CLI-only metadata) |
| Examples | App: `applicationMetadata.exampleTasks[]` | `{ "title", "text", "taskMode": "RESEARCH" or "ACT", "category": "Research", "Actions" or "Reports" }` |
| Default model | App: `applicationMetadata.defaultOutcomeModel` | Leave unset for Default; set only to a model code confirmed on the tenant |
| Capabilities | Workflow `.wf`: `specification.triggers[]` entry with `"type": "CAPABILITY"`, its `inputs[]` | `{ "name", "description", "category": "Generic", "readOnly", "requiresCompute", "typeSpecification": "<JSON schema as a string>" }` |
| App-level capability list | App: `applicationMetadata.capabilities[]` | [UNVERIFIED]: only `agentId` (a workflow code) is documented. Leave as is; capabilities reach Claw from the workflows. |
| Data extracts | App: `applicationMetadata.dataExtracts[]` | [UNVERIFIED] shape. Leave `[]` unless a confirmed stored procedure and shape exist. |
| Built-in skills, container shape | Not found in app files | [UNVERIFIED]. Do not code; list them in the hand-back for a person to set if needed. |

`do-modify-app-config` rejects these autonomy keys, so edit them directly in the `.apps` JSON, then save the whole app with `do-save-app`. Keys you do not touch must survive unchanged.

---

## Step 1: Read the built app

From the app you just built, list:
- the app's purpose and persona
- its sections and the agents mapped to them (`applicationMetadata.agents[*].agent` gives the workflow codes)
- the capabilities those workflows already expose (their CAPABILITY trigger inputs)
- the current autonomy fields in the app file (field map above)
- the demo journey

Then derive:
- **2 to 4 outcomes** as business results, paired where possible: a Research outcome that reads, compares and recommends, plus a Full Auto outcome that carries out one reviewed step.
- **The capability gap:** what Claw needs to read or change that existing capabilities do not cover. If the app has no record to change (for example, embedded demonstration data), plan Research outcomes only and say so.

Account for each user-requested outcome with its required capabilities and any concrete blocker. Do not silently replace a requested Full Auto action with Research-only delivery. A partly built outcome remains incomplete even when its read capabilities pass tests.

## Step 2 (Part A): Code the capabilities into the workflows

Prefer extending the panel workflows when the capability reuses their data and checks. Use a separate capability workflow, registered as an additional autonomous agent, when the panel workflows should stay unchanged or the capability needs different permissions.

For every workflow the app will expose, explicitly verify `aiAppsCompatibleFlag: true`. Set it with `do-modify-workflow-metadata --file <workflow-file> --ai-apps-compatible-flag true` when needed, validate the workflow and verify the flag in the saved server readback. A CAPABILITY trigger alone does not establish Agentic Apps compatibility.

**2a. Add each capability to the workflow's CAPABILITY trigger** (`specification.triggers[]`, `type: "CAPABILITY"`; create the trigger if absent):

| Field | Value |
|---|---|
| `name` | Action-verb snake_case, unique across every workflow the app exposes |
| `description` | Use the patterns below. Claw relies on this wording to decide when to call it |
| `category` | `"Generic"` |
| `readOnly` | `true` if it changes nothing |
| `requiresCompute` | `true` for heavy computation, file processing or large volumes |
| `typeSpecification` | The input JSON schema, serialised as a string, with `"additionalProperties": false` |

**Read description pattern:**
> Use when <need>. Returns <fields> for <scope, limited to the user's data access>. Read only. Not for <out of scope>.

**Action description pattern:**
> Use when <condition>, after <read capability> confirms <current state>. Do not act when <exclusions, policy breaches or missing data>; return Needs Action instead. Changes <field or state> from <a> to <b>. Skips if already in the target state. Confirm by calling <read capability> afterwards. Returns record ID, before state and after state.

**Input schema skeleton:**
```json
{
  "type": "object",
  "properties": {
    "recordId": {"type": "string", "description": "<what it identifies>"},
    "targetValue": {"type": "string", "description": "<allowed values come from <read capability>>"}
  },
  "required": ["recordId"],
  "additionalProperties": false
}
```

**2b. Route capability calls on their own path.** Add a SWITCH near the start of the workflow, after any shared data loading:

```
{{($context.$app?.$OraMessageHint === "InvokeCapability" || $context.$system.$triggerType === "CAPABILITY") ? "InvokeCapability" : "ExistingApp"}}
```

- `ExistingApp` continues to the workflow's normal app-stage routing, unchanged.
- `InvokeCapability` goes to a CODE node that reads the capability name and payload: from `$context.$triggers.CAPABILITY.$input.capabilityName` and `.capabilityPayload` when `$context.$system.$triggerType` is `CAPABILITY`, otherwise from `$context.$app.$OraAction` and `$context.$app.$OraActionPayload`. It validates the payload against the declared schema, dispatches by name and returns a structured result with a status, the scope used and, for actions, the before and after state.
- Use the current workspace's supported `InvokeCapability` output path, including a dedicated LLM or AGENT where required. Its job is the reasoned handover described below. Route invalid input and blocked data explicitly so they cannot fall through into source calls or mutation.

**Calculation contract and reasoned handover.** Code owns validated facts, deterministic calculations and controls. Give the LLM a workflow-specific calculation contract containing the relevant source values, calculated results, units, scope, source/observation dates, source references, statuses, missing data and applicable limits. Distinguish source facts from calculated results in that contract; do not invent platform fields or treat the capability input schema as an output contract.

The LLM must consume that contract and explain the business meaning: what changed, its implications, limitations and the appropriate next step. It may interpret and explain the supplied results, but must not redo financial arithmetic, alter figures, fill missing values or upgrade an unverified status. Preserve any figures it cites, including their units and periods, and distinguish interpretation from calculated results. Delimit the injected contract as untrusted data, not instructions.

Preserve the exact structured facts separately from the reasoned explanation when the capability needs both. Do not make the model's literal reproduction of a large financial JSON payload the authoritative data handover. Use only a documented or verified serialization/return pattern compatible with the current workspace; a deterministic RETURN after reasoning is a candidate to verify, not a universal platform guarantee. Do not add an empty acknowledgement LLM merely to satisfy an assumed terminal-node rule. A workflow debugger pass alone does not prove native Claw/Outcome Receipt acceptance.

The action logic follows this order: re-read → validate against the values allowed now → skip if already done → make one change → read back → return before and after.

**Recovery and idempotency.** Distinguish an already aligned record from a verified earlier transaction. Equality with the target does not prove that earlier business rules completed. Report pending, failed and unknown outcomes explicitly; do not claim either Verified or unchanged when the evidence does not establish it. Verify every required write, rule/job completion and readback before returning Verified. Do not automatically resubmit an uncertain mutation.

For multi-record outcomes, state the transaction boundary and how partial completion is handled. Follow the authorized stop/continue policy and stop further changes on an uncertain action. Declare whether calls must be serialized or whether an atomic concurrency mechanism has been verified. A before-state fingerprint detects drift; it is not an authorization token or an atomic lock. Do not claim rollback or exactly-once execution without evidence.

**2c. Save and test.** Fetch the server DRAFT, compare and back up, then save with `do-save-workflow --expected-remote-version N`. Test each capability with a workflow test whose input uses `"appHint": "InvokeCapability"` with the capability name as the action and its payload. Cover a valid call, invalid input (rejected, no source read), the ordinary panel stages unchanged, and for actions a repeat call (no duplicate change).

**2d. Publication and app bindings.** Before app publication, check every workflow referenced by mapped panel agents and additional autonomous agents, not only the workflows changed in this increment. Verify their codes, compatibility, available published versions and `useDraftWorkflowWhileDeveloping` settings. Draft development still needs a published anchor. Ask for human publication only for workflows that require it, and wait for confirmation; verify the publication afterward. Preserve publication confirmation already given in the session.

If Studio reports no published version but the backend lookup finds one, investigate the app binding and editor state rather than repeatedly requesting publication. Reselecting the exact workflow or using its published version instead of draft development can be a targeted remedy when supported by the evidence and the user's intended configuration. Do not silently change draft-use preferences or assume a CLI-normalized artifact's status field alone proves what Studio's publication check sees. Record the requested server status, returned version and Studio outcome separately.

## Step 3: Write the Claw skills

Write one **object skill** per business object, and one **job skill** per outcome. A skill's `description` decides whether Claw selects it, so include the phrases users actually say ("start my day", "what's blocking close").

Each skill becomes one `skills[]` entry: a new `id` of `skill_` plus a UUID v4, a `name`, a one-to-two sentence `description`, and the `prompt` below. The prompt layout is [UNVERIFIED]; keep it short (the observed working skill's prompt is about 700 characters).

```markdown
# <Skill name>
## Purpose
Intent and scope; when to use it and when not to.
## Inputs
Data needed → capability for each (mark read only).
## Method
The steps. Join records by ID. Use deterministic computation for totals and matching. Validate counts and duplicates before summarising.
## Output
Format: headline, priority cards, sections, plus a spreadsheet if useful. Cite evidence. Say what could not be confirmed.
## Boundaries
Which actions are Full Auto only, and which need approval. Re-read before any change, make one change per call, then read back. Missing data or a policy breach → Needs Action.
## Done
Completed / Blocked (say what is missing) / Escalated (to whom).
```

## Step 4 (Part B): Code the Autonomous Outcomes settings into the app

1. Fetch the app's server DRAFT with `do-fetch-app`, compare it with the local `.apps` file and back up both.
2. Edit `specification.applicationMetadata` per the field map:
   - `enableAutonomousActivities: true`
   - `goals`: one string, one entry per goal (result, who it applies to, what done means, evidence)
   - `skills[]`: merge the skills from Step 3; preserve IDs for existing skills being updated
   - `additionalAutonomousAgents[]`: only for capability workflows that are not mapped to an app section; reuse an existing registration rather than adding a duplicate, and set `useDraftWorkflowWhileDeveloping` deliberately
   - `exampleTasks[]`: merge useful examples, mostly `RESEARCH`, with at least one `ACT` example paired with a Research one. Use `ACT` only when an action capability exists; otherwise every example is `RESEARCH`. Preserve unrelated existing tasks.
   - leave `capabilities[]`, `dataExtracts[]` and any [UNVERIFIED] field as they are
3. Validate locally: the file parses; every skill `id` is unique and every `name` is unique; every `workflowCode` in `additionalAutonomousAgents` exists and has a CAPABILITY trigger; every capability a skill or example relies on exists in a mapped or additional workflow; capability names are unique across those workflows; no client name or internal code appears in names or descriptions.
   Also check semantic consistency across old and new goals, skills, descriptions and examples. Remove superseded claims such as "no Apply capability exists" after adding an action. Scope Research-only restrictions to Research so they do not contradict an authorized Full Auto skill. Keep the code's controls and statuses consistent with the LLM's handover instructions. Successful field persistence does not prove these instructions agree.
4. Save with `do-save-app --file <app file>`.
5. Fetch the server DRAFT again and compare its `specification` with the local file. Every autonomy field must match. If the server dropped or changed a field, stop and report it; do not retry with guessed shapes.
6. If the app's tests are out of sync after the change, run the workspace app-test sync as usual.

## Step 5: Test, then hand back

**In code (this skill):**

| Test | Pass when |
|---|---|
| Each read capability | The `InvokeCapability` workflow test returns the declared output with correct figures and scope |
| Compatibility | Every exposed workflow is Agentic Apps compatible and the saved server flag matches |
| Calculation handover | Code results remain exact; the LLM explains their implications and limitations faithfully without recalculation, invented values or changed statuses |
| Invalid input | Rejected with a clear reason and no source read |
| Ordinary panels | Existing app-stage tests still pass |
| Each action capability | One change with before and after state and a readback; a repeat call makes no duplicate change; a disallowed value returns Needs Action |
| App settings | Server readback matches every autonomy field written |

Label evidence separately: local validation, replay tests, live reads, live actions and native Outcome Receipts. Identify modeled mutation responses explicitly. A replayed Verified receipt proves the tested branch, not that a real transaction occurred; publication and saved settings do not establish live acceptance. Run live side effects only within the user's authorized scope, and report unexecuted action/receipt tests as pending. Check requested outcome coverage as well as test counts.

**On the screen (a person or the independent verifier):** run the outcomes from **Define an Outcome** with the demo journey's data and check the Outcome Receipt:

| Test | Pass when the receipt shows |
|---|---|
| Research outcome | "Confirmed read-only task mode", the expected skills loaded, the expected capabilities called, nothing changed |
| One Full Auto action | "Confirmed transactional task mode", one change with before and after state, readback, Verified |
| Same action again | No duplicate change |
| Disallowed or out-of-policy action | Needs Action, nothing changed |
| Missing data | Blocked, and it says what is missing |

If a receipt test fails, diagnose the failing layer from evidence: compatibility/registration and publication bindings; mode and capability selection; invocation payload and routing; source availability and scope; calculation-contract accuracy; mutation, job completion and readback; then the reasoned handover. Change capability or skill descriptions when selection evidence points to them, rather than assuming every failure is a description problem. Separate pre-existing app validation issues from introduced failures; do not invent references or alter unrelated configuration merely to pass validation.

Hand back the actual state: requested outcomes implemented or still blocked; capabilities coded (read or action) with workflow/version and compatibility; skills; app fields written and readback; publication/registration state; each evidence level achieved; pending live action and native receipt checks; and [UNVERIFIED] items. Say **Publish is pending human confirmation** only when publication is actually pending. Do not label the app or Full Auto outcome accepted solely because its configuration saved or its replay tests passed.
