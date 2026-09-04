# Functional Design Document (FDD) — Production Module

**Module:** Core MES — Production (Discrete + Process)
**Product:** Bangade.ai MES-SaaS
**Scope:** Phase 1 (Core MES), multi-tenant
**Related docs:** [Technical Design](./02-technical-design-document.md) · [Database Model](./03-database-model.md) · [Master Data Configuration](./04-master-data-configuration.md) · [High-Level Architecture](./05-high-level-architecture.md) · [Detailed Architecture](./06-detailed-architecture.md)

---

## 1. Purpose and scope

The Production Module is the core of the MES — it tracks what is being made, where, by whom, and against what plan, in real time. This document defines *what the system must do* for two fundamentally different manufacturing paradigms:

- **Discrete manufacturing** — countable units produced against a Work Order via a Routing (sequence of operations across stations/lines).
- **Process manufacturing** — batches or continuous volumes produced against a Batch/Process Order via a Recipe/Formula (sequence of phases with process parameters).

A single tenant may run **Discrete-only, Process-only, or Hybrid** (both, on different Sites/Areas). The module must support all three without forcing one paradigm's data shape onto the other.

Out of scope for this document: Scheduling/APS (separate module), Quality/SPC detail (separate module — Production only carries pass/fail and sample linkage), Maintenance/CMMS (separate module), AI/Copilot features (Phase 3-4).

---

## 2. Users and roles

| Role | Primary needs from this module |
|---|---|
| **Line Operator / Machine Operator** | Start/pause/complete an operation or phase, log actuals (qty, scrap, downtime reason), see the current order's instructions |
| **Line Supervisor / Shift Lead** | Real-time view of all lines/units in their area, reassign orders, acknowledge alerts, approve exceptions |
| **Plant Manager** | Plant-wide production overview, OEE/yield trends, cross-shift comparison |
| **Production Planner** | Release orders to the floor, view order status against schedule (read-only here; scheduling itself is a separate module) |
| **Quality Inspector** | Record in-process quality checks against an order/batch, trigger holds |
| **Tenant Admin** | Configure Sites/Areas/Lines, production mode (Discrete/Process/Hybrid) — see Master Data Configuration doc |

---

## 3. Core concepts (shared vocabulary)

| Term | Definition |
|---|---|
| **Production Order** | Umbrella term for a unit of work released to the floor — manifests as either a **Work Order** (Discrete) or a **Batch/Process Order** (Process) |
| **Site / Area / Production Unit** | ISA-95 physical hierarchy. "Production Unit" is used instead of "Line" at the schema/functional level because a Process train (reactors, tanks) isn't a "line" in the discrete sense — the UI can still label it "Line" for Discrete tenants |
| **Routing** (Discrete) | Ordered sequence of Operations a Work Order passes through, each tied to a station/equipment |
| **Recipe / Formula** (Process) | Ordered sequence of Phases a Batch Order passes through, each with target process parameters (temp, pressure, mix time, etc.) and ingredient charges |
| **Genealogy** | Traceability linkage — for Discrete, unit/serial → components consumed; for Process, batch → ingredient lots consumed (and forward to downstream batches/packages) |

---

## 4. Functional requirements — Discrete

### 4.1 Work Order lifecycle
States: `Created → Released → In Progress → Paused → Completed → Closed` (plus `Cancelled` from any pre-Completed state).

- FR-D1: The system shall allow a Work Order to be released to a specific Production Unit only if that Unit's `production_mode` includes Discrete.
- FR-D2: The system shall track a Work Order's progress operation-by-operation per its Routing, recording actual start/end time, actual quantity good, and actual scrap quantity per operation.
- FR-D3: An operator shall be able to log a downtime event against an in-progress operation, selecting a reason code from a tenant-configured downtime reason list.
- FR-D4: The system shall compute OEE (Availability × Performance × Quality) per Production Unit per shift, using logged run time, ideal cycle time (from the routing/equipment master), actual output, and scrap.
- FR-D5: The system shall not allow a Work Order to be marked Completed until all Routing operations are either Completed or explicitly skipped with a reason (e.g. rework routing).
- FR-D6: The system shall support partial completion — a Work Order can produce and report quantity in increments before final completion.

### 4.2 Discrete dashboard/detail requirements
- FR-D7: The Production Overview shall show, per Discrete Production Unit: current Work Order ID + product, status badge (Running/Warning/Critical/Idle), OEE/Availability/Performance/Quality, and an 8-hour trend sparkline.
- FR-D8: Per the data-honesty principle already established in the design system: Critical/Idle units shall show `0%`/`—` rather than fabricated numbers, with an explicit "no data" indicator.
- FR-D9: Drilling into a unit shall show the active Work Order's full routing progress (operation-by-operation), recent events, and equipment status — matching the existing Line Detail screen pattern.

---

## 5. Functional requirements — Process

### 5.1 Batch/Process Order lifecycle
States: `Created → Released → Charging → In Process → Holding → Completed → Closed` (plus `Cancelled`, and `On Hold` as an exception state reachable from any active phase, e.g. triggered by a quality excursion).

- FR-P1: The system shall allow a Batch Order to be released only against a Recipe version that is Active (not Draft or Deprecated) for that tenant/product.
- FR-P2: The system shall track batch progress phase-by-phase per the Recipe, recording actual start/end time and actual process parameter readings (or manual entries where no sensor feed exists) against each phase's targets.
- FR-P3: The system shall support ingredient charge tracking — recording actual lot number and quantity consumed per ingredient per phase, against the Recipe's specified ingredients.
- FR-P4: The system shall flag a phase as **out of spec** if a logged/streamed parameter falls outside the Recipe's defined tolerance band, and surface this as an alert without blocking the phase (operator/supervisor decides whether to hold).
- FR-P5: The system shall compute a **yield** metric (actual output ÷ theoretical yield from the Recipe) and an **uptime** metric (in-process time ÷ scheduled time) per Production Unit per shift, as the Process-mode equivalent of OEE.
- FR-P6: A Batch Order shall not be marked Completed until all Recipe phases are Completed or explicitly waived with a reason and approver.

### 5.2 Process dashboard/detail requirements
- FR-P7: The Production Overview shall show, per Process Production Unit: current Batch Order ID + product, status badge, Yield/Uptime metrics, current phase name, and either a level/throughput gauge or a phase-progress bar (not a sparkline, which implies discrete-cycle data that doesn't exist here).
- FR-P8: The data-honesty principle applies identically: a unit not currently running a batch shows explicit "no batch in progress" state, not fabricated gauge values.
- FR-P9: Drilling into a unit shall show the active Batch Order's phase timeline (charge → react → hold → discharge, or tenant-defined phase names), live parameters vs. setpoints, ingredient charge log, and recent events — the Process-mode equivalent of Line Detail.

---

## 6. Shared/cross-cutting functional requirements

- FR-S1: A single Production Overview screen shall render both Discrete and Process Production Units side by side (for Hybrid tenants), using a shared card component that adapts its metric layout by `production_mode`.
- FR-S2: All production events (order state changes, downtime, quality holds, phase transitions) shall be written to a common event/audit log, tenant-scoped, timestamped, and attributable to a user.
- FR-S3: All Production Order data shall be scoped by `tenant_id` at the data layer (not just filtered in application code) per the multi-tenancy architecture decision.
- FR-S4: The system shall allow a Production Unit's `production_mode` to be configured per tenant during Master Data setup (Discrete / Process / Hybrid), and this shall determine which order types can be released to it.
- FR-S5: Genealogy shall be queryable both forward (from a raw material lot, what finished units/batches contain it) and backward (from a finished unit/batch, what lots/components went into it), regardless of Discrete or Process origin.

---

## 7. Explicitly out of scope for Phase 1

- Advanced/constraint-based scheduling (order sequencing beyond simple release order)
- Full SPC (control charts, Cpk) — Production only records pass/fail and links to a sample ID
- Predictive quality/maintenance triggers (Phase 3)
- Multi-site consolidated reporting beyond simple cross-site rollup (Phase 2 — Embedded Analytics)

---

## 8. Open questions for stakeholder sign-off

1. For Hybrid tenants, can a single Production Unit switch between Discrete and Process mode over time (e.g., a changeover line), or is `production_mode` fixed per unit at configuration time? *(Current assumption: fixed per unit; revisit if a real customer needs changeover.)*
2. Should partial Work Order completion (FR-D6) support multiple concurrent partial-completion transactions, or must they be sequential? *(Current assumption: sequential, one open completion transaction at a time.)*
3. What's the minimum viable "yield" definition for Process (FR-P5) when a Recipe has multiple co-products? *(Deferred — single-product yield only for Phase 1.)*
