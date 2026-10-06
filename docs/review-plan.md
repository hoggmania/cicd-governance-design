# Delivery and architecture review plan

> **For Hermes:** Use subagent-driven-development when implementation is authorized; expand each work package into test-first tasks after the runtime and CI platform are selected. This update does not authorize implementation, commits, publication or security-setting changes.

**Goal:** Deliver a self-hosted, hierarchical CI/CD gate-governance service with verified CEL policies and explainable signed decisions.

**Architecture:** Git-approved CEL archives are distributed through S3/Artifactory with in-toto/DSSE provenance and release authorization. PDPs periodically verify and activate policies, evaluate verified evidence and live bypass status, and issue signed decisions enforced by CI/CD PEPs. Vault/OpenBao Transit manages separate signing keys; audit and notifications span the lifecycle.

**Tech stack:** CEL, in-toto Statement v1, DSSE, Vault or OpenBao Transit, S3 or Artifactory; application language, CEL library, database implementation and first CI platform remain selections, not assumed dependencies.

## Scope and current status

- Complete: architecture and review documentation, attestation/decision contracts at design level, rendered C4 and sequence diagrams.
- Not started: runnable services, machine-readable schemas, real Transit signing integration, publication jobs, PDP loader/evaluator, PEP adapter and operational tests.
- First pilot: one program, one application repository and one selected formal gate. Prove the complete trust chain before adding platforms or broader coverage.
- Required duties: Security Manager authors policy in Git; Application Manager requests bypass; Security Reviewer approves/denies. Git policy-approval authority remains a separately assigned permission to confirm.
- Required behavior: exact policy/rule provenance, all known blockers and pending ownership; scoped time-bounded or explicit no-expiry bypasses; audit and interested-party emails; no unsigned or incomplete permit.
- Source specifications: [architecture](architecture.md), [attestation model](attestation-model.md), [policy distribution](policy-distribution.md), [decision contract](decision-contract.md), [governance workflow](governance-workflow.md).

## Dependency-ordered work packages

These are delivery packages, not estimates or claims that a service can be built in a single task. Owners are proposed role assignments, not named commitments. All package statuses are **not started**.

| ID | Package / proposed owner | Depends on | Deliverables and exit evidence |
|---|---|---|---|
| W0 | Scope and architecture decisions / Application Manager + security/platform owners | None | First gate/platform, scope roles, selected runtime/backend/provider, trust/freshness/retention targets and accepted threat model |
| W1 | Contracts and fixtures / API + security engineering | W0 | Versioned CEL inputs, custom predicate schemas, decision API and valid/invalid fixtures; field/subject/digest rules reviewed |
| W2 | Transit and trust integration / platform security | W1 | Real non-production DSSE signing adapter, distinct producer keys/permissions, independent public trust distribution; cross-library verification, rotation and revocation tests pass |
| W3 | Audit and notification foundation / backend engineering | W1 | Durable audit, issuance journal, inbox/outbox, authorized recipients, email test sink, retries and reconciliation; crash/failure fixtures prove no silent loss |
| W4 | Git-to-archive policy release / CI platform engineering | W2, W3 | Protected-review release path, CEL validation, reproducible archive, signed provenance and release authorization; storage read-back verified before advertisement |
| W5 | PDP pickup and activation / PDP engineering | W4 | Periodic polling, full attestation verification, extraction limits, CEL compatibility, persisted anti-rollback, trusted bootstrap floor and atomic snapshot activation |
| W6 | Evidence and bypass workflow / governance engineering | W2, W3 | Verified normalized evidence, scoped role enforcement, signed review issuance, live revocation, expiry, no-expiry safeguards and pending ownership |
| W7 | Explainable signed decisions / PDP engineering | W5, W6 | All rule results, exact versions/digests, remediation and pending tasks; deterministic aggregation, safe projection, Transit signing and durable issued envelope |
| W8 | CI/CD PEP and receipts / CI platform engineering | W7 | Verify decision signature/authority/run/audience/expiry; block every non-permit; durable enforcement intent and signed receipt with safe retry |
| W9 | Integrated shadow and enforcement pilot / QA + platform owner | W8 | Real end-to-end trace, shadow observations, non-production bypass attacks, outage/recovery evidence and separately approved enforcement enablement |
| W10 | Operational readiness and expansion / service owner | W9 | Load/HA/recovery tests, key rotation and compromise drill, dashboards, support/runbooks and scope-by-scope onboarding approval |

Parallelism: W2 and W3 can proceed independently after contracts; W6 can proceed alongside W4/W5 after its prerequisites. W7 must integrate real loader and workflow behavior, not guessed interfaces. W9 cannot pass on mocked signing, an unprotected pipeline or only a happy-path demonstration.

### Proposed planning artifacts

These paths are planned additions, not existing implementation files:

- W0: `docs/decisions/platform-selections.md`, `docs/threat-model.md`.
- W1: `schemas/policy-release-v1.schema.json`, `schemas/bypass-review-v1.schema.json`, `schemas/gate-decision-v1.schema.json`, `schemas/enforcement-receipt-v1.schema.json`, `schemas/cel-input-v1.schema.json`, `api/openapi.yaml`, `tests/fixtures/attestations/`.
- W2: `docs/runbooks/signing-and-trust.md`; implementation paths selected after the runtime decision.
- W3: `docs/runbooks/audit-and-notifications.md`; data migrations and executable test paths selected with the service stack.
- W9/W10: `docs/verification/pilot-report.md`, `docs/runbooks/recovery.md`, `docs/runbooks/onboarding.md`.

Before coding each package: identify exact implementation/test paths in the selected scaffold; write the negative test, run and confirm its failure, implement minimally, rerun focused and regression tests, then retain actual output. Record exact build/test commands once the language and toolchain exist; do not invent commands or passing results for this documentation-only project. Commit only when authorized.

Status: proposed; estimates and technology choices intentionally deferred until platform and first gate are agreed. No implementation work is authorized by this plan alone.

## Phase 0 — Agree governance and scope

Participants: Security Manager, Application Manager, Security Reviewer, Project Lead, CI/CD platform owner and audit representative.

Outputs: first program/repository/gate, authority matrix, inheritance rules, approval and exception rules, evidence trust list, service targets and deployment boundary.

Exit: W0 approved by named owners, residual bypass risks recorded, and test environments/approved access identified. Provisioning keys, ACLs, branch protection or live email routes requires explicit approval; this plan grants none.

## Phase 1 — Policy model and executable design

In-toto Statement v1, DSSE and Vault/OpenBao Transit are architecture decisions. Specify custom policy-release, bypass-review, gate-decision and enforcement-receipt predicates; reuse standard provenance/evidence predicates. Prove DSSE PAE and Transit encoding interoperability, independent public trust distribution, rotation and revocation. See [attestation model](attestation-model.md).

Define schemas for hierarchy, CEL controls, signed release envelopes, evidence, approvals, exceptions and decisions. Specify PDP request/response and PEP behavior. Select a CEL runtime and signing library; prototype typed CEL evaluation, cost limits, composition and explanation. Design protected Git approval, archive packaging, S3/Artifactory publication, independent key trust, periodic pickup, freshness limits, anti-rollback state and atomic per-instance activation.

Exit: W1-W3 contracts and real signing/trust/audit prototypes accepted. Fixture tests demonstrate policy inheritance, issuer/predicate separation, DSSE PAE interoperability, scoped roles, durable issuance and observable notification failure. Custom predicate URIs and schemas are versioned before publishing production attestations.

## Phase 2 — Thin vertical slice

Implement one program, one repository and one formal gate. Include Security Manager CEL authoring in Git, independent PR approval, post-merge archive build/sign/publication to S3 or Artifactory, periodic PDP retrieval and signature verification before activation. Include Application Manager bypass requests, Security Reviewer approve/deny workflow, CEL evaluation, trusted evidence ingestion and a CI adapter. Audit every action and persist email notification events through a transactional outbox. Support explicit time-bounded and no-expiry requests, with expiry/revocation checks and approved recipient routing. These governance workflows are mandatory in the first slice, regardless of whether the gate separately requires release approval.

Run in shadow mode on the selected pipeline; record would-block outcomes without claiming governance is enforced.

The vertical slice must issue and verify real DSSE attestations for policy provenance/release, selected evidence, bypass review, PDP decision and PEP receipt. Use distinct Transit keys/permissions, durable issuance states and locally verified signatures. No simulated signing output satisfies implementation acceptance.

Exit: W4-W8 integrate in a non-production environment. A reviewer traces Git approval → signed policy release → verified activation → evidence/bypass review → signed decision → verified PEP behavior → receipt/audit/email. Exercise permit, deny, pending and indeterminate; include denied rules and pending team/assignee details in retained decision evidence. Shadow mode does not claim production enforcement.

## Phase 3 — Protected enforcement pilot

Complete W9: protect the gate job and its success signal; configure deployment/merge controls under separate explicit approval. Verify the approvals, exceptions and replay protections already built in the vertical slice, rather than deferring them to this phase. Test outages, privilege bypasses and recovery in non-production before any production rollout.

Exit: only a valid subject-bound permit can advance the selected protected action, within the documented platform trust boundary; all non-permit outcomes block.

## Phase 4 — Controlled expansion

Add repository onboarding, optional branch scopes, further gates and CI platforms, impact simulation, reporting, redundant PDP deployment, recovery testing and audit export. Roll out by program with named operational ownership and rollback procedures.

Exit: W10 operational acceptance. Each onboarded scope has verified enforcement coverage, support ownership, accepted service targets, tested recovery and key lifecycle drills. Expansion does not weaken inherited controls or introduce unsigned compatibility fallbacks.

## Mandatory acceptance scenarios

| Scenario | Expected result |
|---|---|
| Program requires security evidence; repository attempts to disable it | Publication rejected or inherited requirement retained; no weakening |
| No branch scope exists | Repository and program requirements still apply |
| Multiple branch patterns match | All applicable requirements evaluated; conflicts visible |
| Evidence is missing, stale, forged or for another commit | No permit |
| Artifact tag points to a new digest after approval | Old approval/permit cannot authorize new digest |
| Author attempts to approve own protected policy or exception | Rejected |
| Application Manager or Project Lead attempts policy authoring | Rejected |
| Security Manager or Application Manager attempts bypass approval | Rejected |
| Security Reviewer attempts policy authoring or requests a bypass | Rejected |
| Conflicting roles are granted directly or through groups for overlapping scope | Rejected |
| Requester changes role and attempts to review own request | Rejected |
| Reviewer lacks authority for an inherited program control | Review rejected |
| Bypass approved or denied | Immutable reasoned decision, audit record and authorized recipient notification events |
| Time-bounded bypass is before start or at/after end | Not applied, regardless of scheduler delay |
| Request omits end time without explicit no-expiry mode | Validation rejects ambiguous validity |
| Explicit no-expiry bypass approved | Remains scoped and revocable; periodic review scheduled |
| Bypass extension or scope change is pending | Existing approval not extended or broadened |
| Approve/deny/withdraw requests race | Only one revision-checked transition succeeds |
| Referenced policy control version changes | Old bypass not applied to new version without reapproval |
| Action occurs, including rejected attempt or authorized read | Audit and authorized notification event recorded |
| Audit/outbox transaction fails | No mutation or permit succeeds |
| Email relay unavailable or recipient unresolved | Event retained, bounded retry or unresolved queue, operational alert; action not replayed |
| Email retry or delivery-status update occurs | Audited without recursive email generation |
| Recipient loses access before delivery | Sensitive email suppressed and delivery decision audited |
| Exception expires or is revoked | Subsequent decisions enforce original requirement |
| Pipeline changes requested repository or target branch claims | Verified identity mapping prevents scope impersonation |
| PDP/evidence/audit service unavailable | Protected action blocks with actionable diagnostic |
| Valid approval exists but another mandatory control fails | Deny; approval does not override unrelated controls |
| Permit replayed for another run, environment or artifact | Rejected |
| Pipeline skips check or forges success | Platform controls prevent advancement; residual gaps documented |
| Policy publication occurs during evaluation | Decision uses one internally consistent version snapshot |
| Policy rollback | New audited activation; earlier versions and decisions retained |
| Historical decision is investigated | Policy, subject, evidence, approvers and exception use explainable |
| PR approval is stale or required checks fail | Merge/release blocked; signing job unavailable to untrusted PR |
| Archive signature missing/invalid, digest changed or signer untrusted | Candidate rejected before extraction/CEL parsing; audit and notification |
| Archive supplies its own signing key | Rejected unless independently provisioned trust already authorizes signer |
| Correctly signed archive contains traversal, links or decompression bomb | Rejected under extraction limits |
| CEL type/runtime/schema incompatible or evaluation exceeds cost limit | Activation rejected or evaluation blocks; never permit on error |
| Older valid release or equal sequence with different digest appears | Rejected by durable anti-rollback checks |
| New PDP starts without trusted release floor | Not ready; cannot bootstrap solely from mutable storage pointer |
| Artifact storage unavailable | Eligible last-known-good bundle continues only within explicit trust/freshness limits |
| Bundle expired, signer revoked or freshness limit exceeded | No permit; stale/invalid active bundle cannot continue |
| Upload incomplete or read-back verification fails | Discovery pointer not advanced |
| Crash during activation | Recovery reconciles journal before serving; no mixed snapshot |
| Replicas activate at different times | Active digest reported; stale instances drained under release-floor policy |
| Rollback required | New Git-approved signed release with higher sequence |
| Git/CI event delivery missed or repeated | Reconciliation recovers missing events; ingestion deduplicates audit/notification records |
| Bypass revoked between policy polls | Dynamic exception check prevents further use independent of bundle polling |
| Decision returned | Exact archive digest, release sequence, source commit and policy/rule versions, or explicit unavailability reason |
| Multiple rules deny | Every known blocking rule returned with evidence references, reason and remediation |
| Denial and approval tasks coexist | Denial remains blocking; all pending tasks and responsible parties remain visible |
| Pending bypass belongs to a team queue | Security Reviewer group shown; no invented individual assignee |
| Review task assigned, overdue or unassigned | Actual assignment, quorum, deadlines and next action are explicit |
| Approval completes after a pending decision | New evaluation and decision ID; original record and pending-owner snapshot immutable |
| Evaluation cost limit prevents complete rule evaluation | Unevaluated rules marked explicitly; no permit |
| CI caller lacks access to reviewer PII or evidence details | Safe explanation, explicit redaction and authorized detail link |
| Large decision requires pagination | Explicit totals/completeness and immutable detail reference; no silent truncation |

## Review meeting sequence

Additional mandatory acceptance: reject valid signatures from the wrong predicate/scope issuer; reject missing or inconsistent policy provenance/release attestations; block new permits during Transit signing failure; verify retired/compromised-key policy; reject unsigned or modified decisions; preserve signed pending-owner details; reject replay across runs; verify live bypass revocation despite an authentic approval attestation; retain pending receipt delivery without rerunning actions; prove public-key verification works without a live Transit call within freshness limits.

1. Confirm scope, hierarchy and accountable roles.
2. Walk through one successful release and one rejected release.
3. Challenge inheritance, delegation, human approval and exception semantics.
4. Trace workload identity, evidence provenance and pipeline bypass paths.
5. Agree outage behavior, service targets and retention.
6. Record accepted decisions, required changes, owners and the first vertical slice.

## Review decision register

| Decision | Proposed default | Status |
|---|---|---|
| Hosting | Self-hosted control plane and PDP | Pending review |
| Repository membership | One program per repository | Pending review |
| Policy inheritance | Additive requirements; no silent weakening | Pending review |
| Branch scope | Optional; all matching scopes contribute | Pending review |
| Non-permit behavior | Block protected actions | Pending review |
| Policy author | Security Manager, not application/repository owners | User requirement |
| Bypass requester | Application Manager | User requirement |
| Bypass decision | Security Reviewer approves or denies | User requirement |
| Optional bypass period | Explicit time-bounded or no-expiry mode | Supported; safeguards pending review |
| Action accountability | Every action audited and emailed to authorized interested parties | User requirement |
| Policy publication and revocation | Separate Security Reviewer permissions | Pending review |
| First platform/gate | Select during Phase 0 | Open |
| Policy language | CEL | User requirement |
| Policy release path | Git approval, signed archive, S3 or Artifactory, periodic PDP verification | User requirement |
| Attestation format | In-toto Statement v1 with DSSE | Adopted |
| Key management | Vault or OpenBao Transit, separate producer keys | User requirement |
| CEL runtime, DSSE/Transit adapter, algorithm and storage backend | Select through Phase 1 spike | Open |
| Polling, freshness, key trust and release floor | Explicit validated operational limits | Pending review |
| Emergency release process | Explicit, bounded, audited authorization | Pending review |

## Completion boundary

This project initially delivers a reviewed design, not a running governance service. Implementation completion requires the applicable acceptance scenarios to pass against the selected real CI/CD integration, with evidence retained in the project.
