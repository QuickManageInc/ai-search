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

## Test run summary (2026-08-31 — intent mode batch)

**Environment:** dev, `gemini-3.5-flash-lite`, AI SDK v7, `AI_TOOL_FILTER=intent`, store `67a27db075551b7c50e7a54e`

| Metric | Finding |
|---|---|
| Intent mode | **15 tools** on category match (~4.3–5.2k tok); **9 CORE** when no keyword match (~3.6–4.8k tok) |
| Anti-stacking | ✅ mix **5,478**, diagnosis **4,590**, compare **4,339–6,059** — 1 tool each |
| Slim-split billing | `get_payment_overview` **297 chars** (was 2,348) — **−87%** |
| Slim-split ops | `get_operations_overview` **227 chars** (was 3,168) — **−93%** |
| NL dual compare | ✅ week-of-month, day ranges, full months → `compare=custom` |
| Response cache | ⚠️ Stale error answers cached for 1h — **fixed**: skip cache on `toolFailed` / outage text |
| **Still open** | 2 routing quirks: `get_revenue_totals` vs summary, `get_kitchen_activity` vs fulfillment |

### Intent-mode token baselines (1-tool ask, Feb 3–28)

| Tools registered | inputTokens (typical) | Example |
|---:|---:|---|
| 9 (CORE only) | ~4,341–4,590 | how-to, diagnosis, voids (CORE) |
| 11 (CORE + workforce) | ~4,623 | employees / permits solo |
| 15 (CORE + category) | ~4,339–5,478 | payment, ops, reservations, revenue mix, compare |

`promptMetrics.approxPromptTokens` ≈ **1,000–1,150** (messages only); full `inputTokens` includes schemas.

---

## Test run summary (2026-09-01 — routing fixes verified)

**Environment:** dev, `gemini-3.5-flash-lite`, `AI_TOOL_FILTER=intent`, store `67a27db075551b7c50e7a54e`, Feb 3–28

| Metric | Finding |
|---|---|
| Priority cap + pins | ✅ reservations pinned through 15-tool cap; workforce solo routed correctly |
| How-to guard | ✅ "How do I schedule shifts?" → CORE only (9 tools), **no** workforce false positive |
| Slim platform | ✅ how-to skipped fat `get_platform_capabilities`; direct `get_feature_howto` (345 chars) |
| Anti-stacking | ✅ 1 tool each on all three asks |
| Tab hints ignored | Portal sent `focusHints: ["revenue"]`; intent mode routed from question only |

| requestId | Question | Tool | inputTokens | resultChars | registered |
|---|---|---|---:|---:|---:|
| `d27f63fd-…` | How many reservations and covers? | `get_reservations_summary` | 5,281 | 162 | 15 |
| `d3f3a73a-…` | How many employees and permit expirations? | `get_workforce_summary` | 4,623 | 248 | 11 |
| `a93ede30-…` | How do I schedule shifts? | `get_feature_howto` | 4,341 | 345 | 9 |

---

## Test run summary (2026-09-01 — golden batch 3)

**Environment:** dev, `AI_TOOL_FILTER=intent`, automated via `npx tsx scripts/run-golden-asks.ts`

| Metric | Finding |
|---|---|
| Pass rate | **10/13** exact 1-tool match; 3 edge cases documented below |
| NL dates | ✅ yesterday, last-week compare; ⚠️ “last 7 days” → `get_revenue_summary` (not `get_revenue_by_day`) |
| Voids | ✅ routed correctly; 2nd ask was **cache hit** (0 tok in meta) |
| Fulfillment | ✅ `\bkitchen prep\b` keyword shipped; was platform miss → pin `get_fulfillment` |

| requestId | Question | Tool | inputTokens | Pass |
|---|---|---|---:|---|
| `0d53f3c3-…` | How did we do vs last week? | `get_period_comparison` | 5,468 | ✅ NL wow |
| `1f5eb18a-…` | Voids and void rate? | *(cache hit)* | 0 | ✅ cached |
| `67545d24-…` | Kitchen prep time? | `get_platform_capabilities` | 4,605 | ⚠️ keyword gap |
| `33877683-…` | Busiest hours? | `get_peak_hours` | 5,313 | ✅ |
| `ac812eb2-…` | Hour-by-hour traffic? | `get_hourly_pattern` | 5,441 | ✅ |
| `c371344c-…` | Why cancelled? | `get_cancellation_stats` | 5,420 | ✅ |
| `1614aca3-…` | Discount and tip rate? | `get_payment_details` | 5,089 | ✅ |
| `285eb3b9-…` | Top/bottom staff? | `get_staff_performance` | 5,262 | ✅ |
| `a8d2e5eb-…` | Clock-ins this period? | `get_attendance_summary` | 4,576 | ✅ |
| `89ccd0cb-…` | What can you help with? | `get_platform_capabilities` | 4,601 | ✅ slim catalog |
| `abaffd12-…` | Sales yesterday? | `get_revenue_summary` | 4,350 | ✅ NL |
| `e259f74d-…` | Last week vs week before | `get_period_comparison` | 5,466 | ✅ NL |
| `c0532085-…` | Revenue for last 7 days | `get_revenue_summary` | 4,356 | ⚠️ summary not by_day |

**Uncached void baseline (pre-cache):** `get_void_summary`, **4,562 tok**, 0 voids / 0% rate.

---

## Test run summary (2026-09-01 — golden batch 4)

**Environment:** dev, `npm run golden:remaining` + `npm run golden:redo`

| Metric | Finding |
|---|---|
| Dashboard atomics | ✅ `get_revenue_summary` solo **4,450 tok**; ✅ `get_best_worst_days` **5,447 tok** |
| Staff ops | ✅ `get_staff_ops_health` **4,725 tok** — route live (no 404) |
| Fulfillment fix | ✅ `get_fulfillment` **5,244 tok** — avg prep 27m 34s |
| Follow-up redo | ✅ turn 2 “redo now” → `get_operations_overview` **5,536 tok** |
| Quirks | ⚠️ “total sales” → `get_revenue_summary` not totals; ⚠️ “kitchen right now” → fulfillment |

| requestId | Question | Tool | inputTokens | Pass |
|---|---|---|---:|---|
| *(user curl)* | Kitchen prep time? | `get_fulfillment` | 5,244 | ✅ keyword fix |
| `599c695d-…` | Overall sales this period? | `get_revenue_summary` | 4,450 | ✅ |
| `325f49ad-…` | Total sales and order count? | `get_revenue_summary` | 4,452 | ⚠️ alias |
| `164052c3-…` | Best and worst day? | `get_best_worst_days` | 5,447 | ✅ |
| `e046c0bf-…` | Orders in kitchen now? | `get_fulfillment` | 5,246 | ⚠️ no live queue |
| `dcce3a50-…` | Staffing and labor ops? | `get_staff_ops_health` | 4,725 | ✅ |
| `0b44057f-…` | redo now (follow-up) | `get_operations_overview` | 5,536 | ✅ |

---

## Test run summary (2026-08-29 — pre-intent / tab hints)

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
| 2 | **Slim-split payloads** (`get_revenue_summary`, ops, billing) | 500–1,100 tok when those tools run | ✅ Measured — payment **297**, ops **227** chars |
| 3 | **Prompt tuning** — no stacking summary after mix/diagnosis | 2,000–4,000 tok on bad paths | ✅ Verified 2026-08-30 — mix **5,478** tok (was 10,189), diagnosis **4,590**, compare **4,339–6,059** |
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
| `off` | All 31 tools every ask (A/B baseline) |

Logs / Mongo: `promptMetrics.toolFilterMode`, `intentCategories`, `activeToolNames`, `registeredToolCount`.

**Slim-split priority (from measured `resultChars`, updated batch 2):**

1. `get_revenue_summary` — was **4,404** → **~407–490** after slim-split ✅ (`f32271d3` re-test pending solo)
2. `get_menu_health` — ~~**4,267**~~ → slim composite ~**2,667** chars est. ✅ (2026-09-01)
3. ~~`get_operations_overview` — **3,168**~~ → **227** ✅ (`ad052a5c`)
4. ~~`get_payment_overview` — **2,348**~~ → **297** ✅ (`db57982e`)
5. `get_revenue_by_day` — ~1,800 chars — acceptable for now
6. Menu atomics — 734–1,757 chars — low priority (schema overhead dominates)

**Routing quirks (prompt tuning — anti-stacking shipped 2026-08-30):**

- ~~`get_revenue_mix` with empty breakdown → model also called `get_revenue_summary` (+4,404 chars, **10,189 tokens**)~~ → ✅ fixed: **5,478 tok**, 1 tool (`bc6eb8bc`)
- ~~`get_revenue_diagnosis` → also called `get_period_comparison` (redundant)~~ → ✅ fixed: **4,590 tok**, 1 tool (`d7a35c26`)
- `get_staff_ops_health` 404 → model recovered with `get_workforce_summary` + `get_attendance_summary` (**11,888 tokens**)
- **Response cache poison** — Mongo outage answer cached 1h; flush Redis or wait TTL. ✅ Fixed: skip cache when tools return `{ error }` or outage apology text.
- ~~Reservations cap drop → wrong `get_revenue_summary`~~ → ✅ fixed: pinned `get_reservations_summary` (`d27f63fd`)
- ~~Workforce keyword gap → `get_staff_ops_health` on employees ask~~ → ✅ fixed: `get_workforce_summary` (`d3f3a73a`)
- ~~How-to false workforce intent → 8,192 tok~~ → ✅ fixed: CORE only, **4,341 tok** (`a93ede30`)
- **Fulfillment keyword** — ✅ `\bkitchen prep\b` in `questionIntent.ts` (operations + pin); local verify: `intent: ["operations"]`, pinned `get_fulfillment`
- **Follow-up "redo now"** — ✅ `get_operations_overview` on turn 2 (`0b44057f`, 5,536 tok)

---

## Run next (unchecked items)

Use **new conversation** for each unless noted. Date range: **Feb 3–28, 2026** unless testing NL.

### Revenue & orders (1 remaining)

- [x] `get_period_comparison` — *How did we do vs last week?* → `0d53f3c3`, 5,468 tok, NL wow ✅
- [x] `get_void_summary` — *How many voids…?* → 4,562 tok uncached; cache hit on re-run ✅
- [x] `get_fulfillment` — **5,244 tok** live ✅ (`kitchen prep` keyword fix); avg 27m 34s
- [x] `get_peak_hours` — *When are our busiest hours?* → `33877683`, 5,313 tok ✅
- [x] `get_hourly_pattern` — *Show me hour-by-hour order traffic.* → `ac812eb2`, 5,441 tok ✅
- [x] `get_cancellation_stats` — *Why are orders getting cancelled?* → `c371344c`, 5,420 tok ✅

### Payments

- [x] `get_payment_details` — *What's our discount rate and tip rate?* → `1614aca3`, 5,089 tok ✅

### People & labor

- [x] `get_staff_performance` — *Who were my top and bottom staff…?* → `285eb3b9`, 5,262 tok ✅
- [x] `get_workforce_summary` — solo → `d3f3a73a`, 248 chars / 4,623 tok ✅
- [x] `get_attendance_summary` — solo → `a8d2e5eb`, 4,576 tok ✅

### Guests & platform

- [x] `get_reservations_summary` → `d27f63fd`, 162 chars / 5,281 tok ✅
- [x] `get_platform_capabilities` → `89ccd0cb`, 4,601 tok ✅ (slim catalog)
- [x] `get_feature_howto` → `a93ede30`, 345 chars / 4,341 tok ✅

### NL date override

- [x] *How were sales yesterday?* → `abaffd12`, NL Aug 31 ✅
- [x] *Compare last week to the week before* → `e259f74d`, NL wow ✅
- [x] *Revenue for last 7 days* → `c0532085`; NL ok but **`get_revenue_summary`** not by_day ⚠️
- [x] Dual compares (feb/mar, week-of-month, day ranges) — prior batch ✅

### Deploy / fix

- [x] **`ai/staff-ops` route** — ✅ live on dev (`dcce3a50`, 4,725 tok)
- [x] **`kitchen prep` keyword** — ✅ live verified 5,244 tok

### Still unchecked / quirks

- [x] `get_revenue_summary` — solo → `599c695d`, **4,450 tok** ✅
- [x] `get_best_worst_days` — solo → `164052c3`, **5,447 tok** ✅
- [x] `get_staff_ops_health` — solo → `dcce3a50`, **4,725 tok** ✅
- [x] Follow-up “redo now” → `0b44057f` ✅
- [ ] `get_revenue_totals` — model picks `get_revenue_summary` (acceptable alias; pin optional)
- [ ] `get_kitchen_activity` — “in kitchen now” → `get_fulfillment`; needs live-queue pin or data gap note

### Verified re-runs (intent mode)

**2026-09-01 golden batch 4** (`npm run golden:remaining` + `golden:redo`):

- [x] `get_fulfillment` — **5,244 tok** (user curl); keyword fix verified
- [x] `get_revenue_summary` solo — **4,450 tok** (`599c695d`)
- [x] `get_best_worst_days` — **5,447 tok** (`164052c3`)
- [x] `get_staff_ops_health` — **4,725 tok** (`dcce3a50`); route live
- [x] Follow-up redo → `get_operations_overview` **5,536 tok** (`0b44057f`)

**2026-09-01 golden batch 3** (`scripts/run-golden-asks.ts`):

- [x] `get_period_comparison` (wow) — **5,468 tok** (`0d53f3c3`); NL last week
- [x] `get_void_summary` — **4,562 tok** uncached; 0 voids
- [x] `get_peak_hours` — **5,313 tok** (`33877683`)
- [x] `get_hourly_pattern` — **5,441 tok** (`ac812eb2`)
- [x] `get_cancellation_stats` — **5,420 tok** (`c371344c`); 2.8% cancel rate
- [x] `get_payment_details` — **5,089 tok** (`1614aca3`)
- [x] `get_staff_performance` — **5,262 tok** (`285eb3b9`)
- [x] `get_attendance_summary` — **4,576 tok** (`a8d2e5eb`); 0 clock-ins
- [x] `get_platform_capabilities` — **4,601 tok** (`89ccd0cb`)
- [x] NL yesterday → `get_revenue_summary` **4,350 tok** (`abaffd12`)
- [x] NL last week compare → **5,466 tok** (`e259f74d`)

**2026-09-01 routing fixes:**

- [x] `get_reservations_summary` — **162 chars**, 1 tool, **5,281 tok** (`d27f63fd`); pinned; 6 reservations / 24 covers
- [x] `get_workforce_summary` — **248 chars**, 1 tool, **4,623 tok** (`d3f3a73a`); solo employees ask; 6 headcount
- [x] `get_feature_howto` — **345 chars**, 1 tool, **4,341 tok** (`a93ede30`); how-to guard; was 8,192 tok pre-fix

**2026-08-31 batch:**

- [x] `get_revenue_mix` — **796 chars**, 1 tool, **5,478 tok** (`bc6eb8bc`)
- [x] `get_revenue_diagnosis` — **942 chars**, 1 tool, **4,590 tok** (`d7a35c26`)
- [x] `get_payment_overview` — **297 chars**, 1 tool, **5,178 tok** (`db57982e`)
- [x] `get_operations_overview` — **227 chars**, 1 tool, **5,170 tok** (`ad052a5c`); 97.2% completion
- [ ] `get_revenue_summary` — solo ask; expect **~400–600** resultChars
- [ ] `get_revenue_totals` / `get_best_worst_days` / `get_kitchen_activity` — solo asks

---

## Decisions locked for slim-split (dashboard)

| Question | Decision |
|---|---|
| One daily tool or two? | **One:** keep `get_revenue_by_day` only; do **not** add `get_revenue_days` |
| `get_revenue_summary` compat | **Totals + best/worst only** — no kitchen; use `get_kitchen_activity` when we add it |
| Billing split | ✅ `get_payment_totals`, `get_payment_days`; thin `get_payment_overview` (297 chars measured) |

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
| ✅ | `get_menu_health` | What's wrong with my menu? | ✓ | **4,985** | ~**2,667** est. | slim-split ✅ (was 4,267 / 7,277 tok pre-slim) |

---

## Revenue & orders

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ✅ | `get_revenue_summary` | How were overall sales this period? | ✓ | **4,450** | — | `599c695d`; intent solo; $1,656.51 / 36 orders |
| ✅ | `get_revenue_by_day` | Show me daily revenue for the last 30 days. | ✓ | 6,519 | 1,863 | `4bf071f3`; NL → Jul 31–Aug 29, $0 (no orders) |
| ✅ | `get_revenue_mix` | What share of sales is dine-in vs takeout? | ✓ | **5,478** | **796** | `bc6eb8bc`; intent 15 tools; anti-stack ✅ (was 10,189 stacked) |
| ✅ | `get_revenue_diagnosis` | Why were sales down this period? | ✓ | **4,590** | **942** | `d7a35c26`; intent CORE; anti-stack ✅ (was 9,665 stacked) |
| ✅ | `get_period_comparison` | Compare february with march | ✓ | **4,339** | **638** | `0be5a79e`; NL dual + `compare=custom` |
| ✅ | `get_period_comparison` | How did we do vs last week? | ✓ | **5,468** | — | `0d53f3c3`; NL wow Aug 24–30 vs prior |
| ✅ | `get_void_summary` | How many voids and void rate? | ✓ | **4,562** | — | uncached; 0 voids / 0%; cache hit on re-run |
| ⚠️ | `get_fulfillment` | How long does kitchen prep take? | ✓ | **5,244** | — | user curl + keyword fix ✅ |
| ✅ | `get_operations_overview` | What's our order completion rate? | ✓ | **5,170** | **227** | `ad052a5c`; slim-split ✅ (was 3,168); 97.2% |
| ✅ | `get_peak_hours` | When are our busiest hours? | ✓ | **5,313** | — | `33877683`; Thursday peak |
| ✅ | `get_hourly_pattern` | Show me hour-by-hour order traffic. | ✓ | **5,441** | — | `ac812eb2` |
| ✅ | `get_cancellation_stats` | Why are orders getting cancelled? | ✓ | **5,420** | — | `c371344c`; 2.8% |

**Also tested (same tool, different phrasing / follow-up):**

| Status | Tool | Question | inputTokens | resultChars | Notes |
|---|---|---|---:|---:|---|
| ✅ | `get_revenue_by_day` | Show me daily revenue for this period. | 6,773 | 1,776 | `c03be51a`; follow-up, Feb 3–28 |
| ✅ | `get_revenue_mix` | and for this date range | 6,248 | 796 | `c9d25464`; follow-up, Feb 3–28; POS 90.9% / KIOSK 9.1% |

---

## Payments

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ✅ | `get_payment_overview` | How much is collected vs outstanding? | ✓ | **5,178** | **297** | `db57982e`; slim-split ✅ (was 2,348); intent 15 tools |
| ✅ | `get_payment_details` | What's our discount rate and tip rate? | ✓ | **5,089** | — | `1614aca3`; 0% discount / 0% tips |

---

## People & labor

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ✅ | `get_staff_performance` | Who were my top and bottom staff by sales? Include tips. | ✓ | **5,262** | — | `285eb3b9`; Reda Haloubi sole staff |
| ⚠️ | `get_staff_ops_health` | How's staffing and labor ops looking? | ✓ | **4,725** | — | `dcce3a50`; route live ✅ (was 404) |
| ✅ | `get_workforce_summary` | How many employees and permit expirations? | ✓ | **4,623** | **248** | `d3f3a73a`; intent workforce; pinned; solo ask (was wrong via staff_ops) |
| ✅ | `get_attendance_summary` | How many clock-ins did we have this period? | ✓ | **4,576** | — | `a8d2e5eb`; 0 clock-ins Feb 3–28 |

---

## Guests & platform

| Status | Tool expected | Test question | Tools called | inputTokens | resultChars | Notes |
|---|---|---|---|---:|---:|---|
| ✅ | `get_reservations_summary` | How many reservations and covers? | ✓ | **5,281** | **162** | `d27f63fd`; pinned; 6 reservations / 24 covers |
| ✅ | `get_platform_capabilities` | What can you help me with? | ✓ | **4,601** | — | `89ccd0cb`; slim catalog |
| ✅ | `get_feature_howto` | How do I schedule shifts? | ✓ | **4,341** | **345** | `a93ede30`; CORE only; direct howto (no capabilities call) |

---

## NL date override (same tool, different range)

Ask with **Feb 3–28** preset selected; server should override for that turn:

| Status | Phrase | Expected `dateRange.source` | Actual | Notes |
|---|---|---|---|---|
| ✅ | Show me daily revenue for the **last 30 days** | `nl` | `nl` | Matched `last 30 days` → Jul 31 – Aug 29 |
| ✅ | How were sales **yesterday**? | `nl` | `nl` | `abaffd12`; Aug 31, $0 |
| ✅ | Compare **last week** to the week before | `nl` | `nl` | `e259f74d`; Aug 24–30 vs Aug 17–23 |
| ✅ | Compare the **last week of february** and the **first week of mars** | `nl` + dual | `nl` | dual + `compare=custom` |
| ✅ | Compare **17 feb to 28 feb** with **30 mar to 15 apr** | `nl` + dual | `nl` | `4d5bff08`; 558 chars |
| ✅ | Compare **february with march** | `nl` + dual | `nl` | `0be5a79e`; 638 chars |
| ⚠️ | Revenue for **last 7 days** | `nl` | `nl` | `c0532085`; NL ok → `get_revenue_summary` not by_day |

---

## High-token suspects (watch `toolResults[].resultChars`)

Measured vs expected — prioritize for slim-split:

| Tool | Measured | Priority |
|---|---|---|
| `get_revenue_summary` | **4,404** (pre-split) → **~400–600** target | ✅ Slim-split shipped — re-measure |
| `get_revenue_totals` | ~200–350 (est.) | New atomic tool |
| `get_best_worst_days` | ~150–250 (est.) | New atomic tool |
| `get_kitchen_activity` | ~100–150 (est.) | New atomic tool |
| `get_menu_health` | ~~**4,267**~~ → ~**2,667** slim | ✅ Done 2026-09-01 |
| `get_operations_overview` | **227** (was 3,168) | ✅ Done |
| `get_payment_overview` | **297** (was 2,348) | ✅ Done |
| `get_revenue_mix` | 796 | OK — anti-stack verified |
| `get_revenue_diagnosis` | 942 | OK |
| `get_period_comparison` | 541–638 | OK |
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

**Known `requestId`s — 2026-08-31 intent batch:**

| requestId | Tool(s) | inputTokens | resultChars |
|---|---|---:|---:|
| `db57982e-f6cc-4bd8-a5bc-d33a33d33eac` | `get_payment_overview` | 5,178 | 297 |
| `ad052a5c-4dea-4e53-9d1d-3ea40b988b43` | `get_operations_overview` | 5,170 | 227 |
| `bc6eb8bc-c62e-408b-88f7-549d6566de18` | `get_revenue_mix` | 5,478 | 796 |
| `d7a35c26-bd41-4d21-a22e-ee298c64169f` | `get_revenue_diagnosis` | 4,590 | 942 |
| `0be5a79e-5989-41a0-8a92-d2cd5c1d0553` | `get_period_comparison` | 4,339 | 638 |
| `4d5bff08-55d6-471b-a85e-489250d0a9d4` | `get_period_comparison` | 4,366 | 558 |
| `2c304d8a-b69a-45a2-a18f-b73c2552261e` | ops×3 (Mongo fail) | 10,428 | 97×3 |

**Known `requestId`s — 2026-09-01 routing-fix batch:**

| requestId | Tool(s) | inputTokens | resultChars |
|---|---|---:|---:|
| `d27f63fd-6e4d-4e8e-baec-ce43ff17a3cd` | `get_reservations_summary` | 5,281 | 162 |
| `d3f3a73a-6294-4835-baf5-2709d1b2f21d` | `get_workforce_summary` | 4,623 | 248 |
| `a93ede30-c1c1-43ff-af0c-6bdc2a27ac5c` | `get_feature_howto` | 4,341 | 345 |

**Known `requestId`s — 2026-09-01 golden batch 3:**

| requestId | Tool(s) | inputTokens |
|---|---|---:|
| `0d53f3c3-d9d1-402f-a0f3-25369a13441d` | `get_period_comparison` (wow) | 5,468 |
| `33877683-40d0-420c-a563-91e8cf281b97` | `get_peak_hours` | 5,313 |
| `ac812eb2-5258-4c3f-9c81-b0690ca0fd36` | `get_hourly_pattern` | 5,441 |
| `c371344c-b999-4597-ae51-c3ce90169de6` | `get_cancellation_stats` | 5,420 |
| `1614aca3-e7ed-4faa-8359-bc27177a2481` | `get_payment_details` | 5,089 |
| `285eb3b9-d326-4fd0-a48e-94ec3954c6b0` | `get_staff_performance` | 5,262 |
| `a8d2e5eb-02da-415e-9f2e-226a4999d878` | `get_attendance_summary` | 4,576 |
| `89ccd0cb-8105-44bf-ae5e-20b5c88e647b` | `get_platform_capabilities` | 4,601 |
| `abaffd12-f68d-4a2d-b2dc-e34d216f8935` | `get_revenue_summary` (NL yesterday) | 4,350 |
| `e259f74d-974b-4cee-bca0-f458de1de01f` | `get_period_comparison` (NL wow) | 5,466 |
| `c0532085-4786-451c-9b32-d7a625b412ba` | `get_revenue_summary` (NL last 7d) | 4,356 |
| `67545d24-23f9-45ef-b032-730b5e3afc41` | `get_platform_capabilities` (fulfillment miss) | 4,605 |

**Known `requestId`s — 2026-09-01 golden batch 4:**

| requestId | Tool(s) | inputTokens |
|---|---|---:|
| `599c695d-e153-44ad-b920-e9da11a5a47e` | `get_revenue_summary` (solo) | 4,450 |
| `325f49ad-b807-4ddb-9c5c-d3b631b74a5c` | `get_revenue_summary` (totals ask) | 4,452 |
| `164052c3-a096-46ce-9e29-ba8cce1ef5a2` | `get_best_worst_days` | 5,447 |
| `e046c0bf-d3f8-49b4-9790-246b772055c8` | `get_fulfillment` (kitchen now miss) | 5,246 |
| `dcce3a50-92ad-45d9-9c09-8cfc3b7e85a3` | `get_staff_ops_health` | 4,725 |
| `0b44057f-…` | `get_operations_overview` (redo follow-up) | 5,536 |

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

## Re-test after routing fixes (2026-09-01)

Shipped in `ai-edge-api`: priority cap, workforce keywords, how-to guard, follow-up retry, slim platform catalog.

| Question | Expected tool | Status | requestId |
|---|---|---|---|
| How many reservations and covers? | `get_reservations_summary` | ✅ | `d27f63fd` |
| How many employees and permit expirations? | `get_workforce_summary` | ✅ | `d3f3a73a` |
| How do I schedule shifts? | `get_feature_howto` or `get_task_howto` | ✅ | `a93ede30` |
| How do I download a report? | `search_platform_help` → `get_task_howto` | ✅ | `0f41225a` (8,010 tok) |
| How do I check orders in the kitchen? | `search_platform_help` → `get_task_howto` | ✅ | `caa879d3` (7,939 tok) |
| How do I export fiscal data? | `search_platform_help` → `get_task_howto` | ✅ | `d06b3b24` (7,886 tok) |

**Platform task tools shipped 2026-09-01:** `get_task_howto`, `search_platform_help` (CORE, ~35 tools total).  
**Automated batch:** `npm run golden:platform` in `ai-edge-api`.  
**Token optimization:** deferred — ~8k tok/how-to from 3-step loop × 11 CORE schemas (see Platform Help plan Phase E).

**Pre-fix batch (superseded for reservations / employees / how-to):**

| requestId | Tool(s) | Notes |
|---|---|---|
| `d7eddacf-…` | `get_payment_details` | 358 chars / 5,015 tok ✅ |
| `5d07b073-…` | `get_attendance_summary` | 110 chars / 4,502 tok ✅ |
| ~~`341c0276-…`~~ | ~~`get_staff_ops_health`~~ | Superseded by `d3f3a73a` |
| ~~`4bff5e47-…`~~ | ~~`get_revenue_summary`~~ | Superseded by `d27f63fd` |
| ~~`10e96f1a-…`~~ | capabilities + howto | Superseded by `a93ede30` (4,341 vs 8,192 tok) |
| `05726760-…` | `get_staff_ops_health` | 4,385 tok ✅ |

---

## Log lines to grep

```bash
# Automated golden batch (local dev, AUTH_DISABLED on analytics-edge-api)
cd ai-edge-api && npm run golden:asks        # full batch
npm run golden:remaining                     # dashboard atomics + staff_ops
npm run golden:platform                    # platform task how-to batch

grep 'copilot.llm.input.before_call'
grep 'copilot.tool.result'
grep 'copilot.llm.step.finish'
```
