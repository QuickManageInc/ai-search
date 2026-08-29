# Copilot golden test questions (token + routing)

Use these after deploying observability (`ai_usage_log` + `copilot.llm.input.before_call` logs).

**How to test**

1. Open Copilot with **Feb 3–28, 2026** (or **30 days** preset) unless testing NL override.
2. Ask each question as a **new conversation** (avoids session history skewing prompt size).
3. After each ask, check:
   - Portal **Usage** panel: input tokens, tools called
   - Mongo `aiDB.ai_usage_log`: `promptMetrics`, `toolResults[].resultChars`, `inputTokens`
   - Server logs: `copilot.llm.input.before_call`, `copilot.tool.result`

**What “measure first” meant (step 3 in slim-split plan)**  
Before building slim-split, run ~10–20 real questions in prod and look at which tools return the biggest `toolResults[].resultChars`. That tells you where token savings matter most — you're doing that now with this list.

---

## Test run summary (2026-08-29)

**Environment:** dev, `gemini-3.5-flash-lite`, AI SDK v7, store `67a27db075551b7c50e7a54e`

| Metric | Finding |
|---|---|
| Tool routing | **12/12 completed asks routed correctly** (100%) |
| Tools measured | **11/27** unique tools with token data |
| Gemini 3.5 tool loop | Working after AI SDK v7 migration (no `thought_signature` errors) |
| Fixed prompt overhead | ~3,600 chars (~900 tokens) but **~6,300–6,800 inputTokens** per 1-tool ask (27 tool schemas) |
| Biggest payloads | `get_menu_health` **4,267** chars, `get_revenue_summary` **3,963** chars |
| NL date override | Verified: “last 30 days” → `dateRangeSource: nl`, Jul 31 – Aug 29 |
| Session follow-ups | Cache skipped (`reason: follow_up`); model re-fetches tools ✓ |
| Latency | 5–11s total; TTFT ~3–4s after tool fetch |

**Slim-split priority (from measured `resultChars`):**

1. `get_revenue_summary` — 3,963 chars ← dashboard split next
2. `get_menu_health` — 4,267 chars ← composite slim later
3. `get_revenue_by_day` — ~1,800 chars — acceptable for now
4. Menu atomics — 734–1,757 chars — low priority

---

## Run next (unchecked items)

Use **new conversation** for each unless noted. Date range: **Feb 3–28, 2026** unless testing NL.

### Revenue & orders (8 remaining)

- [ ] `get_revenue_mix` — *What share of sales is dine-in vs takeout?*
- [ ] `get_period_comparison` — *How did we do vs last week?* (solo; not stacked after diagnosis)
- [ ] `get_void_summary` — *How many voids did we have and what's the void rate?*
- [ ] `get_operations_overview` — *What's our order completion rate?*
- [ ] `get_fulfillment` — *How long does kitchen prep take on average?*
- [ ] `get_peak_hours` — *When are our busiest hours?*
- [ ] `get_hourly_pattern` — *Show me hour-by-hour order traffic.*
- [ ] `get_cancellation_stats` — *Why are orders getting cancelled?*

### Payments (2)

- [ ] `get_payment_overview` — *How much is collected vs outstanding?*
- [ ] `get_payment_details` — *What's our discount rate and tip rate?*

### People & labor (4)

- [ ] `get_staff_performance` — *Who were my top and bottom staff by sales? Include tips.*
- [ ] `get_staff_ops_health` — *How's staffing and labor ops looking?*
- [ ] `get_workforce_summary` — *How many employees do we have and any permit expirations?*
- [ ] `get_attendance_summary` — *How many clock-ins did we have this period?*

### Guests & platform (3)

- [ ] `get_reservations_summary` — *How many reservations and covers this period?*
- [ ] `get_platform_capabilities` — *What can you help me with?*
- [ ] `get_feature_howto` — *How do I schedule shifts in QuickManage?*

### NL date override (3)

Ask with **Feb 3–28** preset selected; confirm `dateRange.source: nl` in logs:

- [ ] *How were sales yesterday?*
- [ ] *Compare last week to the week before*
- [ ] *Revenue for last 7 days*

### Optional re-runs

- [ ] `get_revenue_summary` — new conversation, first turn only (baseline tokens without session history)
- [ ] `get_revenue_diagnosis` — confirm model uses **one** tool when prompt says not to stack comparison

---

## Decisions locked for slim-split (dashboard)

| Question | Decision |
|---|---|
| One daily tool or two? | **One:** keep `get_revenue_by_day` only; do **not** add `get_revenue_days` |
| `get_revenue_summary` compat | **Totals + best/worst only** — no kitchen; use `get_kitchen_activity` when we add it |
| Billing split | **Follow-up PR** after dashboard |

---

## Menu

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ✅ | `get_top_selling_items` | What were my top selling items? | ✓ | 6,285 | 734 | Feb 3–28 |
| ✅ | `get_item_revenue_ranking` | Which menu items earned the most revenue? | ✓ | 6,435 | 1,050 | |
| ✅ | `get_underperforming_items` | Which items are not selling well? | ✓ | 6,421 | 1,028 | |
| ✅ | `get_item_trends` | Which items improved or declined vs the previous period? | ✓ | 6,644 | 1,726 | |
| ✅ | `get_items_by_daypart` | What sells best at lunch vs dinner? | ✓ | 6,745 | 1,757 | |
| ✅ | `get_item_pairs` | What items are often ordered together? | ✓ | 6,345 | 920 | |
| ✅ | `get_menu_health` | What's wrong with my menu? | ✓ | 7,277 | **4,267** | `e6a07c03`; highest menu payload |

---

## Revenue & orders

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ✅ | `get_revenue_summary` | How were overall sales this period? | ✓ | 7,660 | **3,963** | `a042cc79`; follow-up in session — re-run solo for baseline |
| ✅ | `get_revenue_by_day` | Show me daily revenue for the last 30 days. | ✓ | 6,519 | 1,863 | `4bf071f3`; NL → Jul 31–Aug 29, $0 (no orders) |
| ⬜ | `get_revenue_mix` | What share of sales is dine-in vs takeout? | | | | |
| ✅ | `get_revenue_diagnosis` | Why were sales down this period? | ✓ (+ `get_period_comparison`) | 9,665 | 942 + 541 | `b66a6d81`; also called comparison — redundant |
| ⬜ | `get_period_comparison` | How did we do vs last week? | | | | |
| ⬜ | `get_void_summary` | How many voids did we have and what's the void rate? | | | | |
| ⬜ | `get_operations_overview` | What's our order completion rate? | | | | |
| ⬜ | `get_fulfillment` | How long does kitchen prep take on average? | | | | |
| ⬜ | `get_peak_hours` | When are our busiest hours? | | | | |
| ⬜ | `get_hourly_pattern` | Show me hour-by-hour order traffic. | | | | |
| ⬜ | `get_cancellation_stats` | Why are orders getting cancelled? | | | | |

**Also tested (same tool, different phrasing):**

| Status | Tool | Question | inputTokens | resultChars | Notes |
|---|---|---|---:|---:|---|
| ✅ | `get_revenue_by_day` | Show me daily revenue for this period. | 6,773 | 1,776 | `c03be51a`; follow-up, Feb 3–28, $1,656.51 / 36 orders |

---

## Payments

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ⬜ | `get_payment_overview` | How much is collected vs outstanding? | | | | |
| ⬜ | `get_payment_details` | What's our discount rate and tip rate? | | | | |

---

## People & labor

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ⬜ | `get_staff_performance` | Who were my top and bottom staff by sales? Include tips. | | | | |
| ⬜ | `get_staff_ops_health` | How's staffing and labor ops looking? | | | | |
| ⬜ | `get_workforce_summary` | How many employees do we have and any permit expirations? | | | | |
| ⬜ | `get_attendance_summary` | How many clock-ins did we have this period? | | | | |

---

## Guests & platform

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ⬜ | `get_reservations_summary` | How many reservations and covers this period? | | | | |
| ⬜ | `get_platform_capabilities` | What can you help me with? | | | | |
| ⬜ | `get_feature_howto` | How do I schedule shifts in QuickManage? | | | | |

---

## NL date override (same tool, different range)

Ask with **Feb 3–28** preset selected; server should override for that turn:

| Status | Phrase | Expected `dateRange.source` | Actual | Notes |
|---|---|---|---|---|
| ✅ | Show me daily revenue for the **last 30 days** | `nl` | `nl` | Matched `last 30 days` → Jul 31 – Aug 29 |
| ⬜ | How were sales **yesterday**? | `nl` | | |
| ⬜ | Compare **last week** to the week before | `nl` | | |
| ⬜ | Revenue for **last 7 days** | `nl` | | |

---

## High-token suspects (watch `toolResults[].resultChars`)

Measured vs expected — prioritize for slim-split:

| Tool | Measured | Priority |
|---|---|---|
| `get_menu_health` | **4,267** | Composite slim (phase 2) |
| `get_revenue_summary` | **3,963** | **Dashboard split next** |
| `get_revenue_by_day` | ~1,800 | OK for now |
| `get_revenue_diagnosis` | 942 | OK (composite already thin) |
| `get_period_comparison` | 541 | OK |
| Menu atomics | 734–1,757 | Low — schema overhead dominates |
| `get_staff_ops_health` | — | Not tested yet |
| `get_payment_overview` | — | Not tested yet |
| `get_operations_overview` | — | Not tested yet |

---

## Mongo queries (after testing)

```javascript
// Latest asks with tool payload sizes
db.ai_usage_log.find(
  { cached: false },
  {
    question: 1,
    toolsCalled: 1,
    inputTokens: 1,
    'promptMetrics.totalPromptChars': 1,
    toolResults: 1,
    timestamp: 1,
    requestId: 1,
  }
).sort({ timestamp: -1 }).limit(20)

// Biggest tool payloads
db.ai_usage_log.aggregate([
  { $match: { cached: false, toolResults: { $exists: true } } },
  { $unwind: '$toolResults' },
  { $group: {
      _id: '$toolResults.toolName',
      avgChars: { $avg: '$toolResults.resultChars' },
      maxChars: { $max: '$toolResults.resultChars' },
      count: { $sum: 1 },
  }},
  { $sort: { avgChars: -1 } },
])
```

**Known `requestId`s from 2026-08-29 run** (grep logs or Mongo):

| requestId | Tool(s) |
|---|---|
| `4bf071f3-8cd5-40c9-b3c7-23dcc9855507` | `get_revenue_by_day` (NL 30d) |
| `c03be51a-f4f0-449e-bb24-910c7663a3fc` | `get_revenue_by_day` (follow-up) |
| `b66a6d81-285e-4a20-9a12-21007fe5d2d4` | `get_revenue_diagnosis`, `get_period_comparison` |
| `a042cc79-c0d3-41b9-ab5e-84b488caa9da` | `get_revenue_summary` |
| `e6a07c03-a0bf-4666-9ec2-9dd1bc837921` | `get_menu_health` |

---

## Log lines to grep

```bash
grep 'copilot.llm.input.before_call'
grep 'copilot.tool.result'
grep 'copilot.llm.step.finish'
```
