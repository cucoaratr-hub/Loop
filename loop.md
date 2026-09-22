# Loop — AI Stack Orchestration & Adversarial Iteration Engine

> Status: Initial architecture proposal
> Last updated: 2026-09-23
> Purpose: Establish a project-agnostic, human-controlled workflow for turning `spec.md` into verified software while preserving recoverability and auditability.

## Scope and non-negotiable constraints

This document is the operational source of truth for the Loop system.

- The same workflow must support six repository types: newsletter platform, crypto market analysis tool, video creator workflow, PDF investor analysis tool, database builder with audit software, and website builder.
- A project is described in plain Markdown through `spec.md`. The specification states desired behavior, not implementation details.
- The user is a non-programmer. Reviews must use plain-English behavioral tests; code diffs are not a required review artifact.
- Human approval is required at milestone gates. The system may prepare work autonomously, but it must not merge or release without approval.
- GitHub repositories are private and are the durable source for code, specifications, state snapshots, test evidence, and PR history.
- 9router is a fixed upstream proxy layer at `localhost:20128`. This proposal does not redesign, replace, or reconfigure it.
- Local Ollama with the stated Qwen3.8-27B model is intended for continuous low-cost analysis and proposal work, subject to measured capacity and reliability validation.
- OpenCode Go and Merge Gateway are treated as paid allocations available through the existing 9router configuration; their exact product capabilities and limits must be verified before implementation.
- No requirement in this document permits the user to write, maintain, or operate custom Bash/Python daemon scripts.

## System Component Matrix

| Component | Exact software or service | Endpoint or location | Operational role | Verification status |
|---|---|---|---|---|
| Version control | GitHub private repository | Repository-specific GitHub URL | Stores source, `spec.md`, `loop.md`, state snapshots, test evidence, and PR history | Target repository access verified; workflow permissions still require testing |
| Source of truth | `loop.md` | Repository root | Records architecture, state machines, HITL rules, assumptions, and rejected options | This proposal |
| Project input | `spec.md` | Repository root or agreed project path | Plain-English behavioral requirements; implementation-neutral input | Required convention; exact path policy to be finalized |
| Proxy layer | 9router | `http://localhost:20128` | Routes downstream model requests, applies configured compression, and provides the existing fallback policy | Fixed baseline; not redesigned here |
| Local model runtime | Ollama | Local server; endpoint must be confirmed from deployment | Hosts the local Qwen3.8-27B model for background audits, test suggestions, and non-destructive proposals | Model name, context limits, concurrency, and health behavior require runtime verification |
| Local model | Qwen3.8-27B | Through Ollama and the existing 9router route | Performs inexpensive background analysis, static-review assistance, test generation proposals, and improvement audits | User-provided requirement; exact published model identifier must be confirmed |
| Coding agent/orchestrator | OpenCode or the selected agent application | Through 9router | Converts approved tasks into repository changes, runs configured checks, and prepares PRs | Exact application, version, permissions, and supported provider configuration require verification |
| Paid model allocation | OpenCode Go | Through existing 9router binding | Supplies the paid coding/reasoning capacity assigned to implementation work | Product scope and limits require verification; do not assume native features |
| Paid model allocation | Merge Gateway | Through existing 9router binding | Supplies the paid model credit allocation assigned to implementation or review work | Product scope and limits require verification; do not assume it is a GitHub merge controller |
| CI and checks | GitHub Actions or another existing repository-native CI system | Private repository | Runs deterministic tests, linting, type checks, security checks, and build checks | Selection requires capability and cost verification |
| Review artifact | GitHub Pull Request | Private repository | Carries proposed changes, automated evidence, plain-English test instructions, and approval status | Required workflow artifact |
| Durable state | `loop-state.json` | Repository-controlled state path, exact location to be finalized | Stores resumable state, attempt counters, IDs, timestamps, and last successful checkpoint | Schema and write strategy require implementation design |
| Audit trail | Git history, PRs, CI results, and append-only log entries | Private repository | Provides traceability for decisions, executions, failures, and approvals | Retention and secret-redaction policy require finalization |

### Matrix accuracy rules

- An endpoint is not an application. `localhost:20128` identifies the fixed 9router proxy, not OpenCode, Ollama, GitHub, or a model.
- A model is not a provider. Every model entry must name the runtime/provider route actually used.
- A component may not be marked verified merely because it is planned. Verification must include a reproducible health check or an observed successful operation.
- Credentials, tokens, and secrets must never be committed to `loop.md`, `loop-state.json`, logs, PR bodies, or test artifacts.

## Deterministic Loop State Machines

### Shared state model

Every transition has a stable state name, an allowed input, an observable output, and a checkpoint. A failed transition increments its attempt counter. The default retry limit is three attempts; after the third failed attempt the system enters `HUMAN_ESCALATION` and stops making autonomous changes for that task.

Suggested durable fields for `loop-state.json`:

```json
{
  "schema_version": 1,
  "project_id": "repository identifier",
  "active_loop": "idea_to_product | self_improvement | recovery",
  "state": "state name",
  "state_entered_at": "RFC-3339 timestamp",
  "last_checkpoint": "checkpoint identifier",
  "attempts": {"state name": 0},
  "blocked_reason": null,
  "source_commit": "commit SHA",
  "active_pr": null,
  "last_verified_milestone": null,
  "context_reset_count": 0,
  "updated_at": "RFC-3339 timestamp"
}
```

The state file is operational metadata, not a secret store. Updates must be atomic, committed, and recoverable. If concurrent agents can write state, a single-writer rule or serialized queue is mandatory.

### Idea-to-Product Loop

```text
IDLE
  -> INGEST_SPEC
  -> VALIDATE_SPEC
  -> HUMAN_SPEC_GATE
  -> PLAN_MILESTONE
  -> HUMAN_PLAN_GATE
  -> IMPLEMENT
  -> RUN_AUTOMATED_CHECKS
  -> PREPARE_PLAIN_ENGLISH_TEST
  -> HUMAN_MILESTONE_GATE
      -> REJECTED: RECORD_FEEDBACK -> PLAN_MILESTONE
      -> APPROVED: MERGE_MILESTONE
  -> VERIFY_MERGE
      -> SUCCESS: CHECKPOINT -> NEXT_MILESTONE
      -> FAILURE: RETRY_OR_ESCALATE
  -> RELEASE_READY
  -> HUMAN_RELEASE_GATE
  -> COMPLETE
```

Transition rules:

1. `INGEST_SPEC` reads only the agreed `spec.md` and records its commit SHA.
2. `VALIDATE_SPEC` checks for missing behavior, ambiguous acceptance criteria, unsafe requests, and dependencies. It does not invent requirements silently.
3. `HUMAN_SPEC_GATE` presents missing decisions in plain English. The loop cannot plan until the user approves or supplies corrections.
4. `PLAN_MILESTONE` creates a small, observable milestone with acceptance criteria, affected capability, rollback expectation, and test instructions.
5. `HUMAN_PLAN_GATE` requires approval before implementation begins.
6. `IMPLEMENT` may use the approved agent stack, but changes must be isolated to a branch or PR and must not silently alter `spec.md` or `loop.md` requirements.
7. `RUN_AUTOMATED_CHECKS` executes only repository-configured checks. Missing checks are reported as a gap, not treated as success.
8. `PREPARE_PLAIN_ENGLISH_TEST` produces exactly three items: what was built, how the user tests it, and what should appear on screen. A fourth item may be added only when safety or setup requires it.
9. `HUMAN_MILESTONE_GATE` accepts approval, rejection, or feedback. Approval must identify the milestone and commit/PR being approved.
10. `MERGE_MILESTONE` is allowed only after the human gate and required automated checks succeed.
11. `VERIFY_MERGE` confirms the merged commit, expected behavior evidence, and a durable checkpoint.
12. Any failure follows the retry and escalation rules below.

### Self-Improvement Loop

The local model may work only on a clean, known repository snapshot and only within a bounded proposal budget.

```text
SELF_IDLE
  -> SELECT_AUDIT_SCOPE
  -> READ_REPOSITORY_RULES
  -> RUN_READ_ONLY_ANALYSIS
  -> PROPOSE_TEST_OR_IMPROVEMENT
  -> RUN_SAFE_CHECKS
  -> PREPARE_NON_DESTRUCTIVE_PR
  -> HUMAN_IMPROVEMENT_GATE
      -> REJECTED: ARCHIVE_REASON -> SELF_IDLE
      -> APPROVED: MERGE_AFTER_REQUIRED_CHECKS -> CHECKPOINT -> SELF_IDLE
      -> FEEDBACK: REVISE_PROPOSAL -> HUMAN_IMPROVEMENT_GATE
```

Rules:

- The local model must not merge its own work, weaken tests, change security controls, modify credentials, or change the governing requirements without a human gate.
- Every proposal must state the observed problem, evidence, proposed change, expected benefit, risk, rollback, and plain-English test.
- Idle-time work is suspended when an active human-approved implementation milestone has a conflicting branch, when CI is failing for an unrelated reason, or when the repository is in recovery.
- “Continuous” means repeated scheduled or event-driven execution by a verified host/orchestration service. The exact service must be selected and verified; this document does not assume that Ollama itself schedules repository work.
- The proposal must be a PR or equivalent reviewable GitHub change. Direct writes to the default branch are prohibited.

### Recovery and fault-tolerance loop

```text
STARTUP
  -> LOAD_STATE
  -> VALIDATE_STATE
      -> VALID: RESUME_CHECKPOINT
      -> INVALID/MISSING: RECOVER_FROM_GIT_HISTORY
  -> CHECK_DEPENDENCIES
      -> AVAILABLE: RESUME_STATE
      -> UNAVAILABLE: RETRY_OR_ESCALATE
  -> EXECUTE_NEXT_TRANSITION
  -> WRITE_CHECKPOINT
  -> COMMIT_STATE
```

Recovery rules:

- On reboot or crash, load the last committed `loop-state.json` and verify that its referenced commit, branch, and PR still exist.
- If state is missing or malformed, recover from the latest valid Git commit, PR metadata, CI results, and the last human-approved milestone. Never guess a completed transition.
- A transition may be retried at most three times. Retries must record timestamps, error category, and the relevant commit or run identifier.
- After three failures, enter `HUMAN_ESCALATION`, preserve logs, stop the affected loop, and explain the recovery choice in plain English.
- Context resets occur at state boundaries, after a failed attempt, when the context budget is insufficient, or when stale instructions are detected. The next context receives a compact checkpoint containing the specification SHA, state, attempts, active PR, last verified milestone, unresolved decisions, and required tests.
- Context reset is not state reset. Durable state remains authoritative.
- Partial changes must remain isolated and recoverable. The recovery procedure must never automatically delete unreviewed work.
- External side effects must be idempotent or have an explicit deduplication key.

## Non-Technical HITL Protocol

The user reviews outcomes and behavior, not source code, syntax, diffs, terminal output, or implementation details.

### Required milestone review format

Every review request must contain:

1. **What was built:** one short plain-English description.
2. **How to test:** numbered actions using the normal user interface, with no terminal commands.
3. **Expected result on screen:** concrete observable results, including success and meaningful failure behavior.

Example:

```text
What was built: A newsletter signup form that validates an email address.
How to test: Open the signup page, enter a valid email, and select Subscribe. Repeat with an invalid email.
Expected result on screen: A valid address shows a confirmation message; an invalid address shows an understandable correction message and does not claim success.
```

### Gate rules

- The user may choose `Approve`, `Reject`, or `Request changes` using plain language.
- `Approve` is valid only when the review identifies the exact milestone and PR/commit.
- `Reject` must preserve the proposal and record the reason in the audit trail.
- Feedback must become a tracked requirement or decision before implementation resumes.
- If the user cannot test a milestone through a visible interface, the system must explain the missing test surface and request a simpler or better-observable milestone.
- Automated green checks never replace human behavioral approval.
- Human approval never replaces required automated checks.
- Any security, financial, destructive, privacy-sensitive, or production-impacting behavior requires an explicit warning and a separate approval gate.
- The system must not ask the user to inspect `Files Changed`, Git diffs, code, logs, shell output, or configuration syntax.

## Verification and evidence policy

Each milestone must retain:

- The input `spec.md` commit SHA.
- The implementation branch and PR number.
- Automated check names, run IDs, and results.
- The exact three-part behavioral test instruction.
- The user's gate decision and timestamp.
- The merged commit SHA, if merged.
- Any retry, escalation, rollback, or rejected-option record.

Evidence must be reproducible from the private repository and must exclude secrets and personal credentials.

## Technical risks and unresolved decisions

These are intentionally not presented as solved facts:

- The exact identity and feature set of “OpenCode Go” must be verified against authoritative product documentation.
- The exact identity and feature set of “Merge Gateway” must be verified; the name alone does not establish that it performs GitHub merges.
- `Qwen3.8-27B` must be mapped to the exact Ollama model tag actually installed. Context size, quantization, VRAM/RAM needs, throughput, and concurrent-request behavior require measurement on the V100 host.
- The orchestrator and scheduler have not been selected. A scheduler must support restart recovery, serialized state writes, retries, secrets handling, and GitHub integration without requiring the user to maintain scripts.
- GitHub Actions may be unsuitable for continuous local-model work unless a secure self-hosted runner or another verified execution service exists.
- “24/7” operation must include health monitoring, backoff, resource limits, disk/log retention, and a clear stop switch.
- Six repositories versus one repository per project must be explicitly decided while preserving the same `spec.md` and state conventions.
- Merge authority, branch protection, required checks, and human approval enforcement must be configured and tested rather than assumed.

## Log of Disregarded Options

| Option or claim | Status | Reason for rejection or non-adoption | Evidence required to revisit |
|---|---|---|---|
| Redesigning or replacing 9router | Rejected by constraint | 9router is a fixed, operational baseline and is explicitly off-limits | User explicitly changes the baseline |
| Treating `localhost:20128` as Ollama or as a model | Rejected | It is the configured proxy endpoint, not proof of a model runtime or application identity | Runtime health evidence identifying each service |
| Treating a model name as a provider/application | Rejected | Model identity, runtime, and routing endpoint are separate components | Exact installed tag and successful routed request |
| Direct autonomous merges by the local model | Rejected | Violates human-in-the-loop control and makes rollback/accountability weaker | Explicit user policy change plus tested safeguards |
| User review through Git diffs or `Files Changed` | Rejected | Violates the non-technical constraint | User explicitly changes review requirements |
| User-maintained Bash/Python daemon scripts | Rejected | Violates the no-maintenance constraint | A managed visual/service-based alternative is not available and user explicitly accepts maintenance |
| Assuming Ollama schedules repository work | Rejected | A model runtime does not itself prove scheduling, retries, or recovery behavior | Verified scheduler/orchestrator documentation and test |
| Treating continuous execution as automatically guaranteed | Rejected | 24/7 behavior needs host uptime, scheduling, health checks, persistence, and resource controls | Successful reboot/crash recovery test |
| Assuming paid product names imply capabilities | Deferred/rejected as assumption | Product names alone do not verify API access, coding-agent integration, quotas, or merge authority | Authoritative documentation and successful integration test |
| Marking all components production-ready before tests | Rejected | Planning statements are not operational verification | Reproducible health, failure, and recovery tests |
| Direct default-branch writes for implementation | Rejected | Bypasses review, rollback, and required milestone gates | Explicit exception with equivalent approval/audit controls |
| Silent invention of requirements from ambiguous `spec.md` | Rejected | Violates project-agnostic behavior and creates unapproved scope | Human decision recorded in the project decision log |

## Initial implementation gates

The architecture is not production-ready until these gates are passed:

1. **Repository gate:** Confirm private repository settings, default branch, branch protection, and required status checks.
2. **Routing gate:** Demonstrate one harmless request through 9router to each intended downstream route without changing 9router configuration.
3. **Local-model gate:** Demonstrate the exact installed Ollama tag, health check, bounded context behavior, and measured concurrency on the V100 host.
4. **Agent gate:** Demonstrate that the selected coding agent can read `spec.md`, create an isolated proposal, run checks, and create a PR without requiring the user to operate a terminal.
5. **HITL gate:** Demonstrate an approval, rejection, and feedback cycle using only plain-English behavioral instructions.
6. **Recovery gate:** Interrupt a non-destructive run, reboot or restart the execution service, and verify deterministic resume from committed state.
7. **Retry gate:** Force a safe transient failure and verify exactly three attempts followed by human escalation.
8. **Security gate:** Verify secret handling, least-privilege GitHub credentials, redacted logs, and no secret persistence in repository artifacts.
9. **Project-template gate:** Run the same conventions against six empty or representative repositories without adding project-specific orchestration logic.

Until these gates pass, claims such as “fully automated,” “24/7,” “self-healing,” or “production-ready” must not be used as verified descriptions.
