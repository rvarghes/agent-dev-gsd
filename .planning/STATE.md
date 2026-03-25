---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: unknown
last_updated: "2026-03-25T17:47:36.155Z"
progress:
  total_phases: 6
  completed_phases: 0
  total_plans: 0
  completed_plans: 0
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-23)

**Core value:** Any authorized consumer can query any data product — or compose multiple products — through a single GraphQL endpoint, with PII protection enforced transparently via entitlements.
**Current focus:** Ready for Phase 1 planning

---

## Current Status

**Milestone:** v1.0
**Phase:** Not started — project initialized, ready to plan Phase 1
**Last action:** Project initialized with research, requirements, and roadmap

---

## Phase Progress

| Phase | Name | Status | Plans |
|-------|------|--------|-------|
| 1 | Foundation Infrastructure | ○ Pending | Not started |
| 2 | Security & Access Control | ○ Pending | Not started |
| 3 | Data Product Registration & Catalog | ○ Pending | Not started |
| 4 | Consumer Onboarding & Cost Management | ○ Pending | Not started |
| 5 | Row-Level Security & Policy Maturity | ○ Pending | Not started |
| 6 | Platform Governance & Operational Maturity | ○ Pending | Not started |

Progress: ░░░░░░░░░░ 0%

---

## Key Decisions Made

- **GraphQL-first, REST deferred to v2** — all v1 access patterns fit GraphQL
- **Hive Gateway (MIT) over Apollo Router** — ELv2 license; Apollo FGAC directives are enterprise-only
- **Tyk OSS over Kong OSS** — GraphQL-aware features free in Tyk OSS tier
- **OPA for FGAC** — only engine with SQL predicate push-down for row filtering
- **AWS EKS for v1** — multi-cloud design but AWS first; Terraform modules portable to GCP/Azure
- **Centralized FGAC in OPA coprocessor** — not per-subgraph directive logic (prevents policy drift)
- **Kafka audit pipeline from day one** — GDPR compliance activates on first production query
- **Segmented Terraform state** — designed before any infrastructure provisioned

---

## Open Questions

These must be resolved before or during planning:

1. **Which IdP is in use?** (Okta, Auth0, Keycloak, Azure AD) — determines JWT claim structure for Phase 2
2. **Compliance obligations for v1?** (GDPR, SOC 2, HIPAA) — determines audit log retention requirements
3. **Expected data product team count at v1 launch vs 12 months post-launch** — determines if manual onboarding suffices for v1
4. **Existing data products to migrate, or greenfield?** — determines if introspection-based auto-registration is needed in Phase 3
5. **Existing data catalog (Collibra, Alation)?** — determines if DataHub deployment is needed or catalog integration

---

## Research Summary

Research completed 2026-03-23. See `.planning/research/SUMMARY.md` for full details.

**Recommended stack:** Hive Gateway + Hive schema registry, OPA + OpenFGA + Envelop, Tyk OSS, Traefik, DataHub, LGTM (Loki + Grafana + Tempo + Mimir), Kafka, Terraform + Helm + ArgoCD

**Top risks to design against:**

1. PII leaking into distributed traces (must scrub OTel pipeline before any trace export)
2. Distributed FGAC logic producing silent PII exposure (centralize in OPA coprocessor)
3. N+1 network round-trips at federation boundaries (mandate DataLoader batching at registration)
4. Schema composition failures blocking all teams (composition owner + staging variant + serialized publishes)
5. Entitlement cache lag after revocation (event-driven invalidation, 60s TTL for sensitive products)

---

## Session Continuity

**Last session:** 2026-03-25T17:47:36.147Z
**Next action:** `/gsd:plan-phase 1` — create execution plans for Phase 1 (Foundation Infrastructure)

---
*State last updated: 2026-03-23 after project initialization*
