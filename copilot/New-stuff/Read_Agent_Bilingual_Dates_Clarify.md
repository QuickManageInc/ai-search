# Read-agent next: bilingual, dates, clarify, tools

> **Goal:** Make the **read-only** agent better (route → fetch → narrate → ground). HITL stays parked.  
> **Status:** Discussion locks (2026-09-16). Not a build ticket until sliced.  
> **Related:** [Tool candidates](./Read_Only_Tool_Candidates.md) · [Eval matrix](../Module1_Copilot_Eval_Matrix.md) · [Measure snapshot](../Module1_Copilot_Measure_Snapshot.md) · [HITL contract](../Module1_Copilot_HITL_Action_Contract.md) (out of scope here)

Systems own facts. The LLM owns wording. Do **not** copy OpenClaw / v1 / OpenCode `browser`, `exec`, `web_search`, or Cursor `@Browser` into the merchant agent.

---

## What we already have (do not rebuild)

| Piece | Reality |
|-------|---------|
| Date default | Portal **always** sends `dateRange`. Outside Reports: last 31 UTC days. On Reports: the report filter. Session bar shows it. |
| NL override | `relativeDateRange.ts` — **not** the only clock. Comment: *never ask clarifying questions; UI has a visible default.* |
| French in dates | **Month names** already (`février`, `mars`, `août`…). Relatives (`last week`, `yesterday`) are **English-only**. |
| Vague labor | CORE `get_staff_ops_health` (`ai/staff-ops`: hours, open shifts, punches, time-off). |
| Labor slimmers | `employees.slim.ts` has scheduling / time-off / compliance. **No** Copilot tools for those yet. |
| Honest-no registry | `DATA_READINESS` exists. No `get_data_readiness` tool. |
| Pins / how-to | English keyword tables. Gemini can *write* French; routing often will not. |
| Pin-honor | **Done.** Candidates doc’s “finish pins before labor” gate is closed. |

---

## 1. More tools — what is useful now

Do **not** dump the whole A-list from the candidates file. `get_staff_ops_health` stays the vague labor composite. New pins are for **specific** asks.

| Priority | Idea | Verdict |
|----------|------|---------|
| **1** | EN+FR synonyms on **existing** pins, how-to `searchTerms`, and date phrases | Highest ROI. No new fetch. *« préparation cuisine »* never honors `get_fulfillment` today. |
| **2** | `get_data_readiness` **or** louder `dataGaps` on capabilities | Cheap honest-no. One tool unless Flash still invents inventory. |
| **3** | **Three** labor pins from existing slimmers: `get_scheduling_summary`, `get_time_off_summary`, `get_compliance_expiring` | Data already there. Not open-shifts + cannot-work + hours-trend in the same slice. |
| **4** | Pin existing `get_cancellation_stats` / `get_hourly_pattern` / workforce + attendance | Only if logs show composites stealing those asks. |
| Later | Chat wrapper around `GET /insights` | Real data; second path next to For You. |
| Later | `find_analytics_tools` | After registered tools keep growing past ~15. Catalog exists. |
| Skip | `get_orders_trends` | Slim: **ignores selected range**. |
| Skip | PDF-as-tool, weather/neighborhood, remembering last week’s `$` | Binary / G1 / Ground poison. |

---

## 2. Dates — keep a closed catalog; do not let Flash pick the window

`relativeDateRange.ts` is a **phrase table** (yesterday, last N days, this/last week/month, weekdays, ISO, “17 feb to 28 feb”, month names). That is the right shape: analytics tools **do not** take `from`/`to` from the model.

It is weak at:

- French relatives: *hier*, *la semaine dernière*, *ce mois-ci*, *les 7 derniers jours*
- Ambiguous English: “this month vs last month” (T3 stole August)
- Store timezone (keys are **UTC** days)
- Open-ended “second Tuesday of last month”

**Do not** replace this with Gemini `generateObject`. That invents ranges (another T3). OpenClaw’s date code is **IANA timezone + `Intl` display**, not NL understanding. OpenCode has no merchant date parser.

### Locked stack

```
Portal dateRange (always)          ← default, visible in the bar
        ↓
EN+FR phrase table                 ← today’s logic + French relatives
        ↓
Question looks temporal
AND table missed
        → chips (preferred) or chrono-node (optional later)
        ↓
Never let Flash set tool dates
```

`chrono-node` is a **second** parser for leftovers, with `America/Toronto`. It is not a replacement: Ground will not catch a wrong **window**.

---

## 3. Clarification — copy Claude’s mechanism, not the trigger

Claude `AskUserQuestion`, OpenClaw `ask_user`, and OpenCode `question` are the same: structured chips, wait, continue the session.

OpenClaw: ask only when **blocked on a user-owned decision**, not when a **sensible default** exists. On timeout, continue with best judgment.

“What’s our revenue?” with no date words **has a default** (session bar). Blocking it is worse than Square/Toast. The prompt already says to name the applied window.

**Do not** ask on every missing NL date.

**Do** interrupt when the default is not enough:

| Ask | Behavior |
|-----|----------|
| “What’s our revenue?” | Fetch **bar range**. First sentence names it. Optional (non-blocking) chips: Last 7 days / This month / Keep this. |
| Two windows we cannot parse (*hier vs la semaine dernière*) | **Pause.** Chips, not a paragraph. |
| Question says “last month” and picker is February | Conflict — which window? |
| Compare with only one period | Clarify the second window (N2-style). |

### Mechanism (when we build it)

Not a Gemini tool beside `get_revenue_summary`. If `ask_clarify` is inside `maxSteps`, Flash over-asks and skips fetches.

1. Server emits `meta.clarify` (header + 2–4 options + optional Other).
2. **No analytics tools** until the portal `POST`s the choice into the **next** `/ask` `dateRange`.
3. Same `sessionId` (M1 already proved re-tool).
4. Typed “yes” is not a date. Other: phrase table, not Flash.

---

## 4. Language — Canada EN+FR tables; do not error on other languages

Intent / pins / `search_platform_help` are English regex. CORE still registers `get_revenue_summary`, so *« Comment étaient les ventes ? »* can still fetch sales. *« Combien de temps prend la préparation en cuisine ? »* will **not** pin `get_fulfillment`. How-to `searchTerms` are English.

**Do not** return “unsupported language.” The failure is a **missed pin**, not “can’t speak.” Gemini can answer Spanish; a 400 is hostile.

| Approach | Pros | Cons |
|----------|------|------|
| **A. Synonym tables EN+FR** on pins, how-to, date phrases | Deterministic, no extra model, Ground-safe | Maintain two lists; other languages stay on CORE |
| **B. Translate → English for routing only** | Scales past FR | Latency; bad translation breaks pins; **never** translate answer `$`/`%` |
| **C. Error if not EN/FR** | Simple | Worst product for a Canadian POS |

**Lock A.** Same files as today: `tips|pourboires`, `kitchen prep|préparation`, `last week|la semaine dernière`. How-to: `exporter un rapport`, `horaire`, `cuisine`. Portal suggested prompts already have i18n keys — use them.

Optional: portal `locale` (`fr` / `en`) → “answer in the merchant’s language.” Routing stays on the table.

Other languages: CORE + Flash. Optional soft line: “I follow product how-tos best in English or French.” Not an HTTP error.

OpenCode-translate (chat in KO, tools in EN) is **wrong** here: it mangles figures and fights Ground.

---

## 5. What not to copy

| Source | Pattern | Copilot |
|--------|---------|---------|
| OpenClaw `ask_user` | Structured wait | Yes — **ambiguous windows only** |
| OpenClaw date-time | Timezone below cache boundary | Yes — `America/Toronto` in context, not more regex |
| OpenClaw / v1 ToolSearch | Deferred tools | Later |
| OpenCode `question` | Same chips | Same as `ask_user` |
| OpenCode-translate | Inbound translate | No — synonyms instead |
| OpenClaw `browser` / `web_search` | Neighborhood | Stay G1 |
| Cursor `@Browser` | IDE | Never a merchant tool |
| HITL preview/confirm | Writes | Parked — [action contract](../Module1_Copilot_HITL_Action_Contract.md) |

---

## Slice order

One theme per change. Do not mix labor tools with a new clarify loop.

1. **EN+FR phrase tables** — dates + pins + how-to. Makes the agent feel Canadian without a translator.
2. **Date UX** — always echo the applied window; chips on **conflict/ambiguity** only; bar stays default for “what’s our revenue.”
3. **Three labor pins** from existing slimmers (scheduling, time-off, compliance).
4. **`get_data_readiness`** (or louder `dataGaps`) if inventory/CRM asks still get invented facts.
5. **Clarify interrupt** in SSE/portal only after (1)+(2), so French *la semaine dernière* does not always pop a picker.

Do **not** start with `chrono-node`, ToolSearch, daily-highlights-as-tool, or a generic `ask_user` Flash can call on every short question.

First code slice if we implement: **(1) bilingual catalogs**. Claude-like UX is **(2)+(5)** and must stay **deterministic** (portal chips + `dateRange` on the next ask), not a new Gemini tool.
