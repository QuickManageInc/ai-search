# Module 1 — Copilot Tools Inventory & A4 Honest-No Design

> **Purpose:** Study pass before writing any code. List the **maximum** candidate new tools (name + what for), audit whether each one's underlying data is actually real, design the **A4 "honest no"** contract, and define **new endpoints** that return the most sufficient (one-shot, LLM-ready) data for each.  
> **Explicitly excluded from this round:** Inventory tools (D). Team view: inventory should go through CDC → `analyticsDB` first, like every other domain — not bolted on as a second live client. Revisit after that pipeline exists.  
> **Related:** [Module1_Copilot_Deep_Dive.md](./Module1_Copilot_Deep_Dive.md), [Module1_Response_Accuracy_Plan.md](../Module1_Response_Accuracy_Plan.md), [ai-edge-api/README.md](../../ai-edge-api/README.md)

---

## 0. Ground rule carried over

> Systems own facts; the LLM owns wording.

Every new tool below is judged first by **"is the underlying data real?"** — not by how useful the question sounds. A tool that looks obviously right (voids) can ship as-is. A tool that looks obviously right (discounts) turns out to be backed by a placeholder today — found while researching this doc (see §2).

---

## 1. Current tool inventory (baseline — do not duplicate)

23 live tools in `ai-edge-api/src/tools/analyticsTools.ts`, all analytics-shaped:

| Family | Tools |
|---|---|
| Menu | `get_top_selling_items`, `get_item_revenue_ranking`, `get_underperforming_items`, `get_item_trends`, `get_items_by_daypart`, `get_item_pairs`, `get_menu_health` |
| Revenue / ops | `get_revenue_summary`, `get_revenue_by_day`, `get_revenue_mix`, `get_operations_overview`, `get_fulfillment`, `get_peak_hours`, `get_hourly_pattern`, `get_cancellation_stats`, `get_revenue_diagnosis`, `get_period_comparison` |
| Payments | `get_payment_overview`, `get_payment_details` |
| People | `get_staff_performance`, `get_workforce_summary`, `get_attendance_summary` |
| Guests | `get_reservations_summary` |

Composite (one call, AI-ready) endpoints already exist as the pattern to copy: `ai/menu-health`, `ai/revenue-diagnosis`, `ai/compare-periods`.

---

## 2. Data-reality audit (do this before naming a tool)

Traced into `analytics-edge-api` repositories/services. Findings:

| Field | Source | Status |
|---|---|---|
| `billing.void` → `voidBillings` | `daily_store_summary` per day, summed in `queryBillingSummary` | ✅ **Real**, aggregated from CDC |
| `billing.tips` / `tipCount` → `totalTips`, `tipCount` | Same collection | ✅ **Real** |
| `billing.discounts` → `totalDiscounts` (dollar sum) | Same collection | ✅ **Real** (the dollar total) |
| `discountCount`, `discountRate`, `averageDiscount` | `billing.service.ts:116` — `discountCount = totalDiscounts > 0 ? totalTransactions : 0` | ❌ **Placeholder heuristic, not real.** Assumes *every* transaction was discounted whenever the discount total is nonzero. This already ships today inside `get_payment_details` — the LLM can currently narrate a fake discount rate as fact. |
| Daily void counts (per-day trend) | Not exposed — `queryDailyBillings` only returns `totalBillings`, `collected`, `outstanding` per day | ⚠️ **Needs a small repo change**, not new CDC — the per-day `billing.void` field already exists in the same collection, just isn't selected. |
| Guest identity across visits (repeat vs new) | Not modeled anywhere in `analyticsDB` | ❌ **Does not exist.** Would need a new CDC-fed guest identity model — out of scope. |
| Upsell configuration (which pairs a merchant has configured) | No config surface — `get_item_pairs` is discovered affinity, not merchant-set config | ❌ **Does not exist as a product feature yet.** |

**Takeaway:** this audit alone justifies A4 — you already have one live field (`discountCount`/`discountRate`) that looks like a real stat and isn't. That's the first thing to fix, independent of any new tool.

---

## 3. Maximum candidate tool list (name + what for)

Organized by priority family from the deep dive. **Nothing here is approved to build yet** — this is the full menu to review together.

### A — Store facts gaps (real or realistically fixable data)

| Candidate tool | What for | Data status | Honest-no needed? |
|---|---|---|---|
| `get_void_summary` | Void count, void rate %, daily void trend | ✅ Real (needs daily-void field added to one query) | No — ship normal |
| `get_discount_status` | "Do you offer/track discounts" | ⚠️ Partial — total $ real, count/rate/avg fake | **Yes** — wrap count/rate/avg |
| `get_tip_summary` *(may not need a new tool — already covered by `get_payment_details`)* | Tip rate / totals | ✅ Real | No |
| `get_guest_repeat_status` | "How many repeat vs new guests" | ❌ Not modeled | **Yes** — always `available:false` until a guest-identity model exists |
| `get_upsell_config_status` | "What upsells are configured" | ❌ Not a feature yet | **Yes** — redirect to `get_item_pairs` (discovered affinity) instead of pretending config exists |

### A2 — Platform help family (new — not analytics at all)

| Candidate tool | What for | Data status |
|---|---|---|
| `get_platform_capabilities` | Curated catalog: what QuickManage can do (Reports, Social Publisher, Workforce, future Inventory), with deep-link paths | New — static, versioned, in-repo |
| `get_feature_howto(featureId)` | 3–5 step how-to for one catalog feature | New — static, in-repo |

These never call `analytics-edge-api` — no new analytics endpoint needed, just a maintained catalog file.

### B — Proactive insights (deterministic, not merchant-typed questions)

| Candidate | What for | Data status |
|---|---|---|
| `get_daily_highlights` (server-side orchestration, not necessarily an LLM-callable tool) | 3-bullet "For You" feed: WoW revenue swing, menu health flag, ops anomaly | ✅ Real — pure re-composition of `get_period_comparison` + `get_menu_health` + thresholds |

This is **not** part of the ask/tool-calling loop — it's a separate deterministic endpoint (see §5). Listed here because it's still a "new function," per the request to see everything.

### C — Neighborhood / web search (future, not this cycle)

| Candidate | What for | Data status | Note |
|---|---|---|---|
| `search_neighborhood_weather` | Local weather context | External API | Highest hallucination risk in the whole doc — ship only with mandatory citations, after A is trusted |
| `search_neighborhood_events` | Local events context | External API | Same |

Listed for completeness of "maximum tools" — **do not build with A/A2/B.**

### D — Inventory (excluded this round, per your call)

| Candidate | What for | Status |
|---|---|---|
| `get_low_stock_ingredients` | Low-stock alerts in chat | Deferred — `inventoryDB` not in CDC/`analyticsDB` |
| `get_waste_summary` | Waste $ / volume | Deferred — same reason |
| `get_stock_movements_summary` | Movement ledger summary | Deferred — same reason |
| `get_theoretical_food_cost` | Food cost % via BOM × sales | Deferred — needs recipes + CDC join, hardest one |

**Recommendation matches your instinct:** route inventory into the same CDC → `analyticsDB` pipeline every other domain uses *first*. Building a second live JWT client (`inventoryClient`) into `ai-edge-api` before that exists would fork the architecture's one rule ("systems own facts via the same pre-aggregated, slimmed path") for one domain only.

---

## 4. A4 — "Honest no" design

### 4.1 Contract shape

Applied **per data sub-object**, not per whole tool — `get_payment_details` proves a single tool response can have both real and fake sections at once.

```ts
type Maybe<T> =
  | { available: true;  data: T }
  | { available: false; reason: string }
```

Applied to the discount case found in §2:

```jsonc
{
  "period": { "from": "2026-08-01", "to": "2026-08-26" },
  "tips": { "available": true, "totalUsd": 412.30, "tipCount": 96, "averageUsd": 4.30, "tipRatePct": 61.2 },
  "discounts": {
    "available": false,
    "reason": "Per-transaction discount tracking isn't wired yet — only the total discounted amount is known.",
    "totalDiscountedUsd": 88.40
  }
}
```

Notice `totalDiscountedUsd` still rides along **outside** the `available:false` block — it's real, no reason to hide it. Only the fabricated count/rate/average are gated.

### 4.2 Where it's enforced (three layers, all needed)

1. **Slim mapper layer** (`ai-edge-api/src/context/slim/*.ts`) — wraps unreliable sub-fields in the `Maybe<T>` shape before the JSON ever reaches the model. This is the actual fix for the fake discount stat.
2. **Prompt layer** (`buildToolSystemPrompt`) — one added rule: *"If a data block has `available: false`, never state its fields as fact. You may mention `reason` and any sibling fields that are marked available."*
3. **Product-copy layer** (portal capability text + `get_platform_capabilities`) — must not advertise "ask about discount rates" while the field says `available: false`. This is what makes it a genuinely *honest* no, not just a hidden prompt trick.

### 4.3 Single source of truth: a data-readiness registry

To keep layers 2 and 3 in sync without hand-editing prose in two places, add one small config:

```ts
// ai-edge-api/src/config/dataReadiness.ts
export const DATA_READINESS = {
  discountBreakdown: { available: false, reason: 'Per-transaction discount tracking not wired yet.' },
  discountTotal:     { available: true },
  voidSummary:       { available: true },
  guestRepeatStatus: { available: false, reason: 'Guest identity across visits is not tracked.' },
  upsellConfig:      { available: false, reason: 'Upsell configuration is not a feature yet — pairs shown are discovered, not configured.' },
} as const
```

- Slim mappers read from it instead of hardcoding `available: false` inline.
- The system prompt builder can auto-generate the "I can't talk about X yet" sentence from every `available: false` entry — so adding a new gap to this file automatically updates both the model's honesty and (via §5.3) the portal's capability copy, with one edit instead of three.

---

## 5. New endpoints — "most sufficient data" design

Same philosophy as the existing `ai/menu-health` / `ai/revenue-diagnosis` / `ai/compare-periods` composites: **one round trip, already USD, already capped, already period-aligned, no daily grids unless the tool is explicitly about daily trend.**

Checklist for every new endpoint below (from `Module1_Response_Accuracy_Plan.md` Phase 2/3, reused on purpose):

- [ ] Single Mongo round trip (or 2 at most), not N portal-service re-fetches
- [ ] `period` (+ `previousPeriod` if comparative) always included
- [ ] Cents → `*Usd` conversion done server-side
- [ ] Arrays capped (10 default, 20 max) server-side, not left to the LLM/prompt to self-limit
- [ ] Unreliable sub-fields wrapped in the `Maybe<T>` / `available` shape from §4, not silently included as fact
- [ ] A `notes` block only if it adds something the raw fields don't already say

### 5.1 `GET /api/v1/analytics/ai/void-summary` *(new, backs `get_void_summary`)*

```
period,
voidBillings, totalBillings, voidRatePct,
dailyVoids: [{ date, voidCount }]   // requires adding billing.void to queryDailyBillings — small repo change, no new CDC
```

Backed entirely by data already in `daily_store_summary`.

### 5.2 `get_discount_status`, `get_guest_repeat_status`, `get_upsell_config_status`

**No new analytics-edge-api endpoint needed.** These are cheap enough to answer from the `DATA_READINESS` registry directly inside `ai-edge-api` (plus one existing field for the discount dollar total). Don't build a Mongo-backed endpoint just to say "not available yet."

### 5.3 `GET /api/v1/ai/data-readiness` *(new, in `ai-edge-api`, not `analytics-edge-api`)*

Serves the `DATA_READINESS` registry as JSON. Two consumers:

1. Platform-help tool / portal "what can I ask?" panel — so product copy and model honesty never drift apart.
2. Optional: a lightweight admin/ops view of what's still stubbed, without grepping code.

No DB read — served straight from the config module, essentially free to call, cacheable indefinitely (bust on deploy).

### 5.4 Platform-help catalog — no endpoint, a file

`ai-edge-api/src/catalog/platformHelp.ts` — versioned TS/JSON, reviewed in PRs like the tool registry itself. Mongo CMS is explicitly **not** recommended for v1: it adds a write surface and a "who edits this" question for content that changes as rarely as your codebase does.

### 5.5 `GET /api/v1/analytics/ai/insights` *(new, Phase B only — not this round's focus, listed for completeness)*

Deterministic composition of `ai/compare-periods` + `ai/menu-health` + threshold rules. No new Mongo shape beyond what A0–A4 already produce.

---

## 6. Study checklist — go through this per tool before writing code

| Tool | Real data today? | New endpoint? | Honest-no wrapper? | Rough new input tokens | Build with |
|---|---|---|---|---|---|
| `get_void_summary` | ✅ Yes | Yes — `ai/void-summary` (+small repo change) | No | ~120–180 | A |
| `get_discount_status` | ⚠️ Partial | No — registry + existing total | **Yes** | ~60–100 | A |
| `get_guest_repeat_status` | ❌ No | No — registry only | **Yes** | ~40 | A |
| `get_upsell_config_status` | ❌ No (redirect to `get_item_pairs`) | No — registry only | **Yes** | ~40 | A |
| `get_platform_capabilities` | New (static) | No — catalog file | N/A | ~150–300 (catalog size dependent) | A2 |
| `get_feature_howto` | New (static) | No — catalog file | N/A | ~80–150 | A2 |
| `get_daily_highlights` / `ai/insights` | ✅ Yes (recomposed) | Yes — `ai/insights` | Inherits from composed tools | N/A (server-side, not per-ask) | B |
| `search_neighborhood_*` | External | Yes, later | N/A — needs citations, not honest-no | Unknown | C (later) |
| Inventory tools (all) | ❌ Not in CDC | N/A | N/A | N/A | **D — deferred** |

Also re-check tool-count ceiling from the deep dive: current 23 + A(4) + A2(2) ≈ **29** — still well under the ~40–50 ceiling before tool-retrieval or a `query_metric` semantic layer becomes necessary.

---

## 7. What this session decided vs still open

**Decided:**
- Inventory (D) is out for this round — wait for CDC.
- Neighborhood (C) is listed but not being built now.
- A4's actual first job is fixing an existing lie (`discountCount`/`discountRate`), not just gating a hypothetical new field.
- `Maybe<T>` / `available` shape lives at the *sub-object* level, not the whole-tool level.
- One `DATA_READINESS` registry feeds prompt honesty **and** product copy, so they can't drift apart.

**Still open (confirm before coding A):**
1. Does `get_upsell_config_status` even need to exist, or should the prompt just permanently redirect "configure upsells" questions to `get_item_pairs` + a platform-help deep link — one fewer tool?
2. Is the `daily_store_summary` per-day `billing.void` field guaranteed non-null historically, or only from a certain CDC deploy date onward (affects whether `dailyVoids` needs a "data starts on X" caveat)?
3. Where does `DATA_READINESS` physically live if both `ai-edge-api` and the portal need it — duplicated config, or `ai-edge-api` becomes the source of truth via `ai/data-readiness` and the portal fetches it at build/runtime?
