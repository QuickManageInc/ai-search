# QuickManage Copilot — documentation index

This folder holds **Module 1** planning, measurement, and implementation notes for **QuickManage Copilot** (merchant analytics assistant in `ai-edge-api` + portal).

Use this file to find **what each doc is for**, **what we decided**, and **what to read next**.

---

## Quick start — reading order

| If you want to… | Read |
|-----------------|------|
| Understand product direction (Square/Toast-style copilot) | [Module1_Copilot_Deep_Dive.md](./Module1_Copilot_Deep_Dive.md) |
| See all tools and data honesty rules | [Module1_Copilot_Tools_Inventory.md](./Module1_Copilot_Tools_Inventory.md) |
| Map domains → analytics routes → tools | [Module1_Copilot_Domain_Metrics_Map.md](./Module1_Copilot_Domain_Metrics_Map.md) |
| Cut LLM tokens (payloads + tools) | [Module1_Copilot_Slim_Split_Plan.md](./Module1_Copilot_Slim_Split_Plan.md) + [Module1_Copilot_Intent_Tool_Filter_Plan.md](./Module1_Copilot_Intent_Tool_Filter_Plan.md) |
| Run golden tests and record baselines | [Module1_Copilot_Tool_Test_Questions.md](./Module1_Copilot_Tool_Test_Questions.md) |
| Original build spec (historical) | [../Module1_Analytics_Assistant_Plan.md](../Module1_Analytics_Assistant_Plan.md) |

---

## Files in this folder (`ai-search/copilot/`)

### [README.md](./README.md) — this file

**What it is:** Index and narrative map of all Copilot docs.  
**What we discussed:** How docs relate; recommended reading order; status of token/routing work; links to code.

---

### [Module1_Copilot_Deep_Dive.md](./Module1_Copilot_Deep_Dive.md)

**What it is:** Product and architecture deep dive — from Reports-only chat toward a **global store copilot**.  
**Topics covered:**

- What already ships (tool loop, slim DTOs, SSE, sessions, rate limits)
- Competitive landscape (Square AI, Toast IQ, R365)
- Three tool families: **store facts**, **platform help**, optional neighborhood/web
- Global FAB vs Reports-only surface
- What **not** to copy yet (write-from-chat, accounting, guest kiosk LLM)

**Code touchpoints:** `ai-edge-api`, portal shell, `analyticsTools.ts`  
**Status:** Direction doc — still valid for product scope.

---

### [Module1_Copilot_Tools_Inventory.md](./Module1_Copilot_Tools_Inventory.md)

**What it is:** Tool inventory + **data-reality audit** before adding tools.  
**Topics covered:**

- Baseline 23→27 live tools by family
- **A4 honest-no** contract when data is missing or fake (e.g. discount rate placeholder)
- Candidate future tools (`get_void_summary`, platform help, daily highlights feed)
- Which aggregates need CDC/edge fixes before the LLM should speak

**What we discussed:** Never ship a tool whose backing field is a heuristic; voids are real; discount count/rate was fake until aggregator fix.  
**Code touchpoints:** `analytics-edge-api` billing service, `analyticsTools.ts`

---

### [Module1_Copilot_Domain_Metrics_Map.md](./Module1_Copilot_Domain_Metrics_Map.md)

**What it is:** Data-first map: **analytics-service → analytics-edge-api → ai-edge-api tools** per domain.  
**Topics covered:**

- Menu / revenue / payments / staff / workforce / reservations — what exists at each layer
- Staff metrics already aggregated but not always exposed to Copilot
- Highest-ROI adds: voids daily trend, real `discountCount` in CDC

**What we discussed:** Prefer wiring existing edge routes over new domains; inventory deferred until CDC → `analyticsDB`.  
**Code touchpoints:** `analytics-service`, `analytics-edge-api`, slim mappers

---

### [Module1_Copilot_Slim_Split_Plan.md](./Module1_Copilot_Slim_Split_Plan.md)

**What it is:** Plan to shrink **tool result JSON** before the model sees it.  
**Topics covered:**

- Principle: `edge fetch (fat) → request cache → projector (thin) → LLM`
- Dashboard split: `get_revenue_totals`, `get_best_worst_days`, `get_kitchen_activity`, thin `get_revenue_summary`
- Observability fields in `ai_usage_log`
- Rollout order and smoke checklist

**What we discussed / shipped:**

- ✅ Per-ask fetch memo, observability, dashboard slim-split (`get_revenue_summary` ~490 chars vs ~4,400)
- ✅ Tab-based tool filtering (now being **replaced** by intent — see Intent plan)
- ⬜ Billing / ops composite slim, Gemini caching

**Code touchpoints:** `ai-edge-api/src/context/slim/`, `analyticsTools.ts`, `usageLog.repository`

---

### [Module1_Copilot_Intent_Tool_Filter_Plan.md](./Module1_Copilot_Intent_Tool_Filter_Plan.md)

**What it is:** **Next implementation plan** — replace Reports **tab context** with **question-intent tool expansion**.  
**Topics covered:**

- Why tab filtering saves tokens but breaks routing (phantom tool calls)
- Claude Code **ToolSearch / defer_loading** and Cursor **file-based MCP discovery** (patterns from `v1/`)
- Design: **CORE ~9 tools** + keyword category expansion, cap ~15, ignore portal tabs
- Prompt fixes: dynamic tool list, remove dead tool names, shorten system prompt
- Env: `AI_TOOL_FILTER=intent`

**What we discussed:** User does not want tab-driven context; ~85% of tokens are schemas; intent router is best fit for 27 Gemini tools.  
**Status:** **Planned — implement next**  
**Code touchpoints:** `toolFocusMap.ts`, `copilot.service.ts`, `copilot.ts` (prompts)

---

### [Module1_Copilot_Tool_Test_Questions.md](./Module1_Copilot_Tool_Test_Questions.md)

**What it is:** **Golden test matrix** — questions, expected tools, token baselines, Mongo queries.  
**Topics covered:**

- How to test (new conversation, Feb 3–28 range, check logs + `ai_usage_log`)
- **Input token anatomy** — schemas ~85%, system ~14%
- Measured results per tool (resultChars, inputTokens, requestIds)
- Routing quirks (stacking summary after mix, staff_ops 404 fallbacks)
- Run-next checklist

**What we discussed / measured (2026-08-29 batch):**

- 17/27 tools measured; Gemini 3.5 + AI SDK v7 tool loop working
- Slim-split verified: `get_revenue_summary` **490 chars**, ~**4,244** inputTokens with revenue tab filter
- Tab mismatch failures documented (menu/payment/labor asks on Revenue tab)
- Blocker: `ai/staff-ops` 404 on dev

**Status:** Living doc — update after intent mode ships and remaining golden questions run.

---

## Related docs outside this folder (`ai-search/`)

| File | Purpose |
|------|---------|
| [Module1_Analytics_Assistant_Plan.md](../Module1_Analytics_Assistant_Plan.md) | Original end-to-end build plan for `ai-edge-api` (architecture, API, deployment). Historical + still useful for service layout. |
| [Module1_Context_Optimization_Plan.md](../Module1_Context_Optimization_Plan.md) | Phase A/B: move from fat ContextBuilder preload to **tool calling + slim DTOs**. Predates current observability; strategy still applies. |
| [Module1_Response_Accuracy_Plan.md](../Module1_Response_Accuracy_Plan.md) | Tool coverage, contrastive descriptions, composite AI routes (`ai/menu-health`, etc.), follow-up UX. |

---

## Code map (where docs meet implementation)

| Area | Path |
|------|------|
| Ask handler + SSE | `ai-edge-api/src/services/copilot.service.ts`, `handlers/copilot.handler.ts` |
| Tool definitions | `ai-edge-api/src/tools/analyticsTools.ts` |
| Tool filtering | `ai-edge-api/src/tools/toolFocusMap.ts` → **intent mode next** |
| System prompt | `ai-edge-api/src/prompts/copilot.ts` |
| Slim projectors | `ai-edge-api/src/context/slim/` |
| LLM + multi-step loop | `ai-edge-api/src/llm/LLMClient.ts` |
| Usage / observability | `ai-edge-api/src/repositories/usageLog.repository.ts`, `utils/llmObservability.ts` |
| Analytics data | `analytics-edge-api/` routes under `/api/v1/analytics/` |
| Portal Copilot UI | `quickmanage-merchant-portal/` (Reports panel + FAB) |
| Reference: Claude Code tool deferral | `v1/src/utils/toolSearch.ts`, `v1/src/tools/ToolSearchTool/` |

---

## Conversation timeline (what we decided when)

| Date / phase | Decision |
|--------------|----------|
| Module 1 build | One service (`ai-edge-api`); tool calling over atomic analytics routes; slim DTOs |
| Context optimization | Drop preload ContextBuilder; agent picks tools per question |
| Accuracy pass | Composites (`get_menu_health`, `get_revenue_diagnosis`), contrastive tool descriptions |
| AI SDK v7 + Gemini 3.5 | Fix `thought_signature` / multi-step tool loop |
| Observability | `ai_usage_log` + structured logs (`copilot.llm.input.before_call`, `toolResults`) |
| Slim-split (dashboard) | Thin `get_revenue_summary`; atomic totals / best-worst / kitchen |
| Tab tool filter (`hints`) | Shipped — saves ~3.5k tok/step but **wrong routing** when tab ≠ question |
| Dev auth | `AUTH_DISABLED` on analytics-edge-api for local JWKS bypass |
| **Intent filter (next)** | Replace tab driver with question keywords + CORE tools; dynamic prompt tool list |

---

## Current priorities

1. **Implement intent tool filter** — [Intent plan](./Module1_Copilot_Intent_Tool_Filter_Plan.md)
2. **Finish golden questions** without tab context — [Test questions](./Module1_Copilot_Tool_Test_Questions.md)
3. **Slim-split billing/ops** — [Slim plan](./Module1_Copilot_Slim_Split_Plan.md)
4. **Fix data honesty** — discount count in CDC — [Domain map](./Module1_Copilot_Domain_Metrics_Map.md) + [Inventory](./Module1_Copilot_Tools_Inventory.md)
5. **Deploy `ai/staff-ops`** — unblocks `get_staff_ops_health`

---

## Env vars (Copilot token / tools)

| Variable | Default | Meaning |
|----------|---------|---------|
| `AI_TOOL_FILTER` | `hints` today → **`intent` after ship** | Tool registry mode |
| `AI_TOOL_FILTER_MAX` | `15` (planned) | Cap tools per ask in intent mode |
| `AI_MAX_STEPS` | `5` | Max LLM tool loop steps |
| `AI_MODEL` | `gemini-3.5-flash-lite` | Primary model |

See `ai-edge-api/README.md` for full list.
