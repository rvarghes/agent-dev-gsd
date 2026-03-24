# Data Products API Platform

## What This Is

A GraphQL-first platform that exposes structured data products (warehouse tables and views) through a unified federated supergraph with fine-grained access control. Data product owners self-register their products via configuration or CLI — no ops involvement required. Consumers (internal teams and external partners) query a single gateway endpoint with row-level filtering and field-level PII masking enforced per entitlement policy.

## Core Value

Any authorized consumer can query any data product — or compose multiple products — through a single GraphQL endpoint, with PII protection enforced transparently via entitlements.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Data product owners can register a structured table/view as a data product via config or CLI
- [ ] Consumers can query registered data products via a unified GraphQL API
- [ ] Multiple data products can be composed/federated in a single GraphQL query (supergraph)
- [ ] Row-level filtering enforced per consumer entitlement policy
- [ ] Field-level PII masking/redaction enforced per consumer entitlement policy
- [ ] Entitlement policies are manageable without platform team involvement
- [ ] All GraphQL traffic routes through a centralized self-hosted OSS API gateway (Kong or Tyk)
- [ ] Full platform infrastructure provisioned via Terraform (AWS as primary target)
- [ ] CI/CD pipeline included for multi-cloud deployment
- [ ] Query metrics and latency tracked per data product and consumer
- [ ] Access audit logs capture who queried what and when (compliance)
- [ ] Cost attribution tracks query costs per consumer/team for chargeback

### Out of Scope

- REST API — not needed for v1; GraphQL covers all access patterns (deferred to v2)
- Data lineage tracking — valuable but out of scope for v1 to keep complexity manageable
- Mobile SDKs / client libraries — consumers use standard GraphQL clients
- Real-time subscriptions / event streaming — v1 is query-only (tables/views, not Kafka)
- Multi-cloud beyond AWS for v1 — Terraform designed for multi-cloud portability but AWS is the v1 target

## Context

- **Domain**: Data mesh / data product ecosystem — this is infrastructure for exposing analytical and operational data assets via API
- **GraphQL federation strategy**: Multiple data products each expose their own GraphQL schema; a supergraph gateway (e.g., Apollo Federation, The Guild's Hive, or similar) stitches them into a unified endpoint
- **API gateway**: Self-hosted OSS (Kong or Tyk) — cloud-neutral, Kubernetes-deployable, sits in front of the supergraph
- **Access control model**: FGAC at two levels — row filtering (consumer sees only their data scope) and field masking (PII columns redacted or nulled based on entitlement role)
- **Self-service model**: Data product owners define their product schema and access policy; the platform registers it into the supergraph automatically
- **Infrastructure philosophy**: Terraform-first, cloud-neutral by design — AWS for v1 but infra code must be portable to GCP/Azure without rearchitecting

## Constraints

- **Tech stack**: GraphQL-first; REST deferred to v2
- **Deployment**: Must be deployable on Kubernetes; Terraform for infra provisioning
- **Cloud neutrality**: No cloud-vendor-specific services in the core platform path — use OSS equivalents
- **Security**: PII field masking and row-level filtering are hard requirements, not optional features
- **Self-service**: Platform must not require a central ops team for day-to-day product registration

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| GraphQL-first, REST deferred | All known v1 access patterns fit GraphQL; REST adds complexity for no gain now | — Pending |
| Unified supergraph federation model | Enables composing data products without consumer knowing data locations | — Pending |
| Self-hosted OSS API gateway (Kong/Tyk) | Cloud-neutral requirement rules out managed gateways (AWS API GW, Apigee) | — Pending |
| AWS-first Terraform (multi-cloud design) | Ship faster on AWS while keeping infra portable for future cloud targets | — Pending |
| Data lineage out of scope for v1 | Reduces complexity; query metrics + audit logs cover the core observability need | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-03-23 after initialization*
