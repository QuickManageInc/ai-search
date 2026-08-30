# Module 1 — Analytics Assistant → Store Copilot: Deep Dive

> **Scope:** Expand `ai-edge-api` from a Reports-only chat into a Square/Toast-style merchant copilot.  
> **Out of scope:** Module 2 (revenue forecasting), guest/kiosk product suggestions (deferred).  
> **Related:** [README.md](./README.md) (copilot doc index), [Module1_Copilot_Intent_Tool_Filter_Plan.md](./Module1_Copilot_Intent_Tool_Filter_Plan.md), [../Module1_Context_Optimization_Plan.md](../Module1_Context_Optimization_Plan.md), [../Module1_Response_Accuracy_Plan.md](../Module1_Response_Accuracy_Plan.md)

---

## 1. What we already have

| Capability | Status | Where |
|------------|--------|-------|
| Natural-language Q&A over analytics | Live | `ai-edge-api` + Reports `AIAssistantPanel` |
| Tool-calling agent loop (~20 tools) | Live | `src/tools/analyticsTools.ts` |
| Slim DTOs / atomic fetches | Live | `src/context/slim/` |
| SSE streaming + sessions + Redis cache | Live | `analyticsAssistant.service.ts`, `SessionManager`, `responseCache` |
| Rate limits (burst + daily, fail-open) | Live | `src/limits/rateLimit.ts` |
| Usage logging | Live | `ai_usage_log` |
| Social Publisher (separate service) | Live (test IG) | `social-publisher-api` |
| Revenue forecasting (Module 2) | Designed, not built | Explicitly skipped |

**Core rule already in the product:** systems own facts; the LLM owns wording; the merchant owns anything that leaves the building.

---

## 2. Competitive landscape — what the top products actually do

Restaurant POS AI is a vertical of the broader **BI copilot** pattern (ThoughtSpot Spotter, Amazon Q in QuickSight, Power BI Copilot, Tableau Pulse).

| Product | Job | Closest thing we have |
|---------|-----|------------------------|
| **Square AI** | Global merchant chat: *your data* + *local web* + *how to use Square* | Analytics Assistant — but Reports-only, no web, no product help |
| **Toast IQ** | Same chat **plus** “For You” cards **plus** HITL write-actions **plus** live upsells on POS/kiosk | Chat + `items/affinity`; no feed, no actions, no guest surface |
| **Restaurant365 AI** | Back-office: P&L, AP, food cost, schedules | Inventory + labor data exist; **no accounting ledger** — skip R365-style close/AP |

### Square’s question taxonomy vs our tools

Square advertises: sales, labor, menu, guests, discounts, voids, upsells.

| Topic | Our coverage | Gap |
|-------|--------------|-----|
| Sales / menu / channel / cancels / peaks | Live tools | — |
| Labor / attendance / staff performance | Live tools | — |
| Guests (reservations) | Live `get_reservations_summary` | No guest CRM |
| Upsells as pairs | `get_item_pairs` → `items/affinity` | No merchant “configure upsells” UI |
| Voids | `voidBillings` on billing overview | Not wired as a dedicated AI tool yet |
| Discounts / tips | Stubs (zeros) in analytics | Do not promise until CDC has facts |

**Implication:** “Become Square AI” is mostly **product packaging + more tools + leaving Reports**, not a second LLM service.

### Industry pattern everyone converged on

> The LLM never computes the number. A deterministic system computes the number; the LLM only selects which computation to run and narrates the result.

That is exactly `createAnalyticsTools`: model picks `get_revenue_diagnosis` → Mongo aggregation in `analytics-edge-api` → JSON is ground truth. Stronger than document RAG for numeric domains (no retrieval-relevance failure mode).

---

## 3. Product direction — store copilot (not a new service)

Keep **one** service (`ai-edge-api`). Change the **surface**, not the architecture.

| Today | Target |
|-------|--------|
| `AIAssistantPanel` only on `ReportsPage` | Global shell chat (header / FAB) |
| Soft focus hints from Reports tab | Same + optional page context |
| Analytics tools only | Three tool families (below) |

### Three tool families (Square’s beta mix)

1. **Store facts** — existing analytics tools; never invent numbers.
2. **Platform help** — small, versioned catalog of what QuickManage can do (Reports, Social Publisher, Inventory, Workforce). Suggest *our* features, not generic SaaS.
3. **Optional neighborhood** (Square web search) — weather / local events with citations. Ship **after** (1)+(2); highest hallucination risk.

### What not to copy yet

- Square/Toast **write-from-chat** (edit menu, edit shifts) — needs HITL + permissions. Prefer deep-links: “Open Social Publisher for this item.”
- R365 **accounting close / invoice matching** — we don’t own the P&L.
- Nightly forecast worker (Module 2) — out of scope.
- Guest/kiosk LLM chat — deferred; different identity and risk model.

### Honest capability boundary (UI copy)

> I can talk about sales, menu, labor, reservations, and operations. I can’t talk about discounts yet (data not available).

---

## 4. Suggested build order (merchant copilot only)

| Priority | Work | Why |
|----------|------|-----|
| **A** | Move panel out of Reports into shell; platform-help tool; wire voids; clear “no” on discounts | Highest leverage, reuses everything |
| **B** | Proactive “For You” cards (3 bullets from existing tools on login) | Toast-style value without Module 2 |
| **C** | Neighborhood / web search with citations | Only after A is trusted |
| **D** | Inventory food-cost tools via `inventory-edge-api` | New domain; inventory not in CDC |

---

## 5. Sectors & reference architectures

| Sector | Product | Pattern | Distinctive lesson for us |
|--------|---------|---------|---------------------------|
| Restaurant POS | Square AI, Toast IQ | Tool-calling over own APIs + proactive feed | Toast: push feed + HITL actions; Square: read+chat |
| BI platforms | ThoughtSpot, Amazon Q, Power BI Copilot | **Text-to-query over a governed semantic layer** | Fixed metrics×dimensions catalog scales better than N hand-written tools |
| Fintech advisors | e.g. internal advisor assistants | RAG over a **closed curated corpus**; refuse outside it | Mandatory citations; conservative refusal |
| Customer support | Intercom Fin, Zendesk AI | RAG + confidence-gated human handoff | Explicit “I don’t know” threshold |
| CRM / ops | Salesforce Agentforce | Tool-calling + PII mask before LLM | We already strip PII in CDC before `analyticsDB` |
| Back-office | Restaurant365 | Cross-domain P&L + labor + inventory | Requires owning the ledger — N/A for now |

**Scaling ceiling:** ~20 tools with contrastive descriptions works. BI copilots with hundreds of metrics use **one** tool with a constrained query grammar (`metric`, `dimension`, `filters`, `groupBy`, `compare`) validated against a metrics catalog — flag when we pass ~40–50 tools.

---

## 6. ML patterns that apply

### 6.1 Tool-calling / agent loop — already have

`streamWithTools` in `LLMClient.ts` is a `maxSteps`-bounded ReAct-style loop. Correct for structured data behind APIs. Remaining lever: **tool selection quality**.

### 6.2 Tool retrieval (later)

When tool count grows: embed tool descriptions offline, embed the question, retrieve top-k tools by cosine similarity, pass only those schemas to the model. Not needed at ~20 tools.

### 6.3 Semantic (embedding) cache vs exact-hash cache

Today: normalize question → SHA-256 → Redis.  
Miss: “why were sales down Tuesday” ≠ “explain the drop in Tuesday’s revenue.”

**Semantic cache:** store `(embedding, answer)` keyed by store + date range; hit if cosine similarity ≥ ~0.92. Highest leverage if repeat-question cost matters. Restrict to vague diagnosis questions first to limit false positives.

### 6.4 Query routing / intent classification before the big model

“Always call a tool for analysis” is a **prompt rule** — LLMs ignore it sometimes (`toolsCalled: []` on follow-ups). Production pattern: cheap upstream classifier (keyword heuristic first, then small `generateObject`) that **force-gates** tool use before the expensive stream.

### 6.5 Dynamic few-shot from `ai_usage_log`

Logs already store `question`, `toolsCalled`, `sessionId`. Periodically pull good question→tool pairs, embed questions, inject 2–3 similar examples at request time. Improves tool routing without fine-tuning (helps bestsellers vs underperforming confusions).

### 6.6 LLM-as-judge regression harness

Golden set of 30–50 real questions (from usage logs) with expected tool + fact ranges. Replay on every prompt change; second cheap LLM scores tool choice + number grounding. Script against `/ask` + SSE — no new infra.

### 6.7 Text-to-query / semantic layer (longer term)

One tool `query_metric(metric, dimensions[], filters, compare?)` over a metrics catalog instead of tool #25, #30 for every combination. Refactor when tool count hurts.

---

## 7. Data structures

### Already correct

| Structure | Role |
|-----------|------|
| Star-schema pre-agg (`daily_store_summary`, `daily_store_items`) | Cheap tool results; OLAP foundation |
| Redis `INCR`/`EXPIRE` counters | Burst + daily rate limits |
| Char-budgeted session + summarize older turns | ConversationSummaryBuffer-style memory |
| TTL exact-match cache | Cache-aside |

### Natural extensions (priority)

1. **Numeric-claim index for validation** — after tools return JSON, put every number into a `Set`; after answer, regex-extract numbers and check membership. Same idea as Social Publisher’s `validateDraftAgainstFacts.ts`.
2. **Vector index (HNSW)** — for semantic cache / tool retrieval (Redis RediSearch or small in-process HNSW).
3. **Token-budget priority fill** — rank context blocks by relevance/recency; optional; current last-2-turns + char budget is already good.
4. **Trie / synonym routing** — only if tool catalog becomes huge.

---

## 8. Minimizing hallucination (ranked)

Current defenses are **100% prompt-based** (`analyticsAssistant.ts` rules). Prompt rules are probabilistic — documented production failure: analysis follow-up with `toolsCalled: []`.

### Tier 1 — cheap, deterministic (do first)

| Defense | Action |
|---------|--------|
| Post-hoc numeric grounding | Port Social Publisher validator pattern: every number in the answer must appear in tool JSON |
| Hard-gate “must call a tool” | Keyword/heuristic (why, analyse, problem, cause, compare) → force tool path |
| Empty tools ≠ confident answer | If `toolsCalled: []` and question isn’t chitchat → retry with nudge or refuse |

### Tier 2 — structural

| Defense | Action |
|---------|--------|
| Citations | Tag which tool/field a claim came from; extend SSE `meta` with `citedFacts` |
| Temperature | Keep ~0.3 for prose; consider 0 for a separated routing step |
| Calibrated refusal | If date range has few days of data, say the trend is not yet reliable |

### Tier 3 — expensive (only if needed)

| Defense | Action |
|---------|--------|
| Self-consistency | Run twice; only surface if tools + key numbers agree (2× cost) |
| Live LLM-as-judge | Second model: “is every number traceable to tool JSON?” before stream |

---

## 9. Input tokens & data collection

### Already done (big wins)

| Phase | Effect |
|-------|--------|
| Phase A — slim DTOs, atomic routes, compact JSON | Cut fat composites |
| Phase B — tool calling instead of pre-load | Typical input ~20k → ~500–2k tokens |

### What’s left

1. **Provider prompt caching** — Gemini/OpenAI cache stable prefix (system + tool schemas). Deterministic per `(storeId, dateRange, focusHints)`.
2. **Semantic cache** — token cost lever, not just latency.
3. **Anomaly telemetry on `ai_usage_log`** — flag high spend, empty-tool rate, never-compressing sessions (script/cron; no Module 2 worker required).
4. **Mine logs for eval + few-shot** — turn write-only logs into a golden set and dynamic examples.

---

## 10. Architecture invariants (do not break)

```
Merchant copilot                    Guest recommender (deferred)
ai-edge-api (SSE, LLM)              orders-edge / recommend API
JWT = merchant user                 JWT = device / public store context
Question → tools → text             Cart → ranked SKUs → chips
```

| Surface | Auth | LLM on hot path? |
|---------|------|------------------|
| Portal copilot | Merchant | Yes — tools over our APIs |
| Kiosk / POS / guest web | Device / store | No — ranked SKUs only (deferred) |
| Chip copy / posters | Merchant, async | Yes — fact-grounded |

---

## 11. Immediate technical priority

The architecture decisions that matter most are already in place (tool execution + context slim). The gap vs a hardened production copilot:

> Every defense against a wrong or invented number lives in prompt text. Nothing checks the answer against tool data after the fact.

**Highest-ROI next engineering step (when implementing):** port the numeric-validator pattern from `social-publisher-api/src/validation/validateDraftAgainstFacts.ts` into the analytics assistant path — cheap, deterministic, closes the failure mode already logged in `Module1_Response_Accuracy_Plan.md`.

---

## 12. Open decisions

1. Copilot shell placement: header FAB vs dedicated AI page vs both?
2. Platform-help catalog: markdown file in-repo vs Mongo CMS?
3. Semantic cache: Redis vectors vs defer until token spend justifies it?
4. When to start “For You” proactive cards vs finishing shell + voids + help tools?

---

*Document distilled from architecture review of `ai-edge-api`, `ai-search` plans, and competitive analysis (Square AI, Toast IQ, Restaurant365). Module 2 forecasting and guest product suggestions explicitly excluded.*
