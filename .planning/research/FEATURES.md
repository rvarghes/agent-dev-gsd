# Features Research: Data Products API Platform

**Domain:** GraphQL-first unified federation supergraph for data product API management
**Researched:** 2026-03-23
**Overall confidence:** MEDIUM-HIGH (core patterns verified via official docs; cost attribution and chargeback are lower confidence due to limited production reference implementations)

---

## Data Product Registration & Discovery

### What a Data Product Owner Must Provide

Based on DataHub's YAML spec, the Open Data Mesh Data Product Descriptor Specification (DPDS), and the Open Data Contract Standard (ODCS), a mature registration artifact includes the following fields:

**Identity (required)**
- `id` — machine-readable unique identifier, e.g. `orders_v1`
- `domain` — owning business domain, e.g. `commerce`
- `version` — semantic version; major bump required when output ports change

**Discoverability (required)**
- `display_name` — human-readable name for the catalog UI
- `description` — prose description of what the product provides and why it exists
- `tags` — classification tags (for search/filter)
- `glossary_terms` — links to formal business vocabulary (e.g. "Order", "Customer")
- `owners` — list of owners with typed roles: `BUSINESS_OWNER`, `TECHNICAL_OWNER`, `DATA_STEWARD`

**Output Ports (required)**
Each output port is where the data surfaces. For a GraphQL supergraph platform, the primary output port is a subgraph schema. Per the DPDS spec, each port defines:
- `name` — unique port name within the product
- `schema` — the GraphQL SDL (subgraph schema), or a reference to a schema file
- `promises` — declared SLOs, freshness guarantees, availability commitments
- `expectations` — intended audience and approved consumer use cases
- `obligations` — terms of use, data sensitivity classification, billing policy if applicable

**Governance Metadata (optional but expected)**
- `sensitivity_classification` — e.g. `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `PII`
- `pii_fields` — explicit list of fields containing personally identifiable information
- `data_contract` — reference to an ODCS or DPDS data contract document
- `institutional_memory` — links to runbooks, wiki pages, Confluence docs
- `sla` — machine-readable SLA definition (uptime, freshness, latency targets)

### How Products Get Added to the Supergraph

Registration in a GraphQL federation context maps directly onto subgraph publication. The verified Apollo GraphOS pattern is:

1. Product owner defines a subgraph schema (GraphQL SDL) for their output port
2. Owner runs `rover subgraph publish --schema schema.graphql --name my-product` targeting the schema registry
3. The registry triggers automatic composition — it merges the new subgraph into the supergraph schema
4. Composition checks run: breaking-change detection, linting, federation directive validation
5. If composition passes, a new supergraph schema is distributed to the router fleet

For a data products platform, this flow should be extended: before `rover subgraph publish` is allowed to push to production, the product's metadata artifact (domain, owners, sensitivity classification, output port declarations) must be present and validated. This is a platform concern layered on top of schema registry mechanics.

**Confidence: HIGH** — verified via Apollo GraphOS docs, DataHub docs, Open Data Mesh DPDS spec.

### Discovery Catalog Requirements

Mature platforms (DataHub, Collibra, Databricks Marketplace, Immuta) converge on a discovery catalog that provides:
- Full-text search over product names, descriptions, glossary terms, and tags
- Faceted filters: domain, sensitivity, owner, SLA tier
- Schema browser: users can explore the output port's field-level schema before requesting access
- Sample data preview: optional, conditioned on sensitivity classification
- Lineage graph: upstream datasets feeding the product, downstream consumers
- Usage statistics: query count per day/week, distinct consumers, p99 latency
- Access request button: triggers entitlement workflow without leaving the catalog

---

## Entitlement & Policy Management

### ABAC over RBAC for Data Products

Simple RBAC (roles assigned to users, permissions assigned to roles) is insufficient for data products with field-level PII and row-level filtering requirements. ABAC (Attribute-Based Access Control) is the correct model because access decisions depend on combinations of:
- **User attributes**: team, clearance level, purpose of use, geography
- **Resource attributes**: data product sensitivity, domain, field PII tags
- **Environmental attributes**: time of request, request source (internal network vs external)

**Recommendation:** OPA (Open Policy Agent) with Rego policies is the de-facto standard for policy-as-code ABAC in cloud-native platforms. Styra DAS is the enterprise control plane for OPA with built-in row-level and column-level filtering support.

**Confidence: HIGH** — verified via OPA official docs, Styra documentation, CNCF blog.

### Policy-as-Code with OPA Rego

Policies for a data product are expressed as Rego bundles. A policy bundle for a product includes:
- **Entitlement rule**: `allow { user.team == product.allowed_teams[_] }` — coarse-grained access decision
- **Row filter rule**: returns a SQL WHERE clause fragment injected into the query execution context; e.g. `row_filter = "tenant_id = '%s'" % user.tenant_id`
- **Field mask rule**: returns a set of fields to redact; e.g. `masked_fields = {"email", "ssn"} if not user.has_pii_clearance`

The OPA decision response drives:
1. Whether the GraphQL query is allowed at all (entitlement)
2. What rows the resolver's backing SQL query sees (row predicate injection)
3. Which fields are replaced with `null` or a masked token in the GraphQL response

### OPAL for Real-Time Policy Distribution

Static OPA deployments require restart to pick up policy changes. OPAL (Open Policy Administration Layer, from Permit.io) solves this: it watches a Git repo (or policy store API) and pushes updated Rego bundles to all OPA sidecars over a WebSocket pub/sub channel. This is production-proven at Tesla, Walmart, the NBA. OPAL is the correct choice for real-time policy propagation without requiring OPA instance restarts.

**Confidence: HIGH** — verified via OPAL docs, Permit.io, OPA ecosystem page.

### Policy Authoring Interface

Policy authoring is the hardest UX problem. Three tiers of interface:

| Tier | Interface | Audience | Notes |
|------|-----------|----------|-------|
| Low-code | Rule builder UI (like Styra DAS or Permit.io) | Data stewards, product owners | Generates Rego from form inputs |
| Code-level | Rego editor with lint/test in IDE | Platform engineers | OPA Playground-style tooling |
| GitOps | PR-based policy review workflow | Regulated orgs | Policy changes require review + OPAL deploys on merge |

**Recommendation:** Ship a rule-builder UI for product owners (covering common patterns: team allowlist, PII field masking, row-level tenant filter). Support hand-authored Rego for platform engineers. All policies live in Git; OPAL deploys them.

### Policy Attachment to Data Products

Each data product's registration artifact includes a `policy` stanza referencing the Rego bundle or inline policy rules:

```yaml
policy:
  entitlement_bundle: "rego/products/orders_v1/entitlement.rego"
  row_filter_bundle: "rego/products/orders_v1/row_filter.rego"
  field_masks:
    - field: "customer.email"
      mask_type: "hash_sha256"
      clearance_required: "pii_tier_1"
    - field: "customer.phone"
      mask_type: "redact"
      clearance_required: "pii_tier_1"
```

This policy block is version-controlled alongside the product descriptor and deployed via OPAL on merge.

---

## Self-Service Onboarding

### What Mature Platforms Use

Evidence from Apollo GraphOS, Google Cloud Data Mesh architecture docs, and dbt Labs data mesh guide shows convergence on a GitOps-first approach with a CLI as the primary onboarding surface:

**GitOps + CLI (recommended)**: Product owner creates a product descriptor YAML, a subgraph schema SDL, and an OPA policy bundle. They open a PR against a `data-products/` repository. CI runs validation checks (schema composes, policy lints, metadata is complete). On merge, automation publishes the schema to the registry and deploys the policy via OPAL.

**UI-based onboarding**: DataHub supports a point-and-click "New Data Product" flow. Suitable for initial discovery/registration of existing tables, but insufficient for schema-governed GraphQL subgraphs. Use the UI for metadata enrichment (tags, descriptions, owners) after GitOps onboarding establishes the schema.

**Confidence: MEDIUM** — GitOps pattern verified via Apollo GraphOS docs and Google Cloud architecture docs; UI-based verified via DataHub docs.

### Recommended Onboarding Flow (v1)

```
1. Product owner runs: gsd-product init --domain commerce --name orders_v1
   → Scaffolds: product.yaml, schema.graphql, policy/entitlement.rego

2. Owner fills in: display_name, description, owners, sensitivity, output port schema

3. Owner opens PR to data-products/ repo
   → CI validates:
      a. Schema composes against current supergraph (rover subgraph check)
      b. Required metadata fields present
      c. Rego policy lints (opa fmt, opa check)
      d. No breaking changes vs prior version of product

4. Platform team reviews (or auto-approves if domain is pre-cleared)

5. PR merges:
   → GitHub Action: rover subgraph publish (registers schema)
   → OPAL detects policy bundle change, pushes to OPA sidecars
   → Catalog service picks up product.yaml, indexes into discovery catalog
```

### Apollo GraphOS Schema Proposals for Change Management

For schema changes to existing products, Apollo's Schema Proposals feature provides a governed change management flow: the owner drafts a proposal in Studio, automated composition checks run, designated reviewers must approve before the schema can be published. This prevents unreviewed schema changes from breaking downstream consumers.

**Recommendation:** Use Schema Proposals for change management after v1. In v1, the PR review on the data-products/ repo provides equivalent governance.

---

## Consumer Auth & Identity

### Token-Based Identity Flow

The standard pattern for external-facing data API platforms is OAuth 2.0 with OIDC for identity, with the JWT carrying consumer identity claims that flow through to the FGAC engine:

```
Consumer → (client_credentials or auth_code flow) → IdP (Auth0/Okta/Keycloak)
         ← JWT access token (sub, client_id, scope, custom claims)

Consumer → GraphQL request (Authorization: Bearer <jwt>) → Router
Router → validates JWT signature, extracts claims
Router → passes claims in request context to OPA
OPA → evaluates entitlement + row/field policies using claims
OPA → returns: allow/deny, row_filter_predicate, masked_fields
Router/Resolvers → enforce policy decision
```

**Required JWT claims for FGAC:**
- `sub` — consumer subject identity (user ID or service account ID)
- `client_id` — API client identifier for cost attribution
- `scope` — approved scopes (e.g. `read:orders`, `read:customers`)
- Custom claims (added by IdP): `team`, `department`, `pii_clearance`, `tenant_id`

**Confidence: HIGH** — verified via Apollo GraphQL auth docs, AWS AppSync docs, IBM API Connect docs.

### Internal vs External Consumers

| Consumer Type | Auth Method | Identity Claims | Notes |
|---------------|-------------|-----------------|-------|
| Internal service | Client credentials flow (machine-to-machine) | `client_id`, `team`, `service_name` | Short-lived tokens, high-trust network |
| Internal human (analyst) | Auth code + PKCE | `sub`, `team`, `pii_clearance` | Via internal IdP (Okta/Active Directory) |
| External partner | Client credentials | `client_id`, `partner_id`, `tier` | Separate IdP realm or tenant |
| External developer | Auth code + PKCE | `sub`, `external_org_id` | Developer portal issues credentials |

### Apollo Router Client Awareness Headers

For internal consumers using the Apollo Router, consumer identity for metrics and cost attribution can be passed via:
- `apollographql-client-name` — identifies the consuming application
- `apollographql-client-version` — version of the consumer client

These are automatically extracted and included in metrics sent to GraphOS Studio, enabling per-client operation analytics. For cost attribution, these headers should carry the `client_id` or consuming service name, mapped to a billing/cost center record.

**Recommendation:** Require all consumers to set `apollographql-client-name` and `apollographql-client-version`. Enforce this at the router with a coprocessor that rejects requests missing these headers from registered consumers.

### Persisted Operations for External Consumers

External consumer security can be hardened using persisted operations (also called operation allowlists). Consumers register approved GraphQL operations; the platform only executes operations whose hash is in the allowlist. This prevents arbitrary query construction by external parties and reduces attack surface for data exfiltration through deeply nested queries.

**Confidence: MEDIUM** — verified via Apollo GraphQL docs and The Guild's Envelop docs.

---

## Cost Attribution

### The Problem

GraphQL's flexible query structure means two consumers submitting structurally different queries against the same product can generate radically different backend load. A simple "request count" attribution model is wrong. Cost attribution must account for query complexity.

### Query Complexity Scoring

Two complementary methods, both are needed:

**Static cost analysis (pre-execution)**
Traverse the query AST before execution. Assign weights per field type:
- Scalar / enum: 0 points (live inside parent objects)
- Object: 1 point
- Connection / list: 2 points + (page_size × child_cost)
- Mutation: 10 points (side effects are more expensive)

This is Shopify's production model, well-documented and peer-reviewed. The IBM GraphQL Cost Directives specification formalises this with `@cost(weight: "N")` and `@listSize(assumedSize: N, slicingArguments: ["first", "last"])` SDL directives that schema owners annotate on their fields.

**Dynamic cost analysis (post-execution)**
Measure actual execution time, resolver round-trips, and data volume returned. This is more accurate but requires instrumentation at the resolver level. The gap between static (estimated) cost and dynamic (actual) cost can be used for refunds (Shopify's model) or to update cost weights.

**Confidence: HIGH (static analysis)** — verified via Shopify engineering blog, IBM GraphQL cost spec, multiple open-source implementations.
**Confidence: MEDIUM (dynamic analysis)** — pattern described in academic literature and Shopify docs; specific implementation varies.

### Per-Consumer Attribution Model

Attribution workflow:

1. **Consumer identified** via JWT `client_id` claim + `apollographql-client-name` header
2. **Static cost computed** at router ingress before execution (reject if over per-consumer quota)
3. **Execution traced** via OpenTelemetry spans; actual execution time recorded per consumer
4. **Cost record emitted** to a time-series store (e.g. ClickHouse, Prometheus, or a custom events table): `{ timestamp, client_id, operation_name, operation_hash, static_cost, actual_cost_ms, data_product_id, subgraph_id }`
5. **Cost aggregated** daily/monthly per `client_id`, per `data_product_id`
6. **Chargeback report** generated: internal teams receive cost-center allocation reports; external consumers receive usage invoices

### GraphOS Client Segmentation for Attribution

Apollo GraphOS Studio segments operation metrics by `apollographql-client-name`. This provides:
- Operations per client per time period
- Field usage breakdown per client
- Error rate per client
- (Enterprise) per-client-version breakdowns

This is useful for capacity planning and quota management. For financial chargeback, you need the raw cost records emitted in step 4 above — GraphOS does not provide financial cost attribution natively. Build a lightweight cost accounting service on top of the OpenTelemetry trace data.

### Rate Limiting and Quota Enforcement

Based on the static cost model:
- Each consumer is assigned a quota: e.g. 50,000 complexity points per hour
- The router checks the consumer's rolling budget before executing the query
- If budget is exceeded, the router returns HTTP 429 with a `Retry-After` header and the reset time
- Quota tiers (Bronze/Silver/Gold) map to different point budgets and correspond to entitlement tiers in the product catalog

**Recommendation:** Implement `@cost` and `@listSize` directives in the schema from day one. Static cost gating is a v1 feature. Dynamic cost attribution for chargeback is a v2 feature.

---

## Audit Logging

### What a Compliant Audit Log Must Capture

For GDPR (Article 30 records of processing) and SOC 2 (CC6.1, CC7.2 — logical access and monitoring), the audit log must answer: who accessed what, when, from where, and what was returned.

**Minimum required fields per event:**

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | UUID | Globally unique event identifier |
| `timestamp` | ISO 8601 UTC | When the event occurred, millisecond precision |
| `actor.subject` | string | JWT `sub` claim — user or service account identity |
| `actor.client_id` | string | OAuth2 `client_id` — consuming application |
| `actor.ip_address` | string | Source IP (or CIDR if behind NAT) |
| `actor.user_agent` | string | HTTP User-Agent of the consumer |
| `action.type` | enum | `QUERY`, `MUTATION`, `INTROSPECTION_REQUEST`, `ACCESS_DENIED` |
| `action.operation_name` | string | Named GraphQL operation, if provided |
| `action.operation_hash` | string | SHA-256 of the normalized query document |
| `resource.data_product_id` | string | Which data product was accessed |
| `resource.fields_requested` | string[] | Top-level fields requested (not full query — avoids logging PII in query text) |
| `resource.fields_masked` | string[] | Fields masked by PII policy (confirms masking happened) |
| `resource.row_filter_applied` | string | The predicate that was injected (confirms RLS happened) |
| `policy.decision` | enum | `ALLOW`, `DENY`, `PARTIAL_ALLOW` |
| `policy.bundle_version` | string | Git commit hash of the OPA policy bundle that made the decision |
| `response.status` | enum | `SUCCESS`, `ERROR`, `PARTIAL` |
| `response.row_count` | integer | Number of records returned (not content) |
| `response.latency_ms` | integer | Total query execution time in ms |
| `session.request_id` | string | Correlation ID for distributed tracing (ties to OpenTelemetry trace) |

**Do NOT log:**
- Full query response body (contains actual data — logging it creates secondary data exposure risk under GDPR)
- Query variable values that contain PII (log variable names only, or omit)
- Full query document text if it can contain parameterized PII (log the hash instead)

**Confidence: HIGH** — verified via DreamFactory audit log guide, GDPR Article 30, SOC 2 CC6.1, Cloudflare audit log schema, Splunk structured audit trail docs.

### GDPR-Specific Requirements

- **Purpose limitation**: log entry should record the stated purpose/scope (from JWT `scope` claim)
- **Data subject access requests**: the log must support querying "what data about person X was accessed by whom, when" — this requires `actor.subject` and the `resource.fields_requested` to be indexed
- **Right to erasure**: audit logs themselves are records of processing; GDPR requires only "what's necessary" — typical recommended retention is 90 days to 1 year
- **Field-level audit**: when a PII field was requested but masked, the log should record that masking occurred — this is evidence of technical measures under GDPR Article 25 (privacy by design)

### SOC 2 Requirements

- **CC6.1** (Logical and Physical Access Controls): must log all authentication events, access grants, and access denials
- **CC7.2** (System Monitoring): must demonstrate that access to data is monitored continuously; the audit log feeds this evidence
- **Retention**: SOC 2 auditors typically require 12 months of audit log retention
- **Tamper evidence**: logs must be immutable once written; use append-only storage (S3 object lock, Kafka with retention policy, WORM-compliant storage)

### Log Pipeline Architecture

```
GraphQL Router / Resolver Layer
  → emits structured audit events (JSON, AsyncAPI schema)
  → Kafka topic: data-access-audit-events

Kafka consumer
  → writes to immutable audit log store (e.g. ClickHouse, S3 + Glue, or OpenSearch with ILM)
  → writes to SIEM (Splunk, Datadog, or similar) for real-time alerting

Audit Query API
  → internal compliance team queries by actor, resource, time range
  → supports DSAR (Data Subject Access Request) lookups by subject ID
```

**Recommendation:** Use a separate log store from your operational database. OpenSearch with index lifecycle management (ILM) provides the search capability compliance teams need. S3 with object lock provides the tamper-evident archive. Both in parallel is the gold standard.

---

## Feature Priority Recommendations

### v1 — Table Stakes (must ship before any consumer goes live)

| Feature | Rationale | Confidence |
|---------|-----------|------------|
| Data product YAML descriptor with required metadata fields | No catalog without it; blocks everything downstream | HIGH |
| GraphQL subgraph schema registration via CLI (rover subgraph publish) | Core supergraph mechanic | HIGH |
| OAuth 2.0 / OIDC consumer authentication with JWT validation at router | Security prerequisite; no production deployment without it | HIGH |
| Coarse-grained entitlement: allow/deny per consumer per data product | Minimum access control; required for any tenant separation | HIGH |
| Field-level PII masking via OPA policy + GraphQL resolver middleware | GDPR requirement; cannot expose PII without masking in place | HIGH |
| Audit logging with minimum required field set (who, what, when, policy decision) | GDPR Article 30 and SOC 2 CC6.1 require this from day one | HIGH |
| Discovery catalog with search and schema browser | Without discovery, self-service is impossible | MEDIUM |
| Static query complexity scoring + per-consumer quota enforcement | Protects platform from runaway queries before any external consumer is onboarded | HIGH |

### v2 — Core Platform Maturity

| Feature | Rationale | Confidence |
|---------|-----------|------------|
| Row-level filtering via OPA-injected SQL predicates | More granular than coarse entitlement; needed for multi-tenant products | HIGH |
| GitOps-based product registration with CI validation pipeline | Scales to many products without platform team bottlenecks | MEDIUM |
| OPAL-based real-time policy distribution | Enables policy changes without OPA restart; operational necessity at scale | HIGH |
| Consumer access request + approval workflow (data marketplace pattern) | Self-service; without it, all access grants are manual | MEDIUM |
| Per-consumer cost attribution (cost records + chargeback reports) | Required for internal showback / external billing | MEDIUM |
| Schema Proposals for governed schema change management | Prevents breaking changes from reaching production | MEDIUM |
| Data contract registration (SLA, freshness, obligations) alongside schema | Aligns with Open Data Mesh DPDS spec; required for formal data product governance | MEDIUM |

### v3 — Differentiators

| Feature | Rationale | Confidence |
|---------|-----------|------------|
| AI-powered catalog recommendations (surface relevant products to consumers) | Market direction 2024-2025; Collibra and Immuta are building this | LOW |
| Dynamic cost analysis + cost weight auto-calibration | More accurate attribution than static analysis alone | MEDIUM |
| Persisted operation allowlists for external consumers | Hardens security posture for external API tier | MEDIUM |
| Data lineage graph in catalog | High value for trust but complex to build; defer until core is solid | MEDIUM |
| Low-code rule builder UI for product owners to author policies | Reduces friction for non-engineer domain owners | MEDIUM |

### Features to Explicitly Not Build in v1

| Anti-Feature | Why Avoid | Instead |
|---|---|---|
| Build a custom schema registry | Apollo GraphOS schema registry + Rover CLI is production-proven and handles composition | Use Apollo GraphOS or Cosmo (WunderGraph) |
| Build a custom IdP | Auth0/Okta/Keycloak are robust; building JWT issuance is a security liability | Integrate with existing IdP via OIDC standard |
| Log full GraphQL response bodies | Creates secondary PII exposure; violates GDPR data minimisation | Log field names, row counts, and masking decisions only |
| Implement RBAC roles as the primary access control model | Too coarse for field/row-level data control; will require rewrite | Design ABAC with OPA from the start |
| Build static HTML catalog UI before CLI/API registration works | UI without data is worthless and delays the platform team's ability to test with real products | Ship CLI registration first, UI second |

---

## Sources

- [DataHub Data Products](https://docs.datahub.com/docs/dataproducts) — DataHub metadata model and YAML spec
- [DataHub Data Product Entity Metamodel](https://docs.datahub.com/docs/generated/metamodel/entities/dataproduct) — Full aspect model
- [Open Data Mesh DPDS v1.0](https://dpds.opendatamesh.org/specifications/dpds/1.0.0/) — Output port spec, promises/expectations/obligations pattern
- [Open Data Contract Standard (ODCS)](https://github.com/bitol-io/open-data-contract-standard) — LF AI & Data Foundation, 2025
- [Apollo GraphOS Schema Management](https://www.apollographql.com/docs/graphos/platform/schema-management) — Schema registry, Rover CLI, checks, proposals
- [Apollo GraphOS Schema Proposals](https://www.apollographql.com/blog/graphos-schema-proposals-a-principled-development-workflow-for-graphql-api-platforms) — Change management workflow
- [Apollo GraphOS Client Awareness](https://www.apollographql.com/docs/graphos/metrics/client-awareness) — Per-client metrics segmentation
- [Apollo Router Telemetry](https://www.apollographql.com/docs/graphos/routing/observability/telemetry) — OpenTelemetry integration
- [Shopify: Rate Limiting GraphQL APIs by Calculating Query Complexity](https://shopify.engineering/rate-limiting-graphql-apis-calculating-query-complexity) — Production complexity scoring model
- [IBM GraphQL Cost Directives Specification](https://ibm.github.io/graphql-specs/cost-spec.html) — @cost and @listSize directives
- [OPAL Documentation](https://docs.opal.ac/) — Real-time OPA policy distribution
- [OPA Best Practices for Secure Deployment — CNCF](https://www.cncf.io/blog/2025/03/18/open-policy-agent-best-practices-for-a-secure-deployment/) — 2025 OPA guidance
- [Styra Enterprise OPA for Data-Heavy Workloads](https://www.styra.com/enterprise-opa/) — Row/column-level filtering with OPA
- [ABAC with OPA — osohq](https://www.osohq.com/learn/abac-with-open-policy-agent-opa) — ABAC vs RBAC patterns
- [Hasura Row Level Permissions](https://hasura.io/docs/2.0/auth/authorization/permissions/row-level-permissions/) — RLS via session variables pattern
- [GraphQL Authorization — graphql.org](https://graphql.org/learn/authorization/) — Field-level authorization patterns
- [DreamFactory: Ultimate Guide to API Audit Logging](https://blog.dreamfactory.com/ultimate-guide-to-api-audit-logging-for-compliance) — GDPR/SOC2 audit field requirements
- [Splunk Structured Audit Trail Logs](https://help.splunk.com/en/splunk-cloud-platform/administer/manage-users-and-security/10.1.2507/audit-activity-in-the-splunk-platform/about-structured-audit-trail-logs) — Structured log schema patterns
- [Confluent: Real-Time Compliance & Audit Logging with Kafka](https://www.confluent.io/blog/build-real-time-compliance-audit-logging-kafka/) — Kafka-based audit pipeline
- [Google Cloud: Design a Self-Service Data Platform for a Data Mesh](https://docs.cloud.google.com/architecture/design-self-service-data-platform-data-mesh) — Self-service patterns
- [Collibra Data Marketplace](https://www.collibra.com/products/data-marketplace) — Consumer discovery patterns
- [Open Data Product Specification (ODPS) — Linux Foundation](https://opendataproducts.org/) — Vendor-neutral data product standard
- [GraphQL Field Level Authorization — Medium](https://medium.com/@ag_27/graphql-field-level-authorization-solved-using-instrumentation-7710ab2a3af2) — Field-level auth via instrumentation
- [Securing GraphQL APIs — The Guild / Envelop](https://the-guild.dev/graphql/envelop/docs/guides/securing-your-graphql-api) — GraphQL security patterns
- [Persisted Operations — WunderGraph Cosmo](https://cosmo-docs.wundergraph.com/router/persisted-queries/persisted-operations) — Operation allowlist implementation
