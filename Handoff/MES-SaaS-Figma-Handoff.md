# MES SaaS — Figma Design System Handoff

**File:** MES SaaS - UI/UX Design System
**File key:** `0aYcQ2bGx9JD4d5MDLMVWL`
**Team:** Hitesh Bangade's team — `team::1429967728065648141`
**URL:** https://www.figma.com/design/0aYcQ2bGx9JD4d5MDLMVWL

---

## 1. File structure

| Page | Node ID | Contents |
|---|---|---|
| Foundations | `0:1` | Color variables + text styles only. **No visual frames** — this page looks blank by design. |
| Components | `1:2` | StatusBadge, SidebarNav, LineStatusCard master components |
| Screens | `1:3` | 6 screens, laid out left-to-right at y=0, each 1440×900, spaced ~1560px apart on x |

⚠️ Screens are spread horizontally — use **Shift+1** (zoom to fit) after switching to the Screens page or you'll only see empty canvas.

---

## 2. Design tokens (Foundations page)

**Color collection:** `VariableCollectionId:1:4` (single "Value" mode — no light/dark theming yet)

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

**Type ramp** (named text styles, prefix `MES/`): Display, Heading 1, Heading 2, Body, Body Medium, Caption, Data XL, Data Large, Data Medium, Data Small.
- UI text → **IBM Plex Sans** (Regular/Medium/SemiBold)
- Numeric/data text (metrics, timestamps, IDs) → **IBM Plex Mono** (Regular/Medium/SemiBold)

---

## 3. Master components (Components page)

| Component | Node ID | Notes |
|---|---|---|
| StatusBadge (set) | `1:53` | Variants: `Status=Running` (`1:41`), `Warning` (`1:44`), `Critical` (`1:47`), `Idle` (`1:50`) |
| SidebarNav | `1:54` | 6 nav rows named `NavItem-<Label>`. Each row: `children[0]` = icon vector, `children[1]` = label text. Editing this master propagates to all screen instances. |
| LineStatusCard | `1:77` | Header (name + badge instance), WO line, OEE metric row (`OEERow`), `TrendRow` (sparkline, added later) |

**Editing a master propagates to all instances automatically** — this is how the icon set, sparkline, and logo gradient rolled out across all 6 screens in one edit each. Per-instance overrides (active nav-item highlight color per screen) must be **re-applied after** any structural edit to the master, since they're tied to node identity.

---

## 4. Screens (Screens page)

| Screen | Frame ID | X position |
|---|---|---|
| Dashboard | `1:97` | 0 |
| Scheduling | `5:140` | 1560 |
| Quality / SPC | `7:162` | 3120 |
| Maintenance | `8:249` | 4680 |
| Traceability | `14:206` | 6240 |
| Reports | `37:294` | 7800 |

All 6 are cross-linked with real prototype click reactions (30 total) — every sidebar nav item on every screen navigates to the correct target with a Smart Animate transition. Reaction shape used:

```js
await row.setReactionsAsync([{
  trigger: { type: "ON_CLICK" },
  actions: [{
    type: "NODE",
    destinationId: targetScreenId,
    navigation: "NAVIGATE",
    transition: { type: "SMART_ANIMATE", easing: { type: "EASE_OUT" }, duration: 0.3 },
    preserveScrollPosition: false
  }]
}]);
```
Set directly on the nav-item row node (nested nodes work fine — doesn't need to be a top-level frame).

**Nav label → destination map** (reuse this if adding a 7th screen):
```js
{
  "Dashboard": "1:97", "Scheduling": "5:140", "Quality / SPC": "7:162",
  "Maintenance": "8:249", "Traceability": "14:206", "Reports": "37:294"
}
```

---

## 5. Polish pass applied (session 2)

- Elevation drop shadows on all cards/panels (cornerRadius 10, `{color:{0,0,0,0.35}, offset:{0,6}, radius:18, spread:-2}`)
- Teal→blue gradient + glow on logo mark and user avatars
- Distinct outline icons per nav item (grid/calendar/check/hex-nut/route/bars) replacing plain dots
- 8-hour trend sparkline on every LineStatusCard (auto-propagated from master)
- Subtle radial glow background layer on every screen (`GRADIENT_RADIAL`, ~10% opacity teal, top-left)
- Data-state honesty fix: Critical/Idle dashboard cards show greyed "NO DATA"/"IDLE" sparklines instead of fake trend bars

---

## 6. Figma Plugin API gotchas learned this project

1. **Vector paths need space-delimited coordinates, not commas**, and **arc commands (`a`/`A`) reliably fail to parse**. Use only `M`, `L`, `Z` with absolute, space-separated coords: `"M1 1 L6 1 L6 6 Z"`. This is how all connector lines and nav icons were built.
2. **Never call `.resize()` on a vector *after* setting `vectorPaths`** — it remaps/distorts the path into the new bounding box instead of just repositioning. Let Figma auto-size from the path data; only set `x`/`y` if needed.
3. **`combineAsVariants()` requires children created via `figma.createComponent()`**, not `figma.createAutoLayout()` frames. Build each variant with `createComponent()`, then manually set `layoutMode`, padding, sizing modes, etc., before combining.
4. **`layoutMode: "WRAP"` is not supported** by the plugin API (only `NONE | HORIZONTAL | VERTICAL | GRID`). Build multi-row grids as nested `HORIZONTAL` rows inside a `VERTICAL` parent.
5. **Table/grid cells**: if you `resize(width, 1)` an auto-layout cell *before* appending text, the text gets clipped (frames clip content by default). Always append content first, then `resize(width, cell.height)` to lock width while keeping the natural hugged height.
6. **`figma.createAutoLayout()` frames default to a white fill.** Always explicitly set `node.fills = []` on non-background wrapper frames, or you'll get a stray white box (bit us on the sidebar Logo row).
7. **Font loading is per-call**, not persistent across `use_figma` invocations — `await figma.loadFontAsync(...)` at the top of every call that touches text, even if you loaded it in a previous call.
8. **Gradient fills** (`GRADIENT_LINEAR` / `GRADIENT_RADIAL`) use a 2×3 `gradientTransform` matrix + `gradientStops` array; approximate values are fine for decorative glows — no need for pixel-perfect math.
9. Prior established gotchas (still true): variable binding needs `setBoundVariableForPaint()`, not raw variable ID strings; `combineAsVariants` component-only rule (see #3).

---

## 7. What's NOT built yet / good next steps

- **Empty, loading, and error states** for each screen (only happy-path + one partial critical/idle state exists)
- **Responsive/tablet variant** (screens are fixed 1440×900 desktop only — no mobile/tablet sizing considered)
- **Settings / user profile / plant switcher** screens
- **Detail/drill-down views** (e.g. clicking a LineStatusCard doesn't currently go anywhere — only sidebar nav is wired)
- **Light mode** (single dark-mode-only variable collection currently)
- **Accessibility pass**: verify color contrast ratios formally, especially status colors on dark surfaces
- **Design QA**: dashboard card sparkline still shows on Critical/Idle lines in a muted state — consider whether that's the right pattern or if it should be hidden entirely

---

## 8. Recommended prompt to resume next session

> "Continue work on the MES SaaS Figma file (`0aYcQ2bGx9JD4d5MDLMVWL`). [Describe next task — e.g. 'add empty/error states to Dashboard' or 'build a Settings screen matching the existing design system']."

Paste this file's content (or link to it) at the start of the next session so the new context has all IDs without re-discovery.
