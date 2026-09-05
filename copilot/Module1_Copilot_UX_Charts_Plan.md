# Module 1 — Copilot UX + inline charts

> **Status:** Phase 1 ✅ · Phase 2 ✅ · Phase 3 planned  
> **Portal:** `quickmanage-merchant-portal`  
> **API:** `ai-edge-api` (chart SSE payload in Phase 2)

---

## Locked decisions

| Question | Decision |
|----------|----------|
| Panel | **Compact floating panel** (420px, max ~720px height — original size; AppBar icon, not full-height drawer) |
| Trigger | **Icon-only** in AppBar (top right, left of profile) |
| Charts (Phase 2) | **Inline only** in the message — no “Open in Reports” yet |
| Chart data source | **Backend chart payload** on SSE (not client re-fetch) |
| Navigation | Drawer **stays open** across route changes (session sticky) |

---

## Phase 1 — Shell UX (this sprint)

### Goals

1. Remove fixed bottom-right FAB.
2. Add sparkles icon button in `AppBar` (between fiscal badges and profile menu).
3. Open Copilot as a **compact floating panel** (420px desktop, same max-height as v1 FAB panel).
4. Lift `isOpen` into `CopilotShellContext` so:
   - AppBar can toggle
   - For You (`qm:copilot-ask`) can open
   - State survives page navigation inside `MainLayout`
5. Keep existing chat, suggestions, SSE, and session clear behavior.

### Layout

```
[ Sidebar ] [ AppBar: Search ··· Fiscal · Copilot(icon) · Profile ]
            [ page content …………………… ]
            [ compact panel bottom-right when open (420px) ]
```

- Desktop: fixed panel bottom-right, 420px wide, max-height `min(720px, 90vh)`, rounded card.
- Mobile: bottom sheet, max-height 90vh.
- No full-height drawer; no bottom FAB.

### Files

| File | Change |
|------|--------|
| `contexts/CopilotShellContext.jsx` | `isOpen`, `openCopilot`, `closeCopilot`, `toggleCopilot`, optional `turnCount` |
| `components/AppBar.jsx` | Icon-only Copilot trigger + badge |
| `components/ai/AIAssistantPanel.jsx` | Compact panel chrome; consume shell open state |
| `layouts/MainLayout.jsx` | Keep `CopilotHost`; no FAB host change beyond panel |

### Out of scope for Phase 1

- Chart rendering
- SSE schema changes
- Pin-to-dashboard
- Suggested-prompt redesign

---

## Phase 2 — Inline charts (backend payload)

### Contract (draft)

On SSE `done` (or a dedicated `chart` event before `done`), attach optional:

```json
{
  "chart": {
    "type": "bar" | "line" | "donut" | "area",
    "title": "Top items",
    "categories": ["Burger", "Pasta"],
    "series": [{ "name": "Revenue", "data": [120, 80] }],
    "unit": "currency" | "count" | "percent",
    "sourceTools": ["get_menu_health"]
  }
}
```

Built in `ai-edge-api` from **tool results already fetched** (systems own facts — same rule as answers).

### Portal

- Extend message model with optional `chart`.
- Render compact ApexCharts under assistant text after stream completes (reuse Reports chart patterns).
- Chart only when series exist; scalar answers stay text / KPI chips later.

### Allowlist (first tools)

| Tool | Chart |
|------|-------|
| Revenue / daily or hourly series | line / area |
| Top items / categories | horizontal bar |
| Channel or payment mix | donut |

### Explicitly deferred

- “Open in Reports” deep link
- Pin chart to Home (Square-style widgets)
- Client re-fetch of analytics as chart source

### Phase 2.1 — Chart UX polish ✅

- **Expand modal** — maximize icon opens full-size chart (`<dialog>`)
- **Chart header** — title + period subtitle from `chart.period`
- **Skip zero charts** — no chart when all series values are 0 (backend + portal guard)
- **Filter inactive days** — daily series omit days with $0 revenue and 0 orders (sparse ranges)

### Phase 2.2 — Chart | Table + Copy CSV ✅

- **Chart | Table toggle** on the inline card and expand modal
- **Copy CSV** from `categories` + `series` (clipboard; raw numbers)

---

## Phase 3 — Later UX polish (not started)

- Desktop label “Copilot” behind a flag
- Soft push layout (shrink content when drawer open)
- Keyboard shortcut (e.g. `⌘J`)
- Chart pin to dashboard

---

## Progress

- [x] Decisions locked (drawer, icon-only, inline charts, backend payload, sticky open)
- [x] Phase 1: AppBar trigger + compact panel (no FAB)
- [x] Phase 2: SSE `chart` + Apex inline render
- [x] Phase 2.1: expand modal, chart header, skip zero charts
- [x] Phase 2.2: Chart | Table toggle + Copy CSV
- [ ] Phase 3: polish

---

## Related

- Launch plan: [Module1_Copilot_Prod_Launch_Plan.md](./Module1_Copilot_Prod_Launch_Plan.md)
- Deploy handoff: [Module1_Copilot_Deploy_Handoff.md](./Module1_Copilot_Deploy_Handoff.md)
- Deep dive (Toast/Square): [Module1_Copilot_Deep_Dive.md](./Module1_Copilot_Deep_Dive.md)
