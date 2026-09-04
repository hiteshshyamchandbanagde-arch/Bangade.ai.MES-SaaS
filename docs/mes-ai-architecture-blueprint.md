# MES SaaS — AI-Native Architecture & Tooling Blueprint

**Purpose:** Companion strategy doc to the Figma design system handoff (`0aYcQ2bGx9JD4d5MDLMVWL`). Answers: how do you design and build a best-in-class MES SaaS, and which AI tools/approaches actually belong in it, given the 4-phase roadmap (Core MES → Embedded Analytics → Predictive ML → Prescriptive/Agentic).

---

## 0. The one decision that shapes everything else

You are not *deploying* AI into one factory — you are *building a product* that other manufacturers will run. That reframes "which AI tool is best" into two separate questions:

1. **Where should AI be native to your SaaS** (built by you, becomes your differentiation, sits inside your ISA-95 data model)?
2. **Where should you integrate/partner** with a best-of-breed vendor via API instead of rebuilding their 5-10 years of domain-specific model training?

Rule of thumb emerging from the 2026 vendor landscape: **hardware-coupled, physics-heavy AI → partner. Data-and-reasoning AI → build native.**

| Capability | Build native into your MES? | Why |
|---|---|---|
| LLM copilot (natural language over your own data) | **Build** | This *is* your product's differentiation; foundation LLMs make this buildable in weeks, not years |
| Computer vision defect detection | **Build (using foundation vision models) or offer as a connector** | No longer requires a dedicated vision-AI vendor for most defect classes — modern multimodal models + a training loop get you 80% there; keep an integration slot open for specialists (Cognex/Keyence/Elementary) for high-precision optical use cases |
| Vibration/acoustic predictive maintenance | **Partner/integrate** | Requires proprietary sensor hardware and years of failure-mode training data (Augury, Senseye) — not a reasonable build target |
| Anomaly-based predictive maintenance from your own PLC/sensor data | **Build** | You already own the data pipeline (ISA-95 equipment model); standard time-series anomaly detection is well within reach |
| Advanced constraint-based scheduling (APS) | **Build a "good enough" version natively, offer APS connectors for complex cases** | Full APS (PlanetTogether-class) is a multi-year optimization engineering effort; most of your customers don't need that depth on day one |

---

## 1. Reference architecture (layers)

```
┌─────────────────────────────────────────────────────────┐
│  Presentation Layer — your Figma design system            │
│  (Dashboard, Scheduling, Quality/SPC, Maintenance,         │
│   Traceability, Reports)                                   │
├─────────────────────────────────────────────────────────┤
│  Application Layer — multi-tenant MES SaaS                 │
│  (ISA-95/88 objects: Site → Area → Line → Equipment;        │
│   Work Orders, Routings, Genealogy, Quality Specs)          │
├─────────────────────────────────────────────────────────┤
│  AI/Agent Layer                                             │
│  - Copilot (LLM + tool-calling over your own MES API)       │
│  - Predictive models (anomaly detection, forecasting)       │
│  - Vision inference service (defect classification)         │
│  - Scheduling optimizer (constraint solver + heuristics)    │
├─────────────────────────────────────────────────────────┤
│  Data Backbone — Unified Namespace                          │
│  (Time-series store + event bus + relational store for      │
│   ISA-95 master data; this is the layer every AI feature    │
│   above reads/writes from — get this right before Phase 3)  │
├─────────────────────────────────────────────────────────┤
│  Edge/OT Layer — PLC/SCADA/sensor ingestion                 │
│  (OPC-UA, MQTT; where customers' brownfield equipment        │
│   connects in)                                               │
└─────────────────────────────────────────────────────────┘
```

**Critical sequencing insight:** every AI capability in Phase 3-4 depends on the Data Backbone being real (structured, tagged, time-series, tenant-isolated) — not bolted on later. If you only do one architectural thing right in Phase 1, make it this layer.

---

## 2. Multi-tenancy — concrete decisions needed now

**This product is built to be sold to multiple manufacturers, not run for a single plant.** Pune Plant / Building 2 is a reference/pilot case, not the customer. That status is confirmed, not aspirational — the decisions below stop being deferrable the moment a second tenant's data touches the schema, so they belong in Phase 1, not Phase 2.

### 2.1 Tenant isolation strategy

| Approach | Pros | Cons | Fit |
|---|---|---|---|
| **Schema-per-tenant** (separate Postgres schema per customer) | Strong isolation, easy per-tenant backup/restore/export, simple to reason about for compliance-sensitive manufacturing customers | Migration fan-out (one schema change × N tenants), connection pooling gets harder at scale | Better for **early-stage, few large enterprise customers** where isolation and per-customer data export matter more than horizontal scale |
| **Shared schema + Row-Level Security (RLS)** with a `tenant_id` on every table | Single schema to migrate, scales to many small/mid customers cheaply, standard Postgres feature (no app-layer trust required) | Requires rigorous discipline — every query, every index, every new table must carry `tenant_id` from day one; a missed RLS policy is a data leak | Better for **many small-to-mid manufacturers** (the more likely SaaS shape at scale) |

**Recommendation:** start with **RLS + shared schema**, because retrofitting tenant isolation into an already-built single-tenant schema is far more expensive than building it in from Phase 1. Enforce `tenant_id` at the Postgres RLS policy level (not just in application code) so a bug in the API layer can't leak cross-tenant data — this matters especially once the AI/Agent layer starts running queries on customers' behalf.

**Action for Phase 1:** every table in the current schema design — Work Orders, Equipment, Lines, Alerts, Genealogy — needs a `tenant_id` column and an RLS policy before any real customer data goes in, even the pilot.

### 2.2 Plant/site configurability

The current Figma screens and any generated prototypes assume **one fixed plant shape**: 6 lines (Assembly/Welding/Assembly/Paint/Packaging/Inspection), Pune's specific layout. A real tenant might have 2 lines or 40, different line names, different ISA-95 area/line hierarchies, and different KPI targets (a food & beverage plant's quality specs look nothing like an automotive stamping plant's).

What needs to become tenant-configurable rather than hardcoded:
- Number of lines/areas/equipment (ISA-95 Site → Area → Line → Equipment hierarchy, per tenant)
- Line names, shift patterns, and shift labels ("Shift A · 06:00–14:00" is Pune's convention, not universal)
- KPI/OEE target thresholds (what counts as "Warning" vs "Critical" varies by industry and by customer)
- Quality spec definitions (SPC control limits, defect taxonomies)

**Implication for the Data Backbone (Section 1):** the ISA-95 object model needs a tenant-scoped configuration layer sitting above the raw schema — think of it as "tenant settings" that parameterize how the same UI renders for a 3-line plant vs. a 40-line plant, rather than baking Pune's shape into the tables or the screens.

### 2.3 Onboarding / setup flow

A single-tenant internal tool can be configured by an engineer running SQL. A SaaS product needs a **self-serve (or at least sales-assisted) setup flow**: a new tenant admin defines their plant hierarchy, lines, shifts, and initial users without your team touching a database. This doesn't need to exist for the pilot, but the schema and API decisions in Phase 1 should not assume a human will always configure new tenants by hand — that assumption gets expensive to unwind later.

**Design implication:** the Figma file has no admin/tenant-settings screens yet (flagged in the design handoff as "not started"). This is no longer a nice-to-have — plant configuration, user management, and (eventually) billing are core product surface for a multi-tenant SaaS, not an afterthought screen.

### 2.4 Billing / plan tiers

The 4-phase roadmap (Core MES → Embedded Analytics → Predictive ML → Prescriptive/Agentic) is a natural candidate for plan-tier packaging — e.g. Core MES as the base plan, Predictive ML and the LLM copilot as premium add-ons. Worth deciding *roughly* which phase maps to which tier before Phase 3 work starts, since it affects whether predictive/agentic features are built as tenant-wide capabilities or per-seat/per-tenant toggleable features from the start.

### 2.5 Cross-tenant AI/ML data isolation

This is easy to miss and expensive to get wrong: once Phase 3 (Predictive ML) and Phase 4 (LLM copilot) exist, **training data and RAG context must stay tenant-scoped**. A predictive maintenance model trained on aggregate data must not leak one tenant's failure patterns into another's inference, and the copilot's RAG layer (pgvector) must filter by `tenant_id` on every retrieval — the same discipline as the RLS policies in 2.1, applied to the AI layer specifically.

---

## 3. AI capability map, mapped to your roadmap

### Phase 1 — Core MES (current)
No AI yet — but design the schema so AI can read it later:
- Every equipment/line/work-order record needs stable IDs and timestamps (this is what makes Phase 3 anomaly detection and Phase 4 copilot tool-calling possible without a data migration)
- Store raw sensor/event data in a time-series-friendly shape even if you don't yet analyze it

### Phase 2 — Embedded Analytics
- Standard BI: OEE calculations, downtime pareto, SPC control charts — no AI required, this is what your Reports and Quality/SPC screens already model
- This phase is really "prove the data pipeline works end-to-end" before you spend AI budget

### Phase 3 — Predictive ML
Landscape as of 2026:

- **Predictive maintenance vendors** span from CMMS-native AI (fast to deploy, no data science team needed) to enterprise sensor-hardware platforms. Buyers typically evaluate these platforms on deployment speed, sensor flexibility, whether alerts are simple thresholds or true ML anomaly detection, whether alerts auto-generate a work order versus sitting in a siloed dashboard, and total cost of ownership including hidden data-science costs. Augury remains the reference point for large enterprises wanting a fully managed, proprietary hardware-software machine-health stack, while faster-deploying, no-code platforms increasingly target mid-market brownfield plants that can't sustain a six-month implementation.
  - **Implication for you:** don't try to out-build Augury's sensor hardware. Build anomaly detection on the sensor/PLC data your customers *already* feed into your MES, and leave an integration point for customers who also run a specialist platform.

- **Computer vision quality inspection**: the category splits between AI that automates administrative inspection workflows and AI that runs computer vision directly on the line to catch defects in real time, with the second category showing the largest capability gap between vendors and the highest ROI. Newer edge-first platforms differentiate on training speed (models trained in under an hour) and running inference on-device without a cloud dependency or subscription.
  - **Implication for you:** a "good enough" defect-classification feature (fine-tuned or few-shot on a foundation vision model, deployed as an inference microservice your Quality/SPC screen calls) covers most SMB customers. Reserve deep hardware partnerships (Cognex/Keyence) for enterprise deals that demand it.

- **Dynamic/AI scheduling**: the core failure mode of legacy scheduling projects is that a powerful optimization engine sits on top of disconnected, manual shop-floor data — the algorithm can compute a perfect sequence while being blind to a machine that's actually down. The market direction is away from static, weekly re-planning cycles toward continuous scheduling that adjusts automatically to shop-floor events in real time.
  - **Implication for you:** since your MES *is* the shop-floor data source (not a bolted-on APS reading stale ERP exports), you have a structural advantage here — your scheduling AI can react to real work-order/equipment-status changes the moment they happen, which most standalone APS tools can't do natively.

### Phase 4 — Prescriptive/Agentic
- **LLM copilots**: the copilots that actually deliver value are grounded in a company's specific documentation and plant data rather than generic training knowledge, give source-attributed answers, and are integrated into the CMMS/ERP/quality systems rather than existing as a standalone Q&A tool. There's also a growing edge-deployment trend — running smaller, task-scoped models on-site rather than always round-tripping to the cloud — driven by plants wanting narrowly-scoped copilots rather than general-purpose assistants.
  - **Implication for you:** this is your highest-leverage build. A copilot that can query your own ISA-95 objects (via tool-calling against your MES API) and reason over customers' work-order history/quality specs is something you're positioned to build better than any third-party wrapper, because you own the schema.

---

## 4. Concrete tool/stack recommendations for building this yourself

**LLM/copilot layer**
- Anthropic API (Claude) with tool-calling/MCP for the copilot to query your own MES endpoints — same MCP pattern you're already using for Figma, applied to your production API
- RAG layer over customer-specific docs (SOPs, work instructions) using a vector store (pgvector is the pragmatic choice if you're already on Postgres — no separate vector DB to operate)

**Time-series / data backbone**
- TimescaleDB (Postgres extension) or ClickHouse for sensor/event data at scale
- MQTT or OPC-UA gateway for edge ingestion — this is the connector layer that lets brownfield customers plug in

**Predictive ML**
- Start with standard anomaly detection (isolation forest / seasonal-decomposition on your own time-series data) before reaching for a vendor platform — cheap to build, and it's genuinely your data
- Keep an API-connector pattern open for customers who run Augury/Senseye and want that data surfaced inside your dashboards rather than in a separate tool

**Computer vision**
- Foundation vision model (fine-tuned or few-shot) behind an inference microservice your Quality/SPC screen calls — avoids building a full CV platform from scratch

**Scheduling**
- Start with a constraint solver (OR-Tools is the standard open-source choice) driven by your real-time equipment/work-order data — this alone beats most legacy APS on freshness, even before you add ML-based optimization

**Your own dev workflow**
- Claude Code for backend/API development, Figma MCP (already set up) for design, Claude in Chrome if you need to test against customer-facing flows

---

## 5. Suggested build sequence (concrete next steps)

0. **Decide the tenant isolation strategy (Section 2.1) before finalizing the Phase 1 schema** — this is now the first architectural decision, ahead of any UI or AI work, because every table and every screen downstream inherits it
1. **Finish Phase 1 UI gaps** (empty/error states, drill-downs — already built; settings/admin/tenant-config screens — not yet started, and now core scope per Section 2.3, not optional polish)
2. **Nail the data backbone** — this is the unglamorous but non-negotiable prerequisite for everything in Phase 3-4, and it now includes tenant-scoping (`tenant_id` + RLS) as part of "getting it right," not a separate later task
3. **Ship Phase 2 analytics** using data you already have (no AI dependency, proves the pipeline)
4. **Build the LLM copilot early, even in a narrow form** — e.g. "ask about any work order's status" — because it's your best AI differentiation and doesn't require ML training data to start delivering value; scope its RAG retrieval to be tenant-filtered from the first version (Section 2.5)
5. **Add anomaly-based predictive maintenance** once you have months of real customer time-series data, keeping per-tenant model isolation in mind from the start
6. **Layer in computer vision and advanced scheduling** last — they're the most build-intensive and benefit most from having paying customers' real data to tune against

---

## 6. Sources consulted (Sept 2026 landscape)
- Predictive maintenance platform comparisons: f7i.ai, Monitory, phosailabs
- Computer vision inspection landscape: iFactory, Overview.ai, ifactoryapp
- AI/APS scheduling: Fabrico, Phantasma, phosailabs
- LLM copilots: phosailabs, MES Engineer, SymphonyAI, Microsoft/Aufait
