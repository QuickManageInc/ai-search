# QuickManage — AI Module Architecture

> Full architecture, technology decisions, and build order for the three selected AI modules.

---

## Selected modules

1. **Analytics Assistant** — natural language interface over all analytics data (sales, employee, performance)
2. **Revenue Forecasting** — hybrid nightly worker + cache approach to predict merchant revenue
3. **Social Publisher** — human-in-the-loop AI copywriter that drafts and publishes Instagram posts from menu items

---

## Platform overview

All three modules deploy as new services inside the existing EKS cluster, alongside existing edge APIs, with shared infrastructure.

```
EKS cluster (cloud-managed, existing)
│
├── Merchant portal (Vercel — existing dashboard extended with AI panels)
│
├── Existing edge APIs
│   ├── analytics-edge-api     → aggregated daily summaries, item data
│   ├── menus-edge-api         → items, pricing, descriptions, images
│   ├── employee-edge-api      → shifts, availability, performance
│   └── other edge APIs        → orders, loyalty, etc.
│
├── New AI services
│   ├── ai-edge-api            → Analytics Assistant + Revenue features (SSE streaming)
│   ├── ai-worker              → nightly SQS jobs, forecast cache, insight digest
│   └── social-publisher-api   → draft + approve + publish to Instagram
│
└── Shared infrastructure
    ├── MongoDB
    ├── Redis            (cache + rate limiting)
    ├── SQS              (async job queue)
    ├── ai API
    └── Instagram Graph API
```

---

## Module 1 — Analytics Assistant

### What it does

Merchants ask plain-language questions directly from the existing Reports dashboard. The AI fetches structured data from `analytics-edge-api` and `employee-edge-api`, builds a context-rich prompt, and streams the answer back via Server-Sent Events.

Example questions:
- "Why were my sales down on Tuesday?"
- "Who were my top performers this week?"
- "How does this month compare to last month?"

### Data flow

```
Merchant portal (chat panel in Reports)
    │
    │ POST /ai/analytics/ask
    ▼
ai-edge-api
    ├──► analytics-edge-api  →  daily_store_summary, daily_store_items
    └──► employee-edge-api   →  shifts, performance logs
    │
    ▼
Prompt builder (inject JSON context + merchant question)
    │
    ▼
ai API  →  stream response  →  SSE back to portal
    │
Redis cache (keyed by storeId + question_hash + dateRange, TTL 1h)
```

### Key design decisions

- `ai-edge-api` calls existing edge APIs **server-side via internal cluster DNS** — never through the public internet
- Responses are streamed directly as SSE — no polling, no intermediate storage
- Redis caches identical queries: same merchant, same question, same date range = zero LLM cost on repeat
- The chat panel is a new component inside the existing Reports section — no new page or route needed in the portal

### API contract

```
POST /ai/analytics/ask
Authorization: Bearer <merchant_jwt>

{
  "storeId": "store_abc",
  "question": "Why were my sales down on Tuesday?",
  "dateRange": { "from": "2026-06-01", "to": "2026-06-23" }
}

Response: text/event-stream
data: {"delta": "Your sales on Tuesday..."}
data: {"delta": " were 18% lower than..."}
data: [DONE]
```

---

## Module 2 — Revenue Forecasting

### What it does

The AI predicts expected revenue for the next 7 days, broken down by day, using historical patterns and day-of-week seasonality. A confidence gate prevents bad predictions from reaching merchants.

Example output:
> "Based on your last 6 months, expect $4,200–$4,800 this Friday. Your weekend pattern shows a consistent 35% spike — factor that into staffing and purchasing."

### Data flow

**Nightly path (async, runs once per merchant per night)**

```
EventBridge cron  →  SQS (one message per active store)
    │
    ▼
ai-worker
    ├── pulls last 90 days of daily_store_summary from analyticsDB
    ├── computes day-of-week averages, week-over-week trend, standard deviation
    ├── enriches with upcoming public holidays for merchant region
    ├── calls ai API → generates forecast with confidence range
    └── stores result in MongoDB forecast_cache (TTL 24h)
```

**On-demand path (merchant opens forecast panel)**

```
Merchant portal (forecast panel)
    │
    │ GET /ai/forecast?storeId=store_abc
    ▼
ai-edge-api  →  reads from forecast_cache  →  serves pre-computed result
```

**Confidence gate (checked before serving)**

```
Accuracy check: back-test prediction vs actuals for last week
    ├── above 75% accuracy  →  show forecast to merchant
    └── below threshold     →  show "not enough data yet" message
```

### Key design decisions

- The LLM call happens **once per merchant per night** inside `ai-worker` — zero additional LLM cost when the merchant checks the panel multiple times
- `ai-edge-api` never calls ai on-demand for forecasts — it only reads from `forecast_cache`
- The confidence gate suppresses forecasts for new merchants (< 90 days data) or merchants with anomalous history
- Outputs are always shown as **ranges**, never single numbers — builds merchant trust and accounts for model uncertainty

### MongoDB: `forecast_cache` schema

```typescript
{
  _id: ObjectId,
  tenantId: string,
  storeId: string,
  generatedAt: Date,
  expiresAt: Date,           // TTL index: 24h
  accuracy: number,          // back-test score (0–1)
  forecast: {
    days: [
      {
        date: string,        // "2026-06-24"
        low: number,
        high: number,
        confidence: "high" | "medium" | "low"
      }
    ],
    narrative: string        // LLM-generated plain-language summary
  }
}
```

---

## Module 3 — Social Publisher

### What it does

A human-in-the-loop AI module that drafts Instagram posts from selected menu items. The merchant has full control — the AI is a smart copywriter, not an autonomous publisher.

Core workflow:
1. **Select** — merchant picks one or more items from their menu catalog
2. **Draft** — `social-publisher-api` fetches item details and generates an optimized Instagram caption via LLM
3. **Review & approve** — merchant sees a preview, edits freely, then validates
4. **Publish** — platform pushes the finalized caption + item images to Instagram via the Graph API

### Data flow

```
Merchant portal (select menu items)
    │
    │ POST /social/draft
    ▼
social-publisher-api
    │
    ├──► menus-edge-api  →  name, price, description, images
    │
    ▼
ai API  →  generate Instagram caption draft
    │
    ▼
Preview window (merchant edits + approves)
    │
    │ POST /social/publish
    ▼
social-publisher-api
    ├──► Instagram Graph API v18+ (Content Publishing API)
    └──► MongoDB posts_audit_log (store every draft + final + publish result)
```

### Key design decisions

- **Separate service** from `ai-edge-api` — handles OAuth tokens for external platforms, warrants its own security boundary, rate limit budget, and deployment lifecycle
- **OAuth tokens** (Instagram access tokens per merchant) are stored **encrypted** in MongoDB, never in Redis or environment variables
- Every draft, edit, and publish event is written to `posts_audit_log` — full traceability for the merchant and for compliance
- TikTok is **out of scope for now** — revisit after Instagram is stable (TikTok API requires video content natively, a different integration model)

### Meta App Review — important timing constraint

Meta requires App Review before your app can publish to merchant accounts you don't own. Submit the review request early — the process takes **2–4 weeks**. Build the draft + preview UI while waiting so development is not blocked.

### API contracts

```
POST /social/draft
Authorization: Bearer <merchant_jwt>

{
  "storeId": "store_abc",
  "itemIds": ["item_1", "item_2"],
  "platform": "instagram",
  "tone": "casual"           // optional: "casual" | "premium" | "promotional"
}

Response:
{
  "draftId": "draft_xyz",
  "caption": "🍗 Our Spicy Chicken Sandwich just got an upgrade...",
  "hashtags": ["#foodie", "#restaurant", "#spicy"],
  "imageUrls": ["https://..."]
}
```

```
POST /social/publish
Authorization: Bearer <merchant_jwt>

{
  "draftId": "draft_xyz",
  "finalCaption": "...",     // merchant's edited version
  "storeId": "store_abc",
  "platform": "instagram"
}

Response:
{
  "postId": "ig_post_123",
  "publishedAt": "2026-06-23T14:32:00Z",
  "permalink": "https://www.instagram.com/p/..."
}
```

---

## Shared: LLMClient wrapper

All three modules share a single internal `LLMClient` class. Build it once, use it everywhere.

```typescript
// packages/ai-core/src/LLMClient.ts

interface LLMCallOptions {
  feature: 'analytics_assistant' | 'revenue_forecast' | 'social_draft';
  tenantId: string;
  storeId: string;
  systemPrompt: string;
  userPrompt: string;
  maxTokens?: number;       // default: 1024
  stream?: boolean;         // default: false
}

class LLMClient {
  async call(options: LLMCallOptions): Promise<string>
  async stream(options: LLMCallOptions): AsyncIterable<string>
}
```

Responsibilities:
- Wraps ai SDK with automatic retries (3x, exponential backoff)
- Enforces `max_tokens` ceiling per call
- Logs token usage per `(tenantId, storeId, feature)` to MongoDB `ai_usage_log` — cost tracking per merchant
- Applies `cache_control` header for repeated system prompt blocks (saves ~80% on tokens for analytics assistant)
- Single place to swap model versions or providers in the future

---

## Technology stack

| Concern | Technology | Notes |
|---|---|---|
| API framework | Fastify (TypeScript) | Lighter than Express, native async streaming for SSE |
| LLM provider | ai API (`claude-sonnet-4-6`) | Via shared `LLMClient` wrapper |
| Async jobs | SQS + EventBridge cron | Already in stack, no new infra |
| Caching | Redis via `ioredis` | Already in stack |
| Primary database | MongoDB | `forecast_cache`, `posts_audit_log`, `ai_usage_log` |
| Instagram integration | Meta Graph API v18+ | Content Publishing API, OAuth per merchant |
| Streaming | Server-Sent Events (SSE) | Native in Fastify, no WebSocket needed |
| Deployment | EKS (existing cluster) | New Deployments + Services per module |
| Secret management | AWS Secrets Manager | Encrypted Instagram OAuth tokens |

---

## Build order

The sequence matters — each step unblocks the next.

**Step 1 — Foundation (week 1–2)**

Stand up the `ai-edge-api` skeleton on EKS:
- Auth middleware reused from existing services
- Redis connection + cache utility
- `LLMClient` wrapper with usage logging
- Health check endpoint

**Step 2 — Analytics Assistant (week 2–4)**

- Implement `POST /ai/analytics/ask` route
- Fetch from `analytics-edge-api` + `employee-edge-api` server-side
- Build the prompt template (system + data context + merchant question)
- Wire SSE streaming to the portal
- Add the chat panel UI component to the existing Reports dashboard

**Step 3 — ai-worker + forecast cache (week 4–6)**

- Build the SQS consumer in `ai-worker`
- Set up EventBridge nightly cron rule
- Implement nightly stats computation + ai call per store
- Write results to `forecast_cache` collection
- Run for 2+ weeks before building the UI — lets you validate accuracy first

**Step 4 — Revenue Forecasting panel (week 7–8)**

- Implement `GET /ai/forecast` route in `ai-edge-api`
- Add confidence gate logic (back-test check)
- Build the forecast panel UI in the merchant portal

**Step 5 — Social Publisher (parallel from week 3, gate on Meta App Review)**

- Submit Meta App Review immediately (week 1)
- Build `social-publisher-api` service skeleton
- Implement OAuth flow for merchants to connect their Instagram accounts
- Build `POST /social/draft` route + preview UI
- Build `POST /social/publish` route + audit logging
- Go live once App Review is approved

---

## Cost control strategy

Given the priority on minimal infra spend:

- **Prompt result caching** — Redis TTL cache by `(storeId + question_hash + dateRange)` prevents duplicate LLM calls
- **Nightly pre-computation** — Revenue forecasts computed once per merchant per night, served from cache on-demand
- **Per-merchant rate limiting** — daily call limits enforced in `ai-edge-api` via Redis counters
- **`max_tokens` ceiling** — hard limit of 1024 tokens per call enforced in `LLMClient`
- **System prompt caching** — `cache_control` header on repeated system prompt blocks saves ~80% on input tokens for the analytics assistant
- **Usage logging** — every LLM call logged with token counts per `(tenantId, storeId, feature)` — enables per-merchant cost attribution and anomaly alerting

---

*Document prepared for internal architecture review. Stack: Node.js / TypeScript, EKS, MongoDB, Redis, SQS, ai API.*
