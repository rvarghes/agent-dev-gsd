# Phase 1: Foundation Infrastructure - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-25
**Phase:** 01-foundation-infrastructure
**Areas discussed:** Technology corrections, Repo strategy, GitLab CD model, Istio scope, Private connectivity, GraphQL federation router evaluation, Observability platform, EKS topology, Test subgraph

---

## Technology Corrections (overrides to initial research)

Several recommendations from the initial research phase were overridden by firm-approved technology constraints:

| Original Research Recommendation | Correction | Reason |
|----------------------------------|------------|--------|
| ArgoCD for GitOps | **GitLab CI/CD** | ArgoCD not approved in the firm |
| Traefik for ingress | **Istio** | Traefik not approved; Istio is firm-approved |
| Hive Gateway for federation | **WunderGraph Cosmo** | Evaluated all three options — see full comparison below |
| LGTM stack (Loki, Grafana, Tempo, Mimir) | **Observe (observeinc.com)** | Firm has ELK (legacy) and Observe (new strategic platform) |

---

## Repo Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Two repos: platform-infra + platform-gitops | Separate Terraform and K8s manifests | |
| Single monorepo | Terraform + K8s manifests together | ✓ |

**User's choice:** Single monorepo
**Notes:** Simpler for the team; easier cross-referencing; one CI pipeline.

---

## GitLab CD Model

| Option | Description | Selected |
|--------|-------------|----------|
| GitLab CI push deploy | Pipelines run kubectl/helm to push to EKS | ✓ |
| GitLab + Flux (pull model) | Flux agent in cluster pulls from GitLab | |
| GitLab Kubernetes Agent | Native GitLab agent for pull-based deploys | |

**User's choice:** GitLab CI push deploy
**Notes:** ArgoCD not approved in firm. GitLab is the standard.

---

## Istio Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Ingress gateway only | Istio as ingress; mTLS optional; JWT at Tyk | ✓ |
| Full service mesh from day one | mTLS + traffic policies across all services | |

**User's choice:** Ingress gateway only — mTLS optional
**Notes:** Focus on traffic access control via CIDR and security groups. Authentication and authorization via JWT at Tyk, not Istio policy. mTLS deferred as optional.

---

## Private Connectivity

| Option | Description | Selected |
|--------|-------------|----------|
| AWS Direct Connect | Dedicated private circuit | |
| Site-to-Site VPN | IPSec over internet | |
| Already have Direct Connect | Existing connectivity in place | ✓ |

**User's choice:** Already have Direct Connect
**Notes:** Platform deploys into existing VPC/subnet and inherits connectivity. No new provisioning needed. No public internet traffic — all consumers are AWS-hosted or on-prem.

---

## GraphQL Federation Router — Full Three-Way Evaluation

### Comparison Table

| Feature | Hive Gateway | WunderGraph Cosmo | Apollo Router |
|---------|-------------|-------------------|---------------|
| **License** | MIT | Apache 2.0 | ELv2 (source-available, not OSI) |
| **Router language** | JavaScript / Rust | Go | Rust |
| **Federation compliance** | 100% (self-published audit) | 94.3% | Reference implementation |
| **Schema registry — self-hosted** | Yes — no official Helm chart (DIY K8s manifests) | Yes — official Helm chart for router + full platform | **No — SaaS only** (Apollo GraphOS cloud) |
| **OPA / PII masking** | TypeScript plugin via `onExecutionResult` hook — no binary recompile needed | Custom Go module compiled into router binary | `@policy` directive — **Enterprise license required** |
| **OPA coprocessor protocol** | DIY via HTTP hook | DIY via Go module | Well-documented protocol + reference impl — **Enterprise** |
| **JWT authentication** | Yes (free) | Yes (free) | **Enterprise only** |
| **Kubernetes Helm chart** | No official chart for registry (gateway is stateless and easy) | Official Helm charts for router AND full control plane | Official Helm chart per release for router only |
| **EKS deployment** | Gateway: easy. Registry: complex DIY K8s manifests | Official Helm chart + documented EKS examples | No blockers for router; schema registry is SaaS |
| **OpenTelemetry** | Full — traces, metrics, logs (free) | Full — traces, metrics, logs (free) | Full — traces, metrics, logs (free) |
| **Free production use** | Unlimited | Unlimited | Rate-limited to 60 req/min — paid plan required for production |
| **Self-hosted with no vendor cloud** | Yes — fully self-contained | Yes — fully self-contained | Enterprise offline license required for key features; schema registry always SaaS |
| **GitHub stars / contributors** | ~80 stars / 13 contributors | ~1,200 stars / 79 contributors | ~953 stars / 210+ contributors (corporate-backed) |
| **Self-hosting complexity** | Low for gateway; High for registry (no Helm) | Medium — many dependencies but official Helm chart handles it | Router: easy with Helm; registry: impossible self-hosted |
| **Cost** | Free | Free | Enterprise pricing required for JWT, OPA, Redis cache, demand control |

### Why Apollo Router was eliminated
- JWT authentication is Enterprise-only
- `@policy` directive (required for schema-level OPA field masking) is Enterprise-only
- Schema registry (GraphOS Studio) is SaaS-only — no self-hosted option from Apollo
- Free tier capped at 60 req/min — unsuitable for any production workload
- Creates a hard commercial dependency on Apollo for all critical security features

### Why WunderGraph Cosmo was chosen over Hive Gateway
- Official Helm charts for both the stateless router AND the full control plane (schema registry, composition validation, version history)
- Documented EKS deployment examples in the repository
- More community traction: ~15x more GitHub stars, 6x more contributors
- Go binary is more resource-efficient than Node.js for the router workload
- The OPA integration trade-off (Go module vs TypeScript plugin) was accepted given the operational benefits

**User's choice:** WunderGraph Cosmo

---

## Observability Platform

| Option | Description | Selected |
|--------|-------------|----------|
| Observe (observeinc.com) | New firm strategic observability platform | ✓ |
| ELK stack | Legacy firm standard | |
| Both — ELK for logs, Observe for traces/metrics | Dual integration | |

**User's choice:** Observe (observeinc.com)
**Notes:** ELK is the legacy standard in the org; Observe is the new strategic offering. Build against Observe from day one.

---

## EKS Cluster Topology

| Option | Description | Selected |
|--------|-------------|----------|
| Single cluster, 3 AZ, mixed nodes | One cluster, HA across 3 AZs, on-demand + spot | ✓ |
| Single cluster, 2 AZ, on-demand only | Simpler, cheaper bring-up | |

**User's choice:** Single cluster, 3 AZ, mixed nodes
**Notes:** Platform and data product workloads share one cluster (separated by namespace). On-demand for platform-critical, spot for non-critical.

---

## Test Subgraph

| Option | Description | Selected |
|--------|-------------|----------|
| Realistic mock data product | Cosmo federation contract + @pii field + OPA stub | ✓ |
| Minimal hello-world | Validates routing chain only | |

**User's choice:** Realistic mock data product
**Notes:** Should exercise the schema publication workflow through the Cosmo registry. Must include a `@pii`-tagged field and stub OPA policy check to de-risk Phase 2 prerequisites.

---

## Claude's Discretion

- Terraform workspace naming conventions within the monorepo
- Specific EKS node instance types (on-demand vs spot groups)
- OTel Collector DaemonSet resource limits
- Istio ingress gateway sizing
- Cosmo Control Plane dependency sizing (PostgreSQL, ClickHouse, Redis) for Phase 1
