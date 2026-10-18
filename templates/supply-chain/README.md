# Supply Chain Security & Admission Control Templates

Production-grade templates for establishing end-to-end cryptographic supply chain integrity, automated container image signing via Sigstore Cosign keyless OpenID Connect (OIDC), SLSA v1.0 build provenance attestation, and Kubernetes admission enforcement via Kyverno.

## Template Catalog

| Template | Kind | Description |
|:---|:---|:---|
| [`kyverno-cosign-verify.yaml`](./kyverno-cosign-verify.yaml) | `ClusterPolicy` | Kyverno policy enforcing keyless Cosign signature and SLSA provenance verification across protected namespaces. |
| [`cosign-slsa-actions.yml`](./cosign-slsa-actions.yml) | GitHub Actions | Reusable workflow building container images, signing with Cosign via keyless OIDC, and attesting SLSA provenance. |

## Quickstart & Configuration

### 1. Template Placeholders

Before deploying, substitute the placeholder tokens:

| Placeholder | Example Value | Description |
|:---|:---|:---|
| `{{GITHUB_ORG}}` | `my-org` | GitHub organization or user owning the workflow runner. |
| `{{IMAGE_REGISTRY}}` | `ghcr.io` | Fully qualified OCI registry domain. |
| `{{PROTECTED_NAMESPACE}}` | `production` | Target Kubernetes namespace where strict admission applies. |

### 2. Calling the Reusable Workflow

Integrate `cosign-slsa-actions.yml` in your application repository `.github/workflows/release.yml`:

```yaml
jobs:
  release-image:
    permissions:
      id-token: write
      contents: read
      packages: write
    uses: my-org/engineering-playbook/templates/supply-chain/cosign-slsa-actions.yml@main
    with:
      image-name: ghcr.io/my-org/payment-service
      image-tag: ${{ github.sha }}
      dockerfile: ./Dockerfile
```
