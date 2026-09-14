# Section 1 — Code map: which file does what

Treat this like any backend request pipeline. The model is not a magic box sitting in the middle of the app. It is one downstream dependency, like calling Stripe or an analytics API. Our service still owns HTTP, auth, routing, fetching, validation, cache, and logging.

This section is only the map. CORE, pins, grounding, and why we designed it this way come in later sections.

---

## 1. HTTP entry

A merchant question hits `POST /api/v1/ai/copilot/ask` (legacy alias: `/api/v1/ai/analytics/ask`).

| File | Function | Job |
|---|---|---|
| `index.ts` | `main` / HTTP `createServer` | Boot Redis + Mongo, then dispatch routes |
| `src/handlers/copilot.handler.ts` | `askHandler` | Parse JSON, require `storeId` + `question` + Bearer token, resolve dates |
| `src/utils/params.ts` | `parseJsonBody`, `resolveDateRange` | Normal request parsing |
| `src/services/copilot.service.ts` | `ask` | The orchestrator. Almost everything below runs here |

`askHandler` does not talk to the model. It is a controller. `ask` is the service.

There is a second product path: `GET .../insights` → `insightsHandler` → `buildDailyHighlights`. Same analytics data, no chat loop.

---

## 2. Setup before the AI loop (still normal backend)

Inside `ask`, before any model call:

| File | Function | Job |
|---|---|---|
| `src/utils/relativeDateRange.ts` | `resolveAskDateRange` | Turn “last week” / “vs last month” into `{ from, to }` |
| `src/session/SessionManager.ts` | `resolveSession` | Load or create the Redis chat session |
| `src/tools/toolFocusMap.ts` | `resolveActiveToolNames` | Preview which tools this question would get (used for cache key) |
| `src/cache/responseCache.ts` | `getCachedResponse` | Exact-match Redis answer cache (first turn only) |
| `src/limits/rateLimit.ts` | `checkAndConsumeRateLimit` | Per-store rate limit, only on cache miss |

If cache hits, we stream the stored text and return. No model, no tools.

---

## 3. The four steps, mapped to functions

This is the loop: **route → fetch → narrate → ground**.

### Route — decide which tools the model is allowed to see

| File | Function | Job |
|---|---|---|
| `src/tools/analyticsTools.ts` | `createAnalyticsTools` | Build the full tool set (~27 wrappers) |
| `src/tools/toolFocusMap.ts` | `filterAnalyticsTools` | Register only the allowed subset on this request |
| `src/tools/toolFocusMap.ts` | `resolveActiveToolNames` | Compute that subset (CORE + optional pins) |
| `src/tools/questionIntent.ts` | `detectCategoriesFromQuestion` | Keyword match: revenue / menu / staff / … |
| `src/tools/questionIntent.ts` | `getPinnedToolNames` | Extra tools named by the question (peak hours, payments, …) |
| `src/tools/toolFocusMap.ts` | `CORE_TOOL_NAMES` | Frozen always-on list (~12) |

Backend analogy: API gateway + allowlist. The model cannot call a tool that was not registered on this request.

### Fetch — when the model picks a tool, we hit analytics-edge and shrink the JSON

| File | Function | Job |
|---|---|---|
| `src/tools/analyticsTools.ts` | `safeExecute` | Run one tool; on failure return `{ error: "..." }` instead of crashing |
| `src/tools/analyticsTools.ts` | `createRequestFetch` / `fetchOnce` | Memoize HTTP so two tools on the same route share one call |
| `src/context/analyticsClient.ts` | `fetchAnalytics` | GET analytics-edge with the merchant Bearer token |
| `src/context/slim/*.ts` | `slimRevenueTotals`, `slimPopularItems`, … | Project fat portal JSON → small DTO for the model |

Backend analogy: each tool is a thin BFF. Math already happened upstream. We only fetch and reshape.

### Narrate — the model writes English from those DTOs, streamed to the client

| File | Function | Job |
|---|---|---|
| `src/prompts/copilot.ts` | `buildToolMessages` / `buildToolSystemPrompt` | System prompt + user turn + history |
| `src/llm/LLMClient.ts` | `streamWithTools` | Gemini + Vercel AI SDK `streamText`, multi-step tool loop |
| `src/services/copilot.service.ts` | `sseWrite` / `startSse` | Stream tokens over SSE |

Backend analogy: we call a vendor API. It may call *our* tools back (in-process). Then it streams text. We still own the HTTP response.

### Ground — after the text exists, we check numbers against tool JSON

| File | Function | Job |
|---|---|---|
| `src/validation/validateAnswerAgainstToolFacts.ts` | `validateAnswerAgainstToolFacts` | Extract `$` and `%` from the answer; must appear in tool output |
| same file | `applyAnswerValidation` | Log mismatches; optional rewrite |
| `src/services/copilot.service.ts` | (inside `ask`, after stream) | Call those two, then finish the response |

Backend analogy: a response validator / assertion layer. Not ML. Regex + JSON walk.

---

## 4. After the loop (still our code)

| File | Function | Job |
|---|---|---|
| `src/charts/buildCopilotChart.ts` | `buildCopilotChart` | Optional chart payload from the same tool facts |
| `src/session/SessionManager.ts` | `appendTurns` / `saveSession` | Persist user + assistant turns in Redis |
| `src/cache/responseCache.ts` | `setCachedResponse` | Cache first-turn answers if they look safe |
| `src/repositories/usageLog.repository.ts` | `insertUsageLog` | Mongo `ai_usage_log`: tools, tokens, latency, grounding result |

---

## How to read this mentally

```
POST /ask
  → askHandler          (HTTP / auth)
  → ask                 (orchestrator)
       → resolveAskDateRange
       → resolveSession
       → cache / rate limit
       → createAnalyticsTools + filterAnalyticsTools   ← ROUTE
       → streamWithTools
            model may call tool.execute
                 → fetchAnalytics + slim*              ← FETCH
            model writes English                       ← NARRATE
       → validateAnswerAgainstToolFacts                ← GROUND
       → chart + session + cache + usage log
```

`src/router.ts` is **not** the AI “route” step. That file is only `/healthz` and `/metrics`. “Route” in this agent means tool selection.


# Section 2 — Design rule: the LLM never computes

## The split of jobs

This agent is a **read-only BFF in front of analytics-edge**, plus a language model that is only allowed to *talk*.

| Layer | Who | Allowed to do | Not allowed to do |
|---|---|---|---|
| Analytics pipeline (CDC → analytics DB → analytics-edge) | Our data stack | All math: totals, %, rankings, day-over-day, period compare | Chat |
| `ai-edge-api` tools + slim DTOs | Our Node service | Pick a route, GET, project JSON, convert cents → major units | Invent figures |
| LLM (Gemini) | Vendor API | Choose a tool from the allowlist, then narrate the JSON in English | Compute, convert currency, invent `$` / `%` |
| Grounding validator | Our Node service | Check that `$` / `%` in the answer already exist in tool JSON | Trust the model |

Backend analogy: the model is a **renderer**, not a calculator. Same idea as a UI that must display `order.total` from the API and must not `sum(lineItems)` in the browser.

That rule is written into the system prompt in `buildToolSystemPrompt`:

- “Always call one or more tools before answering factual questions. Never invent numbers.”
- “Money amounts returned by tools are already in major currency units (`*Amount`). Do not divide by 100 again.”

If the model divides by 100, or invents a wow %, it is violating the contract. Grounding (Section 1, step 4) is the assertion layer for that contract. It is incomplete on purpose today — it only checks `$` and `%`, not counts, ranks, or “because of rain”. That gap is later.

---

## What a “tool” is (backend terms)

A **tool** is not ML. It is a typed HTTP client method we register with the SDK.

`createAnalyticsTools` builds an object of Vercel AI SDK `tool({ description, inputSchema, execute })` entries.

| Piece | Same as in a normal API |
|---|---|
| `description` | OpenAPI summary — the model uses this to pick *which* function to call |
| `inputSchema` (Zod) | Request DTO / query params (`limit`, `compare`, …) |
| `execute` | The handler: `fetchAnalytics` + a `slim*` function |

The model never sees analytics-edge URLs. It emits a structured call like `get_revenue_summary({ })`. Our process runs `execute`, and we put the JSON result back into the model’s context.

That is the same pattern as: client calls `/users/:id` → your controller hits Postgres → returns a DTO. Here the “client” is Gemini.

Platform how-to tools (`search_platform_help`, `get_task_howto`) skip analytics-edge. They read a static catalog. Same *shape* (schema + execute), different *data source*.

---

## What a DTO is here

**DTO** = the small JSON we actually give the model, not the fat portal payload.

Flow:

1. `fetchAnalytics('items/popular', …)` returns the full analytics-edge body.
2. `slimPopularItems(raw)` (or `slimRevenueTotals`, `slimPeakHours`, …) keeps a few fields.
3. `amount()` / `minorToMajor()` in `src/context/slim/money.ts` convert cents → dollars/CAD major units **in our code**, before the model sees them.
4. That slim object is the tool result.

Example: `slimRevenueTotals` keeps `{ period, summary }` from dashboard-summary and drops the rest. One fat route can feed several tools (`slimBestWorstDays`, `slimKitchenActivity`) via `fetchOnce` memoization — one HTTP call, several projections.

Why slim:

- Tokens cost money and attention. A 4k-char payload vs ~500 chars is a cheaper, less noisy prompt.
- Extra fields are extra ways for the model to quote the wrong number.
- Currency conversion is *our* job. If we sent cents, the model would try to `/ 100` and sometimes fail.

---

## What CORE is (backend terms)

**CORE** is a frozen allowlist of tool *schemas* we register on almost every ask.

Defined in `CORE_TOOL_NAMES` (`src/tools/toolFocusMap.ts`):

- Platform: `get_platform_capabilities`, `get_task_howto`, `search_platform_help`
- Analytics composites / essentials: `get_menu_health`, `get_revenue_diagnosis`, `get_staff_ops_health`, `get_period_comparison`, `get_revenue_summary`, `get_revenue_by_day`, `get_void_summary`, `get_payment_details`, `get_kitchen_activity`

Backend analogy: a **default IAM policy** or a **gateway allowlist** that does not change per request.

Why freeze it:

- Gemini can cache the system prompt + tool schema bytes (prompt cache). If the tool list changes every question, that prefix busts and you pay full input tokens again.
- Default `AI_TOOL_FILTER=intent` still *computes* category pins (`getPinnedToolNames`, `detectCategoriesFromQuestion`) for logs and routing hints, but **does not register** pins that sit outside CORE. That is a cache-vs-reachability tradeoff. Details in the Route section.

What CORE is *not*:

- Not a second model.
- Not embeddings / vector search.
- Not “the most important metrics.” It is “the schema set we always send so the prompt prefix stays byte-stable.”

`filterAnalyticsTools` then takes `createAnalyticsTools(...)` (all ~27) and keeps only names in that allowlist, sorted, so `functionDeclarations` stay byte-stable.

---

## How the three terms fit the loop

```
ROUTE   → CORE (and sometimes pins) = which execute functions exist this request
FETCH   → tool.execute → analytics-edge → slim DTO   = numbers come from our API
NARRATE → LLM reads DTOs, writes English             = no new math
GROUND  → regex $ / % ⊆ DTO values                   = we verify the renderer
```

The safety property of the whole design is: **every number in the merchant-facing sentence should have been produced by analytics-edge (or a static catalog), then copied by the model.** The model’s job is tool selection + wording.

---

# Section 3 — Route: which tools exist this request

## What “route” means here

This is not HTTP routing (`src/router.ts` is health/metrics only).

**Route** = compute an allowlist of tool names, then register only those `execute` functions on the Gemini request.

Backend analogy: an API gateway that exposes a subset of endpoints per caller. If `get_peak_hours` is not in the registered `tools` object, the model cannot call it. There is no 404 — the function simply does not exist in that request’s schema.

Call chain:

```
ask()
  → createAnalyticsTools(...)          // build all ~27 wrappers in memory
  → filterAnalyticsTools(all, input)   // keep only the allowlist
       → resolveActiveToolNames(...)   // decide the names
            → getEffectiveQuestionForIntent
            → detectCategoriesFromQuestion   // logs / metrics
            → getPinnedToolNames             // logs; registration is limited (below)
  → streamWithTools({ tools })         // only this subset is sent to Gemini
```

Files: `src/tools/toolFocusMap.ts`, `src/tools/questionIntent.ts`.

---

## Why we do not send every tool every time

Sending ~27 full schemas on every ask is like dumping your entire OpenAPI spec into every client request.

Costs:

- **Tokens.** Each tool schema (name, description, Zod fields) is prompt bytes you pay for.
- **Wrong calls.** A bigger menu → more “phantom” or sibling-tool calls (e.g. calling `get_top_selling_items` when they asked for weak items).
- **Prompt cache busts.** Gemini can reuse a byte-stable prefix (system prompt + tool schemas). If the tool list changes with every question, that prefix changes and the cache misses.

So we freeze a default set (**CORE**) and try to keep that schema block identical across asks.

---

## Filter modes (`AI_TOOL_FILTER`)

`getToolFilterMode()` reads the env. Default is `intent`.

| Mode | What gets registered | When |
|---|---|---|
| `intent` (default) | `CORE_TOOL_NAMES` only, sorted | Production copilot |
| `always` | Same CORE list | Debug / force CORE |
| `hints` | Platform CORE + `TOOLS_BY_FOCUS[tab]` from body `context` | Legacy portal tabs |
| `off` | All tools (`activeToolNames = null`) | Debug only |

In `intent` mode, `askHandler` **ignores** body `context` (Reports-tab hints). Routing is from question text, not from which portal tab is open.

---

## Intent is not ML

`detectCategoriesFromQuestion` is a keyword router: regexes in `CATEGORY_PATTERNS`, scored by hit count, highest first.

Categories: `payment`, `menu`, `staff`, `workforce`, `operations`, `revenue`.

Example: “what are my bestsellers” hits `menu` (`bestsellers`, `top selling`, …). “tips last week” hits `payment`.

If nothing matches → `[]`. Vague questions stay on CORE only. That is intentional.

This is the same idea as an Express route map or a search synonym list. **No embeddings, no second model, no vector DB.**

Special cases in `questionIntent.ts`:

| Function | Job |
|---|---|
| `isPurePlatformHowToQuestion` | “how do I export a report” with no sales ask → skip analytics categories |
| `getEffectiveQuestionForIntent` | “try again” / “redo” reuses the **previous** user question for routing |
| `getPinnedToolNames` | Extra tool names implied by specific phrases (`peak hours` → `get_peak_hours`) |

`TOOLS_BY_FOCUS` maps each category to a list of atomic tools (menu → bestsellers, underperformers, trends, …). **In default `intent` mode that map is not unioned into the registered set.** Categories are still computed and logged (`intentCategories` on the usage log) so you can see what the question *would* have expanded. Registration stays CORE so the schema prefix does not change.

---

## Pins vs CORE (the important split)

**Pin** = “this question named a specific tool, keep it.”

`TOOL_PIN_RULES` examples:

- `peak hours` / `busiest hours` → `get_peak_hours`
- `fulfillment` / `prep time` → `get_fulfillment`
- `collected` / `outstanding` → `get_payment_overview`
- `bestsellers` → `get_top_selling_items`

How-to questions pin `search_platform_help` + `get_task_howto` (those two are already in CORE).

**What the code actually does in `intent` mode** (`resolveActiveToolNames`):

1. Start with `CORE_TOOL_NAMES`.
2. Compute `pinnedToolNames` from the question.
3. Add a pin **only if it is already in CORE**.

So `get_payment_details` (CORE) can be “pinned” and it is a no-op — already registered. `get_peak_hours` (not CORE) is computed, logged as `pinnedToolNames`, and **not registered**.

That is the cache-vs-reachability tradeoff from Section 2. Prompt cache stays stable. The long-tail tool is invisible to the model.

`capToolNames` + `AI_TOOL_FILTER_MAX` exist in `toolFocusMap.ts` (CORE first, then pins, then category expansion, then trim). **Nothing calls them today.** They are leftover from a CORE+pins union that was not wired into the default path.

---

## What the model actually sees

`filterAnalyticsTools` copies only allowlisted names from `createAnalyticsTools`, **sorted A–Z**, into the `tools` object passed to `streamWithTools`. Sort order keeps `functionDeclarations` byte-stable.

The system prompt (`buildToolSystemPrompt`) is also frozen: store id, dates, and the question live on the **user** message (`buildUserTurnContent`), not in the system prompt. That is the other half of the cache prefix.

`TOOL_ROUTING_HINTS` / `buildToolRoutingSection` / `buildAntiStackingSection` in `src/prompts/copilot.ts` describe “prefer this tool for that phrasing.” They are **not injected** today (would bust the frozen system prompt). Composite rules (“use `get_revenue_diagnosis`, don’t stack atomics”) are instead hardcoded in that stable system prompt.

`src/tools/toolCatalog.ts` (`searchToolCatalog`) is a keyword catalog for a future `find_analytics_tools` meta-tool (Claude Code–style ToolSearch). Gemini has no native `defer_loading`. Not used in the live loop.

---

## After the call: phantom tools

If the model emits a tool name that was **not** registered, `ask` records `phantomToolCalls` and increments `ai_routing_phantom_total`.

Backend analogy: the client called an endpoint that is not in this gateway’s catalog. We log it. We cannot execute it.

Typical cause: the prompt or history mentions a tool that this turn did not register, so Flash invents the name.

---

## The routing gap to remember

Pins outside CORE are a **silent miss**: logs say `get_peak_hours` was pinned; the model never received the schema; it answers from CORE (`get_revenue_diagnosis`, `get_revenue_by_day`, …) or invents.

There is no “I don’t have that tool loaded” path. Fixing that is a product/architecture choice (register pins and bust cache, or a second call with CORE+pins, or a ToolSearch tool). Not implemented.

Route ends when `tools` is passed into `streamWithTools`. Next section is **Fetch**: what happens when the model actually calls one.

---

# Section 4 — Fetch: tools hit analytics-edge and return slim JSON

## What “fetch” means here

Route only *registered* the functions. Fetch is the `execute` body: Gemini emits a structured tool call, our process runs that function, we GET analytics-edge (or a static catalog), we shrink the JSON, we hand that DTO back to the model.

The model never sees the analytics-edge URL, query string, or Bearer token. It sees something like `get_revenue_summary({})` → `{ period, summary, bestDay, worstDay }`.

Backend analogy: the model is a client; each tool is a BFF action. Math already happened in analytics-edge. We only **GET + project**.

```
Gemini: tool call get_revenue_summary
  → tool.execute
       → safeExecute
            → fetchOnce('orders/dashboard-summary')
                 → fetchAnalytics  (HTTP GET, merchant Bearer)
            → slimDashboardSummary(raw)   // cents → major units, drop extra keys
       → onToolResult → toolFactRecords   // later used by Ground
  → JSON result injected into the model’s next step
```

All analytics tools live in `createAnalyticsTools` (`src/tools/analyticsTools.ts`). HTTP is `fetchAnalytics` (`src/context/analyticsClient.ts`). Projections are `slim*` in `src/context/slim/`.

---

## Scope: dates and the merchant token (not the model)

Every tool closes over a `ToolScope` built once in `ask()`:

| Field | Source | Why it is on the server |
|---|---|---|
| `storeId` | request body | Path/query for analytics-edge |
| `dateRange` | `resolveAskDateRange` | `startDate` / `endDate` — model does not pick calendar days for the primary window |
| `compareDateRange` | same NL parser, when the question named two windows | Second period for compare tools |
| `authorization` | `Authorization` header from the portal | Forwarded as-is |
| `storeFeatures` | body | Gates how-to tasks (e.g. reservations) |
| `onToolCall` / `onToolResult` | `ask()` callbacks | Metrics + fact records |

`baseParams` copies `storeId`, dates, and the Bearer onto every analytics GET. Tools do not each remember the token — they cannot forget it either.

**Auth rule:** analytics-edge **rewrites `storeId` from the JWT**. The body `storeId` is only trusted because it matches that token. `askHandler` 401s if there is no `Bearer`. A service-to-service identity would let the model talk about a store the caller does not own. We never substitute one.

`fetchAnalytics` builds `{ANALYTICS_EDGE_API_URL}/analytics/{path}?storeId&startDate&endDate` (+ extras) and GET with `Authorization`. Non-OK → `ApiError(502)`. That throw is caught by `safeExecute` (below), not by the HTTP handler.

---

## One HTTP call, several DTOs (`fetchOnce`)

`createRequestFetch` keeps a **per-ask** `Map<path?extra, Promise>`.

`orders/dashboard-summary` is the fat portal payload. Four CORE tools project different slices of the **same** response:

| Tool | Slimmer | Keeps |
|---|---|---|
| `get_revenue_totals` | `slimRevenueTotals` | `period`, `summary` |
| `get_best_worst_days` | `slimBestWorstDays` | `period`, `bestDay`, `worstDay` |
| `get_kitchen_activity` | `slimKitchenActivity` | `period`, `currentActivity` |
| `get_revenue_summary` | `slimDashboardSummary` | totals + extremes (no kitchen queue) |

Same pattern: `billing/overview` → payment totals / overview / days; `ai/menu-health` → full / items / trends.

Backend analogy: one DB query, several response mappers. The second tool in the same ask hits the in-flight Promise, not a second network call.

Memo key includes extra query params (`limit`, `compare=…`). Different extras → different GETs. `get_revenue_by_day` compare window is a **second** `fetchAnalytics('orders/revenue', …)` with different dates — it cannot share the primary `fetchOnce` entry.

---

## Slim DTO: we compute the presentation, not the metrics

A slimmer is a pure function: fat JSON in → small JSON out. Example (`slimDashboardSummary` / `mapDashboardSummary` in `orders.slim.ts`):

- Keep `totalOrders`, growth %, collection rate.
- Convert money with `amount()` (`src/context/slim/money.ts`): **cents → major units**, round to 2 decimals. Fields are named `*Amount` so the model is told not to `/ 100` again.
- Drop the rest of the portal payload.

That `/ 100` is the one piece of arithmetic we *do* allow in this service — it is unit conversion, not “what were sales.” Totals, ranks, and % still come from analytics-edge.

Why slim (same as Section 2): fewer tokens, fewer stray numbers to quote, currency conversion stays in our code. Comment in `money.ts`: never label amounts “USD”; the store’s orders already carry `currencyCode` (Québec → CAD).

Composites (`get_revenue_diagnosis`, `get_period_comparison`, `get_void_summary`, `get_staff_ops_health`) return the `ai/*` payload mostly as-is (`raw as Record<string, unknown>`). Those routes were already designed as copilot DTOs. Atomics that wrap portal routes always slim.

---

## Soft-fail (`safeExecute`)

Every `execute` is wrapped in `safeExecute`:

1. `onToolCall(name)`
2. `run()` — fetch + slim
3. success → `onToolResult(name, dto, ms)` and return the DTO
4. throw → log warn, return `{ error: "Could not load this analytics data right now. …" }`, still `onToolResult`

The stream does not die. Gemini gets an error object and is prompted to say so and **stop** (system prompt: do not hop to a sibling tool hoping it works).

Backend analogy: a BFF that returns `{ error }` with 200 to the *model*, instead of 502 to the merchant SSE. Tradeoff: the chat stays up; a badly-behaved model can ignore the error and narrate from memory. Grounding will not save you if there are no `$` / `%` in that fluff. That is a known gap, not a Fetch bug.

`ask()` records each result as `toolFactRecords` (full slim JSON) plus metrics (`ai_tool_fetch_duration_seconds`, `ai_tool_result_chars`). Grounding and charts read those records later.

---

## Two data sources: analytics vs static how-to

| Kind | Examples | Network | Math |
|---|---|---|---|
| Analytics atomic | `get_top_selling_items` → `items/popular` | GET analytics-edge | Already aggregated upstream; we slim |
| Analytics composite | `get_menu_health` → `ai/menu-health` | GET analytics-edge | Same |
| Compare | `get_period_comparison` → `ai/compare-periods` | GET with `compare` extras | Edge computes both windows |
| How-to | `search_platform_help`, `get_task_howto` | **None** | Static `PLATFORM_TASKS` catalog |
| Capabilities | `get_platform_capabilities` | None | Catalog + `DATA_READINESS` gaps |

How-to: `searchPlatformHelp(query)` then `getTaskHowto(taskId, storeFeatures)`. If the store flag is off, the DTO says the feature is disabled — still not an LLM guess.

`DATA_READINESS` (`src/config/dataReadiness.ts`) is a hardcoded map of topics we **cannot** claim (inventory, labor cost %, repeat vs new guests, …). Capabilities attach `dataGaps` so the model is supposed to refuse those instead of inventing them.

---

## Compare windows (still our calendar math)

The model must not invent “last week’s dates.”

- Question named two ranges → `scope.compareDateRange` already set. `get_period_comparison` sends `compare=custom&compareStartDate&compareEndDate`. `get_revenue_by_day` fetches the second `orders/revenue` for that window.
- Question did not name a second range, but the tool arg is `compare=previous|wow|mom` → `resolveCompareDateRange` (`src/utils/compareWindow.ts`) shifts the primary `{ from, to }` (equal-length previous, −7 days, −1 month). Mirrors analytics-edge’s own window helper.

Primary `dateRange` still comes from `resolveAskDateRange` in `ask()`, not from the tool args (tools do not take `startDate`).

---

## Fetch vs Route (how they interact)

Fetch runs **only for tools that were registered**. A pin that was not registered never reaches `execute`. `createAnalyticsTools` still *defines* `get_peak_hours`; `filterAnalyticsTools` simply omitted it from the SDK `tools` object.

If fetch fails, the DTO is `{ error }`. The loop continues to Narrate. Next section is **Narrate**: `streamWithTools`, steps, SSE.

---

# Section 5 — Narrate: the model writes English, we stream it

## What “narrate” means here

Fetch returned JSON. Narrate is the part where Gemini turns that JSON into merchant-facing sentences, and we stream those sentences to the portal over SSE.

The model is still a **vendor HTTP API** (`GOOGLE_GENERATIVE_AI_API_KEY`). We do not run weights on our box. `src/llm/LLMClient.ts` is the client: Vercel AI SDK `streamText` + `@ai-sdk/google`.

Backend analogy: you POST a chat completion, the vendor may invoke *your* tools (in-process `execute`), then it streams tokens. We own the Node `ServerResponse`. The vendor does not talk to the merchant.

```
ask()
  → buildToolMessages          // system + Redis history + this user turn
  → startSse                   // HTTP 200 text/event-stream, X-Session-ID
  → streamWithTools
       → streamText (Gemini)
            step: model may emit tool calls  → Fetch (Section 4)
            step: model emits text deltas    → sseWrite({ delta })
       → optional fallback model if empty / quota
  → fullText in memory
  → Ground (next section) then sseDone({ meta })
```

---

## What an LLM is doing in this step (backend view)

**LLM** = next-token predictor. Given the prompt (system rules + tool schemas + history + user question + tool JSON), it emits the most likely next tokens.

It does **not** have a calculator or a live DB. If the DTO says `totalRevenueAmount: 1240.5`, a well-behaved run *copies* `1,240.50` into English. A badly-behaved run invents `1,400` or divides by 100. That is why Ground exists *after* this step.

**Temperature** (`AI_TEMPERATURE`, default `0.3`) is how much the sampler may wander. `0` ≈ greedy / more deterministic; higher ≈ more variety. We keep it low because analytics copy should be boring and faithful.

**Tokens** are chunks of text the API bills. Input = prompt + tool schemas + tool results. Output = the English (and any tool-call JSON). `AI_MAX_TOKENS` (default 1024) caps *output*. Prompt-cache hits show up as `cachedInputTokens` (Gemini implicit cache of the stable prefix from Sections 2–3).

---

## The multi-step loop (`maxSteps`)

One “ask” is not one model call in the SDK sense. `streamText` + `tools` + `stopWhen: stepCountIs(maxSteps)` (`AI_MAX_STEPS`, default **5**):

1. Model sees messages + tool schemas. It either calls a tool or starts writing.
2. If it calls a tool, the SDK runs our `execute` (Fetch), then **calls the model again** with the DTO appended.
3. Repeat until it writes a final answer or hits 5 steps.

Typical happy path: **1 tool call + 1 English step** (2 steps). Six steps for a question that should take two is a routing regression (model stacking sibling tools). The system prompt says: after a tool returns, answer immediately; do not hop to a sibling on `{ error }`.

`onStepFinish` in `ask()` logs each step (tool names, result chars, tokens, cache hit ratio) into `llmSteps` for Mongo.

This is the “agent loop.” It is not a special ML architecture. It is a **for-loop around a chat API** with function calling, same idea as a workflow engine calling HTTP and then calling the LLM again.

---

## Prompt: what we send

`buildToolMessages` (`src/prompts/copilot.ts`) builds:

| Slot | Content | Stable across asks? |
|---|---|---|
| `system` | `buildToolSystemPrompt` — role, never invent numbers, prefer composites, don’t `/100`, how-to vs analytics | **Yes** (frozen string). `activeToolNames` is passed but unused (`_opts`) so we don’t bust cache. |
| history | Redis session turns (`user` / `assistant` text only) | Grows with the chat |
| this user turn | `buildUserTurnContent`: `[Context]` (storeId, date range, today) + `[Question]` | Per request. Range-change line only if dates actually changed. |

`splitMessagesForSdk` peels `system` out for the SDK `system:` field (Gemini treats that as the cacheable prefix along with tool schemas).

Store id and dates are **not** in the system prompt on purpose. A blanket “this is a follow-up” line on every turn was busting the longest common prefix; we only mention a date-range change when `priorDateRange` differs.

Session (`SessionManager`): last 20 turns, TTL default 30 min, max ~24k chars. We persist **prose**, not tool DTOs. Follow-up “why?” must tool-call again (prompt forbids answering analysis from history alone). If it disobeys, Ground has no new facts and numbers from memory can slip through.

---

## SSE: the merchant sees tokens before we finish

`startSse` sets `Content-Type: text/event-stream`, `X-Session-ID`.

Each model text chunk: `sseWrite({ delta })` → `data: {"delta":"..."}\n\n`.

When the SDK loop completes: `sseDone({ meta })` then `data: [DONE]`. Meta includes `toolsCalled`, tokens, `dateRange`, optional `chart` (built from the same tool facts, after Ground).

**Important:** deltas are flushed **during** narrate. `validateAnswerAgainstToolFacts` runs on `fullText` **after** the stream. Default validator is log-only; even `AI_ANSWER_VALIDATOR_STRICT=true` only appends a disclaimer to `finalText` used for **session/cache**, not a rewrite of bytes already sent. The merchant UI may already have shown an ungrounded sentence. Ground is an audit gate, not a proxy that buffers the whole answer.

Setup failure or empty `fullText` after primary+fallback: `sseError` with a generic string (`USER_FACING_LLM_ERROR`). Raw vendor errors stay in logs.

---

## Fallback model

`streamWithTools` tries `AI_MODEL` (default `gemini-3.5-flash`). If the stream is empty or the error looks retryable (429, quota, 503, overloaded, “no output generated”), it runs the **same** messages + tools on `AI_MODEL_FALLBACK` (default `gemini-2.0-flash`).

Same Fetch tools, same prompt. `usedFallback` is logged. If both are empty, the merchant gets the generic error above.

`streamWithFallback` / `call` / `stream` in the same file are siblings: fallback without tools, non-streaming `generateText` (used by session compress), and a deprecated stream helper. Copilot ask uses **`streamWithTools` only**.

---

## What narrate is not allowed to do

The system prompt is the contract (Section 2):

- Call a tool before factual numbers; never invent.
- Don’t answer “why / diagnose” from history without a fresh fetch.
- Don’t invent tool names (phantoms still happen — Section 3).
- Prefer one composite over stacking atomics.
- Report `{ error }` and stop.
- How-to → platform tools, not analytics, unless they also asked for numbers.

The model can still ignore all of that. Narrate has no schema on the **English** (no `{ claims: [...] }`). Constraint is natural language + later regex Ground.

---

Narrate ends when `fullText` is in memory and SSE deltas have been sent. Next section is **Ground**: `$` / `%` checked against `toolFactRecords`.

---

# Section 6 — Ground: check the English against tool JSON

## What “grounding” means here (not ML)

In research papers, **grounding** often means “tie the model’s words to retrieved evidence.” Google also uses “Grounding” for Gemini + Google Search citations. **We do neither of those.**

Our Ground is a **deterministic assertion** after narrate:

> Every `$…` and `…%` in the answer must already appear (within a small tolerance) in the slim DTOs from this ask.

No second model. No embeddings. File: `src/validation/validateAnswerAgainstToolFacts.ts`. Call site: `ask()`, after `streamWithTools`, before session/cache.

**Hallucination** (backend terms): the renderer invented a field that was not on the API response. Same class of bug as a UI showing `$9,999` that was never in `GET /orders`.

```
toolFactRecords[]     ← filled in Fetch (onToolResult)
fullText              ← filled in Narrate
        ↓
validateAnswerAgainstToolFacts(fullText, records, toolsCalled)
        ↓
applyAnswerValidation(...)   // log; optional disclaimer
        ↓
buildCopilotChart(records)   // same facts, not the prose
```

---

## The algorithm (like a unit test)

`validateAnswerAgainstToolFacts` is closer to `expect(answer).toMatchSnapshot(dto)` than to AI.

**1. Skip if there is nothing numeric to check**

| Condition | `skipReason` | Result |
|---|---|---|
| Every tool in `toolsCalled` is platform how-to (`get_task_howto`, `search_platform_help`, `get_platform_capabilities`, `get_feature_howto`) | `platform_only` | `skipped: true`, `ok: true` |
| Analytics tools ran but every result is `{ error: "..." }` | `no_tool_data` | same |
| Otherwise | — | run the check |

How-to answers are **not** validated. The function never returns a distinct “not applicable, but we looked.” Usage logs omit `answerValidation` when `skipped` is true, so you cannot tell “checked and clean” from “never checked” without reading `skipReason` in code. That exemption is a known gap.

**2. Build the allowed number pools from DTOs**

Walk each usable tool JSON (`collectAllowedValues`):

- Key looks like `%` (`pct`, `rate`, `change`) → `allowed.pct`
- Key looks like money (`amount`, `revenue`, `tips`, `aov`, `collected`, …) → `allowed.usd`
- Leaf keys `current` / `compare` / `value` inherit the **parent** key (`aovUsd.current` counts as money). That is the nested-DTO fix in `tests/test-answer-validator.ts`.
- Skip ids, dates, `*count`, `orders`, tiny 0–20 integers (list indexes).

**3. Extract mentions from the answer (regex only)**

- `PCT_PATTERN`: `12.7%`, `80.0 %`
- `USD_PATTERN`: `$12,400`, `$ 59.02`
- Sign is **not** captured. Tool `changePct: -77.5` still matches prose “down 77.5%” (`pctMatches` uses `Math.abs`).
- Dollar mentions below `$1` are ignored.
- Plain `1240` or `3 of 5` with **no** `$` or `%` are **not** mentions. They never fail Ground.

**4. Each mention must hit its pool**

- `%`: absolute diff ≤ 0.25 **or** relative ≤ 5%
- `$`: diff ≤ $1, or 2% if the allowed value is ≥ 100, or $0.50 if it is < 100

`ok` = no unmatched mention strings. `unmatchedMentions` is what gets logged (`copilot.answer.validation.failed`).

If the answer has **zero** `$`/`%`, the function returns `ok: true`, `skipped: false`. Causal fluff with no numbers looks like a pass.

---

## What we do with a failure (almost nothing)

`applyAnswerValidation`:

- Skip or `ok` → return the answer unchanged.
- Fail → `logger.warn` + increment `ai_answer_validation_failed_total`.
- If `AI_ANSWER_VALIDATOR_STRICT=true` → append a disclaimer: *Some figures in this reply may not match your reports…*
- Default (`false`) → **log only**. Merchant text is unchanged.

Session and Redis answer cache store `finalText` (with disclaimer only in strict mode). SSE already streamed `fullText` deltas (Section 5). Ground does not rewind the browser.

`promptMetrics.answerValidation` on the usage log: `{ ok, unmatchedCount }` when not skipped.

`buildCopilotChart(toolFactRecords, toolsCalled)` uses the **same** DTOs, not the prose. Charts are grounded by construction; the sentence next to them might not be.

---

## What Ground does **not** catch (read this twice)

The safety property was “the LLM never computes.” The validator only covers a **subset** of that property.

| Slips through | Why |
|---|---|
| `"Tuesday was your busiest day"` | No `$` / `%`. Rank/date/superlative unchecked. |
| `"3 of your top 5 items"`, `"up from 40 covers"` | Counts have no `$` / `%`. `*count` keys are stripped from the allowed pool anyway. |
| `"revenue dropped 20% because of the rain"` | `20%` may match the DTO; **because of rain** is unconstrained. Causal connectors are not parsed. |
| `"tips went down since you changed staff"` | Zero numeric mentions → `ok: true`. |
| `$12,400` vs last week when the DTO has current only | Compare/delta the model invented. **Mitigated** on `*Usd` compare metrics via `deltaAmount`; still a miss if the model subtracts two numbers that were never derived. |
| Model writes `12,400` without `$` | Regex never sees it. |
| How-to path | Skipped entirely (`platform_only`). |
| Soft-fail `{ error }` then a number from **history** | `no_tool_data` skip, or no usable analytics DTO this turn. |
| Wrong metric, right number | `$1,240` appears as both revenue and collected → mention matches **any** usd in the pool, not the field the sentence names. |

Tolerances also mean a nearby invented % can pass (5% relative on a large rate).

The comment in the file: pattern copied from social-publisher draft validation. Same idea — regex against facts — same limits.

---

## Mental model

```
GROUND is a JSON assertion, not a judge model.

  extract $ and % from answer
  ⊆  numbers collected from this turn’s slim DTOs
  (with slop)

Pass  ≠  the story is true.
Fail  ≠  we blocked the merchant (unless STRICT disclaimer).
Skip  ≠  “checked N/A”; it is “did not run.”
```

That is the whole read loop: **route → fetch → narrate → ground**. How-to, infra, and gaps follow.

---

# Section 7 — How-to path: product help, not analytics

## Same loop, different data source

“How do I export a PDF?” is not a sales question. The agent still does route → fetch → narrate → ground, but **Fetch hits a static catalog in this repo**, not analytics-edge. Ground then **skips** (`platform_only`, Section 6).

Backend analogy: a second bounded context on the same copilot HTTP API — help-center search instead of a reporting BFF.

Intended tool sequence (prompt + tool descriptions):

```
search_platform_help({ query })  → scored task ids
get_task_howto({ taskId })       → steps, path, tip
```

Vague “what can you do?” → `get_platform_capabilities` (area + task index + `DATA_READINESS` gaps).

---

## Catalogs (code, not a CMS)

| File | What it is |
|---|---|
| `src/catalog/platformTasks.ts` | `PLATFORM_TASKS` — merchant tasks (`reports.export_pdf`, `team.create_shift`, …) with `steps`, `path`, `searchTerms` |
| `src/catalog/platformHelp.ts` | `PLATFORM_FEATURES` — portal **areas** (Reports, Team, …) for the legacy feature how-to |
| `src/config/dataReadiness.ts` | Topics we must refuse (inventory, labor cost %, repeat guests, …) |

Reviewed in PRs like the tool registry. No CMS, no live Google Doc.

`searchPlatformHelp(query, maxResults)` is keyword scoring (id / name / summary / `searchTerms`), sorted, top N (default 5). **Not embeddings.** Same family as `detectCategoriesFromQuestion`.

`getTaskHowto(taskId, storeFeatures)`:

- Unknown id → `{ available: false, validIds }`
- `requiresFeature` (e.g. `reservationManagement`) + portal `storeFeatures` flag off → still returns steps, plus `featureEnabled: false` and an availability note

`get_feature_howto` is **legacy** (area-level). Not in CORE. Only registered if the question pins an explicit feature id (Section 3). Prefer task tools.

---

## How routing treats how-to

`isPlatformHowToQuestion` / `isPurePlatformHowToQuestion` (`questionIntent.ts`):

- Pure how-to (“how do I export a report”, no “how much revenue”) → **no** analytics intent categories; pins `search_platform_help` + `get_task_howto`.
- Mixed (“how do I see revenue and what were sales?”) → how-to pins **and** analytics categories still computed.

Those two tools **are already in CORE**, so the pin is a no-op for registration (unlike `get_peak_hours`). How-to reachability is fine under the frozen CORE policy.

System prompt: use platform tools for product how-to; do not call analytics unless they also asked for numbers.

---

## What the model can still get wrong

The catalog is the source of truth; the model can still skip `search_platform_help`, invent a `taskId`, or mix in analytics tools. Unknown ids fail closed in `getTaskHowto`. Ground will not check the steps text. Wrong navigation copy is a content/prompt problem, not a numeric one.

---

# Section 8 — Infra around the loop

The agent is still a Node HTTP service. This section is Redis, Mongo, limits, metrics, and the non-chat Insights sibling.

Boot (`index.ts`): secrets → telemetry → Redis → Mongo (`ensureIndexes`) → listen. Required env: `MONGODB_URI`, `GOOGLE_GENERATIVE_AI_API_KEY`, `REDIS_URL`.

---

## Redis sessions

`src/session/SessionManager.ts` — key `ai:session:{sessionId}`.

| Rule | Default |
|---|---|
| TTL | `AI_SESSION_TTL_SECONDS` = 1800 (30 min) |
| Max turns | 20 (hard slice) |
| Max chars | `AI_SESSION_MAX_CHARS` = 24000 |
| Store check | `loadSession` drops the doc if `storeId` mismatches |

Turns are `{ role: user \| assistant, content, chars }` only. **No tool DTOs.** Follow-ups must fetch again (Section 5).

`resolveSession` loads or creates; `appendTurns` after a successful answer. `maybeCompress`: over char cap → last 2 turns kept, older summarized via `call()` (non-streaming LLM). On compress failure → last 4 turns. Never throws.

`DELETE /api/v1/ai/copilot/session/:sessionId` → `destroySession` (204).

---

## Exact-match answer cache

`src/cache/responseCache.ts` + `buildCacheKey` (`src/utils/hash.ts`).

- Key: SHA-256 of `storeId \| normalized question \| from \| to \| tool-context` → `ai:cache:{hash}`
- Context includes active tool names + date range (+ compare range when present) so a CORE-only vs all-tools world would not share an answer
- **First turn only** (`session.turns.length === 0`). Follow-ups skip cache.
- TTL: `AI_CACHE_TTL_SECONDS` = 3600
- Stores **plain answer text**, not charts / tool facts / SSE meta
- `shouldCacheCopilotResponse`: skip empty, skip if any tool `{ error }`, skip outage phrasing (“could not load”, …)
- Hit: stream cached text via SSE, `ai_cache_hits_total`, usage `outcome: cache_hit`, **no LLM, no rate-limit consume**

This is **not** Gemini prompt cache. Two caches: Redis exact answer vs provider prefix cache (`cachedInputTokens`).

---

## Rate limit

`checkAndConsumeRateLimit` (`src/limits/rateLimit.ts`) — **after cache miss, before LLM**.

| Counter | Redis key | Default |
|---|---|---|
| Burst | `ai:burst:{storeId}` (60s) | `AI_BURST_LIMIT_PER_MINUTE` = 8 |
| Daily | `ai:ratelimit:{storeId}:{UTC date}` | `AI_DAILY_LIMIT_PER_STORE` = 0 (**off**) |

Limit `0` disables that counter. Redis errors **fail open** (allow). 429 JSON to the merchant. Metric `ai_rate_limited_total`.

---

## Mongo usage log

`insertUsageLog` → collection `ai_usage_log` (`src/repositories/usageLog.repository.ts`). Fire-and-forget (`void`); insert failure must not fail the ask.

Indexed (`ensureIndexes`): `storeId+timestamp`, `requestId`, `toolsCalled+timestamp`, TTL **90 days**.

Logged (trimmed): question (500 chars), tools called, tokens + prompt-cache split, duration, date range, `promptMetrics` (CORE/pins/intent/phantoms/grounding), message **previews**, tool result **previews** (`AI_LOG_PREVIEW_CHARS` default 400), per-step SDK stats, outcome.

Previews are **not** hashed. Slim JSON can include item names, staff names, revenue. Fine for internal debug; not a PII-safe warehouse. Forum advice was: hashes + sizes in Mongo, raw I/O in Redis TTL by `requestId`. **Not how it works today.**

---

## Prometheus + health

`src/metrics.ts` + `GET /metrics`. Useful ones: `ai_requests_total{cached}`, `ai_cache_hits_total`, `ai_stream_duration_seconds`, `ai_tokens_total`, `ai_prompt_cache_hit_ratio`, `ai_tool_fetch_duration_seconds`, `ai_tool_result_chars`, `ai_routing_phantom_total`, `ai_answer_validation_failed_total`, `ai_rate_limited_total`.

`GET /healthz` liveness; `GET /health/ready` pings Mongo + Redis (`src/router.ts`).

CORS: `src/middleware/cors.ts`. Request timer: `src/utils/requestTimer.ts` (structured marks in logs).

---

## Insights (sibling, no chat)

`GET /api/v1/ai/copilot/insights` → `insightsHandler` → `buildDailyHighlights`.

Same analytics-edge GETs (`ai/compare-periods`, menu-health, …), **our code** writes titles/bodies. No Gemini, no tools, no Ground. Default window: last 7 UTC days. Partial fetch errors become `partialErrors` on the JSON. “For You” feed, not the agent loop.

---

# Section 9 — Gaps: what is still weak before writes

The read loop is stronger than a typical “stuff JSON in a prompt” bot. It is **not** a safety property you can bet a mutate tool on. Below is what the code actually leaves open (plus what HITL writes would need).

---

## 1. Silent pin miss (Route)

Pins outside CORE are logged and **not registered** (Section 3). `get_peak_hours`, `get_fulfillment`, `get_reservations_summary`, `get_top_selling_items`, … are invisible in default `intent` mode.

No metric for “intent matched, pin not in `activeToolNames`.” Phantoms (`ai_routing_phantom_total`) only fire if the model **invents the name**. If it quietly uses `get_revenue_diagnosis` instead, logs look fine.

`capToolNames` / `AI_TOOL_FILTER_MAX` / `TOOLS_BY_FOCUS` union are unused. README still sounds like CORE + keywords.

**Before writes:** never compute a pin and leave it unregistered. Either always attach pinned schemas (cache cost), or a second call with CORE+pins, or a ToolSearch tool, or answer “I don’t have that tool loaded.”

---

## 2. Grounding is a narrow regex (Ground)

Covered in Section 6. Highest-risk holes:

- Counts, ranks, dates, superlatives
- Causal claims (`because of rain / staffing`) with or without a matching `%`
- Compare/delta the model invents when the DTO has only `current` (or a delta not in the JSON) — “vs last week” trips the validator **or** slips through as a bare number
- `$` / `%` matched against **any** number in the pool, not the named metric
- How-to skip is a branch, not `not_applicable`
- Default: log only; SSE already showed the text

Forum-shaped fixes (not built): force `{ claims: [{ metric, value, period, source_tool }], prose }`; put `this` / `prev` / `delta` on compare DTOs; widen extractor to every numeral + superlative; always run the validator.

---

## 3. Soft-fail shifts crash → hallucination (Fetch)

`{ error }` keeps SSE alive. Prompt says report and stop. If Flash ignores that, it narrates from history or invents. Ground may skip (`no_tool_data`) or pass with zero mentions.

Fine for reads. **Fail closed** for anything near a write.

---

## 4. Multi-turn is thin (Session)

History is prose only. “Redo” reuses the prior **question** for intent (`getEffectiveQuestionForIntent`). We do not replay prior DTOs. The model can quote old numbers without a new fetch; the prompt forbids it; Ground only helps if `$`/`%` appear and this turn has facts.

No scripted follow-up tests in the live service path. Eval harnesses exist under `tests/` (`run-golden-asks`, `run-eval-matrix`, `run-launch-smoke`) — they assert tools/smoke more than “every claim ⊆ DTO.”

---

## 5. Observability vs PII

Mongo stores question + tool JSON previews. Good for debugging routing. Bad as a long-lived PII store. Dashboards you would actually open: grounding fail rate, silent pin miss (once you log it), p95 fetch latency, avg steps, compare-question fail rate. Several of those series do not exist yet.

---

## 6. Do not give the model a mutate tool (HITL — not implemented)

Product next phase was preview → confirm → mutate → audit. **Not in this codebase.** Pattern to copy when it is:

- `preview_action` returns a typed plan `{ action, target_ids, before, after, risk }`
- `confirm_action(plan_id)` is a **UI** call after a human click — not something the agent can invoke in the same turn
- `plan_id` single-use, short TTL, bound to user+merchant, re-validated against current state (etag / optimistic lock)
- Audit: who confirmed, plan hash, before/after snapshot
- If preview and confirm can both fire in one `maxSteps` loop, the design already lost

Until pin-registration and grounding are trustworthy, keep the agent read-only.

---

## 7. The invariant is untested

“The LLM never computes” is the design rule (Section 2). CI does not assert it. Grounding is not a failing test on every ask. Counts and causal prose bypass the only checker.

Minimum bar called out in reviews: golden 30–50 questions with expected tool sequence + expected claim set; assert fetch success, step count ≤ N, every `$`/`%` ⊆ DTO; fixtures from recorded DTOs so analytics-edge flakiness does not flake CI.

---

## Map of the three leftover risks onto the loop

```
ROUTE   silent pin miss, phantom names, unused capToolNames
FETCH   soft-fail → empty facts, then invented prose
NARRATE unconstrained English; causal “because”; SSE before Ground
GROUND  $/% only; skip how-to; log-only default
INFRA   DTO previews in Mongo; no pin-miss metric
WRITE   not built — do not add until the above are explicit
```

That is the architecture we shipped: a read-only analytics copilot with a frozen CORE allowlist, slim DTOs, a Gemini tool loop over SSE, and a post-hoc number checker. How-to is the same machine on a static catalog. Insights is the same data without an LLM.
