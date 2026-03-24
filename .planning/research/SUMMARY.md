# Research Summary: Data Products API Platform

**Project:** Data Products API Platform
**Domain:** GraphQL Federation Supergraph with Fine-Grained Access Control, Self-Hosted on Kubernetes
**Researched:** 2026-03-23
**Confidence:** MEDIUM-HIGH

---

## Executive Summary

This platform is a self-hosted, GraphQL-first API layer that lets data product teams publish their data sources as independently governed subgraph services, compose them into a unified supergraph, and expose them to consumers with fine-grained access control at both the field and row level. The canonical pattern — well documented in Apollo GraphOS, Google Cloud data mesh architecture guides, and The Guild's federation tooling — is a 4-hop request model: external consumer to API gateway (auth, rate limiting) to federation router (query planning, field-level policy) to data product subgraph (resource-level checks, data fetch). Research strongly supports Hive Gateway plus Hive schema registry as the OSS-first federation layer, OPA as the policy engine for row-filter push-down and field masking, Tyk OSS as the API gateway, and the LGTM observability stack (Grafana, Loki, Tempo, Mimir/Prometheus via OpenTelemetry).

The most important architectural choice is centralizing policy enforcement. Every major pitfall identified — PII leaks, entitlement cache drift, silent field bypass via fragment traversal — traces back to FGAC logic scattered across individual subgraphs instead of evaluated in one place and threaded as context. This must be a day-one decision because retrofitting centralized authorization across 50+ live subgraphs is a multi-quarter project. Equally non-negotiable in v1: PII field classification as a schema directive applied at registration time (not retrofitted later), audit logging wired to Kafka from the start (compliance requirements activate before scale does), and full supergraph composition validation at schema registration (not just SDL syntax checking).

The top risks are operational rather than technical: composition failures becoming a platform team bottleneck, schema graveyard from unretired data products, ownership ambiguity when teams reorganize, and PII leaking into distributed traces. All four are solvable with upfront process design — ownership model, TTL-based product health checks, trace scrubbing pipelines — that must be built before the first production product is registered, not added later.

---

## Recommended Stack

| Component | Choice | License | Key Reason |
|-----------|--------|---------|------------|
| Federation Gateway | Hive Gateway (The Guild) | MIT | 100% federation spec compliance; 6x throughput vs Apollo Router; no enterprise paywall for FGAC directives |
| Schema Registry | GraphQL Hive (self-hosted) | MIT | CI/CD schema checks, field usage analytics, CDN serving; pairs natively with Hive Gateway |
| Composition Library | `@graphql-hive/federation` | MIT | Drop-in replacement for Apollo's ELv2-licensed `@apollo/composition`; same API, no vendor lock-in |
| FGAC Engine (Row Filtering) | OPA (Open Policy Agent) | Apache 2.0 | Only engine with partial evaluation for SQL WHERE clause push-down; CNCF governed |
| FGAC Engine (Ownership Checks) | OpenFGA | Apache 2.0 | Zanzibar ReBAC for "can team T access data product D"; keeps OPA policies simpler |
| Field Masking | Envelop plugin (OPA-backed) | MIT | Schema-directive PII annotations; no enterprise license required |
| Policy Distribution | OPAL (Permit.io) | Apache 2.0 | Real-time push of updated Rego bundles to OPA sidecars without restarts |
| API Gateway | Tyk OSS | MPL 2.0 | Full feature set in OSS tier (rate limiting, OIDC, mTLS, analytics); linear K8s scaling; GitOps via CRDs |
| K8s Ingress | Traefik | MIT | Cloud-neutral ingress / TLS termination; do NOT use as the API management layer |
| Data Product Catalog | DataHub OSS | Apache 2.0 | Domain ownership, lineage, data product entities; complements (not replaces) Hive schema registry |
| Metrics | Prometheus + Grafana Mimir | Apache 2.0 | K8s-native; OTel resource attribute promotion enables cost attribution per team |
| Dashboards | Grafana OSS | AGPL 3.0 | De facto standard; integrates all LGTM components |
| Distributed Traces | Grafana Tempo | Apache 2.0 | Apache 2.0; native Grafana integration; federated query waterfall visualization |
| Log Aggregation | Grafana Loki | AGPL 3.0 | K8s-native label-based indexing; low storage cost |
| Audit Event Bus | Apache Kafka | Apache 2.0 | Durable, decoupled audit pipeline; SIEM survives downtime without losing events |
| OTel Instrumentation | OpenTelemetry Collector | Apache 2.0 | Vendor-neutral; mandatory foundation for cost attribution and trace correlation |
| IaC | Terraform + Helm + ArgoCD | MPL 2.0 / Apache 2.0 | Terraform for cluster lifecycle; Helm/ArgoCD for workloads; GitOps-native |

**License posture:** All chosen components are MIT, Apache 2.0, AGPL 3.0, or MPL 2.0. Apollo's ELv2-licensed Router and `@apollo/composition` are deliberately avoided. No proprietary or BSL components in the critical path.

---

## Architecture Overview

The platform follows a layered, namespace-segregated Kubernetes topology. External consumers hit Traefik (ingress / TLS) then Tyk OSS (auth, rate limiting, complexity gating). Tyk routes validated requests to the Hive Gateway inside the `graph-platform` namespace. The gateway plans the query against the composed supergraph schema, calls the FGAC coprocessor (OPA) to resolve field access decisions and row-filter predicates, then fans out to data product subgraphs in isolated `data-product-*` namespaces. Subgraphs are not directly accessible from outside the cluster — NetworkPolicy enforces ingress only from `graph-platform`. The Hive schema registry serves the composed supergraph schema via an internal CDN endpoint; the gateway hot-reloads it on a 10-second poll without restarts. The LGTM observability stack (OTel Collector DaemonSet + Gateway deployment, Tempo, Loki, Prometheus/Mimir, Grafana) runs in a dedicated `observability` namespace. All audit events flow via Kafka to both Loki (operational search) and a SIEM (compliance long-term retention).

```
[Consumer]
    |
    v
[Traefik Ingress] — TLS termination, cloud-neutral
    |
    v
[Tyk OSS API Gateway]  ns: api-gateway
    - JWT validation, OIDC
    - Query complexity + rate limiting
    - Injects identity headers
    |
    v
[Hive Gateway / Supergraph Router]  ns: graph-platform
    - Parses GraphQL operation
    - Calls FGAC Coprocessor (OPA)
      -> receives: field access list, row filter predicates
    - Generates query plan; fans out to subgraphs
    |
    +---> [Data Product Subgraph A]  ns: data-product-orders
    |         -> PostgreSQL
    +---> [Data Product Subgraph B]  ns: data-product-customers
    |         -> MongoDB
    +---> [Data Product Subgraph N]  ns: data-product-*
              -> (any data source)

[Hive Schema Registry]  ns: graph-platform
    - Validates composition on schema:publish
    - Serves supergraph SDL via internal CDN
    - Breaking-change detection in CI

[DataHub]  ns: graph-platform (or separate)
    - Data product discovery catalog
    - Domain ownership, lineage, SLA metadata

[LGTM Stack]  ns: observability
    OTel Collector (DaemonSet + Gateway)
      -> Grafana Tempo (traces)
      -> Prometheus/Mimir (metrics)
      -> Grafana Loki (logs)
      -> Grafana (dashboards)

[Kafka]  ns: observability
    - Audit event bus
    - Consumed by SIEM and Loki

[GitOps Layer]
    data-product-<team> repo -> hive schema:check (CI) -> hive schema:publish (merge)
                              -> PR to platform-gitops repo -> ArgoCD deploys subgraph
    terraform/ -> Terraform -> EKS / GKE / AKS cluster lifecycle
    platform-gitops/ -> ArgoCD -> all Kubernetes workloads
```

---

## Critical Design Decisions (Decide Before Building)

These decisions are expensive to undo after the first data products are registered. Resolve them before Phase 1 begins.

1. **Centralized vs distributed FGAC enforcement.** All authorization decisions must flow through a single OPA policy engine. Do not allow subgraphs to implement their own directive-based authorization. Decide the coprocessor deployment pattern (OPA sidecar per router pod vs standalone OPA service) and enforce it as a platform constraint before any subgraph is registered.

2. **PII field classification as a mandatory schema directive.** Define `@pii` (or equivalent) as a first-class SDL directive before the first schema is registered. Make it a CI-required annotation — the schema:check step rejects schemas with sensitive-looking field names that lack the directive. Retrofitting this across 100 registered products later is measured in months.

3. **Apollo ELv2 vs MIT/Apache licensing posture.** Confirm the decision to use Hive Gateway and `@graphql-hive/federation` instead of Apollo Router and `@apollo/composition`. This has downstream effects on every schema check command, CI pipeline, and router configuration. Switching later requires migrating all existing CI integrations.

4. **Kafka-backed audit pipeline from day one.** Audit logging requirements (GDPR Article 30, SOC 2 CC6.1) activate as soon as the first real consumer accesses data, not at some future "compliance phase." The audit event schema and Kafka pipeline must exist before production launch. Design the event schema once — field additions are cheap, field renames after a SIEM has indexed the data are not.

5. **Terraform state segmentation.** Separate Terraform workspaces (networking, cluster, gateway, registry, data product infra) must be designed before any infrastructure is provisioned. A monolithic state file cannot be split without destroying and re-importing resources. Decide the workspace layout and remote state backend (S3 + DynamoDB for AWS) as the first IaC action.

6. **Subgraph DataLoader batching as a registration requirement, not a best practice.** Decide whether subgraphs that do not implement batched entity resolution are refused registration or just flagged. The right answer is refusal. This policy is easier to enforce before the first team onboards than after 20 teams have deployed non-batched subgraphs.

7. **Multi-cloud target set for v1.** Research shows Terraform module structure, load balancer behavior, and observability backend storage all differ by cloud. Decide which cloud(s) are in scope for v1 before writing any Terraform. Each additional cloud in v1 multiplies IaC complexity and operational surface area.

8. **Ownership model for the platform itself.** Before launch: who owns composition failure remediation? Who owns gateway incidents? What is the escalation path when a data product owner is unreachable? These must be defined in a platform operations runbook before the first product goes live.

---

## v1 Must-Haves (Non-Negotiable)

Research identifies these as genuinely required before any consumer goes live — including items that feel like v2 but are not.

- **Data product YAML descriptor** with required fields: `id`, `domain`, `version`, `display_name`, `description`, `owners`, `sensitivity_classification`, `pii_fields`, output port schema. No registration without complete metadata.
- **GraphQL subgraph schema registration via CLI** with full supergraph composition validation at submission time (not just SDL syntax check). Schema is rejected if the resulting supergraph fails to compose.
- **OAuth 2.0 / OIDC consumer authentication** with JWT validation at the gateway. Required JWT claims defined: `sub`, `client_id`, `scope`, plus custom `team`, `pii_clearance`, `tenant_id` claims from the IdP.
- **Coarse-grained entitlement** (allow/deny per consumer per data product) via OPA. This is the minimum viable FGAC — without it there is no tenant separation.
- **Field-level PII masking via OPA + Envelop plugin.** GDPR compliance requires this from the first query against a PII-bearing data product. Cannot defer.
- **Audit logging** with minimum required field set (actor identity, operation name, fields accessed, fields masked, policy decision, row filter applied) flowing to Kafka. Must be live before production.
- **Static query complexity scoring + per-consumer quota enforcement.** Protects platform from unbounded queries before the first external consumer is onboarded. Implement `@cost` and `@listSize` directives in subgraph schemas at registration.
- **Discovery catalog** (DataHub) with full-text search and schema browser. Without discovery, self-service is impossible — consumers cannot find products and will revert to asking owners directly.
- **`@pii` schema directive** applied to all sensitive fields at registration time, enforced by CI linting.
- **Subgraph health monitoring** and NetworkPolicy enforcement (subgraphs only accessible from `graph-platform` namespace).
- **Schema rollback tooling** — last N versions of every subgraph schema retained as rollback targets. One-command rollback required before any schema is promoted to production.
- **Operational ownership runbook** documented before the first production product is registered.

---

## v2 Deferrals (Safe to Defer)

These add maturity but do not block a governed v1 launch.

- **Row-level filtering via OPA SQL predicate injection.** Coarse-grained allow/deny is sufficient for v1 if data products do not yet require per-row tenant isolation. Row-level filtering is architecturally compatible with the v1 OPA design — it extends it rather than replacing it.
- **OPAL real-time policy distribution.** In v1, OPA policy bundle updates can be applied via rolling pod restart. OPAL eliminates restart downtime but is not critical until policy change frequency warrants it.
- **GitOps-based product registration with fully automated CI validation pipeline.** A platform-team-assisted onboarding workflow with manual CI steps is sufficient for v1. Full automation comes in v2 when product count makes manual onboarding a bottleneck.
- **Consumer access request and approval workflow.** In v1, access grants can be managed manually by the platform team. The self-service marketplace pattern is a v2 feature.
- **Per-consumer dynamic cost attribution and chargeback reports.** Static complexity scoring is in v1. The financial reporting layer (aggregated cost records, chargeback exports) is v2.
- **Schema change proposals / governed change management UI.** PR-based review on the data-products Git repository provides equivalent governance in v1.
- **Data contract registration** (DPDS promises, obligations, SLA). Metadata fields exist in the YAML descriptor but formal ODCS/DPDS contract enforcement is v2.
- **Persisted operation allowlists.** Hardens security for external consumers but is not required for a controlled internal v1 launch.
- **Federated cross-cloud observability** (single Grafana spanning multiple cloud clusters). Run per-cloud LGTM stacks in v1; federate in v2 when multi-cloud operations mature.
- **Low-code policy rule builder UI.** Platform engineers can author Rego directly in v1. The non-engineer-facing rule builder UI is v2/v3.
- **Data lineage graph in catalog.** High value but complex to build — defer until core schema and ownership mechanics are solid.

---

## Top Risks to Mitigate

These five pitfalls have the highest probability of causing production incidents or compliance failures. Design against them from day one.

1. **PII leaking into distributed traces and audit logs.** OpenTelemetry instrumentation is comprehensive by design — without explicit scrubbing, GraphQL query variables, resolver error messages, and response fragments containing PII appear in Grafana Tempo and Loki where any developer with trace access can read them. Mitigation: implement a trace attribute blocklist (variable names, field paths) in the OTel Collector pipeline before any trace is exported. Never log raw query variables in production. Classify trace stores as sensitive and access-control them identically to the data products.

2. **Distributed FGAC logic producing silent PII exposure.** When field masking is implemented via GraphQL directives applied independently by each subgraph team, policy drift is inevitable — renamed fields lose their directives, new fields are added without them, alternate traversal paths via fragment spreads bypass enforcement. Mitigation: centralize all FGAC decisions in the OPA coprocessor; subgraphs receive and apply the pre-evaluated masking list rather than evaluating policy themselves. Automated traversal tests enumerate all schema paths to `@pii`-tagged fields and assert masking — run in CI on every schema change.

3. **N+1 network round-trips at federation boundaries destroying latency.** In a monolith, N+1 is slow database calls. In federation, N+1 is N separate HTTP round-trips from the router to a subgraph, making it catastrophically worse at realistic list sizes (100 entity references = 100 HTTP calls). Mitigation: mandate DataLoader-based batched entity resolution as a hard registration requirement. Validate it in subgraph acceptance tests (send a batch entity request, assert one upstream call, not N). Reject subgraphs that fail this test.

4. **Schema composition failures blocking all teams simultaneously.** Two teams making concurrent valid schema changes can compose to an invalid supergraph. The resulting broken composition blocks every team from deploying schema changes until manually resolved. Mitigation: run composition checks against a `staging` variant that tracks all teams' main branches; implement a schema change queue to serialize concurrent publishes; define a "composition owner" role with a 4-hour SLA to unblock failures.

5. **Entitlement cache lag granting access after revocation.** Entitlement decisions cached at the gateway or policy engine continue granting access for up to 15 minutes after a revocation event (employee termination, role change). Mitigation: implement event-driven cache invalidation — publish revocation events that the policy engine subscribes to and immediately flushes affected cache entries. For high-sensitivity data products, use 60-second TTLs. Maintain a revocation token blocklist checked before cache for immediate enforcement requirements.

---

## Implications for Roadmap

### Phase 1: Foundation Infrastructure
**Rationale:** Nothing else can be built until the cluster topology, IaC, and core platform services (gateway, router, schema registry, observability) exist. This phase creates the skeleton every subsequent phase deploys into.
**Delivers:** Running Kubernetes cluster (EKS for v1), Traefik ingress, Tyk OSS gateway, Hive Gateway, Hive schema registry with one test subgraph, OTel + LGTM observability stack, Terraform state layout, ArgoCD GitOps.
**Addresses:** Terraform state segmentation (critical design decision 5), multi-cloud target scoping (decision 7).
**Avoids:** Monolithic Terraform state (PITFALLS: multi-cloud section), cloud-specific networking lock-in.
**Research flag:** Standard patterns, well-documented — skip deep research phase for this phase.

### Phase 2: Security and Access Control Core
**Rationale:** Must be fully functional before any real consumer or real data product is onboarded. FGAC, auth, and audit logging are prerequisites for production use, not afterthoughts. This is the phase most likely to be underestimated.
**Delivers:** OAuth 2.0 / OIDC integration with JWT validation at Tyk; OPA coprocessor wired to Hive Gateway; coarse-grained entitlement (allow/deny per consumer per product); field-level PII masking via Envelop plugin; `@pii` directive enforcement in schema CI; Kafka-backed audit log pipeline with full required event schema; introspection disabled for unauthenticated requests.
**Addresses:** Centralized FGAC enforcement (critical design decision 1), PII directive (decision 2), audit pipeline (decision 4).
**Avoids:** Distributed FGAC drift (top risk 2), PII in traces (top risk 1), entitlement cache lag (top risk 5).
**Research flag:** Needs careful implementation attention — OPA partial evaluation for row filtering, Envelop plugin construction, and Kafka audit schema are the highest-complexity elements. Consider a `/gsd:research-phase` on the OPA-to-SQL predicate push-down specifically.

### Phase 3: Data Product Registration and Catalog
**Rationale:** With security foundations in place, enable the first real data product teams to self-register. This phase builds the self-service mechanics that justify the platform's existence.
**Delivers:** Data product YAML descriptor schema and validation; GitOps registration workflow (subgraph repo -> hive schema:check CI -> schema:publish on merge -> PR to platform-gitops -> ArgoCD deploys); DataHub catalog with full-text search and schema browser; schema rollback tooling; subgraph health monitoring; composition failure alerting; platform operations runbook.
**Addresses:** DataLoader batching requirement (critical design decision 6), schema rollback, ownership model (decision 8).
**Avoids:** Schema graveyard (PITFALLS: self-service section), composition blocking all teams (top risk 4), semantic type name collisions.
**Research flag:** Standard patterns with a wrinkle — the DataHub integration with the Hive registry (cross-system metadata sync on schema publish) has limited documented reference implementations. Flag for research phase.

### Phase 4: Consumer Onboarding and Cost Management
**Rationale:** Platform is production-ready after Phase 3. Phase 4 adds the consumer-facing features that make it a marketplace rather than just a registry.
**Delivers:** Consumer developer portal (credential issuance, product discovery, access request UI); static query complexity scoring (`@cost` / `@listSize` directives); per-consumer quota enforcement (complexity budget, HTTP 429 with Retry-After); `apollographql-client-name` enforcement at gateway; cost attribution OTel labels; Grafana cost dashboards per consumer team.
**Addresses:** Static cost attribution and quota (v1 feature from FEATURES.md research).
**Avoids:** Unbounded queries before external consumer onboarding, runaway query cost without attribution.
**Research flag:** Consumer portal UX (access request workflow, credential issuance) has no standardized implementation pattern — likely needs a design spike.

### Phase 5: Row-Level Security and Policy Maturity
**Rationale:** Coarse-grained entitlement from Phase 2 is sufficient for products with uniform access. Phase 5 enables multi-tenant products where individual rows must be filtered by consumer identity.
**Delivers:** OPA partial evaluation for SQL WHERE clause push-down; OPAL real-time policy distribution (eliminating OPA restart for policy updates); per-product TTL-based policy cache with classification-tiered TTLs; automated access-control regression tests in CI.
**Addresses:** Row-level security requirement (v2 from FEATURES.md).
**Avoids:** Row filter destroying query performance (index on entitlement columns, materialized views for columnar stores), entitlement cache lag for revocation.
**Research flag:** OPA partial evaluation for SQL generation and integration with diverse subgraph data sources (Postgres, MongoDB, columnar warehouses) is the most technically complex feature in the roadmap. Strongly recommend a `/gsd:research-phase` here.

### Phase 6: Platform Governance and Operational Maturity
**Rationale:** By this phase, product count is growing. Phase 6 adds the governance tooling that prevents the platform from degrading under growth.
**Delivers:** Automated product health checks and TTL-based ownership attestation; ownership gap detection (inactive owner alerts); schema change proposals for governed breaking changes; per-product usage analytics (query volume, distinct consumers); data product health scoring in catalog; supergraph SDL size monitoring with domain-split triggers; dynamic cost attribution and chargeback reports (v2 from FEATURES.md).
**Avoids:** Schema graveyard, data product sprawl degrading discoverability (PITFALLS: operational section), ownership ambiguity on team reorganization.
**Research flag:** Standard governance patterns — skip research phase.

### Phase Ordering Rationale

- Phases 1-2 are strictly sequential and must complete before any production data is exposed: infrastructure before security, security before data.
- Phase 3 (registration) can begin in parallel with Phase 2's later stages once the schema registry is operational.
- Phases 4-6 are sequenced by dependency: consumer onboarding (Phase 4) requires registration (Phase 3); row-level security (Phase 5) extends but does not replace the FGAC foundation from Phase 2; governance (Phase 6) operates on the product population built in Phases 3-5.
- The ordering specifically avoids the most expensive retrofits: centralizing FGAC before any subgraphs register (Phase 2), PII directive enforcement before any schema is submitted (Phase 2/3 boundary), and Terraform state segmentation before any infrastructure exists (Phase 1).

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Core recommendations (Hive Gateway, OPA, Tyk, LGTM) verified against official docs, independent benchmarks (Grafbase Sep 2025), and confirmed CNCF governance. One caveat: OPA maintainer situation (Apple hiring Styra team) warrants monitoring over next 12 months. |
| Features | MEDIUM-HIGH | v1 must-haves and GDPR/SOC 2 audit requirements verified via official compliance specs and multiple production reference implementations. Cost attribution chargeback specifics are MEDIUM confidence — limited public production reference implementations. |
| Architecture | MEDIUM-HIGH | 4-hop model, namespace topology, and GitOps registration flow verified against Apollo, Hive, and HashiCorp official docs. ARCHITECTURE.md contains a minor inconsistency: it recommends Kong in the request flow diagram but STACK.md recommends Tyk based on fuller analysis. Tyk is the correct choice (see Critical Design Decisions). |
| Pitfalls | HIGH | Most pitfalls sourced from official post-mortems, OWASP GraphQL cheat sheet, and production war stories. The N+1 federation problem and composition blocking pitfalls are extensively documented. |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **DataHub + Hive schema registry integration:** The workflow for syncing schema publish events to DataHub entities (step 6 in the registration flow) has no canonical implementation guide. Needs a design spike or prototype in Phase 3 planning.
- **OPA partial evaluation + heterogeneous data sources:** OPA's SQL push-down pattern is well documented for Postgres. Behavior for MongoDB, Snowflake, and other columnar stores used by data product teams is less documented. Validate data source coverage before committing to row-level security scope in Phase 5.
- **Tyk OSS operational maturity at scale:** Tyk's linear-scaling benchmark is based on Tyk's own published benchmarks (not fully independent). Validate under realistic GraphQL federation traffic patterns (not just REST throughput) early in Phase 1.
- **Multi-cloud Terraform behavioral differences:** Load balancer annotations, health check behavior, and OIDC integration vary between EKS, GKE, and AKS. If more than one cloud is in scope for v1, budget a Terraform integration test sprint for each additional cloud before the Phase 1 milestone.
- **Dynamic cost attribution accuracy:** Static complexity scoring has known limitations (cheap queries against cached subgraphs, expensive queries with low schema-complexity scores). The gap between estimated and actual cost will need calibration against real consumer query patterns in Phase 4/5.

---

## Open Questions for the User

These questions are not answerable from research alone and must be resolved before or during planning.

1. **Which cloud(s) are in scope for v1?** Research recommends AWS (EKS) for v1 due to the most mature Terraform modules and the clearest path for Hive, Tyk, and OTel deployment. Each additional cloud adds significant IaC scope. What is the v1 cloud target?

2. **What is the existing IdP?** The platform requires OAuth 2.0 / OIDC for consumer authentication. Does the organization already have Okta, Auth0, Azure AD, or Keycloak? The JWT claim structure (how `team`, `pii_clearance`, `tenant_id` are added to tokens) depends entirely on which IdP is in use.

3. **What are the compliance obligations for the v1 data products?** GDPR, HIPAA, SOC 2, and PCI DSS impose different audit log retention requirements (90 days to 7 years), different PII handling rules, and different access control evidence requirements. Which regulations are in scope determines what "production-ready audit logging" actually means.

4. **How many data product teams are expected in v1 vs 12 months post-launch?** This determines whether the GitOps registration workflow needs full automation in v1 or whether platform-team-assisted onboarding suffices. It also sets the query planning performance baseline for Hive Gateway sizing.

5. **Are there existing data products (tables, APIs, services) to onboard, or is this greenfield?** Migration of existing services requires the introspection-based auto-registration path (Pattern 2 from STACK.md) alongside the GitOps path. Greenfield allows schema-first only.

6. **What is the target consumer population — internal teams, external partners, or both?** External consumers require a developer portal, credential issuance, persisted operation security hardening, and potentially a separate IdP realm. Internal-only v1 is significantly simpler.

7. **Does the organization have an existing data catalog (Collibra, Alation, internal tool)?** DataHub is the recommended choice, but if a catalog already exists, the integration question changes from "deploy DataHub" to "integrate with existing catalog via DataHub's REST/GraphQL API."

8. **What is the row-level security requirement for v1?** Is coarse-grained allow/deny-per-product sufficient for v1 launch, or do any v1 data products require per-consumer row filtering (multi-tenant isolation)? This determines whether OPA SQL push-down is in Phase 2 or can safely wait for Phase 5.

---

## Sources

### Primary (HIGH confidence)
- [The Guild: Federation Gateway Audit](https://the-guild.dev/graphql/hive/federation-gateway-audit) — federation spec compliance scores
- [Grafbase: Benchmarking GraphQL Federation Gateways, Sep 2025](https://grafbase.com/blog/benchmarking-graphql-federation-gateways) — throughput and latency benchmarks
- [Apollo Docs: Licensed GraphOS Router Features](https://www.apollographql.com/docs/graphos/reference/router/graphos-features) — ELv2 enterprise-only FGAC directives confirmed
- [GraphQL Hive Self-Hosted](https://the-guild.dev/graphql/hive/docs/self-hosting/get-started) — schema registry deployment
- [Apollo Router External Coprocessor](https://www.apollographql.com/docs/graphos/routing/customization/coprocessor) — FGAC coprocessor pattern
- [OPA: Row-Level Security Pattern](https://hoop.dev/blog/row-level-security-at-scale-with-open-policy-agent/) — partial evaluation for SQL push-down
- [OPAL Documentation](https://docs.opal.ac/) — real-time policy distribution
- [Open Data Mesh DPDS v1.0](https://dpds.opendatamesh.org/specifications/dpds/1.0.0/) — data product descriptor spec
- [Confluent: Real-Time Compliance & Audit Logging with Kafka](https://www.confluent.io/blog/build-real-time-compliance-audit-logging-kafka/) — audit pipeline pattern
- [IBM GraphQL Cost Directives Specification](https://ibm.github.io/graphql-specs/cost-spec.html) — `@cost` and `@listSize` directives
- [Shopify: Rate Limiting GraphQL APIs by Calculating Query Complexity](https://shopify.engineering/rate-limiting-graphql-apis-calculating-query-complexity) — production complexity model
- [HashiCorp: Terraform Multi-Cloud Kubernetes](https://developer.hashicorp.com/terraform/tutorials/networking/multicloud-kubernetes) — IaC patterns
- [CNCF: OPA Best Practices for Secure Deployment, 2025](https://www.cncf.io/blog/2025/03/18/open-policy-agent-best-practices-for-a-secure-deployment/) — OPA governance status post-Styra
- [DataHub Docs: Data Product Entity](https://docs.datahub.com/docs/generated/metamodel/entities/dataproduct) — catalog metadata model
- [GraphQL OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html) — security pitfalls

### Secondary (MEDIUM confidence)
- [WunderGraph: eBay-backed Series A for Cosmo OSS Federation, Mar 2025](https://techcrunch.com/2025/03/27/ebay-backs-wundergraph-to-build-an-open-source-graphql-federation/) — Cosmo viability as runner-up
- [Cloud Native Now: Apple buys Styra brains, OPA remains open, 2025](https://cloudnativenow.com/editorial-calendar/best-of-2025/apple-buys-styra-brains-opa-remains-open-2/) — OPA maintainer situation
- [OpenFGA: Fine-Grained News Sep 2025 (10x check speedup)](https://openfga.dev/blog/fine-grained-news-2025-09) — OpenFGA performance
- [Grafana Labs: OTel with Prometheus resource attribute promotion, 2025](https://grafana.com/blog/2025/05/20/opentelemetry-with-prometheus-better-integration-through-resource-attribute-promotion/) — cost attribution via OTel labels
- [API7.ai: API Gateway Comparison 2025](https://api7.ai/learning-center/api-gateway-guide/api-gateway-comparison-apisix-kong-traefik-krakend-tyk) — Tyk vs Kong
- [Thoughtworks: State of Data Mesh 2026](https://www.thoughtworks.com/insights/blog/data-strategy/the-state-of-data-mesh-in-2026-from-hype-to-hard-won-maturity) — industry context

---

*Research completed: 2026-03-23*
*Ready for roadmap: yes*
