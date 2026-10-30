---
name: prd-change-plan-template
description: Template for Change Plan section in PRD — decomposition into phases with testing and documentation tasks
---

# PRD Change Plan Template

## Overview

The **Change Plan** is the final output of Step 14 (Decompose). It breaks the PRD Functional Requirements into concrete tasks organized by phase, with ownership and dependencies.

Structure:
- **Phases:** 0 (Prep), 1 (Core), 2 (Integration), 3 (Tests/Docs), optionally 4+ for post-launch.
- **Task ID:** `CP-{phase}.{number}` (e.g., `CP-1.3`).
- **Task Description:** 1 sentence; FR reference.
- **Depends:** other task IDs or external dependencies.
- **Responsible:** role or team (e.g., "Backend Lead", "QA").

---

## Template

```markdown
## Change Plan

| Phase | Task ID | Description | Depends | Responsible |
|-------|---------|-------------|---------|-------------|
| **Phase 0: Prep** |
| 0 | CP-0.1 | [Design decisions] | — | [role] |
| 0 | CP-0.2 | [Setup/config] | — | [role] |
| **Phase 1: Core** |
| 1 | CP-1.1 | [Implement FR-X] | CP-0.X | [role] |
| 1 | CP-1.2 | [Implement FR-Y] | CP-1.1, CP-0.X | [role] |
| **Phase 2: Integration** |
| 2 | CP-2.1 | [Integrate with external system] | CP-1.X | [role] |
| 2 | CP-2.2 | [Handle error paths from integration] | CP-2.1 | [role] |
| **Phase 3: Testing & Documentation** |
| 3 | CP-3.1 | [Unit tests for FR-X, FR-Y] | CP-1.X | QA |
| 3 | CP-3.2 | [Integration tests for Phase 2] | CP-2.X | QA |
| 3 | CP-3.3 | [Update API documentation] | CP-1.X | Backend Lead |
| 3 | CP-3.4 | [Update user guide / runbook] | CP-1.X | Tech Writer / Product |
| 3 | CP-3.5 | [Update admin/ops documentation] | CP-2.X | Ops Lead |
| **Phase 4: QA & Hardening** |
| 4 | CP-4.1 | [End-to-end testing] | CP-3.X | QA |
| 4 | CP-4.2 | [Performance testing] | CP-1.X | [role] |
| 4 | CP-4.3 | [Security review & penetration testing] | CP-2.X | Security |
| **Phase 5: Deployment & Monitoring** |
| 5 | CP-5.1 | [Database migrations (if needed)] | CP-1.X | DBA |
| 5 | CP-5.2 | [Deploy to staging] | CP-4.X | DevOps |
| 5 | CP-5.3 | [Smoke tests & monitoring setup] | CP-5.2 | DevOps / Ops |
| 5 | CP-5.4 | [Deploy to production] | CP-5.3 | DevOps |
```

---

## Guidance by Feature Type

### UI-Only Feature

**Phases:**
- 0: Design & setup (component library, storybook)
- 1: Component implementation
- 2: Integration with state (context, store)
- 3: Testing (unit tests for components, integration with state)
- 3: Documentation (design system update, user guide)
- 4: E2E testing (Cypress, Playwright)

**Testing tasks (Phase 3):**
- Unit tests for each new component (happy path, props variations, error states)
- Integration tests if state management involved
- Accessibility tests (a11y) if critical

**Documentation tasks (Phase 3):**
- Update component library / Storybook
- Update user guide (if user-facing workflow changed)
- Update design system documentation

---

### Backend Feature

**Phases:**
- 0: Design (schema, API contract, error codes)
- 1: Model/schema implementation (database changes, migrations)
- 1: API implementation (endpoints, validation, error handling)
- 2: Integration with external services (if applicable)
- 3: Unit tests (model tests, service tests)
- 3: Integration tests (API tests against DB)
- 3: Documentation (API docs, migration guide, runbook)
- 4: Performance testing (if applicable)
- 5: Deployment (migrations, monitoring)

**Testing tasks (Phase 3):**
- Unit tests for business logic (service layer, validators)
- Unit tests for database models (ORM relationships)
- Integration tests for API endpoints (including error cases)
- Edge case tests (concurrent requests, race conditions, large payloads)

**Documentation tasks (Phase 3):**
- API documentation (OpenAPI/Swagger spec, endpoint descriptions, examples)
- Database migration guide (schema changes, rollback procedure)
- Runbook for ops (how to configure, how to troubleshoot, how to scale)
- Internal architecture docs (if schema design is complex)

---

### Integration Feature (e.g., third-party API integration)

**Phases:**
- 0: Design (data flow, error handling, authentication strategy)
- 1: SDK/client setup (wrapper around external API, retry logic)
- 1: Sync/ingestion logic (pull data, transform, validate)
- 2: Error handling & retry (circuit breaker, fallback, logging)
- 2: Webhook/notification listeners (if two-way sync)
- 3: Unit tests (SDK tests, sync logic tests, error handling)
- 3: Integration tests (end-to-end with sandbox/staging of external service)
- 3: Documentation (integration guide, API mapping, troubleshooting)
- 4: Load testing (if high-volume sync expected)
- 5: Monitoring & alerting (sync failures, latency, data consistency)

**Testing tasks (Phase 3):**
- Unit tests for SDK wrapper (mocked external API)
- Unit tests for data transformation & validation
- Integration tests against external service sandbox/staging
- Error path tests (timeout, rate limit, 5xx from external service)
- Idempotency tests (duplicate webhook, retry logic)

**Documentation tasks (Phase 3):**
- Integration guide (how to set up credentials, configure sync frequency)
- Field mapping documentation (which source fields → which target fields)
- Troubleshooting guide (common errors, how to debug, how to resync)
- API contract documentation (rates, delays, guarantees from external service)
- Webhook handler documentation (payload format, signature verification)

---

### Architecture/Refactoring Feature

**Phases:**
- 0: Design & planning (module boundaries, migration strategy)
- 1: Core refactoring (restructure modules, extract interfaces)
- 1: Backwards-compatibility layer (if needed for zero-downtime)
- 2: Integration & rollout (gradual migration, feature flags)
- 3: Unit tests (refactored modules, contracts)
- 3: Integration tests (how refactored modules interact)
- 3: Documentation (architecture decision record, module responsibilities, migration guide for future dev)
- 4: Performance baseline (if performance was motivation)
- 5: Gradual rollout / feature flag toggle

**Testing tasks (Phase 3):**
- Unit tests for refactored modules (ensure contracts are maintained)
- Integration tests (old system ↔ new system during migration)
- Regression tests (ensure no new bugs in untouched code)
- Load tests (if performance is a concern)

**Documentation tasks (Phase 3):**
- Architecture Decision Record (ADR) — why this refactoring, trade-offs
- Module ownership & responsibilities (who maintains which module)
- Migration guide (for developers: how to use new architecture)
- Deprecation guide (if old code will be removed: timeline, replacement)

---

## Key Rules

1. **Every FR must map to at least one task.**
2. **Testing & Documentation are mandatory phases.** They appear after Core/Integration but before Deployment. DO NOT skip them.
3. **Each testing/documentation task has an owner.** It's not a "if we have time" — it's scheduled upfront.
4. **Blocked tasks:** If a task depends on an open OQ, add `[BLOCKED: OQ-X]` in the Description or Depends column.

---

## Notes

- **No "Polish" or "Bug Fix" phase without concrete tasks.** Every task must be specific (test for X, doc for Y).
- **Docs and tests are NOT "if we have time."** Phase 3 is scheduled before deployment.
