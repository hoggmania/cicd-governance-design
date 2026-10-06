# Git-approved CEL policy distribution

Status: architecture specification, not implemented. CEL, Git approval, signed archive publication to S3 or Artifactory, periodic PDP retrieval and signature verification before use are user requirements. Formats and operational limits below are proposed defaults for review.

## 1. PAP boundary and source of truth

Signing format is now fixed: in-toto Statement v1 in DSSE envelopes, signed through Vault or OpenBao Transit. Require both SLSA build provenance and a custom policy-release authorization attestation for the same archive digest. See [attestation model](attestation-model.md) for predicate contracts, independent trust distribution and key lifecycle.

The PAP is a logical capability spanning the policy Git repository, protected approval workflow, release pipeline and governance management service. Git is the source of truth for CEL expressions, control definitions, applicability, waivability, hierarchy bindings and tests. The management UI displays release/activation status and manages bypass requests; it cannot directly edit or activate policy outside Git.

Security Manager authors CEL in a dedicated policy repository. The hierarchy of governed application repositories is data in this policy repository: its Git branches are authoring/release branches, not the optional governed application-branch scopes.

Git PR approval is mandatory and independent of the author. Security Reviewer as policy PR approver remains a proposed separately assigned permission, distinct from their confirmed bypass-review permission. Require protected branches, authorized path owners, approval of the current revision, invalidation of stale approvals after changes, and successful required tests. A CODEOWNERS file alone does not enforce approval. Protect release-workflow and policy-test changes as well as CEL source.

## 2. Release pipeline

1. Validate policy schema, stable control IDs, hierarchy references and allowed scope changes; parse and type-check CEL against a versioned input schema and approved function environment.
2. Execute positive/negative fixtures, regression and inheritance tests, bypass/non-waivable-control tests, and evaluation-cost limits. Semantic conflicts in arbitrary CEL are not assumed universally decidable; combine structural validation, fixtures and review.
3. Merge only after required approval and checks. The release job revalidates the exact merged commit; no untrusted PR job receives signing or publication credentials.
4. Build an archive containing CEL source, control metadata, hierarchy bindings, input-schema/engine requirements and manifest. Define normalized file order/timestamps for reproducible packaging. Pin build dependencies and permitted CEL extensions; do not execute archive scripts or fetch imports at runtime.
5. Produce a signed release envelope binding the SHA-256 digest of the exact archive bytes to the release metadata. Use in-toto/DSSE libraries with a tested Vault/OpenBao Transit signing adapter; only algorithm/library selection remains open, not the envelope format.
6. Upload the archive and envelope to immutable, versioned or content-addressed locations in S3 or Artifactory. Read back and verify the uploaded objects before advertising the release. Update the discovery pointer last using serialized or conditional publication so concurrent publishers cannot silently move it backwards.
7. Persist publication events and notification records. Publication is complete only after release evidence and durable audit acknowledgement exist; partial failures remain explicitly pending and are reconciled.

A Git commit signature, TLS connection, storage ETag or plain checksum does not substitute for the policy archive signature.

## 3. Proposed release structure

- `policy.tar.gz`: CEL source, control/scope metadata, schema and internal file manifest.
- `provenance.json`: DSSE-signed in-toto SLSA provenance statement whose subject is the exact archive digest and whose predicate identifies builder, source commit and build inputs.
- `release-envelope.json`: DSSE-signed in-toto policy-release authorization statement binding archive subject digest, byte size, media type, bundle ID, sequence, source repository/commit, channel/audience, scope-set ID, creation/validity times, CEL/schema versions and required provenance-envelope digest.
- Channel discovery pointer: references the immutable envelope location; a discovery hint only, never a trust anchor or direct instruction to activate a version.

All authorization-relevant metadata must be covered by the signature, not only the filename. Reject disagreement between the verified envelope and archive manifest. Retain approval evidence, CI run identity, test results and archive digest in release provenance, with integrity-bound references. The trusted publisher must enforce Git approval; a cryptographic signature by itself cannot prove the publisher performed that workflow.

Keep policy source and supported environment/schema versions in the archive. The PDP type-checks/compiles the source locally with the pinned CEL runtime; do not assume a serialized compiled expression is portable or safe across engine versions.

## 4. Signing and trust

Policy release signing occurs only in the approved post-merge job, using dedicated workload identities and separate Vault/OpenBao Transit asymmetric keys for provenance and release authorization. Producer adapters sign DSSE PAE bytes, not bare JSON or only the archive hash. Private keys are non-exportable and never enter Git, archives or PDP configuration. Storage publication and signing permissions are separate and narrowly scoped. PDP has read-only policy artifact access and public verification material for policy issuers; its separate decision-signing workload permission cannot sign policy releases.

Provision trusted verification keys or authorized signer identities to PDP through an independent platform trust configuration. Never trust a key merely because it was packaged beside the archive. A signer key ID selects an already trusted key; it does not establish trust. Restrict accepted algorithms, signer scope/channel and key validity. A policy archive cannot redefine its own trust roots or freshness limits.

Define audited key rotation with overlap, trust-store versioning and revocation. Check signer revocation when activating and before continued use under a bounded trust-status freshness policy. A compromised/revoked signer invalidates affected bundles according to the revocation policy; do not retain them merely because they were valid previously. Trust-root administration is separate from policy authorship and bypass review.

## 5. Periodic PDP pickup and activation

Each PDP instance polls its configured S3/Artifactory channel on a configurable interval with jitter, timeouts and bounded retries. Polling interval and maximum acceptable policy age are distinct settings. Emit per-instance desired/active sequence, digest, last successful refresh and rejection reason; management must show published versus active versions separately.

For each candidate:

1. Fetch bounded-size envelope and archive over authenticated TLS from configured/allowlisted locations. Do not follow arbitrary URLs supplied by untrusted metadata.
2. Verify both DSSE statements using independent public trust material, signer-to-predicate/scope authorization, expected channel and validity. Validate SLSA provenance and release authorization agree and the release references the exact provenance envelope digest. Compute the archive digest and compare it to the signed value. No CEL parse, archive extraction or evaluation occurs before successful signature and archive-integrity verification.
3. Enforce rollback protection against a persisted highest accepted release sequence per channel/scope set. Reject lower sequence and equal-sequence/different-digest releases. Equal sequence/same digest is an idempotent refresh, not a new release.
4. Extract in a bounded staging area: reject absolute/traversal paths, duplicate/conflicting entries, links/devices, unlisted files and size/count/decompression-limit violations. A correctly signed malformed archive is still rejected.
5. Validate manifest, schema, hierarchy, CEL environment compatibility and type-check all expressions; run activation self-tests under resource limits. Missing facts, CEL errors, unknown results or non-boolean control results never become permits. Use explicit typed inputs; disable network/file/time access through extensions and supply trusted evaluation time as input.
6. Durably record verified activation intent and notification event; atomically swap the complete in-memory policy snapshot and persist anti-rollback state through a recoverable journal. Confirm activation in audit. Crash recovery reconciles intent versus active state before serving; never expose partially updated rules or advance durable state in a way that prevents completing recovery.
7. Each decision captures one immutable active snapshot and records bundle digest, release sequence, source commit and applicable control versions. In-flight evaluations finish on their captured snapshot; new evaluations use the new one. Fresh checks at irreversible boundaries remain mandatory.

A new instance must bootstrap from an independently trusted minimum release/checkpoint or control-plane release floor before serving. A valid but superseded signed bundle must not be accepted simply because the node lacks local history. A writable object-store pointer alone cannot supply that trust.

Replicas can activate at different moments during polling; this is not globally atomic rollout. Use readiness based on a required release floor and bounded convergence, drain stale replicas, and block protected gates if required release freshness is not met. Review rollout/rollback coordination and service targets explicitly.

## 6. Rejection, outage and rollback behavior

Invalid/missing signature, untrusted or revoked key, digest mismatch, expired release, replay, incompatible CEL/schema, unsafe archive or failed validation: reject the candidate, audit the reason and notify security/operations. Never overwrite the active snapshot with the candidate.

Continue a previously verified last-known-good bundle only while its signed validity, local maximum staleness, release floor and trust/revocation freshness requirements all remain satisfied. If no eligible bundle exists, PDP is not ready and protected gates fail closed. Failed refresh never resets freshness. Re-fetching the same release cannot extend its signed expiry or maximum bundle age. Storage unavailability alone need not stop gates that have an eligible local bundle.

Rollback is a new Git-approved, newly signed release with a higher sequence containing the intended previous policy content. Do not disable anti-rollback checks or simply point at an old signed archive. Emergency issuance follows a pre-approved audited procedure; an Application Manager bypass cannot waive signature verification, trust validation or the enforcement path.

## 7. Dynamic bypasses remain separate

Recommended boundary: signed archives carry baseline policy and which controls are waivable; Application Manager bypass requests and Security Reviewer decisions remain in the governance service. PDP retrieves authenticated, current approved exceptions for each decision (or within an explicitly bounded freshness policy), binding them to signed control versions. This prevents a slow policy poll from delaying urgent bypass revocation. Bypass data cannot supply CEL, redefine non-waivable controls, or override artifact trust checks. If the exception service is unavailable, protected gates block under the existing availability policy.

If bypasses are later distributed in signed archives too, their independent publication, polling, expiry and revocation-latency contract must be explicitly redesigned; do not silently assume immediate revocation from periodic polling.

## 8. Audit and notifications across systems

Correlate Git push/PR/review/merge events, build/test/sign/upload/publication events, poll/verify/reject/activate events and decisions by provider event ID, source commit and release digest. Deliver the existing interested-party emails for each auditable action, including publication failures and PDP rejection.

A transaction cannot span Git, CI, S3/Artifactory and the governance database. Use durable provider audit events, authenticated webhook/event ingestion, idempotent inbox/outbox records, publisher checkpoints and periodic reconciliation for missed events. Do not claim exactly-once cross-system delivery. Require audit acknowledgement before release advertisement or PDP service activation; fail closed on unacknowledged activation. Raw edits in an author's local editor are not server-observable actions; audit begins when work reaches the managed Git service.

## 9. Decisions for review

Select CEL runtime/library, DSSE/Transit implementation and algorithm, Vault or OpenBao deployment, custom predicate namespace/schema, S3 or Artifactory backend, archive format, poll interval, maximum bundle age, release validity, trusted clock/skew allowance, rollout convergence target, independent bootstrap/release-floor mechanism, key rotation/revocation service targets and retention. The CEL language and signed pull-based release model are already fixed requirements.
