# Explainable PDP decision contract

Status: required design contract, not an implemented API. Field names below are proposed. Every decision must explain what was evaluated, which policy and rules were used, why it permits or blocks, and who or what must act next.

## 1. Decision envelope

The wire authorization artifact is an in-toto Statement v1 inside a DSSE envelope, signed using the PDP-specific Vault/OpenBao Transit key. The custom `gate-decision/v1` predicate carries this contract (or its authorized safe projection plus immutable detail digest). It is a project schema, not a standard predicate; its controlled versioned URI is selected before implementation. See [attestation model](attestation-model.md).

Statement `subject` identifies the artifact or source snapshot by digest; the contract's contextual `subject` below lives inside the predicate and binds repository, commit, run, gate and environment. PEP verifies both. Record input attestation digests, predicate types, signer/key versions and verification outcomes alongside each policy/evidence/exception reference. All issued outcomes are signed; failure to sign returns a blocking service error, never an unsigned permit.

| Field/group | Required meaning |
|---|---|
| `schema_version` | Version of the response contract, independent of policy version |
| `decision_id`, `request_id`, `correlation_id` | Stable identifiers for audit, caller correlation and distributed tracing |
| `outcome` | `PERMIT`, `DENY`, `PENDING_APPROVAL` or `INDETERMINATE` |
| `enforcement` | `ALLOW` only for `PERMIT`; otherwise `BLOCK` |
| `summary`, `reason_codes` | Human-readable explanation and stable machine-readable reasons; never infer outcomes by parsing prose |
| `evaluated_at`, `expires_at` | Server UTC evaluation time and validity bound; null expiry on a blocking record conveys no authorization |
| `subject` | Program ID, provider/repository ID, trusted source/target refs as relevant, exact commit SHA, artifact digest if applicable, pipeline/run/job identity, gate and target environment |
| `request_digest`, `audience` | Binding to canonical request content and intended PEP; canonicalization version recorded |
| `evaluator` | PDP instance and deployment version, CEL runtime/environment version and input-schema version |
| `policy_snapshot` | Exact signed bundle(s), policy/rule versions and resolved scope bindings used for this evaluation |
| `rules` | Rule-level results, including successful, failing, waiting, bypassed and unevaluated rules |
| `blocking_rule_ids` | All known blocking rule IDs, not only the first failure |
| `pending_actions` | All current approval tasks, responsible parties and next actions, even when outcome is not pending |
| `evidence` | Verified evidence IDs/digests, issuer, subject, timestamps, freshness and verification outcomes |
| `exceptions` | Applied and relevant rejected/expired/pending bypass references and reasons |
| `evaluation` | Completeness status, expected/evaluated counts, duration, cost-limit status and evaluation errors |
| `audit` | Audit record ID, persisted-at timestamp, immutable detail reference and integrity digest |
| `links` | Authorized decision details, remediation and pending-request links |

Fields unavailable because authentication, policy loading or evaluation failed are explicitly null/empty with structured errors and completeness status. Never invent a policy version, responsible person or successful check to fill the response. Sensitive metadata is governed by the projection rules below.

## 2. Exact policy provenance

`policy_snapshot` records each bundle ID, release sequence, archive digest, Git repository ID and source commit, signature-verification result/time, trusted signer key/identity reference, trust-configuration version, activation time and validity window. It also records policy ID/version, scope-binding version and each applicable program/repository/branch scope.

Every rule result identifies its rule ID/version, containing policy ID/version, CEL source or expression digest, and inherited origin scope. Record the resolved effective-policy digest so a historical decision can be reconstructed with its retained inputs and context. Do not resolve historical records against whichever policy is current today.

Private signing material is never response metadata. Policy archive attestation verification and mandatory decision attestation signing use separate issuer authorities and key purposes.

## 3. Rule-level explanation

Each entry contains:

- Rule ID, version, safe title, policy reference, origin scope and applicability.
- `status`: `PASS`, `FAIL`, `PENDING`, `BYPASSED`, `ERROR`, `NOT_APPLICABLE` or `NOT_EVALUATED`.
- Stable `reason_code`, safe explanation, expected requirement and observed value when authorized and safe to disclose.
- Evidence references and per-evidence verification/freshness outcomes.
- Remediation guidance and accountable remediation role/team, distinct from an approval assignee.
- Related pending action IDs and applied bypass ID, if any.
- Evaluation time/cost where available and reason for not evaluating, if relevant.

A bypass must not hide the underlying result. For `BYPASSED`, retain the original failing requirement/result where evaluated, the approved exception ID, requester, reviewer, approval time and validity. If the underlying rule could not be evaluated, say so rather than inventing a failure or success.

Evaluate all applicable independent rules within bounded resource limits rather than stopping at the first denial. Dependency failures and cost limits may prevent full evaluation; mark those rules `NOT_EVALUATED` and explain why. An incomplete evaluation must never yield `PERMIT`.

## 4. Who is the decision pending on?

Each `pending_actions` entry contains:

| Field | Meaning |
|---|---|
| `action_id`, `request_id`, `request_revision` | Exact workflow task and immutable request revision |
| `type` | For example `BYPASS_REVIEW` or separate `RELEASE_APPROVAL` |
| `status` | `PENDING`, `CLAIMED` or `UNASSIGNED`; completed actions appear in approval history instead |
| `rule_ids`, `control_ids` | Requirements this task could satisfy or waive |
| `required_role` | For bypass review, `Security Reviewer`; never inferred from ownership |
| `responsible_group` | Stable authorized directory group ID and safe display name |
| `assigned_to` | Actual individual assignee ID/display name if claimed or explicitly assigned; otherwise null |
| `requested_by`, `requested_at` | Requester identity and submission time |
| `required_approvals`, `received_approvals`, `remaining_approvals` | Explicit quorum and progress under the workflow rules |
| `approval_history` | Completed reviewer identities, outcomes, reasons and timestamps, subject to caller authorization |
| `due_at`, `request_expires_at`, `overdue` | Deadline/expiry information; null where no deadline is configured |
| `escalation_owner` | Responsible role/group if unresolved or overdue |
| `required_action`, `action_url` | What the responsible party must do and authenticated UI location |
| `blocking`, `actionable` | Whether this task currently blocks and can still be acted on |

Do not list every group member as an assigned reviewer. If the task belongs to a team queue, return the team and null individual assignee. If no eligible reviewer exists, report `UNASSIGNED` with `REVIEWER_ROUTING_MISSING`, escalation owner and the configuration problem; do not pretend someone has received the task. Denial of a bypass is a completed review and leaves the underlying failed control blocked, not pending.

An unmet rule without a submitted approval request is not automatically pending. Return its failure and remediation/request-bypass option where allowed. Evidence processing is a separate dependency, not a fictitious human approval; identify the producer/service in the rule explanation.

Identity display names are a snapshot at evaluation time. Store stable identity IDs and assignment revision. The immutable decision records who it was pending on then; a linked live workflow view may show today's assignment/status with its own timestamp. Approval or reassignment triggers a fresh evaluation when needed, not mutation of the original decision.

## 5. Outcome aggregation

Proposed deterministic ordering:

1. If one or more conclusively failing mandatory rules remain unsatisfied and are not covered by an applicable approved bypass or a valid pending approval path, return `DENY`. Preserve all other pending tasks and errors; explain that those approvals alone will not unblock this decision.
2. Otherwise, if required inputs/policy verification are unavailable, evaluation has errors, or applicable rules remain unevaluated, return `INDETERMINATE`.
3. Otherwise, if an actual required approval task remains unresolved, return `PENDING_APPROVAL`.
4. Only if every applicable mandatory requirement is satisfied or validly bypassed, evaluation is complete and audit persistence succeeds, return `PERMIT`.

A failed waivable control with a valid submitted bypass request may be represented as `PENDING` with the underlying failure preserved. It still blocks enforcement until approved and freshly evaluated. A non-waivable control, invalid request or denied request remains a failure. A pending approval cannot mask an independent denial.

Only `PERMIT` authorizes advancement. A later approval does not transform an old pending record into a permit. Issue a new decision ID linked by `supersedes_decision_id` or equivalent correlation, rechecking current policy, evidence, exception validity and subject binding.

## 6. Completeness, integrity and access control

Retain the complete authorized explanation as an immutable decision resource. For large responses, inline summary plus authenticated paginated detail is permitted; include `details_complete`, explicit totals, cursor/continuation and a detail integrity digest. Never silently truncate denying rules or waiting parties. The PEP enforces the authoritative outcome, not a client-recomputed result from a partial page.

Provide a least-privilege CI projection: safe control IDs/reasons, policy provenance and authorized pending owner/team labels. Full reviewer identities and evidence detail require scoped access; raw email addresses, secrets, tokens, sensitive findings and full CEL source need not be exposed in pipeline logs. Redaction must be explicit and offer an authorized management link, not manufacture substitute data. Email recipients receive the same authorization-filtered summary and link rather than the entire decision payload.

Bound a permit's expiry to the earliest relevant policy, evidence, exception and approval validity and configured decision TTL. Subsequent revocation can invalidate authorization before that timestamp; fresh checks at irreversible boundaries remain required. The DSSE signature covers outcome, subject, validity, policy provenance, request/audience binding and immutable-detail digest. Construct the authorized projection before signing; never redact or modify an already signed payload. PDP persists the exact signed envelope before returning it, and PEP receipts reference its digest.

## 7. Acceptance criteria

- Every decision identifies the exact archive digest, release sequence, source commit and policy/rule versions actually used, or explicitly explains why no eligible policy was available.
- Multiple denying rules are returned with distinct reasons and remediation; evaluation never reports incomplete work as permit.
- Mixed denial and pending approval returns a blocking outcome and exposes all pending tasks without implying their completion alone is sufficient.
- Pending bypass review identifies Security Reviewer group, actual assignee if any, quorum, required action and authorized link.
- Unassigned, overdue, denied and completed reviews are distinguishable; no identities are guessed.
- Bypassed rules retain the underlying result and approved exception provenance.
- A fresh decision follows approval, reassignment or policy/evidence changes; prior decision remains immutable.
- Detail pagination/redaction is explicit, and unauthorized CI callers cannot retrieve reviewer PII or sensitive evidence.
- Audit and emails correlate to decision IDs; later workflow changes do not rewrite historical pending ownership.
