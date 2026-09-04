# MES SaaS — Figma Design System Handoff

**File:** MES SaaS - UI/UX Design System
**File key:** `0aYcQ2bGx9JD4d5MDLMVWL`
**Team:** Hitesh Bangade's team — `team::1429967728065648141`
**URL:** https://www.figma.com/design/0aYcQ2bGx9JD4d5MDLMVWL
**Save this file to:** `C:\Users\hites\Bangade MES\Handoff\MES-SaaS-Figma-Handoff.md` (overwrite each session)
**Related architecture doc:** `C:\Users\hites\Bangade MES\docs\mes-ai-architecture-blueprint.md` — also on GitHub at `hiteshshyamchandbanagde-arch/Bangade.ai.MES-SaaS`

---

## 1. File structure

| Page | Node ID | Contents |
|---|---|---|
| Foundations | `0:1` | Color variables + text styles only. No visual frames — blank by design. |
| Components | `1:2` | StatusBadge, SidebarNav, LineStatusCard master components |
| Screens | `1:3` | **30 frames** as of this session — 6 core screens, 6 Line Detail drill-downs, 18 empty/loading/error states |

⚠️ Screens page is now very wide (frames run from x=0 to x≈44760). Use **Shift+1** (zoom to fit) after switching to the Screens page.

---

## 2. Design tokens (unchanged — see Foundations page)

**Color collection:** `VariableCollectionId:1:4` — single "Value" mode.

| Token | Variable ID | Hex |
|---|---|---|
| bg/primary | `VariableID:1:5` | #0B0F14 |
| bg/surface | `VariableID:1:6` | #131922 |
| bg/surface-raised | `VariableID:1:7` | #1C2531 |
| border/subtle | `VariableID:1:8` | #2A3441 |
| text/primary | `VariableID:1:9` | #E8ECF1 |
| text/secondary | `VariableID:1:10` | #8B96A5 |
| text/on-accent | `VariableID:1:11` | #04140F |
| accent/primary | `VariableID:1:12` | #2FB8AC |
| status/running | `VariableID:1:13` | #35C77E |
| status/warning | `VariableID:1:14` | #F0A93B |
| status/critical | `VariableID:1:15` | #E85D5D |
| status/info | `VariableID:1:16` | #4E8CFF |
| status/idle | `VariableID:1:17` | #5A6472 |

Type ramp unchanged: IBM Plex Sans (UI text), IBM Plex Mono (numeric/data text).

---

## 3. Master components (unchanged)

| Component | Node ID | Notes |
|---|---|---|
| StatusBadge (set) | `1:53` | Variants: `Running` (`1:41`), `Warning` (`1:44`), `Critical` (`1:47`), `Idle` (`1:50`) |
| SidebarNav | `1:54` | 6 `NavItem-<Label>` rows. **Active-state highlight is a per-instance fill override, NOT inherited from the master** — see §6 gotcha #2, this bit us hard this session. |
| LineStatusCard | `1:77` | Unchanged from last session |

---

## 4. Core screens (from prior session, unchanged this session)

| Screen | Frame ID | X position |
|---|---|---|
| Dashboard | `1:97` | 0 |
| Scheduling | `5:140` | 1560 |
| Quality / SPC | `7:162` | 3120 |
| Maintenance | `8:249` | 4680 |
| Traceability | `14:206` | 6240 |
| Reports | `37:294` | 7800 |

---

## 5. NEW this session — Line Detail drill-downs

Clicking any of the 6 LineStatusCards on the Dashboard now navigates to a dedicated detail screen: Active Work Order stats, expanded 8hr OEE trend chart, and a two-column Recent Events / Equipment list. Back button returns to Dashboard.

| Line | Detail Frame ID | X position | Status |
|---|---|---|---|
| Line 1 — Assembly | `54:340` | 10920 | Running |
| Line 2 — Welding | `54:377` | 12480 | Warning |
| Line 3 — Assembly | `50:316` | 9360 | Running (built first, slightly different node ID range) |
| Line 4 — Paint | `54:414` | 14040 | Critical — **NO DATA honesty state** (see §7) |
| Line 5 — Packaging | `54:451` | 15600 | Running |
| Line 6 — Inspection | `54:488` | 17160 | Idle — **NO DATA honesty state** |

**Design decision worth knowing:** Line 4 and Line 6's event logs intentionally reuse the *exact* alert text already shown in the Dashboard's Priority Alerts feed (e.g. Line 4's "Robot arm E-stop triggered" at 09:38 appears in both places). This was deliberate — it keeps the two screens internally consistent instead of inventing unrelated demo data.

---

## 6. NEW this session — Empty / Loading / Error states (all 6 screens)

Each of the 6 core screens now has 3 additional state frames:

| Screen | Empty State ID | Error State ID | Loading State ID |
|---|---|---|---|
| Dashboard | `64:460` | `64:490` | `64:730` |
| Scheduling | `66:706` | `66:856` | `66:1412` |
| Quality/SPC | `66:736` | `66:944` | `66:1438` |
| Maintenance | `66:766` | `66:1071` | `66:1465` |
| Traceability | `66:796` | `66:1209` | `66:1491` |
| Reports | `66:826` | `66:1297` | `66:1517` |

**Pattern used for each:**
- **Empty**: centered icon circle + heading + body copy + primary CTA button, screen-specific messaging (e.g. Scheduling: "No work orders scheduled yet")
- **Error**: the screen's *real* content is cloned in at full fidelity, dimmed to 45% opacity, with a banner on top ("Unable to connect to plant network" + last-known-data timestamp + Retry button)
- **Loading**: skeleton blocks (`figma.placeholder = true` shimmer) sized to approximate the real layout, positioned where real content will land

X-position ranges: Empty states 18720–29640, Error states 20280–37440, Loading states 21840–44760 (see full frame list in Figma for exact per-screen x).

---

## 7. Design principle applied: data-state honesty

Carried forward from the original polish pass (Critical/Idle dashboard cards showing "NO DATA" instead of fake trends) into the new screens:
- Line 4 and Line 6 detail screens show `0%`/`—` metrics with an explicit "No trend data available while the line is not running" notice — never fabricated numbers.
- Error states show real *stale* data (dimmed + timestamped), never a blank error page — closer to how a real MES should degrade gracefully.

---

## 8. Prototype connectivity — 204 total reactions

| Wiring | Count | What it does |
|---|---|---|
| Original cross-screen nav (prior session) | 30 | Sidebar nav on the 6 core screens |
| Line Detail cards + back buttons | 12 | Dashboard card → Line Detail → back |
| Sidebar nav on all 24 new screens | 144 | Every screen (Line Detail + Empty/Loading/Error) can navigate to any of the 6 core screens |
| Empty state CTAs | 6 | Button → real populated screen |
| Error state Retry buttons | 6 | Button → real live screen |
| Loading state auto-advance | 6 | `AFTER_TIMEOUT` (1.4s) → real screen, no click needed |

**Verified integrity check (end of session):** all 30 frames on the Screens page have a fully-reaction-wired sidebar. Zero orphaned screens.

**Known gap:** nothing currently transitions *into* a Loading state as part of the click flow — they're demo-able endpoints (jump to them directly in Present mode) but not yet the actual "in-between" step when e.g. clicking Dashboard for the first time. Flagged to Hiitesh, not yet actioned.

---

## 9. Figma Plugin API gotchas learned this session (add to prior list)

10. **`layoutSizingHorizontal/Vertical = 'FILL'` must be set strictly AFTER `appendChild`, never before** — setting it before parenting throws `FILL can only be set on children of auto-layout frames` even though the node will eventually have a valid auto-layout parent. This bit us once this session; the fix is pure ordering.
11. **Cloned component instances do NOT inherit per-screen property overrides from other instances of the same master** — only from the master itself. Concretely: `SidebarNav` instances created via `sidebarMaster.createInstance()` all default to whatever fill state the *master* has (in this file, "Dashboard" highlighted), regardless of what other screens' sidebars show. Any per-screen override (like which nav item is highlighted) must be explicitly reapplied on every new clone — there is no way to "copy the override" from another instance. Caught this via a screenshot QA check after building 24 new screens, fixed with one batch pass rather than 24 individual ones.
12. **`get_metadata`'s reported node `name` for a text layer can be stale relative to its live `characters`.** A text layer named `"Line 3 — Assembly"` in `get_metadata` output was actually displaying `"Line 1 — Assembly"` on canvas — the name was set at creation and never updated when the text was edited. Always verify actual content via `.characters` (e.g. in a `use_figma` script), not the metadata tool's `name` field, before trusting which screen/line a node represents.
13. **`frame.query('[name=X]').first()`** is the fast way to find a specific nested named node (e.g. a button inside a cloned subtree) without manual recursive traversal — used repeatedly this session to find `PrimaryButton`/`RetryButton` inside cloned Empty/Error frames.
14. **`AFTER_TIMEOUT` trigger** works for auto-advancing prototype flows without a click: `{ trigger: { type: 'AFTER_TIMEOUT', timeout: 1400 }, actions: [...] }` set on the frame itself (not a child node) fires when that frame becomes the current view in Presentation mode.
15. **Cloning an entire content subtree (`node.clone()`) for a "stale/dimmed" variant is much faster and more visually consistent than rebuilding from scratch** — used for all 5 non-Dashboard Error states. Trade-off: it's a one-time copy, not a live link — if the real screen's content changes later, the Error state's cloned content will NOT update automatically and must be re-cloned.

Prior gotchas (#1–9) from the original handoff still apply — see git history / previous version of this doc if needed.

---

## 10. What's NOT built yet / good next steps

- **Loading states not wired into the actual click flow** (see §8 known gap)
- **Settings / user profile / plant switcher** screens — still not started
- **Light mode** — still single dark-mode-only variable collection
- **Responsive/tablet variant** — still fixed 1440×900 desktop only
- **Accessibility pass** — contrast ratios still not formally verified
- **Error/Empty/Loading states exist only for the 6 core screens** — Line Detail screens have none of their own yet (no "Line Detail — Error State" etc.)
- **Design QA carried over:** dashboard card sparkline still shows in muted state on Critical/Idle lines rather than being hidden — still an open question from the original handoff

---

## 11. Recommended prompt to resume next session

> "Continue work on the MES SaaS Figma file (`0aYcQ2bGx9JD4d5MDLMVWL`). All 30 screens are prototype-connected as of the last session (204 total reactions, verified zero orphans). [Describe next task — e.g. 'wire Loading states into the actual click flow', 'build a Settings screen', or 'add empty/error states to the Line Detail screens']."

Paste this file's content at the start of the next session so the new context has all IDs without re-discovery.
