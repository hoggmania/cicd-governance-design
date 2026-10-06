# CI/CD Gate Governance

Proposed system for governing formal CI/CD gates through a centrally managed policy hierarchy.

**Status: architecture proposal for review — not an implemented service.**

- **PAP — Policy Authoring Point:** Git-based CEL policy authoring and approval, signed release pipeline, and management service for ownership and bypasses.
- **PDP — Policy Decision Point:** periodically retrieves signed policy archives from S3 or Artifactory, verifies before activation, and evaluates gate requests using CEL and verified evidence.
- **PEP — Policy Enforcement Point:** CI/CD pipelines enforce the decision at merge, promotion and deployment boundaries.

Ownership hierarchy: **Program (Application Manager) → Source repository (Project Lead) → Optional branch scope**.

Policy authority is separate from ownership: **Security Manager authors policies → Application Manager requests bypass → Security Reviewer approves or denies**. Every action is audited and produces email notifications to authorized interested parties. Bypasses may be time-bounded or explicitly approved without expiry; a missing date never silently grants an indefinite bypass.

## Review pack

**Attestation architecture:** in-toto Statement v1 with DSSE signatures covers policy provenance and release authorization, application evidence, bypass reviews, PDP decisions and PEP receipts. **Vault or OpenBao Transit** manages separate signing keys; verifiers use independently distributed public trust material. CEL remains the policy language, and signed claims still require authorization, freshness and revocation checks.

- [High-level architecture](docs/architecture.md)
- [Delivery and review plan](docs/review-plan.md)
- [Roles, bypass workflow, audit and notifications](docs/governance-workflow.md)
- [Git-approved CEL policy distribution and signature verification](docs/policy-distribution.md)
- [Explainable decision contract and pending ownership](docs/decision-contract.md)
- [In-toto attestations and Vault / OpenBao trust model](docs/attestation-model.md)
- [Standards and open-source comparison](docs/standards-and-oss-comparison.md)
- [Attested gate sequence](docs/diagrams/rendered/attested-gate-flow.svg) ([source](docs/diagrams/attested-gate-flow.puml))
- [Architecture diagram](docs/diagrams/rendered/cicd-governance.svg)
- [PlantUML source](docs/diagrams/cicd-governance.puml)

The recommended baseline is self-hosted, deny-by-default, with immutable published policy versions and independently approved exceptions. Repository and branch controls may strengthen inherited controls, not silently weaken them.

No application code, CI integration, hosting configuration or deployment is included yet. Technology choices remain proposals until architecture review.
