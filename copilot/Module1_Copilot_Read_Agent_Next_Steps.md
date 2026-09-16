# Copilot — remaining read-agent steps

> **Goal:** Close the read-only agent (route → fetch → narrate → ground) **before HITL writes**.  
> **Status:** Steps 1–3 done (2026-09-14). HITL **design** landed (2026-09-16) — [action contract](./Module1_Copilot_HITL_Action_Contract.md). No mutate code.  
> **Related:** [Hardening plan](./Module1_Copilot_Read_Agent_Hardening_Plan.md) · [Eval matrix](./Module1_Copilot_Eval_Matrix.md) · [Pre-action readiness](./Module1_Copilot_Pre_Action_Readiness.md) · [Measure snapshot](./Module1_Copilot_Measure_Snapshot.md)

Do **not** start preview / confirm / mutate until this file’s “read agent done” bar is met (or explicitly waived).

---

## Where we are

Hardening steps **1–4 landed**:

| Done | Why it mattered |
|------|-----------------|
| A-list pins, CORE-first order | EXISTS tools callable without busting the Gemini prefix |
| How-to Redis only (feature fingerprint, not `storeId`) | Analytics answers never cached wrong |
| Eval fetch / loop / ground | `toolFailed` / invented `$`/`%` go red; Redis how-to hit = skip |
| `deltaAmount` + portal badge on `failed` only | Compare dollar-deltas are allowlisted; merchant sees Ground fail late, not buffered |
| Mix slices padded at `sharePct: 0` | Missing delivery is a copied `0.0%`, not an unmatched invention |

Live `eval:matrix -- --strict` is a **first-turn** gate. Locked decision #5 still holds: that is necessary and **not** “read agent done.”

---

## Why anything else

A merchant does not ask one isolated question. They ask “how were sales?” then “by day” then “why was Tuesday bad?” If turn 2 **invents from chat** instead of calling a tool, Ground only checks this turn’s DTO — last turn’s numbers look “in session” and sail through. Writes will reuse the same session loop. If follow-ups do not re-fetch, a HITL preview will be built on stale chat.

Gemini implicit cache (`cachedInputTokens`) is the other unfinished question. We **paid** for CORE-first mixed schemas so plain asks keep a stable prefix. Eval logs so far show `cachedInputTokens: 0` even on CORE-12. That is data, not a reason to rip the design — **measure before changing**.

---

## Order (do in this sequence)

```
1. Multi-turn eval     ← remaining product gate
2. Measure (logs only) ← decide keep CORE-first vs two-call pins
3. Eval hygiene        ← cheap, can overlap 1
────────────────────────────────────────────
stop  →  call read agent done
   or →  HITL design (not code) using the Reddit/write contract
```

| # | First? | What | Why this order |
|---|--------|------|----------------|
| **1** | **Yes — start here** | Script ≥3 `sessionId` follow-ups that **call a tool again** | This is the locked “done” bar. Single-turn 18/18 cannot catch “narrate from history.” Future writes share this session. |
| **2** | After 1 is running (same week OK) | Sample `ai_usage_log`: numeral coverage + `cachedInputTokens` CORE-12 vs pinned | No product change until data. A two-call pin stage costs TTFT on every EXISTS ask — do not pay that on a guess. |
| **3** | Anytime; do not block 1 | Eval copy + T3 date-range honesty | Fixes confusing fail text and a golden-window hijack. Small. Does not make the agent more trustworthy by itself. |

**Do not** insert new analytics tools, validator widen (counts/dates/tuples), or HITL between these.

---

## Step 1 — Multi-turn eval (first)

### Why

| Single-turn today | Multi-turn gap |
|-------------------|----------------|
| New `sessionId` every case | Real chat reuses session |
| Tool name + fetch + `$`/`%` on **this** DTO | Follow-up can copy last answer and skip tools |
| POLICY refuse on a blank slate | “Delete those orders” after an analytics turn is a different fail |

Sep-7 looked like routing success with a dead edge. First-turn fetch/ground closed that. Multi-turn closes **session amnesia / session hallucination**.

### What to script

Start with **three**, not all six. Same `DATE_RANGE` (Feb 3–28, 2026). Pass `sessionId` from turn 1’s `X-Session-ID`. **Fail if turn 2+ has empty `toolsCalled`** (unless the case is pure refuse).

| ID | Turns | Pass if | Why this one first |
|----|-------|---------|--------------------|
| **M1** | (1) How were sales last week? → (2) Break that down by day → (3) Why was the worst day bad? | Each turn calls a tool: summary/compare → `get_revenue_by_day` → diagnosis. Ground still `$`/`%` ⊆ that turn’s DTO. | Canonical analytics drill-down. If this fails, the agent is a one-shot reporter, not a copilot. Runner wording: **this period** (not “last week”) so NL does not steal the golden Feb window. |
| **M5** | (1) Kitchen right now? → (2) How long does prep usually take? | (1) `get_kitchen_activity` · (2) `get_fulfillment` (pin on follow-up). | Proves **pins still honor on turn 2**, not only first ask. |
| **M6** | After any analytics answer: “try again” / “redo now” | Re-fetches; does not reuse stale numbers. | The “refresh” path merchants will hit; also the Redis/analytics-cache invariant. |

Then (same harness, not a new project):

| ID | Turns | Pass if |
|----|-------|---------|
| **M3** | Export report → and fiscal data? | Two how-to chains; **no** analytics tools |
| **M4** | Discount/tip rate → collected vs outstanding? | `get_payment_details` → `get_payment_overview` pin |

M2 (top items → weak items) is lower ROI; menu health already covers both on CORE.

### How it benefits everything

- **Merchants:** follow-ups stay on live Reports, not last paragraph.  
- **Eval:** one script covers the “done” checklist item that 18/18 cannot.  
- **Ground:** each turn’s DTO is the allowlist; history cannot launder numbers.  
- **Writes later:** preview/confirm will be extra turns on the **same** session. If M1 already re-tools, HITL does not invent a second loop.  
- **Pins:** M5 is the regression that pin-union on first turn only would miss.

### Code

Harness landed in `ai-edge-api` (does **not** run the 18 first-turn cases):

- `tests/evalAsk.ts` — shared POST `/ask` + SSE parse; reads `X-Session-ID`
- `tests/evalRoute.ts` — route judge + `retoolFailReason` (`route: follow-up did not re-tool`)
- `tests/run-eval-multiturn.ts` — M1 / M5 / M6; `mustRetool` on turn 2+
- `tests/run-eval-matrix.ts` — `--multiturn` dynamically imports the harness
- `bun run eval:multiturn` or `bun run eval:matrix -- --multiturn`

Reuse `judgeFetchGround`. `SessionManager` already persists turns. EXISTS on M5 is a hard fail (no `~`).

### Done when

Three cases green on a healthy stack: each follow-up has `toolsCalled.length > 0`, `!toolFailed`, grounding not `failed`. Harness is not that bar.

---

## Step 2 — Measure (no extra product change)

### Why

CORE-first mixed schemas were a **cost/latency bet**: keep Flash implicit cache on plain CORE asks; accept a tail miss when a pin is appended. Eval already showed `cachedInputTokens: 0` / `cacheHitRatio: 0` on first-turn CORE-12. That might be eval (cold sessions, 8s apart) or a real prefix miss. **Logs decide**; code does not.

Numeral coverage (how many answer digits the `$`/`%` regex even sees) tells us whether widening Ground to counts/dates is worth it. Measure first — the code map already lists what Ground does **not** catch.

### What to pull (`ai_usage_log`)

Filter: `cached: false`, `toolFilterMode: intent`, `toolResults.toolFailed` ≠ true.

| Question | Slice |
|----------|--------|
| Does CORE-12 ever hit Gemini cache? | `registeredToolCount === 12` vs `> 12` (pinned) · `cachedInputTokens`, `cacheHitRatio` |
| What % of answer digits are `$`/`%`? | Sample `answerChars` vs unmatched / mentionCount (or a one-off script on stored answers if present) |

### How it benefits everything

- **Avoid a bad rewrite:** two-call “CORE then pins” adds a round-trip to every EXISTS ask. Only do it if CORE-12 cache is proven dead.  
- **Avoid a noisy validator:** counts/dates/superlatives will false-red eval if coverage is still the `$`/`%` subset.  
- **Cost:** if CORE-12 *does* cache in production traffic (longer sessions than eval), keep paying for 12+pin schemas. The bet was never “eval first token is free.”

### Done when

A short written snapshot (even a table in this folder): CORE-12 vs pinned cache hit; “widen Ground: yes/no.” No schema change required to call this done.

**Landed:** [Measure snapshot](./Module1_Copilot_Measure_Snapshot.md) — CORE-12 **0/96** cache hits; pinned **0/51**; widen Ground **no**. Keep CORE-first mixed schemas. Re-run `bun run measure:usage`.

---

## Step 3 — Eval hygiene (do not block step 1)

Cheap. Can ship in the same PR as multi-turn or alone.

| Fix | Why | Benefit |
|-----|-----|---------|
| Stop printing `EXISTS still gated by CORE` on any EXISTS fail | R4 failed **Ground** (`0.0%`) while `get_revenue_mix` **was** called. The line lied. | Operators trust `--strict` output; pin regressions stay distinct from Ground. |
| T3 wording / date pin | “How did this month compare to last month?” NL-matched **last month** and swapped the golden Feb range for August (all zeros). | Compare eval tests **compare**, not empty-month NL. N2 is safer (“last week to the week before”) once the runner’s window is respected. |

Reword T3 to “How did **this period** compare to the prior period?” or disable NL override in the eval body. Do not change production NL parsing just to make T3 green.

**Landed:** T3 eval question is “this period / prior period.” EXISTS summary prints pin/route miss vs fetch/ground fail (no “still gated by CORE”).

---

## What this does *not* add

Stay out until read-agent closeout (or a new plan):

| Temptation | Why wait |
|------------|----------|
| HITL preview / confirm / mutate | Different loop: ground truth exists **after** the write. Reddit notes (idempotency, compare-and-set, read-back, blast-radius cap) belong in thread C, not here. |
| Counts / dates / (value + period) tuples in Ground | Step 2 coverage first. |
| Per-day overlay `deltaAmount` | Period-level compare totals are enough for T2/T3. |
| Pad zeros on `get_revenue_diagnosis` mix | R4 uses `get_revenue_mix`; diagnosis is a different DTO. |
| Register all ~27 tools / live ToolSearch | Pins + CORE cover the matrix. |
| Buffer SSE until Ground | Locked: late badge only. |
| New analytics tools (inventory, guests, email-send) | No facts / POLICY. |

---

## How the three steps benefit the whole product

```
         merchants                 eval                    later writes
            │                        │                          │
   follow-ups re-tool  ←──── 1 ────→ session gate         same session loop
            │                        │                          │
   numbers from DTOs   ←──── 2 ────→ keep or change CORE   don't guess TTFT
            │                        │                          │
   honest compare month ←──── 3 ────→ honest ✗ reasons     n/a
```

One sentence each:

1. **Multi-turn** makes Copilot a conversation over live tools, which is what the portal session already is.  
2. **Measure** stops us from “fixing” Gemini cache with an extra model call we may not need.  
3. **Hygiene** makes the gate we already have readable, so a Ground miss is not reported as a missing pin.

Together they finish the read path the hardening plan defined: **route + healthy fetch + grounded `$`/`%` + ≥3 re-fetches**. Then HITL is a product discussion, not a panic patch.

---

## “Read agent done” (this closeout)

- [x] Pins + R4; how-to Redis; fetch/ground eval; `deltaAmount`; badge on `failed`; mix `0` slices  
- [x] Multi-turn harness (M1 / M5 / M6, `sessionId`, re-tool judge)  
- [x] **M1, M5, M6** green (`sessionId`, re-tool, fetch/ground) — local 2026-09-14, 3/3 chains / 7/7 turns  
- [x] Cache-hit / numeral snapshot recorded (step 2) — [Measure snapshot](./Module1_Copilot_Measure_Snapshot.md)  
- [x] Eval EXISTS fail reason is accurate; T3 does not NL-steal the golden range  
- [x] HITL **design** (not code) — [Action contract](./Module1_Copilot_HITL_Action_Contract.md)

---

## Commands (after step 1 lands)

| Command | Role |
|---------|------|
| `bun run smoke:launch` | 12/12 CORE |
| `bun run eval:matrix -- --strict` | First-turn route + fetch + ground |
| `bun run eval:multiturn` | M1 / M5 / M6 sessioned (`eval:matrix -- --multiturn` is the same) |
| `bun run measure:usage` | Step 2: CORE-12 vs pinned cache + numeral coverage |
| `bun run test:eval-layers` / `test:validator` | Unit judges |

Needs local `ai-edge-api` + `analytics-edge-api` (same as today).
