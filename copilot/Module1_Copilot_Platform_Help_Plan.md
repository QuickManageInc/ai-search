# Copilot platform help & management how-to plan

Status: **Phase A–B shipped**, **Phase C portal UX shipped** (2026-09-01)  
Token optimization (platform-only registry, skip search) → **deferred** until all features land  
Scope: **Family 2 — platform / management help** (static workflows, deep links, honest boundaries)  
Explicitly **out of scope for now:** inventory analytics tools, write-from-chat actions  
Related: [Module1_Copilot_Deep_Dive.md](./Module1_Copilot_Deep_Dive.md), [Module1_Copilot_Intent_Tool_Filter_Plan.md](./Module1_Copilot_Intent_Tool_Filter_Plan.md), [Module1_Copilot_Tools_Inventory.md](./Module1_Copilot_Tools_Inventory.md)

---

## Why this doc exists

Analytics tools (~33 store-fact tools) are largely measured and routed via intent filtering. Platform help lets merchants ask product questions without touching analytics:

- “How do I download a report?”
- “How do I make next week’s schedule?”
- “How do I check orders in the kitchen?”
- “Where do I export fiscal data?”

These are **static product workflows** — navigation, clicks, and honest limits — not numbers from `analytics-edge-api`.

---

## Current state (shipped)

### Platform tools (CORE — always loaded)

| Tool | File | Role |
|------|------|------|
| `get_platform_capabilities` | `platformHelp.ts` | Slim index: `areas` + `tasks` |
| `get_task_howto(taskId)` | `platformTasks.ts` | Task-level steps + deep link |
| `search_platform_help(query)` | `platformTasks.ts` | Keyword search → task ids |
| `get_feature_howto(featureId)` | `platformHelp.ts` | Legacy area-level how-to |

**Total tools:** ~35 (33 analytics + 4 platform, with overlap in CORE count).

### Task catalog (15 tasks)

| taskId | Portal path |
|--------|-------------|
| `reports.export_pdf` | `/dashboard/reports` |
| `reports.change_date_range` | `/dashboard/reports` |
| `team.create_shift` | `/dashboard/team` |
| `team.approve_time_off` | `/dashboard/team/requests` |
| `orders.check_live_board` | `/dashboard/orders` |
| `orders.update_status` | `/dashboard/orders` |
| `menus.publish_changes` | `/dashboard/menus` |
| `reservations.add_booking` | `/dashboard/reservation` |
| `automation.create_post` | `/dashboard/automation` |
| `mev.export_fiscal` | `/dashboard/mev` |
| `settings.timezone` | `/dashboard/settings` |
| `dashboard.overview` | `/dashboard/home` |
| `customers.manage` | `/dashboard/customers` |
| `devices.pair` | `/dashboard/devices` |
| `profile.language` | `/dashboard/profile` |

### Intent routing (how-to vs analytics)

- `isPurePlatformHowToQuestion` → **no** analytics intent categories
- Pins `search_platform_help` + `get_task_howto` on how-to phrasing
- Mixed asks (“how do I export a report and what were sales”) still expand analytics

### Portal (Phase C)

- FAB: **Ask Copilot**
- Panel subtitle: **Your numbers or how to use QuickManage**
- Suggested prompts: **3 analytics + 3 how-to**
- Tab `context` removed — routing is question-only ([Intent plan](./Module1_Copilot_Intent_Tool_Filter_Plan.md))

### Golden results (manual 2026-09-01)

| Question | Tools | inputTokens | requestId |
|----------|-------|------------:|-----------|
| How do I download a report? | search → task | 8,010 | `0f41225a` |
| How do I check orders in the kitchen? | search → task | 7,939 | `caa879d3` |
| How do I export fiscal data? | search → task | 7,886 | `d06b3b24` |

Run batch: `cd ai-edge-api && npm run golden:platform`

**Token note (deferred):** ~8k inputTokens = 3 LLM steps × 11 CORE tool schemas. Optimization tracked separately — not blocking feature work.

---

## Architecture

### Task-first (not feature-first)

```
Layer 1 — Index          get_platform_capabilities  (areas + task ids)
Layer 2 — Task how-to    get_task_howto(taskId)     ← primary
Layer 3 — Search         search_platform_help(query) ← vague asks
Legacy                   get_feature_howto(featureId)
```

### Three tool families

| Family | Purpose | Examples |
|--------|---------|----------|
| **1 — Store facts** | Numbers from analytics | `get_revenue_summary`, `get_menu_health` |
| **2 — Platform help** | How to use QuickManage | task howto, search (**this plan**) |
| **3 — Neighborhood / web** | Optional later | citations required |

**Inventory** stays deferred until CDC → `analyticsDB`.

---

## Build phases

### Phase A — Expand static catalog ✅

1. [x] `platformTasks.ts` (~15 tasks)
2. [x] `get_task_howto(taskId)`
3. [x] Intent pins + pure how-to guard
4. [x] Reports PDF in `reports.export_pdf`

### Phase B — Search helper ✅

5. [x] `search_platform_help(query)`
6. [x] Prompt rule: how-to → platform tools first

### Phase C — Product surfaces (partial ✅)

7. [x] `get_daily_highlights` — `GET /ai/copilot/insights` + portal “For You” cards on dashboard
8. [x] Suggested prompts mix analytics + how-to
9. [x] Global FAB / panel copy

### Phase D — Analytics additions (data-dependent)

10. [ ] Discount honesty in aggregator (verify prod)
11. [ ] Employee/labor tools when aggregates exist

### Phase E — Token optimization (deferred)

- Platform-only tool registry on pure how-to (11 → 4 tools)
- Direct `get_task_howto` when task id is obvious (skip search)
- Per-step token logging in `copilot.llm.step.finish`

---

## Example merchant flows

### “How do I download a report?”

1. `search_platform_help` → `get_task_howto("reports.export_pdf")`
2. Steps + `/dashboard/reports` + Export PDF button
3. Do **not** call `get_revenue_summary`

### “How do I check orders in the kitchen?”

1. How-to: `orders.check_live_board` (live UI)
2. Honest: no live queue in analytics; historical prep via `get_fulfillment` only if they ask for numbers

---

## Golden test questions

| Question | Expected tools | Status |
|----------|----------------|--------|
| How do I download a report? | search → task | ✅ `0f41225a` |
| How do I check orders? | search → task | ✅ `caa879d3` |
| How do I export fiscal data? | search → task | ✅ `d06b3b24` |
| How do I schedule shifts? | task or feature howto | ✅ `a93ede30` (legacy feature) |
| How do I post to social? | task `automation.create_post` | ✅ in `golden:platform` |

See [Module1_Copilot_Tool_Test_Questions.md](./Module1_Copilot_Tool_Test_Questions.md).

---

## What not to do now

- Inventory analytics tools
- Write-from-chat without HITL
- Token optimization before feature set complete (team choice)
- RAG over random docs — keep versioned catalog files

---

## Code touchpoints

| Area | Path |
|------|------|
| Tasks catalog | `ai-edge-api/src/catalog/platformTasks.ts` |
| Areas catalog | `ai-edge-api/src/catalog/platformHelp.ts` |
| Tool registry | `ai-edge-api/src/tools/analyticsTools.ts` |
| Intent / pins | `ai-edge-api/src/tools/questionIntent.ts` |
| CORE tools | `ai-edge-api/src/tools/toolFocusMap.ts` |
| Golden runner | `ai-edge-api/scripts/run-golden-asks.ts` (`npm run golden:platform`) |
| Portal prompts | `quickmanage-merchant-portal/src/config/aiAssistant.config.js` |
| Portal panel | `quickmanage-merchant-portal/src/components/ai/AIAssistantPanel.jsx` |
| Dashboard For You | `GET /ai/copilot/insights` → `CopilotForYouPanel.jsx` on `DashboardPage.jsx` |

---

## Open questions

1. **`get_daily_highlights`** — ✅ shipped as `GET /api/v1/ai/copilot/insights` (REST only, no LLM tool)
2. **`get_feature_howto`** — remove from CORE once task coverage is trusted?
3. **Automation / MEV** — ✅ `requiresFeature` on tasks + portal `storeFeatures` (`socialPublisher`, `reservationManagement`)
