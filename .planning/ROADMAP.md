# Roadmap: Data Products API Platform

## Overview

Six phases take the platform from zero infrastructure to a production-governed data product marketplace. Phases 1-2 build the skeleton and secure it before any real data is exposed. Phase 3 enables the first teams to self-register products. Phase 4 adds consumer-facing quota and cost visibility. Phases 5-6 extend into row-level security and long-term governance — positioned as v2 work after the v1 core is operating in production.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation Infrastructure** - Running EKS cluster with Tyk, Hive Gateway, schema registry, LGTM observability, and ArgoCD GitOps wired end-to-end
- [ ] **Phase 2: Security and Access Control** - OAuth2/OIDC auth, OPA coprocessor, field-level PII masking, Kafka audit pipeline, and introspection lockdown before any production data is exposed
- [ ] **Phase 3: Data Product Registration and Catalog** - Self-service YAML descriptor registration, composition-validated subgraph onboarding, DataHub catalog, schema rollback, and operations runbook
- [ ] **Phase 4: Consumer Onboarding and Cost Management** - Static complexity scoring, per-consumer quota enforcement, cost attribution labels, and Grafana cost dashboards
- [ ] **Phase 5: Row-Level Security and Policy Maturity** - OPA SQL predicate push-down, OPAL real-time policy distribution, classification-tiered cache TTLs, and automated RLS regression tests (v2)
- [ ] **Phase 6: Platform Governance and Operational Maturity** - Ownership attestation, schema change governance, product health scoring, usage analytics, and dynamic chargeback reports (v2)

## Phase Details

### Phase 1: Foundation Infrastructure
**Goal**: The full platform skeleton — EKS cluster, Tyk gateway, Hive Gateway with schema registry, and LGTM observability — is running via ArgoCD GitOps with a test subgraph composing successfully.
**Depends on**: Nothing (first phase)
**Requirements**: INFRA-01, INFRA-02, INFRA-03, INFRA-04, INFRA-05, INFRA-06, INFRA-07, INFRA-08, OBS-01, OBS-02, OBS-03, OBS-04
**Success Criteria** (what must be TRUE):
  1. A test GraphQL query traverses the full 4-hop path: Traefik ingress → Tyk gateway → Hive Gateway → test subgraph, and returns a response with a distributed trace visible in Grafana Tempo
  2. Terraform apply from a clean state provisions the full EKS cluster and platform workloads; state is segmented into separate workspaces (networking, cluster, gateway, registry, observability) with no cross-workspace dependencies
  3. All Kubernetes workloads are managed by ArgoCD; a change pushed to the platform-gitops repo deploys within 5 minutes without manual intervention
  4. Grafana dashboards show query latency, error rate, and throughput per data product; Loki receives structured logs with `data_product_id` and `consumer_team` labels
**Plans**: 4 plans

Plans:
- [ ] 01-01: Terraform EKS cluster with segmented state workspaces and ArgoCD bootstrap
- [ ] 01-02: Tyk OSS gateway and Traefik ingress deployment via ArgoCD
- [ ] 01-03: Hive Gateway, Hive schema registry, and test subgraph composition
- [ ] 01-04: OTel Collector DaemonSet plus LGTM stack (Tempo, Loki, Mimir, Grafana) with baseline dashboards
**UI hint**: yes

### Phase 2: Security and Access Control
**Goal**: Every request is authenticated via OAuth2/OIDC, field-level PII masking is enforced by a centralized OPA coprocessor, and every data access event is durably written to Kafka — all in place before any production data product is registered.
**Depends on**: Phase 1
**Requirements**: FGAC-01, FGAC-02, FGAC-03, FGAC-04, FGAC-05, FGAC-06, FGAC-07, AUDIT-01, AUDIT-02, AUDIT-03, AUDIT-04, OBS-05, OBS-06
**Success Criteria** (what must be TRUE):
  1. An unauthenticated request to the gateway receives HTTP 401; a request with a valid JWT containing required claims (`sub`, `client_id`, `scope`, `team`, `pii_clearance`, `tenant_id`) reaches the supergraph router
  2. A query requesting a `@pii`-tagged field from a consumer without `pii_clearance` returns the field as null or masked, and the OPA policy decision log records the masking event
  3. Every data access produces an audit record in Kafka containing actor identity, operation name, fields accessed, fields masked, and policy decision — with no raw query variables or response body data present
  4. A schema submitted to the registry with a PII-bearing field name lacking the `@pii` directive is rejected by the CI schema:check step before merge
**Plans**: 4 plans

Plans:
- [ ] 02-01: OAuth2/OIDC integration at Tyk gateway with JWT claim validation and introspection lockdown
- [ ] 02-02: OPA coprocessor deployment with coarse-grained entitlement policies and OpenFGA ownership model
- [ ] 02-03: Envelop field-masking plugin with `@pii` directive enforcement and CI schema:check integration
- [ ] 02-04: Kafka audit pipeline with full event schema, Loki consumer, and OTel trace PII scrubbing

### Phase 3: Data Product Registration and Catalog
**Goal**: A data product team can register a validated subgraph via YAML descriptor and GitOps PR, see it appear in the DataHub catalog, and roll back a schema in one command — with health monitoring and a composition failure runbook in place before the first production product goes live.
**Depends on**: Phase 2
**Requirements**: REG-01, REG-02, REG-03, REG-04, REG-05, REG-06, CAT-01, CAT-02, CAT-03, HEALTH-01, HEALTH-02
**Success Criteria** (what must be TRUE):
  1. Submitting a valid YAML descriptor plus subgraph SDL triggers full supergraph composition validation; a schema that fails composition is rejected with a clear error before any deployment occurs
  2. A registered data product appears in DataHub with domain, owners, sensitivity classification, and schema populated automatically via Hive webhook — no manual catalog entry required
  3. A schema rollback to any of the last 10 versions completes with a single command and the supergraph reflects the rolled-back schema within 30 seconds
  4. A composition failure triggers an alert to the designated composition owner within 4 hours; the operations runbook is published and accessible to all platform users
**Plans**: 4 plans

Plans:
- [ ] 03-01: YAML descriptor schema validation, `@pii` field enforcement, and DataLoader batching acceptance test
- [ ] 03-02: GitOps registration workflow — hive schema:check CI, schema:publish on merge, ArgoCD subgraph deploy
- [ ] 03-03: DataHub OSS deployment with Hive webhook integration for automatic catalog entity creation
- [ ] 03-04: Schema rollback tooling, composition failure alerting, subgraph health monitoring, and operations runbook
**UI hint**: yes

### Phase 4: Consumer Onboarding and Cost Management
**Goal**: Every consumer operates under a static complexity budget enforced at the gateway, query costs are attributed to teams via OTel labels, and a Grafana dashboard makes per-consumer consumption visible.
**Depends on**: Phase 3
**Requirements**: CONS-01, CONS-02, CONS-03
**Success Criteria** (what must be TRUE):
  1. A query exceeding a consumer's complexity quota returns HTTP 429 with a Retry-After header; the `apollographql-client-name` header is required for all operations and enforced at the Tyk gateway
  2. Grafana shows query volume and complexity consumption broken down by consumer team, with `team_id` and `data_product_id` as metric dimensions sourced from OTel resource attribute promotion
**Plans**: 3 plans

Plans:
- [ ] 04-01: `@cost` and `@listSize` directive implementation in subgraph schemas with static complexity scoring
- [ ] 04-02: Per-consumer quota enforcement at Tyk with `apollographql-client-name` header validation and HTTP 429 responses
- [ ] 04-03: Cost attribution OTel labels and Grafana cost dashboards per consumer team
**UI hint**: yes

### Phase 5: Row-Level Security and Policy Maturity
**Goal**: Consumers querying multi-tenant data products see only the rows matching their entitlement scope, OPA policy updates propagate without pod restarts, and automated CI tests prevent RLS regressions.
**Depends on**: Phase 2
**Requirements**: RLS-01, RLS-02, RLS-03, RLS-04
**Success Criteria** (what must be TRUE):
  1. A consumer querying a multi-tenant data product receives only rows matching their `tenant_id` claim; a query with a mismatched tenant ID returns zero rows, not an error
  2. An OPA policy bundle update propagates to all running OPA instances via OPAL within 30 seconds, with no pod restarts required
  3. The CI pipeline runs automated row-level security regression tests on every schema or policy change and fails the build on any access control regression
**Plans**: 3 plans

Plans:
- [ ] 05-01: OPA partial evaluation for SQL WHERE clause push-down with per-subgraph data source adapters
- [ ] 05-02: OPAL real-time policy distribution and classification-tiered cache TTL enforcement
- [ ] 05-03: Automated RLS regression test suite integrated into CI

### Phase 6: Platform Governance and Operational Maturity
**Goal**: Product owners attest ownership on a TTL schedule, schema changes follow a governed approval workflow, and per-product usage analytics and health scores are visible in the catalog.
**Depends on**: Phase 5
**Requirements**: GOV-01, GOV-02, GOV-03, GOV-04, GOV-05
**Success Criteria** (what must be TRUE):
  1. A data product with no owner attestation within the configured TTL enters a degraded state visible in DataHub, and the registered owners receive an automated alert
  2. A proposed breaking schema change triggers a governed review workflow; the change cannot merge without explicit approval from the designated schema governance role
  3. DataHub displays a composite health score per data product (freshness, query volume, owner responsiveness) and per-consumer usage analytics including query volume and distinct consumers
**Plans**: 3 plans

Plans:
- [ ] 06-01: TTL-based ownership attestation with degraded-state triggering and owner notification
- [ ] 06-02: Schema change governance workflow for breaking changes with approval gating
- [ ] 06-03: Product health scoring, usage analytics per product, and dynamic cost attribution chargeback reports
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation Infrastructure | 0/4 | Not started | - |
| 2. Security and Access Control | 0/4 | Not started | - |
| 3. Data Product Registration and Catalog | 0/4 | Not started | - |
| 4. Consumer Onboarding and Cost Management | 0/3 | Not started | - |
| 5. Row-Level Security and Policy Maturity | 0/3 | Not started | - |
| 6. Platform Governance and Operational Maturity | 0/3 | Not started | - |
