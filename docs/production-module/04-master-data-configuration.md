# Master Data Configuration — Production Module

**Related docs:** [Functional Design](./01-functional-design-document.md) · [Database Model](./03-database-model.md)

**Purpose:** defines what a new tenant must configure before the Production Module is usable, in what order, and who's responsible — directly implementing architecture blueprint §2.2 (plant configurability) and §2.3 (onboarding).

---

## 1. Configuration hierarchy and dependency order

Master data has hard dependencies — you cannot configure a Production Unit before its Area exists, cannot release an order before a Routing/Recipe exists, etc. The required configuration sequence:

```
1. Site                    (Tenant Admin)
2. Area                    (Tenant Admin)
3. Production Unit         (Tenant Admin)  — sets production_mode here
4. Equipment                (Tenant Admin)
5. Shift patterns           (Tenant Admin)
6. Users + role assignment  (Tenant Admin)
7. Reason codes             (Tenant Admin or Plant Manager)
8. Product master            (Planner or Tenant Admin)
9a. Routing (if Discrete)    (Process Engineer)
9b. Recipe (if Process)      (Process Engineer)
10. Quality specs (optional for Phase 1 minimal path) (Quality Engineer)
```

Steps 1–6 are the **minimum viable setup** — without them, no order can be released. Steps 7–10 can technically be deferred, but a tenant realistically needs at least one Product + Routing/Recipe before Production is usable at all.

---

## 2. Configuration items in detail

### 2.1 Site / Area / Production Unit hierarchy
- **Who configures:** Tenant Admin, during onboarding.
- **What's captured:** Site name + timezone; Area name; Production Unit name + `production_mode` (Discrete/Process/Hybrid) + display sequence.
- **Design implication (carried from architecture blueprint §2.2):** this must be a guided setup flow, not a database migration run by engineering — every new tenant does this independently. The Figma design must include a setup wizard, not assume a fixed 6-unit layout (Pune's shape is not universal).
- **Validation rule:** a Production Unit's `production_mode` cannot be changed once a Production Order has been created against it (per FDD open question #1 — fixed at configuration time for Phase 1).

### 2.2 Equipment
- **Who configures:** Tenant Admin or Plant Manager.
- **What's captured:** name, equipment type, and mode-specific fields — `ideal_cycle_time_seconds` for Discrete stations (drives OEE performance calc), `capacity` for Process reactors/tanks (drives yield calc).
- **Note:** equipment can exist without being referenced by any Routing/Recipe yet (e.g., configured ahead of process engineering work).

### 2.3 Shift patterns
- **Who configures:** Tenant Admin.
- **What's captured:** shift name, start/end time, days active. Explicitly **not hardcoded** as "Shift A · 06:00–14:00" (Pune's convention) — every tenant defines their own.
- **Used by:** OEE/Yield calculations (per-shift aggregation), Production Overview header display.

### 2.4 Users and roles
- **Who configures:** Tenant Admin.
- **What's captured:** user identity, role assignment from the fixed role list in FDD §2 (Operator, Supervisor, Plant Manager, Planner, Quality Inspector, Tenant Admin), and scope (which Sites/Areas a user can act on — supports multi-site tenants where a supervisor is scoped to one plant).

### 2.5 Reason codes
- **Who configures:** Tenant Admin or Plant Manager (industry-specific, so shouldn't be hardcoded).
- **What's captured:** two independent lists —
  - **Downtime reason codes** (e.g. "Changeover," "Material shortage," "Breakdown") — used by FR-D3.
  - **Scrap/waive reason codes** — used when skipping a Discrete operation (FR-D5) or waiving a Process phase (FR-P6).
- **Recommendation:** ship a starter template list per industry (discrete-assembly template, batch-process template) that the tenant can edit, rather than a blank list — reduces onboarding friction without hardcoding.

### 2.6 Product master
- **Who configures:** Planner or Tenant Admin.
- **What's captured:** SKU, name, UOM, `default_production_mode`.
- **Note:** ingredients (used in Recipes) are also `product` rows — no separate "ingredient master," to avoid data duplication when a product is sometimes an ingredient in another product's recipe (e.g., a sub-assembly or a semi-finished blend).

### 2.7 Routing master (Discrete tenants/units)
- **Who configures:** Process/Manufacturing Engineer.
- **What's captured:** ordered operations per product, standard time per operation, default equipment.
- **Versioning:** Routings are versioned (`DRAFT` → `ACTIVE` → `DEPRECATED`). Only one `ACTIVE` version per product should be releasable against at a time — enforce at the application layer.

### 2.8 Recipe master (Process tenants/units)
- **Who configures:** Process Engineer.
- **What's captured:** ordered phases per product, target parameters + tolerances per phase (as flexible JSON since parameters vary by industry — a food recipe's parameters look nothing like a chemical batch's), ingredients + target quantities per phase, theoretical yield.
- **Versioning:** same Draft/Active/Deprecated pattern as Routing. FR-P1 requires an Active recipe to release a Batch Order — this is a hard gate, not a warning.
- **Note on parameter flexibility:** because `recipe_phase.parameter_targets` is JSONB rather than a fixed column set, onboarding a new tenant's recipe format doesn't require a schema migration — this is deliberate, since "what parameters matter" is genuinely different per industry (temp/pressure for chemicals, moisture/brix for food, viscosity for coatings).

### 2.9 Quality specs (optional at minimum-viable setup)
- **Who configures:** Quality Engineer.
- **What's captured:** owned by the Quality/SPC module (out of this module's direct scope per FDD §7), but Production references `quality_result.sample_id` — so at minimum, a tenant needs Quality module configured enough to generate sample IDs before in-process quality linkage (FR-P4's out-of-spec flagging still works without it, since that's Recipe-tolerance-based, not Quality-module-based).

---

## 3. Data ownership matrix

| Master data | Owner (business role) | Change frequency | Requires versioning? |
|---|---|---|---|
| Site/Area/Production Unit | Tenant Admin | Rare (plant changes) | No |
| Equipment | Tenant Admin / Plant Manager | Occasional | No |
| Shift patterns | Tenant Admin | Rare | No |
| Users/roles | Tenant Admin | Ongoing | No |
| Reason codes | Plant Manager | Occasional | No |
| Product | Planner | Ongoing (new products) | No |
| Routing | Process Engineer | Occasional (process improvement) | **Yes** |
| Recipe | Process Engineer | Occasional (formulation changes) | **Yes** |

---

## 4. Open items

- Whether Routing/Recipe versioning needs a formal approval workflow (e.g., QA sign-off before a new version goes Active) — likely yes for regulated industries (pharma/food), likely overkill for others. Recommend making this a tenant-level toggle rather than a universal requirement.
- Whether reason code lists should be shareable/importable across a tenant's multiple Sites, or configured independently per Site — current assumption is tenant-wide (not per-site), revisit if a real multi-site customer needs site-specific reason codes.
