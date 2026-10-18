# Software Supply Chain Security Standards

Standards for container provenance, cryptographic attestation, image signing, and admission control.

## What is Supply Chain Security?

Software supply chain security encompasses the practices, cryptographic tooling, and verification gates that protect every stage of the software delivery lifecycle — from source code commits and third-party dependencies through container builds, packaging, registry distribution, and cluster runtime admission. It ensures that only verified, untampered artifacts built by authorized continuous integration pipelines from vetted source repositories are permitted to run in production environments.

## Key Business Drivers

| Driver | Outcome |
|--------|---------|
| **Zero-Trust Artifact Integrity** | Guarantees container images running in production originate exclusively from trusted source repositories and CI workflows |
| **Non-Repudiation & Traceability** | Binds every artifact irrevocably to a verifiable developer identity, commit SHA, and build invocation via keyless signatures |
| **Proactive Vulnerability Gate** | Automated pre-signing scanning blocks compromised packages, backdoored base images, and high/critical CVEs |
| **Policy-Driven Fail-Closed Admission** | In-cluster admission controllers reject unsigned images, untrusted issuers, or tampering attempts before pod execution |
| **Auditability & Compliance** | Cryptographic transparency logs (Rekor) and SLSA Level 3 provenance provide verifiable audit trails (SOC2, FedRAMP, NIST SP 800-218) |

## Who it's for

- **Security Engineers & SecOps:** Architects defining cryptographic attestation requirements, keyless Sigstore trust policies, and vulnerability threshold gates.
- **Platform & Release Engineers:** SREs configuring GitHub Actions CI/CD pipelines, Cosign signing workflows, SLSA provenance generators, and in-cluster admission controllers (Kyverno, Gatekeeper).
- **Software Engineers:** Developers authoring application code, curating minimal distroless container builds, and remediating high/critical CVE dependencies before promotion.
- **Compliance & Audit Teams:** Assessors verifying verifiable provenance proofs, image digests, and immutable transparency log entries across releases.

---

## Core Principles

### 1. Cryptographic Attestation

Every container image must possess immutable, cryptographic proof of its source code origin, build pipeline, dependencies, and vulnerability posture. Metadata cannot rely on mutable tags; all attestations bind directly to immutable cryptographic image digests (`sha256:...`).

### 2. Keyless Ephemeral Signing

Static, long-lived private signing keys are **strictly forbidden**. Storing private keys in CI/CD secrets exposes the supply chain to credential theft and rotation overhead. Signing must leverage **Sigstore keyless infrastructure**, using OpenID Connect (OIDC) federated identity tokens to issue short-lived X.509 certificates.

### 3. Fail-Closed Admission Gate Enforcement

Security guarantees cannot depend entirely on CI/CD pipelines. The runtime Kubernetes control plane must actively enforce signature and attestation verification at the admission controller. If an image lacks a valid cryptographic signature or SLSA provenance attestation, the admission webhook must block container creation (`fail-closed`).

---

## Supply Chain Pipeline Architecture

Artifacts transition through sequential verification and attestation phases before reaching the runtime cluster:

```mermaid
flowchart TD
    A["Source Commit (git SHA)"] --> B["Hermetic Container Build (Distroless)"]
    B --> C["Trivy Vulnerability Scan"]
    C -- "CVE >= HIGH Detected" --> D["Pipeline Aborted (Exit 1)"]
    C -- "Zero Blocking CVEs" --> E["Keyless Cosign Signing (Fulcio + Rekor)"]
    E --> F["SLSA Level 3 Provenance Attestation"]
    F --> G["Push Image & Attestation to OCI Registry"]
    G --> H["Cluster Admission Webhook (Kyverno / Gatekeeper)"]
    H -- "Invalid Signature / Missing SLSA" --> I["Admission Rejected"]
    H -- "Signature & Provenance Validated" --> J["Pod Scheduled & Executed"]
```

---

## Vulnerability Scanning Gate (Trivy)

All container images must be scanned for operating system and application dependency vulnerabilities prior to cryptographic signing. Signing an image with known, unpatched high-risk vulnerabilities is prohibited.

### Policy Requirements

- **Severity Threshold**: Pipelines must fail and abort on any CVE with severity `HIGH,CRITICAL`.
- **Fail-Closed Execution**: Scanning enforces `exit-code: 1` on vulnerability detection, blocking downstream workflow execution.
- **Pre-Signing Gate**: Scanning occurs strictly before `cosign sign` against the immutable image digest (`${IMAGE_REF}@${IMAGE_DIGEST}`).
- **Minimal Attack Surface**: Images must be built from minimal, distroless base images (e.g., GoogleContainerTools distroless, Wolfi) running as non-root (UID 65534).
- **Scan Scope**: Scans OS packages and application dependencies (`vuln-type: os,library`) with zero unfixed high-risk exemptions.

---

## Keyless Cosign Signing with Sigstore

Signing is executed using Sigstore Cosign via GitHub Actions Workload Identity (OIDC).

### Signature Components

1. **OIDC Provider**: GitHub Actions issues a short-lived OIDC JWT token containing identity claims (repository, workflow, Git ref, commit SHA).
2. **Fulcio Certificate Authority**: Exchanges the OIDC token for an ephemeral X.509 signing certificate valid for 10 minutes.
3. **Cosign Client**: Generates an ephemeral key pair in memory, signs the OCI image digest, and discards the private key.
4. **Rekor Transparency Log**: Stores an immutable, publicly verifiable record of the signature and certificate transparency proof.

### Keyless Signing & Verification Specification

- **OIDC Claims**: Requires minimal permissions `id-token: write` and `contents: read` to authenticate against Fulcio.
- **Sigstore Fulcio**: Issues short-lived (10-minute) X.509 certificates matching GitHub workflow identity (`https://token.actions.githubusercontent.com`).
- **Sigstore Rekor**: Logs signing event and cryptographic inclusion proof to public transparency log (`https://rekor.sigstore.dev`).
- **Signing Target**: Signed via `cosign sign --yes "${IMAGE_REF}@${IMAGE_DIGEST}"`.
- **Verification Gate**:

  ```bash
  cosign verify \
    --certificate-identity-regexp "^https://github.com/mahpatil/.*$" \
    --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
    "${IMAGE_REF}@${IMAGE_DIGEST}"
  ```

---

## SLSA Level 3 Build Provenance

Supply chain integrity requires non-falsifiable build provenance meeting **SLSA (Supply-chain Levels for Software Artifacts) Level 3**.

### Required Provenance Properties

- **Isolated Build Environment**: Builds execute on isolated ephemeral runners where developers cannot inject arbitrary external binaries during artifact construction.
- **Cryptographic Provenance**: Provenance payload generated in `in-toto` specification format recording:
  - Builder ID (canonical GitHub Actions reusable workflow URL).
  - Source repository URL and resolved commit SHA.
  - Complete invocation parameters and dependencies.
- **Non-Falsifiable Identity**: The build system itself generates and signs the provenance attestation, preventing pipeline scripts from modifying build claims.

### Normative Requirements & Attestation Rules

- **Trusted Generator**: Provenance generated via canonical SLSA Level 3 reusable workflow (`slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2.1.0`).
- **Runner Permissions**: Isolated job permissions restricted to `id-token: write`, `contents: read`, and `actions: read`.
- **Payload Format**: Authenticated `in-toto` provenance attestation with predicate type `slsaprovenance`.
- **Verification Gate**:

  ```bash
  cosign verify-attestation \
    --type slsaprovenance \
    --certificate-identity-regexp "^https://github.com/slsa-framework/slsa-github-generator/.*$" \
    --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
    "${IMAGE_REF}@${IMAGE_DIGEST}"
  ```

---

## Cluster Admission Enforcement

Policy engines deployed within Kubernetes clusters intercept pod creation requests and evaluate admission rules.

### Kyverno Policy Enforcement

Kyverno validates image signatures and attestation identities against Fulcio and Rekor before pod admittance:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enforce-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-sigstore-signature
      verifyImages:
        - imageReferences: ["ghcr.io/mahpatil/*"]
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subjectRegExp: "^https://github.com/mahpatil/.*$"
                    rekor: { url: "https://rekor.sigstore.dev" }
```

### OPA / Gatekeeper Enforcement Pattern

Organizations standardizing on Open Policy Agent (OPA) Gatekeeper implement equivalent constraints through `ConstraintTemplate` manifests executing Rego verification against external Cosign validation sidecars or OCI metadata mirrors.

### Rollout Lifecycle

1. **Audit Mode (`validationFailureAction: Audit`)**: Deployed initially for 14 days across staging environments to detect unsigned auxiliary or sidecar containers without blocking deployments.
2. **Enforce Mode (`validationFailureAction: Enforce`)**: Locked across production clusters. Any pod failing signature or attestation verification is rejected immediately with an HTTP 403 API response.

---

## Operational Verification Checklist

Before publishing or deploying container images, confirm that all security controls pass:

- [ ] **Distroless Base**: Base image contains zero system package managers (apt, apk, yum) and runs as non-root UID 65534.
- [ ] **Vulnerability Gate Clean**: Trivy scan reports zero unfixed `HIGH` or `CRITICAL` vulnerabilities with exit code 0.
- [ ] **Digest Immutability**: Deployment manifests and Helm values reference immutable digest pins (`@sha256:...`) instead of mutable tags.
- [ ] **Keyless Signing**: Image signature successfully recorded on public Rekor transparency log.
- [ ] **SLSA Provenance Bound**: SLSA Level 3 provenance attestation generated by trusted builder workflow and attached to OCI registry.
- [ ] **Admission Enforced**: In-cluster admission controller (Kyverno or Gatekeeper) set to `Enforce` mode.
