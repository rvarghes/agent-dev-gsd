# Requirements: Data Products API Platform

**Defined:** 2026-03-23
**Core Value:** Any authorized consumer can query any data product — or compose multiple products — through a single GraphQL endpoint, with PII protection enforced transparently via entitlements.

---

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Infrastructure (INFRA)

- [ ] **INFRA-01**: Platform deploys to AWS EKS via Terraform with state segmented by layer (networking, cluster, gateway, registry, observability)
- [ ] **INFRA-02**: Terraform modules are structured for multi-cloud portability (GCP/Azure without rearchitecting — cloud-specific logic isolated in leaf modules)
- [ ] **INFRA-03**: All Kubernetes workloads managed via ArgoCD GitOps from a `platform-gitops` repository
- [ ] **INFRA-04**: Traefik handles TLS termination and ingress — no cloud-vendor ingress controllers in the critical path
- [ ] **INFRA-05**: Tyk OSS API gateway handles JWT validation, OIDC integration, rate limiting, and query complexity gating
- [ ] **INFRA-06**: Hive Gateway serves as the GraphQL supergraph router with OPA coprocessor wired for FGAC
- [ ] **INFRA-07**: Hive schema registry (self-hosted) validates schema composition and serves the supergraph SDL via internal CDN
- [ ] **INFRA-08**: Data product subgraphs run in isolated `data-product-*` Kubernetes namespaces accessible only from `graph-platform` namespace via NetworkPolicy

### Access Control (FGAC)

- [ ] **FGAC-01**: All consumers authenticate via OAuth 2.0 / OIDC with JWT validation at the gateway; required claims: `sub`, `client_id`, `scope`, `team`, `pii_clearance`, `tenant_id`
- [ ] **FGAC-02**: Coarse-grained entitlement (allow/deny per consumer per data product) enforced via OPA — no tenant cross-contamination
- [ ] **FGAC-03**: Field-level PII masking enforced via OPA coprocessor + Envelop plugin — centralized policy evaluation, not per-subgraph directive logic
- [ ] **FGAC-04**: `@pii` SDL directive applied to all sensitive fields at registration time; CI schema-check rejects schemas with untagged PII-bearing field names
- [ ] **FGAC-05**: GraphQL introspection disabled for unauthenticated requests
- [ ] **FGAC-06**: OpenFGA manages coarse resource ownership (which teams can access which data products)
- [ ] **FGAC-07**: Entitlement cache TTL is 60 seconds for high-sensitivity products; event-driven invalidation on revocation

### Audit & Compliance (AUDIT)

- [ ] **AUDIT-01**: Every data access event produces an audit log record with: actor identity, client_id, operation name, operation hash, fields accessed, fields masked, policy decision, policy bundle version, row filter applied, row count returned
- [ ] **AUDIT-02**: Audit events flow to Kafka before any other destination — no audit events lost during downstream outages
- [ ] **AUDIT-03**: Audit logs consumed by Grafana Loki (operational search) and forwarded to SIEM for compliance retention
- [ ] **AUDIT-04**: No raw query variables or response body data in trace or log records (GDPR data minimisation)

### Observability (OBS)

- [ ] **OBS-01**: OpenTelemetry Collector deployed as DaemonSet + Gateway; all platform components instrument via OTLP
- [ ] **OBS-02**: Query latency, error rates, and throughput tracked per data product and per consumer in Grafana dashboards
- [ ] **OBS-03**: Distributed traces in Grafana Tempo with per-subgraph span waterfall for latency attribution
- [ ] **OBS-04**: Log aggregation in Grafana Loki with structured labels: `data_product_id`, `consumer_team`, `operation_name`
- [ ] **OBS-05**: OTel trace attribute blocklist scrubs PII-bearing field paths and query variables before export
- [ ] **OBS-06**: Cost attribution labels (`team_id`, `data_product_id`) on all OTel metrics via Prometheus resource attribute promotion

### Data Product Registration (REG)

- [ ] **REG-01**: Data product owners register a product by submitting a YAML descriptor (required fields: `id`, `domain`, `version`, `display_name`, `description`, `owners`, `sensitivity_classification`, `pii_fields`) plus a GraphQL SDL subgraph schema
- [ ] **REG-02**: Schema registration triggers full supergraph composition validation — rejected if the resulting supergraph fails to compose, not just SDL syntax-valid
- [ ] **REG-03**: Subgraphs must implement batched entity resolution (DataLoader pattern); acceptance test validates one upstream call per batch — registration rejected if not implemented
- [ ] **REG-04**: Last 10 versions of every subgraph schema retained; one-command rollback to any version available
- [ ] **REG-05**: Schema composition failure triggers alert to a designated composition owner with 4-hour SLA
- [ ] **REG-06**: Breaking schema changes detected by `hive schema:check` in CI and blocked from merging without explicit override approval

### Discovery & Catalog (CAT)

- [ ] **CAT-01**: DataHub OSS deployed with full-text search and schema browser for all registered data products
- [ ] **CAT-02**: DataHub data product entities created/updated automatically on schema publish via Hive webhook
- [ ] **CAT-03**: Each data product catalogued with: domain, owners, sensitivity classification, schema, and SLA metadata

### Consumer Experience (CONS)

- [ ] **CONS-01**: Static query complexity enforced via `@cost` and `@listSize` directives in subgraph schemas; queries exceeding consumer quota return HTTP 429 with Retry-After
- [ ] **CONS-02**: Per-consumer query quota enforced at Tyk gateway; `apollographql-client-name` header required for all operations
- [ ] **CONS-03**: Basic Grafana cost attribution dashboard showing query volume and complexity consumption per consumer team

### Subgraph Health (HEALTH)

- [ ] **HEALTH-01**: Subgraph health endpoints monitored; unhealthy subgraphs surface in Grafana with automatic alert
- [ ] **HEALTH-02**: Platform operations runbook documented before first production product registration: composition failure escalation, gateway incident response, subgraph owner escalation path

---

## v2 Requirements

Deferred to future release. Architecturally compatible with v1 — extends rather than replaces.

### Row-Level Security (RLS)

- **RLS-01**: Row-level filtering via OPA SQL predicate push-down — consumer sees only rows matching their entitlement scope
- **RLS-02**: OPAL real-time policy distribution — OPA policy updates without pod restarts
- **RLS-03**: Classification-tiered OPA policy cache TTLs (public: 5min, internal: 2min, restricted: 60s, highly-restricted: event-driven)
- **RLS-04**: Automated row-level security regression tests in CI

### Self-Service Automation (SSA)

- **SSA-01**: Fully automated GitOps product registration pipeline: PR → hive schema:check CI → auto-merge → platform-gitops PR → ArgoCD deploy — no platform team intervention required
- **SSA-02**: Consumer self-service access request and approval workflow (marketplace model)
- **SSA-03**: Persisted operation allowlists for hardened external consumer API access

### Cost & Governance (GOV)

- **GOV-01**: Dynamic cost attribution — actual query execution cost tracked per consumer/team for financial chargeback
- **GOV-02**: TTL-based ownership attestation — product owners confirm ownership periodically; unconfirmed products enter degraded state
- **GOV-03**: Schema change governance UI — governed change proposal workflow for breaking changes
- **GOV-04**: Data lineage graph in DataHub catalog
- **GOV-05**: Product health scoring in catalog — composite score from freshness, query volume, owner responsiveness

### Platform Scale (SCALE)

- **SCALE-01**: Cross-cloud observability federation — single Grafana spanning multi-cloud LGTM stacks
- **SCALE-02**: Domain-level supergraph partitioning for platforms exceeding ~50 data products
- **SCALE-03**: Low-code Rego policy builder UI for non-engineer policy authors

---

## Out of Scope

| Feature | Reason |
|---------|--------|
| REST API | All v1 access patterns fit GraphQL; REST adds complexity for no gain — defer to v2+ |
| Real-time subscriptions / event streaming | v1 is query-only (tables/views); Kafka streaming exposure is a separate workstream |
| Mobile SDKs / client libraries | Consumers use standard GraphQL clients |
| Managed API gateways (AWS API GW, Apigee, Google Cloud API GW) | Cloud-vendor-specific; violates cloud-neutrality constraint |
| Apollo Router / Apollo GraphOS SaaS | ELv2 license; FGAC directives are enterprise-only; replaced by Hive Gateway (MIT) |
| `@apollo/composition` | ELv2 license; replaced by `@graphql-hive/federation` (MIT) |
| Data lineage tracking | High value, high complexity — defer to v2; core schema and ownership mechanics must be solid first |
| HIPAA / PCI compliance hardening | In-scope compliance requirements to be confirmed before Phase 2 — treated as out of scope until confirmed |

---

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFRA-01 | Phase 1 | Pending |
| INFRA-02 | Phase 1 | Pending |
| INFRA-03 | Phase 1 | Pending |
| INFRA-04 | Phase 1 | Pending |
| INFRA-05 | Phase 1 | Pending |
| INFRA-06 | Phase 1 | Pending |
| INFRA-07 | Phase 1 | Pending |
| INFRA-08 | Phase 1 | Pending |
| OBS-01 | Phase 1 | Pending |
| OBS-02 | Phase 1 | Pending |
| OBS-03 | Phase 1 | Pending |
| OBS-04 | Phase 1 | Pending |
| FGAC-01 | Phase 2 | Pending |
| FGAC-02 | Phase 2 | Pending |
| FGAC-03 | Phase 2 | Pending |
| FGAC-04 | Phase 2 | Pending |
| FGAC-05 | Phase 2 | Pending |
| FGAC-06 | Phase 2 | Pending |
| FGAC-07 | Phase 2 | Pending |
| AUDIT-01 | Phase 2 | Pending |
| AUDIT-02 | Phase 2 | Pending |
| AUDIT-03 | Phase 2 | Pending |
| AUDIT-04 | Phase 2 | Pending |
| OBS-05 | Phase 2 | Pending |
| OBS-06 | Phase 2 | Pending |
| REG-01 | Phase 3 | Pending |
| REG-02 | Phase 3 | Pending |
| REG-03 | Phase 3 | Pending |
| REG-04 | Phase 3 | Pending |
| REG-05 | Phase 3 | Pending |
| REG-06 | Phase 3 | Pending |
| CAT-01 | Phase 3 | Pending |
| CAT-02 | Phase 3 | Pending |
| CAT-03 | Phase 3 | Pending |
| HEALTH-01 | Phase 3 | Pending |
| HEALTH-02 | Phase 3 | Pending |
| CONS-01 | Phase 4 | Pending |
| CONS-02 | Phase 4 | Pending |
| CONS-03 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 36 total
- Mapped to phases: 36
- Unmapped: 0 ✓

---
*Requirements defined: 2026-03-23*
*Last updated: 2026-03-23 after initial definition*
