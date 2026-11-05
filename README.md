# War Room Social Listening — Documentation Index

**Product:** TVK War Room Social Listening, Phase 1 (X + News)  
**PRD Author:** Kishore (Minsky product), 2026-11-03  
**Documentation Status:** Complete — Ready for OpenHands implementation  
**Documentation Author:** Generated from PRD analysis

---

## Reading Order

### 1. Start Here
- [`PRD-GAP-ANALYSIS.md`](./PRD-GAP-ANALYSIS.md) — Missing info, ambiguities, open decisions
- [`REQUIREMENTS-TRACEABILITY.md`](./REQUIREMENTS-TRACEABILITY.md) — PRD → implementation mapping
- [`01-product-overview.md`](./01-product-overview.md) — What we are building and why

### 2. Product Understanding
- [`02-functional-requirements.md`](./02-functional-requirements.md) — What the system must do
- [`03-user-roles-and-permissions.md`](./03-user-roles-and-permissions.md) — Who can do what
- [`04-user-flows-and-workflows.md`](./04-user-flows-and-workflows.md) — End-to-end journeys
- [`05-business-rules.md`](./05-business-rules.md) — Logic and validation rules

### 3. UX/UI Design
- [`06-information-architecture.md`](./06-information-architecture.md) — Navigation and page hierarchy
- [`07-screen-specifications.md`](./07-screen-specifications.md) — Screen-by-screen specs
- [`08-interaction-and-state-specifications.md`](./08-interaction-and-state-specifications.md) — States and interactions

### 4. Technical Architecture
- [`09-system-architecture.md`](./09-system-architecture.md) — System design
- [`10-technology-stack.md`](./10-technology-stack.md) — Tech choices and justification
- [`11-database-schema.md`](./11-database-schema.md) — Entities and relationships
- [`12-api-specification.md`](./12-api-specification.md) — API endpoints
- [`13-authentication-and-security.md`](./13-authentication-and-security.md) — Auth and security
- [`14-integrations-and-dependencies.md`](./14-integrations-and-dependencies.md) — External services

### 5. Development
- [`15-project-structure.md`](./15-project-structure.md) — Repository layout
- [`16-development-guidelines.md`](./16-development-guidelines.md) — Coding standards
- [`17-module-implementation-plan.md`](./17-module-implementation-plan.md) — Feature breakdown
- [`18-testing-and-qa-plan.md`](./18-testing-and-qa-plan.md) — Testing strategy
- [`19-deployment-and-devops.md`](./19-deployment-and-devops.md) — Infrastructure
- [`20-environment-and-setup.md`](./20-environment-and-setup.md) — Local setup

### 6. OpenHands Execution
- [`21-openhands-agent-instructions.md`](./21-openhands-agent-instructions.md) — **Master instructions for OpenHands**
- [`22-implementation-roadmap.md`](./22-implementation-roadmap.md) — Phases and milestones
- [`23-task-breakdown.md`](./23-task-breakdown.md) — Granular dev tasks
- [`24-validation-and-completion-checklist.md`](./24-validation-and-completion-checklist.md) — QA checklist

---

## Key Facts at a Glance

| Item | Value |
|------|-------|
| Client | TVK (Tamil political party; first client, not only one) |
| Phase | 1 — X and News only |
| Core entities tracked | TVK, CM Vijay, AadhavArjuna, all Indian parties |
| Languages | Tamil, English, Tanglish; app UI in English and Tamil |
| Freshness target | ≤5 minutes post-to-screen (X); same-morning (print/e-paper) |
| History | From 2026-05-10 (TVK government sworn in) |
| Publishing | None in Phase 1 — read, analyse, alert, never post |
| Budget | No monthly cap; cost-efficiency built in |
| Must-pass tests | Councillor incident + single high-influence post |

---

## Critical Constraints for Implementers

1. **X API limit:** 3M post reads/month (~100k/day). Enterprise or firehose vendor talks must start immediately — the current limit will not cover full scope.
2. **No hard-coding of TVK specifics:** All client-specific data (entities, tiers, watch terms, thresholds) must be configuration rows, not code. Later phases will onboard other clients without code changes.
3. **Never auto-publish:** The system cannot post, reply, or message anyone outside the war room.
4. **Campaign attribution requires human confirmation** before any party is named.
5. **Ally blame in CM summaries requires a named lead review** before delivery.
6. **Corrections never overwrite originals** — always append with who/when/why.
7. **Every label shows its reason**; every number shows its denominator.
8. **Privacy:** India DPDP Act 2023 applies. No private citizen profiles. No citizen/petition record links.
