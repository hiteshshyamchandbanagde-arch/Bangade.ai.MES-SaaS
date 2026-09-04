# Technical Design Document (TDD) — Production Module

**Related docs:** [Functional Design](./01-functional-design-document.md) · [Database Model](./03-database-model.md) · [High-Level Architecture](./05-high-level-architecture.md) · [Detailed Architecture](./06-detailed-architecture.md)

This document translates the Functional Design Document's requirements into concrete technical decisions: API shape, service boundaries, integration patterns, and cross-cutting concerns (tenancy, auth, error handling).

---

## 1. Tech stack (Production Module specifics)

Per the parent architecture blueprint's stack recommendations, applied to this module:

| Layer | Choice | Rationale specific to Production |
|---|---|---|
| API framework | Node.js/TypeScript or Python (FastAPI) — either works; pick based on team familiarity | Production API is I/O-bound (DB reads/writes, occasional MQTT events), not compute-heavy, so framework choice is a team-preference decision, not a performance one |
| Relational store | PostgreSQL | Owns `production_order`, master data, genealogy — see Database Model doc |
| Time-series store | TimescaleDB (Postgres extension) | `production_event`, `downtime_event` — high write volume, time-range queries |
| Edge ingestion | MQTT broker + OPC-UA gateway | Per parent blueprint — this module consumes from it, doesn't own the broker |
| Auth | Tenant-scoped JWT, `tenant_id` claim set on every request | Feeds the RLS `current_setting('app.current_tenant')` per Database Model §7 |

---

## 2. API design

### 2.1 Design principles
- **Order-type-aware but not order-type-duplicated endpoints.** Rather than `POST /work-orders` and `POST /batch-orders` as fully separate APIs, use `POST /production-orders` with an `order_type` field, and type-specific nested detail objects. This mirrors the Database Model's shared-header pattern and keeps the Production Overview's mixed-list query (FR-S1) a single endpoint, not two the frontend has to merge.
- **State transitions are explicit actions, not raw status PATCHes.** `POST /production-orders/{id}/release`, `.../start`, `.../complete` rather than `PATCH {status: "..."}`  — this lets the API enforce the state-machine rules from FDD §4.1/§5.1 (e.g., FR-D5's "can't complete until all operations done") server-side, not trust the client to only send valid transitions.

### 2.2 Core endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/production-orders` | GET | List orders, filterable by unit/status/type — backs Production Overview |
| `/production-orders` | POST | Create (Created state) |
| `/production-orders/{id}` | GET | Full detail — backs Order Detail / Line Detail drill-down |
| `/production-orders/{id}/release` | POST | Created → Released (validates Routing/Recipe is Active — FR-P1) |
| `/production-orders/{id}/operations/{opId}/start` | POST | Discrete: start an operation |
| `/production-orders/{id}/operations/{opId}/complete` | POST | Discrete: complete an operation, log qty good/scrap |
| `/production-orders/{id}/phases/{phaseId}/start` | POST | Process: start a phase |
| `/production-orders/{id}/phases/{phaseId}/readings` | POST | Process: log/stream a parameter reading (also the Edge Gateway's write path) |
| `/production-orders/{id}/phases/{phaseId}/complete` | POST | Process: complete a phase |
| `/production-orders/{id}/downtime` | POST | Log a downtime event (FR-D3) |
| `/production-orders/{id}/complete` | POST | Final completion — validates all operations/phases done or waived (FR-D5/FR-P6) |
| `/production-units/{id}/summary` | GET | Current order + live metrics for one unit — the Line Detail data source |
| `/genealogy/{lotId}/forward` | GET | Forward trace (FR-S5) |
| `/genealogy/{lotId}/backward` | GET | Backward trace (FR-S5) |

### 2.3 Response shape for mixed Discrete/Process lists
The `GET /production-orders` list response returns a common envelope regardless of type, with an `order_type`-discriminated `detail` object:

```json
{
  "id": "...",
  "order_type": "BATCH_ORDER",
  "production_unit_id": "...",
  "status": "IN_PROGRESS",
  "product": { "sku": "...", "name": "..." },
  "metrics": { "yield_pct": 94.2, "uptime_pct": 88.0 },
  "detail": { "current_phase": "React", "phase_progress_pct": 60 }
}
```

For `order_type: "WORK_ORDER"`, `metrics` instead carries `{oee_pct, availability_pct, performance_pct, quality_pct}` and `detail` carries `{current_operation, operation_progress_pct}` — this is exactly the shape the shared `ProductionOrderCard` component (see design system follow-up) needs to render either variant from one API contract.

---

## 3. Tenancy enforcement (implementation of architecture blueprint §2.1)

- Every request carries a tenant-scoped JWT; middleware extracts `tenant_id` and issues `SET app.current_tenant = '<uuid>'` at the start of each DB transaction, before any query runs.
- RLS policies (Database Model §7) are the enforcement backstop — even if application-layer filtering has a bug, a missing/wrong `tenant_id` in a query still can't return another tenant's rows.
- **Do not** rely on application code alone to filter by `tenant_id` — this is the specific failure mode the architecture blueprint calls out as a data-leak risk.

---

## 4. Error handling and validation

- **State-machine violations** (e.g., trying to complete an order with pending operations) return `409 Conflict` with a structured error naming the specific unmet condition (`"reason": "PENDING_OPERATIONS", "pending_operation_ids": [...]`) — the UI needs this detail to tell the operator *what's* blocking completion, not just that something is.
- **Recipe/Routing not Active** (FR-P1 gate) returns `422 Unprocessable Entity` at release time, not a silent fallback to the last-known version.
- **Out-of-spec parameter readings** (FR-P4) do NOT return an error — they're accepted and flagged (`out_of_spec: true` persisted), with the alert surfaced as a separate event, since the functional requirement is explicitly "flag, don't block."

---

## 5. Integration contracts with sibling modules

| Module | Integration pattern | Data exchanged |
|---|---|---|
| Quality/SPC | Production writes `quality_result.sample_id` reference; Quality module owns sample detail and pushes pass/fail back via webhook or polled status | Sample ID, pass/fail result |
| Maintenance/CMMS | Production's `downtime_event` with reason code "Breakdown" can trigger a CMMS work order creation (async event, not synchronous call) | Downtime event → CMMS ticket |
| Scheduling | Scheduling module reads Production Order status (read-only) to know actual vs. planned; Production does not call into Scheduling | Order status, actual timestamps |

---

## 6. Testing/validation approach (brief)

- State-machine transitions (FDD §4.1, §5.1) should be covered by explicit unit tests per state × per invalid-transition-attempt, since this is where silent bugs cause the most damage (an order stuck in an impossible state is a support escalation, not a minor UI glitch).
- RLS policies should have an automated cross-tenant leak test in CI — attempt every read/write endpoint with Tenant A's token against Tenant B's data and assert `403`/empty result, not just happy-path tests.
