# LLM Hosting Decision — Gemini API vs Self-Hosted

> **Question:** Should we keep paying for Gemini 2.5 Flash (API), or train / run an open-weight LLM in production?  
> **Scope:** Analytics Assistant (`ai-edge-api`) at 20 / 50 / 100 merchants, 6-month horizon.  
> **Related:** [ai-edge-api/README.md](../ai-edge-api/README.md), [Module1_Response_Accuracy_Plan.md](./Module1_Response_Accuracy_Plan.md), [Module1_Context_Optimization_Plan.md](./Module1_Context_Optimization_Plan.md)

---

## 0. Assumptions (locked)

| Assumption | Value | Why |
|------------|-------|-----|
| Usage per merchant | **30 asks/day** | “Good amount” — ~1 question every 20 min of business hours |
| Merchant tiers | **20 / 50 / 100**, independent steady-state (not a ramp) | Cleanest comparison |
| Self-host options | **Both:** 8B fine-tuned (QLoRA) **and** 70B as-is | Covers cheap-small vs quality-proxy |
| Labor in TCO | **Yes** — include ops/MLOps | Usually the cost that flips the verdict |
| Accuracy analysis | **Qualitative** (benchmarks + our tool-calling architecture) | No new eval harness required for this pass |
| Horizon | **6 months** | |
| API baseline | **Gemini 2.5 Flash** — $0.30 / 1M input, $2.50 / 1M output, $0.03 / 1M cached input (Aug 2026) | |

---

## 1. The real workload (from `ai_usage_log`)

Sample asks:

| | inputTokens | outputTokens | totalTokens |
|---|---|---|---|
| Ask 1 | 5,267 | 198 | 5,465 |
| Ask 2 | 6,082 | 355 | 6,437 |
| Ask 3 | 4,238 | 64 | 4,302 |
| **Average** | **5,196** | **206** | **5,402** |

**96% of tokens are input.** Almost the entire bill is the tool-fetched analytics context (slim DTOs + system prompt + history), not the answer. That shape favors:

1. API pricing that is cheap on input (Flash is), and
2. Context caching / prompt compression over self-hosting, as the first cost lever.

### Cost per ask on Gemini 2.5 Flash

```
(5,196 × $0.0000003) + (206 × $0.0000025) ≈ $0.00207 / ask
```

At 30 asks/merchant/day:

| | Per merchant |
|---|---|
| Per day | $0.062 |
| Per month (30 days) | $1.86 |
| Per 6 months | **$11.18** |

---

## 2. Scenario A — Stay on Gemini 2.5 Flash (current path)

No new infra. Linear with merchant count.

| Merchants | $/month | $/6 months |
|---|---|---|
| 20 | $37 | **$224** |
| 50 | $93 | **$559** |
| 100 | $186 | **$1,118** |

At 100 merchants × 30 asks/day × 6 months, the entire LLM bill is **~$1,118** — less than one engineer-day fully loaded.

### Cheap wins still on the table (no self-hosting)

The system prompt + tool schema list is a near-identical prefix on every call. Enabling Gemini **context caching** ($0.03/1M vs $0.30/1M for that portion) is a plausible ~50% further cut for near-zero engineering effort. Slim DTOs and session compression already attack the same input-heavy cost shape (see Module1 Context Optimization).

---

## 3. Scenario B — Self-host

Both tiers need work we do **not** have today:

- GPU-enabled node pool (nothing in the stack runs on GPUs)
- Model artifact storage + registry
- vLLM (or equivalent) deploy, health checks, autoscaling
- Merchant-facing moderation / guardrails (Gemini provides this implicitly)
- Eval harness before trusting tool-selection accuracy in production

**Throughput note:** one L4/L40S can sustain roughly 2,000–4,000 tokens/sec. Even 100 merchants at 30 asks/day average only ~187 tokens/sec across the day. **One GPU (or two for HA) covers 20 / 50 / 100 with headroom.** Self-host cost is dominated by **fixed infra + labor**, not merchant count.

### Tier B1 — 8B fine-tuned on our logs (QLoRA)

| Cost item | Estimate |
|---|---|
| Fine-tune compute (QLoRA, iterative) | $20–50 |
| Data curation + eval pipeline (labor) | ~3–5 eng-weeks |
| GPU serving infra build (labor) | ~2–3 eng-weeks |
| **One-time total** | **~$18,000** |
| GPU (1× L4, 24/7, no HA) | $576/mo |
| Ops / on-call (0.15–0.2 FTE) | $1,800–2,400/mo |
| **Ongoing** | **~$2,700/mo → $16,200 / 6mo** |
| **6-month total** | **≈ $34,000** |

Fine-tuning itself is cheap. The bill is engineering + always-on GPU + on-call for a new failure class.

### Tier B2 — 70B as-is (no fine-tune)

| Cost item | Estimate |
|---|---|
| Eval / validation (labor) | ~2–3 eng-weeks |
| GPU serving infra (tensor parallel / quant) | ~3–4 eng-weeks |
| **One-time total** | **~$16,500** |
| GPU, marketplace (1× H100 @ ~$2.49/hr, 24/7) | $1,793/mo |
| GPU, AWS on-demand (1× A100 @ ~$22/hr, 24/7) | $15,840/mo |
| Ops / on-call (0.2–0.3 FTE) | $2,400–3,600/mo |
| **6-month total (marketplace GPU)** | **≈ $45,000** |
| **6-month total (AWS on-demand GPU)** | **≈ $130,000** |

---

## 4. Side-by-side — 6 months

| Scenario | 20 merchants | 50 merchants | 100 merchants |
|---|---|---|---|
| **Gemini 2.5 Flash API** | $224 | $559 | $1,118 |
| Self-host 8B fine-tuned | ~$34,000 | ~$34,000 | ~$34,000 |
| Self-host 70B as-is (marketplace) | ~$45,000 | ~$45,000 | ~$45,000 |
| Self-host 70B as-is (AWS on-demand) | ~$130,000 | ~$130,000 | ~$130,000 |

Self-hosting is roughly **30×–500× more expensive** than the API at every tier we modeled. API cost is variable and tiny; self-host cost is fixed.

### Break-even merchant count (at 30 asks/day/merchant)

| Self-host tier | Full TCO (incl. labor) | Infra-only (excl. labor, optimistic) |
|---|---|---|
| 8B fine-tuned | ~3,100 merchants | ~310 merchants |
| 70B as-is (marketplace) | ~4,000 merchants | ~950 merchants |
| 70B as-is (AWS) | ~11,600 merchants | ~8,500 merchants |

Even ignoring labor, we need **3–8×** more merchants than the largest modeled tier. With labor: **30–115×**.

Break-even scales with **total token volume**, not headcount. If per-merchant usage grows ~10× (e.g. global store copilot, proactive cards), break-even merchant counts drop by roughly the same factor — into a more plausible multi-year range, still not “next quarter.”

Industry rule of thumb (2026): sustained self-host wins often land around **2–5M tokens/day** with decent GPU utilization. At 100 merchants × 30 asks × 5.4k tokens ≈ **16M tokens/day peak theoretical**, but real concurrent load is far lower and still does not pay for the ops overhead vs Flash’s low $/token.

---

## 5. Work & complexity

| Dimension | Gemini API (current) | Self-host 8B fine-tuned | Self-host 70B as-is |
|---|---|---|---|
| Time to first deploy | Already live | 6–8 eng-weeks | 5–7 eng-weeks |
| New infra surface | None | GPU pool, vLLM, model registry | Same + tensor parallel / quant |
| Ongoing maintenance | None (Google’s problem) | Driver/vLLM upgrades, retrain cadence, new on-call | Same, larger blast radius |
| Safety / moderation | Built into API | We must build it | We must build it |
| Model freshness | Automatic | Manual re-eval + redeploy per open release | Same |
| Burst / elastic load | Instant | Overprovision or slow GPU autoscale | Same, worse |
| Vendor risk | Google pricing / ToS | GPU supply, open-model licenses | Same |

This is a **new operational domain**: nothing in `ai-edge-api`, `analytics-edge-api`, or sibling services today owns GPU lifecycle, model weights, or a first-party safety layer.

---

## 6. Response accuracy (qualitative)

Weighted heavily against self-hosting at this stage, independent of cost.

- **`Module1_Response_Accuracy_Plan.md` already documents Gemini Flash getting tool selection wrong** (bestsellers vs underperforming confusion, empty `toolsCalled` on analytical follow-ups) — and Flash is a frontier lab model with heavy investment in agentic tool-calling.
- **8B fine-tuned:** QLoRA teaches *style and format* well; it generalizes poorly to tool-routing questions unlike the fine-tune set. For a numbers-adjacent product that is a hallucination-risk regression unless we invest in a large, curated “confusable question” dataset (exactly what the accuracy plan had to build by hand).
- **70B as-is:** closer to Flash’s general reasoning tier, but open mid-large models still typically trail frontier flash-class models on **function-calling precision** — the one skill this service depends on.
- **Mitigating factor:** architecture already keeps the LLM’s job narrow (narrate tool JSON; don’t invent numbers). The accuracy gap is mostly tool selection + prose faithfulness. Still, a regression there undoes recent hardening work.

**Net:** swapping to a smaller/self-hosted model is likely a step **backward** on accuracy unless we replicate a meaningful fraction of Google’s tool-calling RLHF — which an $18–45k self-host project does not buy.

---

## 7. Other metrics

| Metric | API | Self-host |
|---|---|---|
| Latency | Network hop to Google; fine at our scale | Can be lower same-VPC if GPU idle; worse under burst if saturated |
| Data residency | Prompts leave to Google (paid tier: not used to improve products, still leaves VPC) | Stays in our infra — **the one real non-cost argument** if an enterprise/regulated merchant requires it |
| Price risk | Google can raise rates (market commentary already flags future increases) | Own the compute cost curve once built |
| Cache / batch levers | Context cache, Batch API (50% off, async — poor fit for interactive ask) | Continuous batching in vLLM helps utilization, not product UX |

---

## 8. Verdict

At **20 / 50 / 100 merchants** and **30 asks/day/merchant**, **staying on Gemini 2.5 Flash is not close**:

- ~**30×–500× cheaper** over 6 months
- **Zero** new infra
- Currently the **stronger** option on the skill that matters most (tool selection)

Self-hosting becomes economically rational somewhere around **~3,000–12,000 merchants** at this usage level (or much sooner if usage per merchant jumps an order of magnitude). Even then we would trade a small, well-understood API cost for a new GPU-ops discipline and a real accuracy downgrade risk unless heavily invested.

### When to reopen this decision

1. **Data residency / compliance** from a specific enterprise merchant that Gemini’s terms cannot satisfy → prefer a **hybrid** (self-host only that tenant’s traffic), not a wholesale switch.
2. Sustained volume approaching **~2–5M tokens/day** with clear GPU utilization, **and** a dedicated owner for inference ops.
3. Open-weight models demonstrably match Flash on **our** tool-calling eval set (golden Q&A + LLM-as-judge) — measure before migrating.

### What to do instead of self-hosting (near term)

1. Keep Gemini 2.5 Flash (or Flash-Lite for cheaper paths where quality allows).
2. Turn on **context caching** for the stable system + tools prefix.
3. Keep pushing **slim DTOs, atomic tools, session compression** (input-heavy bill).
4. Invest accuracy effort in **tool descriptions, validators, golden evals** — not in owning GPUs.
5. Revisit pricing logs quarterly; if merchant × usage product crosses the break-even band above, run the numbers again.

---

## Appendix — Pricing sources (Aug 2026)

- Gemini 2.5 Flash: [Google AI pricing](https://ai.google.dev/gemini-api/docs/pricing) — $0.30 / 1M in, $2.50 / 1M out, $0.03 / 1M cached in
- Representative GPU on-demand: AWS g6 (L4) ~$0.80/hr, g6e (L40S) ~$1.86/hr, A100 class ~$22–27/hr on AWS; marketplace H100 often ~$2.49/hr
- Self-host $/1M token and break-even heuristics: 2026 industry writeups (vLLM throughput ÷ GPU $/hr; crossover often cited ~2–5M tokens/day or tens of millions of tokens/month depending on model size and utilization)
- Fine-tune (QLoRA 8B): typically dollars to tens of dollars per run; project cost dominated by data/eval labor, not GPU hours
