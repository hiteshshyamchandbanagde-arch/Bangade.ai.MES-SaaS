# MES SaaS — Session Handoff

**Date:** 2026-09-05
**Repo:** `hiteshshyamchandbanagde-arch/Bangade.ai.MES-SaaS`
**Latest commit pushed:** `012b4b1` (main)

---

## 🎯 Key Goal for Next Session

**Resolve the two open design questions below. Do NOT start any Figma work until explicitly requested — this is a standing instruction, not a suggestion.**

1. **Can a Production Unit's `production_mode` (Discrete/Process/Hybrid) change after creation, or is it fixed forever?** Still open — not yet decided. Current placeholder assumption in the docs: fixed at configuration time, but this has not been confirmed.
2. **Does Routing/Recipe version promotion (Draft → Active) need a formal approval workflow, or is a simple status field enough for Phase 1?** Still open — not yet decided. Current placeholder assumption in the docs: tenant-level toggle, but this has not been confirmed.

Both questions change the database schema and downstream component design depending on the answer — worth deciding before they get baked into more documents, but that's a documentation/architecture task, not a trigger to move into Figma.

---

## What happened this session

1. **Confirmed and documented multi-tenancy** as a hard product requirement (not aspirational) — Bangade.ai MES-SaaS is sold to multiple manufacturers, Pune Plant is a reference/pilot, not the customer. Added memory entry for this so it's not re-asked.

2. **Updated `docs/mes-ai-architecture-blueprint.md`** with a new §2 "Multi-tenancy — concrete decisions needed now": tenant isolation strategy (recommends RLS + shared schema over schema-per-tenant), plant/site configurability, onboarding flow implications, billing tier mapping, cross-tenant AI/ML data isolation. Downstream sections renumbered 3→6. Pushed as commit `6ef1e60`.

3. **Wrote the full Production Module design doc set** — Core MES, Discrete + Process manufacturing, multi-tenant — at `docs/production-module/`:

   | Doc | What it locks in |
   |---|---|
   | `01-functional-design-document.md` | FR-D1–D9 (Discrete), FR-P1–P9 (Process), FR-S1–S5 (shared), roles, state lists, 3 open questions |
   | `02-technical-design-document.md` | Shared `production-orders` API (not duplicated per type), explicit state-transition endpoints, tenancy enforcement via JWT + RLS |
   | `03-database-model.md` | Full ER diagram, shared-header + type-detail table pattern (`production_order` + `work_order_detail`/`batch_order_detail`), RLS policy SQL |
   | `04-master-data-configuration.md` | Exact tenant setup sequence (Site→Area→Unit→Equipment→Shifts→Users→Reason codes→Products→Routing/Recipe), ownership matrix |
   | `05-high-level-architecture.md` | System context diagram, why Production API is the single write-path |
   | `06-detailed-architecture.md` | State machines (Work Order, Batch Order), sequence diagrams (release flow, edge-fed sensor readings, genealogy trace), OEE/Yield formulas |

   Pushed as commit `012b4b1`.

---

## Current state

- **Architecture docs:** multi-tenancy-aware, Production Module fully specified on paper (functional → technical → data → config → architecture, all cross-linked).
- **Figma file (`0aYcQ2bGx9JD4d5MDLMVWL`):** unchanged this session — still single-tenant-shaped (fixed 6 Discrete lines), pre-dates all of the above. **Nothing in Figma reflects multi-tenancy or Process manufacturing yet.**
- **No code written yet** — this session was 100% design/architecture documentation, zero implementation.

---

## Design decisions worth remembering (so they don't get re-litigated)

- **Discrete vs. Process are NOT two separate products** — one `ProductionOrderCard` component with a `mode` variant, one shared `production_order` DB table with type-specific detail tables. Don't let the next session's Figma work drift into building two disconnected card designs.
- **Data-state-honesty principle** (established earlier, in the original Figma handoff) applies identically to Process mode: a unit not running a batch shows "no batch in progress," never a fabricated gauge value. This must carry into the Process card variant.
- **`recipe_phase.parameter_targets` is JSONB, not fixed columns** — deliberate, because process parameters vary wildly by industry (temp/pressure for chemicals vs. moisture/brix for food). Don't "clean this up" into a rigid schema without re-reading Database Model §3's reasoning.
- **Genealogy is one table (`genealogy_link`) for both Discrete and Process** — resist the urge to split it; the recursive-CTE traversal is what makes forward/backward tracing work identically for both.

---

## On the horizon (after the Figma goal above)

- Settings/tenant-admin screens in Figma — no longer optional polish (per architecture blueprint §2.3), this is core scope for a multi-tenant product
- Plant/site switcher in the Figma header, replacing the hardcoded "Pune Plant — Building 2" label
- Quality/SPC, Maintenance/CMMS, Scheduling, Traceability modules still need their own FDD/TDD/DB-model treatment — Production was done first as the template; the others should follow the same 6-document pattern

---

## Recommended prompt to resume next session

> "Continue the MES SaaS Production Module work. Architecture docs are done and pushed (`docs/production-module/`, commit `012b4b1`). Let's resolve the two open questions in the handoff before anything else. **Do not start Figma work — I'll ask explicitly when I want that.**"

Paste this handoff's content at the start of the next session for full context without re-discovery.

---

## Standing instruction (carries across sessions)

**No Figma work — including planning, component design, or screen layout — until explicitly requested.** This applies even if the conversation naturally arrives at a point where Figma work would seem like the obvious next step (e.g., after the open questions are resolved). Wait for an explicit ask.
