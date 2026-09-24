# Loop

## Purpose

This file defines the execution contract for one autonomous loop. The loop remains simple: inspect the current objective, make one bounded change, verify it, and persist the result. Before every iteration, the runner applies five typed Jev questions. They control context, execution, tools, and permissions without turning the loop into an unstructured transcript.

## Core invariant

Every iteration MUST follow:

```text
retrieve relevant state
→ answer the five Jev questions
→ assign one bounded action to one authorized role
→ execute
→ verify
→ commit the result
→ decide whether to continue
```

The model's prose is advisory. The typed decision, policy checks, tool result, and verification evidence are authoritative.

## The five Jev questions

The runner MUST resolve these questions before every iteration. Deterministic policy may answer a question when the answer is unambiguous; an evaluator/model is used only for ambiguous cases.

### 1. Context visibility

**Question:** Which state, history, instructions, and evidence are relevant to this iteration, and how visible should each item be?

Each context item MUST receive one visibility level:

- `hide`: do not include it.
- `short`: include an identifier and compact result.
- `long`: include the result and relevant details.
- `full`: include the complete content.

The runner MUST prefer targeted state and evidence over the full historical transcript. `record.md` or equivalent history MUST NOT be loaded in full by default.

### 2. Context strategy

**Question:** Should the current assembled context be reused, or should it be rebuilt from authoritative state?

Allowed decisions:

- `reuse`: the relevant state version, objective, permissions, and evidence references are unchanged.
- `rebuild`: state changed, context is stale, an iteration failed, evidence conflicts, or the task requires a different view.

A rebuild MUST use authoritative state and evidence references. It MUST NOT silently inherit an unverified model claim.

### 3. Executor routing

**Question:** Which executor is appropriate for this bounded action?

Allowed executors:

- `deterministic`: parsing, schema validation, hashing, bookkeeping, and fixed policy checks.
- `local_or_cheap_model`: classification, summarization, retrieval ranking, and low-risk read-only work.
- `frontier_model`: architecture, ambiguity, difficult debugging, cross-file reasoning, and final decisions with high uncertainty.
- `human_approval`: irreversible, externally visible, high-impact, or policy-ambiguous actions.

Routing MUST consider context rebuild cost, not only per-token price. A cheap model MUST NOT receive the entire session when a small purpose-built context is sufficient.

### 4. Tool selection

**Question:** Which tool is required, and which minimum schema must be disclosed to execute it safely?

Tools MUST be disclosed progressively:

1. expose a short capability description;
2. select the tool;
3. load and validate only that tool's required schema;
4. execute with typed arguments;
5. remove unnecessary tool detail from the next context.

The executor MUST use one selected tool or an explicitly recorded tool chain. Unknown or unnecessary tools MUST NOT be invoked.

### 5. Permission

**Question:** Is the proposed action allowed, requires approval, or denied?

Allowed decisions:

- `allow`: policy authorizes the exact action.
- `ask`: the action needs human approval before execution.
- `deny`: the action violates policy or role ownership.

Default policy:

- deny access to secrets and credentials, including `.env`, `.ssh`, tokens, private keys, and credential stores;
- deny writes outside the declared workspace or assigned file scope;
- allow read-only inspection when it does not expose restricted data;
- require approval for destructive actions, external publication, irreversible operations, or actions outside the current objective;
- deny a role that attempts to modify an artifact owned by another role.

No action may execute when the decision is `ask` or `deny`.

## Role ownership and write boundaries

Every iteration MUST assign one primary role. Roles may read shared context, but each role may modify only its owned artifact type.

| Role | May modify | Must not modify |
|---|---|---|
| `Planner` | plan, objective decomposition, acceptance criteria, ordered next actions | source code, tests, verification evidence, authoritative state, permissions |
| `Coder` | source code and implementation files explicitly assigned to the coding task | plan, objective, acceptance criteria, verification evidence, permissions |
| `Tester` | test code when explicitly assigned; test outputs and test reports | production code, plan, authoritative state, permissions |
| `Verifier` | verification result, evidence references, defect findings, pass/fail recommendation | source code, plan, authoritative state, permissions |
| `StateManager` | authoritative state, iteration metadata, record entries, hashes, links to evidence | source code, plan content, test implementation, verification conclusions |
| `PermissionGuard` | permission decision and audit event only | every project artifact and execution target |
| `Human` | any artifact after explicit approval | none, subject to repository policy |

Role ownership is enforced by the runner, not by model instructions alone. A proposed patch outside the role's write set MUST be rejected before execution.

The roles are sequentially constrained:

```text
Planner → Coder → Tester/Verifier → StateManager
```

A role may stop the loop by reporting `blocked`, `failed`, or `needs_human_approval`. It may not bypass another role's boundary.

## Iteration contract

Before execution, the runner creates a decision envelope:

```yaml
iteration: <integer>
objective: <stable objective id>
role: Planner | Coder | Tester | Verifier | StateManager | PermissionGuard
intent: <one bounded action>
context:
  items:
    - id: <state/evidence/history id>
      visibility: hide | short | long | full
context_strategy:
  decision: reuse | rebuild
  reason: <short reason>
routing:
  executor: deterministic | local_or_cheap_model | frontier_model | human_approval
  reason: <short reason>
tool:
  capability: <short description>
  selected: <tool id or none>
  arguments_validated: true | false
permission:
  decision: allow | ask | deny
  reason: <policy reason>
expected_result: <observable result>
```

The loop MUST refuse execution if any of these fields is missing, if the selected tool arguments fail validation, or if the role/write scope is invalid.

## Execution and verification

Each iteration performs exactly one bounded mutation or one read-only operation. Prefer deterministic operations for bookkeeping and validation.

The executor MUST report:

```yaml
result:
  status: succeeded | failed | blocked | needs_human_approval
  changed_files: []
  output_refs: []
  summary: <what actually happened>
```

A successful mutation is not complete until a Tester or Verifier produces evidence. Evidence MUST identify the command or method, relevant inputs, result, and timestamp or iteration id.

The loop MUST distinguish:

- `planned`: requested by Planner but not executed;
- `executed`: action ran but is not yet verified;
- `verified`: evidence supports the expected result;
- `failed`: execution or verification failed;
- `blocked`: policy, missing dependency, or ambiguity prevents progress.

The model MUST NOT mark an action `verified` solely because it generated code or because a command was expected to pass.

## State and record rules

`state.md` is authoritative current state. `record.md` is append-only history. Neither file is owned by Planner, Coder, Tester, or Verifier; only `StateManager` may update them.

State updates MUST be versioned and include the expected prior version:

```yaml
state_update:
  expected_version: <integer>
  new_version: <integer>
  status: planned | executing | verified | failed | blocked
  next_action: <one bounded action>
  changes: []
  evidence_refs: []
```

If the expected version does not match, the update MUST be rejected and the context MUST be rebuilt.

Every record entry MUST state the iteration, role, action, result status, changed artifacts, and evidence references. The record is evidence of process, not a substitute for verification.

## Completion and continuation

After StateManager commits the result, the runner evaluates the objective:

- continue only when a valid next action exists and the objective is not verified;
- stop with `verified` when all acceptance criteria have evidence;
- stop with `blocked` when progress requires missing information, policy approval, or a role outside the current loop;
- stop with `failed` after the configured retry limit or when verification disproves the expected result.

The loop MUST NOT repeat an already completed action unless the decision envelope records why a retry is necessary.

## Minimal audit record

For every iteration, persist at least:

```yaml
iteration: <integer>
role: <role>
intent: <bounded action>
context_ids: []
context_decision: reuse | rebuild
executor: <executor>
tool: <tool or none>
permission: allow | ask | deny
result: succeeded | failed | blocked | needs_human_approval
changed_files: []
evidence_refs: []
next_action: <action or stop>
```

This contract keeps `loop.md` Markdown-native while adding the Jev control plane: context is assembled deliberately, routing is cost-aware, tools are progressive, permissions are explicit, and every role has a constrained write boundary.