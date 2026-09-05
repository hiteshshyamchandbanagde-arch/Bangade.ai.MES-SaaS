# Solution Roadmap — Bangade.ai MES-SaaS

**Scope:** Master roadmap governing all manufacturing industry types, all MES modules, all integrations, all UI/UX device modes, and the delivery methodology used to build them.
**Relationship to existing docs:** This sits *above* `mes-ai-architecture-blueprint.md` (which defines the four-phase Core→Analytics→Predictive→Agentic roadmap and tech stack) and *above* the per-module design doc sets (e.g. `production-module/01–06`, which this roadmap treats as the completed template for all future modules).
**Status:** Draft for sign-off — v1.

---

## 0. How to use this document

This is the map, not the territory. It does not contain functional requirements, schemas, or API contracts — those live in each module's own 6-document set (FDD/TDD/DB-Model/Master-Data/HLA/Detailed-Architecture, following the Production Module template). This document exists to answer four questions before any module's detailed design starts:

1. What industries and manufacturing paradigms must the *whole product* support?
2. What is the full catalog of modules, and in what order do we design/build them?
3. How does each module talk to the outside world (ERP/OPC/LIMS/PLM) and to end users (laptop/tablet/mobile)?
4. What's the delivery rhythm (Agile sprints) that turns this roadmap into shipped software?

---

## 1. Guiding principles

| Principle | What it means here |
|---|---|
| **Multi-tenant, industry-agnostic by default** | No module is designed for one industry and generalized later. Every module's first design pass must answer "how does this work for Discrete, Process, *and* Hybrid" before a single schema table is drawn. |
| **Foundation fully documented, modules incrementally documented** | Master Data, Tenancy, Integration Architecture, and Device/UX Strategy (this document + its direct children) are documented completely up front. Each *module* (Quality, Maintenance, Scheduling, etc.) gets its design-doc-set immediately before its build sprint, not all at once. |
| **Agile delivery, waterfall-free foundation** | The foundation above is the one place a heavier up-front pass is justified — it's expensive to change and everything depends on it. Everything built on top of it ships in short, demoable sprints. |
| **One module, all industries — not one module per industry** | Confirmed via the Production Module precedent: one `ProductionOrderCard`/`production_order` shape with a type discriminator, not parallel Discrete-MES and Process-MES products. This pattern repeats for every module below. |
| **Design docs are a contract, not paperwork** | A module's FDD/TDD/DB-Model/Master-Data/HLA/Detailed-Architecture set is what a fresh engineer (or a fresh Claude session) needs to build it with zero re-discovery. If a doc doesn't reduce future rediscovery cost, cut it. |

**Collaborator's note:** I'd resist the temptation to write a 7th "cross-module" mega-document that tries to hold all future modules' details at once. Keep this roadmap thin (industry list, module list, integration list, device list, sequencing, governance) and let each module's own doc set carry the depth. A roadmap that tries to also be a spec becomes stale the moment the first module's real design diverges from the plan.

---

## 2. Industry & manufacturing-paradigm coverage

### 2.1 Paradigm coverage (already locked in Production Module)
- **Discrete** — Work Orders, Routings, Operations
- **Process** — Batch/Process Orders, Recipes, Phases
- **Hybrid** — both, per-tenant, per-Production-Unit (`production_mode`)

### 2.2 Target industry verticals (Phase 1 priority: Discrete)

| Industry | Paradigm | Phase 1 priority | Notes |
|---|---|---|---|
| Automotive / Auto components | Discrete | **High** | Confirmed Phase 1 focus per prior decision |
| Electronics / Electro-mechanical assembly | Discrete | **High** | Similar shape to automotive; SMT/PCB traceability may need a genealogy variant later |
| General/Industrial machinery | Discrete | **High** | |
| Pharma | Process (regulated) | Deferred | Needs approval workflow, e-signature — deliberately deferred per Master Data Config §4 |
| Food & Beverage | Process (semi-regulated) | Deferred | |
| Chemicals | Process | Deferred | |
| Textiles | Hybrid (cut/sew = discrete, dyeing = process) | Deferred | Good future Hybrid showcase tenant |
| Metals/Foundry | Hybrid (melt=process, machining=discrete) | Deferred | |

**Collaborator's note:** "All manufacturing industry types" as a design *goal* (schema must not preclude them) is right and already achieved by the Discrete/Process/Hybrid model. "All manufacturing industry types" as a Phase 1 *build target* would dilute focus — the two open questions we just resolved (mode-fixed, no-approval-workflow) were explicitly justified by a Discrete-first Phase 1. Treat the industry table above as the honesty check: schema supports all rows, build sequence only targets the "High" rows for now.

---

## 3. Full MES module catalog

Based on the standard MES functional model (MESA-11 / ISA-95), mapped to this product's naming and current status.

| # | Module | Status | Depends on |
|---|---|---|---|
| 1 | **Master Data & Configuration** | 🔴 Not yet formally designed (Production Module's `04-master-data-configuration.md` covers Production's *slice*; this needs to become its own first-class module doc set — see §6) | — (foundation) |
| 2 | **Production (Discrete + Process)** | 🟢 Designed (6 docs, commit `012b4b1`) | Master Data |
| 3 | **Quality / SPC** | 🔴 Not started | Master Data, Production |
| 4 | **Maintenance / CMMS** | 🔴 Not started | Master Data, Production |
| 5 | **Scheduling / APS** | 🔴 Not started | Master Data, Production |
| 6 | **Traceability / Genealogy** | 🟡 Partially covered (Production's `genealogy_link` table + forward/backward queries) — needs its own reporting/UI-facing module doc | Production |
| 7 | **Inventory / WIP & Warehouse** | 🔴 Not started | Master Data |
| 8 | **Document Control** (SOPs, work instructions, revision control) | 🔴 Not started | Master Data |
| 9 | **Labor Management / Time & Attendance** | 🔴 Not started | Master Data |
| 10 | **Performance Analysis / OEE-Yield Reporting** (cross-module analytics beyond per-module metrics) | 🔴 Not started — this is Phase 2 "Embedded Analytics" territory per the architecture blueprint | Production, Quality, Maintenance |
| 11 | **Dispatching / Resource Allocation** (real-time "what should this operator work on next") | 🔴 Not started | Scheduling, Production |
| 12 | **AI Copilot / Agentic layer** | 🔴 Phase 3-4, explicitly out of scope until Core MES modules exist | All of the above |

**Collaborator's note — build sequence recommendation:**
Master Data (as its own module, not a Production sub-doc) → Quality/SPC → Maintenance/CMMS → Scheduling/APS → Traceability (dedicated) → Inventory/WIP → the rest. Reasoning: Quality and Maintenance both hang directly off Production (which is done) and are what a real Discrete plant asks for next after "track my orders" — before Scheduling, which is a harder optimization problem better tackled once the data model has real usage patterns behind it.

---

## 4. Integration architecture — ERP, OPC, LIMS, PLM, and beyond

### 4.1 Integration categories

| System type | Examples | Direction | Pattern |
|---|---|---|---|
| **ERP** | SAP, Oracle, Microsoft Dynamics | Bi-directional | ERP pushes Sales Orders/Product Master/BOM → MES; MES pushes actuals (production confirmations, consumption) → ERP. Async, event-based (not synchronous request/response) to avoid ERP downtime blocking the shop floor. |
| **OPC-UA / OPC-DA** | PLC/SCADA tag feeds | Inbound (mostly) | Already modeled in Production's HLA — MQTT/OPC-UA Gateway translates tags into `parameter_readings`/sensor events. This pattern generalizes to Maintenance (equipment health tags) and Quality (in-line gauge readings). |
| **LIMS** | Lab information systems | Bi-directional | MES sends sample requests (`quality_result.sample_id` reference already stubbed in Production); LIMS returns pass/fail/detailed results. Owned by the future Quality/SPC module, not Production. |
| **PLM** | Product Lifecycle Management (Windchill, Teamcenter) | Inbound | PLM is the master of Product/BOM/Routing *design intent*; MES's `product`/`routing`/`recipe` master data should be able to sync from PLM rather than being hand-entered twice, once PLM integration exists. |
| **WMS** | Warehouse Management | Bi-directional | Inventory/WIP module territory — raw material availability in, finished goods movement out. |
| **HR/Time systems** | Workday, ADP, etc. | Inbound | Labor Management module territory — shift/attendance sync. |

### 4.2 Integration design principle — the adapter pattern

**Collaborator's opinion (this is the most important call in this section):** Do NOT build ERP/OPC/LIMS/PLM integrations as bespoke point-to-point connections per tenant. Every tenant will run a different ERP (or none), different PLC vendors, sometimes no LIMS. Build a single **Integration Gateway** service with:
- A stable internal event contract (e.g. `production.order.completed`, `master_data.product.updated`) that the rest of the MES emits/consumes, agnostic of what's on the other side.
- Per-system-type **adapters** (SAP adapter, OPC-UA adapter, LIMS-generic-HL7-or-REST adapter) that translate the internal contract to/from the external system's actual protocol.
- New tenant with SAP instead of Dynamics → write/reuse an adapter, not redesign the internal contract.

This mirrors the Edge/MQTT Gateway pattern already established in Production's High-Level Architecture (§3: "Edge Gateway is a translation layer, not a pass-through") — this roadmap just generalizes that same architectural decision to ERP/LIMS/PLM, not just OPC.

### 4.3 What this means for module design going forward
Every future module's Technical Design Document should have an explicit "External Integration Points" section (Production's TDD §5 sibling-module integration table is the right shape — extend it to include external systems, not just sibling MES modules).

---

## 5. UI/UX — multi-device strategy

### 5.1 Role-to-device mapping (not one responsive design for everyone)

| Role | Primary device | Why |
|---|---|---|
| Line Operator | **Tablet** (mounted at station) or **Mobile** (walking the floor) | Needs large touch targets, minimal typing, works with gloves/one hand. Laptop is wrong form factor for shop-floor use. |
| Line Supervisor | **Tablet** | Walks the floor but needs more screen real estate than an operator's single-order view — multi-line overview. |
| Plant Manager | **Laptop/Desktop** | Dashboards, trend analysis, cross-shift comparison — data-dense, sit-down analysis. |
| Production Planner | **Laptop/Desktop** | Scheduling tools need a large canvas (Gantt-style views don't work on mobile). |
| Quality Inspector | **Tablet** (at the line) or **Mobile** (roaming audits) | Similar to Operator — in-line data entry, not a desk job. |
| Tenant Admin | **Laptop/Desktop** | Configuration/setup wizards are inherently form-heavy, better on a full keyboard. |

**Collaborator's opinion:** "All UI/UX modes" should not mean "every screen renders identically on all three form factors." It should mean **every role has a first-class experience on the device they'll actually use**, and other devices are supported but not optimized. Concretely: don't spend design budget making the Scheduling Gantt view mobile-first — spend it making sure Operator screens are genuinely thumb-friendly, because that's where device mismatch actually loses adoption on a shop floor. This should be an explicit constraint fed into `frontend-design` work per screen, not a blanket "responsive everything" requirement.

### 5.2 Practical implication for the Figma system
Per the existing design system conventions (`docs/mes-ai-architecture-blueprint.md`, Figma handoff): the current Figma file is fixed at 1440×900 desktop only. Before any Operator/Supervisor/Inspector-facing screen gets built for tablet/mobile, the design system needs breakpoint tokens and a tablet/mobile frame convention — this is a Foundations-page addition, not a per-screen retrofit. **Per your standing instruction, this is Figma work and stays untouched until you explicitly ask for it** — flagging it here only so it's on the roadmap, not proposing to start it.

---

## 6. Agile delivery model

### 6.1 Rhythm
- **Foundation phase** (this document + Master Data module's own 6-doc set + Integration Architecture spec): documented fully before any foundation code is written, because schema/contract changes here are expensive later. Target: 1–2 focused sessions, mirroring how Production Module's docs came together in one session.
- **Per-module sprints** (Quality, Maintenance, Scheduling, ...): each module gets its own short doc→build→demo cycle:
  1. **Design sprint**: FDD → TDD → DB-Model → Master-Data-slice → HLA → Detailed-Architecture (the Production Module template, ~1 session each based on precedent)
  2. **Build sprint(s)**: implementation against the frozen design docs, in small increments (a sprint = one or two functional requirements at a time, not "build all of Quality" as one block)
  3. **Demo/review**: does it match FDD? Any open questions surfaced during build feed back into the *next* module's design (this is the actual Agile feedback loop, not just a build-in-small-batches ritual)
- **Delta increments**: once a module's design docs are frozen and build has started, changes are captured as dated deltas (e.g. an addendum section or a new dated handoff), never silent rewrites of the frozen doc — this preserves the "why" history the AI-coding-dictionary definition of a good handoff calls out (a bad handoff loses the *why*, not just the *what*).

### 6.2 Sprint backlog shape (illustrative, not prescriptive)
```
Sprint 0  — Foundation: Master Data module design docs (this roadmap's immediate next step)
Sprint 1  — Master Data: Site/Area/Unit/Equipment setup wizard (build)
Sprint 2  — Master Data: Product/Routing/Recipe master (build)
Sprint 3  — Quality/SPC: design docs
Sprint 4+ — Quality/SPC: build in FR-sized increments
...
```

**Collaborator's opinion on "go agile":** I agree, with one caveat — Agile for a solo/small-team SaaS build like this should mean *short feedback loops and willingness to revise*, not necessarily ceremony (standups, story points, velocity tracking) that mostly pays off with larger teams. The valuable part of Agile here is: never design more than one module ahead of what you're about to build, and let real build experience correct the next module's design. That's already what happened naturally between the architecture blueprint session and the Production Module session — worth keeping deliberately, not just accidentally.

---

## 7. Immediate next step

Per your stated priority — **Master Data as its own complete module design, covering all manufacturing types** — the recommended next session's scope is:

> Design the Master Data & Configuration module as its own first-class 6-document set (FDD/TDD/DB-Model/Master-Data-Config/HLA/Detailed-Architecture), generalized beyond Production's slice of it. This absorbs and supersedes the config concerns currently living in `production-module/04-master-data-configuration.md`, and adds what Production's doc set didn't need to cover: the tenant onboarding wizard flow itself as a first-class flow (not just "what gets configured" but "the guided UX for configuring it"), cross-module master-data consumers (Quality specs, Maintenance equipment hierarchies, Scheduling calendars all read the same Site/Area/Unit/Equipment/Shift data Production already defined — this doc set should make that explicit so Quality/Maintenance/Scheduling don't redefine their own copies).

No Figma work starts as part of this — per your standing instruction, that stays explicit-ask-only even once this design is done.

---

## 8. Open items for your sign-off

1. Confirm module build sequence in §3 (Master Data → Quality → Maintenance → Scheduling → Traceability → Inventory → rest), or reorder.
2. Confirm Integration Gateway/adapter-pattern approach (§4.2) as the standing architecture decision, so it doesn't get re-litigated per integration.
3. Confirm role-to-device mapping (§5.1) as the standing UX constraint before any tablet/mobile design work begins.
