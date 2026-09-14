# Copilot — read-agent hardening plan

> **Goal:** Make the **read-only** agent trustworthy (route → fetch → narrate → ground) without starting HITL writes.  
> **Status:** Locked from 2026-09-13 decisions. Implementation next.  
> **Related:** [Pre-action readiness](./Module1_Copilot_Pre_Action_Readiness.md) · [Eval matrix](./Module1_Copilot_Eval_Matrix.md) · [Intent filter](./Module1_Copilot_Intent_Tool_Filter_Plan.md) · [Code map](../architecture/Code_map.md) · **[Next steps (closeout)](./Module1_Copilot_Read_Agent_Next_Steps.md)**

Writes (preview / confirm / mutate) stay out of this plan.

---

## Locked decisions

| # | Decision |
|---|----------|
| 1 | **Missing tool hurts more than invented story right now.** Honor A-list pins first. |
| 2 | **Redis answer cache = how-to / capabilities only.** Do not cache analytics. See [Redis key](#redis-how-to-cache--not-storeid). |
| 3 | **Gemini prefix is not a $ constraint this month.** Pay for CORE (~12) + a few pin schemas. Frozen CORE stays for Flash 3.x implicit cache on **plain CORE** asks; pinned asks may miss. |
| 4 | **SSE: late badge, do not buffer.** Stream as today. Ship `meta.grounding` / a final SSE warning so the portal can show “figures may not match Reports.” |
| 5 | **`--strict` 18/18 is not “read agent done.”** Need route + healthy fetch + grounded `$`/`%` + ≥3 multi-turn re-fetches. |

Do **not**:

- `uniqueSorted()` the CORE∪pins union (puts `get_peak_hours` in the middle and busts the **whole** Gemini prefix).
- Two-step CORE-then-pins (extra TTFT on every EXISTS ask) until `cachedInputTokens` on CORE-only actually dies after pin-union.
- Category dump via `TOOLS_BY_FOCUS` (phantoms / sibling tools).
- Buffer the full answer until grounding (kills streamed UX).
- Call the agent done because tool-name 18/18 passed (Sep-7 edge-down looked like routing success).

---

## Sequence

| Step | What | Done when |
|------|------|-----------|
| **1** | Allowlisted pin-union, CORE-first schema order, `pin_dropped` metric, R4 mix patterns | EXISTS tools callable; CORE-only asks still 12 schemas in CORE-first order — **code landed** (`bun run test:pin-honor`); confirm with `eval:matrix -- --strict` |
| **2** | Redis: how-to / capabilities only | Analytics asks never `ai:cache` hit; how-to can — **code landed** (`bun run test:howto-cache`) |
| **3** | Eval layers 2–4 on the same cases; Redis hit ≠ pass | `toolFailed` / bad compare / grounding fail = red; cache hit = skip — **code landed** (`bun run test:eval-layers`; live `eval:matrix` fetch/ground on, `--route-only` to disable) |
| **4** | Compare slim `this` / `prev` / `delta`; validator always-on + `not_applicable` | Compare grounding usable; how-to is N/A not fake-ok — **code landed** (`bun run test:validator`; portal badge on `grounding.status === failed`) |
| **5** | Measure numeral coverage + `cachedInputTokens` CORE-only vs pinned | Data to keep mixed schemas or not — **landed** ([snapshot](./Module1_Copilot_Measure_Snapshot.md): keep CORE-first; widen Ground **no**) |

Steps 1 and 3 can overlap (pins + eval:fetch in the same PR series). Step 2 is a small cache change; do it with 1 or immediately after so eval is honest.

---

## Step 1 — Honor pins (thread A)

**Status (2026-09-13):** Implemented in `ai-edge-api`. Unit: `bun run test:pin-honor`. Live: `bun run eval:matrix -- --strict`.

### Problem

`resolveActiveToolNames` (intent) starts from `CORE_TOOL_NAMES`, computes `pinnedToolNames`, then adds a pin **only if it is already in CORE**. `get_peak_hours` is logged and never registered. `capToolNames` exists and is unused. `filterAnalyticsTools` then **A–Z sorts** the allowlist for Gemini `functionDeclarations`.

### Design

1. Keep `CORE_TOOL_NAMES` as the default registered set (same 12, **same relative order every CORE-only ask**).
2. Union **allowlisted** pins into `activeToolNames` for that ask only.
3. Cap with `AI_TOOL_FILTER_MAX` (default 15): CORE first, then pins, then drop non-core extras. Never drop a pin that was in the union if it still fits the cap (existing `capToolNames` safety).
4. **Registration order:** CORE names in `CORE_TOOL_NAMES` order, then extra pins in a stable order (e.g. allowlist order). **Do not** `uniqueSorted()` the whole set.
5. Metric **`pin_dropped`**: pin computed, not in allowlist **or** not in `activeToolNames` after cap. Separate from `ai_routing_phantom_total`.
6. Optional: **`intent_no_pin`** when a category matched and `pinnedToolNames` is empty (keyword miss, e.g. R4 before the mix pattern).

### A-list allowlist

| Tool | Eval | Notes |
|------|------|--------|
| `get_payment_overview` | S5 | Distinct from CORE `get_payment_details` |
| `get_operations_overview` | S6 | Completion vs diagnosis |
| `get_reservations_summary` | S7 | Already golden-tested |
| `get_revenue_mix` | R4 | Needs new pin patterns (below) |
| `get_peak_hours` | R5 | Diagnosis has a weaker `peakHours` |
| `get_fulfillment` | L2 | Distinct from CORE `get_kitchen_activity` |

Do not allowlist every atomic (`get_underperforming_items`, staff performance, …) in this pass. Composites already cover vague asks.

### R4 pin patterns

Today `get_revenue_mix` pins only on `channel mix` / `revenue mix` / `order-type mix`. Eval question is dine-in / takeout / delivery → `pinnedToolNames: []`.

Add patterns (examples): `\bdine-?in\b`, `\btakeout\b`, `\btake-?away\b`, `\bdelivery\b` together with share/mix/split/vs, or a dedicated “channel / order type share” rule that still maps to `get_revenue_mix`.

### Gemini cache expectation

| Ask class | Schemas | Implicit cache |
|-----------|---------|----------------|
| Plain CORE (S1, T2, D1, …) | 12, CORE-first | Should still hit if prefix bytes match today |
| Pinned EXISTS | 12 + 1–3 extras **after** CORE | Miss on the extra tail — accepted |

`cachedInputTokens` / `cacheHitRatio` on CORE-only vs pinned: log now, decide later (step 5). No two-call pin stage in this plan.

### Code

- `ai-edge-api/src/tools/toolFocusMap.ts` — intent branch + `filterAnalyticsTools` order
- `ai-edge-api/src/tools/questionIntent.ts` — R4 patterns; allowlist can live next to `TOOL_PIN_RULES`
- Prometheus: `ai_routing_pin_dropped_total` (or equivalent)

### Verify

- `bun run eval:matrix -- --strict` — EXISTS tools in `toolsCalled`
- `bun run smoke:launch` still 12/12
- Logs: CORE-only `registeredToolCount: 12`; pinned asks 13–15; `activeToolNames` CORE prefix unchanged

---

## Step 2 — Redis how-to-only cache

**Status (2026-09-13):** Implemented in `ai-edge-api`. Unit: `bun run test:howto-cache`. Analytics asks skip Redis; how-to keys are `ai:howto:{hash}` (question + locale + feature fingerprint, not storeId). Eval HOWTO/POLICY Redis hits print `↷ cache-skip`.

### Problem

Exact-match Redis cache stores **plain answer text**, first turn, keyed by `storeId + question + dates + tool names`. Analytics answers can be wrong-but-cached (soft-fail, half-compare). Eval sees `0` tok and `toolsCalled: []`. Cache hits skip Gemini and skip grounding.

### Rule (option A)

**Write** a Redis answer **only if** every `toolsCalled` entry is platform-only:

`get_platform_capabilities` | `get_task_howto` | `search_platform_help` | `get_feature_howto`

No analytics tool in the turn → eligible. Mixed “how-to + sales” → **do not cache**.

**Read** the same way: if the would-be key is analytics-shaped, skip lookup (or never write those keys so lookup misses).

Keep `shouldCacheCopilotResponse` (empty / `toolFailed` / outage copy). How-to should never `toolFailed` on HTTP; still don’t cache empty.

### Redis key — not `storeId`

How-to / capabilities come from a **static catalog** (`platformTasks.ts`, `platformHelp.ts`). The answer does **not** depend on which merchant asked, or on the analytics date range.

`storeId` in the key **partitions identical how-tos** (one cache entry per store) for no correctness reason. Dates in `cacheContextKey` are also irrelevant for these asks.

**Do not** use a fully global key with only the raw question. Task DTOs honor **`storeFeatures`** (`requiresFeature`: `reservationManagement`, `socialPublisher`, …). Store A without Social Publisher must not receive Store B’s “how to post” copy.

Recommended key (how-to only):

```
ai:howto:{sha256(normalize(question) + "|" + locale + "|" + featureFingerprint)}
```

| Part | Why |
|------|-----|
| Normalized question | Same FAQ across merchants |
| Locale | Future FR/EN; default `en` until bilingual |
| Feature fingerprint | Sorted `key=true` for flags that affect `PLATFORM_TASKS` / capabilities (at least `reservationManagement`, `socialPublisher`). Missing flag = false. |

**Omit:** `storeId`, `dateRange`, CORE tool-name list (how-to always has the same platform CORE tools).

Analytics: **no Redis answer key** (or a write that always no-ops). Sessions still per store in Redis; that is unrelated.

TTL can stay ~1h or longer for how-to (catalog is versioned in git; flush on deploy if copy changes). Optional: include a catalog version string in the hash when help content ships often.

### Eval

Redis how-to hit → **skip** the case (or pass-if-text-matches), never treat `toolsCalled: []` + `0` tok as a routing fail or a routing pass.

### Code

- `ai-edge-api/src/cache/responseCache.ts`
- `ai-edge-api/src/services/copilot.service.ts` (`cacheContextKey`, lookup/write guards)

---

## Step 3 — Eval layers 2–4 (thread B, read-only)

**Status (2026-09-14):** Implemented in `ai-edge-api`. Unit: `bun run test:eval-layers`. Live: `bun run eval:matrix` (fetch/ground always-on). `--route-only` disables layers 2–4. Redis how-to hit = `↷ cache-skip`. EXISTS fetch/ground only fails the exit code with `--strict`.

`--strict` = A-list pins **callable**. That is necessary and **not sufficient**.

| Gate | Meaning |
|------|---------|
| `eval:matrix` (no `--strict`) | Daily: CORE / HOWTO / POLICY |
| `--strict` | Route: A-list pins actually called |
| Same script + fetch/ground | `!toolFailed`; no `compareError` + confident compare; grounding fail = case fail; Redis hit = skip |
| Then “done” | Plus ≥3 `sessionId` follow-ups that tool again |

Sep-7: analytics down, `fetchMs` 1–5, `resultChars` 97, routing still “✓”. Fail **fetch**, not only tool name.

### Assertions (same SSE / `requestId`)

| Layer | Fail if |
|-------|---------|
| Route | Expected tool missing (strict EXISTS); POLICY analytics tools present |
| Fetch | Any `toolFailed`; compare overlay asked and compare payload missing / `compareError`; resultChars in the 97-char error band |
| Loop | Analytics steps ≫ 2–3; HOWTO not search→howto→text (when live, not cache skip) |
| Ground | `answerValidation.ok === false` on analytics; `$`/`%` in answer after failed tools |

Join `ai_usage_log` when SSE meta is thin (`llmSteps`, `toolResults[].toolFailed`).

Multi-turn (after pin + fetch gates): at least compare follow-up, collected-after-tips, prep-after-kitchen — `sessionId` must **call a tool again**.

### Code

- `ai-edge-api/tests/evalLayers.ts` + `tests/run-eval-matrix.ts` (`--route-only` to disable fetch/ground)
- SSE `meta`: `toolFailed`, `compareError`, `compareOverlay`, `stepCount`, `grounding`, slim `toolResults`

Portal: consume `meta.grounding` for the late badge (step 4). Eval uses the same field.

---

## Step 4 — Compare DTO + validator + SSE badge

**Status (2026-09-14):** Implemented. Locked: **add `deltaAmount`, do not rename `current`/`compare`**; zero `$`/`%` mentions stay **passed**; portal badge **only** when `grounding.status === failed`.

### Compare slim

Keep existing `current` / `compare` / `changePct`. Add **`deltaAmount`** (`current − compare`) on USD compare metrics so the model copies dollar deltas Ground can allowlist. Do not let the validator invent arithmetic.

| Surface | Field |
|---------|--------|
| `get_period_comparison` (`ai/compare-periods`) | `metrics.revenueUsd.deltaAmount`, `metrics.aovUsd.deltaAmount`, `byChannel[].deltaAmount`. **Not** on `orders` (count ≠ money). |
| `get_revenue_by_day` overlay | `compareTotals.revenueUsd.{current,compare,changePct,deltaAmount}` (period-level). No per-day point deltas. |

If the prior window fetch fails: set `compareError`, **do not Redis-cache**, narrate “this period only; comparison unavailable.” Overlay already does this; whole-tool fail on period comparison is still `toolFailed`.

### Validator

Always run. Distinctions:

| Result | Meaning |
|--------|---------|
| `passed` | Analytics tools + usable DTO; `$`/`%` in answer ⊆ facts |
| `failed` | Unmatched `$`/`%` |
| `not_applicable` | Platform-only how-to / capabilities. Usage log always persists `status` + `skipReason`. Zero `$`/`%` mentions on analytics stay **passed**, not N/A. |

Stop treating `skipped: true` as silent success in logs. Keep usage `answerValidation.status` (or `skipped` + `skipReason` **always persisted**).

Default remains log + SSE badge, not rewrite of already-streamed deltas. `AI_ANSWER_VALIDATOR_STRICT` may still append disclaimer to **session** text.

Widen counts/dates/superlatives is **after** this step (measure coverage first, step 5). Tuple (value + period) is a follow-on once `period` is always on slims.

### SSE / portal (late badge)

Do not buffer.

After Ground, `sseDone` meta includes e.g.:

```json
{
  "grounding": {
    "status": "passed" | "failed" | "not_applicable",
    "unmatchedCount": 0
  }
}
```

Portal: if `failed`, badge the bubble (“figures may not match Reports”). Merchant already saw streamed text — accepted for reads.

### Code

- Slim / `get_period_comparison` / `get_revenue_by_day` compare path
- `validateAnswerAgainstToolFacts.ts` + `copilot.service.ts` SSE meta
- `quickmanage-merchant-portal` Copilot message UI (badge only)

---

## Step 5 — Measure (no extra product change until data)

**Status (2026-09-14):** Snapshot recorded. Keep CORE-first mixed schemas. Do not widen Ground. Re-run `bun run measure:usage`.

On CORE-12 intent, `cached: false`, `!toolFailed`:

- Fraction of answer numerals that current `$`/`%` regex covers vs all digits (usage log sample).
- `cachedInputTokens` / `cacheHitRatio` for `registeredToolCount === 12` vs `> 12`.

Then decide: keep CORE-first mixed schemas, or revisit two-call / ToolSearch. Not a ship gate for steps 1–4.

**Decision:** CORE-12 cache **0/96** in this corpus (pinned **0/51**). Historical CORE-10 *did* cache (~1700 tok). Two-call pin stage would add TTFT with no prefix to reuse. `$`/`%` covers **45.7%** of answer digits; leftovers are years/days/counts — widen Ground **no**. Details: [Measure snapshot](./Module1_Copilot_Measure_Snapshot.md).

---

## Out of scope (this plan)

- HITL preview / confirm / mutate / plan_id
- `find_analytics_tools` / v1 ToolSearch in the live loop
- Structured `{ claims[], prose }` (fights streaming)
- Full SSE buffer until Ground
- Embedding intent router
- Registering all ~27 tools
- Causal-strip post-pass (prompt-only until writes)

---

## “Read agent done” checklist

- [ ] `smoke:launch` 12/12 on a healthy stack
- [ ] `eval:matrix` CORE/HOWTO/POLICY green
- [ ] `--strict` EXISTS reachable (pins + R4)
- [x] Fetch/ground assertions fail on edge-down and invented `$`/`%`
- [ ] Analytics Redis answer cache off; how-to cache feature-fingerprinted, not per-`storeId`
- [ ] ≥3 multi-turn cases re-tool
- [x] Portal badge on `grounding.status === failed`
- [ ] Phantom ≈ 0; `pin_dropped` only for non-allowlist pins

---

## Daily commands (after this lands)

| Command | Role |
|---------|------|
| `bun run smoke:launch` | 12/12 CORE |
| `bun run eval:matrix` | Daily CORE/HOWTO/POLICY **plus** fetch/ground (`toolFailed` / compare overlay / grounding). Cache skip ≠ pass |
| `bun run eval:matrix -- --strict` | Pins callable; EXISTS fetch/ground fail the exit code |
| `bun run eval:matrix -- --route-only` | Tool/policy names only |
| `bun run test:eval-layers` | Unit fetch/ground judges |
| `bun run test:validator` | Unit grounding regex / allowlist |
