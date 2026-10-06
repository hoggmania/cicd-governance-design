# In-toto attestations and Vault / OpenBao trust model

Status: adopted architecture, not implemented. In-toto Statement v1 in DSSE envelopes is the common signed-claim format. Vault or OpenBao Transit manages signing keys. No public transparency log or SaaS service is required.

## Attestation contracts

Use `_type: https://in-toto.io/Statement/v1`, DSSE `payloadType: application/vnd.in-toto+json`, and cryptographic subject digests. Predicate URI, schema version and signer authority are verified independently of storage location. Standard predicates retain their standard semantics; project-specific predicates below are design contracts, not existing in-toto standards. Before implementation, assign their versioned URIs under a controlled namespace and freeze their schemas.

| Attestation | Producer and authority | Subject | Predicate and consumer |
|---|---|---|---|
| Policy build provenance | Trusted policy release builder | Exact CEL archive digest | Standard `https://slsa.dev/provenance/v1`; loader checks source commit, builder and workflow |
| Policy release authorization | Authorized policy publisher | Same CEL archive digest | Custom `policy-release/v1`; loader checks Git approval evidence, sequence, scope, channel, validity and linked provenance digest |
| Application evidence | Authorized builder/test/scanner issuer | Exact artifact or source snapshot digest | Existing SLSA provenance, test-result, vulnerability, CycloneDX/SPDX predicates as appropriate; evidence verifier normalizes verified claims for CEL |
| Bypass review | Governance service recording a Security Reviewer action | Immutable bypass request revision document digest | Custom `bypass-review/v1`; approved/denied outcome, reviewer, requester, controls/versions, scope, validity and conditions; PDP also checks live status |
| Gate decision | Authorized PDP workload | Evaluated artifact digest, or source snapshot digest for pre-build gate | Custom `gate-decision/v1`; full decision contract or signed summary plus immutable detail digest; PEP verifies before enforcement |
| Enforcement receipt | Protected PEP workload | Same artifact/source subject as decision | Custom `enforcement-receipt/v1`; decision-envelope digest, request/run, attempted action, observed result and time; audit consumer verifies |

Names ending `/v1` in the custom rows are logical identifiers pending namespace registration, not public URIs to fetch. Predicate URIs identify schemas; never dynamically fetch or execute arbitrary schema URLs received from attestations.

For source-only gates, define and version a deterministic source-snapshot digest method and bind it to provider repository ID and exact commit SHA. A branch name is not a digest. A scope-wide bypass attests the immutable request revision; its predicate specifies scope and control versions. Do not fabricate an application artifact digest before an artifact exists.

SLSA provenance is build evidence, not proof of Git authorization. Both policy provenance and policy-release authorization are required and must agree on archive digest, source commit and release identity. A signed release cannot authorize itself to replace verifier trust roots or bypass the signature checks. SVR can optionally summarize passing properties and VSA can summarize SLSA verification, but neither replaces the rich deny/pending decision contract.

## Vault / OpenBao responsibilities

Use one selected self-hosted provider with Transit asymmetric keys, signing APIs, rotation and audited access. Keep private keys non-exportable and plaintext key backups disabled. Vault/OpenBao software protection is not itself a claim of HSM-backed keys; HSM requirements are a separate deployment decision.

Separate logical keys and signing workloads for policy provenance, release authorization, governance bypass reviews, PDP decisions and PEP receipts. Evidence producers use issuer-specific identities/keys; approved external issuers may be verified through explicit trust mappings without importing their private keys. Do not give a generic CI job a universal signing key.

Authenticate workloads with short-lived identities using the selected provider's supported deployment integration. Authorize only the specific Transit sign path/key. Policy authors, human reviewers, storage readers and ordinary build jobs receive no direct signing-key administration. Separate key administration, trust distribution and business approval. Manage necessary integration credentials through Vault/OpenBao with scoped delivery and rotation; do not assume one lease mechanism fits every external system.

Transit performs cryptographic signing, not Git-review validation, CEL authorization or attestation construction. Producer code constructs the statement and DSSE pre-authentication encoding (PAE), then a tested signing adapter submits the correct bytes to Transit. Choose an asymmetric algorithm supported by the selected version and verifier; explicitly test prehash and signature encoding behavior. Convert the Transit response into valid DSSE signature bytes; do not put its provider-specific version-prefixed string directly into DSSE `sig`. Record a stable issuer/key/version mapping in `keyid`; keyid alone conveys no trust. Verify cross-library test vectors before rollout.

## Independent verifier trust

Distribute versioned public verification material and authorized signer-to-predicate/scope mappings through protected platform configuration. Obtain public material from the approved Vault/OpenBao keys; never bootstrap from keys bundled with an archive. Public-key distribution is not private-key export. Verify signatures locally in loaders, evidence validators, PEPs and audit consumers.

Maintain explicit revocation and trust-freshness policy outside policy archives. Rotation does not automatically invalidate historical signatures, and revoking a Vault token does not revoke signatures already produced. Update verifier trust rules for compromised keys and define historical audit interpretation separately from permission to authorize new actions. Retain relevant public key versions for audit; an old version is not automatically eligible for new releases.

Vault/OpenBao unavailability prevents new signatures. Do not issue unsigned fallbacks. Policy release publication stops; new signed PDP permits cannot be issued; approved bypass records remain in an issuance-pending state until signed. Local verification of already issued material may continue only within trust, validity, release-floor and freshness rules. Signing requests, failures, rotations and trust changes join the audit/email stream; never email private material or credentials.

## Verification and evaluation

1. Bound envelope size, decode safely and verify DSSE PAE signature with independently trusted material.
2. Validate Statement type, approved predicate type/version, signer authority for that predicate and scope, and schema.
3. Match cryptographic subject digest to the actual archive, artifact or request record. Verify linked attestation/detail digests and cross-statement consistency.
4. Apply channel/audience, sequence, replay, time, issuer revocation and current workflow-status checks.
5. For policy archives, safely extract and validate CEL only after trust/integrity verification. For evidence, expose typed normalized facts with issuer and freshness to CEL, never unchecked caller-supplied claims.
6. Persist result and notification events. Reject missing mandatory attestations, conflicting claims or verification failures; no permit on ambiguity.

An attestation proves an authorized signer made a claim, not that the claim is objectively true. Protect producer workloads and validate expected builder identities and evidence semantics. DSSE does not provide expiration, replay protection, revocation, authorization or a transparency log by itself.

## Decisions, receipts and audit

All successfully issued decisions, including deny and pending, are DSSE-signed. The transport may return blocking service errors if signing/evaluation cannot finish; an unsigned error is not an issued permit. Persist immutable decision/detail data, sign, persist the exact returned envelope and its digest, then return. Use a recoverable issuance state machine and idempotency: there is no atomic transaction across Transit and the database. A signature without a durable issued record is never returned for authorization.

Keep outcome, request digest, audience, gate/run/environment, expiry, policy and evidence digests and safe pending metadata inside the signed payload. Store sensitive details separately with a signed digest and access-controlled reference. Never redact or modify an already signed payload in transit; construct and sign the intended safe projection before issuance. Original decisions stay immutable when approval ownership changes.

Before acting, PEP verifies the signed permit, request binding and current validity, and durably records enforcement intent. After acting or blocking, it signs a receipt referencing the exact decision-envelope digest. If receipt signing/upload fails after an action, record delivery-pending evidence and retry/alert; do not claim the action was rolled back, rerun it, or grant a new permit merely to recreate a receipt. Receipts are trusted PEP claims, not independent proof of infrastructure state.

Store envelopes immutably in scoped S3/Artifactory or evidence storage, indexed by subject digest, predicate type, issuer and decision/request ID. Audit records retain envelope digests and issuer/key versions. Emails link to authorized records, not unrestricted downloadable reviewer PII. Reconcile missing/duplicate producer and storage events without promising exactly-once cross-system delivery.

## Sources and implementation review

- [In-toto Statement v1](https://github.com/in-toto/attestation/blob/main/spec/v1/statement.md)
- [In-toto DSSE envelope](https://github.com/in-toto/attestation/blob/main/spec/v1/envelope.md)
- [Predicate catalog](https://github.com/in-toto/attestation/blob/main/spec/predicates/README.md)
- [SLSA build provenance](https://slsa.dev/spec/v1.2/build-provenance)
- [Vault Transit API](https://developer.hashicorp.com/vault/api-docs/secret/transit)
- [OpenBao Transit API](https://openbao.org/docs/api/secret/transit/)

Remaining selections: Vault versus OpenBao deployment/version, asymmetric algorithm, DSSE/Transit adapter, predicate namespace/schemas, trust-distribution/revocation channel, source snapshot digest method, poll/TTL limits and storage backend. No keys, credentials, live policies or permissions are changed by this design update.
