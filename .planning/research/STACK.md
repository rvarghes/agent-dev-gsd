# Stack Research: Data Products API Platform

**Researched:** 2026-03-23
**Confidence:** MEDIUM-HIGH (verified against official docs and independent benchmarks where available)

---

## GraphQL Federation

### The Landscape (2026)

Five credible options exist for a production federation supergraph. Only three are serious contenders for a data-products platform requiring self-hosting and strong OSS licensing.

| Gateway | License | Federation Spec Compliance | Sep 2025 RPS | P95 Latency | Notes |
|---------|---------|---------------------------|-------------|-------------|-------|
| **Hive Gateway (Rust QP)** | MIT | 100% (192/192) | ~1,827 | ~48ms | Newest; Rust query planner |
| **Hive Gateway (Node QP)** | MIT | 99.5% (191/192) | — | — | Stable; JS query planner |
| **Apollo Router** | ELv2 | 97.4% (187/192) | ~317 | ~201ms | Enterprise lock-in for FGAC directives |
| **Cosmo Router** | Apache 2.0 | 94.3% (181/192) | Competitive | — | Full self-hosted OSS platform |
| **Bramble** | MIT | Custom spec (not Fed v2) | — | — | Movio-internal, low adoption |
| **Hasura DDN** | Proprietary | Metadata-driven (not Fed v2) | — | — | Managed-cloud bias; not truly self-hosted OSS |

Federation audit scores from the independent audit run by The Guild (https://the-guild.dev/graphql/hive/federation-gateway-audit), confirmed as of the audit's 192 test-case suite.

### Key Findings

**Apollo Router (ELv2) critical constraint:** The `@authenticated`, `@requiresScopes`, and `@policy` directives — the native GraphQL-layer FGAC mechanisms — are **Enterprise-license-only features** (confirmed via Apollo docs: https://www.apollographql.com/docs/graphos/reference/router/graphos-features). For a self-hosted, OSS-first platform implementing fine-grained access control at the schema layer, this is a blocking limitation. You would need to implement authorization entirely in resolvers or a co-processor.

**Apollo Router license (ELv2):** Apollo Router is NOT open source (OSI definition). The Elastic License v2 prohibits offering the software as a managed service. For internal deployment this is usually acceptable, but it creates long-term vendor dependency risk. The `@apollo/composition` library (for composing subgraphs) carries the same ELv2 restriction.

**Hive Gateway (The Guild) advantages for this use case:**
- 100% federation spec compliance (best in class)
- MIT licensed — truly open source, no lock-in
- The Guild provides an MIT-licensed drop-in replacement for `@apollo/composition`, removing the last Apollo ELv2 dependency
- Significantly higher throughput than Apollo Router in independent benchmarks (1,827 vs 317 RPS; 6x headroom)
- Lower memory footprint (53 MB vs 193 MB per instance)
- Native Kubernetes deployment via Helm

**Cosmo Router (WunderGraph) as runner-up:**
- Apache 2.0 — fully open source
- Complete self-hostable platform: router + schema registry + analytics in one Helm chart
- Raised $7.5M Series A (eBay Ventures) in March 2025 — funded roadmap
- 94.3% federation compliance (11 test cases failing) — lower than Hive but improving
- Built-in schema registry (Cosmo Control Plane) is a significant differentiator for teams wanting one bundle
- Go-based router; strong K8s-native story

**Bramble:** Implements Movio's own federation spec, not Apollo Federation v2. Incompatible with the Federation v2 ecosystem. Small community (510 stars). Ruled out.

**Hasura DDN:** Metadata-driven supergraph with strong data-product semantics, but v3 is cloud-hosted-first and the self-hosting story for DDN is immature. Does not implement Apollo Federation v2 spec. Ruled out for this use case.

### Recommendation

**Use Hive Gateway (The Guild) with Hive schema registry.**

Rationale:
1. Best federation spec compliance (100%) — reduces edge-case bugs in complex cross-subgraph queries
2. MIT license with MIT-licensed composition library — no ELv2 exposure anywhere in the critical path
3. 6x throughput and 4x lower latency vs Apollo Router in independent benchmarks (Sep 2025, Grafbase)
4. Native FGAC integration via Envelop plugins (The Guild's plugin system), which enables schema-directive-based field masking without enterprise licensing fees
5. Hive schema registry provides CI/CD schema check workflows, changelog, and usage analytics as a self-hostable OSS service

**If you want one self-hosted platform bundle (router + registry + analytics):** Cosmo Router is a credible alternative. Its schema registry is more opinionated but batteries-included. Choose Cosmo if you want less integration work upfront and are comfortable with 94% (vs 100%) federation compliance.

---

## Access Control (FGAC)

### The Landscape

Four engines evaluated: OPA, Casbin, SpiceDB/OpenFGA, and AWS Cedar.

| Engine | Model | Strengths | Weaknesses | GraphQL Fit |
|--------|-------|-----------|------------|-------------|
| **OPA** | General-purpose Rego | Infrastructure-wide policy, partial evaluation for SQL push-down | Rego learning curve; Styra/maintainer uncertainty (Aug 2025) | HIGH — supports both row-filter generation and field-masking decisions |
| **OpenFGA** | ReBAC (Zanzibar-inspired) | Relationship-based, Google-scale patterns, fast (10x check speedup 2025) | No native row-filter generation; relationship-store adds operational overhead | MEDIUM — excellent for "who can see resource Z", less natural for "filter rows by attribute" |
| **SpiceDB** | ReBAC (Zanzibar-inspired) | Most mature OSS Zanzibar impl; multi-backend (CockroachDB, Spanner, Postgres) | Same as OpenFGA: not designed for row-filter generation | MEDIUM |
| **AWS Cedar** | Application-level ABAC | Human-readable, formally verified, strong correctness guarantees | AWS-ecosystem bias; infrastructure to manage; less mature for K8s | LOW for cloud-neutral OSS requirement |
| **Casbin** | Embedded RBAC/ABAC | Lightweight, embeds in application, many language SDKs | No central policy store; logic embedded in services = no audit consistency | LOW for distributed supergraph |

### Key Findings

**OPA's Styra/maintainer situation (Aug 2025):** Apple hired Styra's core OPA maintainers (Torin Sandall, Tim Hinrichs, Teemu Koponen). The OPA project itself remains under CNCF governance — the open source codebase is unaffected. Commercial Styra DAS is dead. Multiple maintainers from the broader OPA community continue active contributions. CNCF project status provides institutional governance continuity. **Confidence: MEDIUM** — monitor for maintainer churn over next 12 months.

**OPA for row-level filtering:** OPA supports **partial evaluation**, which compiles a Rego policy into a set of conditions that can be pushed down as a SQL `WHERE` clause. This is the key mechanism for row-level security without fetching all rows. The service queries OPA with user identity and resource metadata; OPA returns exact filter predicates. This pattern is production-proven (Trino, Uber, Netflix use it).

**OPA for field-level masking in GraphQL:** OPA can return a list of allowed/masked fields per user context. The GraphQL resolver layer (or an Envelop plugin) calls OPA synchronously, receives the field-access decision, and either nulls out PII fields or removes them from the schema projection. Apollo published a pattern for this using `@policy` directives + co-processor (but that requires Enterprise Router). With Hive Gateway + Envelop, the same pattern is implementable in a free plugin.

**OpenFGA for resource ownership checks:** OpenFGA (CNCF project, Auth0-backed) is the better fit for the "can user U read data product D?" coarse-grained check — checking relationships like team membership, product ownership, dataset group. It is NOT the right tool for "filter the rows returned to only show rows where region = user.region" — that is attribute-based, not relationship-based.

**Pattern recommendation — layered approach:**

```
Request → Gateway (JWT validation) → Federation Router
  → Envelop Plugin calls OPA for field-access decision
  → Subgraph resolver receives row-filter SQL fragments from OPA
  → Data source applies WHERE clause from OPA partial evaluation
```

Resource-ownership (coarse) checks can be delegated to OpenFGA and surfaced as claims in the JWT at token-issuance time, to avoid a synchronous OpenFGA lookup on every request.

### Recommendation

**Use OPA (Open Policy Agent) as the primary FGAC engine.**

Rationale:
1. Only engine that natively supports partial evaluation → SQL row filter push-down. This is essential for performance at data-product scale (you cannot fetch 10M rows and filter in memory).
2. CNCF-governed — institutional continuity despite Styra's acquisition by Apple.
3. Battle-tested in data platform contexts (Trino, Envoy, Kubernetes admission).
4. Integrates with GraphQL via Envelop plugin or sidecar HTTP call.
5. Rego policy-as-code integrates cleanly with GitOps CI/CD review workflows.

**Use OpenFGA alongside OPA for coarse-grained resource ownership checks** (which data products a user/team can access at all) to keep OPA policies simpler and avoid attribute explosion.

**Do not use Casbin** — embedding policy logic inside each subgraph service defeats the goal of centralized audit consistency.

---

## API Gateway

### The Landscape

Three OSS gateways evaluated for K8s-native, cloud-neutral, GraphQL-aware deployment.

| Gateway | Language | GraphQL Support | K8s Native | OSS Feature Completeness | Performance |
|---------|----------|----------------|------------|--------------------------|-------------|
| **Tyk OSS** | Go | Native REST+GraphQL+gRPC | Tyk Operator (CRD-based); Helm chart; Ingress controller | HIGH — full OSS, no feature gating | Outperforms Kong on all 3 clouds in Tyk's benchmarks; linear scaling |
| **Kong OSS** | Lua/Nginx | Plugin-based (GraphQL Rate Limiting, GraphQL Proxy Cache, DeGraphQL) | Kong Ingress Controller; Helm | MEDIUM — mTLS, OIDC, advanced metrics locked to Enterprise | Does not scale linearly past 8 cores per benchmarks |
| **Traefik Proxy** | Go | Layer 7 proxy only; no GraphQL-aware plugins | Excellent K8s native; single binary | LOW — minimal API management features | Excellent ingress but not an API gateway |

### Key Findings

**Kong OSS limitations:** Several features critical for a data API gateway are Enterprise-only in Kong: OIDC, mTLS (limited in OSS), advanced rate limiting, API analytics. The OSS version requires third-party tooling for a control panel. The GraphQL-specific plugins (GraphQL Rate Limiting Advanced, GraphQL Proxy Caching Advanced) are also listed as Enterprise plugins in Kong's docs (https://docs.konghq.com/gateway/latest/kong-plugins/graphql/) — the "Advanced" suffix is a reliable indicator of Enterprise tier. This means you lose GraphQL-aware rate limiting and caching in the free tier.

**Tyk OSS advantages:**
- "Batteries included" — all features (rate limiting, OIDC, mTLS, logging, analytics dashboard) are in the OSS release
- Native GraphQL support including Universal Data Graph (UDG), which can compose REST/GraphQL/gRPC APIs into a single GraphQL API — aligns with data-product ingestion pattern
- Tyk Operator provides true GitOps-native K8s management via CRDs, declared as Kubernetes manifests
- Go-based — same language as much K8s tooling, easier to extend
- Linear scaling benchmark characteristic is important for cost-predictable auto-scaling in Kubernetes HPA

**Traefik as sole gateway:** Traefik is outstanding as a K8s ingress controller but is not an API management gateway. It lacks: rate limiting per consumer, API key/JWT issuance and management, GraphQL-aware routing, analytics, or policy enforcement. It belongs in the stack as a front-door ingress layer, not as the API gateway layer.

**Role clarification:** The architecture for this platform should have TWO layers:
1. **Traefik** (or cloud LB) as the external ingress / TLS termination layer
2. **Tyk OSS** as the API management gateway behind Traefik — handling auth token validation, rate limiting, request routing to the federation router

This is the standard K8s-native pattern: Ingress Controller → API Gateway → Service Mesh (optional) → Application.

### Recommendation

**Use Tyk OSS as the API gateway, with Traefik as the ingress controller.**

Rationale:
1. Tyk OSS delivers full feature set with no enterprise paywall — rate limiting, OIDC, mTLS, built-in analytics are all included.
2. Tyk Operator enables declarative GitOps management of API definitions as Kubernetes CRDs.
3. Native GraphQL support and Universal Data Graph align with data-product federation goals.
4. Go-based — consistent with the overall stack (Cosmo Router is Go, K8s tooling is Go) for easier debugging and contribution.
5. Linear performance scaling gives predictable K8s HPA behavior.

**Do not attempt to use Traefik as a full API gateway** — it will require building significant custom middleware or moving security logic into the application layer.

---

## Data Product Registration

### The Problem

Data product teams need a self-service workflow to register a new data source (a database table, view, or service) as a GraphQL subgraph and make it queryable through the federation supergraph. Three patterns exist.

### Pattern 1: Schema-First with CI/CD Schema Registry

Teams write a `.graphql` schema file defining their subgraph's types and queries. This schema is committed to version control, and a CI/CD pipeline:
1. Validates the subgraph schema against the current supergraph composition (schema check)
2. Registers the validated schema with the schema registry on merge
3. The federation gateway picks up the new composition without restart

Tools: **Hive schema registry** (MIT, self-hosted) or **Cosmo Control Plane** (Apache 2.0, self-hosted)

The Guild's Hive and Cosmo both support schema checks via CLI (`hive schema:check`, `cosmo federated-graph check`) as part of PR pipelines. This is the recommended pattern for a governance-oriented data platform.

### Pattern 2: Introspection-Based Auto-Registration

A sidecar or operator introspects the running subgraph service's GraphQL endpoint (`/__schema` or introspection query) and automatically registers the schema. This reduces manual work but:
- Requires subgraphs to be running before registration (chicken-and-egg in GitOps)
- Makes schema changes harder to review in PRs
- Does not enforce schema contracts

Suitable as a migration path for existing GraphQL services, not for greenfield data products.

### Pattern 3: Metadata Catalog Integration (DataHub / OpenMetadata)

DataHub (open source) models data products as first-class entities with domains, ownership, SLAs, and lineage. DataHub's GraphQL API allows creating and managing data product entities. However, DataHub is a **metadata catalog**, not a schema registry for API composition. The two serve different purposes:
- DataHub: "What data assets exist, who owns them, what is their lineage?"
- Hive/Cosmo registry: "What is the schema contract for this subgraph?"

They are complementary, not alternatives. A mature platform registers the subgraph schema in Hive/Cosmo AND creates a corresponding data product entity in DataHub with a link to the subgraph endpoint.

### Registration Workflow (Recommended)

```
Team creates data product:
  1. Author subgraph .graphql schema + resolvers in a Git repository
  2. PR triggers: hive schema:check (or cosmo check) — validates composition
  3. Merge to main: hive schema:publish — registers schema version in Hive registry
  4. Hive notifies federation gateway of new composition (polling or webhook)
  5. Gateway hot-reloads supergraph without downtime
  6. Team publishes data product metadata to DataHub (automated via CI or Hive webhook)
```

### Key Tools

| Tool | Role | License | Self-Hosted |
|------|------|---------|-------------|
| **GraphQL Hive** | Schema registry, schema checks, usage analytics | MIT | Yes (Helm chart) |
| **Cosmo Control Plane** | Schema registry + router config + analytics (bundled) | Apache 2.0 | Yes (Helm chart) |
| **DataHub** | Data product metadata catalog, lineage, ownership | Apache 2.0 | Yes |
| **Hive CLI / Cosmo CLI** | CI/CD schema publish/check commands | MIT / Apache 2.0 | N/A |

### Recommendation

**Use Hive schema registry (paired with Hive Gateway) for API schema registration, and DataHub for data product metadata and lineage.**

The two complement each other:
- Hive owns the API contract (schema composition, breaking change detection, field usage analytics)
- DataHub owns the data product discovery layer (who owns what, lineage, quality, domain hierarchy)

If you choose Cosmo Router instead of Hive Gateway, use the Cosmo Control Plane as the schema registry instead — do not mix Cosmo Router with Hive registry (they use different composition approaches).

---

## Observability

### Requirements Mapping

| Requirement | Signal Type | Tool |
|-------------|-------------|------|
| Query metrics (operation latency, error rate, field usage) | Metrics + Traces | OTel Collector → Prometheus/Mimir → Grafana |
| Access audit logs (who accessed what, when, which fields) | Structured Logs | OTel Collector → Loki → Grafana / SIEM |
| Cost attribution (per team, per data product, per query) | Metrics with labels | OTel resource attributes → Prometheus → Grafana |
| Distributed tracing (subgraph fan-out latency) | Traces | OTel → Tempo → Grafana |

### Layer-by-Layer Stack

#### Instrumentation Layer

**OpenTelemetry (OTel) Collector** is the mandatory foundation. It provides:
- Vendor-neutral instrumentation — no lock-in to Datadog, New Relic, etc.
- Single collection point: receives traces, metrics, and logs via OTLP
- Transformation pipeline: add `team`, `data_product`, `subgraph` resource attributes before forwarding
- DaemonSet deployment in K8s collects all pod logs and Kubelet metrics automatically

GraphQL-specific OTel instrumentation:
- **Envelop** (The Guild) provides a `@envelop/opentelemetry` plugin that creates spans for each GraphQL operation, resolver, and subgraph call — this is the recommended instrumentation path for Hive Gateway
- Apollo Router has native OTel support via its `telemetry.exporters.otlp` config — available in the free tier

#### Metrics and Dashboards

**Prometheus + Grafana** (or **Mimir** for long-term storage in multi-cluster) is the standard K8s-native metrics stack.

Key GraphQL metrics to track:
- `graphql_operation_latency_seconds` (by operation name, subgraph)
- `graphql_field_execution_count` (for cost attribution per field)
- `graphql_errors_total` (by error type, subgraph)
- `graphql_query_cost` (from OPA/cost analysis middleware)

**Cost attribution via OTel resource attributes:** Tag every metric and trace with `team_id`, `data_product_id`, `consumer_id` as OTel resource attributes. Prometheus resource attribute promotion (added in 2025, confirmed in Grafana Labs docs) allows these to become label dimensions in PromQL. Grafana dashboards can then aggregate cost by team using `sum by (team_id)` on query cost metrics. This enables internal chargeback without a commercial metering product.

#### Distributed Tracing

**Grafana Tempo** (OSS) for trace storage and Grafana for trace visualization. Apollo Federation / Hive Gateway generates multi-span traces that show the full lifecycle of a federated query: gateway → subgraph A → subgraph B → database. This is critical for diagnosing cross-subgraph latency issues.

#### Audit Logging (Compliance)

This is the most complex requirement. Audit logs differ from application logs:
- **Immutable** — must not be mutable or deletable by the application
- **Structured** — machine-parseable JSON with well-defined fields
- **Complete** — capture: user identity, operation name, field list accessed, row filters applied, timestamp, source IP, response code
- **Forwarded to SIEM** — most compliance regimes require logs in an external, tamper-evident store

Recommended pattern:

```
Hive Gateway (or Tyk) emits structured JSON audit events
  → OTel Collector (log receiver)
  → Two outputs:
    a. Loki (for operational search in Grafana — short retention)
    b. Kafka topic (for SIEM ingestion — compliance long retention)
  → SIEM subscribes to Kafka topic (Splunk / Elastic Security / OpenSearch)
```

**Why Kafka in the middle:** Kafka provides durability and decoupling. If the SIEM goes down, audit events are not lost. Modern SIEMs (Splunk, Elastic) can directly consume from Kafka topics. This is the pattern described in Confluent's real-time compliance logging guide.

**What to log per GraphQL request:**
```json
{
  "timestamp": "2026-03-23T10:00:00Z",
  "user_id": "u-123",
  "team_id": "t-456",
  "operation_name": "GetCustomerOrders",
  "operation_type": "query",
  "fields_accessed": ["customer.id", "customer.email", "order.total"],
  "fields_masked": ["customer.ssn", "customer.dob"],
  "data_products": ["dp-customers", "dp-orders"],
  "row_filters_applied": {"customer": "region='APAC'"},
  "duration_ms": 45,
  "source_ip": "10.0.0.1",
  "status": "success"
}
```

OPA decisions (field access + row filters) should be included in the audit log — the FGAC engine and audit logging must be connected.

#### Cost Attribution

Two complementary mechanisms:

1. **Query cost scoring:** At the federation gateway, instrument every operation with a cost score (field count × depth × estimated row count). Use The Guild's `@envelop/depth-limit` and a custom cost plugin, or IBM's GraphQL Cost Spec (`@cost` directive). Emit cost as an OTel metric.

2. **Resource attribution labels:** Every OTel span and metric carries `consumer_team_id` and `data_product_id` labels. Prometheus aggregates these into cost-per-team dashboards. For chargeback, export weekly Prometheus query results (sum of query costs per team) via a Grafana report or to a billing system.

### Recommended Observability Stack

| Layer | Tool | License | Notes |
|-------|------|---------|-------|
| Instrumentation | OpenTelemetry Collector | Apache 2.0 | DaemonSet on every node |
| GraphQL tracing | Envelop OTel plugin | MIT | Per-resolver spans |
| Metrics store | Prometheus + Mimir (long-term) | Apache 2.0 | K8s-native |
| Dashboards | Grafana OSS | AGPL 3.0 | Ops + cost dashboards |
| Distributed traces | Grafana Tempo | Apache 2.0 | Federated query waterfall |
| Log aggregation | Grafana Loki | AGPL 3.0 | Operational log search |
| Audit pipeline | Apache Kafka | Apache 2.0 | Durable audit event bus |
| SIEM | OpenSearch Security / Elastic Security | Apache 2.0 / ELv2 | SIEM analytics |

The LGTM stack (Loki + Grafana + Tempo + Mimir/Prometheus) is the standard K8s-native, fully self-hostable observability platform. It avoids vendor lock-in to Datadog or New Relic while providing equivalent functionality.

---

## Recommended Stack Summary

| Component | Recommended Choice | Runner-up | Key Reason |
|-----------|-------------------|-----------|------------|
| **Federation Gateway** | Hive Gateway (The Guild) | Cosmo Router (WunderGraph) | 100% federation compliance, MIT license, 6x throughput vs Apollo Router, no FGAC enterprise paywall |
| **Schema Registry** | GraphQL Hive (self-hosted) | Cosmo Control Plane | MIT license, CI/CD schema checks, field usage analytics; pairs natively with Hive Gateway |
| **Composition Library** | `@graphql-hive/federation` (MIT) | — | Replaces Apollo's ELv2-licensed `@apollo/composition`; same API, MIT licensed |
| **FGAC Engine (Row Filtering)** | OPA (Open Policy Agent) | — | Only engine supporting partial evaluation → SQL WHERE clause push-down; CNCF governed |
| **FGAC Engine (Resource Ownership)** | OpenFGA | SpiceDB | Zanzibar ReBAC model for "can team T access data product D"; 10x check speedup in 2025 |
| **Field Masking Layer** | Envelop plugin (custom, OPA-backed) | — | Schema-directive approach; declarative PII annotations in subgraph schema |
| **API Gateway** | Tyk OSS | Kong OSS | Full feature set in OSS tier (no enterprise paywall); linear scaling; GitOps via Tyk Operator CRDs |
| **K8s Ingress** | Traefik (ingress controller role only) | AWS ALB (v1) | Cloud-neutral, single binary, native K8s |
| **Data Product Catalog** | DataHub OSS | OpenMetadata | Domain ownership, lineage, data product entities; Apache 2.0 |
| **Metrics** | Prometheus + Grafana Mimir | VictoriaMetrics | Standard K8s stack; OTel resource attribute promotion for cost attribution |
| **Dashboards** | Grafana OSS | — | De facto standard; integrates all LGTM components |
| **Distributed Traces** | Grafana Tempo | Jaeger | Apache 2.0; native Grafana integration; federation query waterfall visualization |
| **Log Aggregation** | Grafana Loki | — | K8s-native label-based indexing; low storage cost |
| **Audit Event Bus** | Apache Kafka | Redpanda | Durable, immutable audit log pipeline; SIEM decoupling |
| **Instrumentation SDK** | OpenTelemetry Collector | — | Vendor-neutral; mandatory for long-term observability strategy |
| **IaC / Deployment** | Terraform + Helm + ArgoCD | Flux CD | AWS provider for v1; Helm charts for all components above; GitOps |

### License Posture Summary

All recommended components are MIT, Apache 2.0, or AGPL 3.0 licensed — no ELv2, BSL, or proprietary licenses in the critical path. Apollo's ELv2 (Router + `@apollo/composition`) is explicitly avoided by using Hive Gateway + The Guild's MIT composition library.

---

## Sources

- [Grafbase: Benchmarking GraphQL Federation Gateways - September 2025](https://grafbase.com/blog/benchmarking-graphql-federation-gateways)
- [The Guild: Federation Gateway Audit (Hive)](https://the-guild.dev/graphql/hive/federation-gateway-audit)
- [Apollo Docs: Licensed GraphOS Router Features (@authenticated, @policy)](https://www.apollographql.com/docs/graphos/reference/router/graphos-features)
- [Apollo Docs: Moving Apollo Federation 2 to ELv2](https://www.apollographql.com/blog/announcement/moving-apollo-federation-2-to-the-elastic-license-v2/)
- [WunderGraph Cosmo GitHub (Apache 2.0)](https://github.com/wundergraph/cosmo)
- [WunderGraph: eBay-backed Series A for OSS Federation (TechCrunch, March 2025)](https://techcrunch.com/2025/03/27/ebay-backs-wundergraph-to-build-an-open-source-graphql-federation/)
- [Open Policy Agent: Row-Level Security Pattern](https://hoop.dev/blog/row-level-security-at-scale-with-open-policy-agent/)
- [OPA + GraphQL: Apollo Blog pattern](https://www.apollographql.com/blog/centrally-enforce-policy-as-code-for-graphql-apis)
- [Permit.io: OPA vs Cedar vs Zanzibar 2025](https://www.permit.io/blog/policy-engines)
- [OSO: OPA vs Cedar vs Zanzibar guide](https://www.osohq.com/learn/opa-vs-cedar-vs-zanzibar)
- [Cloud Native Now: Apple buys Styra brains, OPA remains open](https://cloudnativenow.com/editorial-calendar/best-of-2025/apple-buys-styra-brains-opa-remains-open-2/)
- [OpenFGA: Fine-Grained News September 2025 (10x check speedup)](https://openfga.dev/blog/fine-grained-news-2025-09)
- [API7.ai: API Gateway Comparison (Kong, Tyk, Traefik, KrakenD, APISIX)](https://api7.ai/learning-center/api-gateway-guide/api-gateway-comparison-apisix-kong-traefik-krakend-tyk)
- [APIdog: Tyk vs Kong 2025](https://apidog.com/blog/tyk-vs-kong/)
- [Tyk: Kubernetes Native with Tyk Operator](https://tyk.io/docs/api-management/automations/operator)
- [Kong Docs: GraphQL Plugins (Enterprise tier)](https://docs.konghq.com/gateway/latest/kong-plugins/graphql/)
- [DataHub Docs: Data Product Entity](https://docs.datahub.com/docs/generated/metamodel/entities/dataproduct)
- [Confluent: Real-Time Compliance and Audit Logging with Kafka](https://www.confluent.io/blog/build-real-time-compliance-audit-logging-kafka/)
- [CNCF: Cost-effective observability platform with OpenTelemetry (2025)](https://www.cncf.io/blog/2025/12/16/how-to-build-a-cost-effective-observability-platform-with-opentelemetry/)
- [Grafana Labs: OpenTelemetry with Prometheus resource attribute promotion (2025)](https://grafana.com/blog/2025/05/20/opentelemetry-with-prometheus-better-integration-through-resource-attribute-promotion/)
- [Hasura: GraphQL and the data mesh (pattern reference)](https://hasura.io/blog/graphql-and-the-data-mesh-developer-productivity-in-an-age-of-exploding-data)
- [IBM GraphQL Cost Spec](https://ibm.github.io/graphql-specs/cost-spec.html)
