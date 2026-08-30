# Copilot intent-based tool filtering (replace tab context)

Status: **planned — implement next**  
Supersedes: tab-driven `focusHints` as the primary tool filter (keep as optional soft hint only)  
Related: [README.md](./README.md), [Module1_Copilot_Slim_Split_Plan.md](./Module1_Copilot_Slim_Split_Plan.md), [Module1_Copilot_Tool_Test_Questions.md](./Module1_Copilot_Tool_Test_Questions.md)

---

## Problem

**~85% of Gemini `inputTokens` per step is tool schemas**, not the user question or system prompt. Registering all 27 tools costs ~5,500–5,800 tokens **on every LLM step** (up to `AI_MAX_STEPS`).

We shipped **tab-based filtering** (`AI_TOOL_FILTER=hints` + portal `context` / `focusHints`). It saves tokens but picks tools from **where the merchant clicked**, not **what they asked**:

| Question | Tab | Expected tool | What happened |
|----------|-----|---------------|---------------|
| What's wrong with my menu? | Revenue | `get_menu_health` | Menu tools not registered → phantom call → `get_platform_capabilities` |
| Discount & tip rate? | Revenue | `get_payment_details` | Payment tools not registered → wrong revenue tools |
| Staffing & labor ops? | Revenue | `get_staff_ops_health` | Labor tools not registered → phantom `get_scheduling_summary` |

**Phantom tool calls:** model requests tools named in the system prompt that are **not** in the filtered set (`toolResultChars: 0`, no fetch). Root cause: prompt lists tools the SDK did not register.

**Decision:** Drop tabs as the filter driver. Use **question-intent expansion** instead (same token savings, correct routing).

---

## Industry patterns (what we learned)

### Claude Code (`v1/` in repo)

- **Core tools always loaded** (Read, Grep, Bash, Edit, ToolSearch).
- **Deferred tools** (`shouldDefer: true`, MCP): only **names** in initial context; full JSON schema loaded via **`ToolSearchTool`** on demand.
- Subsequent API calls include **only discovered tools** (`extractDiscoveredToolNames`).
- Auto mode: defer when tool definitions exceed ~**10% of context window**.

Key files: `v1/src/utils/toolSearch.ts`, `v1/src/tools/ToolSearchTool/`, `v1/src/services/api/claude.ts`.

### Cursor

- MCP tool descriptions synced to **files**; agent gets names only, reads descriptions when needed (~**47% token reduction** in A/B).
- Hard cap ~**40 active MCP tools**; user toggles in Settings.
- Same philosophy: **pull context on demand**, not push everything upfront.

Reference: [Cursor — Dynamic context discovery](https://cursor.com/blog/dynamic-context-discovery).

### QuickManage constraint

We use **Gemini + Vercel AI SDK**. Anthropic-native `defer_loading` / `tool_reference` **does not apply**. We need a **portable** pattern:

1. **Intent expansion** (recommended for 27 tools) — server-side keyword/category router, no extra LLM step.
2. **Tool search meta-tool** (optional later) — Gemini-safe clone of ToolSearch; +1 step latency, scales to 50+ tools.

---

## Recommended design: Core + question-intent expansion

### Flow

```
Merchant question
  → detectCategoriesFromQuestion(question)     // server, ~0ms, keyword rules
  → CORE_TOOL_NAMES (always)
  → ∪ TOOLS_BY_FOCUS[category] for each match
  → cap at MAX_TOOLS_PER_ASK (default 15)
  → filterAnalyticsTools(allTools, activeNames)
  → buildToolSystemPrompt(activeToolNames only) // no phantom names
  → LLM picks + executes
```

### Always loaded — `CORE_TOOL_NAMES` (~9 tools)

Already defined in `ai-edge-api/src/tools/toolFocusMap.ts`:

| Tool | Role |
|------|------|
| `get_platform_capabilities` | Product help catalog |
| `get_feature_howto` | How-to steps |
| `get_menu_health` | Vague menu / item diagnosis (composite) |
| `get_revenue_diagnosis` | Vague sales / why down (composite) |
| `get_staff_ops_health` | Vague labor / staffing (composite) |
| `get_period_comparison` | WoW / MoM / vs previous period |
| `get_revenue_summary` | Overall sales, totals, collection |
| `get_revenue_by_day` | Daily trend |
| `get_void_summary` | Voids (common cross-cutting ask) |

Est. schema cost: ~**3,000–3,500 tokens/step** (vs ~6,500 for all 27).

### Expand by question intent

Reuse `TOOLS_BY_FOCUS` categories; trigger from **question text**, not portal tab:

| Category | Example signals | Adds |
|----------|-----------------|------|
| `payment` | tip, discount, collected, outstanding, payment, billing, card, cash | `get_payment_overview`, `get_payment_details` |
| `menu` | menu, item, bestseller, underperforming, lunch, dinner, pair, daypart, sell | menu atomics (or rely on `get_menu_health` for vague) |
| `staff` | staff, employee, sales performance, tips by person, top staff | `get_staff_performance` |
| `workforce` | headcount, hire, attendance, clock, timesheet, workforce, schedule | workforce + attendance tools |
| `operations` | void, cancel, peak, hour, fulfillment, completion, kitchen queue, reservation | ops tools |
| `revenue` | channel, mix, dine-in, takeout, POS, growth, AOV, orders | revenue atomics beyond core |

**Rules:**

- Vague questions → composites in CORE only (no atomics unless keywords demand specificity).
- Specific questions → CORE + relevant category atomics.
- **Cap** union at `AI_TOOL_FILTER_MAX` (default **15**); if over cap, prefer composites + highest-scoring category.
- **Ignore** portal `context` for filtering (or treat as lowest-priority tie-breaker only).

### New env / mode

| Env | Values | Behavior |
|-----|--------|----------|
| `AI_TOOL_FILTER` | `intent` (new default) \| `hints` \| `always` \| `off` | See below |
| `AI_TOOL_FILTER_MAX` | number, default `15` | Max tools registered per ask |

| Mode | Behavior |
|------|----------|
| **`intent`** (new) | `CORE` + question-driven category expansion, capped |
| `hints` | Legacy tab filter (portal `context`) — deprecate |
| `always` | CORE only, no expansion |
| `off` | All 27 tools (A/B baseline) |

---

## Prompt changes (same PR or follow-up)

1. **Dynamic routing list** — system prompt only mentions tools in `activeToolNames` for this turn.
2. **Remove dead tool names** — delete references to unregistered tools: `get_scheduling_summary`, `get_time_off_summary`, `get_swaps_summary`, `get_attendance_trend`, `get_compliance_status` (unless we register them).
3. **Shorten system prompt** — drop long routing table (lines 40–61 in `copilot.ts`); tool `description` fields already carry contrastive routing.
4. **Anti-stacking rules** — keep: do not call `get_revenue_summary` after `get_revenue_mix` / `get_revenue_diagnosis` when data is sufficient.

---

## Portal changes

- **Stop sending** `context` / `focusHints` from Reports tab (or send for analytics only, not tool filter).
- Global FAB ask path: `context: []` always.
- Optional future: show “Tools considered: …” in debug/usage panel from `promptMetrics.activeToolNames`.

---

## Implementation checklist

### Code (`ai-edge-api`)

- [ ] Add `detectCategoriesFromQuestion(question: string): ContextCategory[]` — keyword map in `toolFocusMap.ts` or `questionIntent.ts`
- [ ] Add `resolveActiveToolNamesFromIntent(question, focusHints?)` — CORE ∪ expanded, capped
- [ ] Wire `AI_TOOL_FILTER=intent` in `getToolFilterMode()`; default to `intent`
- [ ] Pass `question` into `filterAnalyticsTools()` from `copilot.service.ts`
- [ ] `buildToolSystemPrompt({ activeToolNames })` — dynamic tool list in prompt
- [ ] Log `intentCategories`, `activeToolNames`, `registeredToolCount` in `promptMetrics`
- [ ] Metric: `routing.phantom` when model calls tool not in `activeToolNames`

### Tests / golden questions

- [ ] Re-run cross-domain questions **without tab context** (see test doc)
- [ ] Confirm no phantom calls (`toolResultChars: 0` for undeclared tools)
- [ ] Compare `inputTokens`: intent (~4k) vs off (~6.5k) vs old hints (wrong routing)

### Docs

- [x] This plan
- [ ] Update `Module1_Copilot_Tool_Test_Questions.md` — test without tab, record intent mode baselines
- [ ] Update `ai-edge-api/README.md` — `AI_TOOL_FILTER=intent`

---

## Alternative: tool search meta-tool (phase 2)

If tool count grows past ~40 or intent keywords become brittle:

```typescript
find_analytics_tools: tool({
  description: 'Search analytics tools by keyword or select:ToolName. Returns names and when-to-use.',
  inputSchema: z.object({ query: z.string(), max_results: z.number().optional() }),
  execute: async ({ query }) => { /* keyword match TOOL_CATALOG */ },
})
```

Turn 1: CORE + `find_analytics_tools` only (~10 tools).  
Turn 2: re-register CORE + tools returned by search.

Mirrors `v1/src/tools/ToolSearchTool/`. Cost: +1 LLM step (~700ms TTFT).

---

## Expected outcomes

| Metric | All 27 tools | Tab hints (wrong tab) | Intent (planned) |
|--------|-------------:|----------------------:|-----------------:|
| Tools registered | 27 | 4–14 | 8–15 |
| inputTokens / 1-tool ask | ~6,500 | ~4,200 (revenue) | ~4,000–4,500 |
| Cross-domain routing | ✓ | ✗ | ✓ |
| Phantom tool calls | rare | common | target: zero |
| Portal tab dependency | none | required | **none** |

---

## Token stack (full picture)

Intent filtering addresses **schema** cost (~85%). Keep these in parallel:

| Layer | Lever | Doc |
|-------|-------|-----|
| Tool schemas | Intent expansion | this file |
| Tool results | Slim-split projectors | [Slim_Split_Plan](./Module1_Copilot_Slim_Split_Plan.md) |
| System prompt | Dedupe + dynamic tool list | this file |
| Multi-step stacking | Prompt tuning | [Tool_Test_Questions](./Module1_Copilot_Tool_Test_Questions.md) |
| Session history | Compact after N turns | future |
| Provider | Gemini context caching | future |

---

## References

- `ai-edge-api/src/tools/toolFocusMap.ts` — CORE, TOOLS_BY_FOCUS, current filter modes
- `ai-edge-api/src/prompts/copilot.ts` — system prompt (trim + dynamic list)
- `ai-edge-api/src/services/copilot.service.ts` — ask pipeline, `filterAnalyticsTools`
- `v1/src/utils/toolSearch.ts` — Claude Code deferred tool pattern (reference only)
