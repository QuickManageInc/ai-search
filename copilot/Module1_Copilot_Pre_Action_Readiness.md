# Copilot — pre-action readiness

> **Goal:** Harden the **read-only** agent (route → fetch → narrate → ground) before starting any **HITL action** work (preview / confirm / mutate).  
> **Status:** Agenda locked. **Read-path implementation:** [Read-agent hardening plan](./Module1_Copilot_Read_Agent_Hardening_Plan.md). Thread **C** (HITL) still deferred.  
> **Related:** [Eval matrix](./Module1_Copilot_Eval_Matrix.md) · [Prod launch](./Module1_Copilot_Prod_Launch_Plan.md) · [Intent filter](./Module1_Copilot_Intent_Tool_Filter_Plan.md) · [Deep dive](./Module1_Copilot_Deep_Dive.md)

---

## Why this gate exists

Toast-style **actions** reuse the same agent loop as analytics: pick a tool → execute → narrate. Writes add permissions, preview, confirm, and audit. If routing, fetch health, or numeric grounding are weak on **reads**, they become dangerous on **writes**.

Do **not** start the action section until the exit criteria below are met (or explicitly waived).

---

## Where we are (2026-09)

| Layer | Status |
|-------|--------|
| CORE tool loop + slim DTOs | Done — `bun run smoke:launch` **12/12** |
| Intent filter (frozen ~12 schemas) | Done — pins outside CORE are **logged then dropped** |
| Platform help (`search_platform_help` → `get_task_howto`) | Done |
| Numeric answer validator | Shipped (log-only; compare asks often unmatched) |
| Eval matrix Phase-2 | **12/18** gate; EXISTS **0/6** by design until pins honored |
| Fetch / step / grounding assertions in scripts | **Shipped** — `bun run eval:matrix` ( `--route-only` to disable ) |
| Multi-turn M1–M6 | **Green** M1 / M5 / M6 (`bun run eval:multiturn`, 2026-09-14) |
| Write / HITL actions | Explicitly deferred — correct |

**Evidence:** `aiDB.ai_usage_log` (~257 docs). Filter diagnoses to current era:

```javascript
{
  cached: false,
  "promptMetrics.registeredToolCount": 12,
  "promptMetrics.toolFilterMode": "intent",
  "toolResults.toolFailed": { $ne: true }
}
```

Ignore pre-filter docs (`registeredToolCount: 27`) and the Sep-7 analytics-edge-down batch (`toolFailed`, `fetchMs` 1–5, `resultChars` 97).

---

## Discuss first (product / architecture locks)

Short decisions — not builds:

1. **Pin honor vs CORE grow** — Prefer keep CORE at 12 and **union A-list pins** (see thread A). Do not start actions with a moving schema set on every ask.
2. **Action contract** — Preview → confirm → mutate → audit. No silent writes. POLICY suite must keep refusing until confirm exists.
3. **What “action” means for v1.5** — Deep-link how-tos first; then at most 1–2 HITL writes (e.g. open shift editor / publish draft post). Skip inventory, delete-orders, price-edit until permissions are clear.
4. **Definition of “agent ready”** — Not `--strict` 18/18 alone. Need: correct route **+** healthy fetch **+** grounded `$`/`%` **+** multi-turn session re-fetches.
5. **Outage behavior** — Soft `{ error: "Could not load…" }` must **fail** eval and never look like a green smoke. Actions will share `safeExecute`.

---

## Test next (four layers)

Today `eval:matrix` / `smoke:launch` mostly score **tool name**. Promote to four layers on the same SSE / `requestId` (join `ai_usage_log` when needed):

| Layer | Pass if | Fail examples already seen |
|-------|---------|----------------------------|
| **1. Route** | Expected tool in `toolsCalled`; pin honored when present | EXISTS pins dropped; kitchen-now → help path once |
| **2. Fetch** | `!toolFailed`; `fetchMs` / `resultChars` in band | 97-char / 2–5 ms = edge down; pre-slim 4404-char summary |
| **3. Loop** | Analytics ≈ 2 steps (tool → text); HOWTO ≈ 3 (search → howto → text); POLICY = 1 step, no analytics tools | Extra stack; HOWTO burned on kitchen ask |
| **4. Ground** | `answerValidation.ok` or `unmatchedCount === 0` | Invented `0.3%` on S5; compare asks with 3–4 unmatched `$` |

### Cheap harness upgrades

- Extend `eval:matrix` (or add `eval:fetch`) with layers 2–4; treat Redis cache hits (`0` tok, empty `toolsCalled`) as **skip**, not fail.
- Script **M1–M6** with `sessionId` (follow-up must call a tool again, not invent from chat).
- One **outage case**: analytics unreachable → soft error language, `toolFailed: true`, no Redis cache write.
- Daily gate: `bun run smoke:launch` + `bun run eval:matrix` (**without** `--strict` until pins land).
- `--strict` only after pin-honor (or intentional CORE adds).

### Suggested fetch bands (healthy CORE-12, edge up)

| Tool | Typical `resultChars` | Typical `fetchMs` |
|------|----------------------:|------------------:|
| `get_revenue_summary` | ~400–600 | ~100–800 |
| `get_period_comparison` | ~350–650 | ~100–300 |
| `get_revenue_diagnosis` | ~900–950 | ~100–200 |
| `get_revenue_by_day` (+ compare) | **>1500** (dual fetch) | ~200–900 |
| `get_menu_health` | ~1.5k–2.5k | ~100–300 |
| `get_kitchen_activity` | ~100–150 | ~100–200 |
| `search_platform_help` / `get_task_howto` | ~300–650 | **&lt;5** (in-process) |
| Soft error | **97** | **1–5** → **fail the case** |

---

## Add to final read-only setup (ship order)

### Must before actions

1. **Honor `pinnedToolNames`** outside CORE (+ R4 dine-in / takeout / delivery pin) — closes EXISTS without stuffing CORE.
2. **Eval layers 2–4** — green means data arrived and numbers match.
3. **Multi-turn smoke** — at least 3 cases (compare follow-up; collected after tips; prep after kitchen).
4. **Prod launch leftovers** — `ai/staff-ops` live; discount ask vs Reports; rate limits; `ai_usage_log` indexes; rollout flag.

### Should (same week if time)

5. Fix kitchen-now false how-to path (seen in usage logs).
6. T3 NL: “this month vs last month” full dual window, not only `"last month"`.
7. Validator review on compare asks — log vs `AI_ANSWER_VALIDATOR_STRICT` for prod.
8. Thin `get_platform_capabilities` (~2574 chars) — optional cost cut before write tools add schemas.

### Defer until after first HITL action design

- `find_analytics_tools` / ToolSearch meta-tool (`v1/` pattern)
- Semantic cache / embedding few-shot from `ai_usage_log`
- Inventory / CRM / email-send tools
- Hard-block validator for all merchants
- Module 2 forecasting

---

## Exit criteria — “agent ready” for actions

All of the following (or written waiver):

- [ ] `smoke:launch` 12/12 on a healthy stack
- [ ] `eval:matrix` CORE/HOWTO/POLICY green; EXISTS either reachable (pins) or explicitly still `~`
- [ ] Fetch/ground assertions fail on edge-down and invented `$`/`%`
- [ ] ≥3 multi-turn cases with `sessionId` pass (re-tool on follow-up)
- [ ] POLICY still refuses write-shaped asks (C3 / G2) with no analytics mutate path
- [ ] Compass / log slice shows phantom tool calls ≈ 0 on intent CORE-12

---

Read-only work (pins, Redis how-to cache, eval:fetch, grounding badge) is specified in [Read-agent hardening plan](./Module1_Copilot_Read_Agent_Hardening_Plan.md). Thread **C** stays discussion-only until that plan’s “read agent done” checklist.

## Next discussion threads (pick one)

### (A) Pin-honor design details

**Problem:** Intent mode freezes CORE schemas for Gemini cache stability. `getPinnedToolNames()` still computes pins (`get_payment_overview`, `get_fulfillment`, …) but `resolveActiveToolNames` only adds pins that are already in CORE — no-op.

**Design sketch:**

- Keep `CORE_TOOL_NAMES` as the default registered set.
- Union **allowlisted** pins into `activeToolNames` for that ask only.
- Cap with `AI_TOOL_FILTER_MAX` (default 15).
- Accept: Gemini implicit cache hits on plain CORE asks; may miss on rare pinned asks (correct trade).
- Add R4 patterns for dine-in / takeout / delivery → `get_revenue_mix` (today `pinnedToolNames: []` on that ask).

**A-list allowlist (eval EXISTS):**

| Tool | Eval ID |
|------|---------|
| `get_payment_overview` | S5 |
| `get_operations_overview` | S6 |
| `get_reservations_summary` | S7 |
| `get_revenue_mix` | R4 |
| `get_peak_hours` | R5 |
| `get_fulfillment` | L2 |

**Code:** `ai-edge-api/src/tools/toolFocusMap.ts`, `questionIntent.ts`  
**Verify:** `bun run eval:matrix -- --strict` after change.

---

### (B) Exact `eval:fetch` assertions

**Status (2026-09-14):** Shipped in `eval:matrix` (always-on fetch/ground; `--route-only` to disable). Unit: `bun run test:eval-layers`. SSE meta carries `toolFailed`, `compareError` / `compareOverlay`, `stepCount`, `grounding`. Redis how-to hit = skip. Do **not** use a raw ~97-char size band (`get_fulfillment` can be ~80). Compare overlay is asserted on **T2** only.

**Verify:** Deliberate analytics-down run fails `fetch: toolFailed`; invented `$`/`%` fails `ground: unmatched`. T3/N2 compare-DTO grounding may stay red until step 4 slim `this`/`prev`/`delta`.

---

### (C) First HITL action shape (preview / confirm / audit)

**Problem:** Write-from-chat is out of scope until the read agent is trusted — but the **contract** should be locked before the first mutate tool exists.

**Design sketch:**

| Step | Behavior |
|------|----------|
| **Preview** | Tool returns proposed change + human-readable summary; **no side effect** |
| **Confirm** | Merchant explicit yes (UI button or typed confirm); new turn / dedicated endpoint |
| **Mutate** | Permission-checked call to owning edge API; idempotency key |
| **Audit** | `ai_usage_log` (+ domain audit) with actor, store, payload, result |

**v1.5 candidates (pick ≤2):** deep-link only **or** one low-risk confirmable write (e.g. open Social Publisher draft / navigate to shift publish).  
**Still refuse:** delete orders, change menu prices, inventory 86, email-send — until product + permissions say otherwise.

**Code (future):** new action tools separate from analytics `safeExecute`; portal confirm UX; POLICY tests stay red until confirm path exists.  
**Verify:** POLICY matrix unchanged until preview tool ships; then add confirm/cancel cases.

---

## One-session agenda

1. Confirm exit criteria above.  
2. Pick **(A)**, **(B)**, or **(C)** and detail the design.  
3. Implement that thread only; re-run smoke + eval.  
4. Revisit remaining threads before any mutate tool.

---

## Daily commands (until pins land)

| Command | Role |
|---------|------|
| `cd ai-edge-api && bun run smoke:launch` | 12/12 CORE regression |
| `bun run eval:matrix` | Phase-2; EXISTS print `~` |
| `bun run eval:matrix -- --strict` | Only after pin-honor (or CORE expansion) |
| `bun run test:validator` | Unit self-test for grounding |

Flush Redis only when HOWTO/POLICY rows show `0` tok / empty `toolsCalled` and you distrust cache.
