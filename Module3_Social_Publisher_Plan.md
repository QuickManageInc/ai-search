# Module 3 — Social Publisher: Complete Build Plan

> **Status:** Partially implemented — draft + poster pipeline live; publish + OAuth + portal UI remaining  
> **Service:** `social-publisher-api` (exists, port 8095)  
> **Portal integration:** new panel in Marketing / Social section (planned)  
> **Blocking dependencies:** Meta App Review (2–4 weeks) for Instagram publish to merchant accounts

---

## Table of Contents

1. [What we are building](#1-what-we-are-building)
2. [Architecture and data flow](#2-architecture-and-data-flow)
3. [Technology decisions](#3-technology-decisions)
4. [Package list with versions](#4-package-list-with-versions)
5. [Environment variables](#5-environment-variables)
6. [Service file structure](#6-service-file-structure)
7. [MongoDB schema](#7-mongodb-schema)
8. [API contracts](#8-api-contracts)
9. [Core modules — implementation design](#9-core-modules--implementation-design)
10. [Poster generation pipeline](#10-poster-generation-pipeline)
11. [LLMClient wrapper](#11-llmclient-wrapper)
12. [Prompt design](#12-prompt-design)
13. [Instagram OAuth and publish flow](#13-instagram-oauth-and-publish-flow)
14. [Audit logging](#14-audit-logging)
15. [Portal integration](#15-portal-integration)
16. [Observability](#16-observability)
17. [Deployment](#17-deployment)
18. [Build order and milestones](#18-build-order-and-milestones)
19. [Meta App Review — timing constraint](#19-meta-app-review--timing-constraint)

---

## 1. What we are building

A human-in-the-loop AI module that drafts Instagram posts from a merchant's menu items. The merchant stays in control — the AI is a smart copywriter and graphic designer, not an autonomous publisher.

**Core workflow:**

| Step | Actor | Action |
|------|-------|--------|
| 1. Select | Merchant | Picks a menu item and writes a creative brief |
| 2. Draft | `social-publisher-api` | Fetches item details, generates caption + hashtags via LLM |
| 3. Poster (optional) | `social-publisher-api` | Generates AI background + composites food photo, text, and promo pricing |
| 4. Review | Merchant | Edits caption, previews poster, approves |
| 5. Publish | `social-publisher-api` | Pushes finalized image + caption to Instagram via Graph API |

**Example merchant brief:**

> "Weekend special — casual tone, highlight the spicy sauce, 20% off for Friday only"

**What is already built (MVP draft path):**

- `POST /api/v1/social/draft` — caption + hashtags from menu item + merchant prompt
- Optional promo poster generation (Pollinations background + Sharp compositing)
- `GET /posters/:draftId.png` — serves composed poster images
- MongoDB `post_drafts` persistence
- Health, readiness, and Prometheus metrics endpoints

**What remains:**

- Instagram OAuth connect flow per merchant
- `POST /api/v1/social/publish` — push approved draft to Instagram
- `posts_audit_log` collection — full draft → edit → publish traceability
- Merchant portal UI (item picker, brief input, preview, publish)
- EKS deployment + gateway routing

---

## 2. Architecture and data flow

### Draft path (implemented)

```
Merchant portal (Social Publisher panel)
    │
    │  POST /api/v1/social/draft
    │  Authorization: Bearer <merchant_jwt>  (passed through by gateway)
    ▼
social-publisher-api  (port 8095)
    │
    ├─ [1] Validate storeId, itemId, prompt
    ├─ [2] menusClient.fetchMenuItem() → name, description, price, imageUrl
    ├─ [3] resolvePromo() → regular/sale price for poster overlay
    ├─ [4] buildMessages() → system + user prompt for LLM
    ├─ [5] LLMClient.callObject() → structured caption + hashtags (+ poster plan if generatePoster)
    ├─ [6] composePoster() → Pollinations background + Sharp composite (if generatePoster)
    ├─ [7] insertDraft() → MongoDB post_drafts
    └─ [8] Return draftId, caption, hashtags, imageUrl, posterUrl
```

```
menus-edge-api  (existing — never modified)
    │
    └─ GET /menus/items/:itemId?storeId=...
       (cluster-internal DNS in prod)
```

### Publish path (planned)

```
Merchant portal (preview + approve)
    │
    │  POST /api/v1/social/publish
    ▼
social-publisher-api
    │
    ├─ [1] Load draft from post_drafts by draftId
    ├─ [2] Validate merchant's finalCaption (edited version)
    ├─ [3] Load encrypted Instagram token from instagram_connections
    ├─ [4] Upload image to Instagram (posterUrl or item imageUrl)
    ├─ [5] Create media container → publish via Graph API
    ├─ [6] Update post_drafts status → 'published'
    ├─ [7] Insert posts_audit_log entry
    └─ [8] Return postId, permalink, publishedAt
```

```
Instagram Graph API v18+  (Content Publishing API)
    │
    ├─ POST /{ig-user-id}/media        (create container)
    └─ POST /{ig-user-id}/media_publish
```

### OAuth connect path (planned)

```
Merchant portal → "Connect Instagram"
    │
    │  GET /api/v1/social/oauth/instagram/start?storeId=...
    ▼
social-publisher-api  →  redirect to Meta OAuth
    │
    │  callback: GET /api/v1/social/oauth/instagram/callback?code=...
    ▼
Exchange code → long-lived token → encrypt → store in instagram_connections
    │
    └─ Redirect back to portal with success/error
```

---

## 3. Technology decisions

| Concern | Decision | Rationale |
|---------|----------|-----------|
| **Runtime** | Node 22, TypeScript, npm | Service already scaffolded; consistent with other edge APIs |
| **HTTP server** | `node:http` (raw) | Same pattern as `inventory-edge-api`, `analytics-edge-api` |
| **LLM provider** | Google Gemini (`gemini-3.5-flash`) | Structured output via `generateObject`; lower cost than GPT-4o for creative copy |
| **AI SDK** | Vercel AI SDK (`ai` + `@ai-sdk/google`) | `generateObject` with Zod schemas — reliable JSON for caption + poster plan |
| **Structured output** | Zod schemas + `generateObject` | Eliminates fragile JSON parsing; poster plan fields are typed |
| **Image compositing** | `sharp` | Fast PNG compositing; SVG text overlays; no Canvas native dep issues |
| **Background art** | Pollinations API (`flux` model) | AI-generated backgrounds from LLM `backgroundPrompt`; gradient fallback on failure |
| **Primary DB** | MongoDB `socialDB` | Drafts, OAuth tokens (encrypted), audit log |
| **Poster storage** | Local filesystem (`storage/posters/`) | MVP; migrate to S3 + CloudFront in prod for Instagram URL requirements |
| **Auth** | Trust gateway — no JWT re-validation inside service | Consistent with all other edge APIs |
| **OAuth tokens** | AES-256-GCM encrypted in MongoDB | Never in Redis or env vars; decrypted only at publish time |
| **Rate limiting** | Deferred — infrastructure-ready via env var | MVP: unlimited draft calls |
| **Observability** | pino + OpenTelemetry + prom-client | Already wired in service skeleton |

**Why a separate service from `ai-edge-api`:**

- Instagram OAuth tokens warrant their own security boundary
- Independent rate limit budget and deployment lifecycle
- Poster image generation is CPU-heavy — can scale replicas separately
- Publish failures should not affect analytics assistant availability

---

## 4. Package list with versions

### `package.json` — `social-publisher-api`

```json
{
  "name": "social-publisher-api",
  "version": "0.1.0",
  "description": "Social Publisher for QuickManage — Instagram caption drafts from menu items",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "esbuild index.ts --bundle --platform=node --target=node22 --format=esm --packages=external --outfile=index.js",
    "start": "tsx index.ts",
    "dev": "LOG_PRETTY=true LOG_LEVEL=debug NODE_ENV=development tsx --watch index.ts",
    "typecheck": "tsc --noEmit",
    "seed:indexes": "tsx scripts/create-indexes.ts"
  },
  "dependencies": {
    "@ai-sdk/google": "^1.2.0",
    "@aws-sdk/client-secrets-manager": "^3.1020.0",
    "@opentelemetry/api": "^1.9.1",
    "@opentelemetry/api-logs": "^0.214.0",
    "@opentelemetry/core": "^2.6.1",
    "@opentelemetry/otlp-transformer": "^0.214.0",
    "@opentelemetry/resources": "^2.6.1",
    "@opentelemetry/sdk-logs": "^0.214.0",
    "@opentelemetry/sdk-trace-base": "^2.6.1",
    "ai": "^4.3.0",
    "mongodb": "^7.1.1",
    "pino": "^10.3.1",
    "pino-pretty": "^13.1.3",
    "prom-client": "^15.1.3",
    "sharp": "^0.34.3",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "esbuild": "^0.27.0",
    "tsx": "^4.19.0",
    "typescript": "^5.0.0"
  }
}
```

**Planned additions for publish phase:**

| Package | Role |
|---------|------|
| `@aws-sdk/client-s3` | Upload poster images to S3 for public Instagram-accessible URLs |
| `node:crypto` (built-in) | AES-256-GCM token encryption — no extra dep |

**Key packages explained:**

| Package | Role |
|---------|------|
| `ai` + `@ai-sdk/google` | Structured LLM output via `generateObject` + Zod |
| `zod` | Draft response schemas — caption, hashtags, poster plan |
| `sharp` | Poster compositing — resize food photo, overlay SVG text, PNG output |
| `mongodb` | `socialDB` — drafts, connections, audit log |
| `prom-client` | Prometheus metrics — draft duration, token usage |
| `@aws-sdk/client-secrets-manager` | MongoDB URI + Google API key in production |

---

## 5. Environment variables

### `.env.example`

```bash
# ─── Server ────────────────────────────────────────────────────────────────
PORT=8095
NODE_ENV=development

# ─── Google Gemini ─────────────────────────────────────────────────────────
GOOGLE_GENERATIVE_AI_API_KEY=...              # injected from Secrets Manager in prod

# ─── MongoDB ───────────────────────────────────────────────────────────────
MONGODB_URI=mongodb://localhost:27017
SOCIAL_DB_NAME=socialDB

# ─── Internal service URLs (cluster-internal DNS in EKS) ───────────────────
MENUS_EDGE_API_URL=http://menus-edge-api-svc:3004/api/v1
DEFAULT_LOCALE=en

# ─── LLM settings ──────────────────────────────────────────────────────────
AI_MODEL=gemini-3.5-flash
AI_MAX_TOKENS=600                            # default ceiling; draft handler overrides per mode
AI_TEMPERATURE=0.7                           # higher than analytics — creative copy benefits from variety

# ─── Poster generation ─────────────────────────────────────────────────────
POSTER_WIDTH=1080
POSTER_HEIGHT=1080
POSTER_STORAGE_DIR=storage/posters
POSTER_PUBLIC_BASE_URL=http://localhost:8095  # prod: https://social-api.quickmanage.ca

# ─── Pollinations (background art) ───────────────────────────────────────────
POLLINATIONS_API_URL=https://gen.pollinations.ai
POLLINATIONS_MODEL=flux
POLLINATIONS_API_KEY=                        # optional; omit for free tier
POLLINATIONS_TIMEOUT_MS=60000

# ─── Instagram OAuth (publish phase) ────────────────────────────────────────
META_APP_ID=
META_APP_SECRET=
META_REDIRECT_URI=http://localhost:8095/api/v1/social/oauth/instagram/callback
INSTAGRAM_TOKEN_ENCRYPTION_KEY=              # 32-byte hex key for AES-256-GCM

# ─── S3 poster hosting (publish phase — Instagram requires public image URL) ─
AWS_S3_POSTER_BUCKET=quickmanage-social-posters
AWS_S3_POSTER_REGION=ca-central-1
AWS_S3_POSTER_PUBLIC_URL=https://social-cdn.quickmanage.ca

# ─── Observability ──────────────────────────────────────────────────────────
OTEL_ENABLED=false
OTEL_SERVICE_NAME=social-publisher-api
OTEL_EXPORTER_OTLP_ENDPOINT=https://...
OTEL_ENVIRONMENT=dev
LOG_LEVEL=info
LOG_PRETTY=false
SLOW_QUERY_MS=500

# ─── AWS Secrets Manager (production) ───────────────────────────────────────
AWS_SECRET_MONGODB=quickmanage/prod/social-publisher-api/mongodb
AWS_SECRET_GOOGLE=quickmanage/prod/social-publisher-api/google
AWS_SECRET_META=quickmanage/prod/social-publisher-api/meta
AWS_SECRET_OTEL=quickmanage/dev/shared/otel
AWS_REGION=ca-central-1

# ─── CORS ───────────────────────────────────────────────────────────────────
CORS_ORIGIN=*                                # restrict to portal domain in prod
```

---

## 6. Service file structure

```
social-publisher-api/
├── index.ts                          # Boot: secrets → telemetry → MongoDB → HTTP server
├── package.json
├── tsconfig.json
├── .env.example
├── Dockerfile                        # TODO
│
├── storage/
│   └── posters/                      # Local poster PNGs (gitignored)
│
├── src/
│   ├── loadLocalEnv.ts               # Dev .env loader
│   ├── router.ts                     # Health + metrics dispatch
│   ├── logger.ts                     # pino
│   ├── telemetry.ts                  # OTel
│   ├── metrics.ts                    # prom-client counters + histograms
│   ├── secrets.ts                    # AWS Secrets Manager loader
│   ├── middleware/
│   │   └── cors.ts
│   │
│   ├── db/
│   │   ├── client.ts                 # MongoDB socialDB connection
│   │   └── ensureIndexes.ts          # post_drafts indexes
│   │
│   ├── handlers/
│   │   ├── draft.handler.ts          # POST /social/draft          ✅ implemented
│   │   ├── poster.handler.ts         # GET /posters/:id.png        ✅ implemented
│   │   ├── publish.handler.ts        # POST /social/publish          TODO
│   │   └── oauth.handler.ts          # Instagram OAuth start/callback TODO
│   │
│   ├── services/
│   │   ├── draftService.ts           # Orchestrates draft creation   ✅ implemented
│   │   ├── posterService.ts          # Sharp compositing pipeline    ✅ implemented
│   │   └── publishService.ts         # Instagram Graph API publish   TODO
│   │
│   ├── clients/
│   │   ├── menusClient.ts            # menus-edge-api HTTP client    ✅ implemented
│   │   ├── pollinationsClient.ts     # Background image fetch        ✅ implemented
│   │   └── instagramClient.ts        # Graph API media upload        TODO
│   │
│   ├── llm/
│   │   ├── LLMClient.ts              # Gemini generateObject wrapper ✅ implemented
│   │   └── draftSchemas.ts           # Zod schemas for structured output ✅
│   │
│   ├── prompts/
│   │   └── socialDraft.ts            # System + user prompt builder  ✅ implemented
│   │
│   ├── repositories/
│   │   ├── draft.repository.ts       # post_drafts CRUD              ✅ implemented
│   │   ├── connection.repository.ts  # instagram_connections         TODO
│   │   └── auditLog.repository.ts    # posts_audit_log               TODO
│   │
│   └── utils/
│       ├── params.ts                 # parseJsonBody, ApiError, prompt validation ✅
│       ├── computePromoPrice.ts      # Promo discount resolution     ✅ implemented
│       ├── parseDraftPlan.ts         # Normalize LLM poster plan     ✅ implemented
│       ├── parseCaption.ts           # Caption post-processing       ✅ implemented
│       ├── formatPrice.ts            # Cents → display string        ✅ implemented
│       ├── escapeXml.ts              # SVG text safety               ✅ implemented
│       └── crypto.ts                 # Token encrypt/decrypt           TODO
│
└── scripts/
    └── create-indexes.ts             # MongoDB index creation
```

---

## 7. MongoDB schema

### Collection: `post_drafts` (implemented)

Stores every generated draft before merchant review.

```typescript
interface DraftDocument {
  _id:                ObjectId
  storeId:            string
  itemId:             string
  itemTitle:          string
  itemImageUrl:       string | null
  merchantPrompt:     string
  caption:            string              // AI-generated caption
  hashtags:           string[]
  posterUrl:          string | null
  posterHeadline:     string | null
  posterSubheadline:  string | null
  posterBadge:        string | null
  posterRegularPrice: number | null       // cents
  posterSalePrice:    number | null       // cents
  posterStatus:       'ready' | 'failed' | 'skipped'
  model:              string              // e.g. 'gemini-3.5-flash'
  inputTokens:        number
  outputTokens:       number
  status:             'draft' | 'published' | 'discarded'
  createdAt:          Date
  publishedAt?:       Date                // set on publish
  instagramPostId?: string              // set on publish
  finalCaption?:      string              // merchant-edited version at publish time
}

// Indexes (implemented)
{ storeId: 1, createdAt: -1 }

// Indexes (planned)
{ status: 1, createdAt: -1 }
```

### Collection: `instagram_connections` (planned)

Encrypted OAuth tokens per merchant store.

```typescript
interface InstagramConnection {
  _id:              ObjectId
  storeId:          string              // unique per store
  tenantId:         string
  igUserId:         string              // Instagram Business account ID
  igUsername:       string
  encryptedToken:   string              // AES-256-GCM ciphertext (base64)
  tokenIv:          string              // initialization vector (base64)
  tokenExpiresAt:   Date                // long-lived token expiry (~60 days)
  connectedAt:      Date
  updatedAt:        Date
}

// Indexes
{ storeId: 1 }, { unique: true }
{ tokenExpiresAt: 1 }                   // background refresh job scans expiring tokens
```

### Collection: `posts_audit_log` (planned)

Full traceability for compliance and merchant support.

```typescript
interface AuditLogDoc {
  _id:            ObjectId
  storeId:        string
  tenantId:       string
  userId:         string              // from JWT sub claim
  draftId:        string
  event:          'draft_created' | 'draft_edited' | 'published' | 'publish_failed' | 'discarded'
  platform:       'instagram'
  aiCaption:      string              // original AI output
  finalCaption?:  string              // merchant-edited version (on publish)
  imageUrl?:      string
  posterUrl?:     string
  instagramPostId?: string
  permalink?:     string
  errorMessage?:  string              // on publish_failed
  model?:         string
  inputTokens?:   number
  outputTokens?:  number
  timestamp:      Date
}

// Indexes
{ storeId: 1, timestamp: -1 }
{ draftId: 1 }
{ timestamp: 1 }, { expireAfterSeconds: 7776000 }   // TTL: 90 days
```

---

## 8. API contracts

### `POST /api/v1/social/draft` (implemented)

**Request headers:**
```
Authorization: Bearer <merchant_jwt>
Content-Type: application/json
```

**Request body:**
```typescript
interface DraftRequest {
  storeId:        string
  itemId:         string
  prompt:         string                     // merchant creative brief, max 500 chars
  generatePoster?: boolean                    // default: true
  promo?: {
    discountPercent?: number                 // 1–99; mutually exclusive with salePrice
    salePrice?:       number                 // cents; must be < regular price
    label?:           string                 // e.g. "Weekend Special"
  }
}
```

**Response (200 JSON):**
```typescript
interface DraftResponse {
  draftId:   string
  caption:   string
  hashtags:  string[]
  imageUrl:  string | null                    // menu item photo
  posterUrl: string | null                    // composed poster PNG URL
  poster: {
    headline:     string | null
    subheadline:  string | null
    badge:        string | null
    regularPrice: number | null               // cents
    salePrice:    number | null               // cents
    status:       'ready' | 'failed' | 'skipped'
  } | null
  item: {
    id:    string
    title: string
    price: number | null                      // cents
  }
}
```

**Error responses:**
```
400  { "error": "storeId is required" }
400  { "error": "itemId is required" }
400  { "error": "prompt is required" }
400  { "error": "prompt too long (max 500 chars)" }
404  { "error": "Item not found" }
502  { "error": "Failed to fetch menu item" }
502  { "error": "AI response missing caption" }
500  { "error": "Internal Server Error" }
```

### `GET /posters/:draftId.png` (implemented)

Serves the composed poster PNG. `draftId` must be a 24-char hex MongoDB ObjectId.

```
200  image/png  (Cache-Control: public, max-age=86400)
400  { "error": "Invalid poster id" }
404  { "error": "Poster not found" }
```

### `POST /api/v1/social/publish` (planned)

**Request body:**
```typescript
interface PublishRequest {
  storeId:      string
  draftId:      string
  finalCaption: string                       // merchant-edited caption (may differ from AI draft)
  platform:     'instagram'                  // only instagram for MVP
}
```

**Response (200 JSON):**
```typescript
interface PublishResponse {
  postId:      string                        // Instagram media ID
  publishedAt: string                        // ISO 8601
  permalink:   string                        // https://www.instagram.com/p/...
}
```

**Error responses:**
```
400  { "error": "draftId is required" }
400  { "error": "finalCaption is required" }
404  { "error": "Draft not found" }
409  { "error": "Draft already published" }
422  { "error": "Instagram not connected for this store" }
422  { "error": "No image available to publish" }
502  { "error": "Instagram publish failed" }
```

### `GET /api/v1/social/oauth/instagram/start` (planned)

Redirects merchant to Meta OAuth consent screen.

```
302  Location: https://www.facebook.com/v18.0/dialog/oauth?...
```

Query params: `storeId` (required), `redirectUrl` (optional portal return URL)

### `GET /api/v1/social/oauth/instagram/callback` (planned)

Meta redirects here after consent. Exchanges code for token, stores encrypted connection, redirects to portal.

### `GET /api/v1/social/connection` (planned)

Check if a store has Instagram connected.

```typescript
// Response 200
{ connected: true, igUsername: "myrestaurant", expiresAt: "2026-08-15T00:00:00Z" }
// Response 200
{ connected: false }
```

### `GET /healthz`, `GET /health/live`
```
200  { "status": "ok" }
```

### `GET /health/ready`
```
200  { "status": "ready" }
503  { "status": "not ready", "issues": ["socialDB unreachable"] }
```

### `GET /metrics`
```
200  Prometheus text format
```

---

## 9. Core modules — implementation design

### `src/handlers/draft.handler.ts` (implemented)

```typescript
export async function draftHandler(
  req: IncomingMessage,
  res: ServerResponse,
  requestId: string
): Promise<void> {
  const body           = await parseJsonBody<DraftBody>(req)
  const storeId        = body.storeId?.trim() ?? ''
  const itemId         = body.itemId?.trim() ?? ''
  const prompt         = resolvePrompt(body.prompt)           // required, max 500 chars
  const generatePoster = resolveGeneratePoster(body.generatePoster)  // default true
  const promo          = body.promo as PromoInput | undefined

  // validate storeId, itemId → 400 if missing
  const result = await createDraft({ storeId, itemId, prompt, generatePoster, promo })
  sendJson(res, 200, result, requestId)
}
```

### `src/services/draftService.ts` (implemented)

```typescript
export async function createDraft(input: CreateDraftInput): Promise<DraftResponse> {
  const item  = await fetchMenuItem(input.storeId, input.itemId)
  const promo = resolvePromo(input.promo, item.price)
  const messages = buildMessages(input.prompt, item, promo, input.generatePoster)

  // Structured LLM call — schema selected by generatePoster flag
  const llm = input.generatePoster
    ? await callObject(draftWithPosterSchema, { feature: 'social_draft', messages, maxTokens: 2048 })
    : await callObject(draftCaptionOnlySchema, { feature: 'social_draft', messages, maxTokens: 1024 })

  const plan = input.generatePoster
    ? planFromWithPoster(llm.object)
    : planFromCaptionOnly(llm.object)

  // Optional poster compositing
  let posterUrl = null
  if (input.generatePoster && plan.poster) {
    const composed = await tryComposePoster(draftId, item, plan.poster, promo)
    posterUrl = composed.posterUrl
  }

  const savedId = await insertDraft({ ...fields, model: llm.model, inputTokens, outputTokens })
  return { draftId: savedId, caption: plan.caption, hashtags: plan.hashtags, ... }
}
```

### `src/services/publishService.ts` (planned)

```typescript
export async function publishDraft(input: PublishInput): Promise<PublishResponse> {
  const draft = await findDraftById(input.draftId, input.storeId)
  if (!draft) throw new ApiError(404, 'Draft not found')
  if (draft.status === 'published') throw new ApiError(409, 'Draft already published')

  const connection = await findConnection(input.storeId)
  if (!connection) throw new ApiError(422, 'Instagram not connected for this store')

  const imageUrl = draft.posterUrl ?? draft.itemImageUrl
  if (!imageUrl) throw new ApiError(422, 'No image available to publish')

  const token = decryptToken(connection.encryptedToken, connection.tokenIv)

  const { mediaId, permalink } = await instagramClient.publishImage({
    igUserId:  connection.igUserId,
    token,
    imageUrl,                              // must be publicly accessible HTTPS URL
    caption:   input.finalCaption + '\n\n' + draft.hashtags.join(' '),
  })

  await updateDraftStatus(input.draftId, 'published', {
    finalCaption: input.finalCaption,
    instagramPostId: mediaId,
    publishedAt: new Date(),
  })

  await auditLogRepo.insert({
    event: 'published',
    draftId: input.draftId,
    aiCaption: draft.caption,
    finalCaption: input.finalCaption,
    instagramPostId: mediaId,
    permalink,
    ...
  })

  return { postId: mediaId, publishedAt: new Date().toISOString(), permalink }
}
```

---

## 10. Poster generation pipeline

The poster path is a three-stage pipeline: LLM plans the layout → Pollinations generates background art → Sharp composites the final image.

```
LLM (draftWithPosterSchema)
    │
    ├─ caption, hashtags
    └─ poster: { headline, subheadline, badge, backgroundPrompt, colorTheme }
         │
         ▼
pollinationsClient.fetchBackgroundImage(backgroundPrompt)
    │  model=flux, 1080×1080, nologo=true
    │  on failure → gradient fallback from colorTheme.primary/secondary
         │
         ▼
posterService.composePoster()
    ├─ loadBackground()           → resize to 1080×1080
    ├─ loadFoodImage(item.imageUrl) → resize 560×560, centre crop
    ├─ buildOverlaySvg()          → headline, subheadline, badge, item label, price
    ├─ sharp(background).composite([food, overlay])
    └─ writeFile(storage/posters/{draftId}.png)
         │
         ▼
GET /posters/{draftId}.png  →  posterPublicUrl(draftId)
```

**Poster layout (1080×1080):**

```
┌─────────────────────────────────────┐
│  HEADLINE (62px bold white)         │  y=90
│  Subheadline (34px)                 │  y=145
│  [BADGE] (accent pill)              │  y=175
│                                     │
│         ┌─────────────┐             │
│         │  Food photo │             │  560×560, centred
│         │  560×560    │             │  y=280
│         └─────────────┘             │
│         Item title                  │  y=920
│                          ~~$12.99~~ │  strikethrough if promo
│                          $9.99      │  sale price, accent color
└─────────────────────────────────────┘
```

**Production note:** Instagram requires a publicly accessible HTTPS image URL at publish time. Local `POSTER_PUBLIC_BASE_URL` works for dev; prod must upload to S3 and return a CloudFront URL before calling the Graph API.

---

## 11. LLMClient wrapper

```typescript
// src/llm/LLMClient.ts — implemented

import { createGoogleGenerativeAI } from '@ai-sdk/google'
import { generateObject, generateText } from 'ai'

const MODEL       = process.env.AI_MODEL ?? 'gemini-3.5-flash'
const MAX_TOKENS  = parseInt(process.env.AI_MAX_TOKENS ?? '600', 10)
const TEMPERATURE = parseFloat(process.env.AI_TEMPERATURE ?? '0.7')

// call()        → plain text (not used in draft path)
// callObject()  → Zod-validated structured output (primary path)
```

**Why `generateObject` over `generateText` + JSON.parse:**

- Zod schema enforces exactly 5 hashtags, max caption length, required poster fields
- Eliminates the `parseDraftPlan` fallback path for the happy case
- `parseDraftPlan` remains as a safety net if structured output mode changes

**Token budgets per mode:**

| Mode | maxTokens | Typical usage |
|------|-----------|---------------|
| Caption only | 1024 | ~200 input, ~150 output |
| Caption + poster plan | 2048 | ~250 input, ~400 output |

---

## 12. Prompt design

### Caption-only system prompt

```
You are a social media copywriter for a restaurant.
Write an Instagram caption based on the merchant's creative brief and menu item below.

Return ONLY valid JSON with: caption (max 150 words), hashtags (exactly 5).

Rules:
- Follow the merchant brief for style and occasion
- Exactly 5 hashtags
- Do not invent or mention prices unless the brief explicitly includes them
```

### Poster system prompt

```
You are a social media copywriter and graphic designer for a restaurant.
Create an Instagram caption AND a poster layout plan.

Rules:
- caption: max 80 words (poster mode — keep it concise)
- hashtags: exactly 5
- headline/subheadline rendered ON the poster — short and punchy
- badge reflects promo when discount info provided; null if no promo
- backgroundPrompt: background art ONLY (no text, no logos, square 1:1, clean center for food)
- colorTheme: hex colors matching the brief
- Do NOT invent prices — pricing rendered separately from menu data
```

### User prompt template

```typescript
// Built by buildUserPrompt(prompt, item, promo)

Merchant brief: {prompt}

Item: {item.title}
Description: {item.description}

Promo label: Weekend Special
Discount active: regular price 1299 cents, sale price 1039 cents
Design the poster to highlight the discount prominently.
```

### Zod schemas (`src/llm/draftSchemas.ts`)

```typescript
export const draftCaptionOnlySchema = z.object({
  caption:  z.string().max(900),
  hashtags: z.array(z.string()).length(5),
})

export const draftWithPosterSchema = z.object({
  caption:  z.string().max(600),
  hashtags: z.array(z.string()).length(5),
  poster:   z.object({
    headline:         z.string().max(60),
    subheadline:      z.string().max(80),
    badge:            z.string().nullable(),
    backgroundPrompt: z.string().max(350),
    colorTheme:       z.object({
      primary: z.string(), secondary: z.string(), accent: z.string(),
    }),
  }),
})
```

---

## 13. Instagram OAuth and publish flow

### OAuth scopes required

```
instagram_basic
instagram_content_publish
pages_show_list
pages_read_engagement
```

### Token lifecycle

```
Short-lived token (1 hour)
    │  exchange via GET /oauth/access_token
    ▼
Long-lived token (~60 days)
    │  stored encrypted in instagram_connections
    │  background job refreshes at 7 days before expiry
    ▼
Refreshed long-lived token
```

### Publish sequence (Graph API v18+)

```typescript
// src/clients/instagramClient.ts — planned

async function publishImage(opts: {
  igUserId:  string
  token:     string
  imageUrl:  string    // public HTTPS — S3/CloudFront URL in prod
  caption:   string
}): Promise<{ mediaId: string; permalink: string }> {

  // Step 1: Create media container
  const container = await fetch(
    `https://graph.facebook.com/v18.0/${opts.igUserId}/media`,
    {
      method: 'POST',
      body: new URLSearchParams({
        image_url: opts.imageUrl,
        caption:   opts.caption,
        access_token: opts.token,
      }),
    }
  )

  const { id: creationId } = await container.json()

  // Step 2: Poll container status until FINISHED (max 30s)
  await waitForContainerReady(creationId, opts.token)

  // Step 3: Publish
  const publish = await fetch(
    `https://graph.facebook.com/v18.0/${opts.igUserId}/media_publish`,
    {
      method: 'POST',
      body: new URLSearchParams({
        creation_id:  creationId,
        access_token: opts.token,
      }),
    }
  )

  const { id: mediaId } = await publish.json()
  const permalink = await fetchPermalink(mediaId, opts.token)

  return { mediaId, permalink }
}
```

### Token encryption

```typescript
// src/utils/crypto.ts — planned

import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto'

const ALGORITHM = 'aes-256-gcm'
const KEY       = Buffer.from(process.env.INSTAGRAM_TOKEN_ENCRYPTION_KEY!, 'hex')

export function encryptToken(plaintext: string): { ciphertext: string; iv: string } {
  const iv     = randomBytes(12)
  const cipher = createCipheriv(ALGORITHM, KEY, iv)
  const enc    = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()])
  const tag    = cipher.getAuthTag()
  return {
    ciphertext: Buffer.concat([enc, tag]).toString('base64'),
    iv:         iv.toString('base64'),
  }
}

export function decryptToken(ciphertext: string, iv: string): string {
  const buf      = Buffer.from(ciphertext, 'base64')
  const ivBuf    = Buffer.from(iv, 'base64')
  const tag      = buf.subarray(buf.length - 16)
  const data     = buf.subarray(0, buf.length - 16)
  const decipher = createDecipheriv(ALGORITHM, KEY, ivBuf)
  decipher.setAuthTag(tag)
  return decipher.update(data) + decipher.final('utf8')
}
```

---

## 14. Audit logging

Every state change on a draft is recorded in `posts_audit_log` for merchant transparency and support.

| Event | When | Key fields |
|-------|------|------------|
| `draft_created` | After `createDraft` succeeds | aiCaption, hashtags, model, tokens |
| `draft_edited` | Merchant saves edits in portal (optional pre-publish) | finalCaption diff |
| `published` | Instagram publish succeeds | finalCaption, instagramPostId, permalink |
| `publish_failed` | Graph API error | errorMessage |
| `discarded` | Merchant abandons draft | — |

```typescript
// Fire-and-forget after draft creation (implemented pattern to add)
await auditLogRepo.insert({
  storeId: input.storeId,
  draftId: savedId,
  event:   'draft_created',
  platform: 'instagram',
  aiCaption: plan.caption,
  model: llmModel,
  inputTokens: llmUsage.inputTokens,
  outputTokens: llmUsage.outputTokens,
  timestamp: new Date(),
}).catch(err => logger.error({ err }, 'Audit log write failed'))
```

---

## 15. Portal integration

### Where it lives

A new **Social Publisher** panel in the merchant portal Marketing section — or as a modal launched from the menu item list ("Create social post" action on any item).

### New files in `quickmanage-merchant-portal` (planned)

```
src/pages/SocialPublisherPage.jsx           # main page: item picker + brief + preview
src/components/SocialDraftPreview.jsx       # caption editor + poster preview
src/components/InstagramConnectButton.jsx   # OAuth connect status + CTA
src/controllers/socialPublisher.controller.js
```

### `socialPublisher.controller.js`

```javascript
export async function createSocialDraft({ storeId, itemId, prompt, generatePoster, promo }) {
  const res = await fetch(`${VITE_BACKEND_API}/social/draft`, {
    method:  'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization:  `Bearer ${getQmAccessToken()}`,
    },
    body: JSON.stringify({ storeId, itemId, prompt, generatePoster, promo }),
  })

  if (!res.ok) {
    const err = await res.json()
    throw new Error(err.error ?? 'Draft generation failed')
  }

  return res.json()   // DraftResponse
}

export async function publishSocialDraft({ storeId, draftId, finalCaption }) {
  const res = await fetch(`${VITE_BACKEND_API}/social/publish`, {
    method:  'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization:  `Bearer ${getQmAccessToken()}`,
    },
    body: JSON.stringify({ storeId, draftId, finalCaption, platform: 'instagram' }),
  })

  if (!res.ok) {
    const err = await res.json()
    throw new Error(err.error ?? 'Publish failed')
  }

  return res.json()   // PublishResponse
}

export function getInstagramConnectUrl(storeId) {
  return `${VITE_BACKEND_API}/social/oauth/instagram/start?storeId=${storeId}`
}
```

### `SocialDraftPreview.jsx` — key behaviours

- Shows AI caption in an editable textarea — merchant can freely edit before publish
- Renders poster image from `posterUrl` when `poster.status === 'ready'`
- Shows promo pricing overlay info (regular vs sale price)
- "Regenerate" button calls `createSocialDraft` again with same inputs
- "Publish to Instagram" disabled until Instagram is connected
- Loading state during draft generation (LLM + poster can take 15–30s)
- Success state shows permalink after publish

### Portal `.env` addition

```bash
# No new env var needed — uses existing VITE_BACKEND_API gateway
# Gateway routes /api/v1/social/* → social-publisher-api-svc:8095
```

---

## 16. Observability

### `src/metrics.ts` (implemented)

```typescript
export const socialDraftTotal = new Counter({
  name: 'social_draft_requests_total',
  help: 'Total social draft generation requests',
})

export const socialDraftDurationSeconds = new Histogram({
  name:    'social_draft_duration_seconds',
  help:    'Time to generate and save a social draft',
  buckets: [0.5, 1, 2, 5, 10, 20, 30, 60],
})

export const socialTokensTotal = new Counter({
  name:       'social_tokens_total',
  help:       'Total LLM tokens consumed for social drafts',
  labelNames: ['type'],   // input | output
})
```

### Planned metrics (publish phase)

```typescript
export const socialPublishTotal = new Counter({
  name:       'social_publish_total',
  help:       'Instagram publish attempts',
  labelNames: ['status'],   // success | failed
})

export const socialPublishDurationSeconds = new Histogram({
  name:    'social_publish_duration_seconds',
  help:    'Time from publish request to Instagram confirmation',
  buckets: [1, 2, 5, 10, 20, 30, 60],
})
```

### Pino log fields on draft request

```json
{
  "requestId": "req_abc",
  "storeId": "store_xyz",
  "itemId": "item_123",
  "draftId": "674a1b2c3d4e5f6789012345",
  "posterStatus": "ready",
  "model": "gemini-3.5-flash",
  "inputTokens": 280,
  "outputTokens": 195
}
```

---

## 17. Deployment

### `Dockerfile`

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine
WORKDIR /app
# sharp requires vips runtime on alpine
RUN apk add --no-cache vips-dev
COPY --from=build /app/index.js ./
COPY --from=build /app/node_modules ./node_modules
RUN mkdir -p storage/posters
ENV NODE_ENV=production PORT=8095
EXPOSE 8095
CMD ["node", "index.js"]
```

### `platform-gitops/apps/base/social-publisher-api/deployment.yaml` (sketch)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: social-publisher-api-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: social-publisher-api
  template:
    metadata:
      labels:
        app: social-publisher-api
    spec:
      containers:
        - name: social-publisher-api
          image: <ECR>/social-publisher-api:latest
          ports:
            - containerPort: 8095
          envFrom:
            - secretRef:
                name: social-publisher-api-secrets
          env:
            - name: PORT
              value: "8095"
            - name: NODE_ENV
              value: production
            - name: MENUS_EDGE_API_URL
              value: http://menus-edge-api-svc:3004/api/v1
            - name: POSTER_PUBLIC_BASE_URL
              value: https://social-cdn.quickmanage.ca
            - name: POSTER_STORAGE_DIR
              value: /tmp/posters
          volumeMounts:
            - name: poster-tmp
              mountPath: /tmp/posters
          livenessProbe:
            httpGet: { path: /healthz, port: 8095 }
            periodSeconds: 20
          readinessProbe:
            httpGet: { path: /health/ready, port: 8095 }
            periodSeconds: 10
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits:   { cpu: 1000m, memory: 1Gi }   # poster compositing is CPU-heavy
      volumes:
        - name: poster-tmp
          emptyDir: {}
```

### Gateway routing addition

Add a rule to the existing ALB / API Gateway:

```
/api/v1/social/*  →  social-publisher-api-svc:8095
/posters/*        →  social-publisher-api-svc:8095   (public poster images)
```

---

## 18. Build order and milestones

### Phase 1 — Foundation ✅ (complete)

- [x] Create `social-publisher-api` folder, `package.json`, `tsconfig.json`
- [x] Wire `index.ts` boot sequence: secrets → telemetry → MongoDB → HTTP
- [x] Implement `/healthz`, `/health/ready`, `/metrics` endpoints
- [x] Implement MongoDB `socialDB` client + `post_drafts` indexes
- [x] Copy `logger.ts`, `telemetry.ts`, `secrets.ts`, `cors.ts` from platform pattern

### Phase 2 — Draft generation ✅ (complete)

- [x] Implement `menusClient` — fetch item from `menus-edge-api`
- [x] Implement `LLMClient.callObject()` with Zod schemas
- [x] Implement prompt builder (`socialDraft.ts`)
- [x] Implement `draftService` + `draft.handler`
- [x] Implement promo price resolution (`computePromoPrice.ts`)
- [x] Wire `POST /api/v1/social/draft`
- [x] End-to-end test: item + brief → caption + hashtags

### Phase 3 — Poster pipeline ✅ (complete)

- [x] Implement `pollinationsClient` — background image fetch
- [x] Implement `posterService` — Sharp compositing with SVG overlays
- [x] Implement `GET /posters/:draftId.png`
- [x] Graceful fallback: Pollinations failure → gradient background
- [x] Poster failure does not block draft — `posterStatus: 'failed'`, caption still returned

### Phase 4 — Meta App Review (start immediately — parallel track)

- [ ] Create Meta Developer App with Instagram Basic Display + Content Publishing permissions
- [ ] Submit App Review with screencast of draft → preview → publish flow
- [ ] Build draft + preview UI while waiting (publish button disabled until approved)
- [ ] Target: submit in week 1, approval by week 4–5

### Phase 5 — OAuth + publish backend

- [ ] Implement `crypto.ts` — AES-256-GCM token encrypt/decrypt
- [ ] Implement `connection.repository.ts` — `instagram_connections` CRUD
- [ ] Implement `oauth.handler.ts` — start + callback routes
- [ ] Implement `instagramClient.ts` — container create + publish + permalink fetch
- [ ] Implement `publishService.ts` + `publish.handler.ts`
- [ ] Implement `auditLog.repository.ts` — write on draft + publish events
- [ ] Add S3 upload for poster images (public HTTPS URL required by Instagram)
- [ ] Add `instagram_connections` + `posts_audit_log` indexes
- [ ] Token refresh background job (optional cron in `ai-worker` or inline on publish)

### Phase 6 — Portal UI

- [ ] Build `SocialPublisherPage.jsx` — item picker + creative brief form
- [ ] Build `SocialDraftPreview.jsx` — editable caption + poster preview
- [ ] Build `InstagramConnectButton.jsx` — OAuth status + connect CTA
- [ ] Build `socialPublisher.controller.js`
- [ ] Add "Create social post" action to menu item list
- [ ] Test: full flow item → brief → draft → edit → publish

### Phase 7 — Hardening + deploy

- [ ] Write `Dockerfile`, confirm local Docker build (sharp + vips)
- [ ] Add `social-publisher-api` to `platform-gitops` kustomization
- [ ] Configure gateway routing `/api/v1/social/*` and `/posters/*`
- [ ] Upload posters to S3 + CloudFront in prod
- [ ] Add publish metrics to Prometheus
- [ ] Deploy to dev environment, end-to-end test against real menu data
- [ ] Go live once Meta App Review approved

---

## 19. Meta App Review — timing constraint

Meta requires App Review before your app can publish to Instagram accounts you don't own (i.e., merchant accounts). **Submit the review request in week 1** — the process takes 2–4 weeks.

**What to submit:**

| Asset | Description |
|-------|-------------|
| Screencast | Merchant selects item → writes brief → sees AI draft → edits caption → publishes to Instagram |
| App description | "QuickManage Social Publisher helps restaurant merchants create and publish Instagram posts from their menu items using AI-generated captions and promotional posters" |
| Permissions justification | `instagram_content_publish` — required to post on behalf of connected merchant accounts |
| Test credentials | Demo merchant account with Instagram Business connected |

**What you can build while waiting:**

- Full draft + poster pipeline (no Meta dependency) ✅ already done
- Portal preview UI with "Publish" button disabled / "Pending Instagram approval" badge
- OAuth connect flow against Meta's test/sandbox mode
- All backend publish code tested against Meta's test Instagram account

**TikTok is out of scope** — revisit after Instagram is stable. TikTok API requires native video content, a fundamentally different integration model.

---

*Plan version: 1.0 — Module 3 (Social Publisher) — draft + poster pipeline implemented; publish + OAuth + portal UI remaining. See `QuickManage_AI_Architecture.md` for Modules 1 (Analytics Assistant) and 2 (Revenue Forecasting).*
