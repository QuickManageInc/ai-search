# Module 1 — Context Optimization Plan

> **Problem:** Analytics Assistant prompts cost ~20k input tokens per question because `ContextBuilder` dumps raw composite API responses (`JSON.stringify(..., null, 2)`) into the LLM prompt.  
> **Goal:** Cut typical input tokens to **500–2,000** without changing analytics-edge-api response shapes used by the merchant portal.  
> **Status:** Phase B implemented — tool calling agent loop. Measure `inputTokens` + `toolsCalled` in `ai_usage_log`.

---

## Root causes

1. **Wrong fetch for the question** — e.g. "top selling items" with `context: ["revenue"]` loads `composite/dashboard` (no menu data).
2. **Composites are portal-sized** — `composite/dashboard` fans out to 5 endpoints; `composite/employees` to 12. Heatmaps include full 7×24 grids.
3. **Wasteful prompt serialization** — pretty-printed raw API JSON with IDs, nested stages, daily arrays for the full date range.

---

## Strategy overview

| Layer | Role |
|-------|------|
| **Slim DTOs** | Shape data for the LLM (small, English keys, capped arrays, cents → dollars) |
| **Tool calling** | Fetch only what the question needs (agent picks tools) |

Phases A and B combine both. Phase C adds analytics endpoints only if atomic routes are insufficient.

**Rule:** Keep composite endpoints for the Reports UI. The LLM path uses **atomic routes** + **slim mappers** in `ai-edge-api` only.

---

## Phase A — Slim DTOs + atomic fetches (quick win)

**Goal:** Keep category chips UX; stop sending fat composites; slim every payload before the prompt.

**Expected token drop:** ~20k → **2–4k** typical.

### A1. Atomic fetches per category

Replace composite paths in `ContextBuilder`:

| Category | Before (composite) | After (atomic) |
|----------|--------------------|----------------|
| `revenue` | `composite/dashboard` (5 endpoints) | `orders/dashboard-summary` + `orders/revenue` (range-respecting daily series) |
| `payment` | `composite/billing` | `billing/overview` |
| `operations` | `composite/operations` (4 endpoints) | `orders/overview` + `orders/peak-hours` + `orders/fulfillment-metrics` |
| `menu` | `composite/menu` (6 endpoints) | `items/popular` + `items/revenue` (`limit=10`) |
| `staff` | `orders/employee-performance` | same + slim (top 10) |
| `workforce` | `composite/employees` (12 endpoints) | `employees/workforce/summary` + `employees/attendance/summary` |

### A2. Slim mappers in `ai-edge-api`

Location: `ai-edge-api/src/context/slim/`

| Mapper | Drop | Keep |
|--------|------|------|
| Revenue | order stage grids, per-day stage counts, processing zeros | period summary, growth %, last 7 daily points, current activity |
| Trends | excess days beyond window | `dailySales` capped, `monthlyGrowth` |
| Payment | full daily billings if long range | totals, collection rate, payment method split (USD), last 7 days |
| Operations | daily stage grids | overview summary, peak hours, busiest day, fulfillment summary |
| Menu | `productId`, affinity/by-time/trends | top 10 items with name/sold/revenueUsd |
| Staff | `employeeId`, full list | top 10 by revenue with name + rates |
| Workforce | 10 unused employee sub-APIs | headcount + attendance summary only |

Money: convert cents → dollars in the mapper so the prompt can say amounts plainly.

### A3. Compact JSON in the prompt

- Change `JSON.stringify(x, null, 2)` → `JSON.stringify(x)` (no pretty-print).
- System prompt notes: amounts in the context are already in dollars.

### A4. Out of scope for Phase A

- No tool calling yet (portal still sends `context[]`).
- No new analytics-edge-api endpoints.
- No portal UI changes required (optional: suggested prompts already set context).

### Acceptance criteria

- [ ] Same question + date range uses fewer input tokens (log `inputTokens` in `ai_usage_log`).
- [ ] Answers still ground in real numbers (spot-check revenue / top items / staff).
- [x] `tsc --noEmit` passes in `ai-edge-api`.
- [x] Cache key behavior unchanged (still storeId + question + dateRange + context).
- [x] Slim mappers live under `ai-edge-api/src/context/slim/`.
- [x] `ContextBuilder` fetches atomic routes (no composites on the LLM path).
- [x] Prompt uses compact `JSON.stringify` and USD fields.

---

## Phase B — Tool calling (main architecture)

**Goal:** LLM decides what to fetch. Category chips become optional hints or are removed.

**Expected:** **500–2k** input tokens typical; 3–5k for multi-domain questions.

### B1. Agent loop

Refactor `analyticsAssistant.service.ts` to Vercel AI SDK `streamText` with:

- `tools` registry (atomic analytics calls + slim return)
- `maxSteps` 3–5
- Tiny system prompt (no pre-loaded JSON wall)
- SSE still streams final text deltas to the portal

### B2. Initial tool set

| Tool | Analytics route |
|------|-----------------|
| `get_top_selling_items` | `items/popular` |
| `get_item_revenue_ranking` | `items/revenue` |
| `get_revenue_summary` | `orders/dashboard-summary` |
| `get_revenue_by_day` | `orders/revenue` (respects the requested range; `orders/trends` is rolling-from-today and is NOT used by tools) |
| `get_peak_hours` | `orders/peak-hours` |
| `get_hourly_pattern` | `orders/hourly-heatmap` (slimmed — never send full grid by default) |
| `get_payment_overview` | `billing/overview` |
| `get_staff_performance` | `orders/employee-performance` |
| `get_reservations_summary` | `reservations/summary` |
| `get_workforce_summary` | `employees/workforce/summary` |
| `get_cancellation_stats` | `orders/cancellation-analytics` |

All tool results pass through Phase A slim mappers.

### B3. Portal

- Make `context[]` optional (defaults empty → model uses tools).
- Hide category chips, or keep as "focus area" hints appended to the system prompt.
- Suggested prompts no longer need hard-coded context.

### B4. Observability

Extend `ai_usage_log`:

```ts
{
  toolsCalled: string[]
  contextTokensEstimate?: number
  preFetchMode: 'tools' | 'slim_categories' | 'legacy'
}
```

### Acceptance criteria

- [ ] "What were my top selling items?" calls `get_top_selling_items` (not revenue dashboard).
- [ ] Typical `inputTokens` < 3k for single-domain questions.
- [ ] Mid-stream tool failures surface a clear merchant message; do not leak internals.
- [x] `streamWithTools` + `maxSteps` wired in `LLMClient` / `analyticsAssistant.service`.
- [x] Tool registry in `src/tools/analyticsTools.ts` (11 tools, slim results).
- [x] `context[]` optional — used only as soft focus hints from Reports tab.
- [x] Portal category chips removed; suggested prompts no longer require context.
- [x] `ai_usage_log` records `toolsCalled` + `preFetchMode: 'tools'`.
- [x] `tsc --noEmit` passes.

---

## Phase C — Optional analytics-edge-api endpoints

Add only if Phase B hits latency or awkward multi-call patterns.

| Candidate | Why |
|-----------|-----|
| `GET analytics/compare/revenue` | Week-over-week in one round trip |
| `GET analytics/ai/digest` | Fixed ~500-token store digest for openers |

Prefer existing atomic routes from `analytics-edge-api/src/router.ts` until measured need.

---

## Later (orthogonal to token cost)

From Module 1 plan / product backlog:

1. **Chat sessions** — multi-turn Redis history + compression — **done** (`SessionManager`, `X-Session-ID`, DELETE session)
2. **Per-store rate limits** — Redis daily + burst counters — **done** (`checkAndConsumeRateLimit`)
3. **EKS GitOps deploy** for `ai-edge-api`
4. **Partial context resilience** — one failed fetch should not 502 the whole ask
5. **Module 2** — Revenue forecasting (skip until assistant path is efficient)

---

## Implementation order

```
Phase A  →  slim/ + atomic ContextBuilder + compact prompt   ✅
Phase B  →  tools + agent loop + portal simplify             ✅
Sessions / rate limits / compression                         ✅
Phase C  →  new analytics routes only if needed
Deploy   →  EKS GitOps for ai-edge-api
```

---

## File map (Phase B + sessions)

```
ai-edge-api/src/
  tools/analyticsTools.ts
  llm/LLMClient.ts
  limits/rateLimit.ts           # daily + burst Redis counters
  session/SessionManager.ts     # per-store sessions + compression
  services/analyticsAssistant.service.ts
  prompts/analyticsAssistant.ts # history-aware tool prompt
  handlers/analytics.handler.ts # sessionId + DELETE session
  repositories/usageLog.repository.ts  # sessionId, turnIndex
```

Portal: `sessionId` in hook; `X-Session-ID` from response; DELETE on clear/close.
