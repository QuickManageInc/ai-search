# Copilot slim-split & token plan

Status: **observability shipped** — dashboard tool split next  
Related: date presets + NL override + full-range daily series + per-ask fetch memo already landed.

## Problem

`analytics-edge-api` responses can be large (~10–15kb raw). That is fine for HTTP.
What hurts is sending fat JSON into the LLM as tool results.

We already:

1. Slim projectors (`slimDashboardSummary`, etc.) before the model sees data
2. **Per-ask fetch memo** in `createAnalyticsTools` — one HTTP call per unique edge route+query
3. **`ai_usage_log` observability** — `promptMetrics`, `llmInputMessages`, `toolResults`, `llmSteps`, pre-LLM log `copilot.llm.input.before_call`

Still open: split dashboard projections so totals-only questions don't get 31 daily rows.

## Locked decisions

| Topic | Decision |
|---|---|
| Daily series | **One tool:** slim `get_revenue_by_day` only — no `get_revenue_days` |
| `get_revenue_summary` compat | **Totals + best/worst** — no kitchen (add `get_kitchen_activity` later) |
| Billing split | **Follow-up PR** after dashboard |

## Proposed dashboard tools (next pass)

Shared `fetchOnce('orders/dashboard-summary')`:

| Tool | Projection |
|---|---|
| `get_revenue_totals` | period + summary totals/growth |
| `get_best_worst_days` | bestDay / worstDay only |
| `get_kitchen_activity` | currentActivity only |
| `get_revenue_summary` | totals + best/worst (compat alias, no days[]) |

Daily trend → existing **`get_revenue_by_day`** (`orders/revenue`).

## Principle

```
edge fetch (fat, OK)  →  request cache (once)  →  projector per tool (thin)  →  LLM
```

## Rollout order

1. ✅ Request fetch memo  
2. ✅ Full-range days in slim DTOs (drop last-7 trap)  
3. ✅ Observability in `ai_usage_log` + structured logs (see `Module1_Copilot_Tool_Test_Questions.md`)  
4. 🟡 Golden questions in progress — **17/27 tools measured**, 17 asks (see `Module1_Copilot_Tool_Test_Questions.md`); top payloads: `get_revenue_summary` 4,404, `get_menu_health` 4,267, `get_operations_overview` 3,168; **blocker:** `ai/staff-ops` 404  
5. ⬜ Split dashboard projectors + update prompt routing  
6. ⬜ Slim composite AI routes if logs still show fat payloads  
7. ⬜ Billing split (`get_payment_totals` / `get_payment_days`)  
8. ⬜ Optional: provider prompt caching  

## `ai_usage_log` fields (new)

| Field | Meaning |
|---|---|
| `requestId` | Correlate with HTTP / logs |
| `dateRange` | Effective window (`source`: request \| nl) |
| `promptMetrics` | Chars/tokens of initial messages before agent loop |
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
