# Module 1 — Analytics Assistant: Complete Build Plan

> **Status:** Ready to implement  
> **Service:** `ai-edge-api` (new)  
> **Portal integration:** new component inside existing Reports dashboard  
> **Blocking dependencies:** none — all data sources live and operational

---

## Table of Contents

1. [What we are building](#1-what-we-are-building)
2. [Architecture and data flow](#2-architecture-and-data-flow)
3. [Technology decisions](#3-technology-decisions)
4. [Package list with versions](#4-package-list-with-versions)
5. [Environment variables](#5-environment-variables)
6. [Service file structure](#6-service-file-structure)
7. [MongoDB schema](#7-mongodb-schema)
8. [Redis key schema](#8-redis-key-schema)
9. [API contracts](#9-api-contracts)
10. [Core modules — implementation design](#10-core-modules--implementation-design)
11. [SSE streaming — node:http pattern](#11-sse-streaming--nodehttp-pattern)
12. [Chat session and context management](#12-chat-session-and-context-management)
13. [LLMClient wrapper](#13-llmclient-wrapper)
14. [ContextBuilder — data assembly](#14-contextbuilder--data-assembly)
15. [Prompt design](#15-prompt-design)
16. [Response caching strategy](#16-response-caching-strategy)
17. [Portal integration](#17-portal-integration)
18. [Observability](#18-observability)
19. [Deployment](#19-deployment)
20. [Build order and milestones](#20-build-order-and-milestones)
21. [Phase 2 gRPC note](#21-phase-2-grpc-note)

---

## 1. What we are building

A merchant-facing natural-language assistant embedded in the Reports section of the merchant portal. The merchant types a question in plain language; the service fetches relevant structured data from `analytics-edge-api`, builds a context-rich prompt, and streams the answer back to the browser via Server-Sent Events (SSE).

**Example questions:**

- "Why were my sales down on Tuesday?"
- "Who were my top performers this week?"
- "How does this month compare to last month?"
- "What's my best-selling item in June?"

**Two modes:**

| Mode | When to use | Cost |
|------|-------------|------|
| **Stateless** | Quick one-off question from the Reports header | One LLM call, no session overhead |
| **Session** | Follow-up conversation in the chat panel | Conversation history in Redis, context-compressed at 8k tokens |

---

## 2. Architecture and data flow

```
Merchant portal (Reports page — chat panel)
    │
    │  POST /api/v1/ai/analytics/ask
    │  Authorization: Bearer <merchant_jwt>  (passed through by gateway)
    ▼
ai-edge-api  (new service, port 3010)
    │
    ├─ [1] Extract storeId + question from request body
    ├─ [2] Check Redis cache (keyed by storeId + question_hash + dateRange)
    │       └─ cache hit → stream cached text response immediately (zero LLM cost)
    ├─ [3] Load session from Redis if sessionId present
    ├─ [4] ContextBuilder: internal HTTP calls to analytics-edge-api
    │       ├─ GET /analytics/orders?storeId=...&from=...&to=...
    │       ├─ GET /analytics/items/popular?storeId=...&from=...&to=...
    │       ├─ GET /analytics/orders/dashboard-summary?storeId=...
    │       └─ GET /analytics/employee-performance?storeId=...&from=...&to=...
    ├─ [5] Prompt builder: compose system + data context + conversation history + question
    ├─ [6] LLMClient.stream() → GPT-4o-mini via Vercel AI SDK
    ├─ [7] Stream deltas via SSE to portal
    ├─ [8] On stream complete → save full answer to Redis cache + update session
    └─ [9] Log token usage to MongoDB ai_usage_log
```

```
analytics-edge-api  (existing — never modified)
    │
    └─ answers internal HTTP calls from ai-edge-api
       (via cluster-internal DNS, not through public gateway)
```

---

## 3. Technology decisions

| Concern | Decision | Rationale |
|---------|----------|-----------|
| **Runtime** | Node 22, TypeScript, pnpm | Consistent with `analytics-edge-api`, `inventory-edge-api` |
| **HTTP server** | `node:http` (raw, same as existing services) | No Fastify — keeps stack uniform across the platform |
| **LLM provider** | OpenAI GPT-4o-mini | $0.15/1M input tokens — cheapest major model with solid structured-data Q&A quality |
| **AI SDK** | Vercel AI SDK (`ai` + `@ai-sdk/openai`) | Provider-agnostic; one-line swap to Gemini/Anthropic; built-in SSE streaming and token counting |
| **Streaming** | Server-Sent Events via raw `node:http` flush | No WebSocket needed; SSE is trivial with raw Node.js |
| **Cache** | Redis via `ioredis` (existing cluster instance) | Already in the stack; no new infra |
| **Session store** | Redis (same instance, different key prefix) | TTL-based, 30min inactivity expiry |
| **Primary DB** | MongoDB `aiDB` | Usage logs, session overflow storage |
| **Auth** | Trust gateway — no JWT re-validation inside service | Consistent with all other edge APIs |
| **Rate limiting** | Infrastructure ready (Redis counter), thresholds deferred | MVP: unlimited; thresholds configurable via env var |
| **Observability** | pino + OpenTelemetry + prom-client | Copy-paste from `inventory-edge-api` |

---

## 4. Package list with versions

### `package.json` — `ai-edge-api`

```json
{
  "name": "ai-edge-api",
  "version": "0.1.0",
  "description": "AI features for the QuickManage merchant portal — Analytics Assistant, streamed via SSE",
  "private": true,
  "type": "module",
  "scripts": {
    "dev":       "LOG_PRETTY=true LOG_LEVEL=debug NODE_ENV=development tsx --watch index.ts",
    "start":     "tsx index.ts",
    "build":     "esbuild index.ts --bundle --platform=node --target=node22 --format=esm --packages=external --outfile=index.js",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "ai":                                 "^4.3.0",
    "@ai-sdk/openai":                     "^1.3.0",
    "ioredis":                            "^5.3.2",
    "mongodb":                            "^7.1.1",
    "pino":                               "^10.3.1",
    "pino-pretty":                        "^13.1.3",
    "prom-client":                        "^15.1.3",
    "@opentelemetry/api":                 "^1.9.1",
    "@opentelemetry/api-logs":            "^0.214.0",
    "@opentelemetry/core":                "^2.6.1",
    "@opentelemetry/resources":           "^2.6.1",
    "@opentelemetry/sdk-trace-base":      "^2.6.1",
    "@opentelemetry/sdk-logs":            "^0.214.0",
    "@opentelemetry/otlp-transformer":    "^0.214.0",
    "@aws-sdk/client-secrets-manager":    "^3.1020.0"
  },
  "devDependencies": {
    "@types/node":   "^22.0.0",
    "esbuild":       "^0.27.0",
    "tsx":           "^4.19.0",
    "typescript":    "^5.0.0"
  }
}
```

**Key packages explained:**

| Package | Role |
|---------|------|
| `ai` | Vercel AI SDK core — streaming, token counting, tool calling interface |
| `@ai-sdk/openai` | OpenAI provider adapter for Vercel AI SDK |
| `ioredis` | Redis client — cache + session store |
| `mongodb` | `aiDB` — usage logs |
| `pino` | Structured JSON logging (matches platform standard) |
| `prom-client` | Prometheus metrics — `/metrics` endpoint for Kubernetes scraping |
| `@opentelemetry/*` | Traces + logs to Elastic APM (same config as other services) |
| `@aws-sdk/client-secrets-manager` | Load OpenAI API key from AWS Secrets Manager in production |
| `tsx` | Dev: TypeScript run without compile step |
| `esbuild` | Prod: fast ESM bundle for Docker image |

---

## 5. Environment variables

### `.env.example`

```bash
# ─── Server ────────────────────────────────────────────────────────────────
PORT=3010
NODE_ENV=development

# ─── OpenAI ────────────────────────────────────────────────────────────────
OPENAI_API_KEY=sk-...                         # injected from Secrets Manager in prod

# ─── MongoDB ───────────────────────────────────────────────────────────────
MONGODB_URI=mongodb://localhost:27017
AI_DB_NAME=aiDB

# ─── Redis ─────────────────────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379              # existing cluster instance in prod

# ─── Internal service URLs (cluster-internal DNS in EKS) ───────────────────
ANALYTICS_EDGE_API_URL=http://analytics-edge-api-svc:3002/api/v1

# ─── LLM settings ──────────────────────────────────────────────────────────
AI_MODEL=gpt-4o-mini                         # swap to gpt-4o or gemini-flash with no code change
AI_MAX_TOKENS=1024                           # hard ceiling per call
AI_TEMPERATURE=0.3                           # low temperature = factual, consistent answers

# ─── Cache & session ────────────────────────────────────────────────────────
AI_CACHE_TTL_SECONDS=3600                    # 1 hour for identical question + date range
AI_SESSION_TTL_SECONDS=1800                  # 30 min inactivity before session expires
AI_SESSION_MAX_TOKENS=8000                   # soft threshold: compress context at 8k
AI_SESSION_COMPRESS_TOKENS=1500              # target size after compression

# ─── Rate limiting (deferred — set 0 = unlimited for MVP) ───────────────────
AI_RATE_LIMIT_PER_STORE_DAILY=0             # 0 = unlimited; set e.g. 100 in prod later
AI_RATE_LIMIT_PER_USER_DAILY=0             # 0 = unlimited; per-user option for later

# ─── Observability ──────────────────────────────────────────────────────────
OTEL_ENABLED=false                           # true in production
OTEL_SERVICE_NAME=ai-edge-api
OTEL_EXPORTER_OTLP_ENDPOINT=https://...
OTEL_ENVIRONMENT=dev
LOG_LEVEL=info
LOG_PRETTY=false

# ─── AWS Secrets Manager (production) ───────────────────────────────────────
AWS_SECRET_OPENAI=quickmanage/prod/ai-edge-api/openai
AWS_SECRET_OTEL=quickmanage/dev/shared/otel
AWS_REGION=ca-central-1

# ─── CORS ───────────────────────────────────────────────────────────────────
CORS_ORIGIN=*                                # restrict to portal domain in prod
```

---

## 6. Service file structure

```
ai-edge-api/
├── index.ts                          # Boot: secrets → telemetry → Redis → MongoDB → HTTP server
├── package.json
├── tsconfig.json
├── .env.example
├── Dockerfile
│
├── src/
│   ├── router.ts                     # Dispatch /api/v1/ai/* to handlers
│   ├── logger.ts                     # pino — copy from inventory-edge-api (change service name)
│   ├── telemetry.ts                  # OTel — copy from inventory-edge-api (change service name)
│   ├── metrics.ts                    # prom-client counters + histograms
│   ├── secrets.ts                    # AWS Secrets Manager loader
│   ├── middleware/
│   │   └── cors.ts                   # copy from inventory-edge-api
│   │
│   ├── db/
│   │   └── client.ts                 # MongoDB aiDB connection
│   │
│   ├── redis/
│   │   └── client.ts                 # ioredis connection singleton
│   │
│   ├── handlers/
│   │   └── analytics.handler.ts      # POST /analytics/ask  (routes to service)
│   │
│   ├── services/
│   │   └── analyticsAssistant.service.ts  # orchestrates context + LLM + session + cache
│   │
│   ├── llm/
│   │   └── LLMClient.ts              # Vercel AI SDK wrapper — call + stream + usage logging
│   │
│   ├── context/
│   │   ├── ContextBuilder.ts         # fetch and shape data from analytics-edge-api
│   │   └── analyticsClient.ts        # HTTP client for analytics-edge-api (internal)
│   │
│   ├── session/
│   │   └── SessionManager.ts         # Redis session CRUD + context compression logic
│   │
│   ├── cache/
│   │   └── responseCache.ts          # Redis cache for deterministic questions
│   │
│   ├── prompts/
│   │   └── analyticsAssistant.ts     # System prompt + context injection template
│   │
│   ├── repositories/
│   │   └── usageLog.repository.ts    # MongoDB ai_usage_log writes
│   │
│   └── utils/
│       ├── params.ts                 # parseRequiredQueryString, ApiError (copy pattern)
│       ├── hash.ts                   # SHA-256 question hash for cache key
│       └── tokens.ts                 # Token counting utility (Vercel AI SDK)
│
└── scripts/
    └── create-indexes.ts             # MongoDB index creation for ai_usage_log
```

---

## 7. MongoDB schema

### Collection: `ai_usage_log`

Records every LLM call for cost attribution and anomaly detection.

```typescript
interface UsageLogDoc {
  _id:        ObjectId
  tenantId:   string          // from JWT (platform tenant)
  storeId:    string          // from request body
  userId:     string          // from JWT sub claim
  feature:    'analytics_assistant'
  model:      string          // 'gpt-4o-mini'
  mode:       'stateless' | 'session'
  sessionId:  string | null
  inputTokens:  number
  outputTokens: number
  totalTokens:  number
  cached:     boolean         // true if served from Redis cache (no LLM call)
  durationMs: number          // time from request to stream end
  question:   string          // truncated to 500 chars for storage
  timestamp:  Date
}

// Indexes
{ storeId: 1, timestamp: -1 }           // per-merchant cost query
{ tenantId: 1, timestamp: -1 }          // platform admin cost query
{ timestamp: 1 },  expireAfterSeconds: 7776000  // TTL: 90 days
```

---

## 8. Redis key schema

```
# Response cache (identical question + date range)
ai:cache:{storeId}:{question_hash}:{dateRangeHash}
  → full plain text response
  TTL: AI_CACHE_TTL_SECONDS (default 1h)

# Session (conversation history)
ai:session:{sessionId}
  → JSON of SessionDoc (see section 12)
  TTL: AI_SESSION_TTL_SECONDS (default 30min), refreshed on each turn

# Rate limiting (ready but unlimited in MVP)
ai:rate:{storeId}:{YYYY-MM-DD}
  → integer call count
  TTL: end of current UTC day

ai:rate:user:{userId}:{YYYY-MM-DD}
  → integer call count
  TTL: end of current UTC day
```

---

## 9. API contracts

### `POST /api/v1/ai/analytics/ask`

**Request headers:**
```
Authorization: Bearer <merchant_jwt>
Content-Type: application/json
```

**Request body:**
```typescript
interface AskRequest {
  storeId:    string
  question:   string                     // max 500 chars
  dateRange?: { from: string; to: string }  // ISO 8601 dates; default: last 30 days
  sessionId?: string                     // omit for stateless mode
}
```

**Response (streaming — text/event-stream):**
```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Session-ID: session_abc123             // always present (new or existing session)
X-Request-ID: req_xyz789

data: {"delta":"Your sales on Tuesday"}
data: {"delta":" were 18% lower than"}
data: {"delta":" Monday because..."}
data: [DONE]
```

**Error responses (before stream starts — JSON):**
```
400  { "error": "question is required" }
400  { "error": "storeId is required" }
429  { "error": "Daily limit reached for this store" }   // once rate limiting is enabled
500  { "error": "Internal Server Error" }
```

### `DELETE /api/v1/ai/analytics/session/:sessionId`

Clears a session from Redis. Called when the merchant closes the chat panel.

```
204 No Content   (session deleted or never existed — idempotent)
```

### `GET /healthz`, `GET /health/live`
```
200  { "status": "ok" }
```

### `GET /health/ready`
```
200  { "status": "ready" }
503  { "status": "not ready", "issues": ["Redis unreachable"] }
```

### `GET /metrics`
```
200  Prometheus text format
```

---

## 10. Core modules — implementation design

### `src/handlers/analytics.handler.ts`

```typescript
export async function askHandler(req: Request, res: ServerResponse): Promise<void> {
  const body = await parseJsonBody(req)

  const storeId   = body.storeId   as string | undefined
  const question  = body.question  as string | undefined
  const sessionId = body.sessionId as string | undefined
  const dateRange = body.dateRange as { from: string; to: string } | undefined

  if (!storeId)  { sendJson(res, 400, { error: 'storeId is required' });  return }
  if (!question) { sendJson(res, 400, { error: 'question is required' }); return }
  if (question.length > 500) { sendJson(res, 400, { error: 'question too long (max 500 chars)' }); return }

  // extract userId from the JWT sub claim passed via header (gateway already validated JWT)
  const userId = req.headers['x-user-id'] as string ?? 'unknown'

  await analyticsAssistantService.ask({
    storeId, userId, question, sessionId, dateRange, res
  })
}
```

### `src/services/analyticsAssistant.service.ts`

```typescript
interface AskOptions {
  storeId:    string
  userId:     string
  question:   string
  sessionId?: string
  dateRange?: { from: string; to: string }
  res:        ServerResponse
}

export async function ask(opts: AskOptions): Promise<void> {
  const { storeId, userId, question, res } = opts
  const range = opts.dateRange ?? defaultDateRange()

  // [1] Check response cache
  const cacheKey = buildCacheKey(storeId, question, range)
  const cached   = await responseCache.get(cacheKey)
  if (cached) {
    streamCachedResponse(res, cached, opts.sessionId)
    await usageLogRepo.insert({ storeId, userId, cached: true, ... })
    return
  }

  // [2] Load or create session
  const session = opts.sessionId
    ? await sessionManager.load(opts.sessionId)
    : sessionManager.createEmpty(storeId, userId)

  // [3] Build analytics context
  const context = await contextBuilder.build({ storeId, dateRange: range })

  // [4] Build prompt
  const messages = promptBuilder.build({ session, context, question })

  // [5] Stream LLM response
  const { stream, usage } = await llmClient.stream({
    feature: 'analytics_assistant',
    messages,
    storeId,
  })

  // [6] Start SSE
  res.writeHead(200, {
    'Content-Type':  'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection':    'keep-alive',
    'X-Session-ID':  session.sessionId,
    'X-Request-ID':  crypto.randomUUID(),
  })

  let fullAnswer = ''

  for await (const delta of stream) {
    fullAnswer += delta
    res.write(`data: ${JSON.stringify({ delta })}\n\n`)
  }

  res.write('data: [DONE]\n\n')
  res.end()

  // [7] Post-stream: cache + session update + usage log (fire and forget)
  await Promise.all([
    responseCache.set(cacheKey, fullAnswer),
    sessionManager.appendAndSave(session, question, fullAnswer),
    usageLogRepo.insert({ storeId, userId, ...usage, cached: false }),
  ])
}
```

---

## 11. SSE streaming — `node:http` pattern

The raw `node:http` server passes `IncomingMessage` + `ServerResponse` directly to handlers for SSE routes. The key difference from the standard JSON handler pattern used in other services: we do **not** wrap in `new Response(...)` for streaming endpoints.

```typescript
// index.ts — router branches on SSE vs JSON
const server = createServer(async (req, res) => {
  const url    = new URL(req.url!, `http://localhost:${PORT}`)
  const method = req.method ?? 'GET'

  // SSE routes — handled with native IncomingMessage/ServerResponse
  if (method === 'POST' && url.pathname === '/api/v1/ai/analytics/ask') {
    await analyticsHandler.askHandler(req, res)
    return
  }

  // All other routes — use the standard Response wrapper pattern
  const response = await dispatch(method, url, req)
  const headers: Record<string, string> = {}
  response.headers.forEach((v, k) => { headers[k] = v })
  res.writeHead(response.status, headers)
  res.end(Buffer.from(await response.arrayBuffer()))
})
```

**SSE chunked write utility:**
```typescript
export function sseWrite(res: ServerResponse, data: unknown): void {
  res.write(`data: ${JSON.stringify(data)}\n\n`)
}

export function sseDone(res: ServerResponse): void {
  res.write('data: [DONE]\n\n')
  res.end()
}

export function sseError(res: ServerResponse, message: string): void {
  res.write(`data: ${JSON.stringify({ error: message })}\n\n`)
  res.write('data: [DONE]\n\n')
  res.end()
}
```

---

## 12. Chat session and context management

### Session shape (stored in Redis as JSON)

```typescript
interface ConversationTurn {
  role:    'user' | 'assistant'
  content: string
  tokens:  number          // estimated tokens for this turn
}

interface SessionDoc {
  sessionId:   string
  storeId:     string
  userId:      string
  turns:       ConversationTurn[]
  totalTokens: number      // running sum of all turns
  createdAt:   number      // Unix ms
  updatedAt:   number      // Unix ms
}
```

### Context compression at 8k tokens

When `session.totalTokens` approaches `AI_SESSION_MAX_TOKENS` (8000), we compress rather than truncate. Compression preserves the meaning of earlier turns instead of silently dropping them.

```typescript
// src/session/SessionManager.ts

async function maybeCompress(session: SessionDoc): Promise<SessionDoc> {
  if (session.totalTokens < parseInt(process.env.AI_SESSION_MAX_TOKENS ?? '8000')) {
    return session
  }

  // Keep the last 2 turns untouched (most recent context)
  const recentTurns = session.turns.slice(-2)
  const oldTurns    = session.turns.slice(0, -2)

  if (oldTurns.length === 0) return session

  // Summarize old turns with a cheap LLM call
  const summary = await llmClient.call({
    feature: 'analytics_assistant',
    messages: [
      {
        role: 'system',
        content: 'Summarize the following conversation in 3-5 sentences. Preserve any specific numbers, dates, item names, and conclusions mentioned.',
      },
      {
        role: 'user',
        content: oldTurns.map(t => `${t.role}: ${t.content}`).join('\n'),
      },
    ],
    storeId:   session.storeId,
    maxTokens: 300,
  })

  const summaryTurn: ConversationTurn = {
    role:    'assistant',
    content: `[Earlier conversation summary: ${summary}]`,
    tokens:  estimateTokens(summary),
  }

  return {
    ...session,
    turns:       [summaryTurn, ...recentTurns],
    totalTokens: summaryTurn.tokens + recentTurns.reduce((sum, t) => sum + t.tokens, 0),
    updatedAt:   Date.now(),
  }
}
```

### Session lifecycle

```
POST /ask (no sessionId)
  → create new session in memory
  → respond with X-Session-ID header
  → portal stores sessionId in component state

POST /ask (with sessionId)
  → load session from Redis
  → if not found → create fresh (session expired or invalid)
  → compress if near token limit
  → append new turn after answer
  → save back to Redis with refreshed TTL

DELETE /session/:sessionId
  → del Redis key  (merchant closes chat panel)
```

---

## 13. LLMClient wrapper

```typescript
// src/llm/LLMClient.ts

import { createOpenAI }     from '@ai-sdk/openai'
import { streamText, generateText } from 'ai'

const openai = createOpenAI({ apiKey: process.env.OPENAI_API_KEY })
const MODEL  = process.env.AI_MODEL     ?? 'gpt-4o-mini'
const MAX_TOKENS = parseInt(process.env.AI_MAX_TOKENS ?? '1024')
const TEMPERATURE = parseFloat(process.env.AI_TEMPERATURE ?? '0.3')

type Message = { role: 'system' | 'user' | 'assistant'; content: string }

interface StreamOptions {
  feature:   'analytics_assistant'
  messages:  Message[]
  storeId:   string
  maxTokens?: number
}

interface CallOptions extends StreamOptions {
  maxTokens?: number
}

interface StreamResult {
  stream: AsyncIterable<string>
  usage:  Promise<{ inputTokens: number; outputTokens: number; totalTokens: number }>
}

export async function stream(opts: StreamOptions): Promise<StreamResult> {
  const result = await streamText({
    model:       openai(MODEL),
    messages:    opts.messages,
    maxTokens:   opts.maxTokens ?? MAX_TOKENS,
    temperature: TEMPERATURE,
  })

  return {
    stream: result.textStream,
    usage:  result.usage.then(u => ({
      inputTokens:  u.promptTokens,
      outputTokens: u.completionTokens,
      totalTokens:  u.totalTokens,
    })),
  }
}

export async function call(opts: CallOptions): Promise<string> {
  const result = await generateText({
    model:       openai(MODEL),
    messages:    opts.messages,
    maxTokens:   opts.maxTokens ?? MAX_TOKENS,
    temperature: TEMPERATURE,
  })
  return result.text
}
```

**Provider swap example (zero code change in callers):**
```typescript
// Switch to Gemini Flash: change one import + one line
import { createGoogleGenerativeAI } from '@ai-sdk/google'
const google = createGoogleGenerativeAI({ apiKey: process.env.GOOGLE_API_KEY })
// openai(MODEL)  →  google('gemini-2.0-flash')
```

---

## 14. ContextBuilder — data assembly

Fetches relevant analytics data from `analytics-edge-api` over internal HTTP. All calls run in parallel to minimize latency.

```typescript
// src/context/ContextBuilder.ts

interface AnalyticsContext {
  dailySummaries:      DailyStoreSummary[]    // revenue, orders, avg order value per day
  popularItems:        PopularItem[]           // top 10 by quantity sold
  dashboardSummary:    DashboardSummary        // today's live state
  employeePerformance: EmployeePerformance[]   // top performers (current week)
  dateRange:           { from: string; to: string }
}

export async function build(opts: {
  storeId:   string
  dateRange: { from: string; to: string }
}): Promise<AnalyticsContext> {
  const { storeId, dateRange } = opts
  const base = `${process.env.ANALYTICS_EDGE_API_URL}`

  const [dailySummaries, popularItems, dashboardSummary, employeePerformance] =
    await Promise.all([
      fetchInternal(`${base}/analytics/orders?storeId=${storeId}&from=${dateRange.from}&to=${dateRange.to}`),
      fetchInternal(`${base}/analytics/items/popular?storeId=${storeId}&from=${dateRange.from}&to=${dateRange.to}&limit=10`),
      fetchInternal(`${base}/analytics/orders/dashboard-summary?storeId=${storeId}`),
      fetchInternal(`${base}/analytics/employee-performance?storeId=${storeId}&from=${dateRange.from}&to=${dateRange.to}`),
    ])

  return { dailySummaries, popularItems, dashboardSummary, employeePerformance, dateRange }
}

async function fetchInternal(url: string): Promise<unknown> {
  const res = await fetch(url)
  if (!res.ok) throw new Error(`Context fetch failed: ${url} → ${res.status}`)
  return res.json()
}
```

**Phase 2 gRPC note:** Replace `fetchInternal` with a gRPC client — all callers stay unchanged because they only depend on the `AnalyticsContext` shape, not on the transport.

---

## 15. Prompt design

### System prompt

```typescript
// src/prompts/analyticsAssistant.ts

export function buildSystemPrompt(storeId: string): string {
  return `You are a business analytics assistant for a restaurant manager using QuickManage.
You help merchants understand their sales, staff performance, and business trends.

Rules:
- Answer in plain, direct language. No jargon.
- Always ground your answer in the data provided. Never invent numbers.
- When data shows a trend, explain what likely caused it if the data supports a conclusion.
- If the data does not contain enough information to answer, say so clearly rather than guessing.
- Keep responses concise: 2-4 short paragraphs at most unless the merchant asks for detail.
- Use percentages and absolute values together: "Revenue was $4,200 — up 15% vs last Tuesday."
- The merchant's store ID is: ${storeId}. All data provided is scoped to their store only.
- Today's date: ${new Date().toISOString().split('T')[0]}.`
}
```

### Context injection

```typescript
export function buildContextBlock(context: AnalyticsContext): string {
  return `
## Business data for the period ${context.dateRange.from} to ${context.dateRange.to}

### Daily revenue summary
${JSON.stringify(context.dailySummaries, null, 2)}

### Top 10 items by quantity sold
${JSON.stringify(context.popularItems, null, 2)}

### Today's live activity
${JSON.stringify(context.dashboardSummary, null, 2)}

### Employee performance (current period)
${JSON.stringify(context.employeePerformance, null, 2)}
`
}
```

### Full message array for LLM

```typescript
export function buildMessages(opts: {
  session:  SessionDoc | null
  context:  AnalyticsContext
  question: string
}): Message[] {
  const messages: Message[] = []

  // System: role + data context
  messages.push({
    role:    'system',
    content: buildSystemPrompt(opts.session?.storeId ?? '') + '\n\n' + buildContextBlock(opts.context),
  })

  // Session history (if session mode)
  if (opts.session) {
    for (const turn of opts.session.turns) {
      messages.push({ role: turn.role, content: turn.content })
    }
  }

  // Current question
  messages.push({ role: 'user', content: opts.question })

  return messages
}
```

---

## 16. Response caching strategy

Cache key: `SHA-256(storeId + normalizedQuestion + dateRange.from + dateRange.to)`

Question normalization before hashing: lowercase, trim, collapse whitespace. This catches "why were my sales down tuesday?" and "Why were my sales down Tuesday?" as the same cache hit.

```typescript
// src/cache/responseCache.ts
import { createHash } from 'node:crypto'

export function buildCacheKey(
  storeId:   string,
  question:  string,
  dateRange: { from: string; to: string }
): string {
  const normalized = question.toLowerCase().trim().replace(/\s+/g, ' ')
  const raw = `${storeId}|${normalized}|${dateRange.from}|${dateRange.to}`
  return `ai:cache:${createHash('sha256').update(raw).digest('hex')}`
}

export async function get(key: string): Promise<string | null> {
  return redis.get(key)
}

export async function set(key: string, value: string): Promise<void> {
  await redis.set(key, value, 'EX', parseInt(process.env.AI_CACHE_TTL_SECONDS ?? '3600'))
}
```

**What is NOT cached:** session-mode follow-up questions (they depend on prior turns). Only first-turn stateless questions or stateless session starts are cached.

---

## 17. Portal integration

### Where it lives

A new collapsible **AI Assistant panel** at the bottom of the existing `ReportsPage.jsx` — no new route, no new nav item. The panel floats as an overlay on wide screens or a drawer on mobile.

### New files in `quickmanage-merchant-portal`

```
src/components/AIAssistantPanel.jsx          # chat UI component
src/controllers/aiAssistant.controller.js    # SSE fetch + session state
```

### `aiAssistant.controller.js`

```javascript
export async function askAnalyticsAssistant({
  storeId,
  question,
  dateRange,
  sessionId,
  onDelta,
  onDone,
  onError,
}) {
  const response = await fetch(`${VITE_BACKEND_API}/ai/analytics/ask`, {
    method:  'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${getQmAccessToken()}` },
    body:    JSON.stringify({ storeId, question, dateRange, sessionId }),
  })

  if (!response.ok) {
    const err = await response.json()
    onError(err.error ?? 'Something went wrong')
    return null
  }

  const returnedSessionId = response.headers.get('X-Session-ID')

  const reader = response.body.getReader()
  const decoder = new TextDecoder()

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    const text = decoder.decode(value)
    const lines = text.split('\n')
    for (const line of lines) {
      if (!line.startsWith('data: ')) continue
      const payload = line.slice(6).trim()
      if (payload === '[DONE]') { onDone(); return returnedSessionId }
      try {
        const { delta, error } = JSON.parse(payload)
        if (error) onError(error)
        else if (delta) onDelta(delta)
      } catch { /* skip malformed chunks */ }
    }
  }

  return returnedSessionId
}
```

### `AIAssistantPanel.jsx` — key behaviours

- Stores `sessionId` in `useState` — cleared on panel close → `DELETE /session/:sessionId`
- `isStreaming` state disables the input and send button during a response
- Renders a typing indicator (animated dots) while waiting for first delta
- Appends delta chunks to the current answer in real time
- "New conversation" button clears `sessionId` and local messages
- Date range is picked up from the existing Reports date picker state

### Portal `.env` addition

```bash
# No new env var needed — uses the existing VITE_BACKEND_API gateway
# The gateway routes /api/v1/ai/* to ai-edge-api
```

---

## 18. Observability

### `src/metrics.ts`

```typescript
export const aiRequestsTotal = new Counter({
  name: 'ai_requests_total',
  help: 'Total analytics assistant requests',
  labelNames: ['mode', 'cached'],   // mode: stateless|session, cached: true|false
})

export const aiStreamDurationSeconds = new Histogram({
  name:    'ai_stream_duration_seconds',
  help:    'Time from request to stream end',
  buckets: [0.5, 1, 2, 5, 10, 20, 30],
})

export const aiTokensTotal = new Counter({
  name:       'ai_tokens_total',
  help:       'Total LLM tokens consumed',
  labelNames: ['type'],   // type: input|output
})

export const aiContextBuildDurationSeconds = new Histogram({
  name:    'ai_context_build_duration_seconds',
  help:    'Time to fetch data from analytics-edge-api',
  buckets: [0.05, 0.1, 0.25, 0.5, 1, 2],
})
```

### Pino log fields on every request

```json
{
  "requestId": "req_abc",
  "storeId": "store_xyz",
  "mode": "session",
  "cached": false,
  "contextBuildMs": 120,
  "streamDurationMs": 4200,
  "inputTokens": 1850,
  "outputTokens": 312,
  "model": "gpt-4o-mini"
}
```

---

## 19. Deployment

### `Dockerfile`

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:22-alpine
WORKDIR /app
COPY --from=build /app/index.js ./
COPY --from=build /app/node_modules ./node_modules
ENV NODE_ENV=production PORT=3010
EXPOSE 3010
CMD ["node", "index.js"]
```

### `platform-gitops/apps/base/ai-edge-api/deployment.yaml` (sketch)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-edge-api-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-edge-api
  template:
    metadata:
      labels:
        app: ai-edge-api
    spec:
      containers:
        - name: ai-edge-api
          image: <ECR>/ai-edge-api:latest
          ports:
            - containerPort: 3010
          envFrom:
            - secretRef:
                name: ai-edge-api-secrets    # OPENAI_API_KEY, MONGODB_URI, REDIS_URL
          env:
            - name: PORT
              value: "3010"
            - name: NODE_ENV
              value: production
            - name: ANALYTICS_EDGE_API_URL
              value: http://analytics-edge-api-svc:3002/api/v1
          livenessProbe:
            httpGet: { path: /healthz, port: 3010 }
            periodSeconds: 20
          readinessProbe:
            httpGet: { path: /health/ready, port: 3010 }
            periodSeconds: 10
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 512Mi }
```

### Gateway routing addition

Add a rule to the existing ALB / API Gateway to route `/api/v1/ai/*` → `ai-edge-api-svc:3010`.

---

## 20. Build order and milestones

### Week 1 — Foundation (no LLM yet)

- [ ] Create `ai-edge-api` folder, `package.json`, `tsconfig.json`
- [ ] Copy `logger.ts`, `telemetry.ts`, `secrets.ts`, `cors.ts` from `inventory-edge-api` (change service name)
- [ ] Wire `index.ts` (boot sequence: secrets → telemetry → Redis → MongoDB → HTTP)
- [ ] Implement `/healthz`, `/health/ready`, `/metrics` endpoints
- [ ] Implement Redis client singleton (`src/redis/client.ts`)
- [ ] Implement MongoDB `aiDB` client (`src/db/client.ts`)
- [ ] Write `scripts/create-indexes.ts` and run against dev DB
- [ ] Write `Dockerfile`, confirm local Docker build works
- [ ] Smoke test: `curl http://localhost:3010/healthz` → `{"status":"ok"}`

### Week 2 — Context and prompt (no streaming yet)

- [ ] Implement `ContextBuilder` — fetch from `analytics-edge-api`, confirm data shape
- [ ] Implement `LLMClient.call()` (non-streaming first, easier to test)
- [ ] Implement prompt builder + test with a hardcoded question
- [ ] Wire `POST /ai/analytics/ask` as a non-streaming JSON response
- [ ] Verify full loop: question → context → LLM call → JSON response

### Week 3 — Streaming + session

- [ ] Switch `POST /ai/analytics/ask` to SSE streaming (`LLMClient.stream()`)
- [ ] Implement `SessionManager` (load, append, save, compress)
- [ ] Implement `responseCache` (Redis set/get)
- [ ] Implement `usageLogRepo` (MongoDB insert)
- [ ] End-to-end test: session conversation with 3+ turns + verify compression triggers

### Week 4 — Portal UI

- [ ] Build `AIAssistantPanel.jsx` in merchant portal
- [ ] Build `aiAssistant.controller.js` (SSE fetch + session ID management)
- [ ] Wire panel to Reports page (collapsible, date range auto-filled)
- [ ] Test: stateless mode, session mode, panel close clears session

### Week 5 — Hardening

- [ ] Add rate limit infrastructure (Redis counters), leave limits at 0/unlimited
- [ ] Add error handling: LLM timeout, analytics-edge-api unreachable
- [ ] Add metrics to Prometheus and verify scraping in EKS
- [ ] Add `ai-edge-api` to `platform-gitops` kustomization
- [ ] Deploy to dev environment, end-to-end test against real data

---

## 21. Phase 2 gRPC note

If context-building latency (step [4] in the data flow) becomes a bottleneck — meaning `analyticsClient.ts` parallel fetches are consistently above 300ms — the remedy is gRPC:

1. Define `.proto` contracts for the 4 queries `ContextBuilder` makes
2. Add a gRPC server to `analytics-edge-api` (TypeScript, `@grpc/grpc-js`)
3. Update only `src/context/analyticsClient.ts` in `ai-edge-api` — no changes to any other file

gRPC benefits at that point: binary protocol (faster than JSON), typed contracts (no runtime type errors), multiplexed streams. Not worth the complexity until measured.

---

*Plan version: 1.0 — Module 1 (Analytics Assistant) — ready to implement. See `QuickManage_AI_Architecture.md` for Module 2 (Revenue Forecasting) and Module 3 (Social Publisher).*
