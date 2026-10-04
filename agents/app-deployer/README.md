# app-deployer Agent

Produces a production-ready deployment artefact set for an application service: Dockerfile, Kubernetes manifests, Helm chart, progressive delivery specifications (Argo Rollouts), pre-upgrade database migration jobs, smoke test harnesses, and GitHub Actions CI/CD pipelines.

Enforces standards from `standards/claude-md/infra/CLAUDE.md`, `standards/claude-md/CLAUDE.md`, and `standards/detailed/release-engineering/`.

---

## What It Produces

| Artefact | Description |
|---|---|
| `deploy/Dockerfile` | Multi-stage build with distroless/Alpine runtime, non-root user (UID 65534) |
| `deploy/.dockerignore` | Excludes build artefacts, secrets, and dev configs |
| `deploy/helm/Chart.yaml` | Helm chart metadata and semver versioning |
| `deploy/helm/values*.yaml` | Base values + per-environment overrides |
| `deploy/helm/templates/` | Deployment / Rollout, Service, Ingress, HPA, PDB, NetworkPolicy, ServiceAccount, ConfigMap, ExternalSecret |
| `deploy/helm/templates/rollout.yaml` | Argo Rollouts canary specification with automated step progression |
| `deploy/helm/templates/analysistemplate.yaml` | Real-time PromQL metric evaluation (5xx error rate, P99 latency) |
| `deploy/helm/templates/job-migration.yaml` | Decoupled pre-upgrade database migration Job (Flyway/Liquibase) |
| `tests/smoke/smoke-test.sh` | Automated post-deployment smoke test harness |
| `deploy/k8s/` | Raw Kubernetes manifests (Helm alternative) |
| `.github/workflows/ci.yml` | Lint -> test -> static manifest check -> build -> scan -> Cosign sign -> SLSA provenance |
| `.github/workflows/cd-{env}.yml` | Sequential promotion pipelines with smoke test gates and production approval |
| `.github/workflows/security-scan.yml` | Weekly scheduled full image and dependency vulnerability scan |

---

## Inputs

| Field | Required | Description |
|---|---|---|
| `APP_NAME` | Yes | Service name, e.g. `order-service` |
| `PROJECT` | Yes | Project name, e.g. `acme-payments` |
| `TEAM` | Yes | Owning team |
| `LANGUAGE` | Yes | `java`, `dotnet`, `node`, `python`, `go` |
| `FRAMEWORK` | Yes | `springboot`, `aspnetcore`, `express`, `fastapi`, `gin` |
| `CONTAINER_REGISTRY` | Yes | Container registry URL prefix |
| `CLUSTER_NAME` | Yes | Target Kubernetes cluster name |
| `NAMESPACE` | Yes | Kubernetes namespace |
| `ENVIRONMENTS` | Yes | `[dev, staging, prod]` or subset |
| `PORT` | Yes | Application HTTP port |
| `HEALTH_PATH` | Yes | Path prefix for `/health/live` and `/health/ready` |
| `METRICS_PATH` | Yes | Prometheus metrics scrape path |
| `DR_TIER` | Yes | `1`, `2`, or `3` — determines PDB and replica policies |
| `DEPLOYMENT_STRATEGY` | Yes | `rolling`, `canary`, or `blue-green` |
| `PROGRESSIVE_DELIVERY_TOOL` | Yes | `argo-rollouts`, `flagger`, or `none` |
| `DATABASE_MIGRATION` | No | Object: `enabled`, `engine` (flyway/liquibase), `image` |
| `SUPPLY_CHAIN_SECURITY` | No | Object: `cosign_signing` (bool), `slsa_provenance` (bool) |
| `REPLICAS` | Yes | Min replicas per environment |
| `RESOURCES` | Yes | CPU/memory requests and limits |
| `DEPENDENCIES` | No | Downstream services (used for NetworkPolicy) |

---

## Release Architecture & Delivery Patterns

### 1. Progressive Delivery (Canary)

When configured with `canary` and `argo-rollouts`:

- **Stepped traffic progression**: Canary steps route traffic incrementally: `5%` -> `25%` -> `50%` -> `100%`.
- **Automated metric analysis**: Runs Prometheus PromQL analysis queries during pause windows:
  - HTTP 5xx error rate: `sum(rate(http_requests_total{status=~"5.."}[2m])) / sum(rate(http_requests_total[2m])) < 0.001` (< 0.1%).
  - P99 Latency ceiling: `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[2m])) by (le)) <= 1.15 * baseline`.
- **Automated abort**: Breaching error or latency thresholds aborts promotion and rolls traffic back to stable pods instantly.

### 2. Isolated Database Migrations

- **Decoupled execution**: Schema migrations run in isolated ephemeral Kubernetes Jobs prior to application pod rollout. Migrations never run in app startup hooks or entrypoints.
- **Lifecycle guardrails**: Jobs use Helm pre-upgrade hooks (`helm.sh/hook: pre-install,pre-upgrade`, `helm.sh/hook-weight: "-5"`), `activeDeadlineSeconds: 300` (timeout), and `backoffLimit: 1` (fail fast on bad DDL).
- **Zero-downtime safety**: Application code must maintain backward compatibility with old and new schema versions concurrently.

### 3. Automated Smoke Testing

- Script `tests/smoke/smoke-test.sh` runs as an automated validation step in staging and production CD pipelines.
- Verifies liveness and readiness probe endpoints, executes synthetic synthetic smoke requests, and validates response latency SLAs (< 500ms) before promotion is finalized.

### 4. Cryptographic Supply Chain & Static Manifest Gates

- **Static manifest validation**: CI runs `deployment-validator --mode=static` against rendered manifests before image push.
- **Keyless image signing**: Signs container digests using Sigstore Cosign via GitHub Actions OIDC identity.
- **SLSA provenance**: Attests build provenance at SLSA Level 3 using verifiable GitHub Actions workflows.

---

## How to Invoke

### Via Claude Code (Interactive)

```bash
cat agents/app-deployer/example-input.md | claude agent run app-deployer
```

### Via Anthropic API

```python
import anthropic

with open("agents/app-deployer/AGENT.md") as f:
    system_prompt = f.read()

with open("agents/app-deployer/example-input.md") as f:
    user_input = f.read()

client = anthropic.Anthropic()
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=16000,
    system=system_prompt,
    messages=[{"role": "user", "content": user_input}]
)
print(message.content[0].text)
```

---

## Agent Composition Pipeline

```text
infra-provisioner -> app-deployer -> deployment-validator (CI static gate & runtime audit)
```

1. **infra-provisioner**: Provisions cluster, network, and registry infrastructure.
2. **app-deployer**: Emits Dockerfile, Helm/Rollout manifests, migration jobs, smoke tests, and CI/CD pipelines.
3. **deployment-validator**: Validates rendered manifests in CI (`--mode=static`) and audits running cluster state (`--mode=live`).

---

## Standards Enforced

| Standard | Source |
|---|---|
| Multi-stage Dockerfile, non-root user (UID 65534) | `standards/claude-md/infra/CLAUDE.md` |
| Kubernetes security context, PDB, NetworkPolicy | `standards/claude-md/infra/CLAUDE.md` |
| Liveness/readiness probes, Prometheus scrape | `standards/claude-md/CLAUDE.md` |
| Progressive delivery (Canary, PromQL metric analysis) | `standards/detailed/release-engineering/progressive-delivery.md` |
| Isolated pre-upgrade database migration jobs | `standards/detailed/release-engineering/database-migrations.md` |
| Static manifest verification gate in CI | `standards/detailed/release-engineering/gitops-promotions.md` |
| Keyless Cosign signing and SLSA provenance | `standards/detailed/release-engineering/supply-chain-security.md` |
| Trivy image vulnerability scan (blocking HIGH+) | `standards/claude-md/CLAUDE.md` |
| Production manual approval gate | `standards/claude-md/infra/CLAUDE.md` |
