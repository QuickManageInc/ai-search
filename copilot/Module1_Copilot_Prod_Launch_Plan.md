# Copilot — production launch plan

Status: **Phase 1 code complete** — Phase 2 in progress (local-only; prod deploy blocked)  
Audience: engineering + deploy  
Related: [Platform help](./Module1_Copilot_Platform_Help_Plan.md), [Intent filter](./Module1_Copilot_Intent_Tool_Filter_Plan.md), [Golden tests](./Module1_Copilot_Tool_Test_Questions.md)

---

## No prod access?

Engineering can still validate locally:

| Step | Command |
|------|---------|
| Validator unit self-test | `cd ai-edge-api && npm run test:validator` |
| Health + insights + one ask | `npm run prod:smoke` (against local or staging URL) |
| Full 12-ask launch matrix | `npm run smoke:launch` (~2 min, needs local APIs + Gemini key) |
| Platform golden batch | `npm run golden:platform` |
| Remaining routing batch | `npm run golden:remaining` |

Hand deploy team: [Env vars](#env-vars-prod-checklist) + Phase 1.5 ops checklist below.

---

## Decisions (locked)

| Question | Decision |
|----------|----------|
| Timeline | **Prod merchants soon** |
| Token optimization (Phase E) | **After all features** — ~8k/how-to acceptable for v1 |
| For You cards | **3-card REST feed is enough for v1** — add compliance/reservations sources in v1.1 |
| Rollout | TBD — pilot stores vs all (`VITE_ENABLE_AI_ASSISTANT`) |

---

## v1 prod definition

Copilot is **prod-ready** when:

1. Merchant can ask **analytics + how-to** from the global FAB and get **accurate or honest** answers.
2. Home dashboard shows **For You** cards when AI is enabled (`GET /api/v1/ai/copilot/insights`).
3. **12-ask smoke** on prod passes with no P0 wrong-number failures.
4. Failures degrade gracefully (auth, analytics down, empty data).

**Explicitly not blocking v1:** Phase E token cuts, ToolSearch meta-tool (`v1/`), inventory analytics, write-from-chat.

---

## Phase 1 — Must-do before merchants

### 1.1 Prod smoke

| Check | Command / route |
|-------|-----------------|
| Readiness | `GET /health/ready` |
| Insights | `GET /api/v1/ai/copilot/insights?storeId=` |
| Ask (SSE) | `POST /api/v1/ai/copilot/ask` |
| Gateway | `/api/v1/ai/*` → `ai-edge-api`, SSE idle timeout ≥ 60s |
| Data deps | Redis + Mongo + `analytics-edge-api` + `GOOGLE_GENERATIVE_AI_API_KEY` |

**Script:** `cd ai-edge-api && npm run prod:smoke` (local/staging) or `npm run smoke:launch` for full 12-ask matrix

### 1.2 Trust — numeric answer validator

Port the pattern from `social-publisher-api/src/validation/validateDraftAgainstFacts.ts`:

- After LLM stream, compare **$ amounts** and **% mentions** in the answer against numeric fields in tool JSON.
- **Skip** platform-only asks (no analytics tools called).
- **Log** mismatches to `ai_usage_log` + Prometheus; strict rewrite optional via `AI_ANSWER_VALIDATOR_STRICT=true`.

**Code:** `ai-edge-api/src/validation/validateAnswerAgainstToolFacts.ts` → `copilot.service.ts`

### 1.3 Discount / payment honesty

- `analytics-edge-api` now uses real `billing.discountCount` in `billing.service.ts`.
- Slim mapper adds `notes` when discount count is zero but total discounts > 0 (data inconsistency).
- **Prod verify:** ask *"What's our discount rate?"* and compare to Reports → Payments.

### 1.4 Golden closure + routing observability

| Item | Expected |
|------|----------|
| How do I post to social? | `search_platform_help` → `get_task_howto` (`automation.create_post`) |
| Orders in kitchen **now** | `get_kitchen_activity` (not `get_fulfillment`) |
| Phantom tools | Log + metric when model calls tool ∉ `activeToolNames` |

**Scripts:** `npm run golden:platform`, `npm run golden:remaining`

### 1.5 Ops

- [ ] `ai/staff-ops` deployed on prod analytics-edge-api
- [ ] Rate limits tuned (`AI_BURST_LIMIT_PER_MINUTE`, daily cap)
- [ ] `VITE_ENABLE_AI_ASSISTANT` rollout plan documented
- [ ] `ai_usage_log` indexes for prod dashboards

---

## Phase 2 — Nice-to-have (same release if time)

- Demote `get_feature_howto` from CORE (task-first is shipped)
- Flag-gated platform tasks (MEV / automation off → honest `dataAvailable: false`)
- Refresh [copilot README](./README.md) priorities (stale “daily highlights next”)
- Intent plan doc closure (`routing.phantom`, README `AI_TOOL_FILTER=intent`)

---

## Phase 3 — Deferred (post-v1)

| Item | Doc |
|------|-----|
| Token optimization (~8k → ~3k how-to) | Platform help Phase E |
| ToolSearch meta-tool | Intent plan phase 2 |
| Extra For You sources (compliance, reservations) | Domain metrics map |
| Inventory analytics | Blocked on CDC → `analyticsDB` |
| Numeric validator → hard block + user-visible retry | After prod telemetry review |

---

## 12-ask prod smoke matrix

Run on **prod** with a real store. Mark each: **Correct?** **Fast?** **Right tool?**

| # | Question | Bucket |
|---|----------|--------|
| 1 | How did we do vs last week? | Numbers |
| 2 | What's our discount rate and tip rate? | Numbers |
| 3 | How many voids did we have? | Numbers |
| 4 | What were my top selling items? | Numbers |
| 5 | How do I download a report? | Coverage |
| 6 | How do I post to social? | Coverage |
| 7 | How do I export fiscal data? | Coverage |
| 8 | What's wrong with my menu? | Numbers |
| 9 | Who were my top staff by sales? | Numbers |
| 10 | How many orders are in the kitchen right now? | Coverage + honesty |
| 11 | What can you help me with? | Coverage |
| 12 | Why are orders getting cancelled? | Numbers |

**P0 failure:** answer numbers don't match Reports. **P1:** wrong tool but right-ish answer. **P2:** slow or high tokens only.

---

## Code touchpoints

| Area | Path |
|------|------|
| Ask + validator | `ai-edge-api/src/services/copilot.service.ts` |
| Answer validation | `ai-edge-api/src/validation/validateAnswerAgainstToolFacts.ts` |
| Intent / pins | `ai-edge-api/src/tools/questionIntent.ts` |
| Insights (For You) | `ai-edge-api/src/handlers/insights.handler.ts` |
| Golden runner | `ai-edge-api/scripts/run-golden-asks.ts` |
| Prod smoke | `ai-edge-api/scripts/prod-smoke.ts` |
| Portal FAB | `quickmanage-merchant-portal/src/components/ai/AIAssistantPanel.jsx` |
| Dashboard For You | `quickmanage-merchant-portal/src/components/ai/CopilotForYouPanel.jsx` |

---

## Env vars (prod checklist)

| Variable | Notes |
|----------|-------|
| `AI_TOOL_FILTER` | `intent` (default) |
| `VITE_ENABLE_AI_ASSISTANT` | Portal feature flag |
| `ANALYTICS_EDGE_API_URL` | In-cluster DNS in EKS |
| `GOOGLE_GENERATIVE_AI_API_KEY` | Or AWS secret |
| `CORS_ORIGIN` | Portal domain |
| `AI_ANSWER_VALIDATOR_STRICT` | Optional — rewrite answer on mismatch |
| Ask body `storeFeatures` | Optional — `{ reservationManagement, socialPublisher }` from portal |

---

## Progress tracker

### Phase 1

- [x] Launch plan doc (this file)
- [x] Numeric answer validator (`validateAnswerAgainstToolFacts.ts`)
- [x] `routing.phantom` metric (`ai_routing_phantom_total`)
- [x] Kitchen live-queue pin (`get_kitchen_activity`)
- [x] Platform golden: post to social (`golden:platform`)
- [x] `prod:smoke` script (`npm run prod:smoke`)
- [x] Discount slim `notes` guard (`billing.slim.ts`)
- [ ] Manual prod 12-ask smoke (deploy)

### Phase 2

- [x] `get_feature_howto` demoted from CORE (legacy pin on feature id only)
- [x] Flag-gated tasks (`requiresFeature` + portal `storeFeatures`)
- [x] `get_revenue_totals` pin for total sales / order count phrasing
- [x] Local smoke: `smoke:launch` + `test:validator`
- [x] Doc refresh (this file + copilot README)
