# Phase 1: Foundation Infrastructure - Context

**Gathered:** 2026-03-25
**Status:** Ready for planning

<domain>
## Phase Boundary

Full platform skeleton — EKS cluster, Istio ingress, Tyk OSS gateway, WunderGraph Cosmo router + schema registry, and Observe observability — running via GitLab CI/CD GitOps with a realistic mock data product subgraph composing successfully.

Note: Several decisions from initial research were overridden during discussion based on firm-approved technology constraints. See decisions below.

</domain>

<decisions>
## Implementation Decisions

### Repository Structure
- **D-01:** Single monorepo — Terraform IaC and Kubernetes manifests together in one repo. No split between platform-infra and platform-gitops.

### CI/CD and Deployment
- **D-02:** GitLab CI/CD for all deployments — ArgoCD is not approved in the firm. GitLab CI push deploy model: pipelines run kubectl/helm commands to push changes directly to EKS.
- **D-03:** No ArgoCD. GitLab is the firm-standard GitOps tool.

### Ingress and Service Mesh
- **D-04:** Istio replaces Traefik as the ingress layer — Traefik is not approved; Istio is firm-approved alongside Tyk.
- **D-05:** Istio scope for Phase 1: ingress gateway only. mTLS between services is optional — defer unless needed. Traffic access control via CIDR ranges and AWS security groups. JWT authentication and authorization handled by Tyk (not Istio policy).
- **D-06:** No public internet traffic. All API consumers are internal (AWS-hosted or on-prem). Ingress is private-only.

### Networking and Connectivity
- **D-07:** AWS Direct Connect is already provisioned in the firm. Platform deploys into the existing VPC/subnet and inherits private connectivity. No new connectivity provisioning needed in Phase 1.

### GraphQL Federation Router
- **D-08:** WunderGraph Cosmo (Apache 2.0) replaces Hive Gateway. Chosen over Hive Gateway (no official Helm chart for registry) and Apollo Router (ELv2 license; JWT auth and OPA/`@policy` directive are Enterprise-only; schema registry is SaaS-only with no self-hosted option).
- **D-09:** Cosmo Router handles supergraph federation. Cosmo Control Plane (self-hosted via official Helm chart) handles schema registry, composition validation, version history.
- **D-10:** OPA coprocessor integration for field-level PII masking will be built as a custom Go module compiled into the Cosmo Router binary — scoped to Phase 2, not Phase 1.

### Observability
- **D-11:** Observe (observeinc.com) replaces the LGTM stack. Observe is the firm's new strategic observability platform (ELK is legacy). Platform components ship logs, metrics, and traces to Observe.
- **D-12:** OTel Collector deployed as DaemonSet — collects from all platform components and forwards to Observe via OTLP.

### EKS Cluster
- **D-13:** Single EKS cluster for all workloads (platform + data product namespaces). Separate clusters deferred until scale demands it.
- **D-14:** 3 Availability Zones for HA.
- **D-15:** Mixed node groups — on-demand for platform-critical workloads, spot instances for non-critical/batch workloads to control cost.

### Test Subgraph
- **D-16:** Realistic mock data product subgraph — not a hello-world. Must implement the Cosmo federation contract, include a `@pii`-tagged field, and include a stub OPA policy check. Validates the full routing chain (Istio → Tyk → Cosmo Router → subgraph) and de-risks Phase 2 OPA/PII masking prerequisites.

### Claude's Discretion
- Terraform workspace naming conventions within the monorepo
- Specific node instance types for on-demand vs. spot groups (optimize for cost/availability)
- Exact OTel Collector DaemonSet resource limits
- Istio ingress gateway sizing
- Cosmo Control Plane PostgreSQL/ClickHouse/Redis sizing for Phase 1

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` — Full v1 requirements; Phase 1 covers INFRA-01 through INFRA-08 and OBS-01 through OBS-04
- `.planning/ROADMAP.md` — Phase 1 goal, success criteria, and 4-plan breakdown

### Research Artifacts
- `.planning/research/STACK.md` — Original stack research (note: several recommendations overridden by firm constraints — see decisions above)
- `.planning/research/ARCHITECTURE.md` — Architecture decisions and rationale
- `.planning/research/PITFALLS.md` — Top risks and mitigation strategies

### Technology Comparison (captured during discussion)
- `.planning/phases/01-foundation-infrastructure/01-DISCUSSION-LOG.md` — Full GraphQL router comparison table (Hive Gateway vs WunderGraph Cosmo vs Apollo Router) with evaluation rationale

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — greenfield project. No existing codebase to build on.

### Established Patterns
- None established yet — Phase 1 sets the patterns all subsequent phases will follow.

### Integration Points
- AWS Direct Connect already provisioned — platform VPC/subnet must be placed to inherit existing private connectivity
- GitLab CI/CD — existing firm CI/CD system; pipeline templates may be available in the firm's GitLab instance
- Observe (observeinc.com) — existing firm observability platform; OTLP ingest endpoint and credentials will need to be obtained

</code_context>

<specifics>
## Specific Ideas

- Tyk and Istio are both firm-approved. Tyk handles JWT auth/quota/rate limiting; Istio handles ingress and network-layer access control. They complement rather than duplicate.
- WunderGraph Cosmo was chosen specifically because it ships official Helm charts for both the stateless router and the full control plane — critical for EKS operational simplicity.
- The test subgraph should feel like a real data product team's first registration — it should exercise the schema publication workflow through the Cosmo registry, not just be wired up manually.

</specifics>

<deferred>
## Deferred Ideas

- mTLS between services via Istio — optional, defer to later phase if security posture requires it
- OPA coprocessor / Go module for field-level PII masking — Phase 2 scope
- Separate clusters for platform vs. data products — defer until scale demands it
- Mimir for long-term metrics storage — not needed; Observe replaces entire LGTM stack
- Data lineage tracking — out of scope for v1 per PROJECT.md

</deferred>

---

*Phase: 01-foundation-infrastructure*
*Context gathered: 2026-03-25*
