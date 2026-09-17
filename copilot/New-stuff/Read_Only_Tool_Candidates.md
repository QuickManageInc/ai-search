# Read-only tool candidates (not writes)

> **Goal:** Extra Copilot capabilities while the agent stays **read-only** (route → fetch → narrate → ground).  
> **Sources:** `analytics-edge-api` routes, `ai-edge-api` slimmers / `DATA_READINESS`, [Tools Inventory](../Module1_Copilot_Tools_Inventory.md), [Eval matrix](../Module1_Copilot_Eval_Matrix.md), [Hardening plan](../Module1_Copilot_Read_Agent_Hardening_Plan.md), Claude Code `v1/` ToolSearch/skills, [OpenClaw tools](https://docs.openclaw.ai/tools).  
> **Status:** Research list — pin-honor is **done**. Labor pins / bilingual / dates: [discussion](./Read_Agent_Bilingual_Dates_Clarify.md). Do not stuff CORE.

Systems own facts (analytics-edge or a static catalog). The LLM owns wording. Do **not** copy OpenClaw/v1 `browser`, `exec`, `web_search`, or file/write tools into this merchant agent.

Cursor `@Browser` is for developers. Do not give Gemini a browser over the merchant portal.

---

## Do not add (wrong product)

| Source | Tool / idea | Why skip |
|---|---|---|
| OpenClaw | `browser`, `exec`, `process`, `message`, `cron` mutate | Host control / send / schedule writes. Not a POS copilot. |
| OpenClaw / v1 | `web_search`, `web_fetch`, `x_search` | Square-style “neighborhood.” Inventory + eval G1: **after** store facts + help are trusted. Highest hallucination risk. |
| v1 | `Bash`, `FileEdit`, `FileWrite`, `Agent` subagents | Coding agent. We already have a typed BFF. |
| Our docs | Inventory, guest CRM, labor cost %, write-from-chat | `DATA_READINESS` + eval gap list. Honest-no, don’t fake a tool. |

---

## A — Highest ROI: edge routes + slimmers exist, no Copilot tool

These sit on `analytics-edge-api` `src/router.ts`. Slimmers already live in `ai-edge-api/src/context/slim/employees.slim.ts`. `get_staff_ops_health` (`ai/staff-ops`) only **samples** some of them. Dedicated pins beat stuffing CORE.

| Candidate tool | Edge route | Merchant ask | Notes |
|---|---|---|---|
| `get_scheduling_summary` | `employees/scheduling/summary` | “How many hours did we schedule?” / coverage | Distinct from sales `get_staff_performance`. |
| `get_open_shifts` | `employees/scheduling/open-shifts` | “Do we have unfilled shifts?” | Staff-ops mentions open shifts; a pin is clearer. |
| `get_time_off_summary` | `employees/time-off/summary` | “Time-off backlog?” | Same. |
| `get_swaps_summary` | `employees/swaps/summary` | “How many shift swaps?” | PDF reports already show this; chat does not. |
| `get_compliance_expiring` | `employees/compliance/documents/expiring` | “Permits expiring?” | Workforce summary has counts; this is the list. |
| `get_cannot_work_summary` | `employees/cannot-work/summary` | Availability / cannot-work | Thin; pin only if merchants ask. |
| `get_attendance_trend` | `employees/attendance/trend` | Clock-in trend vs snapshot | Pair with existing `get_attendance_summary`. |
| `get_scheduling_hours_trend` | `employees/scheduling/hours-trend` | Labor hours over weeks | Compare-style; needs `this/prev/delta` later. |
| `get_orders_trends` | `orders/trends` | Rolling ops trend | Slim comment: **ignores selected range** — only ship if documented, or fix the route. |

Do **not** wrap `reports/pdf` as a tool (binary download). Keep `reports.export_pdf` how-to.

---

## B — Already a tool: honor / pin, don’t rebuild

Hardening-plan A-list is the next product move. Also worth **pin allowlisting** (not CORE) when asks are explicit:

| Tool | Already | Add if |
|---|---|---|
| `get_cancellation_stats` | Implemented | “Why cancelled / by channel” — richer than diagnosis. Eval D3. |
| `get_hourly_pattern` | Implemented | Hour-by-hour vs `get_peak_hours`. |
| `get_workforce_summary` / `get_attendance_summary` | Implemented | Headcount / clock-ins vs vague staff-ops. Eval matrix: honor pin. |
| `get_daily_highlights` | **HTTP** `GET .../insights`, not a chat tool | Optional: one tool for “anything unusual / for you?” wrapping `buildDailyHighlights`. Data is real (compare + menu-health). |

`get_void_summary` is already CORE. Don’t add a second voids tool.

---

## C — Honest-no tools (static, no new Mongo)

From `DATA_READINESS` + Tools Inventory. Same shape as capabilities `dataGaps`. Cheap, stops the model inventing.

| Tool | Returns | When |
|---|---|---|
| `get_data_readiness` | Full registry JSON | “Can you talk about inventory / regulars / labor %?” Inventory already wanted `GET /ai/data-readiness`. |
| `get_guest_repeat_status` | `{ available: false, reason }` | “New vs repeat / VIP.” |
| `get_inventory_status` | same | “What’s 86 / low stock?” |
| `get_labor_cost_status` | same | “Labor cost %.” |
| `get_upsell_config_status` | false + pointer to `get_item_pairs` | “Configured upsells.” |

One tool (`get_data_readiness`) is enough if the prompt already lists gaps. Split tools only if eval shows Flash still narrates inventory as fact.

---

## D — Patterns from v1 / OpenClaw (adapt, don’t paste)

These help the **read loop**, not new restaurant metrics.

| Pattern | Source | Copilot version | When |
|---|---|---|---|
| **ToolSearch** (`tool_search` / `ToolSearchTool`) | v1 + OpenClaw | `find_analytics_tools` over `toolCatalog.ts` | After 20–40 *reachable* tools. Hardening plan: **defer** until pin-union is live. Catalog already exists. |
| **Skills** (`SkillTool` / OpenClaw `SKILL.md`) | v1 + OpenClaw | Static playbooks: “diagnose sales,” “menu health,” “how-to then stop” — **prompt packs**, not new fetch | If composites still get stacked. Anti-stack is already in the system prompt. |
| **`ask_user`** | OpenClaw | Portal: structured clarify (“this week vs last week?”) instead of guessing a compare window | Nice for T3 NL gaps; not a fetch tool. |
| **Session memory** (v1 `remember` / OpenClaw memory) | v1 | Store **prefs** only: “always CAD, never labor %,” default range — still **re-fetch** facts | Do not remember last week’s `$` (Ground hole). |
| **`sessions_list` / progress** | OpenClaw | Redis sessions already exist. Optional: `get_session_summary` of *questions asked*, not DTOs | Thin. |
| **Read-only cron** | OpenClaw `cron` | **List** report subscriptions (“what PDFs are scheduled?”) if that API is merchant-visible | Read. Creating a subscription is a write — skip. |
| **Code Mode** | OpenClaw | No. Model must not compute; that breaks the invariant. |

---

## E — How-to catalog (not new `execute` tools)

Platform help is ~15 tasks. Read-only product still grows here: more `PLATFORM_TASKS` rows (devices, customers, MEV already exist). Missing merchant FAQs → add **catalog copy**, not Gemini.

Optional: `relatedAnalyticsTools` is already on tasks — expose it in the how-to DTO so mixed asks don’t wander.

---

## Suggested sequence (still read-only)

1. Finish **pin-honor** (A-list). ~~Don’t add eight new labor tools in the same PR.~~ **Done.** Next: bilingual catalogs, then three labor pins — [discussion](./Read_Agent_Bilingual_Dates_Clarify.md).
2. **Wire 3 labor pins** that slimmers already support: scheduling summary, time-off, compliance expiring (or open shifts). One allowlist bump.
3. **`get_data_readiness`** (or richer capabilities `dataGaps`) so inventory/CRM asks fail honestly.
4. Optional chat tool wrapping **`buildDailyHighlights`**.
5. **`find_analytics_tools`** only if deferred tools keep growing past ~15 registered.
6. Neighborhood / browser / weather: **stay G1**.

**Ticket-sized first slice:** scheduling summary, time-off, compliance expiring, data-readiness, daily-highlights-as-tool — all read-only, all backed by code that already exists.
