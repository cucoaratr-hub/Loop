# Loop — AI Stack Orchestration & Adversarial Iteration Engine

> Status: Corrected architecture proposal
> Last updated: 2026-09-23
> Authority: This file is the living architecture and decision record for the Loop project.

## 1. Objective and constraints

Loop is a repeatable workflow for converting a plain-English `spec.md` into working software across six project types:

1. Newsletter platform.
2. Crypto market analysis tool.
3. Video creator workflow.
4. PDF investor analysis tool.
5. Database builder with audit software.
6. Website builder.

The workflow must remain project-agnostic: the repository-specific input describes what the product must do, while the Loop system supplies the repeatable process for planning, implementation, testing, review, recovery, and improvement.

### User constraints

- The user controls milestones at plain-English approval gates.
- The user must not need to write, read, or maintain code, shell commands, daemon scripts, or Git diffs.
- Every completed milestone must provide: what was built, how to test it through the visible product, and the expected result on screen.
- The user wants automated preparation, not unsupervised merging or release.
- Private GitHub repositories hold source code and the durable project record.
- State must survive reboot or process failure.
- The local server with an NVIDIA V100 and Ollama running the user's specified Qwen3.8-27B model is a required low-cost execution resource.
- OpenCode Go and Merge Gateway are available paid allocations in the user's existing stack.
- 9router is already installed and operational at `localhost:20128`, including the user's configured RTK compression and three-tier fallback. It is fixed and must not be redesigned, replaced, or reconfigured.

### Accuracy rule

The prompt establishes the resources and constraints above. It does **not** establish the existence, feature set, command-line interface, API, scheduler, or GitHub integration of any other named product. Therefore this document distinguishes:

- **Confirmed by the project brief:** a user-provided resource or constraint.
- **Proposed role:** what Loop intends to use a component for.
- **Verification required:** a capability that must be tested or documented before it is treated as real.

No model, provider, endpoint, application, scheduler, or workflow feature may be called production-ready until it has passed an observable verification test.

## 2. System component matrix: what, with what, where, why

| Component | What it does in Loop | Model used | Local or online | Why this choice | Status / proof required |
|---|---|---|---|---|---|
| `spec.md` | Defines desired product behavior and acceptance criteria in plain English | None | Stored in private GitHub; read by the orchestrator | Keeps projects implementation-agnostic and lets the same workflow serve all six products | Required convention; validate path and format |
| `loop.md` | Defines architecture, state machines, constraints, decisions, and rejected options | None | Stored in private GitHub | Single source of truth for the system design and adversarial corrections | This file; current revision |
| GitHub private repository | Stores source, specs, state checkpoints, PRs, evidence, and history | None | Online | Durable version control and recovery trail; user explicitly requires private GitHub | Verify repository privacy, branch protection, and write permissions |
| Orchestrator / control plane | Detects work, loads state, calls models, invokes repository actions, enforces gates, retries failures, and persists checkpoints | Routing depends on task; see model policy below | Execution location not specified by prompt; must be selected and verified | A state machine needs a controller; neither a model nor a proxy is itself a complete controller | Must select a no-code-maintenance execution service; do not invent one |
| 9router | Routes model requests through the user's existing provider bindings, compression, and fallback policy | Depends on selected downstream route | Local network endpoint at `localhost:20128` | Fixed baseline already operational; preserves the user's existing routing investment | Confirmed resource; no redesign permitted |
| Local model runtime | Hosts the local model for inference | Qwen3.8-27B as specified by user; exact installed tag to verify | Local V100 server | Zero incremental model cost and suitable for repeated repository analysis if capacity is adequate | Verify exact Ollama tag, health, context, VRAM/RAM, throughput, and concurrency |
| Local analysis agent | Audits requirements and repository state, proposes tests, identifies defects, and drafts non-destructive improvements | Local Qwen3.8-27B through Ollama and existing 9router route | Local | Background work should use the user's free local compute and avoid consuming paid allocation | Must prove it can produce useful, bounded proposals without merge authority |
| Implementation agent | Converts an approved milestone into code, runs checks, and prepares an isolated PR | Paid route selected from OpenCode Go / Merge Gateway through 9router; exact mapping is not established | Online provider route unless a verified local route is explicitly selected | Implementation may need stronger coding/reasoning capacity than the local background model; paid usage is reserved for high-value work | Verify exact products, API access, quotas, model identity, and PR capability; do not infer from names |
| Review/adversarial analysis | Challenges implementation claims, tests, requirements, and model output | Prefer local Qwen3.8-27B for routine analysis; escalate to an approved paid route for difficult disagreements | Local by default; online only when escalation is approved and routed through 9router | Separates inexpensive continuous checking from scarce paid reasoning capacity | Define escalation trigger and verify both routes |
| Automated checks | Runs tests, lint, type checks, build checks, dependency/security checks, and smoke tests | No model required; models may propose checks but must not replace deterministic checks | Execution environment not specified; likely repository CI or verified runner | Deterministic tools provide evidence that model judgment cannot provide | Select and verify CI/runner; missing checks are failures or explicit gaps |
| Git branch / pull request | Isolates proposed changes and presents evidence for review | None; models create content, GitHub stores it | GitHub online | Provides rollback, review, and auditability without asking the user to inspect diffs | Verify branch protection and PR permissions |
| Human gate | Accepts or rejects a milestone based on visible behavior | None | User-facing interaction, location to be selected | Preserves the required human-in-the-loop control | Must expose only plain-English instructions and results |
| `loop-state.json` | Stores current state, attempts, checkpoints, references, and recovery metadata | None | Committed to private GitHub; transient locks must not be treated as durable state | Enables deterministic restart and prevents guessing after crashes | Define schema, atomic write, single-writer policy, and secret exclusion |
| Audit record | Records decisions, failures, retries, evidence, and rejected options | None | Private Git history plus PR/CI metadata | Makes the process inspectable and recoverable | Define retention and redaction rules |

### Model-routing policy

| Task | Default model and route | Fallback | Reason |
|---|---|---|---|
| Repeated repository scan | Local Qwen3.8-27B via Ollama → existing 9router route | Stop and escalate if local route is unhealthy; do not silently spend paid credits | Lowest incremental cost and adequate for bounded, repetitive analysis if benchmarked |
| Test suggestion and audit draft | Local Qwen3.8-27B via Ollama → existing 9router route | Approved paid route only for unresolved/high-risk findings | Keeps routine background work local |
| Approved feature implementation | Paid route made available by OpenCode Go or Merge Gateway → 9router | User-approved alternate route | Implementation quality and tool capability must be verified before adoption |
| Conflicting model opinions | Local first; paid adversarial review only when the conflict meets escalation criteria | Human decision | Limits cost while preserving a stronger second opinion when needed |
| Deterministic tests and builds | No model | N/A | Tests must produce machine-checkable evidence |
| Secrets/security scanning | Dedicated scanner or repository-native security checks | Human escalation | A language model is not sufficient as the only security control |

The phrase “through 9router” describes routing, not a software application, model, scheduler, or merge authority. The route-to-model mapping must be recorded after a successful harmless request and must include provider, model identifier, timestamp, and result.

## 3. How the components interact

### Control flow

```text
spec.md
  -> Orchestrator loads specification and current loop-state.json
  -> Local Qwen3.8-27B validates completeness and extracts observable behavior
  -> Human approves the milestone plan
  -> Implementation agent receives only the approved milestone and repository context
  -> Agent creates isolated branch/PR and proposes code
  -> Deterministic checks run in the verified CI/runner
  -> Local model audits the result and drafts the plain-English behavioral test
  -> Human tests the visible product behavior
  -> Human approves, rejects, or requests changes
  -> Only approved PRs with required checks may merge
  -> Orchestrator verifies the merge and commits a checkpoint
```

### Responsibility boundaries

- **Models propose and analyze.** They do not become the source of truth, grant themselves permissions, or merge their own work.
- **The orchestrator enforces transitions.** It must own state, retries, locks, gate status, and recovery. A model prompt cannot substitute for these controls.
- **GitHub stores durable artifacts.** It does not automatically provide the complete Loop state machine unless configured to do so.
- **9router routes requests.** It does not automatically provide scheduling, repository mutation, test execution, approval enforcement, or crash recovery.
- **Deterministic tooling produces objective checks.** Passing checks does not prove the feature satisfies the user's intent.
- **The human approves behavior.** Human approval does not replace automated checks or security controls.

## 4. Deterministic state machines

### 4.1 Idea-to-Product loop

```text
IDLE
 -> LOAD_SPEC
 -> VALIDATE_SPEC
 -> NEEDS_DECISION | SPEC_READY

NEEDS_DECISION
 -> HUMAN_SPEC_CLARIFICATION
 -> LOAD_SPEC

SPEC_READY
 -> PLAN_MILESTONE
 -> HUMAN_PLAN_GATE

HUMAN_PLAN_GATE
 -> REJECTED: RECORD_FEEDBACK -> PLAN_MILESTONE
 -> APPROVED: CREATE_IMPLEMENTATION_TASK

CREATE_IMPLEMENTATION_TASK
 -> IMPLEMENT_ON_ISOLATED_BRANCH
 -> RUN_DETERMINISTIC_CHECKS
 -> RUN_MODEL_AUDIT
 -> PREPARE_BEHAVIOR_TEST
 -> HUMAN_BEHAVIOR_GATE

HUMAN_BEHAVIOR_GATE
 -> REJECTED: RECORD_REASON -> CREATE_IMPLEMENTATION_TASK
 -> FEEDBACK: RECORD_FEEDBACK -> CREATE_IMPLEMENTATION_TASK
 -> APPROVED: VERIFY_REQUIRED_CHECKS -> MERGE_PR

VERIFY_REQUIRED_CHECKS
 -> FAIL: RETRY_OR_ESCALATE
 -> PASS: MERGE_PR

MERGE_PR
 -> VERIFY_MERGE
 -> CHECKPOINT
 -> NEXT_MILESTONE | RELEASE_READY

RELEASE_READY
 -> HUMAN_RELEASE_GATE
 -> COMPLETE
```

A state transition is valid only if its preconditions are true and it records an output and checkpoint. A missing test is not a passing test. A model's confidence is not a passing check.

### 4.2 Self-improvement loop

```text
SELF_IDLE
 -> CHECK_ACTIVE_WORK
 -> SELECT_BOUNDED_SCOPE
 -> READ_RULES_AND_SNAPSHOT
 -> LOCAL_READ_ONLY_AUDIT
 -> DRAFT_TEST_OR_IMPROVEMENT
 -> RUN_SAFE_DETERMINISTIC_CHECKS
 -> PREPARE_ISOLATED_PR
 -> HUMAN_IMPROVEMENT_GATE

HUMAN_IMPROVEMENT_GATE
 -> REJECTED: RECORD_DISREGARDED_OPTION -> SELF_IDLE
 -> FEEDBACK: REVISE_PROPOSAL -> HUMAN_IMPROVEMENT_GATE
 -> APPROVED: VERIFY_CHECKS -> MERGE_AFTER_HUMAN_APPROVAL -> CHECKPOINT -> SELF_IDLE
```

The local model may scan, suggest tests, and propose changes. It may not directly modify the default branch, weaken a failing test to obtain green status, alter credentials, or merge without the human gate.

### 4.3 Recovery loop

```text
STARTUP
 -> LOAD_LAST_COMMITTED_STATE
 -> VALIDATE_SCHEMA_AND_REFERENCES
 -> CHECK_REPOSITORY_AND_DEPENDENCIES

VALID
 -> RESUME_FROM_LAST_CHECKPOINT

INVALID_OR_MISSING
 -> RECOVER_FROM_LAST_VERIFIED_MILESTONE
 -> HUMAN_ESCALATION_IF_AMBIGUOUS

RUNNING
 -> CHECKPOINT_AFTER_EACH_COMPLETED_STATE
 -> ATOMICALLY_COMMIT_STATE

FAILURE
 -> RECORD_ERROR
 -> RETRY_SAME_TRANSITION_UP_TO_3_TIMES
 -> HUMAN_ESCALATION_AFTER_THIRD_FAILURE
```

Recovery must never infer that an unverified action succeeded. It must preserve unmerged work and report the last known-good checkpoint. Context reset is allowed at state boundaries, after failure, or when the context budget is exceeded, but a context reset must reload the durable checkpoint and does not erase state.

### Shared state requirements

`loop-state.json` must include, at minimum:

```json
{
  "schema_version": 1,
  "project_id": "repository identifier",
  "active_loop": "idea_to_product | self_improvement | recovery",
  "state": "state name",
  "state_entered_at": "RFC-3339 timestamp",
  "last_checkpoint": "commit or checkpoint identifier",
  "attempts": {},
  "source_spec_sha": "commit SHA",
  "source_code_sha": "commit SHA",
  "active_branch": null,
  "active_pr": null,
  "last_verified_milestone": null,
  "blocked_reason": null,
  "updated_at": "RFC-3339 timestamp"
}
```

The state writer must be single-writer or serialized. Writes must be atomic, secret-free, and recoverable. Retry counters reset only after a successful transition, not after a context reset.

## 5. Non-technical HITL protocol

Every review request must contain exactly these three plain-English sections:

1. **What was built.** Describe the user-visible capability.
2. **How to test.** Give actions using the normal product interface; do not require terminal commands, code, logs, or diffs.
3. **Expected result on screen.** State the observable success and meaningful failure behavior.

Example:

```text
What was built: The website now has a newsletter signup form.
How to test: Open the website, enter an email address, and select Subscribe. Try once with an invalid address.
Expected result on screen: A valid address shows confirmation; an invalid address shows a clear correction message and does not claim that the subscription succeeded.
```

Rules:

- The user can respond with Approve, Reject, or Request changes in ordinary language.
- Approval identifies the milestone and PR/commit, but never requires reading the PR diff.
- Rejection records the reason and leaves the proposal recoverable.
- Security, financial, destructive, privacy-sensitive, or production-impacting actions require an explicit warning and a separate gate.
- Automated checks and human behavioral approval are complementary; neither silently substitutes for the other.
- If a milestone cannot be tested through a visible interface, it is not ready for the normal user gate. The system must first provide a suitable test surface or explain the limitation.

## 6. Retry, security, and operational rules

- Every failed transition may be retried at most three times.
- After the third failure, stop the affected loop and enter `HUMAN_ESCALATION`.
- Record the error category, timestamps, state, attempt number, relevant commit/PR/run ID, and next action.
- External writes must be idempotent or carry a deduplication key.
- Secrets must be supplied through protected configuration and must never appear in Git, prompts, PR bodies, state files, or logs.
- GitHub credentials must have the least privilege required for the verified workflow.
- Local and online model routes must be explicitly labeled in logs and evidence.
- No component may be described as 24/7 until restart, health, backoff, disk, resource, and stop-switch behavior has been tested.
- The user must not be responsible for maintaining custom scripts. If the selected orchestrator requires them, that option fails the constraint.

## 7. Log of disregarded options

| Option or assumption | Decision | Why it is rejected or not yet accepted | Re-entry evidence |
|---|---|---|---|
| Redesign, replace, or reconfigure 9router | Rejected | Explicitly outside scope; the existing endpoint and bindings are fixed | User changes the fixed-baseline constraint |
| Treat `localhost:20128` as a model or scheduler | Rejected | It is a routing endpoint, not proof of a model runtime, scheduler, or GitHub controller | Separate verified health and capability tests |
| Treat Ollama as the complete Loop orchestrator | Rejected | Hosting inference does not prove state management, scheduling, retries, approvals, or GitHub writes | Verified orchestration integration |
| Assume Qwen3.8-27B is an exact installed Ollama tag | Unverified | The prompt gives the model description but not the installed tag, quantization, or capacity | Runtime inventory and benchmark |
| Use the local model for every implementation task by default | Not adopted | The prompt requires local background work but does not prove that the local model is sufficient for all coding tasks | Benchmark against representative milestones |
| Use online models for continuous background scanning | Rejected by cost policy | It would consume paid allocation for work explicitly suited to free local compute | User changes cost priority and approves budget |
| Treat OpenCode Go as a known application/API | Unverified | The prompt names a paid allocation but provides no documented interface or capabilities | Authoritative product documentation plus harmless integration test |
| Treat Merge Gateway as a GitHub merge controller | Unverified | The name does not establish merge authority, repository access, or model identity | Documentation and end-to-end test |
| Let a model merge its own PR | Rejected | Violates human-in-the-loop control | Explicit policy change |
| Ask the user to review code or diffs | Rejected | Violates non-technical constraint | User changes review requirement |
| Require user-maintained Bash/Python daemons | Rejected | Violates no-maintenance constraint | User explicitly accepts maintenance |
| Mark green checks as proof of correctness | Rejected | Automated checks are evidence of configured checks, not complete product validation | Human behavioral test plus adequate checks |
| Guess success after a crash | Rejected | Recovery must resume only from a verified checkpoint | Explicit evidence of the action's result |
| Write implementation directly to the default branch | Rejected | Removes isolated review and rollback controls | Equivalent protected workflow explicitly approved |
| Invent project-specific orchestration logic | Rejected | Violates project-agnostic design | Generalized requirement accepted by human |

## 8. Initial verification gates

The architecture remains a proposal until these tests pass:

1. **Repository:** Confirm private GitHub status, branch protection, required checks, and least-privilege write access.
2. **Routing:** Make one harmless request through the existing 9router route and record the actual downstream provider/model identity.
3. **Local model:** Confirm the exact Ollama model tag, health, context limit, memory use, throughput, and safe concurrency on the V100 server.
4. **Online allocations:** Verify what OpenCode Go and Merge Gateway actually provide; do not infer capabilities from their names.
5. **Controller:** Select and test an orchestrator that can persist state, retry, recover, invoke verified tools, and operate without user-maintained scripts.
6. **Implementation:** Convert a tiny approved milestone into an isolated PR and run deterministic checks.
7. **HITL:** Complete approval, rejection, and feedback cycles using only the three plain-English review sections.
8. **Recovery:** Interrupt a non-destructive run and verify resume from the last committed checkpoint.
9. **Retry:** Force a safe transient failure and verify exactly three attempts followed by human escalation.
10. **Project agnosticism:** Apply the same workflow to representative repositories for all six project types without changing the control logic.

Until these gates pass, use “proposed,” “unverified,” or “requires testing”—not “fully automated,” “self-healing,” “24/7,” or “production-ready.”
