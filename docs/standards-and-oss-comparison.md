# CI/CD governance: standards and open-source comparison

## Bottom line

Evaluate AMPEL before building a bespoke CEL/attestation evaluator. Build or integrate the governance control plane around it: hierarchy, scoped human duties, bypass lifecycle, pending ownership, durable audit/email and protected CI adapters. OPA is a strong reference for bundle management but uses Rego, not the agreed CEL language. Conforma is a useful supply-chain verifier and comparison baseline, also using Rego. Existing pipeline approval controls are PEP/workflow integrations, not demonstrated replacements for the whole proposed service.

This is a documentation/API-based shortlist, not a benchmark or production-readiness certification. Our local project is documentation only with no initial commit or executable test suite. No external products were built, installed or runtime-tested. Repository revisions, commit timestamps and available license API metadata are retained in [source snapshot](comparison-source-snapshot.json). Missing feature evidence means not established by reviewed sources, not proof a feature cannot exist.

## Standards and specifications

| Standard/specification | Contribution | Boundary and recommendation |
|---|---|---|
| [XACML 3.0](https://docs.oasis-open.org/xacml/3.0/xacml-3.0-core-spec-os-en.html) | PAP/PDP/PEP separation, authorization requests/results and combining semantics | Borrow architectural vocabulary; CEL/JSON decisions are not XACML conformance. Pending human approval is our workflow state, not a native XACML decision value |
| [In-toto Statement / DSSE](https://github.com/in-toto/attestation/blob/main/spec/v1/envelope.md) | Signed, typed claims bound to immutable subjects | Adopt for policies, evidence, reviews, decisions and receipts; does not supply business authority, expiry or revocation automatically |
| [SLSA provenance](https://slsa.dev/spec/v1.2/build-provenance) | Standard build provenance for archives and application artifacts | Reuse; does not alone prove required Git approval or authorize deployment; using the format does not establish a SLSA level |
| [SVR](https://github.com/in-toto/attestation/blob/main/spec/predicates/svr.md) / [VSA](https://slsa.dev/spec/v1.2/verification_summary) | Portable verified-properties and SLSA verification summaries | Optional interoperable summary outputs; keep detailed denial/pending contract rather than stretching their core semantics |
| [CDEvents](https://cdevents.dev/docs/) | Continuous delivery event vocabulary built on CloudEvents | Evaluate as event transport vocabulary for audit/integration. Not signature verification, durable delivery or an approval engine; custom approval events need explicit versioned schemas |
| [OSCAL](https://pages.nist.gov/OSCAL/) | Machine-readable security controls, baselines and assessment information | Add control-ID mappings/export when audit consumers need it; not a CEL runtime or gate approval engine |

These standards describe complementary layers. None of the reviewed specifications is a complete interoperable protocol for our Program/Repository/Branch hierarchy, bypass review, pending ownership and signed execution permit lifecycle.

## Open-source candidates

| Candidate | Documented capability | Fit / gap relative to our design |
|---|---|---|
| [AMPEL](https://github.com/policylabs/ampel) | CEL tenets, policy sets, in-toto attestation verification, pluggable signing schemes, rich ResultSet, VSA/SVR outputs and OSCAL control links | Closest evaluation-core fit. Verify raw DSSE + Vault/OpenBao adapter, complete rule-result semantics, embedding/API stability, resource bounds and no-runtime-fetch behavior. Full human bypass workflow, hierarchy and notification service are not established by reviewed docs |
| [Conforma](https://conforma.dev/docs/cli/ec_validate_image.html) | Image and attestation signature verification followed by Rego policy evaluation; attempts to gather all issues; structured outputs | Strong artifact-verification reference or downstream integration. Rego conflicts with fixed CEL requirement; not evidenced as the full business approval control plane. `--strict=false` deliberately returns success on violations and must not appear in enforcing adapters |
| [OPA](https://www.openpolicyagent.org/docs/management-bundles) | Periodic policy/data bundle downloads, signed bundle verification, activation, persistence; decision logging with bundle revisions | Good operational reference. Native Rego and bundle JWT signatures are not CEL and in-toto/DSSE. Not a drop-in replacement for the chosen contract; human workflow remains external |
| [Tekton Chains](https://tekton.dev/docs/chains/) | Observes completed TaskRuns/PipelineRuns, generates and signs attestations and stores results | Adopt as an evidence producer if Tekton is selected. It is not the PAP, synchronous gate PDP or approval UI. Exact Vault/OpenBao interoperability must be verified for the selected backend/version |
| [Spinnaker](https://spinnaker.io/docs/guides/tutorials/codelabs/safe-deployments/) | Deployment pipeline orchestration, manual judgment stages, awaiting-judgment email notifications and rollback flow | Useful existing approval surface/PEP where already deployed. Our control-specific expiring bypasses and signed central decisions require integration; do not deploy a full CD platform solely for its approval button |
| [Jenkins Input Step](https://www.jenkins.io/doc/pipeline/steps/pipeline-input-step/) | Pauses pipeline for proceed/abort; restricts submitter users/groups and can record approving identity | Useful PEP adapter target, not a governance system. Documented caveat: administrators may respond regardless of submitter restriction. Pipeline/job/admin controls remain part of the trust boundary |
| [Minder](https://github.com/mindersec/minder) | Repository enrollment, policy-driven repository/artifact security posture, alerting/remediation and signature checks; self-hostable Helm deployment | Complement for checking/protecting repository posture. Not evidenced as the required CEL gate-decision/bypass service. Its CLI docs describe a hosted default: configure self-hosted endpoint explicitly; do not run default quickstart against private repos |

AMPEL, Conforma CLI, OPA, Tekton Chains, Spinnaker and Minder returned Apache-2.0 license metadata from GitHub's license API, with source file links in the snapshot. This is not a transitive-dependency license audit. The Jenkins plugin metadata request returned 404 on the attempted API path; Jenkins functional comparison is based on official documentation, not verified plugin source/license metadata.

## Capability ownership

- Reuse evaluation/attestation processing: first investigate AMPEL; avoid parallel custom CEL engine work until this spike is resolved.
- Reuse evidence generation: selected CI's provenance tooling, Tekton Chains when applicable, existing scanners/SBOM generators. Dependency-Track remains the inventory/lifecycle system where used; attestations do not replace it.
- Reuse pipeline runtime and human surfaces: existing Jenkins/Spinnaker/other selected CI. Their approval must not directly bypass the central PDP; route business bypass review to the authoritative governance workflow.
- Retain bespoke domain governance: role separation, inherited program/repository/branch controls, reviewer assignment, expiry/revocation, safe decision projection and email recipient rules.
- Retain explicit release trust: Git review, immutable archives, separate policy provenance/release authorization, issuer-specific Vault/OpenBao Transit permissions, independent public trust and anti-rollback.
- Reuse event vocabulary where useful: CDEvents/CloudEvents; persist and reconcile events ourselves or through a selected durable workflow platform. Event format alone gives no delivery guarantee.

## AMPEL adoption spike: go/no-go evidence

1. Pin a release/commit and inspect actual license, API surface, dependency tree, release artifacts and security model.
2. Evaluate the same CEL fixture set in AMPEL and a minimal direct CEL baseline: pass, deny, multiple failures, missing evidence, evaluation errors and cost exhaustion.
3. Verify a real Transit-signed in-toto/DSSE evidence statement with independent public trust. Reject wrong issuer/predicate/scope, subject mismatch, invalid signatures and stale evidence. Do not assume advertised Sigstore support equals compatible raw DSSE/Transit support.
4. Wrap results into our gate-decision predicate without losing individual rule versions, underlying bypassed failures, completeness or pending reviewer/team metadata. Pending workflow is a control-plane concern, not presumed native engine behavior.
5. Prove immutable verified policy loading: no unapproved remote policy/import fetch during evaluation; reject parent-control weakening and OR/warn-only configurations that would defeat the mandatory baseline.
6. Integrate one signed permit and one denial into a protected non-production CI gate; verify replay rejection, signing outage behavior and signed receipt persistence.
7. Compare adapter complexity and operational ownership against direct CEL-library integration. Adopt only if reuse reduces maintained security-sensitive code without weakening our contract.

No pass/fail result is claimed for this spike. Recommended plan change: insert it before choosing the PDP implementation in W1/W2; do not yet change the architecture to require AMPEL.

## Limitations and maturity

Upstream documentation and repository activity provide evidence of feature intent and maintenance, not verified default safety or production readiness. This review does not measure throughput, availability, CVE exposure, test coverage, current CI success or operator experience. Those require pinned builds, source review and deployment tests. Our proposed system cannot be called more capable in practice than working OSS merely because its design lists additional requirements.
