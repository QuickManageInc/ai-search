# Module 1 — Response Accuracy & Tool Coverage Plan

> **Goal:** Improve the quality and accuracy of Analytics Assistant answers by giving the LLM the right data for each question type — via better tools, contrastive descriptions, prompt rules, and (later) AI-optimized analytics endpoints.  
> **Status:** Phase 4 merchant UI done (31-day clamp, session keep-on-close, usage meta). Deploy/admin optional.  
> **Related:** [Module1_Context_Optimization_Plan.md](./Module1_Context_Optimization_Plan.md) (token slim + tool calling)

---

## Problem (from production usage logs)

| Symptom | Example | Root cause |
|---------|---------|------------|
| Wrong tool | “Items not selling well” → `get_top_selling_items` | No underperforming tool; descriptions not contrastive |
| Soft hallucination risk | “Analyse the problem?” → `toolsCalled: []` | Vague follow-up answered from history only |
| Incomplete coverage | Channel mix, fallers, daypart, tips | Routes exist in `analytics-edge-api` but are not tools |

**Principle:** Prefer **one tool = one merchant question type**, with period-aligned, pre-slimmed, USD payloads. New `analytics-edge-api` endpoints are allowed when portal shapes are wrong for the LLM.

---

## Strategy overview

| Phase | Scope | New analytics-edge-api routes? |
|-------|-------|--------------------------------|
| **1** | Wire existing routes as tools + prompt accuracy | No |
| **2** | AI composition endpoints (`ai/menu-health`, `ai/revenue-diagnosis`) | Yes |
| **3** | Compare periods + optional digest | Yes |
| **4** | Follow-up UX / ops (session persistence, usage meta, deploy) | Optional |

---

## Phase 1 — Wire existing endpoints + prompt rules (quick accuracy win)

**Goal:** Fix wrong-tool and no-tool failures without changing `analytics-edge-api`.

**Expected:** “Items not selling well” → `get_underperforming_items`; analysis follow-ups call at least one tool; clearer tool routing via descriptions.

### P1.1 New tools (existing routes only)

| Tool | Analytics route | Use when |
|------|-----------------|----------|
| `get_underperforming_items` | `items/underperforming` | Weak sellers, low revenue share, “not selling well” |
| `get_item_trends` | `items/trends` | Risers / fallers vs previous window |
| `get_items_by_daypart` | `items/by-time` | Lunch / dinner / daypart mix |
| `get_item_pairs` | `items/affinity` | What sells together (expensive — use sparingly) |
| `get_revenue_mix` | `orders/revenue-breakdown` | Channel / order-type mix (POS vs online, dine-in vs takeout) |
| `get_operations_overview` | `orders/overview` | Completion vs cancellation rates |
| `get_fulfillment` | `orders/fulfillment-metrics` | Prep time / kitchen speed |
| `get_payment_details` | `billing/payment-analytics` | Tips, discounts, card types |
| `get_attendance_summary` | `employees/attendance/summary` | Clock-ins / attendance |

### P1.2 Slim mappers

Add / extend under `ai-edge-api/src/context/slim/`:

- Drop `productId` / internal IDs
- Cap arrays (default 10)
- Convert cents → `*Usd`
- Always include `period: { from, to }` when the API provides dates
- For `items/trends`: return `risers` + `fallers` (top N each), plus previous/current window labels
- For `revenue-breakdown`: keep `byChannel` / `byOrderType` summaries; **drop** full `dailyByChannel` / `dailyByOrderType` grids (token bomb)

### P1.3 Contrastive tool descriptions

Update existing tool copy so the model cannot confuse:

| Tool | Must say |
|------|----------|
| `get_top_selling_items` | Bestsellers by **quantity**. **Do not** use for weak / slow / underperforming items. |
| `get_item_revenue_ranking` | Highest **revenue** items — not “problems”. |
| `get_underperforming_items` | Lowest revenue share / weak sellers / “not selling well”. |
| `get_item_trends` | Rising vs falling vs **prior period** — “what declined / improved”. |

### P1.4 System prompt accuracy rules

In `buildToolSystemPrompt`:

- If the user asks to **analyse**, find a **problem**, ask **why**, or ask for a **cause**, always call at least one tool for the **current date range** (do not answer from conversation history alone).
- Prefer the narrowest tool that matches the question intent (table above).
- Never invent numbers; if data is empty, say so.

### P1.5 Portal

- Add human labels for new tools in `TOOL_LABELS` (`aiAssistant.config.js`) so chips under answers stay readable.
- Optional: suggested prompt “Which items are underperforming?”

### P1.6 Out of scope for Phase 1

- No new `analytics-edge-api` routes
- No `ai/menu-health` or diagnosis composites yet
- No change to rate-limit strategy

### Phase 1 acceptance

- [ ] “Specific items not selling well” → `toolsCalled` includes `get_underperforming_items` (not only bestsellers)
- [ ] “Analyse what the problem?” after a revenue answer → at least one tool call
- [x] `get_revenue_mix` returns period + channel/order-type without daily grids
- [x] `tsc --noEmit` passes in `ai-edge-api`
- [x] Portal `TOOL_LABELS` covers all new tools

---

## Phase 2 — AI composition endpoints (accuracy backbone)

**Goal:** One tool call for vague “what’s wrong with my menu / sales?” questions.

### New routes in `analytics-edge-api`

Compose existing services; return **already LLM-ready** JSON (USD, capped lists, period + previousPeriod). Do **not** change portal composites.

#### `GET /api/v1/analytics/ai/menu-health`

```
period, previousPeriod,
topSelling[], topByRevenue[], underperforming[],
risers[], fallers[],
notes: { totalUniqueItems, menuRevenueUsd }
```

Sources: `items/popular`, `items/revenue`, `items/underperforming`, `items/trends`.

Tool: `get_menu_health`.

#### `GET /api/v1/analytics/ai/revenue-diagnosis`

```
period, previousPeriod,
summary: { revenueUsd, orders, growthPct, aovUsd },
bestDay, worstDay,
byChannel[], byOrderType[],
cancellations: { ratePct, estimatedLostUsd, topStages[] },
peakHours[]
```

Sources: dashboard/revenue/breakdown/cancellation/peak-hours (slimmed once).

Tool: `get_revenue_diagnosis`.

### Phase 2 acceptance

- [x] Vague menu questions prefer `get_menu_health` over 3–4 atomic calls *(prompt + tool descriptions)*
- [x] Vague “why were sales down?” prefers `get_revenue_diagnosis` *(prompt + tool descriptions)*
- [x] README + router entries; contract tests optional
- [x] Live check: “what's wrong with my menu?” → `toolsCalled` includes `get_menu_health`
- [x] Live check: “why were sales down?” → `toolsCalled` includes `get_revenue_diagnosis`
- [x] Latency: AI endpoints use 2 Mongo reads each (not N portal service re-fetches)

---

## Phase 3 — Compare periods + optional digest

#### `GET /api/v1/analytics/ai/compare-periods?storeId&startDate&endDate&compare=previous|wow|mom`

Aligned current vs comparison metrics only (no inventing windows in the LLM). **2 Mongo reads.**

| Mode | Window |
|------|--------|
| `previous` (default) | Equal-length range before `startDate` |
| `wow` | Same range −7 days |
| `mom` | Same range −1 calendar month |

Tool: `get_period_comparison`.

Returns: `period`, `comparePeriod`, revenue/orders/AOV/cancel-rate deltas, top channels.

#### Optional: `GET /api/v1/analytics/ai/digest` — deferred

~500-token morning brief for proactive insights (wait for `ai-worker`).

### Phase 3 acceptance

- [x] Endpoint + router + README
- [x] Tool `get_period_comparison` + prompt routing + `TOOL_LABELS`
- [ ] Live check: “how did we do vs last week?” → `get_period_comparison` with `compare=wow`

---

## Phase 4 — UX / ops

### Merchant UI (done)

1. Keep Redis session on panel close; clear only on “New conversation”
2. SSE `meta` includes `inputTokens` / `outputTokens` / `durationMs` — shown in Usage panel
3. AI date range capped at **31 days from start** (portal + server); session bar shows capped hint
4. Suggested prompts updated (underperforming, menu health, vs last week)

### Still optional / ops

3. Deploy `ai-edge-api` to EKS + gateway `/api/v1/ai/*` — see `ai-edge-api/DEPLOY.md` + Dockerfile + CI workflow
4. Admin usage view from `ai_usage_log` (empty-tools rate, wrong-tool rate)

---

## Implementation order

```
Phase 1  →  existing routes as tools + slim + prompt rules     ✓
Phase 2  →  ai/menu-health + ai/revenue-diagnosis              ✓
Phase 3  →  ai/compare-periods                                 ✓  (digest deferred)
Phase 4  →  merchant UI (session / tokens / 31-day clamp)      ✓
         →  deploy / admin usage (optional)
```

---

## File map (Phase 1)

```
ai-edge-api/src/
  context/slim/
    items.slim.ts          # + underperforming, trends, by-time, affinity
    orders.slim.ts         # + revenue breakdown (no daily grids), overview
    billing.slim.ts        # + payment analytics
    employees.slim.ts      # + attendance summary
    index.ts               # re-exports
  tools/analyticsTools.ts  # new tools + contrastive descriptions
  prompts/analyticsAssistant.ts  # analysis-must-fetch + routing hints

quickmanage-merchant-portal/src/config/aiAssistant.config.js  # TOOL_LABELS
```

No changes required in `analytics-edge-api` for Phase 1.

---

## Tool inventory after Phase 1

**Existing (keep, update descriptions):**  
`get_top_selling_items`, `get_item_revenue_ranking`, `get_revenue_summary`, `get_revenue_by_day`, `get_peak_hours`, `get_hourly_pattern`, `get_payment_overview`, `get_staff_performance`, `get_reservations_summary`, `get_workforce_summary`, `get_cancellation_stats`

**New in Phase 1:**  
`get_underperforming_items`, `get_item_trends`, `get_items_by_daypart`, `get_item_pairs`, `get_revenue_mix`, `get_operations_overview`, `get_fulfillment`, `get_payment_details`, `get_attendance_summary`
