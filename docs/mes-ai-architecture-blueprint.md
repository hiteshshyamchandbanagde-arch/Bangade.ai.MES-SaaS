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

## 2. AI capability map, mapped to your roadmap

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

## 3. Concrete tool/stack recommendations for building this yourself

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

## 4. Suggested build sequence (concrete next steps)

1. **Finish Phase 1 UI gaps** (empty/error states, drill-downs, settings — already identified in the design handoff)
2. **Nail the data backbone** — this is the unglamorous but non-negotiable prerequisite for everything in Phase 3-4
3. **Ship Phase 2 analytics** using data you already have (no AI dependency, proves the pipeline)
4. **Build the LLM copilot early, even in a narrow form** — e.g. "ask about any work order's status" — because it's your best AI differentiation and doesn't require ML training data to start delivering value
5. **Add anomaly-based predictive maintenance** once you have months of real customer time-series data
6. **Layer in computer vision and advanced scheduling** last — they're the most build-intensive and benefit most from having paying customers' real data to tune against

---

## 5. Sources consulted (Sept 2026 landscape)
- Predictive maintenance platform comparisons: f7i.ai, Monitory, phosailabs
- Computer vision inspection landscape: iFactory, Overview.ai, ifactoryapp
- AI/APS scheduling: Fabrico, Phantasma, phosailabs
- LLM copilots: phosailabs, MES Engineer, SymphonyAI, Microsoft/Aufait
