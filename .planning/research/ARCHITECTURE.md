# Architecture Research: Data Products API Platform

**Domain:** GraphQL Supergraph with FGAC, Self-Hosted on Kubernetes, Multi-Cloud
**Researched:** 2026-03-23
**Overall Confidence:** MEDIUM-HIGH (most findings from official docs and verified sources)

---

## Request Flow & FGAC Placement

### The Request Lifecycle (4-Hop Model)

```
Consumer
   |
   | HTTP/HTTPS (JWT in Authorization header)
   v
[OSS API Gateway] — Kong or Tyk
   | - Rate limiting, bot protection, mTLS termination
   | - OIDC plugin: validates JWT, enriches request with claims
   | - OPA plugin: coarse-grained GraphQL query-level authorization
   | - Injects X-User-ID, X-User-Scopes, X-Tenant-ID headers
   |
   | HTTP (enriched headers forwarded)
   v
[Apollo Router / Hive Gateway] — Supergraph Layer
   | - Parses GraphQL operation
   | - Applies @authenticated, @requiresScopes, @policy directives
   | - Coprocessor hooks: RouterService (pre-parse) and SubgraphService (pre-fetch)
   | - Generates query plan (which subgraphs satisfy which fields)
   | - FGAC at field level: strips inaccessible fields from plan
   |
   | HTTP (per-subgraph query with forwarded identity headers)
   v
[Subgraph Service] — Data Product Owner
   | - Receives forwarded headers (user identity, scopes)
   | - Re-validates authorization for resource-level checks (ABAC)
   | - Resolves data from its own data source
   |
   | Database/API call
   v
[Data Source] — DB, REST API, message bus, etc.
```

### FGAC Placement: Three-Layer Defense

**Layer 1 — OSS API Gateway (Coarse-Grained)**
- JWT signature validation and expiry enforcement
- OAuth scope presence checks (not value-level)
- GraphQL query complexity and depth limiting
- Rate limiting per consumer/tenant
- Blocks unauthenticated requests before they reach the supergraph
- Confidence: HIGH (verified via Kong OPA integration docs)

**Layer 2 — Supergraph Router (Field-Level)**
- Apollo Router `@authenticated` directive: requires valid identity on any type/field
- `@requiresScopes(scopes: ["data:read:pii"])`: checks OAuth scope values
- `@policy` directive: integrates OPA coprocessor for complex rule evaluation
- Router coprocessor hook at `SubgraphService.request`: can abort, modify, or annotate per-subgraph calls
- Strips `@inaccessible` fields from composed schema entirely (never exposed to consumers)
- This is where cross-cutting FGAC policy is centrally enforced
- Confidence: HIGH (verified via Apollo Router docs, policy-as-code blog post)

**Layer 3 — Subgraph (Resource-Level)**
- Attribute-based access control (ABAC) when the rule depends on data not available at Layer 2
- Example: "User can only read records they own" — requires fetching the record's owner field
- Subgraph receives forwarded identity headers and applies resolver-level checks
- Never trust that upstream has already authorized; each subgraph should verify
- Confidence: MEDIUM (multiple sources agree on this defense-in-depth recommendation)

### FGAC Architecture Recommendation

Apply authorization in this priority order:

1. **Gateway**: Structural checks (is the token valid? does the tenant exist? is this route allowed?)
2. **Supergraph router**: Field/type-level policy enforcement via directives + coprocessor
3. **Subgraph**: Resource-instance-level checks where data-dependent decisions are required

Do NOT consolidate all FGAC at just Layer 1 (gateway). The gateway cannot inspect the query plan or know which fields are being resolved. The supergraph router is the correct primary enforcement point for GraphQL-native FGAC.

### Kong vs Tyk for the Gateway Role

**Recommendation: Kong OSS**

| Criterion | Kong | Tyk |
|-----------|------|-----|
| License | Apache 2.0 | MPL 2.0 |
| GraphQL Federation support | Plugin-based (limited) | Native Federation v1 |
| OSS plugin ecosystem | Very large | Smaller |
| OPA integration maturity | Documented, supported | Less documented |
| K8s Ingress Controller | Kong Ingress Controller (mature) | Available but less mature |
| GraphQL rate limiting | Plugin | Native |

If GraphQL Federation awareness at the gateway level (not just proxying) is required, Tyk has an advantage. For a pure proxy role (auth enforcement, rate limiting, routing to the supergraph), Kong is the more battle-tested choice.

**Decision trigger:** Use Tyk if you want the gateway to parse and validate GraphQL queries natively. Use Kong if the gateway role is purely auth + rate-limiting and GraphQL-awareness lives in the supergraph layer.

---

## Supergraph Composition & Schema Registry

### Two Options: Apollo GraphOS vs GraphQL Hive (Self-Hosted)

**Since the platform is self-hosted, Apollo GraphOS SaaS creates an external dependency. GraphQL Hive is the OSS self-hosted alternative.**

#### Option A: Apollo GraphOS (SaaS Schema Registry)
- Rover CLI: `rover subgraph publish <graph>@<variant> --schema ./schema.graphql --name <subgraph>`
- GraphOS validates compatibility, composes all subgraph schemas into a supergraph schema
- Apollo Router fetches updated supergraph via Apollo Uplink polling
- Built-in breaking change detection, operation checks
- Limitation: requires network access to GraphOS SaaS; not fully self-contained
- Confidence: HIGH (official Apollo docs)

#### Option B: GraphQL Hive (Self-Hosted Schema Registry)
- Fully OSS (MIT license), runs on your own Kubernetes cluster
- Same workflow: Hive CLI publishes subgraph schemas, Hive composes them
- Hive CDN serves the composed supergraph schema to the gateway
- External Composition feature allows bring-your-own composition logic
- Self-contained: no SaaS dependency
- Confidence: HIGH (The Guild official docs)

**Recommendation: GraphQL Hive self-hosted for a fully self-contained platform.**

### Composition Workflow

```
1. Data product team writes GraphQL schema (SDL)
2. CI pipeline runs: hive schema:check --service <name>
   - Validates schema against composition rules
   - Checks for breaking changes against existing consumers
   - Blocks merge if checks fail
3. On merge to main: hive schema:publish --service <name> --url http://subgraph-service/graphql
   - Hive validates + composes all subgraph schemas
   - New supergraph SDL stored in Hive CDN
4. Apollo Router / Hive Gateway polls Hive CDN for schema updates
   - Default poll interval: 10s (configurable)
   - Hot-reload: router updates query plan without restart
5. New data product subgraph is live in the supergraph
```

### Static vs Dynamic Composition

| Approach | Description | When to Use |
|----------|-------------|-------------|
| **Dynamic (recommended)** | Router polls schema registry; supergraph updated without redeploy | Production: live subgraph updates, minimal operator intervention |
| **Static (CI artifact)** | `rover supergraph compose` at CI time produces SDL file baked into router ConfigMap | Dev/staging: deterministic, reproducible builds |

For production: use dynamic composition with Hive CDN. For local dev and staging environments: static composition via Rover CLI produces a deterministic artifact.

### Schema Registration Data Flow

```
Subgraph repo (team-owned)
    |
    | rover/hive schema:publish
    v
Schema Registry (Hive self-hosted)
    | Composition engine validates + merges all subgraph schemas
    v
Hive CDN endpoint (internal)
    | /cdn/schemas/<target-id>/supergraph
    v
Apollo Router / Hive Gateway (polls every 10s)
    | Hot-reloads query plan from new supergraph schema
    v
Consumers receive updated supergraph automatically
```

---

## Kubernetes Topology

### Component Inventory and K8s Resources

#### 1. OSS API Gateway (Kong)

```
Namespace: api-gateway

Resources:
  DaemonSet or Deployment (Deployment recommended)
    replicas: 2-3 minimum, HPA up to N
    image: kong:3.x
    ports: 80 (HTTP), 443 (HTTPS), 8001 (Admin API)
  Service (LoadBalancer or NodePort behind external LB)
  ConfigMap: kong.conf, declarative config
  Secret: TLS certificates, OIDC client secret
  HorizontalPodAutoscaler: scale on CPU/RPS
  PodDisruptionBudget: minAvailable: 1

Notes:
  - Run as Deployment, not DaemonSet
  - DaemonSet is only appropriate if gateway must run on every node
    for local traffic interception (service mesh pattern); not needed here
  - Use Kong Ingress Controller (KIC) as the control plane for
    declarative K8s-native configuration
```

#### 2. Supergraph Router (Apollo Router or Hive Gateway)

```
Namespace: graph-platform

Resources:
  Deployment
    replicas: 2 minimum, HPA on CPU + custom metric (RPS)
    image: ghcr.io/apollographql/router:latest
    ports: 4000 (GraphQL), 8088 (health), 9090 (metrics)
  Service (ClusterIP — internal only, exposed via Gateway)
  ConfigMap: router.yaml (supergraph config, plugin config)
  Secret: Hive CDN credentials, coprocessor auth token
  HorizontalPodAutoscaler
  PodDisruptionBudget: minAvailable: 1

Notes:
  - Apollo Router is Rust-based, CPU-efficient; single container
  - Resource requests: 250m CPU, 256Mi RAM as starting baseline
  - Scale horizontally; Apollo Router is stateless
  - Apollo GraphOS Operator (CRD-based) can manage this declaratively
    if using managed federation
```

#### 3. Schema Registry (GraphQL Hive)

```
Namespace: graph-platform

Resources:
  Deployment: hive-app (main API + UI)
  Deployment: hive-usage-ingestor
  Deployment: hive-schema-service
  Deployment: hive-cdn (schema artifact serving)
  StatefulSet or external: PostgreSQL (schema metadata)
  StatefulSet or external: Redis (caching, session)
  StatefulSet or external: ClickHouse (usage analytics)
  Services: ClusterIP per component
  Ingress: hive.internal.company.com

Notes:
  - Hive has significant operational surface area
  - Use managed PostgreSQL, Redis from cloud provider to reduce ops burden
  - ClickHouse is optional (usage analytics); skip in MVP
```

#### 4. Data Product Subgraphs (per product team)

```
Namespace: data-product-<team-name>

Resources (per subgraph):
  Deployment
    replicas: 2 minimum
    image: <team-registry>/<product>-subgraph:<tag>
    port: 4001 (GraphQL)
  Service: ClusterIP
  ConfigMap: connection strings (non-secret)
  Secret: DB credentials, API keys
  NetworkPolicy: only allow ingress from graph-platform namespace
  ServiceAccount: for IRSA/Workload Identity (cloud DB access)

Notes:
  - Each subgraph is independently deployable
  - Team owns namespace; platform team owns graph-platform namespace
  - NetworkPolicy enforces that subgraphs are NOT directly accessible
    from outside the cluster — only via the supergraph router
```

#### 5. Coprocessor Service (FGAC)

```
Namespace: graph-platform

Resources:
  Deployment
    replicas: 2 minimum
    image: <internal>/fgac-coprocessor:<tag>
    port: 8080
  Service: ClusterIP
  ConfigMap: OPA policy bundle reference
  Secret: Policy engine credentials

Notes:
  - Coprocessor implements the FGAC logic called by Apollo Router
  - Can be language of choice (Go, Node.js, Python) — called over HTTP
  - Alternatively: embed OPA as a sidecar to the router pod
```

#### 6. OTel Collector

```
Namespace: observability (or kube-system)

Resources:
  DaemonSet: otel-collector-agent (one per node)
    - Collects node-level metrics, container logs
  Deployment: otel-collector-gateway (centralized)
    - Aggregates, batches, routes to backends
  ConfigMap: collector config (receivers, processors, exporters)

Notes:
  - DaemonSet agent: lightweight; collects host metrics + logs via filelog receiver
  - Gateway deployment: handles batching, tail sampling, routing
```

### Namespace Layout Summary

```
Namespaces:
  api-gateway        — Kong gateway
  graph-platform     — Supergraph router, Hive registry, coprocessor
  observability      — OTel collector, Prometheus, Grafana, Loki, Tempo
  data-product-*     — One namespace per data product team (N namespaces)
  infra              — Cert-manager, external-secrets, cluster-autoscaler
```

---

## Multi-Cloud Terraform Patterns

### Core Principle: Terraform Manages Clusters, Helm/Operators Manage Workloads

Terraform should NOT manage Kubernetes resources inside clusters (Deployments, Services, ConfigMaps). That creates tight coupling and race conditions between Terraform apply cycles. Terraform scope = cluster lifecycle and cloud resources. Helm / ArgoCD / Flux scope = workloads inside clusters.

Confidence: HIGH (HashiCorp official guidance, corroborated by community best practices)

### Recommended Directory Structure

```
terraform/
  modules/
    cluster/
      aws/           # EKS module
        main.tf      # aws_eks_cluster, aws_eks_node_group, VPC
        variables.tf # name, region, node_count, instance_type, k8s_version
        outputs.tf   # cluster_endpoint, cluster_ca, oidc_provider_arn
      gcp/           # GKE module
        main.tf      # google_container_cluster, google_container_node_pool
        variables.tf # same interface as aws/
        outputs.tf   # same shape as aws/
      azure/         # AKS module
        main.tf      # azurerm_kubernetes_cluster
        variables.tf # same interface as aws/
        outputs.tf   # same shape as aws/
    cloud-infra/
      aws/           # IAM roles, ECR, RDS, ElastiCache
      gcp/           # Workload Identity, Artifact Registry, Cloud SQL
      azure/         # Managed Identity, ACR, Azure Database
    networking/
      aws/           # VPC, subnets, NAT, security groups
      gcp/           # VPC, subnets, Cloud NAT, firewall rules
      azure/         # VNet, subnets, NSG
  environments/
    aws-prod/
      main.tf        # calls modules/cluster/aws + modules/cloud-infra/aws
      terraform.tfvars
      backend.tf     # S3 remote state
    gcp-prod/
      main.tf        # calls modules/cluster/gcp + modules/cloud-infra/gcp
      terraform.tfvars
      backend.tf     # GCS remote state
    azure-prod/
      main.tf        # calls modules/cluster/azure + modules/cloud-infra/azure
      terraform.tfvars
      backend.tf     # Azure Blob remote state
```

### Common Interface Pattern for Cluster Modules

Each cloud cluster module accepts the same input shape and produces the same output shape:

```hcl
# modules/cluster/<cloud>/variables.tf (same across all clouds)
variable "cluster_name"     { type = string }
variable "region"           { type = string }
variable "kubernetes_version" { type = string }
variable "node_pools" {
  type = list(object({
    name          = string
    instance_type = string  # cloud-specific mapping happens inside module
    min_nodes     = number
    max_nodes     = number
  }))
}
variable "tags" { type = map(string) }

# modules/cluster/<cloud>/outputs.tf (same across all clouds)
output "cluster_endpoint"    { value = ... }
output "cluster_ca_cert"     { value = ... }
output "cluster_name"        { value = ... }
output "oidc_provider"       { value = ... }  # for IRSA/Workload Identity
```

### Cross-Cloud State Sharing

Remote state data sources allow the workload layer (Helm releases via Terraform, or ArgoCD) to reference cluster outputs:

```hcl
# In a shared workload module or ArgoCD app-of-apps config:
data "terraform_remote_state" "eks_cluster" {
  backend = "s3"
  config = {
    bucket = "company-tf-state"
    key    = "aws-prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### Do NOT Abstract Managed Services

Cloud-specific managed services (RDS vs Cloud SQL vs Azure Database, ECR vs Artifact Registry vs ACR) have no equivalent API surface. Attempting a single abstraction module for these creates unmaintainable code. Keep provider-specific infrastructure modules separate and accept the duplication.

### Workload Deployment (Helm via ArgoCD, NOT Terraform)

```
Terraform output:
  cluster_endpoint, cluster_ca_cert
      |
ArgoCD (ApplicationSet)
  - watches git repo
  - deploys Helm charts to each cluster
  - cluster endpoints provided via ArgoCD cluster secrets
  - same Helm values.yaml per environment, cloud-specific overrides in values-aws.yaml etc.
```

---

## Observability Architecture

### Recommended OSS Stack: LGTM + OTel (Grafana Observability Stack)

| Signal | Collection | Storage | Query |
|--------|-----------|---------|-------|
| Metrics | OTel Collector → Prometheus scrape | Prometheus (or Mimir for HA) | PromQL in Grafana |
| Traces | OTel SDK → OTel Collector → Tempo | Grafana Tempo | TraceQL in Grafana |
| Logs | OTel Collector filelog receiver | Grafana Loki | LogQL in Grafana |
| Dashboards | — | — | Grafana |

Confidence: HIGH (CNCF blog post, multiple community sources, 2025 Kubernetes monitoring guide)

### Why LGTM over Jaeger + ELK

- All backends (Prometheus/Mimir, Loki, Tempo) accept OTLP directly — single wire protocol
- Grafana provides correlated view: traces link to logs link to metrics in a single UI
- Cloud-neutral: no cloud SDK dependency; runs identically on EKS, GKE, AKS
- Grafana Mimir (HA Prometheus) and Loki are horizontally scalable with object storage backends
- Jaeger is viable for traces-only but Tempo has better Grafana integration and object storage support

### OTel Collector Deployment Pattern

```
[Application Pods]
    | OTLP gRPC/HTTP (4317/4318)
    v
[OTel Collector Agent — DaemonSet]
    - Receives: OTLP from apps, filelog from /var/log/containers
    - Processes: add k8s attributes, filter sensitive fields, batch
    - Exports: OTLP to OTel Collector Gateway
    |
    | OTLP gRPC (compressed, batched)
    v
[OTel Collector Gateway — Deployment, 2-3 replicas]
    - Receives: from all DaemonSet agents
    - Processes: tail sampling for traces, metric aggregation
    - Exports:
        traces  → Tempo (OTLP)
        metrics → Prometheus remote_write (or Mimir)
        logs    → Loki (via Loki exporter)
    |
    +--> Grafana Tempo    (traces storage, object storage backend: S3/GCS/Azure Blob)
    +--> Prometheus/Mimir (metrics storage)
    +--> Grafana Loki     (log storage, object storage backend)
    |
    v
[Grafana] — single visualization layer
    - Unified dashboards: metrics + traces + logs in one view
    - Exemplars: Prometheus metrics link to Tempo traces
    - Alerting: Grafana Alerts or Prometheus AlertManager
```

### GraphQL-Specific Instrumentation

Apollo Router emits OpenTelemetry traces natively:
- Each supergraph request generates a root span
- Each subgraph fetch generates a child span with service name, operation name
- Query planning is instrumented as a separate span
- Configure via `router.yaml`:

```yaml
telemetry:
  exporters:
    tracing:
      otlp:
        endpoint: "http://otel-collector-agent:4317"
  instrumentation:
    spans:
      router:
        attributes:
          graphql.operation.name: true
          graphql.operation.type: true
```

Kong emits OpenTelemetry traces via the `opentelemetry` plugin.

### Multi-Cloud Observability Deployment

Run one centralized Grafana stack per cloud (do NOT try to federate across clouds in Phase 1):

```
AWS cluster:  OTel agents → local Tempo/Loki/Prometheus → local Grafana
GCP cluster:  OTel agents → local Tempo/Loki/Prometheus → local Grafana
Azure cluster: OTel agents → local Tempo/Loki/Prometheus → local Grafana
```

Use object storage native to each cloud as the Tempo/Loki backend:
- AWS: S3 buckets
- GCP: GCS buckets
- Azure: Azure Blob Storage

This avoids cross-cloud data egress costs while keeping the observability stack identical.

---

## GitOps Registration Flow

### The Problem

When a new data product team wants to add their subgraph to the platform, they need to:
1. Register their subgraph schema in the schema registry
2. Deploy their subgraph service to Kubernetes
3. Have the supergraph automatically pick up their schema

This must happen through a PR workflow for auditability and governance.

### Recommended GitOps Flow

```
Two Git Repositories:
  1. data-product-<team>/ (team-owned)  — application code + schema
  2. platform-gitops/                   — cluster state, ArgoCD apps, subgraph registrations
```

#### Step 1: Team opens a PR in their subgraph repo

```
data-product-orders/
  schema.graphql         # GraphQL SDL (subgraph schema)
  src/                   # resolver implementation
  Dockerfile
  .github/workflows/
    check.yml            # runs on PR open
    publish.yml          # runs on merge to main
```

#### Step 2: CI runs on PR (check.yml)

```yaml
# .github/workflows/check.yml
- name: Validate schema compatibility
  run: hive schema:check \
    --service orders \
    --schema ./schema.graphql \
    --registry-endpoint http://hive.internal/
  # Fails PR if: breaking change, composition error, policy violation
```

#### Step 3: On merge to main (publish.yml)

```yaml
# .github/workflows/publish.yml
- name: Build and push Docker image
  run: docker build -t registry/orders-subgraph:$SHA .

- name: Publish schema to Hive
  run: hive schema:publish \
    --service orders \
    --schema ./schema.graphql \
    --url http://orders-subgraph.data-product-orders.svc.cluster.local/graphql \
    --registry-endpoint http://hive.internal/

- name: Open PR to platform-gitops repo
  # Uses GitHub Actions to create PR in platform-gitops:
  # - Updates Helm values with new image tag
  # - Creates/updates ArgoCD Application manifest for this subgraph
```

#### Step 4: Platform team approves PR in platform-gitops

```
platform-gitops/
  apps/
    data-product-orders/
      application.yaml   # ArgoCD Application resource
      values.yaml        # Helm values for orders subgraph
  subgraph-registry/
    orders.yaml          # SubgraphRegistration CRD (if using GraphOS Operator)
```

ArgoCD detects the merged PR and deploys the subgraph to the cluster.

#### Step 5: Supergraph auto-updates

After ArgoCD deploys the subgraph service and the schema was published in Step 3, the Apollo Router / Hive Gateway polls the schema registry and hot-reloads the updated supergraph — no router restart required.

### Full GitOps Registration Sequence

```
[Data Product Team]
    | 1. git push: schema.graphql + resolver code
    v
[PR in data-product-orders repo]
    | 2. CI: hive schema:check (validates composition compatibility)
    | 3. PR review by data product team + optional platform approval
    v
[Merge to main]
    | 4. CI: docker build + push image to registry
    | 5. CI: hive schema:publish (registers schema; Hive composes supergraph)
    | 6. CI: opens auto-PR in platform-gitops repo
    v
[PR in platform-gitops repo]
    | 7. Platform team approves (governance gate)
    v
[Merge to platform-gitops main]
    | 8. ArgoCD detects change; deploys subgraph Deployment + Service
    v
[Subgraph running in Kubernetes]
    | 9. Apollo Router polls Hive CDN; detects new supergraph schema
    v
[Supergraph updated — new data product live]
```

### Self-Service vs Governed Registration

| Model | When to Use | Trade-Off |
|-------|-------------|-----------|
| **Fully governed** (platform approval on platform-gitops PR) | Regulatory environments, strict schema governance | Slower onboarding, human bottleneck |
| **Auto-merge with policy gates** (CI policy checks must pass, no human approval) | Fast-moving teams, trusted internal data products | Requires robust automated schema validation |
| **Hybrid** (CI gates required, human approval optional for first registration only) | Most platforms | Balances governance and speed |

Recommendation: Start with hybrid — require human approval for a subgraph's first registration, auto-merge updates thereafter if CI gates pass.

---

## Reference Architecture Diagram

```
                         ┌─────────────────────────────────────────────────────────────┐
                         │                    KUBERNETES CLUSTER                        │
                         │                                                             │
  [Consumers]            │  ┌──────────────┐    ┌──────────────────────────────────┐  │
  Web / Mobile /  ──────────>│   Kong OSS   │    │       graph-platform ns          │  │
  Service clients        │  │  API Gateway │    │                                  │  │
                         │  │              │    │  ┌────────────────────────────┐   │  │
                         │  │ - JWT verify │────┼─>│    Apollo Router /         │   │  │
                         │  │ - OPA coarse │    │  │    Hive Gateway             │   │  │
                         │  │   auth       │    │  │  (Supergraph)               │   │  │
                         │  │ - Rate limit │    │  │                             │   │  │
                         │  │ - mTLS term  │    │  │  @authenticated             │   │  │
                         │  └──────────────┘    │  │  @requiresScopes            │   │  │
                         │         │            │  │  @policy + coprocessor      │   │  │
                         │         │            │  └────────────┬───────────────┘   │  │
                         │         │            │               │ query plan         │  │
                         │         │            │               │                    │  │
                         │         │            │  ┌────────────v──────────────────┐│  │
                         │         │            │  │      FGAC Coprocessor          ││  │
                         │         │            │  │  (OPA policies, field filter)  ││  │
                         │         │            │  └───────────────────────────────┘│  │
                         │         │            │                                    │  │
                         │         │            │  ┌─────────────────────────────┐  │  │
                         │         │            │  │    Hive Schema Registry      │  │  │
                         │         │            │  │  (composition, CDN, checks)  │  │  │
                         │         │            │  └─────────────────────────────┘  │  │
                         │         │            └──────────────────────────────────┘  │
                         │         │                                                   │
                         │         │     ┌─────────────────────────────────────────┐  │
                         │         │     │         data-product-* namespaces        │  │
                         │         │     │                                          │  │
                         │         │     │  ┌──────────┐  ┌──────────┐  ┌───────┐  │  │
                         │         │     │  │ Products │  │ Orders   │  │ Users │  │  │
                         │         │     │  │ Subgraph │  │ Subgraph │  │Subgraph│  │  │
                         │         │     │  └────┬─────┘  └────┬─────┘  └───┬───┘  │  │
                         │         │     │       │              │            │       │  │
                         │         │     │  ┌────v─────┐  ┌────v─────┐  ┌───v───┐  │  │
                         │         │     │  │PostgreSQL│  │  MongoDB │  │  RDS  │  │  │
                         │         │     │  └──────────┘  └──────────┘  └───────┘  │  │
                         │         │     └─────────────────────────────────────────┘  │
                         │         │                                                   │
                         │         │     ┌─────────────────────────────────────────┐  │
                         │         │     │         observability ns                 │  │
                         │         │     │                                          │  │
                         │  OTel   │     │  OTel Agent     OTel Gateway             │  │
                         │  spans ─┼─────┼─> (DaemonSet) ──> (Deployment)          │  │
                         │  metrics│     │                      │                   │  │
                         │  logs   │     │             ┌────────┼────────┐          │  │
                         │         │     │             v        v        v          │  │
                         │         │     │           Tempo   Prometheus  Loki       │  │
                         │         │     │             └────────┼────────┘          │  │
                         │         │     │                      v                   │  │
                         │         │     │                   Grafana                │  │
                         │         │     └─────────────────────────────────────────┘  │
                         └─────────────────────────────────────────────────────────────┘

                         ┌─────────────────────────────────────────────────────────────┐
                         │                    GITOPS / CI LAYER                        │
                         │                                                             │
                         │  data-product-X repo ──CI──> Hive schema:publish            │
                         │                         ──CI──> PR to platform-gitops repo  │
                         │                                        │                    │
                         │  platform-gitops repo ──ArgoCD──> K8s cluster state         │
                         │                                                             │
                         │  terraform/ ──Terraform──> EKS / GKE / AKS cluster lifecycle│
                         └─────────────────────────────────────────────────────────────┘
```

---

## Sources

- [Apollo Router Kubernetes Quickstart](https://www.apollographql.com/docs/router/containerization/kubernetes) — HIGH confidence
- [Apollo Router External Coprocessor](https://www.apollographql.com/docs/graphos/routing/customization/coprocessor) — HIGH confidence
- [Apollo Policy-as-Code for GraphQL](https://www.apollographql.com/blog/centrally-enforce-policy-as-code-for-graphql-apis) — HIGH confidence
- [Apollo Supergraph Schema Composition](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/composition) — HIGH confidence
- [GraphQL Hive Self-Hosted](https://the-guild.dev/graphql/hive/docs/self-hosting/get-started) — HIGH confidence
- [Kong GraphQL Authorization with OPA](https://konghq.com/blog/engineering/graphql-authorization-at-the-api-gateway-with-kong-konnect-and-opa) — HIGH confidence
- [GraphQL Federation overview (graphql.org)](https://graphql.org/learn/federation/) — HIGH confidence
- [supergraph-demo-k8s-graph-ops (GitOps patterns)](https://github.com/apollographql/supergraph-demo-k8s-graph-ops) — MEDIUM confidence (archived repo, patterns still applicable)
- [Terraform Multi-Cloud Kubernetes Tutorial](https://developer.hashicorp.com/terraform/tutorials/networking/multicloud-kubernetes) — HIGH confidence
- [CNCF: Cost-Effective OTel Observability](https://www.cncf.io/blog/2025/12/16/how-to-build-a-cost-effective-observability-platform-with-opentelemetry/) — HIGH confidence
- [OTel + Grafana LGTM Stack](https://dev.to/improving/end-to-end-observability-with-prometheus-grafana-loki-opentelemetry-and-tempo-3fpf) — MEDIUM confidence
- [Tyk vs Kong Comparison (2025)](https://apidog.com/blog/tyk-vs-kong/) — MEDIUM confidence
- [Multi-Cloud Terraform Module Patterns](https://oneuptime.com/blog/post/2026-02-23-how-to-create-terraform-modules-for-multi-cloud/view) — MEDIUM confidence
- [OSS Authorization Patterns in GraphQL](https://www.osohq.com/post/graphql-authorization) — MEDIUM confidence
- [Grafbase: Security Considerations in GraphQL Federation](https://grafbase.com/blog/security-considerations-in-graphql-federation) — MEDIUM confidence
