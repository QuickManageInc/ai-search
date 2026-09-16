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
| Eval matrix + tool gaps (industry-style asks) | [Module1_Copilot_Eval_Matrix.md](./Module1_Copilot_Eval_Matrix.md) |
| Plan platform help & task how-tos (export report, schedule, orders) | [Module1_Copilot_Platform_Help_Plan.md](./Module1_Copilot_Platform_Help_Plan.md) |
| Prod launch checklist (smoke, validator, golden) | [Module1_Copilot_Prod_Launch_Plan.md](./Module1_Copilot_Prod_Launch_Plan.md) |
| Harden agent before HITL actions (pins, eval:fetch, action contract) | [Module1_Copilot_Pre_Action_Readiness.md](./Module1_Copilot_Pre_Action_Readiness.md) |
| Read-agent hardening (pins, how-to Redis, eval:fetch, grounding badge) | [Module1_Copilot_Read_Agent_Hardening_Plan.md](./Module1_Copilot_Read_Agent_Hardening_Plan.md) |
| Remaining read-agent steps (multi-turn first, then measure, then eval hygiene) | [Module1_Copilot_Read_Agent_Next_Steps.md](./Module1_Copilot_Read_Agent_Next_Steps.md) |
| Cache / numeral measure snapshot (keep CORE-first; do not widen Ground) | [Module1_Copilot_Measure_Snapshot.md](./Module1_Copilot_Measure_Snapshot.md) |
| HITL write contract (preview / confirm / mutate — design only) | [Module1_Copilot_HITL_Action_Contract.md](./Module1_Copilot_HITL_Action_Contract.md) |
| Deploy to dev (GitOps + AWS secrets for supervisor) | [Module1_Copilot_Deploy_Handoff.md](./Module1_Copilot_Deploy_Handoff.md) |
| AppBar drawer UX + inline charts plan | [Module1_Copilot_UX_Charts_Plan.md](./Module1_Copilot_UX_Charts_Plan.md) |
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
**Status:** **Shipped** (2026-08-30) — `AI_TOOL_FILTER=intent` default  
**Code touchpoints:** `questionIntent.ts`, `toolFocusMap.ts`, `copilot.service.ts`, `copilot.ts` (prompts)

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

### [Module1_Copilot_Eval_Matrix.md](./Module1_Copilot_Eval_Matrix.md)

**What it is:** Industry-style **eval question matrix** (~30 singles + multi-turn + guardrails) plus a **gap-only tool list**.  
**Topics covered:**

- Microsoft / Salesforce / Intercom / τ-bench categories mapped onto CORE vs EXISTS vs NONE
- Phase-2 run order (IDs 13–30) after launch smoke 12/12
- Tools that exist but are unreachable in intent CORE; tools that should not be built yet

**Status:** Spec + runner — `bun run eval:matrix` in `ai-edge-api` (`tests/run-eval-matrix.ts`; fetch/ground on, `--route-only` to disable).

---

### [Module1_Copilot_Pre_Action_Readiness.md](./Module1_Copilot_Pre_Action_Readiness.md)

**What it is:** Gate checklist before **HITL write actions** — what to discuss, test, and ship on the read-only agent first.  
**Topics covered:**

- Current status (CORE 12/12, pins honored, fetch/ground in `eval:matrix`)
- Four-layer eval (route / fetch / loop / ground) and `ai_usage_log` era filter
- Must / should / defer lists for final read-only setup
- Next threads: **(A)** pin-honor, **(B)** `eval:fetch` assertions, **(C)** first HITL action shape

**Status:** Agenda — implementation locked in the read-agent hardening plan.  
**Code touchpoints:** `toolFocusMap.ts`, `questionIntent.ts`, `run-eval-matrix.ts`, `validateAnswerAgainstToolFacts.ts`

---

### [Module1_Copilot_Read_Agent_Hardening_Plan.md](./Module1_Copilot_Read_Agent_Hardening_Plan.md)

**What it is:** Locked **read-only** implementation plan (2026-09-13): honor pins, how-to-only Redis cache, eval fetch/ground, compare DTO + grounding badge.  
**Topics covered:**

- CORE-first pin-union (do not A–Z sort the union); A-list + R4 mix patterns
- Redis: cache how-to/capabilities only; key = question + locale + **storeFeatures**, not `storeId`
- `--strict` is not “done”; fetch/ground + multi-turn required
- SSE late badge; do not buffer; writes out of scope

**Status:** Steps 1–4 landed (pin-honor, how-to Redis, eval fetch/ground, compare `deltaAmount` + portal badge). Closeout: [Next steps](./Module1_Copilot_Read_Agent_Next_Steps.md) steps 1–3 done; HITL [design](./Module1_Copilot_HITL_Action_Contract.md) only.  
**Code touchpoints:** `toolFocusMap.ts`, `responseCache.ts`, `run-eval-matrix.ts`, `validateAnswerAgainstToolFacts.ts`, portal Copilot meta

---

### [Module1_Copilot_Read_Agent_Next_Steps.md](./Module1_Copilot_Read_Agent_Next_Steps.md)

**What it is:** Closeout after hardening 1–4 — **what to do next, in order**, and why.  
**Topics covered:**

- Why 18/18 first-turn is not “read agent done”
- **First:** multi-turn M1 / M5 / M6 (`sessionId`, must re-tool)
- **Then:** measure Gemini `cachedInputTokens` + numeral coverage (no product change until data)
- **Also:** eval hygiene (EXISTS fail copy, T3 NL vs golden dates)
- What not to add (HITL, extra tools, wider Ground)

**Status:** Steps 1–3 done. HITL **design** in [Action contract](./Module1_Copilot_HITL_Action_Contract.md); mutate code not started.  
**Code touchpoints:** `tests/run-eval-multiturn.ts`, `tests/measure-usage-log.ts`, `tests/run-eval-matrix.ts`, `SessionManager.ts`, `ai_usage_log`

---

### [Module1_Copilot_Measure_Snapshot.md](./Module1_Copilot_Measure_Snapshot.md)

**What it is:** Step 2 log snapshot — Gemini `cachedInputTokens` CORE-12 vs pinned, and `$`/`%` numeral coverage.  
**Decisions:** Keep CORE-first mixed schemas (no two-call pin stage). Do not widen Ground to counts/dates.  
**Code touchpoints:** `tests/measure-usage-log.ts`, `ai_usage_log`

---

### [Module1_Copilot_HITL_Action_Contract.md](./Module1_Copilot_HITL_Action_Contract.md)

**What it is:** Thread C — preview / confirm / mutate / audit **before** any write tool.  
**Locks:** Confirm is portal HTTP, not a Gemini tool; `plan_id` single-use; compare-and-set; idempotency; read-back; blast radius 1; fail closed. v1.5 = deep-link, then optional Social Publisher **draft** (not publish).  
**Status:** Design only. No mutate code.

---

### [Module1_Copilot_Platform_Help_Plan.md](./Module1_Copilot_Platform_Help_Plan.md)

**What it is:** Plan for **Family 2 — platform / management help**: deep how-to for static product workflows (export report PDF, schedule shifts, check orders, MEV, automation).  
**Topics covered:**

- Current `platformHelp.ts` gaps vs portal routes
- Three-layer helper model (capabilities → feature how-to → **task how-to**)
- v1 ToolSearch lessons applied to platform help (always CORE; search over static catalog)
- New tool ideas without inventory; deep-link-only management actions
- Build phases, golden questions, code touchpoints

**Status:** Phase A–B shipped (2026-09-01) — see [Platform help plan](./Module1_Copilot_Platform_Help_Plan.md).  
**Code touchpoints:** `platformHelp.ts`, `platformTasks.ts`, `questionIntent.ts`, `analyticsTools.ts`

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
| **Intent filter** | Replace tab driver with question keywords + CORE tools | ✅ Shipped 2026-08-30 |

---

## Current priorities

1. **HITL (design locked)** — [Action contract](./Module1_Copilot_HITL_Action_Contract.md) — deep-link first; no mutate tools yet
2. **Read-agent closeout** — [Next steps](./Module1_Copilot_Read_Agent_Next_Steps.md) — steps 1–3 done
2. **Read-agent hardening** — [Read-agent hardening plan](./Module1_Copilot_Read_Agent_Hardening_Plan.md) — steps 1–4 landed
2. **Pre-action readiness** — [Pre-action readiness](./Module1_Copilot_Pre_Action_Readiness.md) — HITL still deferred
3. **Prod launch** — [Prod launch plan](./Module1_Copilot_Prod_Launch_Plan.md) — Phase 1 in progress
4. ~~**Implement intent tool filter**~~ — [Intent plan](./Module1_Copilot_Intent_Tool_Filter_Plan.md) ✅
5. ~~**Platform help & task how-tos**~~ — Phase A–C ✅ including For You insights
6. **Finish golden questions** — [Test questions](./Module1_Copilot_Tool_Test_Questions.md)
7. ~~**Slim-split billing/ops/menu**~~ — [Slim plan](./Module1_Copilot_Slim_Split_Plan.md) ✅
8. **Fix data honesty** — discount verify on prod — [Domain map](./Module1_Copilot_Domain_Metrics_Map.md)
9. **Deploy `ai/staff-ops`** — confirm on prod analytics-edge-api

---

## Env vars (Copilot token / tools)

| Variable | Default | Meaning |
|----------|---------|---------|
| `AI_TOOL_FILTER` | `hints` today → **`intent` (shipped)** | Tool registry mode |
| `AI_TOOL_FILTER_MAX` | `15` | Cap tools per ask in intent mode |
| `AI_ANSWER_VALIDATOR_STRICT` | `false` | Append disclaimer when numeric validation fails |
| `AI_MODEL` | `gemini-3.5-flash-lite` | Primary model |

See `ai-edge-api/README.md` for full list.
