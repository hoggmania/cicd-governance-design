# Roles, bypass workflow, audit and notifications

Status: design requirement, not implemented. This supersedes the initial proposal that Application Managers or Project Leads could author security policies.

## 1. Separation of duties

| Capability | Security Manager | Application Manager | Security Reviewer | Project Lead |
|---|---|---|---|---|
| Author CEL and submit policy PRs in Git | Yes, assigned scopes | No | No | No |
| Read effective policy and authorized audit history | Yes | Yes | Yes | Yes |
| Request a bypass | No | Yes, owned programs and descendants | No | No; supplies supporting evidence |
| Amend or withdraw own pending bypass request | No | Yes | No | No |
| Approve or deny bypass request | No | No | Yes, assigned review scopes | No |
| Revoke an approved bypass | No; can raise concern | Can request revocation | Yes (proposed operational permission) | No; can raise concern |
| Approve policy Git PR (release CI signs/publishes after merge) | No self-approval | No | Proposed separate permission, to confirm | No |

Security roles are scoped independently of the ownership tree. The same identity cannot hold conflicting Security Manager, Application Manager and Security Reviewer duties for overlapping scopes; validate group-derived permissions too. Reviewers cannot act on requests they previously submitted, even after changing roles. All role assignment changes are audited and notified; identity administrators do not gain business approval rights automatically.

Program ownership remains with Application Manager; repository ownership remains with Project Lead; branch scope is optional. Security Manager authors applicable policies across that hierarchy. No business ownership role can edit policy, mark a control waivable, or select a weaker parent policy. Security Reviewer authority must cover the affected inherited controls, not merely the lowest requested scope.

## 2. Bypass request

Baseline CEL policies are authored and approved in Git, then published as signed archives to S3 or Artifactory for periodic PDP verification and activation. The management UI cannot directly author or activate baseline policy. See [policy distribution](policy-distribution.md). Bypass requests remain in the management service and are not delayed by baseline archive polling.

Application Manager submits through the PAP with:

- Program and optional repository/branch restriction; explicit target gates and environments.
- Exact control IDs and policy versions being bypassed; affected commit/artifact/run where the bypass is release-specific.
- Business justification, risk statement, compensating measures and supporting evidence/ticket links.
- Named accountable requester, authorized interested-party subscriptions and requested effective start.
- Explicit validity mode: `TIME_BOUNDED` with an end time, or `NO_EXPIRY` with an additional justification. The UI may leave duration optional, but submission must resolve the mode explicitly.

Use server-validated UTC timestamps and render the user's timezone in the UI. Time windows use start-inclusive/end-exclusive boundaries. Reject end-before-start, already-ended windows and time windows outside configured policy limits. No-expiry means until revoked or invalidated, not immunity from policy changes; require a periodic review date as the proposed safeguard. Non-waivable controls remain non-waivable for both modes.

Recommended default: time-bounded. Explicit no-expiry requests are supported subject to the control's bypass policy and Security Reviewer acceptance; they are not silently replaced with an arbitrary duration.

## 3. Review and lifecycle

Each Security Reviewer approve/deny action produces an in-toto `bypass-review/v1` custom predicate, DSSE-signed by the governance service using its scoped Vault/OpenBao Transit key. The subject is the immutable request-revision digest; the predicate binds reviewer identity, control versions, scope, reason and validity. It records a service-authenticated human action, not a personal cryptographic signature by the reviewer. See [attestation model](attestation-model.md).

Approval is not usable until the signed envelope is durably issued; signing failure leaves issuance pending and cannot enable the bypass. Retain the human decision in audit and retry issuance idempotently. An issued approval attestation is historical evidence: PDP must still check live expiry/revocation/invalidation before every relevant decision. Revocation does not erase or re-sign the old approval.

`DRAFT → SUBMITTED → APPROVED or DENIED`; requester may transition draft/submitted requests to `WITHDRAWN`.

Approval captures reviewer identity, decision reason, exact request revision, accepted scope, controls, validity and conditions. Denial requires a reason and creates no exception. Reviewers may propose narrower scope or shorter validity, but must return that change to the requester for a new immutable revision rather than silently changing what was submitted. No approver can expand scope or duration while approving.

An approved request creates an exception in `SCHEDULED` or `ACTIVE` state, depending on its start time. It becomes `EXPIRED`, `REVOKED` or `INVALIDATED` when applicable. Approval and activation are separate audited events, even if activation is immediate. Scheduler delay cannot extend validity: PDP always checks timestamps and current status directly.

Extensions, new controls, broader scope or a switch to no-expiry require a new request and approval; never edit an active exception in place. Renewal does not extend the original exception while pending. Denied and withdrawn requests are retained; resubmission is a new linked revision. Concurrent approve/deny/withdraw operations use revision checks so only one transition can win.

Policy version changes invalidate a bypass for changed controls unless Security Reviewer explicitly reapproves the new version. An approved bypass is not an approval of future policy changes. Revocation takes effect for subsequent gate evaluations; fresh checks are required at irreversible execution boundaries, with any already-in-progress action documented as a residual timing risk.

## 4. PDP and PEP behavior

A submitted request never allows progress. PDP applies only an approved, active, in-scope exception whose control/version, subject and validity match the request. It evaluates every other control normally. Non-waivable controls cannot be bypassed. Each applied exception ID is included in the decision and its audit trail.

The PEP still requires a valid `PERMIT`. Denied, pending, expired, revoked or out-of-scope bypasses cannot produce a permit by themselves. The bypass is a governed policy exception, not permission to skip the governance check.

## 5. Audit every action

Audit policy create/edit/validate/simulate/submit/approve/publish/retire/rollback; bypass draft/edit/submit/withdraw/approve/deny/activate/use/expire/revoke/invalidate; role and scope changes; evidence and gate decisions; enforcement receipts; reads/exports and rejected authorization attempts. Each externally observable action produces a notification event. Internal audit writes and email-worker bookkeeping do not recursively trigger more emails.

Audit fields: event ID; UTC timestamp; authenticated actor and effective role; scope; action; target ID and revision; result and reason; correlation/request ID; safe before/after references or digests; affected policies; bypass validity; decision ID; recipient-policy version; notification references. Never persist secrets or raw sensitive payloads in audit messages or emails.

Within the management service, a mutation, its audit event and its notification-outbox entry form one durable transaction. Git/CI/storage actions instead use authenticated provider events, durable ingestion and reconciliation; no atomic transaction spans those systems. Audit policy authorship from managed Git events, not unobservable local editor activity. Include signature verification, rejection and activation events in audit and email routing. The decision path also persists its decision/audit/outbox before returning permit. If persistence fails, reject the mutation or return a blocking indeterminate decision. Record failures in an independent operational channel when the primary audit store is unavailable; do not claim an unavailable store recorded them.

Restrict audit modification, retain immutable versions, and provide tamper-evident exports under the retention policy. Authorized audit readers cannot amend source records.

## 6. Email interested parties

Mandatory recipient routing is scoped and centrally configured, not a caller-controlled list of arbitrary external addresses. Resolve authenticated directory groups and access-controlled subscriptions. An actor cannot remove mandatory security reviewers or owners from notifications.

| Event | Default interested parties |
|---|---|
| Policy actions | Scoped Security Manager, Security Reviewer group, affected Application Manager, affected Project Leads and authorized watchers |
| Bypass submission/amendment/withdrawal | Requester, responsible Security Reviewer group, control-owning Security Manager, affected Project Leads and watchers |
| Bypass approval/denial/activation | Requester, deciding reviewer, responsible security group, control owner, affected Project Leads and watchers |
| Bypass use/expiry/revocation/invalidation | Requester/current Application Manager, responsible reviewers, control owner, affected Project Leads and watchers |
| Gate decision/enforcement | Scoped application/repository owners, responsible security contacts and watchers |
| Role/scope changes | Affected identities, responsible security administrators and scope owners |
| Reads/exports/denied attempts | Authorized audit/security contacts; no sensitive target information sent to unauthorized requesters |

Email contains event type, safe scope summary, actor, outcome, validity where relevant, reason summary and an authenticated PAP link. Recipient visibility is checked before delivery; removal of access suppresses sensitive delivery and is audited. If mandatory routing cannot resolve a recipient, retain the event as unresolved and alert operators rather than silently dropping it.

Delivery is asynchronous through an approved email relay. Use a transactional outbox, bounded exponential retries, stable event/recipient/template idempotency keys, dead-letter handling and delivery-status audit. Track queued, attempted, relay-accepted, bounced and terminal failure separately. Relay acceptance does not prove inbox delivery, and ambiguous SMTP retries can yield duplicates; do not promise exactly-once delivery.

Immediate email for governance changes and bypass actions is the baseline. High-volume read/gate events still produce per-action notification records; optional digest delivery requires explicit configuration and must retain each event reference. Email failure does not undo a committed review decision or disable gates. Operators receive a non-email alert when the email path itself is broken, avoiding dependence on that failing channel. Delivery recovery must not require replaying the business action.

This design specifies future notifications only. No live emails are sent by creating these documents.

## 7. Review points

- Confirm policy-publication approval and bypass-revocation permissions proposed for Security Reviewer.
- Confirm bounded-validity limits and periodic-review cadence for explicit no-expiry requests.
- Identify approved email relay, mandatory directory groups, authorized watchers, delivery service targets and retention.
- Confirm whether high-volume read/gate notifications should remain immediate or use an explicitly enabled digest.
