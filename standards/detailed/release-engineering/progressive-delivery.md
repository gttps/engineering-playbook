# Progressive Delivery Standards

Standards for canary deployments, metric-driven promotion gates, and automated blast-radius containment.

## What is Progressive Delivery?

Progressive delivery decouples software deployment from feature release, transforming continuous delivery into an automated, observable, and controlled progression. Rather than exposing 100% of user traffic to a newly deployed version simultaneously, progressive delivery deploys the new release alongside existing stable workloads, routes a tightly bounded initial slice of production traffic (e.g., 5%), continuously queries operational telemetry against statistical baselines, and advances traffic allocation through stepped stages (e.g., 25%, 50%, 100%). If telemetry detects latency spikes, error rate breaches, or downstream saturation, the system triggers an immediate, deterministic automated rollback with zero human intervention.

## Who it's for

- **Platform & SRE Teams:** Designing standardized deployment controllers, traffic routing meshes, telemetry analysis templates, and automated blast-radius guardrails across the platform.
- **Service Owners & Application Developers:** Defining canary traffic steps, bake windows, error rate ceilings, and latency SLAs tailored to specific microservice workload characteristics.
- **Release Engineers:** Orchestrating continuous delivery pipelines, GitOps promotion triggers, and post-deployment validation workflows.

---

## Core Delivery Principles (Tool-Agnostic)

Progressive delivery is defined by fundamental architectural principles that apply universally, regardless of the underlying container orchestrator, traffic router, or telemetry provider:

```mermaid
flowchart TD
    subgraph Release ["Progressive Delivery Lifecycle"]
        Deploy["1. Deploy Candidate Workload"] --> Route["2. Route Bounded Traffic Slice"]
        Route --> Analyze["3. Continuous Telemetry Analysis"]
        Analyze -->|"Metric Breach (Error / Latency)"| Abort["Automated Rollback (Instant)"]
        Analyze -->|"Pass Thresholds"| StepUp["4. Increment Traffic Allocation"]
        StepUp --> Analyze
        StepUp -->|"100% Traffic Verified"| Stable["5. Promote Candidate to Stable"]
    end
```

1. **Decouple Deployment from Release:** Deploying application bits to runtime infrastructure must remain distinct from routing production user traffic to that code.
2. **Blast Radius Containment:** Traffic shifting begins at an isolated, minimal percentage (e.g., 5%) so that unexpected failures or regressions impact a negligible fraction of transactions.
3. **Objective Metric-Driven Verification:** Human observation is replaced by continuous statistical telemetry evaluations sampled during defined bake windows.
4. **Deterministic Automated Rollback:** When operational thresholds are breached, the delivery system reverts traffic to the baseline immediately without human approval or manual intervention.
5. **Dual-Version Backward Compatibility:** Stable and candidate versions must coexist concurrently in production, requiring database schemas, caches, and inter-service APIs to maintain backward compatibility across adjacent versions ($N$ and $N+1$).
6. **Declarative Progression State Machine:** The entire promotion sequence—traffic steps, pause intervals, analysis gates, and rollback policies—is declared as version-controlled code.

---

## Comparative Selection Matrix

Organizations operating in diverse cloud environments or with existing vendor contracts can substitute components of the reference implementation without violating core delivery principles.

### Progressive Delivery Controllers

| Dimension | Argo Rollouts (Reference) | Flagger | Ingress-Native (e.g., NGINX, ALB) |
|---|---|---|---|
| **Resource Model** | Custom `Rollout` CRD (replaces `Deployment`) | Custom `Canary` CRD (wraps existing `Deployment`) | Standard `Deployment` + Ingress / Service annotations |
| **Traffic Routers** | Istio, Linkerd, NGINX, ALB, Traefik, Gateway API | Istio, Linkerd, App Mesh, NGINX, Gloo, Traefik, Gateway API | Ingress-specific (e.g., NGINX canary annotations, ALB weights) |
| **Analysis Engine** | Built-in `AnalysisTemplate` (10+ metric providers) | Built-in metric templates & webhooks | External CI/CD scripts or webhook pollers |
| **Rollback Latency** | Sub-second (controller resets router weights directly) | Sub-second (controller loops back to primary) | Seconds to minutes (driven by pipeline rerun or script execution) |
| **GitOps Fit** | Native (ArgoCD UI extension, health checks) | Excellent (designed for Flux; works with ArgoCD) | Moderate (requires managing two deployments or mutating annotations) |
| **Operational Overhead**| Controller deployment + CRD installation | Controller deployment + CRD installation | Zero in-cluster controller; high CI/CD pipeline scripting complexity |
| **Best Suited For** | Kubernetes platforms using ArgoCD or GitOps with CRDs | Platforms using Flux or teams mandating standard `Deployment` CRDs | Lightweight environments without cluster CRD install permissions |

- **Argo Rollouts (Reference):** Selected for CNCF Graduated maturity, native Kubernetes `Rollout` and `AnalysisTemplate` CRDs, first-class GitOps (ArgoCD) integration, declarative step-based analysis, and pluggable traffic routers (Istio, Linkerd, NGINX, Gateway API).
- **Flagger:** Suitable alternative for Flux-based workflows or platforms requiring standard Kubernetes `Deployment` resources.
- **Ingress-Native:** Suitable for lightweight environments lacking cluster CRD installation permissions; requires external pipeline orchestration.

---

### Telemetry & Metrics Providers

| Dimension | Prometheus (Reference) | Datadog | Cloud Monitoring / OpenTelemetry |
|---|---|---|---|
| **Query Language** | PromQL (vector operations, quantiles, rates) | Datadog Metric Query Syntax (`avg:`, `sum:`, `p99:`) | MQL / CloudWatch Metric Math / OTel PromQL endpoints |
| **Latency / Freshness** | Real-time (10–15s scrape interval, instant index) | Near real-time (15–60s ingest & aggregation delay) | 30–120s ingestion latency depending on cloud provider |
| **Cost Model** | Open-source infrastructure cost (RAM / persistent disk) | Commercial SaaS (per-host, per-custom-metric fees) | Cloud provider per-metric and per-API-call billing |
| **Authentication** | In-cluster service URL (Kubernetes RBAC or internal) | Secret-based `DD_API_KEY` and `DD_APP_KEY` | Cloud IAM (Workload Identity, IRSA, Managed Identity) |
| **Best Suited For** | Cloud-native Kubernetes clusters; unified Prometheus stack | Organizations with enterprise Datadog APM contracts | Managed cloud platforms (GKE, EKS, AKS) minimizing self-hosted tools |

- **Prometheus (Reference):** Selected for CNCF Graduated status, real-time scrape evaluation (10–15s), expressive PromQL vector/quantile math (`rate`, `histogram_quantile`), and production SLO query parity.
- **Datadog:** Alternative for enterprise APM environments; configured via controller API secrets and Datadog metric queries.
- **Cloud Monitoring / OpenTelemetry:** Alternative for managed cloud platforms; queried via Prometheus-compatible gateways (e.g., Google Managed Service for Prometheus, AWS Managed Prometheus) or cloud IAM (Workload Identity, IRSA).

---

## Traffic Shifting & Progression Models

Progressive delivery engines must implement stepped traffic routing through an ingress controller, service mesh, or cloud load balancer.

### Canary Progression Steps & Bake Windows

All production canary rollouts must follow a four-tier stepped progression. Direct jumps from 0% to 100% are strictly prohibited for production environments.

| Progression Step | Traffic Allocation | Minimum Bake Duration | Primary Validation Objective |
|---|---|---|---|
| **Phase 1: Seed** | 5% | 10 minutes | Runtime initialization, panic detection, immediate crash loops |
| **Phase 2: Initial** | 25% | 15 minutes | Early throughput verification, downstream dependency capacity |
| **Phase 3: Half-Load** | 50% | 20 minutes | P95/P99 latency stability, connection pool saturation |
| **Phase 4: Full** | 100% | 15 minutes | Final verification before stable replica set decommissioning |

Total minimum bake duration for a complete production canary progression is **60 minutes**. Lower environments (development and staging) may compress bake windows to 2–5 minutes per step.

### Blue-Green Cutover Criteria

Blue-green deployments maintain two identical environments: active (`blue`) and preview (`green`). Blue-green is required when service workloads exhibit breaking state transitions, database schema lock sensitivity, or long-lived asynchronous batch jobs that cannot tolerate canary traffic splitting.

Cutover from preview to active must satisfy the following deterministic criteria:

1. **Synthetic Verification**: 100% of pre-cutover smoke tests against the preview endpoint pass.
2. **Readiness Parity**: 100% of preview pods report healthy readiness probes for a minimum of 180 seconds.
3. **Instantaneous Atomic Cutover**: Ingress or service selector updates atomically to route 100% of live traffic to the preview environment.
4. **Teardown Delay**: The retired stable environment must remain running in an idle state for a minimum of 30 minutes to facilitate instant zero-rebuild rollback if post-cutover regressions occur.

---

## Telemetry-Driven Automated Rollback

Rollout progression must be coupled to real-time telemetry analysis. Analysis templates must execute automated PromQL queries during every bake window.

### PromQL Error Rate Ceilings

The HTTP 5xx error rate ceiling for canary pods must remain strictly below **0.1%** ($< 0.001$ of total requests). Any sustained breach triggers an immediate rollback. Queries must enforce an absolute traffic floor to prevent false-positive division-by-zero or small-sample anomalies:

```promql
(
  sum(rate(http_requests_total{job="workload-service",status=~"5..",version="canary"}[2m]))
  /
  sum(rate(http_requests_total{job="workload-service",version="canary"}[2m]))
) >= 0.001
and
sum(rate(http_requests_total{job="workload-service",version="canary"}[2m])) > 10
```

### P95 & P99 Latency Baselines

Canary latency degradation must not exceed **1.15x** ($\le 15\%$ increase) of the stable baseline latency across equivalent percentiles:

```promql
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="workload-service",version="canary"}[2m])) by (le))
>
(
  1.15 * histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="workload-service",version="stable"}[2m])) by (le))
)
```

### Analysis Evaluation Contracts

Automated analysis must adhere to the following configuration constraints:

- **Sampling Interval**: Metrics sampled every 30 to 60 seconds.
- **Failure Threshold**: Rollback triggers on **2 consecutive failed metric checks** or a cumulative failure count exceeding 3 across the entire rollout lifecycle.
- **Warmup Grace Period**: Metric evaluation starts 60 seconds after pods transition to `Ready` to prevent false positives from JIT compilation, cold start cache misses, or connection initialization.

---

## Reference Configuration: Argo Rollouts

The reference implementation demonstrates declarative progressive delivery using Argo Rollouts and Prometheus metric analysis:

```mermaid
flowchart LR
    Ingress["Traffic Ingress"] --> Router["Traffic Router (Ingress / Service Mesh)"]
    Router -->|"Canary %"| CanaryRS["Canary ReplicaSet"]
    Router -->|"Stable %"| StableRS["Stable ReplicaSet"]
    Controller["Progressive Controller"] -->|"Updates Weights"| Router
    Controller -->|"Queries Metrics"| Prometheus["Telemetry Provider (Prometheus)"]
```

### Rollout Step & Traffic Specification

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payments-service
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: payments-service-canary
      stableService: payments-service-stable
      trafficRouting:
        istio:
          virtualService:
            name: payments-vservice
            routes: [primary]
      analysis:
        templates:
          - templateName: payments-success-rate
      steps:
        - setWeight: 5
        - pause: { duration: 10m }
        - setWeight: 25
        - pause: { duration: 15m }
        - setWeight: 50
        - pause: { duration: 20m }
        - setWeight: 100
```

Accompanying `AnalysisTemplate` resource:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: payments-success-rate
spec:
  metrics:
    - name: error-rate-ceiling
      interval: 60s
      successCondition: result[0] < 0.001
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus-k8s.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{job="payments-service",status=~"5..",version="canary"}[2m]))
            /
            sum(rate(http_requests_total{job="payments-service",version="canary"}[2m]))
```

---

## Operational Checklist & Anti-Patterns

### Pre-Deployment Verification Checklist

- [ ] Workload is completely stateless or adheres to backward-compatible database schema evolution.
- [ ] Canary and stable services declare identical `PodDisruptionBudget` and resource reservations.
- [ ] PromQL queries in analysis templates have been tested against live Prometheus endpoints in staging.
- [ ] Fallback traffic routing is configured to instantly recover stable service if the rollout controller crashes.
- [ ] Active telemetry queries include sample-size guards (`> 10 req/sec`) to avoid zero-division errors during off-peak hours.

### Release Engineering Anti-Patterns

| Anti-Pattern | Operational Risk | Standard Compliance |
|---|---|---|
| **Manual Promotion Overrides** | Human fatigue and biased judgment overlook subtle error rate regressions. | Mandatory automated metric analysis determines step promotion. |
| **Omitting Latency Checks** | Rollout succeeds while memory leaks or thread locks double response times. | Analysis templates must evaluate both error rates and P95/P99 latency baselines. |
| **Instant 0% to 100% Cutover** | Undetected bugs cause instantaneous full-scale outage. | Minimum 4-step progressive canary progression required for Tier 1/2 services. |
| **In-Process Schema Migrations** | New schema breaks running stable pods during canary execution. | Database migrations must execute via isolated pre-upgrade jobs with backward compatibility. |
