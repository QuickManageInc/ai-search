# Copilot golden test questions (token + routing)

Index: [README.md](./README.md) · Intent filter plan: [Module1_Copilot_Intent_Tool_Filter_Plan.md](./Module1_Copilot_Intent_Tool_Filter_Plan.md)

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
| Tool routing | **17/17 asks** primary tool correct; 2 cases stacked redundant tools |
| Tools measured | **17/27** unique tools with token data |
| Gemini 3.5 tool loop | Working after AI SDK v7 migration (no `thought_signature` errors) |
| Fixed prompt overhead | ~3,600 chars (~900 tokens) but **~6,300–6,800 inputTokens** per 1-tool ask (27 tool schemas) |
| Biggest payloads | `get_revenue_summary` **4,404**, `get_menu_health` **4,267**, `get_operations_overview` **3,168** |
| NL date override | Verified: “last 30 days” → `dateRangeSource: nl`, Jul 31 – Aug 29 |
| Session follow-ups | Cache skipped (`reason: follow_up`); model re-fetches tools ✓ |
| Latency | 5–11s total; TTFT ~3–5s after tool fetch |
| **Blocker found** | `get_staff_ops_health` → **404** `ai/staff-ops` on analytics-edge-api (dev) |

---

## Input token anatomy (why ~6,500 tokens on a 1-tool ask)

`promptMetrics.approxPromptTokens` (~900) counts **message text only** (system + history + user). It does **not** include tool schemas. Gemini `inputTokens` is the full bill.

| Component | ~Tokens (typical 1-tool ask) | Share | Where |
|---|---:|---:|---|
| **27 tool schemas** | ~5,500–5,800 | **~85%** | Every LLM step resends all registered tools |
| System prompt | ~900 | ~14% | `buildToolSystemPrompt()` ~3,500 chars |
| User question | ~10 | &lt;1% | |
| Tool result JSON | +450–1,100 | step 2+ | Fat tools dominate here too |
| Session history | +50–150 | follow-ups | Grows per turn |

**Formula from golden runs:**

```
inputTokens ≈ ~5,500 fixed (schemas + messages) + tool result tokens + extra LLM steps
```

**Multi-step agent loop:** `streamWithTools` runs up to `AI_MAX_STEPS` (default 5). Each step resends system + tools + growing transcript. Cumulative `inputTokens` spikes when the model stacks tools (e.g. mix + summary → **10,189**; staff_ops fallbacks → **11,888**).

### What is NOT the main cost

- **System prompt** is large (~3,500 chars) but secondary vs tool schemas.
- **Menu atomics** have small payloads (734–1,757 chars) — slim-split there is low ROI vs schema overhead.
- System prompt **duplicates** tool routing rules already in each tool `description` (~4,800 chars total across 27 tools).
- System prompt references **4 tools not registered**: `get_scheduling_summary`, `get_time_off_summary`, `get_swaps_summary`, `get_compliance_status`.

### Mitigation priority

| Priority | Lever | Est. savings | Status |
|---|---|---:|---|
| 1 | **Tool filtering** — question **intent** mode (default) | ~3,500–4,500 tok/step | ✅ Shipped — [Intent plan](./Module1_Copilot_Intent_Tool_Filter_Plan.md) |
| 2 | **Slim-split payloads** (`get_revenue_summary`, ops, billing) | 500–1,100 tok when those tools run | ✅ Dashboard done; ops/billing next |
| 3 | **Prompt tuning** — no stacking summary after mix/diagnosis | 2,000–4,000 tok on bad paths | ⬜ |
| 4 | Shorten system prompt / dedupe descriptions | 300–600 tok | ⬜ |
| 5 | Gemini context caching (system + schemas) | varies | ⬜ |

### Tool filtering (shipped 2026-08-30 — intent mode)

Env: `AI_TOOL_FILTER` — **`intent` (default)** | `hints` | `always` | `off`  
Env: `AI_TOOL_FILTER_MAX` — default `15` (cap per ask in intent mode)

| Mode | Behavior |
|------|----------|
| **`intent`** | CORE ~9 tools + categories detected from **question keywords**; portal `context` ignored |
| `hints` | Legacy tab filter when portal sends `context` / `focusHints` |
| `always` | CORE ~9 tools only, no keyword expansion |
| `off` | All 27 tools every ask (A/B baseline) |

Logs / Mongo: `promptMetrics.toolFilterMode`, `intentCategories`, `activeToolNames`, `registeredToolCount`.

**Slim-split priority (from measured `resultChars`, updated batch 2):**

1. `get_revenue_summary` — was **4,404** chars → **~400–600** expected after slim-split ← **re-test**
2. `get_menu_health` — **4,267** chars ← composite slim (phase 2)
3. `get_operations_overview` — **3,168** chars ← ops daily series split
4. `get_payment_overview` — **2,348** chars ← billing split (follow-up PR)
5. `get_revenue_by_day` — ~1,800 chars — acceptable for now
6. Menu atomics — 734–1,757 chars — low priority (schema overhead dominates)

**Routing quirks (prompt tuning, not bugs):**

- `get_revenue_mix` with empty breakdown → model also called `get_revenue_summary` (+4,404 chars, **10,189 tokens**)
- `get_revenue_diagnosis` → also called `get_period_comparison` (redundant)
- `get_staff_ops_health` 404 → model recovered with `get_workforce_summary` + `get_attendance_summary` (**11,888 tokens**)

---

## Run next (unchecked items)

Use **new conversation** for each unless noted. Date range: **Feb 3–28, 2026** unless testing NL.

### Revenue & orders (6 remaining)

- [ ] `get_period_comparison` — *How did we do vs last week?* (solo; not stacked after diagnosis)
- [ ] `get_void_summary` — *How many voids did we have and what's the void rate?*
- [ ] `get_fulfillment` — *How long does kitchen prep take on average?*
- [ ] `get_peak_hours` — *When are our busiest hours?*
- [ ] `get_hourly_pattern` — *Show me hour-by-hour order traffic.*
- [ ] `get_cancellation_stats` — *Why are orders getting cancelled?*

### Payments (1)

- [ ] `get_payment_details` — *What's our discount rate and tip rate?*

### People & labor (2)

- [ ] `get_staff_performance` — *Who were my top and bottom staff by sales? Include tips.*
- [ ] `get_workforce_summary` — *How many employees do we have and any permit expirations?* (solo — already hit via staff_ops fallback)
- [ ] `get_attendance_summary` — *How many clock-ins did we have this period?* (solo)

### Guests & platform (3)

- [ ] `get_reservations_summary` — *How many reservations and covers this period?*
- [ ] `get_platform_capabilities` — *What can you help me with?*
- [ ] `get_feature_howto` — *How do I schedule shifts in QuickManage?*

### NL date override (3)

Ask with **Feb 3–28** preset selected; confirm `dateRange.source: nl` in logs:

- [ ] *How were sales yesterday?*
- [ ] *Compare last week to the week before*
- [ ] *Revenue for last 7 days*

### Deploy / fix before re-test

- [ ] **`ai/staff-ops` route** — 404 on dev analytics-edge-api; blocks `get_staff_ops_health`

### Optional re-runs

- [ ] `get_revenue_summary` — re-run *How were overall sales this period?*; expect **resultChars ~400–600** (was 3,963–4,404)
- [ ] `get_revenue_totals` / `get_best_worst_days` / `get_kitchen_activity` — solo asks (new tools)
- [ ] `get_revenue_mix` — Feb 3–28, new conversation (expect **796 chars**, 1 tool only — no summary stack)
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
| ✅ | `get_revenue_mix` | What share of sales is dine-in vs takeout? | ✓ (+ `get_revenue_summary`) | 10,189 | 82 + **4,404** | `74cac7c6`; Jul 30–Aug 29 empty; stacked summary unnecessarily |
| ✅ | `get_revenue_diagnosis` | Why were sales down this period? | ✓ (+ `get_period_comparison`) | 9,665 | 942 + 541 | `b66a6d81`; also called comparison — redundant |
| ⬜ | `get_period_comparison` | How did we do vs last week? | | | | |
| ⬜ | `get_void_summary` | How many voids did we have and what's the void rate? | | | | |
| ✅ | `get_operations_overview` | What's our order completion rate? | ✓ | 6,877 | **3,168** | `fb11eab8`; 97.2% completion; daily `days[]` bloat |
| ⬜ | `get_fulfillment` | How long does kitchen prep take on average? | | | | |
| ⬜ | `get_peak_hours` | When are our busiest hours? | | | | |
| ⬜ | `get_hourly_pattern` | Show me hour-by-hour order traffic. | | | | |
| ⬜ | `get_cancellation_stats` | Why are orders getting cancelled? | | | | |

**Also tested (same tool, different phrasing / follow-up):**

| Status | Tool | Question | inputTokens | resultChars | Notes |
|---|---|---|---:|---:|---|
| ✅ | `get_revenue_by_day` | Show me daily revenue for this period. | 6,773 | 1,776 | `c03be51a`; follow-up, Feb 3–28 |
| ✅ | `get_revenue_mix` | and for this date range | 6,248 | 796 | `c9d25464`; follow-up, Feb 3–28; POS 90.9% / KIOSK 9.1% |

---

## Payments

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ✅ | `get_payment_overview` | How much is collected vs outstanding? | ✓ | 6,641 | **2,348** | `c760f9a7`; $1,651.92 collected / $4.59 outstanding; daily `days[]` |
| ⬜ | `get_payment_details` | What's our discount rate and tip rate? | | | | |

---

## People & labor

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ⬜ | `get_staff_performance` | Who were my top and bottom staff by sales? Include tips. | | | | |
| ⚠️ | `get_staff_ops_health` | How's staffing and labor ops looking? | ✓ (+ fallbacks) | 11,888 | 97 + 248 + 110 | `dbd2d109`; **404** `ai/staff-ops`; fell back to workforce + attendance |
| 🟡 | `get_workforce_summary` | *(via staff_ops fallback)* | ✓ | — | 248 | 6 employees, 0 new hires, 0 permit expiries |
| 🟡 | `get_attendance_summary` | *(via staff_ops fallback)* | ✓ | — | 110 | 0 clock-ins Feb 3–28 |

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
| `get_revenue_summary` | **4,404** (pre-split) → **~400–600** target | ✅ Slim-split shipped — re-measure |
| `get_revenue_totals` | ~200–350 (est.) | New atomic tool |
| `get_best_worst_days` | ~150–250 (est.) | New atomic tool |
| `get_kitchen_activity` | ~100–150 (est.) | New atomic tool |
| `get_menu_health` | **4,267** | Composite slim (phase 2) |
| `get_operations_overview` | **3,168** | Ops daily series split |
| `get_payment_overview` | **2,348** | Billing split (follow-up PR) |
| `get_revenue_by_day` | ~1,800 | OK for now |
| `get_revenue_mix` | 796 (with data) / 82 (empty) | OK — watch summary stack on empty |
| `get_revenue_diagnosis` | 942 | OK (composite already thin) |
| `get_period_comparison` | 541 | OK |
| Menu atomics | 734–1,757 | Low — schema overhead dominates |
| `get_staff_ops_health` | 97 (error JSON) | **Deploy `ai/staff-ops` route** |
| `get_workforce_summary` | 248 | OK (atomic) |
| `get_attendance_summary` | 110 | OK (atomic) |

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
| `74cac7c6-2c19-4cee-9087-5e1e9ebdc7cf` | `get_revenue_mix`, `get_revenue_summary` (Jul 30–Aug 29 empty) |
| `c9d25464-dd6c-49de-8fef-ddb4dc856173` | `get_revenue_mix` (follow-up, Feb 3–28) |
| `fb11eab8-e81d-44a9-a847-06967c54e82a` | `get_operations_overview` |
| `c760f9a7-8b31-4d02-8f99-f825b6af4289` | `get_payment_overview` |
| `dbd2d109-f5ac-4c47-a3ca-7fbe1a63a33e` | `get_staff_ops_health` (404), `get_workforce_summary`, `get_attendance_summary` |

---

## Log lines to grep

```bash
grep 'copilot.llm.input.before_call'
grep 'copilot.tool.result'
grep 'copilot.llm.step.finish'
```
