# Module 1 — Copilot Domain Metrics Map

> **Purpose:** Data-first view of what each Copilot domain can answer today — across `analytics-service` aggregations, `analytics-edge-api` routes, and `ai-edge-api` tools — plus where to add metrics (especially staff) before coding new tools.  
> **Related:** [Module1_Copilot_Tools_Inventory.md](./Module1_Copilot_Tools_Inventory.md), [Module1_Copilot_Deep_Dive.md](./Module1_Copilot_Deep_Dive.md), [analytics-service/docs/employee-analytics-rollout-plan.md](../../analytics-service/docs/employee-analytics-rollout-plan.md)

---

## 0. How to read this

Three layers:

| Layer | Role |
|-------|------|
| **`analytics-service`** | Aggregates facts → summary collections (`daily_store_*`, `store_workforce_summary`, …) |
| **`analytics-edge-api`** | Cheap read APIs over those summaries |
| **`ai-edge-api` tools** | Subset of edge routes the LLM can call |

A lot of **staff** data is already at layers 1–2 and never reaches the copilot.

**Ground rule:** systems own facts; the LLM owns wording. Prefer wiring existing edge routes + small aggregator upgrades over inventing new domains (inventory stays deferred until CDC → `analyticsDB`).

---

## 1. Menu — strong

| Already aggregated | Edge routes | AI tools |
|---|---|---|
| `daily_store_items`, item pairs | popular, underperforming, revenue, trends, by-time, affinity | All 7 menu tools + `get_menu_health` |

**Verdict:** Enough for Square-style menu Q&A. No urgent new aggregates.

**Optional later:** modifier-level / category rollups if merchants ask a lot about categories.

---

## 2. Revenue / sales / orders — strong

| Already aggregated | Edge | AI tools |
|---|---|---|
| `daily_store_summary` (orders, revenue, channel, orderType, hourly, cancels, fulfillment) | dashboard, revenue, breakdown, overview, peaks, heatmap, fulfillment, cancellations, AI composites | Full set + `get_revenue_diagnosis` / `get_period_comparison` |

**Verdict:** Core sales story is covered.

**Highest-ROI add here:** dedicated voids tool (`get_void_summary`) — `billing.void` already exists in `daily_store_summary`; daily trend needs a small edge repo change (include `billing.void` in daily billing query), not new CDC.

---

## 3. Payments — good dollars, weak honesty on discounts

| In `daily_store_summary.billing` | Status |
|---|---|
| cash / card / debit / online + counts | Real |
| tips + tipCount (`tipUsed` 0/1 per order) | Real |
| discounts **total $ only** | Real |
| void | Real |
| byCardType | Real |

### Gap in `analytics-service`

Tips get both amount and `tipUsed` (order had a tip). Discounts only get `discountTotal` in `billingTotals.ts` — **no `discountUsed`**.

Edge then fabricates:

```ts
discountCount = totalDiscounts > 0 ? totalTransactions : 0
```

So the LLM can narrate a fake discount rate as fact today via `get_payment_details`.

### Useful aggregates to add (payments)

| New field | Where | Why |
|---|---|---|
| `billing.discountCount` (sum of `discountUsed` 0/1, same pattern as tips) | `billingTotals` → `daily_store_summary` (+ backfill) | Real discount rate / average |
| Daily voids in daily billing query | edge repo only (field already on summary) | Powers `get_void_summary` daily trend |

Until `discountCount` exists: **A4 honest-no** on rate/avg; keep total discounted $ available when real.

---

## 4. Reservations / guests — thin but honest

| Aggregated | AI tool |
|---|---|
| `daily_store_reservations_summary` | `get_reservations_summary` only |

No guest identity across visits → no repeat vs new. Do **not** invent a tool; use honest-no if asked.

---

## 5. Staff / employee — biggest Copilot gap

Staff is where “add more metrics” pays off. Much of it is **already built** in analytics — just not wired to the LLM.

### 5.1 What the copilot has today (3 tools)

| Tool | Backing data | Answers |
|---|---|---|
| `get_staff_performance` | `daily_store_employee_performance` | Top staff by revenue, AOV, completion/cancel %, avg prep time |
| `get_workforce_summary` | `store_workforce_summary` | Headcount by status/type, new hires 30/90d, work-permit expiry windows |
| `get_attendance_summary` | `daily_store_attendance_summary` | Clock-in/out/punch counts + net delta for range |

### 5.2 What edge already has but AI does **not** call

From `analytics-edge-api` router (`employees/*` + `composite/employees`):

| Edge route | Merchant questions it unlocks |
|---|---|
| `employees/scheduling/summary` | “How many hours did we schedule this week?” |
| `employees/scheduling/hours-trend` | “Are we scheduling more or less over time?” |
| `employees/scheduling/open-shifts` | “How many open / claimed shifts?” |
| `employees/workforce/headcount-trend` | “Is headcount growing?” |
| `employees/attendance/trend` | “Attendance trend week by week?” |
| `employees/time-off/summary` + `trend` | “Who’s out / time-off volume?” |
| `employees/swaps/summary` | “Shift swap volume?” |
| `employees/cannot-work/summary` | “Cannot-work declarations?” |
| `employees/compliance/documents` (+ expiring) | “Docs / permits about to expire?” |

**First move for staff:** wire existing edge routes as tools (with slim DTOs), not invent new CDC fields.

### 5.3 What’s inside employee performance today

From `dailyStoreEmployeePerformance` + edge:

- `totalOrders`, `totalRevenue`
- completed / cancelled
- avg processing (prep) time

**Not** there yet (would need new aggregates or joins):

| Metric | Useful? | Hard? |
|---|---|---|
| Tips attributed to employee | High (Square-like) | Medium — employeeId on tip path / join billing↔order |
| Orders per hour / sales per labor hour | High | Needs punch or scheduled hours join |
| Bottom performers (not only top) | High for “who’s struggling” | Easy — same collection, different sort / slim |
| Channel mix per employee | Medium | Easy if channel is on the performance fact path |
| Items sold per employee | Medium | New item×employee rollup |
| No-show / late vs schedule | High | Hard — schedule + attendance join |

---

## 6. Recommendation — should we add more staff metrics?

**Yes.** Menu/revenue are already dense; staff is where the copilot feels thin vs Square (“labor”) and Toast.

Do it in this order:

### Tier 1 — no new `analytics-service` work (wire what’s live) — ✅ Done

Added atomic AI tools: `get_scheduling_summary`, `get_time_off_summary`, `get_compliance_status`, `get_attendance_trend`, `get_swaps_summary` (with slim mappers + prompt rules). Composite `get_staff_ops_health` shipped later as P3.

### Tier 2 — small aggregator / edge upgrades (worth doing) — ✅ Done

| Change | Where | Why | Status |
|---|---|---|---|
| `discountUsed` (0/1) like tips | `billingTotals` → `daily_store_summary`/`daily_platform_summary` (+ `billingAdjustment`, + backfill) | Fixes fake discount rate; unlocks honest payment answers | ✅ Real `discountCount`/`discountRate`/`averageDiscount` now flow through `billing.repository.ts` → `billing.service.ts` → `slimPaymentAnalytics`. **Requires running the backfill script** to correct historical `daily_store_summary`/`daily_platform_summary` docs — new events compute it correctly going forward regardless. |
| Bottom-N staff in performance response | edge and/or slim | Same data, better questions (“who underperformed”) | ✅ No edge change needed — `orders/employee-performance` already returns all employees; `slimEmployeePerformance` now returns both `topStaff` and `bottomStaff` (bottom 5, only when there's a distinct tail beyond top 10) |
| Per-employee tip $ | performance aggregator + billing join | High merchant value; slightly more work | ✅ `billingAdjustment.ts` now attributes tip deltas to `daily_store_employee_performance` by the order's `employeeId`; edge + slim expose `tipsUsd`/`avgTipUsd`/`tipCount` per employee. Backfill updated with the same billing join. |

### Tier 3 — deferred / undone (not building now)

> **Status: intentionally not started.** Revisit only if product explicitly asks.

| Candidate | Why deferred | What it would need |
|---|---|---|
| Labor cost / sales-per-labor-hour | No wage data in analytics; punches are counts not hours worked | Payroll/`hourlyRate` source and/or punch→duration pairing; new daily labor fields |
| Late / no-show vs published shifts | Hard join + product rules (thresholds, overnight, open shifts) | New `daily_store_shift_adherence` (or similar) + privacy policy for named individuals |
| Item × employee affinity | Useful but lower ROI than voids/platform help | New `daily_store_employee_items` rollup + capped edge/AI tool |

Do **not** start Tier 3 until product asks. Finish board (P1/P3/P6) is done without it.

---

## 7. Suggested Copilot tool surface by section (after this round)

| Section | Keep | Status | Honest-no / don’t promise |
|---|---|---|---|
| **Menu** | All current | ✅ | — |
| **Revenue / orders** | All current + `get_void_summary` | ✅ | — |
| **Payments** | overview + details (real discount rate/avg after Tier 2 + backfill) | ✅ | — |
| **Staff sales** | `get_staff_performance` (top + bottom + tips) | ✅ | — |
| **Staff ops (HR/labor)** | workforce + attendance + scheduling/time-off/compliance + `get_staff_ops_health` | ✅ | Labor cost %, late/no-show (Tier 3) |
| **Guests** | reservations | ✅ | Repeat guests |
| **Platform** | `get_platform_capabilities`, `get_feature_howto` | ✅ | — |
| **Inventory** | — | Deferred to CDC | All inventory Q&A |

Rough tool count ≈ **30–32** — still under the ~40–50 ceiling before tool-retrieval / semantic-layer refactor.

---

## 8. Ranked build order (metrics + tools, no inventory)

| Priority | Work | New aggregator? | Status |
|---|---|---|---|
| **P0** | ~~A4 honest-no on fake discount rate/avg~~ — superseded by P4 | No | ✅ superseded |
| **P1** | `get_void_summary` + `ai/void-summary` (+ daily void field in edge query) | No | ✅ Done |
| **P2** | Wire staff Tier 1 tools + slim mappers from existing `employees/*` | No | ✅ Done |
| **P3** | `ai/staff-ops` composition endpoint + `get_staff_ops_health` | No (compose reads) | ✅ Done |
| **P4** | `discountUsed` in `analytics-service` + backfill | **Yes** | ✅ Done |
| **P5** | Bottom performers + employee tips | Edge only / aggregator | ✅ Done |
| **P6** | Platform-help tools (static catalog) | No | ✅ Done |
| Later | Neighborhood, inventory via CDC | Separate programs | Open |
| **Tier 3** | Labor cost / SPLH, late/no-show, item×employee | Yes | **Deferred / undone** |

---

## 9. Open questions before coding staff Tier 1

1. One composite `get_staff_ops_health` vs several atomic staff tools first? (Composite reduces wrong-tool risk for vague “how’s labor?” questions.)    => started with several atomic staff tools first; composite shipped as P3
2. Should compliance/expiring be a chat tool **and** a “For You” card source (Phase B)?   
3. Confirm historical coverage of `billing.void` on older `daily_store_summary` rows before promising daily void trends without a caveat. ok — void tool includes a note that older days may show 0
