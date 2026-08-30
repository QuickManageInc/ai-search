# Copilot slim-split & token plan

Status: **billing + ops slim-split shipped** (2026-08-30)  
Related: [README.md](./README.md), [Module1_Copilot_Intent_Tool_Filter_Plan.md](./Module1_Copilot_Intent_Tool_Filter_Plan.md), [Module1_Copilot_Tool_Test_Questions.md](./Module1_Copilot_Tool_Test_Questions.md)  
Also: date presets + NL override + full-range daily series + per-ask fetch memo already landed.

## Problem

`analytics-edge-api` responses can be large (~10–15kb raw). That is fine for HTTP.
What hurts is sending fat JSON into the LLM as tool results.

We already:

1. Slim projectors (`slimDashboardSummary`, etc.) before the model sees data
2. **Per-ask fetch memo** in `createAnalyticsTools` — one HTTP call per unique edge route+query
3. ✅ **`ai_usage_log` observability** — `promptMetrics`, `llmInputMessages`, `toolResults`, `llmSteps`, pre-LLM log `copilot.llm.input.before_call`
4. ✅ **Focus-based tool filtering** — `src/tools/toolFocusMap.ts`, env `AI_TOOL_FILTER` (default `hints`)

Still open: ~~split dashboard projections~~ ✅ dashboard slim-split shipped (2026-08-29).

## Locked decisions

| Topic | Decision |
|---|---|
| Daily series | **One tool:** slim `get_revenue_by_day` only — no `get_revenue_days` |
| `get_revenue_summary` compat | **Totals + best/worst** — no kitchen (add `get_kitchen_activity` later) |
| Billing split | ✅ `get_payment_totals`, `get_payment_days`; thin `get_payment_overview` |
| Ops split | ✅ `get_operations_totals`, `get_operations_days`; thin `get_operations_overview` |

## Proposed dashboard tools (next pass)

Shared `fetchOnce('orders/dashboard-summary')`:

| Tool | Projection |
|---|---|
| `get_revenue_totals` | period + summary totals/growth |
| `get_best_worst_days` | bestDay / worstDay only |
| `get_kitchen_activity` | currentActivity only |
| `get_revenue_summary` | totals + best/worst (compat alias, no days[]) |

Daily trend → existing **`get_revenue_by_day`** (`orders/revenue`).

## Billing tools (shipped 2026-08-30)

Shared `fetchOnce('billing/overview')`:

| Tool | Projection |
|---|---|
| `get_payment_totals` | period + billing counts + collected/outstanding + method split |
| `get_payment_days` | daily billings series only |
| `get_payment_overview` | compat alias → same as totals (no `days[]`) |

Tips/discounts/card brands → existing **`get_payment_details`** (`billing/payment-analytics`).

## Operations tools (shipped 2026-08-30)

Shared `fetchOnce('orders/overview')`:

| Tool | Projection |
|---|---|
| `get_operations_totals` | period + completion/cancellation summary |
| `get_operations_days` | daily ops stats series only |
| `get_operations_overview` | compat alias → same as totals (no `days[]`) |

Cancel breakdown → **`get_cancellation_stats`**; prep time → **`get_fulfillment`**.

## Principle

```
edge fetch (fat, OK)  →  request cache (once)  →  projector per tool (thin)  →  LLM
```

## Rollout order

1. ✅ Request fetch memo  
2. ✅ Full-range days in slim DTOs (drop last-7 trap)  
3. ✅ Observability in `ai_usage_log` + structured logs (see `Module1_Copilot_Tool_Test_Questions.md`)  
4. 🟡 Golden questions — **17/27 tools measured**; token anatomy in test doc  
5. ✅ Focus-based tool filtering (`AI_TOOL_FILTER`) — tab `hints`; **next:** [intent mode](./Module1_Copilot_Intent_Tool_Filter_Plan.md)  
6. ✅ Split dashboard projectors (`get_revenue_totals`, `get_best_worst_days`, `get_kitchen_activity`; thin `get_revenue_summary`)  
7. ⬜ Slim composite AI routes if logs still show fat payloads  
8. ✅ Billing split (`get_payment_totals` / `get_payment_days`; thin `get_payment_overview`)  
9. ✅ Ops split (`get_operations_totals` / `get_operations_days`; thin `get_operations_overview`)  
10. ⬜ Optional: provider prompt caching  

## `ai_usage_log` fields (new)

| Field | Meaning |
|---|---|
| `requestId` | Correlate with HTTP / logs |
| `dateRange` | Effective window (`source`: request \| nl) |
| `promptMetrics` | Chars/tokens of initial messages; `registeredToolCount`, `toolFilterActive`, `activeToolNames` |
| `llmInputMessages` | Role + char count + preview per message |
| `toolResults` | Per tool: `resultChars`, `fetchMs`, JSON preview |
| `llmSteps` | SDK step index, tool calls, tool result chars |
| `answerChars`, `usedFallback`, `outcome` | End-to-end result |

Prometheus: `ai_prompt_chars`, `ai_tool_result_chars{tool}`, `ai_tool_fetch_duration_seconds{tool}`.

Golden test questions: **`Module1_Copilot_Tool_Test_Questions.md`**

## Smoke checklist (ops)

- [x] Deploy analytics-service (post-backfill), analytics-edge-api, ai-edge-api, portal  
- [x] FAB + date presets + NL override  
- [x] Run golden questions (partial — 17/27 tools); logs + Mongo fields populate  
- [ ] Remaining golden questions: voids, fulfillment, peak/hourly, cancellations, payment details, staff performance, reservations, platform (see test doc checklist)  
- [ ] Deploy `ai/staff-ops` on analytics-edge-api (404 blocks `get_staff_ops_health`)  
- [ ] Discount rate / tips / voids / staff ops / platform help  
