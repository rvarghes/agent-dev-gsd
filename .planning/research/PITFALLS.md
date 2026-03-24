# Pitfalls Research: Data Products API Platform

**Domain:** GraphQL Federation + Fine-Grained Access Control + Self-Hosted Gateway + Multi-Cloud
**Researched:** 2026-03-23
**Overall Confidence:** HIGH (multiple verified sources; several findings confirmed by official documentation and real-world post-mortems)

---

## GraphQL Federation Pitfalls

### Pitfall 1: Cross-Subgraph N+1 Queries Are Worse Than Monolithic N+1

**What goes wrong:** The N+1 problem in federation is not just slow database queries — it is slow *network* calls. Every unresolved `__resolveReference` for an entity triggers a separate HTTP fetch from the router to a subgraph. A list of 100 data products, each requiring owner metadata from a separate subgraph, generates 100 individual HTTP round-trips unless DataLoader batching is implemented at the subgraph level.

**Why it happens:** Subgraph frameworks expose `__resolveReference` as the entity resolution hook, but do not enforce or even encourage batched implementations. Each subgraph team assumes this is someone else's problem. The federation router correctly identifies that entities need to be resolved but it is the subgraph's responsibility to batch-load them.

**Consequences:** Query latency increases linearly with list sizes. At scale (data catalogs with hundreds of items per page), P99 latency becomes unacceptable. This is invisible in unit tests and only emerges under realistic data volumes.

**Prevention:**
- Mandate DataLoader batching in every subgraph that resolves entity references. Make it a registry requirement, not a suggestion.
- Implement a federated query plan visualizer in your CI pipeline to flag sequential entity resolution.
- Set query timeout SLOs at the router level to surface slow subgraphs during acceptance testing.

**Detection:** Subgraph fetch count in distributed traces (Apollo Router emits this via OpenTelemetry). If fetch count >> distinct entity types in a single operation, N+1 is occurring.

---

### Pitfall 2: Query Planning Is a Sequential CPU Bottleneck

**What goes wrong:** Apollo Router's query planner is CPU-bound and largely non-parallelizable. For queries spanning multiple subgraphs with abstract types (unions/interfaces), query plan computation can take seconds. This happens on *every* unique query shape — a cache miss for a novel query shape stalls the request pipeline.

**Why it happens:** Query planning requires the router to evaluate all possible execution paths through the supergraph to find the optimal plan. The complexity is proportional to query depth and the number of subgraphs involved. Data product platforms naturally produce wide, varied queries as consumers compose their own selections.

**Consequences:** High query shape diversity (inevitable when data product owners define their own schemas) exhausts the query plan cache and drives up median latency. Under load, the router process becomes CPU-saturated rather than I/O-bound.

**Prevention:**
- Enable and tune the query plan cache aggressively. Pre-warm common query shapes on startup.
- Limit query depth and breadth (max depth: 10-15, max root fields: 10) to reduce planning surface area.
- For the self-service portal, constrain the query builder UI to emit a bounded set of normalized query shapes — do not expose arbitrary query composition to anonymous consumers.
- Benchmark with realistic query diversity, not just a handful of test queries.

**Detection:** Monitor `apollo_router_query_planning_time` metric. Spikes indicate planning cache misses or overly complex queries.

---

### Pitfall 3: Schema Composition Failures Block Unrelated Team Deployments

**What goes wrong:** When two subgraph teams make concurrent schema changes, one team's change can fail composition *after* the other team's change has already been published. The platform's supergraph deploy is now blocked by a conflict neither team authored in isolation. CI passes for each team independently; the integration failure only appears at publish time.

**Why it happens:** Apollo's `rover subgraph check` validates a proposed subgraph schema against the *currently published* supergraph, not against other teams' in-flight changes. Two simultaneous valid changes can compose to an invalid supergraph.

**Consequences:** The supergraph enters a "broken composition" state. No new subgraph publishes succeed until the conflict is manually resolved. During this window, all teams are blocked from deploying schema changes.

**Prevention:**
- Run composition checks against a `staging` variant that always tracks `main` branches of all subgraphs. Any team's merge to main immediately tests against all other teams' latest code.
- Implement a schema change registry (even a simple PR-based queue) to serialize concurrent subgraph publishes.
- Define a "composition owner" role responsible for unblocking integration failures within a defined SLA (e.g., 4 hours).

**Detection:** Alert on CI pipeline when `rover subgraph publish` returns a composition error. Track composition success rate as a platform health metric.

---

### Pitfall 4: Breaking Changes Silently Corrupt Consumer Queries

**What goes wrong:** A data product owner renames or removes a field. Their subgraph deploys successfully. The composition check passes because no other subgraph depends on that field. Downstream API consumers — who are querying that field — now receive `null` or an error. Because GraphQL allows partial results, the consumer application may not crash; it silently renders missing data.

**Why it happens:** Federation composition validates *inter-subgraph* compatibility, not *client-to-supergraph* compatibility. Without tracked client operations, there is no mechanism to detect that external consumers depend on a field being removed.

**Consequences:** Silent data regression in consumer applications. Data products are treated as internal implementation details by their owners, but they have public API contracts. The discoverability of the self-service catalog means consumers appear without registering, making impact analysis incomplete.

**Prevention:**
- Require all consumers to register named operations with the gateway (operation document store). This enables field-usage tracking.
- Enforce the add/deprecate/migrate lifecycle: fields must carry `@deprecated` for at least one full release cycle before removal.
- Block field removal in CI unless zero registered operations use the field in the preceding 30-day window.
- Publish a consumer-facing changelog per data product.

**Detection:** Apollo GraphOS field-usage tracking; custom usage analytics at the router layer if self-hosted.

---

### Pitfall 5: Subgraph Failure Cascades to Full Supergraph Unavailability

**What goes wrong:** A single unhealthy or unresponsive subgraph causes the router to return errors for *all* queries that touch any field owned by that subgraph — even if consumers only requested fields from healthy subgraphs in the same operation.

**Why it happens:** GraphQL's default error behavior propagates subgraph errors up to the operation level. If a required entity cannot be resolved, its parent field becomes null. If that field is non-nullable, the null propagates upward, potentially nullifying the entire response.

**Consequences:** A stale or crashed data product subgraph takes down query paths that consumers consider unrelated. A platform with 50 data product subgraphs has 50 potential points of total-response failure per operation.

**Prevention:**
- Use `@defer` for non-critical subgraph data so consumers receive partial results progressively.
- Make entity fields nullable at the supergraph boundary where partial availability is acceptable.
- Implement per-subgraph circuit breakers in the router (Apollo Router supports this via custom plugins/coprocessors).
- Define an explicit "degraded mode" contract: document which subgraph failures produce partial responses vs. hard failures.

**Detection:** Subgraph error rate metrics per subgraph in the router's observability layer. Alert when a subgraph's error rate exceeds 1% over a 5-minute window.

---

## FGAC Implementation Pitfalls

### Pitfall 1: Policy Drift Between Entitlement Store and Enforcement Points

**What goes wrong:** Access policies are defined in an entitlement management system (e.g., OPA, Casbin, or a custom service) but enforcement is duplicated across subgraphs with hand-written logic. Over time, subgraph-level enforcement diverges from the central policy definition. A field that policy says is restricted passes through because a subgraph's `@auth` directive was not updated.

**Why it happens:** When FGAC is implemented as a cross-cutting concern without a single enforcement point, each subgraph team implements authorization locally. Refactors, new fields, and ownership changes create drift without any integration test to detect it.

**Consequences:** Silent data exposure. PII fields that should be masked are returned in plaintext. The gap is often discovered via a security audit or a breach, not monitoring.

**Prevention:**
- Centralize policy evaluation: all authorization decisions flow through a single policy engine (e.g., OPA sidecar). Subgraphs call the policy engine; they do not replicate policy logic.
- Implement a schema annotation protocol: every field touching PII must carry a custom `@sensitive` or `@pii` directive. Enforce this in schema linting as a mandatory check.
- Run automated access-control regression tests as part of the registry's CI: attempt to access protected fields as a low-privilege principal and assert denial.

**Detection:** Automated pen-test queries in CI that attempt to access field values without proper entitlements. Alert if access succeeds.

---

### Pitfall 2: Row-Level Filter Injection Destroys Query Performance

**What goes wrong:** Row-level security (RLS) is implemented by injecting WHERE clauses into subgraph data source queries at runtime based on the requester's entitlements. For complex entitlement structures (e.g., "user sees rows where region = X OR department = Y AND classification <= CONFIDENTIAL"), the injected filter disables database index usage. Full table scans replace index seeks.

**Why it happens:** Entitlement-derived filters are not known at query-planning time by the database optimizer. Compound filter conditions on non-indexed columns, especially OR chains, are not efficiently executable. The problem is invisible in development where tables have tens of rows.

**Consequences:** Queries that run in milliseconds on development data take seconds or minutes on production data volumes. The platform becomes unusable for large tables precisely when data governance demands it most.

**Prevention:**
- Design the row-level security model with the database's execution planner in mind. Prefer partition-by-attribute patterns (user's visible rows are in a pre-materialized view) over runtime filter injection.
- Index every column that participates in entitlement filters. Test with production-scale data early.
- For columnar data stores (BigQuery, Snowflake, Redshift), prefer pre-materialized authorized views over dynamic RLS.
- Cache evaluated row-filter predicates per (user, data product, entitlement version) with short TTLs to avoid re-evaluating policy on every query.

**Detection:** Monitor query execution times per (subgraph, user entitlement tier). Outlier P99 latency by entitlement complexity is a signal.

---

### Pitfall 3: GraphQL Field Masking Bypasses via Fragment Traversal

**What goes wrong:** A field marked `@masked` or protected by a field-level directive is hidden from one query path but accessible through a different traversal route. Attackers (or curious consumers) use inline fragments, fragment spreads, or nested interface implementations to reach the same underlying field via an unprotected path.

**Why it happens:** GraphQL's type system has no directive inheritance rules for interfaces. A directive on an interface field is not automatically enforced on all implementing types. Additionally, the same data can be reached through multiple graph traversal paths; if only the "primary" path has enforcement, alternate paths bypass it.

**Consequences:** PII fields (SSNs, emails, salary data) are accessible to principals who should see only masked values. This is an authorization bypass, not just a data quality issue — it is a compliance failure.

**Prevention:**
- Do not rely on directive-based field masking as the sole enforcement mechanism. Enforce masking at the data layer (the subgraph resolver returns masked values based on context), not at the GraphQL execution layer.
- Audit every type that implements an interface bearing PII fields — confirm that object-level resolvers also enforce masking.
- Disable or tightly control introspection in production. Introspection reveals the full type hierarchy, enabling attackers to discover alternate traversal paths.
- Use a policy-as-code test suite that traverses all graph paths to a sensitive field and asserts masking.

**Detection:** Automated schema traversal tests that enumerate all paths to `@pii`-tagged fields and verify masking at each path. Run these in CI on every schema change.

---

### Pitfall 4: Introspection as a Schema Reconnaissance Tool

**What goes wrong:** GraphQL introspection is left enabled in production. This exposes the complete type system including field names, argument names, type relationships, and custom directives. Even if `@deprecated` or `@internal` directives are present, they are visible. Competitors, data brokers, and malicious actors can discover the full structure of every data product, PII field names, and the internal data model.

**Why it happens:** Introspection is enabled by default in all major GraphQL servers. Development teams rely on it for tooling (GraphQL Playground, schema browsers) and forget to disable it before production.

**Consequences:** Field names reveal business semantics ("salary", "ssn", "risk_score"). Directive names reveal security implementation ("@masked", "@internal", "@requiresRole"). A schema browser for 200 data products is a complete intelligence document about your data estate.

**Prevention:**
- Disable introspection for unauthenticated requests unconditionally. Consider disabling it for all non-admin principals in production.
- Use persisted queries / operation document stores for known consumers rather than ad-hoc introspection-driven clients.
- If schema browsing is needed (e.g., for the self-service catalog UI), serve it from the data catalog layer, not from live GraphQL introspection.

**Detection:** Alert on introspection queries in access logs. Any introspection query from an unexpected source is an anomaly.

---

### Pitfall 5: Entitlement Cache Invalidation Lag Grants Access After Revocation

**What goes wrong:** User entitlements are cached at the gateway or policy engine layer for performance (typically 5-15 minute TTLs). When an entitlement is revoked (employee termination, role change, data classification upgrade), the cache continues granting access until it expires.

**Why it happens:** Entitlement evaluation is expensive (database lookup, policy evaluation, group membership resolution). Caching is necessary for performance. The revocation event is often synchronous in the IAM system but does not propagate to the authorization cache.

**Consequences:** A terminated employee's token continues accessing sensitive data products for up to 15 minutes after termination. For regulated data (GDPR, HIPAA, SOX), this is a compliance gap.

**Prevention:**
- Implement event-driven cache invalidation: publish an entitlement-revocation event on user permission changes. Policy engines subscribe and immediately invalidate affected cache entries.
- Use short TTLs (60-90 seconds) for high-sensitivity data products. Accept the performance cost.
- Separate TTLs by data classification: public data = 15 min TTL, confidential = 60 sec TTL, restricted = no cache.
- For immediate revocation requirements, maintain a revocation token list (similar to JWT blocklists) that is checked before cache.

**Detection:** Audit log analysis: after a revocation event, check if any access was granted under the revoked principal. Alert on any matches.

---

## API Gateway Pitfalls

### Pitfall 1: The Gateway Becomes a Single Composition + Routing Bottleneck

**What goes wrong:** All GraphQL traffic — including query parsing, validation, query planning, and subgraph fan-out — passes through a single gateway process. As data product count grows (50, 100, 200 subgraphs), the query planner's schema increases in size and complexity. What was a 2ms planning time at 10 subgraphs becomes 200ms at 100 subgraphs.

**Why it happens:** Federation gateways are not inherently horizontally scalable in the query planning dimension. Query planning depends on the full supergraph schema, which grows with each new data product registration. Schema growth is a one-way ratchet.

**Consequences:** The gateway becomes the throughput ceiling of the entire platform. Scaling the gateway horizontally helps with I/O concurrency but not with per-request CPU cost of query planning.

**Prevention:**
- Horizontally scale the gateway (multiple router replicas behind a load balancer) from day one. Design deployment to assume N replicas.
- Implement aggressive query plan caching. The Rust-based Apollo Router has a built-in query plan cache — size it based on expected unique operation count.
- Establish a supergraph schema size budget. If schema SDL exceeds a certain threshold (e.g., 5MB), enforce splitting into domain-specific supergraphs with federation stitching at a higher layer.
- Profile query planning time as a first-class metric from the first deployment.

**Detection:** Monitor `query_plan_cache_hit_rate`. A rate below 80% indicates query shape diversity is overwhelming the cache.

---

### Pitfall 2: GraphQL Is POST-Only — Standard HTTP Caching Infrastructure Does Not Apply

**What goes wrong:** Reverse proxies (NGINX, Cloudflare), CDNs, and standard API gateway caching logic are designed for GET requests with URL-keyed caches. GraphQL operations travel in the POST body. Standard infrastructure caching is bypassed entirely.

**Why it happens:** GraphQL's single-endpoint, POST-body design is a fundamental protocol decision, not a misconfiguration. Teams accustomed to REST assume their existing CDN or reverse-proxy caching will help — it does not, by default.

**Consequences:** Every unique query hits the gateway and subgraphs at full cost. For read-heavy analytics workloads (dashboards, reports), this means the platform cannot leverage the most cost-effective caching layer.

**Prevention:**
- Use Kong's `graphql-proxy-cache-advanced` plugin (enterprise) or implement query-body-keyed caching explicitly. The cache key must hash the full request body plus relevant headers (Authorization, Accept-Language).
- For GET-based persisted queries (supported by Apollo Router): register high-frequency operations as persisted queries, then issue them via GET requests, enabling CDN caching by URL.
- Implement response caching within subgraph resolvers (Redis/Memcached) for data that changes infrequently. This is more targeted than gateway-layer caching.
- Design data product metadata queries (catalog browsing) as dedicated, cacheable REST endpoints rather than GraphQL.

**Detection:** Cache hit rate at the gateway layer. If it reads 0% for all operations, caching has not been configured for GraphQL semantics.

---

### Pitfall 3: Kong/Tyk Plugin Complexity Creates an Untestable Middleware Stack

**What goes wrong:** Each new requirement (rate limiting, auth, PII header injection, request logging, schema validation) is implemented as a separate gateway plugin. Over time, the plugin execution chain becomes 15+ plugins deep. Plugin interactions produce unexpected behaviors: a plugin earlier in the chain mutates a header that a later plugin reads, but only under specific request conditions.

**Why it happens:** Gateway plugins are the natural extension point for cross-cutting concerns. Each new operational requirement adds a plugin. The plugin chain is not a first-class testable artifact in most gateway tooling.

**Consequences:** Production outages caused by plugin interaction bugs. Debugging requires reproducing the exact plugin chain ordering locally, which is non-trivial. Upgrading one plugin requires validating the behavior of all downstream plugins.

**Prevention:**
- Document the full plugin execution order and its rationale in version-controlled configuration (Deck for Kong, API definitions for Tyk).
- Write integration tests for the complete plugin chain using the gateway's test harness or a dedicated traffic replay framework.
- Apply the principle of least plugins: implement business logic in subgraphs, not plugins. Plugins should handle infrastructure concerns (auth token validation, rate limiting, logging) only.
- Treat the plugin chain as a contract: any change to the chain requires a full integration test run before production deployment.

**Detection:** Plugin-induced latency per request (observable via Kong's request lifecycle timing headers). Establish per-plugin latency budgets.

---

### Pitfall 4: Self-Hosted Gateway Assumes Operational Maturity That May Not Exist

**What goes wrong:** Teams choose Kong/Tyk for cost control and data sovereignty reasons. The assumption is that OSS gateway is "just another Kubernetes deployment." In practice, running a high-availability API gateway requires: Postgres HA management (Kong's database mode), cross-cluster configuration synchronization, gateway version upgrades without downtime, and 24/7 on-call ownership.

**Why it happens:** OSS gateway documentation focuses on feature capabilities, not operational burden. The comparison with managed services (AWS API Gateway, Apigee) does not account for hidden operational labor.

**Consequences:** Gateway upgrades are deferred indefinitely (security patches missed). Single-node gateway deployments exist in production. Database-backed gateways (Kong with Postgres) have their own backup/recovery requirements.

**Prevention:**
- Use Kong in DB-less/declarative mode with KIC (Kong Ingress Controller) — eliminates the Postgres HA requirement. All configuration is managed via Kubernetes manifests in Git.
- Design for zero-downtime upgrades from day one: blue/green gateway deployments, rolling updates with readiness probes.
- Define a gateway ownership model explicitly: who is on-call for gateway outages? This must be answered before production launch.
- Budget operational labor for gateway management as a recurring cost, not a one-time setup.

**Detection:** Gateway version age metric. Alert if the running version is more than 2 major versions behind latest stable.

---

## Self-Service Registration Pitfalls

### Pitfall 1: Schema Validation Is Syntactic, Not Semantic — Malformed Products Corrupt the Supergraph

**What goes wrong:** The self-registration pipeline validates that a submitted subgraph schema is syntactically valid GraphQL SDL and federation-spec compliant. It does not validate that field names are consistent with existing naming conventions, that entity key types match existing entity definitions, or that the schema does not introduce a type name collision with an existing subgraph.

**Why it happens:** Automated SDL parsers catch syntax errors. Semantic validation — "does this new `Product` type conflict with the existing `Product` entity in the Catalog subgraph?" — requires awareness of the full supergraph context and is often not implemented at registration time.

**Consequences:** A newly registered data product triggers a composition error that blocks *all* other subgraphs from publishing updates. The platform's supergraph is in a broken state, and the data product owner does not understand why because their individual schema was accepted at registration.

**Prevention:**
- Run full supergraph composition as part of the registration validation step. The schema is only accepted if the resulting supergraph compiles without errors.
- Implement a schema linting layer (GraphQL Inspector, custom Rover-based rules) that enforces:
  - Naming conventions (camelCase fields, PascalCase types)
  - Prohibited type names (types that collide with platform-reserved types)
  - Required metadata fields (description on every type and field)
  - Entity key compatibility
- Provide clear, actionable error messages from the validation pipeline that data product owners can act on without platform team intervention.

**Detection:** Track registration rejection rate by error type. High rejection rates indicate the validation feedback loop is unclear.

---

### Pitfall 2: No Rollback Mechanism for Published Subgraph Schemas

**What goes wrong:** A data product owner publishes a schema update that passes all CI checks. Post-deployment, it causes unexpected consumer failures (a query returns incorrect data shapes, an entity key change breaks federation). There is no mechanism to roll back the subgraph schema to its previous version without redeploying the subgraph service itself.

**Why it happens:** Subgraph schemas are tied to service deployments. Rolling back the schema requires rolling back the service. In continuous deployment environments, the previous service version may no longer be available or its rollback may have other unintended consequences. Schema registries typically track current state, not a full version history with rollback capability.

**Consequences:** Breaking changes remain in production until a forward-fix is developed and deployed. Consumer-facing outages persist for hours while the fix is built.

**Prevention:**
- Treat the schema registry as a version-controlled artifact with explicit rollback operations: `rover subgraph publish --schema previous_version.graphql`.
- Maintain the last N schema versions (N >= 5) as rollback targets in the registry. Automate rollback tooling so it takes one command, not a multi-step process.
- Implement pre-publish schema diffing: show the data product owner a consumer-visible change summary before confirming registration.
- For schema changes (not just SDL changes), require a canary deployment period before full promotion.

**Detection:** Consumer error rate spike correlated with a recent subgraph schema publish event. Alert within 5 minutes of publication if consumer error rates increase.

---

### Pitfall 3: Consumer-Breaking Changes When Product Owner Updates Schema

**What goes wrong:** Data product owners treat their subgraph as a private implementation and update schemas without understanding the consumer impact. Field renames, type changes, argument removal, and non-null constraint additions are all breaking changes under GraphQL's type system, but they appear as routine refactors from the owner's perspective.

**Why it happens:** Data product owners are data engineers, not API engineers. They think in terms of table schemas and ETL pipelines, not API contracts. The connection between "I renamed column `user_id` to `userId`" and "I broke every client that queries that field" is not intuitive.

**Consequences:** Silent downstream breakage. Consumers who registered anonymous clients have no contact point for notification. The platform's trust as a stable API surface erodes.

**Prevention:**
- Block all breaking changes in CI using `rover subgraph check --validation-period 30d`. A breaking change requires explicit override with justification.
- Provide data product owners with a "consumer impact report" before any schema change lands: "This change will break N registered operations from M consumers."
- Require data product owners to complete a "breaking change playbook" before deploying: notify consumers, agree on migration timeline, maintain backward compatibility for the transition period.
- Use schema evolution patterns: add new fields, deprecate old ones, remove only after verified zero-usage.

**Detection:** Operations-to-fields usage tracking in the gateway. Any field approaching 0 active consumers before removal is safe. Any field with active consumers being removed is a production incident waiting to happen.

---

### Pitfall 4: Self-Registration Without Lifecycle Management Produces Schema Graveyard

**What goes wrong:** Data product registration is easy. Data product deregistration and deprecation are not. Teams build products for short-lived use cases, projects end, data sources change ownership, and the registered subgraph continues serving stale or incorrect data. 18 months in, the supergraph has 150 registered data products but 40 have not been updated in over 6 months and 15 have broken data sources.

**Why it happens:** Registration workflows are optimized for the happy path: onboarding. Offboarding workflows are rarely built in v1. Stale products have no owner incentive to deregister them (it requires work) and no platform mechanism to force deregistration.

**Consequences:** Schema composition time grows with dead subgraphs. Query planning complexity increases. Consumer documentation becomes unreliable. Security review surface grows with each stale product.

**Prevention:**
- Implement an automated health check for every registered subgraph. If a subgraph health endpoint fails for more than N consecutive days, escalate to its registered owner.
- Assign time-to-live (TTL) to data product registrations. Products must be renewed annually with a confirmed owner and a health attestation.
- Build a "product retirement" workflow that gracefully migrates consumers before deregistration.
- Track "last active query" per data product in analytics. Products with zero queries in 90 days trigger an ownership review.

**Detection:** Per-product query volume metrics at the gateway. Zero-query products over a rolling window are candidates for retirement review.

---

## Multi-Cloud Terraform Pitfalls

### Pitfall 1: Cloud-Specific Kubernetes Networking Breaks Portability Assumptions

**What goes wrong:** Terraform code using `kubernetes_service` with `type: LoadBalancer` provisions cloud-native load balancers that behave differently across clouds. AWS NLB, GCP Cloud Load Balancer, and Azure Load Balancer have different:
- Annotation namespaces for configuration (health check paths, TLS termination, idle timeouts)
- Supported protocols (HTTP/2 support differences)
- Cross-zone load balancing behavior
- IP address assignment timing (NLB can take 60+ seconds; Terraform times out)

Terraform code that works on EKS fails silently or with confusing errors on GKE or AKS.

**Why it happens:** The `kubernetes` Terraform provider abstracts resource types but not cloud-specific behaviors. Teams write Terraform assuming one cloud and copy-paste to another without accounting for behavioral differences.

**Consequences:** Multi-cloud deployments require cloud-specific branches of Terraform code, defeating the purpose of a unified configuration. Ingress rules that work in AWS silently misconfigure health checks in GCP, causing intermittent 504s.

**Prevention:**
- Use a cloud-agnostic ingress layer (Kubernetes Gateway API standard) rather than cloud-specific annotations. NGINX Ingress Controller or Traefik provide consistent behavior across clouds.
- Separate cloud-specific and cloud-agnostic Terraform into distinct modules. Cloud-specific modules (load balancer provisioning) are explicitly per-cloud; shared modules (application configuration) are cloud-agnostic.
- Test every Terraform change against all target clouds in CI, not just the primary cloud.
- Document cloud-specific behavioral differences in the module README.

**Detection:** Integration tests that validate gateway health check responses across all target cloud deployments, run as part of Terraform plan acceptance.

---

### Pitfall 2: Monolithic Terraform State Files Become Operational Liabilities

**What goes wrong:** The Terraform configuration for the entire platform — networking, Kubernetes clusters, gateway deployments, IAM, and data product registry infrastructure — lives in a single workspace with a single state file. A `terraform plan` takes 10+ minutes. A `terraform apply` locks the state for the duration of the apply, blocking all concurrent changes. A state corruption event requires full manual reconstruction.

**Why it happens:** Starting with a single state file is the path of least resistance. The operational cost only becomes apparent as the resource count grows.

**Consequences:** Infrastructure deploys serialize entirely. A routine security group update must wait for the full state to be evaluated. State file corruption — especially during concurrent applies in CI — can require hours of manual repair.

**Prevention:**
- Separate state from day one by component: networking, Kubernetes cluster, gateway, data product registry, each in its own Terraform workspace with its own remote state backend.
- Use Terraform remote state data sources (`terraform_remote_state`) to share outputs between workspaces rather than combining everything into one.
- Require state locking (S3 + DynamoDB for AWS, GCS + Cloud Spanner for GCP) on all state backends. Never use local state in CI.
- Define blast radius rules: a change to the gateway module cannot accidentally affect the Kubernetes cluster module.

**Detection:** Terraform apply duration as a CI metric. Applies taking more than 5 minutes indicate state file size is becoming problematic.

---

### Pitfall 3: Provider Version Drift Across Clouds Causes Silent Incompatibilities

**What goes wrong:** Each cloud's Terraform provider releases on its own schedule. The `hashicorp/aws` provider v5.x introduced breaking changes to EKS node group resources. Teams on AWS upgraded their provider; teams on GCP did not upgrade `hashicorp/google`. The same Terraform module works on one cloud but fails with a type error on another due to resource attribute differences between provider versions.

**Why it happens:** The `.terraform.lock.hcl` file pins provider versions locally but CI environments re-initialize from scratch and can pull different versions if the lock file is not committed or is inconsistently maintained.

**Consequences:** Terraform plans that differ between developers and CI. Infrastructure changes that apply for one cloud but fail for another with cryptic error messages.

**Prevention:**
- Always commit `.terraform.lock.hcl`. Treat it as a first-class artifact, not a generated file to ignore.
- Pin provider versions with pessimistic constraint operators (`~> 5.0.0`) in every Terraform module. Do not use `>= 5.0`.
- Implement a Dependabot or Renovate configuration for Terraform provider version updates. Treat provider upgrades as explicit, tested changes.
- Define a "tested provider version matrix" document: for each cloud and provider, document the tested-and-approved version range.

**Detection:** Terraform plan output showing provider version differences between CI and local runs is a signal. Alert when CI uses a different provider version than the lock file specifies.

---

### Pitfall 4: Multi-Cloud Increases Blast Radius of Terraform Mistakes

**What goes wrong:** A misconfigured Terraform module is deployed across 3 clouds simultaneously in a CI pipeline. What would be a one-cloud incident becomes a three-cloud incident. Common scenarios: an IAM policy that is too permissive (deployed to all clouds before being caught), a network security group change that accidentally blocks gateway traffic (applied to all cloud clusters simultaneously).

**Why it happens:** Multi-cloud CI pipelines often parallelize applies across clouds for speed. The first cloud's apply reveals an error, but the other clouds' applies have already started or completed.

**Consequences:** Multi-cloud simultaneous incidents are significantly harder to recover from than single-cloud incidents. Rollback requires coordinated applies across all clouds.

**Prevention:**
- Implement a "canary cloud" deployment pattern: apply Terraform changes to one cloud first, run acceptance tests, then proceed to additional clouds. Never parallelize the initial apply of a new change.
- Require manual approval gates between cloud deployments in CI for changes that affect security, networking, or IAM resources.
- Use `terraform plan -out` to save and review plans before applying. Do not apply without reviewing the plan output.
- Maintain per-cloud rollback runbooks: what are the exact steps to restore the previous state for each cloud environment?

**Detection:** Post-apply smoke tests per cloud. If any cloud fails smoke tests after apply, halt deployment to remaining clouds immediately.

---

## Observability Pitfalls

### Pitfall 1: Query Cost Complexity Scores Do Not Reflect Actual Resource Consumption

**What goes wrong:** Query complexity scoring assigns integer weights to fields and types to estimate request cost. The score is used for rate limiting and quotas. In practice, complexity score correlates poorly with actual CPU, memory, and downstream subgraph load because:
- A high-complexity query against a cached subgraph is cheap in practice
- A low-complexity query that triggers a full table scan on an unindexed table is expensive
- Field weights are static estimates set at schema registration time; they do not adapt to data volume growth

**Why it happens:** Query complexity is a static, schema-level calculation. It cannot account for runtime data distribution, cache state, or subgraph-specific resource costs.

**Consequences:** Rate limits are wrong in both directions. Cheap queries are over-throttled; expensive queries are under-throttled. A consumer building a high-frequency dashboard is blocked while a low-frequency analytical query consuming significant compute runs freely.

**Prevention:**
- Supplement complexity-based scoring with actual resource consumption tracking: per-operation CPU time, memory, and downstream subgraph fetch counts.
- Implement token-bucket rate limiting based on observed resource units (compute-seconds, rows scanned) rather than schema-derived complexity.
- Use adaptive limits: start with complexity scoring for initial throttling but adjust limits based on observed cost over time.
- Flag data products with high variance between complexity score and actual cost for manual review.

**Detection:** Correlate complexity scores with actual query latency and subgraph fetch counts in tracing data. High variance = scoring model is inaccurate for those queries.

---

### Pitfall 2: PII Leaks into Distributed Traces and Structured Logs

**What goes wrong:** OpenTelemetry traces and structured logs capture request details for debugging. Without explicit scrubbing, these traces include:
- Query variables containing user-supplied data (search terms, IDs, names)
- Response body fragments (GraphQL partial results may include PII fields)
- Request headers containing authorization tokens or session data
- Error messages containing data values from failed resolvers

**Why it happens:** Observability instrumentation is added for debugging and is comprehensive by design. PII scrubbing is an afterthought that requires knowing in advance which fields, variables, and headers contain sensitive data.

**Consequences:** Trace data becomes a secondary PII store that bypasses all FGAC controls. A developer with read access to Datadog/Jaeger/Grafana Tempo can reconstruct PII data from traces that they would not have access to through the API.

**Prevention:**
- Implement a trace scrubbing pipeline: define a blocklist of variable names, header names, and response field paths that must never appear in trace attributes.
- Never log raw GraphQL query variables in production. Log only the operation name and sanitized variable shape (key names without values).
- Classify trace data as sensitive and apply the same access controls as the data products themselves.
- Use OpenTelemetry's `AttributeCountLimit` and custom span processors to scrub known PII field names before export.
- Audit trace stores regularly against the known PII field schema.

**Detection:** Regular automated scans of trace data for patterns matching PII field names from the schema registry. Alert on matches.

---

### Pitfall 3: Audit Log Volume Grows Proportionally to Query Volume and Overwrites Useful History

**What goes wrong:** Every data access event must be audit-logged for compliance (who accessed what, when, what fields were returned). At scale, this generates an event per field per user per query. A dashboard that refreshes every 30 seconds for 100 concurrent users generates millions of audit events per day per data product. Storage costs grow unboundedly; log retention windows shrink as storage is exhausted.

**Why it happens:** Compliance requirements are interpreted as "log everything." The volume implications of per-field, per-request logging at platform scale are not evaluated at design time.

**Consequences:** Audit logs become too expensive to retain for the required compliance window. Queries against audit logs for compliance investigations time out. The compliance team that requires audit logging cannot actually use it.

**Prevention:**
- Design audit logging at the operation level, not the field level. Record which operation was executed, by whom, at what time, against which data product — not which individual fields were resolved.
- For PII access specifically, log the fact of access (operation + data product + principal) not the values accessed.
- Implement tiered audit retention: hot storage (7 days) for operations review, warm storage (90 days) for compliance queries, cold storage (7 years) for regulatory archival. Cost-optimize each tier separately.
- Use columnar storage formats (Parquet on S3/GCS) for audit logs rather than log aggregation services. Query costs are orders of magnitude lower.
- Batch audit events and write asynchronously. Never let audit logging be in the critical path of query execution.

**Detection:** Audit log ingestion rate (events/second) as a cost metric. Model the cost at target query volume before deployment.

---

### Pitfall 4: Distributed Tracing Gaps Leave Root Cause Analysis Blind Spots

**What goes wrong:** Traces show the router receiving a request and subgraphs returning data, but do not show:
- Which SQL query was executed inside a subgraph
- Which OPA policy evaluation happened for FGAC
- Which rows were filtered by RLS before returning
- What the entitlement cache hit/miss rate was

**Why it happens:** Distributed tracing is straightforward between HTTP services (W3C trace context propagation handles this). But internal subgraph operations (database queries, policy evaluations, cache lookups) require explicit instrumentation that each subgraph team must implement independently.

**Consequences:** Debugging latency issues requires guessing at the subgraph internals. A P99 latency spike might be caused by a slow OPA policy evaluation, but the trace only shows total subgraph response time.

**Prevention:**
- Define a "subgraph instrumentation contract": all subgraphs must emit spans for database queries, policy evaluations, and cache operations. Make this a registry acceptance requirement.
- Provide a shared instrumentation library for each language/framework used by subgraph teams. Standardize span naming conventions.
- Instrument the FGAC layer explicitly: OPA evaluation time, cache hit/miss, rows filtered count.
- Use structured logging alongside traces. Log entries correlated by trace ID fill observability gaps where spans are too expensive.

**Detection:** Trace completeness audit: for a sample of production operations, check if all key internal operations (DB query, policy eval, cache lookup) have associated child spans.

---

## Operational Pitfalls

### Pitfall 1: Supergraph Composition Becomes a Platform Team Bottleneck

**What goes wrong:** Supergraph composition — the process of combining all subgraph schemas into a unified supergraph SDL — requires platform team involvement whenever composition fails. As data product count grows, composition failures from concurrent schema changes become frequent. Each failure requires platform team diagnosis and coordination to resolve. The platform team becomes the critical path for every data product schema change.

**Why it happens:** Composition failure messages from the federation tooling are often cryptic (pointing to specific GraphQL spec violations rather than business-level explanations). Data product owners cannot self-diagnose them. Platform team becomes the translator.

**Consequences:** Platform team spends an increasing fraction of time on reactive composition firefighting instead of building platform capabilities. Data product owners experience long lead times for schema changes. The platform's self-service promise erodes.

**Prevention:**
- Invest early in composition error translation: build tooling that maps federation composition error codes to human-readable explanations with actionable remediation steps.
- Provide a local composition testing tool to data product owners so they can validate composition before pushing. `rover dev` or a Docker-based local supergraph environment enables this.
- Define escalation thresholds: if composition is blocked for more than 4 hours, trigger an incident. Do not let composition failures sit unresolved.
- Track time-to-resolve composition failures as a platform health metric. Rising MTTR indicates the feedback loop is broken.

**Detection:** Composition failure rate (failures per day) and MTTR per failure as leading indicators of platform health.

---

### Pitfall 2: Data Product Sprawl Degrades Discoverability and Performance

**What goes wrong:** The self-service model succeeds — 50 products become 150 products in 18 months. The schema SDL grows to 5MB. Query planning time increases. The data catalog UI loads slowly. Consumers cannot find the data products they need. Schema naming collisions increase. Security review surface becomes unmanageable.

**Why it happens:** Growth is the success metric. No architectural upper bound is defined for data product count. The platform team celebrates registrations but does not track sprawl impact on platform performance.

**Consequences:** The platform paradoxically becomes less useful as it grows. Discoverability collapses — consumers abandon the catalog and start sending requests to data owners directly, rebuilding exactly the data silos the platform was meant to eliminate.

**Prevention:**
- Define a maximum subgraph count per domain supergraph (e.g., 50 products per domain supergraph). When a domain approaches the limit, split it into a child supergraph.
- Implement a tiered architecture for very large platforms: domain supergraphs compose a platform supergraph. Each layer is independently manageable.
- Build a data product health scoring system: active queries, consumer count, owner responsiveness, documentation quality. Surface this score in the catalog.
- Require product owners to justify new registrations against existing products (does this overlap with an existing product?).

**Detection:** Supergraph SDL size as a metric. Query planning P99 latency as a function of time. Both should trend stably, not monotonically upward.

---

### Pitfall 3: Ownership Ambiguity When Data Product Owners Leave or Teams Reorganize

**What goes wrong:** A data product is registered by a team that is subsequently reorganized, the original owner leaves, or the team's mandate changes. The subgraph continues running. No one knows who is responsible for it. When it begins returning stale data, has a security vulnerability, or needs a schema update, no one owns the work.

**Why it happens:** Registration captures an initial owner. Ownership transfer workflows are not built in v1. Organizations change faster than platform ownership records.

**Consequences:** Stale data products continue to be queried by consumers who assume they are authoritative. Security patches are not applied because there is no owner to apply them. The data product becomes technical debt that cannot be retired because consumers depend on it.

**Prevention:**
- Store ownership as a team/group, not an individual. When the individual leaves, ownership transfers to the team.
- Implement quarterly ownership attestation: every data product owner must confirm their product is active and accurate. Non-response triggers escalation.
- Require an organizational backup owner (the data domain lead) for every data product. They become the responsible party when the primary owner is unavailable.
- Implement an automated "ownership gap detector": if a product's primary owner account is inactive for 90 days, alert the backup owner and the platform team.

**Detection:** Track "ownership age" per product: time since last ownership confirmation. Alert at 90 days without confirmation.

---

### Pitfall 4: Zero-Downtime Requirement Conflicts With Subgraph Schema Changes

**What goes wrong:** Data product subgraphs run on Kubernetes. Schema changes are deployed as rolling updates. During the rollout, old pods (serving the old schema) and new pods (serving the new schema) are running simultaneously. The federation router, unaware of this duality, routes some requests to old pods and some to new pods. Consumers receive inconsistent responses during the transition window.

**Why it happens:** Kubernetes rolling updates are designed for stateless service changes. Schema changes are not stateless — they change the type system that the federation router uses to plan and execute queries.

**Consequences:** During rolling deploys, consumers experience intermittent errors or inconsistent field values. The window is typically short (minutes) but is predictable and repeating with every deploy.

**Prevention:**
- Follow the GraphQL schema evolution principle: add fields, deprecate old ones, never remove or rename in a single atomic deploy. With additive-only changes, old and new pods serve compatible responses.
- For breaking changes, use the expand-and-contract pattern: deploy the new field, migrate consumers, then deploy the removal in a separate operation.
- Implement blue/green deployment for subgraphs with breaking schema changes: route traffic fully to the new version only after all pods are running the new schema.

**Detection:** Elevated error rate or response inconsistency correlated with active rolling deployments. Set deployment windows during low-traffic periods for high-impact schema changes.

---

## Top 5 Things to Get Right in v1

These are the design decisions that are the most expensive to retrofit. Getting them wrong means rewrites, not refactors.

### 1. Centralized Policy Enforcement, Not Distributed Directive-Based Authorization

The biggest source of security incidents in this class of platform is FGAC implemented as GraphQL directives that each subgraph team applies inconsistently. Design the authorization architecture so that access decisions are made by a single policy engine (OPA or equivalent) called from the gateway layer, and the result is passed as context to subgraphs. Subgraphs enforce masking and filtering based on pre-evaluated context, not by independently evaluating policies. This is the architecture to get right in v1 — retrofitting from distributed to centralized authorization after 50 subgraphs are live is a multi-quarter project.

### 2. Schema Registry With Full Supergraph Composition Validation at Registration Time

Do not accept a subgraph schema until full supergraph composition has been attempted and succeeded. Validation that does not test actual composition gives a false signal of safety. Build the composition check into the registration API itself: reject the request with a detailed error message if composition fails. This is the foundation of the self-service model — without it, the first 20 data product onboardings will produce recurring composition failures that consume platform team bandwidth indefinitely.

### 3. DataLoader Batching as a Mandatory Subgraph Contract, Not a Recommendation

The N+1 problem across subgraph network boundaries is catastrophic at data product scale because each subgraph hop is an HTTP round-trip, not just a database call. Make DataLoader-based entity resolution a mandatory requirement — not a best practice — for all subgraphs that resolve entity references. Validate this in your subgraph acceptance test suite (send a batch entity resolution request and verify the subgraph makes a single upstream call, not N calls). Subgraphs that fail this test are not accepted into the registry.

### 4. Operational Ownership Model Before Platform Launch, Not After

Define the complete operational ownership model before the first production data product is registered:
- Who owns composition failures?
- Who owns gateway incidents?
- Who owns stale data products?
- What is the SLA for responding to a consumer-reported data product error?
- What is the escalation path when an owner is unreachable?

These questions seem like governance paperwork. In practice, they determine whether the platform operates reliably at scale or degenerates into a data swamp. The technical platform can be perfect; without clear ownership, it still fails.

### 5. PII Field Classification as a Schema-Level Primitive, Not an Afterthought

Define a `@pii` or `@sensitive` directive schema annotation before the first data product is registered. Require data product owners to classify every field at registration time. This single investment pays dividends across:
- FGAC enforcement (masking policies target `@pii` fields automatically)
- Audit logging (PII field access is auto-logged)
- Trace scrubbing (PII fields are auto-excluded from trace attributes)
- Compliance reporting (PII data map is generated directly from the schema registry)

Retrofitting PII classification after 100 data products are registered means auditing every field of every product manually — a task that takes months and produces an incomplete result.

---

## Sources

- [Handling the N+1 Problem - Apollo GraphQL Docs](https://www.apollographql.com/docs/graphos/schema-design/guides/handling-n-plus-one)
- [Hidden Complexities of Scaling GraphQL Federation - DEV Community](https://dev.to/hackmamba/hidden-complexities-of-scaling-graphql-federation-and-how-to-fix-them-2peg)
- [GraphQL Federation Supergraphs at Scale - Crashbytes](https://crashbytes.com/blog/graphql-federation-supergraphs-scale-apollo-router-schema-governance)
- [Battling Latency: Lessons from Implementing GraphQL Federation - Medium](https://medium.com/@ebutrera910322/battling-latency-lessons-from-implementing-graphql-federation-in-a-microservices-architecture-70838b70e0db)
- [Federated Schema Checks - Apollo GraphQL Docs](https://www.apollographql.com/docs/federation/v1/managed-federation/federated-schema-checks)
- [Deploying API Changes with GraphOS - Apollo GraphQL Docs](https://www.apollographql.com/docs/technotes/TN0028-change-management)
- [Schema Composition - Apollo GraphQL Docs](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/composition)
- [Directive Deception: Bypassing Security with GraphQL - InstaTunnel](https://instatunnel.my/blog/directive-deception-exploiting-custom-graphql-directives-for-logic-bypass)
- [Federated Sub-graph Injection: The Blind GraphQL Data Leak - Medium](https://medium.com/@instatunnel/federated-sub-graph-injection-the-blind-graphql-data-leak-e34b7e88ea48)
- [The Complete GraphQL Security Guide - WunderGraph](https://wundergraph.com/blog/the_complete_graphql_security_guide_fixing_the_13_most_common_graphql_vulnerabilities_to_make_your_api_production_ready)
- [GraphQL - OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)
- [Governing GraphQL APIs with Kong Gateway - Kong Inc.](https://konghq.com/blog/engineering/governing-graphql-apis-with-kong-gateway)
- [GraphQL Proxy Caching Advanced - Kong Docs](https://docs.konghq.com/gateway/latest/kong-plugins/graphql/)
- [Understanding and Implementing API Gateway Clusters - Tyk](https://tyk.io/blog/understanding-and-implementing-api-gateway-clusters/)
- [Multi-Cloud Provisioning and Management with Terraform - Spacelift](https://spacelift.io/blog/terraform-multi-cloud)
- [What 3000+ Terraform Files Reveal About Cloud Drift - Firefly](https://www.firefly.ai/academy/what-3000-terraform-files-taught-us-about-cloud-drift)
- [Multi-Cloud Kubernetes Survival Guide - ADEO Tech Blog](https://medium.com/adeo-tech/multi-cloud-kubernetes-survival-guide-49eee9aa58e2)
- [Multi-Cloud Load Balancers Explained: AWS vs GCP vs Azure - HackerNoon](https://hackernoon.com/multi-cloud-load-balancers-explained-aws-vs-gcp-vs-azure-l4-l7-and-global-edge)
- [Securing Your GraphQL API from Malicious Queries - Apollo GraphQL Blog](https://www.apollographql.com/blog/securing-your-graphql-api-from-malicious-queries)
- [Towards Avoiding the Data Mess: Industry Insights from Data Mesh Implementations - arXiv](https://arxiv.org/html/2302.01713v4)
- [The State of Data Mesh in 2026 - Thoughtworks](https://www.thoughtworks.com/insights/blog/data-strategy/the-state-of-data-mesh-in-2026-from-hype-to-hard-won-maturity)
- [Query Complexity in a Federated API - Apollo Community](https://community.apollographql.com/t/query-complexity-in-a-federated-api/2358)
- [Backward Compatibility in Apollo Federation 2 - Apollo GraphQL Docs](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/backward-compatibility)
- [Schema Deprecations - Apollo GraphQL Docs](https://www.apollographql.com/docs/graphos/schema-design/guides/deprecations)
- [Protecting GraphQL APIs from Malicious Queries - Cloudflare](https://blog.cloudflare.com/protecting-graphql-apis-from-malicious-queries/)
- [GraphQL Cyclic Queries and Depth Limiting - Escape](https://escape.tech/blog/cyclic-queries-and-depth-limit/)
- [Best Practices Guide for GraphQL Observability - Hasura](https://hasura.io/blog/best-practices-guide-for-graphql-observability-with-hasura-part-1)
- [State of GraphQL Federation 2025 - WunderGraph](https://wundergraph.com/state-of-graphql-federation/2024)
- [Solving Data Sprawl with GraphQL and Apollo Federation - Trility](https://www.trility.io/insights/graphql-apollo-federation)
