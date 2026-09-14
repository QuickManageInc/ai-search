# Copilot measure snapshot (step 2)

> **When:** 2026-09-14  
> **Source:** `aiDB.ai_usage_log` via `bun run measure:usage` (`tests/measure-usage-log.ts`)  
> **Filter:** `cached: false`, `promptMetrics.toolFilterMode: intent`, `toolResults.toolFailed` ≠ true  
> **n:** 229 docs (76 in last 7 days)

No product change in this step. Two decisions only.

---

## Gemini implicit cache (CORE-12 vs pinned)

| Slice | n | docs with `cachedInputTokens > 0` | Per-step cache hits | t1 hits | t2+ hits |
|-------|--:|----------------------------------:|--------------------:|--------:|---------:|
| **CORE-12** (current prefix) | 96 | **0 (0%)** | 0 | 0/87 | 0/9 |
| **Pinned >12** | 51 | **0 (0%)** | 0 | 0/46 | 0/5 |
| Under-12 (older CORE sizes) | 82 | 5 (6.1%) | 2 | 3/65 | 2/17 |
| All intent | 229 | 5 (2.2%) | 2 | 3/198 | 2/31 |

`registeredToolCount` histogram: `9:23  10:47  11:12  12:96  13:23  14:2  15:26`.

The five hits are **all CORE-10 era** (`get_period_comparison` / `get_revenue_summary` / `get_revenue_by_day`). `cachedInputTokens` ≈ 1700–1775 (~40% of that request’s input). So Gemini **can** cache a frozen prefix.

Last 7 days (current CORE-12 + pins): **0/55 CORE-12, 0/21 pinned**, including 9 CORE-12 follow-ups from `eval:multiturn`.

### Decision: keep CORE-first mixed schemas

**Do not** add a two-call “CORE then pins” stage.

Two-call only pays off if CORE-12 is a reusable implicit-cache prefix. In this log it is not: 0/96 including later turns. Splitting EXISTS asks into a second model call would add TTFT with nothing to reuse.

Pinned >12 also never hits. That is expected if a pin busts the prefix — but it does not justify two-call while CORE-12 itself is cold.

Caveat: this corpus is mostly local eval + one store, short sessions. Re-run `bun run measure:usage` if production volume appears. Do not rewrite on a guess.

---

## Numeral coverage (Ground `$` / `%`)

Answers are not stored whole. Coverage uses **assistant previews** on follow-up turns (`AI_LOG_PREVIEW_CHARS` ≈ 400). 42 samples.

| Metric | Value |
|--------|------:|
| Digit chars inside `$` / `%` mentions | 734 |
| Digit chars in all numerals | 1606 |
| ISO `YYYY-MM-DD` digits | 48 |
| **Coverage `$`/`%` vs all digits** | **45.7%** |
| Coverage excluding ISO dates | 47.1% |
| Previews with ≥1 `$`/`%` | 35 / 42 |
| Stored `answerValidation` | 129 ran, 14 failed, 34 unmatched mentions |

Leftover numeral tokens (not `$`/`%`): years (`2026`), calendar days (`3`, `28`, `23`), counts (`36` orders, `0`, `1`, `6`). Not missed dollar amounts.

`mentionCount` is not persisted on `ai_usage_log` (only `unmatchedCount` / status). Re-measure from previews is enough for this decision.

### Decision: widen Ground: **no**

More than half of answer digits are counts, dates, and years. Flipping the extractor to “every numeral” without DTO allowlists for those would false-red eval (`36 orders`, `February 23`, `2026`). Keep Ground on `$`/`%` until a later plan adds count/date extractors **and** allowlists together.

---

## What this does *not* change

- Tool schema order / pin-union
- Validator regex
- HITL
