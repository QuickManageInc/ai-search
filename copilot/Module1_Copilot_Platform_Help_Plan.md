# Copilot platform help & management how-to plan

Status: **planned** (2026-09-01)  
Scope: **Family 2 — platform / management help** (static workflows, deep links, honest boundaries)  
Explicitly **out of scope for now:** inventory analytics tools, write-from-chat actions  
Related: [Module1_Copilot_Deep_Dive.md](./Module1_Copilot_Deep_Dive.md), [Module1_Copilot_Intent_Tool_Filter_Plan.md](./Module1_Copilot_Intent_Tool_Filter_Plan.md), [Module1_Copilot_Tools_Inventory.md](./Module1_Copilot_Tools_Inventory.md)

---

## Why this doc exists

Analytics tools (~33 store-fact tools) are largely measured and routed via intent filtering. The next high-leverage work is a **deep platform helper** so merchants can ask:

- “How do I download a report?”
- “How do I make next week’s schedule?”
- “How do I check orders in the kitchen?”
- “Where do I export fiscal data?”

These are **static product workflows** — navigation, clicks, and honest limits — not numbers from `analytics-edge-api`.

---

## Current state

### What ships today

| Tool | File | Role |
|------|------|------|
| `get_platform_capabilities` | `ai-edge-api/src/catalog/platformHelp.ts` | Slim index (id, name, path) |
| `get_feature_howto(featureId)` | same | 3–5 steps per feature |

Both tools are in **CORE** (always registered, never behind intent cap).

### Catalog features today (9)

| featureId | Portal path |
|-----------|-------------|
| `reports` | `/dashboard/reports` |
| `team_scheduling` | `/dashboard/team` |
| `team_members` | `/dashboard/team` |
| `team_requests` | `/dashboard/team` |
| `menus` | `/dashboard/menus` |
| `orders_kitchen` | `/dashboard/orders` |
| `reservations` | `/dashboard/reservation` |
| `inventory` | honest-no (data not in analytics yet) |
| `settings` | `/dashboard/settings` |

### Gaps vs real portal

From `quickmanage-merchant-portal/src/config/routes.js` and `navbarItems.config.jsx`:

| Portal area | In catalog? | Merchant asks |
|-------------|-------------|---------------|
| Reports + **Export PDF** | Partial | “How do I download a report?” |
| Orders / Kitchen | Partial | “How do I check orders?” / ticket flow |
| Team / schedule | Partial | “How do I publish shifts?” ✅ golden-tested |
| Automation (Social) | **Missing** | “How do I post to Instagram?” |
| MEV / Fiscal | **Missing** | “Export fiscal data”, RUT report |
| Home dashboard | **Missing** | “What’s on the home page?” |
| Customers | **Missing** | CRM / import |
| Devices / POS | **Missing** | “Pair a device” |
| Profile / language | **Missing** | Account settings |

Also: help is **feature-level** (`reports`), not **task-level** (`reports.export_pdf`, `team.create_shift`).

### Report PDF export (portal, not in how-to yet)

Portal already supports PDF export:

```javascript
// quickmanage-merchant-portal/src/services/analyticsService.js
downloadReportPdf(storeId, startDate, endDate)
// → GET analytics/reports/pdf?storeId&startDate&endDate
```

Reports page has an Export PDF button (`ReportsPage.jsx`). Copilot how-to should document this flow explicitly.

---

## Three tool families (product framing)

From [Module1_Copilot_Deep_Dive.md](./Module1_Copilot_Deep_Dive.md):

| Family | Purpose | Examples |
|--------|---------|----------|
| **1 — Store facts** | Numbers from analytics | `get_revenue_summary`, `get_menu_health`, `get_fulfillment` |
| **2 — Platform help** | How to use QuickManage | capabilities, how-to, task how-to (**this plan**) |
| **3 — Neighborhood / web** | Optional later | citations required; not in scope now |

**Inventory** stays in Family 1 deferred until CDC → `analyticsDB`. Do not add inventory analytics tools yet.

---

## What v1 teaches (Claude Code ToolSearch)

Reference: `v1/src/utils/toolSearch.ts`, `v1/src/tools/ToolSearchTool/`

| v1 pattern | Meaning | QuickManage mapping |
|------------|---------|---------------------|
| **Always loaded** (`alwaysLoad`) | Small set visible every turn | CORE platform help + composites |
| **Deferred** (`shouldDefer` + ToolSearch) | Names only upfront; full schema on search | Future `find_analytics_tools` at ~40+ tools |
| **Keyword search** | `ToolSearch` returns matched schemas for step 2 | `toolCatalog.ts` scaffold exists |
| **`select:ToolName`** | Direct tool pick | Pin rules + intent keywords today |

**Important:** ToolSearch fetches **tool schemas**, not data. It answers “which tool to call,” not “how do I export a PDF.”

For platform help:

- **Do not defer** platform help tools — keep in CORE always (like v1 keeps Brief/SendUserFile).
- Put rich content in **versioned repo files** (`platformHelp.ts`), not LLM memory or a CMS.
- Add a **search layer** over the catalog when task count grows (mini ToolSearch for help only).

---

## Recommended architecture: deep helper

### Three layers

```
Layer 1 — Index (slim)          get_platform_capabilities
Layer 2 — Feature how-to       get_feature_howto(featureId)
Layer 3 — Task how-to (NEW)    get_task_howto(taskId)
                                 OR search_platform_help(query)
```

### Task entry shape (proposed)

Each task is finer-grained than a feature:

```typescript
interface PlatformTask {
  id: string                    // e.g. "reports.export_pdf"
  featureId: string             // parent feature
  name: string
  summary: string
  path: string                  // deep link
  steps: string[]
  tip?: string
  askAbout?: string[]           // keyword hooks for search
  relatedAnalyticsTools?: string[]  // bridge to data tools
  dataAvailable: boolean
  scopes?: string[]             // optional permission hints
}
```

### Task catalog (initial ~15 entries)

| taskId | Typical question | High-level steps |
|--------|------------------|------------------|
| `reports.export_pdf` | Download / export a report | Reports → pick dates → **Export PDF** |
| `reports.change_date_range` | Change report period | Date filter at top; Copilot ~31-day AI cap |
| `team.create_shift` | Add a shift | Team → schedule board → create → **publish** |
| `team.approve_time_off` | Approve time off | Team → Requests → filter → approve/deny |
| `orders.check_live_board` | See live orders | Orders board; Kitchen for production view |
| `orders.update_status` | Move ticket through kitchen | Open ticket → status buttons |
| `menus.publish_changes` | Push menu to POS | Menus → edit → publish |
| `reservations.add_booking` | Add a reservation | Reservation → day view → create booking |
| `automation.create_post` | Social post | Automation → compose → schedule/publish |
| `mev.export_fiscal` | Fiscal export | MEV → transactions → export ZIP/PDF |
| `settings.timezone` | Fix wrong daily totals | Settings → timezone matches store local day |
| `dashboard.overview` | What’s on home | Home dashboard widgets and shortcuts |
| `customers.manage` | CRM basics | Customers list / import (when enabled) |
| `devices.pair` | Pair POS device | Devices settings flow |
| `profile.language` | Change language | Profile / language settings |

### Optional 4th tool: `search_platform_help`

Keyword search over features + tasks (v1 ToolSearch-lite for static content only):

```typescript
search_platform_help({ query: "download report pdf", max_results: 3 })
// → [{ type: "task", id: "reports.export_pdf", ... }, ...]
```

**Flow options:**

1. Search → `get_task_howto` on step 2 (mirrors v1 two-step discovery).
2. Search returns steps inline (simpler for Gemini, one round trip).

**Rule:** All platform help tools stay in **CORE** — never behind intent cap.

---

## Example merchant flows (expected Copilot behavior)

### “How do I download a report?”

1. Call `get_task_howto("reports.export_pdf")` or `search_platform_help("download report")`.
2. Answer with steps + path `/dashboard/reports`.
3. Mention: pick date range → click Export PDF (`downloadReportPdf`).
4. Optional bridge: “Want a summary for that period? Ask me after you set the dates.”

**Do not** call `get_revenue_summary` for a pure how-to ask.

### “How do I make next week’s schedule?”

1. Call `get_task_howto("team.create_shift")` — **not** `get_workforce_summary`.
2. Steps: Team → schedule → create shifts → publish.
3. Optional bridge: “After publishing, ask ‘how many hours did we schedule?’”

Intent guard already routes “how do I schedule shifts?” to platform help (workforce keyword skipped when how-to phrasing detected).

### “Are there orders in the kitchen right now?”

1. **How-to:** Orders / Kitchen boards (live UI) via `orders.check_live_board`.
2. **Honest analytics:** No live queue feed in analytics; offer historical prep via `get_fulfillment`.
3. Do not pretend live queue data exists.

---

## New tools & management ideas (no inventory)

### Family 2 — Platform / management help (build next)

| Tool | Role | Status |
|------|------|--------|
| `get_platform_capabilities` | Slim index | ✅ shipped |
| `get_feature_howto` | Feature-level steps | ✅ shipped |
| `get_task_howto(taskId)` | Task-level deep help | ⬜ planned |
| `search_platform_help(query)` | Keyword router over catalog | ⬜ planned |
| `get_copilot_guide` | Meta: analytics vs how-to split | ⬜ optional |

These **never call** `analytics-edge-api`.

### Family 1 — Store facts (maintain, don’t explode)

At **~33 tools**. Add only when **real aggregated data** exists:

| Candidate | Worth it? | Notes |
|-----------|-----------|-------|
| `get_daily_highlights` | **Yes (product)** | Deterministic 3-bullet feed on login — Toast “For You” style; may be endpoint not LLM tool |
| Fix discount rate in aggregator | **Yes (honesty)** | CDC / `discountUsed` — see domain map P4 |
| Labor cost % / SPLH | **Later** | Tier 3 — no honest aggregate yet |
| Guest repeat vs new | **No until modeled** | Always `available: false` |
| Neighborhood / weather | **Later** | After platform help trusted |

### Management actions — deep-link only

Square/Toast eventually support write-from-chat with HITL. **Do not copy yet.**

| Approach | Example |
|----------|---------|
| ✅ Static how-to + path | “Open **Team → Schedule** to publish shifts” |
| ✅ Future UI button | “Open in QuickManage” from chat bubble |
| ❌ LLM write tools | `create_shift`, `void_order`, `post_to_instagram` without confirmation UI |

---

## Tool management at scale (33 → 50+ tools)

When adding analytics tools, use this tier split:

| Tier | What | Examples |
|------|------|----------|
| **CORE (always)** | Platform help + vague composites | capabilities, howto, task_howto, menu_health, revenue_diagnosis, staff_ops_health, period_comparison, revenue_summary, void_summary |
| **Intent-expanded** | Atomics by keyword | payment_details, peak_hours, staff_performance, menu_health_items |
| **Deferred (40+ tools)** | `find_analytics_tools` meta-tool | Keyword → register schemas on step 2 |

**When to wire ToolSearch:** ~40–50+ tools or frequent cap collisions. Until then: intent + pin rules + `toolCatalog.ts` scaffold.

### Adding any new tool (5-step checklist)

1. `analyticsTools.ts` — definition + handler
2. `toolFocusMap.ts` — category + priority cap order
3. `questionIntent.ts` — keywords + optional pin rules
4. `copilot.ts` — contrastive hints, anti-stacking
5. `toolCatalog.ts` — search metadata for future ToolSearch

Platform help tools skip steps 2–4 category routing but need intent keywords for how-to vs analytics disambiguation.

---

## Suggested build order

### Phase A — Expand static catalog (highest ROI)

1. Add `platformTasks.ts` (or extend `platformHelp.ts`) with ~15 task entries.
2. Wire `get_task_howto(taskId)`.
3. Extend `questionIntent.ts` platform category:
   - Keywords: `export`, `download`, `pdf`, `how do i`, `where do i`
   - Pin: `"download report"` → task how-to, not analytics
4. Document Reports PDF flow using portal `downloadReportPdf`.

### Phase B — Search helper (v1-inspired)

5. Implement `search_platform_help(query)` over features + tasks + `askAbout`.
6. Prompt rule: how-to asks → platform tools first; analytics only if they also want numbers.

### Phase C — Product surfaces

7. `get_daily_highlights` — server endpoint + portal “For You” cards.
8. Suggested prompts mix analytics + how-to.
9. Global FAB copy: “Ask about your numbers **or how to use QuickManage**.”

### Phase D — Analytics additions (data-dependent only)

10. Discount honesty in aggregator (verify prod).
11. Employee/labor tools only when aggregates exist.

---

## Golden test questions (add when shipped)

| Question | Expected tool | Notes |
|----------|---------------|-------|
| How do I download a report? | `get_task_howto` or `search_platform_help` | Must mention Export PDF |
| How do I check orders? | `get_task_howto("orders.check_live_board")` | Not fulfillment for pure how-to |
| How do I export fiscal data? | `get_task_howto("mev.export_fiscal")` | New catalog entry |
| How do I schedule shifts? | `get_feature_howto` or task | ✅ already passes (`a93ede30`) |
| How do I post to social? | `get_task_howto("automation.create_post")` | Flag-gated feature |

Record in [Module1_Copilot_Tool_Test_Questions.md](./Module1_Copilot_Tool_Test_Questions.md).

---

## What not to do now

- Inventory tools or second API client to `inventory-edge-api`
- Write-from-chat (create shift, void order, post to IG) without HITL
- RAG over random docs — keep **one versioned catalog file**
- ToolSearch for analytics until ~40+ tools or painful cap collisions
- Huge platform capabilities payload — slim index + pull steps on demand

---

## Code touchpoints

| Area | Path |
|------|------|
| Platform catalog (today) | `ai-edge-api/src/catalog/platformHelp.ts` |
| Platform tasks (proposed) | `ai-edge-api/src/catalog/platformTasks.ts` |
| Tool registry | `ai-edge-api/src/tools/analyticsTools.ts` |
| Intent / pins | `ai-edge-api/src/tools/questionIntent.ts` |
| Tool filter / CORE | `ai-edge-api/src/tools/toolFocusMap.ts` |
| ToolSearch scaffold | `ai-edge-api/src/tools/toolCatalog.ts` |
| System prompt | `ai-edge-api/src/prompts/copilot.ts` |
| Portal routes | `quickmanage-merchant-portal/src/config/routes.js` |
| Portal navbar | `quickmanage-merchant-portal/src/config/navbarItems.config.jsx` |
| Report PDF export | `quickmanage-merchant-portal/src/services/analyticsService.js` |
| v1 ToolSearch reference | `v1/src/utils/toolSearch.ts`, `v1/src/tools/ToolSearchTool/` |

---

## Open questions

1. Which portal areas matter most for first merchants? (Reports PDF, Team, Orders, Automation, MEV?)
2. One-shot search+steps vs two-step search → task howto?
3. Should `inventory` stay in feature catalog as honest-no only, or hide until CDC ships?
4. Automation / MEV behind feature flags — include in catalog with `dataAvailable: false` when flag off?
