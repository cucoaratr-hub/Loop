# Loop — AI Stack Orchestration & Adversarial Iteration Engine

> Status: Adversarial verification-gate review
> Last updated: 2026-09-23
> Authority: This file is the living architecture and decision record for the Loop project.

## 1. Objective and fixed constraints

Loop is a repeatable workflow for converting plain-English `spec.md` into working software across six project types:

1. Newsletter platform.
2. Crypto market analysis tool.
3. Video creator workflow.
4. PDF investor analysis tool.
5. Database builder with audit software.
6. Website builder.

The user is a non-programmer and must control milestone decisions without reading code, diffs, terminal output, or maintaining scripts. Each milestone must provide exactly three plain-English review sections: **What was built**, **How to test**, and **Expected result on screen**. The same workflow must work across all six project types without project-specific orchestration logic.

The following are fixed project inputs, not redesign targets:

- 9router is installed, operational, and connected to the user's configured local and cloud routes at `localhost:20128`, including RTK token compression and the existing three-tier fallback.
- The local server has an NVIDIA V100 and runs Ollama with the user's specified Qwen3.8-27B model.
- OpenCode Go and Merge Gateway are paid allocations available in the existing stack.
- Private GitHub repositories store source, specifications, state, pull requests, and execution history.

The 9router layer must not be redesigned, replaced, or reconfigured. The fact that a resource is named in the brief does not prove its exact API, scheduler, GitHub integration, model tag, or merge authority; those capabilities require verification.

## 2. System Component Matrix

| Component | What it does | Exact model/provider | Local or online | Why it is used | Verification status |
|---|---|---|---|---|---|
| `spec.md` | States desired behavior and acceptance criteria; never prescribes implementation | None | Stored in private GitHub; consumed by control plane | Keeps the workflow project-agnostic | Required convention; validate schema and path |
| `loop.md` | Stores architecture, state machines, constraints, gate results, decisions, and rejected options | None | Stored in private GitHub | Single living source of truth | This file; updated through reviewed commits |
| GitHub | Stores private repositories, branches, PRs, commits, state snapshots, evidence, and history | None | Online | Durable source control and recovery trail | Verify privacy, branch protection, checks, and least-privilege credentials |
| Control plane/orchestrator | Owns queueing, scheduling, state transitions, locks, retries, gate enforcement, checkpoint writes, and recovery | No model by itself; it invokes task-specific routes | Execution placement must be selected; proposed protected container runtime | A model, proxy, and GitHub are not a complete scheduler or state machine | Must verify restart, locking, backpressure, and no-script operation |
| Background worker | Reads a clean repository snapshot, performs bounded audits, drafts tests, and creates non-destructive PR proposals | Local Ollama → Qwen3.8-27B through the fixed 9router route | Local V100 server | Repeated work uses free local compute and preserves paid capacity | Verify exact installed tag, route identity, GPU/RAM behavior, and output quality |
| Implementation worker | Executes an approved milestone, edits an isolated branch, runs configured tools, and prepares a PR | Exact model/provider is **not established** by the brief; candidate paid allocation must be verified | Online if the verified paid route is online; otherwise not selected | Implementation can be reserved for higher-capability approved routing | Verify OpenCode Go/Merge Gateway capabilities, model identity, API, quotas, and PR permissions |
| Adversarial reviewer | Challenges requirements, implementation claims, test coverage, and failure handling | Local Qwen3.8-27B by default; approved paid route only for escalation | Local by default; online only after explicit routing policy | Independent challenge at lower cost, with paid escalation for difficult cases | Define conflict/escalation criteria and record route identity |
| Deterministic check runner | Executes tests, lint, type checks, builds, smoke tests, and security/dependency checks | None; models may propose checks but never replace them | Verified CI or runner; not assumed | Produces machine-checkable evidence | Verify isolation, artifacts, timeout, cancellation, and required-check policy |
| Branch/PR mechanism | Isolates changes, collects evidence, and supplies rollback/review boundaries | None | GitHub online | Prevents unreviewed default-branch changes | Verify branch protection, stale approval handling, and merge race controls |
| HITL gate | Collects plain-English approve/reject/feedback based on behavior | None | User-facing interface; exact surface to be selected | Enforces the user's high-level control without code review | Verify test instructions are executable by a non-programmer |
| `.loop/loop-state.json` | Stores durable state, attempts, checkpoints, ownership, references, and recovery metadata | None | Committed to private GitHub; ephemeral locks remain outside durable state | Restart recovery and deterministic resume | Verify schema validation, atomic commit, single-writer policy, and secret exclusion |
| Audit/evidence record | Stores gate decisions, route identity, CI results, errors, retries, and rejected options | None | GitHub history plus protected execution logs | Enables auditability and reconstruction | Verify timestamps, correlation IDs, redaction, retention, and access control |
| Container runtime | Isolates workers, dependencies, resource limits, and restart policy | Hosts the workers; does not determine model identity | Proposed local V100 host; exact runtime not specified by brief | Satisfies zero-script maintenance only if managed and health-checked | Must specify runtime, GPU passthrough, restart policy, volume ownership, and UI/stop control; user must not maintain containers manually |

### Model and locality policy

| Workload | Primary route | Fallback/escalation | Local/online reason |
|---|---|---|---|
| Continuous repository scan | Local Ollama/Qwen3.8-27B through existing 9router route | Stop after route failure; human escalation; no silent paid spend | Free local capacity is appropriate for repetitive background work |
| Test generation proposal and routine audit | Local Ollama/Qwen3.8-27B through existing 9router route | Paid route only for defined high-risk or unresolved findings | Preserves cost while retaining escalation capability |
| Approved implementation | Verified paid route associated with OpenCode Go or Merge Gateway, through 9router | Human-selected alternate route if primary unavailable | Paid capacity is reserved for implementation, but exact product capability is not assumed |
| Adversarial disagreement | Local first; paid escalation only under a recorded trigger | Human decision | Avoids spending credits for routine agreement |
| Deterministic checking | No model | Human escalation on infrastructure failure | Machine-checkable tests are independent of model confidence |

`localhost:20128` identifies the fixed proxy layer only. Every execution record must separately identify runtime, provider, model tag, route, local/online classification, timestamp, and correlation ID. RTK compression must not be treated as semantic validation or a substitute for preserving required evidence.

## 3. Responsibility boundaries and interaction

```text
spec.md + loop.md + .loop/loop-state.json
  -> control plane obtains a repository snapshot and exclusive work lease
  -> local worker validates requirements and produces a bounded plan/audit
  -> human approves the milestone plan
  -> implementation worker edits an isolated branch
  -> deterministic runner executes required checks
  -> adversarial worker reviews evidence and drafts the 3-step behavioral test
  -> HITL gate records approve/reject/feedback against exact PR and commit
  -> control plane verifies checks, approval freshness, and branch head
  -> merge occurs only once, under protected policy
  -> post-merge verification writes a new durable checkpoint
```

The control plane is authoritative for transitions. Workers are replaceable executors and cannot advance state by merely returning a successful model response. GitHub is the durable record, not the scheduler. 9router is the fixed model-routing layer, not the worker, orchestrator, test runner, approval system, or recovery system.

## 4. Deterministic state machines

### 4.1 Idea-to-Product loop

```text
IDLE
 -> LOAD_SPEC_AND_STATE
 -> VALIDATE_SPEC
 -> NEEDS_CLARIFICATION | PLAN_MILESTONE
 -> HUMAN_PLAN_GATE
 -> QUEUE_APPROVED_TASK
 -> ACQUIRE_PROJECT_LEASE
 -> IMPLEMENT_ON_ISOLATED_BRANCH
 -> RUN_DETERMINISTIC_CHECKS
 -> RUN_LOCAL_ADVERSARIAL_AUDIT
 -> PREPARE_THREE_STEP_TEST
 -> HUMAN_BEHAVIOR_GATE

HUMAN_BEHAVIOR_GATE
 -> REJECTED: RECORD_REASON -> REPLAN
 -> FEEDBACK: RECORD_FEEDBACK -> REPLAN
 -> APPROVED: VERIFY_FRESH_HEAD_AND_REQUIRED_CHECKS

VERIFY_FRESH_HEAD_AND_REQUIRED_CHECKS
 -> STALE_OR_FAILED: INVALIDATE_APPROVAL_AND_RETRY_OR_ESCALATE
 -> VALID: MERGE_ONCE

MERGE_ONCE
 -> VERIFY_MERGED_COMMIT
 -> CHECKPOINT
 -> NEXT_MILESTONE | RELEASE_READY
```

Required invariants:

- No implementation begins without an approved milestone plan.
- Every task has one project lease, one idempotency key, and one authoritative state writer.
- Human approval is bound to repository, PR number, head commit, milestone ID, and test artifact hash.
- A changed head commit, changed test instructions, new failed check, expired approval, or changed base invalidates approval.
- A model response cannot satisfy a human gate or deterministic check.
- Merge is attempted once per approved head; ambiguous merge results enter reconciliation, not blind retry.

### 4.2 Self-improvement loop

```text
SELF_IDLE
 -> CHECK_ACTIVE_LEASES_AND_BRANCHES
 -> SELECT_BOUNDED_SCOPE
 -> READ_CURRENT_RULES_AND_SNAPSHOT
 -> LOCAL_READ_ONLY_AUDIT
 -> DRAFT_TEST_OR_IMPROVEMENT
 -> RUN_SAFE_CHECKS
 -> PREPARE_ISOLATED_PR
 -> HUMAN_IMPROVEMENT_GATE

HUMAN_IMPROVEMENT_GATE
 -> REJECTED: RECORD_DISREGARDED_OPTION -> RELEASE_LEASE -> SELF_IDLE
 -> FEEDBACK: REVISE_WITH_SAME_SCOPE -> HUMAN_IMPROVEMENT_GATE
 -> APPROVED: VERIFY_FRESH_HEAD_AND_CHECKS -> MERGE_ONCE -> CHECKPOINT -> RELEASE_LEASE -> SELF_IDLE
```

The background worker cannot modify the default branch, weaken tests, change security policy, alter credentials, rewrite `spec.md`, or merge its own proposal. It must stop when active implementation work conflicts with its snapshot or when the repository is recovering.

### 4.3 Recovery and fault-tolerance loop

```text
STARTUP
 -> ACQUIRE_SINGLE_CONTROL_PLANE_LEASE
 -> LOAD_AND_VALIDATE_STATE
 -> RECONCILE_GITHUB_AND_RUNNER
 -> RESUME_LAST_VERIFIED_STATE | HUMAN_ESCALATION_IF_AMBIGUOUS

RUNNING
 -> WRITE_CHECKPOINT_AFTER_EACH_COMPLETED_TRANSITION
 -> COMMIT_STATE_ATOMICALLY

FAILURE
 -> CLASSIFY_TRANSIENT_OR_TERMINAL
 -> RETRY_SAME_IDEMPOTENT_TRANSITION_UP_TO_3_TIMES
 -> HUMAN_ESCALATION_AFTER_THIRD_FAILURE

CRASH_OR_LEASE_EXPIRY
 -> FENCE_OLD_WORKER
 -> RECHECK_BRANCH_HEAD_AND_SIDE_EFFECTS
 -> RESUME_OR_RECONCILE
```

A crash does not prove success or failure. Recovery must reconcile GitHub commit/PR state, CI run state, gate state, and external side effects before continuing. If reconciliation cannot prove a single outcome, stop and escalate; do not issue a duplicate merge, deployment, payment, message, or destructive action.

### Required `.loop/loop-state.json` schema

```json
{
  "schema_version": 1,
  "project_id": "repository identifier",
  "active_loop": "idea_to_product | self_improvement | recovery",
  "state": "stable state name",
  "milestone_id": "stable milestone identifier",
  "spec_sha": "commit SHA",
  "source_sha": "commit SHA",
  "branch": "branch name",
  "pr_number": null,
  "pr_head_sha": null,
  "base_sha": null,
  "test_artifact_sha": null,
  "state_version": 0,
  "lease_owner": null,
  "lease_epoch": 0,
  "attempts": {
    "transition_name": 0
  },
  "last_error": null,
  "last_verified_milestone": null,
  "blocked_reason": null,
  "context_reset_count": 0,
  "updated_at": "RFC-3339 timestamp"
}
```

Durable state rules:

- Exactly one control-plane writer may advance a project at a time.
- `state_version` must increase monotonically; stale writers are rejected.
- `lease_epoch` fences work from a prior process after restart or lease expiry.
- State writes must be atomic and secret-free. A commit is not considered a checkpoint until its SHA is known.
- Ephemeral locks, queues, and leases must have expiry and must not be treated as the only recovery record.
- Retry counters reset only after the transition completes and its checkpoint is verified.
- A context reset reloads the committed checkpoint and does not reset attempts or grant approval.

## 5. Non-technical HITL protocol

Every gate request must contain exactly three sections:

1. **What was built:** one plain-English sentence describing the visible capability.
2. **How to test:** numbered actions using the normal product interface; no terminal, code, logs, or diffs.
3. **Expected result on screen:** the concrete successful result and meaningful failure behavior.

The approval record must bind the decision to `project_id`, `milestone_id`, PR number, PR head SHA, base SHA, test artifact SHA, and timestamp. Approval is invalidated by any change to the PR head, base, required checks, test artifact, implementation scope, or approval freshness window.

A milestone without a user-testable surface is not ready for the normal gate. For backend-only work, the system must provide a temporary visible test surface or explicitly classify the milestone as infrastructure and obtain a separately explained approval; it must never pretend that a non-visible change has an on-screen test.

## 6. Initial verification-gate stress test

### Gate 1 — Repository

**Must prove:** private visibility, protected default branch, least-privilege credentials, required checks, PR approval rules, and that state commits cannot bypass the review policy.

**Gaps addressed:** branch protection alone may not protect state-file writes; bot identities may accidentally approve and merge their own work; force-pushes can invalidate evidence; concurrent workflows can create two active PRs.

**Required tests:** attempt a harmless unauthorized default-branch write, run two concurrent tasks for one project, force-push or change a PR head, and verify that stale approval is invalidated.

### Gate 2 — Routing

**Must prove:** a harmless request through 9router reaches the intended provider/model; local versus online classification is correct; fallback does not silently change model identity or privacy classification.

**Gaps addressed:** endpoint confusion, fallback route drift, RTK-compressed evidence losing metadata, accidental paid usage, and local data leaving the server.

**Required tests:** record provider, model tag, route, locality, timestamp, correlation ID, fallback event, latency, and error; verify a fallback response is labeled and budget-checked before use.

### Gate 3 — Local model

**Must prove:** exact Ollama tag, reproducible health, bounded context, GPU/RAM behavior, concurrency limit, queue backpressure, cancellation, timeout, and recovery after Ollama restart.

**Gaps addressed:** V100 memory exhaustion, model eviction, starvation of implementation work, unbounded queues, stale outputs after restart, and false “24/7” claims.

**Required tests:** run representative audit prompts, concurrency/load test within resource limits, cancel a request, restart Ollama, verify no stale proposal is committed, and confirm the worker stops cleanly when the model is unavailable.

### Gate 4 — Containerized controller and agent

**Must prove:** the container runtime provides GPU passthrough where needed, persistent volumes, non-root/least-privilege execution, health checks, bounded resources, restart behavior, queue persistence, and a visible stop/control path that does not require user-maintained scripts.

**Critical clarification:** “containerized background orchestration” is not itself a product selection. Docker, Podman, Compose, Kubernetes, a hosted runner, or another controller have different operational and security properties. The exact runtime and controller remain unverified until named and tested.

**Agent tests:** read `spec.md`, create an isolated branch, run checks, produce the three-step test artifact, create a PR, survive a worker restart, and avoid direct default-branch writes. Verify that the implementation agent's actual model/provider identity is recorded; do not infer it from OpenCode Go or Merge Gateway names.

### Gate 5 — HITL

**Must prove:** a non-programmer can complete approve, reject, and feedback flows without seeing code, diffs, logs, or terminal instructions.

**Gaps addressed:** vague expected results, backend-only milestones with no visible surface, approval of stale commits, accidental approval of the wrong PR, and ambiguous natural-language decisions.

**Required tests:** perform all three decisions, bind each to exact milestone/PR/head/test hash, alter the head after approval, and verify automatic invalidation and a new plain-English request.

### Gate 6 — Recovery

**Must prove:** reboot/crash recovery resumes only from a committed checkpoint; ambiguous external effects are reconciled; stale workers are fenced; state writes are atomic; no task is duplicated.

**Gaps addressed:** lost state, partial state writes, duplicate PRs/merges, lease races, CI result mismatch, and context-reset confusion.

**Required tests:** interrupt at every side-effect boundary in a non-destructive test project, restart the controller, reconcile GitHub/CI state, and verify one deterministic outcome or human escalation.

### Gate 7 — Escalation

**Must prove:** exactly three attempts occur for retryable failures, terminal failures are not pointlessly retried, logs are redacted, the affected loop stops, and the user receives a plain-English explanation and next decision.

**Gaps addressed:** retry storms, retrying invalid credentials or malformed specs, counter resets, hidden paid spend, and escalation that does not actually stop work.

**Required tests:** inject transient network failure, invalid specification, invalid credentials, provider quota failure, deterministic test failure, and ambiguous merge result; verify classification, attempt count, stop behavior, and escalation record.

## 7. Cross-cutting failure controls

- **Single-writer and leases:** one active controller per project; leases expire; epoch fencing rejects stale workers.
- **Idempotency:** every external write has a stable key; ambiguous writes reconcile before retry.
- **Approval freshness:** approvals bind to exact content and expire on relevant changes or timeout.
- **Backpressure:** bounded queue, per-project concurrency limit, global GPU budget, cancellation, timeout, and dead-letter/escalation state.
- **Privacy routing:** each task declares data classification; local-only tasks cannot fall back online without an explicit policy decision and recorded event.
- **Observability:** correlation ID across model request, router event, worker, CI run, PR, gate, and checkpoint; metrics for queue depth, retries, latency, GPU/RAM, failures, and cost.
- **Stop switch:** one visible control must pause new work and safely drain or cancel active work; emergency stop must prevent further side effects.
- **Secret hygiene:** secrets never enter prompts, Git, state, PR text, or logs; redaction must be tested, not assumed.
- **Clock handling:** timestamps use UTC RFC-3339; approval freshness and leases tolerate clock skew through server-side timestamps where possible.
- **Version pinning:** container images, worker dependencies, model tags, prompt/template versions, and state schema are pinned and recorded.
- **Artifact integrity:** test instructions and results are content-addressed; evidence is tied to the exact source and PR head.
- **Resource isolation:** background local scanning cannot starve implementation, CI, or recovery tasks; quotas and priorities are explicit.

## 8. Log of disregarded options

| Option or assumption | Decision | Reason | Evidence required to revisit |
|---|---|---|---|
| Redesign, replace, or reconfigure 9router | Rejected | Fixed baseline and explicitly off-limits | User changes the constraint |
| Treat `localhost:20128` as a model, scheduler, or GitHub controller | Rejected | It is only the fixed proxy layer | Separate verified capability tests |
| Treat Ollama as the complete orchestrator | Rejected | Inference hosting does not provide state, leases, retries, gates, or recovery | Verified controller integration |
| Treat “containerized orchestration” as a complete design | Rejected as underspecified | Runtime/controller, GPU access, persistence, health, and stop behavior remain unnamed | Named runtime and passing Gate 4 |
| Assume Qwen3.8-27B is an exact installed Ollama tag | Unverified | Brief does not provide tag, quantization, or measured capacity | Runtime inventory and benchmark |
| Use online models for continuous background scanning | Rejected by default | Cost and privacy; local resource is mandated for this workload | Explicit policy and budget approval |
| Treat OpenCode Go as a known API or application | Unverified | Name alone proves no interface, model, quotas, or GitHub permissions | Documentation plus harmless integration test |
| Treat Merge Gateway as a GitHub merge authority | Unverified | Name alone proves no merge capability or identity | Documentation plus end-to-end test |
| Allow model self-merge | Rejected | Violates HITL | Explicit policy change |
| Ask user to inspect code/diffs | Rejected | Violates non-technical constraint | User changes requirement |
| Require user-maintained scripts | Rejected | Violates zero-maintenance constraint | Explicit user change |
| Use green checks as proof of product correctness | Rejected | Checks are necessary but incomplete | Checks plus behavioral test |
| Blindly retry ambiguous external writes | Rejected | Can duplicate merges, deployments, or other effects | Reconciliation evidence |
| Treat context reset as state reset | Rejected | Loses durable progress and can reset retry counters incorrectly | Explicitly versioned checkpoint |
| Run unlimited background work | Rejected | Causes GPU starvation, queue growth, and uncontrolled cost | Bounded scheduler and resource test |

## 9. Three-sentence action summary

The refined baseline correctly assigns continuous, cost-sensitive repository analysis to the local V100/Ollama/Qwen3.8-27B route while preserving 9router as an unchanged proxy and reserving unverified paid routes for approved implementation or escalation. The verification gates are not sufficient until the exact container/controller, model/provider identities, queue and lease semantics, approval freshness, idempotency, privacy fallback policy, and crash-reconciliation behavior are demonstrated. The next implementation step is a non-destructive end-to-end test that exercises all seven gates, records evidence by correlation ID, and proves that every ambiguous or unsafe condition stops at human escalation.

## 10. Continuation prompt for Gemini

```text
Review the latest `loop.md` adversarial gate analysis without assuming that any product name implies capabilities. Focus on resolving the remaining implementation ambiguities:

1. Name the exact container/runtime and control-plane mechanism that satisfies the zero-script maintenance constraint, or explicitly preserve it as unselected if the available evidence is insufficient.
2. Define the authoritative queue, lease, single-writer, fencing, idempotency, and state-checkpoint semantics for `.loop/loop-state.json`.
3. Define the exact model/provider identity and local-versus-online classification required in every execution record, including 9router fallback behavior.
4. Define approval freshness and invalidation rules for changed PR head, base, tests, scope, or required checks.
5. Define retry classification: which failures are retryable, which are terminal, and what happens after exactly three attempts.
6. Define safe reconciliation for ambiguous GitHub, CI, model, and external side effects after a crash.
7. Produce only changes that remain project-agnostic across all six products and preserve the fixed 9router baseline.
8. Do not claim “24/7,” “self-healing,” “production-ready,” or native CLI/API features until a reproducible test proves them.

Return:
- Three sentences summarizing corrections.
- The exact proposed updates to `loop.md`.
- A list of assumptions still requiring verification.
```